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
    - [Batched Atomic Updates](#batched-atomic-updates)
    - [Zero-Cost Collection Principle](#zero-cost-collection-principle)
- [Statistics Recording Architecture](#statistics-recording-architecture)
  - [Record-Statistics Abstraction](#record-statistics-abstraction)
  - [Nested Atom Architecture](#nested-atom-architecture)
    - [Two-Level Atom Structure](#two-level-atom-structure)
  - [Server Statistics Optimization (Queue-Based)](#server-statistics-optimization-queue-based)
- [Implementation Summary](#implementation-summary)
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
             ;; CONNECTION LIFECYCLE (Zero Cost)
             ;; ==========================================
             :last-activity-time timestamp           ; Most recent API call or event
             
             ;; ==========================================
             ;; API CALL METRICS (Zero Cost)
             ;; ==========================================
             :api-calls-total 0                      ; Total API calls made
             :api-calls-success 0                    ; Successful API calls
             :api-calls-failure 0                    ; Failed API calls  
             :last-api-call timestamp                ; Most recent API call time
             :last-api-method string                 ; Most recent method called
             :api-slowest-call-ms 0                  ; Worst response time
             :api-fastest-call-ms Long/MAX_VALUE     ; Best response time
             :api-concurrent-calls-current 0         ; Currently processing
             :api-concurrent-calls-peak 0            ; Max simultaneous calls
             
             ;; ==========================================
             ;; EVENT STREAMING METRICS (Zero Cost)
             ;; ==========================================
             :events-sent 0                          ; Total events delivered to client
             :events-dropped 0                       ; Events lost due to queue overflow
             :server-events-received 0               ; Server-wide events received
             :piece-events-received 0                ; Piece-specific events received
             :connect-events-received 0              ; Client connect notifications  
             :disconnect-events-received 0           ; Client disconnect notifications
             :last-event-time timestamp              ; Most recent event delivery
             :last-event-type keyword                ; Type of most recent event
             
             ;; ==========================================
             ;; QUEUE HEALTH METRICS (Zero Cost)
             ;; ==========================================
             :queue-size-current 0                   ; Current queue depth
             :queue-size-peak 0                      ; Maximum queue depth reached
             :queue-overflow-count 0                 ; Number of overflow incidents  
             :queue-overflow-total-events-dropped 0  ; Events lost across all overflows
             :queue-offer-attempts 0                 ; Total queue insertions attempted
             :queue-offer-successes 0                ; Successful queue insertions
             :queue-consumer-lag-ms 0                ; Delay between queue and delivery
             
             ;; ==========================================
             ;; NETWORK PERFORMANCE (Zero Cost)
             ;; ==========================================
             :bytes-sent 0                           ; Total protobuf bytes to client
             :bytes-received 0                       ; Total protobuf bytes from client  
             :bytes-events 0                         ; Bytes consumed by event messages
             :bytes-api-requests 0                   ; Bytes from API calls
             :bytes-api-responses 0                  ; Bytes in API responses
             :largest-request-bytes 0                ; Biggest API request  
             :largest-response-bytes 0               ; Biggest API response
             :largest-event-bytes 0                  ; Biggest event message
             
             ;; ==========================================
             ;; ERROR TRACKING (Zero Cost)
             ;; ==========================================
             :network-errors 0                       ; Network-level failures
             :serialization-errors 0                 ; Protobuf conversion failures  
             :conversion-errors 0                    ; Clojure<->Protobuf failures
             :timeout-errors 0                       ; Request timeout failures
             :last-error-time timestamp              ; Most recent error
             :last-error-message string              ; Most recent error description
             
             ;; ==========================================
             ;; CLIENT BEHAVIOR PATTERNS (Zero Cost)
             ;; ==========================================
             :subscription-add-count 0               ; Total pieces subscribed to
             :subscription-remove-count 0            ; Total pieces unsubscribed from  
             :peak-subscription-count 0              ; Maximum concurrent subscriptions  
             
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

**Implementation Pattern**: All statistics collection follows the same vector-based pattern across all integration points. Statistics helper functions extract raw operational data, compute ALL relevant statistics for their domain, and return operation vectors for atomic application.

**🔑 Key Architectural Benefit**: These integration point functions remain **completely stable** when adding or removing statistics. All statistics complexity is abstracted into helper functions, so integration points never need to change when the statistics requirements evolve.

#### API Request Processing
```clojure
;; Clean integration point - no manual statistics delta construction
;; ALL statistics complexity abstracted into helper function

(defn execute-unified-method [method-name protobuf-request client-id]
  (let [start-time (System/currentTimeMillis)]
    (try
      ;; Normal business logic
      (let [result (execute-api-method method-name protobuf-request)
            end-time (System/currentTimeMillis)
            
            ;; Single clean call to statistics helper
            {:keys [server-ops client-ops]}
            (compute-api-statistics-operations
             {:start-time start-time
              :end-time end-time
              :result result
              :request protobuf-request
              :method-name method-name
              :client-id client-id})]
              
        ;; Clean abstraction - no manual delta construction
        (record-statistics server-ops client-ops server-component client-id)
        result)
        
      ;; Error path - same clean pattern with augmentation
      (catch Exception e
        (let [end-time (System/currentTimeMillis)
              
              ;; API failure statistics
              api-stats (compute-api-statistics-operations
                         {:start-time start-time :end-time end-time
                          :request protobuf-request :method-name method-name
                          :client-id client-id :exception e})
                          
              ;; Augment with error statistics
              final-stats (compute-error-statistics-operations
                           {:error-type (categorize-exception e)
                            :exception e :client-id client-id
                            :server-ops (:server-ops api-stats)
                            :client-ops (:client-ops api-stats)})]
                            
          (record-statistics (:server-ops final-stats) (:client-ops final-stats) server-component client-id)
          (throw e))))))
```

#### Event Delivery Processing
```clojure  
;; Clean integration point - no manual statistics delta construction
;; ALL event statistics complexity abstracted into helper function

(defn create-and-queue-event [client-id event-message]
  (let [;; Normal event delivery logic
        event-bytes (.size (serialize-event event-message))
        queue (get-client-queue client-id)
        queue-size-before (.size queue)
        delivery-start (System/currentTimeMillis)
        offer-success (.offer queue event-message)
        delivery-end (System/currentTimeMillis)
        queue-size-after (.size queue)
        
        ;; Single clean call to statistics helper
        {:keys [server-ops client-ops]}
        (compute-event-statistics-operations
         {:event-message event-message
          :event-bytes event-bytes
          :queue-size-before queue-size-before
          :queue-size-after queue-size-after
          :offer-success offer-success
          :delivery-lag-ms (- delivery-end delivery-start)
          :client-id client-id})]
          
    ;; Clean abstraction - no manual delta construction
    (record-statistics server-ops client-ops server-component client-id)
    offer-success))
```

#### Connection Lifecycle
```clojure
;; Clean integration point - no manual statistics delta construction
;; ALL connection statistics complexity abstracted into helper function

;; On client connection
(defn handle-client-connect [stream-observer]
  (let [connect-time (System/currentTimeMillis)
        client-id (UUID/randomUUID)
        current-count (inc (count @client-registry))
        
        ;; Single clean call to statistics helper
        {:keys [server-ops client-ops]}
        (compute-connection-statistics-operations
         {:event-type :connect
          :connect-time connect-time
          :client-id client-id
          :current-client-count current-count})]
          
    ;; Register client in system
    (register-client client-id stream-observer)
    
    ;; Clean abstraction - no manual delta construction
    (record-statistics server-ops client-ops server-component client-id)
    client-id))

;; On client disconnection  
(defn handle-client-disconnect [client-id disconnect-reason]
  (let [disconnect-time (System/currentTimeMillis)
        connect-time (get-client-connect-time client-id)
        current-count (dec (count @client-registry))
        
        ;; Single clean call to statistics helper
        {:keys [server-ops client-ops]}
        (compute-connection-statistics-operations
         {:event-type :disconnect
          :client-id client-id
          :connect-time connect-time
          :disconnect-time disconnect-time
          :disconnect-reason disconnect-reason
          :current-client-count current-count})]
          
    ;; Remove client from system
    (unregister-client client-id)
    
    ;; Clean abstraction - no manual delta construction
    (record-statistics server-ops client-ops server-component client-id)))
```

### Performance Considerations

#### Batched Atomic Updates

**Critical Performance Pattern**: Batch all statistics changes into a single `swap!` operation per atom to minimize contention and atomic operation overhead.

**Vector-Based Operations Architecture**:

Statistics collection uses **vector-based operations** for clean composition and transparent augmentation. Instead of complex nested map merging, operations are accumulated as simple vectors and applied sequentially.

```clojure
;; Statistics operations as vectors - transparent and composable
[[:inc :api-calls-total 1]
 [:inc :bytes-sent 1024]  
 [:max :api-slowest-call-ms 150]
 [:set :last-api-call timestamp]]

;; Augmentation via simple vector concatenation
(into existing-ops new-ops)  ; Clean composition, no complex merging
```

**Benefits of Vector-Based Operations**:
- **Transparent**: See exactly what operations will be applied
- **Composable**: Simple vector concatenation for multi-domain statistics  
- **Simple**: No complex nested map merging logic
- **Debuggable**: Operations are visible data structures

**Enhanced apply-deltas Function**:
```clojure
(defn apply-deltas
  "Apply sequence of delta operations to statistics atom.
   
   Operations format: [[:operation :field value] ...]
   - :inc - Increment field by value
   - :set - Set field to absolute value  
   - :max - Update field to maximum of current and new value
   - :min - Update field to minimum of current and new value
   
   All operations applied sequentially within single atomic swap! for consistency."
  [stats-atom operations]
  (swap! stats-atom
    (fn [current-stats]
      (reduce
        (fn [stats [operation field value]]
          (case operation
            :inc (assoc stats field (+ (get stats field 0) value))
            :set (assoc stats field value)
            :max (assoc stats field (max (get stats field 0) value))
            :min (assoc stats field (min (get stats field Long/MAX_VALUE) value))))
        current-stats
        operations))))
```

**Vector-Based Statistics Collection Pattern**:

```clojure
(defn compute-api-statistics-operations
  "Compute ALL API-related statistics operations from raw operational data.
   Returns vector of operations for transparent, composable statistics collection."
  [{:keys [start-time end-time result request method-name client-id exception]
    :or {server-ops [] client-ops []}}]  ; Accept existing operations for augmentation
    
  (let [duration-ms (- end-time start-time)
        success? (and result (not exception))
        request-bytes (.size request)
        response-bytes (if success? (.size (:response result)) 0)
        
        ;; Server operations - ALL 25+ API statistics as transparent operations
        new-server-ops [[:inc :api-calls-total 1]
                        [:inc :api-calls-success (if success? 1 0)]
                        [:inc :api-calls-failure (if success? 0 1)]
                        [:inc :bytes-api-requests-total request-bytes]
                        [:inc :bytes-api-responses-total response-bytes]
                        [:inc :bytes-transferred-total (+ request-bytes response-bytes)]
                        [:max :api-slowest-call-ms duration-ms]
                        [:min :api-fastest-call-ms duration-ms]
                        [:max :largest-api-request-bytes request-bytes]
                        [:max :largest-api-response-bytes response-bytes]]
                        ;; ... PLUS all other API server statistics from table
                        
        ;; Client operations - ALL 15+ API client statistics  
        new-client-ops [[:inc :api-calls-total 1]
                        [:inc :api-calls-success (if success? 1 0)]
                        [:inc :api-calls-failure (if success? 0 1)]
                        [:inc :bytes-sent response-bytes]
                        [:inc :bytes-received request-bytes]
                        [:max :api-slowest-call-ms duration-ms]
                        [:min :api-fastest-call-ms duration-ms]
                        [:set :last-api-call end-time]
                        [:set :last-api-method method-name]
                        [:set :last-activity-time end-time]]]
                        ;; ... PLUS all other API client statistics from table
                        
    {:server-ops (into server-ops new-server-ops)  ; Simple vector concatenation
     :client-ops (into client-ops new-client-ops)}))

;; Integration point usage (see Statistics Collection Points section above):
;; (let [{:keys [server-ops client-ops]} (compute-api-statistics-operations {...})]
;;   (record-statistics server-ops client-ops server-component client-id))
```

**Vector-Based Event Statistics Pattern**:

```clojure
(defn compute-event-statistics-operations
  "Compute ALL event-related statistics operations from raw delivery data.
   Returns transparent operation vectors for composable statistics collection."
  [{:keys [event-message event-bytes queue-size-before queue-size-after 
           offer-success delivery-lag-ms client-id]
    :or {server-ops [] client-ops []}}]  ; Accept existing operations for augmentation
    
  (let [event-type (:type event-message)
        delivery-time (System/currentTimeMillis)
        
        ;; Server operations - ALL 20+ event statistics as transparent operations  
        new-server-ops [[:inc :events-sent-total (if offer-success 1 0)]
                        [:inc :events-dropped-total (if offer-success 0 1)]
                        [:inc :event-delivery-attempts 1]
                        [:inc :event-delivery-successes (if offer-success 1 0)]
                        [:inc :bytes-events-total event-bytes]
                        ;; Event type breakdowns
                        [:inc (case event-type
                                :server-event :server-events-sent
                                :piece-event :piece-events-sent
                                :client-connected :connect-events-sent
                                :client-disconnected :disconnect-events-sent) 1]
                        [:max :largest-event-message-bytes event-bytes]]
                        ;; ... PLUS all other event server statistics from table
                        
        ;; Client operations - ALL 15+ event client statistics
        new-client-ops [[:inc :events-sent (if offer-success 1 0)]
                        [:inc :events-dropped (if offer-success 0 1)]
                        [:inc :queue-offer-attempts 1]
                        [:inc :queue-offer-successes (if offer-success 1 0)]
                        [:inc :bytes-events event-bytes]
                        ;; Event type tracking
                        [:inc (case event-type
                                :server-event :server-events-received
                                :piece-event :piece-events-received  
                                :client-connected :connect-events-received
                                :client-disconnected :disconnect-events-received) 1]
                        [:max :queue-size-peak queue-size-after]
                        [:max :queue-consumer-lag-ms delivery-lag-ms]
                        [:max :largest-event-bytes event-bytes]
                        [:set :queue-size-current queue-size-after]
                        [:set :last-event-time delivery-time]
                        [:set :last-event-type event-type]
                        [:set :last-activity-time delivery-time]]]
                        ;; ... PLUS all other event client statistics from table
                        
    {:server-ops (into server-ops new-server-ops)  ; Simple vector concatenation
     :client-ops (into client-ops new-client-ops)}))

;; Integration point usage (see Statistics Collection Points section above):
;; (let [{:keys [server-ops client-ops]} (compute-event-statistics-operations {...})]
;;   (record-statistics server-ops client-ops server-component client-id))
```

## Statistics Recording Architecture

### Record-Statistics Abstraction

The `record-statistics` function provides clean integration point abstraction that handles the complexity of nested atom architecture:

```clojure
(defn record-statistics
  "Records statistics operations for both server and client metrics.
   
   Handles nested atom architecture:
   - Server statistics: Single shared atom  
   - Client statistics: Individual atoms within connection registry
   
   Parameters:
   - server-ops: Vector of server statistics operations  
   - client-ops: Vector of client statistics operations
   - server-component: Component containing atoms and infrastructure
   - client-id: String identifier for client"
  [server-ops client-ops server-component client-id]
  (let [{:keys [server-statistics connection-registry]} server-component]
    ;; Server statistics: Direct atomic update
    (apply-deltas server-statistics server-ops)
    
    ;; Client statistics: Individual atom within registry
    (when-let [client-atom (get-in @connection-registry [client-id :client-statistics])]
      (apply-deltas client-atom client-ops))))

(defn flush-server-statistics!
  "Forces immediate processing of queued server statistics operations.
   Used for testing and shutdown to ensure no data loss."
  []
  ;; No-op in direct implementation, queue processing in optimized version
  nil)
```

### Nested Atom Architecture

#### Two-Level Atom Structure

**Level 1: Component-Level Atoms**
```clojure
;; Component initialization creates top-level atoms
server-statistics (atom {...})           ; Server-wide metrics (single shared atom)
connection-registry (atom {})            ; Client topology (single shared atom for connect/disconnect)
```

**Level 2: Per-Client Atoms Within Registry**
```clojure
;; Registry structure contains individual client atoms
connection-registry (atom {              
  client-1 {:observer response-observer-1
            :metadata {:connected-at timestamp}
            :piece-subscriptions #{}
            :event-queue queue-1
            :client-statistics (atom {...})}  ; ← Individual atom for client-1 statistics
  
  client-2 {:observer response-observer-2
            :metadata {:connected-at timestamp}
            :piece-subscriptions #{}
            :event-queue queue-2  
            :client-statistics (atom {...})}  ; ← Individual atom for client-2 statistics
  })
```

**Why This Architecture**:
- **Connection registry atom**: Needed for connect/disconnect operations (topology changes)
- **Individual client statistics atoms**: Eliminates inter-client contention for high-frequency statistics updates
- **Server statistics atom**: Single shared atom for server-wide metrics (optimized with async queue under load)

**Note**: The `:client-statistics` field shown in the Per-Client Statistics Structure section (line 247) becomes an **atom containing that structure** in the actual implementation, not a direct map.

#### Contention Characteristics
- **Server statistics**: Single atom, high contention under load (solved with async queue)
- **Client statistics**: Individual atoms, zero inter-client contention  
- **Registry operations**: Low contention (connect/disconnect operations only)

### Server Statistics Optimization (Queue-Based)

**Problem**: Under high load (1000 clients × multiple ops/sec), server statistics atom becomes bottleneck with massive contention.

**Solution**: Async queue with batching to eliminate contention:

```clojure
;; Optimized record-statistics with async server queue
(defn record-statistics [server-ops client-ops server-component client-id]
  (let [{:keys [connection-registry server-stats-queue]} server-component]
    ;; Server stats: Async queue (no contention)
    (async/>!! server-stats-queue server-ops)
    
    ;; Client stats: Individual atoms (no inter-client contention)
    (when-let [client-atom (get-in @connection-registry [client-id :client-statistics])]
      (apply-deltas client-atom client-ops))))

;; Background batch consumer (100ms batching)
(async/go-loop []
  (let [timeout-ch (async/timeout 100)
        batch (loop [ops []]
                (let [[op ch] (async/alts! [server-stats-queue timeout-ch])]
                  (cond
                    (= ch timeout-ch) ops                    ; Timeout: process batch
                    (nil? op) ops                            ; Channel closed
                    :else (recur (into ops op)))))]          ; Accumulate
    (when (seq batch)
      (apply-deltas server-statistics batch)))
  (recur)))

(defn flush-server-statistics! []
  "Forces immediate processing of all queued operations."
  (loop []
    (when-let [ops (async/poll! server-stats-queue)]
      (apply-deltas server-statistics ops)
      (recur))))
```

**Benefits of Queue Optimization**:
- **Eliminates Server Contention**: Queue operations instantly without blocking
- **Batched Efficiency**: 100ms batches reduce atom updates by ~100x
- **Preserves Client Isolation**: Individual client atoms remain uncontended
- **Test-Friendly**: `flush-server-statistics!` for deterministic testing

**Benefits of Batched Updates**:
- **Reduced Contention**: One `swap!` per atom instead of multiple sequential updates
- **Atomic Consistency**: All related statistics update together or not at all  
- **Performance**: Eliminates retry overhead from multiple competing atomic operations
- **Cleaner Code**: Statistics collection logic separated from business logic

#### Zero-Cost Collection Principle

**Capture Available Data**: If data is already computed during normal operation, collect it at zero additional cost using vector-based operations.

**Zero-Cost Data Collection Examples**:
```clojure
;; Zero-cost principle: Capture data already computed during normal operations
(let [start-time (System/currentTimeMillis)]
  ;; Normal business logic
  (let [result (execute-api-method ...)
        end-time (System/currentTimeMillis)]
        
    ;; All this data is ALREADY available at zero cost:
    ;; - duration-ms: (- end-time start-time)     ; Already timed
    ;; - success?: (boolean result)               ; Already computed  
    ;; - request-bytes: (.size request)           ; Already deserialized
    ;; - response-bytes: (.size response)         ; Already serialized
    ;; - method-name: method-name                 ; Already resolved
    
    ;; Statistics helper uses zero-cost data (see full function above)
    (compute-api-statistics-operations
     {:start-time start-time :end-time end-time
      :result result :request request :method-name method-name})))

;; Event delivery zero-cost data:
(let [queue-size-before (.size queue)
      offer-result (.offer queue event)]
  ;; All this data is ALREADY available at zero cost:
  ;; - offer-result: offer-result               ; Offer outcome known
  ;; - queue-size-after: (.size queue)         ; Already queried  
  ;; - event-bytes: (.size event)              ; Already serialized
  ;; - event-type: (:type event)               ; Already extracted
  
  ;; Statistics helper uses zero-cost data (see full function above)
  (compute-event-statistics-operations
   {:offer-success offer-result :event-bytes event-bytes
    :queue-size-before queue-size-before :queue-size-after queue-size-after}))
```

**Cost Categories**:
- **Zero Cost**: Data already available in operation scope
- **Minimal Cost**: Simple arithmetic on existing data (< 0.1ms)
- **Higher Cost**: Aggregations requiring additional computation

**Design Principle**: Comprehensive collection of zero-cost data, selective collection of higher-cost data based on operational value.

### Implementation Summary

**For Implementers**: All statistics collection follows this exact pattern across ALL integration points:

1. **Create Statistics Helper Function** (template):
   ```clojure
   (defn compute-DOMAIN-statistics-operations
     [{:keys [operational-data...] 
       :or {server-ops [] client-ops []}}]  ; Accept existing for augmentation
     ;; ... extract zero-cost data from operational parameters ...
     ;; ... compute ALL statistics operations for this domain ...
     {:server-ops (into server-ops [...new-operations...])
      :client-ops (into client-ops [...new-operations...])})
   ```

2. **Call at Integration Point**:
   ```clojure
   ;; Business logic runs first, collecting operational data:
   (let [start-time (System/currentTimeMillis)  ; Before operation
         result (execute-business-logic ...)    ; Normal business logic
         end-time (System/currentTimeMillis)    ; After operation
         
         ;; Statistics helper computes ALL operations, returns map:
         {:keys [server-ops client-ops]}       ; Destructure the map
         (compute-DOMAIN-statistics-operations
          {:start-time start-time :end-time end-time :result result
           :request request :client-id client-id})]  ; Pass raw data
           
     ;; Clean abstraction for statistics recording:
     (record-statistics server-ops client-ops server-component client-id))
   ```

3. **Multi-Domain Augmentation** (chaining for error paths):
   ```clojure
   ;; Chain multiple statistics domains together via augmentation:
   (let [;; First domain: API statistics (success or failure)
         {:keys [server-ops client-ops]}  ; Destructure consistently
         (compute-api-statistics-operations 
          {:start-time start-time :end-time end-time :exception e ...})
                    
         ;; Second domain: Error statistics (augments API stats)  
         {:keys [server-ops client-ops]}  ; Final destructure
         (compute-error-statistics-operations
          {:error-type :timeout :client-id client-id
           :server-ops server-ops        ; Pass existing ops from first call
           :client-ops client-ops        ; Pass existing ops from first call
           })]  ; Helper merges and returns augmented vectors
                       
     ;; Still only ONE recording call per request:
     (record-statistics server-ops client-ops server-component client-id))  
   ```

**Key Requirements**:
- ALL statistics helpers return `{:server-ops [...] :client-ops [...]}`
- Operations format: `[[:operation :field value] ...]`  
- Exactly ONE `record-statistics` call per request
- Collect ALL relevant statistics for each domain (API, events, connections)
- Support augmentation via `:or {server-ops [] client-ops []}` parameter

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

#### JSON Response Format (Default)

**Content-Type**: `application/json; charset=utf-8`  
**Cache-Control**: `no-store`

```json
{
  "server": {
    "server_uptime_ms": 259200000,
    "clients_connected_current": 15,
    "clients_connected_peak": 23,
    "api_calls_total": 45230,
    "api_calls_success": 45112,
    "api_calls_failure": 118,
    "api_success_rate": 0.998,
    "events_sent_total": 8934,
    "events_dropped_total": 12,
    "bytes_transferred_total": 2147483648,
    "event_queues_healthy_count": 13,
    "event_queues_warning_count": 2,
    "event_queues_critical_count": 0
  },
  "clients": [
    {
      "client_id": "uuid-123",
      "connected_duration_ms": 8280000,
      "api_calls_total": 234,
      "events_received": 89,
      "queue_health_score": 0.95
    }
  ]
}
```

#### Prometheus Text Format

**Content-Type**: `text/plain; version=0.0.4`  
**Cache-Control**: `no-store`

```
# HELP ooloi_server_uptime_seconds Server uptime in seconds
# TYPE ooloi_server_uptime_seconds counter
ooloi_server_uptime_seconds 259200

# HELP ooloi_clients_connected_current Currently connected clients
# TYPE ooloi_clients_connected_current gauge  
ooloi_clients_connected_current 15

# HELP ooloi_clients_connected_peak Peak concurrent client connections
# TYPE ooloi_clients_connected_peak gauge
ooloi_clients_connected_peak 23

# HELP ooloi_api_calls_total Total API calls processed
# TYPE ooloi_api_calls_total counter
ooloi_api_calls_total 45230

# HELP ooloi_api_calls_success_total Successful API calls
# TYPE ooloi_api_calls_success_total counter  
ooloi_api_calls_success_total 45112

# HELP ooloi_api_calls_failure_total Failed API calls
# TYPE ooloi_api_calls_failure_total counter
ooloi_api_calls_failure_total 118

# HELP ooloi_events_sent_total Events sent to clients
# TYPE ooloi_events_sent_total counter
ooloi_events_sent_total 8934

# HELP ooloi_events_dropped_total Events dropped due to queue overflow  
# TYPE ooloi_events_dropped_total counter
ooloi_events_dropped_total 12

# HELP ooloi_bytes_transferred_total Network bytes transferred
# TYPE ooloi_bytes_transferred_total counter
ooloi_bytes_transferred_total 2147483648

# HELP ooloi_event_queues_healthy Event queues in healthy state
# TYPE ooloi_event_queues_healthy gauge
ooloi_event_queues_healthy 13

# HELP ooloi_event_queues_warning Event queues in warning state
# TYPE ooloi_event_queues_warning gauge  
ooloi_event_queues_warning 2

# HELP ooloi_event_queues_critical Event queues in critical state
# TYPE ooloi_event_queues_critical gauge
ooloi_event_queues_critical 0
```

**Prometheus Metadata Completeness**: All metrics in Prometheus text format include `# HELP` and `# TYPE` lines following Prometheus best practices for metric discoverability and tool compatibility.

#### Enterprise Operational Requirements

**Cardinality Management**:
- **Bounded Labels Only**: No unbounded `client_id` labels in Prometheus format by default
- **Aggregate-First Strategy**: Per-client details available in JSON format only  
- **Series Cap**: Maximum 150 active time series under normal load
- **TTL Policy**: Client-specific metrics expire after disconnection
- **Per-Client Metrics Opt-In**: If per-client metrics are ever exposed in Prometheus view, they will be behind an explicit opt-in flag (`--enable-prom-client-metrics`) with strict cardinality limits to prevent TSDB explosion

**Performance Constraints**:
- **Serialization Budget**: < 1ms typical response time for all formats
- **Memory Footprint**: Stateless serialization, no cached representations
- **Rate Limiting**: Per-IP token bucket with Prometheus scrape interval awareness

**Cross-Representation Consistency**: JSON and Prometheus views are always generated from the same raw statistics atom snapshot, ensuring identical data across formats at request time.

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
