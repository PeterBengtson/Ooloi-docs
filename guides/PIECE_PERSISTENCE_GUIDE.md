# 🟢 Piece Persistence Guide

How a piece reaches disk and comes back, and why the two directions are shaped differently.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [How a save works](#how-a-save-works)
- [How an open works](#how-an-open-works)
- [Failures are typed data](#failures-are-typed-data)
- [What follows a completed write](#what-follows-a-completed-write)
- [Serialisation](#serialisation)
- [The I/O backends](#the-io-backends)
- [Integration with the Piece Manager](#integration-with-the-piece-manager)
- [Benchmarking](#benchmarking)
- [Autosave and versioning](#autosave-and-versioning)
- [Common gotchas](#common-gotchas)
- [Cross-references](#cross-references)

## Overview

Reading and writing a piece are not symmetrical operations, and the difference is deliberate.

**A read** has someone waiting for it: a window is about to open. It runs off the JavaFX thread, several can run at once, and the concurrency is bounded so the interface always has a core to draw with.

**A write** has nobody waiting. It splits in two: taking a snapshot of the score is instantaneous, compressing and writing it is not. So the snapshot is taken the moment the user asks and the write is queued behind whatever else is being written. The user's Save returns at once however large the score, and the file records the score as it was when they asked rather than when the disk got round to it.

Two properties follow, and both are requirements rather than conveniences:

- **Nothing waits on the JavaFX thread** — not a read, not a write, not for a millisecond.
- **Editing continues during a write.** The queue holds an immutable snapshot, so there is nothing for an edit to contend with. This is [ADR-0010](../ADRs/0010-Pure-Trees.md) paying for itself.

## Prerequisites

- Basic Clojure: data structures and functions
- [Piece Manager Guide](PIECE_MANAGER_GUIDE.md) — how pieces are stored and identified
- Optional: familiarity with [Nippy](https://github.com/taoensso/nippy)

## How a save works

A save is two operations wearing one name, differing by orders of magnitude in cost:

| Phase | Cost | Where it runs |
|---|---|---|
| Resolve the piece to a snapshot | a ref deref — microseconds | the calling thread, immediately |
| Validate the filename, resolve the destination | negligible | the calling thread, immediately |
| Freeze and write | seconds for a large score | the save queue, later |

`save-piece` and `clone-piece` therefore return a **handle**. Deref it to obtain the outcome:

```clojure
;; Save to a chosen destination (Save As, or the first Save of a new piece)
(let [handle (SRV/save-piece piece-id dir-token "symphony.ooloi")]
  ;; returns immediately — the write is queued
  @handle)                                  ; => true, or {:save-failure <type>}

;; Plain save: re-write to the location already recorded on the piece
@(SRV/save-piece piece-id)

;; Save As: clone with fresh structural ids, then write the clone
@(SRV/clone-piece piece-id dir-token "variant.ooloi")
```

A caller with nothing to wait for ignores the handle. A caller that must sequence on the write — the quit pass, which cannot close a window until its piece is written — derefs it.

### Why the split falls exactly there

**The snapshot must be immediate**, or a save queued behind a large one silently picks up edits made after the user pressed Save.

**Destination resolution cannot leave the calling thread.** Resolving a dir-token reads the client-id from the gRPC context, and that context does not survive a hop to another thread. Which is also where the two validation failures belong: `:invalid-filename` and `:invalid-destination` are known before any I/O is attempted. They are delivered through the handle all the same, already resolved, so a caller discriminates one shape rather than two.

### Writes are serialised; reads are not

The save queue is a single thread, so writes happen **one at a time, in the order asked**. Parallel saves are not desirable rather than merely unnecessary: nobody waits on a write, so parallelism reduces no latency anyone perceives, while two writes to one path would resolve arbitrarily instead of in the order asked, and competing large writes divide disk throughput so both take longer than either alone. Safety is not the reason — a staged write leaves a file complete or unchanged, never partial.

Reads are the opposite, and the asymmetry is about the shared target rather than the direction: two reads of a file cannot interfere, two writes to one path can.

The queue is the `:ooloi.backend.components/save-queue` component. It is the executor itself, published on a handle in `ooloi.shared.ops.persistence` so `save-piece` can reach it with nothing in hand, and its `halt-key!` drains — a write already running finishes before the halt returns, so a queued save is not lost at quit.

A failing write cannot disable it. Submissions are one-shot rather than periodic, so nothing suppresses later executions; the future captures `Throwable` rather than `Exception`, so not even an `OutOfMemoryError` escapes to kill the worker; and the escaping throwable still reaches [ADR-0017](../ADRs/0017-System-Architecture.md)'s surfacing boundary.

## How an open works

An open is one operation, and the caller waits for it — just never on the JavaFX thread:

```clojure
(SRV/open-piece file-token)     ; => piece-id string, or {:open-failure <type>}
(SRV/open-by-uuid piece-uuid)   ; => piece-id string, or {:open-failure <type>}
```

Application code does not call these directly. It goes through `open-piece-token!`, the single unbypassable entry point, which dispatches the call on the shared pool and marshals the result back to the JavaFX thread — raising the mapped error notification on a typed failure, invoking a success continuation on a piece-id.

**The shared pool is the right executor, and that is load-bearing.** Each in-flight read holds one of its `cores-1` slots, so read concurrency is capped at `cores-1` and one core is always left for the interface. Deserialising a score is CPU-bound: moving these waits to an unbounded executor would let as many thaws run as arrive and starve the render thread — not blocking it, but missing frame deadlines, which a user cannot tell apart. Reads are deliberately parallel *and* deliberately bounded, and the bound scales with the machine the deployment chose.

Session restore is the exception: it reopens recorded windows **one at a time**, because the recorded stacking order can only be reproduced by opening back-most first and concurrent opens would race it. Its seriality is about ordering, not thread economy.

## Failures are typed data

Neither direction throws for a failure a user can cause. A read returns `{:open-failure <type>}`, a write's handle delivers `{:save-failure <type>}`, and the frontend discriminates by shape: a map is a failure, anything else is success. Each type maps through `tr` to its own notification, so the user is told *what* went wrong rather than that something did.

[ADR-0051 §7](../ADRs/0051-Filesystem-Operations-Real-and-Virtual.md) holds both taxonomies and is the authority; it is not restated here. Two properties of them are worth knowing while reading this guide:

- **Heap exhaustion is named on both sides.** `:insufficient-memory` on an open, and the same on a save, since freezing a score can exhaust a heap as thawing one can. An `OutOfMemoryError` is an `Error`, so the catches span it deliberately.
- **The catch-alls log the throwable beside the type.** The keyword says only that the operation failed; the throwable is the only thing that says which failure it was, and it reaches the ADR-0017 boundary. That pairing is what makes a catch-all acceptable.

## What follows a completed write

Everything that reports a save happens **when the write lands**, never when it is queued. Nothing may report a save that has not happened.

- **The recorded location** — the path and modification time, so a later plain save needs no picker.
- **`:piece-structure-changed`** — for a full save, whose destination changes the projected filename, so a subscribed window refetches and retitles. A plain save emits nothing: the filename is unchanged.
- **The dirty flag**, conditionally.

**The dirty flag clears only if the piece is unchanged since its snapshot.** Deferring the write opens a window, and a piece can be edited inside it — the user carries on working while a large score is frozen. Clearing regardless would mark as saved a piece whose latest edit exists nowhere but memory, so the user quits without being asked and loses it. The comparison is `=` rather than `identical?`, because the hash-consing daemon rewrites piece values transparently; [ADR-0052 §5](../ADRs/0052-Change-Detection-and-Event-Generation.md) states why, and what it costs on a score that may be an entire opera.

A consequence worth expecting: pressing Save leaves the modified indicator showing for the duration of the write. That is the truth, not a lag.

## Serialisation

**Nippy** provides the serialisation layer: a compact binary format, native handling of Clojure data structures, built-in compression, and streaming for pieces too large to hold twice in memory. See [ADR-0007](../ADRs/0007-Nippy.md).

**Hash-consing survives the round trip.** Ooloi shares identical immutable musical objects — a C4 with a staccato exists once however many times it appears — and serialisation preserves that rather than expanding it. Custom Nippy transforms for `Pitch`, `Rest`, `Chord` and `Articulation` write a **registry** of shared objects once and reference it thereafter, so file size falls with the same ratio memory does. The registry is written first, so reading reconstructs shared identity as it goes. See [ADR-0029](../ADRs/0029-Global-Hash-Consing.md).

**Pure trees are what make the deferred write safe.** A snapshot is an immutable value, so the queue can hold it while the user edits into new versions, and the file that lands is a consistent point-in-time score rather than a smear of two. See [ADR-0010](../ADRs/0010-Pure-Trees.md).

## The I/O backends

A write is parameterised by an output function and a read by an input function, both built by constructors in `ooloi.shared.ops.persistence`. Three pairs exist, and they are not interchangeable:

| Backend | Constructors | Mechanism | Suitable for |
|---|---|---|---|
| **File** | `file-writer`, `file-reader` | `freeze-to-file` / `thaw-from-file` | Everything production does. This is the path `save-piece` and `open-piece` use |
| **Buffer** | `buffer-writer`, `buffer-reader` | `freeze` / `thaw` over a byte array | Tests and benchmarks. **Size-limited by construction**: the whole serialisation is held in memory, so it is unsuitable for a very large score |
| **Socket** | `socket-writer`, `socket-reader` | `freeze-to-out!` / `thaw-from-in!` over a socket stream | Streaming a piece over a network connection. It never materialises the whole serialisation, so unlike the buffer it is not bounded by heap |

The buffer and socket backends have **no production callers** — every occurrence outside tests is a definition. They carry no global state and no lifecycle, so nothing about them is latent; they are the pieces available if a use arrives. Collaboration does not use the socket backend: piece access between a host and a guest goes over gRPC ([ADR-0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)), not raw sockets.

## Integration with the Piece Manager

Opening a piece **registers it** in the piece manager and returns its piece-id; subsequent operations address it by that id. The recorded location travels with the entry as provenance, which is what lets a plain `save-piece` write back without asking where.

A piece's identity is its UUID, and the [persistent catalogue](PIECE_MANAGER_GUIDE.md) maps that UUID to a storage location — which is how `open-by-uuid` reopens a piece that is no longer in memory, and how session restore reopens the windows of a previous run.

## Benchmarking

Serialisation cost is best measured through the **buffer** backend, which removes the disk from the measurement:

```clojure
(let [piece (pm/get-piece piece-id)
      write (persistence/buffer-writer)
      t0    (System/nanoTime)
      bytes (write piece)
      froze (- (System/nanoTime) t0)
      read  (persistence/buffer-reader bytes)
      t1    (System/nanoTime)
      _     (read)
      thawed (- (System/nanoTime) t1)]
  {:freeze-ms (/ froze 1e6)
   :thaw-ms   (/ thawed 1e6)
   :bytes     (count bytes)})
```

This measures the serialiser, not the save path: it neither queues a write nor touches a file. Remember the buffer's constraint — it holds the whole serialisation in memory, so benchmark a representative score rather than the largest one you have.

## Autosave and versioning

Neither exists yet. The deferred write is what makes both possible without interrupting anybody, so what follows is the shape they should take rather than code to copy.

**An autosave submits a save and ignores the handle.** That is the whole of it: the snapshot is taken at the moment autosave fires, the write queues behind anything already writing, and the user notices nothing. Two things it must respect:

- **Do not autosave a piece with a write already in flight.** Two snapshots of one piece resolve in submission order, so nothing corrupts — but the later write is wasted, and an autosave snapshot landing after a user's explicit save would put an older score on disk under a newer timestamp.
- **Never drive it from a sleeping loop.** Periodic work in Ooloi uses a scheduled executor, not `future` plus `Thread/sleep`; the hash-consing daemon is the pattern to follow, including its reason for a fixed *delay* rather than a fixed rate.

**Versioning** — keeping timestamped copies rather than overwriting — is a `clone-piece` to a derived filename, and inherits the whole save-failure taxonomy unchanged. What it needs beyond that is a policy nobody has decided: how many versions, where they live, and when they are pruned.

**Format migration** would rewrite existing files under a new format. It waits on a decision rather than on code: the on-disk format carries no version stamp today, so a file written by a later Ooloi is presently indistinguishable from a damaged one. [ADR-0051 §7](../ADRs/0051-Filesystem-Operations-Real-and-Virtual.md) records that a versioned envelope is a separate future decision, and an `:unsupported-piece-format` type is what it would add.

## Common gotchas

**Forgetting to deref the handle in a test.** A test that saves and then reads the file back must deref, or it races the write:

```clojure
;; ❌ the write may not have happened yet
(SRV/save-piece piece-id dir-token "score.ooloi")
(persistence/file-reader path)                       ; FileNotFoundException

;; ✅ deref, which is what makes the test synchronous again
@(SRV/save-piece piece-id dir-token "score.ooloi")
```

**Assuming the dirty flag clears on every successful save.** It clears only if the piece is unchanged since the snapshot. A test that edits a piece mid-write and then expects it clean is asserting the data-loss behaviour, not the correct one.

**Expecting a nav-token to resolve off the calling thread.** Token resolution reads the client-id from the gRPC context. Anything deferred to another thread must have resolved its destination first.

**Reaching for the buffer backend for a large score.** It holds the entire serialisation in memory. Use the file backend, or the socket backend if it must cross a network.

## Cross-references

- **Piece lifecycle** — [Piece Manager Guide](PIECE_MANAGER_GUIDE.md), for in-memory operations and the persistent catalogue
- **Traversal** — [Timewalking Guide](TIMEWALKING_GUIDE.md)
- **Concurrency** — [Advanced Concurrency Patterns](ADVANCED_CONCURRENCY_PATTERNS.md) for STM, and [Integrant Components](INTEGRANT_COMPONENTS.md) for the save queue's lifecycle
- **Contracts** — [ADR-0051](../ADRs/0051-Filesystem-Operations-Real-and-Virtual.md) for the filesystem operations and both failure taxonomies, [ADR-0012](../ADRs/0012-Persisting-Pieces.md) for the persistence model, [ADR-0007](../ADRs/0007-Nippy.md) for the format, [ADR-0052](../ADRs/0052-Change-Detection-and-Event-Generation.md) §5 for the dirty flag, [ADR-0010](../ADRs/0010-Pure-Trees.md) for why a snapshot can be written while editing continues
