# ADR-0014: Timewalk Temporal Coordination System

## Status: Implemented (Updated 20 July 2025)

## Context

Ooloi requires sophisticated traversal capabilities for its hierarchical musical structures to support critical applications including MIDI playback, visual layout, attachment resolution, and collaborative editing. The system faces several key challenges:

1. **Temporal Coordination**: Musical pieces require traversal that respects temporal ordering - all events at measure N must be processed before any events at measure N+1 across all voices and instruments.

2. **Polyphonic Complexity**: Modern musical scores contain multiple simultaneous voices that must be synchronized during traversal while maintaining their individual structural integrity.

3. **Scope-Limited Operations**: Different operations require traversal of specific subsets of the piece (single instrument, staff, voice, or measure range) without processing irrelevant portions.

4. **Performance Requirements**: Large orchestral scores with hundreds of thousands of musical elements demand efficient traversal that can terminate early and filter elements without full tree processing.

5. **Multiple Application Domains**: The same traversal mechanism must support fundamentally different operations including MIDI event generation, visual layout calculations, attachment endpoint resolution, and collaborative change propagation.

6. **Memory Constraints**: Professional scores can contain millions of elements, requiring memory-efficient processing that avoids intermediate collections.

7. **Computational-Musical Alignment**: Traditional music software forces musical concepts into object hierarchies, creating impedance mismatch between musical thinking and computational thinking. Ooloi requires an abstraction where musical concepts **are** the computational primitives.

## Decision

We implement a comprehensive timewalk system based on temporal coordination principles, using transducer-based traversal with boundary-VPD scope limiting. The system provides "measure N across all voices before measure N+1" traversal guarantees while supporting efficient filtering, early termination, and scope limitation.

## Paradigm Shift: Musical Time as Computational Foundation

The timewalk system represents a fundamental shift from **object-oriented musical manipulation** to **temporal stream processing**. Rather than treating musical time as a byproduct of object relationships, Ooloi makes temporal coordination the foundational abstraction for all musical computation.

This transforms musical software development:

- **Musical relationships become stream transformations** rather than pointer manipulations
- **Complex operations emerge** from composition of simple temporal filters  
- **Mathematical guarantees** apply to musical computation through functional composition
- **Problem domains unify** under temporal stream processing (MIDI, layout, analysis, attachments)
- **Musical intent maps directly** to computational expressions without translation overhead
- **Domain expertise becomes executable code** rather than being lost in abstraction layers

### From Imperative to Temporal-Functional

**Traditional Approach** (imperative object manipulation):
```clojure
;; Navigate object graph, modify state
(doseq [musician (get-musicians piece)]
  (doseq [instrument (get-instruments musician)]
    (when (violin? instrument)
      (transpose-instrument! instrument 4))))
```

**Ooloi Approach** (temporal stream processing):
```clojure
;; Define transformations, compose operations
(into []
      (comp (timewalk {:boundary-vpd []})
            (filter #(violin-instrument? (item %)))
            (filter pitch?)
            (map #(transpose-pitch (item %) 4)))
      [piece])
```

The temporal approach eliminates impedance mismatch—musical thinking and computational thinking become isomorphic.

## Dual-Arity API Pattern Decision

The `timewalk` function follows the established Clojure pattern used by core functions like `map`, `filter`, and `take`, providing two distinct arities:

```clojure
;; 2-arity: Returns lazy sequence directly
(timewalk piece {:boundary-vpd [:musicians 0 :instruments 0]})

;; 1-arity: Returns transducer for composition
(sequence (comp (timewalk {:boundary-vpd staff-boundary})
                (filter pitch?)
                (take 10))
          [piece])
```

This dual-arity design provides significant architectural benefits:

**1. Consistency with Clojure Ecosystem**
Following the same pattern as `map`, `filter`, `remove`, and other core functions creates intuitive APIs that Clojure developers understand immediately. This reduces cognitive load and learning curve.

**2. Flexibility in Usage Patterns**
- **Direct lazy sequences** for simple exploratory analysis and REPL development
- **Transducer composition** for complex processing pipelines with multiple transformation steps
- **Performance optimization** through transducer composition without intermediate collections

