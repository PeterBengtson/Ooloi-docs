# 🟡 Piece Manager Guide: Storage, Retrieval, and Lifecycle Management

> 💡 **CONVENIENCE TIP**: While you CAN use `alter`, `ref`, and low-level STM operations, 
> VPD-based API operations like `(api/set-measure vpd piece-id 5 measure)` are much easier. 
> See [API Convenience](POLYMORPHIC_API_GUIDE.md#api-convenience-why-use-vpd-operations) for details.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Understanding the Piece Manager](#understanding-the-piece-manager)
- [How Pieces Are Registered](#how-pieces-are-registered)
- [The PieceManagerOps Protocol](#the-piecemanagerops-protocol)
- [Function Reference](#function-reference)
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

The piece manager is Ooloi's central system for holding the musical pieces a session is working with. It handles storage, retrieval, reference management, and provides the foundation for all piece-based operations throughout the system.

**Key responsibilities:**
- **Storage**: Maintaining a registry of the pieces open in this session, keyed by piece id
- **Reference management**: Converting between pieces, piece-refs, and piece-IDs
- **Session state**: Holding what a session knows about each piece — where it came from, the value it is clean with respect to, whether it has unsaved changes — beside the piece itself
- **Location**: Keeping the persistent catalogue that maps every opened or saved piece's UUID to where it lives
- **API integration**: Providing the foundation for VPD operations

**What it does not decide** is when a piece should stop being managed. The piece manager stores, resolves and removes on request; the policy that asks for a removal — close-on-last-release — lives in `ooloi.backend.ops.piece-subscription-lifecycle` (see [Lifecycle Patterns](#lifecycle-patterns)).

**Note**: In production a piece's id is the UUID embedded in the piece itself ([ADR-0012 §4](../ADRs/0012-Persisting-Pieces.md#4-piece-identity-uuid-as-primary-key)): `create-piece` mints it, and every production registration keys the entry by it. Examples in this guide that use mnemonic strings such as `"beethoven-9th"` go through `store-piece`, the test and script path, which accepts any string as an id.

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
;; (release-store!). The store lives exactly as long as the component, so nothing
;; in it can survive into the next run to be found stale there.
(def ^:private piece-store (ref nil))

;; The store ref it holds maps piece-id -> a registry entry of the form
;; {:ref piece-ref :provenance {:path :modified} :baseline piece :dirty true};
;; each piece is its own ref.

;; Inside the namespace, while a piece-manager component is running:
@@piece-store ; => {"5a3c1e7f-…" {:ref        #<Ref @piece>
              ;                  :provenance {:path "…/my-symphony.ooloi" :modified 1718900000000}
              ;                  :baseline   piece}}
```

**Why a ref handle rather than an atom?** Every store mutation already happens inside `dosync`: `store-piece`, `remove-piece`, registration and the dirty-flag operations all `alter` the store ref. Making the handle a **ref** too keeps the whole access path under a single STM discipline: the handle read inside a transaction is transactionally consistent by construction, and there is no second reference type to reason about. The handle is populated by the component, so pure-model code — `resolve-into-piece-ref`, `get-piece-ref` — reaches the store through it with no server in scope. The store is created by `init-store!` (called from the component's `init-key`) and released by `release-store!` (from `halt-key!`); there is no JVM-global remnant.

**The registry entry.** Each value in the store is a map rather than a bare ref, holding what a session knows about a piece:

| Key | What it is |
|---|---|
| `:ref` | the piece's STM ref (the dual-ref system above) |
| `:provenance` | where this piece was opened from or last saved to, and when |
| `:baseline` | the value this piece is currently clean with respect to |
| `:dirty` | present iff the piece has unsaved changes |

The `:provenance` — the file path and modification time — is recorded on every entry made through `open-piece`, `open-by-uuid` or `new-piece` (which records `{:path nil}`, the marker of a piece never saved), and replaced by each completed save. An entry made by `store-piece` has none. Provenance is how the Piece Manager distinguishes an idempotent reopen of the same file (same embedded UUID, same provenance) from a variant collision (same UUID, different provenance) — see [How Pieces Are Registered](#how-pieces-are-registered) and [ADR-0012](../ADRs/0012-Persisting-Pieces.md).

The `:baseline` and `:dirty` pair answers "does this piece have unsaved changes", and they answer it in two different ways because the question arrives in two different shapes. Editing forward, the flag is *accumulated*: the write funnel sets it on any change and a save clears it, with no comparison anywhere ([ADR-0052](../ADRs/0052-Change-Detection-and-Event-Generation.md) §5). But a piece can also arrive at a state without editing into it — an undo or redo replaces the whole value — and then the flag has to be *derived*, by comparing that value with `:baseline`. The baseline is set when the piece enters the manager (the loaded value on open, the fresh value on new) and replaced by a completed save only when the piece is still `=` to the snapshot that save wrote. A piece edited while its write was in flight keeps its earlier baseline and stays dirty, rather than being recorded clean against a value it no longer has.

Everything here shares the ref's identity and lifetime — dropped with it when the entry is removed — rather than living in parallel structures to keep in sync. Session facts about a piece belong where the piece does, and expire when it does, which is why the entry carries them as keys rather than the Piece Manager keeping side tables. Distinct from all of it, the **persistent catalogue** — a second, durable store the same component owns, mapping every piece's UUID to its storage location whether or not it is currently open — is described in [The Persistent Catalogue](#the-persistent-catalogue) below.

### Why This Architecture?

**Individual piece refs** enable:
- **Concurrent access**: Multiple threads can work on different pieces simultaneously
- **Isolated transactions**: Changes to one piece don't block access to others
- **Efficient updates**: Only the specific piece being modified needs coordination

**Central piece store** provides:
- **Global registry**: Find any piece by ID across the entire system
- **Session state**: Provenance, baseline and dirty flag held beside each piece
- **API consistency**: Uniform access patterns for all piece operations

### The Three Forms of Piece References

The piece manager handles three different ways to reference a piece:

```clojure
;; 1. Piece object (immutable value)
(def piece-obj (create-piece :musicians [musician1] :layouts [layout1]))

;; 2. Piece ref (STM reference)
(def piece-ref (ref piece-obj))

;; 3. Piece ID (string identifier) — in production, the piece's embedded UUID
(def piece-id (:id piece-obj))
```

### The Persistent Catalogue

The store above is **in-memory**: it holds the pieces open *right now*, and it dies when the component halts. Beside it the same component owns a **second, durable store — the persistent catalogue** — mapping every piece's embedded UUID to *where that piece lives*, for every piece ever opened or saved, whether or not it is currently open. The in-memory store answers "is this piece open, and where is its live ref?"; the catalogue answers "where on disk is the piece with this UUID?" — the durable resolution needed to re-open a piece that is no longer in memory. Because the in-memory registry empties as clients disconnect (close-on-last-release, [ADR-0022](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md)) and is gone entirely at shutdown, without the catalogue there would be nothing to resolve a UUID against on relaunch. Session restore, which reopens the pieces that were open at the last quit, is what consumes it.

**The value is a tagged locator, never a bare path.** Each entry maps a UUID to `{:kind :filesystem :path "…"}`:

```clojure
;; The persisted catalogue EDN — every piece's UUID → its storage locator
{"5a3c1e7f-…" {:kind :filesystem :path "…/my-symphony.ooloi"}
 "b71f9d02-…" {:kind :filesystem :path "…/string-quartet.ooloi"}}
```

The `:kind` tag is what lets `{:kind :s3 …}` or `{:kind :db …}` be added with **no schema change** — storage-backend opacity ([ADR-0012](../ADRs/0012-Persisting-Pieces.md)); only `:filesystem` exists. The locator is **backend-internal**: it never crosses to the frontend, which holds UUIDs (identity) and ephemeral per-session tokens (access), never a storage location.

**Writes ride the provenance recorders — no new seam.** The two operations that record a piece's provenance in the piece-manager — `register-opened-piece` (on open) and `record-piece-provenance` (on save) — each *also* record `uuid → {:kind :filesystem :path}`, whenever the provenance carries a path; a new piece's `{:path nil}` writes no entry. The same fact — a piece's location — is recorded at two durabilities in one place: the ephemeral provenance beside the live ref, and the durable entry in the catalogue.

**Persistence is on every mutation, not at halt.** Each write hands the whole catalogue map to a **writer agent** that writes it immediately (the same writer-agent pattern the Instrument Library component uses) to `catalogue.edn` in the directory `(get-platform-directory "Ooloi" "catalogue")`; `catalogue-file` returns that file. The component's `init-key` calls `init-catalogue!`, which reads the file back — an absent file is an empty catalogue — and that is how the catalogue survives a restart. Its `halt-key!` calls `release-catalogue!`, which waits for any in-flight write through `await-catalogue-writer!` and then clears the handle. The halt only *awaits*; it is not the write trigger, so a crash never loses a committed entry.

**The write is atomic**, through `ooloi.shared.ops.atomic-io` — the shared writer every file under the platform directory goes through, which stages to a sibling temp file and moves it over the target. Two properties of this file make that load-bearing rather than tidy: the whole map is rewritten on every mutation, so a write that failed part-way would lose the location of *every* piece ever opened or saved; and the failure is deliberately swallowed (below), with the file read back unguarded at `init-key`, so nothing downstream would notice a truncated one until a piece went missing from its own catalogue.

**The catalogue is best-effort secondary durability.** A catalogue write must never fail the open or save it rode in on. The write runs on the writer agent under `:error-mode :continue`, so a disk error cannot escape `register-opened-piece` / `record-piece-provenance` onto the wire — the operation still returns its piece. The catalogue is a convenience for later resolution, not a correctness dependency of opening or saving.

**Not escaping is not the same as not being recorded.** The agent carries an `:error-handler` that writes the throwable through the failure boundary — the general rule for every agent in the system, specified in [ADR-0017 §Surfacing Unexpected Runtime Failures](../ADRs/0017-System-Architecture.md), where an agent is the third kind of thread nobody is waiting on. It is load-bearing rather than decorative: `:error-mode :continue` *on its own* discards the exception outright. Without the handler, a catalogue that had quietly stopped being written would surface days later as a piece missing from its own catalogue, with no account anywhere of why. The write stays silent to the caller; it does not stay silent to the record.

**Reads: `catalogue-resolve` and `open-by-uuid`.** The raw read is `catalogue-resolve [uuid] → locator | nil`, backend-internal (the locator stays inside the backend). The consumer-facing read is **opening a piece by its UUID**: `open-by-uuid` resolves the UUID to a locator, loads the file through `open-piece`'s shared load tail (read → deserialize → validate against the typed-failure taxonomy), **verifies the loaded piece's embedded UUID matches the one requested** — before registering, so a mismatched piece is never registered, guarding against the file at that location having become a *different* piece — and only then registers it under the embedded UUID. It returns the piece-id, or a typed `{:open-failure …}`: `:piece-not-in-catalogue` (the UUID has no entry), `:piece-not-found` (the entry's file is gone — a stale locator), `:piece-uuid-mismatch` (the file holds a different piece), plus the shared load-tail failures. `open-by-uuid` is exposed through `api`/`SRV` — session restore sends a UUID and gets back a piece-id or a typed failure it maps to a notification; `catalogue-resolve` is not exposed. The operation, its one-directional relationship to `open-piece`, and the full taxonomy are specified in [ADR-0051](../ADRs/0051-Filesystem-Operations-Real-and-Virtual.md).

## How Pieces Are Registered

In the running application a piece enters the registry through one function, `register-opened-piece` (a `PieceManagerOps` operation, [below](#the-piecemanagerops-protocol)), and three API operations reach it:

- **`new-piece`** mints a piece with `create-piece` — a fresh embedded UUID and a blank title — and registers it with provenance `{:path nil}`. Its baseline is that fresh value. Because the provenance has no path, no catalogue entry is written; the first Save records a real location, and the catalogue entry with it.
- **`open-piece`** and **`open-by-uuid`** share a load tail in `ooloi.shared.ops.filesystem` that ends by registering the loaded piece with provenance `{:path … :modified …}` taken from the file. Its baseline is the loaded value, and the registration writes the catalogue entry.

All three return the piece's embedded UUID, which is its piece-id from then on.

**Registering is idempotent for the same file and refuses a different one.** When the UUID is not yet managed, the piece is stored with its provenance and baseline. When it is managed with the same provenance, the call returns the UUID and leaves the entry alone, so the live ref and any unsaved edits survive the reopen. When it is managed with *different* provenance — a variant collision, which a copy of a file made outside Ooloi produces — `register-opened-piece` throws an `ex-info` whose data is `{:collision {:id … :existing … :incoming …}}`, the incumbent's provenance and the newcomer's side by side, and the incumbent is not overwritten. The open path maps that throw to `{:open-failure :piece-uuid-collision}`. The throw happens before the catalogue write, so a collision writes no catalogue entry.

**`store-piece` is the test and script path.** It stores any value under any string id as `{:ref (ref piece)}` and nothing else — no provenance, no baseline, no catalogue entry — replacing any entry already under that id. No application source in any of the three projects calls it; tests and REPL scripts do.

## The PieceManagerOps Protocol

`PieceManagerOps` is declared in `ooloi.shared.interfaces` and implemented by the piece manager. It is the seam through which shared code — the filesystem operations, the change-detection funnel, the structure projection — works on managed pieces with no compile-time dependency on the backend: dependency inversion, as with `PieceResolver`, rather than a runtime lookup. It has nine operations:

| Operation | What it does |
|---|---|
| `register-opened-piece [piece provenance]` | registers a piece under its embedded UUID ([above](#how-pieces-are-registered)) and returns the UUID |
| `record-piece-provenance [piece-id provenance]` | replaces the entry's provenance after a completed write, records the catalogue entry when the provenance has a path, and returns the piece-id |
| `get-piece-provenance [piece-id]` | the entry's provenance, or nil. Backend-internal: `:path` is an OS path, so this is not exposed through `SRV` |
| `catalogue-resolve [uuid]` | the catalogue's locator for the UUID, or nil |
| `mark-piece-dirty! [piece-id]` | sets `:dirty` |
| `clear-piece-dirty! [piece-id]` | removes `:dirty` |
| `piece-dirty? [piece-id]` | true iff `:dirty` is set; false for a clean piece or an unmanaged id |
| `record-piece-baseline! [piece-id piece]` | replaces `:baseline` with the given value; returns nil |
| `reconcile-piece-dirty! [piece-id]` | sets `:dirty` to whether the current value differs from `:baseline`, compared with `=` |

**The dirty-flag mutators report whether they flipped the flag.** `mark-piece-dirty!`, `clear-piece-dirty!` and `reconcile-piece-dirty!` each return true only when the call changed the flag, and nil otherwise, so the caller emits `:piece-dirty-changed` exactly once per change and never for a call that changed nothing. None of them, and not `record-piece-baseline!` either, creates an entry for an id the manager does not hold.

**Who calls them:**

- `register-opened-piece` — `new-piece`, and the shared load tail of `open-piece` and `open-by-uuid`.
- `mark-piece-dirty!` — the write funnel: `detect-change!` in `ooloi.shared.ops.change-detection`, inside the `vpd-transact` transaction, on any write that changes the piece. It marks the piece by the value's own `:id`.
- `record-piece-provenance`, `record-piece-baseline!` and `clear-piece-dirty!` — `save-piece`, after its write completes; the last two only when the piece is still `=` to the snapshot written (see [What follows a completed write](PIECE_PERSISTENCE_GUIDE.md#what-follows-a-completed-write)).
- `reconcile-piece-dirty!` — undo and redo, through the backend's `restore-piece!`, which replaces the whole value and so has to derive the flag rather than accumulate it.
- `get-piece-provenance` — a plain `save-piece` (the location to rewrite), `piece-location`, and the `get-piece-structure` projection (the `:filename` it reports).
- `piece-dirty?` — the `get-piece-structure` projection, which reports it as the virtual `:dirty` field the piece window's title reads.
- `catalogue-resolve` — `open-by-uuid`.

**Defaults where no manager runs.** `ooloi.shared.interfaces` extends `mark-piece-dirty!`, `clear-piece-dirty!`, `record-piece-baseline!` and `reconcile-piece-dirty!` to `Object` as no-ops returning nil, so the write funnel can run in frontend-local mutations and pure-shared tests with no backend loaded. The piece manager's own extension to `Object` loads after it and replaces those defaults with the real implementations of all nine operations.

## Function Reference

`ooloi.backend.ops.piece-manager` provides the functions below, and in addition implements the `PieceResolver` protocol (see [Resolving to a value or a ref](#resolving-to-a-value-or-a-ref-resolve-into-piece-and-resolve-into-piece-ref)), the `PieceManagerOps` protocol ([above](#the-piecemanagerops-protocol)), and the backend method of the `new-piece` API operation.

### `store-piece`
```clojure
(pm/store-piece piece-id piece) ; => piece
```
- **Purpose**: Stores a piece with given ID — the test and script path
- **Behavior**: Replaces any existing entry under that ID with `{:ref (ref piece)}`, so provenance, baseline and dirty flag are not carried over
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

### `get-all-stored-pieces` and `all-piece-refs`
```clojure
(pm/get-all-stored-pieces) ; => [piece …]
(pm/all-piece-refs)        ; => [piece-ref …]
```
- **Purpose**: Every piece currently in the store — as values, or as their live refs
- **Use case**: The hash-consing cache daemon uses both. It needs the refs themselves because it alters them directly within its own transaction when it replaces a piece with its canonicalised equivalent

### Lifecycle functions
```clojure
(pm/init-store!)             ; publish a fresh, empty store on the handle
(pm/release-store!)          ; clear the handle
(pm/init-catalogue!)         ; load the catalogue from catalogue.edn (empty if absent)
(pm/release-catalogue!)      ; await any in-flight catalogue write, then clear the handle
(pm/await-catalogue-writer!) ; block until any in-flight catalogue write has finished
(pm/catalogue-file)          ; => the catalogue.edn File under the platform directory
```
- **Purpose**: The piece-manager component's `init-key` calls `init-store!` and `init-catalogue!`; its `halt-key!` calls `release-store!` and `release-catalogue!`. Tests call `await-catalogue-writer!` before reading the persisted EDN back

## Basic Operations

### Creating and Registering Pieces

```clojure
(require '[ooloi.shared.api :as api])

;; Production: new-piece mints a piece with a fresh embedded UUID, registers it
;; under that UUID with provenance {:path nil}, and returns the UUID
(def piece-id (api/new-piece))     ; => "5a3c1e7f-…"

;; From the frontend, the same operation over gRPC:
;; (SRV/new-piece)
```

Opening a file registers its piece the same way; see [How Pieces Are Registered](#how-pieces-are-registered). Tests and scripts that hold their own piece value store it directly:

```clojure
(require '[ooloi.backend.ops.piece-manager :as pm])
(require '[ooloi.shared.api :as api])

;; create-piece embeds a fresh UUID as :id and supplies default time signature,
;; key signature and tempo
(def piece (api/create-piece :musicians [musician1 musician2]
                             :layouts [layout1]))

;; Keyed by its embedded UUID, as production keys it
(pm/store-piece (:id piece) piece)
;; => returns the piece object

;; Or by a mnemonic id, in a test
(pm/store-piece "beethoven-9th" piece)
;; => returns the piece object
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

`store-piece` is for tests and scripts (see [How Pieces Are Registered](#how-pieces-are-registered)):

```clojure
;; Basic storage
(pm/store-piece "piece-id" piece)

;; Overwrite existing (always replaces the whole entry)
(pm/store-piece "piece-id" updated-piece)

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
(defn my-operation [piece-thing musician]
  ;; 😎 Easy way: VPD operations resolve whichever form they are handed
  (api/add-musician [] piece-thing musician))

;; All of these work with VPD operations:
(my-operation "stored-piece-id" musician)    ; Uses stored piece
(my-operation some-piece-object musician)    ; Uses object directly

;; 😓 Hard way: You could do this, but why?
;; (let [piece-ref (pm/get-piece-ref piece-thing)]
;;   (dosync
;;     (alter piece-ref update-something)))
;; More code, more complexity, more chances for bugs.
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
;; 😎 Pattern 1: Work with managed pieces using VPD operations
(api/add-musician [] piece-id new-musician)  ; Automatic transaction handling

;; 😎 Pattern 2: Work with an unmanaged piece value using VPD operations
(let [temp-piece (api/create-piece)]
  (api/add-musician [] temp-piece new-musician))  ; Returns the new piece value

;; 😎 Pattern 3: In a test or script, store a value built outside the manager
(let [temp-piece (api/create-piece)
      modified-piece (api/add-musician [] temp-piece new-musician)]
  (pm/store-piece (:id modified-piece) modified-piece))
```

## Lifecycle Patterns

### Complete Piece Lifecycle

In the application, over `SRV`:

```clojure
;; 1. Creation and registration
(def piece-id (SRV/new-piece))

;; 2. Modification — the write funnel marks the piece dirty
(SRV/add-musician [] piece-id organist)
(SRV/set-tempo [] piece-id [0 0] {:bpm 120 :beat-unit 1/4})

;; 3. First Save — writes the piece and records its location; clears the dirty
;;    flag if the piece is unchanged since the snapshot was taken
@(SRV/save-piece piece-id dir-token "choir.ooloi")

;; 4. Closing the last window that holds the piece removes it (below)
```

In a test or script holding its own piece:

```clojure
;; 1. Creation
(def piece (api/create-piece :musicians [soprano alto tenor bass]))

;; 2. Storage
(pm/store-piece "satb-choir" piece)

;; 3. Modification
(api/add-musician [] "satb-choir" organist)
(api/set-tempo [] "satb-choir" [0 0] {:bpm 120 :beat-unit 1/4})

;; 4. Retrieval for processing
(def final-piece (pm/get-piece "satb-choir"))

;; 5. Cleanup
(pm/remove-piece "satb-choir")  ; => the removed piece
```

The script's step 5 is the *programmatic* removal — a test or script that stored a piece and is done with it. **In the running application, pieces are never removed by hand.** A piece is registered when a client first opens it (New or Open) and removed automatically the moment its *last* subscriber leaves — a deterministic **close-on-last-release** driven by the subscriber count the connection registry keeps, with a disconnect treated as unsubscribe-from-every-piece-then-leave. A piece thus lives exactly as long as some client holds it: one client closing its window keeps the piece alive for any other still holding it, and the last client leaving closes it (the saved file re-loaded on a later reopen, not returned from memory). This is what makes Piece-Manager-mediated sharing identical for a single user and for a collaboration.

The check is `close-if-unheld!` in `ooloi.backend.ops.piece-subscription-lifecycle`. Given a snapshot of the connection registry, a piece-id and the undo manager, it does nothing while any client still subscribes to the piece; otherwise it calls `remove-piece` and drops the piece's undo/redo stacks through the undo manager's `remove-resource!`. The stacks go with the piece because each entry in them holds a whole piece value and nothing can reach them after the close. On a server built without an undo manager the argument is nil and the clearing is skipped. A clean unsubscribe reaches `close-if-unheld!` through `unsubscribe!`; both disconnect paths — the Disconnect RPC handler and the channel-close backstop — reach it through `release-client-pieces!`, once for each piece the departed client held. The policy is [ADR-0022 §Event Subscription Management](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md#event-subscription-management) (Piece Lifetime — Close-on-Last-Release), and the disconnect handlers are in [ADR-0024 §Event Notification Architecture](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md#event-notification-architecture).

## Working with Piece IDs

### ID Conventions

**A piece's id is the UUID embedded in it.** `create-piece` sets `:id` to a fresh string UUID unless one is supplied, the id is written into the file with the rest of the piece, and every production registration keys the entry by it, so a file and the registry cannot come to disagree about which piece is which ([ADR-0012 §4](../ADRs/0012-Persisting-Pieces.md#4-piece-identity-uuid-as-primary-key)). There is no separate id to generate or keep track of:

```clojure
(def piece (api/create-piece))
(:id piece)                      ; => "550e8400-e29b-41d4-a716-446655440000"

;; new-piece returns the UUID of the piece it minted
(def piece-id (api/new-piece))
```

**Mnemonic strings belong to tests and scripts.** `store-piece` accepts any string, so a test may store a piece under `"beethoven-op67-mvt1"` for readability. Such an entry is outside what the write funnel sees as the piece's identity: the funnel marks a piece dirty by the value's own `:id`, so a piece stored under any other id never shows as dirty (see [Troubleshooting](#troubleshooting)).

## Integration with VPD Operations

### How Piece Manager Enables VPD Operations

Every VPD operation uses the piece manager internally:

```clojure
;; When you call:
(api/set-measure [:musicians 0 :instruments 0 :staves 0] "piece-id" 5 new-measure)

;; Internally, this happens:
;; 1. piece-ref <- (resolve-into-piece-ref "piece-id") ; Piece manager lookup
;; 2. (dosync (alter piece-ref ...))                   ; STM transaction
;; 3. VPD navigation to target location                ; VPD system
;; 4. Update with validation                           ; Type system
;; 5. detect-change! marks the piece dirty and         ; Change detection
;;    schedules any change event for commit
```

### Piece Manager in Custom Operations

```clojure
(require '[ooloi.shared.ops.change-detection :as cd])

(defn batch-update-measures [piece-id measure-updates]
  "Update multiple measures atomically using VPD operations."
  ;; 😎 Easy way: compose VPD operations in one coalescing scope. It opens the
  ;; single transaction they all join, and their structural changes emit one
  ;; :piece-structure-changed. A bare dosync is atomic too, but each structural
  ;; write inside it emits its own event (ADR-0052 §4).
  (cd/with-coalesced-structural-change
    (doseq [[vpd measure-num measure] measure-updates]
      (api/set-measure vpd piece-id measure-num measure))))

;; Usage with stored piece
(batch-update-measures "symphony"
                      [[[:musicians 0 :instruments 0 :staves 0] 5 measure1]
                       [[:musicians 1 :instruments 0 :staves 0] 5 measure2]])

;; 😓 Hard way: You could manually manage piece-ref, but why?
;; (let [piece-ref (pm/get-piece-ref piece-id)]
;;   (dosync
;;     (doseq [[vpd measure] measure-updates]
;;       (alter piece-ref ...))))
;; Much more verbose and error-prone — and it bypasses change detection.
```

## Best Practices

### 1. Let the Piece Carry Its ID

```clojure
;; Good: register through the operation that mints or loads the piece;
;; the id it returns is the piece's embedded UUID
(def piece-id (api/new-piece))

;; Avoid: assigning ids alongside pieces outside tests and scripts —
;; an id that is not the piece's :id is invisible to the write funnel
```

### 2. Error Handling

```clojure
;; A stale or unknown id throws where it is used, carrying the id
(defn safe-piece-operation [piece-id operation]
  (try
    (operation (pm/get-piece-ref piece-id))
    (catch clojure.lang.ExceptionInfo e
      {:error :piece-not-found :piece-id (:piece-id (ex-data e))})))
```

### 3. Resource Management

```clojure
;; In a test or script, remove what you stored
(defn process-temporary-piece [source-piece-id]
  (let [temp-id (str source-piece-id "-temp-" (System/currentTimeMillis))
        source-piece (pm/get-piece source-piece-id)
        processed-piece (expensive-processing source-piece)]

    ;; Store temporarily for complex operations
    (pm/store-piece temp-id processed-piece)

    (try
      ;; Perform operations that need stored piece
      (complex-multi-step-operation temp-id)

      ;; Return final result
      (pm/get-piece temp-id)

      (finally
        ;; Cleanup temporary piece
        (pm/remove-piece temp-id)))))
```

## Common Patterns

### Pattern 1: Transforming a Managed Piece

A change to a managed piece goes through VPD operations, so that the write funnel sees it — it marks the piece dirty and emits the change events subscribed windows refetch on — and, called over `SRV`, the server records an undo entry for it:

```clojure
(SRV/add-musician [] piece-id organist)
(SRV/set-tempo [] piece-id [0 0] {:bpm 96 :beat-unit 1/4})
```

Replacing the whole value with `store-piece` does none of that, and replaces the entry itself: the new entry has a new ref and no provenance, baseline or dirty flag. It is a pattern for values a test or script owns:

```clojure
(defn transform-piece-pipeline [piece-id transformations]
  "Apply a series of transformations to a stored piece's value (tests and scripts)."
  (let [transformed-piece (reduce (fn [piece transform-fn]
                                    (transform-fn piece))
                                  (pm/get-piece piece-id)
                                  transformations)]
    (pm/store-piece piece-id transformed-piece)))
```

### Pattern 2: Copies

A copy of a piece on disk is `clone-piece` — Save As — which writes a clone with fresh ids, its own piece UUID among them, to a chosen destination without registering it. See the [Piece Persistence Guide](PIECE_PERSISTENCE_GUIDE.md#how-a-save-works).

## Troubleshooting

### Common Issues

**1. Piece not found errors**
```clojure
;; Problem: Piece ID doesn't exist
(pm/get-piece "nonexistent")      ; Returns nil
(pm/get-piece-ref "nonexistent")  ; Throws ExceptionInfo, ex-data {:piece-id "nonexistent"}

;; Solution: Check existence where absence is expected
(when-let [piece (pm/get-piece piece-id)]
  (process-piece piece))
```

**2. A stored piece never shows as dirty**
```clojure
;; Problem: the piece was stored under an id other than its embedded :id
(pm/store-piece "my-test-piece" piece)
(api/add-musician [] "my-test-piece" organist)
(i/piece-dirty? "my-test-piece")   ; => false — the funnel marked (:id piece), which is not managed

;; Solution: store under the piece's own id when dirtiness matters
(pm/store-piece (:id piece) piece)
```

**3. Concurrent modification conflicts**
```clojure
;; Problem: Multiple threads modifying same piece
;; Solution: VPD operations handle STM coordination automatically for you

;; 😎 Easy way: use VPD operations directly
(api/add-musician [] piece-id new-musician)                    ; Automatic STM coordination
(api/set-tempo [] piece-id [0 0] {:bpm 120 :beat-unit 1/4})    ; Automatic STM coordination
```

### Debugging Tips

```clojure
(require '[ooloi.shared.interfaces :as i])

;; Check piece store contents. piece-store is private, so reach it through its var:
;; var -> handle ref -> store ref -> id->entry map
(defn list-stored-pieces []
  (->> @@@#'pm/piece-store
       keys
       sort))

;; Piece statistics
(defn piece-info [piece-id]
  (when-let [piece (pm/get-piece piece-id)]
    {:musicians  (count (:musicians piece))
     :layouts    (count (:layouts piece))
     :provenance (i/get-piece-provenance piece-id)
     :dirty?     (i/piece-dirty? piece-id)}))
```

## Cross-References

### **Core Concepts**
- **VPD operations**: See [Polymorphic API Guide](POLYMORPHIC_API_GUIDE.md#api-convenience-why-use-vpd-operations) for VPD-based operations used in piece management
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
