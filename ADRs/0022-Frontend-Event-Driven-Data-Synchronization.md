# ADR-0022: Frontend Event-Driven Data Synchronization Architecture

## Status

Proposed

## Context

Ooloi's frontend-backend separation ([ADR-0001](0001-Frontend-Backend-Separation.md)) creates the need for explicit data synchronization patterns between the JavaFX/Skija frontend and the Clojure backend. The frontend must efficiently handle real-time collaborative editing, large orchestral scores (30-40 staves), and responsive user interactions while maintaining clean architectural boundaries.

### Performance and Collaboration Requirements

**Large Score Performance**:
- Complex orchestral works with thousands of measures across multiple staves
- Real-time rendering updates without UI blocking
- Efficient memory usage for viewport-based display
- Smooth scrolling and navigation through large documents

**Real-Time Collaboration**:
- Multiple users editing simultaneously with immediate visual feedback
- Coordinated handling of simultaneous edits to same elements
- Event ordering and conflict resolution at the UI level
- Responsive user experience during network operations

### Data Flow Challenges

**Interactive UI Requirements**:
- User clicks on rendered notation elements (notes, symbols, measures)
- Frontend must identify clicked elements and map to backend operations
- Immediate visual feedback before backend confirmation
- Graceful handling of backend operation failures

**Synchronization Complexity**:
- Frontend layout objects must stay synchronized with backend musical data
- Backend layout calculations may invalidate multiple UI regions
- Updates may propagate across staves, systems, pages, or entire layouts
- Network latency requires asynchronous update patterns

## Decision

We will implement an **event-driven data synchronization architecture using direct JavaFX** with five key components:

### 1. Frontend Layout Object Hierarchy

**Mirror Backend Structure**: Frontend maintains object hierarchy parallel to backend Layout→PageView→SystemView→StaffView→MeasureView→Glyph/Curve structure.

**View Objects Pattern**:
```clojure
;; Frontend layout objects are "views" of backend data, not independent copies
(defrecord FrontendMeasureView [vpd backend-version glyphs curves dirty?])
(defrecord FrontendGlyph [backend-item-id position bounds glyph-data])
```

**Key Principles**:
- Frontend objects contain **backend identification metadata** (VPDs, item IDs, version stamps)
- Frontend objects are **views of backend state**, not authoritative copies
- Frontend **never performs musical logic** - pure UI coordination and rendering
- Layout calculations remain **exclusively in backend** - frontend receives computed results
- **Backend-computed drawing instructions are cached** by frontend until backend invalidation
- **Frontend rendering uses cached instructions** - no local glyph or curve computation

### 2. Event-Driven Synchronization Pattern

**Interaction Flow**:
```
User Interaction → Immediate UI Feedback → gRPC Call → Backend Processing → Event Stream → Frontend Updates
```

**Implementation Pattern**:
```clojure
;; 1. User clicks on glyph
(defn handle-glyph-click [glyph-id]
  ;; Immediate visual feedback
  (update-ui-selection glyph-id)
  ;; Fire gRPC call (non-blocking)
  (grpc/add-articulation (:vpd glyph) (:piece-id @app-state) :staccato)
  ;; Result arrives via event stream, not return value
  )

;; 2. Backend events update frontend
(defn handle-layout-update-event [event]
  (case (:type event)
    :measure-view-updated (update-measure-view (:vpd event) (:measure-view event))
    :layout-invalidated   (invalidate-regions (:affected-regions event))
    :piece-changed        (refresh-entire-layout (:piece-id event))))
```

**Event Streaming Architecture**:
- **Server streaming gRPC** for real-time layout updates from backend
- **Event classification**: measure updates, region invalidation, full layout refresh
- **Drawing instruction cache invalidation**: Backend signals when cached glyphs/curves are stale
- **Asynchronous processing**: UI never blocks waiting for backend responses
- **Event ordering**: Backend ensures correct update sequence

### 3. Glyph Identification Strategy