**3. Composability with Standard Library**
The transducer form enables seamless composition with all of Clojure's transducer-aware functions:
```clojure
;; Compose with any transducer-aware context
(into [] (comp (timewalk {:boundary-vpd voice-vpd}) (filter pitch?)) [piece])
(sequence (timewalk options) pieces)  ; Process multiple pieces
(transduce (timewalk options) conj [] [piece])  ; Direct reduction
```

**4. Future-Proof Architecture**
The transducer form naturally supports emerging Clojure contexts like core.async channels, allowing musical processing in concurrent or streaming scenarios without API changes.

This design decision ensures that `timewalk` integrates naturally with both simple and complex musical processing workflows while maintaining the performance characteristics required for large orchestral scores.

## Transducer Architecture Decision

### Why Clojure Transducers?

Clojure transducers provide the optimal solution for Ooloi's timewalk due to several critical advantages:

**1. Composable Processing Pipelines**
Transducers enable elegant composition of operations without intermediate collections:
```clojure
(comp (filter pitch?)
      (map pitch->midi-note)
      (take-while #(< (:time %) end-time)))
```
This composes filtering, transformation, and early termination into a single efficient pipeline.

**2. Memory Efficiency**
Unlike sequence operations that create intermediate collections, transducers process elements one at a time, crucial for large orchestral scores:
- No intermediate vectors for filtered results
- Constant memory usage regardless of piece size
- Streaming-compatible for real-time applications

**3. Early Termination Support**
Transducers naturally support early termination through `take`, `take-while`, etc., essential for:
- MIDI playback of score sections
- Layout calculations that stop at page boundaries
- Interactive editing that processes only visible measures

**4. Process-Agnostic Design**
The same transducer works with different collection types and processing contexts:
- Lazy sequences for exploratory analysis
- Reduced operations for aggregation
- Channel processing for concurrent applications

### Alternative Approaches Considered and Rejected

**1. Sequence-Based Processing (map, filter, etc.)**
```clojure
;; Rejected approach
(->> (get-all-items piece)
     (filter pitch?)
     (map pitch->midi-note)
     (take 1000))
```
**Why Rejected:**
- Creates intermediate collections at each step
- Memory usage grows linearly with piece size
- No early termination until final step
- Poor performance for large orchestral scores

**2. Recursive Tree Walking with Higher-Order Functions**
```clojure
;; Rejected approach - demonstrates temporal coordination problem
(defn walk-tree [piece filter-fn transform-fn]
  (for [musician (get-musicians piece)      ; Process ALL of musician 0
        instrument (get-instruments musician) ; then ALL of musician 1, etc.
        staff (get-staves instrument)
        voice (get-voices staff)
        measure (get-measures voice)        ; Gets ALL measures of voice 0
        item (get-items measure)            ; before ANY measures of voice 1
        :when (filter-fn item)]
    (transform-fn item)))
```
**Why Rejected:**
- **Breaks temporal coordination** - processes all measures of voice 1 before any of voice 2
- Difficult to compose multiple operations
- No built-in early termination
- Hard to unit test individual transformation steps
- Returns results in voice-by-voice order rather than temporal order

**3. Visitor Pattern Implementation**
```clojure
;; Rejected approach
(defprotocol ItemVisitor
  (visit-pitch [this pitch])
  (visit-chord [this chord])
  (visit-rest [this rest]))
```
**Why Rejected:**
- Object-oriented approach conflicts with Clojure's functional philosophy
- Poor composability - each visitor is monolithic
- Difficult to implement early termination
- Complex to extend for new musical element types

**4. Core.async Channels**
```clojure
;; Rejected approach
(defn walk-piece-async [piece]
  (let [ch (chan 1000)]
    (go (doseq [item (get-all-items piece)]
          (>! ch item)))
    ch))
```
**Why Rejected:**
- Unnecessary complexity for synchronous operations
- Channel buffering decisions complicate memory management
- Temporal coordination requires complex channel coordination
- Overkill for most timewalk use cases

**5. Custom Iterator/Generator Protocol**
```clojure
;; Rejected approach
(defprotocol PieceIterator
  (next-item [this])
  (has-next? [this]))
```
**Why Rejected:**
- Stateful design conflicts with functional programming principles
- Poor composability with other Clojure operations
- Complex to implement temporal coordination correctly
- Difficult to test and reason about state transitions

