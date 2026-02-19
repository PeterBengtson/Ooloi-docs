# ADR-0013: Slur Formatting with Convex Hull and Bézier Curves

## Status: Accepted (Revised 18 February 2026)

## Context

Ooloi needs to render slurs as smooth curves with variable thickness connecting musical elements across temporal spans. Slur formatting is part of **pipeline stage 5** (Spanners and Margins) in the [Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md). By stage 5, atom positions are finalized from stage 4, and each musician's spanners (ties, slurs, beams, hairpins, pedal markings, ottava lines) are computed independently in a fan-out per musician — no cross-musician coordination required. Stage 5 is **height-complete**: system heights become definitive here, providing stage 6 (page breaking) with complete information. See [ADR-0028 §Stage 5](0028-Hierarchical-Rendering-Pipeline.md#pipeline-stage-5-spanners-and-margins-fan-out-per-musician) for the full pipeline architecture.

The core challenges are:

1. **Point Collection**: Gathering x,y coordinates for all musical elements under a slur's span using temporal coordination
2. **Shape Determination**: Converting collected coordinate points into a natural slur shape that follows the melodic contour
3. **Variable Thickness Rendering**: Creating slur shapes with rounded endpoints and variable thickness, following copper plate engraving aesthetics
4. **Visual Integration**: Storing the resulting curves in the appropriate MeasureView visual hierarchy structures

This ADR establishes the foundational algorithms for slur shape computation. A follow-up ADR ([ADR-00XX](00XX-Slur-and-Tie-Geometry.md)) will address the complete geometric constraint solving system — collision detection, the progressive solver, nested and overlapping slurs, cross-system breaks, and the interaction between slurs and other notation elements within the phase-5 processing sequence.

## Musical Slur Characteristics (Based on Traditional Engraving)

Analysis of professional musical scores reveals these key slur formatting principles:

### Visual Examples

![Basic slur and tie examples](../img/slurs/slur-1.png)

*Figure 1: Basic slur vs. tie comparison showing thickness and curvature differences*

![Slur placement examples](../img/slurs/slur-2.jpeg)

*Figure 2: Professional slur placement examples showing proper curvature and positioning*

![Complex multi-slur passage](../img/slurs/slur-3.png)

*Figure 3: Overlapping slurs in professional piano score showing nested curves and collision avoidance*

## Decision

We will implement slur formatting through:

1. **General extent point collection** using `collect-extent-items` function that works for any spanning attachment
2. **Convex half-hull calculation** to determine natural slur shape from collected notehead positions
3. **Layout-aware coordinate retrieval** using composable transducers for x,y position calculation
4. **Dual Bézier curve rendering** with variable thickness, rounded endpoints (copper plate aesthetics), and MeasureView integration — curve generation algorithm defined in ADR-00XX

The convex half-hull is the **primary shape model** for slurs that must follow a melodic contour. It produces curves that hug the noteheads — as close as possible to the music, even passing under articulations which the solver may adjust. For unrestricted slurs (no obstacles along the path), the Euler elastica provides the ideal shape; see ADR-00XX for the progressive solver that selects between these approaches.

## Detailed Design

### 1. General Extent Point Collection

```clojure
(defn collect-extent-items
  "Collect all musical items under any spanning attachment using timewalker.
   Works for any spanner attachment."
  [piece attachment start-vpd layout]
  (let [canonical-start-vpd (vpd/canonicalize start-vpd)
        instrument-boundary-vpd (vec (take 4 canonical-start-vpd))
        current-measure (or (get canonical-start-vpd 9) 0)
        ;; Find endpoint using existing attachment resolution
        [end-item end-vpd end-position] (endpoint-item piece attachment start-vpd)
        end-measure (if end-vpd (or (get end-vpd 9) current-measure)
                                (+ current-measure MAX_LOOKAHEAD_MEASURES -1))]
    (if end-item
      (sequence (comp
                 (timewalk {:boundary-vpd instrument-boundary-vpd
                           :start-measure current-measure
                           :end-measure end-measure})
                 (filter takes-attachment?)  ; Only items that can have attachments
                 (obtain-xy layout))  ; Transform to x,y coordinates
                [piece])
      [])))
```

