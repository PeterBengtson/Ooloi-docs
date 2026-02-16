# ADR-0001: Separation of Frontend and Backend into Distinct Clojure Applications

## Status

Accepted (Updated August 2025)

## Context

Ooloi is a complex music notation software that requires both a rich user interface for score editing and manipulation, and powerful backend processing for musical logic, formatting, and data management. We need to decide on the overall architecture of the application, specifically whether to combine frontend and backend into a single Clojure application or separate them into distinct applications.

## Decision

We will implement Ooloi with **separated frontend and backend components** that can be deployed in multiple configurations:

1. **Backend-only** deployment (server mode for dedicated servers)
2. **Combined** deployment (both components in single application with hybrid transport - see ADR-0036)
   - Starts in standalone mode with in-process transport
   - Can dynamically enable network server for collaboration
   - Can connect to remote backends as guest

**Note**: Frontend-only deployment is obsolete. Combined mode with hybrid transport (ADR-0036) provides superior user experience - users can work offline, host collaboration sessions, or connect to remote servers as needed, all from a single deployment mode.

This is achieved through a **three-project structure**: `backend/`, `frontend/`, and `shared/`.

## Rationale

1. **Clear Separation of Concerns**:
   - **Frontend**: User interface, local preferences, client-specific state, visual interactions
   - **Backend**: Musical content, business logic, piece data integrity, concurrent state coordination
   - **Boundary Analysis**: [ADR-0015](0015-Undo-and-Redo.md) undo/redo analysis confirmed no legitimate backend application settings exist

2. **Technology Optimization**:
   - Frontend can leverage JavaFX and Skija for high-performance GUI rendering
   - Backend can focus on efficient musical data processing without GUI overhead
   - STM concurrency model optimized for correctness and deterministic behavior

3. **Deployment Flexibility**:
   - Supports both local and remote server configurations
   - Primary use case: Single-user desktop (99.99% of scenarios)
   - Specialized scenarios: Classroom playback, occasional multi-user editing
   - Backend-only deployments for API servers or headless processing
   - Server location (local vs. remote) is transparent to architecture

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
   - STM-based backend optimized for high-throughput operations (100,000+ operations/second)
   - Frontend optimized for responsive UI without blocking on musical computations

7. **Multi-Client Architecture**:
   - Backend coordinates all musical content changes using STM transactions
   - Frontend clients receive real-time updates via gRPC streaming
   - Architecture naturally supports multiple clients when needed (classroom, teacher-student scenarios)
   - Immutability and STM make multi-client access architecturally straightforward
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
- Builds standalone "Ooloi Server" binary for dedicated server deployments

**Frontend Project** (`frontend/`):
- Implements gRPC client for server communication
- Contains JavaFX/Skija UI implementation
- Generates Java gRPC classes from shared `.proto` definitions
- Code library consumed by the combined application; does not produce a standalone artifact
- Retains `lein run` for development and testing

**Shared Project** (`shared/`):
- Contains `.proto` definitions for gRPC communication
- Houses shared Clojure code accessible to both frontend and backend
- Builds the combined "Ooloi" desktop application with both frontend and backend components
- Serves as the source of truth for API contracts

### Code Sharing Strategy

- **Protocol Definitions**: Located in `shared/src/main/proto/`
- **Shared Model Contracts**: Core data models, interfaces, and traits in `shared/src/main/clojure`
- **Independent Builds**: Each project maintains its own dependencies and can build independently
- **Consistent APIs**: All projects generate Java classes from the same `.proto` files

### Testing Architecture

**Layer-based Testing Strategy**: Tests are distributed by architectural layer rather than mixed within projects:

- **`shared/test/`**: Integration tests requiring both frontend and backend components (gRPC client-server communication, protocol buffer conversion, transport scenarios)
- **`backend/test/`**: Server-side functionality (gRPC server implementation, STM transactions, musical operations)  
- **`frontend/test/`**: Client-side functionality (UI components, user interactions, gRPC client setup)

**Architectural Constraint**: Frontend tests cannot include backend source paths due to dependency conflicts, ensuring clean layer separation and preventing architectural violations.

## Implementation Approach

### 1. Communication Architecture
- **gRPC with Java interop** for frontend-backend communication (see [ADR-0002: gRPC](0002-gRPC.md))
- **Protocol Buffers** for efficient serialization of musical data structures
- **Server-to-client event notifications** for real-time score updates

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

**Combined Deployment** (primary):
- Single process with both frontend and backend components
- Starts with in-process gRPC transport for zero-overhead local operation
- Can dynamically enable network server for collaboration sessions (see [ADR-0036](0036-Collaborative-Sessions-and-Hybrid-Transport.md))
- Built from `shared/` project

**Backend-Only Deployment**:
- Headless server for dedicated server, API access, or integration scenarios
- Built from `backend/` project

### 5. State Management and Responsibility Boundaries

**Backend Responsibilities**:
- **Musical Content**: Authoritative piece data, musical structures, attachments, time signatures
- **Coordination**: STM-based coordination ensuring correctness and deterministic behavior
- **Business Logic**: Musical algorithms, formatting, validation, and processing
- **Real-time Updates**: Streaming gRPC for client synchronization
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
- [ADR-0009: Collaboration](0009-Collaboration.md) - Multi-client collaboration architecture building on frontend-backend separation
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) - Undo/redo architecture leveraging frontend-backend separation boundaries
- [ADR-0016: Settings](0016-Settings.md) - Settings architecture maintaining frontend-backend separation principles
- [ADR-0017: System Architecture](0017-System-Architecture.md) - Component lifecycle management and production-ready operational capabilities
- [ADR-0036: Collaborative Sessions and Hybrid Transport](0036-Collaborative-Sessions-and-Hybrid-Transport.md) - Hybrid transport architecture enabling dynamic collaboration without deployment mode switching
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - Sharpens the authority boundary established by frontend-backend separation

## Notes

The three-project structure provides maximum flexibility while maintaining clear separation of concerns. The shared project enables both combined and distributed deployments from the same codebase, supporting different user preferences and deployment scenarios.

We will monitor the performance and developer experience of this architecture. The gRPC communication layer provides a clean abstraction that could be replaced if inter-process communication becomes a bottleneck, though current evidence strongly supports this approach for production systems.
