# ADR-0001: Separation of Frontend and Backend into Distinct Clojure Applications

## Status

Accepted (Updated July 2025)

## Context

Ooloi is a complex music notation software that requires both a rich user interface for score editing and manipulation, and powerful backend processing for musical logic, formatting, and data management. We need to decide on the overall architecture of the application, specifically whether to combine frontend and backend into a single Clojure application or separate them into distinct applications.

## Decision

We will implement Ooloi with **separated frontend and backend components** that can be deployed in multiple configurations:

1. **Backend-only** deployment (server mode)
2. **Frontend-only** deployment (client connecting to remote server)
3. **Combined** deployment (both components in single application)

This is achieved through a **three-project structure**: `backend/`, `frontend/`, and `shared/`.

## Rationale

1. **Clear Separation of Concerns**: 
   - **Frontend**: User interface, local preferences, client-specific state, visual interactions
   - **Backend**: Musical content, collaborative coordination, business logic, piece data integrity
   - **Boundary Analysis**: [ADR-0015](0015-Undo-and-Redo.md) undo/redo analysis confirmed no legitimate backend application settings exist

2. **Technology Optimization**:
   - Frontend can leverage JavaFX and Skija for high-performance GUI rendering
   - Backend can focus on efficient musical data processing without GUI overhead
   - STM concurrency model optimized for collaborative musical content coordination

3. **Deployment Flexibility**:
   - Backend can be scaled independently for collaborative editing scenarios
   - Supports both distributed client-server and standalone deployments
   - Allows for future cloud-based deployments or desktop-only usage
   - Backend-only deployments for API servers or headless processing

4. **Development Workflow**:
   - Frontend and backend components can be developed independently
   - Easier to test and debug each component in isolation
   - Shared project provides common code and interface definitions
   - Clear architectural boundaries prevent mixing of concerns

5. **Configuration and State Management**:
   - **User Preferences**: Stored entirely in frontend clients (themes, layouts, workflow settings)
   - **Server Configuration**: Environment variables and deployment config files (not application state)
   - **Piece Settings**: Part of musical content, stored with piece data in backend
   - **No Backend Application Settings**: Analysis shows no legitimate use cases for backend user-configurable settings

6. **Performance and Scalability**:
   - Reduced memory footprint for specialized deployments
   - STM-based backend optimized for high-throughput collaborative scenarios (100,000+ operations/second)
   - Frontend optimized for responsive UI without blocking on musical computations

7. **Collaboration Architecture**:
   - Backend coordinates all musical content changes using STM transactions
   - Frontend clients receive real-time updates via gRPC streaming
   - Clear separation enables multiple clients editing same piece simultaneously
   - UI state remains local while musical changes are coordinated

8. **Future Extensibility**:
   - Architecture supports multiple frontend types (desktop, web, mobile)
   - Backend API can serve different client types without modification
   - Clear boundaries enable independent evolution of each component
   - Plugin architecture can extend both components independently

## Project Structure

### Three-Project Architecture

**Backend Project** (`backend/`):
- Implements gRPC server using Integrant components
- Contains backend-specific business logic
- Generates Java gRPC classes from shared `.proto` definitions
- Builds "Ooloi Server" binary

**Frontend Project** (`frontend/`):
- Implements gRPC client for server communication
- Contains JavaFX/Skija UI implementation
- Generates Java gRPC classes from shared `.proto` definitions  
- Builds "Ooloi Client" binary

**Shared Project** (`shared/`):
- Contains `.proto` definitions for gRPC communication
- Houses shared Clojure code accessible to both frontend and backend
- Builds "Ooloi" combined binary with both components
- Serves as the source of truth for API contracts

### Code Sharing Strategy

- **Protocol Definitions**: Located in `shared/src/main/proto/`
- **Clojure Source Sharing**: Both projects reference `../shared/src/main/clojure`
- **Independent Builds**: Each project maintains its own dependencies and can build independently
- **Consistent APIs**: All projects generate Java classes from the same `.proto` files

## Implementation Approach

### 1. Communication Architecture
- **gRPC with Java interop** for frontend-backend communication (see [ADR-0002: gRPC](0002-gRPC.md))
- **Protocol Buffers** for efficient serialization of musical data structures
- **Bi-directional streaming** for real-time score updates and collaboration

### 2. Component Management
- **Integrant** for lifecycle management in all projects (see [ADR-0017: System Architecture](0017-System-Architecture.md))
- **Backend**: gRPC server as Integrant component with comprehensive lifecycle management
- **Frontend**: gRPC client connections as Integrant components
- **Shared**: Component definitions used by combined deployments
- **Production Ready**: Exit codes, health monitoring, and graceful shutdown capabilities