**Backend Linking System**:
```clojure
;; Each frontend glyph contains backend identification and cached drawing instructions
(defrecord FrontendGlyph [
  vpd              ; Vector Path Descriptor to parent measure
  item-id          ; Unique identifier within measure
  backend-version  ; Version stamp for staleness detection
  bounds           ; UI hit-testing rectangle
  drawing-instructions ; Backend-computed Skija drawing instructions (cached)
  cache-valid?     ; Flag indicating if cached instructions are current
])
```

**Hit-Testing to Backend Mapping**:
```clojure
(defn resolve-click-to-backend [click-point viewport]
  (->> (find-glyphs-at-point click-point viewport)
       (map #(select-keys % [:vpd :item-id]))
       (first))) ; Frontend resolves geometry, backend handles musical logic
```

**Drawing Instruction Caching**:
```clojure
(defn render-glyph [glyph canvas]
  (if (:cache-valid? glyph)
    ;; Use cached backend-computed instructions
    (skija/render-instructions canvas (:drawing-instructions glyph))
    ;; Request fresh instructions from backend
    (request-glyph-recomputation (:vpd glyph) (:item-id glyph))))

(defn invalidate-drawing-cache [glyph]
  (assoc glyph :cache-valid? false :drawing-instructions nil))
```

**Identification Principles**:
- **VPD + item-id** uniquely identifies any musical element for backend operations
- **Hit-testing** resolves user clicks to backend identifiers
- **No musical interpretation** in frontend - pure geometric-to-logical mapping
- **Version tracking** prevents stale data operations
- **Backend-computed drawing instructions** are cached until backend signals invalidation
- **Frontend never computes glyph shapes** - all visual rendering comes from backend

### 4. Performance Optimization Framework

**Dirty Region Tracking**:
```clojure
(defrecord Viewport [
  visible-bounds    ; Current viewport rectangle
  dirty-regions     ; Set of regions needing redraw
  layout-objects    ; Visible frontend layout objects
  update-throttle   ; Batch update timing control
])

(defn mark-region-dirty [viewport region-bounds]
  (update viewport :dirty-regions conj region-bounds))
```

**Batch Update Pattern**:
- **10 FPS throttling**: Batch multiple backend events into single UI update
- **Dirty region accumulation**: Collect all affected regions before redraw
- **STM viewport management**: Thread-safe viewport state updates
- **Virtual scrolling**: Only render visible portions of large scores

**Performance Strategies**:
- **Incremental updates**: Only redraw affected regions, not entire layout
- **Event batching**: Accumulate rapid backend changes before UI update
- **Memory efficiency**: Viewport-based object lifecycle management
- **Rendering optimization**: Skija dirty region tracking for minimal GPU work

## Rationale

### Event-Driven vs Polling Architecture

**Why Event-Driven**:
- **Real-time collaboration**: Immediate propagation of collaborative edits
- **Network efficiency**: No constant polling overhead
- **Resource optimization**: Updates only when backend state changes
- **Scalability**: Server can manage many concurrent clients efficiently

**vs Polling Alternative**:
- Polling would create network overhead and delayed updates
- Difficult to achieve smooth collaborative experience with polling
- Backend would need to track "what changed since timestamp X" per client

### Frontend Layout Mirrors vs Independent UI State

**Why Mirror Backend Structure**:
- **Conceptual clarity**: 1:1 correspondence between backend and frontend objects
- **Debugging simplicity**: Easy to trace UI issues to backend state
- **Consistent abstractions**: Same hierarchical thinking in both components
- **Synchronization predictability**: Clear mapping for event updates

**vs Independent UI State**:
- Independent state would require complex translation layers
- Risk of frontend-backend model drift and synchronization bugs
- More difficult to maintain consistency during collaborative editing

### Glyph Identification vs Geometric Calculation

**Why Backend Identification Metadata**:
- **Reliability**: Backend provides authoritative element identification
- **Consistency**: Same identification system for local and collaborative operations
- **Simplicity**: Frontend doesn't need to reverse-engineer musical relationships
- **Performance**: No complex geometric-to-musical mapping algorithms in UI

**vs Pure Geometric Calculation**:
- Geometric calculation would duplicate musical logic in frontend
- Fragile to layout algorithm changes in backend
- Difficult to maintain accuracy for complex notation cases

### Batch Updates vs Immediate Rendering

