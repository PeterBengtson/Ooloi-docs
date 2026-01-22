# ADR: Pure Tree Structure with Integer ID References

## Status: Accepted

## Context

Ooloi is a complex music notation software that needs to represent and manipulate intricate musical structures. The system must efficiently handle large musical scores while supporting various musical elements and their relationships.

## Decision

The Piece model is implemented as a pure tree structure, with integer IDs used for referencing objects across the tree. This architecture supports efficient processing, rendering, and persistence while accurately representing musical relationships and allowing for straightforward serialization and deserialization.

## Detailed Design

### 1. Basic Structure

The Ooloi Piece is organized as a pure tree:

- Pieces contain Musicians and Layouts
- Musicians have Instruments
- Instruments have Staves
- Staves have Measures
- Measures have Voices
- Voices contain musical items (Rests, Pitches, Chords, Tuplets, Tremolandos, etc.)

This forms the basic tree structure of the Piece.

### 2. Key Properties

- The entire structure, including both musical and visual hierarchies, is always a pure tree.
- Cross-references between elements (e.g., for slurs, ties) are implemented using integer IDs.
- Each Instrument maintains its own ID counter for generating unique IDs within its scope.

### 3. Piece Components

- **Piece**: The top-level container for all musical content and layout information.
- **Musician**: Represents a performer or group of performers.
- **Instrument**: Represents a musical instrument played by a Musician.
- **Staff**: Represents a single line of musical notation.
- **Voice**: Represents an independent line of music within a Staff.
- **Measure**: Contains the actual musical content (notes, rests, etc.).
- **Layout**: Represents the visual arrangement of the score.
- **PageView, SystemView, StaffView, MeasureView**: Represent different levels of visual organization in the score.

## Visual Representation

See [this diagram](/img/trees-and-dags.jpg) for a visual representation of the tree structure versus potential alternatives.

### 4. Musical Items

Musical items that can appear within Measures include:

- Pitches: Individual notes with a specific pitch.
- Chords: Groups of simultaneously sounding Pitches.
- Rests: Periods of silence.
- Tuplets: Groups of notes played in a mathematical ratio to the underlying beat.
- Tremolandos: Rapid alternation between two notes or chords.
- Trills: Rapid alternation between two adjacent notes.

### 5. Vector Path Descriptors (VPDs)

VPDs are used for efficient navigation and serialization of the nested piece structure:

- **VPD form (compact)**: `[:m 2 0 0 0 46]` - The standard notation used in all code (for musical elements)
- **Navigator form (expanded)**: `[:musicians 2 :instruments 0 :staves 0 :measures 4 :voices 06]` - Internal form for Specter operations
- Layout elements use `:l` instead of `:m` as the VPD prefix

The compact VPD form is what you always write in code. The navigator form is only used internally by Specter for path navigation.

Key VPD operations:
```clojure
(defn navigator [descriptor])  ; Converts VPD to navigator form
(defn compact [descriptor])    ; Converts navigator to VPD form
(defn retrieve [piece descriptor])
```

### 6. Integer ID References

Certain musical elements require references to other parts of the score:

- Ties
- Slurs
- Glissandos
- Ottava (8va and 8vb)
- Dynamic hairpins and textual dynamics
- Trills with arbitrarily long extension lines

These elements use integer IDs to reference their endpoints. The IDs are unique within the scope of an Instrument.

### 7. ID Generation and Management

- Each Instrument object maintains its own ID counter.
- IDs are generated sequentially for each new reference needed within an Instrument.
- The ID counter can be reset when necessary (e.g., after deleting measures).

### 8. Attachments and References

- Elements that can be referenced (e.g., Pitches, Chords) implement the TakesAttachment trait.
- TakesAttachment elements have an `endpoint-id` field to store their unique ID.
- Elements that reference others (e.g., Slurs) store the ID of their endpoint in an `endpoint-id` field.

**API Abstraction**: The user-facing API abstracts away endpoint-id complexity entirely. Users work with VPDs to specify attachment locations, and the system handles ID generation and resolution automatically:

```clojure
;; User provides VPDs – system manages endpoint-ids internally
(add-attachment start-vpd piece "slur" end-vpd)
(add-attachment start-vpd piece "tie" end-vpd)
(add-attachment start-vpd piece "8va" end-vpd)

;; Behind the scenes: endpoint-ids are automatically assigned and resolved
;; Users never need to work with integer IDs directly
```

This design allows the API to mirror musical thinking ("add slur from this note to that note") while maintaining the pure tree structure with integer ID cross-references internally.

### 9. Serialization and Deserialization

The pure tree structure allows for straightforward serialization and deserialization:

