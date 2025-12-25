# Timewalk Performance Benchmarks

Performance benchmarks for the Ooloi timewalk traversal system using Criterium for statistically valid measurements.

## Test Hardware

**MacBook Pro (2017) - Baseline**
- **Processor**: 2.2 GHz 6-Core Intel Core i7
- **Memory**: 16 GB 2400 MHz DDR4
- **Graphics**: Intel UHD Graphics 630 1536 MB
- **macOS**: Sequoia 15.6.1

**MacBook Air (2024) - M3 Comparison**
- **Processor**: Apple M3 chip
- **Memory**: 16 GB
- **macOS**: Sonoma 14.x

Original benchmarks run on 2017 hardware establish baseline. M3 benchmarks demonstrate 2-3× linear scaling improvements with newer hardware.

## Benchmark Results Summary

All tests use a 1000-measure orchestral piece (29 staves, ~520,000 pitches) - comparable in size and density to Wagner's *Götterdämmerung* or Strauss's *Elektra*. This represents an extreme test case significantly larger and denser than typical musical scores.

### Traversal Scopes in Real Use

These benchmarks measure Ooloi's **timewalk traversal engine** — the substrate that powers cache refreshes, endpoint searches (ties/slurs), and local analyses. Editing and scrolling use **cached** data; timewalk is invoked to **prepare/repair** those caches when structure changes.

For each scope we report two forms:

- **Unrealised (streaming)**: we *reduce* over the traversal and never allocate result tuples. This is **constant-memory** and typical for cache updates and searches.
- **Materialised (collected)**: we *collect* `[item vpd position]` tuples into a vector. This allocates proportionally and is used intentionally for operations that need random access.

**2017 MacBook Pro (Baseline):**

| Scope | Typical Use | Unrealised Mean | Materialised Mean | Notes |
|------|--------------|-----------------|-------------------|-------|
| 10 measures × 1 staff | Cache refresh after edit | ~1.0 ms | ~1.0 ms | Minimal overhead for materialisation |
| 10 measures × 4 staves | Compound cache rebuild | ~4.4 ms | ~4.4 ms | Proportional to number of staves |
| 1 instrument × full piece | Part extraction / analysis | ~35 ms | ~35 ms | Noise within measurement variance |
| 50 measures × 1 staff | Search endpoint (worst case) | ~2.3 ms | N/A | Real slurs end within 1-2 measures |

**2024 M3 MacBook Air (2-3× improvement):**

| Scope | Typical Use | Unrealised Mean | Materialised Mean | Notes |
|------|--------------|-----------------|-------------------|-------|
| 10 measures × 1 staff | Cache refresh after edit | ~360 µs | ~360 µs | 2.8× faster than baseline |
| 10 measures × 4 staves | Compound cache rebuild | ~1.49 ms | ~1.49 ms | 2.95× faster than baseline |
| 1 instrument × full piece | Part extraction / analysis | ~12.9 ms | ~12.9 ms | 2.7× faster than baseline |
| 50 measures × 1 staff | Search endpoint (worst case) | ~788 µs | N/A | 2.9× faster than baseline |
| Typical slur search (1-2 measures) | Common endpoint search | <50 µs | N/A | 2× faster than baseline (<100 µs) |

#### Materialisation vs Streaming

Timewalk yields a stream of **three-element tuples**:

```
[item vpd position]
```

- `item` — the musical object
- `vpd` — its Vector Path Descriptor (stable address in the hierarchy)
- `position` — its temporal position

There are two ways to consume this stream:

1) **Streaming / unrealised**
   We *reduce* over the stream (e.g., counting, aggregating, writing export events).
   No vectors of tuples are created; memory usage remains **constant** (a few MB).

   ```clojure
   (transduce xf rf 0 (timewalk piece opts)) ;; reduce-only, constant memory
   ```

2) **Materialised / collected**
   We *collect* tuples into a vector for **random access** during analysis.

   ```clojure
   (into [] xf (timewalk piece opts)) ;; allocates ~437 bytes per tuple (baseline)
   ```

**Guideline:** Use **streaming** for cache refresh, searches, and exports. Use **materialisation** only when you need random access to tuples in that scope.

### Batch Operations (rare, but important)

**2017 MacBook Pro (Baseline):**

| Operation | Scope | Mean Time | Memory | Notes |
|-----------|-------|-----------|--------|-------|
| **Streaming export** | Full piece | ~1.25 sec | <10 MB | Constant memory, no materialization |
| **Full traversal** | Full piece | ~582 ms | ~244 MB | With materialization (baseline) |
| **Generate piece** | Full piece | ~1.0 sec | - | Creating structure from scratch |
| **Save to disk** | Full piece | ~1.3 sec | - | Async with Nippy serialization |
| **Load from disk** | Full piece | ~3.3 sec | - | Deserialization + verification |
| **File size** | Full piece | 0.17 MB | - | 0.35 bytes/pitch with hash-consing |

**2024 M3 MacBook Air (2-3× improvement):**

