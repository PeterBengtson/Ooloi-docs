# ADR: gRPC Concurrency and Flow Control Architecture

Decision: Implement flow control only for server-to-client event streaming, not for client-to-server API requests, based on gRPC's inherent concurrency model and STM's conflict resolution capabilities.

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

### **Unified Event-First Connection Pattern**

Ooloi clients use a **unified connection architecture** that automatically registers the client and establishes both event streaming and API connections during client component initialization, with client registration and event streaming established first:

```
Client Connection Lifecycle
┌─────────────────┐                    ┌─────────────────┐
│  Frontend       │                    │   Backend       │
│  Component      │                    │   Server        │
│  Initialization │                    │                 │
└─────────┬───────┘                    └─────────┬───────┘
          │                                      │
          │ 1. Create event stream client        │
          │ ──────────────────────────────────► │
          │    registerClient(client-id)           │
          │                                     │
          │ ◄ ──────────────────────────────── │
          │    Event streaming established       │  
          │    (registered in connection        │
          │     registry for flow control)      │
          │                                     │
          │ 2. Create API connection pool        │
          │ ──────────────────────────────────► │
          │    4 concurrent channels             │
          │                                     │
          │ ◄ ──────────────────────────────── │
          │    API pool ready for requests      │
          │                                     │
          │ ✓ Bidirectional communication       │
          │   established automatically         │
          │                                     │
```

### **Connection Architecture Benefits**

1. **Registration-first ordering**: Client registration with integrated event streaming available immediately for real-time collaboration
2. **Automatic establishment**: No manual connection or registration management required 
3. **Unified client behavior**: All clients follow consistent registration and connection pattern
4. **Flow control integration**: Event streaming immediately benefits from per-client queue architecture

### **Connection Count Impact**

The unified architecture affects server connection statistics:
- **Before**: Clients established API connections only
- **After**: Clients establish both streaming (1) + API pool (4) = 5 total connections per client
- **Flow control**: Only streaming connections participate in event queue management

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

**Solution Architecture** (Tests 38a-38c):
- **Per-client event queues**: Independent delivery buffers (Test 38 - implemented)
- **Asynchronous delivery**: Thread pool for non-blocking event distribution (Test 38a)
- **Backpressure handling**: Queue overflow management (Test 38b)
- **Performance monitoring**: Delivery statistics and health metrics (Test 38c)

## Decision Details

### **API Request Processing: No Flow Control**

**Preserved behaviors**:
- gRPC thread-per-request model continues unchanged
- STM transaction wrapping occurs per individual API call
- Multi-client concurrent processing at maximum throughput
- Automatic retry mechanism handles all conflict resolution

**Rationale**: Flow control would be **architectural impedance** - solving problems that don't exist while interfering with optimized STM coordination.

### **Event Streaming: Sophisticated Flow Control**

**New behaviors** (Test 38+ implementation):
- Per-client bounded event queues (1000-item capacity)
- Asynchronous event delivery via dedicated thread pool
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

### Cons:
- **Complexity focus shift**: Flow control concentrated in event system only  
- **Memory overhead**: Per-client event queues consume additional memory
- **Implementation complexity**: Async event delivery more sophisticated than synchronous

## Related Decisions

- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - STM architecture designed for concurrent bombardment
- [ADR-0002: gRPC](0002-gRPC.md) - gRPC transport layer with inherent concurrency
- [ADR-0019: STM-gRPC Batch Transactions](0019-STM-gRPC-Batch-Transactions.md) - Transaction boundary alignment
- [ADR-0022: Lazy Frontend-Backend Architecture](0022-Lazy-Frontend-Backend-Architecture.md) - Event-driven synchronization
- [ADR-0025: Server Statistics Architecture](0025-Server-Statistics-Architecture.md) - Event queue health monitoring and performance metrics

## Implementation Notes

**API Request Architecture** (preserved):
```
gRPC Request → executeMethod → execute-unified-method → execute-method → 
resolve API function → STM-wrapped API function → dosync transaction
```

**Event Streaming Architecture** (enhanced):
```
Server Event → Per-client Queue → Async Thread Pool → 
StreamObserver.onNext → Client Reception
```

**Flow Control Boundary**:
- **Input side**: No flow control - maximum concurrent API throughput
- **Output side**: Sophisticated flow control - robust event delivery

This architecture maximizes both API performance (via STM + gRPC concurrency) and event reliability (via queue-based flow control) while maintaining clear separation of concerns.