# Integrant Components in Ooloi

> This guide is the practical companion to [ADR-0017: System Architecture](../ADRs/0017-System-Architecture.md), which records the rationale for using Integrant. Here we record how to work with that decision — how components are defined, wired, and tested across the three-project architecture.
>
> It is written for Ooloi developers who need to add components, understand the startup sequence, or write integration tests. It assumes familiarity with Clojure but not with Integrant.

---

## Table of Contents

1. [What Integrant Does](#1-what-integrant-does)
2. [The Component Lifecycle](#2-the-component-lifecycle)
3. [The Three-Project Structure](#3-the-three-project-structure)
4. [The Backend System](#4-the-backend-system)
5. [The Combined Application System](#5-the-combined-application-system)
6. [How `ig/build` Differs from `ig/init`](#6-how-igbuild-differs-from-iginit)
7. [Adding a New Component](#7-adding-a-new-component)
8. [Testing Components](#8-testing-components)
9. [Invariants and Pitfalls](#9-invariants-and-pitfalls)
10. [Deprecated Patterns](#10-deprecated-patterns)

---

## 1. What Integrant Does

Ooloi is a long-running desktop application. Components start in a specific order, hold resources (thread pools, gRPC channels, JavaFX stages, agents), and must be shut down cleanly when the application exits or a test completes. Integrant provides two things:

**Dependency injection** — a component declares its dependencies by naming them with `ig/ref`. Integrant resolves the full dependency graph and sorts it topologically before starting anything.

**Lifecycle management** — every component implements `ig/init-key` (start) and optionally `ig/halt-key!` (stop). Integrant calls these in the right order and, on halt, in reverse order.

The result: components know nothing about each other's startup order, and tests get guaranteed cleanup even when they throw.

---

## 2. The Component Lifecycle

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

In practice, Ooloi never calls `ig/init` directly — it uses `ig/build` for the combined app (see §6) and `start-with-config` for the backend. But the mental model is the same.

---

## 2a. Component Design Principles

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

The fix (Phase 0 of #211):

- Extract `connection-registry`, `server-statistics`, and `health-manager` into their own tiny Integrant components.
- `grpc-server` consumes them via `ig/ref` instead of constructing them internally.
- `http-server` drops its `:grpc-server` ref entirely and declares direct refs on the three shared components.

After the refactor:
- The config visibly shows `http-server → {connection-registry, server-statistics, health-manager}`. No more hidden coupling.
- Two gRPC servers can coexist, both depending on the shared state, with no ambiguity about ownership.
- Halting one gRPC server doesn't touch `http-server` — they share state, but not lifecycle.

This is "extract a component" applied to the same code three times. The result is what Integrant was built to produce.

### When to extract a component

Promote a piece of state or capability to its own Integrant component when **any one** of the following is true:

- It has multiple consumers
- It has lifecycle independent of its current "owner"
- It represents a separate concern (different responsibility from the component that currently holds it)
- Consumers are reaching into a sibling component to access it

The bar is intentionally low. Tiny components (an atom + `:status :running`) are perfectly idiomatic and have negligible cost. The cost of the alternative — a god component with a wide implicit API — is much higher.

---

## 3. The Three-Project Structure

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

## 4. The Backend System

When running as a standalone server, the backend starts five components:

```
instrument-library   (no dependencies — starts independently)

piece-manager
    ├── grpc-server  (also depends on instrument-library)
    │       └── http-server
    └── cache-daemon
```

`instrument-library` has no dependencies and starts independently of `piece-manager`. `grpc-server` depends on both `piece-manager` and `instrument-library`. `cache-daemon` depends only on `piece-manager`. `http-server` depends on `grpc-server` for health manager access.

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

The combined app's baseline `combined-config` initialises 14 components, plus an on-demand 15th (the network gRPC server) that the application adds at runtime when the host enables collaboration. The baseline divides into four initialisation groups that must run in order:

```
SHARED FOUNDATION
  thread-pool                              (no dependencies)

FRONTEND EARLY  — splash screen must exist before backend reports progress
  event-bus                                [thread-pool]
  ui-manager          ← shows splash       [thread-pool, event-bus]

BACKEND
  instrument-library                       [ui-manager]
  piece-manager                            [ui-manager]
  undo-manager                             [ui-manager]
  connection-registry  ← shared state      [ui-manager]
  server-statistics    ← shared state      [ui-manager]
  grpc-server         ← in-process only    [piece-manager, instrument-library,
                                            undo-manager, connection-registry,
                                            server-statistics]
  http-server                              [grpc-server]
  cache-daemon                             [piece-manager]

FRONTEND LATE  — connect to backend after it is running
  grpc-clients                             [ui-manager, grpc-server]
  event-router                             [grpc-clients, event-bus]
  fetch-coordinator                        [thread-pool, grpc-clients]
```

Backend components' dependency on `ui-manager` is the load-bearing design in `combined-config`: it forces them to start *after* the UI manager (and thus after the splash screen is showing and i18n is loaded). Without it, a backend component might init before i18n is ready and crash when it calls `tr`. This dependency exists **only** in `combined-config` — in the standalone backend config (§4), the same components have no frontend dependencies. The shared-state components (`connection-registry`, `server-statistics`) carry the dependency uniformly with every other backend component even though their `init-key` does no `tr`-touching work; the rule is uniform precisely so the topological invariant is not contingent on what each component happens to do today.

**Shared backend state has its own components.** `connection-registry` (the single map of registered clients) and `server-statistics` (the single set of counters reported via the HTTP statistics endpoint) live in their own Integrant components, depended on by the `grpc-server` via refs. This is the spec ADR-0036 §Hybrid Transport Architecture commits to: shared state with lifecycle is its own component, depended on by every consumer; no server owns it. The on-demand network gRPC server documented below depends on the same refs, which is what guarantees broadcasts triggered on either transport reach all registered clients and statistics counters report system totality across all transports.

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

   :ooloi.backend.components/undo-manager
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   ;; Shared backend state — see ADR-0036 §Hybrid Transport Architecture.
   ;; Depended on by every transport surface so broadcasts and statistics counters
   ;; are single sources of truth regardless of which transport a client joined through.
   :ooloi.backend.components/connection-registry
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/server-statistics
   {:ui-manager (ig/ref :ooloi.frontend.components/ui-manager)}

   :ooloi.backend.components/grpc-server
   {:piece-manager                (ig/ref :ooloi.backend.components/piece-manager)
    :instrument-library-component (ig/ref :ooloi.backend.components/instrument-library)
    :undo-manager-component       (ig/ref :ooloi.backend.components/undo-manager)
    :connection-registry          (ig/ref :ooloi.backend.components/connection-registry)
    :server-statistics            (ig/ref :ooloi.backend.components/server-statistics)
    :transport                    :in-process}

   :ooloi.backend.components/http-server
   {:grpc-server (ig/ref :ooloi.backend.components/grpc-server)}

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

**`:ooloi.backend.components/network-grpc-server`** — second gRPC transport surface, started when the host enables a collaboration session and stopped on manual termination or after a configurable grace period of no connected guests (ADR-0036 §Hybrid Transport Architecture). It declares the same shared-state dependencies as the in-process `grpc-server` — `piece-manager`, `instrument-library`, `undo-manager`, `connection-registry`, `server-statistics` — and points its config to the **same refs**. The in-process server is unaffected by the network server's lifecycle.

The pattern for dynamic components:

- Their config keys live alongside the static `combined-config` keys; the `init-key` and `halt-key!` methods are normal Integrant methods.
- The application's backend API adds the component to the running system map by calling `ig/init-key` directly, threading the live refs from the running system into the new component's config.
- `ig/halt-key!` removes the component when its lifecycle ends; the next reference to the system map omits the entry.
- The `:status :running` invariant applies while the component is running; the system-health functions (§9) walk every key in the system map uniformly — they need no special handling for dynamic components.
- The §30 conformance test enforces the invariant on whatever is present in the running system: components added dynamically during the test must comply too.

The same pattern can host future dynamic components — a future HTTP REST gateway, a future MIDI listener, anything that is not part of every application run but follows the Integrant lifecycle when it does run.

---

## 6. How `ig/build` Differs from `ig/init`

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

## 7. Adding a New Component

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

| # | Site | What to add |
|---|---|---|
| 1 | `shared/system.clj` `combined-config` | `:my-component-component (ig/ref :ooloi.backend.components/my-component)` in `grpc-server`'s config map |
| 2 | `backend/components/grpc_server.clj` `ig/init-key` return | `:my-component-component (:my-component-component config)` in the final merge |
| 3 | `backend/components/grpc_server.clj` `build-grpc-server` `components` assoc | `:my-component-component (:my-component-component final-config)` — same site as `:instrument-library-component`. **This is the most-missed step**: `create-ooloi-service` captures the `components` map *before* `init-key` returns, so the dep must be assoc'd here or the live request handlers will never see it. |
| 4 | `shared/ops/<your>.clj` impl | `(:my-component-component @(resolve 'ooloi.backend.grpc.server/*server-component*))` |

**Do not add a new `^:dynamic *my-component-component*` var.** Read it from `*server-component*` directly. See [POLYMORPHIC_API_GUIDE.md §Non-VPD Singleton API Functions](POLYMORPHIC_API_GUIDE.md#non-vpd-singleton-api-functions) for the rationale and the canonical `event_subscription.clj` precedent.

---

## 8. Testing Components

### Test utility namespaces

Ooloi's test utilities are split into four namespaces, each in its own source root under `shared/test/util/`. The split is enforced by the classpath: backend tests load only what their test profile maps in, so backend-only tests cannot accidentally load frontend-coupled helpers.

| Namespace | File location | Source root | Auto-discovered by Midje in |
|---|---|---|---|
| `util.common` | `shared/test/util/common/util/common.clj` | `util/common` | shared, backend (via :resource-paths — no auto-discover), frontend |
| `util.server` | `shared/test/util/backend/util/server.clj` | `util/backend` | shared; reachable from backend via :resource-paths (no auto-discover) |
| `util.client` | `shared/test/util/backend/util/client.clj` | `util/backend` | shared only — `util.client` requires both backend and frontend code, so it can only load in the shared project (which has both on its classpath) |
| `util.frontend` | `shared/test/util/frontend/util/frontend.clj` | `util/frontend` | shared, frontend |
| `util.instrument-library` | `shared/test/util/backend/util/instrument_library.clj` | `util/backend` | shared; reachable from backend via :resource-paths |

**Why the split?** `util.server` previously also contained client-side helpers (`register-client`, `with-clients`, `with-combined-system`). Those reference `ooloi.frontend.grpc.event-client` and `ooloi.shared.system` (which transitively pulls in the frontend), making the combined file unloadable from backend-only tests. Splitting the client-side helpers into a separate `util.client` namespace lets the backend's test profile pull `util/backend` in via `:resource-paths` (classpath access without Midje auto-discovery) while leaving anything frontend-coupled out of the backend's loadable surface. The frontend project never references `util.client` (servers and clients are integration concerns and integration tests live in the shared project), so `util.client` sits under `util/backend/` alongside `util.server`: shared has `util/backend` on `:test-paths` and auto-loads it; backend has it on `:resource-paths` (reachable but not auto-loaded; backend tests don't require it); frontend doesn't have `util/backend` on its classpath at all.

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

### Choosing the Right Test Macro — Decision Table

Ooloi provides four primary test macros covering distinct test scopes. Pick the lightest one that exercises the production code path your test is verifying. The heavier macros initialise more components and pay correspondingly higher setup/teardown cost, but they're the only way to exercise certain integration paths (frontend → backend event pipeline, JAT-scheduled callbacks, splash lifecycle).

| What you're testing | Macro to use | Scope started |
|---|---|---|
| gRPC client-server API calls — wire protocol, headers, interceptors, event streaming, security | `with-server` + `with-clients` + `with-srv-client` | `grpc-server` + `http-server` only; backend state components passed as `nil` |
| Full backend Integrant system — multi-client integration with real piece/IL/undo state | `with-system` | All backend components (piece-manager, instrument-library, undo-manager, grpc-server, http-server, cache-daemon) |
| Full combined application — frontend → backend pipeline, JAT callbacks, UI manager state | `with-combined-system` | All 14 baseline components (every backend + every frontend), headless UI by default |
| Frontend UI manager and individual frontend components | `with-ui-manager` | Just thread-pool, event-bus, ui-manager |

The §Component init scope below has the detail on which components each macro actually starts, the consequence for tests that reach into nil dependencies, and the macro signatures and options.

### Component init scope

The three system-level macros differ in *which components actually start*. Pick the lightest that gives your test the state it needs to exercise the production code path it's verifying.

| Macro | Backend components initialised | Frontend components initialised | Use when |
|---|---|---|---|
| `with-server` | `grpc-server` + `http-server` only. Dependencies (`piece-manager`, `instrument-library`, `undo-manager`) are **not** initialised — they're passed to the gRPC server as `nil`. | None | Pure gRPC mechanism tests: TLS handshake, header propagation, event streaming protocol, security interceptors, client registration mechanics. The test must not invoke any SRV call that reaches into a state-holding backend component. |
| `with-system` | Full backend: `piece-manager`, `instrument-library`, `undo-manager`, `grpc-server`, `http-server`, `cache-daemon`. | None | Multi-client integration tests that need real backend state (IL, pieces, undo stacks) but no frontend pipeline. SRV calls that touch IL or pieces are safe here. |
| `with-combined-system` | All backend components (same as `with-system`). | All five frontend components: `event-bus`, `ui-manager`, `grpc-clients`, `event-router`, `fetch-coordinator`. | Tests that need the frontend → backend event pipeline, JAT-scheduled callbacks, or UI manager state. Heaviest macro. |

#### `with-server` with nil backend dependencies: `Future.get` NPE pitfall

**Symptom.** An SRV call inside `with-server` fails with `clojure.lang.ExceptionInfo: SRV/<method> failed: Execution error: Cannot invoke "java.util.concurrent.Future.get()" because "fut" is null`.

**Cause.** `with-server` initialises only the gRPC server and HTTP health server — `piece-manager`, `instrument-library`, and `undo-manager` are passed in as `nil`. When the SRV call reaches one of those state-holding components, the op impl resolves `(:instrument-library-component server-component)` (or the equivalent for piece-manager / undo-manager) to `nil`. The subsequent `(deref nil)` falls through Clojure's `deref` into `deref-future`, which calls `.get` on a `Future` that is `nil`. The server's exception handler catches this and returns `{:success false :error "Execution error: Cannot invoke \"java.util.concurrent.Future.get()\" because \"fut\" is null"}`, which the client wrapper surfaces as the ExceptionInfo above.

The error message is misleading — there's nothing wrong with futures, with promises, with the gRPC wire layer, or with the SRV call mechanism. The dependency was simply never initialised, and the resulting `nil` propagated until it hit a `.get` on a nullable Java reference.

**Fix.** Switch the test from `with-server` to `with-system` (full backend) or `with-combined-system` (full backend + frontend, if the test also needs the frontend pipeline). Use `with-server` only for tests that don't reach state-holding components: pure gRPC mechanism tests — TLS handshake, header propagation, event streaming protocol, security interceptors, client registration mechanics, two-phase connection lifecycle.

### `with-server` / `with-clients` / `with-srv-client`

The standard pattern for gRPC integration tests. `with-server` starts both the gRPC server and HTTP health server, waits for startup (100ms default), then tears them down. `with-clients` creates client components, optionally registers them, then disconnects them. `with-srv-client` binds a client to the `SRV/*` dynamic context.

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

This is a workaround for the current production-side coupling between `http-server` and `grpc-server` (see §2a worked example). **It is scheduled for removal once #211 Phase 0 lands the http-server decoupling.** After that:

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

For testing the full combined application — all 14 baseline components, in-process transport, headless UI by default. This is the heaviest macro and the most faithful to production.

**Macro signature**: `[[system-symbol opts] & body]`. The `opts` map is optional and accepts the following keys: `:transport` (`:in-process` default, or `:network` for real gRPC channels with header propagation), `:extra-config` (a map merged into the Integrant config at component granularity to override any component's settings — port, `:ui-mode`, TLS, etc.), `:on-progress` (called before each component inits), and `:on-ready` (called after each component inits). See §Options below for the full table.

```clojure
(with-combined-system [sys]
  ;; All 14 baseline components are running
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

**Aggregator queue requirement:** every category returned by `derive-category` (in `frontend/event_router/core.clj`) must have a corresponding queue in the aggregator (`frontend/event_router/aggregator.clj`). Missing queues cause events to be silently dropped — `add-event` uses `when-let` on the queue lookup. When adding a new event category, update both files.

**Async synchronisation note:** after `register-with-server`, allow 100ms before reading server registry state — gRPC connections are established asynchronously.

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

;; With extra Integrant config merged into init-key
(th/with-ui-manager [mgr {:extra-config {:some-key some-value}}]
  ...)
```

Defaults: `:pool-size 2`, `:ui-mode :headless`. `:extra-config` is merged into the `ig/init-key` config map. See [Frontend Architecture Guide §12](FRONTEND_ARCHITECTURE_GUIDE.md#12-testing-model) for the full frontend testing model.

**Visual inspection tests** — when a test needs to show a real window on screen (e.g. to verify layout, content, or appearance), pass `:ui-mode :graphical`. In headless mode (the default), `show-window!` builds and registers the Stage but never calls `window/show!`, so nothing appears on screen. Combine with `(visual-pause ms)` and the `OOLOI_UI_VISUAL=true` environment variable:

```clojure
(th/with-ui-manager [mgr {:ui-mode :graphical}]
  (th/run-on-fx-thread-sync! #(my-window/show-my-window! mgr))
  (deref opened 5000 nil) => truthy
  (th/visual-pause 5000)   ; no-op unless OOLOI_UI_VISUAL=true
  ...)
```

Run with: `OOLOI_UI_VISUAL=true lein midje my.namespace`

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

This pattern replaces `CountDownLatch` (Java-style coordination forbidden in Ooloi tests per [§10](#10-deprecated-patterns)) and ad-hoc `(loop [tries] ... Thread/sleep ... recur)` polling loops.

---

## 9. Invariants and Pitfalls

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
Client registration with `register-client` is synchronous. gRPC event delivery is asynchronous. After dispatching an event, wait for it to arrive at the destination — but `Thread/sleep N` is brittle (too short and the test flakes, too long and the suite slows down). The canonical replacement is `(wait-for-event events-atom pred timeout-ms)` for "did the matching event arrive in this atom?", or `(wait-for-state state-atom pred timeout-ms)` for whole-collection conditions like count or absence. Both helpers in `util.common` return as soon as the condition is met, with a hard upper-bound timeout. See [§8 Async helpers](#async-helpers-from-utilcommon).

`Thread/sleep` is still appropriate when *modelling* delay (simulating a slow client's processing time, waiting for a known fixed-duration external event) — i.e. when the sleep IS the test's behaviour, not a workaround for asynchrony. In all other cases, prefer the helpers.

**Component keys in `combined-config` are namespaced keywords matching their `init-key` dispatch.**
The frontend event-router and fetch-coordinator use fully-qualified namespace keys:
- `:ooloi.frontend.components.event-router/event-router`
- `:ooloi.frontend.components.fetch-coordinator/fetch-coordinator`

This is intentional — it allows these components to coexist in a system with other components from the same project without key collision.

---

## 10. Deprecated Patterns

### Manual `ig/init-key` / `ig/halt-key!` in test bodies

The explicit `ig/init-key` / `ig/halt-key!` test pattern — creating and cleaning up components manually in `let` / `try` / `finally` blocks — is still valid Integrant but is no longer the recommended approach for new tests. The macros in §8 provide the same guarantees with far less boilerplate and no risk of missing cleanup on failure.

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

They are correct and will continue to work, but new tests should use `with-server` / `with-clients` instead. The explicit pattern remains the right choice for REPL exploration and for complex scenarios genuinely not covered by the macros (see §8 "Manual-lifecycle exceptions" for the named cases).

The `start-server` / `stop-server` / `register-client` / `disconnect-client` functions are the procedural equivalents of the macros. They are useful at the REPL but not in tests.

### `CountDownLatch` for async coordination

`java.util.concurrent.CountDownLatch` is a legacy Java pattern and should never appear in new Ooloi tests. The canonical replacements:

- For "wait until callback fires" (1 expected delivery): `(promise)` + `(deref p timeout-ms nil)`. The callback delivers the promise.
- For "wait until N callbacks fire": an `atom` counting remaining deliveries plus a single `promise` delivered when the atom reaches zero.
- For "wait until an atom satisfies a predicate" (events arriving, registry counts changing, flag flipping): `wait-for-event` (per-element pred) or `wait-for-state` (whole-value pred) from `util.common`. See [§8 Async helpers](#async-helpers-from-utilcommon).

Why deprecated: `CountDownLatch` requires the caller to know the exact count in advance, doesn't compose with predicates, and conflates "thing happened" with "Java synchroniser tripped" — making test failures harder to diagnose. The promise-based replacements are idiomatic Clojure, compose with any condition, and produce clearer failure messages.

### `Thread/sleep N` for async synchronisation

`(Thread/sleep N)` is appropriate when *modelling* delay (a deliberately slow client, a known fixed-duration external event). It is **not** appropriate when used to "give the async thing time to happen" before an assertion — too short and the test flakes, too long and the suite is slow.

Use `wait-for-event` / `wait-for-state` for the latter pattern (see §8). If neither fits, write a small inline polling loop with a deadline and a `Thread/sleep 10` between iterations — but in practice the helpers cover the vast majority of cases.

---

## Cross-References

- [ADR-0017: System Architecture](../ADRs/0017-System-Architecture.md) — the decision to use Integrant and its rationale
- [ADR-0031: Frontend Event-Driven Architecture](../ADRs/0031-Frontend-Event-Driven-Architecture.md) — full frontend component dependency graph
- [ADR-0036: Collaborative Sessions and Hybrid Transport](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md) — how the combined app transitions between in-process and network transport
- [Frontend Architecture Guide](FRONTEND_ARCHITECTURE_GUIDE.md) — frontend testing model, JAT boundary, `with-ui-manager` in depth
- [gRPC Communication and Flow Control](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md) — two-phase client connection architecture
- [Combined Application Source README](../shared/src/app/README.md) — component wiring checklist with code examples
