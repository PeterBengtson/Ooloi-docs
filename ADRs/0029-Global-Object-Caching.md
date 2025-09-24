# ADR-0029: Global Object Caching and Deduplication for Immutable Musical Objects

## Status
Proposed

## Context

Ooloi creates massive numbers of immutable musical objects during typical usage - a single symphony can contain 50,000+ pitch objects, 10,000+ rest objects, and thousands of attachment objects. Analysis shows that 99%+ of these objects are duplicates with identical parameter values.

Since these objects are immutable, identical instances can be safely shared across the entire system without affecting behavior. This presents an opportunity for dramatic memory optimization through global caching and deduplication strategies.

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

Standard serialization destroys shared object structure, losing all caching benefits:

```clojure
;; Before serialization: shared objects in memory
(def piece {:measures [pitch-instance pitch-instance pitch-instance]})  ; Same instance × 3

;; After deserialization: separate objects again
(def loaded-piece (nippy/thaw (nippy/freeze piece)))  ; 3 separate instances
```

## Decision

We will implement a **Multi-Layer Global Caching System** with four complementary strategies:

1. **Constructor-Level Caching** - Intercept object creation and return cached instances
2. **Cache-First Modifications** - Check cache for desired result instead of modifying existing objects
3. **Nippy-Integrated Serialization** - Preserve shared structure across save/load cycles
4. **Background Optimization** - JVM string deduplication-style consolidation of equal objects

## Design

### Layer 1: Constructor-Level Caching

#### Global Cache Infrastructure
```clojure
;; Global LRU caches for high-impact immutable objects
(def ^:private global-pitch-cache
  (atom (cache/lru-cache-factory {} :threshold 50000)))
(def ^:private global-rest-cache
  (atom (cache/lru-cache-factory {} :threshold 1000)))
(def ^:private global-chord-cache
  (atom (cache/lru-cache-factory {} :threshold 10000)))
(def ^:private global-attachment-cache
  (atom (cache/lru-cache-factory {} :threshold 5000)))
```

#### Enhanced Constructor Pattern
```clojure
(defn create-pitch [& {:keys [note duration attachments endpoint-id]
                       :or {attachments [] endpoint-id nil}}]
  (let [;; CRITICAL: Exclude endpoint-id from cache key (nearly always unique)
        cache-key {:note note
                   :duration duration
                   :attachments (normalize-attachments attachments)}]
    (if-let [cached-pitch (cache/lookup @global-pitch-cache cache-key)]
      (do
        (swap! cache-hits inc)
        ;; Apply endpoint-id to cached instance (don't cache endpoint-ids)
        (if endpoint-id
          (assoc cached-pitch :endpoint-id endpoint-id)
          cached-pitch))
      ;; Cache miss: create new object and cache canonical version
      (let [new-pitch (->Pitch note duration attachments nil)
            final-pitch (if endpoint-id
                         (assoc new-pitch :endpoint-id endpoint-id)
                         new-pitch)]
        (swap! cache-misses inc)
        (swap! global-pitch-cache cache/miss cache-key new-pitch)
        final-pitch))))
```

#### Cache Key Design Principles

**Exclude Instance-Specific Fields:**
- **endpoint-ids** - Nearly always unique, would prevent all cache hits
- **temporary metadata** - Creation timestamps, debug info, etc.
- **runtime state** - Any mutable or session-specific data

**Include Core Musical Identity:**
- **note/duration/attachments** for pitches
- **duration/attachments** for rests
- **type/value/parameters** for attachments (normalized)

### Layer 2: Cache-First Modification Strategy

Instead of traditional "modify existing object → create new instance" pattern:

```clojure
(defn add-attachment [musical-obj attachment]
  ;; Traditional: modify existing, create new instance
  ;; (assoc musical-obj :attachments (conj (:attachments musical-obj) attachment))

  ;; Cache-first: look up desired end result
  (let [new-attachments (conj (:attachments musical-obj) attachment)
        endpoint-id (:endpoint-id musical-obj)]
    (case (type musical-obj)
      Pitch (create-pitch :note (:note musical-obj)
                          :duration (:duration musical-obj)
                          :attachments new-attachments
                          :endpoint-id endpoint-id)  ; Will likely hit cache
      Rest (create-rest :duration (:duration musical-obj)
                        :attachments new-attachments
                        :endpoint-id endpoint-id))))

(defn transpose-pitch [pitch semitones]
  ;; Cache-first transposition
  (let [new-note (transpose-note (:note pitch) semitones)]
    (create-pitch :note new-note
                  :duration (:duration pitch)
                  :attachments (:attachments pitch)
                  :endpoint-id (:endpoint-id pitch))))  ; Cache lookup for result
```

