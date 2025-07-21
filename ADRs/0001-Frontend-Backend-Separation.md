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

1. **Separation of Concerns**: 
   - The frontend focuses on user interface and interaction.
   - The backend handles core musical logic, data processing, and persistence.

2. **Technology Optimization**:
   - Frontend can leverage JavaFX and Skija for high-performance GUI rendering.
   - Backend can focus on efficient data processing without GUI overhead.

3. **Deployment Flexibility**:
   - Backend can be scaled independently of the frontend.
   - Supports both distributed client-server and standalone deployments.
   - Allows for future cloud-based deployments or desktop-only usage.

4. **Development Workflow**:
   - Frontend and backend components can be developed independently.
   - Easier to test and debug each component in isolation.
   - Shared project provides common code and interface definitions.

5. **Performance**:
   - Reduced memory footprint for specialized deployments.
   - Ability to optimize each component for its specific tasks.

6. **Flexibility**:
   - Potential for multiple frontend clients (desktop, web, mobile) in the future.
   - Easier to replace or upgrade either component independently.
   - Backend-only deployments for API servers or headless processing.

7. **Concurrency**:
   - Backend can handle complex, long-running tasks without affecting UI responsiveness.

8. **Cross-platform Compatibility**:
   - Easier to manage platform-specific issues (e.g., GUI on different OS) in the frontend while keeping the backend consistent.

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
- **gRPC with Java interop** for frontend-backend communication (see ADR-0002)
- **Protocol Buffers** for efficient serialization of musical data structures
- **Bi-directional streaming** for real-time score updates and collaboration

### 2. Component Management
- **Integrant** for lifecycle management in all projects
- **Backend**: gRPC server as Integrant component
- **Frontend**: gRPC client connections as Integrant components
- **Shared**: Component definitions used by combined deployments

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

### 5. State Management
- **Backend**: Authoritative state using STM for concurrency
- **Frontend**: Client-side state synchronized via gRPC calls
- **Real-time Updates**: Streaming gRPC for live collaboration features

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

## Notes

The three-project structure provides maximum flexibility while maintaining clear separation of concerns. The shared project enables both combined and distributed deployments from the same codebase, supporting different user preferences and deployment scenarios.

We will monitor the performance and developer experience of this architecture. The gRPC communication layer provides a clean abstraction that could be replaced if inter-process communication becomes a bottleneck, though current evidence strongly supports this approach for production systems.
