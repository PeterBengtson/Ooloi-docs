# 🔴 Advanced Concurrency Patterns: Parallel Processing with STM

**Warning: This guide covers advanced concurrent programming patterns. Only use these techniques when you have significant performance requirements and understand Clojure's STM system.**

## Prerequisites

- **Solid understanding of timewalker**: Complete the [Timewalking Guide](TIMEWALKING_GUIDE.md) first
- **Basic Clojure knowledge**: Comfortable with threading macros, `map`, `filter`, etc.
- **Performance requirements**: These patterns are for large orchestral works (1000+ measures)

**STM concepts will be explained** - no prior experience with Software Transactional Memory required.

## Understanding Software Transactional Memory (STM)

Before diving into parallel processing patterns, let's understand the foundation that makes safe concurrent programming possible in Ooloi. This guide is the *how*; the decision itself is [ADR-0004: STM for Concurrency](../ADRs/0004-STM-for-concurrency.md), and the per-resource choice between a plain atom and an STM ref is [ADR-0045 §"Why atom, not STM ref"](../ADRs/0045-Instrument-Library.md).

### The Concurrency Problem

When multiple threads try to modify the same data simultaneously, you get **race conditions**:

```clojure
;; BROKEN: Multiple threads updating an atom concurrently
(def piece-data (atom {:measures {}}))

;; Thread A and Thread B both try to update measure 5 at the same time
(swap! piece-data assoc-in [:measures 5] measure-a)  ; Thread A
(swap! piece-data assoc-in [:measures 5] measure-b)  ; Thread B
;; Result: One update overwrites the other - data is lost!
```

Traditional solutions involve locks, which are complex, error-prone, and cause deadlocks.

### STM: A Better Solution

**Software Transactional Memory (STM)** treats memory updates like database transactions - they're **atomic**, **consistent**, **isolated**, and **durable** (ACID properties).

#### Key STM Concepts

**1. Refs - Coordinated References**
```clojure
(def piece-ref (ref {:measures {}}))  ; Create a managed reference
```

**2. Transactions - Coordinated Updates**
```clojure
(dosync  ; Start a transaction
  (alter piece-ref assoc-in [:measures 5] new-measure))  ; Update within transaction
```

**3. Automatic Conflict Resolution**
```clojure
;; Both threads start transactions simultaneously:
;; Thread A: (dosync (alter piece-ref assoc-in [:measures 5] measure-a))
;; Thread B: (dosync (alter piece-ref assoc-in [:measures 5] measure-b))
;;
;; STM detects the conflict:
;; - One thread succeeds immediately
;; - The other thread automatically retries with the updated data
;; - Eventually both updates are applied consistently
;; - No data loss, no manual coordination needed!
```

#### Why Atoms Don't Work for Coordination

**Atoms are for independent updates:**
```clojure
(def counter (atom 0))
(swap! counter inc)  ; Fine for single values
```

**But atoms can't coordinate multiple updates:**
```clojure
;; BROKEN: These two updates aren't coordinated
(swap! layout-atom assoc-in [:measures 5] measure-5)
(swap! layout-atom assoc-in [:systems 2] system-2)
;; If another thread updates between these calls, inconsistency results
```

**STM refs coordinate multiple updates:**
```clojure
;; CORRECT: These updates happen atomically together
(dosync
  (alter layout-ref assoc-in [:measures 5] measure-5)
  (alter layout-ref assoc-in [:systems 2] system-2))
;; Either both updates succeed together, or both are retried together
```

#### Transients Are Also Incompatible

**Transients are for single-threaded performance:**
```clojure
(defn process-large-vector [data]
  (persistent! 
    (reduce conj! (transient []) data)))  ; Fast single-threaded updates
```

**But transients break STM's guarantees:**
```clojure
;; BROKEN: Transients aren't thread-safe
(dosync
  (alter piece-ref 
    (fn [piece]
      (-> piece
          (transient)           ; WRONG: Not thread-safe!
          (assoc! :key :value)
          (persistent!)))))
```

**STM requires immutable operations:**
```clojure
;; CORRECT: Pure functions return new immutable values
(dosync
  (alter piece-ref assoc :key :value))  ; Returns new immutable map
```

### STM in Practice: Simple Example

