# ADR-0012: Persisting the Pure Tree Structure with Integer ID References

## Table of Contents
- [Status](#status)
- [Context](#context)
- [Decision](#decision)
- [Detailed Design](#detailed-design)
  - [Pure Tree Structure](#1-pure-tree-structure)
  - [Integer ID Reference Management](#2-integer-id-reference-management)
  - [Serialization and Deserialization](#3-serialization-and-deserialization)
- [Rationale](#rationale)
- [Consequences](#consequences)
- [Related Decisions](#related-decisions)
- [Implementation Notes](#implementation-notes)
- [Future Considerations](#future-considerations)

## Status
Accepted

## Context

Ooloi deals with complex musical structures that include elements referencing other elements. The system needs to efficiently handle these structures during runtime operations, serialization, and deserialization. Key considerations include:

1. Musical pieces can be extremely large, potentially containing hundreds of thousands of musical elements.
2. Some musical elements (e.g., slurs, dynamics) may reference other elements that appear later in the piece.
3. Efficient updates and consistency maintenance are crucial for performance and correctness.
4. The system needs to handle serialization and deserialization of these complex structures.
5. The solution should align with Clojure's philosophy of immutable data structures while addressing the practical needs of the domain.
6. The persistence mechanism should support handling large files efficiently.
7. The solution should be compatible with Nippy, our chosen serialization library.
8. The system should be able to handle future extensions and modifications to data structures.

## Decision

We will implement a comprehensive approach to managing the pure tree structure with integer ID references and persistence in Ooloi, using:

1. A pure tree structure for representing the musical piece.
2. Integer IDs for referencing elements across the tree.
3. A system for managing relationships between musical elements using these integer IDs.

This approach will maintain the benefits of Clojure's immutable data structures while pragmatically handling references between elements.

## Detailed Design

### 1. Pure Tree Structure

The Ooloi Piece is organized as a pure tree:

- Pieces contain Musicians and Layouts
- Musicians have Instruments
- Instruments have Staves
- Staves have Measures
- Measures have Voices
- Voices contain musical items (Rests, Pitches, Chords, Tuplets, Tremolandos, etc.)

### 2. Integer ID Reference Management

Elements that can be referenced implement the TakesAttachment trait:
- These elements have an `end-id` field to store their unique ID.
- Elements that reference others store the ID of their endpoint in an `end` field.
- Each Instrument maintains its own ID counter for generating unique IDs within its scope.

### 3. Serialization and Deserialization

- Serialization: The entire piece structure, including all integer ID references, is serialized directly using Nippy.
- Deserialization: The piece structure is deserialized directly using Nippy, with all integer ID references intact.
- **Hash-Consing Optimization**: Registry-based file size optimization leverages selective hash-consing to achieve substantial performance gains during serialization/deserialization while preserving all ID references and structural integrity (see [ADR-0029: Selective Hash-Consing](0029-Global-Hash-Consing.md)).

## Rationale

1. The pure tree structure simplifies many operations and allows for straightforward serialization and deserialization.
2. Integer ID references provide a clean way to represent relationships between elements across the tree.
3. This approach maintains the ability to represent complex musical relationships while keeping the overall structure simple.

## Consequences

### Positive

- Efficient handling of referenced structures in large musical pieces.
- Preserves exact structure of pieces, including relationships between elements, across serialization cycles.
- Compatible with Nippy serialization.
- Maintains consistency of references across updates.
- Simplifies the overall data structure by maintaining a pure tree.
- Allows for straightforward serialization and deserialization.
- **Substantial performance optimization**: Hash-consing provides 69% file size reduction and 4x serialization/deserialization speed improvement for repetitive musical structures while preserving all ID references.

### Negative

- Requires careful management of integer IDs to maintain consistency.
- Special handling is needed when copying or deleting measures to maintain ID integrity.

## Related Decisions

- [ADR-0010: Pure Trees](0010-Pure-Trees.md) - Tree structure foundation that this persistence model builds upon
- [ADR-0007: Nippy](0007-Nippy.md) - Serialization technology chosen for persistence implementation
- [ADR-0008: VPDs](0008-VPDs.md) - VPD addressing system that gets persisted as part of the structure
- [ADR-0029: Selective Hash-Consing](0029-Global-Hash-Consing.md) - Hash-consing optimization system that dramatically improves serialization performance while maintaining ID reference integrity

### Neutral

- Developers need to be aware of the ID reference system when modifying the system.
- May require synchronization considerations if concurrent processing is implemented in the future.

## Implementation Notes

1. Implement the pure tree structure using Clojure's immutable data structures.
2. Develop a system for generating and managing integer IDs within each Instrument.
3. Implement serialization and deserialization using Nippy.
4. Develop a testing suite to verify correct handling of ID references, including complex musical scenarios.
5. Implement robust mechanisms for maintaining consistency of ID references during updates and deletions.

## Future Considerations

1. Explore implementing concurrent processing for further performance improvements.
2. Develop tools for analyzing and visualizing ID references to aid in debugging and optimization.
3. Implement a versioning system to handle future changes in piece structure.
4. Investigate lazy loading techniques for referenced structures in extremely large pieces.
5. Monitor and profile ID reference operations in real-world usage to identify optimization opportunities.
6. Explore optimizations for traversing and processing structures with ID references in very large musical pieces.
7. Consider implementing a diff-based serialization for storing revision histories efficiently.