**Key Features**:
- **General purpose**: Works for slurs, hairpins, ottavas, pedal markings — any `spanner?` attachment
- **Temporal coordination**: Uses timewalker ([ADR-0014](0014-Timewalk.md)) to ensure proper musical time ordering
- **Layout awareness**: Uses `(obtain-xy layout)` transducer to get coordinates from specific layout
- **Boundary scoping**: Limits search to relevant instrument for performance

### 2. Layout Coordinate Transformation

```clojure
(defn obtain-xy
  "Composable transducer that transforms timewalker results to x,y coordinates.
   Takes a layout and returns a transducer for use in timewalker pipelines."
  [layout]
  (map (fn [result]
         (let [item (item result)
               vpd (vpd result)
               position (position result)]
           {:x (calculate-x-coordinate item vpd layout)
            :y (calculate-y-coordinate item vpd layout)
            :item item
            :vpd vpd
            :position position}))))
```

**Design Principles**:
- **Composable transducer**: Integrates seamlessly with timewalker pipelines
- **Layout-specific**: Each layout may have different x,y positions due to transposition, spacing, etc.
- **Complete information**: Preserves original timewalker result data alongside coordinates

### 3. Slur-Specific Hull Calculation

```clojure
(defn calculate-slur-hull [points above?]
  "Calculate upper or lower convex hull for slur shape, preserving left-to-right order.
   Points are already in temporal order from timewalker - no sorting required."
  (let [comparator (if above? <= >=)]
    (reduce (fn [hull point]
              (loop [hull hull]
                (if (and (>= (count hull) 2)
                         (comparator (cross-product (peek (pop hull)) (peek hull) point) 0))
                  (recur (pop hull))
                  (conj hull point))))
            [] points)))

(defn cross-product [p1 p2 p3]
  "Calculate cross product for hull computation."
  (- (* (- (:x p2) (:x p1)) (- (:y p3) (:y p1)))
     (* (- (:y p2) (:y p1)) (- (:x p3) (:x p1)))))
```

**Why Convex Hull for Slurs**:
- **Notehead-hugging**: The hull gives the minimum-clearance contour — the slur follows the melodic landscape rather than floating above it
- **Preserves note order**: Timewalker provides temporal order, maintaining left-to-right musical sequence without sorting
- **Natural shape**: Upper/lower hull matches traditional slur placement where the curve traces the notehead contour
- **Mathematical stability**: O(n) algorithm, numerically stable

#### From Angular Hull to Smooth Curve

The convex hull is a polygon — angular vertices connected by straight segments. The actual slur is a smooth curve. The hull provides the **constraint envelope**: the set of points the slur must pass through or near. A Bézier curve is then fitted to these hull points to produce the smooth shape.

If a single cubic Bézier can pass through (or within tolerance of) all hull points while respecting configurable quality criteria (maximum slope, maximum notehead departure, minimum curvature radius, endpoint tangent angle) — the slur is done. When the simple fit violates any criterion, the solver escalates to composite Béziers, higher-degree curves, or constrained optimisation. The hull defines *where* the slur goes; the Bézier fitting defines *how smoothly* it gets there. Quality criteria, escalation thresholds, and the progressive solver are defined in ADR-00XX.

### 4. The Euler Elastica: Ideal Unrestricted Shape

The convex hull gives the right shape when there is a melodic landscape to follow. But what about a short slur over two adjacent notes with nothing in between? Or a tie between two noteheads at the same pitch? These have no intermediate points to form a hull — they need a different shape model.

The **Euler elastica** is that model. It minimises the integral of squared curvature along a curve between fixed endpoints:

```
E[C] = ∫₀ᴸ κ(s)² ds

minimised subject to: C(0) = p₀, C(L) = p₁, C'(0) = t₀, C'(L) = t₁
```

This is the curve a thin elastic rod assumes between constrained endpoints — hold a thin metal ruler at both ends and the curve it takes is an elastica. Engravers draw slurs that approximate elastica because minimum-curvature-variation curves are what the eye recognises as aesthetically correct. Gould's rules for slur curvature are a partial verbal encoding of elastica properties.

The elastica serves three roles:

1. **Unrestricted shape**: When no obstacles lie along the slur path, the elastica is the ideal curve. This is the fast path for simple slurs and most ties. The 30%/70% control point placement (§6) is a Bézier approximation of the elastica for practical computation.

2. **Rotation-invariant**: A slur ascending from a low note to a high note, descending, or connecting notes at the same height all take the same elastica shape rotated to match the endpoint geometry. This gives visual consistency across different musical contexts.

