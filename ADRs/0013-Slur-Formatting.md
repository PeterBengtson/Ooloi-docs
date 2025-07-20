# ADR: Slur Formatting with Temporal Stream Processing

## Status: Accepted (Revised 20 July 2025)

## Context

Ooloi needs to efficiently handle the creation, formatting, and rendering of slurs in musical notation. This builds upon Ooloi's existing attachment architecture, which already uses temporal stream processing for endpoint resolution.

The system involves three distinct architectural layers:

**Musical Hierarchy (Creation)**: Slurs are created as `ExtendsForward` attachments using the existing attachment system
**Endpoint Resolution**: The `endpoint-item` multimethod already uses timewalker to locate attachment endpoints through temporal stream processing  
**Visual Hierarchy (Formatting)**: Computing visual representations (Bézier curves) in the separate visual tree, lazily recomputed as needed

This separation ensures:
1. Musical relationships remain independent of visual representation
2. Endpoint resolution leverages proven temporal stream patterns
3. Visual formatting can be recomputed efficiently without affecting musical data
4. The attachment resolver system provides intuitive string-based slur creation
5. Lazy evaluation minimises computational overhead for large scores

The timewalker's temporal coordination transforms slur processing from complex manual traversal into elegant stream operations, building on established patterns for attachment endpoint resolution.

Key architectural insights:
- Slurs represent temporal-spatial relationships that map naturally to stream filtering
- The timewalker's boundary control eliminates the need for bespoke search strategies
- Trait-based filtering replaces manual type checking and nested structure handling
- Temporal coordination ensures proper musical ordering for complex slur spans

## Decision

We will implement slur handling building upon Ooloi's existing attachment architecture:
1. Slur creation using the established `ExtendsForward` attachment system
2. Endpoint resolution leveraging existing timewalker-based `endpoint-item` multimethod
3. Attachment resolver integration for intuitive string-based slur creation (`"slur"`)
4. Visual formatting using temporal stream processing for point collection
5. Position provider abstraction for accurate coordinate calculation
6. Convex hull calculation for natural slur shape determination
7. Bézier curve generation for smooth visual rendering

## Detailed Design

### 1. Slur Creation Using Existing Attachment System

```clojure
(ns ooloi.slur-integration
  (:require [ooloi.backend.models.core :refer :all]))

;; Slur creation leverages existing attachment resolver and add-attachment API
(defn create-slur-attachment [piece start-vpd end-vpd]
  "Create slur using established attachment system - builds on existing patterns."
  (add-attachment start-vpd piece "slur" end-vpd))

;; Alternative: Direct attachment creation for programmatic use
(defn create-slur-programmatic [piece start-vpd end-vpd]
  "Create slur using direct attachment creation."
  (let [slur (create-slur)]  ; Uses existing slur constructor
    (add-attachment start-vpd piece slur end-vpd)))

;; The attachment system automatically:
;; 1. Resolves "slur" string to Slur instance via attachment-resolver
;; 2. Generates endpoint-id for the end-vpd TakesAttachment item
;; 3. Sets the endpoint-id on the slur attachment
;; 4. Adds the slur to the start item's attachment vector
;; 5. Marks visual hierarchy as needing recomputation
```

### 2. Endpoint Resolution (Already Implemented)

```clojure
;; The endpoint-item multimethod already uses timewalker for slur endpoint resolution
;; This is the EXISTING implementation from ooloi.backend.models.traits.attachment

(m/defmethod endpoint-item ::h/ExtendsForward
  [piece attachment start-vpd]
  "Existing implementation - slurs use this pattern for endpoint resolution."
  (let [target-endpoint-id (:endpoint-id attachment)]
    (if target-endpoint-id
      (let [canonical-start-vpd (vpd/canonicalize start-vpd)
            instrument-boundary-vpd (vec (take 4 canonical-start-vpd))
            current-measure (or (get canonical-start-vpd 9) 0)
            end-measure (+ current-measure MAX_LOOKAHEAD_MEASURES -1)
            ;; Temporal stream processing - the pattern we use everywhere
            result (first (sequence (comp 
                                     (timewalk {:boundary-vpd instrument-boundary-vpd 
                                               :start-measure current-measure
                                               :end-measure end-measure})
                                     (filter p/takes-attachment?)
                                     (filter #(= target-endpoint-id (ta/get-endpoint-id (item %))))
                                     (take 1))
                                   [piece]))]
        (or result [nil nil nil]))
      [nil nil nil])))

;; Usage: Find the endpoint of any slur
(defn get-slur-endpoint [piece slur start-vpd]
  "Get slur endpoint using existing endpoint-item multimethod."
  (endpoint-item piece slur start-vpd))
```

### 3. Position Provider for Visual Coordinates

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

### 4. Visual Hierarchy Formatting (Building on Attachment Patterns)

