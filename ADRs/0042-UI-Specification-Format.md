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

1. **Works across boundaries** - Backend and plugins must be able to define UI that frontend renders
2. **Supports lifecycle management** - Windows need automatic persistence, event handling, cleanup
3. **Enables plugin independence** - Plugins shouldn't need frontend-specific code
4. **Serializes over gRPC** - UI specs must cross the network boundary
5. **Integrates with cljfx** - Must work naturally with cljfx's declarative UI approach
6. **Handles diverse types** - Persistent windows, transient dialogs, auto-dismiss notifications

### Requirements

**Functional Requirements:**
- Window lifecycle management (create, track state, save on close, restore on reopen, cleanup)
- Type differentiation (persistent windows, modal/non-modal dialogs, notifications, custom components)
- Plugin support (define UI in backend/plugin code, serialize over gRPC, no frontend dependency)
- Event integration (symbolic event handlers as maps, resolved to functions at render time)

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
> "Backend and plugins describe UI using **cljfx maps directly** - no translation layer needed."

Per ADR-0039, all user-facing strings use translation keys (`:window/title-key`, `:text-key`, etc.), resolved by frontend via `(tr key)` at render time.

## Decision

### Approach

UI specifications are **pure Clojure maps** conforming to cljfx structure, augmented with `:window/` namespace-qualified metadata for lifecycle management.

```clojure
;; Plugin/backend sends this over gRPC
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
                            :on-action {:event/type :plugin/save-settings}}]}}}
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
   - Attaches close handler to save state

   Parameters:
   - spec: Window specification map (pure cljfx + :window/ metadata)

   Returns: JavaFX Stage"
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

        ;; Render cljfx spec to JavaFX Stage
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

```clojure
;; Symbolic event handler in spec
{:fx/type :button
 :text-key :common.save  ; → "Save"
 :on-action {:event/type :plugin/save-settings
             :plugin-id :my-plugin}}

;; Frontend resolves at render time
(defn resolve-event-handler [handler-spec]
  (fn [event]
    (send-to-backend (:event/type handler-spec) handler-spec)))

;; Frontend resolves symbolic event handlers at render time
```

## Consequences

**Positive:**
- Plugins can define UI using standard cljfx knowledge
- No translation layer needed between backend and frontend
- gRPC serialization works transparently
- New window types don't require framework changes
- Consistent with existing event pattern
- Simple to reason about and debug

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
