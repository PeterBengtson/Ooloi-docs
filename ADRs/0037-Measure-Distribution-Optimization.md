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
  - [The Postamble: System-End Width](#the-postamble-system-end-width)
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
  - [Ending Classes and Looseness](#ending-classes-and-looseness)
  - [Fitting Rules](#fitting-rules)
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
│         │              min, min-ratio, ideal, gutter                    │
│         │              per measure stack                                │
│         ▼                     ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │               Stage 3: SYSTEM BREAKING (Single)                 │    │
│  │                                                                 │    │
│  │   Input: N measure stacks with widths + gutter delta            │    │
│  │          {:min :min-ratio :ideal :gutter}, all ratios           │    │
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
│  │   │                     - postamble[t]                 │        │    │
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
│  │  DP O(S²) over systems using actual heights from Stage 5;       │    │
│  │  its cost function is not yet specified                         │    │
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
- Cost model: the badness of each system's adjustment ratio, combined into demerits (Knuth–Plass; see the segment cost model below)

**Distribution constraints**:
- System capacity: Fixed maximum width per system
- Page capacity: Fixed maximum height per page (number of systems)
- Stack atomicity: Measure stacks cannot be subdivided

**Explicit exclusions**:
Connecting elements (ties, slurs, hairpins, beams, glissandi, ottava lines) do not participate in the distribution optimization. They are computed in Stage 5 after positions are finalized, and adapt to the determined geometry.

**Objective**:
Minimize total demerits — each system's cost, derived from how far it is stretched or compressed from its ideal proportions — while satisfying capacity constraints. The intent is that minimizing total demerits produces layouts perceived as stable and professionally typeset.

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
Parallel collision detection produces measure stack metrics. Each of N stacks emerges with definitive width bounds: `min_width` and `ideal_width` for the measure's semantic content, the largest ratio of minimum to ideal width among its column gaps, plus `gutter_width` for additional system-start space.

**Stage 1 is width-complete**: It computes all horizontal space requirements for atoms, including lyrics, dynamics, articulations, and fixed-size anchors for spanners (e.g., "sffzpp<" letters, minimum crescendo wedge width). Stage 1 cannot compute full spanner geometry (that depends on final positions from Stage 4), but it must include all horizontal space the atom requires, including spanner attachment points.

**Stage 5 is height-complete**: After Stage 4 positions atoms, Stage 5 computes connecting element geometry and determines final vertical extent. System heights are definitive after Stage 5.

The vertical alignment problem is solved completely before distribution begins. This reduction is enabled by:
- Immutable data structures eliminating race conditions during parallel processing
- Rational arithmetic preserving exact proportions without floating-point accumulation
- STM coordination ensuring atomic reads of hierarchical musical structure

**Stage 3: System Breaking**
With vertical coordination complete, the problem reduces to:
- A 1-dimensional sequence of N scalar tuples (min_width, ideal_width, largest column ratio, gutter_width)
- Preamble and gutter subtraction at the start of each system, and postamble subtraction at its end, before scaling
- Capacity-constrained segmentation into systems using Knuth-Plass DP
- Cost function on each system's adjustment ratio (badness and demerits)
- No feedback from geometry to distribution logic
- Output: System break decisions and scale factors per system

**Stage 4: Atom Positioning**
Applies scale factors from Stage 3 to position atoms at their actual coordinates. Non-connecting visual elements (noteheads, accidentals, dynamics) are positioned using the computed scale factors. These elements do not influence distribution.

**Stage 5: Spanners and Margins**
Ties, slurs, and spanning attachments compute geometry based on finalized atom positions from Stage 4. They adapt to the determined layout rather than influencing it. For measures at system-start positions, Stage 5 adds graphical decorations (courtesy accidentals, tie continuations) within the reserved gutter space. Determines actual system heights from vertical extent of connecting elements.

**Stage 6: Page Breaking**
A dynamic programme over the system sequence, using actual system heights from Stage 5; its cost function is not yet specified. Produces final page breaks, completing the layout.

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
Dynamic programming over the sequence of N measure stacks determines optimal system break points. For each potential break location, the algorithm evaluates whether remaining measures fit within capacity (after reserving preamble, gutter and postamble space) and computes the resulting demerits. Optimal substructure holds: optimal solution for measures 1..k combined with optimal solution for measures k+1..N yields optimal solution for 1..N.

Break selection relies on **separable system costs** and **optimal substructure**, not convexity. The dynamic programming algorithm computes the break configuration of least total cost for the discrete segmentation problem, under the stated cost function and given its inputs.

**What "optimal" means in this ADR**: *optimal* always means minimal total demerits under the segment cost model specified below — badness of each system's adjustment ratio, the per-system penalty and the breakpoint penalties from `break-penalty-fn`, with the parameters in force — given the metrics Stages 1–2 supply. It is a statement about that cost function and those inputs, not about layout quality in any wider sense: a different cost function, or different upstream metrics, would have a different optimum.

Complexity: O(N²) for system breaks, O(S²) for page breaks where S = number of systems.

**Continuous width allocation**:
Within each system `[s, t)` determined by break selection, actual widths are allocated by **proportional scaling** (normalisation) of the space remaining after preamble, gutter and postamble reservation:

```
available = system_width - preamble[s] - gutter[s] - postamble[t]
scale_factor = available / Σ ideal_i
actual_i = ideal_i × scale_factor
```

This is not optimization—it is a deterministic formula that preserves proportional relationships by construction. All measures in a system receive the same scale factor, so `actual_i / actual_j = ideal_i / ideal_j` always holds.

**Segment cost model**:
The cost of a system is Knuth–Plass's. It is judged by its **adjustment ratio** `r` — how far its glue is stretched or shrunk relative to how far it can stretch or shrink:

```
r = (available − Σ natural) / Σ stretch     when available ≥ Σ natural   (stretching)
r = (available − Σ natural) / Σ shrink      when available <  Σ natural   (shrinking)
```

Under the proportional baseline every gap's stretch and shrink equal its natural width, the ideal, so `Σ stretch = Σ shrink = Σ ideal_i` and **`r = scale_factor − 1`**: negative under compression, positive under stretching.

The **badness** of the system is a function of `r` alone, not weighted by content:

```
b(r) = min(b_max, c × |r|^e)
```

Its **demerits** combine badness with a per-system penalty `l` and the penalty `p` of the break that ends it:

```
d = (l + b)² + p²      when p > 0
d = (l + b)² − p²      when p < 0
d = (l + b)²           when p = 0, and for a forced break
```

`l` is added to every system, so each additional system costs at least `l²`: the model favours fewer systems. Squaring a cubic badness makes a system's demerits grow roughly with the sixth power of `r`, so the DP strongly avoids a single very bad system, preferring to spread adjustment across several. A forced break adds no penalty term; in this ADR forced breaks are realised by pre-segmentation ([Editorial Control Mechanisms](#editorial-control-mechanisms)), so the DP never meets one. The DP minimises total demerits.

**Tolerance**: an optional bound on badness. A segment whose badness exceeds it is treated as infeasible.

**Why the badness of `r`**: the adjustment ratio is what a reader sees — how far a system is stretched or compressed. A cost weighted by content, such as `Σ (actual_i − ideal_i)²`, which under proportional scaling is `(scale − 1)² × Σ ideal_i²`, makes two systems with the same adjustment ratio cost differently according to how their content is divided into measures: the same music in more, shorter measures has a smaller `Σ ideal_i²` and so costs less at the same stretch. That difference has no musical justification.

**Parameters**: the badness coefficient `c`, exponent `e`, cap `b_max`, per-system penalty `l` and tolerance are parameters. Their documented starting points are TeX's implementation of Knuth–Plass: `c = 100`, `e = 3` (badness "a reasonably close approximation to 100(t/s)³"), `b_max = 10000` ("infinitely bad"), `l = 10` (plain TeX's `\linepenalty`) and tolerance 200 (plain TeX's `\tolerance`). The empirical validation decides them ([Evaluation Criteria](#evaluation-criteria)).

**Exact arithmetic**: demerits are exact rationals — with `e = 3`, of degree six in `r` — and are still O(1) per segment. Any approximation, for example of a non-integer exponent, must be deterministic, so that every platform computes identical demerits.

**Additive separable cost model**:
- **System-level cost**: each system's demerits depend only on its own adjustment ratio and the penalty of its break — that is, on the segment alone. The `min` value and the largest column ratio `ρ` participate in feasibility checking; the `preamble`, `gutter` and `postamble` values determine the available space, and through it `r`; none of them enters the cost otherwise.
- **DP operates on system-level cost units**: the dynamic programming algorithm sums per-system demerits to compute the total.

This separability enables independent evaluation of candidate break points during dynamic programming.

**Page breaking: Second-order segmentation**:
Page breaking is treated as a **second segmentation pass over the system sequence, never interleaved with system breaking**. After optimal system breaks are determined, page breaks are computed independently by a second dynamic programme over the system sequence, whose cost function is not yet specified. The two are separate because page breaking needs actual system heights, which exist only once Stage 5 has built the systems; optimising both together would have to work from estimated heights (see [ADR-0028 §Trade-offs](0028-Hierarchical-Rendering-Pipeline.md#trade-offs)).

**Deterministic outcomes**:
Given identical input (measure stacks with their bounds, system/page capacities), the algorithm produces identical output. This determinism arises from:
- Rational arithmetic (no floating-point nondeterminism)
- Immutable data structures (no timing-dependent state)
- Deterministic tie-breaking: equal costs are resolved by a secondary cost that differs for any two configurations (see [Deterministic Tie-Breaking](#deterministic-tie-breaking))
- Normalisation-based allocation (no iterative solver convergence)

**Edit locality**: Preserving break decisions away from an edit, to minimise perceptual layout "jitter" during editing, is a goal. The DP does not provide it: a global optimum is not local. See [Edit Locality](#edit-locality).

## Decision

### Architectural Position

Measure distribution optimization (system breaking) is Stage 3 of the hierarchical rendering pipeline ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)). It receives measure stack metrics from Stages 1-2 and produces system break decisions and scale factors that Stage 4 uses for atom positioning.

The algorithm implements **capacity-constrained segmentation with proportional width allocation**:
1. Dynamic programming (Stage 3) determines optimal system break points, accounting for preamble and gutter space at system starts and postamble space at system ends
2. Proportional scaling computes scale factors for width allocation within the available space (after preamble, gutter and postamble reservation)
3. Atom positioning (Stage 4) applies scale factors to compute actual positions
4. System heights (Stage 5) are computed from final geometry
5. Page breaking (Stage 6) is a second dynamic programme, over systems with actual heights, whose cost function is not yet specified

### Interface Contract

```clojure
;; Input from Stages 1-2 (computed by ADR-00XX, forthcoming):
stacks ;; Vector of stack maps

;; Each stack map:
{:min ratio              ;; Hard collision boundary for measure content
 :min-ratio ratio        ;; Largest column ratio, max_j(min_gap_j / ideal_gap_j)
 :ideal ratio            ;; Target proportional spacing for measure content
 :gutter ratio           ;; Additional space when at system start (default 0N)
 :measure-index int}     ;; Original measure index (for debugging/tracing)

;; Output:
{:breaks [break-positions]    ;; Vector of indices where systems start
 :cost total-demerits}        ;; Total demerits (ratio)
```

**Width component semantics**:

- `:min` — collision floor for the measure's semantic content: the sum of its column gaps' minimum widths
- `:min-ratio` — the largest ratio of minimum to ideal width among the stack's column gaps, written `ρ_i` in formulas; the smallest scale factor at which uniform scaling keeps every column gap at or above its minimum (see [Column-Level Feasibility](#column-level-feasibility))
- `:ideal` — proportional target for the measure's semantic content
- `:gutter` — additional space required when this measure appears first on a system (default 0N); this space is reserved for graphical decorations (courtesy accidentals, tie continuations) and does not scale

The gutter is *additional* to the measure's width, not part of it. When a measure appears first on a system, the system must accommodate `gutter[s] + actual[s]` for that measure.

**Preconditions** (guaranteed by upstream stages):
- `:ideal` must be positive for all stacks (otherwise scale factor computation fails)
- `:min` must be positive and `:min ≤ :ideal` for all stacks
- `:min / :ideal ≤ :min-ratio ≤ 1` for all stacks (the largest column ratio is at least the stack's overall ratio)
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

System layout for a system of stacks `[s, t)`:

```
┌───────────┬──────────┬─────────────┬─────┬─────────────┬────────────┐
│ PREAMBLE  │  GUTTER  │  measure s  │ ... │ measure t-1 │ POSTAMBLE  │
│  (fixed)  │ (fixed)  │  (scaled)   │     │  (scaled)   │  (fixed)   │
└───────────┴──────────┴─────────────┴─────┴─────────────┴────────────┘
      ↑          ↑           ↑                                 ↑
      │          │           │                                 └─ postamble[t] (does not scale)
      │          │           └─ actual[i] = ideal[i] × scale_factor
      │          │
      │          └─ gutter[s] (does not scale)
      │
      └─ preamble[s]: clef + keysig (does not scale)
```

**What the gutter accommodates:**
- Courtesy accidentals for tied-to notes at position 0 (graphical decorations, not semantic accidentals)
- Tie continuation arcs (visual portion of ties broken at system boundaries)
- A start repeat that moves, at a break, to the system's start — the start part of a combined repeat, or a start repeat at the boundary — for the extra width it needs there beyond its mid-system form

**Critical architectural property:** The gutter width is computed in Stage 1 from complete information about the measure's layout, including all semantic accidentals at position 0. If existing accidentals already provide sufficient space for the courtesy accidental, the gutter width is 0N—only the additional space needed beyond the measure's existing layout is reserved. Stage 3 knows exactly how much additional space each measure requires *before* making any distribution decisions. This lets Stage 3 find the distribution that is optimal for its cost function, given these inputs, without heuristics or iteration.

**What is NOT in the gutter:** All semantic accidentals are computed and positioned by [ADR-0035](0035-Remembered-Alterations.md) and are included in the measure's `min_width` and `ideal_width`. The gutter contains only graphical decorations added by Stage 5 for visual clarity at system boundaries.

**User control:** Users can configure when courtesy accidentals appear (`:system`, `:page`, or `:none`). This setting affects Stage 5 rendering, not Stage 3 distribution—the gutter is always reserved; the decorations are optionally rendered. See [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md) for details.

**Allocation with preamble, gutter and postamble:**

For a system containing stacks [s, t):
```
preamble = preamble[s]                       ;; Clef + keysig width (max across staves)
gutter = gutter[s]                           ;; Only first stack contributes
postamble = postamble[t]                     ;; System-end cautionaries and barline; 0N when t = N
available = system_width - preamble - gutter - postamble
scale_factor = available / Σ ideal_i         ;; Scale factor for all measures

actual[i] = ideal[i] × scale_factor          ;; ALL measures scale the same
```

**Verification:**
```
preamble + gutter + Σ actual_i + postamble
  = preamble + gutter + available + postamble = system_width ✓
```

The preamble, gutter and postamble are not added to any measure's width—they are separate space: the preamble and gutter precede the first measure's content, and the postamble follows the last. Stage 5 renders clefs/keysigs in preamble space, gutter decorations and any start repeat in gutter space when the measure appears at system start, and system-end cautionaries and the closing barline's system-end form at the system's end, in the last measure's own barline space extended by the postamble.

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

**Preamble, Gutter and Postamble**:
- **Preamble**: Structural (always present at system start), computed on-demand
- **Gutter**: Contingent (present only when tied-to notes require courtesy accidentals, or a start repeat moves to the system's start), precomputed in Stage 1
- **Postamble**: Contingent (present only when a change takes effect at the start of the next system, or a double, final or repeat barline ends the system), at system end, computed on-demand — see [The Postamble](#the-postamble-system-end-width)

All three are fixed overhead subtracted before proportional scaling. None participates in cost computation.

### The Postamble: System-End Width

A system's end can need space that the same measures do not need mid-system. When a change takes effect at the first stack of the next system — a key signature or time signature change, for example — the system ends with a cautionary announcing it. When the barline at the break is a double, final or repeat barline, it takes its system-end form there: a combined repeat splits, its end repeat closing this system and its start repeat opening the next. The **postamble** is the fixed space these need at the end of a system, the counterpart of the system-start reservations.

For a system `[s, t)`, the postamble is `postamble[t]`: it depends only on where the system ends — on the changes taking effect at stack `t`, the first stack of the next system, and on the barline at the boundary before it. Like the preamble and gutter it does not scale and does not enter the cost; it reduces the space available for scaling:

```
available = system_width - preamble[s] - gutter[s] - postamble[t]
```

Because it depends only on the candidate segment, the DP stays exact. Because it is fixed for a given `t`, it is constant across the inner loop, leaves early termination unaffected, and costs one lookup per outer iteration.

**Contents**:
- **Cautionaries**: the cautionary clef, key signature and time signature for the changes taking effect at stack `t`. A key signature cautionary includes any cancellation naturals the change requires, and its width includes theirs.
- **Barline**: when the barline at boundary `t` is a double, final or repeat barline, the extra width its system-end form needs beyond its mid-system form, which is already part of the last measure's widths. A single barline contributes nothing. This part is never negative: where the system-end form is narrower than the mid-system one, the difference is not reclaimed.

Where neither applies, `postamble[t] = 0N`. At the end of the piece `postamble[N] = 0N`: no change follows, and the closing barline there has no mid-system form, so the last measure's widths already include it.

**The start of the next system**: a start repeat that moves, at a break, to the start of the next system is reserved there by that system's gutter (see [The Gutter](#the-gutter-system-start-width-delta)).

**On-demand computation**: like the preamble, the postamble is computed during Stage 3 when evaluating candidate system breaks, not precomputed in Stage 1:
1. Query ChangeSets for the clef, key signature and time signature changes taking effect at stack `t` (per staff), and read the barline at boundary `t`
2. Compute, for each staff, the width of the corresponding cautionaries and the barline's extra width in its system-end form
3. Take the maximum across all staves

**Caching**: the combinations of changes and barline types are highly repetitive across a score, so the postamble is cached by that combination across staves, as the preamble is. The cached value includes both the computed maximum width and the paintlists for rendering.

**Why uniform width**: as with the preamble, all staves receive the same postamble width, the maximum across staves, so that the systems' closing barlines stay aligned. What is drawn in it may differ from staff to staff.

**Placement**: a cautionary clef sits before the system's closing barline; cautionary key and time signatures sit after it.

**User control**: a setting suppresses system-end cautionaries. When it is set, the cautionary part of `postamble[t]` is 0N and Stage 3 sees the change; the barline part remains, since the barline is drawn whatever the setting. Unlike the gutter, which is always reserved, the cautionary part can depend on the setting only because the setting has no page-dependent option: whether a system ends a page is decided in Stage 6, after Stage 3 has used the postamble, so a reservation that depended on page position could not know which systems need it (the reason the gutter is always reserved; see [Enabling User Control](#enabling-user-control)).

### Width Allocation: Proportional Scaling

The primary width allocation strategy is **proportional scaling** (normalisation) applied to the space remaining after preamble, gutter and postamble reservation:

```
available = system_width - preamble[s] - gutter[s] - postamble[t]
scale_factor = available / Σ ideal_i
actual_i = ideal_i × scale_factor
```

The preamble, gutter and postamble are fixed overhead; none participates in scaling.

This is **normalisation**, not optimisation:
- Deterministic formula with no degrees of freedom
- Preserves proportional relationships by construction
- Preamble, gutter and postamble space are reserved separately; none scales
- No iteration, no convergence, no tuning parameters
- Predictable behavior: within a fixed break configuration, a change to one stack alters only its own system's scale factor
- Near-zero computational cost
- Clean architectural separation: DP selects breaks, allocation is derived

**Why this may be sufficient**:

1. **Proportionality preservation**: The scaling factor is identical for all measures, preserving ratios
2. **Reservation correctness**: System-start decorations and system-end cautionaries and barlines occupy exactly their required space
3. **No heuristics**: Pure mathematical formula, deterministic outcome
4. **Speed**: O(K) per system with K ≈ 10-20, essentially free
5. **Predictability**: Users can mentally predict system behavior
6. **Contained allocation**: Within a fixed break configuration, a change to one stack alters only its own system's scale factor

**Handling constraints**:
```clojure
actual_i = max(min_i, ideal_i × scale_factor)
```

The `max` clamp is defensive. Under the feasibility contract (`scale ≥ max(ρ_i)`, the largest column ratio over the system's stacks), proportional scaling keeps every column gap at or above its minimum, and so always produces `actual_i ≥ min_i`; the clamp never changes any value. If clamping were to activate, it would indicate the segment was incorrectly selected as feasible.

**Notes on the cost**:

With proportional scaling, `actual_i = ideal_i × scale_factor` for all i, so every stack deviates from its ideal by the same ratio, `r = scale_factor − 1`. The system's cost is the badness of that single ratio, combined into demerits (see the segment cost model under [Optimization Characterization](#optimization-characterization)) — not a sum over its stacks.

The preamble, gutter and postamble do not participate in cost computation. They are fixed overhead; they affect the cost only through the available width, and so through `r`.

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

4. **No algorithmic complexity added**: The DP evaluates `(s, t)` segments against the width available for that system position, minus preamble, gutter and postamble. Proportional scaling applies the provided width directly.

**Per-system width policy**:

**Any system may have a distinct width policy** without special-casing, the final system included. Heterogeneous system widths need no additional mechanism because allocation is parameterized by width, not built around a fixed one.

**Expressivity without complexity**:

This architectural freedom enables:
- Natural handling of the final system of a piece or movement (avoid excessive stretch)
- Editorial control over system widths for specific musical reasons
- Adaptation to margins and layout settings
- Future extensions (e.g., systems of varying width for visual effect)

All while preserving the core property: **proportional scaling maintains rhythmic relationships within whatever width is provided**, with preamble and gutter space reserved for system-start elements and postamble space for system-end cautionaries.

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

**Stage 6: Page Breaking** is a second segmentation over the system sequence, never interleaved with system breaking. It is a dynamic programme of the same shape as Stage 3 — candidate pages `[a, b)` of systems, each judged by feasibility and a segment cost — over the actual system heights computed by Stage 5. Stage 3's cost does not carry over: it measures deviation from ideal widths, and systems have heights, not widths.

Stage 6's cost function is not yet specified: how a page's fill is valued, how page turns and page parity enter, and how a page height that depends on the page — a first page carrying a title, running headers — enters the DP state. This ADR makes no optimality claim for page breaks.

```clojure
(defn find-page-breaks
  "Stage 6: dynamic programme over the system sequence, of the same shape as
   Stage 3, over the system heights Stage 5 computed.
   page-cost-fn — the cost of placing systems a..b-1 on one page — is not yet
   specified."
  [systems page-cost-fn]
  ...)
```

The separation exists because system heights are known only after Stage 5 has built the systems; a combined optimisation would have to work from estimated heights.

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
   Only the first stack in the group contributes :gutter; the group's
   largest column ratio is the largest of its members'."
  [stacks no-break-ranges]
  
  (let [grouped (apply-grouping stacks no-break-ranges)]
    (mapv (fn [group]
            {:min (reduce + 0N (map :min group))
             :min-ratio (reduce max (map :min-ratio group))
             :ideal (reduce + 0N (map :ideal group))
             :gutter (:gutter (first group))
             :member-stacks group})
          grouped)))
```

The DP operates on groups; after breaks are determined, groups expand back to constituent stacks for width allocation.

**Width Overrides: Upstream Modification**

User adjustments to individual measure widths ("stretch this measure," "compress this passage") belong upstream of Stage 3, not within it. Stage 3 consumes the stack width metrics; editorial overrides modify these inputs:

```clojure
(defn apply-width-overrides 
  "Applies user width overrides to stack metrics before distribution.
   
   Overrides may specify:
   - :min-override  - New minimum width (takes max with collision minimum)
   - :ideal-override - New ideal width (replaces rhythmic ideal)
   
   The largest column ratio follows both: the stack's column ideals scale
   with its ideal, so their ratios scale inversely, and a raised minimum
   can itself become the binding ratio."
  [stacks user-overrides]
  
  (reduce 
    (fn [stacks {:keys [measure-index min-override ideal-override]}]
      (update stacks measure-index
        (fn [{:keys [min min-ratio ideal] :as stack}]
          (let [ideal' (or ideal-override ideal)
                min'   (if min-override (max min min-override) min)]
            (assoc stack
                   :min min'
                   :ideal ideal'
                   :min-ratio (max (* min-ratio (/ ideal ideal'))
                                   (/ min' ideal')))))))
    stacks
    user-overrides))
```

**Key principle**: The collision-derived `min_width` is a hard floor. User overrides can raise it but not lower it—atoms cannot overlap regardless of editorial intent. The `ideal_width` can be freely overridden since it represents preference, not physics. The `gutter` is not user-adjustable—it reflects SMuFL glyph metrics for system-start decorations.

**Soft Preferences: The break-penalty-fn Hook**

The optional `break-penalty-fn` parameter handles soft preferences that influence but do not constrain break selection. It returns the penalty `p` for selecting segment `s..t-1` as a system — in Knuth–Plass terms, the penalty of its break — which enters the demerits as `+p²` when positive and `−p²` when negative:

```clojure
;; Prefer breaks at rehearsal marks
(defn rehearsal-mark-preference [stacks s t]
  (let [start-stack (nth stacks s)]
    (if (has-rehearsal-mark? start-stack)
      -50N  ; Negative penalty = preference: subtracts p² = 2500 from the demerits
      0N)))

;; Avoid very short systems
(defn minimum-system-length-preference [stacks s t]
  (let [system-length (- t s)]
    (if (< system-length 3)
      100N  ; Positive penalty: adds p² = 10000 to the demerits
      0N)))

;; Combine multiple preferences
(defn combined-preferences [stacks s t]
  (+ (rehearsal-mark-preference stacks s t)
     (minimum-system-length-preference stacks s t)))
```

These preferences shift costs without creating hard constraints. Combined preferences add their penalties before the result is squared. The algorithm may still choose penalized configurations if total demerits are lower.

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
2. **Stage 3** has *complete knowledge* of every scenario (mid-system, system-start and system-end) for every measure
3. **Stage 3** computes the system breaks that are optimal for its cost function, given these inputs - not heuristic, not iterative
4. **Stage 4** positions atoms using scale factors from Stage 3
5. **Stage 5** adds graphical decorations based on actual positions and computes system heights
6. **Stage 6** computes page breaks by a dynamic programme over the systems, using actual system heights; its cost function is not yet specified

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

The setting affects Stage 5 rendering only; the gutter is always reserved (see [The Gutter](#the-gutter-system-start-width-delta)). It has to be: under `:page`, whether a system-start stack also starts a page is decided in Stage 6, after Stage 3 has used the gutter, so a reservation that depended on the setting could not know which systems need it. Changing the setting changes what Stage 5 draws in the reserved space; the distribution is unchanged, and no iteration or manual correction is involved.

**Architectural capability:** The complete-information architecture makes user-controllable courtesy accidental settings straightforward to implement—a feature that requires deterministic distribution to work without manual adjustment or layout jitter. Traditional architectures that lack complete information at distribution time cannot offer such settings without risking non-deterministic behavior or requiring iterative correction.

This demonstrates how complete information turns a potentially complex feature into a choice about what is drawn in space already reserved, which needs no change to the distribution. The architecture enables the feature; the feature validates the architecture.

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

When break configurations produce identical demerits (a "tie" in cost, not a musical tie), the configuration is chosen by a secondary cost, compared only when the primary costs are equal. The secondary cost is separable — a sum of per-break terms — and differs for any two distinct configurations: for example, the sum over the configuration's breakpoints `b` of `2^−b`, which, like a binary fraction, is different for any two different sets of breakpoints. Costs are compared as pairs (primary, secondary), lexicographically. Pairs add componentwise and the lexicographic order is preserved under addition, so the DP remains exact for the pair, and the optimum is unique.

Because the choice is a property of the configuration rather than of the order in which a procedure meets candidates, every procedure that minimises the pair — the full pass, and [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit) from forward and backward tables — returns the same configuration. The result is deterministic across runs, platforms and editing histories. Ties are expected rather than exceptional: under rational arithmetic, identical measures produce identical costs.

In the Stage 3 code sketch a segment `[s, t)` adds its primary cost and the term `2^−t` for the breakpoint it ends at, and a candidate replaces the stored best only when its pair is lexicographically smaller.

If editorial preferences are needed (e.g., prefer structural boundaries), they can be encoded via the optional `break-penalty-fn` parameter, which shifts costs rather than relying on tie-breaking.

### Edit Locality

Edit locality — preserving break decisions away from an edit, so that the layout does not visibly "jitter" while the user works — is a goal. The algorithm does not provide it.

The break configuration Stage 3 selects is a global optimum, and a global optimum is not local. An edit to stack `m` can change system breaks after `m` and before `m`: the DP finds the configuration of least total cost for the whole sequence, and the chosen breaks are traced backwards from its end, so a change of cost at `m` can alter the choice of breaks anywhere in the score. Nor is there any guarantee that, beyond some point, the new break decisions coincide with the previous solution again.

What holds is that recomputation is cheap. Stack metrics are cached (see [Caching and Incremental Recompute](#caching-and-incremental-recompute)), so re-running Stage 3 after an edit is scalar arithmetic over the cached metrics: collision detection, atom formation and vertical reconciliation are not repeated for any stack whose content did not change.

The mechanisms that provide it are specified under [Stability and Quality Extensions](#stability-and-quality-extensions): exact re-optimisation after an edit keeps recomputation to about K² evaluations, and a stability term and frozen systems keep breaks where they were.

### Caching and Incremental Recompute

Stage 3 operates over a sequence of measure stacks whose width metrics are **already finalized upstream**. In addition, Ooloi caches the following per **measure stack**:

* `min_width` — hard lower bound for measure content (from Stage 1–2)
* `min_ratio` — the largest column ratio `ρ`, the lower bound on the scale factor
* `ideal_width` — proportional target for measure content
* `gutter_width` — additional space for system-start decorations
* `actual_width` — realized width after Stage 3-4 (scale factors applied by Stage 4)

This cache is authoritative at the Stage 3-4 boundary: Stage 3 consumes `(min, min-ratio, ideal, gutter)`, together with the preamble and postamble for each candidate system, and produces break assignments and scale factors; Stage 4 applies scale factors to produce `actual` positions; Stage 5 consumes `actual` and never influences Stages 3-4.

#### Cached Invariants

For each stack `i` in a system `[s, t)`:

* `scale_factor(system) ≥ ρ_i`, so every column gap of the stack is at or above its minimum, and `actual_width_i ≥ min_width_i`
* `actual_width_i = ideal_width_i × scale_factor(system)`
* `scale_factor(system) = available / Σ ideal_width_j` where `available = system_width - preamble[s] - gutter[s] - postamble[t]`

The cache additionally implies that per-stack metrics are **stable across edits** unless the edited content is in that stack.

#### Incremental Update Consequences

Edits affect Stages 3-4 only through changes to stack metrics:

1. **Local edit**: modifying or inserting notation in a measure updates only the affected stack's upstream-derived width values, and the postamble of any system boundary at which that change takes effect.

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

- The distribution of per-system adjustment ratios `r`: minimum, maximum, variance
- The difference between adjacent systems' `r`
- The number of systems
- How many breaks an edit changes (stability)
- Running time

These measurements also decide the **badness exponent and the constants** of the segment cost model — the coefficient, exponent, cap, per-system penalty `l` and tolerance — starting from TeX's values ([Optimization Characterization](#optimization-characterization)).

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

`d″(s, t) = d(s, t) + λ × D(s, t)`, where `d` is the segment's demerits and `D` measures departure from an anchored previous layout — for example 1 when `s` is not a break of the anchor, or when `[s, t)` is not one of its systems. `D` depends only on the segment, so the objective stays separable and the result is exact for `d + λD`. That objective is deliberately not the pure demerits of the segment cost model: `λ` sets how much quality is exchanged for stability.

- **The anchor is layout data**: persisted with the layout, and identifying breaks by measure identity rather than by index, so that it survives measure insertion and deletion and the layout remains regenerable from semantics and layout data.
- **Anchor policy sets the refresh cost**: while the anchor is fixed, [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit) applies unchanged, with `F` and `G` computed under the anchored objective. Moving the anchor changes `D` for any segment whose relation to it changed, so both tables are recomputed, O(N×K). When the anchor moves — after every edit, on save, on an explicit reflow — and the value of `λ` are evaluated.

### Frozen Systems

A system the user locks keeps its breaks: its start and end become forced breaks and the breakpoints inside it are prevented, using the pre-segmentation and grouping of [Editorial Control Mechanisms](#editorial-control-mechanisms). A freeze is a constraint, so the result is exact over the configurations that respect it, and it reduces work by partitioning the problem; each partition keeps its own `F` and `G`. Freezes are anchored to measure identity. What happens when an edit inside a frozen system makes it infeasible — refusing the edit, releasing the freeze, or accepting an overfull system — is not yet specified.

### Consistency Between Adjacent Systems

Two exact forms, each needing more DP state than the baseline:

- **Fitness classes**: each system is classified by its adjustment ratio `r` into a small number of bands, and adjacent systems whose bands differ by more than one are penalised. One node per (breakpoint, class): about ×4 state and work with four classes.
- **Squared difference of adjacent adjustment ratios**: `μ × (r(s, t) − r(q, s))²`, where `[q, s)` is the preceding system. The state is the (start, end) of the last system: N×K states with K transitions each, O(N×K²) — about 180,000 evaluations at N = 800, K = 15.

Keeping one node per breakpoint while adding an adjacency term makes the result an approximation (see [Comparison](#comparison)). The term carries a known risk, recorded in a comment in LilyPond's `lily/gourlay-breaking.cc`: where music becomes gradually denser, a uniformity requirement drives cramped lines to become more cramped, because the step from a cramped line of three measures to a loose line of two is large. Scores of gradually changing density are part of the evaluation of either form.

### Ending Classes and Looseness

- **Ending classes**: the class idea applied to the kind of boundary a system ends on — a phrase end, a rehearsal mark, the end of a movement. A penalty that depends only on the boundary needs no classes; `break-penalty-fn` already expresses it. Classes are needed only when the cost depends on the preceding system's ending, and multiply the state by the number of kinds.
- **Looseness**: a user control asking for more or fewer systems than the optimum. The system count joins the state, one node per (breakpoint, count): O(S×N×K), which is O(N²) with S ≈ N / K — about 640,000 evaluations at N = 800. Exact.

### Fitting Rules

Two steps are kept apart. **Ideal spacing** is derived within each measure from the durations of its notes and tuplets, in Stage 1. **Fitting** puts measures onto systems and makes their widths fill each system; Stage 3 decides it and Stage 4 applies it. The breaking decision depends on the fitting rule, because the DP judges each candidate system by its feasibility and cost under that rule.

In Knuth–Plass terms each column gap is glue with two separate parameters ([Relationship to Knuth-Plass](#relationship-to-knuth-plass)):

- its **natural width** — the ideal gap, which is where duration enters;
- its **elasticity** — its stretch and shrink, which decide how the ideal widths give way when a system is looser or tighter than their sum.

A fitting rule is a choice of elasticity. There are four candidates:

1. **Proportional** — the baseline this ADR's formulas specify. Stretch and shrink are proportional to natural width, so one scale factor applies per system. Under stretch and under compression alike, ratios between ideal widths are kept: the rhythmic hierarchy Stage 1 encoded is preserved in proportion, with absolute differences growing when a system is stretched and shrinking when it is compressed.
2. **Even**. Every column gap has the same stretch and shrink. Under stretch every gap gains the same amount, so short values gain proportionally more and the hierarchy flattens. Under compression every gap loses the same amount, so short values lose proportionally more and the hierarchy sharpens.
3. **Shrink bounded by the minimum**. Each gap's shrink is `ideal − min`; stretch is proportional. Under stretch the hierarchy behaves as under the baseline. Under compression gaps give way in proportion to their room above their minimum, so the effect follows the content rather than the durations: where longer values have more room, they give up more and the hierarchy flattens. An incompressible gap has zero shrink and the others absorb the compression, which removes the baseline's rigidity — under `scale ≥ max(ρ_i)`, one incompressible column blocks compression of its whole system.
4. **Duration-weighted elasticity**. Stretch and shrink are weighted by a function of the gap's duration, with shrink capped at `ideal − min` so that no gap goes below its minimum. The hierarchy follows the weighting: shrink weighted toward shorter values — shorter values compress first — sharpens it under compression; stretch weighted toward longer values sharpens it under stretch, and weighted toward shorter values flattens it. This weights elasticity, not natural width, so it does not count duration a second time; it is the counterpart of rule 3, which weights elasticity by content.

Whether the hierarchy should sharpen or flatten under compression is an engraving question, decided by the empirical validation rather than by the model. The baseline is the rule this ADR's formulas and code carry; none of the others is chosen until the validation decides.

**All four are linear glue.** A stack's stretch and shrink are closed-form sums over its gaps, so the adjustment ratio `r` is O(1) per segment from prefix sums of natural width, stretch and shrink; badness and demerits follow from `r` in O(1), and the DP stays exact. Feasibility is O(1) per segment under all four. Under rules 3 and 4 no gap's shrink exceeds its room above its minimum, so feasibility is `Σ min ≤ available` (adjustment ratio `r ≥ −1`), taken from prefix sums. Under rules 1 and 2 a gap can reach its minimum before the system as a whole does, so feasibility is bounded by the gap with the least room — under the baseline the largest column ratio ([Column-Level Feasibility](#column-level-feasibility)), under even fitting the smallest `ideal − min` — carried as a running extremum while the segment grows, one comparison per step.

Under any rule but the baseline, Stage 4 applies the system's adjustment ratio through each gap's own glue rather than one scale factor to every atom. The evaluation covers systems containing one dense or incompressible measure among sparse ones, the behaviour of the rhythmic hierarchy under stretch and under compression, consistency in complex rhythmic contexts, and the proportionality each rule gives up.

### Column-Level Feasibility

Under the baseline, one scale factor is applied to every column gap within every stack. A column gap stays at or above its own minimum only if `scale ≥ min_gap / ideal_gap` for that gap, so the ratio the sufficient feasibility condition takes for a stack is the **largest column ratio within it**, `ρ_i = max_j(min_gap_j / ideal_gap_j)`, not the stack's `min / ideal`. The two coincide only when every column of a stack is equally compressible; otherwise a stack can pass a stack-level check while one of its columns collides.

Each stack therefore carries `ρ_i` as `:min-ratio` ([Interface Contract](#interface-contract)), and the sufficient condition for a system is `scale ≥ max(ρ_i)` over its stacks. `Σ min_i ≤ available` remains the necessary condition, with `min_i` the sum of the stack's minimum gaps. Grouping and width overrides carry the ratio through ([Editorial Control Mechanisms](#editorial-control-mechanisms)).

### End-of-System Width

The [postamble](#the-postamble-system-end-width) is the width at a system's end that depends on where the system breaks: `postamble[t]` depends only on the segment's end, and `available = system_width − preamble[s] − gutter[s] − postamble[t]`. It is fixed overhead outside the cost, O(1) per segment, constant across the inner loop for fixed `t`, and keeps the result exact. An edit that changes it alters every segment ending at `t` as well as those containing the edited stack, so it is a range edit for [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit).

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
(defn pair<
  "Lexicographic order on [primary secondary] cost pairs."
  [[c1 k1] [c2 k2]]
  (or (< c1 c2)
      (and (== c1 c2) (< k1 k2))))

(def knuth-plass-parameters
  "Segment cost parameters. The values are TeX's, as documented starting points;
   the empirical validation decides them."
  {:badness-coefficient 100N  ;; c: badness ≈ c × |r|^e
   :badness-exponent 3        ;; e: an integer, so badness stays an exact rational
   :badness-cap 10000N        ;; b_max: "infinitely bad"
   :line-penalty 10N          ;; l: added to every system; favours fewer systems
   :tolerance nil})           ;; bound on badness beyond which a segment is infeasible;
                              ;; nil = no bound (plain TeX's \tolerance is 200)

(defn badness
  "Badness of a system with adjustment ratio r: c × |r|^e, capped at b_max.
   A function of r alone, not weighted by content."
  [r {:keys [badness-coefficient badness-exponent badness-cap]}]
  (min badness-cap
       (* badness-coefficient
          (reduce * 1N (repeat badness-exponent (abs r))))))

(defn demerits
  "Knuth–Plass demerits of a system of badness b whose break has penalty p.
   Forced breaks never reach the DP: they are realised by pre-segmentation."
  [b p {:keys [line-penalty]}]
  (let [lb (+ line-penalty b)
        d (* lb lb)]
    (cond (pos? p) (+ d (* p p))
          (neg? p) (- d (* p p))
          :else d)))

(defn find-optimal-breaks
  "Finds optimal system breaks for measure stacks using dynamic programming.
   
   Input:
   - stacks: Vector of {:min ratio, :min-ratio ratio, :ideal ratio, :gutter ratio,
                        :measure-index int} maps
   - system-width-fn: Function (fn [start-pos end-pos] -> ratio) 
                      Returns available width for system containing stacks start-pos..end-pos-1
   - break-penalty-fn: Optional. Function (fn [stacks s t] -> ratio)
                       Returns the penalty p for selecting segment s..t-1 as a system;
                       it enters the demerits as +p² or -p²
                       Default: (constantly 0N) - no editorial preferences
   - params: Optional. Segment cost parameters; default knuth-plass-parameters
   
   Output:
   - {:breaks [break-positions], :cost total-demerits-ratio}"
  
  ([stacks system-width-fn]
   (find-optimal-breaks stacks system-width-fn (constantly 0N)))
  
  ([stacks system-width-fn break-penalty-fn]
   (find-optimal-breaks stacks system-width-fn break-penalty-fn knuth-plass-parameters))
  
  ([stacks system-width-fn break-penalty-fn params]
   (let [n (count stacks)
         tolerance (:tolerance params)
         
         ;; Precompute prefix sums for O(1) range queries
         ;; CRITICAL: Use 0N to maintain ratio domain
         min-prefix (vec (reductions + 0N (map :min stacks)))
         ideal-prefix (vec (reductions + 0N (map :ideal stacks)))
         
         ;; Tie-break terms: element t is 2^-t, the secondary cost of a
         ;; segment ending at breakpoint t
         tie-term (vec (reductions (fn [x _] (/ x 2)) 1N (range n)))
         
         ;; State arrays: nil = unreachable, [cost tie] = best pair for the prefix
         ;; Length n+1 where index t represents prefix of length t
         best (transient (vec (repeat (inc n) nil)))
         prev (transient (vec (repeat (inc n) nil)))]
     
     ;; Base case: empty prefix reachable with zero cost
     (assoc! best 0 [0N 0N])
     
     ;; For each prefix length t (represents stacks 0..t-1)
     (doseq [t (range 1 (inc n))]
       
       ;; Reserve postamble space for the system ending at t: cautionaries for
       ;; changes taking effect at stack t, and the extra width of a double,
       ;; final or repeat barline at t in its system-end form (0N when t = n).
       ;; It depends only on t, so it is fixed for the whole inner loop
       (let [postamble (get-postamble-width stacks t)]
         
         ;; Try previous break at prefix length s (represents stacks 0..s-1)
         ;; System contains stacks s..t-1
         ;; Each step leftwards extends the segment by exactly one stack (stack s),
         ;; so max(ρ_i) over the segment is carried forward in O(1)
         ;; rather than rescanned
         (loop [s (dec t)
                max-ratio 0N]
           (when (>= s 0)
             (let [max-ratio (max max-ratio (get-in stacks [s :min-ratio]))
                   system-width (system-width-fn s t)
                   ;; Reserve preamble and gutter space for system-start stack
                   preamble (get-preamble-width stacks s)  ;; On-demand, cached by clef/keysig combo
                   gutter (get-in stacks [s :gutter] 0N)
                   available (- system-width preamble gutter postamble)
                   ;; Check if measures can fit in remaining space
                   total-min (- (nth min-prefix t) (nth min-prefix s))]
               
               ;; Early termination: stop once the minimums exceed the full system width.
               ;; This bound only grows harder to meet as s moves left; `available` does
               ;; not, because preamble[s] and gutter[s] vary with s
               (when (<= total-min system-width)
                 
                 ;; Adjustment ratio under the baseline: stretch and shrink equal
                 ;; natural width, so Σ stretch = Σ shrink = Σ ideal and r = scale - 1
                 (let [total-ideal (- (nth ideal-prefix t) (nth ideal-prefix s))
                       scale-factor (/ available total-ideal)
                       r (- scale-factor 1N)
                       b (badness r params)
                       ;; Necessary: minimums fit after reserving preamble, gutter and postamble
                       ;; Sufficient: the scale factor keeps every column gap at or above
                       ;; its minimum
                       ;; Tolerance: badness within the optional bound
                       feasible? (and (<= total-min available)
                                      (>= scale-factor max-ratio)
                                      (or (nil? tolerance) (<= b tolerance)))]
                   
                   (when (and feasible? (some? (nth best s)))
                     ;; Demerits from badness, the per-system penalty and the break's
                     ;; penalty. Preamble, gutter and postamble enter only through
                     ;; available, and so through r
                     (let [d (demerits b (break-penalty-fn stacks s t) params)
                           [prefix-cost prefix-tie] (nth best s)
                           candidate [(+ prefix-cost d)
                                      (+ prefix-tie (nth tie-term t))]
                           current-best (nth best t)]
                       
                       ;; Update if the candidate pair is lexicographically smaller
                       ;; (nil = infinity); ties in cost are resolved by the secondary term
                       (when (or (nil? current-best)
                                 (pair< candidate current-best))
                         (assoc! best t candidate)
                         (assoc! prev t s)))))
                 
                 (recur (dec s) max-ratio)))))))
     
     ;; Return results
     {:breaks (reconstruct-breaks (persistent! prev) n)
      :cost (first (nth best n))})))
```

**System reservations explained:**

For each candidate segment [s, t):
1. Get `preamble = preamble[s]` (max clef+keysig width across all staves, computed on-demand and cached)
2. Get `gutter = gutter[s]` (only first stack in system contributes)
3. Get `postamble = postamble[t]` (system-end cautionaries and barline; fixed for a given `t`)
4. Compute `available = system_width - preamble - gutter - postamble`
5. Check feasibility: `Σ min_i ≤ available`, and `scale ≥ max(ρ_i)`
6. Compute scale factor: `available / Σ ideal_i`
7. Compute the adjustment ratio `r = scale − 1`, its badness, and the system's demerits; preamble, gutter and postamble are fixed overhead and enter only through `available`

The preamble, gutter and postamble are reserved space that does not participate in scaling or cost computation.

**Feasibility condition:**

The check `Σ min_i ≤ available` is **necessary but not sufficient**. The **sufficient condition** requires:
```
scale_factor = available / Σ ideal_i
scale_factor ≥ max(ρ_i) over the stacks of the system,  ρ_i = max_j(min_gap_j / ideal_gap_j)
```

This ensures proportional scaling keeps every column gap at or above its minimum, and so produces `actual_i ≥ min_i` for all stacks.

**Width Policy Function Examples**:

The `system-width-fn` accepts `(start-pos, end-pos)` and returns the total available width for that system (including preamble, gutter and postamble space):

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

The DP algorithm subtracts preamble, gutter and postamble from the policy-provided width to determine space for scaling.

### Segment Cost Computation

Under the feasibility contract (`scale ≥ max(ρ_i)`), proportional scaling produces `actual_i = ideal_i × scale` with no clamping required, so every stack in the system is stretched or compressed by the same adjustment ratio:

```
r = (available − Σ ideal_i) / Σ ideal_i = scale − 1
b = min(b_max, c × |r|^e)
d = (l + b)² ± p²
```

With precomputed prefix sums of `Σ ideal_i` (and, under the other [fitting rules](#fitting-rules), of `Σ stretch_i` and `Σ shrink_i`), `r` and therefore the demerits are O(1) per segment:

```clojure
;; Segment cost (inline in DP loop)
(let [preamble (get-preamble-width stacks s)
      gutter (get-in stacks [s :gutter] 0N)
      postamble (get-postamble-width stacks t)
      available (- system-width preamble gutter postamble)
      total-ideal (- (nth ideal-prefix t) (nth ideal-prefix s))
      scale-factor (/ available total-ideal)
      r (- scale-factor 1N)
      b (badness r params)
      d (demerits b (break-penalty-fn stacks s t) params)]
  ...)
```

This is the canonical cost definition under the ADR's contract, not an optimisation of a more complex one. The feasibility check guarantees no clamping, so the `r` computed from the sums is the ratio every stack actually receives.

### Width Allocation Implementation

```clojure
(defn allocate-widths
  "Allocates actual widths using proportional scaling (normalisation).
   
   Used after break selection to compute final positions for rendering.
   Under the feasibility contract, every column gap stays at or above its
   minimum, so all stacks satisfy actual_i ≥ min_i without clamping.
   
   preamble and postamble are the system's preamble[s] and postamble[t].
   The first stack (index 0) contributes :gutter. All three are reserved
   space that does not scale; all measures scale within the remaining space.
   
   Returns map with:
   - :preamble, :gutter, :postamble - reserved widths
   - :actuals - vector of actual widths for each measure"
  [system-stacks system-width preamble postamble]
  
  (let [;; Reserve gutter space for first stack
        gutter (get (first system-stacks) :gutter 0N)
        available (- system-width preamble gutter postamble)
        total-ideal (reduce + 0N (map :ideal system-stacks))
        scale-factor (/ available total-ideal)]
    
    {:preamble preamble
     :gutter gutter
     :postamble postamble
     :actuals (mapv (fn [stack]
                      (let [scaled (* (:ideal stack) scale-factor)]
                        ;; Defensive clamp: under feasibility contract, this never changes the value
                        (max scaled (:min stack))))
                    system-stacks)}))
```

**Why this approach**:

1. **Normalisation, not optimisation**: No degrees of freedom, no iteration, no convergence concerns
2. **Preserves proportionality by construction**: `actual_i / actual_j = ideal_i / ideal_j` for all i,j
3. **Reservation correctness**: System-start decorations and system-end cautionaries and barlines have exactly their required space
4. **Deterministic**: Same input always produces same output with no floating-point variation
5. **Fast**: O(K) where K ≈ 10-20, essentially zero cost
6. **Predictable**: Users can mentally predict allocation behavior
7. **Contained**: Within a fixed break configuration, a change to one stack alters only its own system's allocation

**Verification**:
```clojure
(let [{:keys [preamble gutter postamble actuals]}
      (allocate-widths stacks width preamble postamble)]
  (assert (= width (+ preamble gutter (reduce + 0N actuals) postamble))))
```

**Defensive clamp**:

The `max` clamp is purely defensive programming:
- The DP feasibility check ensures `scale_factor ≥ max(ρ_i)`, the largest column ratio over the system's stacks
- This keeps every column gap at or above its minimum, and so guarantees `scaled = ideal_i × scale_factor ≥ min_i` for all stacks
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

**Preamble, Gutter and Postamble Handling**:

Each stack carries `:gutter` (default 0N). Preamble width is computed on-demand and cached by (clef, keysig) combination. Only the first stack in a system contributes its gutter and determines the preamble; the postamble is determined by the breakpoint at which the system ends. All three:
- Are reserved from system width before scaling: `available = system_width - preamble - gutter - postamble`
- Do not scale—they are fixed space for clefs and key signatures, graphical decorations, and system-end cautionaries and barlines
- Do not participate in cost computation
- Are returned separately from `allocate-widths` for rendering coordination

**Baseline: Proportional Scaling (Normalisation)**:

The implementation uses proportional scaling as the primary width allocation strategy. This is normalisation (deterministic formula), not optimisation (iterative solver):

- **Formula**: `actual_i = ideal_i × (available / Σ ideal_i)`
- **Preamble, gutter and postamble reservation**: `available = system_width - preamble[s] - gutter[s] - postamble[t]`
- **Preserves proportionality**: All measures scale by same factor
- **No degrees of freedom**: Allocation is purely derived from break choice
- **Fast**: O(K) per system, essentially zero cost
- **Predictable**: Mental model matches implementation
- **Contained**: Within a fixed break configuration, a change to one stack alters only its own system's allocation

This approach may prove entirely adequate. If empirical testing reveals systematic visual problems, asymmetric optimisation can be considered, but complexity should not be added speculatively.

**Extension Point: break-penalty-fn**:

The algorithm accepts an optional `break-penalty-fn` parameter with signature `(fn [stacks s t] -> ratio)`. It returns the penalty `p` for selecting segment `s..t-1` as a system, which enters the demerits as `+p²` or `−p²`. The default `(constantly 0N)` produces purely geometric optimization. The segment cost parameters (`knuth-plass-parameters`) are a further optional argument. This ADR does not specify any concrete penalty functions; editorial policy is a separate concern that can evolve independently of the core algorithm.

**Feasibility Check**:

The DP uses a two-part feasibility check:
1. **Necessary**: `Σ min_i ≤ available` (minimums must fit after reserving preamble, gutter and postamble) - O(1) via prefix sums
2. **Sufficient for proportional scaling**: `scale_factor ≥ max(ρ_i)`, the largest column ratio over the segment's stacks - O(1) per segment, using a running maximum

The second condition ensures the scale factor is large enough that no column gap falls below its minimum, so no stack requires clamping. Without it, proportional scaling could produce allocations below minimums.

The running maximum works because the inner loop visits segments in a fixed order: for fixed `t`, `s` decreases from `t-1`, so each candidate segment `[s, t)` is the previous one extended by the single stack `s`. The maximum over the new segment is therefore `max(previous maximum, ρ_s)`, one comparison per step. Rescanning the segment instead would cost O(K) per segment and O(N×K²) in total.

**Segment Evaluation**:

Each candidate segment (s, t) requires:
1. **Reservation lookups**: O(1) to get `preamble[s]` and `gutter[s]`; `postamble[t]` is fixed for the inner loop
2. **Feasibility**: O(1) to extend the running `max(ρ_i)` by stack `s`
3. **Cost**: O(1) — the adjustment ratio from prefix sums, then badness and demerits from it

Every per-segment step is O(1). Cost computation uses the `ideal-prefix` sums; feasibility uses the running maximum carried by the inner loop.

**Rational Arithmetic Throughout**:
- All numeric literals use `N` suffix: `0N`
- Preserves exact arithmetic, no floating-point contamination
- Ensures deterministic outcomes across platforms
- Ratio growth managed through Clojure's automatic normalization (gcd)
- Demerits are rationals of degree six in `r` with the documented exponent, so their numerators and denominators grow faster than the widths'; evaluation remains O(1) arithmetic per segment
- The badness exponent is an integer, so badness is computed exactly; any approximation, such as for a non-integer exponent, must be deterministic

**nil Semantics for Infinity**:
- `nil` represents unreachable positions (cost = ∞)
- Avoids mixing `##Inf` (Double) with ratios
- Clean comparison: `(or (nil? x) (pair< new x))`, where each state holds a `[cost tie]` pair
- Type-safe throughout

**Transients for Local Mutation**:
- `best` and `prev` arrays use transients for performance
- Sequential processing only (no parallelism in DP loop)
- Thread safety at piece level, not within algorithm
- Converted to persistent at end

**Prefix Sums for O(1) Queries**:
- Precomputed once: O(N) time for `min-prefix` and `ideal-prefix` — and, under fitting rules other than the baseline, prefix sums of stretch and shrink
- Range sum in O(1): `(- (nth prefix t) (nth prefix s))`
- Enables O(1) computation of the adjustment ratio, and from it the badness and demerits
- Must use `0N` in reductions to maintain ratio domain
- Note: preamble and gutter are O(1) per-segment lookups, and postamble one lookup per outer iteration; none is prefix-summed (each depends on one end of the segment)
- The tie-break terms `2^−t` are precomputed once, one per breakpoint

**Early Termination**:
- The inner loop stops once `total-min` exceeds the full `system-width`
- Monotonicity: as `s` moves left, `total-min` increases (more stacks) and, under the assumption below, `system-width` does not increase; once the minimums exceed the full system width, they do so for every earlier `s`
- The bound is the full system width, not `available`. Since `available = system_width - preamble[s] - gutter[s] - postamble[t]` varies with `s`, a segment that fails against `available` because of a wide preamble or gutter at `s` can be followed by a feasible segment at `s-1`. Preamble, gutter and postamble are non-negative, so `available ≤ system_width`, and every feasible segment satisfies the full-width bound. The postamble is fixed for a given `t` and does not affect the bound's monotonicity
- The check against `available` is made at each step, as the necessary feasibility condition
- Reduces effective complexity to O(N×K), where K is the largest number of consecutive stacks whose minimums fit within a full system width (K ≈ 15 typical)
- **Assumption**: `system-width-fn` must be non-increasing as `s` moves left (for fixed `t`). This holds for typical policies (uniform width, narrower final system). If violated, early termination is invalid and the algorithm must check all `s` values.

**Complexity Analysis**:

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Precomputation | O(N) | Prefix sums for min and ideal; tie-break terms |
| Outer loop | O(N) | Each position once |
| Inner loop | O(K) typical | Early termination once minimums exceed the full system width |
| Reservation lookups | O(1) | Preamble and gutter per segment; postamble per outer iteration |
| Feasibility check | O(1) | Running max(ρ_i), extended by one stack per step |
| Cost computation | O(1) | Adjustment ratio from prefix sums; badness and demerits from it |
| **Total (worst case)** | O(N²) | Without early termination |
| **Total (typical)** | O(N×K) | With monotone width policy, K ≈ 15 |
| Page breaking | O(S²) | S = number of systems |

At N=800 measures, K=15: ~12,000 segment evaluations, each O(1) for both feasibility and cost.

## Relationship to Knuth-Plass

This section defines the relationship once. Elsewhere in this ADR, "Knuth-Plass" is shorthand for it.

**Classification**: Stage 3's baseline is an optimal-segmentation dynamic program in the Knuth–Plass family (Knuth and Plass, 1981; see [Prior Art](#prior-art)). Its cost is Knuth–Plass's: the badness of each system's adjustment ratio, combined with a per-system penalty and the breakpoint penalty into demerits, each O(1) per segment from prefix sums (see [Optimization Characterization](#optimization-characterization) and [Segment Cost Computation](#segment-cost-computation)).

**Mapping onto the Knuth–Plass primitives**:

| Knuth–Plass | Stage 3 |
|-------------|---------|
| Box | Atom |
| Natural width of glue | Ideal distance between atom origins |
| Shrinkability of glue | `ideal − min` for the gap |
| Legal breakpoint | Legal break position, normally a barline |
| Line length | System width less preamble, gutter and postamble |
| Adjustment ratio of a line | Adjustment ratio `r` of a system; `r = scale_factor − 1` in the baseline (below) |
| Badness, about `100 × \|r\|³`, capped | Badness `b(r) = min(b_max, c × \|r\|^e)`, starting from `c = 100`, `e = 3` |
| Demerits `(l + b)² ± p²` | Demerits `(l + b)² ± p²`, with `p` from `break-penalty-fn` |
| Line penalty `l` | Per-system penalty `l` |
| Tolerance | Optional bound on badness |

**Proportional scaling as a special case**: Proportional scaling is the case in which every glue's stretchability and shrinkability equal its natural width. The Knuth–Plass adjustment ratio then gives one uniform scale factor per system: a line of adjustment ratio `r` sets every gap to `natural × (1 + r)`, which is `ideal_i × scale_factor` with `r = scale_factor − 1`. In the baseline, each gap's `min` enters only as a floor on that uniform scale — the feasibility condition `scale_factor ≥ max(ρ_i)`, the largest column ratio — rather than as a per-gap shrinkability.

**In the baseline**: the adjustment ratio, cubic badness, demerits with the per-system penalty, breakpoint penalties through `break-penalty-fn`, and the tolerance are Knuth–Plass's own, with parameters to be settled by validation.

**Extensions available**: The remaining Knuth–Plass features are available extensions of the baseline, not departures from it:

- Separate stretch and shrink, with `ideal − min` as each gap's shrinkability ([Fitting Rules](#fitting-rules); see also [Alternative: Asymmetric Cost Optimization](#alternative-asymmetric-cost-optimization))
- Fitness classes, including classes for different kinds of system endings
- Looseness

**The scalar reduction is preserved**: Summing each stack's glue keeps the reduction to one scalar tuple per stack. Prefix sums of natural width, stretch and shrink give the adjustment ratio, and so the demerits, in O(1) per segment, and feasibility stays O(1) per segment — from prefix sums where no gap can shrink past its minimum, and from a running extremum otherwise, as in the baseline (see [Fitting Rules](#fitting-rules)).

**Exactness and DP state**: The baseline's cost has no term linking adjacent systems, so one DP node per breakpoint is enough for the result to be exact. Separate stretch and shrink keeps that. Two extensions need more state:

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

`ideal_width` encodes rhythmic proportionality, which has musical meaning. Choosing the proportional special case is what preserves it: all measures in a system share one scale factor, so their ratios hold by construction. Graphical decorations (gutter content), the preamble and the postamble are fixed overhead, outside the scaled glue.

Under the separate-stretch-and-shrink extension, compression toward `min_width` would cost proportionality, and the choice of shrinkability would express how much. Whether that extension is needed remains subject to the empirical validation described above.

**Architectural prerequisites**:

The Knuth-Plass algorithm solves a specific problem formulation: capacity-constrained segmentation of a 1-dimensional sequence with separable costs. Music layout, naively formulated, is not this problem—it couples vertical alignment, collision detection, semantic decisions, and geometry into mutual dependencies.

Ooloi's pipeline architecture transforms the problem. By the time Stage 3 executes:
- Vertical coordination is complete (Stages 1-2)
- Collision boundaries are determined (Stage 1)
- Semantic decisions (accidentals, beaming) are resolved (ADR-0035)
- Gutter requirements are computed (Stage 1)
- Preamble and postamble widths are known for every candidate system start and end
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
  - This corresponds to ADR-0037's split: the preamble, gutter and postamble are hard space, ideal widths are scalable space, and `\elemskip` per line is the scale factor per system. ADR-0037 treats only system-start and system-end material as hard; bar rules within a system scale with the measures.
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
| Stage 3 | DP, one node per breakpoint | Knuth–Plass demerits (l + b(r))² ± p², from each system's adjustment ratio | Yes |
| Stage 6 | DP over systems | Not yet specified | Not established |

**Why Stage 3 is exact**: its cost has no term linking adjacent systems, so one node per breakpoint loses nothing; every other input — `system-width-fn`, `preamble[s]`, `gutter[s]`, `postamble[t]`, the column ratios, `break-penalty-fn` — depends only on the candidate segment; forced breaks partition the problem into independent subproblems; prevented breaks only reduce the set of breakpoints; and the early-termination bound excludes only infeasible segments (see [Key Implementation Notes](#key-implementation-notes)). In this sense Stage 3 is an exact Knuth–Plass computation. Its cost is the baseline, not the full Knuth–Plass cost model; which extensions keep exactness at which cost in state is stated under [Relationship to Knuth-Plass](#relationship-to-knuth-plass).

**Why Stage 6 is not established**: its cost function is not specified, and page heights that depend on the page — a first page carrying a title, running headers — require the page number, or at least its parity, in the DP state. Both are open.

### What Is Specific to Ooloi

- The combination of the two lineages over measure-stack scalars, structured so that full Knuth–Plass remains available
- The preamble, gutter and postamble as fixed widths, known to the DP before it chooses breaks, that depend on where a system starts or ends
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
2. **Knuth–Plass demerits** - Badness of each system's adjustment ratio, with TeX's constants as starting points; the constants and any asymmetric treatment are settled by empirical validation
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
- **ADR-00XX: Horizontal Spacing** — forthcoming: upstream computation of min/ideal/gutter widths and column ratios
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
