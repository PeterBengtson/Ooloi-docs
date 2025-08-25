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

The piece manager maintains a **dual-ref system**:

```clojure
;; Internal piece store - ref containing a map of piece-id -> piece-ref
(defonce ^:private piece-store (ref {}))

;; Each piece is stored as its own ref
(def piece-id "my-symphony")
(def piece-ref (ref (create-piece ...)))

;; The store maps IDs to refs
@piece-store ; => {"my-symphony" #<Ref @piece-ref>}
```

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
;; Check piece store contents
(defn list-stored-pieces []
  (->> @piece-store
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