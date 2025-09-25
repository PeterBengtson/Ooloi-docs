# ADR-0029: Selective Hash-Consing for Musical Objects

## Table of Contents
- [Status](#status)
- [Context](#context)
- [Decision](#decision)
- [Design Rationale](#design-rationale)
- [Architecture](#architecture)
- [Implementation Strategy](#implementation-strategy)
- [Consequences](#consequences)
- [References](#references)

## Status
Accepted - September 24, 2025

## Context

Musical notation systems create enormous numbers of immutable objects during operation. A single symphony can contain 50,000+ pitch objects, 10,000+ rest objects, and thousands of attachment objects. Analysis of typical musical pieces reveals that over 99% of these objects are duplicates with identical parameter values.

Since musical objects are immutable by design, identical instances can be safely shared across the entire system without affecting behavior. This enables **hash-consing** - a technique where identical immutable values are represented by the same object instance.

Since over 99% of musical objects are duplicates with identical parameter values, hash-consing provides substantial memory reductions across typical compositions. File sizes shrink proportionally, with corresponding improvements in save/load performance.

This optimization is straightforward in functional programming with immutable data structures, where sharing identical objects is safe. The same optimization is complex in procedural/object-oriented programming where mutable state makes object sharing require synchronization and defensive copying.

### Memory Usage Patterns

Musical compositions exhibit highly repetitive patterns:

- **Common notes**: C4, D4, E4 appear thousands of times per piece
- **Standard durations**: Quarter notes (1/4), eighth notes (1/8) dominate most music
- **Frequent articulations**: Staccato, accent, legato are reused extensively
- **Contextual elements**: Dynamics (forte, piano) and relationships (slurs, ties) are more situational

This repetition pattern suggests that selective caching based on frequency would be more effective than universal caching.

### Functional Programming Advantage

Hash-consing demonstrates a key difference between functional and procedural/object-oriented approaches. In mutable systems, sharing object instances requires defensive copying and synchronization mechanisms to prevent modifications from affecting multiple references, which eliminates performance benefits.

Immutable data structures make sharing safe since objects cannot be modified after creation. This enables:

- Object sharing without defensive copying
- No synchronization or locking mechanisms required
- Elimination of sharing-related bugs through immutability guarantees
- Reduced memory usage and creation overhead

This approach leverages functional programming principles to enable optimizations that require significant complexity in mutable architectures.

## Decision

We implement a **Selective Hash-Consing System** that caches objects based on their repetition frequency in musical usage patterns, rather than attempting to cache all objects universally.

The key architectural principle is that hash-consing effectiveness depends on **repetition frequency** - only objects that appear many times benefit from shared instances, while low-repetition objects add cache overhead without meaningful benefit.

## Design Rationale

### Frequency-Based Caching Strategy

Objects are categorized into two groups based on empirical analysis of musical usage:

**High-Frequency Objects (Cached)**:
- **Core musical elements**: Basic pitches, rests, and chords with common parameter combinations
- **Articulation markings**: Staccato, accent, tenuto - these are reused extensively across musical pieces
- **Objects with articulation-only attachments**: Musical elements that have only articulation markings

**Low-Frequency Objects (Not Cached)**:
- **Dynamic markings**: Forte, piano, crescendo - these are contextual and less frequently duplicated
- **Relationship objects**: Slurs and ties that connect specific musical elements
- **Mixed attachment objects**: Elements with combinations of different attachment types
- **Unique relationship objects**: Any object with endpoint identifiers indicating specific relationships

## Architecture

### Selective Caching System

The hash-consing system employs a central predicate function that analyzes object types and content to determine caching eligibility. This approach ensures that only high-repetition objects participate in caching, while low-frequency objects are excluded to avoid unnecessary overhead.

### Constructor-Level Implementation

Hash-consing is implemented at the object constructor level, where each cacheable type maintains its own global LRU cache. When objects are created, the system first checks if an identical instance exists in the cache before creating a new one.

### Cache Key Strategy

The system uses constructor arguments directly as cache keys, eliminating the need for complex hash key construction while ensuring that identical parameter combinations map to shared instances.

### Memory Optimization Benefits

This selective approach provides significant memory reduction by targeting the highest-impact scenarios:

- **Articulation sharing**: All instances of common articulations (staccato, accent) share single objects system-wide
- **Core object deduplication**: Frequently-used pitches, rests, and chords are automatically deduplicated
- **Cross-type efficiency**: Articulations are shared between different musical element types (pitches, rests, chords)


## Implementation Strategy

The hash-consing system operates through constructor-level caching with selective eligibility based on object repetition patterns. Core musical objects (pitches, rests, chords) and high-repetition attachments (articulations) use global LRU caches, while contextual objects (dynamics) and relationship objects (slurs, ties) remain uncached.

The system integrates with serialization operations to preserve shared structure across save/load cycles and provides enhanced memory efficiency for server-side operations.

## Consequences

### Benefits
- **Significant memory reduction**: Substantial reductions in memory usage as duplicate objects are eliminated system-wide
- **Smaller file sizes**: Save files shrink proportionally to memory reductions, enabling faster storage and network transfers
- **Improved I/O performance**: Save/load operations benefit from reduced data volumes
- **Reduced garbage collection pressure**: Fewer object allocations reduce GC overhead
- **Better scalability**: System can handle larger orchestral scores and complex compositions
- **Transparent implementation**: Existing code benefits without requiring modifications
- **Functional programming benefit**: Leverages immutability for safe object sharing
- **Network efficiency**: Client-server communication benefits from smaller data payloads
- **Selective targeting**: Caching focuses on high-repetition patterns, avoiding cache pollution

### Trade-offs
- **Selective complexity**: System requires analysis of object usage patterns to determine caching eligibility
- **Cache management**: LRU caches require tuning and monitoring to maintain effectiveness
- **Memory profile changes**: Different allocation patterns may shift performance bottlenecks to other areas
- **Debugging considerations**: Shared object instances may require different debugging approaches

### Risk Mitigation
- **Comprehensive testing**: Extensive validation ensures shared instances maintain correctness
- **Bounded resource usage**: LRU eviction prevents unbounded cache growth
- **Fallback mechanisms**: System can operate correctly even if caching becomes ineffective
- **Clear architectural boundaries**: Well-defined rules for what gets cached maintain system predictability

## References

- [JVM String Deduplication (JDK 8u20+)](https://openjdk.org/jeps/192): Background consolidation pattern for memory optimization
- [Hash-consing in functional programming](https://en.wikipedia.org/wiki/Hash_consing): Canonical representation techniques for immutable data structures
- [LRU Cache implementations](https://github.com/clojure/core.cache): Bounded memory usage with clojure.core.cache library