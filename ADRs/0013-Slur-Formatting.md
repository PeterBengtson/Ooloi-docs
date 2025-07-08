# ADR: Slur Formatting and Convex Hull Calculation

## Status: Accepted

## Context

For the basic considerations and graphical examples, please refer to "ADR: Pure Tree Structure with Integer ID References".

Ooloi needs to efficiently handle the creation, formatting, and rendering of slurs in musical notation. This involves:
1. Creating and managing slurs using integer ID references
2. Efficiently collecting points for slur rendering, considering complex nested musical structures
3. Accurately representing the true positions of pitches in the musical score
4. Generating aesthetically pleasing slur shapes using hull calculations and Bézier curves

Key considerations:
- Performance and memory efficiency for large musical pieces
- Compatibility with Ooloi's pure tree structure architecture
- Handling of integer ID references
- Efficient point collection and processing, including nested musical elements
- Accurate representation of pitch positions in complex musical contexts
- Generating natural-looking slur curves that respect the left-to-right order of notes
- Utilizing available context, particularly access to start and end pitches via their IDs
- Adapting the slur shape based on its placement above or below the music

## Decision

We will implement slur handling and curve generation using:
1. Integer ID references for slurs
2. Transducers for efficient point collection, with recursive extraction for nested structures
3. A position provider abstraction for accurate pitch positioning
4. Adaptive search strategy for traversing the musical structure
5. Upper or lower hull calculation for initial slur shape determination, based on slur placement
6. Bézier curve generation for final slur rendering

## Detailed Design

### 1. Slur Data Structure and Creation

```clojure
(ns ooloi.slur)

(defrecord Slur [start-id end-id placement])

(defn create-slur [piece start-pitch-id end-pitch-id placement]
  (let [slur (->Slur start-pitch-id end-pitch-id placement)
        slur-id (generate-id piece [:m (instrument-index start-pitch-id)])]
    (-> piece
        (update-in [:attachments start-pitch-id] (fnil conj []) slur-id)
        (update-in [:slur-ends end-pitch-id] (fnil conj #{}) slur-id)
        (assoc-in [:slurs slur-id] slur))))
```

### 2. Real Position Abstraction

```clojure
(defprotocol PositionProvider
  (get-x-position [this pitch])
  (get-y-position [this pitch]))

(defrecord ScorePositionProvider []
  PositionProvider
  (get-x-position [this pitch]
    ;; Implementation to calculate real X position
    )
  (get-y-position [this pitch]
    ;; Implementation to calculate real Y position
    ))

(def position-provider (ScorePositionProvider.))
```

### 3. Point Collection

```clojure
(defn extract-pitches
  "Recursively extracts pitch IDs from musical elements, maintaining order and structure."
  [elem]
  (cond
    (instance? Pitch elem) [(:id elem)]
    (instance? Chord elem) (map :id (:pitches elem))
    (instance? Tuplet elem) (mapcat extract-pitches (:contents elem))
    (instance? Tremolando elem) (mapcat extract-pitches (:pitches elem))
    :else []))

(defn collect-points-xf [end-pitch-id position-provider piece]
  (comp
    (take-while #(not= % end-pitch-id))
    (mapcat extract-pitches)
    (map (fn [pitch-id]
           (let [pitch (get-in piece [:pitches pitch-id])]
             {:x (get-x-position position-provider pitch)
              :y (get-y-position position-provider pitch)
              :highest (when (instance? Chord (:parent pitch))
                         (= pitch-id (apply max-key #(get-y-position position-provider (get-in piece [:pitches %])) 
                                            (map :id (:pitches (:parent pitch))))))
              :lowest (when (instance? Chord (:parent pitch))
                        (= pitch-id (apply min-key #(get-y-position position-provider (get-in piece [:pitches %])) 
                                           (map :id (:pitches (:parent pitch))))))}))))

(defn collect-points [piece start-pitch-id end-pitch-id position-provider]
  (sequence (collect-points-xf end-pitch-id position-provider piece)
            (search/adaptive-search piece start-pitch-id)))
```

### 4. Hull Calculation

```clojure
(defn cross-product [[x1 y1] [x2 y2] [x3 y3]]
  (- (* (- x2 x1) (- y3 y1))
     (* (- y2 y1) (- x3 x1))))

(defn calculate-hull [points above?]
  (let [comparator (if above? <= >=)]
    (reduce (fn [hull point]
              (loop [h hull]
                (if (and (>= (count h) 2)
                         (comparator (cross-product (peek (pop h)) (peek h) point) 0))
                  (recur (pop h))
                  (conj h point))))
            [] points)))
```

### 5. Bézier Curve Generation

```clojure
(defn calculate-control-points [hull above?]
  (let [start (first hull)
        end (last hull)
        mid-x (/ (+ (:x start) (:x end)) 2)
        extreme-point (apply (if above? max-key min-key) :y hull)
        offset (if above? 10 -10)
        control1 {:x (+ (:x start) (* 0.25 (- (:x end) (:x start))))
                  :y (+ (:y extreme-point) offset)}
        control2 {:x (+ (:x start) (* 0.75 (- (:x end) (:x start))))
                  :y (+ (:y extreme-point) offset)}]
    [start control1 control2 end]))

(defn bezier-point [t [p0 p1 p2 p3]]
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
  (map #(bezier-point (/ % steps) control-points) (range (inc steps))))
```