### Layer 3: Nippy-Integrated Cache-Aware Serialization

#### Custom Freeze/Thaw Transforms

```clojure
(require '[taoensso.nippy :as nippy])

;; Serialization session state
(def ^:dynamic *serialization-cache-registry* nil)
(def ^:dynamic *deserialization-cache-registry* nil)

(nippy/extend-freeze
  ooloi.shared.models.musical.pitch.Pitch
  :ooloi/cached-pitch
  [pitch data-output-stream]
  (let [cache-key (get-cache-key pitch)
        registry @*serialization-cache-registry*]
    (if-let [cache-id (get registry cache-key)]
      ;; Object already serialized in this session - write reference
      (do
        (.writeByte data-output-stream 1)  ; Reference marker
        (.writeInt data-output-stream cache-id))
      ;; First occurrence - write full object and register
      (let [cache-id (count registry)]
        (swap! *serialization-cache-registry* assoc cache-key cache-id)
        (.writeByte data-output-stream 0)  ; Full object marker
        (.writeInt data-output-stream cache-id)
        (nippy/freeze-to-out! data-output-stream
          {:note (:note pitch)
           :duration (:duration pitch)
           :attachments (:attachments pitch)})))))

(nippy/extend-thaw
  :ooloi/cached-pitch
  [data-input-stream]
  (let [is-reference (.readByte data-input-stream)
        cache-id (.readInt data-input-stream)]
    (if (= 1 is-reference)
      ;; Reference - look up in deserialization registry
      (get @*deserialization-cache-registry* cache-id)
      ;; Full object - deserialize and register
      (let [data (nippy/thaw-from-in! data-input-stream)
            pitch (create-pitch :note (:note data)
                               :duration (:duration data)
                               :attachments (:attachments data))]
        (swap! *deserialization-cache-registry* assoc cache-id pitch)
        pitch))))
```

#### Serialization Session Management

```clojure
(defmacro with-cache-session [& body]
  `(binding [*serialization-cache-registry* (atom {})
             *deserialization-cache-registry* (atom {})]
     ~@body))

(defn serialize-piece-with-cache [piece]
  (with-cache-session
    (nippy/freeze piece {:references? true
                         :compressor nippy/lz4-compressor})))

