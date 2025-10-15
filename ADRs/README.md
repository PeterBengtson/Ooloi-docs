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
- **[0029-Global-Hash-Consing](0029-Global-Hash-Consing.md)**: Global hash-consing for immutable musical objects with Nippy integration
- **[0030-MusicXML](0030-MusicXML.md)**: MusicXML interoperability excellence with best-in-class importer/exporter
- **[0031-Frontend-Event-Driven-Architecture](0031-Frontend-Event-Driven-Architecture.md)**: Separate event systems with Event Router for frontend-backend event synchronization

## Decisions by Technical Area

| **Technical Area** | **ADRs** | **Key Decisions** |
|-------------------|----------|-------------------|
| **Core Language & Runtime** | [0000](0000-Clojure.md), [0004](0004-STM-for-concurrency.md), [0007](0007-Nippy.md) | Clojure functional programming, STM concurrency, high-performance serialization |
| **Client-Server Architecture** | [0001](0001-Frontend-Backend-Separation.md), [0002](0002-gRPC.md), [0018](0018-API-gRPC-Interface-and-Events.md), [0019](0019-In-Process-gRPC-Transport-Optimization.md), [0022](0022-Lazy-Frontend-Backend-Architecture.md), [0023](0023-Shared-Model-Contracts.md), [0024](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md), [0025](0025-Server-Statistics-Architecture.md), [0031](0031-Frontend-Event-Driven-Architecture.md) | Clean separation, gRPC communication, event streaming, frontend event routing with category aggregation, performance optimization, operational monitoring |
| **User Interface & Rendering** | [0005](0005-JavaFX-and-Skija.md), [0006](0006-SMuFL.md), [0013](0013-Slur-Formatting.md) | JavaFX+Skija graphics, professional music fonts, advanced curve rendering |
| **Audio Architecture** | [0027](0027-Plugin-Based-Audio-Architecture.md) | Frontend plugin-based audio processing, complete backend-frontend audio separation |
| **Data Architecture & Storage** | [0008](0008-VPDs.md), [0010](0010-Pure-Trees.md), [0011](0011-Shared-Structure.md), [0012](0012-Persisting-Pieces.md), [0029](0029-Global-Hash-Consing.md) | Hierarchical addressing, immutable trees, memory optimization, persistence, global hash-consing |
| **Musical Domain & Collaboration** | [0009](0009-Collaboration.md), [0014](0014-Timewalk.md), [0015](0015-Undo-and-Redo.md) | Multi-user editing, temporal coordination, undo/redo implementation |
| **Musical Representation** | [0026](0026-Pitch-Representation-and-Operations.md) | String-based pitch representation, microtonal support, diatonic/chromatic transposition |
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
