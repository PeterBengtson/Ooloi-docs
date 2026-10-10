# ADR-0037: Measure Distribution Optimization

## Status

Accepted

Revised October 2026: Hegazy and Gourlay's spring model is adopted as the baseline fitting rule. Each column gap is a spring with an ideal width and a blocking width, which may exceed it; the former baseline, one scale factor per system, is kept as an alternative. See [Gaps as Springs](#gaps-as-springs) and [The Spring Model](#the-spring-model).

## Table of Contents
- [Context](#context)
  - [Problem Statement](#problem-statement)
  - [Architectural Reduction](#architectural-reduction)
  - [What "Optimal" Means](#what-optimal-means)
  - [Optimization Characterization](#optimization-characterization)
  - [Calibrating the Adjustment Ratio](#calibrating-the-adjustment-ratio)
- [Decision](#decision)
  - [Architectural Position](#architectural-position)
  - [Interface Contract](#interface-contract)
  - [Gaps as Springs](#gaps-as-springs)
  - [Constraints Spanning Several Columns](#constraints-spanning-several-columns)
  - [Ideal Width Semantics](#ideal-width-semantics)
  - [The Gutter: System-Start Width Delta](#the-gutter-system-start-width-delta)
  - [The Preamble: System-Start Clef and Key Signature Width](#the-preamble-system-start-clef-and-key-signature-width)
  - [The Postamble: System-End Width](#the-postamble-system-end-width)
  - [Width Allocation: Applying the Adjustment Ratio](#width-allocation-applying-the-adjustment-ratio)
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
  - [Feasibility](#feasibility)
  - [End-of-System Width](#end-of-system-width)
  - [Joint System and Page Breaking](#joint-system-and-page-breaking)
- [Implementation](#implementation)
  - [Core Dynamic Programming Structure (Stage 3)](#core-dynamic-programming-structure-stage-3)
  - [Solving for the Adjustment Ratio](#solving-for-the-adjustment-ratio)
  - [Segment Cost Computation](#segment-cost-computation)
  - [Width Allocation Implementation](#width-allocation-implementation)
  - [Break Reconstruction](#break-reconstruction)
  - [Key Implementation Notes](#key-implementation-notes)
- [Relationship to Knuth-Plass](#relationship-to-knuth-plass)
- [Prior Art](#prior-art)
  - [Optimal Breaking by Dynamic Programming](#optimal-breaking-by-dynamic-programming)
  - [The Space Model](#the-space-model)
  - [The Spring Model](#the-spring-model)
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
│         │              min, natural, ordinary ideal,                    │
│         │              prestretched gaps, gutter per stack              │
│         ▼                     ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │               Stage 3: SYSTEM BREAKING (Single)                 │    │
│  │                                                                 │    │
│  │   Input: N measure stacks with gutter delta and                 │    │
│  │          summaries of their gaps as springs, all ratios         │    │
│  │                                                                 │    │
│  │   ┌─────────────────┐                                           │    │
│  │   │  System Break   │                                           │    │
│  │   │     DP O(N²)    │                                           │    │
│  │   └─────────────────┘                                           │    │
│  │            │                                                    │    │
│  │            ▼                                                    │    │
│  │   ┌────────────────────────────────────────────────────┐        │    │
│  │   │       Adjustment Ratio per System                  │        │    │
│  │   │   avail = sys_width - preamble[s] - gutter[s]      │        │    │
│  │   │                     - postamble[t]                 │        │    │
│  │   │   r solves Σ g_j(r) = avail                        │        │    │
│  │   └────────────────────────────────────────────────────┘        │    │
│  │                                                                 │    │
│  │   Output: System breaks + adjustment ratios                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Stage 4: ATOM POSITIONING (Fan-out per Measure)                │    │
│  │  Apply each system's r to its gaps, compute coordinates         │    │
│  │  g_j = max(b_j, w_j + r × k_j)                                  │    │
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
- Its column gaps, each a spring with an ideal width `w_j`, from the durations, and a blocking width `b_j`, the distance at which the atoms on either side would collide; `b_j` may exceed `w_j` (see [Gaps as Springs](#gaps-as-springs))
- `min_width`: Hard lower bound from collision detection, `Σ b_j` (atoms cannot overlap)
- `ideal_width`: Target spacing based on musical density and conventional engraving practice, `Σ w_j`
- `gutter_width`: Additional fixed space required when stack begins a system (default 0N)
- Cost model: the badness of each system's adjustment ratio, combined into demerits (Knuth–Plass; see the segment cost model below)

**Distribution constraints**:
- System capacity: Fixed maximum width per system
- Page capacity: Fixed maximum height per page (number of systems)
- Stack atomicity: Measure stacks cannot be subdivided

**Explicit exclusions**:
Connecting elements (ties, slurs, hairpins, beams, glissandi, ottava lines) do not participate in the distribution optimization. They are computed in Stage 5 after positions are finalized, and adapt to the determined geometry.

**Objective**:
Minimize total demerits — each system's cost, derived from how far it is stretched or compressed from its ideal proportions, measured against how far it can be — while satisfying capacity constraints. The intent is that minimizing total demerits produces layouts perceived as stable and professionally typeset.

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
Parallel collision detection produces measure stack metrics. Each of N stacks emerges with its column gaps as springs, an ideal and a blocking width for each, summarised for Stage 3 as a few sums and a short list of prestretched gaps ([Interface Contract](#interface-contract)), plus `gutter_width` for additional system-start space.

**Stage 1 is width-complete**: It computes all horizontal space requirements for atoms, including lyrics, dynamics, articulations, and fixed-size anchors for spanners (e.g., "sffzpp<" letters, minimum crescendo wedge width). Stage 1 cannot compute full spanner geometry (that depends on final positions from Stage 4), but it must include all horizontal space the atom requires, including spanner attachment points.

**Stage 5 is height-complete**: After Stage 4 positions atoms, Stage 5 computes connecting element geometry and determines final vertical extent. System heights are definitive after Stage 5.

The vertical alignment problem is solved completely before distribution begins. This reduction is enabled by:
- Immutable data structures eliminating race conditions during parallel processing
- Rational arithmetic preserving exact proportions without floating-point accumulation
- STM coordination ensuring atomic reads of hierarchical musical structure

**Stage 3: System Breaking**
With vertical coordination complete, the problem reduces to:
- A 1-dimensional sequence of N stack summaries: sums (minimum, natural width, ordinary ideal), the gutter, and each stack's prestretched gaps
- Preamble and gutter subtraction at the start of each system, and postamble subtraction at its end, before the springs are fitted
- Capacity-constrained segmentation into systems using Knuth-Plass DP
- Cost function on each system's adjustment ratio (badness and demerits)
- No feedback from geometry to distribution logic
- Output: System break decisions and the adjustment ratio of each system

**Stage 4: Atom Positioning**
Applies each system's adjustment ratio from Stage 3 to every gap in it, positioning atoms at their actual coordinates. Non-connecting visual elements (noteheads, accidentals, dynamics) are positioned from the resulting gap widths. These elements do not influence distribution.

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

### What "Optimal" Means

In this ADR, *optimal* always means minimal total demerits under the segment cost model specified below — badness of each system's adjustment ratio, the per-system penalty and the breakpoint penalties from `break-penalty-fn`, with the parameters in force — given the metrics Stages 1–2 supply. It is a statement about that cost function and those inputs, not about layout quality in any wider sense: a different cost function, or different upstream metrics, would have a different optimum.

### Optimization Characterization

The distribution problem exhibits structure that admits polynomial-time exact optimization of the reduced subproblem:

**Discrete break selection**:
Dynamic programming over the sequence of N measure stacks determines optimal system break points. For each potential break location, the algorithm evaluates whether remaining measures fit within capacity (after reserving preamble, gutter and postamble space) and computes the resulting demerits. Optimal substructure holds: optimal solution for measures 1..k combined with optimal solution for measures k+1..N yields optimal solution for 1..N.

Break selection relies on **separable system costs** and **optimal substructure**, not convexity. The dynamic programming algorithm computes the break configuration of least total cost for the discrete segmentation problem, under the stated cost function and given its inputs.

Complexity: O(N²) for system breaks, O(S²) for page breaks where S = number of systems.

**Continuous width allocation**:
Within each system `[s, t)` determined by break selection, every gap receives the width its spring gives at the system's adjustment ratio `r`: the least value at which the gaps exactly fill the space remaining after preamble, gutter and postamble reservation ([Gaps as Springs](#gaps-as-springs)):

```
available = system_width - preamble[s] - gutter[s] - postamble[t]
g_j(r) = max(b_j, w_j + r × k_j)        k_j = σ × w_j when r ≥ 0, max(0, w_j − b_j) when r < 0
r = least r with Σ g_j(r) = available
```

This is not optimization: `Σ g_j` is continuous, piecewise linear and non-decreasing in `r`, and `r` is found exactly. Under stretching, every gap that is free to stretch does so in proportion to its ideal width, so rhythmic proportions are kept among those gaps; a gap held open by an accidental keeps its blocking width until the rest of the system catches up with it. Under compression, every gap gives up the same share of its room above its minimum.

**Segment cost model**:
The cost of a system is Knuth–Plass's. It is judged by its **adjustment ratio** `r`: how far its gaps are stretched or compressed, measured against how far they can go. In the glue model of Knuth and Plass, `r` is one division:

```
r = (available − Σ natural) / Σ stretch     when available ≥ Σ natural   (stretching)
r = (available − Σ natural) / Σ shrink      when available <  Σ natural   (shrinking)
```

Under springs it is the value that solves `Σ g_j(r) = available`, which Hegazy and Gourlay call the fitting force. The two coincide whenever no gap is blocked or prestretched. In the baseline:

- **Shrinking is always one division.** Every gap's shrink is its room above its minimum, `max(0, w_j − b_j)`, so every gap reaches its minimum at the same value, `r = −1`, and none blocks before then: `r = (available − Σ natural) / (Σ natural − Σ min)`, from prefix sums. `r = −1` is exactly `available = Σ min`, the system's collision limit, so Knuth–Plass's condition `r ≥ −1` is the feasibility condition `Σ min ≤ available` ([Feasibility](#feasibility)).
- **Stretching is one division until a prestretched gap starts to stretch.** Every ordinary gap (`b_j ≤ w_j`) stretches by `σ × w_j` per unit of `r` from `r = 0`. A prestretched gap holds its blocking width until `r` reaches its kink, `(b_j − w_j) / (σ × w_j)`, and stretches from there. While `r` stays below the segment's smallest kink, `r = (available − Σ natural) / (σ × Σ ordinary ideal)`; beyond it, the kinks are walked in order ([Solving for the Adjustment Ratio](#solving-for-the-adjustment-ratio)).

`r` is negative under compression and positive under stretching. It depends on the segment alone, so the DP stays exact.

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

**Why the badness of `r`**: the adjustment ratio is what a reader sees — how far a system is stretched or compressed, measured against how far it can go. A cost weighted by content, such as `Σ (actual_i − ideal_i)²`, which with every gap stretched in proportion is `(σ × r)² × Σ ideal_i²`, makes two systems with the same adjustment ratio cost differently according to how their content is divided into measures: the same music in more, shorter measures has a smaller `Σ ideal_i²` and so costs less at the same stretch. That difference has no musical justification.

**Parameters**: the badness coefficient `c`, exponent `e`, cap `b_max`, per-system penalty `l`, tolerance and stretchability `σ` are parameters. The documented starting points of the first five are TeX's implementation of Knuth–Plass: `c = 100`, `e = 3` (badness "a reasonably close approximation to 100(t/s)³"), `b_max = 10000` ("infinitely bad"), `l = 10` (plain TeX's `\linepenalty`) and tolerance 200 (plain TeX's `\tolerance`). The starting point of `σ` is `1/2` ([Calibrating the Adjustment Ratio](#calibrating-the-adjustment-ratio)). TeX's constants are meaningful only under that calibration. The empirical validation decides all six ([Evaluation Criteria](#evaluation-criteria)).

**Exact arithmetic**: `σ` and every ideal and blocking width are rationals, and so is every kink; `r` is the solution of a linear equation on one piece of a piecewise-linear function, so it is exact. Demerits are exact rationals — with `e = 3`, of degree six in `r` — and are still O(1) per segment. Any approximation, for example of a non-integer exponent, must be deterministic, so that every platform computes identical demerits.

**Additive separable cost model**:
- **System-level cost**: each system's demerits depend only on its own adjustment ratio and the penalty of its break — that is, on the segment alone. The `min` value participates in feasibility checking; the `preamble`, `gutter` and `postamble` values determine the available space, and through it `r`; none of them enters the cost otherwise.
- **DP operates on system-level cost units**: the dynamic programming algorithm sums per-system demerits to compute the total.

This separability enables independent evaluation of candidate break points during dynamic programming.

**Page breaking: Second-order segmentation**:
Page breaking is treated as a **second segmentation pass over the system sequence, never interleaved with system breaking**. After optimal system breaks are determined, page breaks are computed independently by a second dynamic programme over the system sequence, whose cost function is not yet specified. The two are separate because page breaking needs actual system heights, which exist only once Stage 5 has built the systems; optimising both together would have to work from estimated heights (see [ADR-0028 §Trade-offs](0028-Hierarchical-Rendering-Pipeline.md#trade-offs)).

**Deterministic outcomes**:
Given identical input (measure stacks with their bounds, system/page capacities), the algorithm produces identical output. This determinism arises from:
- Rational arithmetic (no floating-point nondeterminism)
- Immutable data structures (no timing-dependent state)
- Deterministic tie-breaking: equal costs are resolved by a secondary cost that differs for any two configurations (see [Deterministic Tie-Breaking](#deterministic-tie-breaking))
- Closed-form allocation: each system's adjustment ratio is the exact solution of a piecewise-linear equation (no iterative solver convergence)

**Edit locality**: Preserving break decisions away from an edit, to minimise perceptual layout "jitter" during editing, is a goal. The DP does not provide it: a global optimum is not local. See [Edit Locality](#edit-locality).

### Calibrating the Adjustment Ratio

TeX's constants assume that the adjustment ratio `r` is measured against the limits of the glue. `r = −1` is the tightest line TeX permits, because glue cannot shrink beyond its shrinkability; at `r = 1` a line's glue is stretched by its full stretchability, with badness 100. If a gap's stretch and shrink both equalled its natural width, `r` would measure a deviation from the ideal rather than from those limits, and the constants would lose their meaning:

- **Compression would be nearly free.** A system at its collision limit with its gaps at 80% of their ideal widths would have `r = −0.2` and badness 0.8, which TeX's scale calls nearly perfect.
- **Tolerance 200 would admit about 126% stretch**: it bounds `100 × |r|³` by 200, so `|r| ≤ 1.26`.
- **The per-system penalty would dominate.** `l = 10` contributes at least `l² = 100` per system, while badness stays below 1 up to a deviation of about 21%. The DP would chiefly minimise the number of systems. With content spread evenly over systems of one width, it would compress systems by up to about 18% at about 10 systems, and about 15% at about 30, rather than add one.

The calibration gives `r` its meaning at both ends, and TeX's constants (`c`, `e`, `b_max`, `l` and the tolerance) are meaningful only under it:

- **Shrinking**: each gap's shrink is its room above its minimum, `max(0, w_j − b_j)`, so `r = −1` is where every gap reaches its minimum together: the system's collision limit. Knuth–Plass's limit on shrinking and this ADR's feasibility condition are then the same point.
- **Stretching**: stretch has no collision limit, so its scale is a parameter, the stretchability `σ`: each ordinary gap's stretch is `σ × w_j`. The documented starting point is `σ = 1/2`. A system whose gaps are all free to stretch then reaches `r = 1`, with badness 100, at one and a half times its ideal width: the same badness as a system at its collision limit. Tolerance 200 admits stretch up to about 63%. A prestretched gap contributes to the stretch only once `r` has passed its kink. No engraving source fixes the value of `σ`: it is where the documented constants start, and the empirical validation decides it ([Evaluation Criteria](#evaluation-criteria)).

How the calibration applies under the alternative fitting rules is stated under [Fitting Rules](#fitting-rules).

## Decision

### Architectural Position

Measure distribution optimization (system breaking) is Stage 3 of the hierarchical rendering pipeline ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)). It receives measure stack metrics from Stages 1-2 and produces system break decisions and an adjustment ratio per system, which Stage 4 applies to each system's gaps for atom positioning.

The algorithm implements **capacity-constrained segmentation with spring-based width allocation**:
1. Dynamic programming (Stage 3) determines optimal system break points, accounting for preamble and gutter space at system starts and postamble space at system ends
2. Each system's adjustment ratio is solved exactly from its gaps' springs within the available space (after preamble, gutter and postamble reservation)
3. Atom positioning (Stage 4) applies each system's adjustment ratio to its gaps to compute actual positions
4. System heights (Stage 5) are computed from final geometry
5. Page breaking (Stage 6) is a second dynamic programme, over systems with actual heights, whose cost function is not yet specified

### Interface Contract

```clojure
;; Input from Stages 1-2 (computed by ADR-00XX, forthcoming):
stacks ;; Vector of stack maps

;; Each stack map summarises the stack's column gaps, each gap j a spring
;; with ideal width w_j and blocking width b_j (see Gaps as Springs):
{:min ratio                 ;; Σ b_j: collision floor for the measure's content
 :natural ratio             ;; Σ max(w_j, b_j): width at r = 0
 :ideal ratio               ;; Σ w_j: rhythmic target for the measure's content
 :ordinary-ideal ratio      ;; Σ w_j over gaps with b_j ≤ w_j: what stretches from r = 0
 :prestretched [[w b] ...]  ;; gaps with b_j > w_j, as [ideal blocking] pairs
 :gutter ratio              ;; Additional space when at system start (default 0N)
 :measure-index int}        ;; Original measure index (for debugging/tracing)

;; Output:
{:breaks [break-positions]    ;; Vector of indices where systems start
 :ratios [r ...]              ;; Adjustment ratio of each system (ratio)
 :cost total-demerits}        ;; Total demerits (ratio)
```

**Width component semantics**:

- `:min` — collision floor for the measure's semantic content: the sum of its gaps' blocking widths
- `:natural` — the stack's width at `r = 0`: each gap at its ideal width, or at its blocking width where that is larger
- `:ideal` — rhythmic target for the measure's semantic content: the sum of its gaps' ideal widths
- `:ordinary-ideal` — the ideal width of the gaps that are free to stretch from `r = 0`
- `:prestretched` — the gaps whose blocking width exceeds their ideal width, typically a short note followed by one carrying an accidental; Stage 3 derives their kinks from `σ`
- `:gutter` — additional space required when this measure appears first on a system (default 0N); this space is reserved for graphical decorations (courtesy accidentals, tie continuations) and does not scale

Stage 4 needs every gap's `w_j` and `b_j`, not only the summary; both stay cached with the stack (see [Caching and Incremental Recompute](#caching-and-incremental-recompute)).

The gutter is *additional* to the measure's width, not part of it. When a measure appears first on a system, the system must accommodate `gutter[s] + actual[s]` for that measure.

**Preconditions** (guaranteed by upstream stages):
- `:ideal` must be positive for all stacks (so every segment can stretch)
- `:min` must be positive for all stacks
- The summary is consistent with the stack's gaps: `:natural = :min + Σ max(0, w_j − b_j)`, and `:ideal = :ordinary-ideal + Σ w` over `:prestretched`
- Every prestretched pair has `0 ≤ w < b`; a pair with `w = 0` is fixed space, which never stretches
- `:gutter` must be non-negative (default 0N)
- At least one feasible segmentation must exist (the entire sequence fits on some number of systems)

The computation of these values is handled by upstream stages, and will be specified in a forthcoming ADR on horizontal spacing (ADR-00XX). If preconditions are violated, behavior is undefined.

### Gaps as Springs

Every column gap in a measure stack is a spring in the sense of Hegazy and Gourlay (see [The Spring Model](#the-spring-model)). The springs attach to the origins of the atoms on either side, not to their outer edges, because what the eye measures is the distance from one note to the next; the atoms themselves are boxes that cannot pass through each other. A gap `j` has:

- an **ideal width** `w_j`, the distance between atom origins that the durations call for (Stage 1);
- a **blocking width** `b_j`, the distance at which the atoms on either side would collide, the largest such distance over the stack's staves (Stages 1–2);
- a **stretch** `σ × w_j` and a **shrink** `max(0, w_j − b_j)`, the baseline elasticity ([Fitting Rules](#fitting-rules)).

Under an adjustment ratio `r` its width is

```
g_j(r) = max(b_j, w_j + r × σ × w_j)            when r ≥ 0
g_j(r) = max(b_j, w_j + r × max(0, w_j − b_j))  when r < 0
```

A gap is **ordinary** when `b_j ≤ w_j`. It is **prestretched** when `b_j > w_j`: the atoms need more room than the durations call for, as when a short note is followed by one carrying an accidental. A prestretched gap holds its blocking width under every compression, and under stretching up to its **kink**, `κ_j = (b_j − w_j) / (σ × w_j)`, where its spring catches up with the collision; beyond that it stretches like any other gap. The accidental takes the room it needs and no more, and once the system is loose enough it costs nothing.

This is the engraver's principle, that everything belonging to a note, accidentals included, should fit within its proportional allocation, made exact: where it cannot fit, the allocation yields only as far as the collision requires.

**What the springs replace.** Before this revision the baseline required `b_j ≤ w_j` for every gap and gave every gap the same proportional stretch and shrink, so that a system had one scale factor. A prestretched gap could not be represented at all, and a gap whose minimum equalled its ideal blocked compression of its whole system. The former baseline survives as an alternative under [Fitting Rules](#fitting-rules).

**Summaries for Stage 3.** Stage 3 never needs the gaps one by one. A segment's width at `r` depends on three sums and on its prestretched gaps:

```
Σ g_j(r) = Σ natural + r × (Σ natural − Σ min)                             when −1 ≤ r < 0
Σ g_j(r) = Σ natural + r × σ × Σ ordinary ideal
           + Σ over prestretched gaps with κ_j < r  of  σ × w_j × (r − κ_j)    when r ≥ 0
```

So each stack carries `:min`, `:natural`, `:ordinary-ideal` and its `:prestretched` pairs ([Interface Contract](#interface-contract)), and the sums come from prefix sums.

### Constraints Spanning Several Columns

Some minimum distances span more than one gap: a long syllable sung across a melisma, a dynamic or a word of text that must not run into the next, material crossing between staves. A blocking width belongs to one gap, so such a constraint is reduced to blocking widths before Stage 3 sees it, in Stage 2, in the way LilyPond's spacer reduces its rods (`Simple_spacer::add_rod` in `lily/simple-spacer.cc`):

1. For a constraint of distance `D` over gaps `j..k`, find the least `r_D` at which `Σ g_i(r)` over those gaps reaches `D`, using their springs alone and the same exact solver as Stage 3 ([Solving for the Adjustment Ratio](#solving-for-the-adjustment-ratio)). If `Σ b_i ≥ D` already, the constraint is met and nothing changes.
2. Raise each spanned gap's blocking width to its width at that ratio, `b_i ← g_i(r_D)`; its shrink follows from the new blocking width.

Afterwards the span is at least `D` at every `r ≥ −1`: no spanned gap is narrower than before at any ratio, and each holds at least its share at `r_D`. A raised gap may become prestretched, with its kink at `r_D`. The reduction is conservative: in a tightly compressed system the spanned gaps can hold more room than the constraint strictly needs, because each is fixed at its share at `r_D` rather than free to trade with its neighbours. LilyPond's source accepts the same loss, on the grounds that the denser result looks worse anyway.

Overlapping constraints are applied in a fixed order (by first gap, then by last), so the result is deterministic. Each raises blocking widths only, and raising a blocking width never narrows a gap at any ratio, so the order affects how room is shared, never whether a constraint holds.

```clojure
(defn reduce-span-constraint
  "Raises the blocking widths of gaps [j, k) so that together they are at
   least d wide at every adjustment ratio r ≥ -1 (Stage 2)."
  [gaps j k d sigma]
  (let [{:keys [min natural ordinary-ideal prestretched]} (stack-summary (subvec gaps j k))]
    (if (>= min d)
      gaps
      (let [r-d (solve-ratio d min natural ordinary-ideal
                             (add-kinks (sorted-map) prestretched sigma) sigma)]
        (reduce (fn [gs i]
                  (update gs i (fn [{:keys [w b] :as g}]
                                 (assoc g :b (gap-width w b r-d sigma)))))
                gaps
                (range j k))))))
```

Because `r_D` depends only on the springs inside the span, the reduction is independent of where systems break, and Stage 3's inputs stay fixed before it starts. A constraint that crosses a barline is reduced over the gaps of both stacks: exact within a system and conservative at a break, where the constraint may need less (a lyric extender, say) or be carried by the postamble and gutter. An edit that changes such a constraint changes both stacks, which is a range edit for [Exact Re-optimisation After an Edit](#exact-re-optimisation-after-an-edit).

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
Collisions are resolved upstream (Stage 1), so the distribution never has to discover one. Where the music needs more room than proportion allows, the blocking width says exactly how much, and the springs give it that room and no more ([Gaps as Springs](#gaps-as-springs)).

**Constraint structure**:
- **Hard constraint**: every gap at or above its blocking width, so `actual_width ≥ min_width` (collision prevention)
- **Soft preservation**: every gap near its ideal width, departing from it by its spring's elasticity (proportionality maintenance)
- **No upper bound**: gaps may exceed their ideal widths under stretching

### The Gutter: System-Start Width Delta

When a measure appears first on a system, it may require additional space for graphical decorations. This additional space is the **gutter**—a fixed-width region that follows the preamble (clefs and key signatures) and precedes the measure's scaled content.

System layout for a system of stacks `[s, t)`:

```
┌───────────┬──────────┬─────────────┬─────┬─────────────┬────────────┐
│ PREAMBLE  │  GUTTER  │  measure s  │ ... │ measure t-1 │ POSTAMBLE  │
│  (fixed)  │ (fixed)  │  (springs)  │     │  (springs)  │  (fixed)   │
└───────────┴──────────┴─────────────┴─────┴─────────────┴────────────┘
      ↑          ↑           ↑                                 ↑
      │          │           │                                 └─ postamble[t] (does not scale)
      │          │           └─ actual[i] = Σ g_j(r) over the gaps of stack i
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
r = least r with Σ g_j(r) = available       ;; One ratio for every gap in the system

actual[i] = Σ g_j(r) over stack i's gaps     ;; Each gap by its own spring
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

All three are fixed overhead subtracted before the springs are fitted. None participates in cost computation.

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

### Width Allocation: Applying the Adjustment Ratio

Once breaks are chosen, every gap of a system `[s, t)` receives the width its spring gives at the system's adjustment ratio, within the space remaining after preamble, gutter and postamble reservation:

```
available = system_width - preamble[s] - gutter[s] - postamble[t]
r = least r with Σ g_j(r) = available          (from Stage 3)
actual_j = g_j(r) = max(b_j, w_j + r × k_j)
```

The preamble, gutter and postamble are fixed overhead; none participates.

This is **derivation, not optimisation**:
- Deterministic: `r` is the exact solution of a piecewise-linear equation, already computed by Stage 3
- No gap falls below its blocking width, by construction: the `max` is part of the spring, not a defensive clamp
- Under stretching, every gap free to stretch keeps its proportion to its ideal width; a gap held open by an accidental keeps its blocking width until the system has caught up with it
- Under compression, every gap gives up the same share of its room above its minimum
- No iteration, no convergence, no tuning beyond the documented parameters
- Contained: within a fixed break configuration, a change to one stack alters only its own system's adjustment ratio
- Near-zero computational cost: O(gaps) per system

**Notes on the cost**: the system's cost is the badness of its single adjustment ratio, combined into demerits (see the segment cost model under [Optimization Characterization](#optimization-characterization)), not a sum over its gaps or stacks. The preamble, gutter and postamble affect the cost only through the available width, and so through `r`.

**Relationship to Knuth-Plass**: a spring is Knuth–Plass glue with a blocking width; where no gap is blocked or prestretched, the springs are glue exactly. See [Relationship to Knuth-Plass](#relationship-to-knuth-plass).

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

2. **Width allocation is parameterized per system** after breaks are chosen. The springs are fitted identically whatever the system width.

3. **Determinism is preserved**: Given the same break configuration and per-system width policies, allocation produces identical results.

4. **No algorithmic complexity added**: The DP evaluates `(s, t)` segments against the width available for that system position, minus preamble, gutter and postamble. The springs fill the provided width directly.

**Per-system width policy**:

**Any system may have a distinct width policy** without special-casing, the final system included. Heterogeneous system widths need no additional mechanism because allocation is parameterized by width, not built around a fixed one.

**Expressivity without complexity**:

This architectural freedom enables:
- Natural handling of the final system of a piece or movement (avoid excessive stretch)
- Editorial control over system widths for specific musical reasons
- Adaptation to margins and layout settings
- Future extensions (e.g., systems of varying width for visual effect)

All while preserving the core property: **the springs distribute whatever width is provided according to each gap's ideal width and its collision limit**, with preamble and gutter space reserved for system-start elements and postamble space for system-end cautionaries.

### Alternative: Asymmetric Cost Optimization

An alternative approach would treat width allocation as a constrained optimization problem:

```
minimise Σ f(ideal_i - actual_i)
subject to: Σ actual_i = available, actual_i ≥ min_i
```

where `f` is asymmetric: compression toward `min_width` penalized more heavily than expansion beyond `ideal_width`.

This general formulation is broader than the springs, which already give each gap its own stretch and shrink and a collision limit while keeping allocation in closed form ([Gaps as Springs](#gaps-as-springs)); the costs listed below apply to the general formulation.

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

This refinement should be considered **only if empirical evaluation** of the springs reveals systematic visual problems that justify the complexity. The baseline approach should be implemented first, tested on real scores (simple → complex → Elektra), and evaluated for acceptability.

If the springs prove adequate, asymmetric optimization becomes unnecessary complexity. If visual problems emerge, the specific issues will guide cost function design rather than speculative theory.

**When the springs might be insufficient**:

1. **Extreme compression**: as a system approaches `r = −1`, every gap gives up the same share of its room above its minimum, so long values, which have more room, give up more. Whether that is right, or short values should give way first, is an engraving question ([Fitting Rules](#fitting-rules)). Since stacks are heterogeneous (some staves dense, others sparse), visual degradation affects only the dense staves within each stack.

2. **Mixed density across stacks**: a system with one very dense stack and several sparse ones compresses the sparse ones more in absolute terms, since they have more room. Within the dense stack, some staves may be heavily compressed while others remain adequate.

3. **Cliff avoidance**: a gap with little room above its minimum gives up little. Under the former baseline, one such gap blocked compression of its whole system; the springs avoid that.

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

   Each group becomes a single 'super-stack' with aggregated metrics:
   the sums add, the prestretched gaps are concatenated, and only the
   first stack in the group contributes :gutter."
  [stacks no-break-ranges]

  (let [grouped (apply-grouping stacks no-break-ranges)]
    (mapv (fn [group]
            {:min (reduce + 0N (map :min group))
             :natural (reduce + 0N (map :natural group))
             :ideal (reduce + 0N (map :ideal group))
             :ordinary-ideal (reduce + 0N (map :ordinary-ideal group))
             :prestretched (into [] (mapcat :prestretched) group)
             :gutter (:gutter (first group))
             :member-stacks group})
          grouped)))
```

The DP operates on groups; after breaks are determined, groups expand back to constituent stacks for width allocation.

**Width Overrides: Upstream Modification**

User adjustments to individual measure widths ("stretch this measure," "compress this passage") belong upstream of Stage 3, not within it. Stage 3 consumes the stack width metrics; editorial overrides modify these inputs:

```clojure
(defn apply-width-overrides
  "Applies user width overrides to a stack's gaps, before its summary is formed.

   Overrides may specify:
   - :ideal-override - New ideal width for the stack: every gap's w_j is
                       scaled by ideal-override / ideal, so their ratios hold
   - :min-override   - New minimum width for the stack: a constraint of that
                       distance across all its gaps, reduced like any other
                       constraint spanning several columns, so it can raise
                       blocking widths but never lower them

   gaps: vector of {:w ideal, :b blocking}. Returns the overridden gaps."
  [gaps {:keys [ideal-override min-override]} sigma]
  (let [ideal (reduce + 0N (map :w gaps))
        gaps (if ideal-override
               (let [k (/ ideal-override ideal)]
                 (mapv #(update % :w * k) gaps))
               gaps)]
    (if min-override
      (reduce-span-constraint gaps 0 (count gaps) min-override sigma)
      gaps)))
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
2. **Then**: Apply width overrides to the stacks' gaps, and form their summaries
3. **Then**: Group measures with prevented breaks into super-stacks
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

1. **Stage 1** computes exact measure content (each gap's ideal and blocking width) + exact gutter delta
2. **Stage 3** has *complete knowledge* of every scenario (mid-system, system-start and system-end) for every measure
3. **Stage 3** computes the system breaks that are optimal for its cost function, given these inputs - not heuristic, not iterative
4. **Stage 4** positions atoms using each system's adjustment ratio from Stage 3
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

* its gaps' ideal and blocking widths, `w_j` and `b_j`, after the reduction of constraints spanning several columns (Stages 1–2)
* its summary for Stage 3: `min`, `natural`, `ideal`, `ordinary-ideal` and its prestretched gaps
* `gutter_width` — additional space for system-start decorations
* `actual_width` — realized width after Stages 3–4 (each system's adjustment ratio applied to its gaps by Stage 4)

This cache is authoritative at the Stage 3-4 boundary: Stage 3 consumes the summaries and gutters, together with the preamble and postamble for each candidate system, and produces break assignments and adjustment ratios; Stage 4 applies each system's ratio to its gaps to produce `actual` positions; Stage 5 consumes `actual` and never influences Stages 3-4.

#### Cached Invariants

For each stack `i` in a system `[s, t)` with adjustment ratio `r`:

* `r ≥ −1`, so every gap of the stack is at or above its blocking width, and `actual_width_i ≥ min_width_i`
* `actual_width_i = Σ g_j(r)` over the stack's gaps
* the system's actual widths sum to `available = system_width - preamble[s] - gutter[s] - postamble[t]`

The cache additionally implies that per-stack metrics are **stable across edits** unless the edited content is in that stack, or in a neighbour joined to it by a constraint across the barline.

#### Incremental Update Consequences

Edits affect Stages 3-4 only through changes to stack metrics:

1. **Local edit**: modifying or inserting notation in a measure updates only the affected stack's upstream-derived width values (and its neighbour's, where a constraint spans the barline between them), and the postamble of any system boundary at which that change takes effect.

2. **Fast path (break stability)**: Stage 3 is re-run over the cached metrics. If the resulting break configuration is unchanged, then:

   * the adjustment ratio of the system containing the edited stack may change, and with it the `actual_width` of every stack in that system; no other system's ratio changes
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
* plus **O(K)** (where `K ≈ measures/system`) allocation for each system whose membership or adjustment ratio changed

#### Relationship to Edit Locality

This caching strategy makes recomputation after an edit cheap: re-running Stage 3 costs scalar arithmetic over cached aggregates, and no upstream work is repeated for stacks whose content did not change. It does not make the result local. A re-run may change breaks anywhere in the score; the mechanisms that keep break decisions stable are specified under [Stability and Quality Extensions](#stability-and-quality-extensions).

## Stability and Quality Extensions

The baseline specified above is complete and exact on its own. The mechanisms in this section extend it for stability under editing and for layout quality. Each is specified here with its cost and its effect on exactness. Which are enabled by default, and with which parameters, is decided during implementation by the empirical validation in [Consequences](#consequences) — simple scores, then complex ones, then Elektra — against the criteria below; the one exception is [Feasibility](#feasibility), which states the baseline's own condition.

### Evaluation Criteria

Visual judgement, together with measurements taken on the same scores:

- The distribution of per-system adjustment ratios `r`: minimum, maximum, variance
- The difference between adjacent systems' `r`
- The number of systems
- How many breaks an edit changes (stability)
- Running time
- The number of manual interventions (forced breaks, width overrides) needed to reach the layout of a reference engraving: the measure that matters most to an engraver

The test material is chosen for where the difficulty lies: chromatic passages in short values, bars that mix long values with runs of very short ones, vocal scores with dense lyrics, tight orchestral pages, music of gradually changing density, and parts with page turns. The references are printed engravings, not the output of other programs.

These measurements also decide the **badness exponent and the constants** of the segment cost model — the coefficient, exponent, cap, per-system penalty `l` and tolerance, starting from TeX's values ([Optimization Characterization](#optimization-characterization)) — and the **stretchability `σ`**, starting from `1/2` ([Calibrating the Adjustment Ratio](#calibrating-the-adjustment-ratio)). The constants and `σ` are judged together: TeX's constants are meaningful only under the calibration, and `σ` sets the stretch at which a system counts as badly as one at its collision limit.

### Exact Re-optimisation After an Edit

Two tables are kept with the layout: a forward table `F(s)`, the least cost of segmenting stacks `[0, s)` (the DP's `best`), and a backward table `G(t)`, the least cost of segmenting `[t, N)`, computed the same way from the end, with a `next` pointer per entry.

An edit to stack `m` changes the cost of exactly the segments that contain `m`. `F(s)` for `s ≤ m` and `G(t)` for `t > m` involve no such segment and are unchanged. Every segmentation has exactly one segment containing `m`, and by optimal substructure the parts either side of it are independently optimal, so the new optimum is

```
min over feasible [s, t) with s ≤ m < t of   F(s) + cost′(s, t) + G(t)
```

— at most about K² evaluations. Breaks are recovered through `prev` on the left and `next` on the right. The result is exact.

- **Range edits**: an edit that changes every segment intersecting a range `[a, b]` — several adjacent stacks, a width that depends on where a system ends (see [End-of-System Width](#end-of-system-width)), or a constraint spanning a barline ([Constraints Spanning Several Columns](#constraints-spanning-several-columns)) — is re-optimised by a forward DP over the window `[a − K, b + K]`, seeded from `F` and closed with `G`: O((b − a + 2K) × K).
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

**Not adopted.** Neither form below is part of the baseline, and neither is planned. Badness is cubic and demerits square it, so under an exact optimum a cramped system survives next to a loose one only when every alternative costs more in total. Where that happens, an evenness term can only buy uniformity by distorting other systems, and the known risk recorded below is exactly that. TeX needs fitness classes because a line holds dozens of words; a system holds a handful of bars, and the cost already does most of the work. The forms are recorded so that the decision can be revisited if the validation shows adjacent systems that the cost fails to catch.

Two exact forms, each needing more DP state than the baseline:

- **Fitness classes**: each system is classified by its adjustment ratio `r` into a small number of bands, and adjacent systems whose bands differ by more than one are penalised. One node per (breakpoint, class): about ×4 state and work with four classes.
- **Squared difference of adjacent adjustment ratios**: `μ × (r(s, t) − r(q, s))²`, where `[q, s)` is the preceding system. The state is the (start, end) of the last system: N×K states with K transitions each, O(N×K²) — about 180,000 evaluations at N = 800, K = 15.

Keeping one node per breakpoint while adding an adjacency term makes the result an approximation (see [Comparison](#comparison)). The term carries a known risk, recorded in a comment in LilyPond's `lily/gourlay-breaking.cc`: where music becomes gradually denser, a uniformity requirement drives cramped lines to become more cramped, because the step from a cramped line of three measures to a loose line of two is large. Scores of gradually changing density are part of the evaluation of either form.

### Ending Classes and Looseness

- **Ending classes**: the class idea applied to the kind of boundary a system ends on — a phrase end, a rehearsal mark, the end of a movement. A penalty that depends only on the boundary needs no classes; `break-penalty-fn` already expresses it. Classes are needed only when the cost depends on the preceding system's ending, and multiply the state by the number of kinds.
- **Looseness**: a user control asking for more or fewer systems than the optimum. The system count joins the state, one node per (breakpoint, count): O(S×N×K), which is O(N²) with S ≈ N / K — about 640,000 evaluations at N = 800. Exact.

### Fitting Rules

Two steps are kept apart. **Ideal spacing** is derived within each measure from the durations of its notes and tuplets, in Stage 1. **Fitting** puts measures onto systems and makes their widths fill each system; Stage 3 decides it and Stage 4 applies it. The breaking decision depends on the fitting rule, because the DP judges each candidate system by its feasibility and cost under that rule.

Each column gap is a spring with three kinds of parameter ([Gaps as Springs](#gaps-as-springs)):

- its **ideal width**, which is where duration enters;
- its **blocking width**, which is where collision enters;
- its **elasticity**, its stretch and shrink, which decide how the ideal widths give way when a system is looser or tighter than they are.

A fitting rule is a choice of elasticity.

**The baseline: springs.** Stretch `σ × w_j`, shrink `max(0, w_j − b_j)`. Under stretching, gaps free to stretch keep their proportions and prestretched gaps join at their kinks. Under compression every gap gives up the same share of its room above its minimum: where long values have more room, they give up more, and the rhythmic hierarchy flattens; a gap with no room stays where it is and the others absorb the compression. Every gap reaches its minimum at `r = −1`. This is also LilyPond's default compression.

**Alternatives**, kept as candidates for the validation:

1. **Proportional, normalised by the tightest gap** (the baseline before this revision). Stretch `σ × w_j`, shrink `(1 − max ρ) × w_j` with `ρ = max_j(b_j / w_j)` over the segment, carried as a running extremum, and feasibility `r ≥ −1`. One scale factor per system, so ratios between ideal widths are kept under compression as well as under stretching. It requires `b_j ≤ w_j` for every gap, and one gap at its minimum blocks compression of its whole system.
2. **Even**. Every gap has the same stretch and shrink. Under stretching, short values gain proportionally more and the hierarchy flattens; under compression, they lose proportionally more and it sharpens. Shrink is normalised by the gap with the least room, as in rule 1.
3. **Duration-weighted shrink**. Shrink `κ × v(d_j) × w_j`, weighted by a function of the gap's duration so that, for example, shorter values compress first. Under springs no cap is needed: a gap that reaches its minimum blocks there and the others carry on. Gaps then block at different ratios, so the shrinking side acquires kinks of its own and is solved like the stretching side; `r = −1` needs a definition of its own, for example the ratio at which the last gap blocks. The weighting `v` and the scale `κ` are decided with the rule.
4. **Air-only stretch**. Stretch proportional to the space between bare noteheads, `σ × (w_j − n)` with `n` a notehead's width, rather than to the whole ideal width, so that long values take most of any stretching. This is LilyPond's default (`Spacing_spanner::note_spacing` in `lily/spacing-basic.cc`).

Whether the hierarchy should sharpen or flatten under compression, and which values should take the stretch, are engraving questions, decided by the empirical validation rather than by the model. The baseline is the rule this ADR's formulas and code carry; none of the alternatives is chosen until the validation decides.

**All of them keep the DP exact.** Each system's cost still depends on the segment alone, and `r` is the solution of a piecewise-linear equation built from prefix sums and the segment's kinks. Under the baseline and rules 1, 2 and 4 the shrinking side is one division; rule 3 makes it piecewise, solved exactly by the same walk as the stretching side. Under any rule, Stage 4 applies the system's adjustment ratio through each gap's own spring.

**Who pays.** The adjustment ratio is one value for the whole system, so when a bar needs more room than its proportions give it (a run of thirty-second notes between a half note and a quarter, say), the cost is shared by every gap in the system according to its elasticity, including gaps several bars away. An engraver might prefer the long notes of the same bar to pay. That is a choice between consistency, where the same duration is treated alike across the system, and locality, where each bar settles its own debts. Locality belongs to Stage 1, in how ideal widths are set within a bar before any fitting begins, and needs no change to Stage 3. The baseline is consistency; the validation decides whether locality is needed.

**The raster.** Gaps lie between columns of the stack's vertical raster, one column for each rhythmic position in any staff, and a gap's blocking width is the largest over the staves, so an accidental in one staff widens that column in every staff. Under compression the columns differ according to need; as the system loosens, columns held open by accidentals stop growing while the others catch up, until the grid is even again. No separate policy is specified; the validation decides whether one is needed.

The evaluation covers systems with one dense or incompressible measure among sparse ones, chromatic passages in short values, bars that mix long values with runs of very short ones, the behaviour of the rhythmic hierarchy under stretching and under compression, consistency in complex rhythmic contexts, and the proportionality each rule gives up.

### Feasibility

Under the baseline a system is feasible exactly when its gaps' blocking widths fit:

```
Σ min_i ≤ available          (equivalently, r ≥ −1)
```

The condition is necessary and sufficient. Every gap's shrink is its own room above its minimum, so at `r = −1` every gap is at its minimum and none reaches it sooner. It comes from prefix sums in O(1), and it is the same point as Knuth–Plass's limit on shrinking ([Calibrating the Adjustment Ratio](#calibrating-the-adjustment-ratio)).

Under the former baseline (rule 1 of [Fitting Rules](#fitting-rules)), one scale factor applied to every gap, and the sufficient condition was the largest column ratio, `scale ≥ max_j(b_j / w_j)` over the segment. A stack could pass a stack-level check while one of its columns collided, so every stack had to carry that ratio. Under the springs no such ratio is needed.

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
  "Segment cost parameters, as documented starting points; the empirical
   validation decides them. All but :stretchability are TeX's, and are
   meaningful only under the calibrated adjustment ratio."
  {:badness-coefficient 100N  ;; c: badness ≈ c × |r|^e
   :badness-exponent 3        ;; e: an integer, so badness stays an exact rational
   :badness-cap 10000N        ;; b_max: "infinitely bad"
   :line-penalty 10N          ;; l: added to every system; favours fewer systems
   :tolerance nil             ;; bound on badness beyond which a segment is infeasible;
                              ;; nil = no bound (plain TeX's \tolerance is 200)
   :stretchability 1/2})      ;; σ: stretch per unit of ideal width; an ordinary
                              ;; gap is at (1 + σ) × w at r = 1

(defn gap-width
  "Width of a gap with ideal width w and blocking width b at adjustment
   ratio r, under the baseline springs: stretch σ × w, shrink max(0, w − b)."
  [w b r sigma]
  (if (neg? r)
    (max b (+ w (* r (max 0N (- w b)))))
    (max b (+ w (* r sigma w)))))

(defn stack-summary
  "Summarises a sequence of gaps {:w ideal, :b blocking} for Stage 3
   (see Interface Contract)."
  [gaps]
  (reduce (fn [acc {:keys [w b]}]
            (cond-> (-> acc
                        (update :min + b)
                        (update :natural + (max w b))
                        (update :ideal + w))
              (<= b w) (update :ordinary-ideal + w)
              (> b w)  (update :prestretched conj [w b])))
          {:min 0N :natural 0N :ideal 0N :ordinary-ideal 0N :prestretched []}
          gaps))

(defn add-kinks
  "Adds prestretched gaps to a sorted map {kink Σw}, where a gap's kink is
   (b − w) / (σ × w), the ratio at which it starts to stretch. A gap of zero
   ideal width is fixed space: it never stretches and has no kink."
  [kinks prestretched sigma]
  (reduce (fn [m [w b]]
            (if (zero? w)
              m
              (update m (/ (- b w) (* sigma w)) (fnil + 0N) w)))
          kinks
          prestretched))

(defn solve-ratio
  "The least adjustment ratio r at which a segment's gaps fill `available`,
   or nil when they cannot (available < Σ min). min, natural and
   ordinary-ideal are the segment's sums; kinks is its sorted map {kink Σw}.
   Exact: rationals in, rational out."
  [available min natural ordinary-ideal kinks sigma]
  (cond
    (< available min) nil

    (= available natural) 0N

    ;; Shrinking: every gap gives up the same share of its room above its
    ;; minimum, and all reach their minimums together at r = -1. One division.
    (< available natural)
    (/ (- available natural) (- natural min))

    ;; Stretching: ordinary gaps stretch from r = 0; each prestretched gap
    ;; joins at its kink. The first piece is the fast path. A positive :ideal
    ;; for every stack guarantees a positive slope once all kinks are passed.
    :else
    (loop [width natural
           r0 0N
           slope (* sigma ordinary-ideal)
           ks (seq kinks)]
      (let [[k w] (first ks)
            reach (when k (+ width (* slope (- k r0))))]
        (if (or (nil? k) (<= available reach))
          (+ r0 (/ (- available width) slope))
          (recur reach k (+ slope (* sigma w)) (next ks)))))))

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
   - stacks: Vector of stack summaries {:min, :natural, :ideal, :ordinary-ideal,
             :prestretched, :gutter, :measure-index} (see Interface Contract)
   - system-width-fn: Function (fn [start-pos end-pos] -> ratio)
                      Returns available width for system containing stacks start-pos..end-pos-1
   - break-penalty-fn: Optional. Function (fn [stacks s t] -> ratio)
                       Returns the penalty p for selecting segment s..t-1 as a system;
                       it enters the demerits as +p² or -p²
                       Default: (constantly 0N) - no editorial preferences
   - params: Optional. Segment cost parameters; default knuth-plass-parameters

   Output:
   - {:breaks [break-positions], :ratios [adjustment ratio per system],
      :cost total-demerits-ratio}"

  ([stacks system-width-fn]
   (find-optimal-breaks stacks system-width-fn (constantly 0N)))

  ([stacks system-width-fn break-penalty-fn]
   (find-optimal-breaks stacks system-width-fn break-penalty-fn knuth-plass-parameters))

  ([stacks system-width-fn break-penalty-fn params]
   (let [n (count stacks)
         tolerance (:tolerance params)
         sigma (:stretchability params)

         ;; Precompute prefix sums for O(1) range queries
         ;; CRITICAL: Use 0N to maintain ratio domain
         min-prefix (vec (reductions + 0N (map :min stacks)))
         natural-prefix (vec (reductions + 0N (map :natural stacks)))
         ordinary-prefix (vec (reductions + 0N (map :ordinary-ideal stacks)))

         ;; Tie-break terms: element t is 2^-t, the secondary cost of a
         ;; segment ending at breakpoint t
         tie-term (vec (reductions (fn [x _] (/ x 2)) 1N (range n)))

         ;; State arrays: nil = unreachable, [cost tie] = best pair for the prefix
         ;; Length n+1 where index t represents prefix of length t
         best (transient (vec (repeat (inc n) nil)))
         prev (transient (vec (repeat (inc n) nil)))
         ratio (transient (vec (repeat (inc n) nil)))]

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
         ;; so its prestretched gaps are added to the segment's kinks in
         ;; O(P_s log P) rather than the segment being rebuilt
         (loop [s (dec t)
                kinks (sorted-map)]
           (when (>= s 0)
             (let [kinks (add-kinks kinks (get-in stacks [s :prestretched]) sigma)
                   system-width (system-width-fn s t)
                   ;; Reserve preamble and gutter space for system-start stack
                   preamble (get-preamble-width stacks s)  ;; On-demand, cached by clef/keysig combo
                   gutter (get-in stacks [s :gutter] 0N)
                   available (- system-width preamble gutter postamble)
                   total-min (- (nth min-prefix t) (nth min-prefix s))]

               ;; Early termination: stop once the minimums exceed the full system width.
               ;; This bound only grows harder to meet as s moves left; `available` does
               ;; not, because preamble[s] and gutter[s] vary with s
               (when (<= total-min system-width)

                 ;; The adjustment ratio, or nil when the minimums do not fit after
                 ;; reserving preamble, gutter and postamble: r >= -1 is feasibility
                 (when-let [r (and (some? (nth best s))
                                   (solve-ratio available
                                                total-min
                                                (- (nth natural-prefix t) (nth natural-prefix s))
                                                (- (nth ordinary-prefix t) (nth ordinary-prefix s))
                                                kinks
                                                sigma))]
                   (let [b (badness r params)]

                     ;; Tolerance: badness within the optional bound
                     (when (or (nil? tolerance) (<= b tolerance))
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
                           (assoc! prev t s)
                           (assoc! ratio t r))))))

                 (recur (dec s) kinks)))))))

     ;; Return results: breaks, and the ratio of each system, indexed by where it ends
     (let [prev (persistent! prev)
           ratio (persistent! ratio)
           breaks (reconstruct-breaks prev n)]
       {:breaks breaks
        :ratios (mapv #(nth ratio %) (rest (conj breaks n)))
        :cost (first (nth best n))}))))
```

**System reservations explained:**

For each candidate segment [s, t):
1. Get `preamble = preamble[s]` (max clef+keysig width across all staves, computed on-demand and cached)
2. Get `gutter = gutter[s]` (only first stack in system contributes)
3. Get `postamble = postamble[t]` (system-end cautionaries and barline; fixed for a given `t`)
4. Compute `available = system_width - preamble - gutter - postamble`
5. Check feasibility: `Σ min_i ≤ available`
6. Solve for the adjustment ratio `r`: one division when shrinking, and when stretching until the segment's first kink; beyond it, the kinks in order
7. Compute its badness and, within the tolerance, the system's demerits; preamble, gutter and postamble are fixed overhead and enter only through `available`

The preamble, gutter and postamble are reserved space outside the springs; they do not participate in cost computation.

**Feasibility condition:**

Under the baseline, `Σ min_i ≤ available` is **necessary and sufficient**: every gap's shrink is its own room above its minimum, so every gap reaches its minimum at `r = −1` and none sooner ([Feasibility](#feasibility)). `solve-ratio` returns nil exactly when it fails.

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

The DP algorithm subtracts preamble, gutter and postamble from the policy-provided width to determine the space the springs fill.

### Solving for the Adjustment Ratio

`solve-ratio` finds the least `r` at which a segment's gaps fill the available width ([Gaps as Springs](#gaps-as-springs)):

- **Infeasible**: `available < Σ min`, so no `r ≥ −1` fits.
- **Shrinking**: `r = (available − Σ natural) / (Σ natural − Σ min)`, one division. Under the baseline every gap's shrink is its room above its minimum, so `Σ g_j(r)` is linear on `[−1, 0]`.
- **Stretching**: from `r = 0` the segment grows at `σ × Σ ordinary ideal` per unit of `r`. At each kink, in increasing order, the slope grows by `σ × w` for the prestretched gaps that start to stretch there. The walk stops on the piece that contains `available`, and `r` is one division on it. When `available` is reached before the first kink, which is the common case, this is the fast path and costs what the former baseline did.

The walk visits only the kinks below `r`. A balanced tree over the kinks carrying running sums of `w` would find the piece in O(log P) for `P` prestretched gaps in the segment; the linear walk is specified for clarity, since a segment holds few prestretched gaps and `r` passes few of them within any reasonable tolerance.

Every quantity is a rational, so `r` is exact and identical on every platform. The same function serves Stage 2, which uses it to reduce [constraints spanning several columns](#constraints-spanning-several-columns).

### Segment Cost Computation

Once `r` is known, the system's cost follows ([Optimization Characterization](#optimization-characterization)):

```
r = least r with Σ g_j(r) = available
b = min(b_max, c × |r|^e)
d = (l + b)² ± p²
```

```clojure
;; Segment cost (inline in DP loop, for a feasible segment)
(let [preamble (get-preamble-width stacks s)
      gutter (get-in stacks [s :gutter] 0N)
      postamble (get-postamble-width stacks t)
      available (- system-width preamble gutter postamble)
      r (solve-ratio available total-min total-natural total-ordinary kinks sigma)
      b (badness r params)
      d (demerits b (break-penalty-fn stacks s t) params)]
  ...)
```

This is the canonical cost definition under the ADR's contract, not an optimisation of a more complex one. The `r` it uses is the ratio every gap of the system actually receives in Stage 4.

### Width Allocation Implementation

```clojure
(defn allocate-widths
  "Allocates actual gap widths for one system from its adjustment ratio r.

   Used after break selection to compute final positions for rendering.
   system-gaps is a vector, per stack, of that stack's gaps {:w ideal, :b blocking}.
   preamble and postamble are the system's preamble[s] and postamble[t]; the
   first stack contributes :gutter. All three are reserved space outside the
   springs.

   Returns map with:
   - :preamble, :gutter, :postamble - reserved widths
   - :gaps - per stack, the vector of its gap widths
   - :actuals - per stack, its total width"
  [system-stacks system-gaps preamble postamble r sigma]
  (let [gutter (get (first system-stacks) :gutter 0N)
        gaps (mapv (fn [stack-gaps]
                     (mapv (fn [{:keys [w b]}] (gap-width w b r sigma)) stack-gaps))
                   system-gaps)]
    {:preamble preamble
     :gutter gutter
     :postamble postamble
     :gaps gaps
     :actuals (mapv #(reduce + 0N %) gaps)}))
```

**Why this approach**:

1. **Derivation, not optimisation**: `r` is already known; each gap's width is one formula
2. **Collision-safe by construction**: no gap falls below its blocking width, so there is no clamp to trust
3. **Reservation correctness**: System-start decorations and system-end cautionaries and barlines have exactly their required space
4. **Deterministic**: Same input always produces same output with no floating-point variation
5. **Fast**: O(gaps) per system
6. **Contained**: Within a fixed break configuration, a change to one stack alters only its own system's ratio and widths

**Verification**:
```clojure
;; r is the ratio Stage 3 solved for this system's available width
(let [{:keys [preamble gutter postamble actuals]}
      (allocate-widths stacks gaps preamble postamble r sigma)]
  (assert (= system-width (+ preamble gutter (reduce + 0N actuals) postamble))))
```

The equality is exact, not approximate: `r` solves the system's equation exactly, and the summaries Stage 3 used are consistent with the gaps Stage 4 applies it to ([Interface Contract](#interface-contract)).

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
- Are reserved from system width before the springs are fitted: `available = system_width - preamble - gutter - postamble`
- Do not scale—they are fixed space for clefs and key signatures, graphical decorations, and system-end cautionaries and barlines
- Do not participate in cost computation
- Are returned separately from `allocate-widths` for rendering coordination

**Baseline: Springs**:

The implementation fits each system by its gaps' springs. This is derivation (the exact solution of a piecewise-linear equation), not optimisation (an iterative solver):

- **Ratio**: `r` is the least value with `Σ g_j(r) = available`: one division when shrinking, and when stretching until a prestretched gap starts to stretch
- **Reservation**: `available = system_width - preamble[s] - gutter[s] - postamble[t]`
- **No clamp**: `g_j(r) = max(b_j, w_j + r × k_j)` never falls below the blocking width
- **Fast**: O(gaps) per system to apply, essentially zero cost
- **Contained**: Within a fixed break configuration, a change to one stack alters only its own system's ratio and widths

Whether an alternative fitting rule is needed is decided by the validation, not speculatively ([Fitting Rules](#fitting-rules)).

**Extension Point: break-penalty-fn**:

The algorithm accepts an optional `break-penalty-fn` parameter with signature `(fn [stacks s t] -> ratio)`. It returns the penalty `p` for selecting segment `s..t-1` as a system, which enters the demerits as `+p²` or `−p²`. The default `(constantly 0N)` produces purely geometric optimization. The segment cost parameters (`knuth-plass-parameters`) are a further optional argument. This ADR does not specify any concrete penalty functions; editorial policy is a separate concern that can evolve independently of the core algorithm.

**Feasibility Check**:

`Σ min_i ≤ available` (the minimums must fit after reserving preamble, gutter and postamble), O(1) via prefix sums. Under the baseline it is necessary and sufficient ([Feasibility](#feasibility)); `solve-ratio` returns nil exactly when it fails.

**Kinks**:

The inner loop visits segments in a fixed order: for fixed `t`, `s` decreases from `t-1`, so each candidate segment `[s, t)` is the previous one extended by the single stack `s`. Its prestretched gaps are added to a persistent sorted map of kinks, O(P_s log P) for `P_s` gaps in stack `s` and `P` in the segment; the map's first key decides whether a stretching segment takes the fast path. Rebuilding the map for each segment instead would cost O(P log P) per segment.

**Segment Evaluation**:

Each candidate segment (s, t) requires:
1. **Reservation lookups**: O(1) to get `preamble[s]` and `gutter[s]`; `postamble[t]` is fixed for the inner loop
2. **Feasibility**: O(1) from prefix sums
3. **Kinks**: stack `s`'s prestretched gaps added to the segment's sorted map, O(P_s log P)
4. **Cost**: the adjustment ratio, O(1) when shrinking and on the fast path when stretching, otherwise one step per kink passed; then badness and demerits from it

Every per-segment step is O(1) except where prestretched gaps take part.

**Rational Arithmetic Throughout**:
- All numeric literals use `N` suffix: `0N`
- Preserves exact arithmetic, no floating-point contamination
- Ensures deterministic outcomes across platforms
- Ratio growth managed through Clojure's automatic normalization (gcd)
- Kinks are rationals, so the sorted map orders them exactly
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
- Precomputed once: O(N) time for `min-prefix`, `natural-prefix` and `ordinary-prefix`
- Range sum in O(1): `(- (nth prefix t) (nth prefix s))`
- Enables O(1) feasibility, and the adjustment ratio in O(1) when shrinking and on the fast path when stretching
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
| Precomputation | O(N) | Prefix sums for min, natural and ordinary ideal; tie-break terms |
| Outer loop | O(N) | Each position once |
| Inner loop | O(K) typical | Early termination once minimums exceed the full system width |
| Reservation lookups | O(1) | Preamble and gutter per segment; postamble per outer iteration |
| Feasibility check | O(1) | Prefix sums of minimum widths |
| Kinks | O(P_s log P) | Stack s's prestretched gaps added to the segment's sorted map |
| Cost computation | O(1) fast path | Adjustment ratio from prefix sums; one step per kink passed when a prestretched gap starts to stretch; badness and demerits from it |
| **Total (worst case)** | O(N²) | Without early termination |
| **Total (typical)** | O(N×K) | With monotone width policy, K ≈ 15 |
| Page breaking | O(S²) | S = number of systems |

At N=800 measures, K=15: ~12,000 segment evaluations, each O(1) for feasibility and, on the fast path, for cost.

## Relationship to Knuth-Plass

This section defines the relationship once. Elsewhere in this ADR, "Knuth-Plass" is shorthand for it.

**Classification**: Stage 3's baseline is an optimal-segmentation dynamic program in the Knuth–Plass family (Knuth and Plass, 1981; see [Prior Art](#prior-art)). Its cost is Knuth–Plass's: the badness of each system's adjustment ratio, combined with a per-system penalty and the breakpoint penalty into demerits. Its glue is Hegazy and Gourlay's: springs with a blocking width (Hegazy and Gourlay, 1987; see [The Spring Model](#the-spring-model)).

**Mapping onto the Knuth–Plass primitives**:

| Knuth–Plass | Stage 3 |
|-------------|---------|
| Box | Atom |
| Natural width of glue | Ideal distance between atom origins, `w_j` |
| (none) | Blocking width `b_j`, the distance at which the atoms on either side collide |
| Stretchability of glue | `σ × w_j` in the baseline, with `σ` a parameter |
| Shrinkability of glue | `max(0, w_j − b_j)` in the baseline, the gap's room above its minimum |
| Legal breakpoint | Legal break position, normally a barline |
| Discretionary break: pre-break, post-break and no-break material | `postamble[t]`; `preamble[s]` and `gutter[s]`; the barline's mid-system width |
| Line length | System width less preamble, gutter and postamble |
| Adjustment ratio of a line | Adjustment ratio `r` of a system: the least `r` with `Σ g_j(r) = available`, Hegazy and Gourlay's fitting force |
| Badness, about `100 × \|r\|³`, capped | Badness `b(r) = min(b_max, c × \|r\|^e)`, starting from `c = 100`, `e = 3` |
| Demerits `(l + b)² ± p²` | Demerits `(l + b)² ± p²`, with `p` from `break-penalty-fn` |
| Line penalty `l` | Per-system penalty `l` |
| Tolerance | Optional bound on badness |

**The springs as an extension of glue**: a spring is glue with a blocking width. Where no gap is blocked or prestretched, `Σ g_j(r)` is linear in `r` and the adjustment ratio is Knuth–Plass's single division. A prestretched gap, `b_j > w_j`, adds a kink where it starts to stretch. Under the baseline's shrink every gap blocks exactly at `r = −1`, so the shrinking side has no kinks. `r = −1` is the system's collision limit, so Knuth–Plass's condition `r ≥ −1` is the feasibility condition `Σ min ≤ available` ([Calibrating the Adjustment Ratio](#calibrating-the-adjustment-ratio)).

**In the baseline**: the adjustment ratio, cubic badness, demerits with the per-system penalty, breakpoint penalties through `break-penalty-fn`, and the tolerance are Knuth–Plass's own, with parameters to be settled by validation.

**Extensions available**: The remaining Knuth–Plass features are available extensions of the baseline, not departures from it:

- Fitness classes, including classes for different kinds of system endings (the evenness forms are not adopted; see [Consistency Between Adjacent Systems](#consistency-between-adjacent-systems))
- Looseness

**The reduction is preserved**: each stack is summarised by three sums and its prestretched gaps. Prefix sums of the sums give feasibility in O(1), and the adjustment ratio in O(1) when shrinking and, when stretching, up to the segment's first kink; beyond it the segment's sorted kinks are walked in order. Each system's cost still depends on the segment alone.

**Exactness and DP state**: The baseline's cost has no term linking adjacent systems, so one DP node per breakpoint is enough for the result to be exact. The springs keep that. Two extensions need more state:

- **Fitness classes**: a cost that depends on the class of the preceding system is exact only if the DP keeps one node per (breakpoint, fitness class), which multiplies the work by the number of classes. Keeping a single node per breakpoint with such a term makes the result an approximation.
- **Looseness**: aiming for a number of systems other than the optimal one requires the number of systems so far to be part of the state, one node per (breakpoint, system count).

**Adoption**: Whether looseness or ending classes are adopted is decided during implementation, by the empirical validation this ADR specifies under [Future Considerations](#consequences) (simple scores, then complex ones, then Elektra). The extensions are specified, with their costs and evaluation criteria, under [Stability and Quality Extensions](#stability-and-quality-extensions); none is enabled by this ADR, and the evenness forms are not adopted.

Both problems share the mathematical structure enabling polynomial-time exact optimization of the reduced subproblem:
- 1-dimensional sequence with a summary per element
- Capacity-constrained segmentation
- Separable cost function with optimal substructure
- A continuous, non-decreasing, piecewise-linear width function within each segment

**Explicit non-claim**:
This approach does **not** solve "general music engraving" or "optimal notation layout for all musical structures." It solves the **measure distribution subproblem** after architectural reduction has transformed it into tractable form.

The reduction is what matters: by solving vertical coordination, symbol collision, and semantic determinism in earlier stages, the distribution problem becomes structurally similar to paragraph breaking. This is not algorithmic cleverness finding a better heuristic; it is architectural separation creating a problem formulation where exact optimization applies.

**What the widths mean**:

`ideal_width` encodes rhythmic proportionality, which has musical meaning, and `min_width` encodes collision, which has physical meaning. The springs keep both. Gaps free to stretch keep their proportions under stretching, and a gap the music needs wider than its proportion gets exactly the room its collision requires. Under compression proportionality is given up, gap by gap, in proportion to each gap's room above its minimum; the choice of shrink expresses how ([Fitting Rules](#fitting-rules)). Graphical decorations (gutter content), the preamble and the postamble are fixed overhead, outside the springs.

**Architectural prerequisites**:

The Knuth-Plass algorithm solves a specific problem formulation: capacity-constrained segmentation of a 1-dimensional sequence with separable costs. Music layout, naively formulated, is not this problem—it couples vertical alignment, collision detection, semantic decisions, and geometry into mutual dependencies.

Ooloi's pipeline architecture transforms the problem. By the time Stage 3 executes:
- Vertical coordination is complete (Stages 1-2)
- Collision boundaries are determined (Stage 1)
- Semantic decisions (accidentals, beaming) are resolved (ADR-0035)
- Gutter requirements are computed (Stage 1)
- Constraints spanning several columns are reduced to blocking widths (Stage 2)
- Preamble and postamble widths are known for every candidate system start and end
- Connecting elements are deferred (Stage 5)

What remains is the Knuth-Plass problem formulation, over measure-stack summaries.

**Why the DP runs once per pass**:

Knuth-Plass itself is efficient—O(N²) in the general case, O(N×K) with early termination. In Ooloi it runs once per layout pass, on inputs that do not change while it runs. Three preconditions of the pipeline provide this:

- **Stable scalar inputs**: Stage 3 receives stable scalar inputs from completed upstream stages. Every width it consumes is final before it begins.
- **No feedback between stages**: No later stage changes Stage 3's inputs, so the DP never has to be re-run while geometry settles.
- **Immutability**: Immutability enables precise cache invalidation: edits affect only the stacks whose upstream metrics actually changed, not the entire sequence.

At N=800 measures with K=15 measures per system, a pass is approximately 12,000 segment evaluations, each O(1) scalar arithmetic.

The pipeline stages, immutable data structures and rational arithmetic provide both the problem formulation and the conditions under which the DP runs once per pass.

## Prior Art

Stage 3 draws on three lineages: optimal breaking by dynamic programming, a model of horizontal space that separates fixed width from scalable width, and a model of the gaps between notes as springs that can block.

### Optimal Breaking by Dynamic Programming

- **Knuth and Plass**, "Breaking Paragraphs into Lines", *Software—Practice and Experience* 11 (1981), pp. 1119–1184. Optimal line breaking of paragraphs by dynamic programming over legal breakpoints; the family Stage 3 belongs to.

- **Hegazy and Gourlay**, "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, Department of Computer and Information Science, The Ohio State University (1987); also in *Proceedings of the International Conference on Electronic Publishing, Document Manipulation and Typography*, Nice, April 1988, ed. J. C. van Vliet, Cambridge University Press. Described here from a scan of the report version (15 pages; NSF grant IST-8514308), a product of the MusiCopy project at Ohio State; that copy does not carry the report number.
  - **The problem.** Four ways music differs from text: the last system of a piece is always full, so early breaks must be revisable; long works should fill their last page, which can mean forcing a number of systems; clefs and key signatures repeat at every system start and change in content and width; and notes do not move linearly when a line is stretched or shrunk. The report notes that Byrd's 1984 program did first-fit (greedy) breaking and left breaks to the user, and that "there appears to be no published algorithm" that deals with these problems.
  - **The claim.** Knuth's model already handles the first three. A paragraph without fill glue has a full last line; penalties encourage or discourage breaks; TeX can force a number of lines, and Plass's method breaks pages; TeX's discretionary breaks can change the material at the end of one line and the start of the next. Only the fourth needs new theory.
  - **The model.** Springs pass through the boxes and attach to the left edges of noteheads; boxes cannot pass through each other. Each spring has an ideal width, a stretchability, a shrinkability and a blocking width at which the boxes it connects collide; a spring whose blocking width exceeds its ideal width is "prestretched". The force needed to fit a line is a piecewise-linear function of the required stretch, the line's "characteristic function", whose segments end where a spring blocks or a prestretched spring starts to stretch. This is the model of [Gaps as Springs](#gaps-as-springs).
  - **The search.** TeX's active-list dynamic program, which the report says "does not differ in any significant way" from TeX's: one best predecessor per feasible breakpoint, lines whose force falls outside a window `[f_min, f_max]` discarded, breaks only at barlines, at most (measures × maximum measures per line) evaluations. TeX's fitness classes are not mentioned. The cost is derived from the fitting force in TeX's manner; no formula is given.
  - **Efficiency.** The report places the cost in computing each line's force, never in the DP. It uses a cheap lower bound on the force, from running totals subtracted between breakpoints (prefix sums), designed never to reject a feasible line, and a fast path for forces within the first linear segment, tested with minima precomputed per measure. It observes that for `|f| < 1` most feasible lines fall in the first segment.
  - **Validation.** One 48-bar single-staff piece, Dean K. Roush's 'Lacrimosa': the breaks matched those chosen by a professional musician, who had taken a few hours over it; the program took a few seconds on a personal computer.
  - **Exactness.** The cost is separable per line, the pruning is monotone and the bound conservative, so the result is the exact optimum of its cost within the force window. The pruning is valid only because the model has no width that depends on where a line starts or ends: with a start-dependent preamble or an end-dependent postamble, a line rejected as too long can be followed by a feasible one. That is why this ADR's early termination is against the full system width ([Key Implementation Notes](#key-implementation-notes)).
  - **Relation to Stage 3.** Stage 3 was specified from Knuth and Plass before the report had been read, and its search coincides with the report's: one node per breakpoint without fitness classes, prefix sums between breakpoints, values precomputed per stack for the fast test, monotone termination. The baseline before this revision, proportional fitting normalised by the tightest gap, was the report's model restricted to its first linear segment and rescaled so that the first collision fell at `r = −1`. This revision adopts the full model.

- **LilyPond**:
  - `lily/gourlay-breaking.cc`, by Han-Wen Nienhuys, is present from release 0.1.1 (August 1997) until September 2006. From release 1.1.32 (February 1999) its comment reads: "This algorithms is adapted from the OSU Tech report on breaking lines." What follows describes LilyPond's code, not the report, which may differ from it.
    - In both the 0.1.1 and the 2006 revisions it is a dynamic program over legal breakpoints that keeps one best predecessor per breakpoint. For each breakpoint it tries lines ending there, extending them backwards one breakpoint at a time until a line becomes infeasible, and traces the chosen breaks back from the end.
    - In release 0.1.1 the cost of a configuration is the sum of each line's spacing energy, which is separable. The search is bounded: lines longer than a set number of measures, and lines whose energy exceeds a set bound, are not considered, and every candidate line is first evaluated with an approximate spacing solution; a candidate whose approximate energy cannot improve on the best found is skipped, and the exact spacing is computed only for a remaining candidate whose approximate solution fails its constraints. Where these bounds or the approximate screening exclude a line, the result can differ from the optimum of the cost.
    - In its 2006 form it solves the spacing problem for every candidate line, with no measure or energy bound. A line's demerits are |force| + |previous force − force| + break penalty, with a large penalty for a line that cannot satisfy its spacing constraints. The middle term depends on the previous line, and is evaluated against the single predecessor stored for the line's starting breakpoint; the result is therefore not guaranteed to minimise total demerits.
  - `lily/constrained-breaking.cc`, by Joe Neeman, is added in February 2006 and replaces it; `gourlay-breaking.cc` is removed in September 2006. It runs a dynamic program over (number of systems, breakpoint), with demerits force² + (previous force − force)² + break penalty, and solves a spring-and-rod spacing problem for every candidate line (`get_line_forces` in `lily/simple-spacer.cc`). It keeps one node per (number of systems, breakpoint), so the adjacency term is again evaluated against a single stored predecessor; with ragged-right lines the demerits reduce to force² plus break penalty, which is separable, and the result is then the exact optimum of that cost.
  - It works together with an optimal page breaker (`lily/optimal-page-breaking.cc`) and a page-turn breaker (`lily/page-turn-page-breaking.cc`). Page decisions use "pure" heights (`lily/constrained-breaking.cc`), which LilyPond's Contributor's Guide describes as estimates made before line breaking.

- **Dorico**, from its developers' published statements, not from its source. Daniel Spreadbury wrote on the Steinberg forum (December 2016) that the team had evaluated the Gourlay method and found its performance characteristics unsuitable for real-time use, that Dorico's algorithm runs one way, from the start of the layout to the end, and that tested against Gourlay's it produced very similar results much of the time. In April 2026 he wrote that the paper informed Dorico's casting off, which is greedy, with a separate remainder-distribution step at the end of each flow. The design goal stated in 2015 was to do only the minimal amount of recalculation in response to an edit.

- **MuseScore**, from its source (`SystemLayout::collectSystem` in `src/engraving/rendering/score/systemlayout.cpp`, main branch, October 2026) and its handbook: measures are added to a system until the next one no longer fits at its natural width, and the system is then justified. Greedy.

- **PriMus** and **Neanes**, from their authors' statements, not verified at source: Christof Schardt wrote in 2015 (in a comment on Dorico's development diary) that PriMus uses TeX's total-fit algorithm modified for scores, interactively, with collisions handled by an equivalent linear operation; Basil Crow wrote in April 2026 (Steinberg forum) that his open-source Neanes editor does optimal line breaking for Byzantine neume notation.

### The Space Model

- **MuTeX**: Andrea Steinbach and Angelika Schofer, *Automatisierter Notensatz mit TeX*, master's thesis, Rheinische Friedrich-Wilhelms Universität, Bonn, 1987. Limited to a single staff; it used TeX glue to control horizontal spacing and justification, and so TeX's own line breaking.

- **MusicTeX**: Daniel Taupin, around 1991. It adds multiple staves, as a single-pass system.

- **MusiXTeX**: Daniel Taupin, Ross Mitchell and Andreas Egler, "MusiXTeX : L'écriture de la musique polyphonique ou instrumentale avec TeX", *Cahiers GUTenberg* 21 (1995), pp. 107–113.
  - The MusiXTeX manual, §1.3.1, explains why TeX glue fails for music: a line holds far fewer bars than a line of text holds words, so treating each bar as a word leaves gaps before the bar rules.
  - It divides horizontal space into *hard* space (bar rules, clefs, key signatures), which is fixed, and *scalable* space, defined in multiples of one spacing unit, `\elemskip`. One value of `\elemskip` is computed per line, so that the scalable space fills what the hard space leaves.
  - This corresponds to ADR-0037's split: the preamble, gutter and postamble are hard space, and the gaps are scalable space; `\elemskip` per line corresponds to the adjustment ratio per system, which the springs apply through each gap's elasticity rather than as one multiplier. ADR-0037 treats only system-start and system-end material as hard; bar rules within a system scale with the measures.
  - Its breaking pass, `musixflx` (Ross Mitchell, 1992–1997), sets a target number of lines from the total width divided by the line width, then fills lines one at a time, adding bars until a line overflows. It is greedy, not optimal.

- **TeX's discretionary breaks**: a discretionary has three parts, the material printed before a break, after it, and when no break occurs. The preamble, gutter and postamble are the same construct at every barline: the postamble is the pre-break material, the preamble and gutter the post-break material, and the barline's mid-system width the no-break material. Hegazy and Gourlay pointed to discretionaries for the clefs and keys at each system start without working them out.

### The Spring Model

- **Hegazy and Gourlay** (1987), above: springs through boxes, blocking widths, prestretched springs, and the force of a line as a piecewise-linear function.

- **LilyPond**: `lily/spring.cc` (Joe Neeman, from 2007) defines a spring by its ideal distance, its minimum distance and its inverse stretch and compression strengths. Its length under a force is the larger of the minimum and the ideal plus force times flexibility, with a blocking force below which the length is constant, including the case in which the minimum exceeds the ideal: Hegazy and Gourlay's spring. By default a spring compresses by its room above its minimum (`Spring::set_default_compress_strength`), as in this ADR's baseline, and a note's spring stretches in proportion to the space between bare noteheads (`Spacing_spanner::note_spacing` in `lily/spacing-basic.cc`). Minimum distances spanning several columns, its "rods", are reduced to blocking forces on the springs they span (`Simple_spacer::add_rod` in `lily/simple-spacer.cc`): the reduction of [Constraints Spanning Several Columns](#constraints-spanning-several-columns). LilyPond solves the spacing afresh for every candidate line.

- **MuseScore**: when a system is justified (`HorizontalSpacing::justifySystem` and `stretchSegmentsToWidth` in `src/engraving/rendering/score/horizontalspacing.cpp`), each segment is a spring whose stiffness comes from its duration, and a segment already wider than its duration warrants, from a collision, does not stretch until the force reaches it: the stretching half of the model. Compression is handled separately, by re-spacing with reduced paddings and duration spacing in steps.

### Comparison

Each breaker is described against its own cost: whether the result it returns is guaranteed to be the minimum of the cost it defines.

| Breaker | Search | Cost | Exact for its own cost |
|---------|--------|------|------------------------|
| Hegazy and Gourlay, 1987 | DP, one node per breakpoint | Derived from each line's fitting force under blocking springs; no formula given | Yes, within its force window; its pruning assumes no width that depends on where a line starts or ends |
| `musixflx` (MusiXTeX) | Target line count, then lines filled one at a time | None; `\elemskip` is set to fill each line | No: greedy |
| LilyPond `gourlay-breaking.cc`, 1997 | DP, one node per breakpoint | Sum of line energies (separable) | Where its measure limit, energy bound and approximate screening do not exclude the optimal line |
| LilyPond `gourlay-breaking.cc`, 2006 | DP, one node per breakpoint | \|force\| + \|previous force − force\| + penalty | No: the adjacency term is evaluated against one stored predecessor |
| LilyPond `constrained-breaking.cc` | DP, one node per (system count, breakpoint) | force² + (previous force − force)² + penalty | Justified lines: no, for the same reason. Ragged-right lines: yes |
| LilyPond page breaking | DP over fixed lines; search over system counts | force² per page plus penalties | Page DP over fixed lines: yes. Search over system counts: no, bounded by heuristics and using estimated heights |
| Dorico (by its developers' account) | One pass from start to end, with remainder distribution at the end of each flow | Not published | No: greedy |
| MuseScore | Measures added while they fit at natural width | None | No: greedy |
| Stage 3 | DP, one node per breakpoint | Knuth–Plass demerits (l + b(r))² ± p², from each system's adjustment ratio under blocking springs | Yes |
| Stage 6 | DP over systems | Not yet specified | Not established |

**Why Stage 3 is exact**: its cost has no term linking adjacent systems, so one node per breakpoint loses nothing; every other input (`system-width-fn`, `preamble[s]`, `gutter[s]`, `postamble[t]`, the stack summaries and their kinks, `break-penalty-fn`) depends only on the candidate segment; each system's adjustment ratio is the exact solution of the segment's own piecewise-linear equation; forced breaks partition the problem into independent subproblems; prevented breaks only reduce the set of breakpoints; and the early-termination bound excludes only infeasible segments (see [Key Implementation Notes](#key-implementation-notes)). In this sense Stage 3 is an exact Knuth–Plass computation over Hegazy and Gourlay's springs. Its cost is the baseline, not the full Knuth–Plass cost model; which extensions keep exactness at which cost in state is stated under [Relationship to Knuth-Plass](#relationship-to-knuth-plass).

**Why Stage 6 is not established**: its cost function is not specified, and page heights that depend on the page (a first page carrying a title, running headers) require the page number, or at least its parity, in the DP state. Both are open.

### What Is Specific to Ooloi

- The combination of the three lineages over measure-stack summaries, structured so that full Knuth–Plass remains available
- The preamble, gutter and postamble as fixed widths, known to the DP before it chooses breaks, that depend on where a system starts or ends. This is TeX's discretionary at every barline, which Hegazy and Gourlay pointed to for music without working it out; what is specific is the formulation, and an early termination that stays exact in its presence
- Blocking springs summarised so that feasibility is O(1), and the adjustment ratio O(1) except where prestretched gaps take part, with constraints spanning several columns reduced before breaking
- Exact rational arithmetic and a tie-break that depends only on the configuration, giving identical results on every platform and after any editing history
- Exact re-optimisation after an edit, from forward and backward tables (the technique has precedent in text layout)
- Use inside a collaborative editor, where every client must see the same layout (PriMus's author reports interactive total-fit breaking; the collaborative determinism is specific)

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
2. **Rhythm and collision together** - Gaps free to stretch keep their proportions; a gap the music needs wider gets exactly the room its collision requires, and no more
3. **Complete information** - Gutter deltas enable optimal decisions without feedback
4. **Closed semantic model** - Rendering decorations cannot affect musical semantics
5. **Deterministic output** - Identical input produces identical layout across platforms
6. **Cheap recomputation** - An edit re-runs Stage 3 over cached scalar metrics, without repeating upstream work for unchanged stacks
7. **Variable system widths** - Natural handling of final systems, editorial overrides, margins and layout settings
8. **Clean stage separation** - No feedback loops with connecting elements or decorations
9. **Performance** - O(N×K) segment evaluations, each O(1) except where prestretched gaps take part
10. **Rational arithmetic** - No floating-point drift across platforms
11. **Predictability** - Users can mentally model system behavior

**Neutral:**

1. **Two-pass approach** - System and page breaking separated, not jointly optimized
2. **Knuth–Plass demerits** - Badness of each system's adjustment ratio, calibrated so that `r = −1` is the system's collision limit, with TeX's constants and the stretchability `σ = 1/2` as starting points; the constants, `σ` and any asymmetric treatment are settled by empirical validation
3. **Staff-local effects** - Compression affects individual staves within stacks, not uniform visual degradation
4. **Policy-free core** - Algorithm is purely geometric by default; editorial preferences can be added via optional `break-penalty-fn` without modifying the core
5. **Gutter storage** - Each stack carries `gutter` (typically 0N); overhead is minimal
6. **Global breaks** - The break configuration is a global optimum, so an edit can change system breaks anywhere in the score; stability comes from the mechanisms under [Stability and Quality Extensions](#stability-and-quality-extensions), not from the DP itself
7. **Springs, not one scale factor** - Stage 4 applies each system's adjustment ratio gap by gap, and each stack's input to Stage 3 includes a short list of its prestretched gaps besides its sums

**Future Considerations:**

The spring baseline should be validated empirically on real scores, against printed engravings:
1. Implement the spring baseline as specified
2. Test on real scores: simple → complex → Elektra, with material chosen for where the difficulty lies ([Evaluation Criteria](#evaluation-criteria))
3. Document visual quality, performance characteristics (including the cost of the slow path on chromatic material), edge cases
4. Only if systematic problems emerge, consider the alternative fitting rules or the general asymmetric formulation

The formal validation will determine whether the baseline approach is sufficient or whether additional complexity is justified. The specific visual issues encountered will guide cost function design rather than speculative theory.

## References

- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) (pipeline architecture, Stage 3 system breaking position, gutter model, closed semantic model)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) (accidental algorithm that closes the semantic model; tied-note bypass rules inform gutter requirements)
- **ADR-00XX: Horizontal Spacing** — forthcoming: upstream computation of each gap's ideal and blocking widths, the gutter, and the reduction of constraints spanning several columns
- [ADR-0014: Timewalk](0014-Timewalk.md) (temporal traversal providing measure discovery)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md) (immutable data structures enabling stage separation)
- [Knuth–Plass line-breaking algorithm](https://en.wikipedia.org/wiki/Knuth%E2%80%93Plass_line-breaking_algorithm) - Wikipedia overview
- Knuth, D.E. and Plass, M.F. "Breaking Paragraphs into Lines", *Software—Practice and Experience* 11 (1981), pp. 1119–1184 - foundational algorithm
- Hegazy, W.A. and Gourlay, J.S. "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, The Ohio State University (1987); also in *Proceedings of the International Conference on Electronic Publishing, Document Manipulation and Typography*, Nice, 1988, ed. J.C. van Vliet, Cambridge University Press
- LilyPond source: `lily/gourlay-breaking.cc` (1997–2006), `lily/constrained-breaking.cc`, `lily/simple-spacer.cc`, `lily/spring.cc`, `lily/spacing-basic.cc`, `lily/optimal-page-breaking.cc`, `lily/page-turn-page-breaking.cc`
- MuseScore source (main branch, October 2026): `src/engraving/rendering/score/horizontalspacing.cpp`, `src/engraving/rendering/score/systemlayout.cpp`; MuseScore Handbook 4, "Score size and spacing"
- Spreadbury, D., Dorico development diary, part 11 (June 2015), https://blog.dorico.com/2015/06/development-diary-part-11/ (including the comments by Han-Wen Nienhuys and Christof Schardt); Steinberg forum, "Casting Off" (December 2016), https://forums.steinberg.net/t/casting-off/670560; "Optimal line breaking algorithm" (April 2026), https://forums.steinberg.net/t/optimal-line-breaking-algorithm/1028999
- Taupin, D., Mitchell, R. and Egler, A. "MusiXTeX : L'écriture de la musique polyphonique ou instrumentale avec TeX", *Cahiers GUTenberg* 21 (1995), pp. 107–113
- The MusiXTeX manual, §1.3.1 "The three pass system: Introduction"; `musixflx` (Ross Mitchell, 1992–1997)
- Steinbach, A. and Schofer, A. *Automatisierter Notensatz mit TeX*, master's thesis, Rheinische Friedrich-Wilhelms Universität, Bonn (1987)
- Ross, T. "The Art of Music Engraving and Processing" (1970) - proportional spacing values
- Gould, E. "Behind Bars" (2011) - modern engraving standards