# ADR-0029: Global Hash-Consing for Immutable Musical Objects

## Table of Contents
- [Status](#status)
- [Context](#context)
  - [Current Memory Inefficiency](#current-memory-inefficiency)
  - [Serialization Challenge](#serialization-challenge)
- [Decision](#decision)
- [Design](#design)
  - [Layer 1: Constructor-Level Hash-Consing](#layer-1-constructor-level-hash-consing)
  - [Layer 2: Hash-Consing Modifications](#layer-2-hash-consing-modifications)
  - [Layer 3: Nippy-Integrated Serialization](#layer-3-nippy-integrated-serialization)
  - [Layer 4: Background Optimization](#layer-4-background-optimization)
- [Implementation Strategy](#implementation-strategy)
- [Hash-Cons Key Normalization](#hash-cons-key-normalization)
- [Monitoring and Observability](#monitoring-and-observability)
- [Integration Points](#integration-points)
- [Risk Mitigation](#risk-mitigation)
- [Success Metrics](#success-metrics)
- [Consequences](#consequences)
- [References](#references)
- [Approval](#approval)

## Status
Accepted - September 24, 2025

## Context

Ooloi creates massive numbers of immutable musical objects during typical usage - a single symphony can contain 50,000+ pitch objects, 10,000+ rest objects, and thousands of attachment objects. Analysis shows that 99%+ of these objects are duplicates with identical parameter values.

Since these objects are immutable, identical instances can be safely shared across the entire system without affecting behavior. This presents an opportunity for dramatic memory optimization through **hash-consing** - a technique where identical immutable values are represented by the same object instance.

### Current Memory Inefficiency

```clojure
;; Every constructor call creates a new instance, even for identical parameters
(create-pitch :note "C4" :duration 1/4)  ; Instance #1
(create-pitch :note "C4" :duration 1/4)  ; Instance #2 (identical but separate)
(create-pitch :note "C4" :duration 1/4)  ; Instance #3 (identical but separate)

;; A typical symphony: 85,000 objects with only ~760 unique parameter combinations
;; Memory waste: 99.1% redundant instances
```

### Serialization Challenge

Standard serialization destroys shared object structure, losing all hash-consing benefits:

```clojure
;; Before serialization: shared objects in memory
(def piece {:measures [pitch-instance pitch-instance pitch-instance]})  ; Same instance × 3

;; After deserialization: separate objects again
(def loaded-piece (nippy/thaw (nippy/freeze piece)))  ; 3 separate instances
```

## Decision

We will implement a **Multi-Layer Global Hash-Consing System** with four complementary strategies:

1. **Constructor-Level Hash-Consing** - Intercept object creation and return hash-consed instances
2. **Hash-Consing Modifications** - Check hash table for desired result instead of modifying existing objects
3. **Nippy-Integrated Serialization** - Preserve shared structure across save/load cycles
4. **Background Optimization** - JVM string deduplication-style consolidation of equal objects

## Design

### Layer 1: Constructor-Level Hash-Consing

#### Global Hash-Cons Tables
```clojure
;; Global hash-cons tables for high-impact immutable objects
(def ^:private global-pitch-hash-cons
  (atom (cache/lru-cache-factory {} :threshold 50000)))
(def ^:private global-rest-hash-cons
  (atom (cache/lru-cache-factory {} :threshold 1000)))
(def ^:private global-chord-hash-cons
  (atom (cache/lru-cache-factory {} :threshold 10000)))
(def ^:private global-attachment-hash-cons
  (atom (cache/lru-cache-factory {} :threshold 5000)))
```

#### Enhanced Constructor Pattern
```clojure
(defn create-pitch [& {:keys [note duration attachments endpoint-id]
                       :or {attachments [] endpoint-id nil}}]
  (if endpoint-id
    ;; Has endpoint-id: NEVER cache - create unique instance directly
    (->Pitch note duration attachments endpoint-id)
    ;; No endpoint-id: Use cache-first approach
    (let [hash-cons-key {:note note
                         :duration duration
                         :attachments (normalize-attachments attachments)}]
      (if-let [consed-pitch (cache/lookup @global-pitch-hash-cons hash-cons-key)]
        (do
          (swap! hash-cons-hits inc)
          consed-pitch)
        ;; Hash-cons miss: create new object and add to hash-cons table
        (let [new-pitch (->Pitch note duration attachments nil)]
          (swap! hash-cons-misses inc)
          (swap! global-pitch-hash-cons cache/miss hash-cons-key new-pitch)
          new-pitch)))))
```

#### Hash-Cons Key Design Principles

**Exclude Instance-Specific Fields:**
- **endpoint-ids** - Nearly always unique, would prevent all hash-cons hits
- **temporary metadata** - Creation timestamps, debug info, etc.
- **runtime state** - Any mutable or session-specific data

**Include Core Musical Identity:**
- **note/duration/attachments** for pitches
- **duration/attachments** for rests
- **type/value/parameters** for attachments (normalized)

### Layer 2: Hash-Consing Modification Strategy

Instead of traditional "modify existing object → create new instance" pattern:

```clojure
(defn add-attachment [musical-obj attachment]
  ;; Traditional: modify existing, create new instance
  ;; (assoc musical-obj :attachments (conj (:attachments musical-obj) attachment))

  ;; Hash-consing: look up desired end result using constructors
  (let [new-attachments (conj (:attachments musical-obj) attachment)
        endpoint-id (:endpoint-id musical-obj)]
    (case (type musical-obj)
      ;; Constructor will cache if endpoint-id is nil, create unique if endpoint-id has value
      Pitch (create-pitch :note (:note musical-obj)
                          :duration (:duration musical-obj)
                          :attachments new-attachments
                          :endpoint-id endpoint-id)
      Rest (create-rest :duration (:duration musical-obj)
                        :attachments new-attachments
                        :endpoint-id endpoint-id))))

(defn transpose-pitch [pitch semitones]
  ;; Hash-consing transposition - constructor handles caching decision
  (let [new-note (transpose-note (:note pitch) semitones)]
    (create-pitch :note new-note
                  :duration (:duration pitch)
                  :attachments (:attachments pitch)
                  :endpoint-id (:endpoint-id pitch))))  ; Cached if nil, unique if has value
```

### Layer 3: Nippy-Integrated Hash-Consing Serialization

#### Custom Freeze/Thaw Transforms

```clojure
(require '[taoensso.nippy :as nippy])

;; Serialization session state
(def ^:dynamic *serialization-hash-cons-registry* nil)
(def ^:dynamic *deserialization-hash-cons-registry* nil)

(nippy/extend-freeze
  ooloi.shared.models.musical.pitch.Pitch
  :ooloi/cached-pitch
  [pitch data-output-stream]
  (let [hash-cons-key (get-hash-cons-key pitch)
        registry @*serialization-hash-cons-registry*]
    (if-let [hash-cons-id (get registry hash-cons-key)]
      ;; Object already serialized in this session - write reference
      (do
        (.writeByte data-output-stream 1)  ; Reference marker
        (.writeInt data-output-stream hash-cons-id))
      ;; First occurrence - write full object and register
      (let [hash-cons-id (count registry)]
        (swap! *serialization-hash-cons-registry* assoc hash-cons-key hash-cons-id)
        (.writeByte data-output-stream 0)  ; Full object marker
        (.writeInt data-output-stream hash-cons-id)
        (nippy/freeze-to-out! data-output-stream
          {:note (:note pitch)
           :duration (:duration pitch)
           :attachments (:attachments pitch)})))))

(nippy/extend-thaw
  :ooloi/cached-pitch
  [data-input-stream]
  (let [is-reference (.readByte data-input-stream)
        hash-cons-id (.readInt data-input-stream)]
    (if (= 1 is-reference)
      ;; Reference - look up in deserialization registry
      (get @*deserialization-hash-cons-registry* hash-cons-id)
      ;; Full object - deserialize and register
      (let [data (nippy/thaw-from-in! data-input-stream)
            pitch (create-pitch :note (:note data)
                               :duration (:duration data)
                               :attachments (:attachments data))]
        (swap! *deserialization-hash-cons-registry* assoc hash-cons-id pitch)
        pitch))))
```

#### Serialization Session Management

```clojure
(defmacro with-hash-cons-session [& body]
  `(binding [*serialization-hash-cons-registry* (atom {})
             *deserialization-hash-cons-registry* (atom {})]
     ~@body))

(defn serialize-piece-with-hash-cons [piece]
  (with-hash-cons-session
    (nippy/freeze piece {:references? true
                         :compressor nippy/lz4-compressor})))

(defn deserialize-piece-with-hash-cons [frozen-data]
  (with-hash-cons-session
    (let [result (nippy/thaw frozen-data {:references? true})]
      ;; Ensure all objects are in global hash-cons tables after deserialization
      (ensure-objects-in-global-hash-cons result)
      result)))
```

#### Serialization Performance Impact

**Standard Nippy Serialization:**
- 50,000 pitch objects × 100 bytes = 5MB raw
- LZ4 compression → 2MB final size

**Cache-Aware Nippy Serialization:**
- 500 unique objects × 100 bytes = 50KB object data
- 49,500 references × 4 bytes = 198KB reference data
- Total 250KB raw → 100KB compressed

**Result: 95% reduction in serialized size**

### Layer 4: Background Optimization (JVM String Deduplication Pattern)

#### Background Task Architecture

```clojure
(defn hash-cons-optimization-task []
  "Background daemon modeled after JVM string deduplication"
  (go-loop [piece-queue (pm/get-all-piece-ids)]
    (let [batch-size (min 10 (count piece-queue))
          batch (take batch-size piece-queue)
          remaining (drop batch-size piece-queue)]

      ;; Process batch
      (doseq [piece-id batch]
        (optimize-piece-hash-cons piece-id))

      ;; Yield between batches (like JVM heap region processing)
      (<! (timeout 100))

      (if (empty? remaining)
        ;; Completed full scan - wait before next cycle
        (do
          (<! (timeout (optimization-frequency)))
          (recur (pm/get-all-piece-ids)))
        ;; Continue with remaining pieces
        (recur remaining)))))

(defn optimization-frequency []
  "Memory pressure-aware optimization frequency (like JVM GC triggers)"
  (let [memory-usage (get-memory-usage-percent)]
    (cond
      (> memory-usage 80) (* 5 60 1000)   ; Every 5 minutes under high pressure
      (> memory-usage 60) (* 15 60 1000)  ; Every 15 minutes under medium pressure
      :else (* 30 60 1000))))             ; Every 30 minutes normally
```

#### Object Consolidation Logic

```clojure
(defn optimize-piece-hash-cons [piece-id]
  "Replace equal objects with identical hash-consed objects"
  (when-let [piece-ref (pm/get-piece-ref piece-id)]
    (dosync
      (let [optimization-count (atom 0)
            objects-processed (atom 0)]
        (alter piece-ref
          (fn [piece]
            (walk/postwalk
              (fn [obj]
                (swap! objects-processed inc)
                (if (and (cacheable-object? obj)
                         (should-optimize-object? obj))
                  (let [hash-cons-key (get-hash-cons-key obj)]
                    (if-let [hash-consed-obj (get-from-global-hash-cons hash-cons-key)]
                      (if (identical? obj hash-consed-obj)
                        obj  ; Already using hash-consed object
                        (do
                          (swap! optimization-count inc)
                          hash-consed-obj))  ; Replace with hash-consed instance
                      (do
                        ;; Add to hash-cons table for future consolidation
                        (add-to-global-hash-cons hash-cons-key obj)
                        obj)))
                  obj))
              piece)))

        (when (> @optimization-count 0)
          (log/info "Optimized" @optimization-count "of" @objects-processed
                   "objects in piece" piece-id))))))

(defn should-optimize-object? [obj]
  "Determine if object is candidate for background optimization"
  (and
    (cacheable-object? obj)
    ;; Focus on objects likely to be long-lived (JVM pattern)
    (> (object-age obj) (* 5 60 1000))  ; 5+ minutes old
    ;; Skip objects with endpoint-ids (poor hash-cons candidates)
    (nil? (:endpoint-id obj))
    ;; Skip recently accessed objects (optimization overhead not worth it)
    (not (recently-accessed? obj))))
```

## Implementation Strategy

### Phase 1: Constructor Hash-Consing Foundation
1. Implement global hash-cons tables with LRU eviction
2. Enhance `create-pitch` and `create-rest` with hash-consing
3. Hash-cons key normalization excluding endpoint-ids
4. Monitoring and statistics infrastructure
5. Target: >95% hash-cons hit rates for common objects

### Phase 2: Nippy Integration
1. Custom freeze/thaw transforms for hash-consed object types
2. Serialization session management for hash-cons references
3. Roundtrip testing to ensure shared structure preservation
4. Target: >90% reduction in serialized file sizes

### Phase 3: Hash-Cons-First Operations
1. Implement hash-cons-first `add-attachment`, `transpose-pitch`
2. Extend pattern to other modification operations
3. Target: Eliminate unnecessary object creation in modification chains

### Phase 4: Background Optimization
1. JVM-style background optimization daemon
2. Memory pressure-aware scheduling
3. Incremental batch processing
4. Target: Continuous consolidation of equal objects across all pieces

### Phase 5: Full System Integration
1. Chord hash-consing with normalized pitch collections
2. Common attachment pre-population (hash-cons table warming)
3. Integration with server-side scratchpad (Issues #19/#20)
4. Performance profiling and optimization

## Hash-Cons Key Normalization

### Attachment Normalization
```clojure
(defn normalize-attachments [attachments]
  "Canonical ordering and normalization for better hash-cons hits"
  (->> (or attachments [])
       (map normalize-single-attachment)
       (sort-by attachment-sort-key)
       vec))

(defn normalize-single-attachment [attachment]
  "Remove instance-specific fields from attachment for hash-consing"
  (dissoc attachment :endpoint-id :creation-time :session-id))
```

### Pitch Normalization
```clojure
(defn get-pitch-hash-cons-key [note duration attachments]
  {:note (normalize-note note)
   :duration (normalize-duration duration)
   :attachments (normalize-attachments attachments)})
   ;; NEVER include :endpoint-id in hash-cons keys
```

## Monitoring and Observability

### Hash-Cons Performance Metrics
```clojure
(defn hash-cons-performance-report []
  {:hash-cons-statistics
   {:pitch-hash-cons {:size (cache/size @global-pitch-hash-cons)
                  :hit-rate (/ @hash-cons-hits (+ @hash-cons-hits @hash-cons-misses))
                  :evictions @pitch-hash-cons-evictions}
    :rest-hash-cons {:size (cache/size @global-rest-hash-cons)
                 :hit-rate (/ @rest-hash-cons-hits (+ @rest-hash-cons-hits @rest-hash-cons-misses))
                 :evictions @rest-hash-cons-evictions}}

   :serialization-efficiency
   {:avg-file-size-reduction-percent @avg-serialization-reduction
    :objects-shared-per-save @avg-objects-shared
    :compression-effectiveness @nippy-compression-ratio}

   :background-optimization
   {:last-optimization-run @last-background-run
    :pieces-optimized-today @daily-pieces-optimized
    :objects-consolidated-today @daily-objects-consolidated
    :estimated-memory-saved-mb (/ @estimated-memory-saved 1024 1024)}

   :memory-impact
   {:total-objects-created @lifetime-objects-created
    :total-hash-cons-hits @lifetime-hash-cons-hits
    :memory-savings-percent (/ @lifetime-hash-cons-hits @lifetime-objects-created)}})
```

### Performance Alerting
```clojure
(defn check-hash-cons-health []
  (let [stats (hash-cons-performance-report)]
    (when (< (get-in stats [:hash-cons-statistics :pitch-hash-cons :hit-rate]) 0.90)
      (alert/warn "Pitch hash-cons hit rate below 90%" stats))
    (when (> (get-in stats [:hash-cons-statistics :pitch-hash-cons :size]) 45000)
      (alert/info "Pitch hash-cons table approaching capacity" stats))
    (when (< (get-in stats [:serialization-efficiency :avg-file-size-reduction-percent]) 80)
      (alert/warn "Serialization efficiency degraded" stats))))
```

## Integration Points

### Server-Side Scratchpad Synergy
Hash-consing amplifies benefits of server-side scratchpad from Issues #19/#20:

```clojure
(SRV/atomic [
  ;; Objects created on server - immediate hash-cons benefits
  [:let $quarter-rest (create-rest :duration 1/4)]    ; Hash-cons hit likely
  [:let $c4 (create-pitch :note "C4" :duration 1/4)]  ; Hash-cons hit likely

  ;; Variable reuse amplified by hash-consing
  (add-item [:musicians 0 :instruments 0 :staves 0 :voices 0 :measures 0] "piece" $c4)
  (add-item [:musicians 1 :instruments 0 :staves 0 :voices 0 :measures 0] "piece" $c4)
  ;; Same hash-consed instance used across multiple placements - zero allocation overhead
])
```

### Piece Manager Integration
- Automatic background optimization of all managed pieces
- Hash-cons aware save/load using enhanced Nippy serialization
- Memory usage monitoring and reporting
- Hash-cons table warming during piece manager startup

## Risk Mitigation

### Hash-Cons Invalidation
**Risk**: Hash-consed objects become stale or inconsistent
**Mitigation**: Objects are immutable - no invalidation needed

### Memory Leaks
**Risk**: Unbounded hash-cons table growth
**Mitigation**: LRU eviction with configurable thresholds

### Performance Degradation
**Risk**: Hash-cons lookup overhead exceeds benefits
**Mitigation**:
- Only hash-cons high-repetition object types
- Benchmark hash-cons performance continuously
- Fallback to direct creation if hash-consing becomes ineffective

### Serialization Compatibility
**Risk**: Nippy changes break hash-cons serialization
**Mitigation**:
- Version serialization format
- Graceful degradation to standard serialization
- Comprehensive roundtrip testing

## Success Metrics

### Memory Efficiency
- **Target**: 99%+ memory reduction for redundant musical objects
- **Measurement**: Compare total object instances vs unique hash-cons entries
- **Baseline**: Large symphony pieces for realistic workload testing

### Hash-Cons Performance
- **Target**: >95% hit rates for pitch/rest objects when endpoint-ids excluded
- **Target**: >80% hit rates for attachment objects
- **Measurement**: Hash-cons hit/miss ratios with detailed breakdown by object type

### Serialization Impact
- **Target**: >90% reduction in serialized piece file sizes
- **Target**: Shared structure preserved across save/load roundtrips
- **Measurement**: Before/after file size comparisons, object identity preservation tests

### Background Optimization
- **Target**: Successful consolidation of equal objects in existing pieces
- **Target**: Memory reclamation measurable via GC metrics
- **Measurement**: Objects consolidated per optimization cycle, memory usage trends

### System Integration
- **Target**: Zero API changes for existing code
- **Target**: All existing tests pass unchanged
- **Target**: Performance improvements across plugin development workflows

## Consequences

### Positive Consequences
- **Dramatic memory reduction**: 99%+ savings for typical musical pieces
- **Improved performance**: Faster object creation, serialization, equality checks
- **Better user experience**: Larger pieces load faster, use less memory
- **Transparent benefits**: Existing code automatically benefits
- **Scalability**: System can handle much larger musical pieces

### Potential Negative Consequences
- **Implementation complexity**: Multi-layer caching system requires careful design
- **Cache tuning**: Thresholds and eviction policies need optimization
- **Debugging complexity**: Shared objects may complicate debugging
- **Memory usage patterns**: Different memory usage profile may reveal other bottlenecks

### Mitigation of Negative Consequences
- **Comprehensive testing**: Extensive test coverage for all hash-cons scenarios
- **Monitoring infrastructure**: Detailed metrics and alerting for hash-cons performance
- **Gradual rollout**: Phase-based implementation with fallback mechanisms
- **Documentation**: Clear architectural documentation for future maintainers

## References

- JVM String Deduplication (JDK 8u20+): Background consolidation pattern
- Nippy Documentation: Custom serialization transforms
- Issue #19: SRV/* API Endpoint Macros (server-side scratchpad integration)
- Issue #20: Atomic API Operation Argument Structure (scratchpad synergy)
- Issue #2: Fix defrecord roundtrips for plugin compatibility (foundational requirement)

## Approval

This ADR requires approval from the architecture team, with particular focus on:
1. Cache key design and endpoint-id exclusion strategy
2. Nippy integration approach and serialization compatibility
3. Background optimization scheduling and resource usage
4. Monitoring and observability requirements
5. Integration impact on existing Issues #19/#20