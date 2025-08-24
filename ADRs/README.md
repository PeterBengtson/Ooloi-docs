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
- **[0018-API-gRPC-Interface-Generation](0018-API-gRPC-Interface-Generation.md)**: Automated API-gRPC generation and server-to-client event notification architecture
- **[0019-In-Process-gRPC-Transport-Optimization](0019-In-Process-gRPC-Transport-Optimization.md)**: Performance optimization for combined deployments
- **[0020-TLS-Infrastructure-and-Deployment-Architecture](0020-TLS-Infrastructure-and-Deployment-Architecture.md)**: TLS security infrastructure and certificate management
- **[0021-Authentication](0021-Authentication.md)**: Pluggable authentication architecture with JWT-based security
- **[0022-Lazy-Frontend-Backend-Architecture](0022-Lazy-Frontend-Backend-Architecture.md)**: Lazy frontend-backend architecture with systematic lazy evaluation for collaborative scale
- **[0023-Shared-Model-Contracts](0023-Shared-Model-Contracts.md)**: Unified data models and interfaces across frontend/backend for type fidelity
- **[0024-gRPC-Concurrency-and-Flow-Control-Architecture](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)**: Flow control architecture for event streaming with gRPC concurrency analysis
- **[0025-Server-Statistics-Architecture](0025-Server-Statistics-Architecture.md)**: Comprehensive statistics collection for operational visibility and test validation

## Decisions by Technical Area

### **Core Language & Runtime**
- **[0000-Clojure](0000-Clojure.md)**: Primary language choice with functional programming benefits
- **[0004-STM-for-concurrency](0004-STM-for-concurrency.md)**: Software Transactional Memory for thread-safe operations
- **[0007-Nippy](0007-Nippy.md)**: High-performance Clojure serialization format

### **Client-Server Architecture**
- **[0001-Frontend-Backend-Separation](0001-Frontend-Backend-Separation.md)**: Architectural separation of concerns
- **[0002-gRPC](0002-gRPC.md)**: Primary communication protocol between frontend and backend  
- **[0018-API-gRPC-Interface-Generation](0018-API-gRPC-Interface-Generation.md)**: Automated API generation and event streaming
- **[0019-In-Process-gRPC-Transport-Optimization](0019-In-Process-gRPC-Transport-Optimization.md)**: Performance optimization for combined deployments
- **[0022-Lazy-Frontend-Backend-Architecture](0022-Lazy-Frontend-Backend-Architecture.md)**: Event-driven synchronization and windowing system
- **[0023-Shared-Model-Contracts](0023-Shared-Model-Contracts.md)**: Unified data models across architectural boundaries
- **[0024-gRPC-Concurrency-and-Flow-Control-Architecture](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)**: Flow control and concurrency patterns for event streaming
- **[0025-Server-Statistics-Architecture](0025-Server-Statistics-Architecture.md)**: Comprehensive operational statistics and monitoring

### **User Interface & Rendering**  
- **[0005-JavaFX-and-Skija](0005-JavaFX-and-Skija.md)**: UI framework and high-performance graphics rendering
- **[0006-SMuFL](0006-SMuFL.md)**: Standard Music Font Layout for professional notation rendering
- **[0013-Slur-Formatting](0013-Slur-Formatting.md)**: Advanced curve rendering for musical slurs

### **Data Architecture & Storage**
- **[0008-VPDs](0008-VPDs.md)**: Vector Path Descriptors for hierarchical addressing system
- **[0010-Pure-Trees](0010-Pure-Trees.md)**: Immutable tree structures with cross-references  
- **[0011-Shared-Structure](0011-Shared-Structure.md)**: Memory optimization through data sharing
- **[0012-Persisting-Pieces](0012-Persisting-Pieces.md)**: Musical piece persistence and storage strategy

### **Musical Domain & Collaboration**
- **[0009-Collaboration](0009-Collaboration.md)**: Multi-user editing and conflict resolution approach
- **[0014-Timewalk](0014-Timewalk.md)**: Temporal coordination system for musical element traversal
- **[0015-Undo-and-Redo](0015-Undo-and-Redo.md)**: Undo/redo implementation for musical editing

### **System Infrastructure**
- **[0016-Settings](0016-Settings.md)**: Configuration architecture and user preferences
- **[0017-System-Architecture](0017-System-Architecture.md)**: Component lifecycle and dependency management
- **[0020-TLS-Infrastructure-and-Deployment-Architecture](0020-TLS-Infrastructure-and-Deployment-Architecture.md)**: Security infrastructure and certificate management
- **[0021-Authentication](0021-Authentication.md)**: Pluggable authentication with JWT security

### **Extensibility**
- **[0003-Plugins](0003-Plugins.md)**: Plugin architecture for system extensibility and third-party integration

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
