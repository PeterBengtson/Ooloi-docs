# ADR-0037: Measure Distribution Optimization

## Table of Contents
- [Status](#status)
- [Context](#context)
  - [Problem Statement](#problem-statement)
  - [Architectural Reduction](#architectural-reduction)
  - [Optimization Characterization](#optimization-characterization)
- [Decision](#decision)
  - [Architectural Position](#architectural-position)
  - [Interface Contract](#interface-contract)
  - [Ideal Width Semantics](#ideal-width-semantics)
  - [The Gutter: System-Start Width Delta](#the-gutter-system-start-width-delta)
  - [Width Allocation: Proportional Scaling](#width-allocation-proportional-scaling)
  - [Variable System Widths](#variable-system-widths)
  - [Alternative: Asymmetric Cost Optimization](#alternative-asymmetric-cost-optimization)
  - [Page Breaking: Second Pass](#page-breaking-second-pass)
  - [Editorial Control Mechanisms](#editorial-control-mechanisms)
- [Protecting the Closed Semantic Model](#protecting-the-closed-semantic-model)
- [Stability Guarantees](#stability-guarantees)
  - [Stage Boundary Enforcement](#stage-boundary-enforcement)
  - [Connecting Element Adaptation](#connecting-element-adaptation)
  - [Controlled Reflow for Pathological Cases](#controlled-reflow-for-pathological-cases)
  - [Deterministic Tie-Breaking](#deterministic-tie-breaking)
  - [Edit Locality](#edit-locality)
  - [Caching and Incremental Recompute](#caching-and-incremental-recompute)
- [Implementation](#implementation)
  - [Core Dynamic Programming Structure](#core-dynamic-programming-structure)
  - [Segment Cost Computation](#segment-cost-computation)
  - [Width Allocation Implementation](#width-allocation-implementation)
  - [Break Reconstruction](#break-reconstruction)
  - [Key Implementation Notes](#key-implementation-notes)
- [Relationship to Knuth-Plass](#relationship-to-knuth-plass)
- [Consequences](#consequences)
- [References](#references)

## Status
Accepted

## Context

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Five-Stage Rendering Pipeline                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Stage 1 (Fan-out)     Stage 2 (Fan-in)      Stage 3 (Fan-out)              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐               │
│  │   Collision  │      │   Vertical   │      │    Symbol    │               │
│  │  Detection   │─────▶│Reconciliation│─────▶│ Preparation  │               │
│  │  (parallel)  │      │              │      │  (parallel)  │               │
│  └──────────────┘      └──────────────┘      └──────────────┘               │
│         │                     │                     │                       │
│         │              min, ideal, gutter           │                       │
│         │              per measure stack            │                       │
│         ▼                     ▼                     ▼                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Stage 4: DISTRIBUTION                            │    │
│  │                                                                     │    │
│  │   Input: N measure stacks with widths + gutter delta                │    │
│  │          {:min ratio, :ideal ratio, :gutter ratio}                  │    │
│  │                                                                     │    │
│  │   ┌─────────────────┐    ┌─────────────────┐                        │    │
│  │   │  System Break   │───▶│  Page Break     │                        │    │
│  │   │     DP O(N²)    │    │    DP O(S²)     │                        │    │
│  │   └─────────────────┘    └─────────────────┘                        │    │
│  │            │                     │                                  │    │
│  │            ▼                     ▼                                  │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │           Width Allocation per System                       │   │    │
│  │   │     available = system_width - gutter[s]                    │   │    │
│  │   │     actual_i = ideal_i × (available / Σ ideal_i)            │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                     │    │
│  │   Output: Final positions LOCKED                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Stage 5: Connecting Elements + Gutter Decorations                   │   │
│  │  Adapts to finalized geometry - does NOT influence distribution      │   │
│  │  Adds courtesy accidentals in gutter space when at system start      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Problem Statement

Given a musical score with N measures and M staves, the system must distribute N measure stacks (where each stack represents the vertical alignment of M staves at a single temporal position) across systems and pages.

**Input constraints per stack**:
- `min_width`: Hard lower bound from collision detection (atoms cannot overlap)
- `ideal_width`: Target proportional spacing based on musical density and conventional engraving practice
- `gutter_width`: Additional fixed space required when stack begins a system (default 0N)
- Deviation cost model: Function penalizing distance from ideal (typically convex)

**Distribution constraints**:
- System capacity: Fixed maximum width per system
- Page capacity: Fixed maximum height per page (number of systems)
- Stack atomicity: Measure stacks cannot be subdivided

**Explicit exclusions**:
Connecting elements (ties, slurs, hairpins, beams, glissandi, ottava lines) do not participate in the distribution optimization. They are computed in Stage 5 after positions are finalized, and adapt to the determined geometry. Only in rare pathological cases do they trigger controlled local adjustment.

**Objective**:
Minimize global discomfort derived from system-local stack deviations from ideal proportions, while satisfying capacity constraints. The intent is that minimizing global discomfort produces layouts perceived as stable and professionally typeset.

**Architectural thesis**:
The measure distribution problem, as typically formulated, couples vertical alignment, horizontal spacing, system breaks, page breaks, and connecting element geometry into a single intractable optimization. Ooloi's staged pipeline architecture decouples these concerns, collapsing the distribution problem into a form solvable by known algorithms (specifically, Knuth-Plass style dynamic programming). The innovation is not the algorithm—it is the architectural reduction that makes the algorithm applicable.

### Architectural Reduction

The apparent computational difficulty of music layout arises from coupling between multiple simultaneous concerns:

- Vertical alignment across staves (O(M²) per temporal position)
- Horizontal spacing within measures (dependent on symbol collisions)
- System and page distribution (combinatorial choices)
- Connecting element geometry (dependent on final positions)

Ooloi's pipeline architecture **decouples these concerns through staged computation**:

**Stage 1-2: Vertical Coordination**
Parallel collision detection produces measure stack metrics. Each of N stacks emerges with definitive width bounds: `min_width` and `ideal_width` for the measure's semantic content, plus `gutter_width` for additional system-start space. The vertical alignment problem is solved completely before distribution begins. This reduction is enabled by:
- Immutable data structures eliminating race conditions during parallel processing
- Rational arithmetic preserving exact proportions without floating-point accumulation
- STM coordination ensuring atomic reads of hierarchical musical structure

**Stage 3: Symbol Preparation**
Non-connecting visual elements (noteheads, accidentals, dynamics) are prepared using collision boundaries from Stage 1. These elements do not influence distribution.

**Stage 4: Distribution Optimization**
With vertical coordination complete and connecting elements deferred, the problem reduces to:
- A 1-dimensional sequence of N scalar triples (min_width, ideal_width, gutter_width)
- Gutter subtraction for system-start stacks before scaling
- Capacity-constrained segmentation into systems and pages
- Cost function on width deviations
- No feedback from geometry to distribution logic

**Stage 5: Connecting Elements**
Ties, slurs, and spanning attachments compute geometry based on finalized atom positions. They adapt to the determined layout rather than influencing it. For measures at system-start positions, Stage 5 adds graphical decorations (courtesy accidentals, tie continuations) within the reserved gutter space. Pathological cases (e.g., slur collisions requiring additional space) trigger explicit, localized reflow rather than systemic invalidation.

**Why this is architectural, not algorithmic**:
The reduction does not emerge from clever optimization techniques. It emerges from:
- Immutability preventing cascading state updates during iteration
- Semantic determinism (pitch identity, temporal ordering, accidental logic) established before layout begins
- Explicit stage boundaries preventing premature coupling
- Rational arithmetic eliminating error accumulation that would destabilize convergence

The result: a problem that would otherwise require heuristics, iteration limits, and manual correction becomes solvable by straightforward dynamic programming with guaranteed optimality. The algorithm is not novel; its applicability is what the architecture creates.

### Optimization Characterization

The distribution problem exhibits structure that admits polynomial-time exact optimization of the reduced subproblem:

**Discrete break selection**:
Dynamic programming over the sequence of N measure stacks determines optimal system and page break points. For each potential break location, the algorithm evaluates whether remaining measures fit within capacity (after reserving gutter space) and computes resulting discomfort. Optimal substructure holds: optimal solution for measures 1..k combined with optimal solution for measures k+1..N yields optimal solution for 1..N.

Break selection relies on **separable system costs** and **optimal substructure**, not convexity. The dynamic programming algorithm computes the globally optimal break configuration for the discrete segmentation problem.

Complexity: O(N²) for system breaks, O(S²) for page breaks where S = number of systems.

**Continuous width allocation**:
Within each system determined by break selection, actual widths are allocated by **proportional scaling** (normalisation) of the space remaining after gutter reservation:

```
available = system_width - gutter[s]
scale_factor = available / Σ ideal_i
actual_i = ideal_i × scale_factor
```

This is not optimization—it is a deterministic formula that preserves proportional relationships by construction. All measures in a system receive the same scale factor, so `actual_i / actual_j = ideal_i / ideal_j` always holds.

**Segment cost model**:
The discomfort for a system measures deviation from ideal proportions. Under proportional scaling, the cost simplifies to a function of how far the scale factor deviates from 1:

```
Σ (actual_i - ideal_i)² = Σ (ideal_i × scale - ideal_i)²
                        = (scale - 1)² × Σ ideal_i²
```

The DP selects breaks that minimize this total deviation. Scale factors < 1 indicate compression; > 1 indicates expansion. The gutter space is reserved but does not participate in cost computation—it is fixed overhead for system-start decorations.

**Additive separable cost model**:
The cost structure operates at two levels:
- **Stack-level discomfort**: For feasible segments, each stack's discomfort depends only on its `ideal` and the system's `scale_factor`. The `min` value participates in feasibility checking; the `gutter` value participates in available space calculation; neither participates in cost computation.
- **System-level cost**: Total discomfort for a system = Σ discomfort(stack_i) = `(scale - 1)² × Σ ideal_i²`
- **DP operates on system-level cost units**: The dynamic programming algorithm sums per-system discomfort values to compute global cost

This separability enables independent evaluation of candidate break points during dynamic programming.

**Page breaking: Second-order segmentation**:
Page breaking is treated as a **second segmentation pass over the system sequence, never interleaved with system breaking**. After optimal system breaks are determined, page breaks are computed independently via a second dynamic programming pass. This architectural separation prevents combined optimization attempts that would compromise determinism and tractability.

**Deterministic outcomes**:
Given identical input (measure stacks with their bounds, system/page capacities), the algorithm produces identical output. This determinism arises from:
- Rational arithmetic (no floating-point nondeterminism)
- Immutable data structures (no timing-dependent state)
- Deterministic evaluation order (DP iterates t increasing, s decreasing; updates only on strict improvement)
- Normalisation-based allocation (no iterative solver convergence)

Edit locality as first-class goal: When measures are inserted, deleted, or modified, the system aims to preserve break decisions for unaffected regions, minimizing perceptual layout "jitter" during editing.

## Decision

### Architectural Position

Measure distribution optimization is Stage 4 of the hierarchical rendering pipeline ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)). It receives measure stack metrics from Stages 1-2 and produces finalized positions that Stage 5 consumes.

The algorithm implements **capacity-constrained segmentation with proportional width allocation**:
1. Dynamic programming determines optimal system and page break points, accounting for gutter space at system starts
2. Proportional scaling allocates actual widths within the available space (after gutter reservation)

### Interface Contract

```clojure
;; Input from Stages 1-2 (computed by ADR-0038):
stacks ;; Vector of stack maps

;; Each stack map:
{:min ratio              ;; Hard collision boundary for measure content
 :ideal ratio            ;; Target proportional spacing for measure content
 :gutter ratio           ;; Additional space when at system start (default 0N)
 :measure-index int}     ;; Original measure index (for debugging/tracing)

;; Output:
{:breaks [break-positions]    ;; Vector of indices where systems start
 :cost total-discomfort}      ;; Total deviation cost (ratio)
```

**Width component semantics**:

- `:min` — collision floor for the measure's semantic content
- `:ideal` — proportional target for the measure's semantic content
- `:gutter` — additional space required when this measure appears first on a system (default 0N); this space is reserved for graphical decorations (courtesy accidentals, tie continuations) and does not scale

The gutter is *additional* to the measure's width, not part of it. When a measure appears first on a system, the system must accommodate `gutter[s] + actual[s]` for that measure.

**Preconditions** (guaranteed by upstream stages):
- `:ideal` must be positive for all stacks (otherwise scale factor computation fails)
- `:min` must be positive and `:min ≤ :ideal` for all stacks
- `:gutter` must be non-negative (default 0N)
- At least one feasible segmentation must exist (the entire sequence fits on some number of systems)

The computation of width values is handled by upstream stages and is specified in [ADR-0038: Horizontal Spacing](0038-Horizontal-Spacing.md). If preconditions are violated, behavior is undefined.

### Ideal Width Semantics

`ideal_width` represents **rhythmic proportionality** derived from note-value density through conventional engraving practice. `min_width` represents the **hard collision boundary** below which atoms would overlap.

**What `ideal_width` encodes**:
- Rhythmic proportionality reflecting note density
- Quarter notes occupy more space than eighths, regardless of measure count
- Nonlinear relationships from conventional engraving practice
- Musical meaning, not arbitrary aesthetic preference

**What `min_width` represents**:
- Hard feasibility floor from Stage 1 collision detection
- Below this: atoms would overlap on at least one staff (unacceptable)
- At this: collision-free across all staves, but visual characteristics vary by staff

**Stack-level vs Staff-level Effects**:

Width allocation operates on **measure stacks** (vertical alignment of M staves), not individual staves. The `min_width` of a stack reflects the requirements of its **most horizontally demanding staff**. Other staves in the same stack may have sparse notation and remain visually intact even when the stack is compressed to `min_width`.

Therefore, compression effects are **staff-local**, not stack-global:
- Dense staff (e.g., rapid sixteenth notes): compressed to minimum, proportionality degraded
- Sparse staff (e.g., whole notes): adequate spacing remains, proportionality preserved
- Stack as unit: satisfies collision constraints, but individual staff visual quality varies

This reinforces **stack atomicity** as the correct abstraction boundary. The distribution algorithm operates on stacks without needing staff-aware redistribution or corrective optimization. Visual degradation under compression affects specific staves within stacks, not the system as a whole.

**Key architectural achievement**:
Traditional engraving compromised proportionality to avoid collisions. Ooloi eliminates collisions upstream (Stage 1), removing the historical justification for proportionality sacrifice.

**Constraint structure**:
- **Hard constraint**: `actual_width ≥ min_width` (collision prevention)
- **Soft preservation**: `actual_width ∝ ideal_width` (proportionality maintenance)
- **No upper bound**: Measures may exceed ideal proportionally

### The Gutter: System-Start Width Delta

When a measure appears first on a system, it may require additional space for graphical decorations. This additional space is the **gutter**—a fixed-width region that precedes the measure's scaled content.

```
System layout when measure s is first:

┌──────────┬─────────────┬─────────────┬─────────────┐
│  GUTTER  │  measure s  │ measure s+1 │ measure s+2 │ ...
│ (fixed)  │ (scaled)    │  (scaled)   │  (scaled)   │
└──────────┴─────────────┴─────────────┴─────────────┘
     ↑           ↑
     │           └── actual[s] = ideal[s] × scale_factor
     │
     └── gutter[s] (does not scale)
```

**What the gutter accommodates:**
- Courtesy accidentals for tied-to notes at position 0 (graphical decorations, not semantic accidentals)
- Tie continuation arcs (visual portion of ties broken at system boundaries)

**Critical architectural property:** The gutter width is computed in Stage 1 from complete information about tied notes. Stage 4 knows exactly how much additional space each measure requires *before* making any distribution decisions. This enables mathematically optimal distribution without heuristics or iteration.

**What is NOT in the gutter:** All semantic accidentals are computed and positioned by [ADR-0035](0035-Remembered-Alterations.md) and are included in the measure's `min_width` and `ideal_width`. The gutter contains only graphical decorations added by Stage 5 for visual clarity at system boundaries.

**User control:** Users can configure when courtesy accidentals appear (`:system`, `:page`, or `:none`). This setting affects Stage 5 rendering, not Stage 4 distribution—the gutter is always reserved; the decorations are optionally rendered. See [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md) for details.

**Allocation with gutter:**

For a system containing stacks [s, t):
```
gutter = gutter[s]                           ;; Only first stack contributes
available = system_width - gutter            ;; Remaining space for scaling
scale_factor = available / Σ ideal_i         ;; Scale factor for all measures

actual[i] = ideal[i] × scale_factor          ;; ALL measures scale the same
```

**Verification:**
```
gutter + Σ actual_i = gutter + available = system_width ✓
```

The gutter is not added to any measure's width—it is separate space that precedes the first measure. Stage 5 renders gutter decorations into this space when the measure appears at system start.

### Width Allocation: Proportional Scaling

The primary width allocation strategy is **proportional scaling** (normalisation) applied to the space remaining after gutter reservation:

```
available = system_width - gutter[s]
scale_factor = available / Σ ideal_i
actual_i = ideal_i × scale_factor
```

This is **normalisation**, not optimisation:
- Deterministic formula with no degrees of freedom
- Preserves proportional relationships by construction
- Gutter space is reserved separately; it does not scale
- No iteration, no convergence, no tuning parameters
- Predictable behavior enabling edit locality
- Near-zero computational cost
- Clean architectural separation: DP selects breaks, allocation is derived

**Why this may be sufficient**:

1. **Proportionality preservation**: The scaling factor is identical for all measures, preserving ratios
2. **Gutter correctness**: System-start decorations occupy exactly their required space
3. **No heuristics**: Pure mathematical formula, deterministic outcome
4. **Speed**: O(K) per system with K ≈ 10-20, essentially free
5. **Predictability**: Users can mentally predict system behavior
6. **Edit stability**: Local changes cause minimal layout ripple

**Handling constraints**:
```clojure
actual_i = max(min_i, ideal_i × scale_factor)
```

The `max` clamp is defensive. Under the feasibility contract (`scale ≥ max(min_i / ideal_i)`), proportional scaling always produces `actual_i ≥ min_i`, so the clamp never changes any value. If clamping were to activate, it would indicate the segment was incorrectly selected as feasible.

**Notes on discomfort calculation**:

With proportional scaling, `actual_i = ideal_i × scale_factor` for all i. Therefore:
- All deviations are proportional: `deviation_i = ideal_i × (scale_factor - 1)`
- Quadratic penalty becomes: `ideal_i² × (scale_factor - 1)²`
- System discomfort = `(scale_factor - 1)² × Σ ideal_i²`

The gutter does not participate in cost computation. It is fixed overhead; the cost function measures only how far measure content deviates from ideal proportions.

**Comparison with TeX**:

TeX's glue model optimizes aesthetic badness with symmetric penalties around natural width. Ooloi's proportional scaling optimizes semantic preservation (proportionality) without penalties—it simply maintains ratios.

If asymmetric optimisation were adopted, it would differ from TeX by treating compression as semantically destructive (not merely aesthetically undesirable), but this remains speculative pending empirical validation.

### Variable System Widths

**System width is per-system policy input**, not a page-wide constant. The allocation formula is parameterized by `system_width` for each system independently.

**Any system may have a different width**, including but not limited to:
- The final system on a page (commonly narrower to avoid excessive whitespace)
- Systems following explicit editorial break markings
- Systems constrained by page geometry, margins, or layout policy
- Systems accommodating graphical elements or marginal annotations

**Architectural implications**:

1. **Break selection (DP) operates on feasibility and cost** with whatever width is provided for each candidate system position. The algorithm does not assume uniform justification.

2. **Width allocation is parameterized per system** after breaks are chosen. The proportional scaling formula works identically regardless of system width.

3. **Determinism is preserved**: Given the same break configuration and per-system width policies, allocation produces identical results.

4. **No algorithmic complexity added**: The DP evaluates `(s, t)` segments against the width available for that system position, minus gutter. Proportional scaling applies the provided width directly.

**Generalization of TeX's "last line" case**:

TeX treats the final line of a paragraph as potentially having different width (or justification policy). Ooloi generalizes this: **any system may have distinct width policy** without special-casing. The architecture naturally supports heterogeneous system widths because allocation is parameterized, not assumed.

**Expressivity without complexity**:

This architectural freedom enables:
- Natural handling of final systems (avoid excessive stretch)
- Editorial control over system widths for specific musical reasons
- Adaptation to page geometry constraints
- Future extensions (e.g., systems of varying width for visual effect)

All while preserving the core property: **proportional scaling maintains rhythmic relationships within whatever width is provided**, with gutter space reserved for decorations.

### Alternative: Asymmetric Cost Optimization

An alternative approach would treat width allocation as a constrained optimization problem:

```
minimise Σ f(ideal_i - actual_i)
subject to: Σ actual_i = available, actual_i ≥ min_i
```

where `f` is asymmetric: compression toward `min_width` penalized more heavily than expansion beyond `ideal_width`.

**Potential benefits**:
- Could avoid extreme compression by preferring non-uniform expansion
- Might improve visual clarity in edge cases
- Allows encoding subtle engraving preferences

**Real costs**:
- Requires iterative solver or closed-form derivation (non-trivial)
- Allocation now influences break selection (coupling)
- Tuning parameters and heuristic behavior
- Reduced predictability and edit locality
- Material implementation complexity
- Runtime cost: each of O(N×K) segments requires optimization

**Decision framework**:

This refinement should be considered **only if empirical evaluation** of proportional scaling reveals systematic visual problems that justify the complexity. The baseline approach should be implemented first, tested on real scores (simple → complex → Elektra), and evaluated for acceptability.

If proportional scaling proves adequate, asymmetric optimization becomes unnecessary complexity. If visual problems emerge, the specific issues will guide cost function design rather than speculative theory.

**When proportional scaling might be insufficient**:

1. **Extreme compression**: If `scale_factor` is very small (e.g., 0.6), all stacks compress uniformly. However, since stacks are heterogeneous (some staves dense, others sparse), visual degradation affects only the dense staves within each stack.

2. **Mixed density across stacks**: A system with one very dense stack and several sparse stacks scales uniformly at the stack level. Within the dense stack, some staves may be heavily compressed while others remain adequate.

3. **Cliff avoidance**: If a stack's `min_width` is very close to its `ideal_width` (indicating at least one very dense staff), proportional scaling might push that stack near the collision boundary.

**Note on staff-local effects**: Since compression primarily affects the most demanding staff within each stack while other staves may remain visually intact, the arguments for asymmetric optimization are weaker than they might initially appear. Stack-level allocation may be adequate even under compression.

These scenarios are theoretical. The actual question is: **do they occur in real scores, and do they produce unacceptable visual results?** Implementation should validate empirically before adding complexity.

### Page Breaking: Second Pass

Page breaking is a **second segmentation pass over the system sequence**, never interleaved with system breaking:

```clojure
(defn find-page-breaks [systems page-height-fn]
  ;; Same DP structure, different input:
  ;; - systems instead of stacks
  ;; - page-height-fn instead of system-width-fn
  ;; - system heights instead of widths
  ;; page-height-fn: (fn [start-sys end-sys] -> available-height)
  (find-optimal-breaks systems page-height-fn))
```

This architectural separation prevents combined optimization attempts that would compromise determinism.

### Editorial Control Mechanisms

Users require control over layout decisions: forcing system breaks at specific points, preventing breaks within phrases, adjusting individual measure widths. These controls fall into two distinct categories requiring different architectural treatment.

**Constraints vs Preferences**:

- **Constraints** are inviolable: "This measure *must* start a new system"
- **Preferences** are costs: "Prefer breaking at rehearsal marks"

Modeling constraints as extreme penalties (e.g., cost = ∞ for forbidden breaks) conflates these categories and risks numerical instability. Ooloi handles them through separate mechanisms.

**Forced System Breaks: Pre-Segmentation**

Forced breaks partition the stack sequence into independent optimization problems:

```clojure
(defn distribute-with-forced-breaks 
  "Partitions stack sequence at forced breaks, optimizes each segment independently.
   
   forced-break-indices: Set of measure indices that must start new systems.
   Returns combined break results across all segments."
  [stacks forced-break-indices system-width-fn]
  
  (let [;; Partition stacks at forced break points
        segments (partition-at-indices stacks forced-break-indices)]
    
    ;; Optimize each segment independently, then combine
    (->> segments
         (map #(find-optimal-breaks % system-width-fn))
         (combine-break-results))))
```

This is architecturally correct because:
- Forced breaks are constraints, not preferences—modeling them as partition boundaries is honest
- Each segment is an independent subproblem with its own optimal solution
- No numerical issues from infinity-like costs
- Clear semantics: the algorithm never considers configurations that violate forced breaks

**Prevented Breaks: Measure Grouping**

The inverse constraint—"do not break between measures 12 and 13"—is handled by treating measure groups as atomic units:

```clojure
(defn group-measures 
  "Combines consecutive measures into atomic groups that cannot be split.
   
   Each group becomes a single 'super-stack' with aggregated metrics.
   Only the first stack in the group contributes :gutter."
  [stacks no-break-ranges]
  
  (let [grouped (apply-grouping stacks no-break-ranges)]
    (mapv (fn [group]
            {:min (reduce + 0N (map :min group))
             :ideal (reduce + 0N (map :ideal group))
             :gutter (:gutter (first group))
             :member-stacks group})
          grouped)))
```

The DP operates on groups; after breaks are determined, groups expand back to constituent stacks for width allocation.

**Width Overrides: Upstream Modification**

User adjustments to individual measure widths ("stretch this measure," "compress this passage") belong upstream of Stage 4, not within it. Stage 4 consumes width triples; editorial overrides modify these inputs:

```clojure
(defn apply-width-overrides 
  "Applies user width overrides to stack metrics before distribution.
   
   Overrides may specify:
   - :min-override  - New minimum width (takes max with collision minimum)
   - :ideal-override - New ideal width (replaces rhythmic ideal)"
  [stacks user-overrides]
  
  (reduce 
    (fn [stacks {:keys [measure-index min-override ideal-override]}]
      (update stacks measure-index
        (fn [stack]
          (cond-> stack
            min-override   (update :min max min-override)
            ideal-override (assoc :ideal ideal-override)))))
    stacks
    user-overrides))
```

**Key principle**: The collision-derived `min_width` is a hard floor. User overrides can raise it but not lower it—atoms cannot overlap regardless of editorial intent. The `ideal_width` can be freely overridden since it represents preference, not physics. The `gutter` is not user-adjustable—it reflects SMuFL glyph metrics for system-start decorations.

**Soft Preferences: The break-penalty-fn Hook**

The optional `break-penalty-fn` parameter handles soft preferences that influence but do not constrain break selection:

```clojure
;; Prefer breaks at rehearsal marks
(defn rehearsal-mark-preference [stacks s t]
  (let [start-stack (nth stacks s)]
    (if (has-rehearsal-mark? start-stack)
      -50N  ; Negative cost = preference for this break
      0N)))

;; Avoid very short systems
(defn minimum-system-length-preference [stacks s t]
  (let [system-length (- t s)]
    (if (< system-length 3)
      100N  ; Positive cost = penalty for short systems
      0N)))

;; Combine multiple preferences
(defn combined-preferences [stacks s t]
  (+ (rehearsal-mark-preference stacks s t)
     (minimum-system-length-preference stacks s t)))
```

These preferences shift costs without creating hard constraints. The algorithm may still choose penalized configurations if overall discomfort is lower.

**Interaction Between Mechanisms**:

The mechanisms compose cleanly in a defined order:

1. **First**: Apply forced breaks to partition the problem
2. **Then**: Group measures with prevented breaks into super-stacks
3. **Then**: Apply width overrides to stack metrics
4. **Finally**: Run DP with soft preferences via `break-penalty-fn`

Each mechanism operates at a different level: problem partitioning, input transformation, and cost adjustment. This separation prevents the combinatorial complexity that arises when constraints and preferences are conflated into a single penalty system.

## Protecting the Closed Semantic Model

A fundamental architectural principle of Ooloi is that **rendering may never affect the semantic meaning of the internal model**. Once [ADR-0035](0035-Remembered-Alterations.md) computes accidental decisions, those decisions are final. The semantic model is closed.

### The Problem This Solves

Traditional notation software suffers from feedback loops between layout and semantics:

1. Compute layout
2. Discover system breaks
3. "We need courtesy accidentals here" - modify accidental decisions
4. Squash spacing to fit them
5. Maybe that changes system breaks
6. Iterate until "good enough" or give up

This approach has fundamental problems:
- **Non-deterministic**: Different runs may produce different results
- **Heuristic-driven**: "Good enough" is not optimal
- **Jitter-prone**: Small edits can cause cascading layout changes
- **Performance-limited**: Iteration caps are necessary to prevent infinite loops

### How Complete Information Enables Optimality

The gutter model eliminates feedback through **complete information**:

1. **Stage 1** computes exact measure content (min, ideal) + exact gutter delta
2. **Stage 4** has *complete knowledge* of both scenarios (mid-system and system-start) for every measure
3. **Stage 4** computes *mathematically optimal* distribution - not heuristic, not iterative
4. **Stage 5** adds graphical decorations based on actual positions

The gutter width is not "we might need space" - it's "this is exactly how much space the graphical decoration requires." Stage 4 doesn't guess. It has full information to make the globally optimal decision in one pass.

### Semantic vs Graphical Content

The distinction is critical:

**Semantic content** (determined by ADR-0035, immutable after Stage 1):
- Accidentals required by alteration rules
- Accidentals required by remembered alterations
- Accidentals explicitly requested (French ties)
- Bypass decisions for tied-to notes

**Graphical decorations** (added by Stage 5 based on layout):
- Courtesy accidentals at system start for bypassed tied-to notes
- Tie continuation arcs
- Visual aids that don't affect musical meaning

The semantic model remains closed. Stage 5 only adds visual decoration within pre-reserved space.

### Why This Matters

With complete information and no feedback:
- **Deterministic**: Same input always produces same output
- **Optimal**: Not "good enough" but mathematically best
- **Stable**: No jitter from iteration or heuristics
- **Fast**: One pass through the pipeline, no convergence loops

This is the architectural achievement: problems that seemed to require feedback collapse into pure forward flow when the right information is available at the right stage.

### Enabling User Control

Complete information enables user-controllable settings that would otherwise require manual adjustment or heuristics:

```clojure
:system-break-cautionary-accidentals
  :none   ;; Never show courtesy accidentals at system breaks  
  :page   ;; Show only at page breaks
  :system ;; Show at all system breaks
```

The setting affects gutter computation in Stage 1, not Stage 4 logic. Stage 4 receives different gutter values and computes the optimal distribution for those values. Changing the setting produces a new optimal layout—automatically, deterministically, without iteration or manual correction.

**Architectural capability:** The complete-information architecture makes user-controllable courtesy accidental settings straightforward to implement—a feature that requires deterministic distribution to work without manual adjustment or layout jitter. Traditional architectures that lack complete information at distribution time cannot offer such settings without risking non-deterministic behavior or requiring iterative correction.

This demonstrates how complete information transforms potentially complex features into simple input variations. Settings that would otherwise require manual adjustment, iterative correction, or special-case handling become straightforward parameter changes to a general algorithm. The architecture enables the feature; the feature validates the architecture.

## Stability Guarantees

### Stage Boundary Enforcement

After Stage 4 completes, measure stack positions are **finalized**. No subsequent stage modifies distribution decisions. This architectural invariant prevents feedback loops where:
- Stage 5 connecting elements request more space
- Stage 4 redistribution invalidates Stage 5 geometry
- Mutual invalidation prevents convergence

### Connecting Element Adaptation

Ties, slurs, hairpins adapt their curves and control points to fit finalized atom positions. The vast majority of musical notation admits this adaptation without requiring additional space.

For measures at system-start positions, Stage 5 adds graphical decorations within the gutter space:
- Courtesy accidentals for tied-to notes (visual aids, not semantic accidentals)
- Tie continuation arcs

This is **selection, not computation**. The space was pre-reserved; Stage 5 renders decorations into it.

### Controlled Reflow for Pathological Cases

In rare situations (e.g., short slur with extreme stem directions creating unavoidable collision), the system may trigger **local, explicit reflow**:

- **Stage 5 is not allowed to request redistribution.** It may only request explicit, bounded re-execution of Stage 4 on a constrained interval.
- Reflow scope limited to affected measures and immediate neighbors
- Reflow occurs after Stage 5 detects geometric impossibility, not during Stage 4 optimization
- Reflow outcome cached to prevent repeated recomputation
- User notification of layout adjustment (optional, depending on severity)

This differs fundamentally from continuous mutual invalidation in mutable architectures, where any geometry change can cascade through the entire layout system. Stage 5 does not participate in optimization; any reflow is a restart, not feedback.

### Deterministic Tie-Breaking

When multiple break configurations produce identical discomfort (a "tie" in cost, not a musical tie), the DP evaluation order ensures consistent choices:

1. Outer loop iterates `t` from 1 to N (increasing)
2. Inner loop iterates `s` from `t-1` to 0 (decreasing)
3. Updates occur only on strict improvement (`<`, not `<=`)

This means: for equal costs, the algorithm retains the first valid configuration found (larger segments when possible). The result is deterministic across runs and platforms.

If editorial preferences are needed (e.g., prefer structural boundaries), they can be encoded via the optional `break-penalty-fn` parameter, which shifts costs rather than relying on tie-breaking.

### Edit Locality

When measures are modified, the system attempts to preserve existing break decisions for distant unaffected regions. This minimizes perceptual layout "jitter" during editing. Immutable data structures enable efficient incremental recomputation: only modified subtrees require re-evaluation.

### Caching and Incremental Recompute

Stage 4 operates over a sequence of measure stacks whose width triples are **already finalized upstream**. In addition, Ooloi caches the following per **measure stack**:

* `min_width` — hard lower bound for measure content (from Stage 1–2)
* `ideal_width` — proportional target for measure content
* `gutter_width` — additional space for system-start decorations
* `actual_width` — realized width after Stage 4 allocation

This cache is authoritative at the Stage 4 boundary: Stage 4 consumes `(min, ideal, gutter)` and produces `actual` and break assignments; Stage 5 consumes `actual` and never influences Stage 4 except via explicitly bounded reflow (pathological cases).

#### Cached Invariants

For each stack `i` under a fixed system assignment:

* `actual_width_i ≥ min_width_i`
* `actual_width_i = ideal_width_i × scale_factor(system)` unless clamped
* `scale_factor(system) = available / Σ ideal_width_j` where `available = system_width - gutter[s]`

The cache additionally implies that per-stack metrics are **stable across edits** unless the edited content is in that stack (or an explicitly bounded reflow is triggered).

#### Incremental Update Consequences

Edits affect Stage 4 only through changes to stack metrics:

1. **Local edit**: modifying or inserting notation in a measure updates only the affected stack's upstream-derived width values.

2. **Fast path (break stability)**: if the updated stack's metrics do not change the optimal break configuration (or do not change the stack's system membership), then:

   * only that stack's `actual_width` may change (via the system scale factor), and often not even that
   * Stage 5 updates are confined to connecting elements that touch the edited measure and/or the edited stack
   * no other systems require recomputation

3. **Ripple path (break changes)**: if the updated stack's metrics change feasibility or cost enough to alter system breaks, recomputation is still bounded:

   * unaffected stacks retain cached metrics and therefore remain identical DP input
   * only the region whose optimal substructure changes must be re-evaluated; once break decisions converge back to the previous solution, downstream layout is provably unchanged (determinism + identical input suffix)
   * `actual_width` recomputation is confined to systems whose membership or width policy changed

#### Practical Complexity

With cached stack metrics, Stage 4 runtime is dominated by DP over **scalars**, not by collision/layout work. Edit-time recomputation is typically:

* **O(size of edited measure)** for upstream recalculation of the edited stack
* plus **O(K)** (where `K ≈ measures/system`) for local system allocation updates
* plus at worst **O(Δ×K)** when a breakpoint ripple propagates across `Δ` stacks, with all other stacks remaining cache hits

#### Relationship to Edit Locality

This caching strategy is the concrete mechanism behind the ADR's edit-locality goal:

* locality is achieved not by heuristics, but by **stable, cached stage outputs** and deterministic recomputation on a reduced 1D representation
* the system avoids whole-score reflow not by forbidding it, but because cached aggregates make whole-score reflow unnecessary in the common case

## Implementation

### Core Dynamic Programming Structure

```clojure
(defn find-optimal-breaks
  "Finds optimal system breaks for measure stacks using dynamic programming.
   
   Input:
   - stacks: Vector of {:min ratio, :ideal ratio, :gutter ratio, :measure-index int} maps
   - system-width-fn: Function (fn [start-pos end-pos] -> ratio) 
                      Returns available width for system containing stacks start-pos..end-pos-1
   - break-penalty-fn: Optional. Function (fn [stacks s t] -> ratio)
                       Returns additional cost for selecting segment s..t-1 as a system
                       Default: (constantly 0N) - no editorial preferences
   
   Output:
   - {:breaks [break-positions], :cost total-discomfort-ratio}"
  
  ([stacks system-width-fn]
   (find-optimal-breaks stacks system-width-fn (constantly 0N)))
  
  ([stacks system-width-fn break-penalty-fn]
   (let [n (count stacks)
         
         ;; Precompute prefix sums for O(1) range queries
         ;; CRITICAL: Use 0N to maintain ratio domain
         min-prefix (vec (reductions + 0N (map :min stacks)))
         ideal-prefix (vec (reductions + 0N (map :ideal stacks)))
         ideal-sq-prefix (vec (reductions + 0N (map #(let [x (:ideal %)] (* x x)) stacks)))
         
         ;; State arrays: nil = unreachable, ratio = actual cost
         ;; Length n+1 where index t represents prefix of length t
         best (transient (vec (repeat (inc n) nil)))
         prev (transient (vec (repeat (inc n) nil)))]
     
     ;; Base case: empty prefix reachable with zero cost
     (assoc! best 0 0N)
     
     ;; For each prefix length t (represents stacks 0..t-1)
     (doseq [t (range 1 (inc n))]
       
       ;; Try previous break at prefix length s (represents stacks 0..s-1)
       ;; System contains stacks s..t-1
       (doseq [s (range (dec t) -1 -1)
               :let [system-width (system-width-fn s t)
                     ;; Reserve gutter space for system-start stack
                     gutter (get-in stacks [s :gutter] 0N)
                     available (- system-width gutter)
                     ;; Check if measures can fit in remaining space
                     total-min (- (nth min-prefix t) (nth min-prefix s))]
               :while (<= total-min available)]
         
         ;; Check tighter feasibility: scale factor must not require clamping
         (let [total-ideal (- (nth ideal-prefix t) (nth ideal-prefix s))
               scale-factor (/ available total-ideal)
               segment-stacks (subvec stacks s t)
               max-ratio (reduce max 0N (map #(/ (:min %) (:ideal %)) segment-stacks))
               feasible? (>= scale-factor max-ratio)]
           
           (when (and feasible? (some? (nth best s)))
             ;; Compute cost using closed form (valid because feasibility guarantees no clamping)
             ;; cost = (scale - 1)² × Σ ideal_i²
             ;; Gutter does not participate in cost - it is fixed overhead
             (let [total-ideal-sq (- (nth ideal-sq-prefix t) (nth ideal-sq-prefix s))
                   scale-deviation (- scale-factor 1N)
                   geometric-cost (* (* scale-deviation scale-deviation) total-ideal-sq)
                   penalty-cost (break-penalty-fn stacks s t)
                   segment-cost (+ geometric-cost penalty-cost)
                   total-cost (+ (nth best s) segment-cost)
                   current-best (nth best t)]
               
               ;; Update if this is better (nil = infinity)
               (when (or (nil? current-best)
                         (< total-cost current-best))
                 (assoc! best t total-cost)
                 (assoc! prev t s)))))))
     
     ;; Return results
     {:breaks (reconstruct-breaks (persistent! prev) n)
      :cost (nth best n)})))
```

**Gutter handling explained:**

For each candidate segment [s, t):
1. Get `gutter = gutter[s]` (only first stack in system contributes)
2. Compute `available = system_width - gutter`
3. Check feasibility: `Σ min_i ≤ available`
4. Compute scale factor: `available / Σ ideal_i`
5. Cost is based on scalable content deviation only; gutter is fixed overhead

The gutter is reserved space that does not participate in scaling or cost computation.

**Feasibility condition:**

The check `Σ min_i ≤ available` is **necessary but not sufficient**. The **sufficient condition** requires:
```
scale_factor = available / Σ ideal_i
scale_factor ≥ max(min_i / ideal_i) for all i
```

This ensures proportional scaling produces `actual_i ≥ min_i` for all stacks.

**Width Policy Function Examples**:

The `system-width-fn` accepts `(start-pos, end-pos)` and returns the total available width for that system (including gutter space):

```clojure
;; Uniform width - all systems same width
(fn [s t] standard-width)

;; Narrow final system
(fn [s t] (if (= t n) final-width standard-width))

;; Position-based policy - varying by page position
(fn [s t] (width-for-position t))

;; Editorial control - manual overrides
(fn [s t] (lookup-explicit-width s t))
```

The DP algorithm subtracts gutter from the policy-provided width to determine space for scaling.

### Segment Cost Computation

Under the feasibility contract (`scale ≥ max(min_i / ideal_i)`), proportional scaling produces `actual_i = ideal_i × scale` with no clamping required. The segment cost therefore has a closed form:

```
deviation_i = actual_i - ideal_i = ideal_i × scale - ideal_i = ideal_i × (scale - 1)

cost = Σ deviation_i²
     = Σ (ideal_i × (scale - 1))²
     = (scale - 1)² × Σ ideal_i²
```

With precomputed prefix sums for `Σ ideal_i` and `Σ ideal_i²`, segment cost is O(1):

```clojure
;; Closed-form cost computation (inline in DP loop)
(let [gutter (get-in stacks [s :gutter] 0N)
      available (- system-width gutter)
      total-ideal (- (nth ideal-prefix t) (nth ideal-prefix s))
      total-ideal-sq (- (nth ideal-sq-prefix t) (nth ideal-sq-prefix s))
      scale-factor (/ available total-ideal)
      scale-deviation (- scale-factor 1N)
      geometric-cost (* (* scale-deviation scale-deviation) total-ideal-sq)]
  ...)
```

This is not an optimization of a more complex algorithm—it is the canonical cost definition under the ADR's contract. The feasibility check guarantees no clamping, so the closed form gives mathematically identical results to explicit allocation.

### Width Allocation Implementation

```clojure
(defn allocate-widths [system-stacks system-width]
  "Allocates actual widths using proportional scaling (normalisation).
   
   Used after break selection to compute final positions for rendering.
   Under the feasibility contract, all stacks satisfy actual_i ≥ min_i
   without clamping.
   
   The first stack (index 0) contributes :gutter, which is reserved space
   that does not scale. All measures scale within the remaining space.
   
   Returns map with:
   - :gutter - reserved gutter width
   - :actuals - vector of actual widths for each measure"
  
  (let [;; Reserve gutter space for first stack
        gutter (get (:gutter (first system-stacks)) 0N)
        available (- system-width gutter)
        total-ideal (reduce + 0N (map :ideal system-stacks))
        scale-factor (/ available total-ideal)]
    
    {:gutter gutter
     :actuals (mapv (fn [stack]
                      (let [scaled (* (:ideal stack) scale-factor)]
                        ;; Defensive clamp: under feasibility contract, this never changes the value
                        (max scaled (:min stack))))
                    system-stacks)}))
```

**Why this approach**:

1. **Normalisation, not optimisation**: No degrees of freedom, no iteration, no convergence concerns
2. **Preserves proportionality by construction**: `actual_i / actual_j = ideal_i / ideal_j` for all i,j
3. **Gutter correctness**: System-start decorations have exactly their required space
4. **Deterministic**: Same input always produces same output with no floating-point variation
5. **Fast**: O(K) where K ≈ 10-20, essentially zero cost
6. **Predictable**: Users can mentally predict allocation behavior
7. **Edit-stable**: Local changes cause minimal layout ripple

**Verification**:
```clojure
(let [{:keys [gutter actuals]} (allocate-widths stacks width)]
  (assert (= width (+ gutter (reduce + 0N actuals)))))
```

**Defensive clamp**:

The `max` clamp is purely defensive programming:
- The DP feasibility check ensures `scale_factor ≥ max(min_i / ideal_i)`
- This guarantees `scaled = ideal_i × scale_factor ≥ min_i` for all stacks
- Under the feasibility contract, the clamp never changes any value
- Debug builds may assert this invariant: `(assert (>= scaled (:min stack)))`

### Break Reconstruction

```clojure
(defn reconstruct-breaks [prev n]
  "Walks backward through prev array to build break positions.
   
   Returns vector of break positions (indices where systems start)."
  
  (loop [breaks []
         pos n]
    (if (<= pos 0)
      (vec (reverse breaks))
      (let [prev-break (nth prev pos)]
        (recur (conj breaks prev-break) prev-break)))))
```

### Key Implementation Notes

**Gutter Handling**:

Each stack carries `:gutter` (default 0N). Only the first stack in a system contributes this value:
- Reserved from system width before scaling: `available = system_width - gutter`
- Does not scale—it is fixed space for graphical decorations
- Does not participate in cost computation
- Returned separately from `allocate-widths` for rendering coordination

**Baseline: Proportional Scaling (Normalisation)**:

The implementation uses proportional scaling as the primary width allocation strategy. This is normalisation (deterministic formula), not optimisation (iterative solver):

- **Formula**: `actual_i = ideal_i × (available / Σ ideal_i)`
- **Gutter reservation**: `available = system_width - gutter[s]`
- **Preserves proportionality**: All measures scale by same factor
- **No degrees of freedom**: Allocation is purely derived from break choice
- **Fast**: O(K) per system, essentially zero cost
- **Predictable**: Mental model matches implementation
- **Edit-stable**: Local changes have minimal layout impact

This approach may prove entirely adequate. If empirical testing reveals systematic visual problems, asymmetric optimisation can be considered, but complexity should not be added speculatively.

**Extension Point: break-penalty-fn**:

The algorithm accepts an optional `break-penalty-fn` parameter with signature `(fn [stacks s t] -> ratio)`. It returns additional cost for selecting segment `s..t-1` as a system. The default `(constantly 0N)` produces purely geometric optimization. This ADR does not specify any concrete penalty functions; editorial policy is a separate concern that can evolve independently of the core algorithm.

**Feasibility Check**:

The DP uses a two-part feasibility check:
1. **Necessary**: `Σ min_i ≤ available` (minimums must fit after reserving gutter) - O(1) via prefix sums
2. **Sufficient for proportional scaling**: `scale_factor ≥ max(min_i / ideal_i)` - O(K) segment scan

The second condition ensures the scale factor is large enough that no stack requires clamping. Without it, proportional scaling could produce allocations below minimums.

**Segment Evaluation**:

Each candidate segment (s, t) requires:
1. **Gutter lookup**: O(1) to get `gutter[s]`
2. **Feasibility**: O(K) scan to compute `max(min_i / ideal_i)` over the segment
3. **Cost**: O(1) closed-form computation using prefix sums

The feasibility scan is the only per-segment O(K) work. Cost computation uses precomputed `ideal-sq-prefix` for O(1) evaluation.

**Rational Arithmetic Throughout**:
- All numeric literals use `N` suffix: `0N`
- Preserves exact arithmetic, no floating-point contamination
- Ensures deterministic outcomes across platforms
- Ratio growth managed through Clojure's automatic normalization (gcd)

**nil Semantics for Infinity**:
- `nil` represents unreachable positions (cost = ∞)
- Avoids mixing `##Inf` (Double) with ratios
- Clean comparison: `(or (nil? x) (< new x))`
- Type-safe throughout

**Transients for Local Mutation**:
- `best` and `prev` arrays use transients for performance
- Sequential processing only (no parallelism in DP loop)
- Thread safety at piece level, not within algorithm
- Converted to persistent at end

**Prefix Sums for O(1) Queries**:
- Precomputed once: O(N) time for `min-prefix`, `ideal-prefix`, and `ideal-sq-prefix`
- Range sum in O(1): `(- (nth prefix t) (nth prefix s))`
- Enables O(1) cost computation via closed form
- Must use `0N` in reductions to maintain ratio domain
- Note: gutter is O(1) per-segment lookup, not prefix-summed (only first stack contributes)

**Early Termination**:
- `:while` in inner loop stops when stacks no longer fit
- Monotonicity: as `s` moves left, `total-min` increases (more stacks)
- Once infeasible, all earlier `s` also infeasible
- Reduces effective complexity to O(N×K) where K ≈ 15
- **Assumption**: `system-width-fn` must be non-increasing as `s` moves left (for fixed `t`). This holds for typical policies (uniform width, narrower final system). If violated, early termination is invalid and the algorithm must check all `s` values.

**Complexity Analysis**:

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Precomputation | O(N) | Prefix sums for min, ideal, and ideal² |
| Outer loop | O(N) | Each position once |
| Inner loop | O(K) typical | Early termination when infeasible |
| Gutter lookup | O(1) | Per-segment constant time |
| Feasibility check | O(K) | Scan for max(min_i / ideal_i) |
| Cost computation | O(1) | Closed form using prefix sums |
| **Total (worst case)** | O(N²) | Without early termination |
| **Total (typical)** | O(N×K) | With monotone width policy, K ≈ 15 |
| Page breaking | O(S²) | S = number of systems |

At N=800 measures, K=15: ~12,000 segment evaluations. Feasibility is O(K) per segment; cost is O(1). Still trivial.

## Relationship to Knuth-Plass

**Structural analogy**:

| TeX Paragraph Breaking | Ooloi Measure Distribution |
|------------------------|----------------------------|
| Word sequence | Measure stack sequence |
| Natural width + glue | Ideal width + flexibility |
| Line capacity | System capacity (minus gutter) |
| Badness function | Discomfort function |
| Aesthetic optimization (symmetric) | Proportionality preservation |
| Dynamic programming | Dynamic programming |
| Optimal line breaks | Optimal system breaks |

Both problems share the mathematical structure enabling polynomial-time exact optimization of the reduced subproblem:
- 1-dimensional sequence with scalar preferences
- Capacity-constrained segmentation
- Separable cost function with optimal substructure
- Convex cost model for within-segment allocation

**Explicit non-claim**:
This approach does **not** solve "general music engraving" or "optimal notation layout for all musical structures." It solves the **measure distribution subproblem** after architectural reduction has transformed it into tractable form.

The reduction is what matters: by solving vertical coordination, symbol collision, and semantic determinism in earlier stages, the distribution problem becomes structurally similar to paragraph breaking. This is not algorithmic cleverness finding a better heuristic—it is architectural separation creating a problem formulation where exact optimization applies.

**Critical semantic difference from TeX**:

While structurally analogous, Ooloi's problem has a different semantic emphasis than TeX paragraph breaking:

- **TeX optimizes aesthetic badness**: Natural width represents aesthetic ideal, compression and expansion are symmetrically undesirable deviations from beauty

- **Ooloi preserves proportionality**: `ideal_width` encodes rhythmic proportionality with musical meaning. The baseline approach (proportional scaling) maintains these relationships by construction rather than through optimisation.

The key difference is not in the cost function but in the goal: TeX seeks visual balance through symmetric penalties. Ooloi seeks semantic preservation through proportionality maintenance, with graphical decorations (gutter content) accommodated as fixed overhead.

If future refinement uses asymmetric optimisation, the distinction would sharpen: compression toward `min_width` would be treated as semantically destructive (destroying proportionality), not merely aesthetically undesirable. But this remains speculative pending empirical validation of the baseline approach.

**Architectural prerequisites**:

The Knuth-Plass algorithm solves a specific problem formulation: capacity-constrained segmentation of a 1-dimensional sequence with separable costs. Music layout, naively formulated, is not this problem—it couples vertical alignment, collision detection, semantic decisions, and geometry into mutual dependencies.

Ooloi's pipeline architecture transforms the problem. By the time Stage 4 executes:
- Vertical coordination is complete (Stages 1-2)
- Collision boundaries are determined (Stage 1)
- Semantic decisions (accidentals, beaming) are resolved (ADR-0035)
- Gutter requirements are computed (Stage 1)
- Connecting elements are deferred (Stage 5)

What remains is exactly the Knuth-Plass problem formulation. The algorithm is textbook; its applicability is what the architecture creates.

**Why this approach is commonly avoided in real-time notation editors**:

The Knuth-Plass algorithm is well-known in typesetting circles. Real-time notation editors have commonly avoided it—not because the algorithm is unsuitable to music, but because their architectures lack the preconditions that make it applicable. Mutable state creates feedback loops between spacing and symbol positioning. Coupled evaluation of horizontal and vertical concerns prevents the clean 1-dimensional reduction. The algorithm requires a problem formulation that these architectures cannot provide.

Performance concerns likely compound the architectural barriers. Knuth-Plass itself is efficient—O(N²) in the general case, O(N×K) with early termination. But in architectures where spacing decisions feed back into collision detection, which feeds back into spacing, the algorithm would need to run repeatedly as geometry iteratively stabilizes. Each edit could trigger multiple full recomputations. The cost isn't the algorithm; it's the inability to run it once on stable inputs. When you cannot guarantee that width metrics are finalized before distribution begins, even an efficient algorithm becomes expensive through repetition.

Ooloi's decoupled pipeline inverts this situation. Stage 4 receives stable scalar inputs from completed upstream stages. The DP executes once, on data that will not change during computation. Immutability enables precise cache invalidation: edits affect only the stacks whose upstream metrics actually changed, not the entire sequence. What would be expensive in an iterative architecture becomes trivial: at N=800 measures with K=15 measures per system, approximately 12,000 segment evaluations of simple arithmetic complete in milliseconds.

Ooloi's ability to apply Knuth-Plass is therefore not algorithmic sophistication but architectural consequence. The pipeline stages, immutable data structures, and rational arithmetic create both the problem formulation and the performance characteristics the algorithm requires. This is the pattern throughout Ooloi: apparently simple solutions become available—and become fast—when architecture eliminates the coupling that made them inapplicable.

## Consequences

**Architectural pattern**:

This ADR demonstrates the same architectural property as [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md). In both cases, problems that traditionally require heuristics, special cases, or manual correction collapse into straightforward algorithms when the architecture provides:
- Immutable data structures
- Semantic determinism resolved before the algorithm executes
- Explicit stage boundaries preventing feedback loops
- Rational arithmetic eliminating accumulation errors

For remembered alterations, the timewalk provides temporal ordering independent of visual layout. For measure distribution, the pipeline provides collision-free metrics (plus gutter deltas) independent of system assignment. Both reduce coupled problems to sequential ones.

**Positive:**

1. **Exact optimization** - Polynomial-time algorithm finds globally optimal break configuration
2. **Proportionality preservation** - Rhythmic relationships maintained across systems by construction
3. **Complete information** - Gutter deltas enable optimal decisions without feedback
4. **Closed semantic model** - Rendering decorations cannot affect musical semantics
5. **Deterministic output** - Identical input produces identical layout across platforms
6. **Edit locality** - Changes affect only local regions where possible
7. **Variable system widths** - Natural handling of final systems, editorial overrides, page geometry
8. **Clean stage separation** - No feedback loops with connecting elements or decorations
9. **Performance** - O(N×K) complexity handles large scores efficiently
10. **Rational arithmetic** - No floating-point drift across platforms
11. **Predictability** - Users can mentally model system behavior

**Neutral:**

1. **Two-pass approach** - System and page breaking separated, not jointly optimized
2. **Quadratic cost** - Simple discomfort model; asymmetric penalties deferred pending empirical validation
3. **Staff-local effects** - Compression affects individual staves within stacks, not uniform visual degradation
4. **Policy-free core** - Algorithm is purely geometric by default; editorial preferences can be added via optional `break-penalty-fn` without modifying the core
5. **Gutter storage** - Each stack carries `gutter` (typically 0N); overhead is minimal

**Future Considerations:**

The proportional scaling approach should be validated empirically on real scores:
1. Implement baseline proportional scaling as specified
2. Test on real scores: simple → complex → Elektra
3. Document visual quality, performance characteristics, edge cases
4. Only if systematic problems emerge, consider asymmetric optimization refinements

The formal validation will determine whether the baseline approach is sufficient or whether additional complexity is justified. The specific visual issues encountered will guide cost function design rather than speculative theory.

## References

- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) (pipeline architecture, Stage 4 position, gutter model, closed semantic model)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) (accidental algorithm that closes the semantic model; tied-note bypass rules inform gutter requirements)
- [ADR-0038: Horizontal Spacing](0038-Horizontal-Spacing.md) (upstream computation of min/ideal/gutter widths)
- [ADR-0014: Timewalk](0014-Timewalk.md) (temporal traversal providing measure discovery)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md) (immutable data structures enabling stage separation)
- [Knuth–Plass line-breaking algorithm](https://en.wikipedia.org/wiki/Knuth%E2%80%93Plass_line-breaking_algorithm) - Wikipedia overview
- Knuth, D.E. and Plass, M.F. "Breaking Paragraphs into Lines" (1981) - foundational algorithm
- Ross, T. "The Art of Music Engraving and Processing" (1970) - proportional spacing values
- Gould, E. "Behind Bars" (2011) - modern engraving standards