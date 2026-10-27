# ADR-0038: Backend-Authoritative Rendering: Terminal Frontend Execution with GPU Acceleration

## Status

Under implementation

## Table of Contents

- [Context](#context)
  - [The Rendering Boundary Problem](#the-rendering-boundary-problem)
  - [The Terminal Execution Principle](#the-terminal-execution-principle)
  - [Performance Requirements](#performance-requirements)
  - [Governing Requirements: R1–R3](#governing-requirements-r1r3)
  - [Technology Considerations](#technology-considerations)
- [Decision](#decision)
- [Architecture](#architecture)
  - [Component Architecture](#component-architecture)
  - [Hierarchical Paintlist Structure](#hierarchical-paintlist-structure)
  - [Backend Authority: Paintlist Generation](#backend-authority-paintlist-generation)
  - [Frontend Components](#frontend-components)
  - [Frontend Rendering Threading Model](#frontend-rendering-threading-model)
  - [Event Flow Architecture](#event-flow-architecture)
  - [Progressive Refinement](#progressive-refinement)
  - [Three-Level Cache Architecture](#three-level-cache-architecture)
  - [Memory Management](#memory-management)
  - [Garbage Collection and the Frame Budget](#garbage-collection-and-the-frame-budget)
  - [Small Notation Previews](#small-notation-previews)
  - [Performance Characteristics](#performance-characteristics)
  - [Benchmark Harness](#benchmark-harness)
  - [Implementation Phases](#implementation-phases)
- [Rationale](#rationale)
  - [Why Backend Authority is Absolute](#why-backend-authority-is-absolute)
  - [Why Terminal Execution](#why-terminal-execution)
  - [Why Three-Level Cache](#why-three-level-cache)
  - [Why JavaFX-Composited Tiles](#why-javafx-composited-tiles)
  - [Why GPU Acceleration is Required](#why-gpu-acceleration-is-required)
- [Consequences](#consequences)
  - [Positive](#positive)
  - [Trade-offs](#trade-offs)
  - [Risks and Mitigations](#risks-and-mitigations)
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
- A GUI that never freezes or stutters, and score content that scrolls at an even 30 fps (R1 and R2 below)
- Low latency for visible edits (target <150ms p95)
- Memory efficiency: the derived caches (Pictures, tiles) are proportional to the viewport; paintlists grow only with what has been viewed
- Lazy loading (no downloads until layout windows open)

### Governing Requirements: R1–R3

Three requirements drive the frontend rendering design. Every rule in [Frontend Rendering Threading Model](#frontend-rendering-threading-model), [Progressive Refinement](#progressive-refinement) and [Why JavaFX-Composited Tiles](#why-javafx-composited-tiles) follows from one of them.

**R1 — The GUI never freezes or stutters.** No user-visible hitch is acceptable: not while scrolling, editing, receiving a collaborator's changes, switching fonts, or opening a settings window. The consequence is architectural. No rendering work runs on the JavaFX Application Thread (JAT), and no work is serialised onto any single thread whose stall would be visible. The JAT does bounded, non-blocking work only.

**R2 — Score content runs at an even 30 fps.** Score content does not need to exceed 24–30 fps. The design budget is 33 ms per frame; the acceptance criterion is evenly paced 30 fps, never below 24 fps, and nothing in the score rendering path is designed around rates above 30 fps. Even pacing matters more than the number: 30 fps divides 60 Hz and 120 Hz displays evenly, whereas 24 fps falls into a 3:2 cadence on 60 Hz, which is why 30 is the target and 24 only the floor. JavaFX's pulse rate is left at its default — moving tiles that are already rendered costs almost nothing, and lowering the pulse would slow every other animation in the UI. R2 governs score content only; JavaFX animation and overlays drawn as JavaFX nodes, the playback cursor among them, are outside it.

**R3 — JavaFX UI may overlap the score.** Palettes, rulers, in-window notifications, drag feedback and any other JavaFX UI must be able to appear over the score viewport, intersecting the Skija-rendered area freely. An embedding that cannot draw JavaFX content above score content is disqualified.

### Technology Considerations

- GPU acceleration is **required** for acceptable performance with complex orchestral scores
- Skija provides GPU-accelerated vector rendering (Metal/Vulkan/OpenGL backends)
- Software rasterization is used for **error recovery**, small notation previews and golden-image tests, never for the score in production (see [Small Notation Previews](#small-notation-previews))
- cljfx provides windowing but must NOT draw music notation
- The JavaFX/Skija embedding is **JavaFX-composited raster tiles**: Skija rasterises fixed-size tiles on the GPU, and JavaFX composites them as ordinary nodes. A native GPU view placed over the score area is rejected, above all because it violates R3 ([Why JavaFX-Composited Tiles](#why-javafx-composited-tiles))
- Specific GPU technology (Skija vs alternatives) is swappable, but GPU acceleration itself is not optional

## Decision

We establish **backend-authoritative rendering with terminal frontend execution**:

1. **Backend Authority:** All musical layout and engraving decisions computed by backend's 6-stage rendering pipeline (ADR-0028), producing paintlists containing vector drawing instructions (glyphs with positions, bezier curves with control points).

2. **Terminal Frontend:** Frontend executes backend-computed drawing instructions without interpretation, refinement, or renegotiation. Rendering is the **end** of the pipeline - consuming decisions, never feeding back.

3. **Three-Level Cache Architecture:**
   - **Level 1 (Authoritative):** Rendering Data Manager stores backend-provided paintlists
   - **Level 2 (Derived, Evictable):** Skija Pictures recorded from paintlists, at system or measure granularity
   - **Level 3 (Derived, Evictable):** Raster tiles rasterised on the GPU from Pictures, keyed by zoom level and tile coordinates

4. **Litmus Test Enforcement:** Discarding all frontend rendering state — every Picture and every tile included — and regenerating from backend paintlists must produce identical output. If not, architectural closure is broken.

5. **GPU-Accelerated Rendering in JavaFX-Composited Tiles:** GPU acceleration via Skija (or equivalent technology) is **required for production performance**. Complex orchestral scores (30,000+ objects) require GPU vector rendering for smooth scrolling at scale. Skija rasterises the score into fixed-size tiles; JavaFX composites them, so any JavaFX node can sit above the score (R3). Software rasterization fallback exists for error recovery only (GPU driver failure).

6. **Component Boundaries:**
   - cljfx: Windows, menus, input handlers, and the viewport container (does NOT draw music)
   - Skija: GPU-accelerated drawing execution (does NOT compute layout)
   - Event Router: Protocol adapter for backend events (ADR-0031)
   - Rendering Data Manager: VPD-indexed paintlist storage (ADR-0031)
   - Fetch Coordinator: Priority-based paintlist fetching (ADR-0031)
   - Tile scheduler: orders tile work by viewport priority, off the JAT
   - GPU render thread: owns the GPU context; rasterises and reads back tiles

7. **Nothing that renders runs on the JAT (R1).** The JAT applies the scroll offset, installs finished tiles and placeholders, and forwards input. Picture recording, rasterisation and readback run elsewhere ([Frontend Rendering Threading Model](#frontend-rendering-threading-model)).

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
│  │   - page and system dimensions, without contents          │  │
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
│  │  - Classifies and batches events (16-100ms, by stream)     │ │
│  │  - Publishes batches to frontend event bus (eb/publish!)   │ │
│  │  - No direct handlers; downstream via UI Manager           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Rendering Data Manager (ADR-0031) — Level 1               │ │
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
│  │  - Dimensions before contents for regions entering view    │ │
│  │  - Background threads call gRPC API                        │ │
│  │  - Updates RDM (background); tile scheduler learns of it   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Tile Scheduler (off the JAT)                              │ │
│  │  - Visible tiles first, then direction of travel           │ │
│  │  - Cancels work that has scrolled out of relevance         │ │
│  │  - Requests missing regions from the Fetch Coordinator     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                ↓                              ↓                 │
│  ┌──────────────────────────────┐ ┌───────────────────────────┐ │
│  │  Picture Recording           │ │  GPU Render Thread        │ │
│  │  (shared Claypoole pool)     │ │  - Owns the GPU context   │ │
│  │  - Level 2: Pictures from    │→│  - Level 3: rasterises,   │ │
│  │    paintlists, per system    │ │    flushes and reads back │ │
│  │    or measure                │ │    tiles in batches       │ │
│  └──────────────────────────────┘ └───────────────────────────┘ │
│                                               ↓ fx/run-later!   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Tile Layer (JAT)                                          │ │
│  │  - Pooled ImageView nodes over PixelBuffers, one Group     │ │
│  │  - One translate per pulse; installs finished tiles        │ │
│  │  - Placeholders where a tile is not ready                  │ │
│  │  - Composited by JavaFX; any node above it overlaps (R3)   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               ↓                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  cljfx UI Shell (Windowing Only - ADR-0022)                │ │
│  │  - Windows, palettes, menus                                │ │
│  │  - Input handlers (mouse, keyboard)                        │ │
│  │  - Owns the viewport container; never diffs the tiles      │ │
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

The dimensions of pages and systems are also available on their own, without their contents. They are small and fast to fetch, and they let the frontend give a page or system its final size and position before anything inside it has arrived ([Progressive Refinement](#progressive-refinement)).

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

**Note on Paintlist Format Design:** The examples above are conceptual/pseudocode. The real paintlist format will be designed pragmatically: if aligning closely with Skija's API syntax speeds up translation (paintlist → Skija drawing calls), we'll do that. Otherwise, we'll keep the format as abstract as possible. The format must be serializable via gRPC and sufficient for hit-testing, but internal structure is an implementation detail.

**Backend computes:**
- Exact glyph positions (x, y coordinates)
- Glyph selection from SMuFL font (ADR-0006)
- Bounding boxes for hit-testing
- Bezier curve control points for slurs, ties, beams
- All spatial relationships and collision avoidance

**Frontend receives these as data** via gRPC API calls and renders them faithfully.

**Paintlist properties:**
- **VPD identity:** each paintlist belongs to a specific VPD
- **Spatial data:** bounding boxes and paths enable hit-testing, and the spatial culling that decides which Pictures a tile needs
- **Semantic links:** optional pointers to musical elements (when elements remain semantic)
- **Graphics support:** user-edited bezier paths with control points (Illustrator-level editing)
- **Immutable:** paintlists do not change until backend recomputes and frontend refetches

### Frontend Components

The following components implement the terminal frontend execution model. Events flow top-to-bottom; rendering data is fetched on demand:

```
┌──────────────────────────────────────┐
│          Backend Server              │
│  piece changes · presence · playback │
└───────────────┬──────────────────────┘
                │ gRPC event stream
                ▼
┌─────────────────────────────────────┐
│          Event Client               │
│  protobuf → Clojure map             │
│  process-received-event             │
└───────────────┬─────────────────────┘
                │ individual events
                ▼
┌─────────────────────────────────────┐
│    Event Router — Layer 3           │
│  categorise → batch → publish       │
│  time windows: 16ms – 100ms         │
└───────────────┬─────────────────────┘
                │ eb/publish!
                │ (category, [events])
                ▼
┌─────────────────────────────────────┐    Frontend publishers
│    Frontend Event Bus — Layer 2     │◄── start-app!
│  category-based pub/sub             │◄── show-window!
│  Claypoole thread pool delivery     │◄── set-app-setting!
└───────────────┬─────────────────────┘
                │ Claypoole futures
                ▼
┌─────────────────────────────────────┐
│    UI Manager (central mediator)    │
│  subscribes to the categories it    │
│  serves; dispatches reactions       │
└───────┬─────────────────┬───────────┘
        │                 │
        │ fx/run-later!   │ background work
        ▼                 ▼
┌────────────────┐  ┌────────────────┐
│  JAT — Layer 1 │  │ Fetch Coord.,  │
│  UI updates,   │  │ atom updates,  │
│  theme, notifs │  │ stale marking  │
└────────────────┘  └────────────────┘
```

#### 1. Event Router (Integrant Component)

Pure protocol adapter: it classifies each arriving backend event, coalesces it in the appropriate queue at that stream's cadence, and hands each batch to its consumer (ADR-0031 §Per-Piece Event Routing, which is where routing is specified). The two window-refetch events — `:piece-structure-changed` and `:piece-dirty-changed` — are the exception: they are dispatched straight to the subscribing piece's window, neither classified nor batched. JavaFX local events remain in JavaFX.

**Responsibilities:**
- Subscribe to gRPC event streams
- Derive routing category from event type
- Batch events by category with time windows
- Publish one `eb/publish!` per category batch to the frontend event bus
- No direct category handlers — category processing happens through bus subscribers mediated by the UI Manager
- The window-refetch events (`:piece-structure-changed`, `:piece-dirty-changed`) are dispatched directly to the subscribing window's registered handler, bypassing classification and batching (ADR-0031)
- No rendering details; no UI chrome

**Invalidation Flow:**

`:piece-invalidation` is a piece event: delivered only to that piece's subscribers, and coalesced
per piece at the invalidation cadence (~50–100 ms) rather than mingled with other pieces' events.
Routing is specified in ADR-0031 §Per-Piece Event Routing and is not restated here; what matters at
this boundary is the reaction, which the UI Manager performs on a pool thread:

```clojure
;; runs on a Claypoole future thread — never the JAT
(rdm/mark-stale! rendering-data-manager vpd)              ; Mark stale (background-safe)
(fc/queue-fetch! fetch-coordinator vpd piece-id :critical) ; Queue refetch
```

The two calls belong together: marking stale without queueing a fetch would leave the viewport
showing data known to be wrong with nothing on its way to replace it.

**Neither this stream nor the Fetch Coordinator's use of it is built yet** — `:piece-invalidation`
has no emitter and no consumer today. The design is specified here and in ADR-0028 so that the
rendering pipeline has somewhere to deliver to when it lands.

#### 2. Rendering Data Manager

VPD-indexed storage of backend-provided paintlists with staleness tracking. This is the authoritative frontend cache layer (Level 1).

**Responsibilities:**
- Store backend-provided paintlists by VPD
- Track staleness per VPD
- Provide O(1) lookup for rendering
- Coordinate with Fetch Coordinator for refetch requests
- Pure data cache (atoms) — updates happen on background threads; nothing about it requires the JAT

**Atomic Operations:**

RDM operations are atomic. `update-paintlist!` updates the paintlist AND clears the stale flag in a single operation. No intermediate state exists where paintlist is fresh but stale flag remains set, or vice versa.

**Not responsible for:**
- Any layout computation
- Any engraving decision logic
- Any generation of drawing instructions
- Pictures or tiles, which belong to the rendering layer

#### 3. Fetch Coordinator

Priority-based fetching of paintlists from backend via normal gRPC API calls. All network work happens off the JavaFX Application Thread, and so do the RDM updates its fetches produce.

**Responsibilities:**
- Maintain priority queues (Critical/High/Normal/Low)
- Perform background gRPC fetches
- Fetch coarse to fine: for a region entering view, the dimensions of its pages and systems before their contents
- Update Rendering Data Manager on background thread
- Make each new paintlist known to the tile scheduler, off the JAT
- Implement retry/backoff/error reporting policy

**Priority Level Usage:**

- **Critical:** Cache invalidations from backend events (visible viewport updates)
- **High:** Viewport scroll demand loading (newly visible content)
- **Normal:** Adjacent viewport prefetch (buffer zones)
- **Low:** Speculative prefetch (anticipatory loading)

Invalidation events queue at Critical priority to ensure visible viewport updates process immediately before speculative prefetch.

**Coarse to fine.** Page and system dimensions are small, so a fast scroll obtains the shape of the pages it passes almost at once, and contents follow by viewport priority. Dimensions are cached locally like paintlists. How the dimension requests are batched and prioritised against content fetches is settled when this is built.

**Not responsible for:**
- Determining invalidation semantics (UI Manager + RDM)
- Rendering the paintlists (Skija layer)
- Any engraving/layout logic (backend)

#### 4. Skija GPU Rendering Layer

Skija draws the music. cljfx does not.

This layer turns Level 1 paintlists into what the JAT shows, through two derived levels. **Pictures (Level 2)** are Skija `Picture` objects recorded from paintlists. A Picture is a backend-independent recording of canvas calls: replaying it re-issues the drawing work, on whatever canvas it is replayed into. Replaying a Picture is far cheaper than re-walking a paintlist and reissuing its draw calls, but a Picture is not pixels. **Tiles (Level 3)** are the pixels: fixed-size squares of the page, rasterised on the GPU by replaying the Pictures that intersect them. Correctness comes from the litmus test: discarding all Pictures and all tiles and regenerating them from paintlists must yield identical output.

**Pictures at system or measure granularity.** Skija's `PictureRecorder` exposes only `beginRecording(Rect)` and offers no option to build a bounding-box hierarchy into a Picture, so replaying a Picture into a small clip still walks every operation it recorded. A Picture per page would make every tile pay for the whole page. Pictures are therefore recorded per system or per measure, and Ooloi culls spatially itself: for each tile, it selects the Pictures whose bounds intersect it, from the layout geometry the paintlists already carry.

**Tiles.** A tile is a fixed-size square of device pixels (256 or 512 on a side; the benchmark harness decides), keyed by zoom level and tile coordinates. Each is rasterised on the GPU (Metal on macOS, Vulkan on Windows and Linux, OpenGL as fallback), read back once into a tile buffer, and shown by JavaFX as its own `ImageView` over a `PixelBuffer`. A low-resolution level (for example a quarter of full resolution) is kept for content already loaded, and shown scaled up during flings and zoom changes until sharp tiles replace it.

**Readback is batched.** Skija's GPU readback (`readPixels`) is synchronous; there is no asynchronous readback API. The render thread therefore rasterises every tile due in a batch, flushes once and reads back once: one GPU synchronisation per batch, not per tile, and never on the JAT.

**Key properties:**
- Derived strictly from paintlists: a Picture only from a committed paintlist, a tile only from Pictures
- Pictures and tiles built from a paintlist are dropped when that paintlist is replaced
- Each GPU context is long-lived and owned by one render thread; never created per render call
- Picture and tile cache entries are evictable; regeneration costs a recording and a rasterisation from local paintlists
- Pictures are immutable and may be replayed from any thread

#### 5. cljfx UI Shell

cljfx is used for windowing, menus, palettes, and input handling. It owns the score viewport's container node, and nothing inside it.

**The score viewport is not a ScrollPane.** A `ScrollPane` routes every scroll change through layout on the JAT and implies a content node sized to the whole score, hundreds of pages long. The viewport is instead a fixed-size region holding the tile `Group`, with scrollbars bound to a logical scroll position the viewport holds. The scrollbars and the `AnimationTimer` that applies the scroll drive the same logical position.

**The tile Group is isolated from cljfx.** It has no CSS classes and no managed layout, and the cljfx renderer never diffs it. Tile nodes are recycled from a fixed pool rather than created and destroyed while scrolling.

**cljfx responsibilities:**
- Window management and UI chrome
- Input events (mouse/keyboard) and tool state
- Settings, palettes, menus
- The viewport container, and every JavaFX node placed above the tiles (R3)

**cljfx does not:**
- Compute layout
- Generate paintlists
- Draw music (Skija does)
- Touch the tiles

### Frontend Rendering Threading Model

The threading model is governed by R1: no rendering work on the JAT, and no visible stall from any single thread.

**JavaFX Application Thread.** Per pulse, the JAT only:
- applies the current scroll offset as one translate on the tile `Group`;
- installs tiles that have finished, by swapping a tile node's image or calling `updateBuffer` on a buffer that is no longer being written;
- shows a low-resolution tile or a placeholder where a tile is not ready;
- handles input and forwards intent (scroll, zoom, selection) to the tile scheduler.

The JAT never records Pictures, never rasterises, never reads back, never touches a GPU context, never waits on another thread, network call or lock, and never performs work proportional to score size. Its work per pulse is bounded by the number of visible tiles.

**Picture recording** runs in parallel on the shared Claypoole pool ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md) §Shared Threadpool), prioritised by task ordering like every other workload on it.

**GPU render thread.** A Skia GPU context may be used from only one thread at a time, so each context is owned by one dedicated render thread, never the JAT. This is the one exception to ADR-0028's single shared pool, and the reason is affinity rather than load: the context cannot be handed between pool threads. The render thread rasterises tiles in batches, flushes and reads back once per batch, and hands completed tiles to the JAT with a single `fx/run-later!`. Readback writes into a free buffer from a pool, never into one JavaFX may be reading. One context per window is the starting point; whether more than one render thread helps is a harness question. A stalled render thread is not a visible stall in R1's sense: the tiles and placeholders already installed stay on screen, and the JAT keeps scrolling them.

**Tile scheduler.** Runs off the JAT. It orders work by priority — visible tiles, then tiles in the direction of travel, then others — cancels queued work that has scrolled out of relevance, and asks the Fetch Coordinator for regions that have no paintlists yet. How it learns that a paintlist has been replaced is decided when it is built; the candidates are a direct call from the Fetch Coordinator's completion, and a check against the RDM each time the scheduler plans work.

**Fetch Coordinator and RDM.** Unchanged: network calls and atom updates run off the JAT.

**Consistency invariants.** These apply to every code path that updates a paintlist or shows the score:

1. **Level 1 before Level 2 before Level 3.** A Picture is recorded only from a committed RDM paintlist; a tile is rasterised only from Pictures.
2. **Publication by identity.** Each tile records the paintlists it was built from. On publication, the JAT installs a tile only if every one of them is still the paintlist the RDM holds at its VPD, compared with `identical?`. A tile built from a paintlist since replaced is discarded, and the scheduler queues its rebuild. Paintlists are immutable values, so the value itself is the version: no counter exists, and none is needed. A paintlist merely marked stale has not been replaced, so a tile built from it is no staler than what is already on screen, and is installed.
3. **RDM atoms stay off the JAT.** `rdm/update-paintlist!` is never called inside `fx/run-later!`. Reading the RDM, as publication does, is safe on any thread.
4. **One-version staleness.** A tile already published may stay visible until its replacement is published. It is derived from a previous valid paintlist, never corrupt.

Consistency therefore comes from comparing what a tile was built from with what is current, not from serialising work onto the JAT. Implementation example:

```clojure
;; Fetch Coordinator completion (pool thread, after the gRPC fetch returns)
(rdm/update-paintlist! rendering-data-manager vpd paintlist)  ; Level 1: atom swap, off the JAT
;; The tile scheduler learns of the new paintlist off the JAT, drops the Pictures
;; and tiles built from the old one, and queues their rebuild.

;; Publication (JAT): install a finished tile only if everything it was drawn
;; from is still current. `:sources` (VPD → paintlist) is illustrative.
(defn publishable? [rendering-data-manager tile]
  (every? (fn [[vpd paintlist]]
            (identical? paintlist (rdm/get-paintlist rendering-data-manager vpd)))
          (:sources tile)))
```

### Event Flow Architecture

#### Typical Edit Scenario (Visible Change)

1. User gesture handled by JavaFX (local event system).
2. Frontend sends an API command to backend (gRPC call). This is the only way rendering can change.
3. Backend updates piece in STM, recomputes affected paintlists via ADR-0028 pipeline.
4. Backend emits `:piece-invalidation` events with VPD scope (ADR-0024 FIFO per client).
5. Event Router coalesces the piece's `:piece-invalidation` events in that piece's invalidation queue (50–100 ms) and hands the batch to its consumer.
6. UI Manager's bus subscriber marks affected VPDs stale in RDM and queues refetches.
7. Fetch Coordinator performs background gRPC fetches for fresh paintlists.
8. Fetch Coordinator updates RDM paintlists (background); the tile scheduler learns of them, drops the Pictures and tiles built from the old paintlists, and queues the affected tiles.
9. Pictures are re-recorded on the pool; the render thread rasterises and reads back the affected tiles and hands them to the JAT.
10. On its next pulse the JAT publishes each tile whose paintlists are still current.

**Latency target:** <150ms p95 for visible edits (to be validated by measurement).

#### Scroll Scenario (Demand Loading)

1. User scroll handled by JavaFX; the JAT moves the tile `Group` and shows, in the same pulse, the best representation available for every region now in view — a tile, a low-resolution tile, or a placeholder ([Progressive Refinement](#progressive-refinement)).
2. The tile scheduler determines newly visible regions and the direction of travel.
3. For regions without data, the Fetch Coordinator fetches page and system dimensions first, then contents (HIGH priority for visible missing).
4. When paintlists arrive, RDM updates on a background thread; Pictures are recorded and tiles produced off the JAT, and each refinement replaces its placeholder in place.

No backend events are required for demand loading. Backend events only indicate staleness due to changes.

#### Reconnection Scenario

On reconnect:
- Resubscribe to event streams
- Invalidate cached rendering state (mark all stale or reset staleness)
- Fetch viewport paintlists at CRITICAL priority
- No event replay is required; frontend simply fetches current state

### Progressive Refinement

When scrolling outpaces the arrival of paintlists or the production of tiles, the viewport never stalls and never shows empty gaps. Every region shows the best representation available for it and is refined in place, following the paintlist hierarchy:

1. **Page known, nothing else.** The page is shown blank with fuzzy placeholder content, a blurred generic texture suggesting staves and notation that reads unmistakably as *loading*, never as music.
2. **System dimensions known.** The placeholder is drawn within each system's real rectangle, so the page already has its final shape.
3. **Contents available.** The page, system, staff and measure paintlists are rasterised, at lower resolution first where that is quicker, and replaced by full-resolution tiles once they are ready.

Page and system dimensions come from the backend calls that return them without their contents; when they are already cached locally the frontend uses them at once, and otherwise a placeholder of the page's size stands in until they arrive. The finer detail — which levels refine in which order, and at what resolutions — is settled during implementation.

**Placeholders never depend on what is lagging.** They exist precisely when the network or the render thread is behind, so they wait on neither. A small set of placeholder textures is prepared once at startup and shown by the JAT directly, scaled to each page or system rectangle; their cost per pulse is bounded like any other tile work.

**No layout shift.** Because system placeholders use real dimensions, real content lands where its placeholder was. Each refinement is a swap in place; nothing on the page moves.

**Placeholders are inert.** Hit-testing ignores placeholders and any region whose paintlist is not present: clicks there select nothing and edit nothing. Interaction becomes available as soon as a region's Level 1 paintlist is present, whatever the current resolution of its tiles. This is the frontend side of ADR-0022's rule that hit-testing requires current visual data.

**Low-resolution tiles for zoom and flings.** Separately, the low-resolution tile level keeps already-loaded content visible, scaled up, during flings and zoom changes, until sharp tiles replace it once motion slows.

### Three-Level Cache Architecture

| Level | Content | Authority | Eviction |
|---|---|---|---|
| **L1** | Paintlists, keyed by VPD (Rendering Data Manager) | Authoritative | Never for memory pressure; only by backend invalidation, reconnect policy, or the user closing the piece or layout that holds them |
| **L2** | Skija Pictures recorded from paintlists, per system or measure | Derived from L1 | Evictable; re-recorded from L1 |
| **L3** | Raster tiles keyed by zoom level and tile coordinates, plus a low-resolution level | Derived from L2 | Evictable; re-rasterised from L2 |

#### Level 1: Rendering Data Manager (Authoritative)

- Stores backend-provided paintlists, keyed by VPD.
- Tracks staleness/freshness.
- Updated only by gRPC fetch results.
- Invalidated only by backend invalidation events (or explicit reconnect policy).

#### Level 2: Skija Pictures (Derived, Evictable)

- Recordings of canvas calls, made from paintlists; replaying one re-issues the drawing work.
- Recorded per system or per measure, so that a tile replays only what intersects it.
- Individual entries may be evicted under memory pressure; correctness unaffected (regeneration: one recording from the local paintlist).
- Dropped when the Level 1 paintlist they were recorded from is replaced.

#### Level 3: Raster Tiles (Derived, Evictable)

- The cached pixels: tiles rasterised on the GPU from the Pictures that intersect them, held as JavaFX textures once installed.
- Keyed by zoom level and tile coordinates; a low-resolution level is kept for flings and zoom changes.
- Dropped when any paintlist they were built from is replaced; evictable outside the viewport.
- Litmus test: discard all Pictures and all tiles, regenerate from paintlists → identical output.

**Cache Invalidation Flow:**

```
:piece-invalidation (that piece's subscribers only)
    ↓
Event Router: coalesce in this piece's invalidation queue (~50-100ms)
    ↓
flush batch → UI Manager
    ↓
UI Manager handler (Claypoole thread):
    RDM: mark VPD stale (atom — background-safe)
    FC:  queue gRPC fetch
    ↓
Background Thread: api/fetch-measure-view(vpd)
    ↓
Fetch Coordinator completion (background thread):
    RDM: update paintlist (Level 1, atom — background-safe)
    ↓
Tile scheduler (off the JAT):
    drop Pictures and tiles built from the old paintlist (Levels 2 and 3)
    queue the affected tiles
    ↓
Claypoole pool: re-record Pictures (Level 2)
    ↓
GPU render thread: rasterise, flush, read back (Level 3)
    ↓
fx/run-later!
    ↓
JAT, next pulse: publish each tile whose paintlists are still current
```

The invariants this flow obeys are stated once, in [Frontend Rendering Threading Model](#frontend-rendering-threading-model).

### Memory Management

**Paintlists (Level 1):**
- Stored locally; over time may accumulate to full-piece coverage.
- Between edits (local or remote), cache is stable: no time-based expiry, no spontaneous invalidations.
- Loading is lazy: opening a piece downloads no graphical data until layout windows open.
- Released when the user closes the piece or layout that holds them.

**Pictures (Level 2):**
- Bounded and evictable; viewport-based or LRU eviction is acceptable under memory pressure.
- Regeneration cost is one recording from local paintlists.

**Tiles (Level 3):**
- Bounded by the viewport and its prefetch margin, plus the low-resolution level.
- In the tiled embedding each tile is held roughly three times — the Skija GPU texture, the system-memory tile buffer, and the JavaFX texture. A 512×512 tile is 1 MiB, so a tile cache covering three viewports is estimated at about 210 MiB; this is a hypothesis for the benchmark harness.

**Placeholder textures:** a small fixed set, prepared once at startup and reused.

### Garbage Collection and the Frame Budget

A collection pause stops every thread in the JVM, the JAT with it, so a pause longer than a frame is a visible hitch however well the rendering threads are arranged (R1). Collector pauses therefore count against the 33 ms frame budget, and the choice of collector is part of the rendering design rather than a deployment detail.

The collector in use is G1, with compact object headers, string deduplication, and a pause-time goal of 50 ms (`-XX:MaxGCPauseMillis=50`). That goal is longer than a frame. G1's pause target is a goal rather than a bound — G1 sizes its collection work to aim at it — and a 50 ms goal admits pauses R1 does not.

There are two ways to meet R1, and the benchmark harness decides between them:

- **G1 with a pause goal inside the frame budget.** G1 keeps its throughput, which matters in the combined application, where the backend's layout pipeline runs in the same JVM. Its pauses stay stop-the-world and grow with the live data in the regions collected, so the harness has to show them staying within budget on dense orchestral scores, with enough margin left for the frame's own work.
- **ZGC.** ZGC marks and relocates concurrently with the application, and is designed for pauses that do not exceed a millisecond whatever the heap size. Since JDK 24 it is always generational. It pays for this in throughput and in memory headroom, since collection proceeds while the application keeps allocating. `MaxGCPauseMillis` means nothing to it.

**Two memory measures must survive the choice: string deduplication and compact object headers.** String deduplication was long available only with G1 and Shenandoah; JDK 18 added it to ZGC, Serial and Parallel. Compact object headers, a product feature since JDK 25, are compatible with ZGC. On JDK 25, which Ooloi builds with, both collectors therefore keep both measures, and any later change of collector or JDK is checked against these two before it is adopted.

The collector is a JVM-wide choice. In the combined application it governs the backend as well, so the trade is judged across both workloads; a standalone backend runs in a JVM of its own and is not bound by the frame budget. The configuration belongs to the application, so it holds for the built application as well as for development JVMs.

### Small Notation Previews

Settings windows (engraving parameters such as accidental distances), font selection, and similar small notation examples have no efficiency requirement and use neither Picture caches nor tile caches. They use:

- the same paintlist-to-Skija drawing code as the score, so a preview is drawn exactly as the score would be;
- a Skija CPU raster surface, snapshotted into a JavaFX `Image`;
- rendering on a worker thread, never the JAT (R1), with the finished image delivered by `fx/run-later!` and re-rendered on each parameter change.

CPU raster is deliberate here: dialogs then depend on no GPU context, so they work even if GPU initialisation has failed.

CPU raster is never used for the score in production. It remains the error-recovery fallback, and it is the backend for **golden-image tests**: GPU output varies subtly across drivers and GPU generations, whereas raster output is deterministic and runs on CI machines with no GPU. Golden-image tests are the natural enforcement of the litmus test.

### Performance Characteristics

All numeric targets in this document are hypotheses and must be validated by profiling on real hardware:

- Score scrolling: evenly paced 30 fps, never below 24 fps (R2); the JAT never stalls (R1)
- Visible collaborative edit target: <150ms p95
- Scroll-triggered fetch target: <100ms p95 (where network allows)
- Reconnection recovery target: <500ms for viewport refetch

Performance depends on:
- GPU backend (Metal/Vulkan/GL)
- Score complexity and density
- Viewport size and zoom
- Network latency and backend load

**Transfer is not the bottleneck** (estimates, to be validated). A 512×512 tile is 1 MiB. Fast scrolling, about 5 pages per second on a Retina laptop, needs roughly 140 new tiles per second: about 140 MiB/s of readback and the same of upload. A fling, about 20 pages per second, needs roughly 560 tiles and 560 MiB/s each way. Even the fling is around 1% of a base M1's memory bandwidth (about 68 GB/s). The bottlenecks are GPU rasterisation, Picture recording and the availability of paintlists, and those are the same under any embedding.

### Benchmark Harness

The harness is a precondition of Phase 2. It scrolls a synthetic worst case — dense orchestral pages, about 30,000 objects in view, at realistic zoom levels — at constant speeds and in flings, and measures:

- frame pacing at the JAT: the 99th-percentile interval between presented frames, and dropped or doubled frames, against R2;
- JAT time per pulse, which must stay bounded and small;
- garbage-collection pauses, and whether they coincide with dropped frames;
- the time until every visible region shows at least a placeholder (it must be the same pulse in which the region scrolls into view), until system dimensions appear, and until real content appears;
- the time until the viewport is sharp after a fling stops;
- tile production throughput (tiles per second) and Picture recording throughput;
- peak memory, on the GPU and in system memory.

Measurements are taken on the slowest supported hardware: for macOS a base M1 (ADR-0050); the reference hardware for Windows and Linux is named when the harness is built.

### Implementation Phases

#### Phase 1: Infrastructure Foundation (No Music Rendering Yet)

1. Event Router with category aggregation (full implementation)
2. Rendering Data Manager (data structure + operations, mock paintlist data for testing)
3. Fetch Coordinator (threading model + priority queues, mock backend calls for testing)
4. cljfx UI shell (windows, menus, palettes - **no music notation rendering**)
5. Threading model validation (Event Router → bus → UI Manager → RDM/FC coordination)

**Note:** This phase builds event/cache infrastructure and UI chrome without rendering music. RDM and FC use **mock paintlist data** to validate threading model and async coordination. Real paintlist API integration and GPU-accelerated rendering are Phase 2. This separation validates the event-driven architecture before adding GPU complexity.

**Mock vs Real:**
- Phase 1: Simple test paintlists, mock backend fetches, validates threading coordination
- Phase 2: Real paintlist format (SMuFL glyphs, bezier curves), real gRPC API calls, Skija GPU rendering

#### Phase 2: GPU-Accelerated Music Rendering (Skija Integration)

**Precondition:** the [Benchmark Harness](#benchmark-harness), with its acceptance criteria.

1. GPU render thread owning a long-lived GPU context (Metal/Vulkan/OpenGL)
2. Paintlist → Skija Picture recording on the shared pool, per system or measure
3. Tile production: spatial culling, batched rasterisation and readback, publication by identity
4. Tiled viewport: the isolated tile `Group`, logical scroll position, pooled tile nodes
5. Progressive refinement: placeholders, page and system dimensions, low-resolution tiles
6. Correctness validation against reference renderings (golden images, CPU raster)
7. Harness acceptance against R1 and R2 with complex scores

**Note:** GPU acceleration is **vital** in this phase. Pictures are recorded from paintlists once and replayed only when a tile is rasterised; tiles already on screen are composited by JavaFX without any replay. Re-walking paintlists and reissuing their draw calls for every tile would not keep up with 30,000+ objects.

#### Phase 3: Production Optimization

1. Picture and tile cache eviction policy (LRU based on memory pressure)
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

### Why Three-Level Cache

**Authoritative Paintlists (Level 1):** Stores backend truth. Can be discarded and refetched without correctness issues. This is the data cache containing vector drawing instructions.

**Pictures (Level 2):** Skija Pictures record paintlists as canvas calls for cheap replay. Re-walking 30,000 objects' paintlists and issuing their draw calls for every tile would not keep up at any smooth frame rate; replaying a recording does. But a Picture is a recording, not pixels, so it cannot itself be the thing shown.

**Tiles (Level 3):** The pixels. A tile already on screen costs JavaFX nothing but compositing, so scrolling moves cached textures and does no drawing. Only tiles newly exposed or invalidated are rasterised.

**Separation of Concerns:** Data cache (paintlists) separate from the two execution caches. Paintlists authoritative and immutable. Pictures regenerable from paintlists, tiles from Pictures. Different eviction policies: paintlists stay until invalidated or until their piece or layout closes; Pictures and tiles can be evicted aggressively, since regeneration costs a recording and a rasterisation from local data.

### Why JavaFX-Composited Tiles

**Tiles satisfy R3 by construction.** Tiles are ordinary JavaFX nodes in a `Group`, so any node placed above that `Group` is composited over the score. Palettes, rulers, notifications and drag feedback need nothing special.

**Cost scales with what is newly exposed.** JavaFX composites cached tile textures on its own GPU path; scrolling is one translate per pulse, and no pixels move for tiles already on screen. Full-viewport readback every frame is rejected because its cost scales with the viewport's area on every frame rather than with the area newly exposed. The embedding does not depend on which pipeline JavaFX itself renders with: on macOS its Metal pipeline renders the scene into a backing texture and presents it with vsync, and compositing tens of cached tile textures is trivial work for it.

**The native overlay is rejected.** A native GPU view — a `CAMetalLayer`-backed `NSView` on macOS, and equivalents elsewhere — placed over the score area inside the JavaFX window is not adopted:

- **It violates R3.** A native view sits above everything JavaFX draws in its area, so in-window palettes, rulers, notifications and drag feedback could not overlap the score; only separate windows, such as menus and tooltips, would still appear over it. This alone disqualifies it as conceived; what follows are further costs.
- It changes only the transfer and compositing stages, which are not the bottleneck ([Performance Characteristics](#performance-characteristics)).
- Its frame-rate advantage — display-link rates above JavaFX's pulse — is irrelevant under R2.
- Its memory advantage, each tile held once rather than about three times, is modest: an estimated 70 MiB against 210 MiB for a three-viewport tile cache.
- It needs at least four implementations: Cocoa/Metal, Win32 with Vulkan or Direct3D, Linux X11, and Linux Wayland.
- Mouse input, focus, IME and accessibility must be routed between the native view and JavaFX, and the native view's geometry must track JavaFX layout exactly, live resizing included.
- On macOS it inserts a view into the `NSView` hierarchy JavaFX's Glass layer owns, which does not expect foreign subviews; a JavaFX update could break it.
- The tiled path must exist regardless, as the fallback wherever native embedding is unavailable, so a native overlay could only ever be additive.

**Reconsideration trigger.** A native overlay is reconsidered only if the benchmark harness shows stutter — pacing below R2 — traceable to JAT work that the isolation measures of the threading model cannot remove. Even then it is admissible only in a design that still satisfies R3, for example with every UI element that overlaps the score hosted in its own transparent window positioned over it, which is a substantial change to the UI architecture and is costed as part of the reconsideration. If triggered and R3 can be met, it is built for macOS first, where only Apple Silicon is supported (ADR-0050) and JFA interop is already present (ADR-0005), and extended to other platforms only on the same measured evidence.

**Sharing GPU textures with JavaFX through Prism internals is rejected outright.** It breaks on JavaFX updates, and the macOS pipeline has only recently moved from OpenGL to Metal.

### Why GPU Acceleration is Required

**Performance Requirements:** Complex orchestral scores with 30,000+ simultaneous objects cannot achieve smooth scrolling with CPU-only vector rasterization. Modern orchestral notation requires parallel GPU execution for acceptable user experience.

**Vector Rendering Efficiency:** GPUs excel at parallel vector path rendering. Bezier curve rasterization, glyph rendering, and anti-aliasing benefit massively from GPU compute. Software rasterization would require frame budgets exceeding 100ms for complex viewports.

**Technology Flexibility:** Skija is Ooloi's GPU-accelerated vector rendering API. Skija draws through a GPU context that the frontend creates for the platform's graphics API — Metal on macOS, Vulkan on Linux and Windows, OpenGL as the fallback — from that API's native objects (for Metal, the device and its command queue). Backend paintlists are rendering-library-agnostic, meaning they describe drawing instructions independently of any specific rendering technology.

**Error Recovery:** Software rasterization fallback exists for GPU driver failures or diagnostic scenarios, but is not intended for production use due to performance constraints.

**Where CPU raster is right:** small notation previews in dialogs, which need no GPU context, and golden-image tests, which need deterministic output on machines without a GPU ([Small Notation Previews](#small-notation-previews)).

## Consequences

### Positive

1. **Deterministic Rendering:** All frontends render identically from backend paintlists.

2. **Collaborative Consistency:** No rendering inconsistencies between users viewing same piece.

3. **Closed Semantic Model Preserved:** Frontend cannot reopen semantic decisions through rendering.

4. **Technology Flexibility:** Backend paintlists work with any frontend technology (JavaFX, web, mobile, future).

5. **Testability:** Backend rendering pipeline fully testable without UI. Frontend rendering testable by comparing to reference images.

6. **Simplicity:** Terminal execution eliminates feedback loops and refinement complexity.

7. **Performance via GPU Acceleration:** GPU-accelerated vector rendering enables smooth scrolling through complex orchestral scores (30,000+ objects). Correctness is defined by paintlist fidelity (litmus test) independent of GPU technology choice; production rendering mode requires GPU acceleration.

8. **A JAT that never stalls:** the JAT's work per pulse is bounded by the number of visible tiles, and everything that renders runs elsewhere.

### Trade-offs

1. **Network Dependency:** Frontend rendering requires backend paintlists, and there is **no offline mode** — a client with no backend has no piece ([ADR-0040](0040-Single-Authority-State-Model.md)), so cached paintlists are not a degraded working mode but merely what has not yet been discarded. Nothing can be edited without the backend, and nothing diverges locally to be reconciled later.

2. **Latency Floor:** Visible edits require backend recomputation + network fetch. Cannot achieve sub-backend-batch latency.

3. **Memory for Cache:** Paintlist storage required for smooth scrolling. Larger than storing just musical semantics. In the tiled embedding each tile is also held about three times over.

4. **Implementation Complexity:** Three cache levels with strict derivation, a dedicated render thread and publication by identity require discipline.

5. **No native overlay:** the native GPU view is rejected because it violates R3, at the cost of its modest memory advantage; it returns only on the measured trigger in [Why JavaFX-Composited Tiles](#why-javafx-composited-tiles).

### Risks and Mitigations

**Risk:** The tiled embedding shows stutter that the threading model cannot remove.
**Mitigation:** The benchmark harness measures pacing before Phase 2 is accepted. A native overlay is reconsidered only on that evidence, and only in a design that still satisfies R3. Backend paintlists are rendering-library-agnostic, so an alternative GPU-accelerated vector library would need no backend changes.

**Risk:** Network latency makes paintlist fetching too slow for good UX.
**Mitigation:** ADR-0031 priority queues + viewport-aware fetching + aggressive prefetch. Page and system dimensions are fetched first, and placeholders never wait for the network. Measured latencies guide prefetch strategy.

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
- [ADR-0022: Lazy Frontend-Backend Architecture](0022-Lazy-Frontend-Backend-Architecture.md) - Event-driven synchronization this implements; hit-testing requires current visual data, which is why placeholders are inert
- [ADR-0024: gRPC Concurrency and Flow Control](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) - Event delivery guarantees
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) - Backend pipeline producing paintlists; the shared thread pool Picture recording runs on
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Frontend components (Event Router, RDM, Fetch Coordinator)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) - Closed semantic model this preserves
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - State authority; this document addresses rendering authority
- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) - Format for UI elements (windows) that cljfx renders; the JavaFX UI that overlaps the score (R3); the scope of theme colours, which stops at Skija surfaces
- [ADR-0046: Reference-Passing In-Process Transport](0046-Reference-Passing-In-Process-Transport.md) - Eliminates conversion overhead for in-process paintlist transfer, enabling the <150ms p95 latency target at orchestral scale
- [ADR-0047: Font Management](0047-Font-Management.md) - Each glyph comes from the piece's default music font or a font a setting names; a change of either relays out what it affects, and its Pictures and tiles go by the ordinary route; typefaces are released only after the caches built from them
- [ADR-0050: Platform Support Policy](0050-Platform-Support-Policy.md) - Supported platforms and their GPU paths; the slowest supported Mac is the harness's reference
- [ADR-0054: Automatic Semantic Naming and Numbering of Musicians and Instruments](0054-Automatic-Semantic-Naming-and-Numbering-of-Musicians-and-Instruments.md) - The engraved score and extracted parts compose their numbered labels from the stored numbers through the same shared composer the Piece Window uses, so page and tree cannot disagree

## Notes

This ADR establishes the rendering boundary as an architectural invariant. Backend authority and terminal frontend execution are not implementation details subject to optimization - they are foundational principles that preserve Ooloi's closed semantic model and enable deterministic, collaborative, multi-platform rendering.

The litmus test (discard frontend state, regenerate from backend → identical output) is the enforcement mechanism. Any proposed optimization that fails this test violates architectural closure and must be rejected.

GPU acceleration via Skija is **required** for production performance with complex orchestral scores. Paintlists are recorded once into Pictures — recordings of canvas calls, not pixels — and Pictures are replayed only when a tile is rasterised. The pixels on screen are tiles, which JavaFX composites without redrawing. This is essential - you cannot re-parse and re-issue draw commands for 30,000 objects at any acceptable frame rate.

Skija is Ooloi's committed GPU rendering API. It draws through a GPU context the frontend creates for the platform's graphics API (Metal on macOS, Vulkan on Linux and Windows, OpenGL as fallback); choosing and creating that context is frontend work, not something Skija does on its own. Backend paintlists are rendering-library-agnostic and describe vector drawing instructions independently of the rendering technology. Software rasterization serves error recovery, small previews and golden-image tests, never the score in production.

Pictures and tiles can be evicted from cache under memory pressure (regeneration is cheap: a recording and a rasterisation from local paintlists). But the Picture recording mechanism itself is not optional - it's how GPU acceleration works.
