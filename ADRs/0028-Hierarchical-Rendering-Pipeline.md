# ADR-0028: Hierarchical Rendering Pipeline with Plugin-Based Formatters

**Status**: Accepted  
**Date**: 2024-09-14

## Context

Musical notation software faces computational challenges when handling large orchestral scores containing hundreds of thousands of individual musical elements. The fundamental problem involves coordinating distinct concerns: musical logic resolution, spatial arrangement calculations, and visual rendering - each with different computational characteristics and parallelisation opportunities.

**Real-World Scalability Challenge**: Traditional notation software famously struggles with large complex scores such as Strauss's Elektra recognition scene - dense orchestral passages with intricate notation that cause performance degradation, memory exhaustion, and unresponsive editing. These scenarios expose the limitations of eager computation approaches that attempt to process entire scores regardless of user viewport or editing context.

The challenge lies in providing a unified architecture that handles both fundamental notation formatting (chord layouts, articulation placement, beam positioning) and contemporary notational extensions without architectural distinction or performance compromise.

Ooloi requires an architecture that maintains responsive editing performance with the most demanding symphonic works whilst implementing all notation formatting through a unified plugin system where canonical plugins handle standard notation alongside custom extensions.

### Architectural Foundation: Frontend-Backend Separation

Ooloi's [frontend-backend separation](0001-Frontend-Backend-Separation.md) provides multiple architectural advantages that directly inform the rendering pipeline design:

**Separation of Concerns**: Frontend remains pure UI/rendering while backend handles musical logic and computation, preventing architectural conflation and maintaining clean cognitive boundaries.

**Collaboration as Consequence**: Standalone applications operate as "collaboration groups of 1" - the same architectural patterns scale naturally to multi-user scenarios without special collaborative features.

**Performance Without Compromise**: In-process gRPC transport eliminates 99% of network overhead (36μs roundtrip measured on 2017 MacBook Pro), proving separation costs nothing performance-wise whilst enabling optimal lazy evaluation patterns.

**Type Fidelity Preservation**: Clojure defrecord instances (`Tuplet`, `Pitch`, etc.) maintain identical structure across gRPC boundaries, eliminating serialization impedance mismatch and enabling plugins to operate identically in local or distributed modes.

## Decision

Ooloi implements a **four-stage hierarchical rendering pipeline** with comprehensive plugin integration and intelligent client-server coordination:

## Table of Contents

