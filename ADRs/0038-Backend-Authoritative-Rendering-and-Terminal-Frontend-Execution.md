# ADR-0038: Backend-Authoritative Rendering: Terminal Frontend Execution with GPU Acceleration

**Status:** Accepted
**Date:** 2026-01-07

## Table of Contents

- [Context](#context)
- [Decision](#decision)
- [Architecture](#architecture)
  - [Component Architecture](#component-architecture)
  - [Hierarchical Paintlist Structure](#hierarchical-paintlist-structure)
  - [Backend Authority: Paintlist Generation](#backend-authority-paintlist-generation)
  - [Frontend Components](#frontend-components)
  - [Event Flow Architecture](#event-flow-architecture)
  - [Two-Level Cache Architecture](#two-level-cache-architecture)
  - [Memory Management](#memory-management)
  - [Performance Characteristics](#performance-characteristics)
  - [Implementation Phases](#implementation-phases)
- [Rationale](#rationale)
- [Consequences](#consequences)
- [Related ADRs](#related-adrs)
- [Notes](#notes)

## Context

Ooloi's architecture requires clear boundaries between backend musical semantics/layout computation and frontend rendering execution. The frontend must efficiently display complex orchestral scores while maintaining absolute backend authority over all engraving decisions.

### The Rendering Boundary Problem

When layout computation and rendering execution are tightly coupled, achieving certain architectural goals becomes significantly more complex:
- Guaranteeing deterministic rendering across different frontends requires shared rendering libraries or complex synchronization
- Preserving the closed semantic model established in ADR-0035 requires careful boundary enforcement
- Enabling collaborative editing without rendering inconsistencies requires additional coordination mechanisms
- Implementing ADR-0022's lazy event-driven architecture requires clear separation of concerns

### The Terminal Execution Principle

The frontend must be **terminal** - the end of the pipeline that consumes backend decisions but never renegotiates, refines, or reinterprets them. This preserves architectural closure and enables:
- Backend computes all layout via ADR-0028's 6-stage pipeline
- Multiple frontend technologies rendering identical results
- Collaborative consistency without complex synchronization
- Litmus test: discard all frontend rendering state, regenerate from backend → identical output

### Performance Requirements

- Complex orchestral scores
- Smooth scrolling (target 60fps)
- Low latency for visible edits (target <150ms p95)
- Memory efficiency (viewport-proportional, not score-proportional)
- Lazy loading (no downloads until layout windows open)

### Technology Considerations

- GPU acceleration is **required** for acceptable performance with complex orchestral scores
- Skija provides GPU-accelerated vector rendering (Metal/Vulkan/OpenGL backends)
- Software rasterization fallback exists for **error recovery only** (GPU driver failure, not production mode)
- cljfx provides windowing but must NOT draw music notation
- JavaFX/Skija embedding strategy remains provisional
- Specific GPU technology (Skija vs alternatives) is swappable, but GPU acceleration itself is not optional

## Decision

We establish **backend-authoritative rendering with terminal frontend execution**:

1. **Backend Authority:** All musical layout and engraving decisions computed by backend's 6-stage rendering pipeline (ADR-0028), producing paintlists containing vector drawing instructions (glyphs with positions, bezier curves with control points).

2. **Terminal Frontend:** Frontend executes backend-computed drawing instructions without interpretation, refinement, or renegotiation. Rendering is the **end** of the pipeline - consuming decisions, never feeding back.

3. **Two-Level Cache Architecture:**
   - **Level 1 (Authoritative):** Rendering Data Manager stores backend-provided paintlists
   - **Level 2 (Required Execution Cache, Evictable Entries):** Skija Pictures derived strictly from paintlists - records paintlists as GPU command buffers for fast replay

4. **Litmus Test Enforcement:** Discarding all frontend rendering state and regenerating from backend paintlists must produce identical output. If not, architectural closure is broken.

5. **GPU-Accelerated Rendering:** GPU acceleration via Skija (or equivalent technology) is **required for production performance**. Complex orchestral scores (30,000+ objects) require GPU vector rendering to achieve 60fps scrolling targets. Software rasterization fallback exists for error recovery only (GPU driver failure).

6. **Component Boundaries:**
   - cljfx: Windows, dialogs, menus, input handlers (does NOT draw music)
   - Skija: GPU-accelerated drawing execution (does NOT compute layout)
   - Event Router: Protocol adapter for backend events (ADR-0031)
   - Rendering Data Manager: VPD-indexed paintlist storage (ADR-0031)
   - Fetch Coordinator: Priority-based paintlist fetching (ADR-0031)

## Architecture

### Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Backend (Authority)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │   6-Stage Rendering Pipeline (ADR-0028)                   │  │
│  │   Produces: Layout → PageView → SystemView →              │  │
│  │            StaffView → MeasureView{glyphs, curves}        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │   gRPC API: Paintlist Fetch Methods                       │  │
│  │   - fetch-page-view(vpd, piece-id)                        │  │
│  │   - fetch-system-view(vpd, piece-id)                      │  │
│  │   - fetch-staff-view(vpd, piece-id)                       │  │
│  │   - fetch-measure-view(vpd, piece-id)                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↓                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │   gRPC Event Stream                                       │  │
│  │   - :piece-invalidation events with VPDs                  │  │
│  │   - Per-client FIFO delivery (ADR-0024)                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Frontend (Rendering Only)                  │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Event Router (Integrant Component - ADR-0031)             │ │
│  │  - Receives gRPC event stream                              │ │
│  │  - Routes :piece-invalidation to Rendering Data Manager    │ │
│  │  - Category aggregation: 50-100ms batches                  │ │
│  │  - Posts Platform.runLater() per batch                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Rendering Data Manager (ADR-0031)                         │ │
│  │  - VPD-indexed hierarchy: 4 atoms (Page/System/Staff/      │ │
│  │    Measure)                                                │ │
│  │  - Stores backend-provided paintlists                      │ │
│  │  - Tracks staleness per VPD                                │ │
│  │  - Backend paintlists are authoritative cache              │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Fetch Coordinator (ADR-0031)                              │ │
│  │  - 4 priority queues (Critical/High/Normal/Low)            │ │
│  │  - Background threads call gRPC API                        │ │
│  │  - Updates Rendering Data Manager via Platform.runLater()  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Skija GPU Rendering Layer (Required Execution)            │ │
│  │  - Picture recording/replay required for score viewports   │ │
│  │  - Picture cache entries evictable; mechanism not optional │ │
│  │  - GPU context: long-lived (per window/app)                │ │
│  │  - Renders inside JavaFX-provided viewport/node            │ │
│  │  - Invalidates Pictures when paintlists change             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  cljfx UI Shell (Windowing Only - ADR-0022)                │ │
│  │  - Windows, dialogs, palettes, menus                       │ │
│  │  - Input handlers (mouse, keyboard)                        │ │
│  │  - Provides viewport/node for Skija rendering              │ │
│  │  - Does NOT compute or draw music                          │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Hierarchical Paintlist Structure

Backend produces paintlists at multiple hierarchy levels (per ADR-0028):

```
Layout (structural coordination only)
└── PageView Paintlists (backend-computed)
    ↳ Headers, footers, page numbers, copyright notices
    └── SystemView Paintlists (backend-computed)
        ↳ Braces, brackets, instrument names, tempo markings, slurs/beams spanning measures
        └── StaffView Paintlists (backend-computed)
            ↳ Clefs, key signatures, time signatures, staff lines
            └── MeasureView Paintlists (backend-computed)
                ↳ Notes, rests, accidentals, articulations, dynamics, measure-local slurs/ties
```

**Critical Design Principle:** All levels contain backend-computed drawing instructions. Frontend never computes layout or engraving — it fetches and renders backend paintlists at whatever VPD granularity the invalidation event specifies.

### Backend Authority: Paintlist Generation

Backend's 6-stage pipeline produces all drawing instructions:

```clojure
;; Backend pipeline output (simplified conceptual view)
(defrecord MeasureView [vpd glyphs curves metadata])

;; Example MeasureView from backend:
{:vpd [:layouts 0 :page-views 2 :system-views 1 :staff-views 0 :measure-views 47]
 :glyphs [{:type :notehead
           :glyph-name "noteheadBlack"
           :x 120.5
           :y 45.2
           :bounds {:x 118.5 :y 42.2 :width 4.0 :height 6.0}}
          {:type :accidental
           :glyph-name "accidentalSharp"
           :x 110.3
           :y 45.2
           :bounds {:x 108.3 :y 41.2 :width 6.0 :height 8.0}}]
 :curves [{:type :slur
           :points [[115.0 40.0] [125.0 38.0] [135.0 40.0]]
           :control-points [[120.0 36.0] [130.0 36.0]]}]
 :metadata {:measure-number 47
            :width 85.3
            :height 60.0}}
```

**Note on Paintlist Format Design:** The examples above are conceptual/pseudocode. The real paintlist format will be designed pragmatically: if aligning closely with Skija's API syntax speeds up translation (paintlist → GPU commands), we'll do that. Otherwise, we'll keep the format as abstract as possible. The format must be serializable via gRPC and sufficient for hit-testing, but internal structure is an implementation detail.

**Backend computes:**
- Exact glyph positions (x, y coordinates)
- Glyph selection from SMuFL font (ADR-0006)
- Bounding boxes for hit-testing
- Bezier curve control points for slurs, ties, beams
- All spatial relationships and collision avoidance

**Frontend receives these as data** via gRPC API calls and renders them faithfully.

**Paintlist properties:**
- **VPD identity:** each paintlist belongs to a specific VPD
- **Spatial data:** bounding boxes and paths enable hit-testing
- **Semantic links:** optional pointers to musical elements (when elements remain semantic)
- **Graphics support:** user-edited bezier paths with control points (Illustrator-level editing)
- **Immutable:** paintlists do not change until backend recomputes and frontend refetches

### Frontend Components

#### 1. Event Router (Integrant Component)

Routes backend gRPC events to frontend subsystems. This is a protocol adapter, not a unified event bus. JavaFX local events remain in JavaFX.

**Responsibilities:**
- Subscribe to gRPC event streams
- Derive routing category from event type
- Batch events by category with time windows
- Post a single Platform.runLater() per batch
- Coordinate atomic invalidation response: mark RDM stale AND queue FC fetch
- No rendering details; no UI chrome

**Invalidation Coordination:**

For `:piece-invalidation` events, Event Router coordinates RDM and Fetch Coordinator as a unified atomic response:

```clojure
;; Atomic invalidation response
(rdm/mark-stale! rendering-data-manager vpd)              ; Mark stale
(fc/queue-fetch! fetch-coordinator vpd piece-id :critical) ; Queue refetch
```

This atomic coordination ensures no intermediate state where cache is marked stale but no refetch is queued.

#### 2. Rendering Data Manager

VPD-indexed storage of backend-provided paintlists with staleness tracking. This is the authoritative frontend cache layer.

**Responsibilities:**
- Store backend-provided paintlists by VPD
- Track staleness per VPD
- Provide O(1) lookup for rendering
- Coordinate with Fetch Coordinator for refetch requests
- All mutations on JavaFX Application Thread (via Platform.runLater)

**Atomic Operations:**

RDM operations are atomic. `update-paintlist!` updates the paintlist AND clears the stale flag in a single operation. No intermediate state exists where paintlist is fresh but stale flag remains set, or vice versa.

**Not responsible for:**
- Any layout computation
- Any engraving decision logic
- Any generation of drawing instructions

#### 3. Fetch Coordinator

Priority-based fetching of paintlists from backend via normal gRPC API calls. All network work happens off the JavaFX Application Thread; updates are applied via Platform.runLater.

**Responsibilities:**
- Maintain priority queues (Critical/High/Normal/Low)
- Perform background gRPC fetches
- Apply results on JavaFX Application Thread
- Implement retry/backoff/error reporting policy

**Priority Level Usage:**

- **Critical:** Cache invalidations from backend events (visible viewport updates)
- **High:** Viewport scroll demand loading (newly visible content)
- **Normal:** Adjacent viewport prefetch (buffer zones)
- **Low:** Speculative prefetch (anticipatory loading)

Invalidation events queue at Critical priority to ensure visible viewport updates process immediately before speculative prefetch.

**Not responsible for:**
- Determining invalidation semantics (Event Router + RDM)
- Rendering the paintlists (Skija layer)
- Any engraving/layout logic (backend)

#### 4. Skija GPU Rendering Layer

Skija draws the music. cljfx does not.

This layer implements the required execution path: Skija `Picture` objects record backend paintlists as GPU command buffers for fast replay. Correctness comes from the litmus test: discarding all Pictures and regenerating them from paintlists must yield identical output.

**Key properties:**
- Derived strictly from paintlists (one-time recording per paintlist)
- Must be invalidated when paintlist changes
- GPU context is long-lived; never created per render call
- Picture recording mechanism is required for 60fps performance; individual Picture cache entries are evictable (regeneration cost: one-time recording from local paintlist)

#### 5. cljfx UI Shell

cljfx is used for windowing, dialogs, menus, palettes, and input handling. It provides JavaFX nodes ("viewports") into which Skija renders.

**Important:** The exact JavaFX embedding strategy for Skija is an implementation detail. Any "surface → image → drawImage" approach should be considered provisional and not asserted as final.

**cljfx responsibilities:**
- Window management and UI chrome
- Input events (mouse/keyboard) and tool state
- Dialogs, settings, palettes, menus
- Providing the JavaFX rendering node

**cljfx does not:**
- Compute layout
- Generate paintlists
- Draw music (Skija does)

### Event Flow Architecture

#### Typical Edit Scenario (Visible Change)

1. User gesture handled by JavaFX (local event system).
2. Frontend sends an API command to backend (gRPC call). This is the only way rendering can change.
3. Backend updates piece in STM, recomputes affected paintlists via ADR-0028 pipeline.
4. Backend emits `:piece-invalidation` events with VPD scope (ADR-0024 FIFO per client).
5. Event Router batches invalidations (50–100ms), posts a single Platform.runLater.
6. Rendering Data Manager marks affected VPDs stale.
7. Fetch Coordinator queues and performs gRPC fetches for fresh paintlists.
8. Platform.runLater updates paintlists, invalidates Picture cache entries, requests repaint.
9. JavaFX paint callback triggers Skija to render fresh paintlists.

**Latency target:** <150ms p95 for visible edits (to be validated by measurement).

#### Scroll Scenario (Demand Loading)

1. User scroll handled by JavaFX.
2. Viewport calculation determines newly visible VPDs.
3. For missing or stale VPDs, Fetch Coordinator queues fetches (HIGH priority for visible missing).
4. Placeholders may render while fetching.
5. When paintlists arrive, Platform.runLater updates and repaint occurs.

No backend events are required for demand loading. Backend events only indicate staleness due to changes.

#### Reconnection Scenario

On reconnect:
- Resubscribe to event streams
- Invalidate cached rendering state (mark all stale or reset staleness)
- Fetch viewport paintlists at CRITICAL priority
- No event replay is required; frontend simply fetches current state

### Two-Level Cache Architecture

#### Level 1: Rendering Data Manager (Authoritative)

- Stores backend-provided paintlists, keyed by VPD.
- Tracks staleness/freshness.
- Updated only by gRPC fetch results.
- Invalidated only by backend invalidation events (or explicit reconnect policy).

#### Level 2: Skija Pictures (Required Execution Artifacts, Cached for Replay)

- Skija `Picture` objects are GPU command buffers derived from paintlists.
- Picture recording/replay is the required execution path for 60fps with complex scores.
- Individual cache entries may be evicted under memory pressure; correctness unaffected (regeneration: one-time recording from local paintlist).
- Must be invalidated when Level 1 paintlist changes.
- Litmus test: regenerate all Pictures from paintlists → identical output.

**Cache Invalidation Flow:**

```
Backend Event
    ↓
Event Router
    ↓
Platform.runLater()
    ↓
Rendering Data Manager: Mark VPD stale
    ↓
Fetch Coordinator: Queue gRPC fetch
    ↓
Background Thread: api/fetch-measure-view(vpd)
    ↓
Platform.runLater()
    ↓
Rendering Data Manager: Update paintlist (Level 1)
    ↓
Skija Layer: Invalidate Picture (Level 2)
    ↓
Next Render: Regenerate Picture from fresh paintlist
```

**Critical Synchronization Constraint:** Level 1 paintlist update and Level 2 Picture invalidation MUST occur synchronously within the same `Platform.runLater()` callback. They cannot be split across separate callbacks. This atomic execution prevents any rendering window where Level 1 is fresh but Level 2 still holds a stale Picture (or vice versa). Implementation example:

```clojure
(Platform/runLater
  (fn []
    (rdm/update-paintlist! vpd paintlist)           ; Level 1: Update authoritative paintlist
    (skija/invalidate-execution-cache! vpd)         ; Level 2: Invalidate Picture (same callback!)
    (request-repaint! vpd)))                        ; Now safe to render
```

Never split these operations across different `Platform.runLater()` calls - doing so creates a race condition where rendering could observe inconsistent cache state.

### Memory Management

**Paintlists:**
- Stored locally; over time may accumulate to full-piece coverage.
- Between edits, cache is stable: no spontaneous invalidations.
- Loading is lazy: opening a piece downloads no graphical data until layout windows open.

**Pictures:**
- Required execution artifacts for score viewports; cache entries are bounded/evictable.
- Viewport-based or LRU eviction is acceptable under memory pressure.
- Regeneration cost is one-time recording from local paintlists.

### Performance Characteristics

All numeric targets in this document are hypotheses and must be validated by profiling on real hardware:

- Steady-state scrolling target: 60fps (16ms/frame)
- Visible collaborative edit target: <150ms p95
- Scroll-triggered fetch target: <100ms p95 (where network allows)
- Reconnection recovery target: <500ms for viewport refetch

Performance depends on:
- GPU backend (Metal/Vulkan/GL)
- Score complexity and density
- Viewport size and zoom
- Network latency and backend load

### Implementation Phases

#### Phase 1: Infrastructure Foundation (No Music Rendering Yet)

1. Event Router with category aggregation
2. Rendering Data Manager (VPD-indexed paintlists + staleness)
3. Fetch Coordinator (priority queues + gRPC calls)
4. cljfx UI shell (windows, dialogs, menus, palettes - **no music notation rendering**)
5. Backend paintlist API integration and fetch testing

**Note:** This phase builds event/cache infrastructure and UI chrome without rendering music. GPU acceleration becomes critical in Phase 2 when Skija music rendering is introduced.

#### Phase 2: GPU-Accelerated Music Rendering (Skija Integration)

1. Long-lived GPU context lifecycle (Metal/Vulkan/OpenGL)
2. Paintlist → Skija Picture recording (one-time conversion of paintlist to GPU command buffer)
3. Picture cache with strict invalidation on paintlist update
4. JavaFX/Skija viewport integration
5. Correctness validation against reference renderings
6. Performance profiling to validate 60fps target with complex scores

**Note:** GPU acceleration is **vital** in this phase. Skija Pictures (GPU command buffers) are recorded from paintlists once, then replayed every frame. This is essential for 60fps with 30,000+ objects - you cannot re-parse paintlists and issue draw commands every frame.

#### Phase 3: Production Optimization

1. Picture cache eviction policy (LRU based on memory pressure)
2. Prefetch strategy refinement (viewport + buffer)
3. Cache hit rate monitoring and tuning
4. Litmus test enforcement in CI

#### Phase 4: Production Readiness

1. Cross-platform GPU testing (Metal on macOS, Vulkan on Linux/Windows, OpenGL fallback)
2. Memory leak detection and GPU resource management
3. Error recovery: Software rasterization fallback for GPU driver failures (diagnostic mode only, not production performance)
4. Performance metrics, profiling instrumentation, and error reporting

## Rationale

### Why Backend Authority is Absolute

**Deterministic Rendering:** Multiple frontends must render identically. Backend authority provides a single source of truth for all rendering decisions, ensuring consistency without requiring shared rendering libraries or complex synchronization protocols.

**Closed Semantic Model:** ADR-0035 establishes closed semantics (accidentals, musical logic) before rendering. Frontend decisions would reopen the semantic model, violating architectural closure.

**Collaborative Consistency:** Multiple users must see identical rendering. Backend computes once, broadcasts to all. Frontend decisions would require complex synchronization.

**Testability:** Backend rendering pipeline can be tested deterministically. Frontend visual decisions introduce non-determinism and platform dependencies.

### Why Terminal Execution

**No Feedback Loops:** Frontend refinements (spacing adjustments, collision avoidance) would create feedback into layout, violating the rendering boundary.

**Litmus Test Enforcement:** If frontend state matters for correctness, the litmus test fails. Terminal execution makes frontend state irrelevant - regenerate from backend → identical output.

**Simplicity:** Terminal execution eliminates entire classes of bugs (feedback loops, refinement inconsistencies, optimization-dependent behavior).

### Why Two-Level Cache

**Authoritative Paintlists (Level 1):** Stores backend truth. Can be discarded and refetched without correctness issues. This is the data cache containing vector drawing instructions.

**GPU Command Buffers (Level 2):** Skija Pictures record paintlists as GPU command buffers for fast replay. Essential for performance - cannot re-parse 30,000 objects and issue draw commands every frame at 60fps. Must be strictly derived from Level 1 and invalidated when paintlists change.

**Separation of Concerns:** Data cache (paintlists) separate from execution cache (Pictures). Paintlists authoritative and immutable. Pictures regenerable from paintlists. Different eviction policies: paintlists accumulate until memory pressure, Pictures can be evicted aggressively (regeneration cost is one-time recording from local paintlist).

### Why GPU Acceleration is Required

**Performance Requirements:** Complex orchestral scores with 30,000+ simultaneous objects cannot achieve 60fps scrolling with CPU-only vector rasterization. Modern orchestral notation requires parallel GPU execution for acceptable user experience.

**Vector Rendering Efficiency:** GPUs excel at parallel vector path rendering. Bezier curve rasterization, glyph rendering, and anti-aliasing benefit massively from GPU compute. Software rasterization would require frame budgets exceeding 100ms for complex viewports.

**Technology Flexibility:** Skija is Ooloi's GPU-accelerated vector rendering API. Skija internally uses Metal (macOS), Vulkan (Linux/Windows), or OpenGL (fallback) as GPU backends - these are implementation details managed by Skija. Backend paintlists are rendering-library-agnostic, meaning they describe drawing instructions independently of any specific rendering technology.

**Error Recovery:** Software rasterization fallback exists for GPU driver failures or diagnostic scenarios, but is not intended for production use due to performance constraints.

## Consequences

### Positive

1. **Deterministic Rendering:** All frontends render identically from backend paintlists.

2. **Collaborative Consistency:** No rendering inconsistencies between users viewing same piece.

3. **Closed Semantic Model Preserved:** Frontend cannot reopen semantic decisions through rendering.

4. **Technology Flexibility:** Backend paintlists work with any frontend technology (JavaFX, web, mobile, future).

5. **Testability:** Backend rendering pipeline fully testable without UI. Frontend rendering testable by comparing to reference images.

6. **Simplicity:** Terminal execution eliminates feedback loops and refinement complexity.

7. **Performance via GPU Acceleration:** GPU-accelerated vector rendering enables smooth 60fps scrolling through complex orchestral scores (30,000+ objects). Correctness is defined by paintlist fidelity (litmus test) independent of GPU technology choice; production rendering mode requires GPU acceleration.

### Trade-offs

1. **Network Dependency:** Frontend rendering requires backend paintlists. Offline capability limited to cached paintlists.

2. **Latency Floor:** Visible edits require backend recomputation + network fetch. Cannot achieve sub-backend-batch latency.

3. **Memory for Cache:** Paintlist storage required for smooth scrolling. Larger than storing just musical semantics.

4. **Implementation Complexity:** Two-level cache with strict invalidation requires discipline.

5. **GPU Integration Uncertainty:** JavaFX/Skija embedding strategy provisional. May require iteration to find optimal approach.

### Risks and Mitigations

**Risk:** Skija GPU integration with JavaFX proves problematic or insufficient.
**Mitigation:** Backend paintlists are rendering-library-agnostic. Could switch to alternative GPU-accelerated vector rendering library (different cross-platform library, platform-specific native APIs, web Canvas API with OffscreenCanvas, etc.) without backend changes. Provisional JavaFX/Skija embedding strategy allows iteration.

**Risk:** Network latency makes paintlist fetching too slow for good UX.
**Mitigation:** ADR-0031 priority queues + viewport-aware fetching + aggressive prefetch. Measured latencies guide prefetch strategy.

**Risk:** Paintlist size too large for practical network transfer.
**Mitigation:** VPD-granular fetching means only viewport downloaded. Compression applied to gRPC transport.

**Risk:** Frontend developers violate terminal execution principle by adding refinements.
**Mitigation:** Litmus test in CI. Regular audits. Architectural reviews. Documentation emphasis.

## Related ADRs

- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Establishes separation requiring clear boundaries
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Transport for paintlist fetching and event streaming
- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) - Frontend rendering technologies
- [ADR-0006: SMuFL](0006-SMuFL.md) - Glyph definitions referenced in paintlists
- [ADR-0008: VPDs](0008-VPDs.md) - Hierarchical addressing enabling paintlist granularity
- [ADR-0022: Lazy Frontend-Backend Architecture](0022-Lazy-Frontend-Backend-Architecture.md) - Event-driven synchronization this implements
- [ADR-0024: gRPC Concurrency and Flow Control](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) - Event delivery guarantees
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) - Backend pipeline producing paintlists
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Frontend components (Event Router, RDM, Fetch Coordinator)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) - Closed semantic model this preserves
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - State authority; this document addresses rendering authority
- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) - Format for UI elements (windows, dialogs) that cljfx renders

## Notes

This ADR establishes the rendering boundary as an architectural invariant. Backend authority and terminal frontend execution are not implementation details subject to optimization - they are foundational principles that preserve Ooloi's closed semantic model and enable deterministic, collaborative, multi-platform rendering.

The litmus test (discard frontend state, regenerate from backend → identical output) is the enforcement mechanism. Any proposed optimization that fails this test violates architectural closure and must be rejected.

GPU acceleration via Skija Pictures (GPU command buffers) is **required** for production performance with complex orchestral scores. Paintlists are recorded once into Pictures, then Pictures are replayed every frame. This is essential - you cannot re-parse and re-issue draw commands for 30,000 objects at 60fps.

Skija is Ooloi's committed GPU rendering API. Skija internally uses platform-appropriate GPU backends (Metal on macOS, Vulkan on Linux/Windows, OpenGL as fallback) - this is handled transparently by Skija. Backend paintlists are rendering-library-agnostic and describe vector drawing instructions independently of the rendering technology. Software rasterization exists only for error recovery (GPU driver failure), not production use.

Pictures can be evicted from cache under memory pressure (regeneration is cheap: one-time recording from local paintlist). But the Picture recording mechanism itself is not optional - it's how GPU acceleration works.