**Why Batch Updates (10 FPS)**:
- **Performance**: Prevents UI thread overload during rapid backend changes
- **User experience**: Smooth visual updates without flicker
- **Network efficiency**: Processes multiple events efficiently
- **Resource management**: Reduces GPU rendering overhead

**vs Immediate Rendering**:
- Immediate rendering could overwhelm UI during collaborative editing
- Would create flickering and poor user experience
- Inefficient GPU usage for rapid consecutive updates

## Consequences

### Positive

1. **Responsive Collaborative Editing**: Multiple users can edit simultaneously with immediate visual feedback and coordinated conflict resolution
2. **Efficient Large Score Handling**: Virtual scrolling and dirty region tracking enable smooth navigation through complex orchestral works
3. **Clean Separation of Concerns**: Frontend focuses purely on UI coordination while backend handles all musical logic
4. **Real-Time Synchronization**: Event streaming provides immediate updates across all connected clients
5. **Performance Optimization**: Batch updates and viewport management provide smooth user experience
6. **Maintainable Architecture**: Clear frontend-backend responsibilities simplify debugging and development

### Negative

1. **Increased Complexity**: Event-driven patterns require more sophisticated state management than simple request-response
2. **Network Dependency**: UI updates depend on backend event streaming, creating potential responsiveness issues
3. **Event Ordering Challenges**: Complex collaborative scenarios may require careful event sequencing
4. **Memory Overhead**: Frontend layout objects add memory usage beyond pure rendering needs
5. **Testing Complexity**: Asynchronous event flows require more complex testing strategies

### Mitigations

- **Comprehensive error handling**: Graceful fallback when event streaming fails
- **Local state backup**: Frontend maintains enough state for continued interaction during network issues
- **Event replay capability**: Backend can resynchronize clients after connection recovery
- **Performance monitoring**: Built-in metrics for event processing and UI update performance
- **Testing infrastructure**: Event simulation and async testing patterns

### 5. Organic Pulsating Selection Animation

**Innovative UI Paradigm**: Ooloi introduces organic pulsating selection animation that brings a new dimension to music notation UI, reflecting the living, breathing nature of music itself.

**Animation Implementation Pattern**:
```clojure
;; Organic pulsating selection with varied, asynchronous rhythms
(defn create-organic-pulse-animation [element element-index]
  (let [timeline (Timeline.)
        ;; Each element gets unique timing and intensity variations
        base-duration (+ 1800 (rand-int 800))     ; 1.8-2.6 second base cycles
        phase-offset (* element-index 0.3)        ; Stagger start times
        intensity-factor (+ 0.8 (* 0.4 (rand)))  ; 80-120% intensity variation
        pulse-scale-factor (* 1.12 intensity-factor) ; Varied scale (90-135%)
        opacity-variation (* 0.25 intensity-factor)  ; Varied opacity (20-30%)
        
        scale-x (.getScaleX element)
        scale-y (.getScaleY element)
        base-opacity (.getOpacity element)]
    
    ;; Create unique organic breathing pattern for this element
    (.addAll (.getKeyFrames timeline)
             [(KeyFrame. (Duration/millis (* phase-offset 1000))
                        (KeyValue. (.scaleXProperty element) scale-x Interpolator/EASE_BOTH)
                        (KeyValue. (.scaleYProperty element) scale-y Interpolator/EASE_BOTH)
                        (KeyValue. (.opacityProperty element) base-opacity Interpolator/EASE_BOTH))
              (KeyFrame. (Duration/millis (+ (* phase-offset 1000) (/ base-duration 2)))
                        (KeyValue. (.scaleXProperty element) (* scale-x pulse-scale-factor) Interpolator/EASE_BOTH)
                        (KeyValue. (.scaleYProperty element) (* scale-y pulse-scale-factor) Interpolator/EASE_BOTH) 
                        (KeyValue. (.opacityProperty element) (+ base-opacity opacity-variation) Interpolator/EASE_BOTH))
              (KeyFrame. (Duration/millis (+ (* phase-offset 1000) base-duration))
                        (KeyValue. (.scaleXProperty element) scale-x Interpolator/EASE_BOTH)
                        (KeyValue. (.scaleYProperty element) scale-y Interpolator/EASE_BOTH)
                        (KeyValue. (.opacityProperty element) base-opacity Interpolator/EASE_BOTH))])
    
    ;; Configure infinite organic cycling with element-specific rate
    (.setCycleCount timeline Timeline/INDEFINITE)
    (.setAutoReverse timeline false)
    timeline))

(defn apply-organic-selection [elements]
  ;; Create individual animations for each element with unique characteristics
  (let [animations (vec (map-indexed 
                          (fn [idx element] 
                            (let [timeline (create-organic-pulse-animation element idx)]
                              (.play timeline)
                              timeline))
                          elements))]
    {:stop (fn [] (doseq [animation animations] (.stop animation)))
     :pause (fn [] (doseq [animation animations] (.pause animation)))
     :resume (fn [] (doseq [animation animations] (.play animation)))}))
```