### Single Unified Transducer Architecture

**Critical Design Decision: One Transducer for All Applications**

Rather than creating separate traversal systems for MIDI generation, visual layout, attachment resolution, and analysis, Ooloi implements a **single unified timewalk transducer** that serves all these domains. This architectural choice provides several crucial advantages:

**1. Temporal Consistency Across All Operations**
Every application automatically inherits the same temporal coordination guarantee:
```clojure
;; MIDI, layout, and attachment resolution all use the same temporal ordering
(sequence (comp (timewalk boundary-vpd) midi-transducer) [piece])    ; Temporal order preserved
(sequence (comp (timewalk boundary-vpd) layout-transducer) [piece])  ; Same temporal order
(sequence (comp (timewalk boundary-vpd) attach-transducer) [piece])  ; Same temporal order
```

**2. Reduced Complexity and Maintenance**
- Single algorithm to test, debug, and optimize
- No risk of inconsistent temporal behavior between different systems
- Unified API for all musical traversal needs
- Single point of optimization benefits all applications

**3. Composable Cross-Domain Operations**
Applications can combine concerns within a single traversal:
```clojure
;; Generate MIDI while calculating layout metrics
(sequence (comp (timewalk {:boundary-vpd nil})
                (map (juxt item->midi-event calculate-width))
                (filter first)  ; Only items that generate MIDI
                (map (fn [[midi width]] {:midi midi :width width})))
          [piece])
```

**4. Guaranteed Behavioral Consistency**
- MIDI playback and visual display show identical temporal ordering
- Attachment resolution finds the same endpoints regardless of context
- Analysis results are consistent with user's visual experience
- Eliminates entire classes of synchronization bugs

**Why Not Separate Systems?**

Alternative approaches with domain-specific traversals would create:
- **Temporal Drift**: MIDI and layout could process measures in different orders
- **Maintenance Burden**: Multiple algorithms requiring separate optimization
- **Inconsistent Behavior**: Users experiencing different temporal behavior in different contexts
- **Duplicate Code**: Similar traversal logic repeated across domains
- **Testing Complexity**: Need to verify temporal consistency across multiple systems

### Transducer-Specific Advantages for Musical Applications

**1. Temporal Coordination Preservation**
Transducers maintain the temporal ordering guarantee while allowing arbitrary transformations:
```clojure
(sequence (comp (timewalk {:boundary-vpd [:musicians 0]})
                (filter chord?)
                (map extract-harmony))
          [piece])
;; Still processes measure 1 chords before measure 2 chords
```

**2. MIDI Streaming Compatibility**
Transducers naturally support real-time MIDI generation:
```clojure
(sequence (comp (timewalk {:boundary-vpd nil})
                (map item->midi-event)
                (remove nil?)
                (take-while #(< (:time %) playback-end)))
          [piece])
```

**3. Layout Calculation Optimization**
Visual layout can terminate early when page capacity is reached:
```clojure
(sequence (comp (timewalk {:boundary-vpd nil})
                (map calculate-item-width)
                (scan +)  ; Running width total
                (take-while #(< % page-width)))
          [piece])
```

**4. Attachment Resolution Efficiency**
Find attachment endpoints without processing entire piece:
```clojure
(first
  (sequence (comp (timewalk {:boundary-vpd attachment-scope})
                  (filter #(= (:end-id %) target-id))
                  (take 1))  ; Stop after finding endpoint
            [piece]))
```

## Detailed Design

### 1. Core Algorithm: Temporal Breadth-First Traversal

The timewalk implements temporal coordination through a two-phase process:

**Phase 1: Parallel Measure Grouping**
```clojure
(find-parallel-measures piece boundary-vpd)
```
- Discovers all measures at each temporal position across the specified scope
- Groups measures by their measure number (1, 2, 3, etc.)
- Returns sequence of measure groups in temporal order

**Phase 2: Temporal Item Processing**
```clojure
(walk-items grouped-measures transducer)
```
- Processes each measure group completely before advancing to the next
- Applies transducer to all items within each measure group
- Maintains temporal ordering guarantee throughout processing

