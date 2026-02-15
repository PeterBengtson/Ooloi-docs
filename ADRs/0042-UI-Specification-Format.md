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
  - [Content Builder Pattern](#content-builder-pattern-updated-2026-02-11)
  - [Architectural Invariants](#architectural-invariants)
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

Ooloi needs a format for specifying UI elements (windows, dialogs, notifications) that:

1. **Works across boundaries** - Backend and backend plugins must be able to define UI that frontend renders
2. **Supports lifecycle management** - Windows need automatic persistence, event handling, cleanup
3. **Enables plugin independence** - Backend plugins shouldn't need frontend-specific code
4. **Serializes over gRPC** - UI specs from backend/backend plugins must cross network boundary
5. **Integrates with cljfx** - Must work naturally with cljfx's declarative UI approach (backend and frontend plugins)
6. **Handles diverse types** - Persistent windows, transient dialogs, auto-dismiss notifications
7. **Supports local plugins** - Frontend plugins can use direct callbacks without serialization

### Requirements

**Functional Requirements:**
- Window lifecycle management (create, track state, save on close, restore on reopen, cleanup)
- Type differentiation (persistent windows, modal/non-modal dialogs, notifications, custom components)
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

**Implementation Note (Updated 2026-01-28)**: Initial windowing system (Issue #5) does not implement backend-described dialogs. Backend plugins use piece settings (ADR-0016) with auto-generated UI via `SRV/get-settings-ui-metadata`. Frontend provides all custom dialogs for operations. Backend-described dialog capability (examples below) represents architectural design for future extensibility if needed.

```clojure
;; Backend plugin sends this over gRPC (symbolic event handlers)
;; NOTE: Future capability - not implemented in initial windowing system
{:fx/type :stage
 :window/id :plugin-config           ; Window management metadata
 :window/persist? true
 :window/type :dialog
 :window/default-position {:x 100 :y 100}
 :window/default-size {:width 600 :height 400}
 :window/title-key :plugin.config.title  ; → "Plugin Configuration"
 :scene {:fx/type :scene
         :root {:fx/type :v-box
                :children [{:fx/type :text-field
                            :prompt-text-key :plugin.config.api-key-prompt}  ; → "API Key"
                           {:fx/type :button
                            :text-key :common.save  ; → "Save"
                            :on-action {:event/type :plugin/save-settings}}]}}}  ; Symbolic

;; Frontend plugin can use direct functions (no gRPC boundary)
{:fx/type :stage
 :window/id :local-tool
 :window/title-key :tool.palette.title
 :scene {:fx/type :scene
         :root {:fx/type :v-box
                :children [{:fx/type :button
                            :text-key :common.save
                            :on-action (fn [e] (save-local-data!))}]}}}  ; Direct function
```

### Metadata Keys

**`:window/id`** (keyword, required for persistent windows)
- Unique identifier for this window
- Used for state persistence lookup
- Example: `:main-editor`, `:plugin-config`, `:tool-palette`

**`:window/persist?`** (boolean, default false)
- Whether to save/restore window state
- Persistent: main windows, tool palettes
- Non-persistent: notifications, alerts

**`:window/type`** (keyword, optional)
- Semantic type hint: `:stage`, `:dialog`, `:notification`
- Used for applying type-specific defaults
- Not required - `:fx/type` is authoritative for rendering

**`:window/default-position`** (map, optional)
- Default position if no saved state: `{:x int :y int}`
- Used on first open or when saved state invalid

**`:window/default-size`** (map, optional)
- Default size if no saved state: `{:width int :height int}`

**`:window/default-scale`** (double, optional)
- Default zoom/scale factor: `1.0`, `1.5`, etc.

**`:window/style`** (keyword, optional)
- JavaFX StageStyle: `:decorated` (default), `:undecorated`, `:transparent`, `:utility`
- Example: `:undecorated` for splash screen and About dialog

**`:window/menu-bar`** (JavaFX MenuBar, optional)
- Pre-built MenuBar instance to attach to the window's VBox root
- On macOS with `:use-system-menu-bar true`, the bar integrates with the system menu

**`:window/content`** (JavaFX Node, optional)
- Content node to add to the window's VBox root with `VGrow/ALWAYS`
- Produced by content builder functions (see Content Builder Pattern below)
- When neither `:width` nor `:height` is specified, the Scene sizes itself from content

**`:window/title-key`** (keyword, optional)
- ADR-0039 translation key resolved via `tr` for the window title
- Example: `:window/title-key :window.tool-palette.title` → title "Tools"

### Content Builder Pattern (Updated 2026-02-11)

UI elements that need custom content — splash screens, About dialogs, future tool palettes — follow the **content builder pattern**. A `build-*-content!` function constructs JavaFX nodes and returns a map of the root node plus any references the caller needs. The UI Manager's `show-window!` handles everything else: Stage creation, theming, registry, platform concerns.

```clojure
;; Content builder — returns nodes, never a Stage
(defn build-about-content! []
  (let [root (StackPane.)
        ok-btn (Button. (tr :button.ok))]
    ;; ... assemble content ...
    {:root root :ok-button ok-btn}))

;; Caller uses show-window! with :window/content
(let [{:keys [root ok-button]} (about/build-about-content!)]
  (um/show-window! mgr
    {:window/id :about
     :window/content root
     :window/style :undecorated
     :title "About Ooloi"
     :width 800 :height 700})
  (.setOnAction ok-button
    (reify EventHandler (handle [_ _] (um/close-window! mgr :about)))))
```

**The contract:**
- `build-*-content!` returns `{:root Node, ...}` — the root node and any interactive references (buttons, labels)
- `build-*-content!` never imports Stage, Scene, or StageStyle
- The caller passes `:root` as `:window/content` to `show-window!`
- The UI Manager handles Stage creation, theming, registry tracking, and platform behaviour (e.g. `initOwner` on macOS)

This pattern ensures all windows go through the same path regardless of their content complexity.

### Architectural Invariants

**All Windowing Goes Through the UI Manager**

No code outside the UI Manager creates Stage or Scene objects directly. The UI Manager's `show-window!` is the single entry point for all window creation. This ensures consistent theming, registry tracking, platform behaviour (macOS `initOwner` for system menu bar inheritance), and lifecycle management.

Content builders produce nodes. The UI Manager produces windows.

```clojure
;; ✅ CORRECT — Content builder returns nodes, UI Manager creates window
(let [{:keys [root label]} (splash/build-splash-content!)]
  (um/show-window! mgr {:window/id :splash
                         :window/content root
                         :window/style :undecorated}))

;; ❌ WRONG — Direct Stage creation bypasses all infrastructure
(let [stage (Stage. StageStyle/UNDECORATED)
      scene (Scene. root)]
  (.setScene stage scene)
  (.show stage))
```

**Invariant:** If code imports `javafx.stage.Stage` or `javafx.scene.Scene` to create new instances, it is violating this invariant. Only the windowing infrastructure (`window.clj`, `ui_manager.clj`) may do so.

**macOS Platform Behaviour:** On macOS, `register-window!` automatically calls `.initOwner` on secondary windows, setting the first registered Stage as owner. This ensures secondary windows (About, tool palettes) inherit the system menu bar. Without this, focused secondary windows show a blank menu bar.

**Theme Application is Automatic**

The window manager infrastructure automatically applies the configured UI theme (AtlantaFX) to all windows, dialogs, and notifications. Developers never specify theme details in UI specifications.

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
- Theme preference stored in settings (ADR-0016)
- `render-cljfx-to-stage` applies theme automatically during rendering
- All Scenes receive consistent theme styling without specification
- Theme changes apply system-wide without modifying UI specs

**Benefits:**
- Visual consistency enforced by architecture
- Plugins don't need theme knowledge
- Theme changes don't require spec updates
- Separation of concerns (specification vs presentation)

**Colour Token Invariant:** When CSS overrides are necessary (e.g., notification severity backgrounds, custom component styling), all colours must use AtlantaFX semantic tokens (`-color-fg-default`, `-color-danger-muted`, `-color-success-muted`, etc.). Literal hex values, `rgb()` functions, or JavaFX `Color` constants are prohibited. This extends the theme-as-infrastructure principle to CSS-level customisation — overrides adapt automatically to dark/light mode. See [UI Architecture §7: Colour Tokens](../research/UI_ARCHITECTURE.md) for the full token reference.

**Icons:** Ikonli Material Design 2 icons (`ikonli-javafx` + `ikonli-material2-pack`) provide font-based icons via `FontIcon` nodes. Icons are referenced by string literals (e.g. `"mdal-info"`, `"mdmz-warning"`). Notification severity types have default icons that inherit colour from AtlantaFX severity classes automatically. See [UI Architecture §7: Icons](../research/UI_ARCHITECTURE.md) for the full icon reference.

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

The menu bar itself is also pure data. A function assembles descriptors into a cljfx MenuBar spec, and the result is rendered like any other cljfx component. This is the same pure-map-to-cljfx pattern used for windows and dialogs.

**Accelerators** in Ooloi always use `:mods [:shortcut ...]` — never hardcoded `:meta` or `:control`. The `:shortcut` modifier resolves to Cmd on macOS and Ctrl on other platforms at the JavaFX/cljfx level.

**macOS system menu bar** (the menu bar embedded in the macOS top bar rather than in the window frame) is controlled by `:use-system-menu-bar` on the MenuBar spec. MenuBar specs are produced by the UI toolkit layer, which injects this flag based on an injectable OS predicate. Developer-authored and plugin-authored specs never contain this flag — infrastructure decides, specifications don't know.

**Stub commands** are descriptors with `(constantly false)` as their enablement predicate and no `:on-action`. They exist to stabilise menu structure during early development. They are intentionally inert and do not imply feature commitment.

### Validation Strategy

```clojure
(defn validate-window-spec
  "Validates window specification map following ADR-0018 event validation pattern.

   Validates:
   - Required :fx/type field must be present
   - Persistent windows must have :window/id
   - Position/size maps have correct structure
   - Field names are kebab-case keywords

   Returns: spec if valid
   Throws: ExceptionInfo with descriptive message if validation fails"
  [spec]
  (when-not (map? spec)
    (throw (ex-info "Window spec must be a map" {:spec spec})))

  (when-not (:fx/type spec)
    (throw (ex-info "Window spec missing required :fx/type" {:spec spec})))

  (when (and (:window/persist? spec) (not (:window/id spec)))  
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

```clojure
(defn create-window
  "Creates JavaFX Stage from window spec with automatic lifecycle management.

   Lifecycle:
   - Validates spec
   - Loads saved state if :window/persist? true
   - Merges saved state with defaults
   - Creates JavaFX Stage via cljfx
   - Applies configured theme automatically (AtlantaFX)
   - Attaches close handler to save state

   Parameters:
   - spec: Window specification map (pure cljfx + :window/ metadata)

   Returns: JavaFX Stage with theme applied"
  [spec]
  (validate-window-spec spec)

  (let [window-id (:window/id spec)
        persist? (:window/persist? spec)

        ;; Load saved state if persistent
        saved-state (when (and persist? window-id)
                     (persistence/load-window-state window-id))

        ;; Merge: saved state > defaults from spec
        final-position (or (:position saved-state)
                          (:window/default-position spec))
        final-size (or (:size saved-state)
                      (:window/default-size spec))
        final-scale (or (:scale saved-state)
                       (:window/default-scale spec)
                       1.0)

        ;; Render cljfx spec to JavaFX Stage (theme applied automatically)
        stage (render-cljfx-to-stage spec)]

    ;; Apply window state
    (when final-position
      (.setX stage (:x final-position))
      (.setY stage (:y final-position)))
    (when final-size
      (.setWidth stage (:width final-size))
      (.setHeight stage (:height final-size)))

    ;; Attach close handler for persistence
    (when persist?
      (.setOnCloseRequest stage
        (event-handler [_]
          (persistence/save-window-state
            {:id window-id
             :title (.getTitle stage)
             :position {:x (int (.getX stage))
                        :y (int (.getY stage))}
             :size {:width (int (.getWidth stage))
                    :height (int (.getHeight stage))}
             :scale final-scale}))))

    stage))
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
;; Window manager receives spec (from plugin or local code)
(defn show-window [spec]
  (let [stage (create-window spec)]
    (.show stage)
    stage))

;; No conversion needed - spec IS cljfx
(show-window {:fx/type :stage
              :window/id :tool-palette
              :window/persist? true
              :window/title-key :window.tool-palette.title  ; → "Tools"
              :scene ...})

;; Theme application is automatic
(defn render-cljfx-to-stage [spec]
  "Renders cljfx spec to JavaFX Stage with automatic theme application."
  (let [stage (Stage.)
        scene (create-scene-from-spec spec)]
    ;; Apply configured theme automatically
    (apply-current-theme! scene)
    (.setScene stage scene)
    stage))

(defn apply-current-theme! [scene]
  "Applies configured UI theme to Scene. Reads theme preference from settings."
  (let [theme-name (settings/get ::ui-theme :nord-dark)
        theme (case theme-name
                :nord-dark (NordDark.)
                :nord-light (NordLight.)
                ;; Additional AtlantaFX themes...
                (NordDark.))]  ; default fallback
    (.add (.getStylesheets scene) (.getUserAgentStylesheet theme))))
```

### With Persistence Layer

```clojure
;; On window close, extract state and save
(.setOnCloseRequest stage
  (event-handler [_]
    (when (:window/persist? spec)
      (persistence/save-window-state
        {:id (:window/id spec)
         :position {:x (int (.getX stage)) :y (int (.getY stage))}
         :size {:width (int (.getWidth stage)) :height (int (.getHeight stage))}
         :scale (get-current-scale stage)}))))

;; On window create, load and apply saved state
(when-let [saved (persistence/load-window-state (:window/id spec))]
  (.setX stage (:x (:position saved)))
  (.setY stage (:y (:position saved)))
  (.setWidth stage (:width (:size saved)))
  (.setHeight stage (:height (:size saved))))
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

Backend and remote plugins use symbolic event handlers (maps) because functions don't serialize over gRPC:

```clojure
;; Backend plugin sends symbolic handler
{:fx/type :button
 :text-key :common.save  ; → "Save"
 :on-action {:event/type :plugin/save-settings
             :plugin-id :my-plugin}}

;; Frontend resolves at render time
(defn resolve-event-handler [handler-spec]
  (fn [event]
    (send-to-backend (:event/type handler-spec) handler-spec)))
```

**Frontend Plugins (Local):**

Frontend plugins can use direct callback functions since they don't cross gRPC boundary:

```clojure
;; Frontend plugin uses direct function reference
{:fx/type :button
 :text-key :common.save  ; → "Save"
 :on-action (fn [event]
               ;; Direct function call - no serialization needed
               (save-local-settings!))}

;; OR using symbolic handlers if preferred for consistency
{:fx/type :button
 :text-key :common.save
 :on-action {:event/type :frontend-plugin/save-settings}}
```

**Key Distinction:**
- **Serialization boundary determines handler format**
- Backend plugins → symbolic handlers (required)
- Frontend plugins → direct functions (allowed) or symbolic (optional)
- Same cljfx spec format works for both

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
- **ADR-0038:** Backend-Authoritative Rendering - Defines cljfx responsibilities ("Windows, dialogs, menus, input handlers")
- **ADR-0039:** Localisation Architecture - Requires all UI strings use translation keys
- **ADR-0003:** Plugins - Plugin architecture requirements
- **ADR-0043:** Frontend Settings - Application preferences system with registry-driven Settings dialog
