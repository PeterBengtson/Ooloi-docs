# ADR-0029: Selective Hash-Consing for Musical Objects

## Status

Implemented

## Table of Contents
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
    - [The Counters Are a Required Dependency](#the-counters-are-a-required-dependency)
- [Measured Performance Results](#measured-performance-results)
  - [File Size](#file-size)
  - [Write and Read Times](#write-and-read-times)
  - [What the Tests Establish](#what-the-tests-establish)
- [Consequences](#consequences)
  - [Benefits](#benefits)
  - [Trade-offs](#trade-offs)
  - [Risk Mitigation](#risk-mitigation)
- [References](#references)

## Context

Musical notation systems create enormous numbers of immutable objects during operation. A single symphony can contain 50,000+ pitch objects, 10,000+ rest objects, and thousands of attachment objects. Analysis of typical musical pieces reveals that over 99% of these objects are duplicates with identical parameter values.

Since musical objects are immutable by design, identical instances can be safely shared across the entire system without affecting behavior. This enables **hash-consing** - a technique where identical immutable values are represented by the same object instance.

Since over 99% of musical objects are duplicates with identical parameter values, hash-consing provides substantial memory reductions across typical compositions. A saved file holds each shared object once and an open builds each once, so files are smaller and opens faster as well (see [Measured Performance Results](#measured-performance-results)).

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
  - For a chord, all its pitches cacheable

**Not Cacheable:**
- **TakesAttachment objects with endpoint-id**: Any object participating in a relationship (slurs, ties, glissandos)
- **TakesAttachment objects with non-articulation attachments**: Objects with dynamics, hairpins, or mixed attachment types
- **Chords holding an uncacheable pitch**: a pitch that starts a tie, or ends one, makes its chord unique to that relationship as well
- **Non-articulation attachments**: Dynamics, slurs, ties, hairpins, glissandos
- **All other object types**: Measures, voices, staves, musicians, instruments, pieces, etc.

The predicate uses recursive checking: a TakesAttachment object is cacheable only if all its attachments are themselves cacheable, which implicitly means they must all be articulations, and a chord only if its pitches are as well.

## Architecture

### Selective Caching System

The hash-consing system employs a central predicate function (`hash-cons-cacheable?`) that performs a simple, lightweight structural check on objects to determine caching eligibility. The predicate examines object type and immediate properties only—no usage analysis, pattern recognition, or frequency tracking is required. This approach ensures that only object types likely to have many identical instances participate in caching, while object types that tend to be unique are excluded to avoid unnecessary overhead.

### Constructor-Level Implementation

Hash-consing is implemented at the object constructor level, where each cacheable type maintains its own global LRU cache. When objects are created, the system first checks if an identical instance exists in the cache before creating a new one. See [Cache Key Strategy](#cache-key-strategy) in Implementation Strategy for details on how cache keys are constructed.

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

The system integrates with Nippy serialization to preserve shared object structure across save/load cycles. Nippy does not deduplicate on its own, so a piece is written registry-first: each cacheable object the piece holds is written once, as its constructor arguments, in a registry ahead of the piece, and the piece refers to it by index; uncached objects are written inline. On load the registry is read first and each entry is reconstructed through its caching constructor, so the thawed piece shares objects through the global cache exactly as a newly constructed one would. LZ4 compression applies to the whole. The format, its references and its version stamp are specified in [ADR-0007](0007-Nippy.md) §The File Format.

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
- **Integrant Component**: `ooloi.backend.components/cache-daemon`. Its `init-key` starts the maintenance schedule and its `halt-key!` stops it — a component that reports itself running while starting nothing is worse than no component at all
- **Interval**: deployment configuration rather than a constant of the domain. A server holding
  several large scores and a desktop application holding one want different answers, and a sweep's
  cost grows with the score. It is set with `--cache-daemon-interval-seconds` or
  `OOLOI_CACHE_DAEMON_INTERVAL_SECONDS`, in seconds, refused at startup if not greater than zero,
  and defaults to `maintenance-interval-seconds` when unset. Seconds outside the component,
  milliseconds on its config key, converted once in the injection spec
- **Scheduling**: a dedicated single-thread scheduled executor. The shared Claypoole pool offers no periodic scheduling and is sized for short, latency-sensitive work, so it is the wrong home for a multi-second sweep. Scheduled with a fixed *delay* rather than a fixed rate: the interval is measured from the end of one sweep to the start of the next, so a sweep that outgrows its interval is followed by a gap rather than immediately by another. A sweep whose cost grows with the score must never be able to occupy the machine continuously
- **Thread**: named, and marked daemon. Nothing waits on a sweep — its work is discardable — so there is nothing to hold the JVM open for
- **Shutdown**: `halt-key!` shuts the executor down and then waits, bounded, for a sweep already running. Without the wait the halt proceeds to the piece manager the sweep is still reading. The bound is a courtesy rather than a requirement: proceeding costs one stderr line where hanging the quit costs more
- **Failure**: the sweep runs inside a catch, because a scheduled task that lets a throwable escape is never run again by its executor — one bad sweep would otherwise end hash-consing for the session, silently. What is caught goes to a replaceable handler, log-only by default and replaced in the combined application by the surfacing boundary of [ADR-0017](0017-System-Architecture.md)
- **Piece Manager Dependency**: access to all stored pieces for system-wide optimization

**Optimization Algorithm:**
- **Single-Pass Traversal**: Zero-allocation `transduce` with push-based transducer composition
- **Cache Integration**: Direct usage of existing global cache infrastructure
- **Identity Restoration**: VPD mutations using value branch for identity-preserving writes
- **All Object Types**: Support for Pitch, Rest, Chord, and Articulation optimization

**The daemon is transparent, and that is a requirement rather than a nicety.** It substitutes
values that are `=` to the ones they replace and merely not `identical?` to them: nothing about the
score has changed, and nothing anywhere may behave as though it had. It emits no events, never
marks a piece dirty, pushes no undo entry, and changes nothing about what is saved. It alters how
much memory a piece occupies, and nothing else. An undo that restores a piece to a state the daemon
had already optimised is therefore harmless — the next sweep does the work again, and serialisation
does not depend on it: the file records each object by its value, whether or not it is shared.

That transparency holds because the daemon writes **outside** the API and change-detection layer,
and it must continue to. Routing it through the VPD write funnel would mark every open piece dirty
on every sweep: dirtiness is set by `(not (identical? before after))` read *above* the structural
narrowing ([ADR-0052](0052-Change-Detection-and-Event-Generation.md) §5), and the funnel's no-op
gate compares with `identical?` and deliberately never `=`, an `=` deep-compare over an arbitrary
subtree being O(n) on the write path. The daemon's write is exactly the `=`-but-not-`identical?`
case that gate cannot see, so the funnel would read it as a genuine edit.

**Technical Implementation:**

Each piece is optimised outside any transaction and committed in one of its own. The computation is
a full timewalk, and STM settles a conflict in favour of the older transaction — so a sweep holding
one open across that work would be liable to restart the user's write rather than its own. The
commit yields: a piece that moved while its optimisation was being computed keeps whoever got there
first, and the computed result is discarded. That costs nothing, where clobbering an edit would cost
the edit.

```clojure
(defn daemon-maintenance-cycle [_daemon]
  (doseq [piece-ref (pm/all-piece-refs)]
    (let [before @piece-ref                      ; a plain ref read — atomic for one ref
          after  (optimize-piece-once before)]   ; pure, expensive, outside STM
      (dosync
        (alter piece-ref #(if (identical? % before) after %))))))

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

The `dosync` coordinates nothing. It is there because a ref cannot be written outside one, and it
holds a single identity comparison — required syntax rather than a design element.

**Observability.** A component with no observable effects has no observable failures either, so the
daemon's counters are the only sanctioned window into it: sweeps completed, objects unified, pieces
processed, pieces abandoned because they moved underneath a sweep, and the average cost of optimising
a piece. They reach the server statistics endpoint alongside every other counter
([ADR-0025](0025-Server-Statistics-Architecture.md)).

The figure reported is the interval the running daemon **scheduled**, not the compiled-in default.
The component records it from the same binding it hands its executor, so the reported and the actual
cannot diverge — which matters because they would otherwise part company the moment an operator set
the switch, and the field exists precisely to make a sweep count interpretable. It is `null` where no
daemon exists.

#### The Counters Are a Required Dependency

The server-statistics counters are a required dependency of the cache daemon rather than an optional
extra, and `init-key` **refuses a configuration that does not supply them**. A daemon without
counters is not a degraded daemon but an unobservable one, which is the single state this component
must never occupy: it would report itself running, sweep on schedule, and offer no evidence of doing
either.

Refusing at initialisation reports the fault at the configuration that caused it. The alternative
surfaces as a failure on the daemon's own thread an interval later, with nothing left to connect it
to the mistake — a `NullPointerException` incrementing a counter that is not there, on a thread
nothing is waiting on. The refusal names the missing dependency, so a misconfigured startup exits
with the dependency code rather than a generic one
([ADR-0017](0017-System-Architecture.md) §Error Classification).

## Measured Performance Results

The benchmark's orchestral piece (`bench.persistence-bench` in `shared/dev/bench/`: 29 instruments, about 520 items a measure) at three sizes, written through `file-writer` and read back through `file-reader`, the write a save performs and the read an open performs. Each figure is the median of three runs on each machine, set against the same piece with every object written in full, as a file without a `:format` holds it ([ADR-0007](0007-Nippy.md) §The File Format). The M3 is Ooloi's Mac platform; the Intel Core i7-8750H, which Ooloi does not support ([ADR-0050](0050-Platform-Support-Policy.md)), is there for comparison. Each run's output is kept in `shared/dev/bench/results/`.

### File Size

| Piece | Items | Frozen, in full → registry-first | File, in full → registry-first |
|---|---:|---:|---:|
| 10 measures | 5,200 | 423 KB → 85 KB | 8.2 KB → 6.2 KB |
| 100 measures | 52,000 | 4.07 MB → 691 KB | 32 KB → 10 KB |
| 1000 measures | 520,000 | 40.6 MB → 6.73 MB | 183 KB → 49 KB |

*Frozen* is the serialisation before compression, and *File* is what lands on disk; the two machines agree on both within the generator's variation, under one per cent. Written once rather than at every occurrence, the shared objects leave the frozen piece five to six times smaller. LZ4 had already compressed most of that repetition away, so the file shrinks less: 1.3 times at 10 measures, 3.2 at 100 and 3.8 at 1000. The generated piece repeats far more than a real score does, so a real score of the same length has more distinct objects.

### Write and Read Times

| Piece | Write, M3 | Read, M3 | Write, i7-8750H | Read, i7-8750H |
|---|---:|---:|---:|---:|
| 10 measures | 6.5 → 7.4 ms | 12.5 → 1.0 ms | 13.8 → 12.7 ms | 33.7 → 2.0 ms |
| 100 measures | 46 → 59 ms | 128 → 7.2 ms | 109 → 103 ms | 295 → 13.5 ms |
| 1000 measures | 513 → 511 ms | 1.23 s → 68.5 ms | 1.18 s → 822 ms | 2.71 s → 144 ms |

A read takes a twelfth to a twenty-second of the time: it builds each shared object once, through its caching constructor, and resolves every other occurrence to it. A write does the registry's work as it goes, a lookup for each object it writes; on the M3 it takes as long, or up to 29 per cent longer at the smaller sizes, and on the i7-8750H 5 to 30 per cent less. The write runs on the save queue, which nobody waits on ([ADR-0007](0007-Nippy.md)).

### What the Tests Establish

The tests enter through `save-piece` and `open-piece`, the calls the application makes, and establish that:

- a saved file holds the registry-first map, its registry ahead of the piece;
- each distinct cacheable object appears in the registry once, however often the piece uses it, and holds only the piece's own objects, not others the global cache holds;
- an open builds each distinct cacheable object once, including those held inside other entries, and the reopened piece's objects are `identical?` to newly constructed ones;
- an entry follows the entries its arguments hold;
- an object in a relationship, the two notes of a tie, is written inline and reopens equal;
- saving the same piece again writes a byte-identical file;
- a file written before the format, every object inline, still opens;
- a file whose `:format` is newer than the reader's is refused as `:unsupported-piece-format`, and no piece is registered;
- a save of a piece holding five hundred distinct chords allocates no more than twice what a save of the same piece without them does.

## Consequences

### Benefits
- **Significant memory reduction**: Substantial reductions in memory usage as duplicate objects are eliminated system-wide
- **Smaller files**: a saved file holds each shared object once, in a registry ahead of the piece
- **Faster opens**: an open builds each shared object once, in a twelfth to a twenty-second of the time it takes with every object written in full (see [Measured Performance Results](#measured-performance-results))
- **Reduced garbage collection pressure**: Fewer object allocations reduce GC overhead
- **Better scalability**: System can handle larger orchestral scores and complex compositions
- **Transparent implementation**: Existing code benefits without requiring modifications
- **Functional programming benefit**: Leverages immutability for safe object sharing
- **Network efficiency**: Client-server communication benefits from smaller data payloads
- **Selective targeting**: Caching focuses on object types likely to have many identical instances, avoiding cache pollution

### Trade-offs

Minimal and well-mitigated (bounded LRU memory, configurable daemon CPU, comprehensive test coverage). The one cost a save carries is the registry's lookups: on an Apple M3 a small piece's write takes up to 29 per cent longer than it would with every object written in full, on a queue nobody waits on (see [Measured Performance Results](#measured-performance-results)). Like JVM string deduplication, the optimization is a net-positive with no user-visible downsides.

### Risk Mitigation
- **Comprehensive testing**: Extensive validation ensures shared instances maintain correctness
- **Bounded resource usage**: LRU eviction prevents unbounded cache growth
- **Fallback mechanisms**: System can operate correctly even if caching becomes ineffective
- **Clear architectural boundaries**: Well-defined rules for what gets cached maintain system predictability

## References

- [JVM String Deduplication (JDK 8u20+)](https://openjdk.org/jeps/192): Background consolidation pattern for memory optimization
- [Hash-consing in functional programming](https://en.wikipedia.org/wiki/Hash_consing): Canonical representation techniques for immutable data structures
- [LRU Cache implementations](https://github.com/clojure/core.cache): Bounded memory usage with clojure.core.cache library