**Skija Integration for Organic Effects**:
```clojure
;; Enhanced rendering with varied organic glow effects per element
(defn render-selected-element-with-glow [skija-canvas element element-id timestamp]
  (let [;; Each element gets unique animation characteristics based on its ID
        element-seed (hash element-id)
        base-frequency (+ 0.8 (* 0.6 (mod element-seed 100) 0.01))  ; 0.8-1.4 Hz variation
        phase-offset (* 2 Math/PI (mod element-seed 1000) 0.001)     ; Unique phase offset
        current-phase (+ phase-offset (* base-frequency timestamp 0.001))
        
        ;; Organic glow characteristics that vary per element
        glow-intensity (+ 0.6 (* 0.4 (Math/sin current-phase)))     ; Breathing intensity
        glow-radius (* 12 glow-intensity (+ 0.8 (* 0.3 (Math/sin (* current-phase 1.3))))) ; Varied radius
        
        ;; Color variations that shimmer and shift per element
        hue-shift (* 30 (Math/sin (* current-phase 0.7)))           ; ±30° hue variation
        base-hue (+ 220 hue-shift)                                   ; Blue with shimmer
        saturation (+ 0.6 (* 0.3 (Math/sin (* current-phase 1.1)))) ; Breathing saturation
        brightness (+ 0.8 (* 0.2 glow-intensity))                   ; Synchronized brightness
        
        glow-color (Color/getHSBColor (/ base-hue 360) saturation brightness)]
    
    ;; Draw organic glow effect with element-specific characteristics
    (with-paint [glow-paint (.setColor (Paint.) glow-color)
                             (.setMaskFilter (MaskFilter/makeBlur FilterBlurMode/NORMAL glow-radius))]
      (render-element-outline skija-canvas element glow-paint))
    
    ;; Draw main element with subtle breathing effect
    (render-element skija-canvas element)))

(defn update-organic-animation-system [selected-elements timestamp]
  ;; Update each element's unique organic animation state
  (doseq [[element element-id] (map vector selected-elements (range))]
    (let [element-seed (hash element-id)
          base-frequency (+ 0.9 (* 0.4 (mod element-seed 100) 0.01)))  ; Unique frequency
      ;; Update element-specific animation states for rendering
      (update-element-organic-state element element-id timestamp base-frequency))))
```

**Selection State Management**:
```clojure
(defn create-organic-selection-manager []
  (let [selected-elements (atom {})  ; Map of element -> {:id, :animation, :start-time}
        element-counter (atom 0)]
    {:select-element
     (fn [element]
       (let [element-id (swap! element-counter inc)
             start-time (System/currentTimeMillis)
             animation (create-organic-pulse-animation element element-id)]
         (.play animation)
         (swap! selected-elements assoc element 
                {:id element-id 
                 :animation animation 
                 :start-time start-time})))
     
     :select-multiple-elements
     (fn [elements]
       ;; Add multiple elements with staggered timing for organic effect
       (doseq [[idx element] (map-indexed vector elements)]
         (Thread/sleep (* idx 50))  ; 50ms stagger between additions
         ((:select-element this) element)))
     
     :deselect-element
     (fn [element]
       (when-let [element-data (@selected-elements element)]
         (.stop (:animation element-data))
         (swap! selected-elements dissoc element)))
     
     :clear-selection
     (fn []
       (doseq [[element element-data] @selected-elements]
         (.stop (:animation element-data)))
       (reset! selected-elements {}))
     
     :get-living-selection
     (fn []
       ;; Return elements with their organic animation metadata
       @selected-elements)}))
```

