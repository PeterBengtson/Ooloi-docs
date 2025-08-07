# ADR-0018: API-gRPC Interface Generation and Bidirectional Communication Architecture

## Status

Accepted

## Context

Ooloi's collaborative music notation system requires comprehensive gRPC exposure of its backend API while supporting real-time bidirectional communication between frontend and backend components. This decision builds upon the architectural foundations established in [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md), [ADR-0002: gRPC Communication](0002-gRPC.md), and [ADR-0017: System Architecture](0017-System-Architecture.md).

### Scale Requirements

The `api.clj` namespace currently contains 100+ methods and will contain **hundreds of methods** when Ooloi is complete. All methods follow consistent VPD-based patterns as defined in [ADR-0008: VPDs](0008-VPDs.md), using the signature pattern `(function-name vpd piece-or-id ...other-args)`. Manual gRPC implementation at this scale is architecturally impossible.

### Communication Requirements

Ooloi's collaborative features require sophisticated communication patterns:

1. **Frontend → Backend**: Hundreds of API methods for musical operations, plus parallel commands (saving, async operations)
2. **Backend → Frontend**: Real-time event streaming for collaboration (graphics recomputed, piece changes, state synchronization)
3. **Bidirectional**: Both directions must operate concurrently without blocking

### Cognitive Load Considerations

