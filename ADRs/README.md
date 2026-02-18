# Architectural Decision Records

Technical decisions that shaped Ooloi's architecture, organized by architectural progression.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Language & Runtime](#core-language--runtime)
- [Data Architecture & Storage](#data-architecture--storage)
- [Musical Intelligence](#musical-intelligence)
- [System Infrastructure](#system-infrastructure)
- [Client-Server Architecture](#client-server-architecture)
- [Extensibility & Interoperability](#extensibility--interoperability)
- [Backend: Rendering & Layout](#backend-rendering--layout)
- [Frontend: Execution & UI](#frontend-execution--ui)
- [Audio Architecture](#audio-architecture)

## Architecture Overview

These decisions establish Ooloi as a **collaborative music notation system** with:

### Core Technologies
- **Clojure** for functional programming and immutable data structures
- **Integrant** for component lifecycle and dependency management
- **gRPC** for efficient frontend-backend communication
- **JavaFX + Skija** for high-performance UI rendering
- **STM** for thread-safe concurrent operations

### Key Architectural Patterns
- **Pure tree structures** with ID-based cross-references for data integrity
- **Vector Path Descriptors (VPDs)** for hierarchical addressing throughout the system
- **Timewalk coordination** for proper temporal ordering of musical elements
- **Configuration-driven deployment** supporting multiple operational modes
- **Component-based architecture** with proper resource management

### Production Readiness
- **Multi-deployment support**: backend-only, frontend-only, combined, and dev-engine modes
- **Security architecture**: TLS infrastructure with auto-generated certificates and pluggable authentication
- **Operational integration**: structured error handling, health monitoring, graceful shutdown
- **Resource safety**: comprehensive cleanup handling for partial initialization failures
- **Collaboration features**: real-time multi-user editing with conflict resolution

These architectural decisions form the foundation for a **professional-grade music notation system** capable of supporting both individual composers and collaborative musical workflows.

## Core Language & Runtime

Foundational technology choices that enable functional programming and safe concurrency.

- **[0000-Clojure](0000-Clojure.md)**: Choice of Clojure as primary language
- **[0004-STM-for-concurrency](0004-STM-for-concurrency.md)**: Software Transactional Memory for thread safety

## Data Architecture & Storage

Representing and persisting musical data with immutability, efficiency, and hierarchical addressing.

- **[0007-Nippy](0007-Nippy.md)**: Serialization format
- **[0008-VPDs](0008-VPDs.md)**: Vector Path Descriptors for hierarchical addressing
- **[0010-Pure-Trees](0010-Pure-Trees.md)**: Immutable tree structures with ID-based references
- **[0011-Shared-Structure](0011-Shared-Structure.md)**: Data sharing optimizations
- **[0012-Persisting-Pieces](0012-Persisting-Pieces.md)**: Piece persistence strategy
- **[0016-Settings](0016-Settings.md)**: Settings architecture and persistence strategy
- **[0029-Global-Hash-Consing](0029-Global-Hash-Consing.md)**: Global hash-consing for immutable musical objects with Nippy integration

## Musical Intelligence

Defining the semantic rules of music notation independent of visual layout.

- **[0014-Timewalk](0014-Timewalk.md)**: Temporal coordination system for musical traversal
- **[0026-Pitch-Representation-and-Operations](0026-Pitch-Representation-and-Operations.md)**: String-based pitch representation with factory-based transposition system
- **[0033-Time-Signature-Architecture](0033-Time-Signature-Architecture.md)**: Comprehensive time signature system supporting standard, additive, fractional, and irrational meters with string-based notation
- **[0034-Key-Signature-Architecture](0034-Key-Signature-Architecture.md)**: Key signature architecture supporting standard modes, keyless notation, custom accidentals, per-octave variations, and microtonal systems
- **[0035-Remembered-Alterations](0035-Remembered-Alterations.md)**: Temporal accidental tracking system with wave pattern, measure boundary behavior, courtesy accidentals, and house style settings

## System Infrastructure

Component lifecycle, transaction management, and security foundations.

- **[0015-Undo-and-Redo](0015-Undo-and-Redo.md)**: Undo/redo implementation approach
- **[0017-System-Architecture](0017-System-Architecture.md)**: Component lifecycle management and system startup
- **[0020-TLS-Infrastructure-and-Deployment-Architecture](0020-TLS-Infrastructure-and-Deployment-Architecture.md)**: TLS security infrastructure and certificate management

## Client-Server Architecture

Distributing authority and coordinating state between backend and frontend.

- **[0001-Frontend-Backend-Separation](0001-Frontend-Backend-Separation.md)**: Clean separation of concerns
- **[0002-gRPC](0002-gRPC.md)**: Communication protocol between frontend and backend
- **[0009-Collaboration](0009-Collaboration.md)**: Multi-user editing approach
- **[0018-API-gRPC-Interface-and-Events](0018-API-gRPC-Interface-and-Events.md)**: Automated API-gRPC generation and server-to-client event notification architecture
- **[0019-In-Process-gRPC-Transport-Optimization](0019-In-Process-gRPC-Transport-Optimization.md)**: Performance optimization for combined deployments
- **[0021-Authentication](0021-Authentication.md)**: Pluggable authentication architecture with JWT-based security
- **[0022-Lazy-Frontend-Backend-Architecture](0022-Lazy-Frontend-Backend-Architecture.md)**: Lazy frontend-backend architecture with systematic lazy evaluation for collaborative scale
- **[0023-Shared-Model-Contracts](0023-Shared-Model-Contracts.md)**: Unified data models and interfaces across frontend/backend for type fidelity
- **[0024-gRPC-Concurrency-and-Flow-Control-Architecture](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)**: Flow control architecture for event streaming with gRPC concurrency analysis
- **[0025-Server-Statistics-Architecture](0025-Server-Statistics-Architecture.md)**: Comprehensive statistics collection for operational visibility and test validation
- **[0036-Collaborative-Sessions-and-Hybrid-Transport](0036-Collaborative-Sessions-and-Hybrid-Transport.md)**: Hybrid transport architecture enabling dynamic collaboration sessions with role-based permissions and email-based invitations
- **[0040-Single-Authority-State-Model](0040-Single-Authority-State-Model.md)**: Single-authority state model where pieces exist only on server; operations-only API with capability symmetry but authority asymmetry

## Extensibility & Interoperability

Plugin architecture and external format interchange.

- **[0003-Plugins](0003-Plugins.md)**: Plugin architecture for extensibility
- **[0030-MusicXML](0030-MusicXML.md)**: MusicXML interoperability with comprehensive importer/exporter

## Backend: Rendering & Layout

Computing visual layout and engraving decisions from semantic musical data.

- **[0013-Slur-Formatting](0013-Slur-Formatting.md)**: Convex half-hull slur shape with copper plate aesthetics (rounded endpoints, variable thickness); foundational algorithms for point collection and hull calculation
- **[0028-Hierarchical-Rendering-Pipeline](0028-Hierarchical-Rendering-Pipeline.md)**: Five-stage rendering pipeline with plugin-based formatters for professional music engraving
- **[0037-Measure-Distribution-Optimization](0037-Measure-Distribution-Optimization.md)**: Capacity-constrained segmentation with proportional width allocation using Knuth-Plass dynamic programming for exact optimization of measure distribution
- **[0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)**: Backend-authoritative rendering with terminal frontend execution and GPU-accelerated vector rendering

## Frontend: Execution & UI

Executing rendering decisions and handling user interaction.

- **[0005-JavaFX-and-Skija](0005-JavaFX-and-Skija.md)**: UI framework and rendering technology
- **[0006-SMuFL](0006-SMuFL.md)**: Standard Music Font Layout for notation rendering
- **[0031-Frontend-Event-Driven-Architecture](0031-Frontend-Event-Driven-Architecture.md)**: Separate event systems with Event Router for frontend-backend event synchronization
- **[0032-Flow-Mode](0032-Flow-Mode.md)**: Ooloi Flow Mode – stateful modal keyboard input paradigm revival
- **[0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)**: Backend-authoritative rendering with terminal frontend execution and GPU-accelerated vector rendering
- **[0039-Localisation-Architecture](0039-Localisation-Architecture.md)**: Frontend-only localisation with PO files, single translation API, and build-time verification
- **[0042-UI-Specification-Format](0042-UI-Specification-Format.md)**: Pure cljfx maps with namespace-qualified keys for UI specifications; plugin-friendly format with gRPC serialization and window lifecycle management
- **[0043-Frontend-Settings](0043-Frontend-Settings.md)**: Global application settings with atom-based storage, EDN persistence, translated choice labels, and registry-driven Settings dialog

## Audio Architecture

Playback generation delegated to frontend plugins with backend coordination.

- **[0027-Plugin-Based-Audio-Architecture](0027-Plugin-Based-Audio-Architecture.md)**: Frontend plugin-based audio architecture with complete backend-frontend audio separation
- **[0041-Ooloi-Virtual-Instrument-Definition-OVID](0041-Ooloi-Virtual-Instrument-Definition-OVID.md)**: YAML-based virtual instrument definition with canonical technique taxonomy, calibration requirements, and fallback generators
