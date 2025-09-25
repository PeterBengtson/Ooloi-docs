# ADR-0007: Use of Nippy for File Persistence

## Table of Contents
- [Decision](#decision)
- [Context](#context)
- [Rationale](#rationale)
- [Why Fressian Was Not Chosen](#why-fressian-was-not-chosen)
- [Consequences](#consequences)
- [Implementation Approach](#implementation-approach)
- [Future Considerations](#future-considerations)
- [Related Decisions](#related-decisions)

## Decision

Use Nippy for file persistence in Ooloi, leveraging its built-in features and implementing custom handlers for Ooloi's specific needs.

## Context
- Ooloi deals with complex musical structures that form pure trees with integer ID references for cross-tree relationships.
- The system needs to efficiently serialize and deserialize large musical scores.
- The persistence mechanism should support handling large files efficiently.
- Concurrent serialization of multiple pieces is desirable for performance.
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
  - ~~Doesn't natively handle shared structure, requiring custom solutions for Ooloi's musical structures~~ **RESOLVED**: Custom Nippy transforms now provide registry-based shared object optimization with substantial performance gains.

## Implementation Approach
1. Utilize Nippy's core functions for basic serialization and deserialization.
2. Create a system to handle shared structure using Vector Path Descriptors (VPDs).
3. Implement asynchronous serialization using Clojure futures for concurrent operations.
4. Integrate with the Piece Manager for efficient piece storage and retrieval.
5. Implement chunked serialization and deserialization for handling very large musical scores.
6. **Registry-Based File Size Optimization**: Leverage custom Nippy transforms to replace cached objects with shared references during serialization, achieving 69% file size reduction and 4x performance improvement for repetitive musical structures (see [ADR-0029: Selective Hash-Consing](0029-Global-Hash-Consing.md)).

## Future Considerations
- Implement versioning for serialized data to manage future schema changes.
- Optimize compression settings for very large scores.
- Develop a comprehensive testing suite for serialization/deserialization, especially for complex structures and shared structure.
- Monitor performance and optimize as necessary, potentially implementing lazy loading for large scores.
- Consider implementing a caching mechanism for frequently accessed pieces or sections.

## Related Decisions

- [ADR-0010: Pure Trees](0010-Pure-Trees.md) - Tree data structure that Nippy will serialize
- [ADR-0008: VPDs](0008-VPDs.md) - VPD addressing system that gets serialized as part of the structure
- [ADR-0012: Persisting Pieces](0012-Persisting-Pieces.md) - Persistence architecture that uses Nippy for serialization