**Organic Animation Principles**:
- **Asynchronous rhythms**: Each element pulses at unique rates (1.8-2.6 second cycles) preventing synchronization
- **Varied intensities**: Scale changes range from 90-135% with per-element randomization  
- **Staggered phase offsets**: Elements start pulsing at different times creating wave-like effects
- **Individual glow characteristics**: Each element has unique color shimmer, radius variation, and frequency
- **Living color system**: Hue shifts ±30° with breathing saturation and brightness per element
- **Organic frequency spread**: 0.8-1.4 Hz base frequencies ensure natural variation without chaos
- **Staggered selection**: Multiple elements added with 50ms delays for organic appearance
- **Performance optimization**: Animation only applied to visible selected elements with element-specific timelines

## JavaFX and Skija Technical Implementation Foundation

### JavaFX Windowing and UI Capabilities

**Core Window Management System:**
JavaFX provides comprehensive Stage management for professional applications with multiple windows, floating palettes, modal dialogs, and docking systems. Stage objects represent desktop windows, with the primary Stage provided by the platform and additional Stage objects constructed by the application.

**Professional Application Window Types:**
- **Main Score Window**: Primary editing interface with MenuBar, ToolBar, and Canvas
- **Floating Palettes**: Instrument libraries, symbol palettes, properties panels
- **Modal Dialogs**: Preferences, settings, file operations with APPLICATION_MODAL or WINDOW_MODAL
- **Popup Windows**: Context menus, quick tool selection, tooltips with rich content
- **Dockable Panels**: Using libraries like DockFX for Visual Studio/Eclipse-style interfaces

### Hit Testing and Object Mapping Implementation

**JavaFX Canvas Limitations and Solutions:**
JavaFX Canvas does not provide automatic hit testing for custom-drawn content, requiring manual object tracking and geometric calculations. The recommended approach combines JavaFX for UI framework with custom hit testing implementation:

```clojure
;; Direct JavaFX hit testing implementation using Clojure
(defn create-hit-tester []
  (let [elements (atom [])]
    {:add-element (fn [element] (swap! elements conj element))
     :hit-test (fn [x y]
                 ;; Reverse iteration for top-to-bottom hit testing
                 (->> @elements
                      reverse
                      (some #(when (.contains (.getBounds %) x y)
                               (.getMusicalElement %)))))}))

(defn handle-canvas-click [canvas x y hit-tester]
  (when-let [element ((:hit-test hit-tester) x y)]
    (handle-element-selection element)))
```

**Performance Optimization Strategies:**
- **Canvas vs Node Trade-offs**: Canvas provides ~10x more objects before framerate drops vs Nodes
- **Spatial Optimization**: Quadtree data structures for large scores with many elements
- **Viewport Culling**: Only process hit testing for visible elements
- **Bounding Box Pre-filtering**: Fast rectangular checks before detailed shape testing

### Pagination and Viewport Performance Benefits

**JavaFX Version Considerations Mitigated:**
With pagination as a central feature and viewport-based rendering, JavaFX performance concerns become less critical:
- **JavaFX 8-17**: Excellent performance for large datasets, but pagination makes this less relevant
- **JavaFX 18+**: Performance issues with 10M+ items don't apply to paginated content
- **Typical page content**: 4-8 systems × 4-8 measures = 16-64 measures maximum visible

**Memory Efficiency with Page-Based Architecture:**
```clojure
;; Direct JavaFX page cache implementation
(defn create-page-cache [max-cached-pages]
  (let [cache (atom (into {} (map vector) (repeat nil) (range max-cached-pages)))]
    {:get-page (fn [page-number]
                 (or (@cache page-number)
                     (let [page-data (load-page page-number)]
                       (swap! cache assoc page-number page-data)
                       page-data)))
     :invalidate-page (fn [page-number]
                        (swap! cache dissoc page-number))}))
```