3. **Collision probe**: Fit the elastica between endpoints and test whether any element along the span intersects it. If nothing collides — done, use the elastica. If collisions are detected — escalate to the hull-based approach. This makes the elastica the entry point of the progressive solver defined in ADR-00XX.

The relationship between elastica and hull is not competitive — they address different cases:

- **Unrestricted** (elastica clears everything): keep the elastica. No hull needed.
- **Restricted** (elastica collides with the melodic landscape): the hull gives the contour the slur must follow, fitted with a Bézier as close to the noteheads as possible.

The key reference is Levien (2008), which provides numerical methods for elastica computation and Bézier approximation with known error bounds.

### 5. Minkowski Clearance for Thick Curves

A slur is not a mathematical line — it has physical thickness that varies along its length. Correct clearance computation must account for this extent, not just the centreline.

The **Minkowski sum** provides the mathematically exact approach:

```
Effective obstacle = Obstacle ⊕ SlurProfile(t)
```

In plain terms: take each obstacle (notehead, stem, accidental) and inflate it outward by the slur's half-thickness at the corresponding curve parameter *t*. If the centreline clears the inflated obstacles, the actual thick slur clears the actual obstacles.

With copper plate rounded endpoints (§6), the slur half-thickness *r(t)* is never zero — even at the endpoints, the inflation equals the end cap radius. This eliminates the degenerate zero-width case that arises with pointed endpoints, making the clearance computation well-behaved everywhere along the curve.

**Practical simplification**: for simple placement (elastica within available space), treating the slur thickness as uniform at its maximum gives conservative clearance — if the centreline clears the maximally-inflated obstacles, the actual slur certainly clears. The exact Minkowski profile matters most in tight passages where the progressive solver (ADR-00XX) has escalated to constrained optimisation, and the precise clearance at the thick midsection vs the thinner endpoints makes a difference.

### 6. Bézier Curve Generation with Variable Thickness

#### Engraving Standards

All formatting parameters are user-accessible through Ooloi's settings system ([ADR-0016](0016-Settings.md), [ADR-0043](0043-Frontend-Settings.md)). The values below are representative defaults; production code references configurable settings. This enables users to match different publishing houses' style guidelines.

**Endpoint Shape — Copper Plate Aesthetics:**

In traditional copper plate engraving, the burin entering and leaving the plate produces natural rounding at line terminations. Ooloi adopts this as the default: nothing tapers to a geometric point. Slur and tie endpoints have configurable cap radius settings; the exact settings architecture (per-element, global floor, or combination) is an implementation decision not yet made. The Minkowski clearance system (§5) accounts for this thickness during placement.

| Parameter | Range | Description |
|---|---|---|
| End cap radius | 0.08–0.12 staff spaces | Rounded tip at each endpoint |
| Maximum thickness | 0.12–0.15 staff spaces | At shoulder position |
| Shoulder position | 0.25–0.35 from endpoint | Where maximum thickness occurs |
| Minimum thickness | 0.16–0.24 staff spaces | At endpoint (= 2 × end cap radius); never zero |

**Slur Height Guidelines:**
- **Short slurs** (2-4 notes): 1.5-2 staff spaces above note heads
- **Medium slurs** (5-8 notes): 2-3 staff spaces above note heads
- **Long slurs** (9+ notes or cross-barline): 3-4 staff spaces above note heads
- **Clearance**: Minimum 0.5 staff space between slur and note heads/stems

**Curvature Characteristics:**
- **Control point placement**: 30% and 70% along horizontal span
- **Natural arc**: Follows mathematical curves, not manual sketching
- **Consistency**: Similar spans should produce similar curvature
- **Cross-barline**: Maintains smooth arc across measure boundaries

**Placement Rules:**
- **Single-voice passages**: Slurs go on the notehead side, not the stem side
- **Multi-voice contexts**: Different voices use opposite slur directions when possible
- **Proximity to music**: Slurs should be as close to the noteheads as possible, even under articulations which the solver may adjust
- **Voice separation**: Upper voice slurs typically above, lower voice slurs below

The rendering approach is dual Bézier curves (top edge and bottom edge) with fill between them and rounded end caps. The curve generation algorithm will be defined in ADR-00XX as part of the complete progressive solver.

### Edge Cases

This ADR establishes the basic algorithm. The following cases require the progressive solver and constraint system defined in ADR-00XX:

