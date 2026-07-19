# 🟡 Piece Manager Guide: Storage, Retrieval, and Lifecycle Management

> 💡 **CONVENIENCE TIP**: While you CAN use `alter`, `ref`, and low-level STM operations, 
> VPD-based API operations like `(api/set-measure vpd piece-id 5 measure)` are much easier! 
> See [API Convenience](POLYMORPHIC_API_GUIDE.md#-api-convenience-why-use-vpd-operations) for details.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Understanding the Piece Manager](#understanding-the-piece-manager)
- [Basic Operations](#basic-operations)
- [Piece Storage and Retrieval](#piece-storage-and-retrieval)
- [Ref Management](#ref-management)
- [Lifecycle Patterns](#lifecycle-patterns)
- [Working with Piece IDs](#working-with-piece-ids)
- [Integration with VPD Operations](#integration-with-vpd-operations)
- [Best Practices](#best-practices)
- [Common Patterns](#common-patterns)
- [Troubleshooting](#troubleshooting)
- [Cross-References](#cross-references)

## Overview

The piece manager is Ooloi's central system for managing the lifecycle of musical pieces. It handles storage, retrieval, reference management, and provides the foundation for all piece-based operations throughout the system.

**Key responsibilities:**
- **Storage**: Maintaining a registry of pieces with unique IDs (typically UUIDs in production)
- **Reference management**: Converting between pieces, piece-refs, and piece-IDs
- **Lifecycle coordination**: Creating, storing, and managing piece lifetimes
- **API integration**: Providing the foundation for VPD operations

**Note**: While examples in this guide use mnemonic strings for clarity, production systems typically use UUIDs for piece identification to ensure global uniqueness and avoid naming conflicts.

## Prerequisites

- **Basic Clojure knowledge**: Comfortable with refs, atoms, and basic data structures
- **Understanding of Ooloi pieces**: Familiarity with piece structure and VPDs
- **Optional**: Understanding of STM concepts (covered in [Advanced Concurrency Patterns](ADVANCED_CONCURRENCY_PATTERNS.md))

## Understanding the Piece Manager

### The Core Architecture

The piece manager maintains a **dual-ref system**, and the store is **owned by the piece-manager Integrant component** — it lives for exactly as long as that component runs, not for the lifetime of the JVM:

```clojure
;; The store handle — a ref that holds the live store ref, or nil when no
;; piece-manager component is running. The component's init-key creates a fresh
;; store ref and publishes it here (init-store!); halt-key! clears it back to nil
;; (release-store!). Removing a defonce: the store no longer survives the
;; component's halt, so it cannot leak across a restart.
(def ^:private piece-store (ref nil))

;; The store ref it holds maps piece-id -> a registry entry of the form
;; {:ref piece-ref :provenance {:path :modified}}; each piece is its own ref.
(def piece-id "my-symphony")
(def piece-ref (ref (create-piece ...)))

;; While a piece-manager component is running:
@@piece-store ; => {"my-symphony" {:ref #<Ref @piece-ref>
              ;                     :provenance {:path "…/my-symphony.ooloi" :modified 1718900000000}}}
```

**Why a ref handle rather than an atom?** Every store mutation already happens inside `dosync` (`store-piece` / `remove-piece` `alter` the store ref). Making the handle a **ref** too keeps the whole access path under a single STM discipline: the handle read inside a transaction is transactionally consistent by construction, and there is no second reference type to reason about. The handle is populated by the component, so pure-model code — `resolve-into-piece-ref`, `get-piece-ref` — reaches the store through it with no server in scope. The store is created by `init-store!` (called from the component's `init-key`) and released by `release-store!` (from `halt-key!`); there is no JVM-global remnant.

**The registry entry.** Each value in the store is a map — `{:ref piece-ref :provenance {:path :modified}}` — not a bare ref. The `:ref` is the piece's STM ref (the dual-ref system above). The `:provenance` — the file path and modification time the piece was opened from — is present only for pieces opened through `open-piece` (ADR-0051), and is how the Piece Manager distinguishes an idempotent reopen of the same file (same embedded UUID, same provenance) from a variant collision (same UUID, different provenance) — see [ADR-0012](../ADRs/0012-Persisting-Pieces.md). Provenance shares the ref's identity and lifetime — dropped with it on `remove-piece` — rather than living in a parallel structure to keep in sync. Distinct from this per-open provenance, the **persistent catalogue** — a second, durable store the same component owns, mapping every piece's UUID to its storage location whether or not it is currently open — is described in [The Persistent Catalogue](#the-persistent-catalogue) below.

### Why This Architecture?

**Individual piece refs** enable:
- **Concurrent access**: Multiple threads can work on different pieces simultaneously
- **Isolated transactions**: Changes to one piece don't block access to others
- **Efficient updates**: Only the specific piece being modified needs coordination

**Central piece store** provides:
- **Global registry**: Find any piece by ID across the entire system
- **Lifecycle management**: Track which pieces exist and their states
- **API consistency**: Uniform access patterns for all piece operations

### The Three Forms of Piece References

The piece manager handles three different ways to reference a piece:

```clojure
;; 1. Piece object (immutable value)
(def piece-obj (create-piece :musicians [musician1] :layouts [layout1]))

;; 2. Piece ref (STM reference)
(def piece-ref (ref piece-obj))

;; 3. Piece ID (string identifier)
(def piece-id "symphony-no-9")
```

### The Persistent Catalogue

The store above is **in-memory**: it holds the pieces open *right now*, and it dies when the component halts. Beside it the same component owns a **second, durable store — the persistent catalogue** — mapping every piece's embedded UUID to *where that piece lives*, for every piece ever opened or saved, whether or not it is currently open. The in-memory store answers "is this piece open, and where is its live ref?"; the catalogue answers "where on disk is the piece with this UUID?" — the durable resolution needed to re-open a piece that is no longer in memory. Because the in-memory registry empties as clients disconnect (close-on-last-release, [ADR-0022](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md)) and is gone entirely at shutdown, without the catalogue there would be nothing to resolve a UUID against on relaunch. Re-opening the pieces that were open at last quit is its first consumer.

**The value is a tagged locator, never a bare path.** Each entry maps a UUID to `{:kind :filesystem :path "…"}`:

```clojure
;; The persisted catalogue EDN — every piece's UUID → its storage locator
{"5a3c1e7f-…" {:kind :filesystem :path "…/my-symphony.ooloi"}
 "b71f9d02-…" {:kind :filesystem :path "…/string-quartet.ooloi"}}
```

The `:kind` tag is what lets `{:kind :s3 …}` or `{:kind :db …}` slot in later with **no schema change** — storage-backend opacity ([ADR-0012](../ADRs/0012-Persisting-Pieces.md)); only `:filesystem` is built today. The locator is **backend-internal**: it never crosses to the frontend, which holds UUIDs (identity) and ephemeral per-session tokens (access), never a storage location.

**Writes ride the provenance recorders — no new seam.** The two operations that already record a piece's provenance in the piece-manager — `register-opened-piece` (on open) and `record-piece-provenance` (on save) — each *also* record `uuid → {:kind :filesystem :path}`. The same fact — a piece's location — is recorded at two durabilities in one place: the ephemeral provenance beside the live ref, and the durable entry in the catalogue.

**Persistence is on every mutation, not at halt.** Each write hands the whole catalogue map to a **writer agent** that spits it to EDN immediately (the same writer-agent pattern the Instrument Library component uses), under `get-platform-directory "Ooloi" "catalogue"`. `halt-key!` only *awaits* any in-flight write — it is not the write trigger — so a crash never loses a committed entry. The catalogue is read back from that EDN at `init-key`, which is how it survives a restart.

**The catalogue is best-effort secondary durability.** A catalogue write must never fail the open or save it rode in on. Both `make-parents` and `spit` run on the writer agent under `:error-mode :continue`, so a disk error stays on the agent and cannot escape `register-opened-piece` / `record-piece-provenance` onto the wire — the operation still returns its piece. The catalogue is a convenience for later resolution, not a correctness dependency of opening or saving.

**Reads: `catalogue-resolve` and `open-by-uuid`.** The raw read is `catalogue-resolve [uuid] → locator | nil`, backend-internal (the locator stays inside the backend). The consumer-facing read is **opening a piece by its UUID**: `open-by-uuid` resolves the UUID to a locator, loads the file through `open-piece`'s shared load tail (read → deserialize → validate against the typed-failure taxonomy → register under the embedded UUID), and **verifies the loaded piece's embedded UUID matches the one requested** — guarding against the file at that location having become a *different* piece. It returns the piece-id, or a typed `{:open-failure …}`: `:piece-not-in-catalogue` (the UUID has no entry), `:piece-not-found` (the entry's file is gone — a stale locator), `:piece-uuid-mismatch` (the file now holds a different piece), plus the shared load-tail failures. `open-by-uuid` is exposed through `api`/`SRV` — session restore sends a UUID and gets back a piece-id or a typed failure it maps to a notification; `catalogue-resolve` is not exposed. The operation, its one-directional relationship to `open-piece`, and the full taxonomy are specified in [ADR-0051](../ADRs/0051-Filesystem-Operations-Real-and-Virtual.md).

## Complete API Reference

The piece manager provides five core functions:

### `store-piece`
```clojure
(pm/store-piece piece-id piece) ; => piece
```
- **Purpose**: Stores a piece with given ID
- **Behavior**: Always overwrites existing pieces with same ID
- **Returns**: The stored piece object
- **Thread-safe**: Uses `dosync` for atomic updates

### `get-piece`
```clojure
(pm/get-piece piece-id) ; => piece-or-nil
```
- **Purpose**: Retrieves piece by ID
- **Returns**: Piece object if found, `nil` otherwise
- **Thread-safe**: Reads from STM refs safely

### `get-piece-ref`
```clojure
(pm/get-piece-ref piece-thing) ; => ref
```
- **Purpose**: Polymorphic ref retrieval/creation
- **Input types**: String (piece ID), Ref (returns as-is), Object (wraps in ref)
- **Error handling**: Throws `ExceptionInfo` for non-existent IDs with `:piece-id` in ex-data
- **Returns**: Always returns a ref

### `remove-piece`
```clojure
(pm/remove-piece piece-id) ; => piece-or-nil
```
- **Purpose**: Removes piece from store
- **Returns**: The removed piece if it existed, `nil` otherwise
- **Thread-safe**: Uses `dosync` for atomic removal

### `resolve-piece`
```clojure
(pm/resolve-piece piece-or-id) ; => piece-or-nil
```
- **Purpose**: Flexible piece resolution
- **Logic**: String → lookup, anything else → return as-is
- **Use case**: Handles both piece objects and IDs uniformly

## Basic Operations

### Creating and Storing Pieces

```clojure
(require '[ooloi.backend.ops.piece-manager :as pm])
(require '[ooloi.backend.models.core :as core])

;; Create a new piece
(def piece (core/create-piece :musicians [musician1 musician2]
                              :layouts [layout1]
                              :time-signatures (create-change-set)
                              :key-signatures (create-change-set)
                              :tempos (create-change-set)))

;; Production: Store with UUID
(def piece-id (str (java.util.UUID/randomUUID)))
(pm/store-piece piece-id piece)
;; => returns the piece object

;; Development: Store with mnemonic ID
(pm/store-piece "beethoven-9th" piece)
;; => returns the piece object

;; Note: No automatic ID generation - you must provide an ID
```

### Basic Retrieval

```clojure
(require '[ooloi.backend.ops.piece-manager :as pm])

;; Get piece object (immutable snapshot)
(def piece-snapshot (pm/get-piece "beethoven-9th"))

;; Get piece ref (for transactions)
(def piece-ref (pm/get-piece-ref "beethoven-9th"))

;; Resolve piece from ID or object
(def resolved-piece (pm/resolve-piece "beethoven-9th"))  ; Returns piece object
(def resolved-piece (pm/resolve-piece piece-object))     ; Returns same object
```

## Piece Storage and Retrieval

### Storage Patterns

```clojure
;; Basic storage
(pm/store-piece "piece-id" piece)

;; Overwrite existing (always replaces)
(pm/store-piece "piece-id" updated-piece)  ; Replaces existing automatically

;; Conditional storage (store only if doesn't exist)
(when-not (pm/get-piece "piece-id")  ; Check by attempting retrieval
  (pm/store-piece "piece-id" piece))
```

### Retrieval Patterns

```clojure
;; Safe retrieval with error handling
(if-let [piece (pm/get-piece "piece-id")]
  (process-piece piece)
  (println "Piece not found"))

;; Retrieval with default
(def piece (or (pm/get-piece "piece-id")
               (create-piece)))  ; Use empty piece as fallback

;; Multiple pieces
(def pieces (map pm/get-piece ["piece-1" "piece-2" "piece-3"]))

;; Flexible resolution (handles both IDs and objects)
(def piece (pm/resolve-piece piece-or-id))  ; Works with either
```

## Ref Management

### Understanding `get-piece-ref`

The `get-piece-ref` function is the cornerstone of piece manager's flexibility:

```clojure
;; Handles three input types:
(pm/get-piece-ref "piece-id")     ; String -> looks up stored ref (throws if not found)
(pm/get-piece-ref piece-ref)      ; Ref -> returns as-is
(pm/get-piece-ref piece-object)   ; Object -> wraps in new ref
```

**Error handling:**
```clojure
;; Throws informative exception for non-existent IDs
(pm/get-piece-ref "non-existent")  ; => ExceptionInfo: "Piece with ID 'non-existent' not found"
                                   ; ex-data: {:piece-id "non-existent"}
```

**Why this matters:**
```clojure
(defn my-operation [piece-id-or-object new-element]
  ;; 😎 Easy way: VPD operations handle everything for you
  (api/add-element [] piece-id-or-object new-element))

;; All of these work with VPD operations:
(my-operation "stored-piece-id" element)     ; Uses stored piece
(my-operation some-piece-object element)     ; Uses object directly

;; 😓 Hard way: You could do this, but why?
;; (let [piece-ref (pm/get-piece-ref piece-thing)]
;;   (dosync
;;     (alter piece-ref update-something)))
;; More code, more complexity, more chances for bugs!
```

### Resolving to a value or a ref: `resolve-into-piece` and `resolve-into-piece-ref`

`get-piece-ref` is the backend piece-store lookup. One level up sits the `PieceResolver`
protocol, which the shared tier uses to turn a piece-thing — a value, a ref, or an id — into a
piece. It offers **two** methods for two intents, so a reader never pays for a ref it will only
dereference:

```clojure
;; Readers — return the piece VALUE, allocating nothing for a bare value:
(resolve-into-piece piece-object)   ; Object -> returned unchanged
(resolve-into-piece piece-ref)      ; Ref    -> dereferenced to its value
(resolve-into-piece "piece-id")     ; String -> looked up (backend) and dereferenced

;; Writers — return a Ref, to alter within a transaction:
(resolve-into-piece-ref piece-object) ; Object -> wrapped in a new ref
(resolve-into-piece-ref piece-ref)    ; Ref    -> returned as-is
(resolve-into-piece-ref "piece-id")   ; String -> looked up stored ref (backend)
```

The read funnel `vpd-get` — every `get-xxxxx` on a piece — uses `resolve-into-piece`: a plain
piece value flows straight through, with no throwaway ref allocated on the hot getter path. The
write funnels `vpd-transact` (adders, removers, setters, movers, `individuate`) and
`vpd-mutate-setting` use `resolve-into-piece-ref`, because they `alter` the ref inside `dosync`.
Both methods accept a value, a ref, or a string id; the backend resolves ids through the piece
store (throwing on a missing id, exactly as `get-piece-ref` does), while the frontend rejects
string ids to force gRPC.

### Piece Lifecycle Patterns

```clojure
;; 😎 Pattern 1: Work with stored pieces using VPD operations
(api/add-musician [] "symphony" new-musician)  ; Automatic transaction handling

;; 😎 Pattern 2: Work with temporary pieces using VPD operations  
(let [temp-piece (create-piece ...)]
  (api/modify-element [] temp-piece modification))  ; Direct object operation

;; 😎 Pattern 3: Promote temporary to stored
(let [temp-piece (create-piece ...)
      modified-piece (modify-piece temp-piece)]
  (api/store-piece "new-id" modified-piece))
```

## Lifecycle Patterns

### Complete Piece Lifecycle

```clojure
;; 1. Creation
(def piece (api/create-piece :musicians [soprano alto tenor bass]))

;; 2. Storage
(api/store-piece "satb-choir" piece)

;; 3. Modification
(api/add-musician [] "satb-choir" organist)
(api/set-tempo [] "satb-choir" 0 (api/create-tempo 120))

;; 4. Retrieval for processing
(def final-piece (api/get-piece "satb-choir"))

;; 5. Export/serialization
(spit "choir-piece.edn" (pr-str final-piece))

;; 6. Cleanup (if needed)
(api/remove-piece "satb-choir")  ; Remove from store
```

Step 6 is the *programmatic* removal — a script or test that stored a piece and is done with it. **In the running application, pieces are never removed by hand.** A piece is registered when a client first opens it (New, Open, or a subscription) and removed automatically the moment its *last* subscriber leaves — a deterministic **close-on-last-release** driven by the subscriber count the connection registry keeps, with a disconnect treated as unsubscribe-from-every-piece-then-leave. A piece thus lives exactly as long as some client holds it: one client closing its window keeps the piece alive for any other still holding it, and the last client leaving closes it (the saved file re-loaded on a later reopen, not returned from memory). This is what makes Piece-Manager-mediated sharing identical for a single user and for a collaboration — the policy is [ADR-0022 §Piece Lifetime — Close-on-Last-Release](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md), and the disconnect handlers that invoke `remove-piece` are in [ADR-0024 §Connection Lifecycle](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md).

### Working with Piece Families

```clojure
;; Store related pieces with consistent naming
(api/store-piece "symphony-9-full-score" full-score)
(api/store-piece "symphony-9-piano-reduction" piano-version)
(api/store-piece "symphony-9-vocal-score" vocal-version)

;; Batch operations on related pieces
(def symphony-9-versions ["symphony-9-full-score" 
                         "symphony-9-piano-reduction" 
                         "symphony-9-vocal-score"])

(doseq [piece-id symphony-9-versions]
  (api/set-tempo [] piece-id 0 (api/create-tempo 120)))
```

## Working with Piece IDs

### ID Conventions

**In practice, piece IDs are typically UUIDs, not mnemonic strings:**

```clojure
;; Real-world production usage - UUIDs
"550e8400-e29b-41d4-a716-446655440000"  ; UUID v4
"6ba7b810-9dad-11d1-80b4-00c04fd430c8"  ; UUID v1
"01234567-89ab-cdef-0123-456789abcdef"  ; UUID format

;; UUIDs provide:
;; - Guaranteed uniqueness across distributed systems
;; - No naming conflicts or collision concerns
;; - Consistent format regardless of content
;; - Database-friendly indexing properties
```

**For examples and testing, mnemonic strings are useful:**
```clojure
;; Development/testing conventions (not production)
"composer-opus-movement"          ; => "beethoven-op67-mvt1"
"genre-key-tempo"                ; => "sonata-cmajor-allegro"
"project-date-version"           ; => "wedding-2024-03-15-v2"
"ensemble-repertoire-year"       ; => "choir-messiah-2024"
```

### Dynamic ID Generation

**Production approach - UUID generation:**
```clojure
(require '[clojure.java-time :as time])

(defn generate-piece-id []
  "Generate a UUID for piece identification."
  (str (java.util.UUID/randomUUID)))

;; Usage in production
(def piece-id (generate-piece-id))  ; => "550e8400-e29b-41d4-a716-446655440000"
(pm/store-piece piece-id piece)
```

**Development approach - mnemonic IDs with uniqueness:**
```clojure
(defn generate-mnemonic-piece-id [composer title & [version]]
  "Generate human-readable ID for development/testing."
  (let [base (str (clojure.string/lower-case composer) "-" 
                 (clojure.string/replace (clojure.string/lower-case title) #"\\s+" "-"))
        versioned (if version (str base "-v" version) base)]
    ;; Ensure uniqueness by checking existing pieces
    (loop [candidate versioned
           counter 1]
      (if (pm/get-piece candidate)  ; Check if exists
        (recur (str base "-" counter) (inc counter))
        candidate))))

;; Usage for testing/development
(def piece-id (generate-mnemonic-piece-id "Mozart" "Eine kleine Nachtmusik"))
(pm/store-piece piece-id piece)
```

## Integration with VPD Operations

### How Piece Manager Enables VPD Operations

Every VPD operation uses the piece manager internally:

```clojure
;; When you call:
(api/set-measure [:musicians 0 :instruments 0 :staves 0] "piece-id" 5 new-measure)

;; Internally, this happens:
;; 1. piece-ref <- (get-piece-ref "piece-id")  ; Piece manager lookup
;; 2. (dosync (alter piece-ref ...))           ; STM transaction
;; 3. VPD navigation to target location        ; VPD system
;; 4. Update with validation                   ; Type system
```

### Piece Manager in Custom Operations

```clojure
(defn batch-update-measures [piece-id measure-updates]
  "Update multiple measures atomically using VPD operations."
  ;; 😎 Easy way: Compose multiple VPD operations in outer transaction
  (dosync
    (doseq [[vpd measure-num measure] measure-updates]
      ;; Each VPD operation participates in the same atomic transaction
      (api/set-measure vpd piece-id measure-num measure))))

;; Usage with stored piece  
(batch-update-measures "symphony" 
                      [[[:musicians 0 :instruments 0 :staves 0] 5 measure1]
                       [[:musicians 1 :instruments 0 :staves 0] 5 measure2]])

;; 😓 Hard way: You could manually manage piece-ref, but why?
;; (let [piece-ref (api/get-piece-ref piece-id)]
;;   (dosync
;;     (doseq [[vpd measure] measure-updates]
;;       (alter piece-ref ...))))
;; Much more verbose and error-prone!
```

## Best Practices

### 1. Consistent ID Management

```clojure
;; Good: Centralized ID generation
(defn create-and-store-piece [metadata content]
  (let [piece-id (generate-piece-id (:composer metadata) (:title metadata))
        piece (create-piece-from-content content)]
    (pm/store-piece piece-id piece)
    piece-id))

;; Avoid: Scattered ID generation
;; Don't generate IDs in multiple places without coordination
```

### 2. Error Handling

```clojure
;; Robust piece access
(defn safe-piece-operation [piece-id operation]
  (if-let [piece-ref (try 
                       (api/get-piece-ref piece-id)
                       (catch Exception e
                         (log/warn "Piece not found:" piece-id)
                         nil))]
    (operation piece-ref)
    (log/error "Cannot perform operation: piece" piece-id "not found")))
```

### 3. Resource Management

```clojure
;; For temporary processing, consider cleanup
(defn process-temporary-piece [source-piece-id]
  (let [temp-id (str source-piece-id "-temp-" (System/currentTimeMillis))
        source-piece (api/get-piece source-piece-id)
        processed-piece (expensive-processing source-piece)]
    
    ;; Store temporarily for complex operations
    (api/store-piece temp-id processed-piece)
    
    (try
      ;; Perform operations that need stored piece
      (complex-multi-step-operation temp-id)
      
      ;; Return final result
      (api/get-piece temp-id)
      
      (finally
        ;; Cleanup temporary piece
        (api/remove-piece temp-id)))))
```

## Common Patterns

### Pattern 1: Piece Transformation Pipeline

```clojure
(defn transform-piece-pipeline [piece-id transformations]
  "Apply a series of transformations to a piece."
  ;; 😎 Clean functional approach: get → transform → store
  (let [current-piece (api/get-piece piece-id)
        transformed-piece (reduce (fn [piece transform-fn]
                                   (transform-fn piece))
                                 current-piece
                                 transformations)]
    (api/store-piece piece-id transformed-piece)))

;; Usage
(transform-piece-pipeline "my-piece" 
                         [add-tempo-markings
                          add-dynamic-markings
                          adjust-spacing])
```

### Pattern 2: Piece Comparison and Merging

```clojure
(defn merge-piece-versions [base-piece-id variant-piece-id target-piece-id]
  "Merge two piece versions into a new piece."
  (let [base-piece (api/get-piece base-piece-id)
        variant-piece (api/get-piece variant-piece-id)
        merged-piece (merge-pieces base-piece variant-piece)]
    (api/store-piece target-piece-id merged-piece)
    target-piece-id))
```

### Pattern 3: Piece Backup and Versioning

```clojure
(defn create-piece-backup [piece-id]
  "Create a timestamped backup of a piece."
  (let [timestamp (java.time.Instant/now)
        backup-id (str piece-id "-backup-" (.toEpochMilli timestamp))
        piece (api/get-piece piece-id)]
    (api/store-piece backup-id piece)
    backup-id))

(defn restore-piece-backup [backup-id original-id]
  "Restore a piece from backup."
  (let [backup-piece (api/get-piece backup-id)]
    (api/store-piece original-id backup-piece)))
```

## Troubleshooting

### Common Issues

**1. Piece not found errors**
```clojure
;; Problem: Piece ID doesn't exist
(api/get-piece "nonexistent")  ; Returns nil

;; Solution: Always check existence
(when-let [piece (api/get-piece piece-id)]
  (process-piece piece))
```

**2. Memory leaks with temporary refs**
```clojure
;; Problem: Creating refs without storing them
(let [piece-ref (api/get-piece-ref some-piece-object)]
  ;; Long-running operation
  (time-consuming-process piece-ref))
;; Ref stays in memory until GC

;; Solution: Use stored pieces for long-running operations
(let [temp-id (generate-temp-id)]
  (api/store-piece temp-id some-piece-object)
  (try
    (time-consuming-process temp-id)
    (finally
      (api/remove-piece temp-id))))
```

**3. Concurrent modification conflicts**
```clojure
;; Problem: Multiple threads modifying same piece
;; Solution: VPD operations handle STM coordination automatically for you

;; 😎 Easy way: For specific operations, use VPD operations directly
(api/add-musician [] piece-id new-musician)  ; Automatic STM coordination
(api/set-tempo [] piece-id 0 new-tempo)      ; Automatic STM coordination

;; 😎 For complex transformations, functional approach is cleaner
(defn safe-piece-update [piece-id update-fn]
  (let [current-piece (api/get-piece piece-id)
        updated-piece (update-fn current-piece)]
    (api/store-piece piece-id updated-piece)))

;; 😓 Hard way: You could manage piece-ref manually, but why bother?
;; (let [piece-ref (api/get-piece-ref piece-id)]
;;   (dosync (alter piece-ref update-fn)))
;; More complex, more error-prone, harder to test
```

### Debugging Tips

```clojure
;; Check piece store contents (double deref: handle -> store ref -> id->entry map)
(defn list-stored-pieces []
  (->> @@piece-store
       keys
       sort))

;; Piece statistics
(defn piece-info [piece-id]
  (when-let [piece (api/get-piece piece-id)]
    {:musicians (count (:musicians piece))
     :layouts (count (:layouts piece))
     :has-tempo (not (empty? (:tempos piece)))
     :has-key-sig (not (empty? (:key-signatures piece)))}))
```

## Cross-References

### **Core Concepts**
- **VPD operations**: See [Polymorphic API Guide](POLYMORPHIC_API_GUIDE.md#-api-convenience-why-use-vpd-operations) for VPD-based operations used in piece management
- **VPD addressing**: See [VPDs Guide](VPDs.md) for understanding the addressing system used throughout piece operations
- **Basic piece operations**: See [Timewalking Guide](TIMEWALKING_GUIDE.md) for traversal patterns

### **Advanced Patterns**
- **STM coordination**: See [Advanced Concurrency Patterns](ADVANCED_CONCURRENCY_PATTERNS.md) for complex piece ref coordination and parallel processing
- **Piece persistence**: See [Piece Persistence Guide](PIECE_PERSISTENCE_GUIDE.md) for complementary piece lifecycle management

### **Architecture**
- **STM fundamentals**: See [ADR-0004: STM for Concurrency](../ADRs/0004-STM-for-concurrency.md) for the architectural foundation of piece ref management
- **Server integration**: See [Ooloi Server Architectural Guide](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md) for how piece management integrates with distributed gRPC architecture and STM transactions
- **Performance monitoring**: See [ADR-0025: Server Statistics Architecture](../ADRs/0025-Server-Statistics-Architecture.md) for piece management performance tracking and optimization

## Next Steps

- **API integration**: Learn how piece manager integrates with all Ooloi APIs
- **Performance optimization**: Understand piece manager's role in efficient operations
- **Plugin development**: Use piece manager for plugin piece lifecycle management
- **Serialization**: Explore piece export/import patterns using piece manager