### Skija Integration for High-Quality Graphics

**Superior Rendering Capabilities:**
Skija provides Java bindings for Skia graphics library used by Google Chrome, Android, Flutter, offering:
- **Modern typography** for musical symbols (SMuFL fonts)
- **GPU acceleration** for smooth scrolling and rendering
- **High-quality graphics** for both screen display and print output
- **Cross-platform consistency** with automatic memory management

**JavaFX-Skija Integration Pattern:**
```clojure
;; Direct JavaFX Canvas + Skija rendering using Clojure
(defn create-skija-canvas [width height]
  (let [canvas (Canvas. width height)
        graphics-context (.getGraphicsContext2D canvas)]
    {:canvas canvas
     :render (fn [skija-drawing-fn]
               ;; Create Skija surface
               (let [skija-surface (.makeRasterN32Premul Surface width height)]
                 ;; Execute Skija drawing operations
                 (skija-drawing-fn (.getCanvas skija-surface))
                 ;; Convert and draw to JavaFX
                 (let [javafx-image (convert-skija-to-javafx-image skija-surface)]
                   (.drawImage graphics-context javafx-image 0 0))))}))

(defn render-musical-notation [canvas-info notation-data]
  ((:render canvas-info)
   (fn [skija-canvas]
     ;; Use backend-provided drawing instructions
     (doseq [instruction (:drawing-instructions notation-data)]
       (execute-skija-instruction skija-canvas instruction)))))
```

### Professional UI Component Architecture

**Docking System Integration:**
Using DockFX library for comprehensive docking capabilities:
```clojure
;; Direct JavaFX docking system setup
(defn create-docking-system []
  (let [dock-pane (DockPane.)]
    {:dock-pane dock-pane
     :setup-docking (fn [score-canvas instrument-palette]
                      ;; Main score area (center)
                      (let [score-node (DockNode. score-canvas "Score")]
                        (.dock score-node dock-pane DockPos/CENTER))
                      ;; Dockable palettes
                      (let [instrument-node (DockNode. instrument-palette "Instruments")]
                        (.dock instrument-node dock-pane DockPos/LEFT)
                        (.setFloatable instrument-node true)))
     :add-palette (fn [palette-content palette-name position floatable?]
                    (let [palette-node (DockNode. palette-content palette-name)]
                      (.dock palette-node dock-pane position)
                      (.setFloatable palette-node floatable?)))}))
```

**Window State Management:**
```clojure
;; Direct JavaFX window state management using Java Preferences API
(defn create-window-state-manager [app-class]
  (let [prefs (.userNodeForPackage Preferences app-class)]
    {:save-window-state 
     (fn [primary-stage palettes]
       ;; Save main window bounds and state
       (.putDouble prefs "main.x" (.getX primary-stage))
       (.putDouble prefs "main.y" (.getY primary-stage))
       (.putDouble prefs "main.width" (.getWidth primary-stage))
       (.putDouble prefs "main.height" (.getHeight primary-stage))
       ;; Save palette visibility and positions
       (doseq [[name stage] palettes]
         (.putBoolean prefs (str name ".visible") (.isShowing stage))
         (when (.isShowing stage)
           (.putDouble prefs (str name ".x") (.getX stage))
           (.putDouble prefs (str name ".y") (.getY stage)))))
     :restore-window-state
     (fn [primary-stage palettes]
       ;; Restore main window
       (.setX primary-stage (.getDouble prefs "main.x" 100.0))
       (.setY primary-stage (.getDouble prefs "main.y" 100.0))
       ;; Restore palettes
       (doseq [[name stage] palettes]
         (when (.getBoolean prefs (str name ".visible") false)
           (.show stage)
           (.setX stage (.getDouble prefs (str name ".x") 200.0))
           (.setY stage (.getDouble prefs (str name ".y") 200.0)))))}))
```