1. **Nested slurs**: Inner slur takes its natural shape; outer slur encompasses it. Deterministic post-processing adjusts only on detected tight fit.
2. **Overlapping slurs**: Independently placed; local collision adjustment only where overlap is detected.
3. **Cross-system slurs**: Break into two curves at line breaks with visual continuation.
4. **Stem direction and beam conflicts**: Slurs interact with beam groups; processing sequence ensures beams are resolved before slurs.
5. **Accidental and articulation collisions**: The solver may move articulations to accommodate slur proximity to noteheads.

## Rationale

### Algorithm Choices

1. **General extent collection**: `collect-extent-items` works for any spanning attachment, enabling code reuse across slurs, hairpins, ottavas, etc.

2. **Timewalker integration**: Ensures proper temporal coordination and handles nested musical structures automatically.

3. **Layout abstraction**: `(obtain-xy layout)` transducer allows same algorithm to work with different layout contexts (transposed parts, different spacing, etc.).

4. **Convex hull as primary shape**: The half-hull produces notehead-hugging curves that follow the melodic contour — the right default for music. Polynomial fitting or manual control points cannot achieve this without explicit knowledge of the note positions. For unrestricted paths (no obstacles), the Euler elastica provides the smoothest possible curve; the progressive solver in ADR-00XX selects between them.

5. **Dual Bézier curves with rounded ends**: The rendering approach — two edge curves with fill between them and copper plate endpoint rounding — is specified here; the curve generation algorithm belongs to the progressive solver in ADR-00XX.

6. **Single-staff focus**: Simplifies implementation while covering the majority of slur cases. Multi-staff spanning can be addressed in future iterations.

### Performance Characteristics

- **Time Complexity**: O(n) for hull calculation where n = points under slur
- **Memory**: Linear in number of points under slur
- **Scalability**: Boundary scoping limits processing to single instrument
- **Composability**: All operations use transducers for efficient pipeline composition

## Consequences

### Positive

- **Notehead-hugging shapes**: Hull algorithm produces curves that follow the melodic contour rather than floating independently
- **Code reuse**: `collect-extent-items` works for all spanning attachments
- **Layout flexibility**: Same algorithm works with different layout contexts
- **Mathematical stability**: Convex hull and Bézier algorithms are numerically robust
- **Performance**: Efficient algorithms suitable for interactive editing
- **Copper plate aesthetics**: Rounded endpoints match traditional engraving quality

### Negative

- **Complexity**: More sophisticated than simple straight-line connections
- **Single-staff limitation**: Multi-staff slurs require additional complexity (future work)
- **Hull alone insufficient**: Complex cases require the progressive solver (ADR-00XX)

### Neutral

- **Domain-specific**: Optimized for musical applications rather than general curve fitting
- **Layout dependency**: Requires coordinate calculation from layout system

## Related Decisions

- **[ADR-00XX: Slur and Tie Geometry](00XX-Slur-and-Tie-Geometry.md)** — Follow-up: progressive solver, collision detection, nested/overlapping slurs, cross-system breaks, tie geometry, and the complete phase-5 processing sequence
- **[ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md)** — Slur formatting is part of pipeline stage 5 (Spanners and Margins)
- **[ADR-0038: Backend-Authoritative Rendering](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)** — Rendering boundary constraints: the slur algorithm consumes resolved semantics and layout decisions without introducing new semantic state or backward causality
- **[ADR-0014: Timewalk](0014-Timewalk.md)** — Timewalking provides the point collection algorithm
- **[ADR-0011: Shared Structure](0011-Shared-Structure.md)** — Shared structure concepts that slur formatting builds upon
- **[ADR-0008: VPDs](0008-VPDs.md)** — VPD addressing system used for navigation
- **[ADR-0010: Pure Trees](0010-Pure-Trees.md)** — Tree structure that the algorithm traverses
- **[ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md)** — Copper plate aesthetic principle: all Skija-rendered line terminations and corners default to rounded
- **[ADR-0016: Settings](0016-Settings.md)** / **[ADR-0043: Frontend Settings](0043-Frontend-Settings.md)** — All slur formatting parameters are user-configurable

## See Also

- **[Frontend Architecture Guide](../guides/FRONTEND_ARCHITECTURE_GUIDE.md)** — Window lifecycle, rendering pipeline context, and the rendering boundary constraints that slur formatting must respect
