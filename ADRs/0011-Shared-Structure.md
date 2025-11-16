# ADR: Shared Structure

## Status: Accepted (Updated 22 July 2025)

## Related ADRs

This ADR builds on the pure tree structure described in [ADR-0010 Pure Tree Structure](0010-Pure-Trees.md).

- [ADR-0014: Timewalk](0014-Timewalk.md) - Timewalking provides the traversal mechanism for shared structure processing
- [ADR-0013: Slur Formatting](0013-Slur-Formatting.md) - Demonstrates shared structure algorithms using timewalker for point collection
- [ADR-0008: VPDs](0008-VPDs.md) - VPD addressing system used for navigation in shared structure algorithms

## Context

Ooloi's representation of a musical structure – a Piece - is a pure tree. However, certain elements require references to other parts of the score, such as Slurs, Ties, Glissandos, and Dynamic Hairpins that span across the musical piece tree structure. These connecting elements sometimes require searching forward to collect intermediate elements for further processing.

Key considerations:
1. **Performance**: The solution must be efficient for musical scores of varying sizes.
2. **Temporal Coordination**: Musical processing requires proper temporal ordering - "measure N across all voices before measure N+1".
3. **Memory Efficiency**: Minimize memory allocation through transducers and lazy evaluation, especially for large orchestral scores.
4. **Flexibility**: Must accommodate different types of connecting elements with varying rules.
5. **Maintainability**: The solution should be clear and extensible for future modifications.
6. **Simplicity**: Core code contributors should be shielded from the complexities of the underlying mechanism.
7. **Composability**: Processing operations should compose efficiently without intermediate collections.

Additional context:
- Connecting elements can vary greatly in length and complexity, from simple ties and short slurs (see [slur-1.png](../img/slurs/slur-1.png)) to longer slurs within and across measures (see [slur-2.jpeg](../img/slurs/slur-2.jpeg)).
- In complex musical pieces, slurs can span across multiple staves and voices (see [slur-3.png](../img/slurs/slur-3.png)).
- Dynamic hairpins and ties can span across multiple measures (see [hairpin-1.png](../img/hairpin-1.png)), requiring efficient handling of measure boundaries.
- Extreme cases exist where slurs can span entire systems or pages (see [slur-4.png](../img/slurs/slur-4.png)), as seen in Sorabji's "Opus Clavicembalisticum IX".
- Ties connect notes of identical pitch using explicit endpoint-id specification, like other span attachments.
- Formatting slurs and dynamic hairpins requires special consideration: When these elements are encountered, all relevant musical elements under them must be collected to calculate the required presentation data.
- The timewalker system provides temporal coordination guarantees essential for musical processing.

## Decision

We will implement an efficient approach for processing and rendering musical elements in Ooloi using the **timewalker system**, which provides:

1. **Pure tree structure** for representing the musical piece.
2. **Integer ID references** for cross-tree element relationships.
3. **Temporal coordination** through the timewalker system that guarantees proper musical time ordering.
4. **Transducer-based processing** for efficient, composable operations without intermediate collections.
5. **Boundary-scoped traversal** using VPDs to limit processing to relevant sections.
6. **Specialized algorithms** that leverage timewalking for specific musical elements (slurs, ties, hairpins, etc.).

## Detailed Design

### 1. Pure Tree Structure

The Ooloi Piece is organized as a pure tree:

- Pieces contain Musicians and Layouts
- Musicians have Instruments
- Instruments have Staves
- Staves have Measures
- Measures have Voices
- Voices contain musical items (Rests, Pitches, Chords, Tuplets, Tremolandos, etc.)

### 2. Integer ID References

Elements that need to reference other parts of the score (e.g., slurs, ties) use integer IDs:

- Each Instrument maintains its own ID counter.
- IDs are generated sequentially for each new reference needed within an Instrument.
- Elements that can be referenced (e.g., Pitches, Chords) implement the TakesAttachment trait and have an `endpoint-id` field.
- Elements that reference others (e.g., Slurs) store the ID of their endpoint in an `endpoint-id` field.

### 3. Timewalker-Based Traversal

The timewalker system provides the foundation for all shared structure processing:

```clojure
;; Core traversal with temporal coordination
(timewalk piece {:boundary-vpd boundary-vpd 
                 :start-measure start-measure
                 :end-measure end-measure})

;; Returns tuples of [item vpd position] with proper temporal ordering
```

Key features:
- **Temporal Coordination**: Processes "measure N across all voices before measure N+1"
- **Boundary Scoping**: Uses VPDs to limit traversal to relevant sections
- **Early Termination**: Stops processing when criteria are met
- **Memory Efficiency**: Uses transducers to avoid intermediate collections

### 4. Unified Element Collection

The shared structure system provides a unified approach to collecting musical elements for any spanning attachment:

Architectural principles:
- **Single collection mechanism**: One system works for all spanning attachments (slurs, hairpins, ottavas, pedal markings)
- **Temporal coordination**: Leverages timewalker to ensure proper musical time ordering
- **Layout abstraction**: Coordinate transformation separated from collection logic
- **Boundary efficiency**: Processing limited to relevant instrument sections only
- **Early termination**: Stops processing when endpoints are found

### 5. Processing Pattern Unification