**Dynamic Toolbar and Menu System:**
```clojure
;; Direct JavaFX toolbar management
(defn create-toolbar-manager []
  (let [toolbars (atom {})]
    {:register-toolbar (fn [name toolbar] (swap! toolbars assoc name toolbar))
     :show-floating-toolbar 
     (fn [name]
       (when-let [toolbar (@toolbars name)]
         (let [floating-window (Stage.)
               scene (Scene. (VBox. (into-array Node [toolbar])))]
           (.initStyle floating-window StageStyle/UTILITY)
           (.setScene floating-window scene)
           (.show floating-window))))
     :create-context-menu
     (fn [items]
       (let [context-menu (ContextMenu.)]
         (doseq [{:keys [text action]} items]
           (let [menu-item (MenuItem. text)]
             (.setOnAction menu-item (event-handler action))
             (.add (.getItems context-menu) menu-item)))
         context-menu))}))
```

### Advanced UI Patterns and Capabilities

**Modal Dialog Management:**
```clojure
;; Direct JavaFX modal dialog creation
(defn create-modal-dialog [primary-stage title content-fn]
  (let [dialog (Stage.)]
    (.initStyle dialog StageStyle/DECORATED)
    (.initOwner dialog primary-stage)
    (.initModality dialog Modality/APPLICATION_MODAL) ; Blocks all windows
    (.setTitle dialog title)
    (.setScene dialog (Scene. (content-fn)))
    {:show (fn [] (.show dialog))
     :show-and-wait (fn [] (.showAndWait dialog))
     :close (fn [] (.close dialog))
     :stage dialog}))

(defn create-preferences-dialog [primary-stage preferences-data]
  (create-modal-dialog 
    primary-stage 
    "Preferences"
    (fn [] 
      ;; Create preferences UI using direct JavaFX
      (let [tabs (TabPane.)]
        (doseq [{:keys [tab-name content]} preferences-data]
          (let [tab (Tab. tab-name content)]
            (.add (.getTabs tabs) tab)))
        tabs))))
```

**Context-Sensitive UI Elements:**
```clojure
;; Direct JavaFX context menu creation based on musical element
(defn create-score-context-menu [musical-element]
  (let [context-menu (ContextMenu.)]
    ;; Add context-sensitive menu items based on element type
    (when (note? musical-element)
      (let [add-articulation (MenuItem. "Add Articulation")]
        (.setOnAction add-articulation 
                      (event-handler #(show-articulation-palette musical-element)))
        (.add (.getItems context-menu) add-articulation)))
    (when (measure? musical-element)
      (let [change-time-sig (MenuItem. "Change Time Signature")]
        (.setOnAction change-time-sig
                      (event-handler #(show-time-signature-dialog musical-element)))
        (.add (.getItems context-menu) change-time-sig)))
    context-menu))

(defn show-context-menu [canvas x y musical-element]
  (let [context-menu (create-score-context-menu musical-element)]
    (.show context-menu canvas x y)))
```

**Rich Tooltip System:**
```clojure
;; Direct JavaFX rich tooltip creation
(defn create-smart-tooltip [musical-element]
  (let [tooltip (Tooltip.)
        content (VBox.)
        element-label (Label. (str "Element: " (type musical-element)))
        duration-label (Label. (str "Duration: " (get-duration musical-element)))
        position-label (Label. (str "Position: " (get-position musical-element)))]
    ;; Add styling for rich content
    (.setStyle element-label "-fx-font-weight: bold;")
    (.addAll (.getChildren content) [element-label duration-label position-label])
    (.setGraphic tooltip content)
    (.setShowDelay tooltip (Duration/millis 500))
    tooltip))

(defn install-tooltip [node musical-element]
  (let [tooltip (create-smart-tooltip musical-element)]
    (.install Tooltip node tooltip)))
```

### Performance and Scalability Framework

