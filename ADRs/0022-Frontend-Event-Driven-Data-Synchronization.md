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

We will implement an **event-driven data synchronization architecture** with four key components:

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

```java
public class MusicNotationHitTester {
    private List<RenderableElement> elements = new ArrayList<>();
    
    public Optional<MusicalElement> hitTest(double x, double y) {
        // Reverse iteration for top-to-bottom hit testing
        for (int i = elements.size() - 1; i >= 0; i--) {
            RenderableElement element = elements.get(i);
            if (element.getBounds().contains(x, y)) {
                return Optional.of(element.getMusicalElement());
            }
        }
        return Optional.empty();
    }
}
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
```java
public class PageCache {
    private int maxCachedPages = 3; // Current + adjacent pages
    private Map<Integer, PageData> cache = new LRU<>();
    
    public PageData getPage(int pageNumber) {
        return cache.computeIfAbsent(pageNumber, this::loadPage);
    }
}
```

### Skija Integration for High-Quality Graphics

**Superior Rendering Capabilities:**
Skija provides Java bindings for Skia graphics library used by Google Chrome, Android, Flutter, offering:
- **Modern typography** for musical symbols (SMuFL fonts)
- **GPU acceleration** for smooth scrolling and rendering
- **High-quality graphics** for both screen display and print output
- **Cross-platform consistency** with automatic memory management

**JavaFX-Skija Integration Pattern:**
```java
// JavaFX Canvas + Skija rendering
Canvas javaFxCanvas = new Canvas(800, 600);
GraphicsContext gc = javaFxCanvas.getGraphicsContext2D();

// Render Skija content to JavaFX
Surface skijaCanvas = Surface.makeRasterN32Premul(width, height);
// ... Skija drawing operations ...
Image skijaImage = convertSkijaToJavaFXImage(skijaCanvas);
gc.drawImage(skijaImage, 0, 0);
```

### Professional UI Component Architecture

**Docking System Integration:**
Using DockFX library for comprehensive docking capabilities:
```java
public class OoloiDockingSystem {
    private DockPane dockPane;
    
    public void setupDocking() {
        // Main score area (center)
        DockNode scoreNode = new DockNode(scoreCanvas, "Score");
        scoreNode.dock(dockPane, DockPos.CENTER);
        
        // Dockable palettes
        DockNode instrumentNode = new DockNode(instrumentPalette, "Instruments");
        instrumentNode.dock(dockPane, DockPos.LEFT);
        instrumentNode.setFloatable(true);
    }
}
```

**Window State Management:**
```java
public class WindowStateManager {
    public void saveWindowState() {
        Preferences prefs = Preferences.userNodeForPackage(OoloiApp.class);
        // Save main window bounds and state
        prefs.putDouble("main.x", primaryStage.getX());
        // Save palette visibility and positions
        palettes.forEach((name, stage) -> {
            prefs.putBoolean(name + ".visible", stage.isShowing());
        });
    }
}
```

**Dynamic Toolbar and Menu System:**
```java
public class OoloiToolbarManager {
    private Map<String, ToolBar> toolbars = new HashMap<>();
    
    public void showFloatingToolbar(String name) {
        ToolBar toolbar = toolbars.get(name);
        Stage floatingWindow = new Stage();
        floatingWindow.initStyle(StageStyle.UTILITY);
        floatingWindow.setScene(new Scene(new VBox(toolbar)));
        floatingWindow.show();
    }
}
```

### Advanced UI Patterns and Capabilities

**Modal Dialog Management:**
```java
public Stage createModalDialog(String title) {
    Stage dialog = new Stage();
    dialog.initStyle(StageStyle.DECORATED);
    dialog.initOwner(primaryStage);
    dialog.initModality(Modality.APPLICATION_MODAL); // Blocks all windows
    return dialog;
}
```

**Context-Sensitive UI Elements:**
```java
public class ScoreContextMenu extends ContextMenu {
    public ScoreContextMenu(MusicalElement element) {
        MenuItem addArticulation = new MenuItem("Add Articulation");
        addArticulation.setOnAction(e -> showArticulationPalette());
        getItems().add(addArticulation);
    }
}
```

**Rich Tooltip System:**
```java
public class SmartTooltip extends Tooltip {
    public SmartTooltip(MusicalElement element) {
        VBox content = new VBox();
        content.getChildren().addAll(
            new Label("Element: " + element.getType()),
            new Label("Duration: " + element.getDuration())
        );
        setGraphic(content);
    }
}
```

### Performance and Scalability Framework

**Virtual Scrolling with Pagination:**
```java
public class MusicScrollPane {
    private int visiblePageStart;
    private int visiblePageEnd;
    
    public void updateScrollPosition(double scrollY) {
        // Calculate which pages are visible
        visiblePageStart = (int) (scrollY / pageHeight);
        visiblePageEnd = (int) ((scrollY + viewportHeight) / pageHeight) + 1;
        
        // Load/unload pages as needed
        for (int page = visiblePageStart; page <= visiblePageEnd; page++) {
            ensurePageLoaded(page);
        }
    }
}
```

**Collaborative Update Optimization:**
```java
public void handleBackendEvent(LayoutUpdateEvent event) {
    switch (event.getScope()) {
        case MEASURE_UPDATED:
            int affectedPage = findPageContainingMeasure(event.getMeasureVPD());
            invalidatePage(affectedPage);
            break;
        case PAGE_LAYOUT_CHANGED:
            event.getAffectedPages().forEach(this::invalidatePage);
            break;
    }
}
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