# ADR: Slur Formatting with Temporal Stream Processing

## Status: Accepted (Revised 20 July 2025)

## Context

Ooloi needs to efficiently handle the creation, formatting, and rendering of slurs in musical notation. This involves two distinct architectural concerns:

**Musical Hierarchy (Creation)**: Establishing slur relationships within the pure musical tree structure
**Visual Hierarchy (Formatting)**: Computing visual representations (glyphs and Bézier curves) in the separate visual tree, lazily recomputed as needed

This separation ensures:
1. Musical relationships remain independent of visual representation
2. Visual formatting can be recomputed efficiently without affecting musical data
3. The timewalker operates on musical structure to generate visual output
4. Lazy evaluation minimises computational overhead for large scores

The timewalker's temporal coordination transforms slur processing from complex manual traversal into elegant stream operations, bridging musical relationships and visual computation.

Key architectural insights:
- Slurs represent temporal-spatial relationships that map naturally to stream filtering
- The timewalker's boundary control eliminates the need for bespoke search strategies
- Trait-based filtering replaces manual type checking and nested structure handling
- Temporal coordination ensures proper musical ordering for complex slur spans

## Decision

We will implement slur handling using:
1. VPD-based slur definitions with temporal boundaries
2. Timewalker stream processing for point collection
3. Trait-based filtering for robust element identification
4. Position provider abstraction for accurate coordinate calculation
5. Convex hull calculation for natural slur shape determination
6. Bézier curve generation for smooth visual rendering

## Detailed Design

### 1. Slur Creation in Musical Hierarchy

```clojure
(ns ooloi.slur
  (:require [ooloi.backend.ops.timewalk :refer [timewalk]]
            [ooloi.backend.models.core :refer :all]))

(defrecord Slur [start-vpd end-vpd placement])

(defn create-slur [piece start-vpd end-vpd placement]
  "Create slur relationship in musical hierarchy - no visual computation."
  (let [slur (->Slur start-vpd end-vpd placement)
        slur-id (generate-id)]
    (-> piece
        (add-attachment start-vpd slur-id slur)
        (mark-visual-dirty start-vpd end-vpd))))  ; Flag visual recomputation needed

(defn slur-temporal-scope [slur]
  "Determine optimal boundary VPD and measure range for slur processing."
  (let [start-measure (vpd->measure-number (:start-vpd slur))
        end-measure (vpd->measure-number (:end-vpd slur))
        common-scope (find-common-scope (:start-vpd slur) (:end-vpd slur))]
    {:boundary-vpd common-scope
     :start-measure start-measure  
     :end-measure end-measure}))
```

### 2. Position Provider Abstraction

```clojure
(defprotocol PositionProvider
  (pitch->coordinates [this pitch vpd] "Convert pitch to x,y coordinates"))

(defrecord ScorePositionProvider [layout-context]
  PositionProvider
  (pitch->coordinates [this pitch vpd]
    {:x (calculate-x-position pitch vpd layout-context)
     :y (calculate-y-position pitch vpd layout-context)
     :pitch pitch
     :vpd vpd}))
```

### 3. Visual Hierarchy Formatting (Lazy Computation)

```clojure
(defn format-slur-visual [piece slur position-provider]
  "Compute visual representation in visual hierarchy - called lazily when needed."
  (let [points (collect-slur-points piece slur position-provider)
        above? (= (:placement slur) :above)
        hull (calculate-hull points above?)
        control-points (calculate-control-points hull above?)
        curve-points (generate-bezier-curve control-points 100)]
    {:type :bezier-curve
     :points curve-points
     :stroke-width 1.5
     :fill nil}))

(defn update-visual-hierarchy [visual-tree piece slur position-provider]
  "Add computed slur curve to visual hierarchy at appropriate location."
  (let [visual-slur (format-slur-visual piece slur position-provider)
        target-vpd (slur-visual-location slur)]
    (add-visual-element visual-tree target-vpd visual-slur)))

(defn lazy-slur-formatting [visual-tree piece slurs position-provider]
  "Lazily compute slur visuals only when visual hierarchy is accessed."
  (reduce (partial update-visual-hierarchy piece position-provider)
          visual-tree
          (filter needs-visual-update? slurs)))
```

### 4. Stream-Based Point Collection (Musical → Visual Bridge)