```clojure
(defn collect-slur-points [piece slur start-vpd position-provider]
  "Collect points for slur rendering using same temporal pattern as endpoint resolution."
  (let [canonical-start-vpd (vpd/canonicalize start-vpd)
        instrument-boundary-vpd (vec (take 4 canonical-start-vpd))
        current-measure (or (get canonical-start-vpd 9) 0)
        ;; Find the endpoint first using existing endpoint resolution
        [end-item end-vpd end-position] (endpoint-item piece slur start-vpd)
        end-measure (if end-vpd (or (get end-vpd 9) current-measure) 
                                (+ current-measure MAX_LOOKAHEAD_MEASURES -1))]
    (if end-item
      ;; Use temporal stream processing to collect all points between start and end
      (sequence (comp 
                 (timewalk {:boundary-vpd instrument-boundary-vpd 
                           :start-measure current-measure
                           :end-measure end-measure})
                 (filter takes-attachment?)
                 (filter #(temporal-between? % start-vpd end-vpd))  ; Within slur span
                 (map (fn [result]
                        (pitch->coordinates position-provider 
                                           (item result) 
                                           (vpd result)))))
                [piece])
      [])))  ; No endpoint found, empty point collection

(defn format-slur-visual [piece slur start-vpd position-provider]
  "Compute visual representation - called lazily when visual hierarchy accessed."
  (let [points (collect-slur-points piece slur start-vpd position-provider)]
    (if (seq points)
      (let [above? true  ; Could derive from slur placement or automatic detection
            hull (calculate-hull points above?)
            control-points (calculate-control-points hull above?)
            curve-points (generate-bezier-curve control-points 100)]
        {:type :bezier-curve
         :points curve-points
         :stroke-width 1.5
         :fill nil})
      nil)))  ; No points to render

(defn lazy-slur-formatting [visual-tree piece slurs position-provider]
  "Lazily compute slur visuals only when visual hierarchy is accessed."
  (reduce (fn [tree [slur start-vpd]]
            (when-let [visual-slur (format-slur-visual piece slur start-vpd position-provider)]
              (add-visual-element tree (slur-visual-location slur) visual-slur)))
          visual-tree
          (filter #(needs-visual-update? (first %)) slurs)))
```

### 5. Compositional Slur Processing (Building on Attachment Patterns)

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

### 8. Complete Integration (Attachment System + Visual Formatting)

```clojure
(defn create-and-format-slur 
  "Complete slur workflow: creation through attachment system + visual formatting."
  [piece start-vpd end-vpd position-provider]
  ;; 1. Create slur using existing attachment system
  (let [updated-piece (add-attachment start-vpd piece "slur" end-vpd)
        ;; 2. Retrieve the created slur using existing attachment API
        slur (last (get-attachments (retrieve start-vpd updated-piece)))
        ;; 3. Use existing endpoint resolution to verify connection
        endpoint-result (endpoint-item updated-piece slur start-vpd)
        ;; 4. Generate visual representation if endpoint found
        visual-slur (when (first endpoint-result)
                     (format-slur-visual updated-piece slur start-vpd position-provider))]
    {:piece updated-piece
     :slur slur
     :start-vpd start-vpd
     :endpoint endpoint-result
     :visual visual-slur}))

(defn batch-slur-formatting [piece slur-definitions position-provider]
  "Process multiple slurs efficiently using existing attachment patterns."
  (reduce (fn [acc-piece [start-vpd end-vpd]]
            (let [result (create-and-format-slur acc-piece start-vpd end-vpd position-provider)]
              (:piece result)))
          piece
          slur-definitions))

;; Integration with existing attachment resolver system
(defn intuitive-slur-creation [piece]
  "Demonstrate string-based slur creation using attachment resolver."
  (-> piece
      ;; These all create slurs using the existing "slur" string resolution
      (add-attachment [:m 0 0 0 0 0 :items 0] "slur" [:m 0 0 0 0 0 :items 3])  ; Phrase slur
      (add-attachment [:m 0 0 0 0 1 :items 0] "slur" [:m 0 0 0 0 1 :items 1])  ; Short slur
      (add-attachment [:m 0 0 0 0 2 :items 0] "slur" [:m 0 0 0 0 4 :items 2]))) ; Cross-measure slur

;; Visual formatting happens lazily when visual hierarchy is accessed
(defn render-page-slurs [visual-tree piece page-boundary position-provider]
  "Render only slurs visible on current page - leverages lazy computation."
  (let [page-slurs (find-slurs-in-boundary piece page-boundary)]
    (reduce (fn [tree [slur start-vpd]]
              (if-let [visual-slur (format-slur-visual piece slur start-vpd position-provider)]
                (add-visual-element tree (slur-visual-location slur start-vpd) visual-slur)
                tree))
            visual-tree
            page-slurs)))
```

## Rationale

### Building on Established Architecture

1. **Existing Attachment System**: Slur creation leverages the proven `add-attachment` API and attachment resolver system, requiring no new musical hierarchy patterns.

2. **Proven Endpoint Resolution**: The `endpoint-item` multimethod already uses timewalker with identical temporal stream patterns for finding attachment endpoints.

3. **Consistent API Patterns**: String-based attachment creation (`"slur"`) follows established attachment resolver conventions used throughout the system.

4. **Architectural Separation**: Musical relationships (attachment system) remain completely separate from visual computation (lazy visual hierarchy), following existing design principles.

