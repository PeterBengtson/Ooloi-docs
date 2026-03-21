# ADR-0015: Undo and Redo Architecture in Collaborative Environment

## Table of Contents

- [Status](#status)
- [Context](#context)
  - [Key Architectural Realities](#key-architectural-realities)
- [Decision](#decision)
  - [Tier 1: Backend Undo/Redo (Coordinated)](#tier-1-backend-undoredo-coordinated)
  - [Tier 2: Frontend UI Undo/Redo (Local)](#tier-2-frontend-ui-undoredo-local)
  - [Tier 3: No Backend Application Settings Component](#tier-3-no-backend-application-settings-component)
  - [Unified User Experience](#unified-user-experience)
- [Rationale](#rationale)
- [Implementation Approach](#implementation-approach)
  - [Tier 2: Frontend Implementation](#tier-2-frontend-implementation)
  - [Tier 1: Backend Implementation — The Undo Manager](#tier-1-backend-implementation--the-undo-manager)
    - [Design Principle: Closures Over Immutable State](#design-principle-closures-over-immutable-state)
    - [The Undo Manager API](#the-undo-manager-api)
    - [Stack Semantics](#stack-semantics)
    - [Mutation Sites — How Resources Register Undo Steps](#mutation-sites--how-resources-register-undo-steps)
    - [Push-Based State Notification](#push-based-state-notification)
    - [Undo Operation — Sequence Diagram](#undo-operation--sequence-diagram)
    - [Collaborative Undo — Multi-Client Sequence](#collaborative-undo--multi-client-sequence)
  - [gRPC Extensions](#grpc-extensions)
  - [Unified Frontend Routing — Tier 1 and Tier 2 Merge](#unified-frontend-routing--tier-1-and-tier-2-merge)
    - [Frontend Undo State — Separate Stacks](#frontend-undo-state--separate-stacks)
    - [Routing Algorithm](#routing-algorithm)
    - [Single-User Guarantee](#single-user-guarantee)
    - [Multi-User Properties](#multi-user-properties)
    - [Undo Routing — Sequence Diagram](#undo-routing--sequence-diagram)
    - [Description Localisation](#description-localisation)
    - [Clock Skew Mitigation](#clock-skew-mitigation)
- [Consequences](#consequences)
- [Alternatives Considered](#alternatives-considered)
- [References](#references)
- [Notes](#notes)
  - [Why This Model Works](#why-this-model-works)
  - [Resource Types Under the Undo Manager](#resource-types-under-the-undo-manager)

---

## Status

Accepted

## Context

Ooloi's architecture presents unique challenges for implementing undo/redo functionality across a collaborative, distributed system. The system must coordinate undo/redo operations across multiple concerns:

1. **Musical content changes** (notes, rhythms, attachments) stored on the backend
2. **UI state changes** (themes, layouts, zoom levels) on individual frontend clients  
3. **Application settings** (server configuration, user preferences) with unclear scope and location

The architectural constraints established by previous ADRs create specific requirements:

- **ADR-0001**: Frontend-backend separation with two deployment models (combined desktop application, backend-only server)
- **ADR-0002**: gRPC streaming for real-time collaboration between frontend clients and backend server
- **ADR-0004**: STM-based concurrency for coordinated updates to musical structures
- **ADR-0009**: Multi-user collaborative editing with conflict resolution

### Key Architectural Realities

**Backend (Server):**
- Stores authoritative piece data using STM refs (single authority — ADR-0040)
- Handles all piece modifications with automatic conflict resolution
- Broadcasts lightweight staleness notifications via gRPC event streaming
- Does NOT store UI preferences or client-specific state

**Frontend (Clients):**
- Receives staleness notifications via gRPC streaming
- Fetches piece data on demand and maintains local non-authoritative caches
- Manages local UI state (themes, panel layouts, zoom levels)
- Handles UI interactions and forwards piece modifications to backend

**Collaborative Context:**
- Multiple frontend clients can edit the same piece simultaneously
- Backend coordinates all piece changes using STM transactions
- Frontend clients see real-time updates via the invalidate→fetch cycle
- Piece modifications must be coordinated; UI changes are client-local

## Decision

We will implement a **three-tier undo/redo architecture** that separates concerns according to the system's architectural boundaries:

### Tier 1: Backend Undo/Redo (Coordinated)
- **Scope**: All backend-managed resources — pieces (musical content), Instrument Library, future catalogues
- **Implementation**: Resource-agnostic Undo Manager component; closures over immutable state
- **Coordination**: One undo/redo stack per resource (per piece, one for IL), shared across all clients subscribed to that resource
- **Storage**: Backend server maintains per-resource undo/redo history in the Undo Manager component
- **Distribution**: Lightweight `:undo-state-changed` notifications (timestamps only) broadcast to subscribed clients via gRPC streaming; descriptions fetched lazily on demand

### Tier 2: Frontend UI Undo/Redo (Local)
- **Scope**: Client-specific UI state (themes, panel arrangements, zoom levels, selection state)
- **Implementation**: Simple local undo/redo stack per frontend client
- **Coordination**: No coordination needed - purely local to each client
- **Storage**: Frontend client memory (not persisted)
- **Distribution**: Not distributed - each client manages its own UI history

### Tier 3: No Backend Application Settings Component
- **Decision**: Backend will NOT implement an application settings component
- **Rationale**: Analysis reveals no legitimate backend application settings that require undo/redo
- **Configuration**: Server deployment configuration handled via environment variables and config files
- **User Preferences**: Stored and managed entirely by frontend clients

### Unified User Experience
- **Single Undo/Redo Interface**: Frontend presents one undo/redo button to users
- **Timestamp-Based Routing**: Frontend compares local stack timestamps with backend notification timestamps; the most recent action wins
- **Always Truthful Menu**: The undo/redo menu item always shows what will actually happen — never a stale prediction
- **Descriptive Messages**: Undo/redo operations display clear descriptions like "Undo Add Note", "Redo Change Key Signature" rather than generic "Undo"/"Redo"

## Rationale

### STM Provides Natural Collaborative Undo/Redo
Clojure's STM system already provides the foundation for coordinated undo/redo:
- **Atomic Operations**: All piece modifications occur within `dosync` transactions
- **Conflict Resolution**: Automatic retry mechanism handles concurrent modifications
- **Consistency**: Either all changes in a transaction apply, or none do
- **Scalability**: Designed for high-performance concurrent operations (100,000+ transactions/second)

### Clear Separation Prevents Architectural Complexity
Attempting to unify undo/redo across different architectural tiers would require:
- Complex event coordination systems layered on top of STM
- Artificial coupling between UI state and musical content
- Global state management breaking STM paradigm
- Synchronization complexity across network boundaries

### Backend Settings Analysis Reveals Minimal Needs
Examination of potential backend application settings shows:
- **Server configuration** (ports, TLS, capacity limits) → Environment variables/config files
- **User session preferences** → Frontend client storage
- **Piece-specific settings** → Part of piece data, not separate settings
- **Musical algorithm parameters** (like `MAX_LOOKAHEAD_MEASURES`) → Code constants, not user settings

### Frontend-Backend Boundary Alignment
The three-tier approach aligns perfectly with existing architectural boundaries:
- **Backend responsibility**: Musical data integrity and coordination
- **Frontend responsibility**: User interface and local preferences  
- **gRPC boundary**: Clean separation maintained without artificial unification

## Implementation Approach

### Tier 2: Frontend Implementation

The Tier 2 undo/redo module (`ooloi.frontend.undo_redo`) owns two module-level atoms — an
undo stack and a redo stack — each holding a vector of stack entries. Stack entries mirror
the `:setting-changed` event payload: `{:key, :old-value, :new-value, :timestamp}`. The stack is
capped at 50 entries; oldest entries are dropped when the cap is reached.

The module subscribes to the `:app-settings` event bus category via `wire-undo-redo!`. On
each `:setting-changed` event, it calls `record-setting-change!` — unless the event carries
`:sender :undo-redo`, which marks an undo/redo-triggered replay and must not push to the
stack (feedback loop prevention).

`undo!` pops the top entry from the undo stack, calls `set-app-setting!` with the
`:old-value` and `{:sender :undo-redo}`, and pushes the entry onto the redo stack. `redo!`
does the inverse. Both are no-ops on an empty stack.

The undo and redo menu items use `text-key` functions that derive the setting name from the
top stack entry: `:ui/theme` → `:setting.ui.theme.name` via the convention
`(keyword (str "setting." (namespace k) "." (name k) ".name"))`. When the stack is empty,
the item falls back to the static `:menu.edit.undo` / `:menu.edit.redo` key. Both items are
enabled only when their respective stacks are non-empty.

The menu bar is built imperatively once via `build-menu-item!`. Each menu item stores its
`enabled?` predicate in the item's JavaFX `::dynamic-enabled?` property at build time.
`refresh-dynamic-items!` in `menus.clj` then updates both text (via `::dynamic-text-key`)
and enabled state (via `::dynamic-enabled?`) on every refresh call. The `enabled?` predicates
for undo and redo read directly from the atoms — `(fn [_] (undo-redo/can-undo?))` and
`(fn [_] (undo-redo/can-redo?))` — ignoring the state parameter, exactly as `text-key` does.
This means the correct enabled state is produced on every refresh regardless of which caller
triggered it (undo/redo stack change, locale change, theme change).

`wire-undo-redo!` accepts an optional `on-change` callback. In `start-app!` this is wired
as `(fn [] (fx/run-later! #(um/refresh-menu-text! mgr)))`. After every stack mutation,
`undo!`, `redo!`, and `record-setting-change!` call this callback, which queues a menu
refresh on the JavaFX Application Thread.

Action handlers `:ui/undo` and `:ui/redo` are registered in `system.clj` and delegate to
`undo-redo/undo!` and `undo-redo/redo!`. The stacks are not persisted — they reset on
application restart.

**Note**: The Tier 2 implementation above predates Tier 1. When Tier 1 is implemented, the
Tier 2 local undo/redo stacks remain as-is, but routing is extended to compare local stack
timestamps with the backend timestamp cache (see
[Unified Frontend Routing](#unified-frontend-routing--tier-1-and-tier-2-merge)). The Tier 2
code above documents the current implementation; the timestamp-based routing is the target
architecture.

```clojure
;; ooloi.frontend.undo_redo — module structure

(def ^:private max-stack-depth 50)
(def ^:private undo-stack (atom []))
(def ^:private redo-stack (atom []))
(def ^:private on-change-callback (atom nil))

(defn can-undo? [] (boolean (seq @undo-stack)))
(defn can-redo? [] (boolean (seq @redo-stack)))
(defn top-undo-key [] (:key (peek @undo-stack)))
(defn top-redo-key [] (:key (peek @redo-stack)))

(defn record-setting-change! [{:keys [key old-value new-value timestamp]}]
  (swap! undo-stack #(vec (take-last max-stack-depth (conj % {:key key :old-value old-value :new-value new-value :timestamp timestamp}))))
  (reset! redo-stack [])
  (when-let [cb @on-change-callback] (cb)))

(defn undo! []
  (when-let [entry (peek @undo-stack)]
    (swap! undo-stack pop)
    (swap! redo-stack conj entry)
    (settings/set-app-setting! (:key entry) (:old-value entry) {:sender :undo-redo})
    (when-let [cb @on-change-callback] (cb))))

(defn redo! []
  (when-let [entry (peek @redo-stack)]
    (swap! redo-stack pop)
    (swap! undo-stack conj entry)
    (settings/set-app-setting! (:key entry) (:new-value entry) {:sender :undo-redo})
    (when-let [cb @on-change-callback] (cb))))

(defn wire-undo-redo!
  ([event-bus] (wire-undo-redo! event-bus nil))
  ([event-bus on-change]
   (reset! on-change-callback on-change)
   (eb/subscribe! event-bus :app-settings
     (fn [events]
       (doseq [{:keys [type sender] :as event} events]
         (when (and (= type :setting-changed)
                    (not= sender :undo-redo))
           (record-setting-change! event)))))))
```

### Tier 1: Backend Implementation — The Undo Manager

Tier 1 covers all backend-managed resources: pieces (STM-based), the Instrument Library
(atom-based), and any future singleton catalogues. A single backend Integrant component —
the **Undo Manager** — provides undo/redo for all of them through a uniform, resource-agnostic
API.

#### Design Principle: Closures Over Immutable State

The undo manager stores pairs of zero-argument closures alongside descriptive metadata.
It knows nothing about what it is undoing. Each closure captures immutable references to the
resource state before and after the mutation. Clojure's persistent data structures mean both
snapshots share structure — holding both is cheap, with no copying, no cloning, no delta
computation.

This is the core elegance of the model: the undo manager is pure infrastructure. After it
is implemented, making any mutation undoable requires a single function call at the mutation
site. The undo-fn and redo-fn are closures that restore immutable state snapshots. For
atom-based resources (IL), the closure calls `reset!`. For STM-based resources (pieces), the
closure calls `ref-set` inside `dosync`. The undo manager does not distinguish between these
cases — the closure abstracts the storage mechanism.

**ADR-0040 boundary**: Snapshot restoration is an implementation detail of the undo
manager's accepted `undo!` and `redo!` operations. It is not a second authority path and
not an implicit `set-piece`. The undo manager is a backend component operating within the
single-authority boundary — the backend remains the sole authority over resource state.
No client can supply a snapshot or trigger a restoration outside the undo manager's API.

#### The Undo Manager API

```clojure
;; backend/components/undo_manager.clj — Integrant component
;;
;; Internal state: atom holding
;;   {resource-key → {:undo-stack [entry ...] :redo-stack [entry ...]}}
;;
;; Each entry:
;;   {:id              (UUID)
;;    :description-key :il.undo/delete-instruments   ;; translation key
;;    :description-params {:names "Flute, Oboe"}     ;; interpolation params
;;    :timestamp       1711023456789012               ;; epoch microseconds
;;    :undo-fn         (fn [] ...)                    ;; restores previous state
;;    :redo-fn         (fn [] ...)}                   ;; re-applies mutation

(push-undo! undo-mgr resource-key description-key description-params undo-fn redo-fn)
;; Pushes an undo entry. Clears the redo stack for this resource (standard undo semantics:
;; a new forward mutation invalidates any redo history). Broadcasts :undo-state-changed.

(undo! undo-mgr resource-key)
;; Pops the top undo entry, calls its undo-fn, pushes the entry to the redo stack.
;; Broadcasts :undo-state-changed. Returns the entry, or nil if the stack was empty.

(redo! undo-mgr resource-key)
;; Pops the top redo entry, calls its redo-fn, pushes the entry to the undo stack.
;; Broadcasts :undo-state-changed. Returns the entry, or nil if the stack was empty.

(current-state undo-mgr resource-key)
;; Returns {:undo {:description-key ... :description-params ... :timestamp ...}
;;          :redo {:description-key ... :description-params ... :timestamp ...}}
;; Either or both may be nil when the respective stack is empty.

(remove-resource! undo-mgr resource-key)
;; Clears both stacks for a resource. Called by the Piece Manager when
;; no clients have the piece open anymore. The IL stack persists for the
;; session (the IL is always available).
```

The `resource-key` is `:instrument-library` for the IL, a piece UUID for pieces, and
any future keyword or UUID for other resources. The undo manager imposes no constraints
on the key type.

#### Stack Semantics

- **Depth cap**: 50 entries per resource (matching Tier 2). Oldest entries are silently
  dropped when the cap is exceeded.
- **Redo invalidation**: `push-undo!` clears the redo stack for that resource. If the user
  undoes A→B→C back to state A, then performs a new mutation D, the redo history (B, C) is
  gone. This is standard undo/redo behaviour in every editor.
- **Session-scoped**: Undo history lives in memory. Closures cannot survive application
  restart. All stacks are empty on startup. This is not a limitation — it matches user
  expectations for undo (nobody expects to undo across application restarts) and avoids
  the complexity of serialising arbitrary closures.
- **Resource removal**: `remove-resource!` clears stacks for a resource. Called by the
  Piece Manager when no clients have the piece open anymore — not when an individual
  client disconnects. The undo stack belongs to the resource, not to any client. The IL
  stack persists for the session because the IL is always available.

#### Mutation Sites — How Resources Register Undo Steps

Each mutation site calls `push-undo!` after a successful mutation. The pattern is identical
regardless of the resource type:

```clojure
;; Instrument Library — atom-based, optimistic locking
;; Inside set-instrument-library, after successful version-checked write:
(let [old-instruments (:instruments @il-atom)
      old-version     (:version @il-atom)]
  ;; ... apply mutation (swap! il-atom ...) ...
  (undo/push-undo! undo-mgr :instrument-library
    :il.undo/set-instruments {:count (count new-instruments)}
    (fn [] (restore-instruments! il-atom old-instruments old-version))
    (fn [] (restore-instruments! il-atom new-instruments (inc old-version)))))

;; Piece — STM-based, dosync transactions
;; Inside any piece mutation, after successful STM commit:
(let [before @piece-ref]
  (dosync (alter piece-ref apply-mutation args))
  (let [after @piece-ref]
    (undo/push-undo! undo-mgr piece-id
      description-key description-params          ;; e.g. :piece.undo/add-note {:pitch "C4" :measure 3}
      (fn [] (dosync (ref-set piece-ref before)))
      (fn [] (dosync (ref-set piece-ref after))))))
```

The `description-key` and `description-params` are translation keys for the menu display.
`(tr description-key description-params)` produces the user-facing string — e.g.
`(tr :piece.undo/add-note {:pitch "C4" :measure 3})` → `"Add Note (C4, m. 3)"`.
The undo manager stores these alongside the closures but never interprets them.

In both cases, `before` and `after` are immutable Clojure values captured by the closure.
They share structure via persistent data structures. The undo manager never inspects them,
never serialises them, never knows what they contain. It calls the closure; the closure
restores the state.

#### Push-Based State Notification

Every `push-undo!`, `undo!`, and `redo!` operation broadcasts an `:undo-state-changed`
notification through the existing gRPC event streaming infrastructure. This follows the
standard Ooloi invalidation pattern: a lightweight notification tells the client that its
cached state is stale; the client fetches details on demand.

The notification carries only timestamps — no descriptions, no params:

```clojure
{:type            :undo-state-changed
 :resource-key    :instrument-library    ;; or piece UUID
 :undo-timestamp  1711023456789012       ;; epoch microseconds, or nil
 :redo-timestamp  nil}                   ;; nil = redo unavailable
```

The client caches these timestamps per subscribed resource and marks the description as
stale. Descriptions are fetched lazily via `SRV/get-undo-description` only when the menu
needs to display them — see [Unified Frontend Routing](#unified-frontend-routing--tier-1-and-tier-2-merge).

The notification is routed via `derive-category` to a new `:undo` bus category.
Notifications are scoped by audience: IL notifications go to all connected clients (the IL
is a shared global resource); piece notifications go only to clients subscribed to that
piece. This follows the same scoping as other piece events (`:piece-structure-invalidated`).

After `undo!` or `redo!`, the undo manager broadcasts **two** events:
1. `:undo-state-changed` — so subscribed clients update their undo/redo menu state
2. The resource-specific event (`:instrument-library-changed` or
   `:piece-structure-invalidated`) — so all clients see the state change through
   the normal invalidate→fetch→replace pipeline

This dual broadcast means the undo operation is transparent to all existing event handlers.
A client that refetches on `:instrument-library-changed` will see the restored state without
knowing it was caused by an undo.

```
  Mutation (any client)
    │
    ▼
  ┌──────────────────────────────────────────────────────┐
  │  Mutation site                                        │
  │  1. Apply mutation to resource (atom/STM)             │
  │  2. push-undo!(undo-mgr, key, desc, undo-fn, redo-fn)│
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Undo Manager                                         │
  │  1. Push entry to undo stack                          │
  │  2. Clear redo stack for this resource                │
  │  3. Broadcast :undo-state-changed notification        │
  └──────────────────────┬───────────────────────────────┘
                         │ gRPC event stream
                         ▼
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  Client A   │  │  Client B   │  │  Client C   │
  │  (mutator)  │  │  (observer) │  │  (observer) │
  │             │  │             │  │             │
  │  Timestamp  │  │  Timestamp  │  │  Timestamp  │
  │  cache      │  │  cache      │  │  cache      │
  │  updated    │  │  updated    │  │  updated    │
  └─────────────┘  └─────────────┘  └─────────────┘
```

#### Undo Operation — Sequence Diagram

```mermaid
sequenceDiagram
    participant C as Client (any)
    participant G as gRPC Transport
    participant UM as Undo Manager
    participant R as Resource (IL/Piece)
    participant E as Event Stream

    C->>G: SRV/undo-resource(:instrument-library, nil)
    G->>UM: undo!(:instrument-library)
    UM->>UM: Pop top undo entry
    UM->>R: Call entry's undo-fn
    R->>R: Restore previous state (atom reset!/STM ref-set)
    UM->>UM: Push entry to redo stack
    UM-->>G: Response {undo-ts, redo-ts}
    G-->>C: Update timestamp cache immediately
    UM->>E: Broadcast :undo-state-changed {undo-ts, redo-ts}
    UM->>E: Broadcast :instrument-library-changed
    E->>C: :undo-state-changed (other clients update cache)
    E->>C: :instrument-library-changed (all clients refetch)
```

#### Collaborative Undo — Multi-Client Sequence

In a multi-user session, the undo stack is shared. Any client can undo the most recent
operation on a resource, regardless of which client performed it. This is deliberate: the
shared undo stack models a single shared editing context, not per-user history.

```mermaid
sequenceDiagram
    participant A as Client A
    participant B as Client B
    participant UM as Undo Manager
    participant IL as Instrument Library
    participant E as Event Stream

    Note over A,E: Client A deletes "Flute"
    A->>IL: set-instrument-library (without Flute)
    IL->>UM: push-undo!(:il, :il.undo/delete, undo-fn, redo-fn)
    UM->>E: :undo-state-changed {undo-ts: 100, redo-ts: nil}
    E->>A: Timestamp cache: undo-ts=100, stale
    E->>B: Timestamp cache: undo-ts=100, stale

    Note over A,E: Client B opens Edit menu → needs description
    B->>UM: SRV/get-undo-description(:instrument-library)
    UM-->>B: {:undo {:description-key :il.undo/delete, :description-params {:names "Flute"}}}
    Note over B: Menu shows "Undo: Delete Instruments (Flute)"

    Note over A,E: Client B undoes Client A's deletion
    B->>UM: SRV/undo-resource(:instrument-library, nil)
    UM->>IL: undo-fn() → Flute restored
    UM-->>B: Response {undo-ts: nil, redo-ts: 100}
    UM->>E: :undo-state-changed {undo-ts: nil, redo-ts: 100}
    UM->>E: :instrument-library-changed
    E->>A: Timestamp cache: undo-ts=nil, redo-ts=100, stale
    E->>B: Refetch IL → sees Flute restored
    E->>A: Refetch IL → sees Flute restored
```

### gRPC Extensions

Three new API operations, declared `^{:api true}` in `interfaces.clj`:

```clojure
(undo-resource resource-type resource-id)
;; Undoes the most recent mutation on the specified resource.
;; Returns: {:ok true
;;           :undo-timestamp ...   ;; new top of undo stack (or nil)
;;           :redo-timestamp ...}  ;; new top of redo stack
;;      or: {:empty true}          ;; nothing to undo

(redo-resource resource-type resource-id)
;; Re-applies the most recently undone mutation on the specified resource.
;; Returns: {:ok true
;;           :undo-timestamp ...
;;           :redo-timestamp ...}
;;      or: {:empty true}          ;; nothing to redo

(get-undo-description resource-type resource-id)
;; Returns the descriptions for the current top undo and redo entries.
;; Returns: {:undo {:description-key ... :description-params ...}  ;; or nil
;;           :redo {:description-key ... :description-params ...}} ;; or nil
```

These are **resource-scoped**, not "undo the latest across everything." The client
knows from its cached `:undo-state-changed` timestamps which resource has the most recent
step. It sends `SRV/undo-resource(:instrument-library, nil)` or
`SRV/undo-resource(:piece, piece-id)`.

The `undo-resource` and `redo-resource` responses include the new undo/redo timestamps,
allowing the calling client to update its cache immediately without waiting for the push
notification. The push notification is for other clients.

`get-undo-description` is the lazy fetch call: the client calls it only when the menu
needs to display a backend undo/redo entry whose description has not yet been fetched.
This follows the standard Ooloi invalidate→fetch pattern — the notification says *what
changed*; the fetch says *what it looks like*.

### Unified Frontend Routing — Tier 1 and Tier 2 Merge

The frontend presents a single Undo/Redo interface to the user. Routing uses **timestamp
comparison** across separate data structures — the local undo/redo stacks and a per-resource
backend timestamp cache — to determine whether to undo locally or via the backend. The
entry with the most recent timestamp is what Cmd+Z will undo. The menu always shows what
will actually happen.

#### Frontend Undo State — Separate Stacks

The frontend maintains three data structures:

- **Local undo stack** — Tier 2 entries only (UI settings changes). Identical to the
  existing implementation: `{:key, :old-value, :new-value, :timestamp}`.
- **Local redo stack** — entries popped from the local undo stack during local undo.
- **Backend timestamp cache** — per subscribed resource, stores timestamps from the most
  recent `:undo-state-changed` notification and a stale flag for lazy description fetching:

```clojure
;; Backend timestamp cache — updated by :undo-state-changed notifications
{:instrument-library {:undo-timestamp 1711023456789012
                      :redo-timestamp nil
                      :undo-description nil     ;; fetched lazily
                      :redo-description nil
                      :stale true}

 #uuid "piece-1..."  {:undo-timestamp 1711023400000000
                      :redo-timestamp 1711023350000000
                      :undo-description nil
                      :redo-description nil
                      :stale true}}
```

The local stacks are always complete and current — they record the user's own local
actions. The backend cache is updated every time an `:undo-state-changed` notification
arrives (setting `:stale true` and clearing cached descriptions). Descriptions are fetched
via `SRV/get-undo-description` only when the menu needs to display a backend entry.

#### Routing Algorithm

**Undo (Cmd+Z):** Compare the local undo stack's top timestamp with the highest
`:undo-timestamp` across all entries in the backend cache. The most recent timestamp wins:

```
  Cmd+Z pressed
    │
    ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Compare timestamps:                                          │
  │    local undo top:  :ui/zoom, timestamp 300                   │
  │    backend cache:   :instrument-library, undo-timestamp 250   │
  │                     #uuid "piece-1", undo-timestamp 180       │
  │                                                               │
  │  max(300, 250, 180) = 300 → local wins                        │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                            local wins
                                 │
                                 ▼
                  set-app-setting!(:ui/zoom, old-value, {:sender :undo-redo})
                  Pop local undo → push to local redo
                  refresh-dynamic-items!
```

When a backend resource wins:

```
  Cmd+Z pressed
    │
    ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Compare timestamps:                                          │
  │    local undo top:  :ui/theme, timestamp 100                  │
  │    backend cache:   :instrument-library, undo-timestamp 250   │
  │                     #uuid "piece-1", undo-timestamp nil       │
  │                                                               │
  │  max(100, 250) = 250 → backend (:instrument-library) wins     │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                            backend wins
                                 │
                                 ▼
                  SRV/undo-resource(:instrument-library, nil)
                                 │
                                 ▼
                  Response: {:ok true
                             :undo-timestamp 1711023400000000
                             :redo-timestamp 1711023456789012}
                  → update backend cache immediately
                                 │
                                 ▼
                  :undo-state-changed notification → other clients
                  :instrument-library-changed → all clients refetch
```

**Redo (Cmd+Shift+Z):** Compare the local redo stack's top timestamp with the lowest
`:redo-timestamp` across all entries in the backend cache. The **oldest** timestamp wins —
this reverses the undo order exactly:

```
Undo order:  300 (local), 250 (backend), 100 (local)  — highest first
Redo order:  100 (local), 250 (backend), 300 (local)  — lowest first
```

**Menu text:** The menu always displays the description of the winning entry — the one
that Cmd+Z (or Cmd+Shift+Z) would act on. For local entries, the description is derived
from the setting key (existing convention). For backend entries, the description is fetched
lazily via `SRV/get-undo-description` and cached until the next `:undo-state-changed`
notification marks it stale. When no undo is available (local stack empty and all backend
undo timestamps are nil), the item shows the static `:menu.edit.undo` key and is disabled.

#### Single-User Guarantee

In combined desktop mode (one frontend, one backend, in-process), there is exactly one
client. Every `:undo-state-changed` notification corresponds to the user's own action. The
backend timestamps and local timestamps interleave perfectly. Undo/redo behaves identically
to any desktop application:

- Every action is undoable in reverse chronological order
- Redo is always available after undo (until a new forward mutation)
- Description fetches are in-process (zero network latency)
- No limitations, no surprises

This is guaranteed by the architecture. No special-case code is needed.

#### Multi-User Properties

In collaborative mode, the shared per-resource backend stacks introduce inherent
limitations. These are properties of shared undo stacks — the price of determinism and
consistency:

1. **The menu always tells the truth.** When the `:undo-state-changed` notification arrives
   (whether from the user's own action or another client's), the backend cache updates. The
   menu shows whichever entry has the highest timestamp — always what Cmd+Z will actually do.
   The user is never surprised.

2. **Other clients' actions appear in the undo chain.** When Client B mutates a resource,
   Client A receives the notification with a timestamp that may be higher than Client A's
   local top. If so, Client B's action becomes what Cmd+Z would undo. The menu shows Client
   B's action description (fetched lazily). Client A sees this before pressing Cmd+Z.

3. **Redo is fragile in collaborative mode.** Any mutation by any client on a shared
   resource calls `push-undo!`, which clears the redo stack for that resource. The
   notification arrives with `:redo-timestamp nil`. The menu reflects this immediately. In
   practice, redo is only reliable when the user is the sole active editor of a resource.

4. **Buried actions are unreachable.** The user cannot skip past other clients' actions to
   undo their own earlier action on the same resource. They must undo from the top of the
   shared stack downward. The menu makes this transparent — it always shows what the next
   undo will actually do.

#### Undo Routing — Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend (undo_redo module)
    participant LC as Local Stacks
    participant BC as Backend Cache
    participant G as gRPC Transport
    participant UM as Undo Manager

    Note over FE,BC: Notification arrives (from any client's mutation)
    UM-->>BC: :undo-state-changed {undo-timestamp, redo-timestamp}
    BC->>BC: Cache timestamps, mark stale

    Note over U,UM: User presses Cmd+Z
    U->>FE: Cmd+Z
    FE->>LC: Get local undo top timestamp
    FE->>BC: Get max backend undo timestamp

    alt local timestamp > backend timestamp
        FE->>LC: Pop undo → push to redo
        FE->>FE: set-app-setting!(key, old-value, {:sender :undo-redo})
    else backend timestamp > local timestamp
        opt description stale
            FE->>G: SRV/get-undo-description(resource-key)
            G-->>FE: {:description-key ... :description-params ...}
            FE->>BC: Cache description, clear stale
        end
        FE->>G: SRV/undo-resource(resource-type, resource-id)
        G->>UM: undo!(resource-key)
        UM-->>G: Response with new timestamps
        G-->>FE: Update backend cache immediately
        UM-->>FE: :undo-state-changed (for other clients)
        UM-->>FE: :resource-changed (all clients refetch)
    end

    FE->>FE: refresh-dynamic-items! (menu text + enabled state)
```

#### Description Localisation

Undo description keys are translation keys stored in the undo manager alongside the
closures. The push notification does not carry them — descriptions are fetched lazily via
`SRV/get-undo-description` when the menu needs to display a backend entry. The
`:description-params` map allows interpolation via the `%{param}` convention established
in ADR-0039.

**Threading rule**: The `SRV/get-undo-description` call is a gRPC operation and must not
run on the JavaFX Application Thread. The menu refresh dispatches the fetch on a pool
thread; the result is delivered back to the JAT via `fx/run-later!` for cache update and
menu text assignment. This follows the standard UI architecture invariant: no blocking
or network calls on the JAT.

```clojure
;; Client fetches description for the winning backend resource:
(SRV/get-undo-description :instrument-library)
;; → {:undo {:description-key    :il.undo/delete-instruments
;;           :description-params {:names "Flute, Oboe"}}
;;    :redo nil}

;; Client resolves via tr:
(tr :il.undo/delete-instruments {:names "Flute, Oboe"})
;; → "Delete Instruments"  (or locale-specific equivalent)

;; Menu item displays:
"Undo: Delete Instruments"
```

The fetched description is cached in the backend timestamp cache until the next
`:undo-state-changed` notification for that resource marks it stale. For local entries,
the existing setting-name convention applies (`:ui/theme` → `:setting.ui.theme.name`).
When no undo is available, the item falls back to the static `:menu.edit.undo` key.

#### Clock Skew Mitigation

The routing algorithm compares timestamps from two different clocks: the client's local
clock (Tier 2 entries stamped with `time/epoch-usec` on the client machine) and the server's
clock (Tier 1 entries stamped with `time/epoch-usec` on the server machine). If these clocks
disagree, the comparison is biased — a server clock that runs ahead makes backend actions
appear more recent than they are, and vice versa.

The mitigation computes a per-connection clock offset at registration time and applies it
to all subsequent timestamp comparisons, normalising backend timestamps to the client's
clock frame. This eliminates clock skew as a routing factor, reducing the residual error
to the one-way network delay — which is negligible for all practical deployments.

**Offset computation.** The server includes its current `epoch-usec` timestamp in the
`:client-registration-confirmed` event as a `:server-timestamp` field. The client records
its own `epoch-usec` at the moment it receives the confirmation and computes:

```
offset = server-timestamp - client-timestamp
```

This offset is stored as an immutable per-connection value on the gRPC client component.
It is computed once and never updated — clock drift during a single editing session is
negligible.

**Offset application.** The routing algorithm adjusts all backend timestamps before
comparing them with local timestamps:

```
adjusted-backend-ts = backend-ts - offset
```

This converts server time to the client's clock frame. The comparison then uses
`adjusted-backend-ts` in place of the raw `backend-ts` everywhere: undo routing (highest
timestamp wins), redo routing (lowest timestamp wins), and menu display (which entry is
the winning candidate).

**Residual error.** The offset includes the one-way network delay: `offset = skew - delay`.
This means the adjustment slightly under-corrects by the one-way delay. The residual error
by transport type:

| Transport | One-way delay | Residual error |
|---|---|---|
| In-process (combined desktop app) | ~0 | ~0 |
| LAN / same machine | < 1ms | < 1ms |
| WAN (remote server) | 5–50ms | 5–50ms |

Human actions are separated by hundreds of milliseconds to seconds. A 50ms residual error
in the worst case (WAN with high latency) cannot cause a wrong routing decision in practice.
For the combined desktop application — the primary deployment model — the offset and
residual are both effectively zero.

**Backward compatibility.** If the server does not include `:server-timestamp` in the
confirmation event (older server version), the offset defaults to zero. This is today's
behaviour — no clock adjustment. The feature degrades gracefully.

**Confirmation event extension:**

```clojure
;; Extended confirmation event (server side)
{:type             :client-registration-confirmed
 :client-id        "client-42"
 :message          "Registration successful"
 :server-timestamp 1711023456789012}    ;; epoch microseconds — NEW

;; Client side (on receiving confirmation):
(let [offset (if-let [server-ts (:server-timestamp event)]
               (- server-ts (time/epoch-usec))
               0)]                       ;; default: no adjustment
  (reset! clock-offset-atom offset))
```

**Why not rely on NTP?** NTP synchronisation is helpful but neither guaranteed nor
sufficient. Ooloi cannot assume or enforce NTP configuration on user machines — VMs,
containers, and misconfigured desktops can be minutes or even hours off. Without the
offset, a 5-minute clock skew means every Cmd+Z for 5 minutes routes to the wrong tier
(whichever clock is ahead always "wins"). This is not an edge case to tolerate — it is a
correctness failure that breaks the user experience silently. The connection-time offset
handles any skew magnitude, from 10ms residual after NTP to 5 minutes on an unsynchronised
machine, with no dependency on external infrastructure.

## Consequences

### Positive

1. **Resource-agnostic**: One undo manager handles all backend resources through a uniform four-function API. Adding undo to a new resource requires one `push-undo!` call at the mutation site — no infrastructure changes.
2. **Closures over immutable state**: Persistent data structures make before/after snapshots cheap. No serialisation, no delta computation, no deep copy.
3. **Invalidate→fetch pattern**: `:undo-state-changed` notifications follow the standard Ooloi pattern — lightweight timestamps via the existing gRPC event streaming pipeline; descriptions fetched lazily on demand. No polling, no new transport mechanism.
4. **Timestamp-based routing**: The frontend compares timestamps across local stacks and the backend cache. The highest timestamp wins for undo, the lowest for redo. No interleaved data structure to maintain — the correct ordering falls out of timestamp arithmetic.
5. **Clean architectural separation**: Backend owns all backend undo stacks and descriptions; frontend owns local undo entries and the routing logic. The backend cache is a frontend-side timestamp cache, not a violation of the frontend-backend boundary.
6. **Collaborative by default**: Shared per-resource stacks mean any client can undo the most recent operation. No per-user history complexity.
7. **Honest user experience**: Single Cmd+Z with descriptive labels. The menu always shows what will actually happen — never a stale prediction. In single-user mode, this is identical to any desktop application.
8. **Clock skew resilience**: Per-connection clock offset computed at registration time normalises server timestamps to the client's clock frame. Handles arbitrary clock differences (even minutes) with residual error bounded by one-way network delay. No dependency on NTP or external time synchronisation.

### Negative

1. **Collaborative undo limitations**: In multi-user sessions, the menu may show another client's action as the next undo target (because it has a more recent timestamp on the shared stack). Redo is frequently invalidated by other clients' mutations.
2. **Network dependency**: Backend undo/redo requires connectivity. When the backend is unreachable, only local undo entries are available.
3. **Session-scoped only**: Undo history is lost on application restart. Closures cannot be serialised.
4. **Lazy fetch latency**: The first time a backend undo entry needs to be displayed, a `get-undo-description` round trip is required. Subsequent displays use the cached description until marked stale.

### Mitigations

1. **Collaborative limitations are transparent**: The menu always shows the true current state — the user sees "Undo: [another client's action]" before pressing Cmd+Z and can choose not to. Redo unavailability is immediately visible (menu item disabled). These are inherent properties of shared undo stacks, not bugs — the price of determinism and consistency.
2. **Offline**: Local entries are undoable offline. When the backend is unreachable and the backend cache has the highest timestamp, the undo item can show the backend entry as disabled (greyed) while local entries remain available.
3. **Session scope**: Matches user expectations — no editor provides cross-restart undo. The depth cap (50 entries per resource) bounds memory usage.
4. **Fetch latency**: The `get-undo-description` call returns only two small maps (description key + params). In combined mode, this is in-process with zero network latency. Over gRPC, it is sub-millisecond. The fetched description is cached until the next notification, so repeated menu displays do not re-fetch.

## Alternatives Considered

### Alternative 1: Unified Global Undo/Redo
**Approach**: Single undo/redo stack handling all changes (UI + piece content)
**Rejection Reasons**:
- Requires complex event coordination on top of STM
- Breaks architectural separation established by ADR-0001
- Introduces artificial coupling between UI state and musical content
- Performance overhead of coordinating UI changes across network

### Alternative 2: Piece-Only Settings (Everything in Piece Data)
**Approach**: Store all "settings" within individual pieces
**Rejection Reasons**:
- Confuses piece content with application configuration
- No settings actually belong to pieces vs UI preferences vs deployment config
- Would duplicate UI preferences across all pieces unnecessarily

### Alternative 3: Event Sourcing on Top of STM
**Approach**: Add event sourcing layer for unified undo/redo
**Rejection Reasons**:
- Adds complexity without benefits (STM already provides event coordination)
- Duplicates STM's built-in transaction and retry mechanisms
- Performance overhead of additional abstraction layer

### Alternative 4: Backend Application Settings Component
**Approach**: Implement settings component for backend configuration
**Rejection Reasons**:
- Analysis reveals no legitimate backend settings requiring undo/redo
- Server configuration belongs in deployment config, not application state
- User preferences belong in frontend, not backend
- Would create artificial coordination complexity

## References

### Related ADRs
- [ADR-0000: Clojure](0000-Clojure.md) - Persistent data structures enabling immutable state references in undo closures
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural boundaries defining undo/redo scope separation
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Event streaming for `:undo-state-changed` push notifications
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model for piece undo/redo closures
- [ADR-0009: Collaboration](0009-Collaboration.md) - Collaborative editing context requiring shared undo stacks
- [ADR-0016: Settings](0016-Settings.md) - Settings architecture implementing piece-specific configuration identified in this analysis
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Event bus routing `:undo-state-changed` to frontend cache; `:app-settings` category for Tier 2
- [ADR-0039: Localisation Architecture](0039-Localisation-Architecture.md) - Translation key resolution for undo description display
- [ADR-0043: Frontend Settings](0043-Frontend-Settings.md) - `set-app-setting!` and the `:setting-changed` event that Tier 2 accumulates
- [ADR-0045: Instrument Library](0045-Instrument-Library.md) - First consumer of Tier 1 undo manager

## Notes

This architecture decision removes the "Application Settings" component from the backend
development queue. The analysis demonstrates that no legitimate backend application settings
exist that would require the complexity of a dedicated component.

However, the analysis identified that **piece-specific settings** (configuration attributes
that travel with piece data) do have legitimate use cases. The implementation of this
piece-specific settings architecture is detailed in [ADR-0016: Settings](0016-Settings.md),
which provides a comprehensive solution for configuration attributes across all musical and
visual entities within pieces.

### Why This Model Works

The elegance of this distributed undo model rests on three properties of Clojure:

1. **Persistent data structures**: The "before" and "after" states of any mutation are both
   immutable values that share structure. Storing both in a closure costs almost nothing.
   There is no serialisation, no deep copy, no diff/patch — just two references to values
   that already exist in memory.

2. **Closures as first-class values**: The undo-fn and redo-fn capture everything needed to
   reverse or re-apply a mutation. The undo manager stores and invokes them without knowing
   what they do. This makes the manager truly resource-agnostic — it works for atoms, STM
   refs, or any future storage mechanism.

3. **Event-driven architecture**: The invalidate→fetch notification model follows the
   standard Ooloi pattern. Lightweight `:undo-state-changed` notifications (timestamps
   only) flow through the existing gRPC event streaming pipeline. Descriptions are fetched
   lazily when the menu needs them. No polling, no new transport mechanism.

The result is a distributed, collaborative, multi-resource undo/redo system implemented as
a single small component with a four-function API.

### Resource Types Under the Undo Manager

The undo manager is resource-agnostic — it stores closures and metadata without knowing what
kind of resource it manages. The distinction between atom-based and STM-based resources is
hidden inside the closures:

| Resource | Concurrency model | Undo-fn mechanism | Resource key |
|---|---|---|---|
| Instrument Library | Atom + optimistic locking | `reset!` on IL atom | `:instrument-library` |
| Pieces | STM refs | `ref-set` inside `dosync` | Piece UUID |
| Future catalogues (clefs, articulations) | Atom + optimistic locking | `reset!` on catalogue atom | Catalogue keyword |

This is not multiple variants of Tier 1 — it is one mechanism. The undo manager provides the
same API, the same event broadcasting, the same frontend routing regardless of the underlying
concurrency model. The closure abstraction makes the storage mechanism invisible.