(defn deserialize-piece-with-cache [frozen-data]
  (with-cache-session
    (let [result (nippy/thaw frozen-data {:references? true})]
      ;; Ensure all objects are in global cache after deserialization
      (ensure-objects-in-global-cache result)
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
(defn cache-optimization-task []
  "Background daemon modeled after JVM string deduplication"
  (go-loop [piece-queue (pm/get-all-piece-ids)]
    (let [batch-size (min 10 (count piece-queue))
          batch (take batch-size piece-queue)
          remaining (drop batch-size piece-queue)]

      ;; Process batch
      (doseq [piece-id batch]
        (optimize-piece-cache piece-id))

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
(defn optimize-piece-cache [piece-id]
  "Replace equal objects with identical cached objects"
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
                  (let [cache-key (get-cache-key obj)]
                    (if-let [cached-obj (get-from-global-cache cache-key)]
                      (if (identical? obj cached-obj)
                        obj  ; Already using cached object
                        (do
                          (swap! optimization-count inc)
                          cached-obj))  ; Replace with cached instance
                      (do
                        ;; Add to cache for future consolidation
                        (add-to-global-cache cache-key obj)
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
    ;; Skip objects with endpoint-ids (poor cache candidates)
    (nil? (:endpoint-id obj))
    ;; Skip recently accessed objects (optimization overhead not worth it)
    (not (recently-accessed? obj))))
```

## Implementation Strategy

### Phase 1: Constructor Caching Foundation
1. Implement global caches with LRU eviction
2. Enhance `create-pitch` and `create-rest` with caching
3. Cache key normalization excluding endpoint-ids
4. Monitoring and statistics infrastructure
5. Target: >95% cache hit rates for common objects

### Phase 2: Nippy Integration
1. Custom freeze/thaw transforms for cached object types
2. Serialization session management for cache references
3. Roundtrip testing to ensure shared structure preservation
4. Target: >90% reduction in serialized file sizes

### Phase 3: Cache-First Operations
1. Implement cache-first `add-attachment`, `transpose-pitch`
2. Extend pattern to other modification operations
3. Target: Eliminate unnecessary object creation in modification chains

### Phase 4: Background Optimization
1. JVM-style background optimization daemon
2. Memory pressure-aware scheduling
3. Incremental batch processing
4. Target: Continuous consolidation of equal objects across all pieces

### Phase 5: Full System Integration
1. Chord caching with normalized pitch collections
2. Common attachment pre-population (cache warming)
3. Integration with server-side scratchpad (Issues #19/#20)
4. Performance profiling and optimization

## Cache Key Normalization

### Attachment Normalization
```clojure
(defn normalize-attachments [attachments]
  "Canonical ordering and normalization for better cache hits"
  (->> (or attachments [])
       (map normalize-single-attachment)
       (sort-by attachment-sort-key)
       vec))

(defn normalize-single-attachment [attachment]
  "Remove instance-specific fields from attachment for caching"
  (dissoc attachment :endpoint-id :creation-time :session-id))
```

### Pitch Normalization
```clojure
(defn get-pitch-cache-key [note duration attachments]
  {:note (normalize-note note)
   :duration (normalize-duration duration)
   :attachments (normalize-attachments attachments)})
   ;; NEVER include :endpoint-id in cache keys
```

## Monitoring and Observability

### Cache Performance Metrics
```clojure
(defn cache-performance-report []
  {:cache-statistics
   {:pitch-cache {:size (cache/size @global-pitch-cache)
                  :hit-rate (/ @pitch-hits (+ @pitch-hits @pitch-misses))
                  :evictions @pitch-evictions}
    :rest-cache {:size (cache/size @global-rest-cache)
                 :hit-rate (/ @rest-hits (+ @rest-hits @rest-misses))
                 :evictions @rest-evictions}}

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
    :total-cache-hits @lifetime-cache-hits
    :memory-savings-percent (/ @lifetime-cache-hits @lifetime-objects-created)}})
```

### Performance Alerting
```clojure
(defn check-cache-health []
  (let [stats (cache-performance-report)]
    (when (< (get-in stats [:cache-statistics :pitch-cache :hit-rate]) 0.90)
      (alert/warn "Pitch cache hit rate below 90%" stats))
    (when (> (get-in stats [:cache-statistics :pitch-cache :size]) 45000)
      (alert/info "Pitch cache approaching capacity" stats))
    (when (< (get-in stats [:serialization-efficiency :avg-file-size-reduction-percent]) 80)
      (alert/warn "Serialization efficiency degraded" stats))))
```

## Integration Points

### Server-Side Scratchpad Synergy
Cache amplifies benefits of server-side scratchpad from Issues #19/#20:

```clojure
(SRV/atomic [
  ;; Objects created on server - immediate cache benefits
  [:let $quarter-rest (create-rest :duration 1/4)]    ; Cache hit likely
  [:let $c4 (create-pitch :note "C4" :duration 1/4)]  ; Cache hit likely

  ;; Variable reuse amplified by caching
  (add-item [:musicians 0 :instruments 0 :staves 0 :voices 0 :measures 0] "piece" $c4)
  (add-item [:musicians 1 :instruments 0 :staves 0 :voices 0 :measures 0] "piece" $c4)
  ;; Same cached instance used across multiple placements - zero allocation overhead
])
```

### Piece Manager Integration
- Automatic background optimization of all managed pieces
- Cache-aware save/load using enhanced Nippy serialization
- Memory usage monitoring and reporting
- Cache warming during piece manager startup

## Risk Mitigation

### Cache Invalidation
**Risk**: Cached objects become stale or inconsistent
**Mitigation**: Objects are immutable - no invalidation needed

### Memory Leaks
**Risk**: Unbounded cache growth
**Mitigation**: LRU eviction with configurable thresholds

### Performance Degradation
**Risk**: Cache lookup overhead exceeds benefits
**Mitigation**:
- Only cache high-repetition object types
- Benchmark cache performance continuously
- Fallback to direct creation if cache becomes ineffective

### Serialization Compatibility
**Risk**: Nippy changes break cache serialization
**Mitigation**:
- Version serialization format
- Graceful degradation to standard serialization
- Comprehensive roundtrip testing

## Success Metrics

### Memory Efficiency
- **Target**: 99%+ memory reduction for redundant musical objects
- **Measurement**: Compare total object instances vs unique cache entries
- **Baseline**: Large symphony pieces for realistic workload testing

### Cache Performance
- **Target**: >95% hit rates for pitch/rest objects when endpoint-ids excluded
- **Target**: >80% hit rates for attachment objects
- **Measurement**: Cache hit/miss ratios with detailed breakdown by object type

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
- **Comprehensive testing**: Extensive test coverage for all cache scenarios
- **Monitoring infrastructure**: Detailed metrics and alerting for cache performance
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