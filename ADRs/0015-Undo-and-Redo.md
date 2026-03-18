# ADR-0015: Undo and Redo Architecture in Collaborative Environment

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
- Stores authoritative piece data using STM refs
- Handles all piece modifications with automatic conflict resolution
- Streams piece updates to connected frontend clients via gRPC
- Does NOT store UI preferences or client-specific state

**Frontend (Clients):**
- Receives piece data via gRPC streaming (no local piece storage)
- Manages local UI state (themes, panel layouts, zoom levels)
- Displays piece content received from backend
- Handles UI interactions and forwards piece modifications to backend

**Collaborative Context:**
- Multiple frontend clients can edit the same piece simultaneously
- Backend coordinates all piece changes using STM transactions
- Frontend clients see real-time updates from other collaborators
- Piece modifications must be coordinated; UI changes are client-local

## Decision

We will implement a **three-tier undo/redo architecture** that separates concerns according to the system's architectural boundaries:

### Tier 1: Backend Piece Undo/Redo (Coordinated)
- **Scope**: Musical content changes (notes, rhythms, tempo, key signatures, attachments)
- **Implementation**: STM-based coordinated undo/redo using existing transactional infrastructure
- **Coordination**: Single authoritative undo/redo stack shared across all collaborating clients
- **Storage**: Backend server maintains the undo/redo history in STM refs
- **Distribution**: Undo/redo operations broadcast to all connected clients via gRPC streaming

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
- **Smart Coordination**: Frontend determines whether to undo local UI change or send undo request to backend
- **Contextual Behavior**: Undo behavior depends on what user action was most recent (UI change vs piece modification)
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

### Tier 1: Backend Implementation _(provisional — not yet implemented)_

Tier 1 covers piece undo/redo. The backend piece management infrastructure (STM-based piece
refs, piece-manager component) will be extended with per-piece undo/redo stacks when Tier 1
is implemented. The gRPC protocol will be extended with `UndoPieceChange` and
`RedoPieceChange` operations at that time.

The unified undo routing logic — routing to Tier 2 or Tier 1 based on which change was most
recent — will be designed when Tier 1 is implemented and the timestamp-based recency
comparison can be properly specified against the actual backend architecture.

### gRPC Extensions _(provisional — Tier 1)_

`UndoPieceChange` and `RedoPieceChange` RPCs will be added to the gRPC service definition
when Tier 1 is implemented. The exact message shapes will be specified at that time based on
the piece management architecture as it stands.

## Consequences

### Positive

1. **Leverages Existing STM Infrastructure**: No additional concurrency complexity required
2. **Clean Architectural Separation**: Maintains frontend/backend boundaries established by ADR-0001
3. **Optimal Performance**: STM provides automatic optimization for collaborative scenarios
4. **Simplified Development**: No artificial unification complexity
5. **Scalable Collaboration**: STM-based approach proven to handle 100,000+ concurrent operations
6. **Consistent User Experience**: Single undo interface despite underlying architectural separation
7. **Reduced Complexity**: Eliminates need for backend application settings component

### Negative

1. **Frontend Coordination Logic**: Frontend must intelligently route undo/redo operations
2. **Network Dependency**: Piece undo/redo requires backend connectivity
3. **State Synchronization**: Frontend must track what operations are local vs backend
4. **Testing Complexity**: Must test coordination across network boundaries

### Mitigations

1. **Clear State Tracking**: Frontend maintains clear distinction between local and remote operations
2. **Offline Resilience**: UI undo/redo works offline; piece undo/redo queued until reconnection
3. **Comprehensive Testing**: Integration tests covering collaborative undo/redo scenarios
4. **Developer Tools**: Clear debugging for undo/redo operation routing

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
- [ADR-0000: Clojure](0000-Clojure.md) - Language providing STM foundation for coordinated undo/redo
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural boundaries defining undo/redo scope separation
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Communication protocol for coordinating undo/redo across clients
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model enabling collaborative undo/redo
- [ADR-0009: Collaboration](0009-Collaboration.md) - Collaborative editing context requiring coordinated undo/redo
- [ADR-0016: Settings](0016-Settings.md) - Settings architecture implementing piece-specific configuration identified in this analysis
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Event bus providing the `:app-settings` category that Tier 2 subscribes to
- [ADR-0043: Frontend Settings](0043-Frontend-Settings.md) - `set-app-setting!` and the `:setting-changed` event that Tier 2 accumulates

### Technical Considerations
- STM performance characteristics (100,000+ transactions/second on modest hardware)
- gRPC streaming capabilities for real-time undo/redo coordination
- Frontend state management patterns for UI undo/redo
- Network resilience patterns for offline undo/redo operations

## Notes

This architecture decision removes the "Application Settings" component from the backend development queue. The analysis demonstrates that no legitimate backend application settings exist that would require the complexity of a dedicated component.

However, the analysis identified that **piece-specific settings** (configuration attributes that travel with piece data) do have legitimate use cases. The implementation of this piece-specific settings architecture is detailed in [ADR-0016: Settings](0016-Settings.md), which provides a comprehensive solution for configuration attributes across all musical and visual entities within pieces.

The three-tier approach provides clean separation while maintaining the user experience expectation of unified undo/redo. The STM-based backend tier leverages Clojure's proven concurrency model, while the frontend tier handles local concerns without artificial coordination complexity.

Future enhancements could include:
- Persistent undo/redo history across application restarts
- Advanced undo/redo visualization showing collaborative change attribution
- Selective undo/redo for specific types of changes
- Integration with version control systems for long-term history management

The implementation should monitor performance characteristics in collaborative scenarios and user experience feedback to validate the three-tier approach effectiveness.

### Tier 1 Variant: Atom-Based Editor Windows

Not all backend-managed undo/redo falls under the STM-based Tier 1 mechanism. Editor windows for
singleton backend resources — the Instrument Library ([ADR-0045](0045-Instrument-Library.md)), and
potentially future editors for clefs, articulations, and similar catalogues — use atom-based
optimistic locking rather than STM refs. These resources have no need for multi-ref coordination
(`dosync`); their concurrency model is a single atom with a version counter and compare-and-swap.

These resources will have their own lightweight undo/redo stacks on the backend, following the same
version-based concurrency model as the resource itself. The frontend interaction is identical to
Tier 1: the frontend requests undo/redo from the backend and never maintains its own stack. The
backend implementation uses atom CAS rather than `dosync`. This is a variant of Tier 1, not a new
tier.