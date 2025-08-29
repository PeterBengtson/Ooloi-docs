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
  - [Statistics Collection Implementation](#statistics-collection-implementation)
    - [Direct Increment Architecture](#direct-increment-architecture)
    - [Performance Characteristics](#performance-characteristics)
    - [Zero-Cost Collection Principle](#zero-cost-collection-principle)
- [Health Monitoring Integration](#health-monitoring-integration)
  - [Dual Health System Architecture](#dual-health-system-architecture)
  - [HTTP Statistics Endpoints - Content Negotiation Architecture](#http-statistics-endpoints---content-negotiation-architecture)
    - [Content Negotiation Contract](#content-negotiation-contract)
    - [JSON Response Format (Default)](#json-response-format-default)
    - [Prometheus Text Format](#prometheus-text-format)
    - [Enterprise Operational Requirements](#enterprise-operational-requirements)
  - [External Monitoring Tool Integration](#external-monitoring-tool-integration)
    - [Standard Production Monitoring Stack](#standard-production-monitoring-stack)
    - [Enterprise Monitoring Solutions](#enterprise-monitoring-solutions)
    - [Cloud-Native Monitoring](#cloud-native-monitoring)
    - [Load Balancer Health Checking](#load-balancer-health-checking)
    - [Content Negotiation Examples](#content-negotiation-examples)
    - [Observability Platforms](#observability-platforms)
- [Consequences](#consequences)
  - [Positive Outcomes](#positive-outcomes)
  - [Trade-offs and Considerations](#trade-offs-and-considerations)
  - [Mitigation Strategies](#mitigation-strategies)
- [Related ADRs](#related-adrs)

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

1. **Server-wide statistics**: Aggregate metrics that survive client connection churn
2. **Per-client statistics**: Operational metrics specific to individual client connections

Statistics will be collected in real-time during operation with minimal performance impact, accessible via both programmatic APIs and health monitoring endpoints.

## Architecture

## Complete Statistics Field Specification

### Server-Wide Statistics Structure

Add new `server-statistics` component field containing a map of thread-safe LongAdder counters:

```clojure
{ ;; ==========================================
 ;; CONNECTION LIFECYCLE COUNTERS
 ;; ==========================================
 :clients-connected-total (LongAdder.)         ; Total connections since server start
 :clients-disconnected-total (LongAdder.)      ; Total disconnections
 :clients-disconnected-graceful (LongAdder.)   ; Clean disconnections
 :clients-disconnected-error (LongAdder.)      ; Error-based disconnections
 :clients-disconnected-timeout (LongAdder.)    ; Timeout-based disconnections
 :connection-duration-total-ms (LongAdder.)    ; Aggregate connection time

 ;; ==========================================
 ;; API CALL COUNTERS
 ;; ==========================================
 :api-calls-total (LongAdder.)                 ; Total API calls processed
 :api-calls-success (LongAdder.)               ; Successful API calls
 :api-calls-failure (LongAdder.)               ; Failed API calls

 ;; ==========================================
 ;; EVENT STREAMING COUNTERS
 ;; ==========================================
 :server-events-sent (LongAdder.)              ; Total server events broadcast
 :piece-events-sent (LongAdder.)               ; Total piece events sent  
 :connect-events-sent (LongAdder.)             ; Client connect notifications
 :disconnect-events-sent (LongAdder.)          ; Client disconnect notifications
 :events-sent-total (LongAdder.)               ; All event types combined
 :events-dropped-total (LongAdder.)            ; Total events dropped (all clients)
 :event-queues-overflow-total (LongAdder.)     ; Total queue overflow incidents
 :event-delivery-attempts (LongAdder.)         ; Total event delivery attempts
 :event-delivery-successes (LongAdder.)        ; Successful event deliveries

 ;; ==========================================
 ;; NETWORK TRAFFIC COUNTERS
 ;; ==========================================  
 :bytes-transferred-total (LongAdder.)         ; Total network traffic (all clients)
 :bytes-api-requests-total (LongAdder.)        ; Bytes from API calls
 :bytes-api-responses-total (LongAdder.)       ; Bytes in API responses  
 :bytes-events-total (LongAdder.)              ; Bytes in event messages

 ;; ==========================================
 ;; ERROR TRACKING COUNTERS
 ;; ==========================================
 :conversion-errors-total (LongAdder.)         ; Protobuf conversion failures
 :serialization-errors-total (LongAdder.)     ; Data serialization failures
 :network-errors-total (LongAdder.)           ; Network-level errors
 :internal-errors-total (LongAdder.)          ; Unexpected server errors  
 :timeout-errors-total (LongAdder.)           ; Request timeout failures
 :client-errors-total (LongAdder.)            ; Client-side error responses
 :server-errors-total (LongAdder.)            ; Server-side error responses

 ;; ==========================================
 ;; COLLABORATIVE EDITING COUNTERS
 ;; ==========================================
 :subscription-adds-total (LongAdder.)        ; Total piece subscriptions across all clients
 :subscription-removes-total (LongAdder.)     ; Total piece unsubscriptions across all clients

 ;; ==========================================  
 ;; SYSTEM COUNTERS
 ;; ==========================================
 :server-restart-count (LongAdder.)           ; Number of restarts
 }
```

### Per-Client Statistics Structure

Extend existing connection registry with separate top-level `:client-statistics` key for operational visibility:

```clojure
{:client-id "client-uuid"
 :observer stream-observer-ref
 :event-queue bounded-queue-ref  
 :piece-subscriptions #{set-of-piece-ids}
 :metadata {
           ;; ==========================================
           ;; CONNECTION METADATA FIELDS (Preserved)
           ;; ==========================================
           :connected-at timestamp                   ; Connection establishment time
           :client-ip "127.0.0.1"                  ; Client IP address  
           :client-port 54321                        ; Client port number
           :connection-id "conn-uuid"               ; Transport connection identifier
           }
 :client-statistics {
   ;; ==========================================
   ;; API CALL COUNTERS
   ;; ==========================================
   :api-calls-total (LongAdder.)             ; Total API calls made
   :api-calls-success (LongAdder.)           ; Successful API calls
   :api-calls-failure (LongAdder.)           ; Failed API calls

   ;; ==========================================
   ;; EVENT STREAMING COUNTERS
   ;; ==========================================
   :events-sent (LongAdder.)                 ; Total events delivered to client
   :events-dropped (LongAdder.)              ; Events lost due to queue overflow
   :server-events-received (LongAdder.)      ; Server-wide events received
   :piece-events-received (LongAdder.)       ; Piece-specific events received
   :connect-events-received (LongAdder.)     ; Client connect notifications  
   :disconnect-events-received (LongAdder.)  ; Client disconnect notifications

   ;; ==========================================
   ;; QUEUE HEALTH COUNTERS
   ;; ==========================================
   :queue-overflow-count (LongAdder.)        ; Number of overflow incidents  
   :queue-overflow-total-events-dropped (LongAdder.) ; Events lost across all overflows
   :queue-offer-attempts (LongAdder.)        ; Total queue insertions attempted
   :queue-offer-successes (LongAdder.)       ; Successful queue insertions

   ;; ==========================================
   ;; NETWORK TRAFFIC COUNTERS
   ;; ==========================================
   :bytes-sent (LongAdder.)                  ; Total protobuf bytes to client
   :bytes-received (LongAdder.)              ; Total protobuf bytes from client  
   :bytes-events (LongAdder.)                ; Bytes consumed by event messages
   :bytes-api-requests (LongAdder.)          ; Bytes from API calls
   :bytes-api-responses (LongAdder.)         ; Bytes in API responses

   ;; ==========================================
   ;; ERROR TRACKING COUNTERS
   ;; ==========================================
   :network-errors (LongAdder.)              ; Network-level failures
   :serialization-errors (LongAdder.)        ; Protobuf conversion failures  
   :conversion-errors (LongAdder.)           ; Clojure<->Protobuf failures
   :timeout-errors (LongAdder.)              ; Request timeout failures

   ;; ==========================================
   ;; SUBSCRIPTION COUNTERS
   ;; ==========================================
   :subscription-add-count (LongAdder.)      ; Total pieces subscribed to
   :subscription-remove-count (LongAdder.)   ; Total pieces unsubscribed from  
}}
```

### Raw vs. Derived Statistics Architecture

#### Design Principle: Store Raw Data, Compute Analytics On-Demand

**The server component holds only raw LongAdder counters** (zero/minimal collection cost):
```clojure
;; Raw LongAdder counters - minimal overhead to collect
:api-calls-total (LongAdder.)     ; Current sum: (.sum this) => 15
:api-calls-success (LongAdder.)   ; Current sum: (.sum this) => 14
:events-dropped (LongAdder.)      ; Current sum: (.sum this) => 2
:bytes-sent (LongAdder.)          ; Current sum: (.sum this) => 4096
```

**Health endpoints compute derived metrics** when requested:
```clojure
;; GET /health/clients/{id} - calculated on-demand from LongAdder sums
(defn compute-client-analytics [client-stats]
  {:api-success-rate (/ (.sum (:api-calls-success client-stats)) 
                       (.sum (:api-calls-total client-stats)))
   :event-delivery-reliability (- 1.0 (/ (.sum (:events-dropped client-stats)) 
                                         (.sum (:events-sent client-stats))))
   :client-health-score (composite-health-calculation client-stats)})
```

#### Benefits of This Architecture:

**Runtime Performance**:
- **Zero calculation overhead** during normal operation
- **No expensive aggregations** in hot paths  
- **Raw data collection** has minimal performance impact

**Data Freshness**:  
- **Always current**: Derived metrics computed from latest raw data
- **No stale calculations**: No cached derived values to become outdated
- **Lock-free reads**: Derived metrics are computed by summing counters; slight skew is acceptable for ops

**Flexibility**:
- **Add new analytics** without changing storage schema
- **Experiment with calculations** without affecting production data collection  
- **Different views** can compute different derived metrics from same raw data

**Memory Efficiency**:
- **No redundant storage** of calculated values
- **Compact raw data** in memory atoms
- **On-demand computation** uses temporary memory only during health requests

### Statistics Collection Points

**Implementation Pattern**: All statistics collection uses direct LongAdder increments at integration points. Statistics helper functions extract raw operational data and directly increment the appropriate LongAdder counters.

**🔑 Key Architectural Benefit**: These integration point functions remain **completely stable** when adding or removing statistics. All statistics complexity is abstracted into helper functions, so integration points never need to change when the statistics requirements evolve.

#### API Request Processing
```clojure
;; Clean integration point - statistics complexity abstracted into helper function

(defn execute-unified-method [method-name protobuf-request client-id]
  (let [start-time (System/currentTimeMillis)]
    (try
      ;; Normal business logic
      (let [result (execute-api-method method-name protobuf-request)
            end-time (System/currentTimeMillis)]
            
        ;; Clean abstraction - direct LongAdder increments
        (increment-api-stats server-component client-id 
                           {:start-time start-time
                            :end-time end-time
                            :result result
                            :request protobuf-request
                            :method-name method-name
                            :success? true})
        result)
        
      ;; Error path - same clean pattern
      (catch Exception e
        (let [end-time (System/currentTimeMillis)]
              
          ;; API failure and error statistics
          (increment-api-stats server-component client-id 
                             {:start-time start-time :end-time end-time
                              :request protobuf-request :method-name method-name
                              :success? false :exception e})
                          
          (increment-error-stats server-component client-id 
                               {:error-type (categorize-exception e)
                                :exception e})
          (throw e))))))
```

#### Event Delivery Processing
```clojure  
;; Clean integration point - statistics complexity abstracted into helper function

(defn create-and-queue-event [client-id event-message]
  (let [;; Normal event delivery logic
        event-bytes (.size (serialize-event event-message))
        queue (get-client-queue client-id)
        queue-size-before (.size queue)
        delivery-start (System/currentTimeMillis)
        offer-success (.offer queue event-message)
        delivery-end (System/currentTimeMillis)
        queue-size-after (.size queue)]
        
    ;; Clean abstraction - direct LongAdder increments
    (increment-event-stats server-component client-id
                         {:event-type (:type event-message)
                          :event-bytes event-bytes
                          :offer-success offer-success})
    offer-success))
```

#### Connection Lifecycle
```clojure
;; Clean integration point - statistics complexity abstracted into helper function

;; On client connection
(defn handle-client-connect [stream-observer]
  (let [connect-time (System/currentTimeMillis)
        client-id (UUID/randomUUID)
        current-count (inc (count @client-registry))]
          
    ;; Register client in system
    (register-client client-id stream-observer)
    
    ;; Clean abstraction - direct LongAdder increments
    (increment-connection-stats server-component client-id
                              {:event-type :connect
                               :connect-time connect-time
                               :current-client-count current-count})
    client-id))

;; On client disconnection  
(defn handle-client-disconnect [client-id disconnect-reason]
  (let [disconnect-time (System/currentTimeMillis)
        connect-time (get-client-connect-time client-id)
        current-count (dec (count @client-registry))]
          
    ;; Remove client from system
    (unregister-client client-id)
    
    ;; Clean abstraction - direct LongAdder increments
    (increment-connection-stats server-component client-id
                              {:event-type :disconnect
                               :connect-time connect-time
                               :disconnect-time disconnect-time
                               :disconnect-reason disconnect-reason
                               :current-client-count current-count}))
```

### Statistics Collection Implementation

#### Direct Increment Architecture

Statistics collection uses direct LongAdder increments at integration points for maximum performance:

**Note**: The increment functions below will be implemented in `backend/src/main/clojure/ooloi/backend/grpc/stats.clj` to separate statistics concerns from gRPC business logic.

```clojure
;; Direct increment helper functions
(defn increment-api-stats [server-component client-id {:keys [success? bytes-sent bytes-received]}]
  (let [{:keys [server-statistics connection-registry]} server-component]
    ;; Server statistics - shared counters
    (.add (:api-calls-total server-statistics) 1)
    (if success?
      (.add (:api-calls-success server-statistics) 1)
      (.add (:api-calls-failure server-statistics) 1))
    (when bytes-sent
      (.add (:bytes-api-responses-total server-statistics) bytes-sent))
    (when bytes-received  
      (.add (:bytes-api-requests-total server-statistics) bytes-received))
    
    ;; Client statistics - individual counters  
    (when-let [client-stats (:client-statistics (get @connection-registry client-id))]
      (.add (:api-calls-total client-stats) 1)
      (if success?
        (.add (:api-calls-success client-stats) 1)
        (.add (:api-calls-failure client-stats) 1))
      (when bytes-sent (.add (:bytes-api-responses client-stats) bytes-sent))
      (when bytes-received (.add (:bytes-api-requests client-stats) bytes-received)))))

(defn increment-event-stats [server-component client-id {:keys [event-type offer-success event-bytes]}]
  (let [{:keys [server-statistics connection-registry]} server-component]
    ;; Server statistics - shared counters
    (.add (:events-sent-total server-statistics) 1)
    (if offer-success
      (.add (:event-delivery-successes server-statistics) 1)
      (.add (:events-dropped-total server-statistics) 1))
    (.add (:event-delivery-attempts server-statistics) 1)
    (.add (:bytes-events-total server-statistics) event-bytes)
    
    ;; Event type specific counters
    (case event-type
      :server-event (.add (:server-events-sent server-statistics) 1)
      :piece-event (.add (:piece-events-sent server-statistics) 1)
      :client-connected (.add (:connect-events-sent server-statistics) 1)
      :client-disconnected (.add (:disconnect-events-sent server-statistics) 1)
      nil)
    
    ;; Client statistics - individual counters
    (when-let [client-stats (:client-statistics (get @connection-registry client-id))]
      ;; Queue offer tracking
      (.add (:queue-offer-attempts client-stats) 1)
      (if offer-success
        (do (.add (:events-sent client-stats) 1)
            (.add (:queue-offer-successes client-stats) 1))
        (.add (:events-dropped client-stats) 1))
      (.add (:bytes-events client-stats) event-bytes)
      
      ;; Event type specific client counters
      (case event-type
        :server-event (.add (:server-events-received client-stats) 1)
        :piece-event (.add (:piece-events-received client-stats) 1)
        :client-connected (.add (:connect-events-received client-stats) 1)
        :client-disconnected (.add (:disconnect-events-received client-stats) 1)
        nil))))

(defn increment-connection-stats [server-component client-id {:keys [event-type disconnect-reason]}]
  (let [{:keys [server-statistics]} server-component]
    (case event-type
      :connect (.add (:clients-connected-total server-statistics) 1)
      :disconnect (do
                    (.add (:clients-disconnected-total server-statistics) 1)
                    (case disconnect-reason
                      :graceful (.add (:clients-disconnected-graceful server-statistics) 1)
                      :error (.add (:clients-disconnected-error server-statistics) 1)
                      :timeout (.add (:clients-disconnected-timeout server-statistics) 1)
                      nil))
      nil)))

(defn increment-error-stats [server-component client-id {:keys [error-type]}]
  (let [{:keys [server-statistics connection-registry]} server-component]
    ;; Server error counters
    (case error-type
      :conversion (.add (:conversion-errors-total server-statistics) 1)
      :serialization (.add (:serialization-errors-total server-statistics) 1)
      :network (.add (:network-errors-total server-statistics) 1)
      :internal (.add (:internal-errors-total server-statistics) 1)
      :timeout (.add (:timeout-errors-total server-statistics) 1)
      :client (.add (:client-errors-total server-statistics) 1)
      :server (.add (:server-errors-total server-statistics) 1)
      nil)
    
    ;; Client error counters
    (when-let [client-stats (:client-statistics (get @connection-registry client-id))]
      (case error-type
        :network (.add (:network-errors client-stats) 1)
        :serialization (.add (:serialization-errors client-stats) 1)
        :conversion (.add (:conversion-errors client-stats) 1)
        :timeout (.add (:timeout-errors client-stats) 1)
        nil))))
```

### Performance Characteristics

**Infinite Scalability**: LongAdder uses per-CPU buckets for zero contention
- 1 thread × 100 ops/sec: ~0.001ms overhead per operation
- 1000 threads × 1000 ops/sec: Still ~0.001ms overhead per operation  
- Linear scaling with no performance degradation

**Memory Efficiency**: ~40 bytes overhead per LongAdder
- 20 server counters × 40 bytes = 800 bytes server statistics  
- 15 client counters × 40 bytes × 1000 clients = 600KB client statistics
- Total memory footprint negligible even at scale

**Architecture Benefits**:
- **Server statistics**: LongAdders eliminate all contention between operations
- **Client statistics**: Individual LongAdders within connection registry, zero inter-client contention
- **No queues needed**: Direct `.add()` calls scale infinitely without blocking

#### Zero-Cost Collection Principle

**Capture Available Data**: If data is already computed during normal operation, collect it at zero additional cost using simple counter increments.

**Simple Counter Collection Examples**:
```clojure
;; Zero-cost principle: Increment counters based on data already computed
(let [result (execute-api-method ...)]
  ;; Simple success/failure tracking - data already known
  (increment-api-stats server-component client-id 
                     {:success? (boolean result)
                      :bytes-sent (.size response)      ; Already serialized
                      :bytes-received (.size request)}) ; Already deserialized
  result)

;; Event delivery - track outcomes already determined
(let [offer-success (.offer queue event)]
  ;; Simple success/drop tracking - outcome already known
  (increment-event-stats server-component client-id
                       {:event-type (:type event)      ; Already extracted
                        :offer-success offer-success   ; Already determined
                        :event-bytes (.size event)})   ; Already serialized
  offer-success)
```

**Collection Overhead**: 
- **LongAdder increment**: ~0.001ms per `.add()` call
- **No complex calculations**: Only simple counter increments
- **No aggregations**: All analytics computed on-demand by health endpoints

**Design Principle**: Collect only simple counters that require zero additional computation beyond what's already available in the operation flow.


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
  (if health-service "SERVING" "NOT_SERVING"))
```

This ensures **consistent health status** across both protocols while providing different access patterns for different use cases.

### HTTP Statistics Endpoints - Content Negotiation Architecture

**Enterprise-grade content negotiation** serving multiple formats from single URLs:

```http
GET /health                     # Basic health status (existing)
GET /health/server             # Server-wide aggregate metrics
GET /health/clients            # Per-client metrics summary  
GET /health/clients/{id}       # Specific client detailed metrics
GET /health/performance        # Response times, throughput
GET /health/errors             # Error rates and categorization
GET /health/resources          # Memory, threads, queue usage
```

#### Content Negotiation Contract

**Field Naming Convention**: All HTTP responses use underscored field names (`clients_connected_current`) for universal compatibility. Internal ADR-0025 statistics maintain hyphenated names (`:clients-connected-current`) with transformation only at serialization boundary.

**Format Selection Rules**:
1. **Accept Header Priority**:
   - `Accept: text/plain` or `Accept: application/openmetrics-text` → Prometheus text format
   - Default → JSON format
2. **Query Parameter Override**: `?format=prom` or `?format=json` overrides Accept header
3. **User-Agent Detection**: `User-Agent: Prometheus/*` → Prometheus text format

**OpenMetrics Evolution**: Ooloi may emit OpenMetrics headers when Prometheus defaults to OpenMetrics format, but the text exposition format remains stable for backward compatibility.

**Versioning and Compatibility**:
- **Schema Version Header**: `X-Ooloi-Metrics-Schema: 1` in all responses
- **Stable URLs**: Endpoints never change, only representations evolve
- **Backward Compatibility**: Add new metrics, never rename existing ones in Prometheus view

**Content Differentiation**: JSON may include **ratios derived from counters** (e.g., success rates); Prometheus exposes **raw counters only** for time-series analysis. If a denominator is 0, JSON ratios are `null`.

#### JSON Response Format (Default)

**Content-Type**: `application/json; charset=utf-8`  
**Cache-Control**: `no-store`

```json
{
  "server": {
    "clients_connected_total": 1847,
    "clients_disconnected_total": 1832,
    "clients_disconnected_graceful": 1829,
    "clients_disconnected_error": 2,
    "clients_disconnected_timeout": 1,
    "connection_duration_total_ms": 47382910000,
    "api_calls_total": 45230,
    "api_calls_success_total": 45112,
    "api_calls_failure_total": 118,
    "api_success_rate": 0.99739,
    "server_events_sent": 2341,
    "piece_events_sent": 5892,
    "connect_events_sent": 1847,
    "disconnect_events_sent": 1832,
    "events_sent_total": 11912,
    "events_dropped_total": 12,
    "event_queues_overflow_total": 3,
    "event_delivery_attempts": 11924,
    "event_delivery_successes": 11912,
    "event_delivery_success_rate": 0.99899,
    "bytes_api_requests_total": 117384920,
    "bytes_api_responses_total": 204010472,
    "bytes_events_total": 12899876,
    "conversion_errors_total": 0,
    "serialization_errors_total": 1,
    "network_errors_total": 2,
    "internal_errors_total": 0,
    "timeout_errors_total": 0,
    "client_errors_total": 9,
    "server_errors_total": 3,
    "subscription_adds_total": 144,
    "subscription_removes_total": 98,
    "server_restart_count": 2
  },
  "clients": [
    {
      "client_id": "uuid-123",
      "api_calls_total": 234,
      "api_calls_success_total": 233,
      "api_calls_failure_total": 1,
      "events_sent": 89,
      "events_dropped": 0,
      "server_events_received": 23,
      "piece_events_received": 66,
      "connect_events_received": 0,
      "disconnect_events_received": 0,
      "queue_overflow_count": 0,
      "queue_overflow_total_events_dropped": 0,
      "queue_offer_attempts": 89,
      "queue_offer_successes": 89,
      "bytes_sent": 51234,
      "bytes_received": 93456,
      "bytes_events": 8123,
      "bytes_api_requests": 93456,
      "bytes_api_responses": 51234,
      "network_errors": 0,
      "serialization_errors": 0,
      "conversion_errors": 0,
      "timeout_errors": 0,
      "subscription_add_count": 3,
      "subscription_remove_count": 1
    }
  ]
}
```

#### Prometheus Text Format

**Content-Type**: `text/plain; version=0.0.4`  
**Cache-Control**: `no-store`

```
# HELP ooloi_clients_connected_total Total client connections ever
# TYPE ooloi_clients_connected_total counter
ooloi_clients_connected_total 1847

# HELP ooloi_clients_disconnected_total Total disconnections ever
# TYPE ooloi_clients_disconnected_total counter
ooloi_clients_disconnected_total 1832

# HELP ooloi_clients_disconnected_graceful Clean disconnections
# TYPE ooloi_clients_disconnected_graceful counter
ooloi_clients_disconnected_graceful 1829

# HELP ooloi_clients_disconnected_error Error-based disconnections
# TYPE ooloi_clients_disconnected_error counter
ooloi_clients_disconnected_error 2

# HELP ooloi_clients_disconnected_timeout Timeout-based disconnections
# TYPE ooloi_clients_disconnected_timeout counter
ooloi_clients_disconnected_timeout 1

# HELP ooloi_connection_duration_total_ms Aggregate connection time
# TYPE ooloi_connection_duration_total_ms counter
ooloi_connection_duration_total_ms 47382910000

# HELP ooloi_api_calls_total Total API calls processed
# TYPE ooloi_api_calls_total counter
ooloi_api_calls_total 45230

# HELP ooloi_api_calls_success_total Successful API calls
# TYPE ooloi_api_calls_success_total counter
ooloi_api_calls_success_total 45112

# HELP ooloi_api_calls_failure_total Failed API calls
# TYPE ooloi_api_calls_failure_total counter
ooloi_api_calls_failure_total 118

# HELP ooloi_server_events_sent_total Total server events broadcast
# TYPE ooloi_server_events_sent_total counter
ooloi_server_events_sent_total 2341

# HELP ooloi_piece_events_sent_total Total piece events sent
# TYPE ooloi_piece_events_sent_total counter
ooloi_piece_events_sent_total 5892

# HELP ooloi_connect_events_sent_total Client connect notifications
# TYPE ooloi_connect_events_sent_total counter
ooloi_connect_events_sent_total 1847

# HELP ooloi_disconnect_events_sent_total Client disconnect notifications
# TYPE ooloi_disconnect_events_sent_total counter
ooloi_disconnect_events_sent_total 1832

# HELP ooloi_events_sent_total All event types combined
# TYPE ooloi_events_sent_total counter
ooloi_events_sent_total 11912

# HELP ooloi_events_dropped_total Total events dropped (all clients)
# TYPE ooloi_events_dropped_total counter
ooloi_events_dropped_total 12

# HELP ooloi_event_queues_overflow_total Total queue overflow incidents
# TYPE ooloi_event_queues_overflow_total counter
ooloi_event_queues_overflow_total 3

# HELP ooloi_event_delivery_attempts Total event delivery attempts
# TYPE ooloi_event_delivery_attempts counter
ooloi_event_delivery_attempts 11924

# HELP ooloi_event_delivery_successes Successful event deliveries
# TYPE ooloi_event_delivery_successes counter
ooloi_event_delivery_successes 11912

# HELP ooloi_bytes_api_requests_total Bytes from API calls
# TYPE ooloi_bytes_api_requests_total counter
ooloi_bytes_api_requests_total 117384920

# HELP ooloi_bytes_api_responses_total Bytes in API responses
# TYPE ooloi_bytes_api_responses_total counter
ooloi_bytes_api_responses_total 204010472

# HELP ooloi_bytes_events_total Bytes in event messages
# TYPE ooloi_bytes_events_total counter
ooloi_bytes_events_total 12899876

# HELP ooloi_conversion_errors_total Protobuf conversion failures
# TYPE ooloi_conversion_errors_total counter
ooloi_conversion_errors_total 0

# HELP ooloi_serialization_errors_total Data serialization failures
# TYPE ooloi_serialization_errors_total counter
ooloi_serialization_errors_total 1

# HELP ooloi_network_errors_total Network-level errors
# TYPE ooloi_network_errors_total counter
ooloi_network_errors_total 2

# HELP ooloi_internal_errors_total Unexpected server errors
# TYPE ooloi_internal_errors_total counter
ooloi_internal_errors_total 0

# HELP ooloi_timeout_errors_total Request timeout failures
# TYPE ooloi_timeout_errors_total counter
ooloi_timeout_errors_total 0

# HELP ooloi_client_errors_total Client-side error responses
# TYPE ooloi_client_errors_total counter
ooloi_client_errors_total 9

# HELP ooloi_server_errors_total Server-side error responses
# TYPE ooloi_server_errors_total counter
ooloi_server_errors_total 3

# HELP ooloi_subscription_adds_total Total piece subscriptions across all clients
# TYPE ooloi_subscription_adds_total counter
ooloi_subscription_adds_total 144

# HELP ooloi_subscription_removes_total Total piece unsubscriptions across all clients
# TYPE ooloi_subscription_removes_total counter
ooloi_subscription_removes_total 98

# HELP ooloi_server_restart_count Number of server restarts
# TYPE ooloi_server_restart_count counter
ooloi_server_restart_count 2
```

**Prometheus Metadata Completeness**: All metrics in Prometheus text format include `# HELP` and `# TYPE` lines following Prometheus best practices for metric discoverability and tool compatibility.

#### Enterprise Operational Requirements

**Cardinality Management**:
- **Bounded Labels Only**: No unbounded `client_id` labels in Prometheus format by default
- **Aggregate-First Strategy**: Per-client details available in JSON format only  
- **Bounded series count**: No per-client labels to prevent TSDB explosion
- **TTL Policy**: Client-specific metrics expire after disconnection
- **Per-Client Metrics Opt-In**: If per-client metrics are ever exposed in Prometheus view, they will be behind an explicit opt-in flag (`--enable-prom-client-metrics`) with strict cardinality limits to prevent TSDB explosion

**Performance Constraints**:
- **Serialization Budget**: < 1ms typical response time for all formats
- **Memory Footprint**: Stateless serialization, no cached representations
- **Rate Limiting**: Per-IP token bucket with Prometheus scrape interval awareness

**Cross-Representation Consistency**: JSON and Prometheus views are always generated from the same point-in-time read of counter sums (non-atomic, lock-free), ensuring near-identical data across formats at request time.

**Security Integration**:
- **Authentication**: Support `Authorization: Bearer` for JSON endpoints
- **Prometheus Access**: Allow unauthenticated scrape via IP allowlist/ingress policy  
- **TLS**: HTTPS required in production; mTLS optional for scrape networks

### External Monitoring Tool Integration

The content negotiation architecture provides universal compatibility with monitoring ecosystems through standard HTTP interfaces.

#### Standard Production Monitoring Stack

**Prometheus + Grafana + AlertManager** - Zero configuration required:

```yaml
# prometheus.yml - works immediately with content negotiation
scrape_configs:
  - job_name: 'ooloi'
    static_configs:
      - targets: ['ooloi-server:10701']
    metrics_path: /health/server
    scrape_interval: 15s
    # Prometheus User-Agent automatically triggers text format response
```

**Grafana Development Integration**:
```bash
# JSON Datasource configuration (manual setup)
# URL: http://localhost:10701
# All /health/* endpoints return JSON by default - no Accept header needed
```

**Integration Benefits**:
- **Zero Custom Code**: Standard HTTP content negotiation, no exporters required
- **Single URL Strategy**: Same endpoints serve both JSON (dev) and Prometheus (prod)  
- **Automatic Format Detection**: Prometheus scraper gets text format, browsers get JSON
- **Enterprise Ready**: Follows HTTP standards for multi-representation resources

#### Enterprise Monitoring Solutions

**DataDog** - JSON ingestion via HTTP checks:
```yaml
# datadog.yaml - automatically gets JSON format
instances:
  - url: http://ooloi-server:10701/health/server
    tags:
      - service:ooloi
      - environment:production
```

**New Relic** - Direct JSON API integration:
```bash
# Custom integration script - JSON format by default
curl -H "Authorization: Bearer $NR_API_KEY" \
     "http://ooloi-server:10701/health/server" | \
     newrelic-cli events post --data @-
```

**Enterprise Benefits**:
- **Universal JSON Support**: All enterprise tools consume JSON by default
- **No Format Translation**: Direct API ingestion without middleware
- **Standard HTTP**: Works with existing enterprise HTTP monitoring infrastructure

#### Cloud-Native Monitoring

**Kubernetes + Prometheus Operator** - Native Prometheus text format:
```yaml
# ServiceMonitor - Prometheus Operator automatically configures scraping
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
    interval: 15s
    # Prometheus scraper User-Agent triggers text format automatically
```

**Service Mesh Integration** - Works with any proxy:
```yaml
# Istio VirtualService example - standard HTTP health checks
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  http:
  - match:
    - uri:
        prefix: /health
    route:
    - destination:
        host: ooloi-server
        port:
          number: 10701
```

#### Load Balancer Health Checking

**NGINX/HAProxy** - Standard HTTP health endpoint:
```nginx
upstream ooloi_servers {
    server ooloi1:10701 max_fails=3 fail_timeout=30s;
    server ooloi2:10701 max_fails=3 fail_timeout=30s;
}

# Health check - gets JSON format by default
location /health {
    proxy_pass http://ooloi_servers/health;
    proxy_connect_timeout 1s;
    proxy_read_timeout 1s;
    # Load balancer doesn't need to parse JSON, just checks HTTP 200
}
```

**Cloud Load Balancers** - Universal HTTP support:
```bash
# AWS ALB health check configuration
# Target: /health (returns JSON with HTTP 200/503)
# All cloud providers support standard HTTP health checks

# GCP Load Balancer
# Health check path: /health/server
# Success criteria: HTTP 200 + JSON response

# Azure Application Gateway  
# Health probe: /health
# Content negotiation transparent to load balancer
```

#### Content Negotiation Examples

**Manual Testing and Discovery**:
```bash
# JSON format (default) - for development and API exploration
curl http://localhost:10701/health/server | jq .

# Prometheus text format - explicit Accept header
curl -H 'Accept: text/plain; version=0.0.4' http://localhost:10701/health/server

# Query parameter override - forces format regardless of headers
curl http://localhost:10701/health/server?format=prom

# Browser access - automatically gets JSON due to Accept: text/html
open http://localhost:10701/health/server
```

**Production Integration Patterns**:
```yaml
# Prometheus scrape config - zero configuration required
scrape_configs:
  - job_name: 'ooloi'
    static_configs: [{ targets: ['host:10701'] }]
    metrics_path: /health/server
    scrape_interval: 15s
    # User-Agent: Prometheus/* automatically triggers text format

# Grafana JSON datasource - points to same URLs
# URL: http://host:10701
# Paths: /health/server, /health/clients, /health/performance
# Format: Automatic JSON (default response)
```

**Format Verification**:
```bash
# Verify Prometheus format is valid
curl -H 'Accept: text/plain' http://localhost:10701/health/server | promtool check metrics

# Verify JSON schema version
curl -I http://localhost:10701/health/server | grep X-Ooloi-Metrics-Schema

# Test query parameter override priority
curl -H 'Accept: application/json' http://localhost:10701/health/server?format=prom
# Should return Prometheus text despite JSON Accept header
```

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