1. The entire Piece can be serialized directly as a tree structure.
2. Integer IDs are preserved during serialization, maintaining all necessary references.
3. Deserialization recreates the tree structure with all IDs intact.
4. No special resolution of references is needed after deserialization.

### 10. Measure Copying and Deletion

Special handling is required when copying or deleting measures:

- When copying measures, new IDs must be generated for all TakesAttachment elements in the copied measures.
- References within the copied measures must be updated to use the new IDs.
- When deleting measures, care must be taken to update or remove any references to the deleted elements.

### 11. Efficient Processing and Rendering

Ooloi employs several strategies for efficient processing and rendering of musical elements:

- Adaptive search strategy for traversing the musical structure
- Transducers for efficient processing of musical elements
- Specialized algorithms for processing specific musical elements (e.g., slurs, ties)
- Concurrent processing per Musician for improved performance

### 12. Extensibility

The Ooloi architecture is designed to be extensible, allowing for:

- Addition of new musical elements and notations
- Implementation of new processing and rendering algorithms
- Extension of the structure to accommodate new musical concepts

## Consequences

### Positive

- Maintains a pure tree structure, simplifying many operations and allowing straightforward serialization/deserialization.
- Allows for efficient traversal and manipulation of the musical score.
- Supports the representation of advanced musical notations and techniques.
- Enables efficient serialization and deserialization of complex musical structures.
- Facilitates accurate and aesthetically pleasing rendering of musical elements like slurs.
- Eliminates memory management complexity found in traditional object-oriented notation systems through structural sharing.
- Enables natural collaborative editing capabilities where changes create new versions that can be automatically merged, solving synchronization challenges that plague mutable architectures.
- Supports automatic history tracking and elegant undo/redo through version navigation without complex operational transforms.

### Negative

- Requires careful management of integer IDs to maintain consistency.
- Special handling is needed when copying or deleting measures to maintain ID integrity.

### Neutral

- Necessitates specialized algorithms for efficient searching and processing of elements that reference others.
- Requires developers to understand the ID reference system when implementing new features.
- May require performance tuning for very large or complex musical scores.

## Implementation Notes

1. Implement the pure tree structure using Clojure's immutable data structures.
2. Develop a system for generating and managing integer IDs within each Instrument.
3. Create specialized algorithms for traversing and manipulating elements that use ID references.
5. Develop an enhanced VPD system for efficient navigation, serialization, and deserialization of the tree structure.
6. Implement comprehensive serialization and deserialization processes that preserve the tree structure and ID references.
7. Develop procedures for safely copying and deleting measures, including ID regeneration and reference updates.
8. Implement efficient lookup mechanisms for resolving ID references during rendering and other operations.
9. Ensure all ID references are properly handled during updates to the Piece structure.
10. Implement transactional updates to maintain consistency of the tree structure and ID references.
11. Develop a comprehensive testing suite covering various musical scenarios and edge cases, with particular focus on ID reference integrity.

## Future Considerations

1. Optimize performance for extremely large musical scores.
2. Explore potential for parallel processing of independent sections of the score.
3. Investigate machine learning techniques for enhancing musical element rendering (e.g., slur shapes).
4. Consider implementing a plugin system for extending Ooloi's capabilities.
5. Explore real-time collaborative editing features leveraging the pure tree structure and ID reference system.
6. Investigate integration with audio playback and MIDI systems.
7. Consider developing a visual debugger for inspecting the structure and rendered elements.
8. Investigate opportunities for sharing functional data structure approaches with other open-source music notation projects seeking to solve similar architectural challenges.
9. **Ossia staff support**: Future implementation of ossia staves (instruments that enter at arbitrary measures) will use Staff-level `entry-measure` and `exit-measure` fields rather than Voice-level temporal positioning. This design aligns with the current architecture where Voices are measure-local content containers, not temporal streams. Ossia is primarily a visual/layout concern that should be expressed at the Staff level, keeping the Voice model pure and simple while enabling proper score rendering and layout coordination.

## Note on Structure Terminology

The structure described in this document is a pure tree, with cross-references implemented using integer IDs. This approach maintains the benefits of a tree structure (such as easy serialization and traversal) while allowing for the representation of complex musical relationships through ID references.

## Related Decisions

- [ADR-0008: VPDs](0008-VPDs.md) - Vector Path Descriptor system for navigating the tree structure
- [ADR-0011: Shared Structure](0011-Shared-Structure.md) - Shared structure concepts building on the pure tree foundation
- [ADR-0012: Persisting Pieces](0012-Persisting-Pieces.md) - Persistence model based on the tree structure and ID references
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - Operations-only API modifying the pure tree structure