**Virtual Scrolling with Pagination:**
```clojure
;; Direct JavaFX virtual scrolling implementation
(defn create-music-scroll-pane [page-height viewport-height]
  (let [visible-page-start (atom 0)
        visible-page-end (atom 0)
        loaded-pages (atom #{})]
    {:update-scroll-position
     (fn [scroll-y]
       (let [start-page (int (/ scroll-y page-height))
             end-page (inc (int (/ (+ scroll-y viewport-height) page-height)))]
         (reset! visible-page-start start-page)
         (reset! visible-page-end end-page)
         ;; Load/unload pages as needed
         (doseq [page (range start-page (inc end-page))]
           (when-not (@loaded-pages page)
             (ensure-page-loaded page)
             (swap! loaded-pages conj page)))
         ;; Unload pages outside visible range + buffer
         (let [buffer 2
               pages-to-keep (set (range (- start-page buffer) (+ end-page buffer 1)))]
           (doseq [page @loaded-pages]
             (when-not (pages-to-keep page)
               (unload-page page)
               (swap! loaded-pages disj page))))))
     :get-visible-pages (fn [] (range @visible-page-start (inc @visible-page-end)))}))
```

**Collaborative Update Optimization:**
```clojure
;; Direct JavaFX collaborative update handling
(defn handle-backend-event [event page-cache]
  (case (:scope event)
    :measure-updated
    (let [affected-page (find-page-containing-measure (:measure-vpd event))]
      ((:invalidate-page page-cache) affected-page)
      (mark-page-for-redraw affected-page))
    
    :page-layout-changed
    (doseq [page (:affected-pages event)]
      ((:invalidate-page page-cache) page)
      (mark-page-for-redraw page))
    
    :full-layout-refresh
    (do
      (clear-all-page-cache page-cache)
      (trigger-full-redraw))))

(defn setup-collaborative-updates [grpc-client page-cache]
  (let [event-stream (grpc/subscribe-to-layout-events grpc-client)]
    (go-loop []
      (when-let [event (<! event-stream)]
        (handle-backend-event event page-cache)
        (recur)))))
```

## Implementation Approach

### Component Integration

**Frontend Layout Manager**:
```clojure
(defrecord LayoutManager [
  viewport-state    ; STM ref containing current viewport
  event-processor   ; Core.async channel for backend events
  render-scheduler  ; Batch update timing coordination
  glyph-registry   ; Hit-testing lookup tables
])
```

**Event Processing Pipeline**:
```clojure
(defn start-event-processing [layout-manager grpc-client]
  (go-loop []
    (when-let [event (<! (grpc/subscribe-to-layout-events grpc-client))]
      (process-layout-event layout-manager event)
      (recur))))
```

### Event Streaming Setup

**gRPC Integration**:
- Server streaming from backend for layout updates
- Client streaming for batch user operations
- Bidirectional streaming for collaborative session management
- Event classification and filtering for efficient updates

### Viewport Management

**STM-Based State**:
```clojure
(def viewport-state (ref {:bounds nil :dirty-regions #{} :objects {}}))

(defn update-viewport [update-fn]
  (dosync
    (alter viewport-state update-fn)))
```

### Testing Strategies

**Event Simulation**:
- Mock event streams for unit testing
- Synthetic collaborative scenarios
- Performance testing with large score simulations
- Network failure and recovery testing

**UI Testing**:
- Hit-testing accuracy verification
- Dirty region tracking validation
- Batch update timing verification
- Memory usage profiling for large scores

## References

### Related ADRs
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Establishes architectural separation requiring this synchronization approach
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Communication protocol enabling event streaming architecture
- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) - Frontend technologies requiring integration with event-driven patterns
- [ADR-0018: API-gRPC Interface Generation](0018-API-gRPC-Interface-Generation.md) - Generated API methods used in frontend-backend communication
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model used for viewport state management
- [ADR-0009: Collaboration](0009-Collaboration.md) - Collaborative features enabled by event streaming architecture

### Technical Documentation
- [Frontend Research](../research/FRONTEND.md) - Detailed exploration of data flow patterns and UI requirements
- [gRPC Technical Deep Dive](../DEV_PLAN_GRPC_DEEPDIVE.md) - Implementation patterns for event streaming

## Notes

This architecture represents a significant evolution from traditional music notation software, which typically uses monolithic architectures with implicit data sharing. The event-driven approach enables Ooloi's vision of seamless collaborative music notation while maintaining clean architectural boundaries and high performance for large orchestral works.

The frontend's role as a "view" of backend state, rather than an independent authority, ensures consistency in collaborative scenarios while allowing for responsive user interactions through immediate visual feedback patterns.