```clojure
;; Create a piece reference - THE unified object containing both hierarchies
(def piece-ref (ref (create-piece :musicians [musician1 musician2]
                                  :layouts [layout1 layout2]
                                  :time-signatures (create-change-set)
                                  :key-signatures (create-change-set)
                                  :tempos (create-change-set))))

;; Coordinated update across both hierarchies - all part of the SAME piece
(dosync
  (alter piece-ref assoc-in [:musicians 0 :instruments 0 :staves 0 :measures 5] new-measure)
  (alter piece-ref assoc-in [:layouts 0 :page-views 0 :system-views 0 :staff-views 5] new-staff-view))
;; CRITICAL: Everything fails or succeeds together - there's only ONE piece-ref!

;; Automatic retry example - you don't write this code, STM does it:
;; 1. Transaction starts with current value of the piece ref
;; 2. If another thread changes the piece during the transaction
;; 3. STM detects the conflict and automatically retries
;; 4. Transaction runs again with the updated piece value
;; 5. Eventually succeeds when no conflicts occur
```

### Why STM is Perfect for Music Notation

Musical notation has **complex coordinated relationships within a single unified piece**:
- Measures must align across all staves (musical hierarchy)
- Layout changes affect spacing and page breaks (visual hierarchy)  
- Attachments (slurs, ties) connect elements across measures (cross-hierarchy)
- Format changes cascade from musical content to visual presentation

**Ooloi's unified piece architecture** means musical and visual updates must stay synchronized:
- Adding a measure affects both `:musicians` and `:layouts` hierarchies
- Time signature changes impact both musical content and layout formatting
- All relationships exist within ONE piece object - everything succeeds or fails together

STM ensures these complex inter-hierarchy relationships remain consistent during parallel processing.

### How Ooloi Simplifies STM

While STM provides the foundation, **Ooloi's API abstracts away most of the complexity**:

```clojure
;; You don't write this low-level STM code:
(dosync
  (alter piece-ref assoc-in [:musicians 0 :instruments 0 :staves 0 :measures 5] new-measure))

;; Instead, you use Ooloi's VPD-based API:
(api/set-measure [:musicians 0 :instruments 0 :staves 0] piece-id 5 new-measure)
;; The dosync/alter coordination happens automatically!
```

**Key advantages of Ooloi's abstraction:**
- **Automatic transactions**: Each VPD operation is automatically wrapped in `dosync`
- **Composable coordination**: A VPD operation's own transaction nests into an enclosing one, so several operations compose into one atomic change
- **Type safety**: Polymorphic multimethods ensure correct operation dispatch
- **Path validation**: VPDs are validated to prevent invalid references
- **Consistent API**: Same patterns work across all musical and visual elements

**Automatic Transaction Coordination:**
```clojure
;; These operations from different threads automatically coordinate:
;; Thread A:
(api/set-measure [:musicians 0 :instruments 0 :staves 0] piece-id 5 measure-a)

;; Thread B (concurrent):
(api/set-staff-view [:layouts 0 :page-views 0 :system-views 0] piece-id 0 staff-view-b)

;; STM automatically coordinates these updates - no manual synchronization needed!
```

**Composable Operations:**
```clojure
;; Results computed in parallel are written back one VPD operation each.
;; A setter's VPD addresses the parent; the index and the new item follow the piece-id.
(doseq [{:keys [staff-view-vpd index measure-view]} parallel-results]
  (api/set-measure-view staff-view-vpd piece-id index measure-view))
;; Each operation is automatically transactional and coordinates with others
```

This means you get **enterprise-grade STM coordination** with **simple, readable code**.

### 🚨 Critical Development Principle: Never Use `alter`

**Ooloi developers should NEVER need to use `alter`, `ref`, or other low-level STM operations.**

The VPD API provides complete abstraction over STM complexity:

```clojure
;; ❌ WRONG: Never drop to STM level
(let [piece-ref (pm/get-piece-ref piece-id)]   ; pm = ooloi.backend.ops.piece-manager
  (dosync
    (alter piece-ref assoc-in [:musicians 0 :instruments 0 :staves 0 :measures 5] new-measure)))

;; ✅ RIGHT: Use VPD abstraction
(api/set-measure [:musicians 0 :instruments 0 :staves 0] piece-id 5 new-measure)
```

**For atomic coordination of multiple operations:**
```clojure
(require '[ooloi.shared.ops.change-detection :as cd])

;; ✅ RIGHT: Compose VPD operations in one coalescing scope
(cd/with-coalesced-structural-change
  (api/set-measure [:musicians 0 :instruments 0 :staves 0] piece-id 5 measure-a)
  (api/set-staff-view [:layouts 0 :page-views 0 :system-views 0] piece-id 0 staff-view-a)
  (api/set-tempo [] piece-id position new-tempo))
;; One transaction: every operation succeeds or none does, and the structural
;; changes among them, if any, emit a single :piece-structure-changed on commit
```

