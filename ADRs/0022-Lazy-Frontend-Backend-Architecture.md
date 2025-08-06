# ADR-0022: Lazy Frontend-Backend Architecture

## Status

Accepted 2025-08-06

## Table of Contents

- [Context](#context)
  - [Architectural Vision and Historical Foundation](#architectural-vision-and-historical-foundation)
  - [Performance and Collaboration Requirements](#performance-and-collaboration-requirements)
  - [Data Flow Challenges](#data-flow-challenges)
- [Decision](#decision)
- [Core Architecture: Lazy Frontend-Driven Updates](#core-architecture-lazy-frontend-driven-updates)
  - [Foundational Principles](#foundational-principles)
  - [The Two-Phase Communication System](#the-two-phase-communication-system)
  - [Simple Interaction Examples](#simple-interaction-examples)
  - [Complex Collaborative Example](#complex-collaborative-example)
- [Lazy Architecture Deep Dive: Why Laziness Is Essential](#lazy-architecture-deep-dive-why-laziness-is-essential)
  - [Beyond Traditional Desktop Patterns](#beyond-traditional-desktop-patterns)
  - [The Scaling Challenge](#the-scaling-challenge)
  - [Comprehensive Lazy Implementation](#comprehensive-lazy-implementation)
  - [Two-Tier Performance System](#two-tier-performance-system)
  - [Performance Characteristics](#performance-characteristics)
  - [Collaborative Scale Example](#collaborative-scale-example)
  - [Layout Opening Examples](#layout-opening-examples)
  - [Performance Optimization: Raster Deduplication](#performance-optimization-raster-deduplication)
- [Implementation Architecture](#implementation-architecture)
  - [1. Frontend Layout Object Hierarchy](#1-frontend-layout-object-hierarchy)
  - [2. Two-Phase Communication Implementation](#2-two-phase-communication-implementation)
  - [3. Frontend Cache Management: Clean vs Dirty Tracking](#3-frontend-cache-management-clean-vs-dirty-tracking)
  - [4. Raster-Based Performance Optimization](#4-raster-based-performance-optimization)
  - [Internalized MVC Architecture: Platform Independence Through Design](#internalized-mvc-architecture-platform-independence-through-design)
  - [Frontend Technology Integration](#frontend-technology-integration)
- [Architectural Analysis and Reasoning](#architectural-analysis-and-reasoning)
  - [Historical Context: Proven Patterns Applied to Collaborative Domain](#historical-context-proven-patterns-applied-to-collaborative-domain)
  - [Alternative Architectures Considered](#alternative-architectures-considered)
  - [Critical Design Decisions and Trade-offs](#critical-design-decisions-and-trade-offs)
- [Rationale: Why Lazy Frontend-Driven Architecture](#rationale-why-lazy-frontend-driven-architecture)
  - [Architectural Evolution from Desktop to Collaborative Systems](#architectural-evolution-from-desktop-to-collaborative-systems)
  - [Lazy vs Eager Computation](#lazy-vs-eager-computation)
  - [Two-Phase Pull/Push vs Single Request-Response](#two-phase-pullpush-vs-single-request-response)
  - [Raster Batching vs Immediate Processing](#raster-batching-vs-immediate-processing)
  - [Frontend Authority vs Backend Authority](#frontend-authority-vs-backend-authority)
- [Consequences](#consequences)
  - [Positive Outcomes](#positive-outcomes)
  - [Operational Considerations and Risks](#operational-considerations-and-risks)
  - [Trade-offs and Limitations](#trade-offs-and-limitations)
  - [Mitigation Strategies](#mitigation-strategies)
  - [5. Visual Feedback Integration](#5-visual-feedback-integration)
- [Implementation Approach](#implementation-approach)
  - [Component Integration](#component-integration)
  - [Event Streaming Setup](#event-streaming-setup)
  - [Viewport Management](#viewport-management)
  - [Testing Strategies](#testing-strategies)
- [Example Frontend Implementation: JavaFX + Skija](#example-frontend-implementation-javafx--skija)
  - [Core Implementation Components](#core-implementation-components)
  - [Professional UI Architecture](#professional-ui-architecture)
  - [Event Streaming and State Management](#event-streaming-and-state-management)
  - [Performance Optimizations](#performance-optimizations)
- [References](#references)
  - [Related ADRs](#related-adrs)
  - [Technical Documentation](#technical-documentation)
- [Notes](#notes)

## Context

Ooloi's frontend-backend separation ([ADR-0001](0001-Frontend-Backend-Separation.md)) creates the need for explicit data synchronization patterns between any frontend implementation and the Clojure backend. The frontend must efficiently handle real-time collaborative editing, large orchestral scores (30-40 staves or more), and responsive user interactions while maintaining clean architectural boundaries.

### Architectural Vision and Historical Foundation

This lazy event-driven architecture builds on patterns proven in Igor Engraver (1996), but represents **fundamental architectural evolution** beyond its inspiration. Igor Engraver demonstrated successful viewport-driven computation and invalidation cycles within Common Lisp's MVC environment, but **did not employ systematic lazy evaluation**.

**The Translation Challenge**: Igor Engraver benefited from Common Lisp's mature MVC environment on Mac, where viewport management, invalidation cycles, and rendering coordination were provided "for free" by the desktop framework. Ooloi operates in a fundamentally different context—distributed systems with unknown frontend technologies where **nothing can be assumed** about viewport handling, event management, or rendering capabilities.

**Architectural Internalization**: MVC mechanisms that Igor inherited from its desktop environment have been **deliberately internalized into Ooloi's architecture** to ensure any frontend technology can implement the lazy patterns successfully. Rather than requiring frontends to implement complex MVC patterns, Ooloi carries the complexity burden in its backend where it can be managed systematically.

**Scaling Through Systematic Laziness**: Most critically, Ooloi operates under fundamentally different scaling constraints—distributed collaboration, massive orchestral scores, unknown frontend performance characteristics, network latency—that **demand comprehensive lazy evaluation** at every architectural layer. Where Igor could rely on desktop performance and single-user scenarios, Ooloi must **exploit every laziness opportunity** to surmount scaling problems that would be intractable with traditional eager computation approaches.

**Multi-Client as Architectural Default**: This architecture assumes **collaborative multi-user is the default** and single-user combined applications are just transport-optimized special cases (N=1 client with in-process gRPC). The lazy communication patterns, raster synchronization, and cache management are designed for the general case of multiple simultaneous users, making single-user scenarios automatically benefit from the collaborative optimization work.

Previous Ooloi ADRs—STM for concurrency (ADR-0004), VPDs for hierarchical addressing (ADR-0008), Timewalk for efficient traversal (ADR-0014)—were chosen specifically to enable this architectural vision in a modern, collaborative context. This represents **architectural necessity driven by collaborative scale**, not just modernization of proven patterns.

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

We will implement a **lazy, event-driven data synchronization architecture** that enables real-time collaborative editing of large orchestral scores through two fundamental patterns: **pull-based data requests** and **push-based invalidation notifications**.

## Core Architecture: Lazy Frontend-Driven Updates

### Foundational Principles

**Backend Authority, Frontend Rendering**:
- **The backend** holds all musical data and performs all layout calculations
- **The frontend** renders using backend-computed drawing instructions
- **Platform Independence**: Backend generates abstract drawing descriptions (fonts, positions, curves, lines) without knowledge of how frontends will render them

**Key Insight**: The frontend never computes musical layout - it only requests pre-computed, platform-independent drawing data from the backend and renders it using its chosen rendering technology.

### The Two-Phase Communication System

Ooloi operates through two completely separate, asynchronous communication patterns:

**Phase 1: Pull-Based Data Requests (Frontend → Backend)**
```
Frontend needs drawing data → gRPC request → Backend computes OR retrieves cached data → Returns drawing primitives
```
*Frontend drives when computation happens by requesting what it needs to display*

**Phase 2: Push-Based Invalidation Notifications (Backend → Frontend)**  
```
Backend musical change → Measures marked "dirty" → Periodic broadcast → All clients mark their caches invalid
```
*Backend announces what data is now stale, but doesn't send new data yet*

**Critical Understanding**: Phase 2 notifications contain **NO drawing data** - only measure identifiers that are now invalid. Clients request fresh data via Phase 1 when they actually need to render those measures.

### Simple Interaction Examples

**Example 1: User Scrolls to New Page**
1. Frontend detects scroll to page 15 (measures 85-92)
2. Frontend requests drawing data for these measures (Phase 1)
3. Backend has clean cached data → immediate response (0.1ms)
4. Frontend renders page instantly using drawing instructions

**Example 2: User Adds Simple Articulation**  
1. User clicks note in measure 127, adds staccato
2. Frontend sends mutation request to backend
3. Backend modifies musical data, marks measure 127 dirty
4. At next broadcast interval, backend notifies all clients: "measure 127 invalidated" (Phase 2)
5. Clients viewing measure 127 request fresh drawing data (Phase 1)
6. Backend computes new layout, sends updated drawing instructions
7. All clients re-render measure 127 with staccato

### Complex Collaborative Example

**Scenario**: User adds staccato to a note in a complex orchestral symphony

**Step 1: User Interaction**
- User clicks note in Violin I, measure 127 of full orchestral score
- Frontend immediately shows visual selection (organic pulsating animation)
- Frontend identifies the musical element using cached backend identification data

**Step 2: Backend Mutation (Phase 1 - Pull)**
- Frontend sends gRPC mutation: `add-articulation` with precise musical coordinates
- Backend receives call and modifies the pitch object in the musical data
- **No drawing data is computed yet** - just the musical change

**Step 3: Cascade Effect Analysis**
- Backend uses timewalk transducers to efficiently analyze impact across massive score:
```clojure
;; Find all layouts containing the modified pitch using timewalk
(defn find-affected-layouts [piece modified-pitch-vpd]
  (sequence (comp (timewalk {:boundary-vpd []})  ; Search entire piece
                  (filter layout?)               ; Only layout objects
                  (filter #(layout-contains-measure? (first %) modified-pitch-vpd))
                  (map #(let [[layout vpd position] %]
                          {:layout-id (get-layout-id layout)
                           :affected-measures (calculate-cascade-measures layout modified-pitch-vpd)})))
            [piece]))
```
- **Discovered layouts**:
  - Full score (30 staves): measures 125-130 (spacing cascade)
  - Violin I part: measure 127  
  - Piano reduction: condensed measure ~42 (different measure numbering)
  - MIDI playback layout: measure 127
- Backend marks **~150 affected measures as "dirty"** using efficient transducer pipeline
- **Key Point**: Timewalk enables efficient traversal of massive orchestral hierarchies - no layout computation yet

**Step 4: Invalidation Broadcast (Phase 2 - Push)**  
- At next 100ms boundary, backend broadcasts to ALL clients:
- `{:type "measures-invalidated", :affected-measures [{:layout-id "score", :measures [125,126,127,128,129,130]}, {:layout-id "violin-part", :measures [127]}]}`
- **Broadcast contains NO drawing data** - just "these measures are now stale"

**Step 5: Client-Side Cache Management**
- **Conductor** (viewing score page 12): Sees measures 125-130 marked invalid → immediately requests fresh drawing data (Phase 1)
- **Violinist** (viewing part page 2): Receives invalidation for measure 127 → marks local cache dirty but **doesn't request data** (measure not visible)
- **Pianist** (viewing piano reduction): Marks measure 42 dirty but **doesn't request data** (not currently viewing that page)

**Step 6: Lazy Computation Triggered**
- Only the conductor's request triggers actual computation
- Backend computes layout for measures 125-130 in score layout
- Returns ~15KB of glyph/curve drawing primitives to conductor
- **Other layouts remain uncomputed** until someone requests them

**Why This Architecture Works**: Single user change in massive orchestral score → 99% of potential computation work deferred until actually needed → sub-50ms response times for visible content

## Lazy Architecture Deep Dive: Why Laziness Is Essential

### Beyond Traditional Desktop Patterns

While Igor Engraver proved viewport-driven computation patterns, it operated in a single-user desktop context where systematic lazy evaluation wasn't required. Traditional music notation software could afford eager computation because of single-user scenarios, desktop performance, and bounded score complexity.

Ooloi's collaborative distributed context faces **multiplicative complexity** that makes lazy evaluation architecturally essential, not just beneficial.

### The Scaling Challenge

**Collaborative Multiplication Problem**:
- 4 users × 120-page score = potential for 480 pages of redundant computation without laziness
- Each user edit could trigger recomputation across all layouts for all users  
- Network bandwidth explosion: eager updates could generate megabytes of unused layout data
- Frontend performance unpredictability: cannot assume any frontend can handle eager data floods

**Scale-Dependent Performance Requirements**:
- 120-page orchestral score: ~100,000 musical objects requiring layout computation
- 30+ simultaneous staves: spacing changes cascade across entire systems
- Real-time collaboration: multiple users editing simultaneously with immediate feedback requirements
- Cross-platform deployment: mobile devices, web browsers, desktop applications with varying performance characteristics

### Comprehensive Lazy Implementation

**1. Computation Laziness**: Backend never computes layout until explicit client request
```clojure
(defn request-drawing-data [layout-id measures]
  (if-let [cached (get-cached-data layout-id measures)]
    (send-immediately cached)      ; Path A: instant for clean data  
    (queue-for-computation layout-id measures))) ; Path B: deferred for dirty data
```

**2. Data Transfer Laziness**: Invalidation notifications completely separate from data transfer
```clojure
;; Broadcast: "measure 127 is stale" (50 bytes)
;; NOT: "here's the new 15KB layout data for measure 127"  
{:type :measures-invalidated :measures [127] :layout-ids ["score" "parts"]}
```

**3. Memory Laziness**: Persistent caching with selective eviction under pressure
```clojure
;; Never discard computed data unless memory pressure forces eviction
;; Invisibility is temporary - users scroll, switch views
(defn cache-drawing-data [layout-id measure-num data]
  (when-not (memory-pressure?)
    (persistent-cache-store layout-id measure-num data)))
```

**4. Traversal Laziness**: Timewalk transducers with boundary-vpd pruning
```clojure
;; O(log n) lazy traversal instead of O(n) eager scanning
(sequence (comp (timewalk {:boundary-vpd [:staves 0 :measures 127]})
                (filter note?)
                (take 1))  ; Stop at first match
          [piece])
```

**5. Rendering Laziness**: Frontend only processes visible viewport content

### Two-Tier Performance System

Every data request follows one of two completely different performance paths:

**Path A - Clean Data (95% of requests)**: 0.1ms response time for cached drawing instructions
**Path B - Dirty Data (5% of requests)**: Minimum raster interval delay (configurable timing) plus computation time for uncached layout data

This creates the performance breakthrough: most user interactions (scrolling, navigation, reopening views) feel instantaneous while only novel content experiences brief computational delay.

### Performance Characteristics

- **Score complexity independence**: 120 pages performs identically to 12 pages
- **Collaborative user independence**: 4 users don't multiply computation cost
- **Frontend performance independence**: works efficiently on mobile and desktop
- **Network latency tolerance**: lazy patterns minimize data transfer requirements

**The Necessity Principle**: Every architectural component must maximize lazy evaluation because collaborative scale makes eager approaches computationally intractable. This isn't optimization—it's scaling survival.

### Collaborative Scale Example

**120-page symphonic score, 4 simultaneous users:**
- **Conductor** (full score, page 12): Sees immediate update to measure 127
- **Violinist** (part only): Continues working on page 2, unaware until opening page containing measure 127  
- **Pianist** (piano reduction): Working on different section, no immediate impact
- **Copyist** (full score, page 50): No computational or network overhead until scrolling to affected area

**Network efficiency**: Single note change → ~4KB of invalidation notifications → Only visible measures trigger data requests (~10KB per visible page) → Total network impact: <50KB instead of reformatting entire 120-page score

**Computational efficiency**: Only the conductor's visible page reformatted immediately → Other layouts remain uncomputed until needed → 99% of the score's formatting work deferred

### Layout Opening Examples

**First-Time Layout Opening: Violin I Part**

1. **User Action**: Violinist opens part layout for first time
2. **Frontend Request**: Requests drawing data for all visible measures (1-12)
3. **Backend State**: Part layout never computed → All measures dirty
4. **Mass Computation**: Backend computes 12 measures of violin part
5. **Cascade Invalidation**: Computing measures 8-10 reveals spacing issues → measures 7-11 marked dirty in other parts
6. **Data Transfer**: ~50KB of glyph/curve data sent to frontend
7. **Frontend Rendering**: Violin part displays immediately with drawing instructions
8. **Background Effect**: Other parts now have measures 7-11 marked dirty, but won't compute until those parts are opened

**Subsequent Layout Opening: Violin I Part**

1. **User Action**: Violinist reopens part layout (after working on full score)  
2. **Frontend Request**: Requests drawing data for visible measures (1-12)
3. **Backend State**: Part layout fully computed and cached
4. **Instant Transfer**: ~50KB of cached drawing data sent immediately
5. **No Computation**: Zero backend computation time - pure data retrieval
6. **Frontend Rendering**: Violin part displays instantly

**Layout Opening with Partial Staleness: Full Score Page 15**

1. **User Action**: Conductor scrolls to page 15 (measures 85-92)
2. **Frontend Request**: Requests drawing data for 8 measures × 30 staves = 240 measure views
3. **Backend State**: Mixed - measures 87-89 dirty from recent edits, others cached
4. **Selective Computation**: Only 3 measures × 30 staves recomputed (~90 measure views)  
5. **Cached Retrieval**: Remaining 150 measure views served from cache
6. **Cascade Effect**: Reformatting measure 88 affects page layout → measure 93 (off-page) marked dirty
7. **Data Transfer**: ~300KB mixed fresh/cached data sent to frontend
8. **Deferred Impact**: Measure 93 dirty but won't compute until page 16 becomes visible

**Cache Persistence Philosophy**:
- **Never discard**: Computed drawing data permanently cached until piece structure changes
- **Invisibility tolerance**: Closed layouts, scrolled-out pages maintain full cache
- **Memory efficiency**: Terse drawing instructions (fonts + beziers) vs bitmap storage
- **Collaborative benefit**: User A's computation work benefits User B when they open same layout

### Performance Optimization: Raster Deduplication

**The Fixed Computation Raster**

Backend uses a **configurable raster** (e.g., 10 FPS / 100ms) for **invalidation notifications** and **dirty data computation requests**. Normal API requests (musical mutations, clean data retrieval) process immediately without raster delays. This creates natural deduplication windows for computation-heavy operations while keeping normal operations responsive.

**Deduplication Through Raster Synchronization:**

```clojure
;; Requests queue until next raster pulse
(def pending-drawing-requests (atom #{}))

;; Every 100ms - the raster pulse processes all queued requests
(defn process-raster-pulse []
  (let [requests @pending-drawing-requests]
    (reset! pending-drawing-requests #{})
    ;; Group by [layout-id measure-num] - compute once per unique measure
    (doseq [[[layout-id measure-num] client-requests] 
            (group-by #(select-keys % [:layout-id :measure-num]) requests)]
      (let [drawing-data (compute-or-retrieve-drawing-data layout-id measure-num)]
        ;; Send same result to all clients who requested this measure
        (doseq [{:keys [client-id]} client-requests]
          (send-drawing-data client-id drawing-data))))))
```

**Duplicate Work Prevention Examples:**

**Scenario 1: Simultaneous Layout Opening**
- **1047ms**: Conductor opens full score page 12 → requests measures 85-92
- **1089ms**: Violinist opens same full score page 12 → requests measures 85-92  
- **1100ms**: Raster pulse processes both requests → each measure computed once, sent to both clients
- **Result**: Zero duplicate computation - natural batching

**Scenario 2: Collaborative Scrolling Storm**
- **1034ms**: User A scrolls to page 15 → requests 240 measure views
- **1067ms**: User B scrolls to page 15 → requests same 240 measure views
- **1091ms**: User C opens page 15 → requests same 240 measure views
- **1100ms**: Raster pulse → 240 measures computed once, results distributed to all three users
- **CPU savings**: 66% reduction (3 users × 240 measures = 720 computations → 240 computations)

**Edge Case: Cross-Raster Boundary**
- **1099ms**: User A requests measure 127 → queued for 1100ms raster
- **1101ms**: User B requests measure 127 → queued for 1200ms raster
- **Result**: Two separate computations (1% probability timing collision)

**Raster Benefits:**
- **Natural deduplication**: No explicit request tracking needed
- **Batch efficiency**: Processes all requests at consistent intervals
- **Collaborative optimization**: Multiple users automatically benefit from shared computation
- **Bounded scheduling**: Dirty data requests scheduled at consistent raster intervals, though actual computation may take longer for complex layouts
- **STM compatibility**: Raster pulse operates within single transaction scope

**Performance Impact**: Reduces duplicate computation from ~50% probability (immediate duplicate requests) to ~1% probability (cross-boundary timing collisions). For MVP deployment, raster synchronization provides sufficient deduplication without additional complexity.


## Implementation Architecture

We will implement this lazy system with five key components (illustrated using the standard Clojure/JavaFX frontend, though any frontend technology can implement these patterns):

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

### 2. Two-Phase Communication Implementation

**Implementing the Pull-Push Architecture**: Frontend and backend communicate through two completely independent gRPC streams that implement our foundational two-phase system.

**Phase 1 Implementation: Pull-Based Data Requests**
```clojure
;; Frontend requests drawing data when needed for rendering
(defn request-drawing-data [layout-id measure-nums]
  (let [response-stream (grpc/request-layout-data 
                         {:layout-id layout-id 
                          :measures measure-nums
                          :client-id @client-id})]
    ;; Response follows two-tier system:
    ;; - Clean data: immediate response with cached drawing instructions  
    ;; - Dirty data: queued until next 100ms raster pulse
    (handle-drawing-data-response response-stream)))
```

**Phase 2 Implementation: Push-Based Invalidations**
```clojure
;; Backend streams invalidation notifications (no drawing data)
(defn handle-invalidation-stream []
  (let [invalidation-stream (grpc/subscribe-to-invalidations @client-id)]
    (go-loop []
      (when-let [event (<! invalidation-stream)]
        (case (:type event)
          :measures-invalidated 
          (doseq [{:keys [layout-id measures]} (:affected-measures event)]
            ;; Mark caches dirty - don't request data yet
            (mark-measures-dirty layout-id measures)
            ;; Only request fresh data if measures are currently visible
            (when (measures-visible? layout-id measures)
              (request-drawing-data layout-id measures)))
          :layout-structure-changed 
          (invalidate-entire-layout (:layout-id event)))
        (recur)))))
```

**Critical Implementation Details**:
- **Phase 1 requests trigger raster computation**: Dirty data waits for next 100ms pulse
- **Phase 2 notifications contain zero drawing data**: Only measure identifiers
- **Lazy request triggering**: Frontend only pulls data for visible measures
- **Independent streams**: Pull and push operate on separate gRPC connections

### 3. Frontend Cache Management: Clean vs Dirty Tracking

**Frontend Data Structure**: Each frontend measure view tracks whether its drawing data is clean (valid) or dirty (needs refresh from backend).

```clojure
;; Frontend measure view implements our clean/dirty architecture
(defrecord FrontendMeasureView [
  vpd                    ; Vector Path Descriptor for backend identification
  layout-id              ; Which layout this measure belongs to  
  measure-num            ; Measure number within layout
  measure-view-data      ; Backend MeasureView record (when clean): {:glyphs [...] :curves [...]}
  cache-state           ; :clean, :dirty, or :requesting
  last-backend-version   ; Version stamp from backend
])

;; Backend MeasureView structure (from ooloi.backend.models.visual.measure-view)
;; (defrecord MeasureView [glyphs curves])
;; - glyphs: Vector of platform-agnostic glyph descriptions (structure TBD)
;; - curves: Vector of platform-agnostic curve descriptions (structure TBD)
;; 
;; Frontend responsibility: interpret glyphs/curves for chosen rendering technology
;; - JavaFX+Skija frontend: converts to Skija drawing commands
;; - Canvas frontend: converts to HTML Canvas operations  
;; - Future VR frontend: converts to 3D rendering primitives
```

**Clean/Dirty State Management**:
```clojure
;; Rendering follows two-tier performance model - frontend interprets backend drawing data
(defn render-measure [measure-view frontend-renderer]
  (case (:cache-state measure-view)
    :clean 
    ;; Path A: Immediate rendering using cached platform-independent drawing data
    (doseq [element (:drawing-instructions measure-view)]
      (frontend-renderer/render-element element))  ; Frontend interprets drawing-data
    
    :dirty
    ;; Path B: Request from backend, show placeholder until data arrives
    (do
      (request-drawing-data (:layout-id measure-view) [(:measure-num measure-view)])
      (update-cache-state measure-view :requesting)
      (frontend-renderer/render-placeholder "Computing..."))))

;; Phase 2 invalidations mark frontend caches dirty - backend agnostic
(defn handle-invalidation-notification [affected-measures]
  (doseq [{:keys [layout-id measures]} affected-measures]
    (doseq [measure-num measures]
      (mark-measure-cache-dirty layout-id measure-num))))
```

**Hit-Testing Integration**:
```clojure
;; User click → identify backend element for Phase 1 mutation call  
(defn resolve-click-to-backend [click-point measure-view]
  (when (= :clean (:cache-state measure-view))
    ;; Only clickable if we have clean cached data
    (some (fn [glyph]
            (when (point-in-bounds? click-point (:bounds glyph))
              {:vpd (:vpd measure-view) :item-id (:item-id glyph)}))
          (:drawing-instructions measure-view))))
```

**Identification Principles**:
- **VPD + item-id** uniquely identifies any musical element for backend operations
- **Timewalk traversal** efficiently searches layout hierarchy using transducer composition
- **VPD backlinks** can use search instead of direct addressing for flexibility
- **Hit-testing** resolves user clicks to backend identifiers using timewalk filtering
- **No musical interpretation** in frontend - pure geometric-to-logical mapping
- **Version tracking** prevents stale data operations
- **Backend-computed drawing instructions** are cached until backend signals invalidation
- **Frontend never computes glyph shapes** - all visual rendering comes from backend

### 4. Raster-Based Performance Optimization

**10 FPS Raster System Implementation**: The backend's fixed 100ms raster pulse drives all performance optimizations, implementing our foundational lazy architecture.

```clojure
;; Backend raster pulse - processes all queued requests every 100ms
(defn process-raster-pulse []
  (let [queued-requests @pending-drawing-requests]
    (reset! pending-drawing-requests #{})
    ;; Group identical requests for deduplication
    (let [grouped-requests (group-by #(select-keys % [:layout-id :measure-num]) queued-requests)]
      (doseq [[[layout-id measure-num] client-requests] grouped-requests]
        ;; Compute using timewalk for efficient large-score traversal
        (let [drawing-data (compute-measure-drawing-data layout-id measure-num)]
          (doseq [{:keys [client-id]} client-requests]
            (send-drawing-data client-id drawing-data)))))))

;; Backend MeasureView contains platform-independent glyphs and curves
(defn compute-measure-drawing-data [layout-id measure-num]
  (let [layout (get-layout layout-id)
        measure-vpd [:layout layout-id :pages 0 :systems 0 :staves 0 :measures measure-num]]
    ;; Use timewalk to efficiently extract the MeasureView 
    (first (sequence (comp (timewalk {:boundary-vpd measure-vpd})
                           (filter measure-view?)
                           (map first)) ; Extract the MeasureView record
                     [layout]))))

;; MeasureView structure from backend
;; (defrecord MeasureView [glyphs curves])
;; - glyphs: Vector of platform-independent glyph descriptions 
;; - curves: Vector of platform-independent curve descriptions
;; Frontend interprets these according to its rendering technology
```

**Frontend Viewport Management**:
```clojure
;; Frontend uses timewalk transducers for efficient visibility calculations in large scores
(defrecord Viewport [
  visible-measures     ; Set of [layout-id measure-num] currently on screen
  dirty-measures       ; Set of measures marked invalid by Phase 2 notifications
  requesting-measures  ; Set of measures queued for next raster pulse
  layout-hierarchy     ; Cached frontend layout structure for timewalk traversal
])

(defn calculate-visible-measures [viewport bounds]
  ;; Use timewalk to efficiently find measures intersecting viewport bounds
  (sequence (comp (timewalk {:boundary-vpd (:visible-region viewport)})
                  (filter measure-view?)
                  (filter #(bounds-intersect? bounds (get-measure-bounds (first %))))
                  (map #(let [[measure vpd position] %]
                          [(:layout-id measure) (:measure-num measure)])))
            [(:layout-hierarchy viewport)]))

(defn handle-scroll [viewport new-bounds]
  (let [newly-visible (calculate-visible-measures viewport new-bounds)]
    ;; Only request dirty measures that are now visible - transducer pipeline
    (sequence (comp (filter #(contains? (:dirty-measures viewport) %))
                    (map #(request-drawing-data (first %) (second %))))
              newly-visible)))
```

**Lazy Performance Characteristics**:
- **Timewalk efficiency**: Transducer pipelines enable O(log n) traversal of massive orchestral hierarchies instead of O(n) linear search
- **Computation batching**: Multiple client requests deduplicated in single raster pulse
- **Visibility-driven requests**: Only visible measures trigger backend computation using bounded timewalk searches
- **Persistent caching**: Computed drawing data cached until musical structure changes
- **Bounded scheduling**: Dirty data requests scheduled at raster intervals, with actual computation time varying based on layout complexity

**Large Score Scalability**: A 120-page orchestral score with 30 staves contains ~100,000 musical objects. Without timewalk transducers:
- Finding affected layouts: O(100,000) linear scan
- Visibility calculations: O(100,000) bounds checking
- Measure data extraction: O(100,000) object filtering

With timewalk transducers:
- Finding affected layouts: O(log 100,000) + filtered objects only
- Visibility calculations: O(viewport measures) using boundary-vpd pruning  
- Measure data extraction: O(measure objects) using precise VPD targeting

### Internalized MVC Architecture: Platform Independence Through Design

**The Igor Engraver Context**: In 1996, Igor Engraver operated within Common Lisp's desktop environment where viewport management, invalidation handling, and event coordination were provided by the GUI framework. The application could rely on mature MVC infrastructure.

**The Ooloi Challenge**: Operating across diverse frontend technologies (JavaFX, web browsers, mobile platforms), Ooloi cannot assume MVC infrastructure exists. Each frontend handles viewports, events, and rendering differently, if at all.

**Architectural Solution - Internalized Coordination Mechanisms**:

Rather than requiring frontends to implement MVC patterns, Ooloi internalizes coordination mechanisms into its architecture:

**Viewport Management**: Backend calculates which measures intersect viewport bounds rather than assuming frontend viewport management capabilities.

**Invalidation Logic**: Backend handles cascade analysis and dirty/clean state tracking. Frontends receive notifications but don't need sophisticated invalidation frameworks.

**Event Coordination**: Two-phase pull/push communication eliminates complex frontend event management requirements.

**Cache Management Structure**: Backend provides clean/dirty state specifications that any frontend caching approach can implement.

**Results**: This enables compatibility across frontend sophistication levels—from simple canvas drawing to complex frameworks—while maintaining consistent behavior.

### Frontend Technology Integration

**Platform Independence**: The architecture works across frontend technologies. Each implementation interprets backend's platform-independent drawing data according to its rendering capabilities.

**Implementation Examples**:
- **Clojure/JavaFX**: Desktop client using JavaFX windowing + Skija rendering
- **Web Frontend**: Browser-based client using HTML Canvas or WebGL
- **Mobile Frontend**: Native iOS/Android clients with platform-specific rendering
- **Other Frontends**: VR, AR, or specialized display technologies

**Integration Requirements**: All frontends implement the two-phase communication pattern and cache management, but rendering technology remains flexible.

## Architectural Analysis and Reasoning

### Historical Context: Proven Patterns Applied to Collaborative Domain

**MVC Viewport Pattern Foundation**: This architecture applies the classic desktop application Model-View-Controller viewport pattern to collaborative web applications. Each `MeasureView` functions as a traditional GUI viewport element:

- **Model**: Backend musical data (authoritative)  
- **View**: Platform-independent drawing instructions (`MeasureView{glyphs, curves}`)
- **Controller**: Frontend requesting updates when viewport changes

**Igor Engraver Precedent**: Similar invalidation/recomputation logic has been successfully implemented in Igor Engraver's desktop architecture. The cascade analysis (adding staccato → invalidating measures 125-130) follows deterministic rules proven in production music notation software.

**Classic Lisp MVC Controllers**: The two-phase pull/push pattern mirrors Common Lisp MVC controllers where views request updates on-demand while models broadcast invalidation events. This is mature, well-understood technology.

### Alternative Architectures Considered

**Alternative 1: Eager Computation with Immediate Propagation**
```
User Edit → Recompute All Affected Layouts → Send Complete Updates → Client Rendering
```

**Pros**: Simple, consistent state, immediate updates
**Cons**: 
- Prohibitive for large scores (30 staves × 1000 measures = massive computation)
- Network bandwidth explosion for collaborative editing
- Poor scaling with score complexity

**Why Rejected**: A single note change in a large orchestral work could trigger minutes of computation for layouts no user is viewing.

**Alternative 2: Client-Side Layout Computation**
```  
Backend Musical Data → Frontend Computes Layout → Independent Rendering
```

**Pros**: Reduced backend load, immediate local updates
**Cons**:
- Massive code duplication (layout algorithms in every frontend)
- Collaborative consistency nightmares (different clients compute different layouts)
- Platform dependency issues (layout varies by rendering technology)

**Why Rejected**: Multiple frontends would require maintaining identical layout computation in JavaScript, Java, Swift, etc. Collaborative editing becomes impossible with divergent layout results.

**Alternative 3: Traditional Request-Response with Full Data**
```
User Edit → Backend Update → Frontend Requests All Affected Data → Complete Transfer
```

**Pros**: Simple protocol, stateless communication
**Cons**:
- Forces computation of invisible layouts
- Large data transfers for every edit
- No deduplication of identical requests from multiple clients

**Why Rejected**: Doesn't scale to collaborative scenarios where multiple users edit simultaneously.

**Alternative 4: Event Sourcing with Client State Replay**
```
User Edit → Event Log → All Clients Replay Events → Derived Layout State
```

**Pros**: Strong consistency, audit trail, offline capability
**Cons**:
- Requires every client to maintain complete layout computation capability
- Complex conflict resolution for simultaneous edits
- Large event logs for complex scores

**Why Rejected**: Pushes layout complexity to every client, undermining platform independence.

### Critical Design Decisions and Trade-offs

**Decision 1: Raster Timing (configurable interval vs immediate processing)**

**Why Chosen**: Batches duplicate requests naturally without complex coordination
**Trade-off**: Introduces bounded latency for dirty data requests
**Alternatives Considered**:
- Immediate processing: Race conditions, duplicate computation waste
- Fixed longer batching: Better deduplication but potential perceived lag
- Request coalescing: Complex state tracking, unclear benefit

**Analysis**: Configurable raster timing allows tuning between deduplication efficiency and response latency based on deployment architecture:
- **Combined model (in-process gRPC)**: 20ms provides near-instantaneous response while maintaining deduplication benefits
- **SaaS deployment (network gRPC)**: 100ms optimizes for network efficiency and collaborative deduplication
- **High-latency networks**: Longer intervals may be appropriate for international collaboration

Proper presentation (loading indicators, progressive rendering) makes these delays imperceptible to users regardless of chosen interval.

**Decision 2: Backend Authority vs Distributed Computation**

**Why Chosen**: Single source of truth prevents consistency issues
**Trade-off**: Backend becomes bottleneck, requires more server resources  
**Alternatives Considered**:
- Peer-to-peer layout computation: Complex conflict resolution
- Frontend layout computation: Platform dependency, code duplication
- Hybrid approach: Increased complexity, unclear benefits

**Analysis**: Musical layout computation requires precise, deterministic results. Distributed computation risks divergent visual layouts between collaborators.

**Decision 3: Two-Phase Communication vs Unified Protocol**

**Why Chosen**: Decouples invalidation (broadcast) from data transfer (on-demand)
**Trade-off**: Protocol complexity, potential invalidation stream failures
**Alternatives Considered**:
- Unified request-response: Simple but forces unnecessary computation
- WebSocket with mixed messages: Protocol ambiguity, harder to debug
- Polling-based updates: Network inefficiency, delayed updates

**Analysis**: Separation enables network optimization and lazy evaluation. Complexity is manageable with proper error handling.

**Decision 4: Platform Independence vs Frontend-Specific Optimization**

**Why Chosen**: Enables multiple frontend technologies using same backend
**Trade-off**: Potential performance loss from abstraction layer
**Alternatives Considered**:
- Skija-specific backend: Optimal for primary frontend but locks in technology
- Multiple backend renderers: Code duplication, maintenance burden
- Frontend-negotiated formats: Protocol complexity

**Analysis**: Platform independence provides strategic flexibility for future frontends (mobile, web, VR) at minimal performance cost.

## Rationale: Why Lazy Frontend-Driven Architecture

### Architectural Evolution from Desktop to Collaborative Systems

This architecture represents the purposeful evolution of proven desktop patterns for collaborative distributed systems, not accidental architectural emergence. The sophisticated foundational decisions in previous ADRs—STM concurrency (ADR-0004), hierarchical VPDs (ADR-0008), efficient Timewalk traversal (ADR-0014)—were chosen specifically to enable this lazy event-driven architecture in a collaborative context.

**Deliberate Vision Realization**: Where Igor Engraver succeeded with eager computation in single-user desktop scenarios, Ooloi's collaborative distributed context requires comprehensive lazy evaluation at every architectural layer to achieve practical scaling characteristics. This represents systematic advancement rather than incremental improvement.

**MVC Architecture Internalization**: The decision to internalize MVC coordination mechanisms (viewport management, invalidation logic, event coordination) into the backend rather than assuming frontend capabilities enables universal frontend compatibility while preserving proven patterns from desktop environments.

### Lazy vs Eager Computation

**Why Lazy Computation**:
- **Massive score scalability**: 99% of layout work deferred until actually needed
- **Collaborative efficiency**: Multiple users don't trigger redundant computation
- **Resource optimization**: CPU cycles spent only on visible content
- **Predictable performance**: Response time independent of total score size

**vs Eager Computation**:
- Eager systems would compute all layouts immediately after any change
- Prohibitive for large orchestral scores (30+ staves, 1000+ measures)
- Collaborative editing would trigger exponential computation waste
- Response times scale with total piece complexity, not viewport complexity

### Two-Phase Pull/Push vs Single Request-Response

**Why Separate Pull and Push**:
- **Decoupled communication**: Data requests independent of invalidation notifications
- **Network efficiency**: Invalidations broadcast tiny payloads, data flows only when needed
- **Lazy triggering**: Clients decide when they actually need fresh data
- **Collaborative scalability**: One user's edit notifies all users, but doesn't force computation

**vs Traditional Request-Response**:
- Single-phase systems couple mutation with data retrieval
- Frontend must request all potentially affected data after any change
- No way to defer computation until data is actually needed for rendering
- Collaborative updates require either polling or complex change-tracking

### Raster Batching vs Immediate Processing

**Why Fixed 100ms Raster**:
- **Natural deduplication**: Multiple clients requesting same data get single computation
- **Predictable latency**: Users know maximum wait time for any request  
- **Batch efficiency**: Process multiple requests in single transaction
- **STM compatibility**: All computations within single consistent state snapshot

**vs Immediate Processing**:
- Immediate processing creates race conditions in collaborative scenarios
- No deduplication of duplicate requests
- Higher CPU usage due to redundant computations
- Unpredictable response times during collaborative editing storms

### Frontend Authority vs Backend Authority

**Why Backend Authority**:
- **Single source of truth**: All musical logic and layout computation centralized
- **Collaborative consistency**: Multiple clients can't create conflicting musical state
- **Frontend simplicity**: No need to duplicate complex musical algorithms
- **Lazy compatibility**: Frontend can defer requesting data it doesn't need

**vs Frontend Authority**:
- Multiple frontends would require complex state synchronization
- Risk of musical logic inconsistencies between clients
- Difficult to implement lazy evaluation with distributed authority
- Complex conflict resolution for simultaneous edits

## Consequences

### Positive Outcomes

1. **Massive Score Scalability**: 120-page orchestral works with 30+ staves perform identically to simple piano pieces - 99% of computation deferred until needed
2. **Sub-Millisecond Responsiveness**: 95% of user interactions (scrolling, navigation, reopening views) receive 0.1ms response times via Path A clean data retrieval  
3. **Collaborative Efficiency**: Multiple users viewing different sections don't trigger redundant computation - only requested layouts get computed
4. **Predictable Performance**: Maximum 100ms wait for any request, independent of total score complexity or number of collaborative users
5. **Network Optimization**: Single musical change generates 4KB invalidation broadcast instead of megabytes of layout data transfers
6. **Memory Efficiency**: Persistent caching means User A's computation work benefits User B when opening same layout - no duplicate computation waste
7. **Clean Architecture**: Complete separation of musical authority (backend) and rendering responsibility (frontend) - no logic duplication
8. **Strategic Platform Position**: Abstract drawing instructions enable potential adoption by other notation software of Ooloi's backend while maintaining their frontends
9. **Universal Frontend Compatibility**: Internalized MVC coordination enables compatibility across frontend sophistication levels without requiring complex framework implementation

### Operational Considerations and Risks

**Risk 1: Network Reliability Dependency**
- **Issue**: Frontend depends on Phase 2 invalidation stream for cache coherence
- **Impact**: Stream failures lead to stale data display
- **Mitigation**: Automatic reconnection with state resynchronization, graceful degradation with user indicators
- **Detection**: Heartbeat mechanisms, invalidation sequence numbers for gap detection

**Risk 2: Viewport Calculation Accuracy**  
- **Issue**: Frontend must accurately determine visible measures to request correct data
- **Impact**: Missing content (under-requesting) or performance waste (over-requesting)
- **Mitigation**: Conservative viewport calculation, buffer zones around visible areas
- **Validation**: Cross-check requested vs rendered measures

**Risk 3: Cold-Start Performance**
- **Issue**: First-time layout opening requires computing everything from scratch
- **Impact**: Large orchestral scores have significant initial delay
- **Measurement**: 30 staves × 100 measures ≈ 2-5 seconds computation time
- **Mitigation**: Background pre-computation for likely-viewed layouts, progressive loading
- **Presentation**: Proper UI feedback (progress indicators, partial rendering) makes longer delays acceptable to users

**Risk 4: Cascade Analysis Correctness**
- **Issue**: Over-invalidation wastes computation, under-invalidation shows stale data
- **Impact**: Performance degradation or visual inconsistencies
- **Validation**: Deterministic rules proven in Igor Engraver, comprehensive test coverage
- **Debugging**: Invalidation event logging for trace analysis

### Trade-offs and Limitations

1. **Two-Tier Complexity**: System has two distinct performance modes (immediate vs raster-delayed) requiring careful UI state management
2. **Configurable Latency Floor**: Dirty data requests wait for raster pulse (configurable timing) - cannot achieve sub-raster intervals for novel content  
3. **Cache Invalidation Precision**: Phase 2 notifications must accurately identify all affected measures across multiple layouts
4. **Network Dependency**: Frontend becomes non-functional if Phase 2 invalidation stream fails - requires robust reconnection logic
5. **Memory Growth**: Persistent caching strategy means backend memory usage grows with number of opened layouts until manual cleanup

### Mitigation Strategies

- **Deployment-specific tuning**: Raster timing optimized per deployment model (20ms in-process, 100ms SaaS, adaptive for high-latency)
- **Graceful degradation**: Frontend continues working with stale data if invalidation stream fails, with clear user indicators
- **Predictive loading**: Smart algorithms pre-fetch likely-to-be-needed measures during idle periods
- **Memory management**: Backend implements LRU eviction of cached layout data under memory pressure
- **Connection resilience**: Automatic reconnection with state resynchronization for invalidation streams
- **Performance monitoring**: Built-in metrics track two-tier response characteristics and raster pulse efficiency

### 5. Visual Feedback Integration

**Selection Animation**: Frontend selection animations integrate with the lazy architecture by only operating on visible, clean cached elements. Animation details are covered in separate UI specifications.

## Implementation Approach

### Component Integration

**Frontend Layout Manager**:
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
;; Direct JavaFX collaborative update handling with timewalk
(defn find-affected-pages [measure-vpd piece]
  ;; Use timewalk to find which pages contain the affected measure
  (sequence (comp (timewalk {:boundary-vpd []})
                  (filter page-view?)
                  (filter #(contains-measure? (first %) measure-vpd))
                  (map #(get-page-number (first %))))
            [piece]))

(defn handle-backend-event [event page-cache piece]
  (case (:scope event)
    :measure-updated
    (let [affected-pages (find-affected-pages (:measure-vpd event) piece)]
      (doseq [page affected-pages]
        ((:invalidate-page page-cache) page)
        (mark-page-for-redraw page)))
    
    :page-layout-changed
    (doseq [page (:affected-pages event)]
      ((:invalidate-page page-cache) page)
      (mark-page-for-redraw page))
    
    :full-layout-refresh
    (do
      (clear-all-page-cache page-cache)
      (trigger-full-redraw))))

(defn setup-collaborative-updates [grpc-client page-cache piece]
  (let [event-stream (grpc/subscribe-to-layout-events grpc-client)]
    (go-loop []
      (when-let [event (<! event-stream)]
        (handle-backend-event event page-cache piece)
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

;; Example: Search for all notes in visible region using timewalk
(defn find-visible-notes [piece viewport]
  (sequence (comp (timewalk {:boundary-vpd (:visible-region viewport)})
                  (filter note?)
                  (map #(let [[item vpd position] %]
                          {:item item :vpd vpd :position position})))
            [piece]))

;; Example: VPD backlink resolution using search instead of direct addressing
(defn resolve-staff-reference [staff-id piece]
  ;; Instead of constructing VPD directly, search for the staff
  (sequence (comp (timewalk {:boundary-vpd []})
                  (filter staff?)
                  (filter #(= staff-id (get-staff-id (first %))))
                  (map second) ; Extract VPD
                  (take 1))
            [piece]))
```

**Event Processing Pipeline**:
```clojure
(defn start-event-processing [layout-manager grpc-client piece]
  (go-loop []
    (when-let [event (<! (grpc/subscribe-to-layout-events grpc-client))]
      (process-layout-event layout-manager event piece)
      (recur))))

;; Example: Process selection events using timewalk for element discovery
(defn process-selection-event [event piece]
  (case (:type event)
    :select-measures
    ;; Use timewalk to find all measures in the selection range
    (let [selected-measures (sequence (comp (timewalk {:boundary-vpd (:staff-vpd event)})
                                           (filter measure?)
                                           (filter #(in-range? (:position event) (get-position (first %))))
                                           (take (:count event)))
                                     [piece])]
      (apply-selection-animation selected-measures))
    
    :select-chord
    ;; Search for all notes at the same rhythmic position
    (let [chord-notes (sequence (comp (timewalk {:boundary-vpd (:measure-vpd event)})
                                     (filter note?)
                                     (filter #(= (:position event) (get-position (first %))))
                                     (map first))
                               [piece])]
      (apply-organic-selection chord-notes))))
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