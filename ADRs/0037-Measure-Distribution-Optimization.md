# ADR-0037: Measure Distribution Optimization

## Status

Accepted

## Table of Contents
- [Context](#context)
  - [Problem Statement](#problem-statement)
  - [Architectural Reduction](#architectural-reduction)
  - [Optimization Characterization](#optimization-characterization)
- [Decision](#decision)
  - [Architectural Position](#architectural-position)
  - [Interface Contract](#interface-contract)
  - [Ideal Width Semantics](#ideal-width-semantics)
  - [The Gutter: System-Start Width Delta](#the-gutter-system-start-width-delta)
  - [The Preamble: System-Start Clef and Key Signature Width](#the-preamble-system-start-clef-and-key-signature-width)
  - [Width Allocation: Proportional Scaling](#width-allocation-proportional-scaling)
  - [Variable System Widths](#variable-system-widths)
  - [Alternative: Asymmetric Cost Optimization](#alternative-asymmetric-cost-optimization)
  - [Page Breaking: Second Pass (Stage 6)](#page-breaking-second-pass-stage-6)
  - [Editorial Control Mechanisms](#editorial-control-mechanisms)
- [Protecting the Closed Semantic Model](#protecting-the-closed-semantic-model)
  - [The Problem This Solves](#the-problem-this-solves)
  - [How Complete Information Enables Optimality](#how-complete-information-enables-optimality)
  - [Semantic vs Graphical Content](#semantic-vs-graphical-content)
  - [Why This Matters](#why-this-matters)
  - [Enabling User Control](#enabling-user-control)
- [Stability Guarantees](#stability-guarantees)
  - [Stage Boundary Enforcement](#stage-boundary-enforcement)
  - [Connecting Element Adaptation](#connecting-element-adaptation)
  - [Deterministic Tie-Breaking](#deterministic-tie-breaking)
  - [Edit Locality](#edit-locality)
  - [Caching and Incremental Recompute](#caching-and-incremental-recompute)
- [Stability and Quality Extensions](#stability-and-quality-extensions)
  - [Evaluation Criteria](#evaluation-criteria)
  - [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit)
  - [Stability Term](#stability-term)
  - [Frozen Systems](#frozen-systems)
  - [Consistency Between Adjacent Systems](#consistency-between-adjacent-systems)
  - [Badness, Ending Classes and Looseness](#badness-ending-classes-and-looseness)
  - [Fitting: Proportional or Even](#fitting-proportional-or-even)
  - [Column-Level Feasibility](#column-level-feasibility)
  - [End-of-System Width](#end-of-system-width)
  - [Joint System and Page Breaking](#joint-system-and-page-breaking)
- [Implementation](#implementation)
  - [Core Dynamic Programming Structure (Stage 3)](#core-dynamic-programming-structure-stage-3)
  - [Segment Cost Computation](#segment-cost-computation)
  - [Width Allocation Implementation](#width-allocation-implementation)
  - [Break Reconstruction](#break-reconstruction)
  - [Key Implementation Notes](#key-implementation-notes)
- [Relationship to Knuth-Plass](#relationship-to-knuth-plass)
- [Prior Art](#prior-art)
  - [Optimal Breaking by Dynamic Programming](#optimal-breaking-by-dynamic-programming)
  - [The Space Model](#the-space-model)
  - [Comparison](#comparison)
  - [What Is Specific to Ooloi](#what-is-specific-to-ooloi)
- [Consequences](#consequences)
- [References](#references)

## Context

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Six-Stage Rendering Pipeline                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Stage 1 (Fan-out)     Stage 2 (Fan-in)                                 │
│  ┌──────────────┐      ┌──────────────┐                                 │
│  │ Atom Form.   │      │   Vertical   │                                 │
│  │ & Collision  │─────▶│Reconciliation│                                 │
│  │  (parallel)  │      │              │                                 │
│  └──────────────┘      └──────────────┘                                 │
│         │                     │                                         │
│         │              min, ideal, gutter                               │
│         │              per measure stack                                │
│         ▼                     ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │               Stage 3: SYSTEM BREAKING (Single)                 │    │
│  │                                                                 │    │
│  │   Input: N measure stacks with widths + gutter delta            │    │
│  │          {:min ratio, :ideal ratio, :gutter ratio}              │    │
│  │                                                                 │    │
│  │   ┌─────────────────┐                                           │    │
│  │   │  System Break   │                                           │    │
│  │   │     DP O(N²)    │                                           │    │
│  │   └─────────────────┘                                           │    │
│  │            │                                                    │    │
│  │            ▼                                                    │    │
│  │   ┌────────────────────────────────────────────────────┐        │    │
│  │   │       Scale Factor per System                      │        │    │
│  │   │   avail = sys_width - preamble[s] - gutter[s]      │        │    │
│  │   │   scale = avail / Σ ideal_i                        │        │    │
│  │   └────────────────────────────────────────────────────┘        │    │
│  │                                                                 │    │
│  │   Output: System breaks + scale factors                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Stage 4: ATOM POSITIONING (Fan-out per Measure)                │    │
│  │  Apply scale factors to atoms, compute absolute coordinates     │    │
│  │  actual_i = ideal_i × scale_factor                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Stage 5: SPANNERS & MARGINS (Fan-out per Musician)             │    │
│  │  Connecting elements + gutter decorations, compute sys heights  │    │
│  │  Adds courtesy accidentals in gutter space at system start      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Stage 6: PAGE BREAKING (Single)                                │    │
│  │  DP O(S²) over systems using actual heights from Stage 5        │    │
│  │  Output: Final page breaks, layout LOCKED                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
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
Connecting elements (ties, slurs, hairpins, beams, glissandi, ottava lines) do not participate in the distribution optimization. They are computed in Stage 5 after positions are finalized, and adapt to the determined geometry.

**Objective**:
Minimize global discomfort derived from system-local stack deviations from ideal proportions, while satisfying capacity constraints. The intent is that minimizing global discomfort produces layouts perceived as stable and professionally typeset.

**Architectural thesis**:
Taken as a whole, the measure distribution problem couples vertical alignment, horizontal spacing, system breaks, page breaks, and connecting element geometry into a single optimization. Ooloi's staged pipeline architecture decouples these concerns, collapsing the distribution problem into a form solvable by known algorithms (specifically, Knuth-Plass style dynamic programming). The algorithm is not new, and optimal breaking by dynamic programming has been applied to music before (see [Prior Art](#prior-art)); this ADR specifies how Ooloi's pipeline reduces its own problem to that form.

### Architectural Reduction

The apparent computational difficulty of music layout arises from coupling between multiple simultaneous concerns:

- Vertical alignment across staves (O(M²) per temporal position)
- Horizontal spacing within measures (dependent on symbol collisions)
- System and page distribution (combinatorial choices)
- Connecting element geometry (dependent on final positions)

Ooloi's pipeline architecture **decouples these concerns through staged computation**:

**Stage 1-2: Vertical Coordination**
Parallel collision detection produces measure stack metrics. Each of N stacks emerges with definitive width bounds: `min_width` and `ideal_width` for the measure's semantic content, plus `gutter_width` for additional system-start space.

**Stage 1 is width-complete**: It computes all horizontal space requirements for atoms, including lyrics, dynamics, articulations, and fixed-size anchors for spanners (e.g., "sffzpp<" letters, minimum crescendo wedge width). Stage 1 cannot compute full spanner geometry (that depends on final positions from Stage 4), but it must include all horizontal space the atom requires, including spanner attachment points.

**Stage 5 is height-complete**: After Stage 4 positions atoms, Stage 5 computes connecting element geometry and determines final vertical extent. System heights are definitive after Stage 5.

The vertical alignment problem is solved completely before distribution begins. This reduction is enabled by:
- Immutable data structures eliminating race conditions during parallel processing
- Rational arithmetic preserving exact proportions without floating-point accumulation
- STM coordination ensuring atomic reads of hierarchical musical structure

**Stage 3: System Breaking**
With vertical coordination complete, the problem reduces to:
- A 1-dimensional sequence of N scalar triples (min_width, ideal_width, gutter_width)
- Gutter subtraction for system-start stacks before scaling
- Capacity-constrained segmentation into systems using Knuth-Plass DP
- Cost function on width deviations
- No feedback from geometry to distribution logic
- Output: System break decisions and scale factors per system

**Stage 4: Atom Positioning**
Applies scale factors from Stage 3 to position atoms at their actual coordinates. Non-connecting visual elements (noteheads, accidentals, dynamics) are positioned using the computed scale factors. These elements do not influence distribution.

**Stage 5: Spanners and Margins**
Ties, slurs, and spanning attachments compute geometry based on finalized atom positions from Stage 4. They adapt to the determined layout rather than influencing it. For measures at system-start positions, Stage 5 adds graphical decorations (courtesy accidentals, tie continuations) within the reserved gutter space. Determines actual system heights from vertical extent of connecting elements.

**Stage 6: Page Breaking**
Uses Knuth-Plass DP over the system sequence with actual system heights from Stage 5. Produces final page breaks, completing the layout.

**Why this is architectural, not algorithmic**:
The reduction does not emerge from clever optimization techniques. It emerges from:
- Immutability preventing cascading state updates during iteration
- Semantic determinism (pitch identity, temporal ordering, accidental logic) established before layout begins
- Explicit stage boundaries preventing premature coupling
- Rational arithmetic eliminating error accumulation that would destabilize convergence

The result: the distribution problem becomes solvable by straightforward dynamic programming, whose result is guaranteed optimal for its cost function, given its inputs. The algorithm is not novel (see [Prior Art](#prior-art)).

### Optimization Characterization

The distribution problem exhibits structure that admits polynomial-time exact optimization of the reduced subproblem:

**Discrete break selection**:
Dynamic programming over the sequence of N measure stacks determines optimal system and page break points. For each potential break location, the algorithm evaluates whether remaining measures fit within capacity (after reserving preamble and gutter space) and computes resulting discomfort. Optimal substructure holds: optimal solution for measures 1..k combined with optimal solution for measures k+1..N yields optimal solution for 1..N.

Break selection relies on **separable system costs** and **optimal substructure**, not convexity. The dynamic programming algorithm computes the break configuration of least total cost for the discrete segmentation problem, under the stated cost function and given its inputs.

**What "optimal" means in this ADR**: *optimal* always means minimal total cost under the cost function specified here (the segment cost model below, plus any `break-penalty-fn` terms), given the metrics Stages 1–2 supply. It is a statement about that cost function and those inputs, not about layout quality in any wider sense: a different cost function, or different upstream metrics, would have a different optimum.

Complexity: O(N²) for system breaks, O(S²) for page breaks where S = number of systems.

**Continuous width allocation**:
Within each system determined by break selection, actual widths are allocated by **proportional scaling** (normalisation) of the space remaining after preamble and gutter reservation:

```
available = system_width - preamble[s] - gutter[s]
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

The DP selects breaks that minimize this total deviation. Scale factors < 1 indicate compression; > 1 indicates expansion. The preamble and gutter space are reserved but do not participate in cost computation—they are fixed overhead for system-start elements.

**Additive separable cost model**:
The cost structure operates at two levels:
- **Stack-level discomfort**: For feasible segments, each stack's discomfort depends only on its `ideal` and the system's `scale_factor`. The `min` value participates in feasibility checking; the `preamble` and `gutter` values participate in available space calculation; none of these participate in cost computation.
- **System-level cost**: Total discomfort for a system = Σ discomfort(stack_i) = `(scale - 1)² × Σ ideal_i²`
- **DP operates on system-level cost units**: The dynamic programming algorithm sums per-system discomfort values to compute global cost

This separability enables independent evaluation of candidate break points during dynamic programming.

**Page breaking: Second-order segmentation**:
Page breaking is treated as a **second segmentation pass over the system sequence, never interleaved with system breaking**. After optimal system breaks are determined, page breaks are computed independently via a second dynamic programming pass. The passes are separate because page breaking needs actual system heights, which exist only once Stage 5 has built the systems; optimising both together would have to work from estimated heights (see [ADR-0028 §Trade-offs](0028-Hierarchical-Rendering-Pipeline.md#trade-offs)).

**Deterministic outcomes**:
Given identical input (measure stacks with their bounds, system/page capacities), the algorithm produces identical output. This determinism arises from:
- Rational arithmetic (no floating-point nondeterminism)
- Immutable data structures (no timing-dependent state)
- Deterministic evaluation order (DP iterates t increasing, s decreasing; updates only on strict improvement)
- Normalisation-based allocation (no iterative solver convergence)

**Edit locality**: Preserving break decisions away from an edit, to minimise perceptual layout "jitter" during editing, is a goal. The DP does not provide it: a global optimum is not local. See [Edit Locality](#edit-locality).

## Decision

### Architectural Position

Measure distribution optimization (system breaking) is Stage 3 of the hierarchical rendering pipeline ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)). It receives measure stack metrics from Stages 1-2 and produces system break decisions and scale factors that Stage 4 uses for atom positioning.

The algorithm implements **capacity-constrained segmentation with proportional width allocation**:
1. Dynamic programming (Stage 3) determines optimal system break points, accounting for preamble and gutter space at system starts
2. Proportional scaling computes scale factors for width allocation within the available space (after preamble and gutter reservation)
3. Atom positioning (Stage 4) applies scale factors to compute actual positions
4. System heights (Stage 5) are computed from final geometry
5. Page breaking (Stage 6) uses a second DP pass over systems with actual heights

### Interface Contract

```clojure
;; Input from Stages 1-2 (computed by ADR-00XX, forthcoming):
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

The computation of width values is handled by upstream stages, and will be specified in a forthcoming ADR on horizontal spacing (ADR-00XX). If preconditions are violated, behavior is undefined.

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
- **Soft preservation**: `actual_width ≈ ideal_width` (proportionality maintenance)
- **No upper bound**: Measures may exceed ideal proportionally

### The Gutter: System-Start Width Delta

When a measure appears first on a system, it may require additional space for graphical decorations. This additional space is the **gutter**—a fixed-width region that follows the preamble (clefs and key signatures) and precedes the measure's scaled content.

System layout when measure s is first:

```
┌───────────┬──────────┬─────────────┬─────────────┬─────────────┐
│ PREAMBLE  │  GUTTER  │  measure s  │ measure s+1 │ measure s+2 │ ...
│  (fixed)  │ (fixed)  │  (scaled)   │  (scaled)   │  (scaled)   │
└───────────┴──────────┴─────────────┴─────────────┴─────────────┘
      ↑          ↑           ↑
      │          │           └─ actual[s] = ideal[s] × scale_factor
      │          │
      │          └─ gutter[s] (does not scale)
      │
      └─ preamble[s]: clef + keysig (does not scale)
```

**What the gutter accommodates:**
- Courtesy accidentals for tied-to notes at position 0 (graphical decorations, not semantic accidentals)
- Tie continuation arcs (visual portion of ties broken at system boundaries)

**Critical architectural property:** The gutter width is computed in Stage 1 from complete information about the measure's layout, including all semantic accidentals at position 0. If existing accidentals already provide sufficient space for the courtesy accidental, the gutter width is 0N—only the additional space needed beyond the measure's existing layout is reserved. Stage 3 knows exactly how much additional space each measure requires *before* making any distribution decisions. This lets Stage 3 find the distribution that is optimal for its cost function, given these inputs, without heuristics or iteration.

**What is NOT in the gutter:** All semantic accidentals are computed and positioned by [ADR-0035](0035-Remembered-Alterations.md) and are included in the measure's `min_width` and `ideal_width`. The gutter contains only graphical decorations added by Stage 5 for visual clarity at system boundaries.

**User control:** Users can configure when courtesy accidentals appear (`:system`, `:page`, or `:none`). This setting affects Stage 5 rendering, not Stage 3 distribution—the gutter is always reserved; the decorations are optionally rendered. See [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md) for details.

**Allocation with preamble and gutter:**

For a system containing stacks [s, t):
```
preamble = preamble[s]                       ;; Clef + keysig width (max across staves)
gutter = gutter[s]                           ;; Only first stack contributes
available = system_width - preamble - gutter ;; Remaining space for scaling
scale_factor = available / Σ ideal_i         ;; Scale factor for all measures

actual[i] = ideal[i] × scale_factor          ;; ALL measures scale the same
```

**Verification:**
```
preamble + gutter + Σ actual_i = preamble + gutter + available = system_width ✓
```

The preamble and gutter are not added to any measure's width—they are separate space that precedes the first measure's content. Stage 5 renders clefs/keysigs in preamble space and gutter decorations in gutter space when the measure appears at system start.

### The Preamble: System-Start Clef and Key Signature Width

Every system begins with clefs and key signatures for each staff. This **preamble** occupies fixed space that cannot be scaled:

```
preamble_width = max over all staves of (clef_width + keysig_width + spacing)
```

**Why uniform width**: All staves in a stack must have the same preamble width to maintain vertical alignment. Even if one staff has a simpler clef/keysig combination, it receives the same left margin as the most complex staff.

**On-demand computation**: Unlike gutter (precomputed in Stage 1), preamble width is computed during Stage 3 when evaluating candidate system breaks:
1. Query ChangeSets for active clef at the candidate measure's temporal position (per staff)
2. Query ChangeSets for active key signature at the same position (per staff)
3. Compute width for each staff: clef width + keysig width + spacing (before, between, after)
4. Take the maximum width across all staves in the stack

**Caching**: The (clef, keysig) combinations are highly repetitive across a score—a symphony might have only 3-4 distinct combinations across hundreds of measures. Caching by the combination tuple across staves provides near-instant lookup after the first computation. The cached value includes both the computed maximum width and the paintlists for rendering.

**Preamble vs Gutter**:
- **Preamble**: Structural (always present at system start), computed on-demand
- **Gutter**: Contingent (present only when tied-to notes require courtesy accidentals), precomputed in Stage 1

Both are fixed overhead subtracted before proportional scaling. Neither participates in cost computation.

**Postamble**: A system-end counterpart to these system-start reservations will be added: a **postamble**, fixed space at the end of a system for the cautionaries that precede a change taking effect at the start of the next system, such as a key signature or time signature change. It depends on where a system ends, as the preamble and gutter depend on where it starts. The formulas, the dynamic program and the code in this ADR will be updated to reserve it; until then they reserve system-start space only. The same pass makes explicit the column ratio of [Column-Level Feasibility](#column-level-feasibility) and the pair comparison of [Deterministic Tie-Breaking](#deterministic-tie-breaking).

### Width Allocation: Proportional Scaling

The primary width allocation strategy is **proportional scaling** (normalisation) applied to the space remaining after preamble and gutter reservation:

```
available = system_width - preamble[s] - gutter[s]
scale_factor = available / Σ ideal_i
actual_i = ideal_i × scale_factor
```

Both preamble and gutter are fixed overhead; neither participates in scaling.

This is **normalisation**, not optimisation:
- Deterministic formula with no degrees of freedom
- Preserves proportional relationships by construction
- Preamble and gutter space are reserved separately; neither scales
- No iteration, no convergence, no tuning parameters
- Predictable behavior: within a fixed break configuration, a change to one stack alters only its own system's scale factor
- Near-zero computational cost
- Clean architectural separation: DP selects breaks, allocation is derived

**Why this may be sufficient**:

1. **Proportionality preservation**: The scaling factor is identical for all measures, preserving ratios
2. **Gutter correctness**: System-start decorations occupy exactly their required space
3. **No heuristics**: Pure mathematical formula, deterministic outcome
4. **Speed**: O(K) per system with K â‰ˆ 10-20, essentially free
5. **Predictability**: Users can mentally predict system behavior
6. **Contained allocation**: Within a fixed break configuration, a change to one stack alters only its own system's scale factor

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

**Relationship to Knuth-Plass**:

Proportional scaling is the Knuth–Plass glue model in the special case where every glue's stretch and shrink are proportional to its natural width; see [Relationship to Knuth-Plass](#relationship-to-knuth-plass) for the mapping and the extensions it leaves available.

### Variable System Widths

**System width is per-system policy input**, not a page-wide constant. The allocation formula is parameterized by `system_width` for each system independently.

**Any system may have a different width**, including but not limited to:
- The final system of the piece or of a movement (commonly narrower to avoid excessive whitespace)
- Systems following explicit editorial break markings
- Systems constrained by margins or layout policy
- Systems accommodating graphical elements or marginal annotations

**What a width may depend on**: Stage 3 chooses system breaks before Stage 6 chooses page breaks, so a system's width is a function of the candidate segment `(s, t)` and of layout settings, never of the page the system will land on. Widths that depend on page position — a narrower last system on each page, widths that vary with a system's place on its page — are not available to Stage 3 (see [ADR-0028 §Trade-offs](0028-Hierarchical-Rendering-Pipeline.md#trade-offs)).

**Architectural implications**:

1. **Break selection (DP) operates on feasibility and cost** with whatever width is provided for each candidate system position. The algorithm does not assume uniform justification.

2. **Width allocation is parameterized per system** after breaks are chosen. The proportional scaling formula works identically regardless of system width.

3. **Determinism is preserved**: Given the same break configuration and per-system width policies, allocation produces identical results.

4. **No algorithmic complexity added**: The DP evaluates `(s, t)` segments against the width available for that system position, minus gutter. Proportional scaling applies the provided width directly.

**Per-system width policy**:

**Any system may have a distinct width policy** without special-casing, the final system included. Heterogeneous system widths need no additional mechanism because allocation is parameterized by width, not built around a fixed one.

**Expressivity without complexity**:

This architectural freedom enables:
- Natural handling of the final system of a piece or movement (avoid excessive stretch)
- Editorial control over system widths for specific musical reasons
- Adaptation to margins and layout settings
- Future extensions (e.g., systems of varying width for visual effect)

All while preserving the core property: **proportional scaling maintains rhythmic relationships within whatever width is provided**, with preamble and gutter space reserved for system-start elements.

### Alternative: Asymmetric Cost Optimization

An alternative approach would treat width allocation as a constrained optimization problem:

```
minimise Σ f(ideal_i - actual_i)
subject to: Σ actual_i = available, actual_i ≥ min_i
```

where `f` is asymmetric: compression toward `min_width` penalized more heavily than expansion beyond `ideal_width`.

This general formulation is broader than the Knuth–Plass extension of separate stretch and shrink per glue (see [Relationship to Knuth-Plass](#relationship-to-knuth-plass)). That extension keeps allocation in closed form within a system and segment evaluation O(1); the costs listed below apply to the general formulation.

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

### Page Breaking: Second Pass (Stage 6)

**Stage 6: Page Breaking** is a second segmentation pass over the system sequence, never interleaved with system breaking. It uses actual system heights computed by Stage 5:

```clojure
(defn find-page-breaks [systems page-height-fn]
  ;; Stage 6: Page Breaking
  ;; Same DP structure as Stage 3, different input:
  ;; - systems instead of stacks (from Stage 3 output)
  ;; - page-height-fn instead of system-width-fn
  ;; - system heights instead of widths (from Stage 5 output)
  ;; page-height-fn: (fn [start-sys end-sys] -> available-height)
  (find-optimal-breaks systems page-height-fn))
```

The separation exists because system heights are known only after Stage 5 has built the systems; a combined optimisation would have to work from estimated heights.

### Editorial Control Mechanisms

Users require control over layout decisions: forcing system breaks at specific points, preventing breaks within phrases, adjusting individual measure widths. These controls fall into two distinct categories requiring different architectural treatment.

**Constraints vs Preferences**:

- **Constraints** are inviolable: "This measure *must* start a new system"
- **Preferences** are costs: "Prefer breaking at rehearsal marks"

Modeling constraints as extreme penalties (e.g., cost = âˆž for forbidden breaks) conflates these categories and risks numerical instability. Ooloi handles them through separate mechanisms.

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

User adjustments to individual measure widths ("stretch this measure," "compress this passage") belong upstream of Stage 3, not within it. Stage 3 consumes width triples; editorial overrides modify these inputs:

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
2. **Stage 3** has *complete knowledge* of both scenarios (mid-system and system-start) for every measure
3. **Stage 3** computes the system breaks that are optimal for its cost function, given these inputs - not heuristic, not iterative
4. **Stage 4** positions atoms using scale factors from Stage 3
5. **Stage 5** adds graphical decorations based on actual positions and computes system heights
6. **Stage 6** computes the page breaks that are optimal for its cost function, using actual system heights

The gutter width is not "we might need space" - it's "this is exactly how much space the graphical decoration requires." Stage 3 doesn't guess. It has full information to make, in one pass, the decision that is globally optimal for its cost function.

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
- **Optimal**: Minimal cost under the stated cost function, given its inputs - an exact result for that function, not an approximation of it
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

The setting affects gutter computation in Stage 1, not Stage 3 logic. Stage 3 receives different gutter values and computes the optimal distribution for those values. Changing the setting produces a new optimal layout—automatically, deterministically, without iteration or manual correction.

**Architectural capability:** The complete-information architecture makes user-controllable courtesy accidental settings straightforward to implement—a feature that requires deterministic distribution to work without manual adjustment or layout jitter. Traditional architectures that lack complete information at distribution time cannot offer such settings without risking non-deterministic behavior or requiring iterative correction.

This demonstrates how complete information transforms potentially complex features into simple input variations. Settings that would otherwise require manual adjustment, iterative correction, or special-case handling become straightforward parameter changes to a general algorithm. The architecture enables the feature; the feature validates the architecture.

## Stability Guarantees

### Stage Boundary Enforcement

After Stage 3 system breaking and Stage 4 atom positioning complete, measure stack positions are **finalized**. No subsequent stage modifies distribution decisions. This architectural invariant prevents feedback loops where:
- Stage 5 connecting elements request more space
- Stage 3 redistribution invalidates Stage 5 geometry
- Mutual invalidation prevents convergence

### Connecting Element Adaptation

Ties, slurs, hairpins adapt their curves and control points to fit finalized atom positions. The vast majority of musical notation admits this adaptation without requiring additional space.

For measures at system-start positions, Stage 5 adds graphical decorations within the gutter space:
- Courtesy accidentals for tied-to notes (visual aids, not semantic accidentals)
- Tie continuation arcs

This is **selection, not computation**. The space was pre-reserved; Stage 5 renders decorations into it.

### Deterministic Tie-Breaking

When break configurations produce identical discomfort (a "tie" in cost, not a musical tie), the configuration is chosen by a secondary cost, compared only when the primary costs are equal. The secondary cost is separable — a sum of per-break terms — and differs for any two distinct configurations: for example, the sum over the configuration's breakpoints `b` of `2^−b`, which, like a binary fraction, is different for any two different sets of breakpoints. Costs are compared as pairs (primary, secondary), lexicographically. Pairs add componentwise and the lexicographic order is preserved under addition, so the DP remains exact for the pair, and the optimum is unique.

Because the choice is a property of the configuration rather than of the order in which a procedure meets candidates, every procedure that minimises the pair — the full pass, and [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit) from forward and backward tables — returns the same configuration. The result is deterministic across runs, platforms and editing histories. Ties are expected rather than exceptional: under rational arithmetic, identical measures produce identical costs.

The Stage 3 code sketch still resolves ties by evaluation order (`t` increasing, `s` decreasing, update only on strict improvement), which a backward table cannot reproduce. It is updated to the pair comparison in the same pass that adds the postamble and the column ratio.

If editorial preferences are needed (e.g., prefer structural boundaries), they can be encoded via the optional `break-penalty-fn` parameter, which shifts costs rather than relying on tie-breaking.

### Edit Locality

Edit locality — preserving break decisions away from an edit, so that the layout does not visibly "jitter" while the user works — is a goal. The algorithm does not provide it.

The break configuration Stage 3 selects is a global optimum, and a global optimum is not local. An edit to stack `m` can change system breaks after `m` and before `m`: the DP finds the configuration of least total cost for the whole sequence, and the chosen breaks are traced backwards from its end, so a change of cost at `m` can alter the choice of breaks anywhere in the score. Nor is there any guarantee that, beyond some point, the new break decisions coincide with the previous solution again.

What holds is that recomputation is cheap. Stack metrics are cached (see [Caching and Incremental Recompute](#caching-and-incremental-recompute)), so re-running Stage 3 after an edit is scalar arithmetic over the cached metrics: collision detection, atom formation and vertical reconciliation are not repeated for any stack whose content did not change.

The mechanisms that provide it are specified under [Stability and Quality Extensions](#stability-and-quality-extensions): exact re-optimisation after an edit keeps recomputation to about K² evaluations, and a stability term and frozen systems keep breaks where they were.

### Caching and Incremental Recompute

Stage 3 operates over a sequence of measure stacks whose width triples are **already finalized upstream**. In addition, Ooloi caches the following per **measure stack**:

* `min_width` — hard lower bound for measure content (from Stage 1–2)
* `ideal_width` — proportional target for measure content
* `gutter_width` — additional space for system-start decorations
* `actual_width` — realized width after Stage 3-4 (scale factors applied by Stage 4)

This cache is authoritative at the Stage 3-4 boundary: Stage 3 consumes `(min, ideal, gutter)` and produces break assignments and scale factors; Stage 4 applies scale factors to produce `actual` positions; Stage 5 consumes `actual` and never influences Stages 3-4.

#### Cached Invariants

For each stack `i` under a fixed system assignment:

* `actual_width_i ≥ min_width_i`
* `actual_width_i = ideal_width_i × scale_factor(system)` unless clamped
* `scale_factor(system) = available / Σ ideal_width_j` where `available = system_width - preamble[s] - gutter[s]`

The cache additionally implies that per-stack metrics are **stable across edits** unless the edited content is in that stack.

#### Incremental Update Consequences

Edits affect Stages 3-4 only through changes to stack metrics:

1. **Local edit**: modifying or inserting notation in a measure updates only the affected stack's upstream-derived width values.

2. **Fast path (break stability)**: Stage 3 is re-run over the cached metrics. If the resulting break configuration is unchanged, then:

   * the scale factor of the system containing the edited stack may change, and with it the `actual_width` of every stack in that system; no other system's scale factor changes
   * Stage 4 repositioning and Stage 5 regeneration are confined to that system and to connecting elements that touch it
   * no other systems require recomputation

3. **Ripple path (break changes)**: if the re-run produces a different break configuration:

   * unaffected stacks retain cached metrics, so the DP runs over the same scalar input as before except at the edited stack, and no upstream work is repeated for them
   * the changed breaks are not confined to the neighbourhood of the edit: they may occur after it and before it, and nothing guarantees that the new configuration coincides with the previous one beyond any particular point (see [Edit Locality](#edit-locality))
   * `actual_width` recomputation is confined to systems whose membership or width policy changed

#### Practical Complexity

With cached stack metrics, Stage 3 runtime is dominated by DP over **scalars**, not by collision/layout work. Edit-time recomputation is typically:

* **O(size of edited measure)** for upstream recalculation of the edited stack
* plus a Stage 3 re-run: **O(N×K)** scalar segment evaluations over cached metrics, with no collision or layout work for any unchanged stack
* plus **O(K)** (where `K ≈ measures/system`) allocation for each system whose membership or scale factor changed

#### Relationship to Edit Locality

This caching strategy makes recomputation after an edit cheap: re-running Stage 3 costs scalar arithmetic over cached aggregates, and no upstream work is repeated for stacks whose content did not change. It does not make the result local. A re-run may change breaks anywhere in the score; the mechanisms that keep break decisions stable are specified under [Stability and Quality Extensions](#stability-and-quality-extensions).

## Stability and Quality Extensions

The baseline specified above is complete and exact on its own. The mechanisms in this section extend it for stability under editing and for layout quality. Each is specified here with its cost and its effect on exactness. Which are enabled by default, and with which parameters, is decided during implementation by the empirical validation in [Consequences](#consequences) — simple scores, then complex ones, then Elektra — against the criteria below; the one exception is [Column-Level Feasibility](#column-level-feasibility), which corrects the baseline itself.

### Evaluation Criteria

Visual judgement, together with measurements taken on the same scores:

- The distribution of per-system scale factors: minimum, maximum, variance
- The difference between adjacent systems' scale factors
- The number of systems
- How many breaks an edit changes (stability)
- Running time

### Exact Re-optimisation After an Edit

Two tables are kept with the layout: a forward table `F(s)`, the least cost of segmenting stacks `[0, s)` (the DP's `best`), and a backward table `G(t)`, the least cost of segmenting `[t, N)`, computed the same way from the end, with a `next` pointer per entry.

An edit to stack `m` changes the cost of exactly the segments that contain `m`. `F(s)` for `s ≤ m` and `G(t)` for `t > m` involve no such segment and are unchanged. Every segmentation has exactly one segment containing `m`, and by optimal substructure the parts either side of it are independently optimal, so the new optimum is

```
min over feasible [s, t) with s ≤ m < t of   F(s) + cost′(s, t) + G(t)
```

— at most about K² evaluations. Breaks are recovered through `prev` on the left and `next` on the right. The result is exact.

- **Range edits**: an edit that changes every segment intersecting a range `[a, b]` — several adjacent stacks, or a width that depends on where a system ends (see [End-of-System Width](#end-of-system-width)) — is re-optimised by a forward DP over the window `[a − K, b + K]`, seeded from `F` and closed with `G`: O((b − a + 2K) × K).
- **Full passes**: a change of clef or key signature alters the preamble of every later system, and a key signature change alters the accidentals of later measures ([ADR-0035](0035-Remembered-Alterations.md)); a change to forced or prevented breaks changes the problem's partition. These re-run Stage 3 in full.
- **Refresh**: after an edit, `F(s)` for `s > m` and `G(t)` for `t ≤ m` are stale. They are recomputed off the edit's critical path, O(N×K), or lazily: the next edit at `m₂` needs only the part of each table between the two edits.
- **Backward early termination** needs the mirror of the width assumption under [Key Implementation Notes](#key-implementation-notes): for fixed `t`, the system width must not increase as the end moves right. Where it does, the backward pass scans every end — slower, still exact.
- **Ties** are resolved as [Deterministic Tie-Breaking](#deterministic-tie-breaking) specifies, which both the full pass and re-optimisation apply, so a score produces the same layout however its edits arrived.

### Stability Term

`cost″(s, t) = cost(s, t) + λ × D(s, t)`, where `D` measures departure from an anchored previous layout — for example 1 when `s` is not a break of the anchor, or when `[s, t)` is not one of its systems. `D` depends only on the segment, so the objective stays separable and the result is exact for `cost + λD`. That objective is deliberately not the pure proportionality cost: `λ` sets how much quality is exchanged for stability.

- **The anchor is layout data**: persisted with the layout, and identifying breaks by measure identity rather than by index, so that it survives measure insertion and deletion and the layout remains regenerable from semantics and layout data.
- **Anchor policy sets the refresh cost**: while the anchor is fixed, [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit) applies unchanged, with `F` and `G` computed under the anchored objective. Moving the anchor changes `D` for any segment whose relation to it changed, so both tables are recomputed, O(N×K). When the anchor moves — after every edit, on save, on an explicit reflow — and the value of `λ` are evaluated.

### Frozen Systems

A system the user locks keeps its breaks: its start and end become forced breaks and the breakpoints inside it are prevented, using the pre-segmentation and grouping of [Editorial Control Mechanisms](#editorial-control-mechanisms). A freeze is a constraint, so the result is exact over the configurations that respect it, and it reduces work by partitioning the problem; each partition keeps its own `F` and `G`. Freezes are anchored to measure identity. What happens when an edit inside a frozen system makes it infeasible — refusing the edit, releasing the freeze, or accepting an overfull system — is not yet specified.

### Consistency Between Adjacent Systems

Two exact forms, each needing more DP state than the baseline:

- **Fitness classes**: each system is classified by its scale factor into a small number of bands, and adjacent systems whose bands differ by more than one are penalised. One node per (breakpoint, class): about ×4 state and work with four classes.
- **Squared difference of adjacent scale factors**: `μ × (scale(s, t) − scale(r, s))²`, where `[r, s)` is the preceding system. The state is the (start, end) of the last system: N×K states with K transitions each, O(N×K²) — about 180,000 evaluations at N = 800, K = 15.

Keeping one node per breakpoint while adding an adjacency term makes the result an approximation (see [Comparison](#comparison)). The term carries a known risk, recorded in a comment in LilyPond's `lily/gourlay-breaking.cc`: where music becomes gradually denser, a uniformity requirement drives cramped lines to become more cramped, because the step from a cramped line of three measures to a loose line of two is large. Scores of gradually changing density are part of the evaluation of either form.

### Badness, Ending Classes and Looseness

- **Cubic badness** in place of the quadratic cost: Knuth–Plass rates a line by badness roughly cubic in its adjustment ratio, which punishes large deviations harder and small ones less and is not weighted by content. O(1) per segment; exact. Evaluated by comparing the break choices of both costs on the same scores.
- **Ending classes**: the class idea applied to the kind of boundary a system ends on — a phrase end, a rehearsal mark, the end of a movement. A penalty that depends only on the boundary needs no classes; `break-penalty-fn` already expresses it. Classes are needed only when the cost depends on the preceding system's ending, and multiply the state by the number of kinds.
- **Looseness**: a user control asking for more or fewer systems than the optimum. The system count joins the state, one node per (breakpoint, count): O(S×N×K), which is O(N²) with S ≈ N / K — about 640,000 evaluations at N = 800. Exact.

### Fitting: Proportional or Even

Two steps are kept apart. **Ideal spacing** is derived within each measure from the durations of its notes and tuplets, in Stage 1; duration enters there and only there. **Fitting** puts measures onto systems and scales their ideal widths to fill each system; Stage 3 decides it and Stage 4 applies it. The breaking decision depends on the fitting rule, because the DP judges each candidate system by its feasibility and cost under that rule. A fitting rule weighted by duration would count duration twice, and is not a candidate.

In Knuth–Plass terms the fitting rule is the choice of each glue's stretch and shrink ([Relationship to Knuth-Plass](#relationship-to-knuth-plass)):

- **Proportional** — the baseline: stretch and shrink proportional to natural width, so one scale factor per system. Ratios between ideal widths, and with them the rhythmic hierarchy Stage 1 encoded, are kept.
- **Even**: every column gap has the same stretch and shrink, so surplus and deficit are shared equally among columns. Ratios are not kept, and the hierarchy flattens.
- **Shrink bounded by the minimum**: each column gap shrinks by up to `ideal − min`, with proportional stretch. An incompressible gap has zero shrink and the others absorb the compression, which removes the baseline's rigidity — under `scale ≥ max(min_i / ideal_i)` one incompressible stack blocks compression of its whole system. Ratios are kept under stretching and not under compression.

Each is linear glue: a stack's stretch and shrink are closed-form sums, so feasibility and a least-squares cost are O(1) from prefix sums, and the DP stays exact. Under the last, feasibility is `Σ min ≤ available` (adjustment ratio `r ≥ −1`), which keeps every column gap at or above its minimum. Under anything but the baseline, Stage 4 applies the system's adjustment ratio through each gap's own glue rather than one scale factor to every atom. The evaluation covers systems containing one dense or incompressible measure among sparse ones, preservation of rhythmic hierarchy, consistency in complex rhythmic contexts, and the proportionality each rule gives up.

### Column-Level Feasibility

Under the baseline, one scale factor is applied to every column gap within every stack. A column gap stays at or above its own minimum only if `scale ≥ min_gap / ideal_gap` for that gap, so the ratio the sufficient feasibility condition takes for a stack is the **largest column ratio within it**, `max_j(min_gap_j / ideal_gap_j)`, not the stack's `min / ideal`. The two coincide only when every column of a stack is equally compressible; otherwise a stack can pass a stack-level check while one of its columns collides. Wherever this ADR writes `max(min_i / ideal_i)` in the feasibility condition, `min_i / ideal_i` is the stack's largest column ratio. `Σ min_i ≤ available` remains the necessary condition, with `min_i` the sum of the stack's minimum gaps. The formulas, interface contract and code are updated to carry the column ratio explicitly in the same pass that adds the [postamble](#the-preamble-system-start-clef-and-key-signature-width).

### End-of-System Width

The postamble — courtesy clefs, key signatures and time signatures at the end of a system, before a change taking effect at the start of the next — is a width `tail[t]` depending only on where the segment ends: `available = system_width − preamble[s] − gutter[s] − tail[t]`. It is fixed overhead outside the cost, O(1) per segment, constant across the inner loop for fixed `t`, and keeps the result exact. An edit that changes it is a range edit for re-optimisation. It is added to this ADR's formulas and code in a pass of its own (see [The Preamble](#the-preamble-system-start-clef-and-key-signature-width)).

### Joint System and Page Breaking

The design separates system breaking from page breaking ([ADR-0028 §Trade-offs](0028-Hierarchical-Rendering-Pipeline.md#trade-offs)). The alternative is for Stage 3 to emit the best breaking for each system count — one node per (breakpoint, count), O(S×N×K) — and for Stage 6 to choose among them, as LilyPond's optimal page breaker does. Stage 6 would then need the heights of systems that only exist for the breaking Stages 4 and 5 actually build. The ways of supplying them each give something up:

- **Estimated heights**, as LilyPond's "pure" heights are: the breaking is chosen on estimates, and the real heights may not match — a page may overflow, or a better choice may have existed. Correcting it by re-running Stage 6 in a way that can change the breaking is iteration, which the pipeline excludes.
- **Every candidate built**: Stages 4 and 5 run for each candidate breaking, so Stage 6 chooses on real heights. Exact and unidirectional, but Stage 5 runs once per candidate; limited to a few counts around the optimum (`S − 1`, `S`, `S + 1`), it multiplies that work by the number of candidates.
- **Separation**, the current design: system breaks are chosen without pages in view, and page turns can only use system boundaries Stage 3 produced.

Determinism holds under all three. Adopting either of the first two changes the ADR-0028 trade-off as well as this ADR, and is decided by the same evaluation.

## Implementation

### Core Dynamic Programming Structure (Stage 3)

The following implementation performs **Stage 3: System Breaking** - distributing measure stacks across systems:

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
       ;; Each step leftwards extends the segment by exactly one stack (stack s),
       ;; so max(min_i / ideal_i) over the segment is carried forward in O(1)
       ;; rather than rescanned
       (loop [s (dec t)
              max-ratio 0N]
         (when (>= s 0)
           (let [max-ratio (max max-ratio (/ (get-in stacks [s :min])
                                             (get-in stacks [s :ideal])))
                 system-width (system-width-fn s t)
                 ;; Reserve preamble and gutter space for system-start stack
                 preamble (get-preamble-width stacks s)  ;; On-demand, cached by clef/keysig combo
                 gutter (get-in stacks [s :gutter] 0N)
                 available (- system-width preamble gutter)
                 ;; Check if measures can fit in remaining space
                 total-min (- (nth min-prefix t) (nth min-prefix s))]

             ;; Early termination: stop once the minimums exceed the full system width.
             ;; This bound only grows harder to meet as s moves left; `available` does
             ;; not, because preamble[s] and gutter[s] vary with s
             (when (<= total-min system-width)

               ;; Necessary: minimums fit after reserving preamble and gutter
               ;; Sufficient: scale factor must not require clamping
               (let [total-ideal (- (nth ideal-prefix t) (nth ideal-prefix s))
                     scale-factor (/ available total-ideal)
                     feasible? (and (<= total-min available)
                                    (>= scale-factor max-ratio))]

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
                       (assoc! prev t s)))))

               (recur (dec s) max-ratio))))))
     
     ;; Return results
     {:breaks (reconstruct-breaks (persistent! prev) n)
      :cost (nth best n)})))
```

**System-start reservations explained:**

For each candidate segment [s, t):
1. Get `preamble = preamble[s]` (max clef+keysig width across all staves, computed on-demand and cached)
2. Get `gutter = gutter[s]` (only first stack in system contributes)
3. Compute `available = system_width - preamble - gutter`
4. Check feasibility: `Σ min_i ≤ available`
5. Compute scale factor: `available / Σ ideal_i`
6. Cost is based on scalable content deviation only; preamble and gutter are fixed overhead

Both preamble and gutter are reserved space that does not participate in scaling or cost computation.

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

;; Narrower final system of each movement
(fn [s t] (if (movement-end? t) final-width standard-width))

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
5. **Fast**: O(K) where K â‰ˆ 10-20, essentially zero cost
6. **Predictable**: Users can mentally predict allocation behavior
7. **Contained**: Within a fixed break configuration, a change to one stack alters only its own system's allocation

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

**Preamble and Gutter Handling**:

Each stack carries `:gutter` (default 0N). Preamble width is computed on-demand and cached by (clef, keysig) combination. Only the first stack in a system contributes:
- Reserved from system width before scaling: `available = system_width - preamble - gutter`
- Does not scale—it is fixed space for graphical decorations
- Does not participate in cost computation
- Returned separately from `allocate-widths` for rendering coordination

**Baseline: Proportional Scaling (Normalisation)**:

The implementation uses proportional scaling as the primary width allocation strategy. This is normalisation (deterministic formula), not optimisation (iterative solver):

- **Formula**: `actual_i = ideal_i × (available / Σ ideal_i)`
- **Preamble and gutter reservation**: `available = system_width - preamble[s] - gutter[s]`
- **Preserves proportionality**: All measures scale by same factor
- **No degrees of freedom**: Allocation is purely derived from break choice
- **Fast**: O(K) per system, essentially zero cost
- **Predictable**: Mental model matches implementation
- **Contained**: Within a fixed break configuration, a change to one stack alters only its own system's allocation

This approach may prove entirely adequate. If empirical testing reveals systematic visual problems, asymmetric optimisation can be considered, but complexity should not be added speculatively.

**Extension Point: break-penalty-fn**:

The algorithm accepts an optional `break-penalty-fn` parameter with signature `(fn [stacks s t] -> ratio)`. It returns additional cost for selecting segment `s..t-1` as a system. The default `(constantly 0N)` produces purely geometric optimization. This ADR does not specify any concrete penalty functions; editorial policy is a separate concern that can evolve independently of the core algorithm.

**Feasibility Check**:

The DP uses a two-part feasibility check:
1. **Necessary**: `Σ min_i ≤ available` (minimums must fit after reserving gutter) - O(1) via prefix sums
2. **Sufficient for proportional scaling**: `scale_factor ≥ max(min_i / ideal_i)` - O(1) per segment, using a running maximum

The second condition ensures the scale factor is large enough that no stack requires clamping. Without it, proportional scaling could produce allocations below minimums.

The running maximum works because the inner loop visits segments in a fixed order: for fixed `t`, `s` decreases from `t-1`, so each candidate segment `[s, t)` is the previous one extended by the single stack `s`. The maximum over the new segment is therefore `max(previous maximum, min_s / ideal_s)`, one comparison per step. Rescanning the segment instead would cost O(K) per segment and O(N×K²) in total.

**Segment Evaluation**:

Each candidate segment (s, t) requires:
1. **Gutter lookup**: O(1) to get `gutter[s]`
2. **Feasibility**: O(1) to extend the running `max(min_i / ideal_i)` by stack `s`
3. **Cost**: O(1) closed-form computation using prefix sums

Every per-segment step is O(1). Cost computation uses precomputed `ideal-sq-prefix`; feasibility uses the running maximum carried by the inner loop.

**Rational Arithmetic Throughout**:
- All numeric literals use `N` suffix: `0N`
- Preserves exact arithmetic, no floating-point contamination
- Ensures deterministic outcomes across platforms
- Ratio growth managed through Clojure's automatic normalization (gcd)

**nil Semantics for Infinity**:
- `nil` represents unreachable positions (cost = âˆž)
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
- The inner loop stops once `total-min` exceeds the full `system-width`
- Monotonicity: as `s` moves left, `total-min` increases (more stacks) and, under the assumption below, `system-width` does not increase; once the minimums exceed the full system width, they do so for every earlier `s`
- The bound is the full system width, not `available`. Since `available = system_width - preamble[s] - gutter[s]` varies with `s`, a segment that fails against `available` because of a wide preamble or gutter at `s` can be followed by a feasible segment at `s-1`. Preamble and gutter are non-negative, so `available ≤ system_width`, and every feasible segment satisfies the full-width bound
- The check against `available` is made at each step, as the necessary feasibility condition
- Reduces effective complexity to O(N×K), where K is the largest number of consecutive stacks whose minimums fit within a full system width (K ≈ 15 typical)
- **Assumption**: `system-width-fn` must be non-increasing as `s` moves left (for fixed `t`). This holds for typical policies (uniform width, narrower final system). If violated, early termination is invalid and the algorithm must check all `s` values.

**Complexity Analysis**:

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Precomputation | O(N) | Prefix sums for min, ideal, and ideal² |
| Outer loop | O(N) | Each position once |
| Inner loop | O(K) typical | Early termination once minimums exceed the full system width |
| Gutter lookup | O(1) | Per-segment constant time |
| Feasibility check | O(1) | Running max(min_i / ideal_i), extended by one stack per step |
| Cost computation | O(1) | Closed form using prefix sums |
| **Total (worst case)** | O(N²) | Without early termination |
| **Total (typical)** | O(N×K) | With monotone width policy, K ≈ 15 |
| Page breaking | O(S²) | S = number of systems |

At N=800 measures, K=15: ~12,000 segment evaluations, each O(1) for both feasibility and cost.

## Relationship to Knuth-Plass

This section defines the relationship once. Elsewhere in this ADR, "Knuth-Plass" is shorthand for it.

**Classification**: Stage 3's baseline is an optimal-segmentation dynamic program in the Knuth–Plass family (Knuth and Plass, 1981; see [Prior Art](#prior-art)). Its cost is least-squares deviation from proportional widths, evaluated in closed form from prefix sums (see [Segment Cost Computation](#segment-cost-computation)).

**Mapping onto the Knuth–Plass primitives**:

| Knuth–Plass | Stage 3 |
|-------------|---------|
| Box | Atom |
| Natural width of glue | Ideal distance between atom origins |
| Shrinkability of glue | `ideal − min` for the gap |
| Legal breakpoint | Legal break position, normally a barline |
| Line length | System width less preamble and gutter |
| Adjustment ratio of a line | Determines the scale factor of a system (below) |

**Proportional scaling as a special case**: Proportional scaling is the case in which every glue's stretchability and shrinkability are proportional to its natural width. The Knuth–Plass adjustment ratio then gives one uniform scale factor per system: with stretch `c × natural` for every glue, a line of adjustment ratio `r` sets every gap to `natural × (1 + r × c)`, which is `ideal_i × scale_factor` with `scale_factor − 1 = r × c`. In the baseline, each gap's `min` enters only as a floor on that uniform scale — the feasibility condition `scale_factor ≥ max(min_i / ideal_i)` — rather than as a per-gap shrinkability.

**Extensions available**: Each Knuth–Plass feature is therefore an available extension of the baseline, not a departure from it:

- Cubic badness in place of the quadratic cost
- Separate stretch and shrink, with `ideal − min` as each gap's shrinkability (see [Alternative: Asymmetric Cost Optimization](#alternative-asymmetric-cost-optimization))
- Fitness classes, for example for different kinds of system endings
- Penalties at breakpoints, through the existing `break-penalty-fn`
- Looseness

**The scalar reduction is preserved**: Summing each stack's glue keeps the reduction to one scalar tuple per stack. Prefix sums of natural width, stretch and shrink give O(1) segment cost and O(1) feasibility, as they do for the baseline.

**Exactness and DP state**: The baseline's cost has no term linking adjacent systems, so one DP node per breakpoint is enough for the result to be exact. Cubic badness, separate stretch and shrink, and breakpoint penalties keep that. Two extensions need more state:

- **Fitness classes**: a cost that depends on the class of the preceding system is exact only if the DP keeps one node per (breakpoint, fitness class), which multiplies the work by the number of classes. Keeping a single node per breakpoint with such a term makes the result an approximation.
- **Looseness**: aiming for a number of systems other than the optimal one requires the number of systems so far to be part of the state, one node per (breakpoint, system count).

**Adoption**: Whether any extension is adopted is decided during implementation, by the empirical validation this ADR specifies under [Future Considerations](#consequences) (simple scores, then complex ones, then Elektra). The extensions are specified, with their costs and evaluation criteria, under [Stability and Quality Extensions](#stability-and-quality-extensions); none is enabled by this ADR.

Both problems share the mathematical structure enabling polynomial-time exact optimization of the reduced subproblem:
- 1-dimensional sequence with scalar preferences
- Capacity-constrained segmentation
- Separable cost function with optimal substructure
- Convex cost model for within-segment allocation

**Explicit non-claim**:
This approach does **not** solve "general music engraving" or "optimal notation layout for all musical structures." It solves the **measure distribution subproblem** after architectural reduction has transformed it into tractable form.

The reduction is what matters: by solving vertical coordination, symbol collision, and semantic determinism in earlier stages, the distribution problem becomes structurally similar to paragraph breaking. This is not algorithmic cleverness finding a better heuristic—it is architectural separation creating a problem formulation where exact optimization applies.

**What the natural widths mean**:

`ideal_width` encodes rhythmic proportionality, which has musical meaning. Choosing the proportional special case is what preserves it: all measures in a system share one scale factor, so their ratios hold by construction. Graphical decorations (gutter content) and the preamble are fixed overhead, outside the scaled glue.

Under the separate-stretch-and-shrink extension, compression toward `min_width` would cost proportionality, and the choice of shrinkability would express how much. Whether that extension is needed remains subject to the empirical validation described above.

**Architectural prerequisites**:

The Knuth-Plass algorithm solves a specific problem formulation: capacity-constrained segmentation of a 1-dimensional sequence with separable costs. Music layout, naively formulated, is not this problem—it couples vertical alignment, collision detection, semantic decisions, and geometry into mutual dependencies.

Ooloi's pipeline architecture transforms the problem. By the time Stage 3 executes:
- Vertical coordination is complete (Stages 1-2)
- Collision boundaries are determined (Stage 1)
- Semantic decisions (accidentals, beaming) are resolved (ADR-0035)
- Gutter requirements are computed (Stage 1)
- Connecting elements are deferred (Stage 5)

What remains is the Knuth-Plass problem formulation, over measure-stack scalars.

**Why the DP runs once per pass**:

Knuth-Plass itself is efficient—O(N²) in the general case, O(N×K) with early termination. In Ooloi it runs once per layout pass, on inputs that do not change while it runs. Three preconditions of the pipeline provide this:

- **Stable scalar inputs**: Stage 3 receives stable scalar inputs from completed upstream stages. Every width it consumes is final before it begins.
- **No feedback between stages**: No later stage changes Stage 3's inputs, so the DP never has to be re-run while geometry settles.
- **Immutability**: Immutability enables precise cache invalidation: edits affect only the stacks whose upstream metrics actually changed, not the entire sequence.

At N=800 measures with K=15 measures per system, a pass is approximately 12,000 segment evaluations, each O(1) scalar arithmetic.

The pipeline stages, immutable data structures and rational arithmetic provide both the problem formulation and the conditions under which the DP runs once per pass.

## Prior Art

Stage 3 draws on two lineages: optimal breaking by dynamic programming, and a model of horizontal space that separates fixed width from scalable width.

### Optimal Breaking by Dynamic Programming

- **Knuth and Plass**, "Breaking Paragraphs into Lines", *Software—Practice and Experience* 11 (1981), pp. 1119–1184. Optimal line breaking of paragraphs by dynamic programming over legal breakpoints; the family Stage 3 belongs to.

- **Hegazy and Gourlay**, "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, Department of Computer and Information Science, The Ohio State University (1987); also in *Proceedings of the International Conference on Electronic Publishing, Document Manipulation and Typography*, Nice, April 1988, ed. J. C. van Vliet, Cambridge University Press. It is cited here for its existence and its place in this lineage: it is the report that LilyPond's first line breaker names as its source. This ADR describes nothing of its contents.

- **LilyPond**:
  - `lily/gourlay-breaking.cc`, by Han-Wen Nienhuys, is present from release 0.1.1 (August 1997) until September 2006. From release 1.1.32 (February 1999) its comment reads: "This algorithms is adapted from the OSU Tech report on breaking lines." What follows describes LilyPond's code, not the report, which may differ from it.
    - In both the 0.1.1 and the 2006 revisions it is a dynamic program over legal breakpoints that keeps one best predecessor per breakpoint. For each breakpoint it tries lines ending there, extending them backwards one breakpoint at a time until a line becomes infeasible, and traces the chosen breaks back from the end.
    - In release 0.1.1 the cost of a configuration is the sum of each line's spacing energy, which is separable. The search is bounded: lines longer than a set number of measures, and lines whose energy exceeds a set bound, are not considered, and every candidate line is first evaluated with an approximate spacing solution; a candidate whose approximate energy cannot improve on the best found is skipped, and the exact spacing is computed only for a remaining candidate whose approximate solution fails its constraints. Where these bounds or the approximate screening exclude a line, the result can differ from the optimum of the cost.
    - In its 2006 form it solves the spacing problem for every candidate line, with no measure or energy bound. A line's demerits are |force| + |previous force − force| + break penalty, with a large penalty for a line that cannot satisfy its spacing constraints. The middle term depends on the previous line, and is evaluated against the single predecessor stored for the line's starting breakpoint; the result is therefore not guaranteed to minimise total demerits.
  - `lily/constrained-breaking.cc`, by Joe Neeman, is added in February 2006 and replaces it; `gourlay-breaking.cc` is removed in September 2006. It runs a dynamic program over (number of systems, breakpoint), with demerits force² + (previous force − force)² + break penalty, and solves a spring-and-rod spacing problem for every candidate line (`get_line_forces` in `lily/simple-spacer.cc`). It keeps one node per (number of systems, breakpoint), so the adjacency term is again evaluated against a single stored predecessor; with ragged-right lines the demerits reduce to force² plus break penalty, which is separable, and the result is then the exact optimum of that cost.
  - It works together with an optimal page breaker (`lily/optimal-page-breaking.cc`) and a page-turn breaker (`lily/page-turn-page-breaking.cc`). Page decisions use "pure" heights (`lily/constrained-breaking.cc`), which LilyPond's Contributor's Guide describes as estimates made before line breaking.

### The Space Model

- **MuTeX**: Andrea Steinbach and Angelika Schofer, *Automatisierter Notensatz mit TeX*, master's thesis, Rheinische Friedrich-Wilhelms Universität, Bonn, 1987. Limited to a single staff; it used TeX glue to control horizontal spacing and justification, and so TeX's own line breaking.

- **MusicTeX**: Daniel Taupin, around 1991. It adds multiple staves, as a single-pass system.

- **MusiXTeX**: Daniel Taupin, Ross Mitchell and Andreas Egler, "MusiXTeX : L'écriture de la musique polyphonique ou instrumentale avec TeX", *Cahiers GUTenberg* 21 (1995), pp. 107–113.
  - The MusiXTeX manual, §1.3.1, explains why TeX glue fails for music: a line holds far fewer bars than a line of text holds words, so treating each bar as a word leaves gaps before the bar rules.
  - It divides horizontal space into *hard* space (bar rules, clefs, key signatures), which is fixed, and *scalable* space, defined in multiples of one spacing unit, `\elemskip`. One value of `\elemskip` is computed per line, so that the scalable space fills what the hard space leaves.
  - This corresponds to ADR-0037's split: the preamble and gutter are hard space, ideal widths are scalable space, and `\elemskip` per line is the scale factor per system. ADR-0037 treats only system-start material as hard; bar rules within a system scale with the measures.
  - Its breaking pass, `musixflx` (Ross Mitchell, 1992–1997), sets a target number of lines from the total width divided by the line width, then fills lines one at a time, adding bars until a line overflows. It is greedy, not optimal.

### Comparison

Each breaker is described against its own cost: whether the result it returns is guaranteed to be the minimum of the cost it defines.

| Breaker | Search | Cost | Exact for its own cost |
|---------|--------|------|------------------------|
| `musixflx` (MusiXTeX) | Target line count, then lines filled one at a time | None; `\elemskip` is set to fill each line | No: greedy |
| LilyPond `gourlay-breaking.cc`, 1997 | DP, one node per breakpoint | Sum of line energies (separable) | Where its measure limit, energy bound and approximate screening do not exclude the optimal line |
| LilyPond `gourlay-breaking.cc`, 2006 | DP, one node per breakpoint | \|force\| + \|previous force − force\| + penalty | No: the adjacency term is evaluated against one stored predecessor |
| LilyPond `constrained-breaking.cc` | DP, one node per (system count, breakpoint) | force² + (previous force − force)² + penalty | Justified lines: no, for the same reason. Ragged-right lines: yes |
| LilyPond page breaking | DP over fixed lines; search over system counts | force² per page plus penalties | Page DP over fixed lines: yes. Search over system counts: no, bounded by heuristics and using estimated heights |
| Stage 3 | DP, one node per breakpoint | (scale − 1)² × Σ ideal² + `break-penalty-fn` | Yes |
| Stage 6 | DP over systems | Not yet specified | Not established |

**Why Stage 3 is exact**: its cost has no term linking adjacent systems, so one node per breakpoint loses nothing; every other input — `system-width-fn`, `preamble[s]`, `gutter[s]`, `break-penalty-fn` — depends only on the candidate segment; forced breaks partition the problem into independent subproblems; prevented breaks only reduce the set of breakpoints; and the early-termination bound excludes only infeasible segments (see [Key Implementation Notes](#key-implementation-notes)). In this sense Stage 3 is an exact Knuth–Plass computation. Its cost is the baseline, not the full Knuth–Plass cost model; which extensions keep exactness at which cost in state is stated under [Relationship to Knuth-Plass](#relationship-to-knuth-plass).

**Why Stage 6 is not established**: its cost function is not specified, and page heights that depend on the page — a first page carrying a title, running headers — require the page number, or at least its parity, in the DP state. Both are open.

### What Is Specific to Ooloi

- The combination of the two lineages over measure-stack scalars, structured so that full Knuth–Plass remains available
- The gutter and preamble as precomputed widths that depend on where a system starts
- Exact rational arithmetic, giving identical results on every platform
- Use inside an interactive, collaborative editor

## Consequences

**Architectural pattern**:

This ADR demonstrates the same architectural property as [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md). In both cases, a problem that looks as though it needs heuristics, special cases, or manual correction reduces to a straightforward algorithm once the architecture provides:
- Immutable data structures
- Semantic determinism resolved before the algorithm executes
- Explicit stage boundaries preventing feedback loops
- Rational arithmetic eliminating accumulation errors

For remembered alterations, the timewalk provides temporal ordering independent of visual layout. For measure distribution, the pipeline provides collision-free metrics (plus gutter deltas) independent of system assignment. Both reduce coupled problems to sequential ones.

**Positive:**

1. **Exact optimization** - Polynomial-time algorithm finds the break configuration of least cost under the stated cost function, given its inputs
2. **Proportionality preservation** - Rhythmic relationships maintained across systems by construction
3. **Complete information** - Gutter deltas enable optimal decisions without feedback
4. **Closed semantic model** - Rendering decorations cannot affect musical semantics
5. **Deterministic output** - Identical input produces identical layout across platforms
6. **Cheap recomputation** - An edit re-runs Stage 3 over cached scalar metrics, without repeating upstream work for unchanged stacks
7. **Variable system widths** - Natural handling of final systems, editorial overrides, margins and layout settings
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
6. **Global breaks** - The break configuration is a global optimum, so an edit can change system breaks anywhere in the score; stability comes from the mechanisms under [Stability and Quality Extensions](#stability-and-quality-extensions), not from the DP itself

**Future Considerations:**

The proportional scaling approach should be validated empirically on real scores:
1. Implement baseline proportional scaling as specified
2. Test on real scores: simple → complex → Elektra
3. Document visual quality, performance characteristics, edge cases
4. Only if systematic problems emerge, consider asymmetric optimization refinements

The formal validation will determine whether the baseline approach is sufficient or whether additional complexity is justified. The specific visual issues encountered will guide cost function design rather than speculative theory.

## References

- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) (pipeline architecture, Stage 3 system breaking position, gutter model, closed semantic model)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) (accidental algorithm that closes the semantic model; tied-note bypass rules inform gutter requirements)
- **ADR-00XX: Horizontal Spacing** — forthcoming: upstream computation of min/ideal/gutter widths
- [ADR-0014: Timewalk](0014-Timewalk.md) (temporal traversal providing measure discovery)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md) (immutable data structures enabling stage separation)
- [Knuth–Plass line-breaking algorithm](https://en.wikipedia.org/wiki/Knuth%E2%80%93Plass_line-breaking_algorithm) - Wikipedia overview
- Knuth, D.E. and Plass, M.F. "Breaking Paragraphs into Lines", *Software—Practice and Experience* 11 (1981), pp. 1119–1184 - foundational algorithm
- Hegazy, W.A. and Gourlay, J.S. "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, The Ohio State University (1987); also in *Proceedings of the International Conference on Electronic Publishing, Document Manipulation and Typography*, Nice, 1988, ed. J.C. van Vliet, Cambridge University Press
- LilyPond source: `lily/gourlay-breaking.cc` (1997–2006), `lily/constrained-breaking.cc`, `lily/simple-spacer.cc`, `lily/optimal-page-breaking.cc`, `lily/page-turn-page-breaking.cc`
- Taupin, D., Mitchell, R. and Egler, A. "MusiXTeX : L'écriture de la musique polyphonique ou instrumentale avec TeX", *Cahiers GUTenberg* 21 (1995), pp. 107–113
- The MusiXTeX manual, §1.3.1 "The three pass system: Introduction"; `musixflx` (Ross Mitchell, 1992–1997)
- Steinbach, A. and Schofer, A. *Automatisierter Notensatz mit TeX*, master's thesis, Rheinische Friedrich-Wilhelms Universität, Bonn (1987)
- Ross, T. "The Art of Music Engraving and Processing" (1970) - proportional spacing values
- Gould, E. "Behind Bars" (2011) - modern engraving standards