All spanning attachments follow the same high-level processing pattern:
1. **Endpoint resolution** - Find the attachment's target element using temporal traversal
2. **Element collection** - Gather all musical items within the attachment's span
3. **Coordinate transformation** - Convert to layout-specific positioning
4. **Specialized rendering** - Apply attachment-specific algorithms (curves, lines, etc.)

This unified pattern enables:
- **Code reuse** across all attachment types
- **Consistent temporal behavior** 
- **Efficient processing** through early termination and boundary scoping
- **Specialized implementations** - see ADR-0013 for slur formatting details

## Rationale

1. **Pure tree structure** simplifies many operations and allows for straightforward serialization and deserialization.
2. **Integer ID references** provide a clean way to represent relationships between elements across the tree without creating cycles.
3. **Timewalker system** provides temporal coordination essential for musical applications, ensuring "measure N before measure N+1" ordering.
4. **Transducer architecture** enables efficient, composable operations without intermediate collections, crucial for handling large orchestral scores.
5. **Boundary-scoped processing** using VPDs allows efficient limitation of traversal to relevant sections only.
6. **Layout abstraction** through `obtain-xy` transducer allows the same algorithms to work with different layout contexts.
7. **Unified collection pattern** through `collect-extent-items` enables code reuse across all spanning attachment types.
8. **Early termination support** prevents unnecessary processing in large scores through efficient `take` operations.

## Consequences

### Positive

- **Maintains pure tree structure**, simplifying operations and allowing straightforward serialization/deserialization
- **Temporal coordination guarantee** ensures proper musical time ordering for all applications
- **Memory efficient processing** through transducers avoids intermediate collections in large orchestral scores  
- **Unified traversal system** - single timewalker serves MIDI generation, layout, attachment resolution, and analysis
- **Boundary-scoped efficiency** - VPD addressing limits processing to relevant sections only
- **Code reuse** - `collect-extent-items` works for all spanning attachments (slurs, hairpins, ottavas, pedal markings)
- **Early termination support** prevents unnecessary processing through efficient `take` operations
- **Layout flexibility** - same algorithms work with different layout contexts through `obtain-xy` abstraction
- **Composable operations** - transducer architecture enables flexible operation composition

### Negative

- **Learning curve** - developers must understand both temporal coordination and transducer principles  
- **ID management complexity** - requires careful management of integer IDs to maintain consistency
- **Specialized knowledge** - requires understanding of timewalker system and VPD addressing
- **Debugging complexity** - transducer pipelines can be harder to debug than explicit loops

### Neutral

- **Musical domain specialization** - optimized for musical applications rather than general tree traversal
- **VPD dependency** - requires understanding of Ooloi's addressing system
- **Functional paradigm** - benefits developers familiar with functional programming concepts
- **Performance tuning** - requires careful optimization based on real-world usage patterns

## Implementation Notes

1. **Pure tree structure** - Implemented using Clojure's immutable data structures
2. **Integer ID system** - Each Instrument maintains ID counter for attachment endpoint references  
3. **Timewalker system** - Provides temporal coordination and boundary-scoped traversal (see ADR-0014)
4. **Transducer architecture** - Enables efficient, composable operations without intermediate collections
5. **General extent collection** - `collect-extent-items` function works for all spanning attachments
6. **Layout abstraction** - `obtain-xy` transducer handles coordinate transformation for specific layouts
7. **Early termination** - Use `take` operations to prevent unnecessary processing in large scores
8. **Comprehensive testing** - Unit tests cover temporal coordination, boundary scoping, and attachment resolution
9. **Performance monitoring** - Profile timewalker operations to optimize boundary selection and filtering
10. **ID management procedures** - Safe copying and deletion of measures with ID regeneration and reference updates

## Future Considerations

1. **Performance optimization** for extremely large orchestral scores with specialized boundary strategies
2. **Parallel processing** potential for independent musical sections using core.async or similar
3. **Advanced attachment algorithms** - machine learning for enhanced slur shapes and spacing
4. **Plugin architecture** for extending shared structure processing to custom attachment types  
5. **Real-time collaboration** - leveraging pure tree structure and temporal coordination for multi-user editing
6. **Audio integration** - MIDI playback using timewalker temporal coordination
7. **Visual debugging tools** - inspect timewalker traversal and attachment resolution
8. **High-density optimization** - specialized algorithms for scores with many overlapping attachments
9. **Intelligent caching** - memoization strategies for frequently accessed attachment endpoints
10. **Profiling integration** - built-in tools to identify performance bottlenecks in real-world scores

## Example Applications

The shared structure system handles a wide range of spanning musical elements:

- **Simple connections**: Ties between identical pitches, basic slurs over a few notes within measures
- **Cross-measure elements**: Slurs and dynamic hairpins (crescendo/diminuendo) spanning multiple measures and voices
- **Multi-staff elements**: Slurs connecting notes across different staves in piano music
- **System-wide elements**: Long slurs spanning entire musical systems in complex orchestral works
- **Other spanning attachments**: Glissandos, ottava markings, pedal markings, and other elements that extend across musical time

For a detailed implementation example, see [ADR-0013: Slur Formatting](0013-Slur-Formatting.md), which demonstrates how the shared structure system is applied to slur rendering using convex hull algorithms and dual Bézier curves.
