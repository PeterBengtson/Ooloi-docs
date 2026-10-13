# 🟠 gRPC Communication and Flow Control: Understanding Ooloi's Communication Architecture

**This guide explains Ooloi's gRPC communication patterns and the rationale behind its flow control decisions. While the examples use Clojure, the principles apply to any gRPC client implementation.**

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Introduction: Two Communication Patterns](#introduction-two-communication-patterns)
- [Explicit App-Controlled Connection Architecture](#explicit-app-controlled-connection-architecture)
- [Understanding the Core Problem](#understanding-the-core-problem)
- [API Requests: Why Flow Control Is Counterproductive](#api-requests-why-flow-control-is-counterproductive)
- [Event Streaming: Why Flow Control Is Essential](#event-streaming-why-flow-control-is-essential)
- [The Queue-Based Solution](#the-queue-based-solution)
- [STM and gRPC: Complementary Technologies](#stm-and-grpc-complementary-technologies)
- [Practical Examples](#practical-examples)
- [Performance Implications](#performance-implications)
  - [Event Structure and Conventions](#event-structure-and-conventions)
  - [Event Broadcasting Examples](#event-broadcasting-examples)
  - [Event Streaming Performance](#event-streaming-performance)
- [Related Architecture](#related-architecture)
- [Conclusion](#conclusion)
- [Related Documentation](#related-documentation)

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
Real-time notifications naming what has gone stale:
- "The structure of this piece has changed"
- "Measures 45–47 of this piece need redrawing"
- "The instrument library has changed"

These patterns have different concurrency characteristics and flow control requirements.

---

## Explicit App-Controlled Connection Architecture

### Connection Sequence

Ooloi clients use a **bracketed connection architecture**: component initialization prepares the gRPC infrastructure without network activity, applications explicitly establish the connection via `register-with-server`, and explicit `Disconnect` calls (typically from `switch-to!`) close the connection symmetrically. The `setOnCancelHandler` backstop catches involuntary disconnects.

```
Client Connection Sequence
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
       │ ... Client operates normally ...          │
       │                                          │
       │ PHASE 3: Graceful Disconnect (RPC)       │
       │ disconnect-from-server() called          │
       │ ──────────────────────────────────────── ▶│
       │     Disconnect(empty),                   │
       │     client-id from CLIENT_ID_HEADER       │
       │     in gRPC context                      │
       │                                          │
       │                          Server handler:│
       │                            dissoc entry │
       │                            drainer down │
       │                            broadcast    │
       │                                          │
       │              ◀ ──────────────────────────│
       │         {success:true, error:""}         │
       │                                          │
       │ Client tears down channels               │
       │ ──────────────────────────────────────── ▶│
       │         (setOnCancelHandler backstop,    │
       │          identity-aware: no-op if entry  │
       │          already cleaned by Disconnect)  │
       │                                          │
```

An involuntary disconnect — a network drop, or the client's JVM exiting — skips PHASE 3 entirely, and the `setOnCancelHandler` backstop is then what cleans up. A server halt ends its clients' sessions from the other side: the halting server completes each of its own clients' streams with `.onCompleted`, stops their drainer executors and removes their registry entries itself ([ADR-0024 §Server Shutdown: Graceful Only, Never Forced](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md#server-shutdown-graceful-only-never-forced)). See [ADR-0024 §Connection Lifecycle](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) for the full lifecycle model and the identity-aware handler design.

### Benefits of App-Controlled Architecture

1. **Application Timing Control**: Apps can establish connections after UI frameworks initialize, windowing systems are ready, or other prerequisites are met
2. **Resource Management**: Network resources allocated precisely when needed, preventing premature resource allocation
3. **Error Separation**: Clear distinction between component setup failures vs network connectivity issues  
4. **Testing Control**: Tests can control exactly when connections are established for better resource lifecycle management

### Implementation Impact

A sketch of the sequence (`use-client` stands for whatever the application does next; `register-with-server` is in `ooloi.frontend.grpc.event-client`):

```clojure
(let [grpc-clients (ig/init-key :ooloi.frontend.components/grpc-clients config)]
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

Ooloi's backend uses Software Transactional Memory (STM) to coordinate concurrent modifications. Every VPD mutation — adders, removers, setters, movers — passes through one write funnel, `vpd-transact` in `shared/src/main/clojure/ooloi/shared/models/core.clj`, which runs it as a transaction on the piece's ref:

```clojure
(defn vpd-transact
  "Execute an STM transaction that applies an operation function to a VPD-resolved
  location within a piece. Used by adders, removers, setters, movers, set-seq,
  set-attribute, and change-set operations. After the mutation, notifies the change
  detector (ADR-0052): a structural-slot change schedules one :piece-structure-changed,
  deferred to commit."
  [op-fn vpd piece-or-id-or-ref attr-name args]
  (let [piece-ref (i/resolve-into-piece-ref piece-or-id-or-ref)]
    (dosync
     (let [before @piece-ref
           after  (alter piece-ref
                         (fn [piece] (apply op-fn vpd piece attr-name args)))]
       (cd/detect-change! before after vpd attr-name)
       after))))
```

When the call arrives over gRPC, the server's handler opens the enclosing transaction itself, so that it can read the piece value before and after the mutation and record one undo entry for the call; the funnel's `dosync` nests into it.

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
- **One single-thread executor per client** — the drainer — for asynchronous delivery; its thread is the only one that writes to that client's stream
- **Per-client statistics** for monitoring and health (separate top-level `:client-statistics` key)

Each client's queue is a `java.util.concurrent.ArrayBlockingQueue`, held in its connection-registry entry. A simplified sketch of the two halves (the real functions, `send-server-event`, `send-piece-event` and `start-stream-drainer!` in `backend/src/main/clojure/ooloi/backend/grpc/server.clj`, are shown and explained in [gRPC Server Streaming: Threading and Backpressure in Clojure](GRPC_STREAMING_THREADING_GUIDE.md)):

```clojure
;; Sketch. Publishing: enqueue for each recipient, then kick its drainer.
(doseq [[client-id {:keys [event-queue drain]}] @registry]
  (offer-with-drop-oldest event-queue event-message)   ; drops the oldest when full
  (drain))                                             ; schedules a drain on the client's executor

;; Sketch. Draining, on the client's own executor thread:
(loop []
  (when (and (.isReady server-obs) (not (.isCancelled server-obs)))
    (when-some [ev (.poll event-queue)]
      (try
        (.onNext server-obs ev)
        (catch Exception e
          (tp/log-error e)))      ; recorded; the next event is still delivered
      (recur))))
```

A failed delivery is recorded through `tp/log-error`, the single surfacing boundary of [ADR-0017](../ADRs/0017-System-Architecture.md), and never ends the stream.

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
;; Multiple simultaneous API calls from three clients, each one ExecuteMethod RPC
;; Client A:
(SRV/add-item [:musicians 0 :instruments 0 :staves 0 :measures 5 :voices 0] piece-id note)

;; Client B (concurrent):
(SRV/set-name [:musicians 1] piece-id "Viola")

;; Client C (concurrent):
(SRV/set-tempo [] piece-id 12 tempo)
```

**What happens internally**:
1. Each call gets its own gRPC thread
2. Each thread runs the operation in an STM transaction
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
;; Application making many calls: one note into each successive measure
(defn upload-analysis-results [piece-id note-list]
  (doseq [[i note] (map-indexed vector note-list)]
    ;; Each call: separate ExecuteMethod RPC, gRPC thread and STM transaction
    (SRV/add-item [:musicians 0 :instruments 0 :staves 0 :measures i :voices 0]
                  piece-id
                  note)))
```

Each of these calls is also its own undo step. Where the whole upload should be one atomic change and one undo step, the operations go into a single `SRV/atomic` batch instead ([POLYMORPHIC_API_GUIDE](POLYMORPHIC_API_GUIDE.md)).

Without flow control: each call is processed as soon as it arrives, concurrently with every other client's calls, with STM handling any conflicts.

With flow control: calls would queue behind one another and behind other clients' calls, reducing concurrency benefits.

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

### Message Size Limits

gRPC caps the size of a single received message. Ooloi sets this limit to 64 MB on both client and server (gRPC's own default is 4 MB). The limit governs one message — a single API call's request or response. No whole pieces are ever transferred (there is no `get-piece`/`set-piece`), but a single query result or mutation argument can still be large, and 64 MB gives it comfortable headroom.

The limit is enforced by the **receiver**, so it must be set on both sides: a large result has to clear the client's limit, a large argument the server's.

One case is deliberately not bound by this limit: a large multistep atomic batch. Rather than arrive as one oversized message, its operations are chunked and streamed, so the batch is limited only by the memory its resulting transaction needs on the server. In-process transport has no message-size limit at all — it does no serialization.

See [ADR-0024](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) for the authoritative configuration and rationale.

### Event Structure and Conventions

All events follow a consistent structure to ensure predictable handling across the event streaming pipeline:

```clojure
;; Standard event structure received by clients
{:type :event-type-keyword          ; Required: Event type as keyword
 :timestamp 1729800000000000        ; Required: epoch microseconds (added automatically by the server)
 :client-id "client-123"            ; Optional: Originating client identifier
 :message "Human readable message"  ; Optional: User-friendly description
 :piece-id "piece-123"              ; Required for piece events: Target piece identifier
 ...additional-fields...}           ; Event-specific data fields
```

**Event Type Conventions**:
- **Always keywords**: `:server-maintenance`, `:client-registration-confirmed`, `:piece-structure-changed`
- **Kebab-case naming**: Multi-word types use dashes (`:client-registration-confirmed`)
- **The prefix names the delivery**: every type begins with `server-`, `client-`, `piece-`, `collaboration-`, `instrument-library-` or `undo-`, which the server's event validation enforces. A `piece-` event is sent with `send-piece-event` and reaches only the clients subscribed to that piece; every other prefix is sent with `send-server-event` to every connected client. The validator also accepts a namespace-style name containing `/`, but no Ooloi event uses one.

**Core Event Categories** (the taxonomy of ADR-0018):
- **Piece events**, delivered to that piece's subscribers:
  - `:piece-structure-changed`, `:piece-dirty-changed` — structural metadata changed; the unsaved-changes flag flipped
  - `:piece-invalidation` — visual hierarchy invalidation at any level; the depth of the VPD carries the level
  - `:piece-playback-position`, `:piece-playback-started`, `:piece-playback-stopped`
  - `:piece-collaboration-user-joined`, `:piece-collaboration-user-left`, `:piece-collaboration-cursor-moved`, `:piece-collaboration-selection-changed`
  - `:piece-validation-error`, `:piece-operation-failed`
- **Global events**, delivered to every connected client:
  - `:server-maintenance`, `:server-shutdown`, `:server-status`
  - `:server-client-connected`, `:server-client-disconnected` (sent by a network server only)
  - `:server-warning`, `:server-info`
  - `:client-registration-confirmed` — the one event a client receives about itself, carrying `:server-timestamp` for its clock offset
  - `:instrument-library-changed`
- **Delivery by payload**: `:undo-state-changed` goes to every client when its `:resource-key` is the Instrument Library, and only to a piece's subscribers when it is a piece

The `collaboration-` prefix is accepted but unused: collaboration is always about a piece, so those events carry the `piece-` prefix.

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
   :message "System update in 5 minutes"})

;; Warning to every connected client
(send-server-event server-component
  {:type :server-warning
   :message "Server is under heavy load"})
```

#### Piece Events - Targeted Broadcasts

Piece events are broadcast only to clients subscribed to the specific piece. Use these for piece-specific notifications:

```clojure
;; Formatting invalidation, scoped by measure numbers
(send-piece-event server-component
  {:type :piece-invalidation
   :piece-id "symphony-123"
   :measures [47 48 49]})

;; Formatting invalidation, scoped by VPD: the depth carries the level (here a page)
(send-piece-event server-component
  {:type :piece-invalidation
   :piece-id "symphony-123"
   :vpd [:layouts 0 :page-views 2]})

;; A collaborator joined this piece
(send-piece-event server-component
  {:type :piece-collaboration-user-joined
   :piece-id "quartet-in-d"
   :user-id "user-james"
   :user-name "James"})

;; An operation on this piece failed
(send-piece-event server-component
  {:type :piece-operation-failed
   :piece-id "symphony-unfinished"
   :message "Cannot harmonise: voice-leading constraints violated"})
```

`:piece-structure-changed` and `:piece-dirty-changed` are not sent by hand. The change detector schedules them inside the mutating transaction and the notifier agent delivers them on commit, through the `PieceChangeNotifier` seam that the server implements with `send-piece-event` ([ADR-0052 §4](../ADRs/0052-Change-Detection-and-Event-Generation.md#4-exactly-one-event-per-outermost-transaction) and [§5](../ADRs/0052-Change-Detection-and-Event-Generation.md#5-the-dirty-flag)).

#### What an Event Carries

An event names what has gone stale and carries the identifiers needed to find it — a piece-id, a VPD or VPDs, measure numbers. It never carries the change itself: no change type, no changed values, no list of affected elements beyond the scope ([ADR-0022 §Communication Patterns](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md#communication-patterns); [ADR-0052 §6](../ADRs/0052-Change-Detection-and-Event-Generation.md#6-two-emission-regimes)). A client that cares fetches the current state when it needs it; one that is not displaying the data marks its cache stale and fetches nothing.

Timestamps are the server's business. The publishing functions add `:timestamp` in epoch microseconds (`ooloi.shared.ops.time/epoch-usec`), which is comparable across machines and across a restart. `System/nanoTime` is monotonic but has no epoch relation, so it belongs only in durations measured inside one process and never in a field that crosses the wire ([ADR-0018 §Event Consistency Requirements](../ADRs/0018-API-gRPC-Interface-and-Events.md#event-consistency-requirements)).

**Key Benefits of This Approach**:
- **Simplified signatures**: Both functions use identical `(server-component event-data)` pattern
- **Automatic validation**: Events validated against ADR conventions before broadcast
- **Consistency**: Event-type strings automatically derived from `:type` fields
- **Type safety**: Piece events automatically validated for required `:piece-id` field
- **Comparable timestamps**: Epoch-microsecond timestamps generated automatically by the server

### Event Streaming Performance

Resource characteristics:
- Memory overhead: approximately 1000 events per client queue
- Thread overhead: one single-thread drainer executor per client for asynchronous delivery
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

**Frontend Event-Driven Architecture ([ADR-0031](../ADRs/0031-Frontend-Event-Driven-Architecture.md))**:
- Event Router consumes the FIFO-ordered event stream this flow control provides
- Rendering Data Manager relies on predictable event delivery for paintlist invalidation
- Fetch Coordinator priority queues integrate with event-driven invalidation patterns

**Backend-Authoritative Rendering ([ADR-0038](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md))**:
- Piece invalidation events drive frontend paintlist cache updates
- GPU-accelerated rendering execution depends on reliable event delivery
- Terminal frontend execution model requires backend events as sole source of rendering changes

### Future Extensions

**Authentication Integration ([ADR-0021](../ADRs/0021-Authentication.md))**:
- Per-client queues support per-user authentication naturally
- Event filtering can be applied based on user permissions
- JWT tokens can be validated at queue level

**TLS Infrastructure ([ADR-0020](../ADRs/0020-TLS-Infrastructure-and-Deployment-Architecture.md))**:
- Event streaming works transparently with TLS
- Queue-based architecture doesn't interfere with certificate management
- Performance characteristics maintained across all security configurations
- **Known temporary limitation:** *direct peer-to-peer* encryption is not yet trustable — a peer's auto-generated certificate is self-signed and cannot be validated without insecure-dev-mode (a dev-cycle convenience, never shipped). Peer-to-peer sessions run unencrypted on a trusted LAN until a trust-on-first-use / automated certificate-distribution mechanism lands. Encryption-on works today against a CA-signed host (e.g. the shared Ooloi server). See ADR-0020 §"Known Temporary Limitation: Secure Peer-to-Peer Trust".

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
- **[FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)** - Frontend event architecture and how gRPC events drive rendering
- **[PIECE_MANAGER_GUIDE.md](PIECE_MANAGER_GUIDE.md)** - STM-based storage operations that integrate with gRPC transactions
- **[ADVANCED_CONCURRENCY_PATTERNS.md](ADVANCED_CONCURRENCY_PATTERNS.md)** - Advanced STM coordination patterns used in server implementation
