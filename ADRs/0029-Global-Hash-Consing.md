# ADR-0029: Selective Hash-Consing for Musical Objects

## Table of Contents
- [Status](#status)
- [Context](#context)
  - [Object Duplication Characteristics](#object-duplication-characteristics)
  - [Functional Programming Advantage](#functional-programming-advantage)
- [Decision](#decision)
- [Design Rationale](#design-rationale)
  - [Type-Based Caching Strategy](#type-based-caching-strategy)
- [Architecture](#architecture)
  - [Selective Caching System](#selective-caching-system)
  - [Constructor-Level Implementation](#constructor-level-implementation)
  - [Memory Optimization Benefits](#memory-optimization-benefits)
- [Implementation Strategy](#implementation-strategy)
  - [Constructor-Level Caching Architecture](#constructor-level-caching-architecture)
  - [Cache Key Strategy](#cache-key-strategy)
  - [Serialization Integration with Nippy](#serialization-integration-with-nippy)
  - [Background Cache Optimization Daemon](#background-cache-optimization-daemon)
- [Measured Performance Results](#measured-performance-results)
  - [Registry-Based File Size Optimization](#registry-based-file-size-optimization)
  - [Serialization Performance Gains](#serialization-performance-gains)
  - [Deserialization Performance Gains](#deserialization-performance-gains)
  - [Implementation Verification](#implementation-verification)
- [Consequences](#consequences)
  - [Benefits](#benefits)
  - [Trade-offs](#trade-offs)
  - [Risk Mitigation](#risk-mitigation)
- [References](#references)

## Status
Accepted - September 24, 2025
**IMPLEMENTATION COMPLETE** - September 26, 2025

## Context

Musical notation systems create enormous numbers of immutable objects during operation. A single symphony can contain 50,000+ pitch objects, 10,000+ rest objects, and thousands of attachment objects. Analysis of typical musical pieces reveals that over 99% of these objects are duplicates with identical parameter values.

Since musical objects are immutable by design, identical instances can be safely shared across the entire system without affecting behavior. This enables **hash-consing** - a technique where identical immutable values are represented by the same object instance.

Since over 99% of musical objects are duplicates with identical parameter values, hash-consing provides substantial memory reductions across typical compositions. File sizes shrink proportionally, with corresponding improvements in save/load performance.

This optimization is straightforward in functional programming with immutable data structures, where sharing identical objects is safe. The same optimization is complex in procedural/object-oriented programming where mutable state makes object sharing require synchronization and defensive copying.

### Object Duplication Characteristics

Musical compositions create enormous numbers of duplicate identical objects:

- **Common notes**: Notes like C4, D4, E4 appear thousands of times per piece - identical pitch objects with same parameters
- **Limited duration set**: Musical durations are few in number (e.g., 1/4, 1/8, 1/2) - identical duration values appear repeatedly
- **Frequent articulations**: Articulations like staccato, accent, legato are reused extensively - identical attachment objects
- **Contextual elements**: Dynamics (forte, piano) and relationships (slurs, ties) tend to be unique due to endpoint-ids

Since immutable objects with identical parameters can safely share a single instance, selective caching based on object type and structure enables substantial memory reduction.

### Functional Programming Advantage

Hash-consing demonstrates a key difference between functional and procedural/object-oriented approaches. In mutable systems, sharing object instances requires defensive copying and synchronization mechanisms to prevent modifications from affecting multiple references, which eliminates performance benefits.

Immutable data structures make sharing safe since objects cannot be modified after creation. This enables:

- Object sharing without defensive copying
- No synchronization or locking mechanisms required
- Elimination of sharing-related bugs through immutability guarantees
- Reduced memory usage and creation overhead

This approach leverages functional programming principles to enable optimizations that require significant complexity in mutable architectures.

## Decision

We implement a **Selective Hash-Consing System** that caches objects based on their type and structural properties, rather than attempting to cache all objects universally.

The key architectural principle is that hash-consing effectiveness depends on **object identity** - identical immutable objects can share a single instance. Certain object types (articulations, pitches without relationships, rests) tend to have many identical instances, while others (objects with endpoint-ids, dynamics) tend to be unique. A simple structural predicate determines caching eligibility based on type and properties, not on pattern recognition or usage analysis.

## Design Rationale

### Type-Based Caching Strategy

The `hash-cons-cacheable?` predicate determines caching eligibility through simple structural checks:

**Cacheable Objects:**
- **Articulation attachments**: Always cacheable (staccato, accent, tenuto, etc.)
- **TakesAttachment objects with articulation-only attachments**: Pitches, rests, chords that have:
  - No `:endpoint-id` (not part of a relationship)
  - All attachments recursively cacheable (which means articulations only)

**Not Cacheable:**
- **TakesAttachment objects with endpoint-id**: Any object participating in a relationship (slurs, ties, glissandos)
- **TakesAttachment objects with non-articulation attachments**: Objects with dynamics, hairpins, or mixed attachment types
- **Non-articulation attachments**: Dynamics, slurs, ties, hairpins, glissandos
- **All other object types**: Measures, voices, staves, musicians, instruments, pieces, etc.

The predicate uses recursive checking: a TakesAttachment object is cacheable only if all its attachments are themselves cacheable, which implicitly means they must all be articulations.

## Architecture

### Selective Caching System

The hash-consing system employs a central predicate function (`hash-cons-cacheable?`) that performs a simple, lightweight structural check on objects to determine caching eligibility. The predicate examines object type and immediate properties only—no usage analysis, pattern recognition, or frequency tracking is required. This approach ensures that only object types likely to have many identical instances participate in caching, while object types that tend to be unique are excluded to avoid unnecessary overhead.

### Constructor-Level Implementation

Hash-consing is implemented at the object constructor level, where each cacheable type maintains its own global LRU cache. When objects are created, the system first checks if an identical instance exists in the cache before creating a new one. See [Cache Key Strategy](#cache-key-strategy-1) in Implementation Strategy for details on how cache keys are constructed.

### Memory Optimization Benefits

This selective approach provides significant memory reduction by targeting the highest-impact scenarios:

- **Articulation sharing**: All instances of common articulations (staccato, accent) share single objects system-wide
- **Core object deduplication**: Frequently-used pitches, rests, and chords are automatically deduplicated
- **Cross-type efficiency**: Articulations are shared between different musical element types (pitches, rests, chords)


## Implementation Strategy

### Constructor-Level Caching Architecture

The hash-consing system operates through constructor-level caching with selective eligibility determined by the `hash-cons-cacheable?` predicate. Each cacheable object type maintains its own global LRU cache using `clojure.core.cache/lru-cache-factory` with configurable thresholds (typically 1,000-10,000 entries).

Core musical objects (pitches, rests, chords) without endpoint-ids and articulation attachments use global LRU caches, while objects with endpoint-ids (participating in relationships) and non-articulation attachments (dynamics, slurs, ties) remain uncached based on the `hash-cons-cacheable?` predicate.

### Cache Key Strategy

Constructor arguments are used directly as cache keys, eliminating the need for complex hash key construction. The system uses a simple cache-miss pattern:

```clojure
(if-let [cached-object (cache/lookup @global-cache args)]
  cached-object
  (do
    (swap! global-cache cache/miss args new-object)
    new-object))
```

### Serialization Integration with Nippy

The system integrates with Nippy serialization to preserve shared object structure across save/load cycles. Nippy automatically handles object deduplication during serialization, ensuring that shared instances in memory remain shared in the serialized form. Upon deserialization, objects are reconstructed through the same constructor caching mechanism, maintaining the hash-consing benefits.

### Background Cache Optimization Daemon

**Concept and JVM Parallel:**

Similar to JVM String Deduplication (JEP 192), the system employs a background consolidation approach where object sharing occurs transparently during normal operation without requiring explicit consolidation phases. The daemon performs dynamic hash-consing of objects created through mutations, **directly paralleling JVM string consolidation**. Just as the JVM string consolidation daemon identifies duplicate strings created during program execution and consolidates them in the background, our musical object daemon identifies duplicate musical objects and consolidates them asynchronously.

**JVM String Consolidation Parallel:**
- JVM: String concatenation creates `"Hello" + " " + "World"` → daemon later consolidates identical results
- Ooloi: Cached C4 pitch + staccato attachment → daemon later consolidates identical "C4 with staccato" objects

When cached objects are modified (such as adding a staccato to a C4 pitch), the resulting new objects can themselves become cached if they pass the `hash-cons-cacheable?` predicate:

- A cached C4 pitch receives a staccato attachment, creating a "C4 with staccato" object
- This new object passes the caching predicate (it's a TakesAttachment with only articulation attachments)
- The background daemon identifies this duplicate cacheable object during traversal
- Future creations of "C4 with staccato" will return the same cached instance
- This process continues recursively - cached objects that are mutated can produce new cached objects

Rather than trying to predict all possible object combinations at creation time, the daemon traverses stored pieces and consolidates duplicate cacheable objects it encounters. The `hash-cons-cacheable?` predicate determines eligibility through simple structural checks, not usage frequency analysis. The LRU cache management handles memory pressure automatically by evicting least-recently-used entries when cache thresholds are exceeded, maintaining bounded memory usage while preserving frequently-accessed shared instances.

**Component Architecture:**
- **Integrant Component**: `ooloi.backend.components/cache-daemon` with proper init/halt lifecycle
- **Background Thread Management**: Configurable maintenance intervals with graceful shutdown
- **STM Integration**: Thread-safe piece modifications using `dosync` coordination
- **Piece Manager Dependency**: Access to all stored pieces for system-wide optimization

**Optimization Algorithm:**
- **Single-Pass Traversal**: Zero-allocation `transduce` with push-based transducer composition
- **Cache Integration**: Direct usage of existing global cache infrastructure
- **Identity Restoration**: VPD mutations using value branch for identity-preserving writes
- **All Object Types**: Support for Pitch, Rest, Chord, and Articulation optimization

**Technical Implementation:**
```clojure
(defn daemon-maintenance-cycle [_daemon]
  (dosync
    (doseq [piece-ref (vals @(var-get #'pm/piece-store))]
      (alter piece-ref optimize-piece-once))))

(defn ^:private optimize-piece-once [piece]
  (transduce
    (comp (timewalk {})                                ; Traverse piece, emit [item vpd position] tuples
          (filter #(p/hash-cons-cacheable? (item %)))) ; Keep only cacheable items
    (completing                                        ; Wrap reducing fn to provide completion arity
      (fn [piece tuple]                                ; Reducing fn: thread piece state through
        (let [it (item tuple)
              cached (get-or-cache-object it)]         ; Fetch/create cached version
          (if (identical? cached it)                   ; Already identical?
            piece                                      ; No change needed
            (vpd-ops/mutate (vpd tuple) piece cached))))) ; Replace with cached version at VPD location
    piece                                              ; Initial accumulator value
    [piece]))
```

**Verification Results:**
- **Identity Sharing Achieved**: Test verification shows identical objects become `(identical? obj1 obj2) => true` after daemon processing
- **All Test Types Pass**: 4 behavioral tests (19 total checks) including component lifecycle, safe access, thread management, and cache optimization
- **System Integration**: Full backend test suite (610 checks) passes with daemon enabled

**Implementation Complete (September 26, 2025):** The background cache optimization daemon has been fully implemented as an Integrant component with proper lifecycle management, background thread operations, STM transaction coordination, and timewalker integration. The daemon performs single-pass optimization using reduce over timewalk traversal, achieving identity sharing restoration for identical musical objects across pieces.

## Measured Performance Results

Implementation of the selective hash-consing system has been completed and comprehensively tested with 50,000-object datasets to measure real-world performance characteristics. The results demonstrate substantial gains across all optimization targets: file size reduction, serialization speed, and deserialization speed.

**Test Environment:** Performance measurements were conducted on a 2017 MacBook Pro to provide conservative baseline figures. Modern hardware would show proportionally better absolute performance while maintaining the same optimization ratios.

### Registry-Based File Size Optimization

The system implements a registry-based serialization format that replaces cached objects with shared references during storage operations. Testing with large datasets shows dramatic file size reductions:

**File Size Comparison (50,000 objects):**
- **Identical cached objects**: 14,540 bytes
- **Varied uncached objects**: 24,590 bytes
- **Compression improvement**: **69% better** (1.69x file size reduction)

The registry system scales excellently - compression effectiveness actually **improves** with larger datasets containing more duplicate identical objects. Musical pieces naturally contain many duplicate objects (multiple C4 pitches, quarter-note rests, staccato markings), and larger pieces amplify these benefits.

### Serialization Performance Gains

Hash-consed objects demonstrate substantial serialization speed improvements compared to uncached objects. The registry-based approach eliminates redundant serialization work by replacing cached objects with lightweight references:

**Serialization Speed (50,000 objects):**
- **Cached objects**: 556.58 ms
- **Uncached objects**: 2,294.79 ms
- **Performance improvement**: **4.12x faster** serialization

The performance improvement results from two factors: (1) cached objects are replaced with compact shared references during serialization, and (2) registry lookup operations are significantly faster than full object serialization.

### Deserialization Performance Gains

The registry-based system achieves even greater deserialization speed improvements. Shared references are resolved through constructor calls that immediately return cached instances, eliminating object reconstruction overhead:

**Deserialization Speed (50,000 objects):**
- **Cached objects**: 258.72 ms
- **Uncached objects**: 1,144.52 ms
- **Performance improvement**: **4.42x faster** deserialization

Constructor-based cache lookup provides faster object restoration than full deserialization, while simultaneously preserving cache identity and shared object benefits in the restored system state.

### Implementation Verification

The complete implementation has been verified through comprehensive testing:

- **76 passing tests** including full serialization/deserialization roundtrip verification
- **Registry extraction** from all cached object types (pitch, rest, chord, articulation)
- **Reference replacement** via Nippy transforms with dynamic binding
- **Cache identity preservation** through constructor-based reconstruction
- **Structured file format** with metadata and registry-first layout

Performance testing demonstrates that the optimization system scales linearly - larger datasets with more duplicate identical objects achieve proportionally better compression and speed improvements.

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
- **Selective targeting**: Caching focuses on object types likely to have many identical instances, avoiding cache pollution

### Trade-offs

Minimal and well-mitigated (bounded LRU memory, configurable daemon CPU, comprehensive test coverage). Like JVM string deduplication, the optimization is a net-positive with no user-visible downsides.

### Risk Mitigation
- **Comprehensive testing**: Extensive validation ensures shared instances maintain correctness
- **Bounded resource usage**: LRU eviction prevents unbounded cache growth
- **Fallback mechanisms**: System can operate correctly even if caching becomes ineffective
- **Clear architectural boundaries**: Well-defined rules for what gets cached maintain system predictability

## References

- [JVM String Deduplication (JDK 8u20+)](https://openjdk.org/jeps/192): Background consolidation pattern for memory optimization
- [Hash-consing in functional programming](https://en.wikipedia.org/wiki/Hash_consing): Canonical representation techniques for immutable data structures
- [LRU Cache implementations](https://github.com/clojure/core.cache): Bounded memory usage with clojure.core.cache library