# 🟠 gRPC Communication and Flow Control: Understanding Ooloi's Communication Architecture

**This guide explains Ooloi's gRPC communication patterns and the rationale behind its flow control decisions. While the examples use Clojure, the principles apply to any gRPC client implementation.**

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Introduction: Two Kinds of Communication](#introduction-two-kinds-of-communication)
- [Unified Client Connection Architecture](#unified-client-connection-architecture)
- [Understanding the Core Problem](#understanding-the-core-problem)
- [API Requests: Why Flow Control Is Counterproductive](#api-requests-why-flow-control-is-counterproductive)
- [Event Streaming: Why Flow Control Is Essential](#event-streaming-why-flow-control-is-essential)
- [The Queue-Based Solution](#the-queue-based-solution)
- [STM and gRPC: A Perfect Partnership](#stm-and-grpc-a-perfect-partnership)
- [Practical Examples](#practical-examples)
- [Performance Implications](#performance-implications)
  - [Event Structure and Conventions](#event-structure-and-conventions)
  - [Event Broadcasting Examples](#event-broadcasting-examples)
  - [Event Streaming Performance](#event-streaming-performance)
- [Related Architecture](#related-architecture)

---

## Prerequisites

- **Basic gRPC knowledge**: Understanding of request/response vs streaming patterns
- **Concurrency fundamentals**: Threads, blocking operations, and performance bottlenecks
- **Ooloi architecture familiarity**: Frontend-backend separation ([ADR-0001](../ADRs/0001-Frontend-Backend-Separation.md)) and STM concepts ([ADR-0004](../ADRs/0004-STM-for-concurrency.md))
- **Musical context**: Understanding collaborative music editing scenarios

---

## Introduction: Two Communication Patterns

Ooloi's collaborative music notation system uses two distinct communication patterns between clients and server:

### 1. **Request/Response API Calls** (Client → Server)
Deliberate changes to musical content:
- "Add a C# to measure 47"
- "Change this section to 4/4 time"
- "Transpose the violin part up an octave"

### 2. **Event Streaming** (Server → Clients)
Real-time notifications about changes:
- "Note added in measure 12"
- "Tempo marking changed to Allegro"
- "Auto-save completed"

These patterns have different concurrency characteristics and flow control requirements.

---

## Explicit App-Controlled Connection Architecture

### Two-Phase Connection Sequence

Ooloi clients use a **two-phase connection architecture** where component initialization prepares the gRPC infrastructure but applications explicitly control when network connections are established:

```
Two-Phase Client Connection Sequence
┌─────────────┐                           ┌─────────────────┐
│   Client    │                           │    Server       │
│             │                           │                 │
│  Component  │                           │                 │
│    Init     │                           │                 │
└──────┬──────┘                           └─────────────────┘
       │                                           │
       │ PHASE 1: Component Preparation            │
       │ (No network activity)                     │
       │                                          │
       │ ✓ gRPC client infrastructure ready        │
       │ ✓ Connection pools prepared               │
       │ ✓ Event handlers configured               │
       │                                          │
       │ ... Application decides when to connect...│
       │                                          │
       │ PHASE 2: Explicit Connection             │
       │ register-with-server() called by app       │
       │ ──────────────────────────────────────── ▶│
       │        registerClient(clientId)             │
       │                                          │
       │              ◀ ──────────────────────────│
       │         Server registers client with      │
       │         integrated streaming capability   │
       │                                          │
       │ API connection pool activated             │
       │ ──────────────────────────────────────── ▶│
       │    4 pooled channels established          │
       │                                          │
       │              ◀ ──────────────────────────│
       │         API pool ready for requests       │
       │                                          │
       │ ✓ Bidirectional communication ready       │
       │   when application needs it               │
       │                                          │
```

### Benefits of App-Controlled Architecture

1. **Application Timing Control**: Apps can establish connections after UI frameworks initialize, windowing systems are ready, or other prerequisites are met
2. **Resource Management**: Network resources allocated precisely when needed, preventing premature resource allocation
3. **Error Separation**: Clear distinction between component setup failures vs network connectivity issues  
4. **Testing Control**: Tests can control exactly when connections are established for better resource lifecycle management

### Implementation Impact

```clojure
;; Before: Immediate connection during component init
(let [grpc-clients (ig/init-key :grpc-clients config)]
  ;; Network connections already established
  (use-client grpc-clients))

;; After: Explicit app-controlled connection
(let [grpc-clients (ig/init-key :grpc-clients config)]
  ;; Only component infrastructure ready - no network connections
  
  ;; App decides when to connect (e.g., after UI initialization)
  (register-with-server config grpc-clients)
  
  ;; Now bidirectional communication is ready
  (use-client grpc-clients))
```

---

## Understanding the Core Problem

Consider a typical collaborative scenario: multiple clients editing the same musical score, with varying network conditions and processing capabilities.

```
Collaborative Editing Scenario
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Desktop       │    │   Tablet        │    │   Mobile        │
│   Fast WiFi     │    │   Good WiFi     │    │   Slow 3G       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

The architectural question: How should communication patterns handle clients with different performance characteristics?

---

## API Requests: Why Flow Control Is Counterproductive

### The gRPC Threading Model

gRPC handles each request in a separate thread, allowing concurrent processing:

```
API Request Processing
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│ Client A    │    │              │    │   Backend   │
│ Add Note    │───▶│  Thread 1    │───▶│ Processing  │
└─────────────┘    │              │    └─────────────┘
                   │              │    
┌─────────────┐    │   gRPC       │    ┌─────────────┐
│ Client B    │    │   Server     │    │   Backend   │
│ Add Dynamic │───▶│  Thread 2    │───▶│ Processing  │
└─────────────┘    │              │    └─────────────┘
                   │              │    
┌─────────────┐    │              │    ┌─────────────┐
│ Client C    │    │              │    │   Backend   │
│ Change Key  │───▶│  Thread 3    │───▶│ Processing  │
└─────────────┘    └──────────────┘    └─────────────┘
```

Each client's request processes independently. A slow request from one client doesn't block requests from other clients.

### STM Concurrency Management

Ooloi's backend uses Software Transactional Memory (STM) to coordinate concurrent modifications:

```clojure
;; Each API call is wrapped in an STM transaction
(defn add-note [vpd piece-id pitch]
  (dosync                                    ; Transaction boundary
    (alter piece-ref                         ; Atomic update
      (fn [piece]
        (vpd/mutate vpd piece add-note-fn pitch)))))
```

STM handles conflicts when multiple clients edit the same musical element simultaneously by automatically retrying transactions.

### Why Flow Control Would Interfere

Adding flow control to API requests would create artificial bottlenecks:

```
API Flow Control (Problematic)
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│ Client A    │    │              │    │             │
│ Add Note    │───▶│   Queue      │    │   Waiting   │
└─────────────┘    │   System     │    │   State     │
                   │              │    │             │
┌─────────────┐    │              │    │             │
│ Client B    │    │              │    │   All       │
│ Add Dynamic │───▶│   Waiting    │    │   Requests  │
└─────────────┘    │              │    │   Queued    │
                   │              │    │             │
┌─────────────┐    │              │    │             │
│ Client C    │    │              │    │             │
│ Slow Change │───▶│   Processing │    │   Behind    │
└─────────────┘    └──────────────┘    │   This One  │
                                       └─────────────┘
```

This would override gRPC's natural concurrency and STM's conflict resolution mechanisms.

---

## Event Streaming: Why Flow Control Is Essential

### The Event Broadcasting Challenge

When changes occur, the server needs to notify all connected clients. With synchronous broadcasting, slow clients create bottlenecks:

```
Synchronous Event Broadcasting
                    ┌─────────────────┐
                    │     Server      │
                    │   Event Loop    │
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │ "Note Added to  │
                    │   Measure 47"   │
                    └─────────────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Client A    │ │  Client B    │ │   Client C   │
    │  (Fast)      │ │  (Fast)      │ │  (Slow)      │
    └──────────────┘ └──────────────┘ └──────────────┘
                                            │
                                    ┌───────▼────────┐
                                    │ Server blocks  │
                                    │ waiting for    │
                                    │ slow delivery  │
                                    │                │
                                    │ Other clients  │
                                    │ receive no     │
                                    │ updates        │
                                    └────────────────┘
```

The issue: One slow client prevents all clients from receiving timely updates.

### Real-World Impact

Consider these scenarios:

1. **Network congestion**: Cellist's 3G connection is saturated
2. **UI thread blocking**: Violinist's tablet is busy with expensive graphics rendering
3. **Client crash**: One client fails, should this break everyone else's collaboration?

In synchronous event delivery, any of these issues stops real-time collaboration for everyone.

---

## The Queue-Based Solution

### Per-Client Event Queues

Ooloi addresses this through independent per-client event queues:

```
Asynchronous Event Broadcasting
                    ┌─────────────────┐
                    │     Server      │
                    │   Event Loop    │
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │ "Note Added to  │
                    │   Measure 47"   │
                    └─────────────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │   Queue 1    │ │   Queue 2    │ │   Queue 3    │
    │ [Event1]     │ │ [Event1]     │ │ [Event1]     │
    │ [Event2]     │ │ [Event2]     │ │ [Event2]     │
    │ [Event3]     │ │ [Event3]     │ │ [Event3]...  │
    │     ...      │ │     ...      │ │ [Queue Full] │
    └──────▼───────┘ └──────▼───────┘ └──────▼───────┘
            │                │                │
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Client A    │ │  Client B    │ │   Client C   │
    │  (Fast)      │ │  (Fast)      │ │  (Slow but   │
    │              │ │              │ │   isolated)  │
    │  Immediate   │ │  Immediate   │ │              │
    │  updates     │ │  updates     │ │              │
    └──────────────┘ └──────────────┘ └──────────────┘
```

Each client receives events independently, preventing slow clients from affecting fast ones.

### Queue Architecture Details

**Implementation characteristics**:
- **1000-item capacity** per client queue
- **Drop-oldest policy** when queue overflows
- **Dedicated thread pool** for asynchronous delivery
- **Per-client statistics** for monitoring and health (separate top-level `:client-statistics` key)

```clojure
;; Simplified queue architecture
(defn broadcast-event [event]
  (doseq [[client-id client-state] @client-registry]
    ;; Each client gets independent queue processing
    (async/put! (:event-queue client-state) event)))

(defn process-client-events [client-id]
  ;; Each client processed by separate thread
  (async/go-loop []
    (when-let [event (async/<! (:event-queue @client-state))]
      (try
        (.onNext (:stream-observer client-state) event)
        (catch Exception e
          (log/warn "Client" client-id "event delivery failed")))
      (recur))))
```

### Graceful Degradation

When clients fall behind:

```
Queue Overflow Handling
┌─────────────────┐
│   Client Queue  │
│  [Event-995]    │  ← Oldest event
│  [Event-996]    │
│  [Event-997]    │
│  [Event-998]    │
│  [Event-999]    │  
│  [Event-1000]   │  ← Queue full!
└─────────────────┘
         │
         ▼
┌─────────────────┐    New event arrives
│ Drop Event-995  │ ←  Drop oldest, add newest
│ Add Event-1001  │    Client gets recent updates
└─────────────────┘    without blocking others
```

**Graceful degradation principles**:
- Slow clients receive recent events (not old ones)
- Fast clients always get real-time updates
- No client can disrupt the system for others

---

## STM and gRPC: Complementary Technologies

### Concurrent Request Processing

The combination of gRPC's threading model and STM's transaction system handles concurrent operations efficiently:

```clojure
;; Multiple simultaneous API calls
;; Thread 1:
(api/add-note [:m 0 1 0 0 3] piece-id :C4)

;; Thread 2 (concurrent):
(api/set-dynamic [:m 0 1 0 2] piece-id :forte)

;; Thread 3 (concurrent):
(api/change-key-signature [:m 0 2] piece-id :G-major)
```

**What happens internally**:
1. Each call gets its own gRPC thread
2. Each thread wraps the operation in an STM transaction
3. STM coordinates conflicts automatically
4. Operations that don't conflict run in parallel
5. Operations that do conflict retry transparently

### Transaction Boundaries

```
STM Transaction Flow
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   gRPC      │    │     STM      │    │   Musical   │
│  Request    │    │ Transaction  │    │   State     │
│             │    │              │    │             │
│ add-note    │───▶│   (dosync    │───▶│  Piece Ref  │
│             │    │     (alter   │    │   Updated   │
│             │    │      ...))   │    │             │
└─────────────┘    └──────────────┘    └─────────────┘
                           │
                   ┌───────▼───────┐
                   │ Automatic     │
                   │ Retry on      │
                   │ Conflict      │
                   └───────────────┘
```

**Key insight**: STM transaction boundaries align perfectly with gRPC request boundaries, creating natural atomicity for musical operations.

---

## Practical Examples

### Example 1: High-Volume API Calls

A client application making many rapid API calls:

```clojure
;; Application making multiple concurrent calls
(defn upload-analysis-results [piece-id note-list]
  (dotimes [i (count note-list)]
    ;; Each call: separate gRPC thread + STM transaction
    (execute-method client "add-note" 
                   {:vpd [:m 0 1 i 0 3] 
                    :piece-id piece-id
                    :pitch (nth note-list i)})))
```

Without flow control: All calls process concurrently, with STM handling any conflicts.

With flow control: Calls would queue behind each other, reducing concurrency benefits.

### Example 2: Real-Time Collaboration

Three musicians editing simultaneously while a fourth watches:

```
Musicians Making Changes        Server Processing           Event Broadcasting
┌─────────────────┐            ┌─────────────────┐         ┌─────────────────┐
│ Conductor:      │───────────▶│ Thread 1:       │         │ Event: "Tempo   │
│ "Set tempo=120" │            │ STM Transaction │────────▶│  Changed"       │
└─────────────────┘            │ Process & Store │         └─────────────────┘
                               └─────────────────┘                   │
┌─────────────────┐            ┌─────────────────┐                   │
│ Violinist:      │───────────▶│ Thread 2:       │                   ▼
│ "Add staccato"  │            │ STM Transaction │         ┌─────────────────┐
└─────────────────┘            │ Process & Store │────────▶│ All Musicians   │
                               └─────────────────┘         │ Get Updates     │
┌─────────────────┐            ┌─────────────────┐         │ Via Individual  │
│ Cellist:        │───────────▶│ Thread 3:       │         │ Event Queues    │
│ "Change clef"   │            │ STM Transaction │────────▶│                 │
└─────────────────┘            │ Process & Store │         └─────────────────┘
                               └─────────────────┘
```

**API requests**: No flow control, maximum concurrency via STM + gRPC

**Event streaming**: Flow control ensures all musicians receive updates regardless of individual network conditions

### Example 3: Network Failure Scenarios

```
Network Issues During Collaboration
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Conductor       │    │ Violinist       │    │ Cellist         │
│ (Making API     │    │ (Normal         │    │ (Network        │
│  calls)         │    │  operation)     │    │  disconnected)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ API calls       │    │ Receives events │    │ Queue fills     │
│ continue        │    │ normally        │    │ gracefully      │
│ processing      │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**Result**: Conductor and violinist continue normal collaboration. When cellist reconnects, they receive recent (not stale) updates.

---

## Performance Implications

### API Request Performance

Concurrency characteristics:
- Performance scales with available CPU cores
- gRPC and STM work together without artificial constraints
- STM handles conflicts through automatic retry mechanisms

### Event Structure and Conventions

All events follow a consistent structure to ensure predictable handling across the event streaming pipeline:

```clojure
;; Standard event structure received by clients
{:type :event-type-keyword          ; Required: Event type as keyword
 :timestamp 1693827465123456789     ; Required: Nanosecond timestamp (added during transport)
 :client-id "client-123"            ; Optional: Originating client identifier
 :message "Human readable message"  ; Optional: User-friendly description
 ...additional-fields...}           ; Event-specific data fields
```

**Event Type Conventions**:
- **Always keywords**: `:server-maintenance`, `:client-registration-confirmed`, `:piece-updated`
- **Kebab-case naming**: Multi-word types use dashes (`:client-registration-confirmed`)
- **Hierarchical naming**: Complex events can use namespace-style (`:piece/content-changed`)

**Core Event Categories**:
- **Server events**: `:server-maintenance`, `:server-shutdown`, `:server-status`
- **Client events**: `:client-registration-confirmed`, `:client-connected`
- **Piece events**: `:piece-updated`, `:piece-created`, `:piece-deleted`
- **Collaboration events**: `:user-joined`, `:user-left`, `:concurrent-edit`

**Event Processing Pipeline**:
1. **Server creation**: Event data created with `:type` and content fields
2. **Transport**: Event data serialized to protobuf, timestamp added to EventMessage wrapper
3. **Client reception**: EventMessage metadata (timestamp) merged with event data
4. **Final structure**: Client applications receive complete event with all metadata

For comprehensive event structure documentation, see [ADR-0018: API-gRPC Interface and Events](../ADRs/0018-API-gRPC-Interface-and-Events.md#event-structure-and-conventions).

### Event Broadcasting Examples

The server provides two optimized functions for event broadcasting with automatic field derivation. Both functions feature identical signatures for consistency and ease of use.

#### Server Events - Global Broadcasts

Server events are broadcast to all connected clients regardless of piece subscriptions. Use these for system-wide notifications:

```clojure
;; System maintenance notification
(send-server-event server-component
  {:type :server-maintenance
   :message "System will restart in 5 minutes"
   :affected-services ["api" "streaming"]
   :estimated-downtime "2 minutes"})

;; Server status updates  
(send-server-event server-component
  {:type :server-status
   :status :degraded
   :message "Experiencing high load"
   :response-time-ms 850})

;; User session events
(send-server-event server-component
  {:type :server-user-limit
   :message "Maximum concurrent users reached"
   :current-users 100
   :max-users 100})
```

#### Piece Events - Targeted Broadcasts

Piece events are broadcast only to clients subscribed to the specific piece. Use these for piece-specific notifications:

```clojure
;; Content changes in a musical piece
(send-piece-event server-component
  {:type :piece-content-changed
   :piece-id "symphony-op27"
   :measures [45 46 47]
   :change-type :notes-added
   :editor-client "user-maria"
   :instruments ["violin-1" "violin-2"]})

;; Collaborative editing events
(send-piece-event server-component
  {:type :piece-user-joined
   :piece-id "quartet-in-d"
   :user-id "user-james"
   :user-role "violinist"
   :join-time (System/nanoTime)})

;; Auto-save notifications  
(send-piece-event server-component
  {:type :piece-auto-saved
   :piece-id "concerto-no3"
   :save-version 47
   :changed-sections ["exposition" "development"]})

;; Layout invalidation
(send-piece-event server-component
  {:type :piece-layout-invalidated
   :piece-id "sonata-k545"
   :affected-pages [1 2]
   :reason "time-signature-change"
   :requires-reflow true})
```

#### Practical Usage Patterns

**Event Chaining for Complex Operations**:
```clojure
;; Multi-step operation with progress updates
(defn transpose-piece [server-component piece-id semitones]
  ;; Notify start
  (send-piece-event server-component
    {:type :piece-operation-started
     :piece-id piece-id
     :operation :transpose
     :parameters {:semitones semitones}})
  
  ;; Perform operation
  (let [result (perform-transposition piece-id semitones)]
    ;; Notify completion
    (send-piece-event server-component
      {:type :piece-operation-completed
       :piece-id piece-id
       :operation :transpose
       :affected-measures (:measures result)
       :success true})))
```

**Error Event Broadcasting**:
```clojure
;; Piece-specific errors
(send-piece-event server-component
  {:type :piece-operation-failed
   :piece-id "symphony-unfinished" 
   :operation :auto-harmonize
   :error-code "insufficient-voice-leading"
   :message "Cannot harmonize: voice leading constraints violated"
   :suggested-action "Review voice ranges in measures 23-27"})

;; Server-wide errors
(send-server-event server-component
  {:type :server-error
   :error-level :warning
   :message "Database connection pool exhausted"
   :affected-operations ["save" "auto-save"]
   :recovery-eta "30 seconds"})
```

#### Integration with Component Lifecycle

```clojure
;; In your component's event handling
(defn handle-piece-update [server-component piece-id changes]
  (when (significant-change? changes)
    (send-piece-event server-component
      {:type :piece-content-changed
       :piece-id piece-id
       :changes changes
       :change-id (generate-change-id)
       :requires-layout-update (layout-affecting? changes)}))
  
  ;; Check for conflicts
  (when-let [conflicts (detect-conflicts piece-id changes)]
    (send-piece-event server-component
      {:type :piece-conflict-detected
       :piece-id piece-id
       :conflicts conflicts
       :resolution-required true})))
```

**Key Benefits of This Approach**:
- **Simplified signatures**: Both functions use identical `(server-component event-data)` pattern
- **Automatic validation**: Events validated against ADR conventions before broadcast
- **Perfect consistency**: Event-type strings automatically derived from `:type` fields
- **Type safety**: Piece events automatically validated for required `:piece-id` field
- **Nanosecond precision**: Timestamps automatically generated for accurate timing

### Event Streaming Performance

Resource characteristics:
- Memory overhead: approximately 1000 events per client queue
- CPU overhead: dedicated thread pool for asynchronous delivery
- Network considerations: opportunities for batching and compression
- Monitoring: per-client delivery statistics available

### Memory Calculations

For typical deployment scenarios:

```
Example: 10 clients, 1000 events/queue, 500 bytes/event
Memory overhead = 10 × 1000 × 500 bytes = 5MB

Larger deployment: 50 clients
Memory overhead = 50 × 1000 × 500 bytes = 25MB
```

The memory cost is generally small relative to the architectural benefits gained.

---

## Related Architecture

### Integration with Existing Systems

This flow control architecture builds upon and enhances several existing Ooloi architectural decisions:

**STM for Concurrency ([ADR-0004](../ADRs/0004-STM-for-concurrency.md))**:
- Flow control decision preserves STM's natural conflict resolution
- API requests continue to use STM's optimised coordination
- No artificial constraints on transaction processing

**gRPC Communication ([ADR-0002](../ADRs/0002-gRPC.md))**:
- Maintains gRPC's thread-per-request model for API calls
- Adds sophisticated event streaming on top of gRPC's streaming capabilities
- Preserves all deployment models (combined, distributed, etc.)

**API-gRPC Interface and Events ([ADR-0018](../ADRs/0018-API-gRPC-Interface-and-Events.md))**:
- Generated gRPC interfaces continue to work without flow control
- Event streaming adds new capabilities without breaking existing patterns
- Bidirectional communication enhanced by proper flow control

**In-Process Transport Optimization ([ADR-0019](../ADRs/0019-In-Process-gRPC-Transport-Optimization.md))**:
- Flow control works equally well with network and in-process transport
- Performance benefits of in-process transport maintained

**Server Statistics Architecture ([ADR-0025](../ADRs/0025-Server-Statistics-Architecture.md))**:
- Event queue health monitoring enables proactive flow control management
- Per-client queue statistics (top-level `:client-statistics` key) inform overflow prevention strategies  
- Performance metrics guide flow control parameter tuning
- Queue overhead minimal compared to transport optimisation gains

### Future Extensions

**Authentication Integration ([ADR-0021](../ADRs/0021-Authentication.md))**:
- Per-client queues support per-user authentication naturally
- Event filtering can be applied based on user permissions
- JWT tokens can be validated at queue level

**TLS Infrastructure ([ADR-0020](../ADRs/0020-TLS-Infrastructure-and-Deployment-Architecture.md))**:
- Event streaming works transparently with TLS
- Queue-based architecture doesn't interfere with certificate management
- Performance characteristics maintained across all security configurations

---

## Conclusion

Different communication patterns have different concurrency requirements and constraints.

**API requests work well with**:
- gRPC's natural threading model
- STM's automatic conflict resolution
- No additional flow control layer

**Event streaming requires**:
- Client isolation through independent queues
- Asynchronous delivery to prevent blocking
- Graceful degradation for slow or failed clients

This approach applies the appropriate concurrency model to each communication pattern rather than using a uniform solution. The result is a system that handles both high-throughput API operations and reliable real-time event distribution, regardless of individual client performance characteristics.

For implementers of custom gRPC clients, these patterns are relevant regardless of the client-side programming language or framework used.

---

## Related Documentation

### Deep Dive Server Architecture
- **[OOLOI_SERVER_ARCHITECTURAL_GUIDE.md](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Comprehensive analysis of the server architecture implementing these communication patterns, including enterprise comparisons and production characteristics

### Architectural Decision Records
- **[ADR-0024: gRPC Concurrency and Flow Control Architecture](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)** - Detailed technical decisions behind these communication patterns

### Implementation Guides
- **[PIECE_MANAGER_GUIDE.md](PIECE_MANAGER_GUIDE.md)** - STM-based storage operations that integrate with gRPC transactions
- **[ADVANCED_CONCURRENCY_PATTERNS.md](ADVANCED_CONCURRENCY_PATTERNS.md)** - Advanced STM coordination patterns used in server implementation