| Operation | Scope | Mean Time | Memory | Notes |
|-----------|-------|-----------|--------|-------|
| **Streaming export** | Full piece | ~592 ms | <10 MB | 2.1× faster than baseline |
| **Full traversal** | Full piece | ~316 ms | ~244 MB | 1.84× faster than baseline |

**Key insight**: The streaming export (~1.27 sec) uses constant memory (<10 MB) vs full traversal with materialization (528 ms, 244 MB). For MIDI/MusicXML export, the system maintains constant memory regardless of piece size.

**Persistence notes**:
- File size does NOT include visual formatting hierarchy yet (will likely double when added)
- Even at 2× size (~0.34 MB), file sizes remain negligible for modern systems
- Igor Engraver was notoriously slow at saving/loading - Ooloi's persistence eliminates this bottleneck
- Round-trip fidelity: ✓ Perfect (original = loaded after deserialization)

## Performance Assessment

These results demonstrate **optimal performance characteristics** for a functional music notation system:

**Traversal substrate performance**: Typical **traversal windows** that feed cache updates and searches complete in 1–5 ms for unrealised (streaming) operations. Editing and scrolling themselves run on **cached data**; traversal occurs only when dependencies must be recomputed. The difference between unrealised and materialised for small scopes is **negligible**, validating that the transducer architecture delivers genuine zero-intermediate-allocation. The cost you pay for materialisation is purely the final vector allocation, not intermediate lazy sequence overhead. Endpoint searches (slur/tie resolution) complete in ~2.3ms even for the worst case (50 measures ahead with no match) - real slurs typically end within 1-2 measures, making actual searches sub-millisecond.

**Memory discipline**: The contrast between streaming export (~1.25 sec, <10 MB constant memory) and full traversal with materialization (~582 ms, ~244 MB) validates the architectural design. For MIDI/MusicXML export, the system maintains **constant memory regardless of piece size** - this is critical for production systems handling large scores. Cache refresh operations use constant memory via streaming reduction; materialisation is intentionally used only when random access is needed. The negligible materialisation overhead means you can freely materialize small scopes without performance concerns.

**Batch traversal**: Half a second to traverse over half a million musical elements with temporal coordination guarantees is **exceptionally fast**. The push-based transducer architecture delivers on its zero-intermediate-allocation promise - we only allocate the result tuples themselves, which is unavoidable. The 1M+ pitches/second traversal rate means even the most complex orchestral scores feel instantaneous to users.

**File persistence**: At 0.35 bytes per pitch, Ooloi achieves **remarkable compression** through hash-consing. A massive *Götterdämmerung*-scale work compresses to 172KB - smaller than a typical email. The 1.3 second save time eliminates the notorious slowness of systems like Igor Engraver. Combined with the 3.3 second load time, users experience **near-instant file operations** even for extreme scores. This is production-ready performance.

**Generation speed**: Creating 520,000 pitches from scratch in one second demonstrates that **piece construction is not a bottleneck**. The system can generate complex musical structures faster than users can think about them.

**Overall**: These benchmarks validate that the Phase 6.9 optimization delivered production-ready performance. The system is **fast enough** that performance concerns disappear from user consciousness - operations complete before users notice. File sizes are **small enough** to be irrelevant in modern contexts. The architecture is **efficient enough** to handle extreme scores without compromise. This is exactly where a professional notation system should be.

## Overview

These benchmarks validate the Phase 6.9 timewalk optimization, which refactored from lazy sequence wrapper to true push-based transducer with:

- **Zero intermediate allocation** - No lazy sequence nodes, bare arguments throughout
- **True early termination** - Processing stops immediately on `reduced`
- **Single-pass position calculation** - Position accumulated in loop variables
- **5-10× performance improvement** - For early termination scenarios
- **70-90% allocation reduction** - Through bare argument passing

## Files

### `large_piece_generator.clj`

Generates realistic large orchestral pieces for performance testing.

**Main function:**
```clojure
(generate-orchestral-piece options)
```

**Options:**
- `:measures` - Number of measures per instrument (default: 1000)
- `:instruments` - Number of instruments (default: 20)
- `:staves-per-instrument` - Staves per instrument (default: 1)
- `:voices-per-staff` - Voices per staff (default: 1)
- `:items-per-measure` - Musical items per measure (default: 8)

**Example usage:**
```clojure
(require '[bench.large-piece-generator :as gen])

;; Large orchestral score
(def piece (gen/generate-orchestral-piece {:measures 1000}))

;; Verify structure
(gen/piece-stats piece)
;; => {:musicians 1, :instruments 20, :staves 20, :measures 20000, :items 160000}
```

### `timewalk_bench.clj`

Criterium-based benchmarks testing various timewalk scenarios.

**Available benchmarks:**

| Function | Tests | Expected Performance |
|----------|-------|---------------------|
| `bench-full-traversal` | Complete piece traversal | Baseline for comparison |
| `bench-early-termination` | `take 100` from 1000 measures | 5-10× faster vs lazy |
| `bench-filtered-search` | Filter pitches across piece | 2-4× faster vs lazy |
| `bench-bounded-scope` | Single instrument traversal | Proportional reduction |
| `bench-measure-range` | Measure 100-200 filtering | Efficient range limiting |

