# Timewalk Performance Benchmarks

Performance benchmarks for the Ooloi timewalk traversal system using Criterium for statistically valid measurements.

## Test Hardware

**Apple M3, MacBook Air 15-inch (2024)**: the main figures. Ooloi's Mac platform is Apple Silicon ([ADR-0050](https://github.com/PeterBengtson/Ooloi-docs/blob/main/ADRs/0050-Platform-Support-Policy.md)).
- **Processor**: Apple M3, four performance and four efficiency cores
- **Memory**: 16 GB
- **macOS**: 27.2.1

**Intel Core i7-8750H, MacBook Pro 15-inch (2018)**: for comparison only. Ooloi does not support Intel Macs.
- **Processor**: 2.2 GHz, six cores, twelve hardware threads
- **Memory**: 16 GB
- **macOS**: 15.8.3

Both run OpenJDK 25 with the JVM options in `shared/project.clj`, on the same code, on mains power with Low Power Mode off. Each machine ran both suites three times on the code measured here; every figure below is the median of those three runs, `run13` to `run15` under the MacBook Pro's name and `run11` to `run13` under the M3's in `shared/dev/bench/results/`, where each run's output is kept.

## Benchmark Results Summary

All tests use a 1000-measure orchestral piece (29 staves, ~520,000 pitches) - comparable in size and density to Wagner's *Götterdämmerung* or Strauss's *Elektra*. This represents an extreme test case significantly larger and denser than typical musical scores.

### Traversal Scopes in Real Use

These benchmarks measure Ooloi's **timewalk traversal engine** — the substrate that powers cache refreshes, endpoint searches (ties/slurs), and local analyses. Editing and scrolling use **cached** data; timewalk is invoked to **prepare/repair** those caches when structure changes.

For each scope we report two forms:

- **Unrealised (streaming)**: we *reduce* over the traversal and never allocate result tuples. This is **constant-memory** and typical for cache updates and searches.
- **Materialised (collected)**: we *collect* `[item vpd position]` tuples into a vector. This allocates proportionally and is used intentionally for operations that need random access.

| Scope | Typical Use | M3, unrealised | M3, materialised | i7-8750H, unrealised | i7-8750H, materialised |
|------|--------------|---------------:|-----------------:|---------------------:|-----------------------:|
| 10 measures × 1 stave | Cache refresh after edit | 310 µs | 313 µs | 722 µs | 727 µs |
| 10 measures × 4 staves | Compound cache rebuild | 1.25 ms | 1.26 ms | 2.93 ms | 2.98 ms |
| 1 instrument × full piece | Part extraction / analysis | 6.49 ms | 6.85 ms | 13.7 ms | 14.8 ms |
| 50 measures × 1 stave | Search endpoint (worst case) | 517 µs | N/A | 1.22 ms | N/A |

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

| Operation | Scope | M3 | i7-8750H | Memory | Notes |
|-----------|-------|---:|---------:|--------|-------|
| **Streaming export** | Full piece | 319 ms | 522 ms | <10 MB | Constant memory, no materialisation |
| **Full traversal** | Full piece | 293 ms | 484 ms | ~244 MB | With materialisation |
| **Generate piece** | Full piece | 1.81 s | 3.61 s | - | Creating structure from scratch |
| **Save to disk** | Full piece | 504 ms | 827 ms | - | Registry-first Nippy serialisation with LZ4 compression, on a thread of its own |
| **Load from disk** | Full piece | 68.7 ms | 150 ms | - | Deserialisation + verification |
| **Write** | 10 / 100 / 1000 measures | 7.4 ms / 59 ms / 511 ms | 12.7 ms / 103 ms / 822 ms | - | `persistence/file-writer`, the write a save performs |
| **Read** | 10 / 100 / 1000 measures | 1.0 ms / 7.2 ms / 68.5 ms | 2.0 ms / 13.5 ms / 144 ms | - | `persistence/file-reader`, the read an open performs |
| **File size** | Full piece | 0.05 MB | 0.05 MB | - | 0.09 bytes an item, registry-first and LZ4-compressed |

**Key insight**: The streaming export (319 ms) uses constant memory (<10 MB), against the full traversal with materialisation (293 ms, ~244 MB). For MIDI/MusicXML export, the system maintains constant memory regardless of piece size.

**Persistence notes**:
- File size does NOT include visual formatting hierarchy yet (will likely double when added)
- Even at 2× size (~0.1 MB), file sizes remain negligible for modern systems
- Igor Engraver was notoriously slow at saving/loading - Ooloi's persistence eliminates this bottleneck
- Round-trip fidelity: ✓ Perfect (original = loaded after deserialization)

## Performance Assessment

These results demonstrate **optimal performance characteristics** for a functional music notation system:

**Traversal substrate performance**: On the M3, the traversal windows that feed cache updates and searches complete in 0.3 to 1.3 ms for unrealised (streaming) operations, 0.7 to 3 ms on the i7-8750H. Editing and scrolling themselves run on **cached data**; traversal occurs only when dependencies must be recomputed. The difference between unrealised and materialised for small scopes is **negligible**, validating that the transducer architecture delivers genuine zero-intermediate-allocation. The cost you pay for materialisation is purely the final vector allocation, not intermediate lazy sequence overhead. Endpoint searches (slur/tie resolution) complete in 0.5 ms even for the worst case (50 measures ahead with no match); real slurs typically end within 1-2 measures, making actual searches far shorter.