`with-coalesced-structural-change` opens the transaction itself; each operation's own `dosync` nests into it. The same composition expressed as operation maps is `api/atomic`, which runs them inside this scope through the backend's `execute-atomic-operations`, and from a client it is `SRV/atomic`, which reaches that same function over gRPC.

A bare `dosync` around the same three calls still gives atomicity and isolation, but it is **not** a coalescing scope: it binds no gate, so each structural change inside it emits its own `:piece-structure-changed` ([ADR-0052 §4](../ADRs/0052-Change-Detection-and-Event-Generation.md#4-exactly-one-event-per-outermost-transaction)). Neither form records an undo step when called from backend code. Undo is captured on the client-facing `SRV/` boundary, one entry per lone call or batch; backend code that wants a step registers it deliberately ([ADR-0015](../ADRs/0015-Undo-and-Redo.md)).

**Why this matters:**
- **Abstraction integrity**: VPD operations handle validation, type checking, and coordination
- **Future-proof**: Implementation can change without breaking your code
- **Error handling**: VPD operations provide proper error messages and validation
- **Consistency**: All operations follow the same patterns and conventions
- **Maintenance**: No need to understand STM internals to write reliable concurrent code

**The only composition primitive developers need:** `with-coalesced-structural-change` (or `api/atomic`, given operation maps) to compose multiple VPD operations atomically.

When a composed transaction in that scope mutates piece structure, change detection emits a single `:piece-structure-changed` event for the whole transaction, not one per operation — see [ADR-0052: Change Detection and Event Generation](../ADRs/0052-Change-Detection-and-Event-Generation.md) for how agent-deferred emission and a transaction-scoped gate guarantee exactly-once-per-commit, even under retry.

## Overview

Ooloi's layout engine can format individual measures in parallel while coordinating updates to the visual hierarchy - a capability that gives it significant performance advantages over traditional single-threaded notation systems. Piece-walker's temporal coordination enables this safe parallelization within measure groups.

This guide demonstrates how to leverage Ooloi's sophisticated concurrency architecture for high-performance musical processing.

## Parallel Measure Formatting with Coordinated Updates

The examples below are sketches. `calculate-measure-view`, `format-staff-view` and `visible-elements` stand for the formatting work itself and are not Ooloi functions; everything addressed through `api/`, `pm/`, `cd/` and `vpd/` is. Measure views live in a staff view, which lives in a system view, so a measure view is set with the staff view's VPD, the measure view's index within that staff view, and the new measure view.

```clojure
;; Advanced: Parallel measure formatting showcasing Ooloi's elegant concurrency
(require '[ooloi.shared.api :as api]
         '[ooloi.backend.ops.piece-manager :as pm]
         '[ooloi.shared.ops.change-detection :as cd]
         '[ooloi.shared.ops.vpd :as vpd])

(defn extract-measures-by-range
  "Extract one staff's measures from the piece using temporal coordination."
  [piece-id staff-vpd measure-range]
  (->> (pm/get-piece piece-id)
       vector
       (sequence (comp (timewalk {:boundary-vpd staff-vpd
                                  :start-measure (first measure-range)
                                  :end-measure (last measure-range)})
                       (filter measure??)))
       (group-by #(vpd/measure (vpd %)))))  ; Group by measure number

(defn format-single-measure
  "Process individual measure with temporal item coordination."
  [[measure-num measure-results]]
  (let [visible-items (sequence (comp (mapcat #(timewalk (item %) {}))
                                      visible-elements)
                                measure-results)
        measure-view (calculate-measure-view visible-items)]
    [measure-num measure-view]))

(defn update-layout-measures
  "Coordinate parallel results into one staff view with composed VPD operations."
  [piece-id staff-view-vpd first-measure formatted-measures]
  ;; *** APPROACH 1: Compose multiple VPD operations ***
  ;; Each VPD operation is separate but runs inside one enclosing transaction
  (cd/with-coalesced-structural-change
    (doseq [[measure-num measure-view] formatted-measures]
      ;; Each VPD call has its own dosync, which nests into the scope's transaction
      ;; Result: All measure view updates succeed or fail as a single atomic operation
      (api/set-measure-view staff-view-vpd
                            piece-id
                            (- measure-num first-measure)   ; index within the staff view
                            measure-view))))

(defn update-layout-measures-batch
  "Alternative: Update the staff view's entire measure-views vector in one operation."
  [piece-id staff-view-vpd first-measure formatted-measures]
  ;; *** APPROACH 2: Single vector update operation ***
  ;; Collect all results and update the entire vector at once
  ;; This is cleaner and truly atomic - one API call, one transaction
  (let [current-measure-views (api/get-measure-views staff-view-vpd piece-id)
        ;; Build updated vector with parallel results
        updated-measure-views (reduce (fn [measure-views [measure-num measure-view]]
                                        (assoc measure-views (- measure-num first-measure) measure-view))
                                      current-measure-views
                                      formatted-measures)]
    ;; Single atomic update of the entire vector
    (api/set-measure-views staff-view-vpd piece-id updated-measure-views)))

(defn format-measures-parallel
  "Parallel measure formatting with temporal coordination and elegant addressing."
  [piece-id staff-vpd staff-view-vpd measure-range]
  (->> measure-range
       (extract-measures-by-range piece-id staff-vpd)
       (pmap format-single-measure)      ; "Just make this parallel" - that's it!
       doall                             ; "Wait for everything to finish" - explicit coordination
       (update-layout-measures piece-id staff-view-vpd (first measure-range))))  ; Composed atomically

(defn format-measures-parallel-batch
  "Alternative: Single vector update approach for comparison."
  [piece-id staff-vpd staff-view-vpd measure-range]
  (->> measure-range
       (extract-measures-by-range piece-id staff-vpd)
       (pmap format-single-measure)      ; "Just make this parallel" - that's it!
       doall                             ; "Wait for everything to finish" - explicit coordination
       (update-layout-measures-batch piece-id staff-view-vpd (first measure-range))))  ; One vector update

;; Both approaches are atomic in execution:
;; 1. Composed approach: Multiple VPD operations in one with-coalesced-structural-change scope
;; 2. Batch approach: Single VPD operation updating entire vector
;; Choose based on your preference for granularity vs. simplicity

;; The elegant simplicity: Complex concurrent programming reduced to three concepts:
;; 1. pmap  - "make this parallel" 
;; 2. doall - "wait for everything to finish"
;; 3. VPD   - "update safely via STM"
;; Thread management, synchronization, race conditions - all handled transparently.
;;
;; Notice how the threading macro (->> ) makes the concurrent flow crystal clear:
;; • Read top-to-bottom like a recipe: extract → parallelize → coordinate → update
;; • No nested function calls or callback hell
;; • Concurrency coordination points are explicit and obvious
;; • Automatic transaction retries, conflict resolution, and thread safety
;; • Enterprise-grade concurrency that reads like simple data transformation

;; Usage: format measures 10-50 of the first staff, assuming one system displays them all
(pm/store-piece "symphony-id" symphony-piece)
(format-measures-parallel "symphony-id"
                          [:musicians 0 :instruments 0 :staves 0]
                          [:layouts 0 :page-views 0 :system-views 0 :staff-views 0]
                          (range 10 51))
```

## Collection and Integration Patterns

```clojure
;; Advanced: Staff-level parallel processing with hierarchical addressing
(defn extract-staff-elements
  "Extract visible elements from a single staff using boundary-VPD scoping."
  [piece staff-vpd]
  (into [] 
        (comp (timewalk {:boundary-vpd staff-vpd})
              visible-elements)
        [piece]))

(defn format-single-staff
  "Process individual staff with spatial layout calculations."
  [piece [staff-view-index staff-vpd]]
  (let [elements (extract-staff-elements piece staff-vpd)
        staff-view (format-staff-view elements)]
    [staff-view-index staff-view]))

(defn integrate-staff-layouts
  "Integrate parallel staff results using sophisticated VPD addressing."
  [piece-id system-view-vpd staff-results]
  (doseq [[staff-view-index staff-view] staff-results]
    ;; Elegant nested addressing: the system view's VPD, then the staff view's index in it
    (api/set-staff-view system-view-vpd
                        piece-id
                        staff-view-index
                        staff-view)))

(defn parallel-staff-formatting
  "Parallel staff processing showcasing VPD boundary scoping and hierarchical addressing.
   staff-vpds are given in the order of the system view's staff views."
  [piece-id staff-vpds system-view-vpd]
  (let [piece (pm/get-piece piece-id)]
    (->> staff-vpds
         (map-indexed vector)                        ; [staff-view-index staff-vpd]
         (pmap (partial format-single-staff piece))  ; Multi-core processing - complexity hidden
         doall                                       ; Coordination barrier - wait for all staves
         (integrate-staff-layouts piece-id system-view-vpd))))  ; Safe concurrent updates

;; Usage: Process multiple staves in parallel with elegant addressing
(parallel-staff-formatting "symphony-id"
                           [[:musicians 0 :instruments 0 :staves 0]
                            [:musicians 0 :instruments 0 :staves 1]
                            [:musicians 1 :instruments 0 :staves 0]]
                           [:layouts 0 :page-views 0 :system-views 0])
```

## Why Ooloi's STM Abstractions Are Essential

```clojure
;; Problem: Direct parallel updates to atoms create race conditions
(defn unsafe-parallel-update [piece-atom measure-data-list]
  (doall
    (pmap (fn [measure-data]
            ;; RACE CONDITION: Multiple threads modifying same atom concurrently
            (swap! piece-atom assoc-in [:layouts 0 :measures (:num measure-data)] measure-data))
          measure-data-list)))

;; Solution: Ooloi's VPD-based API handles STM coordination automatically
(defn safe-parallel-update [piece-id staff-view-vpd measure-data-list]
  (doall
    (pmap (fn [{:keys [index measure-view]}]
            ;; Each API call is automatically wrapped in dosync transaction
            (api/set-measure-view staff-view-vpd piece-id index measure-view))
          measure-data-list)))

;; For complex multi-step operations, just use multiple VPD operations
;; Each VPD operation is already automatically transactional
(defn coordinated-batch-update [piece-id updates]
  (doseq [{:keys [staff-view-vpd index measure-view]} updates]
    ;; Each call is automatically wrapped in its own transaction
    (api/set-measure-view staff-view-vpd piece-id index measure-view)))
```

## Key Benefits

- **Temporal coordination**: Piece-walker's measure ordering enables safe parallel processing
- **API abstraction**: VPD-based updates automatically handle STM coordination
- **Performance**: Parallel measure formatting for large orchestral works
- **Consistency**: All updates are atomic and consistent via Ooloi's transaction system
- **Type safety**: Polymorphic multimethods ensure correct operation dispatch

**Readability & Efficiency Advantages:**
- **Threading macro clarity**: The `->>` pipeline reads like a recipe - extract, parallelize, coordinate, update
- **No callback hell**: Clean linear flow instead of nested async callbacks or promise chains
- **Explicit coordination**: `doall` makes the "wait for all" step obvious and intentional
- **Zero boilerplate**: No manual thread pools, locks, or synchronization primitives
- **Automatic scaling**: Utilizes all CPU cores without thread management code
- **Failure handling**: STM retries and conflict resolution happen transparently
- **Enterprise-grade**: Production-ready concurrency with development-friendly simplicity

## When to Use These Patterns

- Large orchestral scores (1000+ measures)
- Real-time layout updates
- Multi-core layout engines
- Complex visual hierarchy mutations requiring coordinated updates

**API patterns to prefer:**
- Use `api/set-xxx` with VPDs for automatic transaction handling
- Use `cd/with-coalesced-structural-change` around the VPD calls, or `api/atomic` with operation maps, for complex multi-step operations (`SRV/atomic` from a client)
- Always work with piece-ids rather than direct refs for better abstraction

## Cross-References

- **Prerequisites**: [Timewalking Guide](TIMEWALKING_GUIDE.md) - Complete this first
- **Piece management**: [Piece Manager Guide](PIECE_MANAGER_GUIDE.md) - STM coordination with piece refs and lifecycle management
- **Server architecture**: [Ooloi Server Architectural Guide](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md) - How these STM patterns integrate with distributed gRPC architecture
- **gRPC integration**: [gRPC Communication and Flow Control](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md) - STM-gRPC coordination patterns
- **Architecture**: [ADR-0004: STM for Concurrency](../ADRs/0004-STM-for-concurrency.md) - Architectural foundation for STM approach
- **Atom vs STM ref**: [ADR-0045: Instrument Library](../ADRs/0045-Instrument-Library.md) §"Why atom, not STM ref" - when a single atom is correct and STM would only add overhead
- **STM fundamentals**: Clojure documentation on Software Transactional Memory
- **Performance considerations**: [Timewalk Performance Benchmarks](https://github.com/PeterBengtson/Ooloi-docs/blob/main/READMEs/BENCHMARKS_README.md) - traversal costs at 1000-measure scale, for the extraction step these patterns parallelise

## Next Steps

- **Attachment processing**: Parallel resolution of slurs, ties, and hairpins
- **MIDI generation**: Concurrent event stream processing for real-time playback
- **Layout optimization**: Parallel spacing and collision detection algorithms