# ADR-0042: UI Specification Format

**Status:** Accepted

**Date:** 2026-01-27

## Table of Contents
- [Context](#context)
  - [Problem Statement](#problem-statement)
  - [Requirements](#requirements)
  - [Existing Patterns](#existing-patterns)
- [Decision](#decision)
  - [Approach](#approach)
  - [Metadata Keys](#metadata-keys)
  - [Content Builder Pattern](#content-builder-pattern-updated-2026-02-19)
  - [Custom cljfx Component Functions](#custom-cljfx-component-functions-updated-2026-02-19)
  - [Per-Window Reactive Renderer](#per-window-reactive-renderer-updated-2026-02-19)
  - [Architectural Invariants](#architectural-invariants)
  - [Theme-Independent Styling](#theme-independent-styling)
  - [Command Descriptors (Menu Specialisation)](#command-descriptors-menu-specialisation)
  - [Validation Strategy](#validation-strategy)
  - [Window Manager Integration](#window-manager-integration)
- [Benefits](#benefits)
- [Trade-offs](#trade-offs)
- [Alternatives Considered](#alternatives-considered)
  - [Constructor Functions](#constructor-functions)
  - [Defrecords with Protocol Conversion](#defrecords-with-protocol-conversion)
- [Comparison to Musical Elements](#comparison-to-musical-elements)
- [Integration Points](#integration-points)
- [Consequences](#consequences)
- [Related ADRs](#related-adrs)

## Context

### Problem Statement

Ooloi needs a format for specifying UI elements (windows, notifications) that:

1. **Works across boundaries** - Backend and backend plugins must be able to define UI that frontend renders
2. **Supports lifecycle management** - Windows need automatic persistence, event handling, cleanup
3. **Enables plugin independence** - Backend plugins shouldn't need frontend-specific code
4. **Serializes over gRPC** - UI specs from backend/backend plugins must cross network boundary
5. **Integrates with cljfx** - Must work naturally with cljfx's declarative UI approach (backend and frontend plugins)
6. **Handles diverse types** - Persistent windows, auto-dismiss notifications
7. **Supports local plugins** - Frontend plugins can use direct callbacks without serialization

### Requirements

**Functional Requirements:**
- Window lifecycle management (create, track state, save on close, restore on reopen, cleanup)
- Type differentiation (persistent windows, notifications, custom components)
- Backend plugin support (define UI in backend/plugin code, serialize over gRPC, no frontend dependency)
- Frontend plugin support (define UI with direct cljfx, use callback functions or symbolic handlers)
- Event integration (symbolic handlers for serialization, direct functions for local plugins)

**Non-Functional Requirements:**
- Simplicity - obvious how to create UI
- Flexibility - new window types without framework changes
- Type Safety - errors caught early (compile or runtime)
- Debuggability - clear error messages, inspectable data
- Performance - no serialization overhead

### Existing Patterns

**Musical Elements** (Pitch, Rest, Measure):
- Defrecords implementing traits
- Constructor functions with clojure.spec validation
- Hash-consing for performance
- Serialize via protobuf conversion

**Events** (gRPC event system):
- Pure maps (NOT defrecords)
- Namespace-qualified keywords (`:type`, `:piece-id`, `:client-id`)
- Runtime validation via custom functions (`validate-event-structure`)
- Serialize directly over gRPC
- Example: `{:type :piece-invalidation :piece-id "123" :measures [1 2 3]}`

**Design Requirement:**
> "Backend and plugins (both backend and frontend) describe UI using **cljfx maps directly** - no translation layer needed."

**Plugin Context:**
- **Backend plugins**: Send cljfx maps over gRPC with symbolic event handlers (maps)
- **Frontend plugins**: Use cljfx maps locally with direct callback functions or symbolic handlers

Per ADR-0039, all user-facing strings use translation keys (`:window/title-key`, `:text-key`, etc.), resolved by frontend via `(tr key)` at render time.

## Decision

### Approach

UI specifications are **pure Clojure maps** conforming to cljfx structure, augmented with `:window/` namespace-qualified metadata for lifecycle management.

**Implementation Note (Updated 2026-01-28)**: Initial windowing system (Issue #5) does not implement backend-described dialogs. Backend plugins use piece settings (ADR-0016) with auto-generated UI via `SRV/get-settings-ui-metadata`. Frontend provides all custom windows for operations. Backend-described dialog capability (examples below) represents architectural design for future extensibility if needed.

```clojure
;; Backend plugin sends this over gRPC (symbolic event handlers)
;; NOTE: Future capability - not implemented in initial windowing system.
;; Custom ooloi components referenced by qualified symbol (resolved on the frontend).
{:window/id :plugin-config
 :window/persist? true
 :window/title-key :plugin.config.title  ; → "Plugin Configuration"
 :window/content {:fx/type :v-box
                  :children [{:fx/type :text-field
                              :prompt-text-key :plugin.config.api-key-prompt}  ; future: custom component resolves via tr
                             {:fx/type 'ooloi.frontend.ui.core.cljfx/ooloi-button
                              :text-key :common.save
                              :on-action {:event/type :plugin/save-settings}}]}}

;; Frontend plugin uses var directly (no gRPC boundary, no serialization)
;; and publishes a :window-open-requested event to :window-requests, or builds content nodes via cljfx
{:window/id :local-tool
 :window/title-key :tool.palette.title
 :window/content (cljfx/instance
                   (cljfx/create-component
                     {:fx/type :v-box
                      :children [{:fx/type ooloi-button
                                  :text-key :common.save
                                  :on-action (fn [e] (save-local-data!))}]}))}
```

### Metadata Keys

**`:window/id`** (keyword, required for persistent windows)
- Unique identifier for this window
- Used for state persistence lookup
- Example: `:main-editor`, `:plugin-config`, `:tool-palette`

**`:window/persist?`** (boolean, default true)
- Whether to save/restore window geometry (position, size) across sessions
- Persistent (default): main windows, tool palettes
- Non-persistent: splash screen (`:window/persist? false`), notifications (no Stage)

**`:window/type`** (keyword, optional)
- Reserved key — not currently in the recognised key set. Passing it to `show-window!`
  will throw "Unknown window/* keys". Documented for future use as a semantic type hint
  (e.g. `:stage`, `:dialog`, `:notification`).
- Not required — `:fx/type` is authoritative for rendering

**`:window/default-position`** (map, optional)
- Default position if no saved state: `{:x int :y int}`
- Used on first open or when saved state invalid

**`:window/default-size`** (map, optional)
- Default size if no saved state: `{:width int :height int}`

**`:window/default-scale`** (double, optional)
- Default zoom/scale factor: `1.0`, `1.5`, etc.

**`:window/style`** (keyword, optional)
- JavaFX StageStyle: `:decorated` (default), `:undecorated`, `:transparent`, `:utility`
- Example: `:undecorated` for splash screen and About window

**`:window/menu-bar`** (JavaFX MenuBar, optional)
- Pre-built MenuBar instance to attach to the window's VBox root
- On macOS with `:use-system-menu-bar true`, the bar integrates with the system menu

**`:window/content`** (JavaFX Node, optional)
- Content node to add to the window's VBox root with `VGrow/ALWAYS`
- Produced by content builder functions (see Content Builder Pattern below)
- When neither `:width` nor `:height` is specified, the Scene sizes itself from content

**`:window/title-key`** (keyword, optional)
- ADR-0039 translation key resolved via `tr` at render time for the window title
- Example: `:window/title-key :window.tool-palette.title` → title "Tools"
- When present, overrides any `:title` value in the spec

**`:window/modal?`** (boolean, optional)
- Reserved key — not currently in the recognised key set. Passing it to `show-window!`
  will throw "Unknown window/* keys".
- **Frontend modal dialogs** (e.g. confirmation dialogs) use cljfx's `:alert` type directly
  and do not use this key.
- **Backend-described modal windows** over gRPC: not implemented.
- Documented for future use when a generic modal dialog infrastructure exists.

**`:title`** (string, optional)
- Raw JavaFX Stage title string, passed directly to `Stage.setTitle()`
- This is the **low-level target** that `:window/title-key` resolves into — `build-window!` calls `(tr title-key)` and places the result in `:title`
- `:window/title-key` takes precedence: if both are present, the resolved translation replaces `:title`
- Spec authors must use `:window/title-key` for all titles — all user-facing strings must be localisable per ADR-0039
- No production code should pass hardcoded strings via `:title`; all window titles go through `tr-declare` + `:window/title-key`

### Content Builder Pattern (Updated 2026-02-19)

UI elements that need custom content — splash screens, About windows, piece windows, settings windows — follow the **content builder pattern**. A private content function constructs the content; the public `show-*!` function materialises it and publishes a `:window-open-requested` event to `:window-requests`. The UI Manager subscriber handles everything else: Stage creation, theming, registry, platform concerns.

```clojure
;; about.clj — private spec function returns a cljfx description map
(defn- about-content-spec [manager]
  ;; Returns a cljfx description map — materialised by show-about! via create-component/instance
  {:fx/type :stack-pane
   :children [... {:fx/type cljfx/ext-on-instance-lifecycle
                   :on-created (fn [btn]
                                 (.setOnAction btn
                                   (reify EventHandler
                                     (handle [_ _]
                                       (eb/publish! (:event-bus manager) :window-requests
                                         [{:type :window-close-requested :window/id :about}])))))
                   :desc (ofx/ooloi-ok-button {})}]})

(defn show-about! [manager dispatch-fn]
  (eb/publish! (:event-bus manager) :window-requests
    [{:type             :window-open-requested
      :window/id        :about
      :window/content   (cljfx/instance (cljfx/create-component (about-content-spec manager)))
      :window/title-key :window.about.title}]))

;; system.clj — one-line delegation
show-about-fn (fn [] (about/show-about! mgr dispatch-fn))
```

**The contract:**
- The content function is private (`defn-`). The public API is `show-*!`.
- For cljfx-based modules: a private `*-spec` function returns a cljfx description map;
  `show-*!` materialises it via `cljfx/create-component` + `cljfx/instance`.
- For modules not yet fully migrated: `build-*-content!` constructs JavaFX nodes directly
  and returns `{:root Node, ...}`.
- The content function never imports Stage, Scene, or StageStyle.
- The materialised Node is placed in `:window/content` in the `:window-open-requested` event.
- The UI Manager handles Stage creation, theming, registry tracking, and platform behaviour (e.g. `initOwner` on macOS).

This pattern ensures all windows go through the same path regardless of their content complexity.

### Custom cljfx Component Functions (Updated 2026-02-19)

cljfx supports **functions as `:fx/type` values**. Ooloi uses this mechanism to define a library of custom component functions in `ooloi.frontend.ui.core.cljfx`. Each function:

1. Receives an Ooloi-enriched props map (with `:text-key`, layout conventions, Ooloi-specific keys)
2. `dissoc`s Ooloi-specific keys so they don't leak to cljfx
3. Resolves Ooloi keys to standard cljfx values (`:text-key` → `:text` via `(tr text-key)`, etc.)
4. Passes all other props through unchanged (`:style-class`, `:disable`, `:on-action`, etc.)
5. Returns a standard cljfx description map

```clojure
;; Definition — pure function, map → map
(defn ooloi-button [{:keys [text-key] :as props}]
  (-> (dissoc props :text-key)
      (assoc :fx/type :button
             :text (tr text-key)
             :min-width 90.0)))

;; Usage — keywords, no tr calls, serialisable over gRPC
{:fx/type ooloi-button
 :text-key :piece-window.piece-settings-button}

;; Composite component — encapsulates layout convention
{:fx/type ooloi-button-bar
 :children [{:fx/type ooloi-ok-button}
            {:fx/type ooloi-cancel-button}]}
```

**Why this matters for the ADR-0042 vision:** The spec format requires that backend plugins describe UI using keyword-based specs over gRPC — no frontend-specific code, no `tr` calls. The custom component functions are the resolution mechanism: plugins write `{:fx/type ooloi-button :text-key :some.key}`, the frontend's cljfx pipeline calls `ooloi-button` during materialisation, and `tr` runs at render time on the frontend where the locale is known.

**Testing:** Custom component functions are pure — testable without JavaFX:

```clojure
(fact "ooloi-button resolves text-key and sets min-width"
  (ooloi-button {:text-key :button.ok})
  => (contains {:fx/type :button :text "OK" :min-width 90.0}))

(fact "ooloi-button passes through arbitrary props"
  (ooloi-button {:text-key :button.ok :style-class ["accent"] :disable false})
  => (contains {:style-class ["accent"] :disable false}))
```

The complete component inventory is in the `ooloi.frontend.ui.core.cljfx` namespace.

### Per-Window Reactive Renderer (Updated 2026-02-19)

Windows with rich, stateful content use a **per-window cljfx renderer** alongside the standard `show-window!` boundary. The renderer manages the **content** node reactively; the UI Manager manages the **Stage**. These two responsibilities are absolute and never overlap.

#### Architecture

```
┌────────────── piece_window.clj ───────────────────────────────────┐
│                                                                   │
│  backend events ──(future)──► swap!                               │
│                                  │                                │
│                     ┌────────────▼───────────────────────────┐    │
│                     │      *piece-state (atom {})            │    │
│                     └────────────┬───────────────────────────┘    │
│                                  │ mount-renderer watches         │
│                                  ▼                                │
│                     ┌───────────────────────────────────────┐     │
│                     │         cljfx renderer                │     │
│                     │                                       │     │
│                     │  :middleware — wrap-map-desc          │     │
│                     │    s → {:fx/type piece-window-spec    │     │
│                     │         :state s}                     │     │
│                     │                                       │     │
│                     │  :fx.opt/map-event-handler            │     │
│                     │    dispatch-fn                        │     │
│                     └────────────┬──────────────────────────┘     │
│                                  │ cljfx/instance                 │
│                     ┌────────────▼───────────────────────────┐    │
│                     │    JavaFX Node (BorderPane)            │    │
│                     │                                        │    │
│                     │  SplitPane (centre)                    │    │
│                     │  ├─ ooloi-vscroll-pane: Musicians      │    │
│                     │  └─ ooloi-vscroll-pane: Layouts        │    │
│                     │                                        │    │
│                     │  ooloi-button-bar (bottom)             │    │
│                     │  └─ ooloi-button "Piece Settings…"     │    │
│                     │     :on-action {:ooloi/event           │    │
│                     │                 :ui/show-piece-settings}│   │
│                     └────────────┬───────────────────────────┘    │
└──────────────────────────────────┼────────────────────────────────┘
                                   │ :window/content
                                   ▼
                        show-window! (UI Manager)
                        Stage → window-registry

Map event flow:
  Button click
    │ cljfx appends :fx/event, calls map-event-handler
    ▼
  dispatch-fn (injected from system.clj)
    ├─ piece-internal events ──► piece_window.clj handler (future)
    └─ app commands ────────────► action-handlers (system.clj)
                                   :ui/show-piece-settings → fn
                                   :ui/quit → fn
                                   ...
```

#### How it works

1. `piece-window-content` allocates a fresh state atom per call — each piece window has its own isolated state.
2. `piece-window-content` creates the renderer with two configurations:
   - **`:middleware`** — `cljfx/wrap-map-desc` transforms each new atom value into a cljfx description by calling `piece-window-spec` with the current state.
   - **`:fx.opt/map-event-handler`** — routes all map-form `:on-action` values to a single dispatcher. When a cljfx element has `:on-action {:ooloi/event :foo}` (a map, not a function), cljfx appends `:fx/event` and calls this handler on the JAT.
3. The renderer is called once with the initial state. `cljfx/mount-renderer` then keeps it watching the atom.
4. `cljfx/instance` extracts the JavaFX Node. This is passed as `:window/content` in the `:window-open-requested` event.
5. The renderer continues managing the Node reactively after the window opens — `swap!`ing the state atom diffs and patches the live scene graph without Stage recreation.
6. A one-shot `:window-lifecycle` subscriber calls `cljfx/unmount-renderer` when `:window-closed` fires for the window's id, releasing the atom watch.

Because each call to `piece-window-content` allocates a fresh atom and renderer, any number of piece windows can be open simultaneously with fully isolated state.

#### Map events and dispatch

Map-form `:on-action` values are pure data (serialisable, inspectable). The `:fx.opt/map-event-handler` is the window module's complete internal event dispatch system.

Routing follows a two-tier strategy:

- **Module-internal events** are handled entirely within the window module. Navigation, selection, drag-and-drop reactions, and backend-update reactions live here.
- **App-level commands** (e.g., `:ui/show-piece-settings`) are delegated through `dispatch-fn` to the application's `action-handlers`. The `dispatch-fn` is injected by the caller — the window module has no direct reference to application state or action tables.

This keeps each window module a **self-contained dispatch world**: internal events never leave the module; commands that affect the wider application bubble out through `dispatch-fn` only.

#### Boundary invariants

- **Renderers manage content nodes, never Stages.** The windowing infrastructure is unaware of renderers; renderers are unaware of Stages.
- **`dispatch-fn` is injected, not captured.** Window content functions receive it as a parameter. This keeps modules independently testable.
- **State atoms are private.** Only the owning module writes to them. Callers cannot mutate window state directly.
- **State updates from any source are safe.** `swap!` may come from user interactions, event bus subscriptions, or settings changes. The renderer reacts uniformly.

#### When to use a renderer

**Use a renderer for all application windows.** Every window that goes through `show-window!` uses `cljfx/mount-renderer` + a private state atom. `swap!`ing the state atom from any source — user interactions, event bus subscriptions, or settings changes such as locale — causes the renderer to diff and patch the live scene graph without Stage recreation. This is what makes locale and theme reactivity automatic for free.

**Use `cljfx/create-component` + `cljfx/instance` once** (no renderer) only for `show-confirmation!`, which materialises a blocking cljfx `:alert` spec and returns synchronously via `.showAndWait`. Confirmation dialogs are not full windows — they bypass `show-window!` entirely and have no ongoing lifecycle.

**Locale reactivity pattern.** Windows call `(ui-manager/register-renderer! manager window-id *state)` in their `:window-opened` lifecycle handler, passing their private state atom to the UI Manager's window registry. When the UI Manager receives a `:setting-changed` event for `:ui/locale`, it calls `tr/set-locale!` synchronously on the event bus thread, then posts `(fx/run-later! ...)` to the JAT to call `(swap! *state identity)` on every registered renderer atom. The renderer re-evaluates the spec after the locale is already updated. Windows do not subscribe to `:app-settings` directly — locale reactivity is centralised in the UI Manager. The `:window-closed` branch of the `:window-lifecycle` handler calls `cljfx/unmount-renderer`.

### Architectural Invariants

**All Windowing Goes Through the UI Manager**

Window lifecycle is event-driven. Application code publishes `:window-open-requested` to the `:window-requests` category on the frontend event bus. The UI Manager subscribes, creates the Stage internally via `show-window!`, and publishes `:window-opened` to `:window-lifecycle`. No application code calls `show-window!` directly.

Content builders produce nodes. The UI Manager produces windows.

```clojure
;; ✅ CORRECT — publish a request; UI Manager creates window internally
(eb/publish! event-bus :window-requests
  [{:type             :window-open-requested
    :window/id        :splash
    :window/content   root
    :window/style     :undecorated}])

;; ❌ WRONG — calling show-window! directly bypasses the event-driven architecture
(um/show-window! mgr {:window/id :splash
                       :window/content root
                       :window/style :undecorated})

;; ❌ WRONG — Direct Stage creation bypasses all infrastructure
(let [stage (Stage. StageStyle/UNDECORATED)
      scene (Scene. root)]
  (.setScene stage scene)
  (.show stage))
```

**Invariant:** If code imports `javafx.stage.Stage` or `javafx.scene.Scene` to create new instances, it is violating this invariant. Only the windowing infrastructure (`window.clj`, `ui_manager.clj`) may do so.

**macOS Platform Behaviour:** On macOS, `register-window!` automatically calls `.initOwner` on secondary windows, setting the first registered Stage as owner. This ensures secondary windows (About, tool palettes) inherit the system menu bar. Without this, focused secondary windows show a blank menu bar.

**All Window Close Is Event-Driven**

Application code publishes `:window-close-requested` to `:window-requests`; the UI Manager subscriber calls `close-window!` internally. `close-window!` performs three operations in sequence: persists the window's geometry to disk, closes the Stage, and publishes a `:window-closed` lifecycle event. Bypassing it loses geometry persistence and breaks lifecycle event consumers.

Native close (the window's X button) is intercepted by `onCloseRequest` wiring in `register-window!`, which consumes the JavaFX event and delegates to `close-window!` directly — no event bus round-trip for native close. Component shutdown (`halt-key!`) iterates the window registry and calls `close-window!` for each open window.

```clojure
;; ✅ CORRECT — publish a close request
(eb/publish! event-bus :window-requests
  [{:type :window-close-requested :window/id :settings}])

;; ❌ WRONG — calling close-window! directly bypasses the event-driven architecture
(um/close-window! mgr :settings)

;; ❌ WRONG — bypasses persistence and lifecycle events entirely
(.close stage)
```

**Invariant:** If code calls `.close` on a Stage outside of the windowing infrastructure (`window.clj`), or calls `close-window!` from outside the UI Manager's own event subscriber, it is violating this invariant.

**Theme Application is Automatic**

The window manager infrastructure automatically applies the configured UI theme (AtlantaFX) to all windows and notifications. Developers never specify theme details in UI specifications.

**Invariant:** Theme selection is infrastructure, not specification data.

```clojure
;; ✅ CORRECT - Developer writes pure cljfx
{:fx/type :stage
 :window/id :tool-palette
 :scene {:fx/type :scene
         :root {:fx/type :v-box
                :children [...]}}}

;; ❌ WRONG - Never specify theme in spec
{:fx/type :stage
 :window/id :tool-palette
 :theme :nord-dark  ; NO - theme is infrastructure
 :scene {:fx/type :scene
         :stylesheets ["atlantafx.css"]  ; NO - automatic
         :root {:fx/type :v-box
                :children [...]}}}
```

**Implementation:**
- Theme preference stored in settings (ADR-0043)
- `build-window!` in `window.clj` calls `(theme/apply-theme!)` when constructing each
  Stage — theme is applied as a global JavaFX user agent stylesheet, so all Scenes
  receive it automatically
- Theme changes propagate to all open windows via `apply-theme-to-all!` in the UI Manager

**Benefits:**
- Visual consistency enforced by architecture
- Plugins don't need theme knowledge
- Theme changes don't require spec updates
- Separation of concerns (specification vs presentation)

**Threading: JavaFX Application Thread Required**

All UI Manager operations that create, modify, or close JavaFX objects must be called on the JavaFX Application Thread (JAT). This is a JavaFX platform constraint, not an Ooloi-specific rule.

**Functions that require the JAT:**
- `show-window!`, `register-window!`, `close-window!` — Stage lifecycle
- `build-window!` — Stage construction
- `show-notification!`, `dismiss-notification!`, `build-notification!` — Notification overlay operations
- Any `build-*-content!` function that creates JavaFX nodes
- `show-confirmation!` — blocks on `.showAndWait`

**Enforcement:** `build-window!` and `build-notification!` include an explicit `assert (Platform/isFxApplicationThread)`. Other functions document the requirement in their docstring. Calling from the wrong thread produces unpredictable JavaFX behaviour (visual corruption, race conditions, silent failures).

**For callers on a background thread:** Use `fx/run-later!` to post work to the JAT:

```clojure
;; ✅ CORRECT — post to JAT from background thread
(fx/run-later!
  (fn [] (um/show-window! mgr {:window/id :tool-palette ...})))

;; ❌ WRONG — calling directly from background thread
(future (um/show-window! mgr {:window/id :tool-palette ...}))
```

**Plugin implication:** Backend plugins sending specs over gRPC are unaffected — the frontend's gRPC handler is responsible for dispatching to the JAT. Frontend plugins that call UI Manager functions directly must ensure they are on the JAT or use `fx/run-later!`.

**Translation Key Resolution**

Keys ending in `-key` (`:window/title-key`, `:text-key`) are ADR-0039 translation keys, resolved to localised strings by calling `(tr key)` at render time. Resolution happens inside `build-window!` (for `:window/title-key`) and inside custom component functions like `ooloi-button` (for `:text-key`), not in the spec author's code:

1. The spec author writes `:window/title-key :window.settings.title`
2. The rendering function calls `(tr :window.settings.title)` → `"Settings"`
3. The resolved string is set as the Stage's `:title`

The spec author never calls `tr` — the rendering infrastructure does. This ensures localisation is consistent and the spec remains language-neutral (important for specs crossing gRPC from backend plugins where `tr` is not available).

**Unresolved keys:** If a `-key` value has no translation registered, `tr` returns the key itself as a string (e.g., `":window.settings.title"`). This is a development-time indicator, not a silent failure.

**Icons:** Ikonli Material Design 2 icons (`ikonli-javafx` + `ikonli-material2-pack`) provide font-based icons as JavaFX `Node` instances via `FontIcon`. Icons are referenced by string literals (e.g. `"mdal-info"`, `"mdmz-warning"`) — no enum imports needed from Clojure.

```clojure
(import '[org.kordamp.ikonli.javafx FontIcon])

(FontIcon. "mdal-info")              ; Create icon from literal
(doto (FontIcon. "mdal-check")
  (.setIconSize 24))                 ; With explicit size
```

`FontIcon` can be passed to any JavaFX method accepting `Node`, including `Notification.setGraphic()`, `Button.setGraphic()`, `MenuItem.setGraphic()`, etc.

**Icon colour:** Ikonli icons inherit the `-fx-icon-color` CSS property from their parent control. AtlantaFX severity classes (`.success`, `.warning`, `.danger`, `.accent`) set this property, so icons placed inside severity-styled controls (e.g. notifications) pick up the correct colour automatically.

**Current icon usage:**

| Icon Literal | Where | Purpose |
|-------------|-------|---------|
| `mdal-info` | `cljfx.clj` | Default icon for `:info` notifications |
| `mdal-check` | `cljfx.clj` | Default icon for `:success` notifications |
| `mdmz-warning` | `cljfx.clj` | Default icon for `:warning` notifications |
| `mdal-error` | `cljfx.clj` | Default icon for `:error` notifications |
| `mdmz-undo` | `settings_window.clj` | Reset-to-default button beside each setting field |

**Icon packs:** Only Material Design 2 (`ikonli-material2-pack`) is currently included. Additional packs (Feather, FontAwesome, etc.) can be added as separate dependencies if needed. Browse available icons: [Material2 cheat sheet](https://kordamp.org/ikonli/cheat-sheet-material2.html).

**Notification opacity:** Notifications are semi-transparent by default (0.8 opacity) for a lighter visual presence. Configurable per-notification via `:opacity` in the spec.

### Theme-Independent Styling

All UI styling in Ooloi must work identically in dark and light mode without code changes. This is achieved through two complementary mechanisms provided by [AtlantaFX](https://mkpaz.github.io/atlantafx/): **style classes** for component-level styling and **CSS semantic tokens** for inline CSS when style classes don't cover the need. Both adapt automatically when the theme changes.

**Why this matters for plugins:** UI specifications are pure data (maps) that cross boundaries — a backend plugin sends a cljfx spec over gRPC, and the frontend renders it. If a plugin hardcodes `#888888` for muted text, it breaks in one theme. If it uses `"text-muted"` as a style class, it works in every theme the frontend supports. Theme independence is not cosmetic — it is a serialization and interoperability requirement.

#### Two Levels of Styling

**Level 1 — Style Classes (preferred).** AtlantaFX's [`Styles`](https://mkpaz.github.io/atlantafx/apidocs/atlantafx.base/atlantafx/base/theme/Styles.html) class provides named string constants for typography, severity, elevation, borders, and interactive states. These map to CSS classes that AtlantaFX themes define. Use style classes whenever possible.

**Level 2 — CSS Semantic Tokens (when needed).** When no style class covers the requirement (e.g., a custom background colour for a specific panel), use AtlantaFX's CSS custom properties (`-color-fg-default`, `-color-danger-muted`, etc.) in inline CSS. These resolve to theme-appropriate values at runtime.

**Prohibited:** Literal hex values (`#888`), `rgb()`/`rgba()` functions, and JavaFX `Color` constants in theme-dependent contexts. See [Documented Exceptions](#documented-exceptions) below for the narrow cases where this is acceptable.

#### cljfx Declarative Specs — The Preferred Path

[cljfx](https://github.com/cljfx/cljfx) is Ooloi's declarative UI layer. Its `:style-class` property accepts a vector of strings — exactly what `Styles` constants evaluate to. This is the **preferred** way to apply styling because the result is pure data that serializes over gRPC:

```clojure
;; ✅ PREFERRED — cljfx declarative spec (pure data, serializable)
{:fx/type :label
 :text "Section Header"
 :style-class ["title-2" "text-muted"]}

;; ✅ PREFERRED — cljfx spec using Styles constants (evaluated at definition time)
{:fx/type :label
 :text "Section Header"
 :style-class [Styles/TITLE_2 Styles/TEXT_MUTED]}

;; ✅ ACCEPTABLE — Java interop for dynamic mutations after construction
(-> (.getStyleClass node) (.addAll [Styles/TITLE_2 Styles/TEXT_MUTED]))

;; ❌ WRONG — bare strings without constants (fragile, undiscoverable)
(-> (.getStyleClass node) (.add "title-2"))

;; ❌ WRONG — inline CSS for something a style class handles
(.setStyle node "-fx-font-size: 24; -fx-text-fill: #888;")
```

**When to use Java interop:** Only for dynamic style mutations on already-constructed nodes (e.g., adding a `Styles/DANGER` class when validation fails, removing it when the input is corrected). Initial construction should use cljfx specs wherever possible.

**Plugin implication:** A backend plugin that includes `:style-class ["title-2" "accent"]` in its cljfx spec will render correctly on any Ooloi frontend regardless of the active theme. The string values are stable AtlantaFX API — they don't change between themes.

#### The setAll() Rule — Include Base Classes

**cljfx's `:style-class` calls `getStyleClass().setAll()`**, which replaces the *entire* default style class list. It does not add to the defaults — it replaces them.

JavaFX controls ship with default style classes that AtlantaFX uses as CSS selectors for border and background rendering. When you specify `:style-class` in a cljfx spec, those defaults are gone unless you include them explicitly.

**Controls where this matters most:**

| Control | JavaFX defaults | AtlantaFX border CSS selector |
|---------|-----------------|-------------------------------|
| `ComboBox` | `["combo-box" "combo-box-base"]` | `.combo-box-base` |
| `TextField` | `["text-input" "text-field"]` | `.text-input` |
| `TextArea` | `["text-input" "text-area"]` | `.text-input` |
| `Button` | `["button"]` | `.button` |

**Wrong and correct:**

```clojure
;; ❌ WRONG — strips "combo-box-base"; control renders without visible border
{:fx/type :combo-box
 :style-class ["combo-box" Styles/DENSE]}

;; ✅ CORRECT — includes all three: both defaults plus DENSE
{:fx/type :combo-box
 :style-class ["combo-box" "combo-box-base" Styles/DENSE]}

;; ❌ WRONG — strips "text-input"; control renders without visible border
{:fx/type :text-field
 :style-class ["text-field" Styles/DENSE]}

;; ✅ CORRECT — includes all three: both defaults plus DENSE
{:fx/type :text-field
 :style-class ["text-input" "text-field" Styles/DENSE]}
```

**To check the defaults for any control:** look in the cljfx jar source (e.g. `cljfx/fx/combo_box.clj`) for the `:style-class` property's `:default` value. That is the list you must reproduce when specifying `:style-class`.

**Tempting shortcut — do not use:** using `ext-on-instance-lifecycle :on-created` to call `.add` on the style class list sidesteps the need to know the defaults, but it breaks the pure-data spec paradigm. The result is no longer serialisable over gRPC, no longer testable without JavaFX, and is inconsistent with ADR-0042's core requirement. There is one correct approach: write the complete `:style-class` list in the spec.

**Consequence of stripping base classes:** AtlantaFX simulates control borders using multi-stop `-fx-background-color` layers — the outermost stop is the border paint, the inner stop is the fill. The border-rendering CSS rule is anchored to the base selector (e.g. `.combo-box-base`). Without that class on the node, the rule never fires and the control appears as plain text on the background with no visible border.

#### AtlantaFX Style Class Reference

The [`Styles`](https://mkpaz.github.io/atlantafx/apidocs/atlantafx.base/atlantafx/base/theme/Styles.html) class in `atlantafx.base.theme` provides these constants. Each is a `public static final String` that evaluates to the CSS class name shown.

**Typography:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `TITLE_1` | `"title-1"` | Largest heading |
| `TITLE_2` | `"title-2"` | Second heading |
| `TITLE_3` | `"title-3"` | Third heading |
| `TITLE_4` | `"title-4"` | Smallest heading |
| `TEXT` | `"text"` | Standard body text |
| `TEXT_BOLD` | `"text-bold"` | Bold body text |
| `TEXT_CAPTION` | `"text-caption"` | Caption/small text |
| `TEXT_MUTED` | `"text-muted"` | De-emphasised text (theme-aware grey) |
| `TEXT_SUBTLE` | `"text-subtle"` | Even lighter de-emphasis |
| `TEXT_SMALL` | `"text-small"` | Reduced font size |

**Severity / Accent:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `ACCENT` | `"accent"` | Theme accent colour |
| `SUCCESS` | `"success"` | Green success state |
| `DANGER` | `"danger"` | Red error/danger state |
| `WARNING` | `"warning"` | Yellow/orange warning state |

**Elevation:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `ELEVATED_1` | `"elevated-1"` | Subtle shadow |
| `ELEVATED_2` | `"elevated-2"` | Medium shadow |
| `ELEVATED_3` | `"elevated-3"` | Prominent shadow |
| `ELEVATED_4` | `"elevated-4"` | Maximum shadow |

**Buttons:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `FLAT` | `"flat"` | No background until hover |
| `BUTTON_CIRCLE` | `"button-circle"` | Circular button |
| `BUTTON_ICON` | `"button-icon"` | Icon-only button (no text padding) |
| `BUTTON_OUTLINED` | `"button-outlined"` | Border, no fill |

**Backgrounds:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `BG_DEFAULT` | `"bg-default"` | Standard background |
| `BG_INSET` | `"bg-inset"` | Recessed background |
| `BG_SUBTLE` | `"bg-subtle"` | Subtle differentiation |
| `BG_NEUTRAL_MUTED` | `"bg-neutral-muted"` | Neutral muted background |
| `BG_NEUTRAL_EMPHASIS` | `"bg-neutral-emphasis"` | Neutral prominent background |

**Borders:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `BORDERED` | `"bordered"` | Standard border |
| `BORDERED_DEFAULT` | `"bordered-default"` | Default colour border |
| `BORDERED_MUTED` | `"bordered-muted"` | Muted border |
| `BORDERED_ACCENT` | `"bordered-accent"` | Accent colour border |
| `BORDERED_SUCCESS` | `"bordered-success"` | Success colour border |
| `BORDERED_DANGER` | `"bordered-danger"` | Danger colour border |
| `BORDERED_WARNING` | `"bordered-warning"` | Warning colour border |

**Layout / Behaviour:**

| Constant | CSS Class | Effect |
|----------|-----------|--------|
| `DENSE` | `"dense"` | Reduced padding/spacing |
| `INTERACTIVE` | `"interactive"` | Hover/click visual feedback |
| `STRIPED` | `"striped"` | Alternating row colours (tables) |
| `LEFT` | `"left"` | Left alignment |
| `CENTER` | `"center"` | Centre alignment |
| `RIGHT` | `"right"` | Right alignment |
| `TOP` | `"top"` | Top alignment |
| `BOTTOM` | `"bottom"` | Bottom alignment |

**Utility Methods:**

`Styles` also provides `addStyleClass(Node, String...)`, `removeStyleClass(Node, String...)`, and `toggleStyleClass(Node, String)` for imperative use. In Clojure these are less useful than direct `.getStyleClass` manipulation, but plugins targeting Java interop may use them.

#### CSS Semantic Token Reference

When inline CSS is needed (and no style class suffices), use AtlantaFX's CSS custom properties. These resolve to theme-appropriate values automatically. Apply via `:style` in cljfx specs or `.setStyle` in interop:

```clojure
;; ✅ CORRECT — semantic token adapts to theme
{:fx/type :label
 :text "Status"
 :style "-fx-text-fill: -color-fg-muted;"}

;; ❌ WRONG — hardcoded colour breaks in other theme
{:fx/type :label
 :text "Status"
 :style "-fx-text-fill: #888888;"}
```

AtlantaFX follows the [GitHub Primer colour system](https://primer.style/foundations/color). Tokens are organised into functional categories that describe *purpose*, not *appearance*. The same token resolves to different concrete colours in dark vs light mode.

**Available Functional Tokens:**

| Category | Tokens | Purpose |
|----------|--------|---------|
| **Foreground** | `-color-fg-default`, `-color-fg-muted`, `-color-fg-subtle`, `-color-fg-emphasis` | Text and icon colours at varying prominence levels |
| **Background** | `-color-bg-default`, `-color-bg-overlay` | Surface colours for panels, popups, overlays |
| **Border** | `-color-border-default`, `-color-border-muted`, `-color-border-subtle` | Separator and outline colours |
| **Accent** | `-color-accent-fg`, `-color-accent-muted`, `-color-accent-subtle`, `-color-accent-emphasis` | Primary brand/info colour in four intensities |
| **Success** | `-color-success-fg`, `-color-success-muted`, `-color-success-subtle`, `-color-success-emphasis` | Positive/confirmation colour in four intensities |
| **Warning** | `-color-warning-fg`, `-color-warning-muted`, `-color-warning-subtle`, `-color-warning-emphasis` | Caution colour in four intensities |
| **Danger** | `-color-danger-fg`, `-color-danger-muted`, `-color-danger-subtle`, `-color-danger-emphasis` | Error/destructive colour in four intensities |
| **Neutral** | `-color-neutral-emphasis-plus`, `-color-neutral-emphasis`, `-color-neutral-muted`, `-color-neutral-subtle` | Grey tones for secondary UI elements |

**Choosing Tokens — Intensity Levels:**

Each severity category offers four intensity levels. Choose based on the UI element's role:

- **`-fg`**: Use as text colour on a default or subtle background. High contrast.
- **`-muted`**: Use as background for medium-emphasis surfaces (e.g., notification backgrounds). Good contrast with `-color-fg-default` text.
- **`-subtle`**: Use as background for low-emphasis surfaces. Lighter tint, less prominent.
- **`-emphasis`**: Use as background for high-emphasis elements (e.g., badges, pills). Often requires `-color-fg-emphasis` text for contrast.

**Invariant:** Scale tokens (`-color-base-0` through `-color-base-9`, `-color-accent-0` through `-color-accent-9`, etc.) must NOT be used. They are raw palette values that do not adapt semantically across themes. Always use the functional tokens above.

#### Established Usage Patterns

The following patterns are established in the Ooloi codebase. New UI code should follow these conventions.

**Notifications** (severity-typed popup messages):

Notifications are shown via `(um/show-notification! mgr spec)` where `spec` is a map with `:message` (string), `:type` (`:info`, `:success`, `:warning`, `:error`), and optional `:timeout-ms` (auto-dismiss delay) and `:opacity` (default 0.8). Notifications do not create a Stage — they are rendered into a shared overlay Popup attached to the primary window.

The notification component (`ooloi-notification` in `ooloi.frontend.ui.core.cljfx`) is an `ext-instance-factory` that materialises an AtlantaFX `Notification` control. It resolves `:text-key` via `tr`, applies severity style classes, and sets a default icon from the type. `build-notification!` in `ui_manager.clj` is the materialisation wrapper called by the notification system.

Notifications demonstrate the two-layer mechanism. The style class (`Styles/SUCCESS`, etc.) selects the severity variant. A generated CSS stylesheet maps each variant to its theme-aware background using CSS semantic tokens:

```css
/* Generated at runtime by init-notification-overlay! */
.notification.success > .container {
  -fx-background-color: -color-success-muted;   /* ← CSS token, adapts to theme */
}
.notification.warning > .container {
  -fx-background-color: -color-warning-muted;
}
.notification.danger > .container {
  -fx-background-color: -color-danger-muted;
}
```

The style classes applied per notification type:

| Notification type | Style classes | Background token |
|-------------------|---------------|------------------|
| `:info` | `"notification"` `Styles/ELEVATED_2` | Default (no severity accent) |
| `:success` | `"notification"` `Styles/SUCCESS` `Styles/ELEVATED_2` | `-color-success-muted` |
| `:warning` | `"notification"` `Styles/WARNING` `Styles/ELEVATED_2` | `-color-warning-muted` |
| `:error` | `"notification"` `Styles/DANGER` `Styles/ELEVATED_2` | `-color-danger-muted` |

This is the general pattern: **style classes** name the variant, **CSS tokens** provide the themed visual. The `Styles` class constants are selectors; the `-color-*` tokens are values. Neither works alone — both are needed for theme-independent styling that also carries semantic meaning.

**Window content** (headings, placeholder text):

| Purpose | Style classes | Example |
|---------|---------------|---------|
| Section heading | `Styles/TITLE_2` | Piece window placeholder |
| De-emphasised text | `Styles/TEXT_MUTED` | Placeholder/hint text |
| Combined heading + muted | `[Styles/TITLE_2 Styles/TEXT_MUTED]` | "Temporary Piece Window" label |

**Form controls** (settings window, input validation):

| Purpose | Style class | Usage |
|---------|-------------|-------|
| Flat icon button (reset) | `Styles/FLAT` | Reset-to-default buttons beside settings fields |
| Invalid input | `Styles/DANGER` | Added dynamically when validation fails |
| Valid input (restore) | Remove `Styles/DANGER` | Removed dynamically when input becomes valid |

**Windows** (About, Settings):

| Purpose | Approach | Notes |
|---------|----------|-------|
| Window title bar | `Styles/TITLE_3` or `Styles/TITLE_4` | Depending on window size |
| OK / Cancel buttons | Standard Button (no extra class) | AtlantaFX default styling is sufficient |
| Action buttons | `Styles/ACCENT` | For primary action emphasis |

**Plugin guidelines:** Plugins should use the same patterns. A plugin notification uses `:style-class (into ["notification"] [(get type->severity type)])`. A plugin window heading uses `:style-class ["title-3"]`. The string values (not the Java constants) travel over gRPC — `"title-3"` and `Styles/TITLE_3` are the same string at runtime.

#### Documented Exceptions

Two narrow cases permit literal colour values:

1. **Pre-theme rendering (splash screen).** The splash renders before AtlantaFX themes are loaded. It uses hardcoded CSS strings for branding colours that are constant regardless of theme: `:style "-fx-background-color: #1a1a2e;"` for the background, `:style "-fx-text-fill: white; ..."` for the progress label, and `:color "#000000cc"` for the drop shadow effect. These are cljfx spec properties, not Java API calls — but the colour values themselves are intentionally non-adaptive.

2. **Theme-aware transparency overlays.** The About window's translucent background implements both light and dark appearances directly using `rgba()` values that are chosen per-theme at construction time. This is legitimate because transparency blending over arbitrary content cannot be expressed as a single semantic token — the overlay must know the target luminance.

#### Font Handling

Ooloi relies on the platform's system font as selected by JavaFX — no application-wide font override. AtlantaFX themes set font metrics (size, weight, family) through their CSS; the `Styles` typography classes (`TITLE_1` through `TITLE_4`, `TEXT`, `TEXT_BOLD`, `TEXT_CAPTION`, `TEXT_SMALL`) adjust size and weight relative to this base.

**Preferred approach:** Use `Styles` typography classes for all text sizing and weight. They respect the platform font and adapt to theme changes.

```clojure
;; ✅ PREFERRED — typography class handles size and weight
{:fx/type :label
 :text "Section Header"
 :style-class ["title-3"]}

;; ✅ ACCEPTABLE — interop equivalent
(-> (.getStyleClass label) (.add Styles/TITLE_3))
```

**When style classes are insufficient:** If a specific font size or weight is genuinely needed beyond what the typography classes provide (e.g., branding text in the splash screen), inline CSS font properties are acceptable. However, font *colour* must still use style classes or CSS semantic tokens — never hardcoded values, except in the documented exceptions above.

```clojure
;; Acceptable for special-purpose sizing (splash branding)
{:fx/type :label
 :style "-fx-font-size: 14px; -fx-font-weight: bold; -fx-text-fill: white;"}
;; Note: white text here is a documented exception (pre-theme rendering)
;; Outside of documented exceptions, always use style classes or semantic tokens
```

**Font family overrides are discouraged.** Specifying `-fx-font-family` ties the UI to a font that may not exist on all platforms. If a specific font family is ever required (e.g., for music-specific glyphs), it should be declared as a project-level decision and loaded as a resource, not hardcoded in individual components.

### Command Descriptors (Menu Specialisation)

Menu items in Ooloi are defined as **command descriptors** — pure maps that each describe a single menu command. A command descriptor holds everything needed to render a menu item: the i18n text key, the keyboard accelerator, an enablement predicate, and the action handler. This eliminates duplication — each command is defined once, and the menu bar is generated from the collection of descriptors.

```clojure
;; A command descriptor — one per menu action
{:id :ui/connect
 :text-key :menu.file.connect                    ; ADR-0039 translation key
 :accelerator {:key :O :mods [:shortcut :shift]}  ; Platform-aware shortcut
 :enabled? (fn [state] (:backend-ready? state))    ; When is this command available?
 :on-action {:event/type :ui/connect-dialog}}      ; What happens when selected
```

A small pure rendering function transforms each descriptor into a cljfx menu-item map. At render time, it applies the `:enabled?` predicate against current UI state to produce a concrete `:disable` boolean, and keeps `:text-key` unresolved (the frontend's i18n layer resolves translation keys to strings, consistent with ADR-0039):

```clojure
;; What the rendering function produces from the descriptor above
{:fx/type :menu-item
 :text-key :menu.file.connect
 :accelerator {:key :O :mods [:shortcut :shift]}
 :disable true   ; result of (not ((:enabled? cmd) current-state))
 :on-action {:event/type :ui/connect-dialog}}
```

The menu bar itself is also pure data. A function assembles descriptors into a cljfx MenuBar spec, and the result is rendered like any other cljfx component. This is the same pure-map-to-cljfx pattern used for windows.

**Accelerators** in Ooloi always use `:mods [:shortcut ...]` — never hardcoded `:meta` or `:control`. The `:shortcut` modifier resolves to Cmd on macOS and Ctrl on other platforms at the JavaFX/cljfx level.

**macOS system menu bar** (the menu bar embedded in the macOS top bar rather than in the window frame) is controlled by `:use-system-menu-bar` on the MenuBar spec. MenuBar specs are produced by the UI toolkit layer, which injects this flag based on an injectable OS predicate. Developer-authored and plugin-authored specs never contain this flag — infrastructure decides, specifications don't know.

**Stub commands** are descriptors with `(constantly false)` as their enablement predicate and no `:on-action`. They exist to stabilise menu structure during early development. They are intentionally inert and do not imply feature commitment.

### Validation Strategy

```clojure
(defn validate-window-spec
  "Validates window specification map following ADR-0018 event validation pattern.

   Validates:
   - Persistent windows must have :window/id
   - Position/size maps have correct structure
   - Field names are kebab-case keywords

   Note: :fx/type is NOT expected in window specs. It is added internally by
   build-window! when constructing the cljfx stage description. Window specs
   passed to show-window! are plain maps with :window/* keys and :window/content.

   Returns: spec if valid
   Throws: ExceptionInfo with descriptive message if validation fails"
  [spec]
  (when-not (map? spec)
    (throw (ex-info "Window spec must be a map" {:spec spec})))

  (when (and (not (false? (:window/persist? spec))) (not (:window/id spec)))
    (throw (ex-info "Persistent window must have :window/id" {:spec spec})))  
  
  ;; Validate position structure if present  
  (when-let [pos (:window/default-position spec)]  
    (when-not (and (map? pos) (int? (:x pos)) (int? (:y pos)))  
      (throw (ex-info "Invalid :window/default-position format" {:spec spec :position pos}))))  
  
  ;; Validate size structure if present  
  (when-let [size (:window/default-size spec)]
    (when-not (and (map? size) (int? (:width size)) (int? (:height size)))
      (throw (ex-info "Invalid :window/default-size format" {:spec spec :size size}))))
  
  spec)  
```  
  
### Window Manager Integration

**Implementation Note (Updated 2026-02-17)**: The actual implementation uses an atom + lock architecture for thread-safe persistence. Window geometry is persisted on close (immediate), move, and resize (debounced 250ms) — not just on shutdown. An in-memory atom is the source of truth; disk writes are serialized via a lock. `show-window!` restores persisted geometry on open. Windows with `:window/persist? false` (e.g. splash screen) skip both persistence and restore.

```clojure
;; show-window! restores persisted geometry (when :window/persist? is true, the default)
(defn show-window! [manager spec]
  (let [persisted (when (get spec :window/persist? true)
                    (persistence/get-persisted-geometry (:window/id spec)))
        merged-spec (if persisted
                      (merge spec
                             {:x     (get-in persisted [:position :x])
                              :y     (get-in persisted [:position :y])
                              :width  (get-in persisted [:size :width])
                              :height (get-in persisted [:size :height])})
                      spec)
        stage (register-window! manager merged-spec)]
    ...))

;; register-window! wires debounced geometry listeners (when :window/persist? is true)
;; Each property change resets a 250ms timer; persist fires once when changes stop
(defn- wire-geometry-listeners! [stage window-id manager]
  ;; Listeners on xProperty, yProperty, widthProperty, heightProperty
  ;; Each calls schedule-geometry-persist! which debounces via ScheduledExecutorService
  ...)

;; close-window! cancels any pending debounce timer and persists immediately
(defn close-window! [manager window-id]
  (when-let [stage (get @(:window-registry manager) window-id)]
    ;; Cancel pending debounce — close persists immediately
    (when-let [pending (get @(:geometry-timers manager) window-id)]
      (.cancel pending false))
    (let [geom (window/stage-geometry stage)]
      (persistence/persist-window!
        {:id window-id :title (:title geom)
         :position {:x (:x geom) :y (:y geom)}
         :size {:width (:width geom) :height (:height geom)}
         :scale 1.0 :state :normal}))
    (window/close! stage)
    ...))

;; persist-window! is thread-safe: atom swap + disk write under lock
(defn persist-window! [state-map]
  (ensure-loaded!)
  (swap! window-states merge-window-state state-map)
  (locking persistence-lock
    (save-window-state @window-states)))
```

## Benefits

✅ **Matches event pattern** - Events already use pure maps with namespace-qualified keys  
✅ **Fulfills design requirement** - "plugins send cljfx maps directly"  
✅ **Zero serialization complexity** - Maps serialize over gRPC transparently  
✅ **Maximum flexibility** - Any cljfx component works immediately  
✅ **Plugin-friendly** - Standard cljfx + `:window/` metadata  
✅ **Simple validation** - Custom validator like `validate-event-structure`  
✅ **Namespace separation** - `:window/`, `:fx/`, `:event/` domains clear  
✅ **No translation layer** - Direct rendering via cljfx  
✅ **Extensible** - New metadata keys don't break existing code  

## Trade-offs

❌ **No compile-time type safety** - Typos caught at runtime  
❌ **No IDE autocomplete** - Can't discover available `:window/` keys  
❌ **Less documentation** - No defrecord docstring or field docs  
❌ **Runtime validation only** - Errors appear when window created  

## Alternatives Considered

### Constructor Functions

Provide constructor functions (e.g., `create-dialog-spec`, `create-window-spec`) that return validated cljfx maps with proper metadata. Functions offer API guidance and parameter validation while still producing pure data.

**Rejected because:**
- Adds unnecessary abstraction layer between plugin and UI
- Requires plugins to import constructor functions (shared dependency)
- Less flexible - each window type needs a constructor
- Inconsistent with event pattern (events use raw maps)
- Doesn't align with design requirement: "plugins send cljfx maps directly"

### Defrecords with Protocol Conversion

Use defrecords for UI specifications with compile-time type safety, then convert to cljfx maps via protocol at render time.

**Rejected because:**
- Breaks plugin architecture - defrecords don't serialize simply over gRPC
- Requires complex transit serialization in gRPC layer
- Two-step process (create record → convert to map) adds complexity
- Contradicts existing event pattern (pure maps)
- Directly contradicts ADR-0038: "plugins send cljfx maps directly"
- Musical elements use defrecords because they're domain models with behavior; UI specs are transient boundary data

## Comparison to Musical Elements

### Why Musical Elements Use Defrecords

Musical elements (Pitch, Rest, Measure) are **domain models with behavior**:

1. **Long-lived** - Exist throughout composition lifecycle
2. **Complex relationships** - Hierarchical structure, VPD addressing
3. **Trait membership** - Implement RhythmicItem, TakesAttachment, etc.
4. **Cached** - Hash-consing for memory efficiency
5. **Local computation** - Methods operate on musical semantics
6. **Type-based dispatch** - Different behavior for Pitch vs Rest vs Chord

Musical elements need:
- Compile-time type safety (can't accidentally create invalid Pitch)
- Protocol dispatch (same operation, different implementations)
- Performance optimization (hash-consing)
- Rich behavior (duration calculations, transposition, etc.)

### Why UI Specs Use Maps

UI specifications are **transient data crossing boundaries**:

1. **Short-lived** - Created on demand, rendered, discarded
2. **No behavior** - Pure data describing what to render
3. **Serialization-heavy** - Cross gRPC boundary frequently
4. **Plugin-generated** - Backend/plugins define UI
5. **Declarative** - Describe structure, not behavior
6. **Flexible schema** - New metadata shouldn't break existing code

UI specs need:
- Simple serialization (already works with gRPC)
- Plugin independence (standard cljfx knowledge)
- Flexible extension (new `:window/` keys)
- No compilation requirement (dynamic plugin loading)

### The Pattern

**Domain models** (long-lived, behavioral) → Defrecords + Traits
**Boundary data** (transient, descriptive) → Pure Maps + Validation

This is consistent with how events work: they're boundary data, so they're maps.

## Integration Points

### With cljfx Rendering

```clojure
;; Frontend code calls show-window! (never creates Stage directly)
(defn show-about! [manager]
  (let [content (cljfx/instance (cljfx/create-component (about-content-spec manager)))]
    (um/show-window! manager
      {:window/id :about
       :window/content content
       :window/style :undecorated
       :window/title-key :window.about.title})))

;; Theme application is global — one call sets the UA stylesheet for all windows
;; (Application/setUserAgentStylesheet sets it once for the entire application)
(theme/apply-theme!)
;; All Scenes inherit the theme automatically — no per-scene stylesheet manipulation
```

### With Persistence Layer

```clojure
;; Geometry persists on every close, move, and resize via atom + lock.
;; close-window! captures geometry before closing:
(persistence/persist-window!
  {:id window-id :title (:title geom)
   :position {:x (:x geom) :y (:y geom)}
   :size {:width (:width geom) :height (:height geom)}
   :scale 1.0 :state :normal})

;; On window open, show-window! restores persisted geometry:
(when-let [persisted (persistence/get-persisted-geometry (:window/id spec))]
  ;; Merge persisted x/y/width/height into spec, overriding defaults
  ...)

;; JavaFX property listeners on xProperty, yProperty, widthProperty,
;; heightProperty trigger persist-window! on every move and resize.
;; No shutdown-only save — state is always current on disk.
```

### With gRPC/Plugin Communication

```clojure
;; Plugin/backend sends UI request
(api-call client "show-dialog"
  {:fx/type :stage
   :window/id :plugin-settings
   :window/title-key :plugin.settings.title  ; → "Plugin Settings"
   :scene {:fx/type :scene
           :root {:fx/type :v-box
                  :children [...]}}})

;; Frontend receives, validates, renders
(defn handle-show-dialog [spec]
  (validate-window-spec spec)
  (show-window spec))

;; No serialization issues - it's just a map
```

### With Event Handler Resolution

**Backend Plugins (Over gRPC):**

Backend and remote plugins use symbolic event handlers (maps) because functions don't serialize over gRPC. For Ooloi-standard translation, they use `ooloi-button` (the custom component var), referenced by qualified symbol:

```clojure
;; Backend plugin sends ooloi-button spec with symbolic handler
{:fx/type 'ooloi.frontend.ui.core.cljfx/ooloi-button
 :text-key :common.save   ; → "Save" resolved by ooloi-button via tr
 :on-action {:event/type :plugin/save-settings
             :plugin-id :my-plugin}}
;; NOTE: future capability — backend-described dialogs not yet implemented
```

**Frontend Plugins (Local):**

Frontend plugins reference Clojure vars directly — no serialization needed:

```clojure
;; Frontend plugin uses ooloi-button var directly, with function handler
{:fx/type ooloi-button
 :text-key :common.save   ; → "Save"
 :on-action (fn [event] (save-local-settings!))}

;; OR using symbolic handlers if preferred for consistency with backend plugins
{:fx/type ooloi-button
 :text-key :common.save
 :on-action {:event/type :frontend-plugin/save-settings}}
```

**Key Distinction:**
- **Serialization boundary determines handler format and how `:fx/type` is expressed**
- Backend plugins → qualified symbols (serializable), symbolic handlers (required)
- Frontend plugins → direct var references, direct functions (allowed) or symbolic (optional)
- Same spec structure, different serialisation form of `:fx/type`

## Consequences

**Positive:**
- Backend plugins can define UI using standard cljfx knowledge
- Frontend plugins can use direct callback functions (no serialization overhead)
- No translation layer needed between backend and frontend
- gRPC serialization works transparently for backend plugins
- New window types don't require framework changes
- Consistent with existing event pattern
- Simple to reason about and debug
- Automatic theme application ensures visual consistency
- Theme changes apply system-wide without spec updates
- Plugin location (backend vs frontend) determines handler format naturally

**Negative:**
- Runtime validation only (no compile-time safety)
- No IDE autocomplete for `:window/` metadata keys
- Documentation must be explicit (no defrecord field docs)
- Typos in metadata keys caught at runtime

**Neutral:**
- Validation function must be maintained as `:window/` keys evolve
- Documentation needs to clearly explain metadata key usage
- Plugin developers need to understand `:window/` namespace convention

## Related ADRs

- **ADR-0018:** API gRPC Interface and Events - Establishes event structure pattern (pure maps)
- **ADR-0038:** Backend-Authoritative Rendering - Defines cljfx responsibilities ("Windows, menus, input handlers")
- **ADR-0039:** Localisation Architecture - Requires all UI strings use translation keys
- **ADR-0003:** Plugins - Plugin architecture requirements
- **ADR-0043:** Frontend Settings - Application preferences system with registry-driven Settings window
