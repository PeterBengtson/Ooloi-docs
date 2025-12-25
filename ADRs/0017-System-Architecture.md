# ADR-0017: System Architecture and Component Lifecycle Management

## Status

Accepted (July 2025)

## Context

Ooloi requires a system architecture that can support multiple deployment modes (backend-only, frontend-only, combined) while providing production-ready operational capabilities. The architecture must handle component lifecycle management, configuration-driven deployment, error handling, and operational requirements like health monitoring and graceful shutdown.

Key architectural challenges:
- **Component Dependencies**: gRPC server depends on piece manager; initialization order matters
- **Partial Failure Handling**: If piece manager starts but gRPC server fails, we need cleanup
- **Deployment Flexibility**: Same codebase must support different operational configurations
- **Operational Requirements**: Production deployment needs health monitoring, error classification, and graceful shutdown
- **Configuration Management**: Support for environment variables, CLI arguments, and deployment-specific settings

## Decision

We will use **Integrant for component lifecycle management** combined with **structured application entry points** that provide production-ready operational capabilities.

### Core Decisions

1. **Integrant for Component Management**: All system components (piece-manager, gRPC server, future components) use Integrant's `init-key`/`halt-key!` pattern with declarative dependency injection
2. **Configuration-Driven Architecture**: Deployment modes and component selection determined by configuration data, not code
3. **Structured Error Handling**: Categorized error types with specific exit codes for operational tooling integration
4. **Comprehensive Resource Management**: All components must implement proper resource cleanup with partial failure handling

### Key Architectural Patterns

#### Component Architecture
```clojure
;; Declarative system configuration
{:ooloi.backend.components/piece-manager {}
 :ooloi.backend.components/grpc-server 
 {:piece-manager (ig/ref :ooloi.backend.components/piece-manager)}}

;; Components manage resources, not business logic  
(defmethod ig/init-key :ooloi.backend.components/grpc-server
  [_ {:keys [piece-manager port]}]
  {:server (create-server piece-manager port)
   :health-manager (create-health-manager)})
```

#### Configuration Management
- **Command-line arguments** override **environment variables** override **defaults**
- **Deployment modes**: `backend` (default), `frontend`, `combined`
- **Flexible port/timeout configuration** for different operational environments

#### Error Classification
- **Structured exit codes**: Port conflicts (11), configuration errors (12), resource exhaustion (14)
- **User-friendly error messages** with actionable guidance
- **Partial cleanup handling**: Failed component initialization cleans up successful components

## Rationale

### Why Integrant Over Alternatives

**Integrant vs. Component**: Integrant's data-driven configuration provides more flexibility than Component's protocol-based approach. Configuration as data enables runtime deployment mode selection without code changes.

**Integrant vs. Mount**: Mount's global state management conflicts with functional programming principles and creates testing complexity. Integrant's explicit dependency injection aligns better with Clojure's functional philosophy.

**Integrant vs. Custom Framework**: Integrant solves component lifecycle and dependency injection thoroughly, avoiding the need to reinvent well-established patterns.

### Architectural Benefits

1. **Deployment Flexibility**: Single codebase supports backend-only, frontend-only, and combined deployments through configuration changes
2. **Operational Integration**: Structured exit codes and error messages enable automated deployment tooling and operational monitoring
3. **Resource Safety**: Automatic cleanup of partially initialized systems prevents resource leaks during failure scenarios
4. **Development Productivity**: Components can be developed and tested independently; clear separation between system concerns and business logic
5. **Production Readiness**: Built-in health monitoring, graceful shutdown, and error classification provide operational capabilities from day one

### Key Problem Solutions

**Partial Failure Handling**: Integrant doesn't automatically clean up successful components when later ones fail. Our `start-with-config` wrapper handles this critical gap.

**Configuration Complexity**: Multiple deployment modes with environment variables and CLI arguments require structured precedence handling.

**Operational Requirements**: Production deployment needs more than just "start the application" - requires health monitoring, structured error reporting, and graceful shutdown.

## Consequences

### Positive

- **Operational Integration**: Exit codes and health monitoring enable automated deployment and monitoring
- **Resource Safety**: Automatic cleanup prevents resource leaks during partial initialization failures
- **Deployment Flexibility**: Single codebase supports multiple operational configurations through data-driven configuration
- **Development Productivity**: Clear separation of system concerns from business logic; independent component development and testing
- **Production Readiness**: Built-in error classification, graceful shutdown, and health monitoring

### Negative

- **Learning Curve**: Developers must understand Integrant patterns and component lifecycle concepts
- **Configuration Complexity**: Multiple configuration sources (CLI, environment, defaults) require careful precedence management
- **Testing Requirements**: Component lifecycle and system integration require more sophisticated testing approaches than simple function testing

### Mitigations

- **Clear Documentation**: Integrant concepts explained with concrete examples and established patterns
- **Configuration Validation**: Runtime validation catches configuration errors early with actionable error messages
- **Testing Patterns**: Established mock-based unit testing and integration testing approaches for component lifecycle

## Alternatives Considered

1. **Component (Stuart Sierra's Component)**: Rejected due to less flexible configuration and more complex dependency injection patterns compared to Integrant's data-driven approach.

2. **Mount**: Rejected due to global state management that conflicts with functional programming principles and creates testing complexity.

3. **Custom Application Framework**: Rejected to avoid reinventing well-solved component lifecycle and dependency injection problems.

4. **Simple Application Startup**: Rejected due to insufficient operational capabilities for production deployment (no structured error handling, resource management, or deployment flexibility).

## References

- [ADR-0000: Clojure](0000-Clojure.md) - Language foundation enabling functional system architecture
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Deployment architecture requiring flexible system configuration
- [ADR-0002: gRPC Communication](0002-gRPC.md) - gRPC components managed by this system architecture
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model that remains unaffected by component architecture
- [Integrant Documentation](https://github.com/weavejester/integrant)