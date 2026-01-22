# Architectural Decision Records

Technical decisions that shaped Ooloi's architecture.

## Complete Chronological Index

- **[0000-Clojure](0000-Clojure.md)**: Choice of Clojure as primary language
- **[0001-Frontend-Backend-Separation](0001-Frontend-Backend-Separation.md)**: Clean separation of concerns
- **[0002-gRPC](0002-gRPC.md)**: Communication protocol between frontend and backend
- **[0003-Plugins](0003-Plugins.md)**: Plugin architecture for extensibility
- **[0004-STM-for-concurrency](0004-STM-for-concurrency.md)**: Software Transactional Memory for thread safety
- **[0005-JavaFX-and-Skija](0005-JavaFX-and-Skija.md)**: UI framework and rendering technology
- **[0006-SMuFL](0006-SMuFL.md)**: Standard Music Font Layout for notation rendering
- **[0007-Nippy](0007-Nippy.md)**: Serialization format
- **[0008-VPDs](0008-VPDs.md)**: Vector Path Descriptors for hierarchical addressing
- **[0009-Collaboration](0009-Collaboration.md)**: Multi-user editing approach
- **[0010-Pure-Trees](0010-Pure-Trees.md)**: Immutable tree structures with ID-based references
- **[0011-Shared-Structure](0011-Shared-Structure.md)**: Data sharing optimizations
- **[0012-Persisting-Pieces](0012-Persisting-Pieces.md)**: Piece persistence strategy
- **[0013-Slur-Formatting](0013-Slur-Formatting.md)**: Slur rendering approach
- **[0014-Timewalk](0014-Timewalk.md)**: Temporal coordination system for musical traversal
- **[0015-Undo-and-Redo](0015-Undo-and-Redo.md)**: Undo/redo implementation approach
- **[0016-Settings](0016-Settings.md)**: Settings architecture and persistence strategy
- **[0017-System-Architecture](0017-System-Architecture.md)**: Component lifecycle management and system startup
- **[0018-API-gRPC-Interface-and-Events](0018-API-gRPC-Interface-and-Events.md)**: Automated API-gRPC generation and server-to-client event notification architecture
- **[0019-In-Process-gRPC-Transport-Optimization](0019-In-Process-gRPC-Transport-Optimization.md)**: Performance optimization for combined deployments
- **[0020-TLS-Infrastructure-and-Deployment-Architecture](0020-TLS-Infrastructure-and-Deployment-Architecture.md)**: TLS security infrastructure and certificate management
- **[0021-Authentication](0021-Authentication.md)**: Pluggable authentication architecture with JWT-based security
- **[0022-Lazy-Frontend-Backend-Architecture](0022-Lazy-Frontend-Backend-Architecture.md)**: Lazy frontend-backend architecture with systematic lazy evaluation for collaborative scale
- **[0023-Shared-Model-Contracts](0023-Shared-Model-Contracts.md)**: Unified data models and interfaces across frontend/backend for type fidelity
- **[0024-gRPC-Concurrency-and-Flow-Control-Architecture](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)**: Flow control architecture for event streaming with gRPC concurrency analysis
- **[0025-Server-Statistics-Architecture](0025-Server-Statistics-Architecture.md)**: Comprehensive statistics collection for operational visibility and test validation
- **[0026-Pitch-Representation-and-Operations](0026-Pitch-Representation-and-Operations.md)**: String-based pitch representation with factory-based transposition system
- **[0027-Plugin-Based-Audio-Architecture](0027-Plugin-Based-Audio-Architecture.md)**: Frontend plugin-based audio architecture with complete backend-frontend audio separation
- **[0028-Hierarchical-Rendering-Pipeline](0028-Hierarchical-Rendering-Pipeline.md)**: Five-stage rendering pipeline with plugin-based formatters for professional music engraving
- **[0029-Global-Hash-Consing](0029-Global-Hash-Consing.md)**: Global hash-consing for immutable musical objects with Nippy integration
- **[0030-MusicXML](0030-MusicXML.md)**: MusicXML interoperability excellence with best-in-class importer/exporter
- **[0031-Frontend-Event-Driven-Architecture](0031-Frontend-Event-Driven-Architecture.md)**: Separate event systems with Event Router for frontend-backend event synchronization
- **[0032-Flow-Mode](0032-Flow-Mode.md)**: Ooloi Flow Mode – stateful modal keyboard input paradigm revival
- **[0033-Time-Signature-Architecture](0033-Time-Signature-Architecture.md)**: Comprehensive time signature system supporting standard, additive, fractional, and irrational meters with string-based notation
- **[0034-Key-Signature-Architecture](0034-Key-Signature-Architecture.md)**: Key signature architecture supporting standard modes, keyless notation, custom accidentals, per-octave variations, and microtonal systems
- **[0035-Remembered-Alterations](0035-Remembered-Alterations.md)**: Temporal accidental tracking system with wave pattern, measure boundary behavior, courtesy accidentals, and house style settings
- **[0036-Collaborative-Sessions-and-Hybrid-Transport](0036-Collaborative-Sessions-and-Hybrid-Transport.md)**: Hybrid transport architecture enabling dynamic collaboration sessions with role-based permissions and email-based invitations
- **[0037-Measure-Distribution-Optimization](0037-Measure-Distribution-Optimization.md)**: Capacity-constrained segmentation with proportional width allocation using Knuth-Plass dynamic programming for exact optimization of measure distribution
- **[0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)**: Backend-authoritative rendering with terminal frontend execution and GPU-accelerated vector rendering
- **[0040-Single-Authority-State-Model](0040-Single-Authority-State-Model.md)**: Single-authority state model where pieces exist only on server; operations-only API with capability symmetry but authority asymmetry
- **[0041-Ooloi-Virtual-Instrument-Specification-OVIS](0041-Ooloi-Virtual-Instrument-Specification-OVIS.md)**: YAML-based virtual instrument specification with canonical technique taxonomy, calibration requirements, and fallback generators

