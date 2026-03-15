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

### Frontend — components only, no standalone app

The frontend defines Integrant components but has **no `system.clj` and is never deployed standalone**. Its components exist to be assembled by the shared combined system. Running `lein run` in `frontend/` has no meaning — the frontend project is a code library and a test harness.

```
frontend/src/main/clojure/ooloi/frontend/components/
  event_bus.clj
  ui_manager.clj
  grpc_clients.clj
  event_router.clj
  fetch_coordinator.clj
```

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

The combined app runs 11 components. They divide into four initialisation groups that must run in order:

```
SHARED FOUNDATION
  thread-pool                              (no dependencies)

FRONTEND EARLY  — splash screen must exist before backend reports progress
  event-bus                                [thread-pool]
  ui-manager          ← shows splash       [thread-pool, event-bus]

BACKEND
  instrument-library                       [ui-manager]
  piece-manager                            [ui-manager]
  grpc-server         ← in-process only    [piece-manager, instrument-library]
  http-server                              [grpc-server]
  cache-daemon                             [piece-manager]

FRONTEND LATE  — connect to backend after it is running
  grpc-clients                             [ui-manager, grpc-server]
  event-router                             [grpc-clients, event-bus]
  fetch-coordinator                        [thread-pool, grpc-clients]
```

Backend components' dependency on `ui-manager` is the load-bearing design in `combined-config`: it forces them to start *after* the UI manager (and thus after the splash screen is showing and i18n is loaded). Without it, a backend component might init before i18n is ready and crash when it calls `tr`. This dependency exists **only** in `combined-config` — in the standalone backend config (§4), the same components have no frontend dependencies.

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

   :ooloi.backend.components/grpc-server
   {:piece-manager                (ig/ref :ooloi.backend.components/piece-manager)
    :instrument-library-component (ig/ref :ooloi.backend.components/instrument-library)
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

`get-system-health` in `backend/system.clj` iterates every value in the system map. For map-type components it checks `(:status v) = :running`. A component that returns a map without this key is silently reported as `:unhealthy`, making the whole system `:unhealthy`. No exception is thrown; health checks simply fail.

```clojure
(defmethod ig/init-key :ooloi.backend.components/my-component [_ _]
  {:my-state (atom nil)
   :status   :running})     ; required
```

### 3–7. The full checklist

3. Entry in `splash-message-keys` in `shared/system.clj`
4. Entry in `tr-declare` in `shared/system.clj`
5. Run `lein i18n` in `shared/` — adds the new key to `en_GB.po`
6. Copy and translate the new PO entry into all other locale files — `lein i18n` only touches `en_GB.po`
7. Use dynamic counts in tests — `(count (keys (system/combined-config)))` and `(count (system/splash-message-keys))`, never hardcoded integers

---

## 8. Testing Components

All test macros live in `util.server` (for server/system tests) and `util.frontend` (for frontend UI tests). Load them with:

```clojure
(require '[util.server :refer :all])
(require '[util.frontend :as th])
```

### Choosing the right macro

| What you're testing | Macro to use |
|---|---|
| gRPC client-server API calls | `with-server` + `with-clients` + `with-srv-client` |
| Full backend Integrant system | `with-system` |
| Full combined application | `with-combined-system` |
| Frontend UI manager and components | `with-ui-manager` |

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

### `with-system`

For testing the full backend Integrant system — component coordination, health reporting, configuration, TLS. Uses `backend/system.clj`'s `start-with-config`.

```clojure
(with-system [system {}]
  (get-in system [:ooloi.backend.components/grpc-server :status]) => :running
  (get-in system [:ooloi.backend.components/piece-manager :status]) => :running)

(with-system [system {:tls true}]
  (get-in system [:ooloi.backend.components/grpc-server :config :tls]) => true
  (let [health (ooloi.backend.system/get-system-health system)]
    (:status health) => :healthy))
```

The system map contains all Integrant components as returned by `start-with-config`. `with-system` halts everything in the `finally` block.

### `with-combined-system`

For testing the full combined application — all 11 components, in-process transport, headless UI. This is the heaviest macro and the most faithful to production.

```clojure
(with-combined-system [sys]
  ;; All 11 components are running
  (get-in sys [:ooloi.backend.components/grpc-server :status]) => :running
  (get-in sys [:ooloi.frontend.components/ui-manager :status]) => :running

  ;; Combined system with a gRPC client
  (with-clients sys [[client "test-client"]]
    (with-srv-client client
      (SRV/create-rest :duration 1/4) => truthy)))
```

`with-combined-system` forces `:in-process` transport (no port binding) and `:headless` UI mode (no visual output). Tests that need network transport or a visible UI must start components individually.

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
```

Defaults: `:pool-size 2`, `:ui-mode :headless`. See [Frontend Architecture Guide §12](FRONTEND_ARCHITECTURE_GUIDE.md#12-testing-model) for the full frontend testing model.

---

## 9. Invariants and Pitfalls

**Every map-type component must return `:status :running`.**
`get-system-health` checks this for every value in the system map. Omitting it produces silent `:unhealthy` health status with no error.

**Backend components in `combined-config` must depend on `ui-manager`.**
Without it, topological sort may place them before the UI manager, causing `tr` to fail before i18n is loaded. This does not apply to the standalone backend config (§4), which has no UI manager.

**`ig/build` does not auto-cleanup on partial failure.**
If a component throws during init, the partial system (components that started successfully) is in `ex-data` under `:system`. `start-system!` handles this explicitly — it halts the partial system before re-throwing. If you call `ig/build` directly, you must do the same.

**Use dynamic counts in tests, never hardcoded.**
`(count (keys (system/combined-config)))` and `(count (system/splash-message-keys))` change whenever a component is added. Hardcoding the number breaks every test that depends on it.

**Tests run sequentially — fixed ports are safe.**
All servers use default ports (gRPC: 10700, HTTP: 10701). There is no need for random port allocation. The `with-combined-system` macro uses in-process transport and binds no ports at all.

**`Thread/sleep 50` after event dispatch, not after registration.**
Client registration with `register-client` is synchronous. gRPC event delivery is asynchronous. Sleep after sending an event; never after registering a client.

**Component keys in `combined-config` are namespaced keywords matching their `init-key` dispatch.**
The frontend event-router and fetch-coordinator use fully-qualified namespace keys:
- `:ooloi.frontend.components.event-router/event-router`
- `:ooloi.frontend.components.fetch-coordinator/fetch-coordinator`

This is intentional — it allows these components to coexist in a system with other components from the same project without key collision.

---

## 10. Deprecated Patterns

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

They are correct and will continue to work, but new tests should use `with-server` / `with-clients` instead. The explicit pattern remains the right choice for REPL exploration and for complex scenarios genuinely not covered by the macros.

The `start-server` / `stop-server` / `register-client` / `disconnect-client` functions are the procedural equivalents of the macros. They are useful at the REPL but not in tests.

---

## Cross-References

- [ADR-0017: System Architecture](../ADRs/0017-System-Architecture.md) — the decision to use Integrant and its rationale
- [ADR-0031: Frontend Event-Driven Architecture](../ADRs/0031-Frontend-Event-Driven-Architecture.md) — full frontend component dependency graph
- [ADR-0036: Collaborative Sessions and Hybrid Transport](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md) — how the combined app transitions between in-process and network transport
- [Frontend Architecture Guide](FRONTEND_ARCHITECTURE_GUIDE.md) — frontend testing model, JAT boundary, `with-ui-manager` in depth
- [gRPC Communication and Flow Control](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md) — two-phase client connection architecture
- [Combined Application Source README](../shared/src/app/README.md) — component wiring checklist with code examples
