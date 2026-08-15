# Integrant Components in Ooloi

> This guide is the practical companion to [ADR-0017: System Architecture](../ADRs/0017-System-Architecture.md), which records the rationale for using Integrant. Here we record how to work with that decision — how components are defined, wired, and tested across the three-project architecture.
>
> It is written for Ooloi developers who need to add components, understand the startup sequence, or write integration tests. It assumes familiarity with Clojure but not with Integrant.

---

## Table of Contents

1. [What Integrant Does](#1-what-integrant-does)
2. [The Three-Project Structure](#2-the-three-project-structure)
3. [The Component Lifecycle](#3-the-component-lifecycle)
4. [The Backend System](#4-the-backend-system)
5. [The Combined Application System](#5-the-combined-application-system)
6. [Component Design Principles](#6-component-design-principles)
7. [How `ig/build` Differs from `ig/init`](#7-how-igbuild-differs-from-iginit)
8. [Adding a New Component](#8-adding-a-new-component)
9. [Testing Components](#9-testing-components)
10. [Invariants and Pitfalls](#10-invariants-and-pitfalls)
11. [Deprecated Patterns](#11-deprecated-patterns)

---

## 1. What Integrant Does

Ooloi is a long-running desktop application. Components start in a specific order, hold resources (thread pools, gRPC channels, JavaFX stages, agents), and must be shut down cleanly when the application exits or a test completes. Integrant provides two things:

**Dependency injection** — a component declares its dependencies by naming them with `ig/ref`. Integrant resolves the full dependency graph and sorts it topologically before starting anything.

**Lifecycle management** — every component implements `ig/init-key` (start) and optionally `ig/halt-key!` (stop). Integrant calls these in the right order and, on halt, in reverse order.

The result: components know nothing about each other's startup order, and tests get guaranteed cleanup even when they throw.

---

## 2. The Three-Project Structure

The three projects use Integrant differently, and understanding the asymmetry is essential.

### Backend — standalone deployable

The backend has its own `system.clj` and can run as a standalone gRPC server. It starts with `start-with-config`, which calls `ig/init` on the backend component set directly.

```
backend/src/main/clojure/ooloi/backend/system.clj
backend/src/main/clojure/ooloi/backend/components/
  piece_manager.clj
  grpc_server.clj
  http_server.clj
  cache_daemon.clj
  instrument_library.clj
```

### Frontend — components only, not deployed as an application

The frontend defines Integrant components but is **never deployed standalone**. Its components exist to be assembled by the shared combined system in `shared/system.clj` — the only shipped Ooloi desktop product. The frontend project's `project.clj` has no `:main` entry; running `lein run` in `frontend/` does not start an application.

```
frontend/src/main/clojure/ooloi/frontend/components/
  event_bus.clj
  ui_manager.clj
  grpc_clients.clj
  event_router.clj
  fetch_coordinator.clj
```

**`frontend/system.clj` exists as a test harness only.** It assembles the same frontend components the combined system uses (mirror of the frontend portion of `combined-config`) so the frontend project's own test suite can exercise component lifecycle in isolation without booting the full combined application. Production runs go through `shared/system.clj`.

### Shared — the combined application

The `src/app/clojure` source tree (only on the shared project's classpath) contains `ooloi.shared.system` — the combined application entry point. It assembles all backend and all frontend components into a single Integrant system, always using in-process gRPC transport.

```
shared/src/app/clojure/ooloi/shared/system.clj   ← combined-config, start-system!, start-app!
```

This is the **primary product** — what end users download and run.

---

## 3. The Component Lifecycle

Every Integrant component is a Clojure multimethod pair dispatching on a namespaced keyword:

```clojure
;; Start: receives the config map for this component.
;; Returns whatever the running component is — a map, a record, a channel, anything.
(defmethod ig/init-key :ooloi.backend.components/piece-manager [_ {:keys [ui-manager]}]
  {:store  (ref {})
   :status :running})

;; Stop: receives the running component returned by init-key.
;; Returns nil. Releases resources — close channels, await agents, shut down executors.
(defmethod ig/halt-key! :ooloi.backend.components/piece-manager [_ {:keys [store]}]
  ;; Nothing to close for a bare STM ref, but more complex components do work here.
  nil)
```

> This is a simplified illustration of the lifecycle shape. The real piece-manager publishes its store to a **namespace handle** in `ooloi.backend.ops.piece-manager` (via `init-store!` / `release-store!`) rather than returning it on the component value, so that pure-model code can reach the store with no server in scope — see [PIECE_MANAGER_GUIDE §The Core Architecture](PIECE_MANAGER_GUIDE.md#the-core-architecture).

**Dependencies are declared with `ig/ref`:**

```clojure
;; In the config map, ig/ref marks a value as "resolve this to the running component
;; with this key before passing it to init-key."
{:ooloi.backend.components/grpc-server
 {:piece-manager (ig/ref :ooloi.backend.components/piece-manager)
  :transport     :in-process}}
```

When `init-key` for `:grpc-server` is called, `:piece-manager` in its config map is already the live, running piece-manager component — not the config entry.

**Starting and stopping a system:**

```clojure
;; Start all components in dependency order
(def system (ig/init config))

;; Stop all components in reverse dependency order
(ig/halt! system)
```

In practice, Ooloi never calls `ig/init` directly — it uses `ig/build` for the combined app (see §7) and `start-with-config` for the backend. But the mental model is the same.

---

## 4. The Backend System

The standalone backend runs the same backend components as the combined app — it simply omits the frontend layer. It starts ten components:

```
thread-pool          (no dependencies, lives in shared/)
instrument-library   (no dependencies)
backend-undo-manager (no dependencies)
piece-manager        (no dependencies)
connection-registry  ← shared state   (no dependencies)
server-statistics    ← shared state   (no dependencies)
health-manager       ← shared state   (no dependencies)
grpc-server          [piece-manager, instrument-library, backend-undo-manager,
                      connection-registry, server-statistics, health-manager]
http-server          [connection-registry, server-statistics, health-manager]
cache-daemon         [piece-manager]
```

**The ten components:**

| Component | Purpose |
|---|---|
| `thread-pool` | Shared Claypoole thread pool (lives in `shared/`). |
| `instrument-library` | The bundled instrument catalogue (the instrument-library atom; undo-managed). |
| `backend-undo-manager` | Backend coordinated undo/redo manager (ADR-0015 Tier 1). |
| `piece-manager` | Lifecycle management for the piece storage system (STM). |
| `connection-registry` | Shared client connection registry (O(1) lookup). |
| `server-statistics` | Shared server-statistics counters (ADR-0025). |
| `health-manager` | Shared gRPC `HealthStatusManager`. |
| `grpc-server` | The backend's gRPC server (network transport). |
| `http-server` | HTTP health / statistics endpoint (multi-format monitoring). |
| `cache-daemon` | Background cache-optimisation daemon. |

The three shared-state components — `connection-registry`, `server-statistics`, `health-manager` — are their own Integrant components, depended on by `grpc-server` and `http-server` via refs. `http-server` reads health and statistics from those components directly and holds no dependency on `grpc-server`; `cache-daemon` depends only on `piece-manager`. The same wiring holds in the combined app (§5), which adds only the frontend layer and, for splash ordering, a `ui-manager` dependency on each backend component (a `combined-config`-only concern — the standalone backend has no frontend). A component is wired one way; only the *set* of components present differs between the two deployments.

**Entry point:** `ooloi.backend.system/start-with-config`

```clojure
;; Starts the backend system with configuration merged from CLI/env
(def system (ooloi.backend.system/start-with-config {}))

;; Stop
(ooloi.backend.system/stop system)
```

`start-with-config` calls `ig/init` on the backend config after merging CLI arguments and environment variables into the component configs via the injection spec.

---

## 5. The Combined Application System

The combined app's baseline `combined-config` initialises 16 components, plus an on-demand 17th (the network gRPC server) that the application adds at runtime when the host enables collaboration. The baseline divides into four initialisation groups that must run in order:

```
SHARED FOUNDATION
  thread-pool                              (no dependencies)

FRONTEND EARLY  — splash screen must exist before backend reports progress
  event-bus                                [thread-pool]
  ui-manager          ← shows splash       [thread-pool, event-bus]

BACKEND
  instrument-library                       [ui-manager]
  piece-manager                            [ui-manager]
  backend-undo-manager                     [ui-manager]
  connection-registry  ← shared state      [ui-manager]
  server-statistics    ← shared state      [ui-manager]
  health-manager       ← shared state      [ui-manager]
  grpc-server         ← in-process only    [piece-manager, instrument-library,
                                            backend-undo-manager, connection-registry,
                                            server-statistics, health-manager]
  http-server                              [connection-registry, server-statistics,
                                            health-manager]
  cache-daemon                             [piece-manager]

FRONTEND LATE  — connect to backend after it is running
  grpc-clients                             [ui-manager, grpc-server]
  event-router                             [grpc-clients, event-bus]
  fetch-coordinator                        [thread-pool, grpc-clients]
  frontend-undo-manager                    [ui-manager, thread-pool, grpc-clients]
```

**The 16 baseline components, plus the on-demand network server:**

| Component | Layer | Purpose |
|---|---|---|
| `thread-pool` | shared | Shared Claypoole thread pool used across frontend and backend. |
| `event-bus` | frontend | Category-based pub/sub bus for all frontend event delivery. |
| `ui-manager` | frontend | Orchestrates all UI infrastructure — windows, dialogs, notifications, splash, theme. |
| `instrument-library` | backend | The bundled instrument catalogue (the instrument-library atom; undo-managed). |
| `piece-manager` | backend | Lifecycle management for the piece storage system (STM). |
| `backend-undo-manager` | backend | Backend coordinated undo/redo manager (ADR-0015 Tier 1). |
| `connection-registry` | backend | Shared cross-transport client connection registry (O(1) lookup). |
| `server-statistics` | backend | Shared cross-transport server-statistics counters (ADR-0025). |
| `health-manager` | backend | Shared cross-transport gRPC `HealthStatusManager`. |
| `grpc-server` | backend | In-process gRPC server — the combined app's local backend transport. |
| `http-server` | backend | HTTP health / statistics endpoint (multi-format monitoring). |
| `cache-daemon` | backend | Background cache-optimisation daemon. |
| `grpc-clients` | frontend | The frontend's gRPC client(s) to the backend (in-process transport in the combined app). |
| `event-router` | frontend | Routes backend events onto the frontend event bus. |
| `fetch-coordinator` | frontend | Priority-based paintlist fetching via the shared Claypoole pool. |
| `frontend-undo-manager` | frontend | Owns the frontend's undo/redo state: the Tier 2 local stacks, the Tier 1 cache of backend descriptions, and the menu-refresh callback (ADR-0015 Tier 2). Its `grpc-clients` dependency is for halt order — the callback dispatches through `SRV/`, so it must be torn down before the client it dispatches through. |
| `network-grpc-server` *(on-demand)* | backend | Second gRPC transport surface, added at runtime when the host enables collaboration (ADR-0036). |

Backend components' dependency on `ui-manager` is the load-bearing design in `combined-config`: it forces them to start *after* the UI manager (and thus after the splash screen is showing and i18n is loaded). Without it, a backend component might init before i18n is ready and crash when it calls `tr`. This dependency exists **only** in `combined-config` — in the standalone backend config (§4), the same components have no frontend dependencies. The shared-state components (`connection-registry`, `server-statistics`, `health-manager`) carry the dependency uniformly with every other backend component even though their `init-key` does no `tr`-touching work; the rule is uniform precisely so the topological invariant is not contingent on what each component happens to do today.

**Shared backend state has its own components.** `connection-registry` (the single map of registered clients), `server-statistics` (the single set of counters reported via the HTTP statistics endpoint), and `health-manager` (the gRPC `HealthStatusManager`) live in their own Integrant components, depended on by both gRPC servers and by `http-server` via refs. In `combined-config`, `http-server` holds **no** ref to `grpc-server` — it reads the three shared components directly (see §6 "Worked example: extracting shared state from grpc-server"). This is the spec ADR-0036 §Hybrid Transport Architecture commits to: shared state with lifecycle is its own component, depended on by every consumer; no server owns it. The on-demand network gRPC server documented below depends on the same refs, which is what guarantees broadcasts triggered on either transport reach all registered clients and statistics counters report system totality across all transports.

**The complete dependency graph as declared in `combined-config`:**

```clojure
(defn combined-config []
  {;; Shared
   :ooloi.shared.components/thread-pool {}

   ;; Frontend early
   :ooloi.frontend.components/event-bus
   {:thread-pool (ig/ref :ooloi.shared.components/thread-pool)}

   :ooloi.frontend.components/ui-manager
   {:thread-pool (ig/ref :ooloi.shared.components/thread-pool)
    :event-bus   (ig/ref :ooloi.frontend.components/event-bus)}

   ;; Backend
   :ooloi.backend.components/instrument-library
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/piece-manager
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/backend-undo-manager
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   ;; Shared backend state — see ADR-0036 §Hybrid Transport Architecture.
   ;; Depended on by every transport surface so broadcasts and statistics counters
   ;; are single sources of truth regardless of which transport a client joined through.
   :ooloi.backend.components/connection-registry
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/server-statistics
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/health-manager
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/grpc-server
   {:piece-manager                (ig/ref :ooloi.backend.components/piece-manager)
    :instrument-library-component (ig/ref :ooloi.backend.components/instrument-library)
    :undo-manager-component       (ig/ref :ooloi.backend.components/backend-undo-manager)
    :connection-registry          (ig/ref :ooloi.backend.components/connection-registry)
    :server-statistics            (ig/ref :ooloi.backend.components/server-statistics)
    :health-manager               (ig/ref :ooloi.backend.components/health-manager)
    :transport                    :in-process}

   :ooloi.backend.components/http-server
   {:connection-registry (ig/ref :ooloi.backend.components/connection-registry)
    :server-statistics   (ig/ref :ooloi.backend.components/server-statistics)
    :health-manager      (ig/ref :ooloi.backend.components/health-manager)}

   :ooloi.backend.components/cache-daemon
   {:piece-manager (ig/ref :ooloi.backend.components/piece-manager)}

   ;; Frontend late
   :ooloi.frontend.components/grpc-clients
   {:transport   :in-process
    :grpc-server (ig/ref :ooloi.backend.components/grpc-server)
    :ui-manager  (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.frontend.components.event-router/event-router
   {:grpc-clients (ig/ref :ooloi.frontend.components/grpc-clients)
    :event-bus    (ig/ref :ooloi.frontend.components/event-bus)}

   :ooloi.frontend.components.fetch-coordinator/fetch-coordinator
   {:thread-pool  (ig/ref :ooloi.shared.components/thread-pool)
    :grpc-clients (ig/ref :ooloi.frontend.components/grpc-clients)}})
```

### Dynamic (On-Demand) Components

Some components are not part of `combined-config` and therefore do not start with the application. They are added to the running Integrant system through an application-level API, then removed when no longer needed. The component itself is a full Integrant component — it implements `ig/init-key` and `ig/halt-key!` and complies with the `:status :running` invariant — but its lifecycle is driven by application logic rather than the system bootstrap.

**`:ooloi.backend.components/network-grpc-server`** — second gRPC transport surface, started when the host enables a collaboration session and stopped on manual termination or after a configurable grace period of no connected guests (ADR-0036 §Hybrid Transport Architecture). It declares the same shared-state dependencies as the in-process `grpc-server` — `piece-manager`, `instrument-library`, `backend-undo-manager`, `connection-registry`, `server-statistics`, `health-manager` — and points its config to the **same refs**. The in-process server is unaffected by the network server's lifecycle.

The pattern for dynamic components:

- Their config keys live alongside the static `combined-config` keys; the `init-key` and `halt-key!` methods are normal Integrant methods.
- The application's backend API adds the component to the running system map by calling `ig/init-key` directly, threading the live refs from the running system into the new component's config.
- **It also registers the component in the system's `::origin` metadata**, declaring its dependencies there as `ig/ref`s rather than as the live components threaded into its config. This is what allows a system-wide `ig/halt!` to reach it, and what orders it correctly against the state it depends on.
- `ig/halt-key!` removes the component when its lifecycle ends; the next reference to the system map omits the entry.
- The `:status :running` invariant applies while the component is running; the system-health functions (§10) walk every key in the system map uniformly — they need no special handling for dynamic components.
- The §30 conformance test enforces the invariant on whatever is present in the running system: components added dynamically during the test must comply too.

**The `::origin` registration is the step that is easy to omit, and omitting it fails silently.** `ig/halt!` does not walk the system map. It walks the `::origin` config in the map's metadata, filtered by the map's keys — so a component that was only `assoc`ed in is not part of what halt considers. Nothing raises; `halt-key!` is simply never called, and whatever the component holds — a bound port, an open channel, a running thread — it keeps holding.

Two properties make this worth stating explicitly rather than leaving to be discovered. It is **invisible in production**, because an exiting JVM has the operating system reclaim what the component failed to release, so the application appears to shut down cleanly. And it is **loud in a test suite**, where no process exits between namespaces: the resource survives into everything that follows, and the failure surfaces far from its cause, as a later namespace unable to acquire something an earlier one never let go of.

Declare the dependencies in that metadata entry as `ig/ref`s, not as the live components passed to `ig/init-key`. Integrant reads `::origin` only to compute halt order, and refs are what create the edges — without them the component has no declared dependencies and may be halted after the state it uses.

The same pattern can host future dynamic components — a future HTTP REST gateway, a future MIDI listener, anything that is not part of every application run but follows the Integrant lifecycle when it does run.

---

## 6. Component Design Principles

Integrant gives you the tools to express dependencies cleanly. The framework doesn't enforce good design — it makes good design *possible*. Three principles guide how Ooloi uses Integrant.

### 1. A component's value is its public API

What `init-key` returns is what every dependent will see. If you put stuff on that map, you're publishing it. Consumers can — and will — read whatever you put there.

This means: be deliberate about what you return. The component value should expose what the component is, not its bag of internals.

### 2. The dependency graph belongs in the config, not in `init-key` bodies

If component A needs X, A declares `(ig/ref :X)` in its config. The wiring is then visible by reading the config map.

The antipattern: component A declares `(ig/ref :B)` but only because B happens to hold X on its component value. A then drills `(:X b)` inside its `init-key`. The config says "A depends on B"; the reality is "A depends on X". The config lies.

When `init-key` bodies reach into sibling components' fields for unrelated state, the dependency graph stops matching the config. The fix: extract X into its own Integrant component and have A declare a ref on X directly.

### 3. The right size for a component is "owns one concern"

Components should be small. If a single `init-key` produces a map carrying:
- A network resource (server, channel, port binding),
- Shared mutable state (atoms, counter maps),
- Lifecycle metadata (started-at, status), and
- A service registry (health-manager, connection tracker)

…then it's a god component. Each of those is a separable concern with its own lifecycle and consumers. The fact that they all happen to start at the same time, in the same `init-key`, is incidental.

A focused component has:
- One clearly stated responsibility
- A small surface area on the returned value
- Obvious init and halt semantics
- Exactly the dependencies it needs, declared in the config

### Worked example: extracting shared state from `grpc-server`

`grpc-server`'s `init-key` originally created and returned:
- `:server` (the gRPC server instance)
- `:health-manager` (gRPC HealthStatusManager)
- `:connection-registry` (atom of client connections)
- `:server-statistics` (LongAdder counter map)
- `:started-at-ns` / `:started-at-ms` (uptime)
- `:status`

`http-server` then drilled into all of these to build its handlers. Two architectural problems:

1. **Dual-server impossibility.** Adding a second gRPC server (Phase 1 network server) made every drilled field ambiguous: which server's health-manager? which startup time? Their values are now contested between two consumers.
2. **Coupling that wasn't in the config.** `http-server` declared a single `(ig/ref :grpc-server)` dependency, but actually consumed four pieces of state from inside it. The config understated the coupling by a factor of four.

The fix:

- Extract `connection-registry`, `server-statistics`, and `health-manager` into their own tiny Integrant components.
- `grpc-server` consumes them via `ig/ref` instead of constructing them internally.
- `http-server` drops its `:grpc-server` ref entirely and declares direct refs on the three shared components.

After the refactor:
- The config visibly shows `http-server → {connection-registry, server-statistics, health-manager}`. No more hidden coupling.
- Two gRPC servers can coexist, both depending on the shared state, with no ambiguity about ownership.
- Halting one gRPC server doesn't touch `http-server` — they share state, but not lifecycle.

This is "extract a component" applied to the same code three times. The result is what Integrant was built to produce.

### Sharing state across components with independent lifecycles

Extracting shared state is half the job. The other half is making sure each consumer's `halt-key!` cleans up *its own* contribution to the shared state without touching contributions belonging to other consumers that are still running.

`connection-registry` is the canonical Ooloi example. Two gRPC servers (in-process and on-demand network) consume the same registry atom. When the network server halts, the obvious-looking `halt-key!` — iterate `@registry`, close every observer, `(reset! registry {})` — silently breaks the lifecycle-independence guarantee: every in-process client's streaming observer also closes, and the in-process client's `:api-connection-pool` atom is nilled by the resulting `onCompleted` callback. The test passes (the registry is empty as the halting server expected) but the system is broken.

The pattern that fixes this: **stamp each contribution with the contributing component's identity, and have `halt-key!` filter by that identity.**

For `grpc-server` specifically:

- `init-key` generates a fresh `:server-id` (UUID-based) per server instance.
- The registry entry created for each client is stamped with the registering server's `:server-id`.
- `halt-key!` filters the shared registry by its own `:server-id` — closes only its own clients' observers, stops only its own clients' drainer executors, `dissoc`s only its own keys from the registry atom. Other servers' entries are left intact.

The same shape applies whenever multiple lifecycle-independent components write to a single shared atom: a per-writer identifier in each entry, filtered cleanup. The alternative — every consumer trying to remember which entries it owns in its own separate side-list — duplicates state and goes out of sync. The stamp lives with the entry; the entry knows who owns it.

(The lifecycle-independence property is asserted for the dual-server case in both directions — halting one server, and starting one alongside another.)

### A halted component stays halted: `halt-key!`'s return value cannot tell anyone

`halt-key!` conventionally returns the component with its status flipped:

```clojure
(assoc component :status :stopped)
```

**Integrant stores that map, and nothing else ever sees it.** Every closure created while the
component was running — a gRPC response observer, a callback, a `revert-fn` — captured the map
`init-key` returned, and that map is immutable. Its `:status` says `:running`, and always will.
The two are separate values from the moment `halt-key!` returns.

The consequence is easy to miss because it is invisible in the common case: a component whose only
callers are Integrant itself is unaffected. It bites where **something outside the lifecycle holds
the component and can act on it later**. Ooloi's case is `grpc-clients`: `halt-key!` clears the
event-client and connection-pool atoms and tears down what they held, but a registration completing
afterwards wrote a live channel and a live connection pool straight back into them, leaving a halted
component holding network resources that nothing would ever close — because the thing responsible
for closing them had already run.

**Neither `:status` nor the component's existing state can carry the answer.** `:status` is a plain
value on a map the caller no longer shares. The atoms cannot serve either, because `nil` already
means *not yet connected* and so cannot also mean *halted*. Promoting `:status` itself to an atom
looks like the economical fix and is not: the three system-health functions read `(:status v)` as a
plain value while walking every component uniformly, and `get-combined-system-health` permits no
per-component workarounds — so one component's problem would be paid for by all of them.

**The remedy is one mutable marker on the component, set by `halt-key!` before it does anything
else, and consulted by every entry point that would resurrect it.** Set it first, so a call racing
the teardown is refused rather than half-served. Consult it *early* — above any resource
acquisition — so a refusal costs nothing and reaches no collaborator. And have the refusal be
**data the caller can recognise** (`{:refused :client-halted}`), not `nil` or `false`: a caller
testing for failure with `(false? result)` treats `nil` as success, and one testing for a specific
sentinel treats a general one as the wrong cause.

The general rule: **if a component can be reached by anything that outlives its halt, "halted" must
be expressed in something mutable that those holders share.** Otherwise a halted component stays
halted only by luck of ordering — which is a property of the current wiring, not a guarantee.

### Self-stopping on a dedicated thread: reap it with `.shutdown`, not `.shutdownNow`

A component that schedules its own halt on a dedicated executor must reap that executor with `.shutdown` (graceful), never `.shutdownNow` (forceful) — whenever the scheduled task routes back through the component's own `halt-key!`.

The `grpc-server` auto-halt grace timer (ADR-0036 §Auto-Halt Grace Period) is the canonical example. A single-thread scheduled executor fires a self-halt after the grace period; that self-halt runs the component's `halt-key!` (via `stop-collaboration-server!`) on the **same scheduler thread**, and `halt-key!` does interrupt-sensitive blocking work (`.awaitTermination` on the gRPC server). If the self-halt (or `halt-key!`) reaps the scheduler with `.shutdownNow`, that call **interrupts its own running task** — the current thread — and `.awaitTermination` then throws `InterruptedException` the moment it has to wait, aborting the rest of the shutdown: the server is never removed and the green session-stopped notification is never posted. The bug is invisible in isolation (the server is already terminated when the await runs, so it returns without checking the interrupt) and surfaces only under load.

`.shutdown` marks the executor for orderly shutdown but lets the current task finish without interrupting it; the single-thread executor then terminates on its own. Rule of thumb: **never `.shutdownNow` an executor from a task running on that same executor if anything after the call can be aborted by an interrupt.**

### When to extract a component

Promote a piece of state or capability to its own Integrant component when **any one** of the following is true:

- It has multiple consumers
- It has lifecycle independent of its current "owner"
- It represents a separate concern (different responsibility from the component that currently holds it)
- Consumers are reaching into a sibling component to access it

The bar is intentionally low. Tiny components (an atom + `:status :running`) are perfectly idiomatic and have negligible cost. The cost of the alternative — a god component with a wide implicit API — is much higher.

---

## 7. How `ig/build` Differs from `ig/init`

The combined app uses `ig/build` rather than `ig/init`. The distinction matters.

`ig/init` hardwires the transform: it calls `ig/init-key` for each component and nothing else. There is no hook between component initialisations.

`ig/build` is the public generic version of the same algorithm — same topological sort, same ref resolution — but it accepts a custom **transform function** that wraps each `ig/init-key` call. Ooloi uses this to inject splash screen progress reporting between components:

```clojure
(ig/build config (keys config)
  ;; Custom transform: report progress, then init
  (fn [key value]
    (when on-progress (on-progress key))
    (let [result (ig/init-key key value)]
      (when on-ready (on-ready key result))
      result))
  ...)
```

This means:
- Components know nothing about splash screens
- The splash update fires once per component, in init order, with no boilerplate in `init-key`
- Partial failure is handled explicitly: if a component throws, all already-started components are halted before the exception propagates

`start-system!` in `shared/system.clj` is the implementation. `start-app!` wraps it with the full splash lifecycle — showing the splash when the UI manager comes up, dismissing it when startup is complete.

---

## 8. Adding a New Component

Every new backend component added to `combined-config` requires all of the following. The first two are genuinely unintuitive and cause failures with no obvious connection to the missing item.

### 1. `ui-manager` dependency in `combined-config` — even for non-UI components

This requirement applies **only in `combined-config`** (§5), not in the backend standalone config (§4). In the standalone backend there is no UI manager, no splash screen, and no i18n loaded at init time — backend components start with no frontend dependencies at all.

In the combined application, the UI manager loads i18n and shows the splash screen. Backend components that call `tr` during init will crash if they start before the UI manager is running. Integrant sorts by the declared dependency graph, so **every backend component in `combined-config` must declare a `ui-manager` dependency** to guarantee it starts after i18n is ready — even if the component itself has nothing to do with the UI. Without this dependency, Integrant may place the component before the UI manager in the topological sort, producing a NullPointerException with "target is null" deep in the translation machinery. Nothing in the stack trace points to a missing dependency.

Note the asymmetry: in `backend-config` the same component may have `{}` (no dependencies), while in `combined-config` it must have `{:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}`. The `init-key` implementation is shared — it is the *config entry* that differs between the two systems.

```clojure
;; In combined-config (shared/system.clj) — ui-manager dependency required:
:ooloi.backend.components/my-component
{:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

;; In backend-config (backend/system.clj) — no frontend dependencies:
:ooloi.backend.components/my-component {}
```

### 2. `:status :running` in `init-key` return value

Every component (backend, frontend, or shared) must return either a `java.util.concurrent.ThreadPoolExecutor` (not shutdown) or a map containing `:status :running`. The three system-health functions enforce this invariant uniformly:

- `ooloi.backend.system/get-backend-system-health` (backend standalone deployment)
- `ooloi.frontend.system/get-frontend-system-health` (frontend test harness)
- `ooloi.shared.system/get-combined-system-health` (combined desktop application — the shipped product)

A component that returns a map without `:status :running` is silently reported as `:unhealthy`, making the whole system `:unhealthy`. No exception is thrown; health checks simply fail.

```clojure
(defmethod ig/init-key :ooloi.backend.components/my-component [_ _]
  {:my-state (atom nil)
   :status   :running})     ; required
```

**This invariant is test-enforced** for the combined system by the conformance test in `shared/test/app/clojure/ooloi/shared/system_test.clj` Section 30. Any future component added to `combined-config` that violates the invariant will fail the conformance test with the offending component name in the failure output. No per-component workarounds are permitted in `get-combined-system-health`.

### 3–7. The full checklist

3. Entry in `splash-message-keys` in `shared/system.clj`
4. Entry in `tr-declare` in `shared/system.clj`
5. Run `lein i18n` in `shared/` — adds the new key to `en_GB.po`
6. Copy and translate the new PO entry into all other locale files — `lein i18n` only touches `en_GB.po`
7. Use dynamic counts in tests — `(count (keys (system/combined-config)))` and `(count (system/splash-message-keys))`, never hardcoded integers

### 8. gRPC-accessible backend component: carry the dep through *every* site

If the new component is reached from a `shared/ops/` impl during a gRPC request (i.e. it's exposed via an `^{:api true}` multimethod and called via `SRV/*`), its Integrant dep must be carried through **all four** of these sites. Missing any one routes the component as `nil` at the impl, and the SRV call silently returns `{:result nil}` with no error visible to the caller.

**First consider a protocol.** If the component's state can be reached without the running server's per-request context — most can (a piece store, a registry) — the structurally-decoupled alternative is to declare the operation as a **protocol** in `shared` and implement it in `backend` (dependency inversion; `PieceResolver` is the existing example, `PieceManagerOps` another). That needs **none** of the four sites below, no `*server-component*`, and no runtime `resolve`. Use the wiring below to extend the existing `undo`/`instrument_library`/`event_subscription` seam, or where an op genuinely needs the per-request server context (`:transport`, `:server-id`).

| # | Site | What to add |
|---|---|---|
| 1 | `shared/system.clj` `combined-config` | `:my-component-component (ig/ref :ooloi.backend.components/my-component)` in `grpc-server`'s config map |
| 2 | `backend/components/grpc_server.clj` `ig/init-key` return | `:my-component-component (:my-component-component config)` in the final merge |
| 3 | `backend/components/grpc_server.clj` `build-grpc-server` `components` assoc | `:my-component-component (:my-component-component final-config)` — same site as `:instrument-library-component`. **This is the most-missed step**: `create-ooloi-service` captures the `components` map *before* `init-key` returns, so the dep must be assoc'd here or the live request handlers will never see it. |
| 4 | `shared/ops/<your>.clj` impl | `(:my-component-component @(resolve 'ooloi.backend.grpc.server/*server-component*))` |

**Do not add a new `^:dynamic *my-component-component*` var.** Read it from `*server-component*` directly. See [POLYMORPHIC_API_GUIDE.md §Non-VPD Singleton API Functions](POLYMORPHIC_API_GUIDE.md#non-vpd-singleton-api-functions) for the rationale and the canonical `event_subscription.clj` precedent.

---

## 9. Testing Components

### Test utility namespaces

Ooloi's test utilities are split into six namespaces, each in its own source root under `shared/test/util/`. The split is enforced by the classpath: backend tests load only what their test profile maps in, so backend-only tests cannot accidentally load frontend-coupled helpers.

| Namespace | File location | Source root | Auto-discovered by Midje in |
|---|---|---|---|
| `util.common` | `shared/test/util/common/util/common.clj` | `util/common` | shared, backend (via :resource-paths — no auto-discover), frontend |
| `util.server` | `shared/test/util/backend/util/server.clj` | `util/backend` | shared; reachable from backend via :resource-paths (no auto-discover) |
| `util.client` | `shared/test/util/backend/util/client.clj` | `util/backend` | shared only — `util.client` requires both backend and frontend code, so it can only load in the shared project (which has both on its classpath) |
| `util.frontend` | `shared/test/util/frontend/util/frontend.clj` | `util/frontend` | shared, frontend |
| `util.instrument-library` | `shared/test/util/backend/util/instrument_library.clj` | `util/backend` | shared; reachable from backend via :resource-paths |
| `util.filesystem` | `shared/test/util/backend/util/filesystem.clj` | `util/backend` | shared; reachable from backend via :resource-paths |

**Why the split?** `util.server` previously also contained client-side helpers (`register-client`, `with-clients`, `with-combined-system`). Those reference `ooloi.frontend.grpc.event-client` and `ooloi.shared.system` (which transitively pulls in the frontend), making the combined file unloadable from backend-only tests. Splitting the client-side helpers into a separate `util.client` namespace lets the backend's test profile pull `util/backend` in via `:resource-paths` (classpath access without Midje auto-discovery) while leaving anything frontend-coupled out of the backend's loadable surface. The frontend project never references `util.client` (servers and clients are integration concerns and integration tests live in the shared project), so `util.client` sits under `util/backend/` alongside `util.server`: shared has `util/backend` on `:test-paths` and auto-loads it; backend has it on `:resource-paths` (reachable but not auto-loaded; backend tests don't require it); frontend doesn't have `util/backend` on its classpath at all.

#### A test helper may never live in a production file

Test helpers belong in these namespaces and nowhere else. Not in a `src/` namespace, not even beside machinery they share with production, and a docstring reading "Test-only" in `src/` is the smell rather than a mitigation.

Production code is what ships. A test-only function sitting in it is dead weight at best, and at worst it gets *reached for* by later production code precisely because it is there. `run-on-fx-thread-sync!` — a blocking JAT bridge whose ten-second timeout exists so a deadlocked JAT fails a test run rather than hanging it — was promoted from a test utility into production `fx.clj` so that one caller could block and the suite could drop four flush calls. That caller was later reverted; the function stayed; and the dirty-close work eventually used it to raise a modal dialog, turning a test's deadlock guard into a deadline on the user's thinking time in front of "do you want to save the changes?". Four separate docstring prohibitions did not prevent any of that. Returning the function to `util.frontend` did, because production then has no var to call.

**When a helper needs something private from a production namespace, make it public.** Widening a namespace's own surface costs far less than housing test code in it, and nothing is hard to relocate in Clojure. Do not duplicate the private helper, and do not invent a third namespace to share it — both are heavier than dropping a hyphen. `util.filesystem/make-alias!` writes a macOS Finder alias for the `[:alias t]` fixture entry; the CFURL primitives it needs (`cf-fn`, `cfurl-from-path`, `release!`) are public in `ooloi.shared.platform.macos-alias`, which itself keeps only the alias *resolution* production actually calls.

The helper's own tests move with it: `make-alias!` is tested in `util.filesystem-test`, beside the helper, while `macos-alias-test` asserts only on production's `resolve-alias-target` and creates its alias as fixture setup.

#### A helper wanted by *both* sides lives in production

The rule above runs one way only, and its converse is not "so keep everything test-shaped in the test namespaces". **Where the same helper is wanted on both sides of the `src/`–`test/` line, there is one implementation and it lives in production.** Tests can require production code; production cannot require tests. Anything else means two implementations of one idea, and two implementations drift.

They drift silently, because both work. `wait-for-state` and `wait-for-event` were test-only in `util.common` while the combined application hand-rolled the same `add-watch`/promise pattern separately, in the wait a piece window's close performs for its own in-flight refetches. Neither was wrong. But only the test copy had the pre-watch race written into it — that a watcher installed after its value has already arrived never fires for it, so the current value must be tested *after* installation — and a second production consumer was about to make it three copies.

The implementation now lives in `ooloi.shared.async`, which every project can reach: `shared/src/main/clojure` is on both the backend's and the frontend's `:source-paths`, so `ooloi.shared.*` is available to all three projects and all their tests. `util.common` **re-exports** the two names rather than defining them, so several hundred existing call sites keep the name they know. A re-export is one implementation, not a duplicate.

**The test is who uses it, and it is worth applying deliberately rather than by feel:**

| Used by | Lives in |
|---|---|
| Production and tests, identically — or near-identically with a parameter | Production (`ooloi.shared.*`), re-exported under the test-facing name if one already exists |
| Tests only | The `util.*` namespaces above, and never `src/` |

Getting this backwards in either direction is a mistake, and they are different mistakes: a test helper in `src/` gets reached for by production that should not have it, while a shared idea kept test-side grows a private production twin that nobody maintains.

#### One idiom per lifecycle

**Where a macro exists for a lifecycle, tests use it.** When a test needs something the macro does not offer, the macro grows the option — it is not bypassed.

The reason is not tidiness. A hand-rolled copy of a lifecycle cannot be corrected centrally, so when a new subsystem arrives, every copy silently becomes wrong at once — and "silently" is exact, because teardown that abandons work in flight produces no failure, only work that fails later against components already dismantled around it.

Two functions in particular get hand-rolled and should not be:

- **`util.client/halt-app!`** is the teardown for any test that starts an application. It waits for the shared pool to fall idle and then halts, with the halt in a `finally` so that a drain which times out still tears the system down rather than leaking it. A test that halts a started application itself goes through this rather than writing its own flush-and-halt sequence.
- **`util.client/create-client-preserving-root`** is for a client created *beside a running application*. `grpc-clients`' `init-key` unconditionally claims the `*srv-client*` root binding — correct and wanted when no application is running, and wrong when one is, because it points the application's own pool-dispatched startup work at the new client. That work then fails against a client that has not registered (a nil connection pool), has been disconnected (dead channels), or is unknown to the server — on a pool thread, with no assertion involved, while the suite reports green. The 26 app-less creations correctly keep the plain `create-client-component`.

**Awkwardness is evidence, not an obstacle.** Where replacing an older form with the macro makes the resulting code *worse*, that is information about the macro rather than about the site: the older form is fulfilling a purpose the macro does not cover. It stays, and the reason is recorded. It may instead mean the macro should grow an option, or that two macros should merge — and **dropping a macro outright is a legitimate outcome**, not a failure. What is not legitimate is bending a test out of shape to satisfy a tidiness rule. The point of one idiom per lifecycle is that cognitive load drops; a contorted call site raises it.

Standard imports per project:

```clojure
;; Backend test — only what's loadable from this project:
(:require [util.server :refer :all]
          [util.common :refer [wait-for-event wait-for-state]])

;; Shared test exercising client-server integration:
(:require [util.server :refer :all]
          [util.client :refer :all]
          [util.common :refer [wait-for-event wait-for-state]])

;; Frontend UI test:
(:require [util.frontend :as th]
          [util.common :as tc])
```

### `with-tmp-filesystem` — a real temp directory tree

`util.filesystem/with-tmp-filesystem` builds a real temporary directory tree for tests that exercise the ADR-0051 filesystem operations (`list-filesystem-roots` / `list-filesystem-directory`, `open-piece`, `save-piece`). `(with-tmp-filesystem [root-sym structure] & body)` binds `root-sym` to a fresh temp-root `File`, materialises `structure` under it, runs the body, then recursively deletes the root — even when the body throws.

`structure` is a `name → kind` map, where the kind is one of:

- `{}` — an empty directory; `{<children>}` — a directory containing the described children (recurses).
- `:ooloi` — an empty `.ooloi` file (directory listings classify by extension and never deserialise, so an empty file is enough for listing tests); `:txt` — a text file; `:empty` — an empty file.
- `[:ooloi p]` — an `.ooloi` file holding a **real serialised piece**, where `p` is a piece value or a 0-arg fn producing one (e.g. `"score.ooloi" [:ooloi fxt/piece-with-notes]`) — the one to use when the test must `open-piece` or `save-piece` it.
- `[:symlink t]` — a symlink to relative path `t`; `[:alias t]` — a macOS Finder alias to sibling `t` (macOS-only; guard with `platform/macos?`).

To reach the tree through the opaque SRV token API, redef the platform roots at it — `platform/user-folders` → `(constantly [{:name "Test" :file root}])` and `platform/mounted-volumes` → `(constantly [])` — then mint dir-tokens with `list-filesystem-roots` / `list-filesystem-directory`. Combine with `with-server` + `with-clients` + `with-srv-client` for the client-scoped token registry.

### Choosing the Right Test Macro — Decision Table

Ooloi provides five primary test macros covering distinct test scopes. Pick the lightest one that exercises the production code path your test is verifying. The heavier macros initialise more components and pay correspondingly higher setup/teardown cost, but they're the only way to exercise certain integration paths (frontend → backend event pipeline, JAT-scheduled callbacks, splash lifecycle).

| What you're testing | Macro to use | Scope started |
|---|---|---|
| gRPC client-server API calls — wire protocol, headers, interceptors, event streaming, security | `with-server` + `with-clients` + `with-srv-client` | `grpc-server` + `http-server` + `piece-manager` (store via the ops namespace handle); `instrument-library` / `backend-undo-manager` passed as `nil` |
| Full backend Integrant system — multi-client integration with real piece/IL/undo state | `with-system` | All backend components (piece-manager, instrument-library, backend-undo-manager, grpc-server, http-server, cache-daemon) |
| The running **application** — startup sequence, menu bar, windows, anything reached through a real launch | `with-started-app` | Everything `start-app!` starts, plus the startup window-set; headless unless `{:graphical? true}` |
| Full combined application — frontend → backend pipeline, JAT callbacks, UI manager state | `with-combined-system` | All 15 baseline components (every backend + every frontend), headless UI by default |
| Frontend UI manager and individual frontend components | `with-ui-manager` | Just thread-pool, event-bus, ui-manager |

`with-started-app` and `with-combined-system` both give a whole application, and the difference is what starts it: `with-combined-system` initialises the component graph directly, while `with-started-app` goes through `start-app!` — the production launch, splash and all — so it is the one to use when the *startup* is part of what the test is about. See [Frontend Architecture Guide §12](FRONTEND_ARCHITECTURE_GUIDE.md#12-testing-model).

The §Component init scope below has the detail on which components each macro actually starts, the consequence for tests that reach into nil dependencies, and the macro signatures and options.

### Component init scope

The three system-level macros differ in *which components actually start*. Pick the lightest that gives your test the state it needs to exercise the production code path it's verifying.

| Macro | Backend components initialised | Frontend components initialised | Use when |
|---|---|---|---|
| `with-server` | `grpc-server` + `http-server`, plus a **`piece-manager`** — lightweight (its `init-store!` only creates the store ref on the ops namespace handle), so `with-server` tests get a real piece store, mirroring production where the gRPC server always runs alongside one. `instrument-library` and `backend-undo-manager` are **not** initialised — they're passed to the gRPC server as `nil`. | None | gRPC mechanism tests (TLS handshake, header propagation, event streaming, security interceptors, client registration) and tests that store/read pieces directly. The test must not invoke an SRV call that reaches `instrument-library` or `backend-undo-manager` state. |
| `with-system` | Full backend: `piece-manager`, `instrument-library`, `backend-undo-manager`, `grpc-server`, `http-server`, `cache-daemon`. | None | Multi-client integration tests that need real backend state (IL, pieces, undo stacks) but no frontend pipeline. SRV calls that touch IL or pieces are safe here. |
| `with-combined-system` | All backend components (same as `with-system`). | All five frontend components: `event-bus`, `ui-manager`, `grpc-clients`, `event-router`, `fetch-coordinator`. | Tests that need the frontend → backend event pipeline, JAT-scheduled callbacks, or UI manager state. Heaviest macro. |

#### `with-server` with nil backend dependencies: `Future.get` NPE pitfall

**Symptom.** An SRV call inside `with-server` fails with `clojure.lang.ExceptionInfo: SRV/<method> failed: Execution error: Cannot invoke "java.util.concurrent.Future.get()" because "fut" is null`.

**Cause.** `with-server` initialises the gRPC server, the HTTP health server, and a `piece-manager` — but **not** `instrument-library` or `backend-undo-manager`, which are passed to the gRPC server as `nil`. Those two are reached through `*server-component*`: the op impl resolves `(:instrument-library-component server-component)` (or `(:undo-manager-component server-component)`) to `nil`. The subsequent `(deref nil)` falls through Clojure's `deref` into `deref-future`, which calls `.get` on a `Future` that is `nil`. The server's exception handler catches this and returns `{:success false :error "Execution error: Cannot invoke \"java.util.concurrent.Future.get()\" because \"fut\" is null"}`, which the client wrapper surfaces as the ExceptionInfo above.

The error message is misleading — there's nothing wrong with futures, with promises, with the gRPC wire layer, or with the SRV call mechanism. The dependency was simply never initialised, and the resulting `nil` propagated until it hit a `.get` on a nullable Java reference.

**Piece operations are *not* subject to this pitfall.** They reach the store through the ops **namespace handle** (`pm/store-piece`, `resolve-into-piece-ref`), which the `with-server` piece-manager populates — not through `*server-component*`. So storing and reading pieces under `with-server` works; only `instrument-library` and `backend-undo-manager` are the nil-dependency hazards.

**Fix.** If the test needs `instrument-library` or `backend-undo-manager` state, switch from `with-server` to `with-system` (full backend) or `with-combined-system` (full backend + frontend, if the test also needs the frontend pipeline). Piece state needs no switch — `with-server` already provides it.

> **Why `with-server` starts a piece-manager (and IL/undo don't).** Earlier, the piece store was a JVM-global `defonce` — ambient under any `with-server`, so the piece-manager component didn't need starting here, and this section originally listed the piece-manager among the nil dependencies. Once the store became **component-owned** (created by `init-store!` at the component's `init-key`, released at `halt-key!`), that ambient store was gone, so `with-server` now starts a piece-manager to keep piece operations working — cheap, because it only creates a ref. `instrument-library` (loads its bundle from disk) and `backend-undo-manager` are heavier and reached via `*server-component*`, so they stay opt-in through `with-system`.

### `with-server` / `with-clients` / `with-srv-client`

The standard pattern for gRPC integration tests. `with-server` starts the gRPC server, the HTTP health server, and a piece-manager (plus the shared connection-registry / server-statistics / health-manager — see the init-scope table above), waits for startup (100ms default), then tears them down. `with-clients` creates client components, optionally registers them, then disconnects them. `with-srv-client` binds a client to the `SRV/*` dynamic context.

Every piece-manager-booting macro here — `with-server`, `with-system`, `with-combined-system`, and the narrower `with-piece-manager` / `with-shared-components` — redirects the platform directory (`~/.ooloi`) to a temporary directory for the duration of the test, via `util.common/with-test-platform-directory`. So no integration test touches the developer's real user directory: the persistent piece catalogue, the on-disk instrument library, and TLS certs all land in a temp directory that is deleted afterwards. A test that boots a persistent component *without* a piece-manager wraps `with-test-platform-directory` explicitly.

```clojure
;; Basic: one client, auto-registered
(with-server [server]
  (with-clients server [[client "test-client"]]
    (with-srv-client client
      (SRV/create-rest :duration 1/4) => truthy)))

;; In-process transport (no port binding, faster)
(with-server [server :in-process]
  (with-clients server [[client "test-client"]]
    (with-srv-client client
      (SRV/create-rest :duration 1/4) => truthy)))

;; TLS enabled — clients automatically inherit TLS settings from server
(with-server [server {:transport :network :tls true}]
  (with-clients server [[client "test-client"]]
    (with-srv-client client
      (SRV/create-rest :duration 1/4) => truthy)))

;; Test TLS mismatch — explicit client override
(with-server [server {:transport :network :tls true}]
  (with-clients server [[client "test-client" {:tls false}]]
    (event-client/register-with-server client-config client)
    => (throws io.grpc.StatusRuntimeException)))

;; Skip startup delay for simple tests
(with-server [server :sleep-ms 0]
  ...)

;; Deferred registration — when you need with-redefs before connecting
(with-server [server]
  (with-clients server [[client "test-client" :non-registered]]
    (let [events (atom [])]
      (with-redefs [event-client/process-received-event
                    (fn [event _] (swap! events conj event))]
        (register-client client)
        (server/send-server-event (:grpc server) {:type :test :message "hi"})
        (Thread/sleep 50)
        (some #(= (:message %) "hi") @events) => truthy))))
```

**Multi-client pattern** — for broadcast and selective delivery tests:

```clojure
(with-server [server]
  (with-clients server [[c1 "client-a" :non-registered]
                        [c2 "client-b" :non-registered]
                        [c3 "client-c" :non-registered]]
    (let [events1 (atom []) events2 (atom []) events3 (atom [])]
      (with-redefs [event-client/process-received-event
                    (fn [event client-id]
                      (cond
                        (= client-id "client-a") (swap! events1 conj event)
                        (= client-id "client-b") (swap! events2 conj event)
                        (= client-id "client-c") (swap! events3 conj event)))]
        (register-client c1)
        (register-client c2)
        ;; c3 not yet registered — first event reaches only c1 and c2
        (server/send-server-event (:grpc server) {:type :first :message "first"})
        (Thread/sleep 50)
        (register-client c3)
        ;; c3 now registered — second event reaches all three
        (server/send-server-event (:grpc server) {:type :second :message "second"})
        (Thread/sleep 50)
        (some #(= (:message %) "first") @events3) => falsey
        (some #(= (:message %) "second") @events3) => truthy))))
```

**Transport options:**
- `:network` (default) — real TCP/HTTP2, full flow control, production-equivalent
- `:in-process` — in-memory transport, faster, bypasses network layers; use for pure API tests

**Event validation** — validate content, not counts:

```clojure
;; ✅ Robust — checks what matters
(some #(= (:type %) :my-event) @events) => truthy
(some #(= (:piece-id %) "symphony-1") @events) => truthy

;; ❌ Fragile — breaks if implementation sends different numbers of internal events
(count @events) => 1
```

**Root binding behaviour — load-bearing, not hygiene** — `with-clients` deliberately leaves `*srv-client*`'s root binding pointing at the last client it initialised, and most tests depend on this.

The frontend `grpc-clients` component's `init-key` unconditionally `alter-var-root`s `ooloi.shared.srv-client/*srv-client*` to itself ("single-client always" — correct for production, where exactly one frontend client ever initialises). In tests, every `with-clients` invocation re-fires this clobber. That is load-bearing: SRV calls inside the body without an explicit `with-srv-client` wrapper, and off-thread production code (Claypoole pool dispatches, JAT-scheduled SRV calls) reached from inside the body, both read the root and route through the client `with-clients` just created. Both established patterns rely on this.

**The combined-system + `with-clients` case is different.** When the test has already started a combined-system frontend via `start-app!`, the root was set to that frontend's grpc-clients (call it B). A subsequent `with-clients` for an additional bare client A clobbers root to A. Whether you want root = A or root = B inside the body is a per-test decision, not something the macros can resolve:

- If the body's pool dispatches must run through the registered bare client (the typical case — e.g. `instrument_library_test`'s DELETE round-trip needs the registered `client` to receive backend events), leave root = A. Default behaviour, no action needed.
- If the body's pool dispatches must run through the combined-system frontend (e.g. Test 43 in `undo_redo_test`, where B is the focus and A is a foreign mutator), restore root explicitly:

```clojure
(with-clients server [[client-a "foreign-mutator"]]
  (alter-var-root #'srv-client/*srv-client*
                  (constantly (:ooloi.frontend.components/grpc-clients sys)))
  ;; … body. with-srv-client client-a still overrides per A's calls.
)
```

The symptom of getting this wrong is identity-dependent server responses (`:own?` flipping unexpectedly, event subscribers not seeing broadcasts) with no exception and no stack trace pointing at client identity — the SRV calls succeed but route through the wrong client.

**`register-client-with-server` forwards the full component.** When a test uses `:non-registered` in `with-clients` and then registers later, the function call is `(uc/register-client-with-server client server)`. The helper passes the entire client component as the config argument to `event-client/register-with-server` — it does not `select-keys` a curated subset. The receiving side's destructure picks the keys it needs (`:client-id`, `:transport`, `:tls`, `:cert-path`, `:insecure-dev-mode`, `:deadline-ms`, etc.) and ignores everything else. This is deliberate: a `select-keys` form had to be patched once when TLS keys were added and would have needed another patch for `:deadline-ms`. Passing the whole component is harmless and future-proof.

**`:http false` opt-out — TEMPORARY WORKAROUND** — when `with-server`'s default behaviour (start an HTTP statistics server on port 10701 alongside the gRPC server) collides with a test that needs multiple gRPC servers running concurrently, pass `:http false` in the config to skip the HTTP server:

```clojure
(with-server [s-net {:transport :network :port net-port :http false}]
  (with-server [s-ip {:transport :in-process :http false}]
    ;; Two grpc-servers, zero http-servers — no port collision
    ...))
```

This is a workaround for the current production-side coupling between `http-server` and `grpc-server` (see §6 worked example). **It is scheduled for removal once the http-server decoupling lands.** After that:

- `http-server` no longer holds a ref to any specific `grpc-server`; it depends on the shared `connection-registry`, `server-statistics`, and `health-manager` components directly
- Tests can instantiate any number of gRPC servers with at most one shared `http-server` (or none) without port collision
- The `:http false` flag will be deleted from `start-server`, and the few tests that use it will be updated to instantiate `http-server` separately (or not at all)

If you find yourself reaching for `:http false` in a new test today, leave a comment indicating it's a temporary workaround so the migration sweep finds it.

**Manual-lifecycle exceptions.** `with-server` is the canonical entry point, but some test patterns can't fit it and retain inline `ig/init-key` calls. These get the shared-component refs (`connection-registry`, `server-statistics`, and after Phase 0 expansion: `health-manager`) threaded through their hand-rolled configs via a file-local helper pair (`init-server-refs` / `halt-server-refs`):

| Pattern | Why `with-server` doesn't fit |
|---|---|
| Binding-conflict tests | Need to assert that a second `ig/init-key` THROWS on port conflict; `with-server`'s startup-wait fires before the body can catch it |
| Double-halt shutdown tests | Need direct access to halt-key's return value for assertions; want to halt twice in sequence |
| Concurrent multi-server stress | Spin up N servers in `future` blocks; `with-server` is a single-server wrapper |
| Init-failure tests | Assert that init fails under specific conditions; bundle leak from `start-server`'s partial init isn't problematic but explicit `ig/init-key` is cleaner |

When you use this exception, declare it: a comment at the fact explaining why the test stays on manual lifecycle keeps the audit trail clear.

### `with-system`

For testing the full backend Integrant system — component coordination, health reporting, configuration, TLS. Uses `backend/system.clj`'s `start-with-config`.

```clojure
(with-system [system {}]
  (get-in system [:ooloi.backend.components/grpc-server :status]) => :running
  (get-in system [:ooloi.backend.components/piece-manager :status]) => :running)

(with-system [system {:tls true}]
  (get-in system [:ooloi.backend.components/grpc-server :config :tls]) => true
  (let [health (ooloi.backend.system/get-backend-system-health system)]
    (:status health) => :healthy))
```

The system map contains all Integrant components as returned by `start-with-config`. `with-system` halts everything in the `finally` block.

### `with-combined-system`

For testing the full combined application — all 15 baseline components, in-process transport, headless UI by default. This is the heaviest macro and the most faithful to production.

**Macro signature**: `[[system-symbol opts] & body]`. The `opts` map is optional and accepts the following keys: `:transport` (`:in-process` default, or `:network` for real gRPC channels with header propagation), `:extra-config` (a map merged into the Integrant config at component granularity to override any component's settings — port, `:ui-mode`, TLS, etc.), `:on-progress` (called before each component inits), and `:on-ready` (called after each component inits). See §Options below for the full table.

```clojure
(with-combined-system [sys]
  ;; All 15 baseline components are running
  (get-in sys [:ooloi.backend.components/grpc-server :status]) => :running
  (get-in sys [:ooloi.frontend.components/ui-manager :status]) => :running

  ;; Combined system with a gRPC client
  (with-clients sys [[client "test-client"]]
    (with-srv-client client
      (SRV/create-rest :duration 1/4) => truthy)))
```

**Signature**: `[[system-symbol opts] & body]`. `opts` is an optional map; when omitted, the macro defaults to `:in-process` transport and `:headless` UI mode — the original single-argument behaviour. All existing call sites continue to work unchanged.

**Options**:

| Option | Default | Effect |
|---|---|---|
| `:transport` | `:in-process` | gRPC server transport. Use `:network` for tests that need real wire-level metadata propagation (e.g. multi-client `client-id` flow through the auth interceptor). |
| `:extra-config` | `nil` | Map merged into the Integrant config at component granularity via `(merge-with merge ...)`. Use to override any component config key — port, `:ui-mode`, TLS settings — without replacing the whole component config. |
| `:on-progress` | no-op | `(fn [component-key])` called before each component inits. Pass-through to `start-system!`. |
| `:on-ready` | `nil` | `(fn [component-key result])` called after each component inits. Pass-through to `start-system!`. |

**Examples**:

```clojure
;; Network transport — exercise real gRPC channels, headers, auth interceptor
(with-combined-system [sys {:transport :network}]
  (with-clients sys [[c1 "a"] [c2 "b"]]
    ;; Each client's id flows through CLIENT_ID_HEADER on every call
    ...))

;; Component config override (e.g. non-default port, alternate UI mode)
(with-combined-system [sys {:extra-config
                            {:ooloi.backend.components/grpc-server {:port 11000}}}]
  ...)

;; Startup observation
(let [started (atom [])]
  (with-combined-system [sys {:on-ready (fn [k _] (swap! started conj k))}]
    (count @started) => (count (system/combined-config))))
```

**Defaults preserved**: With no opts, the macro forces `:in-process` and `:headless` exactly as before. This is the right choice for combined-desktop end-to-end tests where there is exactly one client and no JavaFX windows should appear.

**Domain event subscriptions are NOT wired by `with-combined-system`.** The macro calls `start-system!` (Integrant init only), not `start-app!` (which adds post-init wiring). Tests that need the full backend-event → frontend-handler pipeline — e.g. backend broadcasts `:instrument-library-changed` → event router → aggregator → event bus → `handle-library-changed!` → re-fetch — must call `wire-domain-subscriptions!` explicitly:

```clojure
(with-combined-system [sys]
  (system/wire-domain-subscriptions! sys)
  ;; Now backend events reach frontend handlers through production wiring
  ...)
```

`wire-domain-subscriptions!` is defined in `shared/src/app/clojure/ooloi/shared/system.clj`. Production gets this wiring from `start-app!`, which calls it before `event-client/register-with-server`. The function is deliberately separate so tests can opt in without dragging in splash screens, menu bars, and piece window creation.

**Nor is the client registered with the server.** `with-combined-system` starts the components but does not call `event-client/register-with-server` (also a `start-app!` step). A test that makes a direct `SRV/*` call — or exercises production code that does, such as a piece window's `:window/on-open` hook subscribing to its piece — must call `(event-client/register-with-server grpc-clients grpc-clients)` first (both arguments are the `grpc-clients` component), or the SRV connection pool is unavailable and the call throws *"API connection pool not available - call register-with-server first"*.

**And under `{:transport :network}` it cannot be registered at all without more.** The `:transport` option is applied to the **backend** gRPC server only; the frontend `grpc-clients` keeps the `:transport :in-process` that `combined-config` gives it, and takes its `backend-server-name` from the server component it refs — which, in network mode, has none. Calling `register-with-server` on the application's own client therefore takes the in-process branch with a nil server name and throws `NullPointerException` out of `InProcessChannelBuilder/forName`. A network combined system that needs its app client connected must override the frontend component too, via `:extra-config`. The remedy the paragraph above prescribes is, in other words, unavailable in exactly the configuration where the client is furthest from working.

**Aggregator queue requirement:** every category returned by `derive-category` (in `frontend/event_router/core.clj`) must have a corresponding queue in the aggregator (`frontend/event_router/aggregator.clj`). Missing queues cause events to be silently dropped — `add-event` uses `when-let` on the queue lookup. When adding a new event category, update both files.

**Async synchronisation note:** after `register-with-server`, allow 100ms before reading server registry state — gRPC connections are established asynchronously.

**Pool sizing for thread-parking tests.** `with-combined-system` inherits the production pool default (`cores-1`). A test that **parks a pool thread** — e.g. blocking a fetch inside a mocked `SRV/*` call on a promise to force an out-of-order landing, as the latest-wins refetch tests do — must set a fixed floor via `:extra-config`:

```clojure
(with-combined-system [sys {:extra-config {:ooloi.shared.components/thread-pool {:size 8}}}]
  ...)
```

Two traps make this necessary. `cores-1` is `1` on a two-core CI runner, which **deadlocks** the blocked task — no thread is left to run anything else. And capping the pool *below* `cores-1` **starves** the other pool work: under a multi-namespace sweep the threads are busy with combined-system startup, so a `cp/future` dispatched at window-open queues behind them and loses its expected ordering to a later-dispatched refetch — a flake that never reproduces run alone, only under sweep contention. A fixed floor at or above the peak concurrent pool tasks (8 is ample) avoids both.

### `with-ui-manager`

For tests that need a UI manager without the full combined system. Creates a thread pool, event bus, and UI manager; flushes outstanding JAT callbacks before halting. Prevents `RejectedExecutionException` teardown races.

```clojure
(th/with-ui-manager [mgr]
  ;; mgr is a running UI manager
  ;; Access the event bus via (:event-bus mgr)
  (some-fn-that-needs-ui-manager mgr) => expected-result)

;; With options
(th/with-ui-manager [mgr {:pool-size 4 :ui-mode :headless}]
  ...)

;; Bind the manager's live Claypoole pool as a second symbol, to hand to a show-*
;; fn that fetches on the pool — no nested with-thread-pool needed
(th/with-ui-manager [mgr pool]
  (th/run-on-fx-thread-sync! #(picker/show-piece-picker! mgr :pool pool))
  ...)

;; Pool symbol and options together
(th/with-ui-manager [mgr pool {:pool-size 4}]
  ...)

;; With extra Integrant config merged into init-key
(th/with-ui-manager [mgr {:extra-config {:some-key some-value}}]
  ...)
```

Binding forms: `[mgr]`, `[mgr opts-map]`, `[mgr pool]`, or `[mgr pool opts-map]`. A symbol after the manager binds the manager's live pool — so a test can pass it to a `show-*` fn that fetches on the pool, with no nested `with-thread-pool`; a map is the options. Defaults: `:pool-size 2`, `:ui-mode :headless`. `:extra-config` is merged into the `ig/init-key` config map. See [Frontend Architecture Guide §12](FRONTEND_ARCHITECTURE_GUIDE.md#12-testing-model) for the full frontend testing model.

**Visual inspection tests** — when a test needs to show a real window on screen (e.g. to verify layout, content, or appearance), pass `:ui-mode :graphical`. In headless mode (the default), `show-window!` builds and registers the Stage but never calls `window/show!`, so nothing appears on screen. Combine with `(visual-pause ms)` and the `OOLOI_UI_VISUAL=true` environment variable:

```clojure
(th/with-ui-manager [mgr {:ui-mode :graphical}]
  (th/await-window-event mgr :window-opened :my-window
    (fn [] (th/run-on-fx-thread-sync! #(my-window/show-my-window! mgr)))) => truthy
  (th/visual-pause 5000)   ; no-op unless OOLOI_UI_VISUAL=true
  ...)
```

Run with: `OOLOI_UI_VISUAL=true lein midje my.namespace`

### Menu bars in tests: the host contract, and driving a menu open

**`:menu-bar-host` holds either `nil` or `{:stage … :menu-bar …}` — never a partial map.** `halt-key!` closes the `:stage` on teardown, so a host installed without one makes that close throw `NullPointerException: Cannot invoke "javafx.stage.Stage.close()"` during halt, which the failure guard reports as a recorded failure rather than as anything resembling its cause. A test that installs a bar so `refresh-menu-text!` can reach it supplies both keys, on the JavaFX thread:

```clojure
(th/run-on-fx-thread-sync!
  (fn []
    (reset! (:menu-bar-host mgr)
            {:stage    (Stage.)
             :menu-bar (menus/build-menu-bar! :linux {} {} (atom nil) (atom #{}))})))
```

**`.show()` on a `Menu` fires `onShowing` even when the bar is attached to no scene**, headless, for every platform structure. A test can therefore drive a real menu-open through the handlers `build-menu-bar!` installs, rather than setting whatever state those handlers write. Since `build-menu-bar!` takes `platform` as a parameter, one test covers macOS, Windows and Linux on any host without a `platform/macos?` conditional — stub `menus/setup-macos-app-menu!` while doing so, per [ADR-0042](../ADRs/0042-UI-Specification-Format.md) §macOS native integration, which reserves `init-menu-bar-host!` for tests whose subject *is* the native menu.

### `await-window-event`

Window opens and closes are asynchronous: `show-window!` / `close-window!` publish to `:window-requests`, the UI Manager performs the operation on the JAT, and only then publishes `:window-opened` / `:window-closed` on `:window-lifecycle`. A test that reads the window registry immediately after triggering a show races that pipeline. `await-window-event` is the standard synchronisation:

```clojure
(th/await-window-event mgr :window-opened :my-window
  (fn [] (th/run-on-fx-thread-sync! #(my-window/show-my-window! mgr)))) => truthy
```

It subscribes to `:window-lifecycle` **before** running `trigger-fn` (avoiding the race where the event fires before the subscription is in place), then blocks up to `timeout-ms` (default 5000) for the matching `(event-type, window-id)` event — returning truthy if it fired, `nil` on timeout. `trigger-fn` is whatever makes the window emit the event: a `run-on-fx-thread-sync!` show or close, or a raw `:window-requests` publish. The same helper covers closes:

```clojure
(th/await-window-event mgr :window-closed :my-window
  (fn [] (th/run-on-fx-thread-sync! #(um/close-window! mgr :my-window)))) => truthy
```

`await-window-event` is for **trigger-based** waits. Combined-system `start-app!` tests, where the piece window opens autonomously during startup with no test-triggered action, use a different shape — subscribe to `:window-lifecycle` *and* pre-check the registry for the already-open case. See [Frontend Architecture Guide §12](FRONTEND_ARCHITECTURE_GUIDE.md#12-testing-model).

### `force-headless`

A one-line config transform: `(th/force-headless config)` returns `config` with `[:ooloi.frontend.components/ui-manager :ui-mode] :headless`.

**Tests do not call it.** `with-started-app` applies it, and `with-ui-manager` and `with-combined-system` already default to headless; the transform is named here because it is what those macros do, not because a test should reach for it. A test needing real shown windows — modal-gating, where the modal takes its owner from a shown Stage, or Robot input — asks for them with `(with-started-app [sys-atom {:graphical? true}] …)` rather than by omitting the transform.

Headless suppresses only `window/show!`; registration, scene assembly, menu wiring, and lifecycle events are unchanged. See [Frontend Architecture Guide §12](FRONTEND_ARCHITECTURE_GUIDE.md#12-testing-model) for what `with-started-app` guarantees and the two shapes it cannot express.

### Async helpers from `util.common`

For tests that need to wait on asynchronous state changes (events arriving in an atom, a registry counting up or down, a flag flipping), `util.common` provides two helpers built on `promise` + `add-watch` + `(deref _ timeout-ms nil)`. They return as soon as the condition is satisfied, with a hard upper-bound timeout — replacing brittle `Thread/sleep N` waits.

#### `wait-for-event` — predicate per element

For collection-valued atoms (a vector of received events, a map of clients keyed by id) where the test asks **"did any matching element arrive?"**. Predicate is applied per element via `some`.

```clojure
(require '[util.common :refer [wait-for-event]])

;; Wait up to 1 second for an event with :message "hello" to arrive
(wait-for-event client-events #(= (:message %) "hello") 1000)

;; Wait for a piece event with a specific piece-id
(wait-for-event client-events #(= (:piece-id %) "symphony-1") 1000)
```

Use for the common case of "the test sends a message and waits for it to reach the destination atom".

#### `wait-for-state` — predicate on whole atom value

For assertions that can only be expressed against the entire atom value — counts, absences, structural checks. Predicate is applied to the full value, not to individual elements.

```clojure
(require '[util.common :refer [wait-for-state]])

;; Wait for registry to shrink back to 1 entry after a client is halted
(wait-for-state registry #(= 1 (count %)) 1000)

;; Wait for a specific client-id to be removed from a registry
(wait-for-state registry #(not (contains? % "test-client-x")) 1000)

;; Wait for an atom to satisfy any whole-value condition
(wait-for-state undo-stack #(<= 2 (count %)) 1000)
```

**Why two helpers, not one.** Per-element `(some pred coll)` cannot express absence or count: pred only sees one element at a time, with no view of the rest. To wait for "client X is gone", the predicate must inspect the whole collection. Conversely, `wait-for-event` is more terse for the common "did any matching event arrive?" case. The split avoids forcing every caller into a `#(some matching-pred %)` wrapper.

Both helpers:
- Add a watcher to the atom that delivers a promise the first time pred is satisfied
- Cover the pre-watch race by also testing pred against the current value before deref
- Always remove the watcher before returning, regardless of timeout or success

This pattern replaces `CountDownLatch` (Java-style coordination forbidden anywhere in Ooloi, tests or production, per [§11](#11-deprecated-patterns)) and ad-hoc `(loop [tries] ... Thread/sleep ... recur)` polling loops.

#### `wait-until` — polled predicate, for conditions with nothing to watch

The last resort, and genuinely last. `wait-for-state` and `wait-for-event` are *woken* by the change; `wait-until` samples for it. Reach for it only where the condition is not an atom reaching a state, so there is nothing to hang a watcher on.

```clojure
(require '[util.common :refer [wait-until]])

;; a JavaFX property — not an atom, so no watcher is possible
(wait-until #(= "Öppna…" (.getText menu-item)) 5000) => truthy

;; a LongAdder statistic (ADR-0025) mutates in place inside a map whose identity never changes
(wait-until #(= 1 (stats/get-server-stat server :clients-disconnected-total)) 5000)

;; it returns what pred returned, so it also serves find-and-return
(let [cell (wait-until #(list-cell-for stage token) 5000)]
  (robot/robot-click! robot (centre-of cell)))
```

**Returns pred's own value on success, nil on timeout.** That costs nothing over returning `true` and covers the find-and-return case, which is otherwise the one shape that still needs a hand-rolled loop — a `VirtualFlow` cell that only exists after a layout pulse, a live dialog button that has to be located among `Window/getWindows`.

**It is still better than a sleep, for the reason that governs this whole class:** it returns the moment the condition holds instead of always paying the full delay, and it fails at a deadline instead of proceeding silently to an assertion that was never going to hold.

**Do not reach for it when an atom does express the condition.** Three separate suites had written *"the target is a JavaFX property, not a Clojure atom, so `wait-for-state` can't watch it"* — correct, and the wrong conclusion, because polling was available all along. The opposite error is commoner: polling a view when the model behind it is an atom, which samples a consequence instead of watching the cause.

### Teardown that provokes production behaviour must wait for it to finish

A teardown step is not always inert. Halting a component can set real production work in motion, and if the test returns before that work completes, the rest of the teardown dismantles the system underneath it.

The clearest case is the collaboration server. Halting a gRPC server completes its clients' observers, so the frontend sees a terminal event and begins an involuntary reversion to the local backend ([ADR-0036 §Involuntary Reversion](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)). That reversion is asynchronous — it runs on the gRPC executor, not on the test thread — and it re-registers against the in-process backend. A test that returns the instant the server is halted abandons it mid-flight, and the enclosing `with-combined-system`'s `ig/halt!` then takes away the very backend it is re-registering against. The reversion fails; and since the frontend now records such failures rather than discarding them ([ADR-0017 §Surfacing Unexpected Runtime Failures](../ADRs/0017-System-Architecture.md)), the result is a stack trace printed inside a green run.

The running application never shows this, because nothing there cuts the reversion short. The defect is in the test, not in the code beneath it.

**Wait on the post-condition the behaviour establishes, not on a signal that merely correlates with it.** The reversion's post-condition is the API connection pool back on `:in-process`:

```clojure
(ig/halt-key! :ooloi.backend.components/network-grpc-server net)
(wait-for-state (:api-connection-pool grpc-clients)
                #(= :in-process (:transport (:config %)))
                5000)
```

Waiting instead for the "host disconnected" notification looks equivalent and is not: the reversion posts that notification *before* it switches transports, so the wait returns while the switch is still in flight. The distinction is not cosmetic — with everything else identical, it was the difference between a namespace that printed stack traces and one that did not.

The general shape: ask what a teardown step **causes**, not merely what it **stops**. Where it causes asynchronous work, wait for that work's own post-condition before letting the rest of the teardown proceed. The same reasoning applies to any halt that triggers a reaction — a component whose shutdown publishes an event, or whose closure prompts a reconnect elsewhere.

### Stack traces in a green run — stub `log-error` at the fact that provokes it

A suite log is expected to contain **zero stack traces**. A run that is green but prints one is not a clean run, and the trace is a defect to be fixed rather than an artefact to be explained.

The distinction that matters is between a message and a trace. A production error handler that prints *"Instrument library persistence error: Simulated disk error"*, driven by a fact that redefs `spit` to throw, is expected furniture: the fact provokes the handler on purpose and the handler's entire job is to say so. A block of `at …` frames is never furniture. It means a throwable reached the single write site and nothing in the run absorbed it.

**Every such trace arrives through one function.** `ooloi.shared.components.thread-pool/log-error` is the one place a failure becomes a written record ([ADR-0017 §What a Deliberate `catch` May Do](../ADRs/0017-System-Architecture.md)). It is public, on every classpath, and reachable from all three projects. Whether the throwable came from the pool's `afterExecute` net, a deliberate `catch`, or a JavaFX callback, the writing happens there — so that is where a test intercepts it.

**Stub it at the fact that provokes it, in one of two forms.** Where the fact merely provokes the boundary and asserts something else:

```clojure
;; log-error stubbed to keep the run's log clean: this fact provokes the catch-all
;; on purpose, and the record it writes is correct behaviour rather than a defect.
(with-redefs [tp/log-error (fn [_])]
  ...)
```

Where the fact asserts that the record was made:

```clojure
(fact "an unforeseen save failure records its throwable through log-error"
  (let [recorded (atom [])]
    (with-redefs [tp/log-error (fn [e] (swap! recorded conj e))]
      ...
      (count @recorded) => 1)))
```

Always carry a comment naming what the stub absorbs and why. A bare `(fn [_])` with no comment is indistinguishable from silencing a trace nobody understood.

**Never quiet a trace by changing production.** The trace means the error boundary did precisely what ADR-0017 requires of it. Widening a `catch`, dropping a `throw`, or adding a guard so the failure cannot arise all remove real coverage in order to tidy a log. The write is correct; only the writing *during a test that provoked it deliberately* is noise.

**A trace that appears intermittently still has to go.** Some are race-dependent — a registry entry removed asynchronously between a snapshot and its use, for instance — so the same fact prints on one run and not the next. Stubbing at the seam is deterministic either way, which is why it is preferred over capturing the stream: nothing reaches stderr whether the race fires or not, and there is no intermittent assertion to flake.

**Capturing `System/err` is for a different question.** Redirecting the stream (`System/setErr` over a `ByteArrayOutputStream`, restored in a `finally`) asserts on the *route* — that the default handler is still `log-error`, or that the text genuinely reaches stderr in a deployment with no UI Manager. Use it when the stderr route itself is the subject. When the subject is anything else and the trace is merely incidental, stub `log-error`.

Worked examples:

| Where | Form |
|---|---|
| `shared/…/ops/persistence_test.clj` | both — `(fn [_])` for the facts that only provoke the catch, `(swap! recorded conj e)` for the two that assert the record |
| `backend/…/components/http_server_test.clj` | `(swap! recorded conj e)` across the stats-handler failure paths |
| `shared/…/grpc/event_streaming_test.clj` | `(fn [_])` for a race-dependent registry throw, and a `System/setErr` capture where the route is the subject |

### The failure guard — a body that records a failure fails

The convention above is enforced rather than remembered. Every test macro that owns a pool, a server or an application — `with-thread-pool`, and through it `with-ui-manager` and `with-event-bus`; `with-server`; `with-system`; `with-started-app`; `with-combined-system` — captures `log-error` for the duration of its body and **fails the test if anything was recorded**. A deliberately-provoked record has to be declared; everything else surfaces.

**`with-clients` deliberately does not carry it, and the reason generalises.** Every call site passes a server symbol, so every body it wraps already sits inside `with-server`'s guard: a second guard adds no coverage. It would subtract correctness, because being the *inner* net it captures records a fact declared at the natural place — around the whole scenario, outside the client block — leaving that fact's own atom empty and firing on a test that had done everything right. **The guard belongs on the macro that owns the lifecycle, not on one nested within it.** Where two guards would nest, the outer one is the only one that should exist.

There is nothing new to learn to satisfy it. The two forms above *are* the declaration, because the guard hooks the same seam they do, which is what lets the two compose at all.

**A `System/err` capture is not a declaration.** `(System/setErr …)` changes where `log-error` writes, never whether it writes: the call still happens, the record still reaches the write site, and only the bytes land in a buffer instead of the console. A capture therefore removes the evidence and keeps the cause — which is why a suite log can be clean while facts in it have been recording failures all along, and why the guard fires on a fact whose output is silent.

That decides which of the two tools a fact wants, and it is not a matter of taste:

- **The trace is incidental** — the fact provokes the boundary and asserts on a notification, a return value, an atom. Declare through `log-error`.
- **The write is the subject** — the fact asserts that the full report reaches stderr, carrying the `ex-data` and the frames the notification deliberately omits. Capture, and put the fact where nothing has replaced `log-error`. Inside a guarded scope the stream belongs to the guard, so a capture there observes test infrastructure rather than production.

**Where the subject is the record rather than the stream, assert the record.** A fact wanting the *rendered* report can take the captured throwable and render it with production's own `throwable->text`. What it gives up is the last inch only — that those bytes reach the stream — and that is `log-error`'s one-line body, covered by the thread pool's own tests against an unguarded pool.

**The declaration goes INSIDE the macro's body.** This is the opposite of where a mocked *collaborator* goes:

```clojure
(with-thread-pool [pool]                  ; correct — the declaration nests inside
  (with-redefs [tp/log-error (fn [_])]    ; the guard's capture, and absorbs
    …))
```

Placed outside, the guard's capture becomes the innermost binding: it takes the record the fact is trying to observe, the fact's atom stays empty, and the guard fires. The collaborator rule in [FRONTEND_ARCHITECTURE_GUIDE §6.5](FRONTEND_ARCHITECTURE_GUIDE.md) points the other way for a different reason — there the drain must run while the mock is still in place — and the two do not conflict, because the guard is itself the outer net. A record written during the drain reaches it, which is exactly what should happen to work that outlived the body.

**A declaration covers its own extent, not the whole fact.** One `(fn [_])` somewhere does not switch the guard off for the rest of the body; a record written after that form returns is undeclared and fails. And since the rebinding is a var root rather than a thread-local, a declaration absorbs records from *any* thread while it is in force — so keep it tight around the provocation.

**Not everything that prints is a record.** A throwable that never reaches `log-error` is invisible to the guard, and correctly so: a JavaFX runnable that throws after the toolkit's uncaught-exception handler has been detached goes to the toolkit's own reporting rather than to the write site. Such a fact keeps its `System/err` capture, because there is no record for it to declare.

**When the guard fires it takes the namespace with it.** Midje evaluates facts at load time, so a throw out of a top-level `fact` aborts the load: the run reports `LOAD FAILURE` carrying *"Failures were recorded during this body"*, and every fact after it in that file is skipped. **That is a RED, not a harness error** — the fix is at the fact, and rerunning changes nothing.

### Proving a string is translated, not concatenated

`lein i18n` cannot see a user-facing string that never became a key — there is no key to extract, so nothing is missing and the build stays green. Concatenation and inline literals are therefore invisible to it, and two of ADR-0039's invariants rest on test instead of on the build task. [ADR-0039 §What Verification Cannot See](../ADRs/0039-Localisation-Architecture.md#what-verification-cannot-see) states why; this is how.

**Produce the same output under two locales and compare.** A `tr` call consults a catalogue and yields different text; a hardcoded prefix yields identical text both times. A single-locale assertion cannot tell them apart.

```clojure
(uco/with-saved-atom tr/current-locale
  (tr/set-locale! :en-GB) => :en-GB
  (let [en (produce-the-thing)]
    ;; Assert the switch. set-locale! falls back to :en-GB for an unavailable
    ;; locale and returns what it actually selected; an unasserted fallback
    ;; compares English against English and fails for an unrelated reason.
    (tr/set-locale! :sv-SE) => :sv-SE
    (let [sv (produce-the-thing)]
      ;; The runtime value survives translation …
      (re-find #"the-parameter" en) => some?
      (re-find #"the-parameter" sv) => some?
      ;; … and the surround does not, because it came from a catalogue.
      en =not=> sv)))
```

Keep the *input* identical across both halves. Any difference between the two outputs is then the translated surround and nothing else — which is the whole assertion.

**Restore the locale.** `tr/current-locale` is a process-global atom; `uco/with-saved-atom` restores it. Frontend tests get this from `with-frontend-test-config`, which also restores `tr/catalogs` — restoring only the selector leaves a test free to empty the catalogues for every namespace loaded after it.

**A failure means one of two things, and both are real.** Either the production code never called `tr`, or the key was never translated into the second locale — a key missing from a catalogue returns the UK English text (ADR-0039 §Forward Compatibility), so an untranslated key is indistinguishable from a hardcoded one. The second is why this assertion cannot pass until the 22-locale sweep has landed; that ordering is a property of the test, not an obstacle to it.

Worked examples at three altitudes:

| Where | What it compares |
|---|---|
| `frontend/…/instrument_library/staff_editor_test.clj` | cljfx description maps — label text and a combo-box `:button-cell` describe fn, rendered under `:en-GB` then `:de-DE` |
| `shared/test/clojure/ooloi/shared/i18n/tr_test.clj` | `tr` itself, and the `set-locale!` return-value guard |
| `shared/test/app/clojure/ooloi/shared/system_test.clj` | a notification raised through the running application, read back from the UI manager's registry |

---

## 10. Invariants and Pitfalls

**Every map-type component must return `:status :running`.**
All three system-health functions (`get-backend-system-health`, `get-frontend-system-health`, `get-combined-system-health`) check this uniformly for every value in the system map. Omitting it produces silent `:unhealthy` status with no error. The combined system enforces this invariant via the Section 30 conformance test.

**Backend components in `combined-config` must depend on `ui-manager`.**
Without it, topological sort may place them before the UI manager, causing `tr` to fail before i18n is loaded. This does not apply to the standalone backend config (§4), which has no UI manager.

**`ig/build` does not auto-cleanup on partial failure.**
If a component throws during init, the partial system (components that started successfully) is in `ex-data` under `:system`. `start-system!` handles this explicitly — it halts the partial system before re-throwing. If you call `ig/build` directly, you must do the same.

**Use dynamic counts in tests, never hardcoded.**
`(count (keys (system/combined-config)))` and `(count (system/splash-message-keys))` change whenever a component is added. Hardcoding the number breaks every test that depends on it.

**Tests run sequentially — fixed ports are safe.**
All servers use default ports (gRPC: 10700, HTTP: 10701). There is no need for random port allocation. The `with-combined-system` macro defaults to in-process transport (no port binding); when called with `{:transport :network}`, it binds the default 10700/10701, same as `with-server`.

**Prefer `wait-for-event` / `wait-for-state` over `Thread/sleep` for async assertions.**
Client registration with `register-client` is synchronous. gRPC event delivery is asynchronous. After dispatching an event, wait for it to arrive at the destination — but `Thread/sleep N` is brittle (too short and the test flakes, too long and the suite slows down). The canonical replacement is `(wait-for-event events-atom pred timeout-ms)` for "did the matching event arrive in this atom?", or `(wait-for-state state-atom pred timeout-ms)` for whole-collection conditions like count or absence. Both helpers in `util.common` return as soon as the condition is met, with a hard upper-bound timeout. See [§9 Async helpers](#async-helpers-from-utilcommon).

`Thread/sleep` is still appropriate when *modelling* delay (simulating a slow client's processing time, waiting for a known fixed-duration external event) — i.e. when the sleep IS the test's behaviour, not a workaround for asynchrony. In all other cases, prefer the helpers.

**A halted component that outside callers still hold needs a mutable marker, not `:status`.**
`halt-key!` returns a new map; every closure created while the component ran still holds the original, whose `:status` says `:running` for ever. Where something outside the lifecycle can act on the component after its halt, express "halted" in something mutable both maps share, set it before the teardown, and consult it above any resource acquisition. See [§6 A halted component stays halted](#a-halted-component-stays-halted-halt-keys-return-value-cannot-tell-anyone).

**Component keys in `combined-config` are namespaced keywords matching their `init-key` dispatch.**
The frontend event-router and fetch-coordinator use fully-qualified namespace keys:
- `:ooloi.frontend.components.event-router/event-router`
- `:ooloi.frontend.components.fetch-coordinator/fetch-coordinator`

This is intentional — it allows these components to coexist in a system with other components from the same project without key collision.

---

## 11. Deprecated Patterns

### Manual `ig/init-key` / `ig/halt-key!` in test bodies

The explicit `ig/init-key` / `ig/halt-key!` test pattern — creating and cleaning up components manually in `let` / `try` / `finally` blocks — is still valid Integrant but is no longer the recommended approach for new tests. The macros in §9 provide the same guarantees with far less boilerplate and no risk of missing cleanup on failure.

If you encounter existing tests that look like this:

```clojure
;; Deprecated explicit pattern
(let [server (ig/init-key :ooloi.backend.components/grpc-server {:transport :network :port 10700})
      client (ig/init-key :ooloi.frontend.components/grpc-clients client-config)]
  (try
    ;; test logic
    (finally
      (ig/halt-key! :ooloi.frontend.components/grpc-clients client)
      (ig/halt-key! :ooloi.backend.components/grpc-server server))))
```

They are correct and will continue to work, but new tests should use `with-server` / `with-clients` instead. The explicit pattern remains the right choice for REPL exploration and for complex scenarios genuinely not covered by the macros (see §9 "Manual-lifecycle exceptions" for the named cases).

The `start-server` / `stop-server` / `register-client` / `disconnect-client` functions are the procedural equivalents of the macros. They are useful at the REPL but not in tests.

### `CountDownLatch` for async coordination

`java.util.concurrent.CountDownLatch` is a legacy Java pattern and should appear nowhere in Ooloi — tests or production. There are none left in either. The canonical replacements:

- For "wait until callback fires" (1 expected delivery): `(promise)` + `(deref p timeout-ms nil)`. The callback delivers the promise.
- For "wait until N callbacks fire": an `atom` counting remaining deliveries plus a single `promise` delivered when the atom reaches zero.
- For "wait until an atom satisfies a predicate" (events arriving, registry counts changing, flag flipping): `wait-for-event` (per-element pred) or `wait-for-state` (whole-value pred) from `util.common`. See [§9 Async helpers](#async-helpers-from-utilcommon).
- For "wait until a condition holds that is **not** an atom" (a JavaFX property, a `LongAdder` statistic, a node that only exists after a layout pulse): `wait-until` from `util.common`, which polls and returns what its predicate found.

Why deprecated: `CountDownLatch` requires the caller to know the exact count in advance, doesn't compose with predicates, and conflates "thing happened" with "Java synchroniser tripped" — making test failures harder to diagnose. The promise-based replacements are idiomatic Clojure, compose with any condition, and produce clearer failure messages.

### `Thread/sleep N` for async synchronisation

`(Thread/sleep N)` is **not** appropriate when used to "give the async thing time to happen" before an assertion — too short and the test flakes, too long and the suite is slow, and either way the test asserts against a clock rather than against the code. Use `wait-for-event` / `wait-for-state` when an atom expresses the condition, and `wait-until` when nothing does (see §9). **Do not hand-roll a polling loop**: `wait-until` is that loop, extracted, and a new copy of it is a regression rather than a leftover.

But a sleep is not always a stand-in for a wait, and converting one that isn't silently deletes what the test was testing. Every sleep is read and decided; the four categories below are what the decision comes down to, and a sleep that stays says at its own site which one it is.

| Category | Verdict | Example |
|---|---|---|
| Stands in for a condition the assertion already names | **Convert** | a sleep before `(some #(= (:message %) "first") @events)` → `wait-for-event` on that same predicate |
| Waits for something already guaranteed | **Delete** | a sleep after `ig/init-key` on a gRPC or HTTP server — `Server.start()` and `HttpServer/create` both bind before returning |
| Establishes that something did **not** happen | **Keep** | after unsubscribing, publish again and assert the handler did not fire. **You cannot wake on the absence of an event**: a test proving nothing happened by T must let T pass |
| The delay **is** the requirement | **Keep** | see the two kinds below |

The last category splits in two, and both are easy to mistake for laziness on a later sweep:

- **The delay is the code under test.** A stubbed server stalling past a client's deadline, so there is a timeout to observe at all. A slow subscriber whose latency is asserted. Ten health checks spaced across two seconds *as* the sustained load. Work submitted to a pool precisely so a drain has something to drain. Half an animation's duration, to sample it mid-flight — waiting would move the sample past the thing being sampled.
- **No JVM-observable signal exists.** `javafx.scene.robot.Robot` injects genuine OS input, which returns through the *native* event queue rather than the JAT's `runLater` queue, so a `run-on-fx-thread-sync!` barrier straight after an injection proves nothing about delivery. A shown `Stage` must be *mapped* by the window system before `requestFocus` takes effect. A `Popup` and its owner tearing down. `System/gc` is a request the JVM answers whenever it likes and never reports.

**One further trap, which is not a category but a hazard.** Waiting for a condition and then asserting that same condition is one claim made twice, and the assertion is then vacuous. Where that happens the wait *becomes* the assertion — `(wait-for-event received #(= :window-focused (:type %)) 5000) => truthy` in place of a sleep followed by `(some …) => truthy`. Where the fact makes a second, different claim, keep both: wait for the batch a drop composed and assert *what* it composed; wait for a selection to appear and assert *which* ids it holds.

### `requiring-resolve` (or runtime `resolve`) to break a dependency cycle

Reaching another namespace's var at call time via `requiring-resolve` / `resolve` to avoid a compile-time `:require` is, in most cases, a smell — coupling solved by substandard means. It hides the real edge from the namespace graph and from tooling, and the next maintainer works around it instead of questioning it. When you find one (outside the two exceptions below), the fix is almost always structural, and which structure depends on *why* the direct call was avoided:

- **A genuine cycle** (A depends on B which transitively depends on A) → break it with the **event bus**: the producer publishes a fact on a frontend event-bus category (ADR-0031), and the owner of the affected state subscribes and reacts. This is how the collaboration menu (`:collaboration-state-changed`) and the undo cache (`:backend-changed`) were decoupled from `switch-to!`.
- **A misplaced function** → **relocate** it to a lower namespace both callers can `:require`. If a function has no dependency on the namespace it currently lives in, it does not belong there, and moving it removes the need for resolution entirely.
- **A late-binding need** (the collaborator only exists at runtime, after this component inits) → **inject** it as an Integrant dependency at `init-key`, or share the underlying state component (e.g. inject the connection-registry rather than reaching the whole running server).

**Legitimate exceptions — not smells:**

- **By-name dispatch** where the symbol genuinely is not known at compile time: the gRPC method dispatcher (`ns-resolve 'ooloi.shared.api` by method-name string) and record deserialization (`map->Record`). These are exactly what make the API surface and the record set plugin-extensible. The two differ in one way that matters: the method dispatcher resolves against a **literal** namespace, whereas record deserialization must *derive* one from a Java class name — and that name is munged, so the package segment has to be demunged before it names a namespace. Get that wrong and `require` no longer recognises the namespace as loaded, loads its file a second time, and mints a duplicate class whose instances compare unequal to the originals under `=`. See [ADR-0003: Plugins §Namespace Naming Requirement for Plugin Defrecords](../ADRs/0003-Plugins.md#namespace-naming-requirement-for-plugin-defrecords). Any future by-name dispatch that derives a namespace from a string inherits the same trap; one that uses a literal namespace does not.
- **The `*server-component*` late-binding seam**, documented in `ooloi.backend.grpc.server`: shared-tier op implementations reach injected backend components via `@(resolve '…*server-component*)` because the shared tier must not compile-depend on the backend tier; the rule there is *"never introduce a new per-component dynamic var."* This is a legitimate exception for the **existing** consumers (`undo`/`instrument_library`/`event_subscription`) — leave them be — but it is **not** the default for new shared→backend access. The same "shared can't compile-depend on backend" is satisfied by a **protocol** (declared in shared, implemented in backend, like `PieceResolver` or `PieceManagerOps`), which needs no `resolve` and no `*server-component*` wiring. The existing consumers reach only known, global state, so they are decouplable too. **Prefer a protocol for new work.**

If a runtime resolution is not one of those two exceptions, treat it as debt to be removed structurally.

---

## Cross-References

- [ADR-0017: System Architecture](../ADRs/0017-System-Architecture.md) — the decision to use Integrant and its rationale
- [ADR-0031: Frontend Event-Driven Architecture](../ADRs/0031-Frontend-Event-Driven-Architecture.md) — full frontend component dependency graph
- [ADR-0036: Collaborative Sessions and Hybrid Transport](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md) — how the combined app transitions between in-process and network transport
- [Frontend Architecture Guide](FRONTEND_ARCHITECTURE_GUIDE.md) — frontend testing model, JAT boundary, `with-ui-manager` in depth
- [gRPC Communication and Flow Control](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md) — two-phase client connection architecture
- [Combined Application Source README](../shared/src/app/README.md) — component wiring checklist with code examples