### 3. Build Configuration
```clojure
;; Each project includes gRPC dependencies:
[io.grpc/grpc-netty "1.60.0"]
[io.grpc/grpc-protobuf "1.60.0"] 
[io.grpc/grpc-stub "1.60.0"]
[com.google.protobuf/protobuf-java "3.25.1"]
[integrant "0.13.0"]

;; Each project generates Java classes from shared .proto files:
:protoc {:proto-paths ["../shared/src/main/proto"]  ; or ["src/main/proto"] for shared
         :output-path "target/generated-sources/grpc"
         :java-options {:language "java"}}
```

### 4. Deployment Models

**Combined Deployment**:
- Single process with both frontend and backend components
- gRPC communication over localhost or in-process
- Built from `shared/` project

**Distributed Deployment**:
- Backend process hosts gRPC server on network port
- Frontend process connects as gRPC client
- Built from separate `backend/` and `frontend/` projects

**Backend-Only Deployment**:
- Headless server for API access or integration
- Built from `backend/` project

### 5. State Management and Responsibility Boundaries

**Backend Responsibilities**:
- **Musical Content**: Authoritative piece data, musical structures, attachments, time signatures
- **Coordination**: STM-based coordination for collaborative editing and conflict resolution
- **Business Logic**: Musical algorithms, formatting, validation, and processing
- **Real-time Updates**: Streaming gRPC for live collaboration features
- **Piece Settings**: Configuration attributes that travel with piece data (as part of musical content)

**Frontend Responsibilities**:
- **UI State**: Client-side state synchronized via gRPC calls from backend musical content
- **User Preferences**: Themes, panel layouts, zoom levels, workflow preferences
- **Local Interactions**: Selection state, editing modes, temporary visual indicators
- **Client Configuration**: Application settings specific to individual user installations

**Explicit Boundary Clarifications** (from [ADR-0015](0015-Undo-and-Redo.md) analysis):
- **Server Deployment Configuration**: Environment variables and config files (NOT backend application state)
- **User Session Preferences**: Frontend client storage (NOT backend responsibility)
- **Musical Algorithm Parameters**: Code constants and piece-embedded settings (NOT user-configurable backend settings)
- **Application Settings Component**: NOT implemented in backend - no legitimate use cases identified

### 6. Error Handling and Recovery
- **gRPC Status Codes**: Structured error communication
- **Retry Logic**: Client-side retry strategies for transient failures
- **Circuit Breakers**: Protection against cascading failures
- **Connection Management**: Automatic reconnection and health checks

## Consequences

### Positive

- **Clear separation** of responsibilities between UI and core logic
- **Improved maintainability** and testability of each component
- **Deployment flexibility** supporting multiple architectural patterns
- **Performance optimization** through component-specific builds
- **Development workflow** enabling independent component development
- **Future expansion** capability for web, mobile, or cloud deployments
- **Code reuse** through shared project structure

### Negative

- **Increased complexity** in application architecture
- **Build coordination** across three projects
- **Network considerations** for distributed deployments
- **Data synchronization** complexity for real-time features
- **Testing complexity** requiring integration testing across components

### Mitigations

- **Comprehensive testing** strategy covering unit, integration, and end-to-end scenarios
- **Development tooling** supporting multi-project workflows
- **Clear documentation** of API contracts and deployment procedures
- **Monitoring and observability** for distributed deployments

## Alternatives Considered

1. **Single Clojure Application**: Rejected due to concerns about mixing UI and core logic, potential performance issues with large scores, and limited deployment flexibility.

2. **Web Frontend with Clojure Backend**: Rejected for the initial implementation due to performance requirements for complex score rendering, but architecture supports future web clients.

3. **Microservices Architecture**: Rejected as overly complex for initial requirements, but the backend component could be further decomposed if needed.

## References

- [ADR-0000: Clojure](0000-Clojure.md) - Foundational language choice enabling this architectural separation
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Details of the communication protocol implementation
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model supporting responsive UI and efficient backend processing
- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) - Frontend GUI framework choice for the separated frontend component
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) - Undo/redo architecture leveraging frontend-backend separation boundaries
- [ADR-0016: Settings](0016-Settings.md) - Settings architecture maintaining frontend-backend separation principles
- [ADR-0017: System Architecture](0017-System-Architecture.md) - Component lifecycle management and production-ready operational capabilities

## Notes

The three-project structure provides maximum flexibility while maintaining clear separation of concerns. The shared project enables both combined and distributed deployments from the same codebase, supporting different user preferences and deployment scenarios.

We will monitor the performance and developer experience of this architecture. The gRPC communication layer provides a clean abstraction that could be replaced if inter-process communication becomes a bottleneck, though current evidence strongly supports this approach for production systems.
