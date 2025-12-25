# ADR-0018: API-gRPC Interface and Events

## Table of Contents

- [Status](#status)
- [Context](#context)
- [Decision](#decision)
- [Event Structure and Conventions](#event-structure-and-conventions)
  - [Standard Event Format](#standard-event-format)
  - [Event Type Conventions](#event-type-conventions)
  - [Event Lifecycle and Processing](#event-lifecycle-and-processing)
  - [Event Consistency Requirements](#event-consistency-requirements)
  - [Event Validation Guarantees](#event-validation-guarantees)
  - [Event Scope Fields](#event-scope-fields)
  - [Event Broadcasting API](#event-broadcasting-api)
  - [Testing Event Structure](#testing-event-structure)
- [Implementation Approach](#implementation-approach)
- [Rationale](#rationale)
- [Consequences](#consequences)
- [Alternatives Considered](#alternatives-considered)
- [References](#references)
- [Notes](#notes)

## Status

Accepted

## Context

Ooloi's collaborative music notation system requires comprehensive gRPC exposure of its backend API while supporting real-time event notifications from backend to frontend components. This decision builds upon the architectural foundations established in [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md), [ADR-0002: gRPC Communication](0002-gRPC.md), and [ADR-0017: System Architecture](0017-System-Architecture.md).

### Scale Requirements

The `api.clj` namespace currently contains 100+ methods and will contain **hundreds of methods** when Ooloi is complete. All methods follow consistent VPD-based patterns as defined in [ADR-0008: VPDs](0008-VPDs.md), using the signature pattern `(function-name vpd piece-or-id ...other-args)`. Manual gRPC implementation at this scale is architecturally impossible.

### Communication Requirements

Ooloi's collaborative features require sophisticated communication patterns:

1. **Frontend → Backend**: Hundreds of API methods for musical operations via ExecuteMethod
2. **Backend → Frontend**: Real-time event notifications for collaboration (graphics recomputed, piece changes, state synchronization)

### Cognitive Load Considerations

The `api.clj` namespace supports **dual usage patterns**:
- **Local Backend Usage**: Both VPD forms `(api/add-articulation [:m 0 1 0 0 3] piece-id :staccato)` and direct object forms `(api/add-articulation measure-object :staccato)`
- **Remote Usage**: Only VPD forms make sense across network boundaries (objects don't serialize)

Remote developers need a **single, consistent mental model** based on VPDs, while local developers retain flexibility.

## Decision

We will implement **unified Clojure-aware gRPC architecture with server-to-client event notifications** using the following approach:

### 1. Unified Protobuf Schema (Updated 2025)

**Paradigm Shift**: Replace complex API introspection with a **unified Clojure-aware protobuf schema**:
- **Unified OoloiValue message** handles all Clojure data types (ratios, keywords, maps, vectors, sets)
- **Unified OoloiRequest/OoloiResponse** with method name + parameters pattern
- **Simple ExecuteMethod service** replaces hundreds of generated method definitions
- **Perfect type fidelity** preserving Clojure semantics across network boundaries

### 2. Dynamic API Method Resolution

**Runtime Flexibility**: All API methods accessible through unified endpoint:
- **Dynamic function resolution**: `(ns-resolve 'ooloi.shared.api (symbol method-name))`
- **VPD + piece-id patterns** work identically to local API usage
- **Zero code generation overhead** - new API methods immediately available
- **Plugin compatibility** - new plugin methods work automatically

### 3. Server-to-Client Event Notification Architecture

**Comprehensive Communication Patterns**:
- **Generated API Methods**: All VPD-based operations from `api.clj`
- **Event Streaming**: Backend streams invalidation events to subscribed frontend clients
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
  
  // Server-to-client event notification infrastructure
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

**Why Server-to-Client Event Notifications vs. Request-Response Only:**
- **Collaboration requirements**: Multiple clients editing shared pieces need real-time invalidation updates
- **User experience**: Frontend operations (MIDI playback, saving) must run parallel to musical edits
- **Event-driven synchronization**: Graphics recomputation, layout changes require pushing invalidation notifications to all clients
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

## Event Structure and Conventions

### Standard Event Format

All events in Ooloi's gRPC system follow a consistent structure to ensure predictable handling across client and server components:

```clojure
;; Standard event structure
{:type :event-type-keyword          ; Required: Event type as keyword
 :timestamp 1693827465123456789     ; Required: Nanosecond timestamp (added automatically by system)
 :client-id "client-123"            ; Optional: Originating client identifier
 :message "Human readable message"  ; Optional: User-friendly description
 :piece-id "piece-123"              ; Required for piece events: Target piece identifier
 ...additional-fields...}           ; Event-specific data fields
```

### Event Type Conventions

**Event Type Field** (`:type`):
- **Always a keyword**: `:server-maintenance`, `:client-registration-confirmed`, `:piece-updated`
- **Never a string**: Avoid `"server-maintenance"` or `{:event-type "..."}`
- **Kebab-case naming**: Use dashes for multi-word types (`:client-registration-confirmed`)
- **Hierarchical naming**: Use namespace-style for complex events (`:piece/content-changed`)

**Event Type Taxonomy** (Complete Catalog):

All event types must follow naming pattern enforced by `validate-event-structure`:
- Prefix with `server-`, `client-`, `piece-`, or `collaboration-`
- OR use namespace-style with `/` separator

**Cache Invalidation Events**:
- `:piece-invalidation` - Visual hierarchy invalidation at any level (Layout/PageView/SystemView/StaffView/MeasureView)

**Presence/Collaboration Events**:
- `:collaboration-user-joined` - User joins collaborative session
- `:collaboration-user-left` - User leaves collaborative session
- `:collaboration-cursor-moved` - User cursor position updated
- `:collaboration-selection-changed` - User selection updated

**Playback Events**:
- `:piece-playback-position` - Playback position updated
- `:piece-playback-started` - Playback started
- `:piece-playback-stopped` - Playback stopped

**System Events**:
- `:server-maintenance` - Server maintenance notification
- `:server-shutdown` - Server shutting down
- `:server-status` - Server status update
- `:server-client-connected` - Client connected to server
- `:server-client-disconnected` - Client disconnected from server

**Notification Events**:
- `:piece-validation-error` - Validation error occurred
- `:piece-operation-failed` - Operation failed
- `:server-warning` - Warning message
- `:server-info` - Informational message
- `:client-registration-confirmed` - Client registration successful

### Event Lifecycle and Processing

**Event Creation** (Server-side):
```clojure
;; Server events (broadcast to all clients) - automatic timestamp & event-type derivation
(send-server-event server-component
  {:type :server-maintenance
   :message "System update in 5 minutes"})

;; Piece events (broadcast to subscribed clients) - automatic piece-id extraction
(send-piece-event server-component
  {:type :piece-invalidation
   :piece-id "symphony-123"
   :measures [47 48 49]})

;; Internal event creation (for framework use)
(create-and-queue-event queue observer event-data event-type timestamp source)
```

**Event Transport** (Protocol Buffer):
- Event data serialized to `OoloiValue` protobuf message
- Timestamp and metadata added to `EventMessage` wrapper
- Transport via gRPC streaming to subscribed clients

**Event Reception** (Client-side):
```clojure
;; Client merges EventMessage metadata with event data
(let [event-data (.getEventData event-message)
      clj-data (-> event-data bridge/protobuf-object->internal-map conv/proto->clj)
      timestamp (.getTimestamp event-message)
      complete-event (assoc clj-data :timestamp timestamp)]
  (process-received-event complete-event client-id))
```

**Final Event Structure**: Client applications receive events with all metadata merged:
```clojure
{:type :client-registration-confirmed
 :client-id "test-client-42h"
 :message "Registration successful"
 :timestamp 1693827465123456789}  ; Nanosecond precision added during client processing
```

### Event Consistency Requirements

**Field Naming Standards**:
- Use kebab-case for all field names: `:client-id`, `:piece-id`, `:event-data`
- Prefer keywords over strings for enumerated values
- Use consistent field names across similar event types

**Timestamp Handling**:
- **Server responsibility**: Automatically generated at event creation time using `System/nanoTime()`
- **Transport level**: Timestamp carried in `EventMessage.timestamp`
- **Client processing**: Merge timestamp into final event data
- **Format**: Nanosecond precision timestamp for high precision and drift avoidance

**Event Broadcasting Functions**:
Both `send-server-event` and `send-piece-event` feature simplified signatures with automatic derivation:
- **Perfect signature parity**: `(send-server-event server-component event-data)` and `(send-piece-event server-component event-data)`
- **Automatic timestamp generation**: Nanosecond precision timestamps added internally
- **Automatic event-type derivation**: Protobuf event-type string derived from validated `:type` keyword field
- **Automatic piece-id extraction**: `send-piece-event` extracts `:piece-id` from event data for subscription routing
- **Comprehensive validation**: All events validated against field naming, type requirements, and piece-id presence before broadcast

**Event Content Guidelines**:
- Include human-readable `:message` field for user-facing events
- Use `:client-id` for client-specific events
- Include relevant entity IDs (`:piece-id`, `:user-id`) for context
- Keep event data flat when possible; nest only when semantically grouped

### Event Validation Guarantees

All events emitted by `send-server-event` and `send-piece-event` are validated by `validate-event-structure` before transmission. Clients can rely on the following guarantees:

**Structural Guarantees** (all events):
- Event data is a map (not nil, not a scalar)
- `:type` field exists and is a keyword
- `:type` matches naming pattern: `server-*`, `client-*`, `piece-*`, `collaboration-*`, or contains `/`
- All field names are keywords (kebab-case)
- `:timestamp` field added automatically (nanosecond precision number)

**Type-Specific Guarantees**:
- **Piece events** (sent via `send-piece-event`): `:piece-id` field exists and is a string
- **Collaboration events** (sent via `send-piece-event`): `:piece-id` field exists and is a string

**Optional Field Type Guarantees** (when present):
- `:client-id` is a string if present
- `:message` is a string if present

**What Clients Do NOT Need to Validate**:
- Event structure (guaranteed to be a map)
- Presence of `:type` field (guaranteed)
- Type of `:type` field (guaranteed keyword)
- Naming pattern of `:type` (guaranteed to match validation rules)
- Presence of `:piece-id` for piece-* events (guaranteed)
- Type correctness of optional fields (guaranteed if present)

**Validation Failures**:
Events that fail validation throw `ex-info` with descriptive error data and never reach clients. Clients can trust all received events are structurally valid.

### Event Scope Fields

Certain events include scope fields to specify the extent of their impact:

**Invalidation Scope** (`:piece-invalidation` events):

Visual hierarchy invalidation events use one of these scope patterns:

```clojure
;; Single VPD - invalidates one specific hierarchy element
{:type :piece-invalidation
 :piece-id "symphony-123"
 :vpd [:layouts 0 :page-views 2 :system-views 1 :staff-views 0 :measure-views 47]
 :timestamp ...}

;; Multiple VPDs - invalidates several specific elements
{:type :piece-invalidation
 :piece-id "symphony-123"
 :vpds [[:layouts 0 ... :measure-views 47]
        [:layouts 0 ... :measure-views 48]
        [:layouts 0 ... :measure-views 49]]
 :timestamp ...}

;; Measure numbers - shorthand for common case (clients expand to VPDs)
{:type :piece-invalidation
 :piece-id "symphony-123"
 :measures [47 48 49]
 :timestamp ...}

;; Layout-level - entire layout invalidated
{:type :piece-invalidation
 :piece-id "symphony-123"
 :layout-id "layout-uuid"
 :timestamp ...}
```

**VPD Depth Indicates Hierarchy Level**:
- `[:layouts 0]` → Layout-level invalidation
- `[:layouts 0 :page-views 2]` → PageView-level invalidation
- `[:layouts 0 :page-views 2 :system-views 1]` → SystemView-level invalidation
- `[:layouts 0 :page-views 2 :system-views 1 :staff-views 0]` → StaffView-level invalidation
- `[:layouts 0 ... :measure-views 47]` → MeasureView-level invalidation

**Position Scope** (collaboration and playback events):

```clojure
;; Cursor position
{:type :collaboration-cursor-moved
 :piece-id "symphony-123"
 :vpd [:layouts 0 ... :measure-views 47]
 :user-id "user-456"
 :timestamp ...}

;; Selection (multiple VPDs)
{:type :collaboration-selection-changed
 :piece-id "symphony-123"
 :vpds [[:layouts 0 ... :measure-views 47]
        [:layouts 0 ... :measure-views 48]]
 :user-id "user-456"
 :timestamp ...}

;; Playback position
{:type :piece-playback-position
 :piece-id "symphony-123"
 :vpd [:layouts 0 ... :measure-views 12]
 :timestamp ...}
```

### Event Broadcasting API

The server provides two primary functions for event broadcasting with automatic field derivation:

#### `send-server-event`

**Purpose**: Broadcasts events to all connected clients regardless of piece subscriptions.

**Signature**: `(send-server-event server-component event-data)`

**Parameters**:
- `server-component`: The gRPC server component containing the connection registry
- `event-data`: Map containing event data with required `:type` keyword field

**Automatic Processing**:
- Validates event structure against ADR conventions
- Derives protobuf event-type string from `:type` field
- Generates nanosecond precision timestamp
- Broadcasts to all clients in connection registry

**Example**:
```clojure
(send-server-event server-component 
  {:type :server-maintenance
   :message "System will restart in 5 minutes"
   :affected-services ["api" "streaming"]})
```

#### `send-piece-event`

**Purpose**: Broadcasts events only to clients subscribed to the specified piece.

**Signature**: `(send-piece-event server-component event-data)`

**Parameters**:
- `server-component`: The gRPC server component containing the connection registry  
- `event-data`: Map containing event data with required `:type` and `:piece-id` fields

**Automatic Processing**:
- Validates event structure including required `:piece-id` field
- Derives protobuf event-type string from `:type` field
- Extracts piece-id for subscription routing
- Generates nanosecond precision timestamp
- Broadcasts only to clients subscribed to the specified piece

**Example**:
```clojure
(send-piece-event server-component
  {:type :piece-content-changed
   :piece-id "symphony-123"
   :measures [12 13 14]
   :change-type :notes-added
   :editor-client "user-456"})
```

#### Function Design Principles

**Perfect Signature Parity**: Both functions use identical parameter patterns `(server-component event-data)` for consistency and ease of use.

**Automatic Derivation**: All metadata (timestamps, event-type strings, piece-id extraction) derived automatically from validated event data, eliminating redundant parameters and potential inconsistencies.

**Comprehensive Validation**: Events validated against field naming conventions, required fields, and type constraints before broadcast, ensuring system reliability.

### Testing Event Structure

**Event Validation Pattern**:
```clojure
(fact "Event has correct structure"
  (let [event (receive-test-event)]
    (:type event) => :expected-event-type
    (:timestamp event) => #(and (number? %) (pos? %))
    (:client-id event) => "expected-client-id"
    (:message event) => "Expected message text"))
```

**Event Capture for Testing**:
```clojure
(with-redefs [event-client/process-received-event
              (fn [event client-id]
                (swap! captured-events conj event))]
  ;; Test operations that generate events
  ;; Validate captured events structure and content
  )
```

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
(defn clj->proto [obj]
  (cond
    (ratio? obj) {:ratio-val {:numerator (numerator obj) :denominator (denominator obj)}}
    (keyword? obj) {:keyword-val {:namespace (namespace obj) :name (name obj)}}
    (map? obj) {:map-val {:entries (map (fn [[k v]] {:key (clj->proto k) 
                                                     :value (clj->proto v)}) obj)}}
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
- [ADR-0009: Collaboration](0009-Collaboration.md) - Collaboration features requiring real-time server-to-client event notifications
- [ADR-0017: System Architecture](0017-System-Architecture.md) - Component architecture and production deployment patterns
- [ADR-0024: gRPC Concurrency and Flow Control Architecture](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) - Flow control architecture for event streaming performance
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Frontend event routing and subscription management using the event streaming infrastructure defined here

### Technical Documentation
- [API Guide](../guides/POLYMORPHIC_API_GUIDE.md) - Complete API usage patterns and VPD integration
- [VPD Guide](../guides/VPDs.md) - VPD usage patterns and examples

## Notes

This decision represents the convergence of several architectural foundations: VPD-based universal addressing, frontend-backend separation, component lifecycle management, and collaborative real-time features. The automated generation approach is not just convenient but architecturally essential given the scale requirements.

The server-to-client event notification architecture directly supports Ooloi's collaborative music notation goals while maintaining the cognitive simplicity that makes the system approachable for developers. The event-driven synchronization model ensures that multiple musicians can work on the same piece with real-time coordination.

Implementation should prioritize clear error messages and debugging support for the generated code, as this will be the primary interface for remote development. The code generation pipeline should include comprehensive validation and testing to ensure reliability at the scale required.