- [Context](#context)
  - [Architectural Foundation: Frontend-Backend Separation](#architectural-foundation-frontend-backend-separation)
- [Decision](#decision)
- [Four-Stage Pipeline Architecture](#four-stage-pipeline-architecture)
  - [Stage 1: Spatial Analysis](#stage-1-spatial-analysis)
  - [Stage 2: Rhythmic Distribution](#stage-2-rhythmic-distribution)
  - [Stage 3: Hierarchical Layout Organization](#stage-3-hierarchical-layout-organization)
  - [Stage 4: Visual Generation](#stage-4-visual-generation)
- [Plugin Integration Architecture](#plugin-integration-architecture)
  - [Spacing Hooks](#spacing-hooks)
  - [Paint Hooks](#paint-hooks)
- [Client-Server Coordination](#client-server-coordination)
  - [Lazy Rendering Strategy](#lazy-rendering-strategy)
  - [Hierarchical Invalidation](#hierarchical-invalidation)
- [Claypoole Integration Architecture](#claypoole-integration-architecture)
  - [Dependencies](#dependencies)
  - [Parallel Processing Approach Selection](#parallel-processing-approach-selection)
  - [Core Architecture Principles](#core-architecture-principles)
  - [Resource Lifecycle Management](#resource-lifecycle-management)
  - [Threadpool Component Integration](#threadpool-component-integration)
  - [Cross-Thread Context Management](#cross-thread-context-management)
  - [STM Integration with Cooperative Cancellation](#stm-integration-with-cooperative-cancellation)
  - [Four-Stage Pipeline Implementation](#four-stage-pipeline-implementation)
  - [Discomfort-Driven Iterative Optimization](#discomfort-driven-iterative-optimization)
  - [Core Discomfort Algorithm](#core-discomfort-algorithm)
  - [Discomfort Optimization Convergence Process](#discomfort-optimization-convergence-process)
  - [Plugin-Driven Optimization Strategies](#plugin-driven-optimization-strategies)
  - [Outward Rippling and Constraint Boundaries](#outward-rippling-and-constraint-boundaries)
  - [Integration with Pipeline Processing](#integration-with-pipeline-processing)
  - [Architectural Benefits Summary](#architectural-benefits-summary)
  - [Performance Characteristics](#performance-characteristics)
- [Caching and Incremental Processing](#caching-and-incremental-processing)
- [Benefits](#benefits)
- [Trade-offs](#trade-offs)
- [References](#references)

```mermaid
flowchart TD
    A[Musical Content Changes] --> B[Stage 1: Musical Logic & Spatial Analysis]
    B --> C[Stage 2: Rhythmic Proportional Distribution]
    C --> D[Stage 3: System & Page Breaking]
    D --> E[Stage 4: Visual Element Generation]
    E --> F[Client Invalidation Events]
    
    B -.-> B1[Parallel Processing<br/>Per Measure]
    D -.-> D1[Parallel Processing<br/>Per System/Page]
    
    B --> Cache1[(Stage 1 Cache)]
    C --> Cache2[(Stage 2 Cache)]
    D --> Cache3[(Stage 3 Cache)]
    E --> Cache4[(Stage 4 Cache)]
    
    Cache1 -.-> B
    Cache2 -.-> C
    Cache3 -.-> D
    Cache4 -.-> E
    
    style B fill:#b3e5fc,color:#000
    style D fill:#c8e6c8,color:#000
    style Cache1 fill:#ffe0b2,color:#000
    style Cache2 fill:#ffe0b2,color:#000
    style Cache3 fill:#ffe0b2,color:#000
    style Cache4 fill:#ffe0b2,color:#000
```

### Pipeline Stage Overview

```mermaid
flowchart LR
    A[Measure<br/>Changes] --> B[Stage 1<br/>Spatial Analysis]
    B --> C[Stage 2<br/>Rhythmic Distribution]
    C --> D[Stage 3<br/>System Breaking]
    D --> E[Stage 4<br/>Visual Generation]
    
    B1[Width Requirements<br/>Height Indicators<br/>Collision Bounds] --> B
    B --> B2[Minimum Spacing<br/>Element Positions]
    
    C1[Rhythmic Proportions<br/>Cross-Staff Sync] --> C
    C --> C2[Final Coordinates<br/>Absolute Positions]
    
    D1[System Layout<br/>Page Breaking] --> D
    D --> D2[System Boundaries<br/>Page Structure]
    
    E1[Coordinate Finalisation<br/>Visual Assembly] --> E
    E --> E2[Rendering Instructions<br/>Paint Lists]
    
    style B fill:#bbdefb,color:#000
    style C fill:#c5e1a5,color:#000
    style D fill:#f8bbd9,color:#000
    style E fill:#d1c4e9,color:#000
```

### Pipeline Stage 1: Musical Logic Resolution and Spatial Analysis
Individual measures process their internal musical content independently:
- Note transpositions and pitch calculations
- Effective accidental determination based on key signatures and measure context
- Tie, slur, and beam resolution
- Key signature and time signature propagation
- **Engraving Atom Formation**: Each rhythmic position calculates its complete spatial unit - noteheads, stems, accidentals, articulations positioned relative to each other as an indivisible "engraving atom"
- **Plugin Hook Integration**: Spacing hooks fire for each notational element, contributing spatial requirements to atom formation
- **Atom Dimension Caching**: Once calculated for a rhythmic configuration, engraving atoms remain **immutable** until measure content changes, enabling efficient repositioning without recomputation

### Pipeline Stage 2: Rhythmic Proportional Distribution
Measures coordinate optimal spacing using pre-computed engraving atoms:
- **Atom Repositioning**: Pre-computed engraving atoms moved horizontally for optimal rhythmic alignment - no internal atom recalculation required
- **Rhythmic proportion calculation**: Distributes space according to rhythmic relationships - longer note values receive proportionally more horizontal space
- **Cross-staff synchronisation**: Ensures consistent rhythmic spacing across all staves within each measure number
- **Computational Efficiency**: Much faster than Stage 1 since atom dimensions are cached - only atom positioning changes

### Pipeline Stage 3: Hierarchical Layout Organization
Measure streams are organized into visual layouts with full hierarchical cascade awareness:
- **Stage 3a - System Breaking**: Groups measures into horizontal systems based on available width and discomfort optimization
- **Stage 3b - Page Breaking** (conditional): Arranges systems vertically within page boundaries when system changes affect pagination
- **Stage 3c - Layout Restructuring** (conditional): Full layout reorganization when page changes require adding/removing pages or cross-movement adjustments
- **Cross-hierarchy element coordination**: Manages ties, slurs, and other elements spanning system and page breaks
- **Hierarchical cascade detection**: Automatically determines when system changes require page recalculation, and when page changes require full layout restructuring

### Pipeline Stage 4: Visual Element Generation
With final positioning established, measures generate their visual output:
- **Coordinate finalisation**: Combines positioning data with spatial measurements from earlier stages
- **Visual element assembly**: Creates rendering instructions for glyphs, curves, and text elements
- **Layout-specific adjustments**: Applies zoom levels, display preferences, and viewport optimisations

### Plugin Architecture Integration

Every notational element participates through a unified plugin interface with **mandatory two-stage compliance**:

```mermaid
flowchart LR
    A[Musical Element] --> B[Plugin Registry]
    B --> C[Spacing Hook]
    B --> D[Paint Hook]
    
    C --> C1[Input: Element + Context]
    C1 --> C2[Output: Width, Height, Bounds]
    
    D --> D1[Input: Element + Coordinates]
    D1 --> D2[Output: Rendering Instructions]
    
    C2 --> E[Stage 1]
    D2 --> F[Stage 4]
    
    style C fill:#bbdefb,color:#000
    style D fill:#f8bbd9,color:#000
    style E fill:#c5e1a5,color:#000
    style F fill:#d1c4e9,color:#000
```

#### Spacing Hook
Plugins declare the spatial requirements of their elements:
- **Input**: Musical element data and contextual information
- **Output**: Width requirements, indicative height, and collision boundaries

#### Paint Hook  
Plugins generate visual rendering instructions:
- **Input**: Musical element data with finalised positioning coordinates
- **Output**: Rendering instructions (glyphs, curves, text) with precise coordinates

**Registry-Based Discovery**: The plugin registry maintains mappings between musical elements and their formatters, enabling runtime composition and replacement of notational elements.

## User Interaction Flow

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant B as Backend
    participant P as Pipeline
    participant C1 as Client 1
    participant C2 as Client 2

    Note over U,C2: Phase 1: User Interaction & Frontend Processing
    U->>F: Click on staff location
    F->>F: Hit-testing & VPD resolution
    F->>B: add-note(measure-vpd, piece-id, note-data)

    Note over U,C2: Phase 2: Backend STM Transaction (36μs roundtrip)
    B->>B: STM transaction begins
    B->>B: Musical data updated (types preserved)
    B->>B: STM transaction commits
    B->>F: API response confirmation

    Note over U,C2: Phase 3: Asynchronous Pipeline Processing (100ms batch)
    B->>P: Queue affected elements for raster
    P->>P: Stage 1: Engraving atom formation<br/>(plugin spacing hooks fire)
    P->>P: Stage 2: Atom repositioning<br/>(cached atoms moved horizontally)
    P->>P: Stage 3: System/page breaking<br/>(if measure width changed)
    P->>P: Stage 4: Visual element generation<br/>(plugin paint hooks fire)
    P->>B: Updated MeasureView{glyphs, curves}

    Note over U,C2: Phase 4: Event Broadcasting & Cache Invalidation
    B->>F: Event: measures-invalidated [127]
    B->>C1: Event: measures-invalidated [127]
    B->>C2: Event: measures-invalidated [127]

    F->>F: Mark measure 127 cache dirty
    C1->>C1: Mark measure 127 cache dirty
    C2->>C2: Mark measure 127 cache dirty

    Note over U,C2: Phase 5: Lazy Visual Refresh (viewport-aware)
    alt Measure 127 visible in viewport
        F->>B: Request fresh MeasureView(127)
        B->>F: Updated paintlist data
        F->>F: Update local cache
        F->>F: Skija rendering: paintlist → pixels
        F->>U: Visual refresh (note appears)
    else Measure 127 not visible
        F->>F: Keep cache dirty, defer until navigation
    end

    alt Client 1 viewing measure 127
        C1->>B: Request fresh MeasureView(127)
        B->>C1: Same updated paintlist (shared computation)
        C1->>C1: Render updated view
    end

    alt Client 2 not viewing measure 127
        Note over C2: Cache stays dirty until user navigates there
    end
```

This sequence illustrates the complete system flow when a user adds a note, demonstrating the pipeline's lazy evaluation characteristics, asynchronous batching, collaborative event distribution, and viewport-aware client responses.

## Computational Scaling Characteristics

Independent measure analysis can run in parallel during Stage 1. System chunks can also be parallelised during Stage 3 processing. Pipeline stages naturally separate concerns, enabling different parallelisation strategies for each computational phase.

**Engraving Atom Efficiency**: The two-phase spatial computation (atom formation + positioning) provides intelligent caching granularity:
- Adding notes to existing rhythmic positions → atom recalculation + positioning
- Changing note spacing → positioning only (atoms unchanged)
- Adding accidentals → atom recalculation + positioning
- Changing time signatures → positioning only (atoms unchanged)

Memory access patterns prove favourable as each processing unit operates on distinct memory regions during parallel operations, minimising cache conflicts and false sharing between processors.

The 100-millisecond **asynchronous** batching interval amplifies computational efficiency by amortising processing costs across multiple edits, accumulating changes and processing them in optimally-sized batches. Crucially, mutating API calls return immediately without waiting for batch processing.

## Claypoole Integration Architecture

After comprehensive evaluation of parallel processing options, **Claypoole** has been selected as the optimal solution for Ooloi's CPU-intensive layout calculations, providing 16x performance improvement over pmap with superior resource control.

### Dependencies

```clojure
;; Add to backend/project.clj
[org.clj-commons/claypoole "1.2.2"]
```

### Parallel Processing Approach Selection

**Claypoole Selection Rationale:**

After comprehensive evaluation of parallel processing options, Claypoole provides the optimal combination of performance, control, and integration characteristics for musical notation rendering:

**Performance Advantages:**
- **16x performance improvement** over built-in `pmap` for CPU-intensive layout calculations
- **Near-linear scaling** up to available CPU cores (3.5-3.8x speedup on 4-core systems)
- **Superior CPU utilization** through configurable thread pools vs pmap's fixed threading

**Compared to Alternatives:**
- **`pmap`**: Unpredictable performance due to lazy evaluation and fixed `(ncpus + 2)` threading - avoided for production
- **`reducers/fold`**: Better for pure mathematical operations but limited by associative function requirements - doesn't fit measure-level processing
- **`core.async` pipelines**: Higher overhead with fixed 8-thread default, suboptimal for CPU-bound work
- **Custom `ForkJoinPool`**: Maximum performance potential but requires complex implementation and lacks Clojure integration

**`cp/pmap` Selection within Claypoole:**

Within Claypoole's API, `cp/pmap` provides the best fit for musical notation processing:

**Ordered Processing Requirement:**
- Musical measures must maintain sequence relationships for cross-measure elements (ties, slurs, spacing)
- `cp/pmap` preserves input order, unlike `cp/upmap` which returns results unordered
- Stage-to-stage data flow requires predictable result positioning

**API Simplicity:**
- `cp/pmap` provides direct replacement for `pmap` with controllable threadpool
- Simpler than manual `cp/future` coordination for batch processing
- More natural than `cp/pfor` for functional processing patterns

**Resource Control:**
- Explicit threadpool specification enables priority scheduling based on musical structure dependencies
- Controlled memory overhead through threadpool sizing
- Clean integration with Integrant component lifecycle

### Core Architecture Principles

The Claypoole integration follows a strict separation of concerns that prevents architectural conflation:

* **Claypoole handles execution substrate**: threadpool management, bounded parallelism, resource lifecycle, and backpressure control
* **Domain scheduler handles musical coordination**: operation tracking, STM transactions, and cooperative cancellation
* **Clean separation prevents complexity explosion**: execution primitives never mix with musical business logic

This separation is critical because musical notation rendering has sophisticated coordination requirements (operation-based cancellation, STM transaction management, hierarchical invalidation) that generic parallel processing libraries cannot and should not address. Claypoole provides the execution foundation while Ooloi's domain logic maintains complete control over coordination semantics.

### Resource Lifecycle Management

Musical notation software requires deterministic resource management because layout calculations run continuously in response to user edits. Unlike batch processing scenarios, notation editors must manage threadpools as long-lived resources with predictable cleanup behavior.

### Threadpool Component Integration

```clojure
(defmethod ig/init-key ::renderer [_ {:keys [cpu-cores]}]
  (let [cores (or cpu-cores (.availableProcessors (Runtime/getRuntime)))]
    {:cpu-pool (cp/threadpool cores)
     :priority-pool (cp/priority-threadpool cores)  ; For visible measures
     :metrics (atom {})}))

(defmethod ig/halt-key! ::renderer [_ renderer]
  (when-let [cpu (:cpu-pool renderer)]
    (cp/shutdown cpu))
  (when-let [prio (:priority-pool renderer)]
    (cp/shutdown prio)))
```

**Integrant Integration**: The component system manages threadpool lifecycle deterministically. Manual resource management is essential because threadpools maintain daemon threads that prevent JVM shutdown if not explicitly cleaned up.

**Dual Pool Strategy**: Standard and priority threadpools enable responsive editing behavior. Priority pools process high-priority measures first, ensuring critical musical structure updates receive immediate attention during complex score processing.

**Thread Count Configuration**: Pool sizing defaults to physical CPU cores, optimizing for CPU-bound layout calculations. This prevents thread oversubscription and context switching overhead.

### Cross-Thread Context Management

Claypoole executes tasks on threadpool threads that don't inherit dynamic variable bindings from the calling thread. Musical notation rendering requires consistent context (operation tracking, musical settings) across all parallel tasks.

**Binding Conveyance with `cp/pmap`**: Claypoole automatically handles binding conveyance for dynamic variables, ensuring that `*current-operation*` is available to all parallel tasks without explicit capture.

**Exception Context Preservation**: Task failures need to maintain debugging information across thread boundaries. This is critical for diagnosing layout calculation failures in complex musical scenarios:

```clojure
(cp/pmap cpu-pool
  (fn [measure-id]
    (try
      (with-cancellation
        (analyze-measure piece measure-id))
      (catch Exception e
        (throw (ex-info "Layout calculation failed"
                        {:measure-id measure-id
                         :operation-id *current-operation*}
                        e)))))
  measure-ids)
```

### STM Integration with Cooperative Cancellation

The rendering pipeline coordinates Claypoole parallel processing within STM transactions. Since only one formatting operation runs at a time, cancellation uses simple global state coordination.

### Single-Operation Architecture

**Key Constraint**: Only one formatting operation runs at any given time, eliminating complex concurrency coordination requirements.

**Cancellation Strategy**: Global operation tracking with dynamic variable binding enables responsive cancellation of parallel tasks within STM transactions.

```clojure
;; Global formatting coordination
(defonce ^:private current-formatting-operation (atom nil))

(def ^:dynamic *current-operation* nil)

(defn cancelled? []
  "Check if current parallel task should abort"
  (not= *current-operation* @current-formatting-operation))

(defmacro with-cancellation [& body]
  "Execute body with cancellation checking"
  `(if (cancelled?) ::cancelled (do ~@body)))
```

### Integrated STM and Claypoole Implementation

```clojure
(defn run-formatting-pipeline! [renderer piece-ref measure-ids]
  "Complete formatting pipeline with Claypoole and STM integration"
  (let [operation-id (java.util.UUID/randomUUID)
        {:keys [cpu-pool]} renderer]

    ;; Set current operation (outside STM)
    (reset! current-formatting-operation operation-id)

    ;; Execute pipeline within STM transaction
    (dosync
      (alter piece-ref
        (fn [piece]
          (binding [*current-operation* operation-id]
            ;; Stage 1: Parallel spatial analysis
            (let [stage1-results (cp/pmap cpu-pool
                                          (fn [measure-id]
                                            (with-cancellation
                                              (analyze-measure piece measure-id)))
                                          measure-ids)]

              (if (some #{::cancelled} stage1-results)
                piece  ; Return unchanged if cancelled
                ;; Continue with remaining stages
                (-> piece
                    (apply-rhythmic-distribution stage1-results)
                    (apply-system-breaking)
                    (generate-visual-elements)
                    (generate-invalidation-events))))))))))
```

### Cancellation Flow

**Responsive Editing**: When user makes new edits during active formatting:

1. **New operation starts**: `(reset! current-formatting-operation new-id)`
2. **Old tasks detect cancellation**: `(cancelled?)` returns `true`
3. **Tasks abort cleanly**: Return `::cancelled` without corrupting state
4. **STM handles consistency**: Transaction proceeds with partial results or unchanged piece

**Benefits**:
- **Simple cancellation**: Global operation tracking enables clean task abortion
- **STM compatibility**: No side effects within transactions
- **Responsive editing**: Long-running calculations abort quickly when new edits arrive
- **Clean state**: Cancelled operations leave no partial modifications

### Four-Stage Pipeline Implementation

Each pipeline stage uses Claypoole for parallel processing within the STM transaction:

```clojure
(defn run-stage-1-spatial-analysis! [cpu-pool piece measure-ids]
  "Stage 1: Measure-level spatial analysis with ideal width caching.

  GOAL: Establish the foundation for all subsequent stages by computing and caching
  each measure's ideal width and spatial requirements.

  APPROACH:
  - Analyze each measure independently in parallel
  - Compute engraving atoms (noteheads, stems, accidentals as spatial units)
  - Calculate ideal width based on musical content density
  - Cache ideal widths for fast discomfort evaluation in later stages
  - Establish collision boundaries and height indicators

  HIERARCHICAL DISCOMFORT TARGET: Measure-stack level optimization
  - Each measure gets its preferred width based on musical content
  - Creates foundation for 0.0 measure-stack discomfort when achieved

  INPUT: Musical piece data and list of measure IDs to process
  OUTPUT: Spatial analysis results with cached ideal widths per measure

  PERFORMANCE: Fully parallelizable across available CPU cores - measures are independent"
  (cp/pmap cpu-pool
           (fn [measure-id]
             (with-cancellation
               (let [spatial-data (analyze-spatial-requirements piece measure-id)
                     ideal-width (calculate-ideal-measure-width spatial-data)]
                 ;; Cache ideal width for fast discomfort evaluation
                 (cache-ideal-width! piece measure-id ideal-width)
                 spatial-data)))
           measure-ids))

(defn run-stage-2-rhythmic-distribution! [cpu-pool piece measure-ids spatial-results]
  "Stage 2: Rhythmic proportional distribution with width optimization.

  GOAL: Distribute horizontal space according to rhythmic relationships while
  respecting cached ideal widths from Stage 1.

  APPROACH:
  - Use cached ideal widths as optimization targets
  - Apply rhythmic proportion rules (longer notes get more space)
  - Balance individual measure preferences with system-wide consistency
  - Maintain cross-staff synchronization within measure numbers

  HIERARCHICAL DISCOMFORT TARGET: System-level optimization
  - Minimize combined discomfort of component measures within systems
  - Balance individual measure ideal widths with system spacing requirements
  - Prefer solutions that keep measures close to their cached ideal widths

  INPUT: Spatial analysis results with cached ideal widths
  OUTPUT: Rhythmic distribution with width assignments per measure

  PERFORMANCE: Parallelizable per measure, with cross-measure synchronization"
  (cp/pmap cpu-pool
           (fn [[measure-id spatial-data]]
             (with-cancellation
               (let [cached-ideal (get-cached-ideal-width piece measure-id)]
                 (calculate-rhythmic-distribution piece measure-id spatial-data cached-ideal))))
           (map vector measure-ids spatial-results)))

(defn run-stage-2-iteration! [cpu-pool piece measure-ids current-rhythmic system-results]
  "Stage 2 Iteration: Fast rhythmic redistribution using cached ideal widths.

  GOAL: Quickly adjust rhythmic distribution when measures move between systems
  during Stage 3 optimization, using cached ideal widths as adjustment targets.

  APPROACH:
  - Identify measures that moved to different systems
  - For moved measures: recalculate distribution toward cached ideal width
  - For unchanged measures: preserve existing rhythmic distribution
  - Use cached ideals to minimize computation time during iteration

  HIERARCHICAL DISCOMFORT TARGET: Minimize system-level discomfort
  - Moved measures adapt to new system context while targeting ideal widths
  - Unchanged measures maintain their optimized distribution

  CONVERGENCE STRATEGY:
  - Fast iteration enabled by cached ideal width targets
  - Each measure knows its preferred width, simplifying adjustment calculations
  - Converges when measures achieve acceptable proximity to cached ideals

  INPUT: Current rhythmic distribution, system results showing measure movements
  OUTPUT: Updated rhythmic distribution with moved measures readjusted

  PERFORMANCE: Only recalculates affected measures, uses cached values for speed"
  (let [affected-systems (find-systems-with-moved-measures current-rhythmic system-results)
        affected-measures (get-all-measures-in-systems affected-systems)]

    (cp/pmap cpu-pool
             (fn [measure-id]
               (if (contains? affected-measures measure-id)
                 (with-cancellation
                   (let [cached-ideal (get-cached-ideal-width piece measure-id)]
                     (adjust-rhythmic-distribution-toward-ideal piece measure-id system-results cached-ideal)))
                 ;; Keep existing result for unchanged measures
                 (get-rhythmic-result current-rhythmic measure-id)))
             measure-ids)))

(defn check-stage-3-convergence [current-rhythmic system-results piece-ref previous-discomfort discomfort-history]
  "Check if Stage 3 system-level optimization should continue or converge.

  CONVERGENCE ANALYSIS: Uses hierarchical discomfort model to determine when
  system-level optimization has achieved acceptable quality.

  CONVERGENCE CONDITIONS:
  1. TOLERANCE THRESHOLD: Multiplicative discomfort change falls below tolerance
  2. LOCAL MINIMUM: Current discomfort equals minimum in recent iteration history
  3. PLATEAU: Discomfort no longer improves between consecutive iterations

  HIERARCHICAL DISCOMFORT EVALUATION:
  - Measures current total discomfort (multiplicative across all levels)
  - System-level focus: combined discomfort of component measures + spacing
  - Tracks history to detect optimization patterns and stopping points

  CONVERGENCE REASONING:
  - Returns specific reason for convergence decision
  - Enables different handling based on why optimization stopped
  - Supports debugging and performance analysis

  INPUT: Current state, previous discomfort, iteration history
  OUTPUT: Convergence decision with detailed reasoning

  PERFORMANCE: Fast evaluation using cached ideal widths for discomfort calculation"
  (let [measures-moved? (measures-changed-systems? current-rhythmic system-results)
        ;; Use hierarchical multiplicative discomfort calculation
        current-discomfort (calculate-total-discomfort @piece-ref)

        ;; Check if discomfort change is below tolerance threshold
        discomfort-converged? (and previous-discomfort
                                  (<= (Math/abs (- current-discomfort previous-discomfort))
                                     (get-convergence-tolerance @piece-ref)))

        ;; Check if we've reached a local minimum (no improvement in recent iterations)
        local-minimum-reached? (and (>= (count discomfort-history) 3)
                                   (let [recent-history (take-last 3 discomfort-history)]
                                     (= current-discomfort (apply min recent-history))))

        ;; Check if discomfort has stopped improving (plateau detection)
        plateau-reached? (and (>= (count discomfort-history) 2)
                             (<= current-discomfort previous-discomfort))

        ;; Convergence achieved when optimization quality conditions are met
        converged? (or discomfort-converged? local-minimum-reached? plateau-reached?)

        updated-history (conj (vec (take-last 4 discomfort-history)) current-discomfort)]

    {:convergence-needed? (and measures-moved? (not converged?))
     :current-discomfort current-discomfort
     :discomfort-history updated-history
     :convergence-reason (cond
                          discomfort-converged? :tolerance-threshold
                          local-minimum-reached? :local-minimum
                          plateau-reached? :plateau
                          :else :continuing)}))

(defn run-stage-3-system-breaking! [cpu-pool renderer piece-ref operation-id rhythmic-results]
  "Stage 3: System-level optimization with iterative measure movement convergence.

  GOAL: Optimize system-level layout by balancing individual measure preferences
  with system-wide spacing requirements and measure movement opportunities.

  APPROACH:
  - Iterative optimization with Stage 3 ↔ Stage 2 feedback loop
  - Use cached ideal widths for fast discomfort evaluation
  - Move measures between systems when beneficial for overall layout
  - Recalculate rhythmic distribution when measures move (Stage 2 iteration)
  - Continue until convergence through quality-based stopping conditions

  HIERARCHICAL DISCOMFORT TARGET: System-level optimization
  - Minimize combined discomfort of component measures within each system
  - Balance measure-stack ideals with system spacing and organization
  - Consider measure movement opportunities for global discomfort reduction

  CONVERGENCE STRATEGY:
  - Uses multiplicative hierarchical discomfort for optimization quality assessment
  - Stops at local minimum, plateau, or tolerance threshold - whichever comes first
  - No arbitrary iteration limits - driven by actual optimization quality

  MEASURE MOVEMENT HANDLING:
  - Detects when system optimization moves measures between systems
  - Triggers Stage 2 iteration to recalculate rhythmic distribution for moved measures
  - Uses cached ideal widths as targets for fast adjustment
  - Continues iteration until width requirements stabilize

  INPUT: Rhythmic distribution results from Stage 2
  OUTPUT: Optimized system organization with stable measure width assignments

  PERFORMANCE: Cached widths enable fast iteration, parallel system processing"
  (loop [current-rhythmic-results rhythmic-results
         iteration 0
         previous-discomfort nil
         discomfort-history []]

    (let [;; Use cached ideal widths for fast evaluation
          system-results (cp/pmap cpu-pool
                                  (fn [system-data]
                                    (with-cancellation
                                      (optimize-system-with-cached-widths!
                                        renderer piece-ref system-data operation-id)))
                                  (group-into-systems current-rhythmic-results))]

      (if (some #{::cancelled} system-results)
        {:result ::cancelled}

        (let [{:keys [convergence-needed? current-discomfort discomfort-history convergence-reason]}
              (check-stage-3-convergence current-rhythmic-results system-results piece-ref
                                       previous-discomfort discomfort-history)]

          (if convergence-needed?
            ;; Continue iteration - recalculate rhythmic distribution using cached widths
            (let [updated-rhythmic-results
                  (run-stage-2-iteration-fast! cpu-pool @piece-ref (keys current-rhythmic-results)
                                              current-rhythmic-results system-results)]

              (if (some #{::cancelled} updated-rhythmic-results)
                {:result ::cancelled}
                (recur updated-rhythmic-results (inc iteration) current-discomfort discomfort-history)))

            ;; Good enough solution found - return results
            {:result :converged
             :system-results system-results
             :rhythmic-results current-rhythmic-results
             :convergence-reason convergence-reason
             :final-discomfort current-discomfort
             :iterations iteration}))))))

(defn run-stage-3-iterative-convergence! [cpu-pool renderer piece-ref operation-id rhythmic-results]
  "Stage 3 Backup: Iterative convergence for complex optimization scenarios.

  GOAL: Fallback optimization approach for large or complex solution spaces where
  the main Stage 3 approach may need additional iteration strategies.

  APPROACH:
  - Legacy iterative approach maintained for complex edge cases
  - Uses enhanced system breaking with full optimization capabilities
  - Maintains same convergence logic as main Stage 3 function
  - Provides compatibility layer for complex musical scenarios

  HIERARCHICAL DISCOMFORT TARGET: System-level optimization (same as main Stage 3)
  - Uses same multiplicative hierarchical discomfort calculation
  - Same convergence detection (local minimum, plateau, tolerance)

  USAGE: Automatically selected for scenarios where main approach insufficient
  - Complex cross-staff musical elements requiring specialized handling
  - Large solution spaces that exceed main approach capabilities
  - Edge cases in musical notation that need additional optimization flexibility

  INPUT: Rhythmic distribution results from Stage 2
  OUTPUT: Optimized system organization (same format as main Stage 3)

  PERFORMANCE: May be slower than main approach but handles complex edge cases"
  (loop [current-rhythmic-results rhythmic-results
         iteration 0
         previous-discomfort nil
         discomfort-history []]

    (let [system-results (cp/pmap cpu-pool
                                  (fn [system-data]
                                    (with-cancellation
                                      (enhanced-system-breaking-with-optimization!
                                        renderer piece-ref system-data operation-id)))
                                  (group-into-systems current-rhythmic-results))]

      (if (some #{::cancelled} system-results)
        {:result ::cancelled}

        (let [{:keys [convergence-needed? current-discomfort discomfort-history convergence-reason]}
              (check-stage-3-convergence current-rhythmic-results system-results piece-ref
                                       previous-discomfort discomfort-history)]

          (if convergence-needed?
            ;; Continue iteration - recalculate rhythmic distribution
            (let [updated-rhythmic-results
                  (run-stage-2-iteration! cpu-pool @piece-ref (keys current-rhythmic-results)
                                        current-rhythmic-results system-results)]

              (if (some #{::cancelled} updated-rhythmic-results)
                {:result ::cancelled}
                (recur updated-rhythmic-results (inc iteration) current-discomfort discomfort-history)))

            ;; Convergence achieved - return final system results
            {:result :converged
             :system-results system-results
             :rhythmic-results current-rhythmic-results
             :convergence-reason convergence-reason
             :convergence-method :iterative
             :final-discomfort current-discomfort
             :iterations iteration})))))))

(defn run-stage-3-hierarchical-layout! [cpu-pool piece system-results]
  "Stage 3b & 3c: Page and layout-level hierarchical optimization.

  GOAL: Handle hierarchical cascade from system-level changes up to page-level
  and layout-level restructuring when system optimization affects higher levels.

  APPROACH:
  - Stage 3b (Conditional): Page breaking when system changes affect pagination
  - Stage 3c (Conditional): Layout restructuring when page changes require adding/removing pages
  - Uses hierarchical discomfort evaluation to determine when each level is needed
  - Only executes expensive higher-level recalculation when actually required

  HIERARCHICAL DISCOMFORT TARGETS:
  - Page-level: Pages want to be appropriately filled (not too empty, not overcrowded)
  - Layout-level: Overall structure wants optimal page count (triggers page addition/removal)

  CASCADE LOGIC:
  - System changes → check if pagination affected (heights, system count per page)
  - If yes: recalculate page breaking to achieve desired page fullness
  - Page changes → check if layout restructuring needed (page count optimization)
  - If yes: add/remove pages to achieve optimal page count for content density

  CONDITIONAL EXECUTION:
  - Page breaking only when system changes actually affect pagination
  - Layout restructuring only when page changes require page count adjustment
  - Avoids expensive recalculation when system changes are purely local

  INPUT: Optimized system organization from Stage 3a
  OUTPUT: Complete hierarchical layout with optimized page and layout structure

  PERFORMANCE: Conditional execution minimizes work, parallel page processing when needed"
  (let [page-recalc-needed? (system-changes-affect-pagination? piece system-results)

        ;; Stage 3b: Page Breaking (conditional)
        page-results (if page-recalc-needed?
                       (cp/pmap cpu-pool
                               (fn [page-data]
                                 (with-cancellation
                                   (recalculate-page-layout piece page-data system-results)))
                               (group-into-pages system-results))
                       (preserve-existing-page-structure piece system-results))]

    (if (some #{::cancelled} page-results)
      {:result ::cancelled}

      (let [layout-recalc-needed? (page-changes-affect-full-layout? piece page-results)

            ;; Stage 3c: Full Layout Restructuring (conditional)
            layout-results (if layout-recalc-needed?
                            (with-cancellation
                              (restructure-full-layout piece page-results))
                            page-results)]

        (if (= layout-results ::cancelled)
          {:result ::cancelled}
          {:result :success
           :page-results page-results
           :layout-results layout-results})))))

(defn run-stage-4-visual-generation! [cpu-pool piece layout-results]
  "Stage 4: Visual element generation with finalized positioning.

  GOAL: Generate final visual output elements using completed layout positioning
  from all previous stages with established measure widths, system organization,
  and page/layout structure.

  APPROACH:
  - Use finalized positioning data from hierarchical layout optimization
  - Generate visual elements (noteheads, stems, beams, slurs, text) at precise coordinates
  - Combine cached spatial measurements with final layout positioning
  - Apply visual refinements (collision avoidance, aesthetic adjustments)

  HIERARCHICAL DISCOMFORT TARGET: Visual quality optimization
  - No layout changes at this stage - positioning is finalized
  - Focus on visual clarity, collision avoidance, and aesthetic refinement
  - Uses established layout structure to optimize visual presentation

  VISUAL GENERATION PROCESS:
  - Flatten all hierarchical layout results into measure-level visual specifications
  - Generate visual elements in parallel across all finalized measure layouts
  - Apply visual collision detection and aesthetic adjustments
  - Produce final renderable output for client display

  INPUT: Complete hierarchical layout results (measures, systems, pages, layout structure)
  OUTPUT: Final visual elements ready for client rendering

  PERFORMANCE: Fully parallelizable across measures - no interdependencies at visual stage"
  (cp/pmap cpu-pool
           (fn [measure-layout]
             (with-cancellation
               (generate-visual-elements-with-final-positioning piece measure-layout)))
           (flatten-all-layouts layout-results)))

(defn run-complete-pipeline! [renderer piece-ref measure-ids]
  "Complete four-stage hierarchical rendering pipeline with integrated optimization.

  ORCHESTRATION GOAL: Coordinate all four pipeline stages to transform musical
  data into optimized visual layout using hierarchical discomfort minimization.

  HIERARCHICAL PIPELINE STAGES:
  1. STAGE 1 (Measure-level): Spatial analysis with ideal width caching
  2. STAGE 2 (System-level): Rhythmic distribution targeting cached ideals
  3. STAGE 3 (System/Page/Layout): Hierarchical optimization with measure movement
  4. STAGE 4 (Visual): Final visual generation with optimized positioning

  HIERARCHICAL DISCOMFORT OPTIMIZATION:
  - Each stage targets specific levels of the discomfort hierarchy
  - Multiplicative discomfort calculation creates sophisticated level interactions
  - Cached ideal widths enable fast discomfort evaluation across stages
  - Quality-based convergence ensures optimization stops at appropriate quality level

  STM TRANSACTION ARCHITECTURE:
  - Single atomic transaction ensures consistency across all stages
  - No side effects within transaction (events sent afterward)
  - Cancellation supported at every stage for responsive editing
  - Either complete pipeline succeeds or piece remains unchanged

  PERFORMANCE CHARACTERISTICS:
  - Extensive parallelization using Claypoole across available CPU cores
  - Cached computations minimize redundant work during optimization
  - Conditional stage execution (page/layout recalc only when needed)
  - Modern hardware can achieve near-instantaneous response for typical editing

  INPUT: Musical piece reference and list of measure IDs requiring processing
  OUTPUT: Fully optimized visual layout ready for client rendering"
  (let [operation-id (java.util.UUID/randomUUID)
        {:keys [cpu-pool]} renderer]

    ;; Set current operation
    (reset! current-formatting-operation operation-id)

    ;; Execute pipeline within STM transaction (no side effects)
    (let [pipeline-result
          (dosync
            (alter piece-ref
              (fn [piece]
                (binding [*current-operation* operation-id]

                  ;; Stage 1: Spatial Analysis
                  (let [spatial-results (run-stage-1-spatial-analysis! cpu-pool piece measure-ids)]
                    (if (some #{::cancelled} spatial-results)
                      piece

                      ;; Stage 2: Rhythmic Distribution
                      (let [rhythmic-results (run-stage-2-rhythmic-distribution! cpu-pool piece measure-ids spatial-results)]
                        (if (some #{::cancelled} rhythmic-results)
                          piece

                          ;; Stage 3: System Breaking with Adaptive Optimization Strategy
                          (let [stage-3-result (run-stage-3-system-breaking! cpu-pool renderer piece-ref operation-id rhythmic-results)]
                            (case (:result stage-3-result)
                              ::cancelled piece

                              (:converged :optimal-found)
                              (let [{:keys [system-results rhythmic-results]} stage-3-result

                                    ;; Stage 3b & 3c: Hierarchical Layout Organization
                                    hierarchical-result (run-stage-3-hierarchical-layout! cpu-pool piece system-results)]

                                (case (:result hierarchical-result)
                                  ::cancelled piece

                                  :success
                                  (let [{:keys [page-results layout-results]} hierarchical-result

                                        ;; Stage 4: Visual Generation
                                        visual-results (run-stage-4-visual-generation! cpu-pool piece layout-results)]

                                    (if (some #{::cancelled} visual-results)
                                      piece
                                      ;; All stages completed successfully
                                      (apply-all-hierarchical-updates piece spatial-results rhythmic-results
                                                                     system-results page-results layout-results visual-results)))))))))))))]

      ;; Send invalidation events AFTER STM transaction completes (side effect)
      (when (not= pipeline-result ::cancelled)
        (send-invalidation-events! renderer measure-ids pipeline-result))

      pipeline-result)))

;; Stage 3 ↔ Stage 2 iteration functions
(defn measures-changed-systems? [previous-rhythmic-results current-system-results]
  "Detect if any measures moved to different systems during optimization"
  (not= (extract-measure-to-system-mapping previous-rhythmic-results)
        (extract-measure-to-system-mapping current-system-results)))

(defn find-systems-with-moved-measures [previous-rhythmic current-system]
  "Identify systems containing measures that moved during optimization"
  (let [prev-mapping (extract-measure-to-system-mapping previous-rhythmic)
        curr-mapping (extract-measure-to-system-mapping current-system)]
    (distinct (concat (vals prev-mapping) (vals curr-mapping)))))

(defn recalculate-rhythmic-distribution-for-new-system [piece measure-id system-results]
  "Recalculate rhythmic distribution when measure moves to different system"
  (let [new-system-context (find-system-containing-measure system-results measure-id)
        new-width-constraints (calculate-system-width-constraints new-system-context)]
    (calculate-rhythmic-distribution piece measure-id
                                   (get-spatial-data piece measure-id)
                                   new-width-constraints)))

(defn get-convergence-tolerance [piece]
  "Get discomfort convergence tolerance for iterative optimization"
  (or (get-piece-setting piece :stage-convergence-tolerance)
      1.0)) ; Default 1 unit of discomfort change

;; Hierarchical cascade decision functions
(defn system-changes-affect-pagination? [piece system-results]
  "Determine if system changes require page-level recalculation"
  (or (system-heights-changed-significantly? piece system-results)
      (system-count-per-page-changed? piece system-results)
      (cross-page-elements-affected? piece system-results)))

(defn page-changes-affect-full-layout? [piece page-results]
  "Determine if page changes require full layout restructuring"
  (or (page-count-changed? piece page-results)
      (page-scaling-changed? piece page-results)
      (cross-movement-boundaries-affected? piece page-results)
      (table-of-contents-affected? piece page-results)))

(defn apply-all-hierarchical-updates [piece spatial rhythmic system page layout visual]
  "Apply updates from all pipeline stages with proper hierarchical integration"
  (-> piece
      (integrate-spatial-analysis spatial)
      (integrate-rhythmic-distribution rhythmic)
      (integrate-system-layout system)
      (integrate-page-layout page)
      (integrate-layout-restructuring layout)
      (integrate-visual-elements visual)
      (validate-hierarchical-consistency)))
```

**Pipeline Characteristics:**
- **Single STM transaction**: All stages execute within one `dosync` block with no side effects
- **Iterative Stage 3 ↔ Stage 2 convergence**: When system optimization moves measures, rhythmic distribution recalculates for affected systems until width requirements stabilize
- **Hierarchical cascade handling**: System changes trigger page recalculation; page changes trigger full layout restructuring
- **Conditional stage execution**: Page breaking and layout restructuring execute only when system/page changes require them
- **Cached width optimization**: Pre-computed ideal widths enable fast discomfort evaluation and adjustment toward optimal values
- **Good enough threshold**: Optimization stops when discomfort falls below acceptable level rather than pursuing mathematical optimum
- **Optional parallel evaluation**: Available for small, critical sections when explicitly enabled, but not the default path
- **Quality-based convergence**: Local minimum detection, plateau detection, and tolerance threshold ensure intelligent stopping
- **Post-transaction events**: Invalidation events sent after STM transaction completes successfully
- **Cancellation at each stage**: Any stage can abort cleanly if new edits arrive, including during iterative convergence
- **Claypoole parallelism**: Each stage uses `cp/pmap` for parallel processing where applicable
- **Progressive computation**: Each stage builds on results from previous stages with iterative refinement awareness
- **Atomic updates**: Either entire hierarchical pipeline succeeds or piece remains unchanged

### Discomfort-Driven Iterative Optimization

Musical notation layout quality is maintained through an iterative discomfort minimization algorithm that automatically reorganizes measures, systems, and pages to achieve optimal spacing while respecting musical and user-imposed constraints.

### Core Discomfort Algorithm

The layout engine continuously measures and minimizes "discomfort" - the deviation of each layout element from its ideal proportions:

```clojure
(defn calculate-total-discomfort [piece]
  "Calculate system-wide layout discomfort across all hierarchical levels"
  ;; Multiply normalized discomfort values (0.0-1.0) for sophisticated interaction
  (* (normalize-measure-stack-discomfort piece)      ; Individual measures vs ideal widths
     (normalize-system-spacing-discomfort piece)     ; Systems vs optimal measure distribution
     (normalize-page-density-discomfort piece)       ; Pages vs desired fullness
     (normalize-layout-structure-discomfort piece))) ; Layout vs optimal page count

(defn measure-stack-discomfort [piece]
  "Sum of width deviations from ideal for all measure stacks"
  (->> (get-all-measure-stacks piece)
       (map (fn [stack]
              (Math/abs (- (actual-width stack)
                          (ideal-width stack)))))
       (reduce +)))

(defn system-spacing-discomfort [piece]
  "Combined discomfort of component measures within each system"
  (->> (get-all-systems piece)
       (map (fn [system]
              (let [measures-in-system (get-measures system)
                    ;; System discomfort = sum of its component measure discomforts
                    component-discomfort (reduce + (map measure-discomfort measures-in-system))
                    ;; Plus system-level spacing issues (overcrowding, too sparse)
                    spacing-discomfort (calculate-system-spacing-discomfort system)]
                (+ component-discomfort spacing-discomfort))))
       (reduce +)))

(defn page-density-discomfort [piece]
  "Page discomfort based on deviation from desired fullness"
  (->> (get-all-pages piece)
       (map (fn [page]
              (let [current-fullness (calculate-page-fullness page)
                    ;; Pages want to be filled - empty pages have high discomfort
                    ideal-fullness (get-ideal-page-fullness page)
                    fullness-gap (Math/abs (- ideal-fullness current-fullness))
                    ;; Weight heavily against underfilled pages in publication contexts
                    underfill-penalty (if (< current-fullness 0.7) (* 2 fullness-gap) fullness-gap)]
                underfill-penalty)))
       (reduce +)))

(defn layout-structure-discomfort [piece]
  "Layout-level discomfort about adding or removing pages"
  (let [current-page-count (get-page-count piece)
        content-density (calculate-overall-content-density piece)

        ;; Layout wants optimal page count for content amount
        optimal-page-count (calculate-optimal-page-count piece content-density)
        page-count-discomfort (Math/abs (- current-page-count optimal-page-count))

        ;; Heavy penalty for layouts requiring page addition/removal
        restructuring-penalty (cond
                               ;; Too many pages - content could be condensed
                               (> current-page-count optimal-page-count)
                               (* 10 page-count-discomfort)  ; Encourage page removal

                               ;; Too few pages - content is overcrowded
                               (< current-page-count optimal-page-count)
                               (* 15 page-count-discomfort)  ; Strongly encourage page addition

                               ;; Optimal page count
                               :else 0)]

    restructuring-penalty))

;; Normalized discomfort functions (all return 0.0-1.0)
(defn normalize-measure-stack-discomfort [piece]
  "Normalize measure-stack discomfort to 0.0-1.0 range"
  (let [raw-discomfort (measure-stack-discomfort piece)
        max-possible-discomfort (calculate-max-measure-discomfort piece)]
    (min 1.0 (/ raw-discomfort max-possible-discomfort))))

(defn normalize-system-spacing-discomfort [piece]
  "Normalize system spacing discomfort to 0.0-1.0 range"
  (let [raw-discomfort (system-spacing-discomfort piece)
        max-possible-discomfort (calculate-max-system-discomfort piece)]
    (min 1.0 (/ raw-discomfort max-possible-discomfort))))

(defn normalize-page-density-discomfort [piece]
  "Normalize page density discomfort to 0.0-1.0 range"
  (let [raw-discomfort (page-density-discomfort piece)]
    ;; Page discomfort naturally ranges 0.0-1.0 (fullness percentages)
    (min 1.0 raw-discomfort)))

(defn normalize-layout-structure-discomfort [piece]
  "Normalize layout structure discomfort to 0.0-1.0 range"
  (let [raw-penalty (layout-structure-discomfort piece)
        max-reasonable-penalty (* 15 (get-max-reasonable-page-variance piece))]
    (min 1.0 (/ raw-penalty max-reasonable-penalty))))

;; Hierarchical Discomfort Summary with Multiplicative Interaction
;; 1. Measure-stack level: Individual measures want their ideal widths (0.0-1.0)
;; 2. System level: Combined discomfort of component measures + system spacing (0.0-1.0)
;; 3. Page level: Pages want to be appropriately filled (0.0-1.0)
;; 4. Layout level: Overall structure wants optimal page count (0.0-1.0)
;;
;; Multiplication creates sophisticated interactions:
;; - Perfect measure widths (0.0) make total discomfort 0.0 regardless of other factors
;; - Any single factor at maximum discomfort (1.0) doesn't necessarily doom the layout
;; - Small improvements in multiple areas compound multiplicatively

(defn run-discomfort-optimization! [renderer piece-ref operation-id optimization-plugins]
  "Iterative discomfort minimization with plugin-driven strategies"
  (let [{:keys [cpu-pool]} renderer
        acceptable-discomfort (get-discomfort-threshold piece-ref)
        max-iterations (get-max-optimization-iterations piece-ref)]

    (loop [iteration 0
           current-discomfort (calculate-total-discomfort @piece-ref)
           affected-region (get-initially-affected-measures @piece-ref)]

      (cond
        ;; Success: Acceptable quality achieved
        (<= current-discomfort acceptable-discomfort)
        {:result :converged :iterations iteration :final-discomfort current-discomfort}

        ;; Hard stop: Maximum iterations reached
        (>= iteration max-iterations)
        {:result :max-iterations :iterations iteration :final-discomfort current-discomfort}

        ;; Continue: Try optimization strategies
        :else
        (let [optimization-result
              (dosync
                (alter piece-ref
                  (fn [piece]
                    (binding [*current-operation* operation-id]
                      ;; Try plugin-driven optimization strategies in parallel
                      (let [strategy-results
                            (cp/pmap cpu-pool
                                     (fn [plugin]
                                       (with-cancellation
                                         (apply-optimization-strategy plugin piece affected-region)))
                                     optimization-plugins)

                            ;; Select best strategy result (lowest discomfort)
                            best-strategy (apply min-key :resulting-discomfort strategy-results)]

                        (if (= (:resulting-discomfort best-strategy) ::cancelled)
                          piece  ; Return unchanged if cancelled
                          ;; Apply best optimization and expand affected region
                          (-> piece
                              (apply-layout-changes (:layout-changes best-strategy))
                              (track-expanded-affected-region (:expanded-region best-strategy))))))))

              new-discomfort (calculate-total-discomfort @piece-ref)
              expanded-region (get-current-affected-region @piece-ref)]

          ;; Check for hard constraint boundaries that stop expansion
          (if (reached-hard-constraint-boundary? @piece-ref expanded-region)
            {:result :constraint-boundary :iterations iteration :final-discomfort new-discomfort}
            ;; Continue optimization with expanded region
            (recur (inc iteration) new-discomfort expanded-region)))))))
```

### Discomfort Optimization Convergence Process

```mermaid
flowchart TD
    A[Initial Layout Change] --> B[Calculate Current Discomfort]
    B --> C{Discomfort ≤ Acceptable Threshold?}
    C -->|Yes| D[✅ Converged - Layout Complete]
    C -->|No| E[Apply Optimization Strategies in Parallel]

    E --> F[Strategy 1: Measure Reflow<br/>Redistribute measures across systems]
    E --> G[Strategy 2: System Rebreak<br/>Rebreak systems across pages]
    E --> H[Strategy 3: Vertical Spacing<br/>Adjust staff spacing]
    E --> I[Strategy 4: Page Margins<br/>Adjust margins and scaling]

    F --> J[Select Best Strategy Result<br/>Lowest resulting discomfort]
    G --> J
    H --> J
    I --> J

    J --> K[Apply Best Layout Changes]
    K --> L[Expand Affected Region]
    L --> M{Reached Hard Constraint Boundary?}

    M -->|Yes - Manual Page Break| N[🛑 Constraint Boundary Stop]
    M -->|Yes - Movement Boundary| N
    M -->|Yes - Performance Directive| N
    M -->|No| O{Max Iterations Reached?}

    O -->|Yes| P[⏱️ Max Iterations Stop]
    O -->|No| Q[Calculate New Discomfort]
    Q --> R[Increment Iteration Count]
    R --> C

    style A fill:#ffcdd2,color:#000
    style B fill:#fff3e0,color:#000
    style C fill:#e8f5e8,color:#000
    style D fill:#c8e6c9,color:#000
    style E fill:#e3f2fd,color:#000
    style J fill:#f3e5f5,color:#000
    style M fill:#fff8e1,color:#000
    style N fill:#ffcdd2,color:#000
    style P fill:#ffcdd2,color:#000
```

**Convergence Characteristics:**
- **Guaranteed Termination**: Process always stops through acceptable discomfort, constraint boundaries, or iteration limits
- **Progressive Improvement**: Each iteration reduces total layout discomfort through best-strategy selection
- **Automatic Scope Expansion**: Affected region grows organically until optimization boundaries are reached
- **Constraint Respect**: Hard boundaries (manual breaks, movement divisions) prevent inappropriate cross-boundary optimization
- **Plugin Extensibility**: Custom optimization strategies can be added for domain-specific layout intelligence

### Plugin-Driven Optimization Strategies

Optimization strategies are implemented as plugins, enabling domain-specific layout intelligence and user customization:

```clojure
(defprotocol OptimizationStrategy
  "Plugin interface for layout optimization strategies"
  (analyze-discomfort [strategy piece affected-region]
    "Analyze current discomfort and propose optimization approach")
  (apply-optimization [strategy piece region-analysis]
    "Apply optimization changes and return updated piece with expanded affected region"))

;; Built-in optimization strategies
(def default-optimization-strategies
  [(->MeasureReflowStrategy)      ; Redistribute measures across systems
   (->SystemRebreakStrategy)      ; Rebreak systems across pages
   (->VerticalSpacingStrategy)    ; Adjust staff spacing within systems
   (->PageMarginStrategy)])       ; Adjust page margins and scaling

(defn apply-optimization-strategy [strategy piece affected-region]
  "Execute a single optimization strategy"
  (let [analysis (analyze-discomfort strategy piece affected-region)
        result (apply-optimization strategy piece analysis)]
    {:layout-changes (:changes result)
     :expanded-region (:new-affected-region result)
     :resulting-discomfort (calculate-total-discomfort (:updated-piece result))}))

;; Fast width-based evaluation using cached ideal widths
(defn fast-system-breaking-with-cached-widths! [renderer piece-ref system-data operation-id]
  "Fast system breaking using pre-cached ideal widths for quick evaluation"
  (let [measures-in-system (get-measures-in-system system-data)
        ;; Get cached ideal widths - computed once during Stage 1
        cached-ideal-widths (map #(get-cached-ideal-width piece-ref %) measures-in-system)

        ;; Fast discomfort calculation using cached values
        current-widths (map #(get-current-width %) measures-in-system)
        discomfort-deltas (map - current-widths cached-ideal-widths)
        total-discomfort (reduce + (map #(Math/abs %) discomfort-deltas))]

    (if (<= total-discomfort (get-acceptable-discomfort-threshold piece-ref))
      ;; Good enough - no optimization needed
      system-data
      ;; Default: Try simple width adjustments toward ideal values
      (if (and (enable-parallel-evaluation? piece-ref)
               (small-solution-space? measures-in-system))
        ;; Optional: Parallel evaluation for small, critical sections
        (evaluate-alternatives-in-parallel system-data cached-ideal-widths)
        ;; Default: Fast iterative adjustment toward cached ideal widths
        (apply-width-adjustments-toward-ideal system-data cached-ideal-widths)))))

(defn evaluate-alternatives-in-parallel [system-data cached-ideal-widths]
  "Parallel evaluation of layout alternatives for critical sections"
  (let [cpu-pool (get-cpu-pool)
        ;; Generate reasonable alternatives for this specific system
        width-alternatives (generate-width-alternatives system-data cached-ideal-widths)
        measure-alternatives (generate-measure-grouping-alternatives system-data)

        ;; Evaluate all combinations in parallel
        evaluated-alternatives
        (cp/pmap cpu-pool
                 (fn [alternative]
                   (let [trial-layout (apply-alternative system-data alternative)
                         trial-discomfort (calculate-system-discomfort trial-layout)]
                     {:alternative alternative
                      :layout trial-layout
                      :discomfort trial-discomfort}))
                 (combine-alternatives width-alternatives measure-alternatives))]

    ;; Select minimum discomfort solution
    (:layout (apply min-key :discomfort evaluated-alternatives))))

(defn run-stage-2-iteration-fast! [cpu-pool piece measure-ids current-rhythmic system-results]
  "Fast Stage 2 iteration using cached ideal widths"
  (let [affected-systems (find-systems-with-moved-measures current-rhythmic system-results)
        affected-measures (get-all-measures-in-systems affected-systems)]

    (cp/pmap cpu-pool
             (fn [measure-id]
               (if (contains? affected-measures measure-id)
                 ;; Fast recalculation using cached ideal width as target
                 (with-cancellation
                   (adjust-rhythmic-distribution-toward-ideal piece measure-id system-results))
                 ;; Keep existing result
                 (get-rhythmic-result current-rhythmic measure-id)))
             measure-ids)))

;; Practical examples of when to use parallel evaluation
(defn enable-parallel-evaluation? [piece-ref]
  "Determine when parallel evaluation is beneficial"
  (or (title-page-formatting? piece-ref)
      (first-page-of-movement? piece-ref)
      (final-publication-mode? piece-ref)
      (user-requested-perfect-layout? piece-ref)))

(defn small-solution-space? [measures-in-system]
  "Check if solution space is small enough for parallel evaluation"
  (and (<= (count measures-in-system) 8)  ; Small number of measures
       (no-complex-cross-staff-elements? measures-in-system)))  ; No complex interactions
```

## Practical Parallel Evaluation Examples

### Example 1: Title Page Optimization
```clojure
;; When formatting the title page of a symphony
(defn format-title-page! [piece-ref]
  "Title pages justify parallel evaluation for perfect layout"
  (binding [*enable-parallel-evaluation* true]
    (run-stage-3-system-breaking! cpu-pool renderer piece-ref operation-id title-measures)))

;; Generates alternatives like:
;; - Different composer name positioning (left, center, right)
;; - Various title font sizes that fit available space
;; - Alternative dedication placement options
;; Evaluates all combinations in parallel, selects optimal layout
```

### Example 2: First System of Movement
```clojure
;; First system of each movement gets special treatment
(defn format-movement-opening! [piece-ref movement-number]
  "Movement openings benefit from perfect layout optimization"
  (let [first-system-measures (get-first-system-measures piece-ref movement-number)]
    (if (and (<= (count first-system-measures) 6)  ; Small enough for parallel eval
             (important-movement? movement-number))    ; Worth the extra computation
      ;; Parallel evaluation: try different measure groupings and spacing
      (evaluate-alternatives-in-parallel
        {:width-distributions [{:measures [1 2] :ratio 0.4}
                              {:measures [3 4 5] :ratio 0.6}]
         :tempo-marking-positions [:above-staff :between-staves :left-margin]
         :key-signature-spacing [:tight :normal :spacious]})
      ;; Default: fast cached-width adjustment
      (apply-width-adjustments-toward-ideal first-system-measures))))
```

### Example 3: Complex Orchestral Passage
```clojure
;; Dense orchestral writing with many cross-staff elements
(defn format-complex-orchestral-passage! [piece-ref measures]
  "Complex passages may benefit from parallel evaluation when user requests it"
  (when (user-enabled-perfect-mode?)
    (let [alternatives [{:beam-breaking :conservative :stem-direction :auto}
                       {:beam-breaking :aggressive :stem-direction :forced-up}
                       {:beam-breaking :minimal :stem-direction :forced-down}]]

      ;; Evaluate each approach in parallel
      (cp/pmap cpu-pool
               (fn [approach]
                 (let [trial-formatting (apply-formatting-approach measures approach)]
                   {:approach approach
                    :collision-count (count-collisions trial-formatting)
                    :readability-score (calculate-readability trial-formatting)}))
               alternatives))))
```

### Example 4: Publication-Quality Final Pass
```clojure
;; Final publication pass - parallel evaluation for entire score sections
(defn run-publication-quality-pass! [piece-ref]
  "Final pass uses parallel evaluation for publication-quality optimization"
  (doseq [page-number (range 1 (inc (get-page-count piece-ref)))]
    (let [page-measures (get-measures-on-page piece-ref page-number)]

      ;; For each page, evaluate different layout strategies in parallel
      (when (enable-publication-optimization?)
        (cp/pmap cpu-pool
                 (fn [layout-strategy]
                   (optimize-page-with-strategy page-measures layout-strategy))
                 [:compact :spacious :balanced :traditional])))))
```

### When NOT to Use Parallel Evaluation
```clojure
;; Normal editing scenarios - use fast cached approach
(defn format-during-editing! [piece-ref changed-measures]
  "Normal editing uses fast cached-width approach for immediate response"
  ;; These scenarios prioritize speed over perfection:
  ;; - User adding/deleting notes during composition
  ;; - Transposing passages
  ;; - Adding dynamics and articulations
  ;; - Adjusting rhythms

  (let [cached-ideals (map #(get-cached-ideal-width piece-ref %) changed-measures)]
    ;; Fast adjustment toward cached ideal widths
    (apply-width-adjustments-toward-ideal changed-measures cached-ideals)))
```

### Outward Rippling and Constraint Boundaries

Optimization automatically expands its scope until discomfort is minimized or hard constraints prevent further expansion:

**Ripple Expansion Pattern:**
1. **Local Adjustment**: Initially optimize only directly affected measures
2. **System Expansion**: If local changes create system-level discomfort, expand to full system optimization
3. **Page Expansion**: If system changes create page-level discomfort, expand to page-level rebreaking
4. **Cross-Page Expansion**: If necessary, optimize across multiple pages until discomfort is acceptable

**Hard Constraint Boundaries (Optimization Stops):**
- **Manual Page Breaks**: User-inserted page breaks cannot be crossed during optimization
- **Manual System Breaks**: User-inserted system breaks constrain measure redistribution
- **Movement Boundaries**: Optimization cannot move measures across movement or section boundaries
- **Performance Directives**: Tempo changes, rehearsal marks, and other performance indicators create layout boundaries

```clojure
(defn reached-hard-constraint-boundary? [piece affected-region]
  "Check if optimization has reached uncrossable layout boundaries"
  (or (contains-manual-page-breaks? piece affected-region)
      (spans-movement-boundaries? piece affected-region)
      (conflicts-with-performance-directives? piece affected-region)
      (exceeds-maximum-optimization-scope? piece affected-region)))

(defn get-constraint-boundaries [piece starting-region]
  "Identify hard boundaries that limit optimization scope expansion"
  {:manual-breaks (find-manual-breaks-near piece starting-region)
   :movement-boundaries (find-movement-boundaries piece starting-region)
   :performance-directives (find-performance-directives piece starting-region)
   :scope-limits (calculate-maximum-reasonable-scope piece starting-region)})
```

### Integration with Pipeline Processing

Discomfort optimization integrates with the four-stage pipeline, running after initial layout calculation to refine results:

**Stage 3 Enhancement**: System Breaking stage now includes iterative optimization:

```clojure
(defn enhanced-system-breaking-with-optimization! [renderer piece-ref system-data operation-id]
  "System breaking with integrated discomfort optimization"
  (binding [*current-operation* operation-id]
    ;; Initial system breaking (existing logic)
    (let [initial-systems (calculate-initial-system-breaks piece-ref system-data)
          initial-discomfort (calculate-total-discomfort @piece-ref)]

      ;; Apply discomfort optimization if threshold exceeded
      (if (> initial-discomfort (get-discomfort-threshold @piece-ref))
        (let [optimization-result
              (run-discomfort-optimization! renderer piece-ref operation-id
                                          (get-active-optimization-plugins @piece-ref))]
          ;; Return enhanced system breaking with optimization results
          (merge initial-systems optimization-result))
        ;; Return initial systems if discomfort acceptable
        {:systems initial-systems :optimization-result :not-needed}))))
```

**Optimization Plugin Registry:**

```clojure
(defn register-optimization-plugin! [plugin-registry strategy-impl]
  "Register custom optimization strategy"
  (swap! plugin-registry conj strategy-impl))

;; Enable user and domain-specific optimization strategies
(register-optimization-plugin! optimization-plugins (->CustomChoralSpacingStrategy))
(register-optimization-plugin! optimization-plugins (->UserPreferenceStrategy user-prefs))
```

**Algorithmic Characteristics:**
- **Convergent**: Always terminates through discomfort thresholds, iteration limits, or constraint boundaries
- **Plugin-Extensible**: Optimization strategies can be replaced or supplemented with domain-specific logic
- **Constraint-Aware**: Respects user layout intentions while optimizing within allowable boundaries
- **Parallel**: Multiple optimization strategies execute concurrently with best-result selection
- **Cancellable**: Integrates with cooperative cancellation for responsive editing during long optimizations

### Architectural Benefits Summary

The Claypoole integration with discomfort-driven optimization provides several critical advantages:

**Responsive Musical Processing**: Iterative optimization ensures that layout changes automatically achieve optimal musical spacing within user-defined constraints, maintaining professional engraving quality during complex score manipulation.

**Resource Predictability**: Explicit threadpool management provides deterministic resource usage patterns, essential for professional audio workstation integration where resource conflicts must be avoided.

**Simple Cancellation**: Global operation tracking with dynamic binding provides clean cancellation semantics without complex coordination machinery.

**STM Integration**: Parallel processing within STM transactions leverages automatic conflict resolution while maintaining transactional consistency across the entire pipeline.

**Computational Scalability**: Near-linear scaling with CPU cores provides predictable performance improvements on professional workstations with high core counts, enabling complex orchestral score processing.

**Architectural Simplicity**: Single-operation constraint eliminates complex concurrency coordination, enabling straightforward implementation and maintenance.

### Performance Characteristics

- **16x performance improvement** over pmap for CPU-intensive layout calculations
- **Near-linear scaling** up to available CPU cores (3.5-3.8x speedup on 4-core systems)
- **Controlled memory overhead**: Buffer size scales as `2 × pool_size` (eager) or `1 × pool_size` (lazy)
- **STM automatic conflict handling**: Transactions retry with fresh data when conflicts occur
- **Cooperative cancellation** with responsive abort behavior through operation tracking

## Caching and Incremental Processing

Pipeline results are cached until local edits change them. Invalidation works hierarchically: musical events trigger measure recalculation, which may trigger system recalculation, which may trigger page recalculation.

```mermaid
flowchart TD
    A[Musical Event Change] --> B[Measure Invalidation]
    B --> C{ALL Measures in<br/>System Invalidated?}
    C -->|Yes| D[System Invalidation<br/>Instead of Individual Measures]
    C -->|No| E[Individual Measure Events]
    D --> F{ALL Systems in<br/>Page Invalidated?}
    F -->|Yes| G[Page Invalidation<br/>Instead of Individual Systems]
    F -->|No| H[Individual System Events]
    G --> I[Single Page Event]
    
    E --> J[Client: Individual Measures]
    H --> K[Client: Individual Systems]
    I --> L[Client: Single Page]
    
    style A fill:#ffcdd2,color:#000
    style B fill:#ffe0b2,color:#000
    style D fill:#c8e6c8,color:#000
    style G fill:#bbdefb,color:#000
    style J fill:#e8f5e8,color:#000
    style K fill:#e8f5e8,color:#000
    style L fill:#e8f5e8,color:#000
```

**Stage 1 Caching**: Spatial analysis results persist until any element affecting measure content changes. Measures that remain completely unchanged never require recalculation.

**Conditional Processing**: Later pipeline stages process only measures that underwent earlier stage recalculation. Unchanged measures retain their cached results unless positioning changes affect their coordinates.

**Cache Granularity**: Invalidation operates at measure-level precision, ensuring that unrelated changes don't trigger unnecessary recalculations across the composition.

### Client-Server Event Coordination

#### Batched Reformatting (100ms Intervals)
The backend accumulates formatting requests and processes them in 100-millisecond intervals, providing perceived real-time response whilst avoiding computational waste on rapid input sequences.

#### Hierarchical Invalidation Events
Clients receive optimised invalidation notifications that respect the visual hierarchy. Deduplication logic eliminates redundant updates - if an entire page requires recalculation, individual measure invalidations within that page are automatically eliminated.

#### Lazy Visual Realisation
Clients implement demand-driven rendering data fetching. Open layouts immediately request updated rendering instructions, whilst closed layouts mark invalidated elements for cleanup and fetch data only when subsequently opened.

**Collaborative Data Sharing**: Multiple clients requesting identical updated data receive shared computation results without triggering redundant backend processing, as all clients typically fetch identical MeasureView paintlists.

## Rationale

### Positive Aspects

1. **Performance Scalability**: The four-stage pipeline enables parallelisation across different computational characteristics, with each stage optimised for its specific processing requirements.

2. **Plugin-Based Fundamental Architecture**: All notation formatting - from basic chord layouts and articulation placement to custom extensions - operates through the same unified plugin system, eliminating distinction between "core" and "extended" functionality.

3. **Intelligent Resource Management**: Client-server coordination minimises computational waste whilst maintaining responsive user experience through demand-driven visual realisation.

4. **Musical Accuracy**: Pipeline separation ensures that musical logic resolution occurs independently of spatial concerns, preventing layout constraints from corrupting musical meaning.

5. **Incremental Processing**: Comprehensive caching with hierarchical invalidation provides dramatic performance improvements for typical editing scenarios.

6. **Transparent Distribution**: Type fidelity across gRPC boundaries eliminates serialization impedance, enabling plugins and musical logic to operate identically in local or distributed modes.

7. **Efficient Spatial Computation**: Engraving atom caching prevents unnecessary recomputation - most changes only require fast atom repositioning rather than expensive dimension recalculation.

### Negative Aspects

1. **Implementation Complexity**: The four-stage pipeline requires sophisticated coordination mechanisms and careful state management across stage boundaries.

2. **Plugin Development Requirements**: All formatting logic must implement both spacing and paint hooks, creating consistent architectural requirements across canonical and custom plugins.

3. **Memory Usage During Transitions**: Intermediate state between stages requires temporary storage, particularly significant for large scores.

4. **Debugging Complexity**: Issues may arise from interactions between stages, requiring debugging tools that can trace problems across the entire pipeline.

5. **Network Dependency for Collaboration**: Real-time collaborative features require reliable network connectivity; offline scenarios may experience degraded functionality.

### Mitigations

1. **Comprehensive Testing Framework**: Automated testing covers cross-stage interactions and performance characteristics under realistic load conditions.

2. **Plugin Development Tools**: Scaffolding tools and examples reduce the complexity of implementing compliant plugins.

3. **Memory Monitoring**: Profiling tools track memory usage across stage transitions, enabling optimisation of large score handling.

4. **Debugging Infrastructure**: Specialised debugging tools provide visibility into stage transitions and plugin interactions.

5. **Offline Resilience**: Local formatting capabilities maintain basic functionality during network interruptions.

## Alternatives Considered

### Alternative 1: Hardcoded Core Notation Elements
**Approach**: Implement fundamental notation formatting (chord layouts, articulation placement, beam positioning) directly in core system with plugin architecture only for extensions.
**Rejection Reasons**:
- Creates artificial architectural distinction between fundamental and extended formatting logic
- Prevents unified optimization and caching strategies across all notation elements
- Violates architectural uniformity principle where all formatting logic follows identical patterns
- Complicates maintenance by requiring separate code paths for equivalent functionality
- Limits ability to customize or replace fundamental formatting behaviors when needed

## References

### Implementation Research
- **research/CLAYPOOLE_FOR_OOLOI_RESEARCH.md** - Comprehensive technical assessment and suitability analysis
- **research/LAYOUT_COMPUTATION_WITH_CLAYPOOLE.md** - Complete production-ready implementation guide

### Related ADRs
- [ADR-0000: Clojure](0000-Clojure.md) - Language providing parallel processing capabilities essential for pipeline stages
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural foundation enabling clean separation with transparent distribution
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Communication protocol supporting efficient invalidation event transmission with type fidelity
- [ADR-0003: Plugins](0003-Plugins.md) - Plugin architecture foundation extended by mandatory two-stage compliance
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model supporting coordinated multi-stage updates
- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) - Client rendering technology consuming generated rendering instructions
- [ADR-0006: SMuFL](0006-SMuFL.md) - Standardised music font providing glyph metadata for spatial calculations
- [ADR-0008: VPDs](0008-VPDs.md) - Vector Path Descriptors enabling hierarchical element addressing in invalidation events
- [ADR-0022: Lazy Frontend-Backend Architecture](0022-Lazy-Frontend-Backend-Architecture.md) - Event-driven client synchronization patterns underlying the rendering pipeline

### Technical Dependencies
- **Claypoole**: Threadpool-based parallel processing library providing controlled CPU-intensive task execution
- **Clojure STM**: Transaction system coordinating multi-stage updates
- **gRPC Streaming**: Bi-directional communication for real-time invalidation events
- **SMuFL Font Metadata**: Glyph dimension data essential for spatial requirement calculations
- **JavaFX Canvas**: Client rendering surface for visual element realisation
- **Skija Graphics**: High-performance graphics rendering for complex musical elements

### Performance Characteristics
- **Batching Interval**: 100ms asynchronous processing provides optimal balance between responsiveness and computational efficiency
- **API Responsiveness**: 36μs roundtrip measured on 2017 MacBook Pro proves gRPC separation adds negligible latency
- **Memory Efficiency**: Lazy rendering data generation reduces memory footprint for large scores
- **Parallel Scaling**: Linear performance improvement with available CPU cores during formatting stages
- **Engraving Atom Efficiency**: Two-phase spatial computation minimizes redundant calculations through intelligent caching

## Notes

This architectural decision establishes Ooloi's rendering pipeline as a scalable system capable of handling professional-scale musical compositions through unified plugin-based formatting for all notation elements.

The mandatory plugin compliance ensures that fundamental notation formatting (chords, articulations, beams) and custom extensions operate through identical architectural patterns, providing a foundation for comprehensive musical expression without artificial distinctions between "core" and "extended" functionality.

The four-stage approach recognises that musical logic, spatial arrangement, layout organisation, and visual rendering represent distinct computational problems that benefit from separation and targeted optimisation, with all formatting logic implemented through canonical and custom plugins as first-class citizens.
