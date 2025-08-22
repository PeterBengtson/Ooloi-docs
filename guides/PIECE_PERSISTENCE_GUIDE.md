# 🟢 Piece Persistence Guide: Asynchronous Save/Load Operations

## Table of Contents

- [Quick Start](#quick-start)
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Concurrency Primer: Agents vs STM](#concurrency-primer-agents-vs-stm)
- [Understanding Piece Persistence](#understanding-piece-persistence)
- [Basic Save Operations](#basic-save-operations)
- [Basic Load Operations](#basic-load-operations)
- [Multiple I/O Backends](#multiple-io-backends)
- [Real-World Workflows](#real-world-workflows)
- [Error Handling](#error-handling)
- [Integration with Piece Manager](#integration-with-piece-manager)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Cross-References](#cross-references)

## Quick Start

**Need to save/load a piece right now?** Here's the essential code:

```clojure
(require '[ooloi.backend.ops.persistence :as persist]
         '[ooloi.backend.ops.piece-manager :as pm])

;; Save a piece to file
(def piece-id "my-symphony")  ; Assuming piece exists in piece manager
(persist/save-piece-async piece-id (persist/file-writer "~/Documents/symphony.ooloi"))

;; Wait for save to complete
(while (not (persist/save-complete? piece-id))
  (Thread/sleep 100))

;; Load a piece from file  
(persist/load-piece-async (persist/file-reader "~/Documents/symphony.ooloi"))

;; Wait for load to complete
(while (not (persist/load-complete?))
  (Thread/sleep 100))
;; Piece is now available: (pm/get-piece "piece-id")

;; Check if everything worked
(if-let [error (persist/save-error piece-id)]
  (println "Save failed:" error)
  (println "Save succeeded"))
```

**That's it!** Read on for details, multiple backends, error handling, and advanced features.

## Overview

Ooloi's persistence system provides sophisticated asynchronous save and load operations for musical pieces. Built on Clojure's agent-based concurrency model and Taoensso Nippy serialization, it handles complex musical structures efficiently while supporting multiple I/O backends.

**Key capabilities:**
- **Asynchronous I/O**: Non-blocking save/load operations using Clojure agents
- **Multiple backends**: File, in-memory buffer, and network socket I/O
- **High-performance serialization**: Nippy-based binary format optimized for Clojure data structures
- **Large piece handling**: Efficient processing of scores with hundreds of thousands of elements
- **Comprehensive error handling**: Robust error recovery and state tracking

**Architecture Foundation:**
- **Agent-based concurrency**: Two dedicated agents for save and load operations
- **Pure tree structure**: Leverages Ooloi's immutable musical hierarchy ([ADR-0010: Pure Trees](../ADRs/0010-Pure-Trees.md), [ADR-0012: Persisting Pieces](../ADRs/0012-Persisting-Pieces.md))
- **Nippy serialization**: Binary format with built-in compression and encryption support ([ADR-0007: Nippy](../ADRs/0007-Nippy.md))
- **Integration layer**: Seamless integration with piece manager for complete lifecycle management

## Prerequisites

- **Basic Clojure knowledge**: Understanding of basic data structures and functions
- **Piece Manager familiarity**: Understanding of piece storage and ID management (see [Piece Manager Guide](PIECE_MANAGER_GUIDE.md))
- **Ooloi concepts**: Basic understanding of piece structure and VPDs
- **Optional**: Knowledge of Nippy serialization library

## Concurrency Primer: Agents vs STM

Ooloi uses **two complementary concurrency models** for different purposes:

### STM (Software Transactional Memory) - Core Operations
**Used by**: Piece manager, VPD operations, musical structure modifications

```clojure
;; STM coordinates multiple piece modifications atomically using VPD API
(dosync
  (api/add-musician piece-ref [] new-musician)
  (api/set-tempo piece-ref [] 0 fast-tempo))
```

**Characteristics:**
- **Coordinated**: Multiple refs can be updated together atomically
- **Blocking**: Operations wait for coordination and retry on conflicts
- **ACID**: Atomic, Consistent, Isolated, Durable transactions
- **Perfect for**: Musical structure modifications where consistency matters

### Agents - Asynchronous I/O Operations  
**Used by**: Persistence system, file operations, network communications

```clojure
;; Async operations handle I/O without blocking other operations
(persist/save-piece-async piece-id (persist/file-writer "/path/to/piece.ooloi"))
;; Returns immediately while save happens in background
```

**Characteristics:**
- **Independent**: Each agent operates separately without coordination
- **Non-blocking**: Operations never block the calling thread
- **Queue-based**: Operations are queued and processed asynchronously
- **Perfect for**: I/O operations where independence and non-blocking behavior matter

### Why This Separation?

**STM and Agents serve different needs:**
- **STM**: "Update multiple pieces atomically while maintaining musical consistency"
- **Agents**: "Save this piece to disk without blocking the user interface"

**They don't interfere with each other:**
```clojure
;; This works perfectly - STM and agents operate independently
(dosync                                    ; STM transaction
  (api/add-measure piece-ref [:musicians 0 :instruments 0 :staves 0 :voices 0] new-measure))

(persist/save-piece-async "piece-id" (persist/file-writer "/path/to/piece.ooloi"))
;; User interface remains responsive while save happens in background
```

**Integration points:**
- **Piece manager** (STM) provides pieces to **persistence** (agents)  
- **Persistence** (agents) loads pieces into **piece manager** (STM)
- Each system handles its domain optimally without interference

This architectural choice enables Ooloi to provide **consistent musical operations** (STM) while maintaining **responsive I/O** (agents) - getting the best of both concurrency models.

## Understanding Piece Persistence

### Core Architecture

The persistence system operates through **asynchronous I/O operations** that handle all piece storage and retrieval without blocking:

```clojure
;; Core async operations
(persist/save-piece-async piece-id writer-fn)  ; Non-blocking save
(persist/load-piece-async reader-fn)           ; Non-blocking load
```

**Internal Implementation**: The system uses two dedicated Clojure agents (`save-agent` and `load-agent`) to manage I/O operations asynchronously. This agent-based architecture ensures that disk operations never block the main application thread, keeping the user interface responsive while large musical pieces are saved or loaded in the background.

### Status Tracking

The system provides clean status checking functions:

```clojure
;; Check save status
(persist/save-status "piece-id")    ; => :pending | :completed | :error
(persist/save-complete? "piece-id") ; => true/false  
(persist/save-error "piece-id")     ; => error-message or nil

;; Check load status
(persist/load-status)               ; => :pending | :completed | :error
(persist/load-complete?)            ; => true/false
(persist/load-error)                ; => error-message or nil
```

### I/O Function Architecture

The system uses **function builders** that create I/O operations:

```clojure
;; Output functions for saving
(file-writer "/path/to/piece.ooloi")    ; Returns function that writes to file
(buffer-writer)                         ; Returns function that writes to memory
(socket-writer "host" 1234)            ; Returns function that writes to network

;; Input functions for loading  
(file-reader "/path/to/piece.ooloi")    ; Returns function that reads from file
(buffer-reader byte-array)              ; Returns function that reads from memory
(socket-reader "host" 1234)            ; Returns function that reads from network
```

### Serialization Technology

**Nippy provides the serialization layer:**
- **Binary format**: Compact, efficient representation
- **Clojure-native**: Handles all Clojure data structures natively
- **Built-in compression**: Automatic size optimization
- **Streaming support**: Handles large pieces without memory issues
- **Cross-platform**: Consistent serialization across systems

## Basic Save Operations

### File-Based Saving

```clojure
(require '[ooloi.backend.ops.persistence :as persist])
(require '[ooloi.backend.ops.piece-manager :as pm])

;; Save piece object directly to file
(def piece (create-piece :musicians [musician1 musician2]))
(persist/save-piece-async piece (persist/file-writer "~/Documents/my-symphony.ooloi"))

;; Save stored piece by ID
(pm/store-piece "symphony-uuid" piece)
(persist/save-piece-async "symphony-uuid" (persist/file-writer "~/Documents/symphony.ooloi"))

;; Check operation status
(await persist/save-agent)
@persist/save-agent  ; => {"symphony-uuid" :saved}
```

### Basic Save Workflow

```clojure
;; Complete save workflow with error checking
(defn save-piece-to-file [piece-or-id file-path]
  "Save a piece to file with error handling."
  (let [save-op (persist/save-piece-async piece-or-id (persist/file-writer file-path))]
    (await save-op)
    (if-let [error (:error @save-op)]
      (println "Save failed:" error)
      (println "Piece saved successfully to" file-path))))

;; Usage
(save-piece-to-file "my-piece-id" "~/Documents/pieces/sonata.ooloi")
(save-piece-to-file piece-object "~/Documents/pieces/direct-save.ooloi")
```

### Understanding the Save Process

```clojure
;; The save process happens asynchronously:
;; 1. save-piece-async initiates asynchronous save operation
;; 2. Internal save agent resolves piece (if ID provided) using piece manager
;; 3. Agent calls output function with piece object  
;; 4. Output function serializes piece using Nippy
;; 5. Agent updates its internal state with result (accessible via status functions)
;; 6. Operation completes, can be checked with await/deref
```

## Basic Load Operations

### File-Based Loading

```clojure
;; Load piece from file (automatically registers in piece manager)
(persist/load-piece-async (persist/file-reader "~/Documents/symphony.ooloi"))

;; Check operation status and result
(await persist/load-agent)
@persist/load-agent  ; => {"piece-id" loaded-piece-object}

;; Piece is now available in piece manager
(pm/get-piece "piece-id")  ; => loaded-piece-object
```

### Custom ID Assignment

```clojure
;; Load with custom ID function
(persist/load-piece-async 
  (persist/file-reader "~/Documents/legacy-piece.ooloi")
  :id-fn :custom-id)  ; Use :custom-id field instead of :id

;; Load with computed ID
(persist/load-piece-async 
  (persist/file-reader "~/Documents/untitled.ooloi")
  :id-fn (fn [piece] (str "imported-" (System/currentTimeMillis))))
```

### Basic Load Workflow

```clojure
(defn load-piece-from-file [file-path & {:keys [id-fn]}]
  "Load piece from file with error handling."
  (let [load-op (if id-fn
                  (persist/load-piece-async (persist/file-reader file-path) :id-fn id-fn)
                  (persist/load-piece-async (persist/file-reader file-path)))]
    (await load-op)
    (if-let [error (:error @load-op)]
      (do (println "Load failed:" error) nil)
      (let [piece-id (-> @load-op keys (remove #{:error}) first)]
        (println "Piece loaded successfully with ID:" piece-id)
        (pm/get-piece piece-id)))))

;; Usage
(load-piece-from-file "~/Documents/symphony.ooloi")
(load-piece-from-file "~/Documents/legacy.ooloi" :id-fn :legacy-id)
```

## Multiple I/O Backends

### Buffer-Based Operations

**Useful for in-memory processing, testing, and data transformation:**

```clojure
;; Save to memory buffer
(def piece {:id "buffer-piece" :musicians []})
(def buffer-result (atom nil))

(persist/save-piece-async piece 
  (fn [piece]
    (reset! buffer-result ((persist/buffer-writer) piece))))

(await persist/save-agent)
(def serialized-bytes @buffer-result)

;; Load from memory buffer  
(persist/load-piece-async (persist/buffer-reader serialized-bytes))
(await persist/load-agent)
(pm/get-piece "buffer-piece")  ; => loaded piece
```

### Network Socket Operations

**For distributed systems and real-time collaboration:**

```clojure
;; Server setup (simplified example)
(import '[java.net ServerSocket])

(defn start-piece-server [port]
  "Start a simple piece receiving server."
  (let [server (ServerSocket. port)]
    (future
      (with-open [socket (.accept server)
                  in (java.io.DataInputStream. (.getInputStream socket))]
        (let [piece (nippy/thaw-from-in! in)]
          (pm/store-piece (:id piece) piece)
          (println "Received piece:" (:id piece)))))))

;; Client save to server
(persist/save-piece-async "my-piece" (persist/socket-writer "piece-server.local" 8080))

;; Client load from server  
(persist/load-piece-async (persist/socket-reader "piece-server.local" 8081))
```

### I/O Backend Comparison

| Backend | Use Case | Pros | Cons |
|---------|----------|------|------|
| **File** | Standard save/load | Persistent, simple | File system dependent |
| **Buffer** | Testing, processing | Fast, no I/O | Memory limited |
| **Socket** | Network operations | Real-time, distributed | Network dependent |

### Advanced I/O Patterns

```clojure
;; Backup strategy: save to multiple locations
(defn backup-piece [piece-id backup-locations]
  "Save piece to multiple backup locations."
  (let [save-ops (map #(persist/save-piece-async piece-id (persist/file-writer %)) 
                      backup-locations)]
    (run! await save-ops)
    (every? #(= :saved (get @% piece-id)) save-ops)))

;; Usage
(backup-piece "important-symphony" 
              ["~/Documents/pieces/symphony.ooloi"
               "/Volumes/Backup/symphony.ooloi"
               "/cloud/pieces/symphony.ooloi"])

;; Serialization size analysis
(defn analyze-piece-size [piece]
  "Analyze serialized size of piece."
  (let [buffer-result (atom nil)]
    (persist/save-piece-async piece 
      (fn [p] (reset! buffer-result ((persist/buffer-writer) p))))
    (await persist/save-agent)
    {:original-piece piece
     :serialized-bytes (count @buffer-result)
     :compression-ratio (/ (count (pr-str piece)) (count @buffer-result))}))
```

## Real-World Workflows

### Composer's Save/Load Workflow

```clojure
(defn save-current-work [piece-id file-path]
  "Save the piece a composer is currently working on."
  (try
    (persist/save-piece-async piece-id (persist/file-writer file-path))
    (await persist/save-agent)
    (if-let [error (:error @persist/save-agent)]
      {:success false :error error}
      {:success true :file file-path :timestamp (java.time.Instant/now)})
    (catch Exception e
      {:success false :error (.getMessage e)})))

;; Usage
(save-current-work "beethoven-symphony-9" "~/Documents/works/symphony9-draft.ooloi")
```

### Project Backup and Versioning

```clojure
(defn create-timestamped-backup [piece-id backup-dir]
  "Create a timestamped backup of current work."
  (let [timestamp (.format (java.time.LocalDateTime/now)
                           (java.time.format.DateTimeFormatter/ofPattern "yyyy-MM-dd_HH-mm-ss"))
        backup-file (str backup-dir "/" piece-id "_" timestamp ".ooloi")]
    (persist/save-piece-async piece-id (persist/file-writer backup-file))
    (await persist/save-agent)
    (if (:error @persist/save-agent)
      (throw (Exception. (str "Backup failed: " (:error @persist/save-agent))))
      backup-file)))

;; Automatic backup every 5 minutes
(defn start-auto-backup [piece-id backup-dir]
  (future
    (while true
      (Thread/sleep (* 5 60 1000))  ; 5 minutes
      (try
        (create-timestamped-backup piece-id backup-dir)
        (println "Auto-backup completed:" (java.time.LocalTime/now))
        (catch Exception e
          (println "Auto-backup failed:" (.getMessage e)))))))
```

### Collaborative Workflow

```clojure
(defn share-piece-with-collaborator [piece-id collaborator-host port]
  "Send piece to collaborator over network."
  (persist/save-piece-async piece-id (persist/socket-writer collaborator-host port))
  (await persist/save-agent)
  (if-let [error (:error @persist/save-agent)]
    (println "Failed to send to collaborator:" error)
    (println "Piece sent successfully to" collaborator-host)))

(defn receive-from-collaborator [host port & {:keys [piece-suffix]}]
  "Receive piece from collaborator and store with modified ID."
  (let [id-fn (if piece-suffix
                #(str (:id %) "-" piece-suffix)
                :id)]
    (persist/load-piece-async (persist/socket-reader host port) :id-fn id-fn)
    (await persist/load-agent)
    (if-let [error (:error @persist/load-agent)]
      (println "Failed to receive piece:" error)
      (println "Received piece successfully"))))
```

### Format Migration Workflow

```clojure
(defn migrate-legacy-pieces [source-dir target-dir]
  "Migrate pieces from old format to new format."
  (doseq [file (file-seq (java.io.File. source-dir))
          :when (.endsWith (.getName file) ".ooloi")]
    (try
      ;; Load piece
      (persist/load-piece-async (persist/file-reader (.getPath file)))
      (await persist/load-agent)
      
      (if-let [error (:error @persist/load-agent)]
        (println "Failed to load" (.getName file) ":" error)
        
        ;; Get the loaded piece and save in new format
        (let [piece-id (-> @persist/load-agent keys (remove #{:error}) first)
              piece (pm/get-piece piece-id)
              new-file (str target-dir "/" (.getName file))]
          
          ;; Apply any format updates here
          (persist/save-piece-async piece-id (persist/file-writer new-file))
          (await persist/save-agent)
          
          (if (:error @persist/save-agent)
            (println "Failed to save" (.getName file))
            (println "Migrated" (.getName file)))))
      (catch Exception e
        (println "Error migrating" (.getName file) ":" (.getMessage e))))))
```

### Performance Testing Workflow

```clojure
(defn benchmark-save-load [piece-id iterations]
  "Benchmark save/load performance."
  (let [results (atom [])]
    (dotimes [i iterations]
      (let [start-time (System/nanoTime)
            
            ;; Save to buffer (fastest option)
            _ (persist/save-piece-async piece-id (persist/buffer-writer))
            _ (await persist/save-agent)
            save-time (- (System/nanoTime) start-time)
            
            ;; Get the buffer data
            buffer-data (persist/buffer-writer)
            piece (pm/get-piece piece-id)
            serialized (buffer-data piece)
            
            load-start (System/nanoTime)
            ;; Load from buffer
            _ (persist/load-piece-async (persist/buffer-reader serialized))
            _ (await persist/load-agent)
            load-time (- (System/nanoTime) load-start)]
        
        (swap! results conj {:iteration i
                            :save-time-ms (/ save-time 1000000.0)
                            :load-time-ms (/ load-time 1000000.0)
                            :size-bytes (count serialized)})))
    
    ;; Calculate statistics
    (let [save-times (map :save-time-ms @results)
          load-times (map :load-time-ms @results)
          sizes (map :size-bytes @results)]
      {:iterations iterations
       :avg-save-ms (/ (reduce + save-times) (count save-times))
       :avg-load-ms (/ (reduce + load-times) (count load-times))
       :avg-size-bytes (/ (reduce + sizes) (count sizes))
       :total-time-ms (+ (reduce + save-times) (reduce + load-times))})))

;; Usage
(benchmark-save-load "large-orchestral-piece" 50)
```

## Error Handling

### Agent State Monitoring

```clojure
;; Check for errors in save operations
(defn check-save-status [piece-id]
  "Check if piece was saved successfully."
  (let [state @persist/save-agent]
    (cond
      (:error state) (println "Save error:" (:error state))
      (= :saved (get state piece-id)) (println "Piece saved successfully")
      :else (println "Piece not found in save state"))))

;; Check for errors in load operations  
(defn check-load-status [piece-id]
  "Check if piece was loaded successfully."
  (let [state @persist/load-agent]
    (cond
      (:error state) (println "Load error:" (:error state))
      (contains? state piece-id) (println "Piece loaded successfully")
      :else (println "Piece not found in load state"))))
```

### Error Recovery Patterns

```clojure
(defn robust-save [piece-or-id file-path max-retries]
  "Save with automatic retry on failure."
  (loop [attempt 1]
    (persist/save-piece-async piece-or-id (persist/file-writer file-path))
    (await persist/save-agent)
    (let [state @persist/save-agent]
      (cond
        (:error state) 
        (if (< attempt max-retries)
          (do (println "Save attempt" attempt "failed, retrying...")
              (Thread/sleep 1000)
              (recur (inc attempt)))
          (throw (Exception. (str "Save failed after " max-retries " attempts: " (:error state)))))
        
        (= :saved (get state (if (string? piece-or-id) piece-or-id (:id piece-or-id))))
        (println "Save successful on attempt" attempt)
        
        :else
        (throw (Exception. "Unexpected save state"))))))
```

### Common Error Scenarios

```clojure
;; File permission errors
(try
  (persist/save-piece-async piece (persist/file-writer "/root/forbidden.ooloi"))
  (await persist/save-agent)
  (catch Exception e
    (println "Permission denied:" (.getMessage e))))

;; Network timeout errors
(try
  (persist/save-piece-async piece (persist/socket-writer "unreachable-host" 8080))
  (await persist/save-agent 5000)  ; 5 second timeout
  (catch java.util.concurrent.TimeoutException e
    (println "Network operation timed out")))

;; Serialization errors
(try
  (persist/save-piece-async {:unserializable (Object.)} (persist/buffer-writer))
  (await persist/save-agent)
  (catch Exception e
    (println "Serialization failed:" (.getMessage e))))
```

## Integration with Piece Manager

### Automatic Registration

```clojure
;; Load operations automatically register pieces in piece manager
(persist/load-piece-async (persist/file-reader "/path/to/piece.ooloi"))
(await persist/load-agent)

;; Piece is now available through piece manager
(pm/get-piece "piece-id")          ; Direct access
(pm/get-piece-ref "piece-id")      ; Get ref for transactions
```

### Save/Load Workflow Integration

```clojure
;; Complete workflow: create → store → save → remove → load
(defn piece-lifecycle-example []
  ;; 1. Create and store piece
  (let [piece (create-piece :musicians [musician1])]
    (pm/store-piece "lifecycle-demo" piece)
    
    ;; 2. Save to file  
    (persist/save-piece-async "lifecycle-demo" 
                             (persist/file-writer "/tmp/lifecycle.ooloi"))
    (await persist/save-agent)
    
    ;; 3. Remove from memory
    (pm/remove-piece "lifecycle-demo")
    (assert (nil? (pm/get-piece "lifecycle-demo")))
    
    ;; 4. Load from file (auto-registers)
    (persist/load-piece-async (persist/file-reader "/tmp/lifecycle.ooloi"))
    (await persist/load-agent)
    
    ;; 5. Verify restoration
    (assert (= piece (pm/get-piece "lifecycle-demo")))
    (println "Piece lifecycle completed successfully")))
```

### Cross-System Consistency

```clojure
;; Ensure piece manager and persistence stay synchronized
(defn save-and-verify [piece-id file-path]
  "Save piece and verify it can be restored identically."
  (let [original-piece (pm/get-piece piece-id)]
    ;; Save
    (persist/save-piece-async piece-id (persist/file-writer file-path))
    (await persist/save-agent)
    
    ;; Create backup ID and load
    (persist/load-piece-async (persist/file-reader file-path)
                             :id-fn (constantly "verification-copy"))
    (await persist/load-agent)
    
    ;; Verify
    (let [restored-piece (pm/get-piece "verification-copy")]
      (if (= original-piece restored-piece)
        (do (pm/remove-piece "verification-copy")
            (println "Save/load verification successful"))
        (throw (Exception. "Piece corrupted during save/load cycle"))))))
```

## Performance Considerations

### Large Piece Handling

```clojure
;; For pieces with hundreds of thousands of musical elements
(defn handle-large-piece [piece]
  "Process large piece efficiently with modern timewalker."
  (require '[ooloi.backend.ops.walk :as walk])
  
  ;; Use modern transducer-based traversal
  (->> (walk/timewalk piece {})
       (map-indexed (fn [idx item]
                      (when (zero? (mod idx 1000))
                        (println "Processed" idx "items"))
                      item))
       (run!)))  ; Force evaluation for side effects

;; Usage for orchestral scores - memory efficient due to lazy evaluation
(handle-large-piece orchestral-symphony)
```

### Concurrent Operations

```clojure
;; Multiple pieces can be saved/loaded concurrently
(defn batch-save-pieces [piece-map base-path]
  "Save multiple pieces concurrently."
  (let [save-futures 
        (doall 
          (for [[piece-id piece] piece-map]
            (future
              (persist/save-piece-async piece-id 
                                       (persist/file-writer 
                                         (str base-path "/" piece-id ".ooloi")))
              (await persist/save-agent)
              piece-id)))]
    
    ;; Wait for all saves to complete
    (mapv deref save-futures)))

;; Usage
(batch-save-pieces {"symphony-1" symphony1 
                   "sonata-2" sonata2 
                   "quartet-3" quartet3}
                   "~/Documents/batch-export")
```

### Memory Management

```clojure
;; Monitor agent memory usage
(defn persistence-memory-status []
  "Check memory usage of persistence agents."
  {:save-agent-ops (count @persist/save-agent)
   :load-agent-ops (count @persist/load-agent)
   :memory-usage (-> (Runtime/getRuntime)
                     (.freeMemory)
                     (/ 1024 1024)
                     (int))})

;; Clear agent history periodically for long-running systems
(defn clear-persistence-history []
  "Clear old operations from agent state."
  ;; Status is automatically cleared when new operations start
```

## Best Practices

### File Management

```clojure
;; Use consistent file extensions and naming
(defn piece-file-path [composer title opus]
  "Generate standardized file path for piece."
  (let [safe-name (-> (str composer "-" title "-" opus)
                      (clojure.string/lower-case)
                      (clojure.string/replace #"[^a-z0-9-]" "-")
                      (clojure.string/replace #"-+" "-"))]
    (str "~/Documents/pieces/" safe-name ".ooloi")))

;; Usage
(piece-file-path "Mozart" "Eine kleine Nachtmusik" "K525")
; => "~/Documents/pieces/mozart-eine-kleine-nachtmusik-k525.ooloi"
```

### Backup Strategies

```clojure
(defn create-timestamped-backup [piece-id backup-dir]
  "Create timestamped backup of piece."
  (let [timestamp (.format (java.time.LocalDateTime/now)
                           (java.time.format.DateTimeFormatter/ofPattern "yyyy-MM-dd-HH-mm-ss"))
        backup-path (str backup-dir "/" piece-id "-" timestamp ".ooloi")]
    (persist/save-piece-async piece-id (persist/file-writer backup-path))
    (await persist/save-agent)
    backup-path))

;; Automatic backup before major operations
(defn safe-piece-operation [piece-id operation-fn]
  "Perform operation with automatic backup."
  (let [backup-path (create-timestamped-backup piece-id "~/Documents/backups")]
    (try
      (operation-fn piece-id)
      (println "Operation completed. Backup at:" backup-path)
      (catch Exception e
        (println "Operation failed. Restoring from backup:" backup-path)
        (persist/load-piece-async (persist/file-reader backup-path))
        (await persist/load-agent)
        (throw e)))))
```

### Error Prevention

```clojure
;; Validate pieces before saving
(defn validate-piece-for-save [piece]
  "Validate piece structure before persistence."
  (cond
    (nil? piece) (throw (Exception. "Piece cannot be nil"))
    (not (map? piece)) (throw (Exception. "Piece must be a map"))
    (not (:id piece)) (throw (Exception. "Piece must have an :id field"))
    (empty? (:musicians piece)) (println "Warning: Piece has no musicians")
    :else true))

;; Safe save wrapper
(defn safe-save [piece file-path]
  "Save piece with validation and error handling."
  (validate-piece-for-save piece)
  (persist/save-piece-async piece (persist/file-writer file-path))
  (await persist/save-agent)
  (if-let [error (:error @persist/save-agent)]
    (throw (Exception. (str "Save failed: " error)))
    (println "Piece saved successfully to" file-path)))
```

## Troubleshooting

### Common Issues

**1. Agent Timeout Issues**
```clojure
;; Problem: Operations hang indefinitely
;; Solution: Use timeout with await
(if (await persist/save-agent 10000)  ; 10 second timeout
  (println "Save completed")
  (println "Save timed out"))
```

**2. File Permission Problems**
```clojure
;; Check file writability before saving
(defn can-write-file? [file-path]
  (let [file (java.io.File. file-path)
        parent (.getParentFile file)]
    (and (.exists parent) (.canWrite parent))))

;; Usage
(when-not (can-write-file? "/protected/path/piece.ooloi")
  (throw (Exception. "Cannot write to specified path")))
```

**3. Memory Issues with Large Pieces**
```clojure
;; Monitor memory during operations
(defn memory-safe-save [piece file-path]
  "Save with memory monitoring."
  (let [runtime (Runtime/getRuntime)
        initial-memory (.freeMemory runtime)]
    (persist/save-piece-async piece (persist/file-writer file-path))
    (await persist/save-agent)
    (let [final-memory (.freeMemory runtime)
          memory-used (- initial-memory final-memory)]
      (println "Memory used for save:" (/ memory-used 1024 1024) "MB"))))
```

### Debugging Techniques

```clojure
;; Agent state inspection
(defn debug-agent-state []
  "Print detailed agent state information."
  (println "Save agent state:")
  (clojure.pprint/pprint @persist/save-agent)
  (println "\nLoad agent state:")
  (clojure.pprint/pprint @persist/load-agent)
  (println "\nShared structure count:" (count @persist/shared-map)))

;; Operation tracing
(defn trace-save-operation [piece-or-id file-path]
  "Save with detailed tracing."
  (println "Starting save operation for:" piece-or-id)
  (println "Target file:" file-path)
  (let [start-time (System/currentTimeMillis)]
    (persist/save-piece-async piece-or-id (persist/file-writer file-path))
    (await persist/save-agent)
    (let [end-time (System/currentTimeMillis)
          duration (- end-time start-time)]
      (println "Save completed in" duration "ms")
      (debug-agent-state))))
```

## Cross-References

- **Piece lifecycle management**: See [Piece Manager Guide](PIECE_MANAGER_GUIDE.md) for in-memory operations
- **Musical structure concepts**: See [Timewalking Guide](TIMEWALKING_GUIDE.md) for piece traversal patterns  
- **Advanced concurrency**: See [Advanced Concurrency Patterns](ADVANCED_CONCURRENCY_PATTERNS.md) for STM integration
- **Architecture decisions**: See [ADR-0007: Nippy](../ADRs/0007-Nippy.md) and [ADR-0012: Persisting Pieces](../ADRs/0012-Persisting-Pieces.md) for serialization and persistence foundations

## Common Gotchas

### **1. Forgetting to await agents**
```clojure
;; ❌ Wrong - save starts but may not complete
(persist/save-piece-async piece-id (persist/file-writer "piece.ooloi"))
(println "Done!")  ; This prints immediately, save might still be running

;; ✅ Correct - wait for completion
(persist/save-piece-async piece-id (persist/file-writer "piece.ooloi"))
(await persist/save-agent)
(println "Done!")  ; This prints after save actually completes
```

### **2. Not checking for errors**
```clojure
;; ❌ Wrong - errors silently ignored
(persist/save-piece-async piece-id (persist/file-writer "piece.ooloi"))
(await persist/save-agent)

;; ✅ Correct - check agent state for errors
(persist/save-piece-async piece-id (persist/file-writer "piece.ooloi"))
(await persist/save-agent)
(when-let [error (:error @persist/save-agent)]
  (throw (Exception. (str "Save failed: " error))))
```

### **3. Mixed up agent states**
The save and load agents are separate - check the right one!
```clojure
;; ❌ Wrong - checking wrong agent
(persist/load-piece-async (persist/file-reader "piece.ooloi"))
(await persist/load-agent)
(if (:error @persist/save-agent)  ; Wrong agent!
  (println "Load failed"))

;; ✅ Correct - check matching agent
(persist/load-piece-async (persist/file-reader "piece.ooloi"))
(await persist/load-agent)
(if (:error @persist/load-agent)  ; Correct agent
  (println "Load failed"))
```

## Summary

Ooloi's persistence system provides **professional-grade async I/O** for musical pieces:

**Key strengths:**
- **Non-blocking**: UI stays responsive during large file operations
- **Multiple backends**: File, buffer, and network I/O with same API
- **Error resilient**: Comprehensive error handling and recovery
- **High performance**: Nippy binary serialization with compression
- **Integration ready**: Seamless piece manager integration

**Best practices recap:**
1. **Always await agents** before assuming operations complete
2. **Check agent state** for errors after operations
3. **Use appropriate backends** (file for persistence, buffer for testing, socket for collaboration)
4. **Validate pieces** before saving to catch issues early
5. **Handle errors gracefully** with try/catch and agent state checking

**Quick reference:**
```clojure
;; Save pattern
(persist/save-piece-async piece-id (persist/file-writer path))
(await persist/save-agent)
(when-let [error (:error @persist/save-agent)]
  (handle-error error))

;; Load pattern  
(persist/load-piece-async (persist/file-reader path))
(await persist/load-agent)
(when-let [error (:error @persist/load-agent)]
  (handle-error error))
```

This system handles everything from simple save/load to complex collaborative workflows and performance testing. See the Real-World Workflows section for production-ready examples.

## Next Steps

- **Format integration**: Learn about MusicXML and MIDI export (future development)
- **Network protocols**: Explore distributed piece sharing and collaboration
- **Plugin development**: Use persistence for plugin data management  
- **Performance optimization**: Advanced chunking and streaming techniques for massive scores