### 6. Slur Formatting

```clojure
(defn format-slur [piece slur-id position-provider]
  (let [slur (get-in piece [:slurs slur-id])
        points (collect-points piece (:start-id slur) (:end-id slur) position-provider)
        above? (= (:placement slur) :above)
        hull (calculate-hull points above?)
        control-points (calculate-control-points hull above?)
        curve-points (generate-bezier-curve control-points 100)]
    (assoc-in piece [:slurs slur-id :curve-points] curve-points)))
```

### 7. Combined Slur Creation and Formatting

```clojure
(defn add-and-format-slur [piece start-pitch-id end-pitch-id placement position-provider]
  (let [piece-with-slur (create-slur piece start-pitch-id end-pitch-id placement)
        slur-id (first (filter #(instance? Slur (get-in piece-with-slur [:slurs %])) 
                               (get-in piece-with-slur [:attachments start-pitch-id])))]
    (format-slur piece-with-slur slur-id position-provider)))
```

## Rationale

1. The `Slur` record uses integer IDs to reference start and end pitches, maintaining the pure tree structure.
2. The `placement` field in the Slur record allows for determining whether to use upper or lower hull calculation.
3. Transducers in `collect-points` provide efficient, lazy processing of points.
4. The hull calculation gives a good initial shape for the slur while preserving the left-to-right order of notes.
5. Bézier curve generation creates a smooth, aesthetically pleasing final curve.
6. The adaptive search strategy (assumed in `search/adaptive-search`) allows for efficient traversal of the musical structure.
7. The pitch extraction process handles complex nested musical structures while maintaining efficiency through the use of transducers.
8. The position provider abstraction allows for accurate representation of pitch positions in complex musical contexts.

### Key Considerations for Hull Calculation

1. **Preservation of Note Order**: We calculate either the upper or lower hull, preserving the left-to-right order of notes, crucial for creating a natural-looking slur.
2. **Efficiency**: Focusing on a single hull reduces computational complexity, beneficial for large musical pieces with many slurs.
3. **No Sorting Required**: Points are already in their natural left-to-right order as they appear in the score.
4. **Suitability for Slur Shape**: The hull provides an excellent basis for generating a slur shape, naturally following the contour of the notes under the slur.
5. **Adaptability**: The hull calculation adapts to whether the slur is placed above or below the notes.

### Complex Musical Structures and Pitch Extraction

The `extract-pitches` function recursively traverses nested musical structures to extract all relevant pitch IDs, maintaining the left-to-right order and handling various musical elements like chords, tuplets, and tremolandos.

### Real Position Abstraction

The `PositionProvider` protocol and `ScorePositionProvider` implementation allow for accurate calculation of pitch positions, considering various factors that affect positioning in a real musical score.

## Consequences

### Positive

- Efficient handling of slurs in large musical pieces
- Aesthetically pleasing slur shapes that respect the natural order of notes
- Consistent with Ooloi's pure tree structure architecture
- Lazy evaluation and efficient memory usage
- Flexible handling of complex nested musical structures
- Accurate representation of pitch positions in complex musical contexts
- Adapts to slur placement above or below the music

### Negative

- Potential performance impact for pieces with many slurs, though mitigated by the optimized hull calculation
- Increased complexity in pitch extraction logic, though localized to the collection phase
- Additional complexity introduced by the position provider abstraction

### Neutral

- Developers need to understand the concept of hull calculation and its application to slur generation
- Requirement for careful management of nested musical structures in pitch extraction
- Need for comprehensive understanding of musical notation and layout principles for accurate position provider implementation

## Implementation Notes

1. Implement the `search/adaptive-search` function to efficiently traverse the musical structure.
2. Thoroughly test the hull calculation with various musical scenarios, including edge cases.
3. Optimize the Bézier curve generation for performance, considering the number of steps needed for smooth rendering.
4. Add slur midpoint thickness: the slur shape really consists of two bezier curves and a fill.
5. Implement comprehensive error handling, especially for edge cases in complex musical structures.
6. Develop a suite of unit and integration tests covering various slur scenarios and nested structures.
7. Optimize the `ScorePositionProvider` implementation for efficient calculation of pitch positions.
8. Consider caching strategies for frequently calculated positions or curves to improve performance.
9. Implement efficient lookup mechanisms for resolving ID references during slur processing.
10. Ensure that serialization and deserialization processes correctly handle the integer ID references of slurs.

## Future Considerations

1. Explore optimizations for scenarios with a high density of slurs.
2. Investigate adaptive curve generation based on musical context (e.g., different curves for different musical styles or notations).
3. Consider extending this approach to other curved notations (e.g., ties, phrase markings).
4. Implement user controls for fine-tuning slur shapes if needed.
5. Explore the possibility of using machine learning techniques to refine slur shapes based on expert human-drawn slurs.
6. Investigate the potential for parallelizing certain aspects of the slur generation process for very large scores.
7. Develop a visual debugging tool for inspecting the calculated pitch positions and resulting slur shapes.
8. Consider implementing a system for handling collisions between slurs and other notation elements.
9. Investigate optimizations for traversing and processing structures with ID references in very large musical pieces.
