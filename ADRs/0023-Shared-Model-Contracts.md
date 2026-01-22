# ADR-0023: Shared Model Contracts

## Table of Contents

- [Status](#status)
- [Context](#context)
  - [Previous Architecture Limitations](#previous-architecture-limitations)
  - [gRPC Unified Architecture Enables This](#grpc-unified-architecture-enables-this)
- [Decision](#decision)
  - [What Gets Moved to Shared](#what-gets-moved-to-shared)
  - [Current Architecture Distribution](#current-architecture-distribution)
  - [Implementation Approach](#implementation-approach)
- [Rationale](#rationale)
  - [Benefits](#benefits)
  - [Trade-offs and Considerations](#trade-offs-and-considerations)
- [Consequences](#consequences)
  - [Positive](#positive)
  - [Negative](#negative)
  - [Neutral](#neutral)
- [Architecture Details](#architecture-details)
  - [Shared Project Structure](#shared-project-structure)
  - [Unified API Access](#unified-api-access)
- [Multi-Language gRPC Support Impact](#multi-language-grpc-support-impact)
  - [No Impact on Non-Clojure gRPC Consumers](#no-impact-on-non-clojure-grpc-consumers)
  - [gRPC Layer Separation](#grpc-layer-separation)
  - [Multi-Language Support Architecture](#multi-language-support-architecture)
  - [Language-Specific Implementation Examples](#language-specific-implementation-examples)
  - [Benefits by Language Type](#benefits-by-language-type)
  - [Architectural Isolation](#architectural-isolation)
  - [Plugin Ecosystem Implications](#plugin-ecosystem-implications)
- [Summary](#summary)

## Status

Accepted - **Implementation Complete** (August 2025)

## Context

With the implementation of the unified gRPC architecture, we need to enable frontend clients to work with the same data models as the backend. The previous architecture had all model definitions (`defrecord`), interfaces (multimethods), and predicates located in the backend project, creating a barrier to proper client-server round-trip type fidelity.

### Previous Architecture Limitations

1. **Frontend Data Model Duplication**: Frontend had to recreate all `defrecord` definitions to achieve type fidelity
2. **Interface Inconsistency**: Multimethods and predicates were unavailable to frontend code
3. **Plugin Development Complexity**: Plugin authors had to define models twice (frontend + backend)
4. **Maintenance Burden**: Type definitions were scattered and duplicated across projects

### gRPC Unified Architecture Enables This

The unified gRPC implementation with `OoloiValue` provides perfect type fidelity for any Clojure data structure, making shared model definitions both possible and beneficial:

- Round-trip preservation of ratios, keywords, custom records
- Dynamic method resolution works with any shared interface
- Plugin hot installation requires consistent model definitions

## Decision

We will create **Shared Model Contracts** by moving pure data model definitions and interfaces from the backend to the shared project, creating a true contract between frontend and backend.

### What Gets Moved to Shared

**1. Core Interfaces, Predicates, and Traits**
- `models/interfaces.clj` → `shared/src/main/clojure/ooloi/shared/interfaces.clj`
- `models/predicates.clj` → `shared/src/main/clojure/ooloi/shared/predicates.clj` 
- `models/hierarchy.clj` → `shared/src/main/clojure/ooloi/shared/hierarchy.clj`
- `models/traits/` → `shared/src/main/clojure/ooloi/shared/traits/`

**2. All Data Model Records**
- All `defrecord` definitions from `models/musical/` and `models/visual/`
- Move to `shared/src/main/clojure/ooloi/shared/models/`
- Maintain same namespace structure: `ooloi.shared.models.musical.pitch`, etc.

### Current Architecture Distribution

**Shared Project (The Ooloi Engine)**
- All domain models, interfaces, predicates, traits, and hierarchy
- All multimethod definitions and implementations
- Core business logic and musical knowledge
- All trait implementations and musical operations

**Backend Project (Server Wrapper)**
- gRPC server implementation
- STM-based piece management operations
- VPD operations and timewalk functionality
- Piece manager component (Integrant lifecycle coordination)
- Persistence and storage operations
- Network service layer

**Frontend Project (UI Wrapper)**
- JavaFX user interface components
- User interaction handling
- Visual rendering and graphics
- UI-specific state management

### Implementation Approach

The architecture has fundamentally shifted to **shared project as the core engine**:

1. **Shared Project = Ooloi Engine**: Contains all domain knowledge, classes, models, STM operations, VPD system, timewalk, interfaces, predicates, and traits
2. **Backend Project = Server Wrapper**: Thin wrapper providing gRPC server, piece manager component coordination, and persistence layer around the shared engine
3. **Frontend Project = UI Wrapper**: Thin wrapper providing user interface and JavaFX rendering around the shared engine

Both backend and frontend are now essentially lightweight adapters that expose the shared engine through different interfaces (network vs. UI).

## Rationale

### Benefits

**1. True Type Fidelity**
- Frontend creates identical data structures to backend
- No conversion layer needed - unified gRPC handles everything
- Plugin data models work identically client/server

**2. Consistent Interface Contract**
- Same predicates work on frontend and backend
- Multimethod dispatch available to frontend
- Unified validation logic across tiers

**3. Plugin Development Simplification**
- Define models once in shared project
- Work seamlessly across frontend/backend/gRPC
- Hot plugin installation maintains type consistency

**4. Architectural Clarity**
- Clear separation: shared engine vs. interface-specific adapters
- Shared contains all business logic and musical knowledge
- Backend/frontend are thin wrappers for their respective interfaces

### Trade-offs and Considerations

**1. Namespace Changes**
- **Trade-off**: New import paths require developer familiarity with shared model structure
- **Mitigation**: Shared `core.clj` and `api.clj` provide unified access points
- **Benefit**: Clear separation between contracts and implementations

**2. Dependency Management**
- **Trade-off**: Shared project carries responsibility for core engine functionality
- **Architecture**: Shared project contains all business logic and musical operations
- **Guideline**: Backend/frontend are thin adapters around the shared engine

**3. Testing Architecture**
- **Layer-based Testing**: Integration tests requiring both frontend and backend components reside in shared project
- **Contract Validation**: Shared model behavior tested independently from implementation-specific logic
- **Implementation Testing**: Backend and frontend test their specific architectural responsibilities

## Consequences

### Positive

1. **Unified Plugin Ecosystem**: Plugins define models once, work everywhere
2. **Frontend Data Consistency**: No more client/server model mismatches
3. **Development Velocity**: Shared predicates and interfaces speed frontend development
4. **gRPC Architecture Completion**: Enables full potential of unified type system

### Negative

1. **Learning Curve**: Developers must understand shared vs. backend model boundaries
2. **Import Path Changes**: New namespace patterns require familiarity with shared structure
3. **Project Complexity**: Shared project carries critical system contracts

### Neutral

1. **Build Dependencies**: Frontend now depends on shared models (expected)
2. **Testing Distribution**: Model tests move to shared project (appropriate)

## Architecture Details

### Shared Project Structure

```
shared/src/main/clojure/ooloi/shared/
├── models/
│   ├── musical/
│   │   ├── piece.clj          # (defrecord Piece ...)
│   │   ├── pitch.clj          # (defrecord Pitch ...)
│   │   └── ...
│   └── visual/
│       └── layout.clj         # (defrecord Layout ...)
├── interfaces.clj             # Multimethod definitions
├── predicates.clj             # Type predicates  
├── hierarchy.clj              # Type hierarchy
└── traits/
    └── attachment.clj         # Trait definitions
```

### Unified API Access

The shared project provides unified access to all contracts through `core.clj` and `api.clj` that consolidate shared functionality using Potemkin's `import-vars`:

```clojure
;; shared/src/main/clojure/ooloi/shared/core.clj
(ns ooloi.shared.core
  (:require [ooloi.shared.interfaces :as interfaces]
            [ooloi.shared.predicates :as predicates]
            [potemkin :refer [import-vars]]))

;; Unified access point for all shared contracts
(import-vars
  [ooloi.shared.interfaces
   get-duration add-item set-name ...]
  [ooloi.shared.predicates  
   pitch? chord? measure? ...]
  [ooloi.shared.models.musical.piece
   create-piece]
  [ooloi.shared.models.musical.pitch
   create-pitch])
```

This approach provides a single import point while preserving function metadata, docstrings, and identity.

## Multi-Language gRPC Support Impact

### No Impact on Non-Clojure gRPC Consumers

The shared model contracts **only affect Clojure clients**. Non-Clojure clients (Python, WebAssembly, JavaScript) are completely unaffected by this architectural decision.

### gRPC Layer Separation

**Non-Clojure clients** interact **only** with:
1. **The protobuf schema** (`ooloi_service.proto`) - unchanged
2. **The gRPC service interface** - unchanged  
3. **The OoloiValue message format** - unchanged

**Clojure clients** get **additional benefits**:
1. **Same protobuf + gRPC interface** as other languages
2. **Plus shared `defrecord` structures** for native Clojure development
3. **Plus shared predicates** for type checking
4. **Plus shared interfaces** for polymorphic dispatch

### Multi-Language Support Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    gRPC Service                         │
│  ExecuteMethod(OoloiRequest) → OoloiResponse           │
│  ExecuteBatch(stream OoloiRequest) → OoloiResponse     │
└─────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Python    │    │  Clojure    │    │ WebAssembly │
│   Client    │    │   Client    │    │   Client    │
├─────────────┤    ├─────────────┤    ├─────────────┤
│ Protobuf    │    │ Protobuf    │    │ Protobuf    │
│ generated   │    │ +           │    │ generated   │
│ classes     │    │ Shared      │    │ classes     │
│             │    │ Records     │    │             │
│             │    │ Predicates  │    │             │
│             │    │ Interfaces  │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
```

### Language-Specific Implementation Examples

**Python Client (Unchanged)**:
```python
# Python works with generated protobuf classes
request = ooloi_pb2.OoloiRequest()
request.method = "create-pitch"
request.params.map_val.entries.extend([...])  # Standard protobuf

response = stub.ExecuteMethod(request)
```

**Clojure Client (Enhanced)**:
```clojure
;; Clojure gets native data structures + protobuf conversion
(require '[ooloi.shared.models.musical.pitch :refer [create-pitch pitch?]])

(let [pitch (create-pitch :note "C4" :duration 1/4)]  ; ← Native Clojure record
  (when (pitch? pitch)                  ; ← Shared predicate validation
    (grpc-call :create-pitch pitch)))   ; ← Automatic OoloiValue conversion

;; Expressive musical operations using shared API
(-> piece
    (add-attachment [:m 0 0 0 0 0 :items 0] "p")              ; Piano
    (add-attachment [:m 0 0 0 0 0 :items 0] "slur"            ; Slur
                    [:m 0 0 0 0 0 :items 3])
    (add-attachment [:m 0 0 0 0 0 :items 7] "f"))             ; Forte
```

### Benefits by Language Type

**Shared Models Enable (Clojure Only)**:

1. **Type-safe Clojure development** - work with `Pitch` records, not protobuf objects
2. **Client-side validation** - use `pitch?` predicates before sending requests
3. **Plugin consistency** - plugins define models once, work in client + server
4. **Perfect round-trip fidelity** - client creates same structures server expects
5. **Native Clojure semantics** - ratios, keywords, immutable collections

**Non-Clojure Clients Retain Full Capability**:

1. **Complete API access** via ExecuteMethod with any method name
2. **Perfect data fidelity** via OoloiValue conversion preserving all types
3. **Streaming support** via ExecuteBatch for atomic STM transactions
4. **All backend features** without any limitations or performance overhead
5. **Standard protobuf workflow** with generated client code

### Architectural Isolation

The shared model contracts create a **Clojure-specific enhancement layer** that:

- **Sits on top of** the universal gRPC interface
- **Does not replace** the universal gRPC interface
- **Adds value** for Clojure ecosystem without affecting others
- **Maintains compatibility** with all existing and future non-Clojure clients

### Plugin Ecosystem Implications

**For Plugin Authors**:
- **Clojure plugins**: Define models once in shared project, work everywhere
- **Non-Clojure plugins**: Continue using standard protobuf + gRPC patterns
- **Mixed ecosystems**: Clojure and non-Clojure plugins coexist seamlessly

**For Client Developers**:
- **Clojure developers**: Rich native data model integration
- **Python/JS/WASM developers**: Standard gRPC client development patterns
- **API compatibility**: All languages access identical backend functionality

This architecture ensures that enhancing the Clojure development experience does not create barriers or limitations for the broader multi-language ecosystem.

## Summary

This ADR establishes the foundation for a true client-server contract that leverages the unified gRPC architecture's type fidelity capabilities while maintaining clear architectural boundaries and preserving universal multi-language gRPC compatibility.

## See Also

- [ADR-0026: Pitch Representation and Operations](0026-Pitch-Representation-and-Operations.md) - String-based representation pattern for musical data
- [ADR-0033: Time Signature Architecture](0033-Time-Signature-Architecture.md) - TimeSignature model implementing RhythmicItem trait
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - Authority distinction despite shared model access