**Example usage:**
```clojure
(require '[bench.timewalk-bench :as bench])

;; Run individual benchmark
(bench/bench-full-traversal)

;; Run complete suite (~5-10 minutes)
(bench/run-all-benchmarks)
```

## Quick Start

```bash
cd /Users/pjotr/ProjectCode/FrankenScore/Ooloi/shared
lein repl
```

```clojure
;; Load and run benchmarks
(require '[bench.timewalk-bench :as bench])
(bench/run-all-benchmarks)
```

## Understanding Criterium Output

Criterium provides statistical analysis of performance:

```
Evaluation count : 6000 in 60 samples of 100 calls.
Execution time mean : 10.234 ms
Execution time std-deviation : 0.456 ms
Execution time lower quantile : 9.876 ms (2.5%)
Execution time upper quantile : 11.123 ms (97.5%)
Overhead used : 2.123 ns
```

**Key metrics:**
- **Mean execution time**: Average performance across all samples
- **Standard deviation**: Performance variance (lower is more consistent)
- **Quantiles**: 95% of measurements fall within this range
- **Overhead**: JVM/measurement overhead (negligible for meaningful benchmarks)

## Benchmark Design Principles

### Statistical Validity

All benchmarks use Criterium's `quick-bench` (60-second sampling) to ensure:
1. **JVM warmup** - Code optimized by JIT compiler before measurement
2. **Multiple iterations** - Thousands of executions for statistical validity
3. **GC isolation** - Garbage collection between samples prevents interference
4. **Outlier detection** - Identifies and reports anomalous results

### Realistic Test Data

The `large_piece_generator` creates realistic musical structures:
- **Varied note durations** - Tests position calculation under realistic conditions
- **Multi-instrument scores** - Stresses temporal coordination
- **Configurable complexity** - From simple tests to 1000-measure orchestral scores

### Performance Claims Validation

Each benchmark validates specific optimization claims:

1. **Full Traversal** - Baseline showing zero-allocation efficiency
2. **Early Termination** - Validates true computation stopping (not just sequence stopping)
3. **Filtered Search** - Shows transducer composition maintains performance
4. **Bounded Scope** - Confirms scope limiting reduces work proportionally
5. **Measure Range** - Tests efficient measure-level filtering

## Expected Results

Based on Phase 6.9 optimization goals:

| Scenario | Expected Improvement | Why |
|----------|---------------------|-----|
| Full traversal | 2-5× faster | Zero lazy sequence overhead |
| Early termination | 5-10× faster | True computation stopping |
| Filtered search | 2-4× faster | Single-pass with filter transducer |
| Bounded scope | Proportional | Only processes specified scope |

## When to Run Benchmarks

### Before Committing Performance Changes

Always benchmark before claiming performance improvements:
```bash
# Run benchmarks before and after changes
# Compare mean execution times
```

### After Infrastructure Changes

Run benchmarks after changes that might affect timewalk:
- VPD system modifications
- Measure/voice/item structure changes
- Protocol/multimethod dispatch changes

### Regression Detection

Run periodically to detect performance regressions:
```bash
# Baseline: record current performance
# After changes: compare against baseline
```

## Profiling

For deeper performance analysis beyond Criterium:

```clojure
;; Use JVisualVM or YourKit profiler
;; Run benchmark under profiler to identify hotspots
(bench/bench-full-traversal)
```

**Common hotspots to investigate:**
- Ratio arithmetic (consider caching common fractions)
- VPD conj operations (check for reflection warnings)
- Protocol dispatch (verify no reflection)

## Adding New Benchmarks

When adding new benchmarks:

1. **Use Criterium** - Always use `quick-bench` or `bench` for statistical validity
2. **Document expectations** - State what you're testing and expected results
3. **Use generator** - Create test data with `large-piece-generator`
4. **Follow naming** - Use `bench-<scenario>` naming convention
5. **Add to suite** - Include in `run-all-benchmarks` if appropriate

**Example template:**
```clojure
(defn bench-your-scenario
  "Benchmark description.

  Tests the specific optimization claim or performance characteristic."
  []
  (println "\n" (apply str (repeat 60 "=")))
  (println "BENCHMARK: Your Scenario")
  (println (apply str (repeat 60 "=")))

  (let [piece (gen/generate-orchestral-piece {:measures 1000})]
    (println "\nRunning benchmark...")
    (crit/quick-bench
      ;; Your benchmark code
      )))
```

## Related Documentation

- **[ADR-0014: Timewalk](https://github.com/PeterBengtson/Ooloi-docs/blob/main/ADRs/0014-Timewalk.md)** - Architecture and implementation
- **[TIMEWALKING_GUIDE.md](https://github.com/PeterBengtson/Ooloi-docs/blob/main/guides/TIMEWALKING_GUIDE.md)** - Practical usage guide