### 2. Boundary-VPD Scope Limiting

The system uses Vector Path Descriptors (VPDs) to define traversal boundaries:

```clojure
;; Full piece traversal
{:boundary-vpd nil}

;; Single musician traversal  
{:boundary-vpd [:musicians 2]}

;; Single instrument traversal
{:boundary-vpd [:musicians 0 :instruments 1]}

;; Single staff traversal
{:boundary-vpd [:musicians 0 :instruments 1 :staves 0]}
```

Boundary-VPDs provide precise control over traversal scope while maintaining the temporal coordination guarantee within the specified boundary.

### 3. Transducer Integration Examples

```clojure
;; MIDI generation with early termination
(sequence (comp (timewalk {:boundary-vpd [:musicians 0]})
                (filter rhythmic-item?)
                (map item->midi-event)
                (remove nil?)
                (take-while #(< (:time %) end-time)))
          [piece])

;; Harmonic analysis with filtering
(into #{}
      (comp (timewalk {:boundary-vpd nil})
            (filter chord?)
            (map extract-harmony))
      [piece])

;; Layout calculation with running totals
(sequence (comp (timewalk {:boundary-vpd [:musicians 0 :instruments 0]})
                (map calculate-width)
                (scan +)
                (take-while #(< % page-width)))
          [piece])
```

### 4. Deep Traversal Support

The walker performs deep traversal within each measure, finding all nested items including:
- Direct measure items (rests, pitches, chords)
- Tuplet-contained items (items within tuplets)
- Tremolando-contained items (items within tremolandos)
- Arbitrarily nested structures

This ensures complete temporal coordination across all musical elements regardless of their nesting level.

## Key Applications

### Cross-Domain Pattern Unification

The timewalk demonstrates that diverse musical operations share identical computational patterns:

```clojure
;; Attachment endpoints: temporal stream → filter by ID → first match
(first (sequence (comp (timewalk scope) (filter has-endpoint-id?) (take 1)) [piece]))

;; Visual formatting: temporal stream → filter by span → map coordinates  
(sequence (comp (timewalk scope) (filter within-slur?) (map pitch->coordinates)) [piece])

;; MIDI generation: temporal stream → map to events → maintain time order
(sequence (comp (timewalk scope) (map item->midi-event) (remove nil?)) [piece])

;; Harmonic analysis: temporal stream → group by simultaneity → analyze
(sequence (comp (timewalk scope) (group-by measure-position) (map analyze-harmony)) [piece])

;; Layout calculation: temporal stream → map to widths → accumulate
(sequence (comp (timewalk scope) (map calculate-width) (scan +)) [piece])
```

This pattern unification proves that **musical time is the correct organizing principle** for musical computation. Complex musical operations reduce to temporal stream operations with different predicates and transformations.

### 1. MIDI Playback Generation

The temporal coordination guarantee directly solves MIDI's core requirement for time-ordered event streams:

- **Temporal Order**: All MIDI events at measure N emit before any at measure N+1
- **Polyphonic Support**: Natural handling of multiple simultaneous voices
- **Partial Playback**: Boundary-VPD enables instrument/staff-specific playback
- **Real-time Streaming**: Transducer composition supports efficient event generation

### 2. Visual Layout Calculation

The walker provides the foundation for professional music engraving:

- **System Layout**: Temporal coordination ensures measures align properly across staves
- **Page Breaking**: Early termination when page capacity is reached
- **Scope-Limited Layout**: Boundary-VPD enables partial re-layout after edits
- **Performance Optimization**: Avoid processing off-screen content

### 3. Attachment Resolution

Musical attachments (slurs, hairpins, ties) require endpoint resolution across temporal boundaries:

- **Cross-Measure Attachments**: Temporal traversal finds endpoints in later measures
- **Scope Awareness**: Boundary-VPD prevents attachment resolution outside relevant scope
- **Efficient Processing**: Early termination when all endpoints are found

### 4. Collaborative Editing

Multi-user editing requires precise change propagation:

- **Temporal Change Ordering**: Ensure changes apply in correct temporal sequence
- **Scope-Limited Updates**: Boundary-VPD enables efficient partial updates
- **Conflict Resolution**: Temporal coordination assists in resolving editing conflicts

