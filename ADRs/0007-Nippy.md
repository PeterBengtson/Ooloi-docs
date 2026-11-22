# ADR-0007: Use of Nippy for File Persistence

## Status

Implemented

## Table of Contents
- [Decision](#decision)
- [Context](#context)
- [Rationale](#rationale)
- [Why Fressian Was Not Chosen](#why-fressian-was-not-chosen)
- [Consequences](#consequences)
- [Implementation Approach](#implementation-approach)
- [The File Format](#the-file-format)
  - [The Registry](#the-registry)
  - [The Version Stamp](#the-version-stamp)
  - [Every Write Freezes on a Thread of Its Own](#every-write-freezes-on-a-thread-of-its-own)
- [Future Considerations](#future-considerations)
- [Related Decisions](#related-decisions)

## Decision

Use Nippy for file persistence in Ooloi, leveraging its built-in features and implementing custom handlers for Ooloi's specific needs.

## Context
- Ooloi deals with complex musical structures that form pure trees with integer ID references for cross-tree relationships.
- The system needs to efficiently serialize and deserialize large musical scores.
- The persistence mechanism should support handling large files efficiently.
- Concurrent **deserialisation** of multiple pieces is desirable for performance — several scores can be opened at once, and the concurrency scales with the hardware the deployment chose.
- **Concurrent serialisation is not desirable, and writes are serialised deliberately.** Parallel saves buy nothing: nobody is waiting on a write, so there is no latency to reduce. What they cost is real. Two writes to one path resolve in arbitrary order rather than the order they were asked for. Competing large writes divide disk throughput, so both take longer than either would alone. And a single queue is what lets an autosave notice that a write is already in flight for a piece and decline to add a second. One writer, in submission order, is the shape the problem wants ([ADR-0051](0051-Filesystem-Operations-Real-and-Virtual.md) §7).
- The system should be able to handle future extensions and modifications to data structures.

## Rationale
1. Simplicity and Ease of Use: Nippy provides a simple API that's easy to use out of the box, requiring minimal configuration for basic use cases.

2. Built-in Support for Clojure Data Structures: Nippy is designed specifically for Clojure and has native support for all Clojure data structures, including records and protocols.

3. Performance: Nippy shows excellent performance for Clojure-specific use cases, as it's optimized for Clojure data structures and use patterns.

4. Active Maintenance: Nippy is actively maintained and updated, ensuring ongoing support and improvements.

5. Compression: Nippy includes built-in compression options, which can be very useful for reducing the size of serialized data, especially important for large musical scores.

6. Encryption Support: Nippy offers built-in support for data encryption, which may be useful for future security requirements.

7. Community Adoption: Nippy has wide adoption in the Clojure community, providing ample resources, examples, and community support.

8. Flexibility: Nippy allows for easy extension with custom data types, crucial for our complex musical data structures.

## Why Fressian Was Not Chosen
While Fressian was initially considered due to its language-agnostic nature and handling of complex data structures, it was ultimately not selected for the following reasons:

1. Complexity: Fressian requires more configuration and custom coding for basic use cases compared to Nippy's simpler API.

2. Clojure-specific Optimization: While Fressian is language-agnostic, it lacks the Clojure-specific optimizations that Nippy provides, potentially leading to lower performance for our use case.

3. Community Support: Fressian has less community adoption in the Clojure ecosystem, which could lead to fewer resources and examples for complex use cases.

4. Maintenance: Fressian has seen less active maintenance compared to Nippy, which could be a concern for long-term support and improvements.

5. Built-in Features: Nippy offers built-in compression and encryption, which would require additional implementation if using Fressian.

6. Performance: In our specific use case with Clojure data structures, Nippy often outperforms Fressian.

## Consequences
- Pros:
  - Simple API and seamless integration with Clojure projects.
  - Efficient handling of Clojure-specific data structures.
  - Built-in compression options for managing large data sets.
  - Active maintenance and strong community support.
  - Excellent performance in Clojure-specific use cases.
  - Built-in encryption support for potential future security needs.
- Cons:
  - Clojure-specific, which may limit interoperability with non-JVM systems if needed in the future.
  - Doesn't natively handle shared structure: Nippy writes an object again each time it meets it. Ooloi's custom transforms write each shared object once, in a registry ahead of the piece (see [The File Format](#the-file-format)).

## Implementation Approach
1. Utilize Nippy's core functions for basic serialization and deserialization.
2. Create a system to handle shared structure using Vector Path Descriptors (VPDs).
3. Defer serialisation rather than performing it inline: a save takes its snapshot immediately and submits the write to a single-threaded queue, returning a handle that delivers the outcome. Reads run on the shared thread pool, bounded so the interface always has a core. Neither uses `clojure.core/future`, whose exceptions vanish unobserved ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md), [ADR-0017](0017-System-Architecture.md)).
4. Integrate with the Piece Manager for efficient piece storage and retrieval.
5. Implement chunked serialization and deserialization for handling very large musical scores.
6. **The registry-first file format**: custom Nippy transforms write each distinct shared object once, in a registry ahead of the piece, and the piece refers to it (see [The File Format](#the-file-format) and [ADR-0029: Selective Hash-Consing](0029-Global-Hash-Consing.md)).

## The File Format

Every writer writes one frozen map, `{:format 1 :registry [...] :piece <bytes>}`, and every reader returns the value it holds. A file, an in-memory buffer and a socket stream carry the same format: it belongs to the persistence layer (`ooloi.shared.ops.persistence`), not to the file.

### The Registry

The registry holds one entry, `{:type … :args …}`, for each distinct `Pitch`, `Rest`, `Chord` and `Articulation` that the global cache would hold (`hash-cons-cacheable?`, [ADR-0029](0029-Global-Hash-Consing.md)): the object's Nippy type id, and the arguments its caching constructor takes. The registry fills while the piece is frozen, so the piece is frozen to bytes first, uncompressed, and the registry goes ahead of those bytes in the map; LZ4 compresses the whole.

In the piece's bytes such an object is a reference to its entry: one negative int, `-(index + 1)`, under the object's own type id. Any other object of the four types, one in a relationship or carrying a non-articulation attachment, is written inline, as the length of its frozen arguments followed by them. A length is never negative, so a reader tells the two apart by sign.

An entry is registered only after the objects its own arguments hold (a chord's pitches, a pitch's articulations), and inside the registry those objects are references too, so an entry refers only to entries before it. Registration is add-if-absent, in the order first met: Nippy writes a value again from the start each time it grows its buffer, and a repeated pass adds nothing.

A reader builds the entries in index order through their caching constructors (`create-pitch`, `create-rest`, `create-chord`, `create-articulation`), each entry's references resolved to the objects already built, and then thaws the piece with every reference resolved the same way. A reopened piece therefore shares its objects through the global cache exactly as a newly constructed one would.

### The Version Stamp

`:format` is a positive integer, 1 for this format (`persistence/format-version`). A reader has one question to ask of it, whether it can read the file, and an integer answers it with one comparison:

- A value with no `:format`, as files written before the stamp existed are, is format 0 and is read as it is, every object inline.
- A number up to the reader's own is read.
- A newer number is refused before anything is built or thawed, and `open-piece` reports it as `:unsupported-piece-format` ([ADR-0051](0051-Filesystem-Operations-Real-and-Virtual.md) §7).

Any change a reader must know about is the next number. A save writes the current format, so an older file becomes the current one the first time it is saved.

### Every Write Freezes on a Thread of Its Own

Nippy keeps an outermost freeze's final buffer on the thread that made it when that buffer is at most 1 MiB, and gives every nested freeze on that thread a new buffer that size. A writer's nested freezes are the arguments of each object registered for the first time and of each object written inline, and a writer makes two outermost freezes, the piece's and then the map's. On a thread used again, as the save queue's is, each nested freeze would allocate up to a mebibyte. Each writer therefore runs on a thread created for the write and shut down after it, which takes its buffer with it; and an object already registered is written as its reference without its arguments being frozen again ([Piece Persistence Guide](../guides/PIECE_PERSISTENCE_GUIDE.md) §Serialisation).

## Future Considerations
- Optimize compression settings for very large scores.
- Develop a comprehensive testing suite for serialization/deserialization, especially for complex structures and shared structure.
- Monitor performance and optimize as necessary, potentially implementing lazy loading for large scores.
- Consider implementing a caching mechanism for frequently accessed pieces or sections.

## Related Decisions

- [ADR-0010: Pure Trees](0010-Pure-Trees.md) - Tree data structure that Nippy will serialize
- [ADR-0008: VPDs](0008-VPDs.md) - VPD addressing system that gets serialized as part of the structure
- [ADR-0012: Persisting Pieces](0012-Persisting-Pieces.md) - Persistence architecture that uses Nippy for serialization