```clojure
(defn within-slur? [result slur]
  "Check if a timewalker result falls within the slur's temporal span."
  (let [item-vpd (vpd result)
        start-pos (vpd->absolute-position (:start-vpd slur))
        end-pos (vpd->absolute-position (:end-vpd slur))
        item-pos (vpd->absolute-position item-vpd)]
    (and (>= item-pos start-pos) (<= item-pos end-pos))))

(defn collect-slur-points [piece slur position-provider]
  "Bridge musical hierarchy to visual coordinates using temporal stream processing."
  (let [scope (slur-temporal-scope slur)]
    (->> (timewalk piece scope)
         (filter takes-attachment?)  ; Only elements that can have slurs
         (filter #(within-slur? % slur))
         (map (fn [result]
                (pitch->coordinates position-provider 
                                   (item result) 
                                   (vpd result)))))))
```

### 5. Compositional Slur Processing

```clojure
(defn slur-processing-pipeline [position-provider]
  "Reusable transducer for slur point processing."
  (comp (filter takes-attachment?)
        (map (fn [result]
               (pitch->coordinates position-provider 
                                  (item result) 
                                  (vpd result))))))

(defn process-slur-in-context [piece slur position-provider context]
  "Process slur with specific temporal context (e.g., system, page)."
  (let [scope (merge (slur-temporal-scope piece slur) context)]
    (sequence (comp (timewalk scope)
                    (filter #(within-slur? % slur))
                    (slur-processing-pipeline position-provider))
              [piece])))
```

### 6. Hull Calculation (Unchanged - Still Optimal)

```clojure
(defn cross-product [[x1 y1] [x2 y2] [x3 y3]]
  (- (* (- x2 x1) (- y3 y1))
     (* (- y2 y1) (- x3 x1))))

(defn calculate-hull [points above?]
  "Calculate upper or lower convex hull preserving left-to-right order."
  (let [comparator (if above? <= >=)]
    (reduce (fn [hull point]
              (loop [h hull]
                (if (and (>= (count h) 2)
                         (comparator (cross-product (peek (pop h)) (peek h) point) 0))
                  (recur (pop h))
                  (conj h point))))
            [] points)))
```

### 7. Bézier Curve Generation (Unchanged - Still Optimal)

```clojure
(defn calculate-control-points [hull above?]
  "Generate control points for smooth Bézier curve from hull."
  (let [start (first hull)
        end (last hull)
        extreme-point (apply (if above? max-key min-key) :y hull)
        offset (if above? 10 -10)
        control1 {:x (+ (:x start) (* 0.25 (- (:x end) (:x start))))
                  :y (+ (:y extreme-point) offset)}
        control2 {:x (+ (:x start) (* 0.75 (- (:x end) (:x start))))
                  :y (+ (:y extreme-point) offset)}]
    [start control1 control2 end]))

(defn bezier-point [t [p0 p1 p2 p3]]
  "Calculate point on cubic Bézier curve at parameter t."
  (let [t1 (- 1 t)
        t2 (* t t)
        t3 (* t2 t)]
    {:x (+ (* t1 t1 t1 (:x p0))
           (* 3 t1 t1 t (:x p1))
           (* 3 t1 t2 (:x p2))
           (* t3 (:x p3)))
     :y (+ (* t1 t1 t1 (:y p0))
           (* 3 t1 t1 t (:y p1))
           (* 3 t1 t2 (:y p2))
           (* t3 (:y p3)))}))

(defn generate-bezier-curve [control-points steps]
  "Generate points along Bézier curve for rendering."
  (map #(bezier-point (/ % steps) control-points) (range (inc steps))))
```

### 8. Complete Slur Pipeline (Musical Creation + Visual Formatting)

```clojure
(defn render-slur 
  "Complete slur rendering as composable transformation."
  ([position-provider] 
   (fn [slur]
     (comp (timewalk (slur-temporal-scope piece slur))
           (filter #(within-slur? % slur))
           (slur-processing-pipeline position-provider)
           (map (fn [points]
                  (let [above? (= (:placement slur) :above)
                        hull (calculate-hull points above?)
                        control-points (calculate-control-points hull above?)]
                    (generate-bezier-curve control-points 100)))))))

(defn format-slur [piece slur position-provider]
  "Format single slur with full rendering pipeline."
  (let [points (collect-slur-points piece slur position-provider)
        above? (= (:placement slur) :above)
        hull (calculate-hull points above?)
        control-points (calculate-control-points hull above?)]
    {:slur slur
     :points points
     :hull hull  
     :control-points control-points
     :curve-points (generate-bezier-curve control-points 100)}))

(defn format-all-slurs [piece slurs position-provider]
  "Process multiple slurs efficiently using stream composition."
  (sequence (comp (map #(format-slur piece % position-provider))
                  (filter #(not-empty (:points %))))
            slurs))
```

## Rationale