## Decisions by Technical Area

| **Technical Area** | **ADRs** | **Key Decisions** |
|-------------------|----------|-------------------|
| **Core Language & Runtime** | [0000](0000-Clojure.md), [0004](0004-STM-for-concurrency.md), [0007](0007-Nippy.md) | Clojure functional programming, STM concurrency, high-performance serialization |
| **Client-Server Architecture** | [0001](0001-Frontend-Backend-Separation.md), [0002](0002-gRPC.md), [0018](0018-API-gRPC-Interface-and-Events.md), [0019](0019-In-Process-gRPC-Transport-Optimization.md), [0022](0022-Lazy-Frontend-Backend-Architecture.md), [0023](0023-Shared-Model-Contracts.md), [0024](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md), [0025](0025-Server-Statistics-Architecture.md), [0031](0031-Frontend-Event-Driven-Architecture.md), [0036](0036-Collaborative-Sessions-and-Hybrid-Transport.md), [0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md), [0040](0040-Single-Authority-State-Model.md) | Clean separation, gRPC communication, event streaming, pull-based synchronization, frontend event routing with Event Router and category aggregation, backend-authoritative rendering with GPU-accelerated terminal frontend execution, single-authority state model with operations-only API, performance optimization, operational monitoring, hybrid transport for dynamic collaboration |
| **User Interface & Rendering** | [0005](0005-JavaFX-and-Skija.md), [0006](0006-SMuFL.md), [0013](0013-Slur-Formatting.md), [0032](0032-Flow-Mode.md) | JavaFX+Skija graphics, professional music fonts, advanced curve rendering, stateful modal keyboard input |
| **Audio Architecture** | [0027](0027-Plugin-Based-Audio-Architecture.md), [0041](0041-Ooloi-Virtual-Instrument-Specification-OVIS.md) | Frontend plugin-based audio processing, complete backend-frontend audio separation, YAML-based virtual instrument specification with canonical taxonomy and calibration |
| **Data Architecture & Storage** | [0008](0008-VPDs.md), [0010](0010-Pure-Trees.md), [0011](0011-Shared-Structure.md), [0012](0012-Persisting-Pieces.md), [0029](0029-Global-Hash-Consing.md) | Hierarchical addressing, immutable trees, memory optimization, persistence, global hash-consing |
| **Musical Domain & Collaboration** | [0009](0009-Collaboration.md), [0014](0014-Timewalk.md), [0015](0015-Undo-and-Redo.md) | Multi-user editing, temporal coordination, undo/redo implementation |
| **Musical Representation** | [0026](0026-Pitch-Representation-and-Operations.md), [0033](0033-Time-Signature-Architecture.md), [0034](0034-Key-Signature-Architecture.md), [0035](0035-Remembered-Alterations.md) | String-based pitch representation, microtonal support, diatonic/chromatic transposition, comprehensive time signature system with fractional and irrational meters, key signature architecture with standard modes and custom accidentals, temporal accidental tracking with house style settings |
| **Rendering & Layout** | [0028](0028-Hierarchical-Rendering-Pipeline.md), [0037](0037-Measure-Distribution-Optimization.md) (see also [0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md) for frontend execution) | Five-stage rendering pipeline with collision detection, vertical reconciliation, symbol preparation, measure distribution optimization using Knuth-Plass dynamic programming, and connecting element adaptation |
| **System Infrastructure** | [0016](0016-Settings.md), [0017](0017-System-Architecture.md), [0020](0020-TLS-Infrastructure-and-Deployment-Architecture.md), [0021](0021-Authentication.md) | Configuration management, component lifecycle, security infrastructure, authentication |
| **Extensibility & Interoperability** | [0003](0003-Plugins.md), [0030](0030-MusicXML.md) | Plugin architecture for third-party integration, MusicXML interoperability excellence |

## Architecture Overview

These decisions establish Ooloi as a **collaborative music notation system** with:

### **Core Technologies**
- **Clojure** for functional programming and immutable data structures
- **Integrant** for component lifecycle and dependency management
- **gRPC** for efficient frontend-backend communication
- **JavaFX + Skija** for high-performance UI rendering
- **STM** for thread-safe concurrent operations

### **Key Architectural Patterns**
- **Pure tree structures** with ID-based cross-references for data integrity
- **Vector Path Descriptors (VPDs)** for hierarchical addressing throughout the system
- **Timewalk coordination** for proper temporal ordering of musical elements
- **Configuration-driven deployment** supporting multiple operational modes
- **Component-based architecture** with proper resource management

### **Production Readiness**
- **Multi-deployment support**: backend-only, frontend-only, combined, and dev-engine modes
- **Security architecture**: TLS infrastructure with auto-generated certificates and pluggable authentication
- **Operational integration**: structured error handling, health monitoring, graceful shutdown
- **Resource safety**: comprehensive cleanup handling for partial initialization failures
- **Collaboration features**: real-time multi-user editing with conflict resolution

These architectural decisions form the foundation for a **professional-grade music notation system** capable of supporting both individual composers and collaborative musical workflows.
