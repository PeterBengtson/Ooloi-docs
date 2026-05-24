# ADR: gRPC Concurrency and Flow Control Architecture

Decision: Implement flow control only for server-to-client event streaming, not for client-to-server API requests, based on gRPC's inherent concurrency model and STM's conflict resolution capabilities.

## Table of Contents

- [Context](#context)
- [Client Connection Architecture](#client-connection-architecture)
  - [Explicit App-Controlled Connection Pattern](#explicit-app-controlled-connection-pattern)
  - [Connection Architecture Benefits](#connection-architecture-benefits)
  - [Event Notification Architecture](#event-notification-architecture)
  - [Connection Count Impact](#connection-count-impact)
  - [Client Validation Architecture](#client-validation-architecture)
- [Rationale](#rationale)
  - [API Request Concurrency: Flow Control Not Needed](#api-request-concurrency-flow-control-not-needed)
  - [Event Streaming: Flow Control Required](#event-streaming-flow-control-required)
- [Decision Details](#decision-details)
- [Consequences](#consequences)
- [Related Decisions](#related-decisions)
- [Implementation References](#implementation-references)

## Context

Ooloi's gRPC architecture handles two distinct communication patterns:
1. **Request/Response API calls**: Client requests data or mutations from server
2. **Server-to-client event streaming**: Server pushes real-time notifications to clients

During flow control system design, analysis revealed fundamental differences in how these patterns handle concurrency and potential bottlenecks. Understanding where flow control is needed (and where it's counterproductive) is crucial for system performance and architectural integrity.

Key considerations:
- Multi-threaded client applications (piece uploads, plugin bombardment, UI interactions)
- STM-based server designed for concurrent transaction processing
- Event streaming requiring coordination between fast and slow clients
- gRPC's built-in concurrency model vs custom flow control needs

## Client Connection Architecture

### **Explicit App-Controlled Connection Pattern**

Ooloi clients use a **six-phase connection lifecycle** where component initialization creates client infrastructure but applications control when network connections are actually established, followed by comprehensive event notifications for client coordination, and ending with an explicit Disconnect RPC plus an always-on cancel-handler backstop:

```
Client Connection Lifecycle
┌─────────────────┐                    ┌─────────────────┐
│  Frontend       │                    │   Backend       │
│  Component      │                    │   Server        │
│  Initialization │                    │                 │
└─────────┬───────┘                    └─────────┬───────┘
          │                                      │
          │ PHASE 1: Component Setup             │
          │ (no network activity)                │
          │                                      │
          │ ✓ gRPC client components ready       │
          │ ✓ Connection pools prepared          │
          │ ✓ Event handlers configured          │
          │                                      │
          │ ... Application decides when ...     │
          │                                      │
          │ PHASE 2: Explicit Connection         │
          │ register-with-server() called        │
          │ ───────────────────────────────────► │
          │    registerClient(client-id)         │
          │                                      │
          │ ◄ ────────────────────────────────── │
          │    Event streaming established       │
          │    (registered in connection         │
          │     registry for flow control)       │
          │                                      │
          │ PHASE 3: Connection Confirmation     │
          │ ◄ ────────────────────────────────── │
          │    Confirmation event sent to        │
          │    this client only                  │
          │                                      │
          │ PHASE 4: Connection Broadcast        │
          │ ◄ ────────────────────────────────── │
          │    server-client-connected event     │
          │    sent to ALL connected clients     │
          │    (including this one)              │
          │                                      │
          │ API pool activated for requests      │
          │ ───────────────────────────────────► │
          │    4 concurrent channels active      │
          │                                      │
          │ ✓ Bidirectional communication        │
          │   ready when app needs it            │
          │                                      │
          │ ... Client operates normally ...     │
          │                                      │
          │ PHASE 5: Graceful Disconnect (RPC)   │
          │ disconnect-from-server() called      │
          │ ───────────────────────────────────► │
          │   Disconnect(empty), client-id       │
          │   from CLIENT_ID_HEADER context      │
          │                                      │
          │                  Server-side handler:│
          │                    swap-vals! dissoc │
          │                    drainer shutdown  │
          │                    broadcast PHASE 6 │
          │                                      │
          │ ◄ ────────────────────────────────── │
          │   {success:true, error:""}           │
          │                                      │
          │ PHASE 6: Disconnect Broadcast        │
          │                    ─────────────────►│
          │   server-client-disconnected event   │
          │   sent to ALL remaining clients      │
          │                                      │
          │ Client tears down channels           │
          │ (after Disconnect returns)           │
          │ ───────────────────────────────────► │
          │   Channel close triggers             │
          │   setOnCancelHandler                 │
          │   (identity-aware backstop;          │
          │    no-op if entry already gone)      │
          │                                      │
```

### **Connection Architecture Benefits**

1. **Application-controlled timing**: Apps can establish connections after UI initialization, windowing system readiness, or other prerequisites
2. **Resource management**: Network resources allocated precisely when needed, not during component startup
3. **Error separation**: Clear distinction between component initialization failures vs network connectivity issues
4. **Testing control**: Tests can control exactly when connections are established for better resource management
5. **Client awareness**: All connected clients receive notifications about connection/disconnection events for coordination
6. **Connection confirmation**: New clients receive explicit confirmation that their connection was successful
7. **Real-time collaboration**: Event broadcasting enables clients to respond to other clients joining or leaving

### **Event Notification Architecture**

The connection lifecycle includes comprehensive event notifications for client coordination:

**Connection Events**:
- **Phase 3 - Connection Confirmation**: Server sends a confirmation event exclusively to the newly connected client
- **Phase 4 - Connection Broadcast**: Server sends `server-client-connected` event to ALL connected clients (including the new client)

**Disconnection Events**:
- **Phase 5 - Graceful Disconnect (RPC)**: Client invokes the unary `Disconnect` RPC (typically from `switch-to!` during transport switching, or at application shutdown). The server's `handle-disconnect` reads the client-id from the gRPC context (populated by the auth interceptor from `CLIENT_ID_HEADER`), atomically dissocs the registry entry via `swap-vals!`, shuts down the per-client drainer executor, and broadcasts the disconnect event (PHASE 6). The handler does **not** call `.onCompleted` on the streaming `RegisterClient` call from this code path: the same client typically owns both the Disconnect unary call and the streaming call, and a server-initiated `.onCompleted` would fire client-side teardown that closes the API channels — including the one carrying the Disconnect's still-in-flight response — causing a self-closing race. The streaming call is cleaned up instead via PHASE 7's cancel-handler backstop after the client tears down channels.
- **Phase 6 - Disconnect Broadcast**: Server sends `server-client-disconnected` event to ALL remaining connected clients. Gated on whether the dissoc actually changed the registry (`(not= old new)` from `swap-vals!`), so an idempotent no-op Disconnect (client-id already absent) does not emit a spurious broadcast.
- **Phase 7 - Cancel-Handler Backstop**: When the client tears down its channels (after Disconnect returns, or involuntarily on network drop / JVM exit / server-initiated kick), the gRPC layer fires `setOnCancelHandler` on the server. This handler is **identity-aware**: it captures the response-observer in its closure at registration time, and at fire time atomically dissocs the registry entry only if the current entry's `:server-obs` is `identical?` to the captured one. This protects re-registered entries (post-`switch-to!`) from being wiped by stale cancellations and ensures the involuntary-disconnect path produces the same observable disconnect event as the graceful path. The drainer executor is shut down unconditionally — its lifecycle is per-stream.

**Event Routing**:
- **Confirmation events**: Targeted delivery to specific client for acknowledgment
- **Broadcast events**: Delivered to all connected clients for awareness and coordination
- **Event delivery**: Uses the same event streaming infrastructure as piece notifications

### **Connection Count Impact**

The explicit connection architecture affects server connection statistics:
- **Component initialization**: No network connections established
- **After register-with-server**: Clients establish both streaming (1) + API pool (4) = 5 total connections per client
- **Flow control**: Only streaming connections participate in event queue management
- **Lifecycle**: Applications control exactly when these connections are created and destroyed

### **Client Validation Architecture**

The connection establishment includes comprehensive client-id validation as a security gate:

**Validation Rules**:
- **Format**: Client-ids must match pattern `^[a-zA-Z0-9_-]{3,64}$`
- **Length**: 3-64 characters inclusive
- **Characters**: Alphanumeric characters, dashes, and underscores only
- **Uniqueness**: Server-side registry check prevents duplicate registrations

**Authentication Interceptor (RPC-level)**:

The server installs a `ServerInterceptor` that validates `CLIENT_ID_HEADER` for unary RPCs that require identity. It matches both `ExecuteMethod` and `Disconnect`:

- For `ExecuteMethod`: identity is required to authorise the API call; missing/blank/unregistered client-ids are rejected with `UNAUTHENTICATED`.
- For `Disconnect`: identity is required to determine *which* client to disconnect; the same validation applies, ensuring a client can only disconnect itself.
- For `RegisterClient`: exempt from the interceptor — registration is the act of *claiming* the identity, so the client-id comes from the request payload, not from a previously-established context.

When validation passes, the interceptor stores the client-id in the gRPC `Context` (via `CLIENT_ID_CONTEXT_KEY`). Handlers read it via `headers/get-client-id-from-context`. This indirection means handlers never trust payload-level identity claims, closing the attack surface where a client could try to disconnect another client by sending its id in the body.

**Security Enforcement**:
- **Injection Prevention**: Restricted character set prevents protocol injection attacks
- **Resource Protection**: Length limits prevent memory exhaustion via unlimited client names
- **Connection Integrity**: Uniqueness enforcement ensures proper resource management and cleanup

**Error Handling**:
- **Duplicate client-id**: Returns `ALREADY_EXISTS` gRPC status
- **Invalid format/length**: Returns `INVALID_ARGUMENT` gRPC status  
- **Statistics Integration**: Failed validations increment `:clients-disconnected-error`
- **Resource Cleanup**: Failed registrations properly clean up allocated gRPC channels

**Implementation Location**: `ooloi.backend.grpc.server/validate-client-connection` with constants in `ooloi.backend.constants/client-id-pattern`

## Rationale

### **API Request Concurrency: Flow Control Not Needed**

**gRPC Thread-Per-Request Model**:
- Each `executeMethod` call runs in dedicated thread
- Multiple requests from same client processed concurrently server-side
- No head-of-line blocking: slow requests don't block fast requests
- gRPC's built-in thread pool handles request distribution

**STM Transaction Architecture**:
```clojure
;; Each API call automatically wrapped in STM transaction
(dosync 
  (alter piece-ref# 
    (fn [piece#] 
      (vpd/mutate vpd# piece# setter-function))))
```

- **Automatic conflict resolution**: STM handles concurrent modifications optimally
- **Coordinated updates**: Multiple refs updated atomically within single transaction
- **Retry mechanism**: STM automatically retries conflicted transactions
- **Server designed for bombardment**: ADR-0004 STM architecture explicitly built for concurrent load

**Client Multi-Threading Scenarios**:
```clojure
;; Plugin making 100 sequential API calls
(dotimes [i 100]
  (execute-method client "add-note" {:measure i :note "C4"}))
```

- **Per-client ordering preserved**: gRPC connection maintains request sequence
- **Cross-client concurrency**: Different clients' requests processed simultaneously
- **STM coordination**: Conflicting operations handled by automatic retry mechanism

**Conclusion**: Adding flow control to API requests would **interfere with STM's optimized conflict resolution** and provide no benefit since gRPC already prevents blocking.

### **Event Streaming: Flow Control Required**

**Current Synchronous Event Delivery**:
```clojure
;; Blocking delivery - slow client blocks all others
(doseq [[client-id {:keys [observer]}] @registry]
  (.onNext observer event-message))  ; BLOCKS if client is slow
```

**Problem Scenarios**:
- Slow client network connection delays event delivery to all clients
- Client processing delays (UI thread blocked) affect server performance
- One failed client can disrupt entire event system

**Threading Complexity**: Network transport event streaming requires sophisticated threading coordination beyond simple queue management. Unlike in-process transport where StreamObserver operations are simple method calls, network transport requires:

- **HTTP/2 connection context**: StreamObserver objects maintain strict thread affinity with specific Netty event loops
- **Flow control compliance**: Operations must respect `isReady()` status and use `onReady` callbacks for backpressure
- **Context propagation**: gRPC context must be properly maintained across background processing threads

These transport-layer threading constraints necessitate careful architectural patterns to ensure reliable event delivery without violating gRPC's threading contracts.

**Solution Architecture**:
- **Per-client event queues**: Independent delivery buffers
- **Asynchronous delivery**: Backpressure-aware drainer pattern for non-blocking event distribution
- **Backpressure handling**: Queue overflow management with drop-oldest policy
- **Performance monitoring**: Delivery statistics and health metrics

## Decision Details

### **API Request Processing: No Flow Control**

**Preserved behaviors**:
- gRPC thread-per-request model continues unchanged
- STM transaction wrapping occurs per individual API call
- Multi-client concurrent processing at maximum throughput
- Automatic retry mechanism handles all conflict resolution

**Rationale**: Flow control would be **architectural impedance** - solving problems that don't exist while interfering with optimized STM coordination.

### **Event Streaming: Sophisticated Flow Control**

**New behaviors**:
- Per-client bounded event queues (1000-item capacity)
- Backpressure-aware drainer pattern for reliable delivery across transport types
- Drop-oldest policy for queue overflow situations
- Statistical monitoring for client health and performance

**Event routing unchanged**:
- Server events broadcast to all connected clients
- Piece events routed only to subscribed clients
- Subscription management via existing API methods

## Consequences

### Pros:
- **Optimal STM performance**: No interference with conflict resolution mechanisms
- **Maximum API throughput**: gRPC concurrency leveraged fully
- **Robust event delivery**: Slow clients don't affect fast clients
- **Architectural clarity**: Flow control applied only where needed
- **Scalability preservation**: STM's linear core scaling maintained
- **Operational insight**: Event delivery monitoring provides visibility
- **Transport reliability**: Threading patterns work correctly across in-process and network transport

### Cons:
- **Complexity focus shift**: Flow control concentrated in event system only  
- **Memory overhead**: Per-client event queues consume additional memory
- **Implementation complexity**: Backpressure-aware delivery more sophisticated than synchronous approach
- **Threading constraints**: Network transport requires careful threading coordination

## Related Decisions

- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - STM architecture designed for concurrent bombardment
- [ADR-0002: gRPC](0002-gRPC.md) - gRPC transport layer with inherent concurrency
- [ADR-0019: STM-gRPC Batch Transactions](0019-STM-gRPC-Batch-Transactions.md) - Transaction boundary alignment
- [ADR-0022: Lazy Frontend-Backend Architecture](0022-Lazy-Frontend-Backend-Architecture.md) - Event-driven synchronization
- [ADR-0025: Server Statistics Architecture](0025-Server-Statistics-Architecture.md) - Event queue health monitoring and performance metrics
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Frontend event routing architecture relying on FIFO delivery guarantees from per-client drainer threads

## Implementation References

- **[gRPC Server Streaming: Threading and Backpressure in Clojure](../guides/GRPC_STREAMING_THREADING_GUIDE.md)** - Complete implementation patterns for the backpressure-aware drainer architecture, including solutions to network transport threading constraints

## Implementation Notes

**API Request Architecture** (preserved):
```
gRPC Request → ExecuteMethod → handle-execute-method →
resolve API function → STM-wrapped API function → dosync transaction
```
ADR-0046 moved the wire ↔ Clojure conversion into transport-aware marshallers; the
handler receives a native Clojure request map regardless of transport, so the older
`execute-unified-method`/`execute-method` indirection is no longer needed.

**Event Streaming Architecture** (enhanced):
```
Server Event → Per-client Queue → Backpressure-Aware Drainer Pattern → 
Context-Propagated Thread → StreamObserver.onNext → Client Reception
```

**Flow Control Boundary**:
- **Input side**: No flow control - maximum concurrent API throughput
- **Output side**: Sophisticated flow control with transport-aware threading - robust event delivery

This architecture maximizes both API performance (via STM + gRPC concurrency) and event reliability (via queue-based flow control with proper threading patterns) while maintaining clear separation of concerns.