### Paradigm Shift: From Manual Traversal to Stream Processing

1. **Architectural Separation**: Musical relationships (slur creation) remain in the pure musical hierarchy, whilst visual computation (formatting) occurs lazily in the separate visual hierarchy.

2. **Temporal Coordination**: The timewalker's natural temporal ordering eliminates complex search strategies. Musical time becomes the organizing principle for bridging musical and visual representations.

3. **Trait-Based Filtering**: `(filter takes-attachment?)` replaces manual type checking across nested structures like chords, tuplets, and tremolandos.

4. **VPD-Based Scope**: Slur boundaries map naturally to VPD scope control, eliminating bespoke traversal logic.

5. **Lazy Visual Computation**: Visual hierarchy elements (Bézier curves, glyphs) are computed only when needed, keeping musical data separate from presentation.

6. **Compositional Processing**: Slur rendering becomes a reusable transformation that bridges musical structure to visual output.

### Technical Benefits

1. **Clean Architectural Separation**: Musical hierarchy remains pure whilst visual hierarchy contains only rendering elements
2. **Lazy Visual Computation**: Bézier curves computed only when visual hierarchy is accessed  
3. **No Adaptive Search Required**: Timewalker provides optimal traversal by design
4. **Natural Boundary Handling**: VPD scope control handles complex slur spans automatically  
5. **Trait System Integration**: Robust element identification without manual type checking
6. **Temporal Guarantees**: Items arrive in proper musical time order
7. **Memory Efficiency**: Lazy evaluation processes only what's needed
8. **Visual Independence**: Musical changes don't immediately trigger visual recomputation

### What Disappears

- Complex `adaptive-search` implementation
- Manual `extract-pitches` recursion across nested structures  
- Bespoke ID resolution mechanisms
- Edge case handling for structure traversal
- Complex transducer composition for basic operations

### What Emerges

- **Musical Intent as Code**: Slur processing reads like musical thinking
- **Compositional Reusability**: Same transformation works across different contexts
- **Natural Performance**: Stream processing scales automatically
- **Robust Error Handling**: Trait-based filtering is inherently safe

## Consequences

### Positive

- **Clean Architectural Separation**: Musical and visual concerns completely decoupled
- **Lazy Visual Efficiency**: Visual computations occur only when rendering is needed
- **Dramatic Simplification**: Eliminates entire categories of traversal complexity
- **Musical Clarity**: Code directly expresses musical relationships  
- **Performance by Design**: Lazy evaluation and temporal coordination provide optimal processing
- **Compositional Power**: Slur rendering becomes part of larger formatting pipelines
- **Robust Processing**: Trait-based filtering handles edge cases automatically
- **Reusable Abstractions**: Same patterns apply to ties, hairpins, and other spanning elements
- **Memory Efficiency**: Visual hierarchy grows only as needed, not with musical complexity

### Negative

- **Paradigm Learning Curve**: Developers must understand stream-based musical thinking
- **Temporal Reasoning**: Requires understanding of VPD-based temporal boundaries

### Neutral

- **Hull Calculation Unchanged**: Geometric algorithms remain optimal as designed
- **Position Provider**: Coordinate calculation abstraction still required
- **Bézier Generation**: Curve mathematics unchanged from original design

## Implementation Notes

1. **Leverage Timewalker Patterns**: Use established temporal coordination for all spanning elements
2. **Trait-Based Design**: Extend `takes-attachment?` trait for new musical elements  
3. **VPD Scope Optimization**: Cache common scope calculations for performance
4. **Stream Composition**: Build reusable transformation libraries for musical formatting
5. **Early Termination**: Use `take` and filtering for efficient processing of large scores
6. **Memory Management**: Rely on timewalker's lazy evaluation for memory efficiency

## Future Considerations

1. **Universal Spanning Elements**: Extend pattern to ties, hairpins, glissandos, ottavas
2. **Parallel Processing**: Leverage stream processing for concurrent slur rendering  
3. **Visual Debugging**: Stream processing enables elegant debugging pipelines
4. **Machine Learning Integration**: Stream transformations could feed ML curve optimization
5. **Real-Time Formatting**: Temporal streams enable incremental slur updates
6. **Cross-Staff Optimization**: Timewalker's temporal coordination naturally handles complex spans

## Meta-Insight

This revision demonstrates the paradigm shift from **imperative object manipulation** to **functional stream processing** in musical software. Slurs become temporal-spatial filters operating on musical event streams rather than complex graph traversals.

The timewalker transforms slur rendering from a **traversal problem** into a **stream transformation problem**, making musical intent explicit in the computational model.
