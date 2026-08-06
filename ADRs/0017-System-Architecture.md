# ADR-0017: System Architecture and Component Lifecycle Management

## Status

Implemented

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Core Decisions](#core-decisions)
  - [Key Architectural Patterns](#key-architectural-patterns)
- [Rationale](#rationale)
  - [Why Integrant Over Alternatives](#why-integrant-over-alternatives)
  - [Architectural Benefits](#architectural-benefits)
  - [Key Problem Solutions](#key-problem-solutions)
- [Consequences](#consequences)
  - [Positive](#positive)
  - [Negative](#negative)
  - [Mitigations](#mitigations)
- [Alternatives Considered](#alternatives-considered)
- [References](#references)

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
catch it, so the coverage lives at the boundaries where work is dispatched rather than at the
call sites that dispatch it.

**Every feed arrives at one named function**, which is what makes the routing below a single
auditable decision rather than a rule each dispatch site must remember. It exists only where a UI
Manager does, so the standalone backend is gated out by construction.

Four kinds of thread carry such a net, each covering what the others cannot.

#### Off-thread work: the shared thread pool

The shared thread pool is a `ScheduledThreadPoolExecutor` that recovers the throwable of any
completed task in `afterExecute`, taking it from the executor's own argument or, where a thrown
`cp/future` body deposits it, from the task's future. It hands the throwable to a per-pool handler,
replaceable via `install-ooloi-error-handler!`.

#### Scheduled executors: one-shot tasks take the net, periodic tasks also catch

A scheduled executor is a thread of the same kind, and takes the same net. Its tasks are worse off
than a pool's rather than better: a one-shot `schedule` deposits its throwable in the task's future,
which nothing dereferences, so it reaches not even the stderr that an escaping runnable would
produce. The UI Manager's scheduler — the notification auto-dismiss timers and the debounced
window-geometry writes — is therefore built by the same constructor as the shared pool and handed
the same handler.

**A *periodic* task must additionally catch inside its own body.** `scheduleAtFixedRate` permanently
suppresses every later execution of a task that throws, so the net alone would report the failure
faithfully and leave the schedule dead — the diagnostic arrives and the work never runs again.
Recovering the throwable afterwards cannot undo that; only a `catch` inside the task can. Where the
net suffices the task carries no `catch` of its own, and where it does not the `catch` is the point.

#### The JavaFX Application Thread needs two mechanisms, not one

Each covers precisely what the other cannot.

- **Every renderer is created with an explicit `:error-handler`.** cljfx catches `Throwable`
  around the render and hands it to that handler, so a render failure never escapes to anything
  else — this handler is the only thing that can see one. The cljfx default prints to stderr and
  re-throws `Error`s, which is not sufficient: a render failure would reach no user, and a
  re-thrown `Error` escapes onto a thread with nothing above it.
- **An uncaught-exception handler is attached to the thread itself.** Nothing else on that
  thread passes through a renderer — a `Platform.runLater` body, an event handler, an animation
  callback — so the renderer's handler sees none of it. Each of those leaves its runnable and
  arrives at the thread's uncaught-exception handler, which is nowhere unless one is installed.
  It is attached to the JavaFX thread specifically rather than installed process-wide: that is
  the thread this covers, and other threads have nets of their own. It is released when the UI
  Manager halts, so a stopped component leaves nothing behind on a thread that outlives it.

#### Clojure agents: both `:error-mode` defaults lose the throwable

An agent is a further such thread, and its net is its own `:error-handler`. An action that throws on
an agent reaches none of the mechanisms above: it is not a pool task, not a scheduled task, and not
a JavaFX runnable. What happens instead is decided by the agent's `:error-mode`, and **both defaults
lose the throwable**:

- `:fail`, the default when no handler is supplied, stores the exception and blocks the agent.
  Nothing reads what it stored, and every later `send` throws `Agent is failed, needs restart` at
  its caller — a message naming nothing about what actually broke. Where an agent is sent to on
  every mutation, one failed action is fatal to the session rather than to itself.
- `:continue` **without** an `:error-handler` discards the exception outright. It is not stored,
  `agent-error` returns `nil`, and nothing is retrievable anywhere.

Every agent therefore carries `:error-mode :continue` together with a handler that records. Not
propagating is frequently correct — a best-effort write must not fail the operation it rode in on
— but *not propagating* is not the same as *not recording*, and the two are easy to conflate when
the code reads `:continue` and the reader stops there.

**Enumerating `catch` forms does not find these.** A `catch` is only one of the ways a throwable is
dropped. An agent's error mode, a `future` whose result is never dereferenced, a `submit` whose
`Future` is discarded, an absent uncaught-exception handler — each loses one without a `catch`
appearing anywhere. An audit that greps for `catch` is blind to them by construction, and will
report a clean sweep while they remain.

**The surfacing path must be total.** Nothing on it may raise on the thread that called it.
A boundary that throws destroys the report it was carrying — on the JavaFX thread the toolkit
swallows the failure *and* the original diagnostic with it; on a pool thread it leaves
`afterExecute`, terminating the worker and losing the throwable. Displaying a failure is also the
one thing that cannot be reported by displaying a failure: a throw while showing a notification
would be surfaced as a notification, which would be shown, which would throw. Every step therefore
degrades to stderr rather than raising, and degrades by *reporting* rather than by swallowing — a
silent catch inside the mechanism that exists to end silent failures would be self-defeating.

**What reaches the screen is bounded; what reaches the record is not.** A failure already showing
is not reported again, neither to the user nor to stderr: it is the same fact, and the first report
already carried the full trace. A *distinct* failure beyond the on-screen limit is suppressed only
on screen — a different failure is a different fact, and losing its record is worse than a verbose
log. The limit counts only notifications that wait to be dismissed; those that fade on their own
are neither counted nor capped. Bounding matters because the notification overlay grows from a
screen corner: past its capacity the newest are pushed out of view, where they cannot be dismissed
until those below them have been, so an unbounded stack shows *less*, not more.

**Routing follows the deployment, not the call site.** Where a UI Manager exists — the combined
desktop application — the failure becomes an error notification in the persistent, user-dismissed
tier ([ADR-0043](0043-Frontend-Settings.md) §Error Display). Where none exists — the standalone
backend server — it goes to stderr; a log file is a natural future refinement, and one that may
well arrive as a plugin rather than as built-in machinery — conceivably more than one, for
destinations other than a local file — which is a further reason the routing is a decision the
boundary makes rather than a hardcoded call. This gating is the general
rule for all notifications, stated in [ADR-0036](0036-Collaborative-Sessions-and-Hybrid-Transport.md)
§Notification Model.

**There is no logging framework, and nothing may depend on one.** Ooloi carries no logging
dependency — by decision, not omission. In the combined desktop application the surface *is* the
notification tier, and the frontend acquires no log surface at all. In the standalone backend
server, which has no frontend, the surface is stderr, and writing to a text file may additionally be
made *activatable*; the same option may later be offered for the combined application. Such a file
is an additional sink and never a required one: no code may assume a log exists, and no diagnostic
may be reachable only through one. This keeps a consumer application free of machinery it does not
need, while leaving the server free to acquire a file sink without disturbing the boundary — which
is precisely why the routing is a decision the boundary makes.

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

#### What a Deliberate `catch` May Do

The net above is the backstop. A `catch` written deliberately is the foreground, and it carries an
obligation the backstop cannot discharge on its behalf: the net never sees what a `catch` has
already swallowed.

**A caught throwable has three ordinary destinations. It is rethrown, it becomes a value the caller
acts on, or it is recorded through `log-error`.** Discarding it is a fourth, it is legitimate, and it
is the minority — see *Deliberate suppression* below. What is never legitimate is arriving at the
fourth by default: binding a throwable to `_` and returning is not a decision about the exception,
it is the absence of one, and it reads on review as though a decision had been made. `log-error` is
the single write site for the record — public in `ooloi.shared.components.thread-pool`, and
therefore on every classpath, reachable from backend and frontend alike.

Where a catch already returns typed data the application presents, that value satisfies the second
destination — but only in the arms that name a specific cause. A **catch-all arm names none**, so it
takes the third as well: the type tells the user which category of thing failed, and only the record
can say why. The same applies to a catch that prints the message and drops the throwable. A message
is not a record; it carries neither the trace, nor the cause chain, nor the class, and it bypasses
the single write site any future sink attaches to.

**A catch clause must be no wider than the condition it names.** This is the rule that matters most
in practice, because the failure it prevents is invisible on review. `catch Exception` beneath a
comment reading *"channels may already be closed"* destroys every failure that is not a closed
channel — and the comment is precisely what makes the loss look considered, because one case was
thought about and the form therefore reads as though all of them were. Where the narrow type is
genuinely unknown, the catch stays wide **and records**; it does not stay wide and discard.

**Deliberate suppression.** Discarding a throwable is sometimes right, and the rule above is not a
prohibition on it — it is a prohibition on reaching it without deciding. Two shapes qualify. The
first is a catch whose thrown case *is* the expected outcome: a toolkit already initialised, a
filesystem already open, an interrupt that ends a loop. There the throw is control flow rather than
failure and there is nothing to report. The second is a throwable that carries nothing the record
will not already have — the same failure a caller immediately above is about to report with more
context. That second shape is a judgement, and judgements are allowed; what is not allowed is making
one silently.

**Volume is not a reason, because volume is the boundary's problem and not the call site's.** The
dedupe above already suppresses a failure that is the same fact as one still showing. A catch site
that discards to avoid noise is solving a centrally-solved problem locally, and pays for it by losing
the first occurrence along with the repeats.

**"It is going away anyway" is not a reason either, and it is the tempting one.** A throw while
closing a channel, halting a component or tearing down a pool feels safe to discard: the thing is
being destroyed, so what is there to act on? But a shutdown step that throws is evidence that the
shutdown is not ordered correctly, and the throwable is the only notice of it. Discarding buys
silence at the price of the diagnosis, and the defect stays — leaking a thread, a registry entry or
a port, invisibly, because an exiting JVM has the operating system reclaim what the code failed to
release.

**The correct answer is a shutdown that does not throw, and it is reachable.** The composed
guarantee in [ADR-0036](0036-Collaborative-Sessions-and-Hybrid-Transport.md) §Network Server
Lifecycle is the worked example: a quitting application sees a terminal event from its own halt, and
rather than suppressing it, halt *order* and *identity* compose so the event is declined. Measured
across every deployment shape, quitting produces no failure record at all — which is what makes the
converse trustworthy, that a diagnostic at shutdown means something genuinely went wrong. Suppression
would have bought the same quiet log while destroying that guarantee.

**Tests inherit this, and get it wrong in a way production does not.** A teardown that halts a
component provokes real production behaviour, and a test returning before that behaviour completes
dismantles the system underneath it — producing exactly the shutdown-path exception that then invites
suppression. The fix is the teardown, never the catch. See
[INTEGRANT_COMPONENTS](../guides/INTEGRANT_COMPONENTS.md) §*Teardown that provokes production
behaviour must wait for it to finish*.

**Both are recognised by the same two marks, because nothing else distinguishes deliberate
suppression from an oversight.** The catch names the specific throwable class it expects rather than
reaching for `Exception`, and the body says why the throwable is discarded. An unexplained `_` is
indistinguishable in the source from a case nobody thought about — which is exactly why the burden
of saying so falls on the deliberate case rather than on the reader.

**Naming the class is not the same as matching the condition, and sometimes cannot be.** A throwable
class can be broader than the condition it names, because two quite different causes may share one:
a `FileNotFoundException` may mean *this optional resource is legitimately absent* or *this required
resource is misnamed*, and the class cannot tell them apart. Where that happens, narrowing the clause
is not sufficient and is actively misleading — the catch reads as disciplined while still swallowing
the case that mattered. The discrimination then belongs in the body. If it cannot be made there
either, that is a finding to record and raise, not a corner to round off: a suppression that cannot
be justified precisely is one that has not been justified.

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