5. **Temporal Stream Processing**: Visual point collection uses the same timewalker patterns as endpoint resolution, ensuring consistency across the codebase.

6. **Lazy Visual Computation**: Visual hierarchy elements (Bézier curves) are computed only when needed, consistent with existing visual formatting approaches.

### Technical Benefits

1. **Clean Architectural Separation**: Musical hierarchy remains pure whilst visual hierarchy contains only rendering elements
2. **Lazy Visual Computation**: Bézier curves computed only when visual hierarchy is accessed  
3. **No Adaptive Search Required**: Timewalker provides optimal traversal by design
4. **Natural Boundary Handling**: VPD scope control handles complex slur spans automatically  
5. **Trait System Integration**: Robust element identification without manual type checking
6. **Temporal Guarantees**: Items arrive in proper musical time order
7. **Memory Efficiency**: Lazy evaluation processes only what's needed
8. **Visual Independence**: Musical changes don't immediately trigger visual recomputation

### What This Approach Avoids

- **New Attachment Patterns**: No need to create slur-specific attachment handling
- **Duplicate Endpoint Logic**: Visual point collection reuses existing endpoint resolution patterns
- **Custom Search Strategies**: Timewalker provides all necessary traversal capabilities
- **API Inconsistency**: Slur creation follows established attachment resolver conventions
- **Architectural Fragmentation**: Visual formatting builds on proven lazy computation patterns

### What This Approach Provides

- **Seamless Integration**: Slurs work like any other attachment with existing APIs
- **Proven Reliability**: Endpoint resolution inherits battle-tested timewalker patterns
- **Intuitive Creation**: `(add-attachment start-vpd piece "slur" end-vpd)` follows established conventions
- **Efficient Rendering**: Visual computation leverages existing lazy evaluation architecture
- **Pattern Consistency**: Same temporal stream processing across all attachment types

## Consequences

### Positive

- **Zero Architectural Risk**: Builds entirely on proven attachment system patterns
- **Immediate API Consistency**: Slurs work with existing attachment APIs without modification
- **Proven Reliability**: Endpoint resolution inherits battle-tested timewalker temporal coordination
- **Intuitive User Experience**: String-based creation (`"slur"`) follows established attachment resolver conventions
- **Performance by Design**: Lazy visual computation leverages existing visual hierarchy patterns
- **Pattern Reuse**: Same temporal stream processing works for all attachment types (ties, hairpins, glissandos)
- **Robust Error Handling**: Trait-based filtering provides existing safety guarantees
- **Memory Efficiency**: Visual curves computed only when rendering, following established lazy evaluation
- **Development Velocity**: No new patterns to learn - leverages existing knowledge

### Negative

- **None Identified**: This approach builds entirely on existing, proven architecture without introducing new complexity or risk

### Neutral

- **Hull Calculation Unchanged**: Geometric algorithms remain optimal as originally designed
- **Position Provider Required**: Coordinate calculation abstraction still needed for visual formatting
- **Bézier Generation Unchanged**: Curve mathematics remain optimal from original design
- **Learning Curve**: Developers already familiar with attachment system can immediately work with slurs

## Implementation Notes

1. **Leverage Existing Patterns**: No new attachment architecture required - slurs integrate seamlessly with existing `add-attachment` and `endpoint-item` APIs
2. **Reuse Timewalker Patterns**: Visual point collection follows identical temporal stream patterns as existing endpoint resolution
3. **Attachment Resolver Integration**: Slur string resolution (`"slur"`) already implemented in existing attachment resolver system
4. **Visual Hierarchy Integration**: Lazy computation follows existing visual formatting patterns without modification
5. **Performance Optimization**: Inherit all existing timewalker performance characteristics (early termination, boundary scoping, etc.)
6. **Testing Strategy**: Build on existing attachment system test patterns - no new testing paradigms required

## Future Considerations

1. **Universal Attachment Patterns**: All spanning elements (ties, hairpins, glissandos, ottavas) follow identical patterns established by slur implementation
2. **Enhanced Visual Debugging**: Existing attachment system debugging tools automatically work with slur endpoint resolution
3. **Performance Optimization**: Leverage existing timewalker optimizations without slur-specific modifications
4. **Parallel Processing**: Existing attachment processing patterns enable concurrent slur rendering without additional complexity
5. **Machine Learning Integration**: Temporal stream patterns provide natural data feeds for ML curve optimization
6. **Cross-Platform Consistency**: Attachment system abstractions ensure consistent slur behavior across all deployment scenarios

## Meta-Insight

This implementation demonstrates that **architectural maturity** enables **effortless feature integration**. Rather than building new systems, slurs emerge naturally from existing attachment architecture patterns.

The timewalker's temporal stream processing, combined with the proven attachment system, creates a **composable musical computing platform** where complex features like slur rendering become simple applications of established patterns.

This validates the original architectural decisions: **musical relationships** (attachment system) and **visual computation** (lazy hierarchy) separate cleanly, while **temporal coordination** (timewalker) bridges them seamlessly. Slurs don't require new architecture - they demonstrate the power of existing architecture applied consistently.
