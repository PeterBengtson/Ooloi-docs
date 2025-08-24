# ADR-0025: Server Statistics Architecture

## Table of Contents

- [Status](#status)
- [Context](#context)
  - [Critical Need for Comprehensive Introspection](#critical-need-for-comprehensive-introspection)
  - [Current Limitations](#current-limitations)  
  - [Operational Requirements](#operational-requirements)
- [Decision](#decision)
- [Architecture](#architecture)
- [Complete Statistics Field Specification](#complete-statistics-field-specification)
  - [Server-Wide Statistics Structure](#server-wide-statistics-structure)
  - [Per-Client Statistics Structure](#per-client-statistics-structure)
  - [Raw vs. Derived Statistics Architecture](#raw-vs-derived-statistics-architecture)
  - [Statistics Collection Points](#statistics-collection-points)
  - [Performance Considerations](#performance-considerations)
- [Health Monitoring Integration](#health-monitoring-integration)
  - [Dual Health System Architecture](#dual-health-system-architecture)
  - [HTTP Statistics Endpoints](#http-statistics-endpoints)
  - [External Monitoring Tool Integration](#external-monitoring-tool-integration)
- [Implementation Phases](#implementation-phases)
- [Consequences](#consequences)
  - [Positive Outcomes](#positive-outcomes)
  - [Trade-offs and Considerations](#trade-offs-and-considerations)
  - [Mitigation Strategies](#mitigation-strategies)
- [Related ADRs](#related-adrs)
- [Future Considerations](#future-considerations)

## Status

Accepted 2025-08-24

## Context

Ooloi's gRPC architecture ([ADR-0002](0002-gRPC.md), [ADR-0024](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)) represents a **novel server architecture** combining:

1. **Client-to-server API requests**: STM-wrapped method calls for piece manipulation, queries, and mutations
2. **Server-to-client event streaming**: Queue-based real-time notifications for collaborative editing and system events

This is a **new, experimental server design** with unique characteristics:
- STM-gRPC transaction integration
- Per-client event queues with drop-oldest overflow handling  
- Real-time collaborative editing coordination
- Complex client lifecycle management with subscription tracking

### Critical Need for Comprehensive Introspection

The architecture's unique characteristics require operational insight:
- How do queue-based event streams behave under load?
- What are the performance characteristics of STM-gRPC integration?
- How do concurrent collaborative sessions scale?
- Which client usage patterns cause performance issues?

Current tests rely on opaque counts that provide limited behavioral insight:
```clojure
;; Limited information: What actually happened?
(count @client-events) => 3  ; Tells us nothing about event types, timing, failures
```

Statistics enable detailed test validation:
```clojure  
;; Detailed: Validates actual system behavior
(get-server-stat :server-events-sent) => 2
(get-client-stat client-id :server-events-received) => 2  
(get-client-stat client-id :queue-overflow-count) => 0
```

Without detailed statistics collection, operational issues, performance bottlenecks, and capacity planning become reactive rather than proactive.

### Current Limitations

The existing architecture lacks systematic statistics collection:
- No visibility into API call patterns (success/failure rates, performance characteristics)
- No tracking of event delivery performance or client queue health
- No aggregate metrics for capacity planning or operational monitoring
- No historical data for performance analysis or debugging

### Operational Requirements

Production operations require metrics for:
- **Performance monitoring**: API response times, event delivery latency
- **Capacity planning**: Peak client connections, memory usage patterns
- **Error analysis**: Failure rates, error categorization, debugging context
- **Resource utilization**: Queue sizes, thread pool usage, network throughput
- **Health monitoring**: System status, client health, service availability

## Decision

Implement comprehensive two-level statistics collection:

1. **Per-client statistics**: Operational metrics specific to individual client connections
2. **Server-wide statistics**: Aggregate metrics that survive client connection churn

Statistics will be collected in real-time during operation with minimal performance impact, accessible via both programmatic APIs and health monitoring endpoints.

## Architecture

## Complete Statistics Field Specification

### Server-Wide Statistics Structure

Add new `server-statistics` atom to component with aggregate visibility:

```clojure
{ ;; ==========================================
 ;; CONNECTION LIFECYCLE AGGREGATES (Zero Cost)
 ;; ==========================================
 :clients-connected-total 0                    ; Total connections since server start
 :clients-connected-current 0                  ; Currently active connections  
 :clients-connected-peak 0                     ; Peak concurrent connections
 :clients-disconnected-total 0                 ; Total disconnections
 :clients-disconnected-graceful 0              ; Clean disconnections
 :clients-disconnected-error 0                 ; Error-based disconnections
 :clients-disconnected-timeout 0               ; Timeout-based disconnections
 :connection-duration-total-ms 0               ; Aggregate connection time
 :shortest-session-ms Long/MAX_VALUE           ; Fastest client disconnect
 :longest-session-ms 0                         ; Longest client session
 
 ;; ==========================================
 ;; API CALL AGGREGATES (Zero Cost)
 ;; ==========================================
 :api-calls-total 0                            ; Total API calls processed
 :api-calls-success 0                          ; Successful API calls
 :api-calls-failure 0                          ; Failed API calls
 :api-concurrent-calls-current 0               ; Currently processing API calls
 :api-concurrent-calls-peak 0                  ; Max simultaneous API processing
 :api-slowest-call-ms 0                        ; Worst API response time ever
 :api-fastest-call-ms Long/MAX_VALUE           ; Best API response time ever
 
 ;; ==========================================
 ;; EVENT STREAMING AGGREGATES (Zero Cost)
 ;; ==========================================
 :server-events-sent 0                         ; Total server events broadcast
 :piece-events-sent 0                          ; Total piece events sent  
 :connect-events-sent 0                        ; Client connect notifications
 :disconnect-events-sent 0                     ; Client disconnect notifications
 :events-sent-total 0                          ; All event types combined
 :events-dropped-total 0                       ; Total events dropped (all clients)
 :event-queues-overflow-total 0                ; Total queue overflow incidents
 :event-queues-healthy-count 0                 ; Queues with no overflow
 :event-queues-warning-count 0                 ; Queues near capacity
 :event-queues-critical-count 0                ; Queues experiencing overflow
 :event-delivery-attempts 0                    ; Total event delivery attempts
 :event-delivery-successes 0                   ; Successful event deliveries
 
 ;; ==========================================
 ;; SYSTEM RESOURCE UTILIZATION (Zero Cost)
 ;; ==========================================  
 :bytes-transferred-total 0                    ; Total network traffic (all clients)
 :bytes-api-requests-total 0                   ; Bytes from API calls
 :bytes-api-responses-total 0                  ; Bytes in API responses  
 :bytes-events-total 0                         ; Bytes in event messages
 :largest-api-request-bytes 0                  ; Biggest API request ever
 :largest-api-response-bytes 0                 ; Biggest API response ever
 :largest-event-message-bytes 0                ; Biggest event message ever
 
 ;; ==========================================
 ;; ERROR TRACKING AND CATEGORIZATION (Zero Cost)
 ;; ==========================================
 :conversion-errors-total 0                    ; Protobuf conversion failures
 :serialization-errors-total 0                 ; Data serialization failures
 :network-errors-total 0                       ; Network-level errors
 :internal-errors-total 0                      ; Unexpected server errors  
 :timeout-errors-total 0                       ; Request timeout failures
 :client-errors-total 0                        ; Client-side error responses
 :server-errors-total 0                        ; Server-side error responses
 :last-error-time timestamp                    ; Most recent error
 :error-free-duration-ms 0                     ; Time since last error
 :longest-error-free-period-ms 0               ; Best error-free streak
 
 ;; ==========================================
 ;; COLLABORATIVE EDITING METRICS (Zero Cost)
 ;; ==========================================
 :piece-subscriptions-total 0                  ; Total piece subscriptions
 :piece-subscriptions-active 0                 ; Current piece subscriptions
 :piece-subscriptions-peak 0                   ; Peak piece subscriptions
 :most-subscribed-piece-count 0                ; Max clients on single piece
 :subscription-changes-total 0                 ; Total subscription add/remove operations
 
 ;; ==========================================  
 ;; SYSTEM TIMING AND METADATA (Zero Cost)
 ;; ==========================================
 :server-start-time timestamp                  ; When server started
 :server-restart-count 0                       ; Number of restarts
 
 ;; ==========================================
 ;; NOTES: Derived analytics computed on-demand for health endpoints  
 ;; Raw data above enables calculation of:
 ;; - :server-uptime-ms (current-time - server-start-time)
 ;; - :server-health-score (composite from all error rates and performance)
 ;; - :api-success-rate-overall (api-calls-success / api-calls-total) 
 ;; - :event-delivery-success-rate ((events-sent-total - events-dropped-total) / events-sent-total)
 ;; - :system-load-score (composite from resource utilization metrics)
 ;; - :performance-trend-24h (trend analysis from historical data)
 ;; - :collaborative-effectiveness (metrics derived from subscription patterns)
 ;; - :subscription-churn-rate-per-hour (subscription-changes-total / uptime-hours)
 ;; - :api-calls-per-hour (api-calls-total / uptime-hours)
 ;; - :api-peak-calls-per-minute (external tool analysis of call timing patterns)
 ;; - :api-method-usage-map (external tool aggregation from method labels)
 ;; - :grpc-error-breakdown-by-status (external tool status code analysis)
 ;; - :collaborative-sessions-active (calculated from current subscription data)
 ;; - :pieces-with-multiple-subscribers (derived from subscription state)
 ;; - :memory-usage-current-bytes (Runtime.getRuntime().totalMemory())
 ;; - :memory-usage-peak-bytes (external tool tracking of memory peaks)
 ;; - :thread-pool-active-count (ThreadPoolExecutor.getActiveCount())
 ;; - :thread-pool-usage-peak (external tool thread pool monitoring)
 ;; - :thread-pool-queue-size (ThreadPoolExecutor.getQueue().size())
 ;; - :queue-memory-usage-total-bytes (calculated estimate from queue sizes)
 ;; ==========================================
 }
```

### Per-Client Statistics Structure

Expand existing connection registry `:metadata` field with operational visibility:

```clojure
{:client-id "client-uuid"
 :observer stream-observer-ref
 :event-queue bounded-queue-ref  
 :piece-subscriptions #{set-of-piece-ids}
 :metadata {
           ;; ==========================================
           ;; CONNECTION LIFECYCLE (Zero Cost)
           ;; ==========================================
           :connected-at timestamp                    ; Connection establishment time
           :last-activity-time timestamp              ; Most recent API call or event
           
           ;; ==========================================
           ;; API CALL METRICS (Zero Cost)
           ;; ==========================================
           :api-calls-total 0                        ; Total API calls made
           :api-calls-success 0                      ; Successful API calls
           :api-calls-failure 0                      ; Failed API calls  
           :last-api-call timestamp                  ; Most recent API call time
           :last-api-method string                   ; Most recent method called
           :slowest-api-call-ms 0                    ; Worst response time
           :fastest-api-call-ms Long/MAX_VALUE       ; Best response time
           :concurrent-api-calls-current 0           ; Currently processing
           :concurrent-api-calls-peak 0              ; Max simultaneous calls
           
           ;; ==========================================
           ;; EVENT STREAMING METRICS (Zero Cost)
           ;; ==========================================
           :events-sent 0                            ; Total events delivered to client
           :events-dropped 0                         ; Events lost due to queue overflow
           :server-events-received 0                 ; Server-wide events received
           :piece-events-received 0                  ; Piece-specific events received
           :connect-events-received 0                ; Client connect notifications  
           :disconnect-events-received 0             ; Client disconnect notifications
           :last-event-time timestamp                ; Most recent event delivery
           :last-event-type keyword                  ; Type of most recent event
           
           ;; ==========================================
           ;; QUEUE HEALTH METRICS (Zero Cost)
           ;; ==========================================
           :queue-size-current 0                     ; Current queue depth
           :queue-size-peak 0                        ; Maximum queue depth reached
           :queue-overflow-count 0                   ; Number of overflow incidents  
           :queue-overflow-total-events-dropped 0    ; Events lost across all overflows
           :queue-offer-attempts 0                   ; Total queue insertions attempted
           :queue-offer-successes 0                  ; Successful queue insertions
           :queue-consumer-lag-ms 0                  ; Delay between queue and delivery
           
           ;; ==========================================
           ;; NETWORK PERFORMANCE (Zero Cost)
           ;; ==========================================
           :bytes-sent 0                             ; Total protobuf bytes to client
           :bytes-received 0                         ; Total protobuf bytes from client  
           :bytes-events 0                           ; Bytes consumed by event messages
           :bytes-api-requests 0                     ; Bytes from API calls
           :bytes-api-responses 0                    ; Bytes in API responses
           :largest-request-bytes 0                  ; Biggest API request  
           :largest-response-bytes 0                 ; Biggest API response
           :largest-event-bytes 0                    ; Biggest event message
           
           ;; ==========================================
           ;; ERROR TRACKING (Zero Cost)
           ;; ==========================================
           :network-errors 0                         ; Network-level failures
           :serialization-errors 0                   ; Protobuf conversion failures  
           :conversion-errors 0                      ; Clojure<->Protobuf failures
           :timeout-errors 0                         ; Request timeout failures
           :last-error-time timestamp                ; Most recent error
           :last-error-message string                ; Most recent error description
           
           ;; ==========================================
           ;; CLIENT BEHAVIOR PATTERNS (Zero Cost)
           ;; ==========================================
           :subscription-add-count 0                 ; Total pieces subscribed to
           :subscription-remove-count 0              ; Total pieces unsubscribed from  
           :peak-subscription-count 0                ; Maximum concurrent subscriptions  
           
           ;; ==========================================
           ;; NOTES: Derived analytics computed on-demand for health endpoints
           ;; Raw data above enables calculation of:
           ;; - :api-success-rate (success/total)
           ;; - :response-time-range-ms (slowest - fastest)
           ;; - :event-delivery-reliability ((sent - dropped) / sent)
           ;; - :queue-health-score (composite from queue metrics)
           ;; - :client-health-score (overall health composite) 
           ;; - :performance-trend-7d (trend analysis from historical data)
           ;; - :connection-duration-ms (current-time - connected-at)
           ;; - :api-call-frequency-hz (total-calls / connection-duration)
           ;; - :event-consumption-rate-hz (events-sent / connection-duration)
           ;; - :active-subscription-count (count piece-subscriptions)
           ;; - :api-success-rate (success/total)
           ;; - :api-method-usage-map (external tool aggregation from method labels)
           ;; - :session-pieces-accessed (derived from API call logs or subscription events)
           ;; ==========================================
           }}
```

### Raw vs. Derived Statistics Architecture

#### Design Principle: Store Raw Data, Compute Analytics On-Demand

**Atoms store only raw measurements** (zero/minimal collection cost):
```clojure
;; Raw data - minimal overhead to collect
:api-calls-total 15
:api-calls-success 14  
:events-dropped 2
:bytes-sent 4096
```

**Health endpoints compute derived metrics** when requested:
```clojure
;; GET /health/clients/{id} - calculated on-demand
(defn compute-client-analytics [raw-stats]
  {:api-success-rate (/ (:api-calls-success raw-stats) 
                       (:api-calls-total raw-stats))
   :response-time-range-ms (- (:slowest-api-call-ms raw-stats)
                             (:fastest-api-call-ms raw-stats))
   :event-delivery-reliability (- 1.0 (/ (:events-dropped raw-stats) 
                                         (:events-sent raw-stats)))
   :client-health-score (composite-health-calculation raw-stats)})
```

#### Benefits of This Architecture:

**Runtime Performance**:
- **Zero calculation overhead** during normal operation
- **No expensive aggregations** in hot paths  
- **Raw data collection** has minimal performance impact

**Data Freshness**:  
- **Always current**: Derived metrics computed from latest raw data
- **No stale calculations**: No cached derived values to become outdated
- **Consistent snapshots**: All derived metrics calculated from same raw data snapshot

**Flexibility**:
- **Add new analytics** without changing storage schema
- **Experiment with calculations** without affecting production data collection  
- **Different views** can compute different derived metrics from same raw data

**Memory Efficiency**:
- **No redundant storage** of calculated values
- **Compact raw data** in memory atoms
- **On-demand computation** uses temporary memory only during health requests

### Statistics Collection Points

#### API Request Processing
```clojure
;; In execute-unified-method function
(let [start-time (System/currentTimeMillis)
      request-size (count (serialize request))
      result (execute-method ...)]
  ;; Update per-client statistics
  (update-client-api-stats client-id start-time request-size result)
  ;; Update server-wide statistics  
  (update-server-api-stats method-name start-time request-size result))
```

#### Event Delivery Processing
```clojure  
;; In create-and-queue-event function
(let [event-size (count (serialize event))]
  ;; Track successful delivery
  (update-client-event-stats client-id :events-sent event-size)
  ;; Track queue overflow if dropped
  (when dropped? 
    (update-client-event-stats client-id :events-dropped event-size))
  ;; Update server aggregates
  (update-server-event-stats event-type event-size))
```

#### Connection Lifecycle
```clojure
;; On client connection
(update-server-connection-stats :connected (System/currentTimeMillis))

;; On client disconnection  
(let [duration (- disconnect-time connect-time)]
  (update-server-connection-stats :disconnected duration))
```

### Performance Considerations

#### Atomic Updates
Statistics updates use atomic operations to prevent contention:
```clojure
;; Per-client updates
(swap! registry update-in [client-id :metadata] update-statistics-fn)

;; Server-wide updates  
(swap! server-statistics update-server-metrics-fn)
```

#### Zero-Cost Collection Principle

**Capture Available Data**: If data is already computed during normal operation, collect it at zero additional cost.

**API Call Processing Example**:
```clojure
;; We're already doing this work:
(let [request-bytes (.size protobuf-request)
      start-time (System/currentTimeMillis)
      result (execute-api-method ...)
      end-time (System/currentTimeMillis)
      response-bytes (.size protobuf-response)]
  
  ;; Capture data already available at zero cost:
  (update-stats client-id {
    :api-calls-total 1                    ; Already tracking
    :api-calls-success (if success 1 0)   ; Already computed
    :bytes-sent response-bytes            ; Already serialized  
    :bytes-received request-bytes         ; Already deserialized
    :api-call-duration (- end-time start-time)  ; Already timed
    :method-name method-name}))           ; Already resolved
```

**Event Queue Operations Example**:
```clojure
;; We're already checking queue state:
(let [queue-size-before (.size event-queue)
      event-bytes (.size serialized-event)
      offer-result (.offer event-queue event-message)
      queue-size-after (.size event-queue)]
  
  ;; All data is already computed:
  (update-stats client-id {
    :events-sent (if offer-result 1 0)
    :events-dropped (if offer-result 0 1)     ; Offer result known
    :queue-size-current queue-size-after      ; Already queried
    :queue-size-peak (max peak queue-size-after)  ; Simple comparison
    :bytes-queued event-bytes}))              ; Already serialized
```

**Cost Categories**:
- **Zero Cost**: Data already available in operation scope
- **Minimal Cost**: Simple arithmetic on existing data (< 0.1ms)
- **Higher Cost**: Aggregations requiring additional computation

**Design Principle**: Comprehensive collection of zero-cost data, selective collection of higher-cost data based on operational value.

#### Performance Optimization
- Statistics collection adds < 1ms per operation
- Updates performed asynchronously where possible  
- Aggregates calculated on-demand rather than continuously

## Health Monitoring Integration

### Dual Health System Architecture

Ooloi implements two complementary health monitoring systems:

#### 1. gRPC Health Service (Standard Protocol)
- **Protocol**: Standard gRPC health checking (`grpc.health.v1.Health`)
- **Port**: Same as gRPC server (10700 by default)
- **Usage**: gRPC clients call `Check()` method
- **Response**: `HealthCheckResponse.ServingStatus.SERVING/NOT_SERVING`
- **Purpose**: Internal service-to-service health checking

#### 2. HTTP Health Endpoints (External Monitoring)
- **Protocol**: Lightweight HTTP server on dedicated port
- **Port**: Separate from gRPC (10701 by default - `:health-port`)
- **Usage**: Standard HTTP GET requests
- **Response**: JSON formatted metrics and status
- **Purpose**: External monitoring tool integration

#### System Relationship
The HTTP endpoint **builds on** the gRPC health service:
```clojure
;; HTTP endpoint queries same HealthStatusManager as gRPC service
(let [health-service (.getHealthService health-manager)]
  (if health-service "healthy" "unhealthy"))
```

This ensures **consistent health status** across both protocols while providing different access patterns for different use cases.

### HTTP Statistics Endpoints

Extend existing HTTP health server with comprehensive statistics APIs:

```http
GET /health                     # Basic health status (existing)
GET /health/server             # Server-wide aggregate metrics
GET /health/clients            # Per-client metrics summary  
GET /health/clients/{id}       # Specific client detailed metrics
GET /health/performance        # Response times, throughput
GET /health/errors             # Error rates and categorization
GET /health/resources          # Memory, threads, queue usage
```

### Response Format
```json
{
  "server": {
    "uptime-hours": 72.5,
    "clients-connected": 15,
    "clients-peak": 23,
    "api-calls-total": 45230,
    "api-success-rate": 0.998,
    "events-sent": 8934,
    "bytes-transferred-gb": 2.1
  },
  "clients": [
    {
      "client-id": "uuid-123",
      "connected-duration-hours": 2.3,
      "api-calls": 234,
      "events-received": 89,
      "queue-health": "healthy"
    }
  ]
}
```

### External Monitoring Tool Integration

#### Standard Production Monitoring Stack

**Prometheus + Grafana + AlertManager**:
```yaml
# prometheus.yml configuration
- job_name: 'ooloi-server'
  static_configs:
    - targets: ['ooloi-server:10701']
  metrics_path: '/health/server'
  scrape_interval: 15s
```

**Integration Benefits**:
- HTTP JSON format without custom exporters
- Standard Prometheus service discovery compatibility
- Grafana templates for gRPC services
- AlertManager rules for response time/error rate thresholds

**Integration Complexity**: Minimal - Standard HTTP endpoints work without custom code.

#### Enterprise Monitoring Solutions

**DataDog**:
```yaml
# datadog.yaml integration
instances:
  - url: http://ooloi-server:10701/health/server
    tags:
      - service:ooloi
      - environment:production
```
- APM integration with distributed tracing for gRPC calls
- Custom metrics via direct JSON ingestion
- Setup complexity: Low - Standard HTTP integration

**New Relic**:
- Infrastructure monitoring for server resource utilization
- Statistics API data as structured events  
- Setup complexity: Low - REST API compatible

#### Cloud-Native Monitoring

**Kubernetes + Prometheus Operator**:
```yaml
# ServiceMonitor custom resource  
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ooloi-server
spec:
  selector:
    matchLabels:
      app: ooloi-server
  endpoints:
  - port: health-port
    path: /health/server
```
- Auto-scaling integration via HPA based on custom metrics
- Service mesh integration (Istio/Linkerd) via standard endpoints
- Setup complexity: Minimal - Kubernetes-native configuration

#### Load Balancer Health Checking

**NGINX/HAProxy**:
```nginx
upstream ooloi_servers {
    server ooloi1:10701 max_fails=3 fail_timeout=30s;
    server ooloi2:10701 max_fails=3 fail_timeout=30s;
}

# Health check configuration
location /health {
    proxy_pass http://ooloi_servers/health;
    proxy_connect_timeout 1s;
    proxy_read_timeout 1s;
}
```
- HTTP health checks with built-in load balancer support
- Setup complexity: Standard HTTP health endpoint

#### Observability Platforms

**Jaeger (Distributed Tracing)**:
- gRPC tracing via OpenTracing support
- Custom spans for API call and event delivery tracing
- Setup complexity: Medium - Requires tracing instrumentation

**ELK Stack (Elasticsearch + Logstash + Kibana)**:
- Structured logging with JSON metrics as log entries
- Query-based operational analysis
- Setup complexity: Low - Standard JSON log ingestion

#### Comparative Completeness Assessment

**Ooloi's Statistics vs. Server Metrics**:

#### Coverage Areas
- **Connection Lifecycle**: Detailed per-client connection tracking
- **Request/Response Metrics**: Success rates, timing percentiles, method-level analytics  
- **Resource Utilization**: Queue usage, memory consumption, thread pool metrics
- **Error Categorization**: gRPC status codes, error type classification
- **Network Performance**: Bytes transferred, serialization performance

#### Implementation Features
- **Per-Client Granularity**: Individual client behavior patterns and health
- **Queue-Based Architecture**: Event delivery pipeline visibility
- **Real-Time Streaming**: Live event delivery performance metrics
- **Collaborative Metrics**: Multi-client interaction patterns

#### Operational Support
- **Capacity Planning**: Peak usage patterns, growth trend analysis
- **Performance Baselines**: Historical comparison and regression detection  
- **Health Scoring**: Composite health metrics for automated decision making
- **Debugging Context**: Rich error context for rapid troubleshooting

#### Comparison to Industry Standards:

**Redis**: Comprehensive INFO command metrics (190+ fields), real-time MONITOR streaming, built-in latency detection
- **Ooloi's Focus**: Novel per-client queue architecture with overflow handling, collaborative editing integration

**PostgreSQL**: Industry-leading 25+ system views with granular per-session tracking, cumulative statistics persistence  
- **Ooloi's Focus**: STM-gRPC transaction integration monitoring, queue-based event streaming architecture

**Kafka**: Comprehensive 100+ JMX metrics across broker/topic/partition/client dimensions, detailed latency breakdowns
- **Ooloi's Focus**: Per-client queue overflow strategies, collaborative session analytics, behavioral pattern tracking

**gRPC Services (Modern)**: Per-method success rates via Envoy, streaming message counts, distributed tracing integration
- **Ooloi's Focus**: Streaming connection health, collaborative session coordination, client behavioral analysis

#### Overall Assessment: Comprehensive with Novel Elements

Ooloi's statistics architecture provides thorough operational visibility through architectural innovations not commonly found in traditional server implementations. The distinctive characteristics include:
- Per-client queue management with overflow handling  
- STM-gRPC transaction integration monitoring
- Real-time collaborative editing session analytics
- Sophisticated client behavioral pattern tracking
- Two-level statistics approach balancing granular and aggregate metrics

This represents a comprehensive monitoring approach that addresses the specific operational challenges of collaborative music editing systems while maintaining alignment with modern observability practices.

## Implementation Phases

### Phase 1: Basic Collection Infrastructure
- Extend connection registry metadata structure
- Add server-wide statistics atom to component  
- Implement atomic update patterns for statistics

### Phase 2: Collection Point Integration  
- Integrate API call timing and success/failure tracking
- Add event delivery metrics collection
- Implement connection lifecycle tracking

### Phase 3: Health Endpoint Exposure
- Create HTTP endpoints for statistics access
- Implement JSON serialization for metrics
- Add monitoring system integration points

### Phase 4: Advanced Analytics
- Implement trend analysis and performance baselines  
- Add predictive capacity planning metrics
- Create automated health scoring algorithms

## Consequences

### Positive Outcomes

**Operational Visibility**:
- Complete insight into system performance and client behavior
- Proactive identification of performance bottlenecks  
- Data-driven capacity planning and resource allocation
- Comprehensive error tracking and debugging context

**Production Reliability**:
- Early detection of system degradation
- Client health monitoring prevents service disruption
- Performance trending enables proactive scaling  
- Error categorization improves troubleshooting efficiency

**Development Benefits**:
- Performance regression testing with concrete metrics
- Client usage pattern analysis informs feature development  
- Load testing validation with detailed performance data
- Architecture optimization guided by real operational data

### Trade-offs and Considerations

**Memory Overhead**:
- Per-client statistics consume additional memory per connection
- Aggregate metrics require ongoing memory allocation  
- Statistics persistence across server restarts requires additional storage

**Performance Impact**:
- Statistics updates add minimal latency to request processing
- Atomic operations on shared statistics may introduce contention
- JSON serialization for health endpoints requires CPU resources

**Implementation Complexity**:
- Two-level statistics tracking adds code complexity
- Concurrent update handling requires careful synchronization
- Health endpoint security and access control needs consideration

### Mitigation Strategies

**Memory Management**:
- Configurable retention periods for historical data
- Optional statistics persistence to external systems

**Performance Optimization**:
- Asynchronous statistics updates where possible
- Batched updates to reduce atomic operation frequency  
- Lazy calculation of derived metrics

**Operational Safety**:
- Statistics collection failure never impacts core functionality
- Graceful degradation when statistics systems are unavailable
- Health endpoint rate limiting and authentication

## Related ADRs

- [ADR-0002: gRPC](0002-gRPC.md) - Base communication architecture
- [ADR-0024: gRPC Concurrency and Flow Control](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) - Event streaming and queue management  
- [ADR-0022: Lazy Frontend-Backend Architecture](0022-Lazy-Frontend-Backend-Architecture.md) - Event-driven collaboration patterns
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrent transaction processing

## Future Considerations

**Analytics and Machine Learning**:
- Client behavior pattern recognition
- Predictive scaling based on usage patterns
- Anomaly detection in system performance  
- Automated performance optimization recommendations

**Distributed Statistics**:
- Statistics aggregation across multiple server instances
- Cross-server performance correlation analysis
- Distributed health monitoring and alerting

**Client-Side Integration**:
- Frontend performance metrics collection  
- End-to-end latency measurement
- User experience quality metrics