**Memory discipline**: The contrast between streaming export (319 ms, <10 MB constant memory) and full traversal with materialisation (293 ms, ~244 MB) validates the architectural design. For MIDI/MusicXML export, the system maintains **constant memory regardless of piece size** - this is critical for production systems handling large scores. Cache refresh operations use constant memory via streaming reduction; materialisation is intentionally used only when random access is needed. The negligible materialisation overhead means you can freely materialise small scopes without performance concerns.

**Batch traversal**: Under a third of a second to traverse over half a million musical elements with temporal coordination guarantees is **exceptionally fast**: about 1.8 million pitches a second on the M3, and 1.1 million on the i7-8750H. The push-based transducer architecture delivers on its zero-intermediate-allocation promise - we only allocate the result tuples themselves, which is unavoidable.

**File persistence**: Each distinct shared object is written once, in a registry ahead of the piece, which refers to it, and LZ4 compresses the whole: the 1000-measure piece freezes to 6.7 MB and compresses to 0.05 MB, 0.09 bytes an item. The generated piece repeats far more than a real score, so a real score's file is larger. A save takes half a second and a load 69 ms on the M3, eliminating the notorious slowness of systems like Igor Engraver: users experience **near-instant file operations** even for extreme scores.

**Generation speed**: Creating 520,000 pitches from scratch in under two seconds demonstrates that **piece construction is not a bottleneck**.

**Overall**: The system is **fast enough** that performance concerns disappear from user consciousness - operations complete before users notice. File sizes are **small enough** to be irrelevant in modern contexts. The architecture is **efficient enough** to handle extreme scores without compromise.

### Apple Silicon against Intel

On this code the M3 is 1.6 to 2.4 times as fast as the i7-8750H, comparing the medians of each machine's three runs, and Geekbench 6 rates the M3's single core 2.49 times the i7-8750H's. The timewalk scopes gain 2.1 to 2.4 times. Two kinds of result gain less:

- **The two whole-piece traversals**: the streaming export 1.64 times and the materialised traversal 1.65. The export runs in constant memory, so allocation does not account for it alone; the cause has not been measured.
- **Persistence**: a write 1.6 to 1.8 times, a read 1.9 to 2.2, and generating the piece 2.0.

The runs agree within eleven per cent on every figure but two: the M3's round-trip save, 432 to 548 ms, and the i7-8750H's 1000-measure read, 119 to 154 ms.

Each run's full output is in `shared/dev/bench/results/`, one directory per run and machine.

## Overview

These benchmarks validate the timewalk's push-based transducer design, which replaced a lazy sequence wrapper with:

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
- `:voices-per-staff` - Voices per stave (default: 1)
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

Both suites, from the repository root, with their results and the circumstances of the run written to one directory:

```bash
shared/dev/bench/run_suites.sh <output directory>
```

The script runs the timewalk suite and then the persistence suite, each in a fresh JVM with the options in `shared/project.clj`, and writes:

| File | Contents |
| --- | --- |
| `timewalk.log`, `persistence.log` | Each suite's output |
| `machine.log` | Processor, model, core and thread counts, memory, OS, Java version, the commit and any uncommitted changes, and the JVM options |
| `state.log` | Power source, Low Power Mode and thermal state before and after each suite (macOS) |
| `exits.log` | Each suite's exit code, then `ALL DONE` |

On macOS the script keeps the machine awake for the whole run. The suites take about ten minutes on an M3; nothing else should run on the machine meanwhile, and on a laptop it should be on mains power with Low Power Mode off.

**Comparing machines.** Results compare only when they come from the same commit and the same JVM options, which `machine.log` records. The timewalk suite's host line gives the operating system, architecture, processor count and maximum heap as the JVM sees them; `machine.log` names the processor and the machine. To record a machine's results for comparison:

1. Pull. Runs compare while the code they measure is the same between the commits their `machine.log` files record: `git diff --stat <commit> <commit> -- shared/src shared/dev shared/project.clj` lists nothing but this README, `run_suites.sh` and `results/`.
2. Run the script twice into `shared/dev/bench/results/`, under a name for the machine, its model, processor and year, and the run:

   ```bash
   shared/dev/bench/run_suites.sh shared/dev/bench/results/macbook-pro-15-i7-8750h-2018-run1
   shared/dev/bench/run_suites.sh shared/dev/bench/results/macbook-pro-15-i7-8750h-2018-run2
   ```

   One run is not enough. The timewalk suite's scopes take microseconds to milliseconds, and activity on the machine during a run moves them by a third or more while leaving no trace in `state.log`. Where two runs disagree by more than a tenth on any scope, a third is made.
3. Check each run's `exits.log`: both suites `EXIT=0`, then `ALL DONE`. A run whose `state.log` shows battery power, Low Power Mode or a thermal warning is run again.
4. Commit the directories, five files each, and push.

Each directory under `results/` is one run on one machine, and the directories side by side are the comparison.

From a REPL, one suite at a time:

```bash
cd shared
lein repl
```

```clojure
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

Based on the transducer design's goals:

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