### 5. Analysis and Export

Musical analysis tools require comprehensive piece traversal:

- **Harmonic Analysis**: Temporal coordination for chord progression analysis
- **Statistical Analysis**: Complete piece traversal with filtering
- **Format Export**: MusicXML, MIDI, and other format generation

## Rationale

1. **Temporal Coordination**: The "measure N before measure N+1" guarantee is fundamental to musical applications and distinguishes this system from generic tree traversal.

2. **Transducer Architecture**: Leverages Clojure's composable, efficient data transformation model while maintaining the temporal guarantee.

3. **Memory Efficiency**: Transducers avoid intermediate collections, crucial for large orchestral scores.

4. **Composability**: Different operations compose naturally without performance penalties.

5. **Boundary-VPD Integration**: Reuses Ooloi's existing VPD addressing system for intuitive scope specification.

6. **Performance Focus**: Early termination and filtering capabilities handle large orchestral scores efficiently.

7. **Multi-Application Support**: Single traversal mechanism serves diverse use cases from MIDI to layout to analysis.

8. **Mathematical Foundation**: Functional composition provides mathematical guarantees for musical computation.

## Consequences

### Positive

- **Unified Traversal Model**: Single system serves all temporal coordination needs
- **Memory Efficient**: Constant memory usage regardless of piece size
- **Performance Optimized**: Handles large scores with early termination and filtering
- **Composable Operations**: Transducer architecture enables flexible operation composition
- **Precise Scope Control**: Boundary-VPD provides fine-grained traversal control
- **MIDI-Ready Architecture**: Directly enables professional-quality MIDI playback
- **Layout Foundation**: Provides basis for sophisticated visual layout algorithms
- **Streaming Compatible**: Supports real-time applications without modification
- **Mathematical Guarantees**: Functional composition provides mathematical properties for musical computation
- **Pattern Unification**: Diverse musical operations reduce to identical computational patterns

### Negative

- **Learning Curve**: Developers must understand both temporal coordination and transducer principles
- **Complexity**: More sophisticated than simple tree traversal or sequence operations
- **Debugging**: Transducer pipelines can be harder to debug than explicit loops

### Neutral

- **Specialized Design**: Optimized for musical applications rather than general tree traversal
- **VPD Dependency**: Requires understanding of Ooloi's VPD addressing system
- **Functional Paradigm**: Benefits developers familiar with functional programming concepts

## Architectural Validation

The timewalk's success across multiple domains validates the core architectural insight: **when you find the right abstraction for a domain, complex problems become simple applications of fundamental patterns**.

This creates a **composable musical computing platform** where:

- **Attachment resolution** and **visual formatting** use identical temporal stream patterns
- **Complex features emerge naturally** from existing abstractions without new architecture
- **Musical intent maps directly** to computational expressions
- **Domain expertise becomes executable code** rather than being lost in translation
- **Mathematical rigor** applies to musical computation through functional composition

The fact that slur rendering, MIDI generation, attachment endpoint resolution, and harmonic analysis all reduce to temporal stream operations with different predicates demonstrates that musical time is the **natural computational structure** of the musical domain.

This represents **architecture as compression** - reducing the complexity of an entire problem domain to a small set of composable primitives. The timewalker demonstrates that Ooloi has discovered **musical time's computational essence**, making musical software development fundamentally more powerful and elegant.

## Related ADRs

- [ADR-0010: Pure Trees](0010-Pure-Trees.md) - Provides the foundational tree structure that timewalk traverses
- [ADR-0008: VPDs](0008-VPDs.md) - Defines the Vector Path Descriptor system used for boundary specification
- [ADR-0012: Persisting Pieces](0012-Persisting-Pieces.md) - Establishes the ID reference system that timewalk respects during traversal
- [ADR-0011: Shared Structure](0011-Shared-Structure.md) - Shared structure algorithms use timewalk for element collection and processing
- [ADR-0013: Slur Formatting](0013-Slur-Formatting.md) - Slur formatting demonstrates timewalk application for point collection

## Documentation

For practical usage examples and comprehensive guidance, see [TIMEWALKING_GUIDE.md](../guides/TIMEWALKING_GUIDE.md).
