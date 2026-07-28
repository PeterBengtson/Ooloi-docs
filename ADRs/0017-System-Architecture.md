# ADR-0017: System Architecture and Component Lifecycle Management

## Status

Implemented

## Context

Ooloi requires a system architecture that can support multiple deployment modes (backend-only standalone server, combined desktop app bundling both layers) while providing production-ready operational capabilities. There is no frontend-only deployment — the frontend always runs together with an in-process backend in the combined app. The architecture must handle component lifecycle management, configuration-driven deployment, error handling, and operational requirements like health monitoring and graceful shutdown.

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

#### Surfacing Unexpected Runtime Failures

The exit codes above classify failures that prevent the system from *starting*. Once it is running,
a different hazard applies: a failure on a thread nobody is waiting on disappears without trace. A
`cp/future` whose body throws captures the exception and re-throws it only when the result is
dereferenced, and the UI paths never dereference; a `Platform.runLater` runnable has no caller at
all. Neither produces a crash or a freeze — merely an application that silently fails to do what
was asked. Because the failure is *unexpected* by definition, no call site can be relied upon to
catch it, so the coverage lives at the two boundaries where work is dispatched rather than at the
call sites that dispatch it.

- **Off-thread work** — the shared thread pool is a `ScheduledThreadPoolExecutor` that recovers the
  throwable of any completed task in `afterExecute`, taking it from the executor's own argument or,
  where a thrown `cp/future` body deposits it, from the task's future. It hands the throwable to a
  per-pool handler, replaceable via `install-ooloi-error-handler!`.
- **The JavaFX Application Thread** — every renderer is created with an explicit `:error-handler`.
  The cljfx default prints to stderr and re-throws `Error`s, which is not sufficient: a render
  failure would reach no user, and a re-thrown `Error` escapes onto a thread with nothing above it.

**Routing follows the deployment, not the call site.** Where a UI Manager exists — the combined
desktop application — the failure becomes an error notification in the persistent, user-dismissed
tier ([ADR-0043](0043-Frontend-Settings.md) §Error Display). Where none exists — the standalone
backend server — it goes to stderr; a log file is a natural future refinement, and the routing is
therefore a decision the boundary makes rather than a hardcoded call. This gating is the general
rule for all notifications, stated in [ADR-0036](0036-Collaborative-Sessions-and-Hybrid-Transport.md)
§Notification Model.

**The exception detail is shown to the user, deliberately.** The message resolves through `tr` like
every user-facing string ([ADR-0039](0039-Localisation-Architecture.md)), carrying the exception
text as a parameter rather than by concatenation. A user who can see what failed can report it; a
user shown only that "something went wrong" cannot. This is as valuable in the finished application
as during development.

**Rendering a throwable to text must be depth-bounded.** `ExceptionInfo.toString` folds `ex-data`
into the message, and Ooloi's own structures are legitimately cyclic — a piece window's state atom
holds `:*piece-state` pointing back at itself, so that gesture handlers resolve against live
structure after a refetch. Printing such a value without a depth bound recurses until the stack
overflows. The consequence is worse than a long message: a `StackOverflowError` is an `Error`, so it
escapes the very handler meant to report the failure, destroying the original diagnostic and
replacing it with a thousand frames of the printer calling itself. The bound is what makes the net
trustworthy.

**The bound belongs wherever a throwable becomes text — including where the JDK does it on your
behalf.** It is tempting to read the rule as "bound your printers", and that reading is not
sufficient. Recovering the failure is itself enough to trigger the fault: asking a completed task
for its outcome constructs an `ExecutionException`, and `java.lang.Throwable(Throwable cause)` sets
`detailMessage` to `cause.toString()`. The payload is therefore walked inside JDK code, during
recovery, before any handler of the application's own has been reached and with none of its bindings
in scope. A net that bounds only its reporting still overflows on the way in. Both the recovery and
the reporting carry the bounds.

The net is a backstop for the genuinely unexpected. A failure that can be anticipated — an
unreadable file, a rejected write — belongs caught at its source and surfaced as typed data the
application presents deliberately, well before it could arrive here as a raw exception
([ADR-0051](0051-Filesystem-Operations-Real-and-Virtual.md)).

## Rationale

### Why Integrant Over Alternatives

**Integrant vs. Component**: Integrant's data-driven configuration provides more flexibility than Component's protocol-based approach. Configuration as data enables runtime deployment mode selection without code changes.

**Integrant vs. Mount**: Mount's global state management conflicts with functional programming principles and creates testing complexity. Integrant's explicit dependency injection aligns better with Clojure's functional philosophy.

**Integrant vs. Custom Framework**: Integrant solves component lifecycle and dependency injection thoroughly, avoiding the need to reinvent well-established patterns.

### Architectural Benefits

1. **Deployment Flexibility**: Single codebase supports the standalone backend server and the combined desktop app deployments through configuration changes
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
- [Guide: INTEGRANT_COMPONENTS.md](../guides/INTEGRANT_COMPONENTS.md) - Practical guide to wiring new components, the three-project asymmetry, testing macros, and invariants