The `api.clj` namespace supports **dual usage patterns**:
- **Local Backend Usage**: Both VPD forms `(api/add-articulation [:m 0 1 0 0 3] piece-id :staccato)` and direct object forms `(api/add-articulation measure-object :staccato)`
- **Remote Usage**: Only VPD forms make sense across network boundaries (objects don't serialize)

Remote developers need a **single, consistent mental model** based on VPDs, while local developers retain flexibility.

## Decision

We will implement **unified Clojure-aware gRPC architecture with comprehensive bidirectional asynchronous communication** using the following approach:

### 1. Unified Protobuf Schema (Updated 2025)

**Paradigm Shift**: Replace complex API introspection with a **unified Clojure-aware protobuf schema**:
- **Unified OoloiValue message** handles all Clojure data types (ratios, keywords, maps, vectors, sets)
- **Unified OoloiRequest/OoloiResponse** with method name + parameters pattern
- **Simple ExecuteMethod service** replaces hundreds of generated method definitions
- **Perfect type fidelity** preserving Clojure semantics across network boundaries

### 2. Dynamic API Method Resolution

**Runtime Flexibility**: All API methods accessible through unified endpoint:
- **Dynamic function resolution**: `(ns-resolve 'ooloi.backend.api (symbol method-name))`
- **VPD + piece-id patterns** work identically to local API usage
- **Zero code generation overhead** - new API methods immediately available
- **Plugin compatibility** - new plugin methods work automatically

### 3. Bidirectional Asynchronous Architecture

**Comprehensive Communication Patterns**:
- **Generated API Methods**: All VPD-based operations from `api.clj`
- **Event Streaming**: Backend streams piece events to subscribed frontend clients
- **Parallel Command Processing**: Frontend issues concurrent operations without blocking UI
- **Real-time Synchronization**: Multiple clients stay synchronized on shared pieces

### 4. Service Architecture

```protobuf
service OoloiService {
  // Auto-generated from api.clj VPD-based signatures (hundreds when complete)
  rpc AddArticulation(AddArticulationRequest) returns (AddArticulationResponse);
  rpc GetMeasure(GetMeasureRequest) returns (GetMeasureResponse);
  rpc SetTimeSignature(SetTimeSignatureRequest) returns (SetTimeSignatureResponse);
  // ... hundreds more generated methods
  
  // Bidirectional communication infrastructure
  rpc SubscribeToPieceEvents(PieceSubscriptionRequest) returns (stream PieceEvent);
  rpc ExecuteAsyncCommand(AsyncCommandRequest) returns (AsyncCommandResponse);
}
```

## Rationale

### Scale and Maintenance

**Why Unified Schema vs. Generated Schema:**
- **Scale simplicity**: Unified schema handles hundreds of methods without generation
- **Plugin compatibility**: New plugin data types work immediately without schema changes
- **Zero maintenance overhead**: No code generation pipeline to maintain or debug
- **Hot deployment**: New API methods available instantly without build steps

**Why VPD-Only vs. Full Polymorphic Exposure:**
- **Cognitive load reduction**: Remote developers learn one pattern (VPD-based) instead of multiple
- **Architectural consistency**: VPDs are the universal addressing mechanism throughout Ooloi
- **Network compatibility**: Object references don't serialize; VPDs work identically locally and remotely
- **Leverages existing design**: `api.clj` was designed around VPDs as the primary interface

### Communication Architecture

**Why Bidirectional vs. Request-Response Only:**
- **Collaboration requirements**: Multiple clients editing shared pieces need real-time updates
- **User experience**: Frontend operations (MIDI playback, saving) must run parallel to musical edits
- **Event-driven synchronization**: Graphics recomputation, layout changes require pushing updates to all clients
- **Performance**: Eliminates polling overhead for real-time features

**Why Event Streaming vs. Polling:**
- **Efficiency**: Real-time updates without polling overhead
- **Consistency**: All clients receive updates simultaneously
- **Scalability**: Supports multiple concurrent collaborative sessions
- **Responsiveness**: Immediate UI updates when backend state changes

### Integration with Existing Architecture

**Builds on Established Foundations:**
- **VPD addressing** ([ADR-0008](0008-VPDs.md)): Generated gRPC uses VPDs as universal addressing
- **Frontend-Backend separation** ([ADR-0001](0001-Frontend-Backend-Separation.md)): Clear responsibility boundaries maintained
- **gRPC communication** ([ADR-0002](0002-gRPC.md)): Java interop and security patterns established
- **Component architecture** ([ADR-0017](0017-System-Architecture.md)): Integrant lifecycle management for generated services

## Consequences

### Positive

- **Perfect API coverage**: Every `api.clj` function automatically available remotely with zero maintenance
- **Architectural consistency**: Single universal addressing system (VPDs) across local and remote interfaces  
- **Real-time collaboration**: Event streaming enables multiple users editing shared pieces simultaneously
- **Responsive user experience**: Parallel operations prevent UI blocking during backend processing
- **Cognitive load reduction**: Remote developers learn one simple, consistent pattern
- **Future scalability**: Architecture handles hundreds of methods and scales to more complex collaboration scenarios
- **Development efficiency**: New API functions immediately available via gRPC without additional work
- **Hot plugin installation**: Zero-downtime plugin deployment with automatic data type support
- **Eliminated complexity**: No code generation, API introspection, or build-time dependencies

### Negative

- **Type safety trade-off**: Unified schema reduces compile-time type checking compared to generated specific messages
- **Debugging challenges**: Dynamic method resolution and async event flows can be harder to trace
- **Connection management**: Client reconnection, event replay, and subscription management complexity  
- **Performance overhead**: Recursive conversion for complex nested structures

### Mitigations

- **Comprehensive testing**: Round-trip conversion tests ensure perfect type fidelity
- **Clear error messages**: Dynamic method resolution includes actionable error reporting
- **Performance optimization**: Conversion function optimization and caching for frequently used patterns
- **Development tooling**: Runtime introspection tools for debugging dynamic method calls
- **Monitoring and observability**: Built-in metrics for conversion performance and method usage

## Implementation Approach

### 1. Unified Conversion System (Replaces Code Generation)

**Simple Approach Replacing Complex Generation**:
- **Static protobuf schema**: Unified OoloiValue message handles all data types
- **Simple conversion functions**: Deterministic Clojure ↔ Protocol Buffer conversion
- **No code generation**: Eliminates build-time complexity and fragile introspection
- **Runtime method resolution**: Dynamic API method discovery and invocation
- **Perfect round-trip fidelity**: Ratios stay ratios, keywords preserve namespaces

**Core Conversion Implementation**:
```clojure
(defn clj->protobuf-value [obj]
  (cond
    (ratio? obj) {:ratio-val {:numerator (numerator obj) :denominator (denominator obj)}}
    (keyword? obj) {:keyword-val {:namespace (namespace obj) :name (name obj)}}
    (map? obj) {:map-val {:entries (map (fn [[k v]] {:key (clj->protobuf-value k) 
                                                     :value (clj->protobuf-value v)}) obj)}}
    ;; ... handles all Clojure types recursively
    ))
```

### 2. Bidirectional Communication Infrastructure

**Streaming Implementation Details:**

The generated gRPC service implements the complete streaming architecture detailed in [ADR-0002: gRPC Streaming Architecture](0002-gRPC.md#grpc-streaming-architecture):

**Server Streaming for Real-Time Collaboration**:
```protobuf
// Generated service method for piece event streaming
rpc SubscribeToPieceEvents(PieceSubscriptionRequest) returns (stream PieceEvent);
```
- **Event classification**: Musical changes, graphics updates, collaboration events, user presence
- **Piece-based subscriptions**: Clients subscribe to specific pieces or global events
- **Event filtering**: Intelligent filtering to prevent overwhelming clients with irrelevant updates
- **Connection recovery**: Automatic reconnection with event replay for missed updates

**Client Streaming for STM-Composable Batch Operations**:
```protobuf
// Generated service method for STM-composable distributed transactions
rpc ExecuteBatchOperations(stream BatchOperationRequest) returns (BatchOperationResponse);
```

**🎯 Unique Architectural Innovation: STM-gRPC Batch Transaction Composability**

Ooloi's batch processing represents a **unique architectural achievement** in distributed music notation systems - **STM transaction boundaries naturally align with gRPC batch boundaries**, enabling true **distributed transactions for collaborative editing**.

**STM-gRPC Integration Pattern**:
```clojure
;; Client streams operations → Server accumulates → Single dosync wraps all → Atomic result
(defn handle-batch-operations [operation-stream response-observer]
  (let [operations (collect-streamed-operations operation-stream)]
    (try
      (dosync  ; ← STM transaction boundary = gRPC batch boundary
        (doseq [op operations]
          (apply-vpd-operation op)))  ; Multiple VPD operations compose atomically
      (send-success-response response-observer)
      (catch Exception e
        (send-failure-response response-observer e)))))
```

**Collaborative Editing Advantages**:
- **Atomic batch processing**: Multiple VPD-based operations processed as single distributed transaction
- **ACID compliance across network**: Either all operations succeed or all fail, maintaining data consistency
- **Collaborative conflict resolution**: Multiple users' edits can be batched and coordinated atomically
- **Undo/redo optimization**: Streaming operation chains create natural transaction boundaries for state management
- **Import processing**: Large MusicXML/MIDI files streamed and applied atomically in manageable transaction chunks
- **Performance optimization**: Reduced network roundtrips combined with transactional integrity

**Industry Differentiation**: This STM-gRPC batch architecture is **unique among music notation systems** - traditional notation software cannot provide distributed transactional guarantees for collaborative editing operations.

**Bidirectional Streaming for Interactive Collaboration**:
```protobuf
// Generated service method for real-time interactive editing
rpc CollaborateOnPiece(stream CollaborationInput) returns (stream CollaborationOutput);
```
- **Real-time coordination**: Multiple users editing simultaneously with immediate feedback
- **Conflict resolution**: Server-side coordination of simultaneous edits to same elements
- **Interactive feedback**: Immediate visual responses to collaborative actions
- **Session management**: User join/leave events, presence indicators, collaborative cursors

**Streaming Performance Characteristics**:
- **Connection persistence**: Single connection handles both API calls and streaming events
- **Mixed operation support**: Unary API calls and streaming operations over same connection
- **Flow control**: Built-in backpressure prevents overwhelming slow clients during heavy collaboration
- **Resource efficiency**: Shared connection state reduces memory overhead for multiple concurrent operations

**Event Streaming Architecture**:
- **Event sourcing**: All piece changes captured as events for replay and synchronization
- **Client state synchronization**: Event replay ensures clients stay synchronized after disconnection
- **Selective subscription**: Clients subscribe only to relevant events (specific pieces, event types)
- **Event ordering**: Guaranteed ordering within piece context prevents race conditions

**Parallel Command Processing**:
- **Non-blocking operations**: Long-running operations (MIDI generation, complex layouts) don't block UI
- **Progress streaming**: Real-time progress updates for long-running operations
- **Client-side coordination**: MIDI playback, local rendering, background saves run parallel to editing
- **Resource coordination**: Intelligent scheduling prevents resource conflicts during parallel operations

### 3. Integration with Component Architecture

**Integrant Component Integration**:
- **Generated gRPC server**: Integrant component managing generated service lifecycle
- **Event streaming service**: Component handling client subscriptions and event distribution  
- **Client connection management**: Component coordinating frontend gRPC client connections
- **Build into existing system**: Extends current `grpc-server` component with generated methods

## Alternatives Considered

### 1. Manual gRPC Implementation

**Rejected** due to scale impossibility:
- Hundreds of methods make manual implementation unmanageable
- High risk of inconsistencies and missed methods
- Maintenance overhead grows linearly with API size
- Human error in implementation and updates

### 2. Selective API Exposure

**Rejected** due to architectural inconsistency:
- `api.clj` is designed as the complete public interface
- Creates cognitive overhead (which methods are available remotely?)
- Breaks the principle of universal VPD-based addressing
- Manual selection process introduces errors and omissions

### 3. Synchronous-Only Communication

**Rejected** due to collaboration requirements:
- Collaborative editing requires real-time event distribution
- UI responsiveness demands parallel operations
- Polling-based updates are inefficient and create poor user experience
- Backend-driven updates (graphics recomputation) need push capability

### 4. Service Layer Abstraction

**Rejected** due to unnecessary complexity:
- Adds indirection without benefits
- Creates maintenance burden (changes needed in two places)
- Duplicates `api.clj` functionality
- Contradicts thin delegation principle

### 5. Object-Based gRPC Interface

**Rejected** due to serialization constraints:
- Objects don't serialize across network boundaries
- Creates inconsistency between local and remote interfaces
- VPDs are already the universal addressing mechanism
- Would require parallel object management system

## References

### Related ADRs
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Establishes the distributed architecture requiring comprehensive gRPC interface
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Defines the gRPC foundation and Java interop approach this builds upon
- [ADR-0008: VPDs](0008-VPDs.md) - Defines the VPD addressing system that enables universal API interface
- [ADR-0009: Collaboration](0009-Collaboration.md) - Collaboration features requiring real-time bidirectional communication
- [ADR-0017: System Architecture](0017-System-Architecture.md) - Component architecture and production deployment patterns

### Technical Documentation
- [API Guide](../guides/POLYMORPHIC_API_GUIDE.md) - Complete API usage patterns and VPD integration
- [VPD Guide](../guides/VPDs.md) - VPD usage patterns and examples
- [gRPC Tutorial](../research/GRPC_TUTORIAL.md) - Implementation patterns for gRPC integration
- [gRPC Generator Research](../research/GRPC_GENERATOR.md) - Code generation strategies and patterns

## Notes

This decision represents the convergence of several architectural foundations: VPD-based universal addressing, frontend-backend separation, component lifecycle management, and collaborative real-time features. The automated generation approach is not just convenient but architecturally essential given the scale requirements.

The bidirectional communication architecture directly supports Ooloi's collaborative music notation goals while maintaining the cognitive simplicity that makes the system approachable for developers. The event-driven synchronization model ensures that multiple musicians can work on the same piece with real-time coordination.

Implementation should prioritize clear error messages and debugging support for the generated code, as this will be the primary interface for remote development. The code generation pipeline should include comprehensive validation and testing to ensure reliability at the scale required.