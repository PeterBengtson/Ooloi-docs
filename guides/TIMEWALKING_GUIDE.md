# 🟢 Timewalking Guide: Musical Traversal with Temporal Coordination

*"Let's do the timewalk again" - The Unconventional Conventionalists*

## Table of Contents

- [🟢 Overview](#-overview)
- [🟢 Prerequisites and Intended Audience](#-prerequisites-and-intended-audience)
- [🟢 What Timewalking Actually Does](#-what-timewalking-actually-does)
- [🟢 What Timewalking Returns: The Power of Tuples](#-what-timewalking-returns-the-power-of-tuples)
- [What the Stream Contains](#what-the-stream-contains)
- [🟡 The Fundamental Insight: Structure → Stream](#-the-fundamental-insight-structure--stream)
- [🟢 Starting Simple: Direct Lazy Sequences](#-starting-simple-direct-lazy-sequences)
- [🟡 The Magic of Composition](#-the-magic-of-composition)
- [🟡 Why Not Just Use `->>` for Everything?](#-why-not-just-use---for-everything)
  - [Choosing the Right Consumption Strategy](#choosing-the-right-consumption-strategy)
  - [🔴 Advanced Pattern: Using `reduce` for Stateful Transformations](#-advanced-pattern-using-reduce-for-stateful-transformations)
- [🟠 Understanding Timewalk's Architecture: Push vs Pull](#-understanding-timewalks-architecture-push-vs-pull)
- [🟢 Efficiency Tips](#-efficiency-tips)
  - [Grace End Markers and Grace Positions](#grace-end-markers-and-grace-positions)
- [🔴 Real-World Example: Remembered Alterations](#-real-world-example-remembered-alterations)
- [Cross-References](#cross-references)

## 🟢 Overview

The `timewalk` function provides powerful traversal of musical piece structures with **temporal coordination** across measures. This ensures proper musical time ordering essential for formatting, analysis, and finding musical elements.

**This guide serves two purposes:**

1. **For Ooloi developers**: Learn how to use `timewalk` for musical data processing—finding pitches, transposing melodies, analyzing musical data, and building efficient processing pipelines.

2. **For Clojure learners**: This guide teaches **transducers, lazy sequences, functional composition, and threading macros** through practical musical examples. The musical domain makes these concepts natural and intuitive, demonstrating how functional programming patterns solve real-world problems.

As the timewalker supports both a direct sequence form and a transducer form, you'll see both approaches throughout. The guide progresses from simple operations to advanced transducer patterns, teaching both Ooloi's API and core Clojure concepts simultaneously. The two forms do not cost the same: the direct form walks the whole scope before you see its first result, while the transducer form can stop the walk the moment you have what you need. [Understanding Timewalk's Architecture: Push vs Pull](#-understanding-timewalks-architecture-push-vs-pull) explains why.

## 🟢 Prerequisites and Intended Audience

This guide assumes:
- **Comfortable fluency with basic Clojure syntax and REPL usage**
- **Familiarity with `map`, `filter`, and basic sequence operations**
- **Some exposure to music notation concepts** (measures, voices, pitches)
- **No prior experience with transducers required** - they're explained through musical examples

The guide progresses from basic timewalking to advanced transducer patterns. Each section builds on previous concepts, so working through sequentially is recommended.

## 🟢 What Timewalking Actually Does

Before diving into examples, here's the key insight:

**Timewalking transforms your piece into a stream of musical events in time order.**

Instead of navigating a tree of musicians → instruments → staves → measures → voices, you get a single stream where everything flows in musical time: all of measure 1, then all of measure 2, then all of measure 3.

**What this means for you:**

- **No manual synchronization**: Want all forte passages? Just filter the stream. No tracking which voice you're in.
- **Natural musical thinking**: "Find the first C# after measure 10" is exactly what you write—no tree navigation.
- **Composable operations**: Layout, harmonic analysis, and data processing all use the same stream patterns.

Think of it like this: A score is a spatial thing (ink on paper, nested data structures). Music is a temporal thing (events in time). Timewalking bridges the gap—it transforms the spatial score into the temporal music.

**For Clojure learners**: This structure-to-stream transformation is what **all transducers do**, conceptually. Whether you're processing a vector of numbers, a nested tree of data, or a musical score, transducers transform your data into a stream that can be filtered, mapped, and reduced efficiently. The timewalker demonstrates this fundamental pattern in a domain where the transformation is especially natural and visible—where spatial structure (the score) becomes temporal flow (the music).

The rest of this guide shows you how to work with that stream.

## 🟢 What Timewalking Returns: The Power of Tuples

To demonstrate all this, let's get specific with the timewalker. Every element of the stream is a **three-element tuple** containing everything you need:

```clojure
[item vpd position]
```

- **`item`** - The actual musical object (pitch, chord, rest, measure, etc.)
- **`vpd`** - Vector Path Descriptor - the exact location of this item in the piece
- **`position`** - Rhythmic position within the measure (0, 1/4, 1/2, 3/4, etc.)

One tuple carries no musical object: the grace end marker `[:end :Grace nil]`, which the walker emits after each grace group only when asked to (see [Grace End Markers and Grace Positions](#grace-end-markers-and-grace-positions)). Its "item" is the keyword `:end` and its position is `nil`.

This tuple gives you **complete context** for any musical element:

```clojure
;; Example tuple for a pitch
[#Pitch{:note "C4" :duration 1/4} [:m 0 0 0 2 0 :items 1] 1/4]
;;   ^actual pitch                  ^exact location in piece structure (staff → measure → voice → item)  ^beat position
```

With this information, you can:
- **Access the item**: Get its properties like `(:note pitch)`
- **Know its location**: Navigate to related elements using the VPD
- **Know its timing**: Understand rhythmic placement for analysis or formatting

**For Clojure learners**: This tuple design demonstrates **data-oriented programming**—a core functional programming principle. Instead of hiding context in object state or method calls, everything you need is explicitly present in the data structure. This makes the code **transparent** (you can see what you're working with), **testable** (just compare data), and **composable** (pass tuples through transformation pipelines). The tuple pattern appears throughout functional programming: database query results, parser outputs, and stream processing all benefit from this "carry the context with you" approach.

### Essential Tuple Helpers and Predicates

Since the `timewalk` function returns `[item vpd position]` tuples, Ooloi provides both helper functions for tuple destructuring and specialized `??` predicates for filtering. These `??` predicates are counterparts to the normal `?` predicates, designed specifically for timewalk tuples and require tuples with at least 3 elements:

```clojure
;; Core destructuring helpers (from timewalk.clj)
(item result)      ; Extract the musical object (first element)
(vpd result)       ; Extract the location path (second element)  
(position result)  ; Extract the rhythmic position (third element)

;; Specialized ?? predicates - counterparts to normal ? predicates for timewalk tuples
;; The ?? symbolizes "descending into tuple" to check the item type
;; These REQUIRE tuples with at least 3 elements [item vpd position ...] or will signal an error
(pitch?? result)          ; Counterpart to pitch? - checks if tuple contains a pitch
(chord?? result)          ; Counterpart to chord? - checks if tuple contains a chord  
(measure?? result)        ; Counterpart to measure? - checks if tuple contains a measure
(rhythmic-item?? result)  ; Counterpart to rhythmic-item? - checks behavioral trait
(takes-attachment?? result) ; Counterpart to takes-attachment? - checks behavioral trait
;; All normal ? predicates have ?? counterparts for timewalk contexts
```

**Why ?? predicates exist - timewalking is central to Ooloi:**
```clojure
;; Manual tuple destructuring - repetitive and error-prone
(filter (fn [[item _ _]] (isa? (type item) ::h/Pitch)) timewalk-results)

;; With ?? predicates - clean and focused on musical intent
(filter pitch?? timewalk-results)

;; Clear separation: ? for raw items, ?? for timewalk tuples
(pitch? raw-item)      ; ✓ Normal predicate for raw musical item
(pitch?? tuple-result)  ; ✓ Counterpart predicate for timewalk tuple [item vpd position]

;; Error prevention - ?? predicates validate their input
(pitch?? raw-item)      ; ✗ IllegalArgumentException - expects tuple with at least 3 elements
(pitch?? [item vpd pos]) ; ✓ Correct usage with tuple
```

These helpers transform the timewalker from a low-level traversal tool into a readable musical analysis API.

## What the Stream Contains

A filter only behaves as you expect when you know what arrives in the stream, and the stream holds more than notes. In order (`walk!`, `walk-measure!`, `walk-voice!` and `walk-item!` in `ooloi.shared.ops.timewalk`):

1. **The boundary object comes first**, at position 0: the piece when the options name no `:boundary-vpd`, the instrument for `[:m 0 0]`, and so on. If the boundary is itself a measure, the measure is the first tuple.
2. **Then measure by measure.** Every measure numbered N within the boundary, staff by staff in structural order, comes before any measure numbered N+1. Each measure is emitted at position 0, followed by its voices; each voice is emitted at position 0, followed by its items.
3. **Musicians, instruments and staves below the boundary are not emitted.** The walker descends through them to find the measures and passes nothing on.
4. **Items arrive container first.** A tuplet, tremolando or grace group is emitted before its contents, which follow at their own positions.
5. **Chord pitches and attachments are tuples of their own**, at the position of the item they belong to, with VPDs ending in `:pitches k` or `:attachments k`. `(filter pitch??)` therefore yields the pitches inside chords as well as the single notes; to keep only the single notes, drop the tuples whose VPD contains `:pitches`.
6. **`:start-position` and `:end-position` filter only the items directly in a voice**, by the position at which each starts. The contents of a tuplet that starts inside the window are emitted in full, wherever they end.

```clojure
;; The opening of a walk over a piece with one staff, one measure, one voice
;; holding a C major chord of a half note and a rest:
[piece          []                                0]
[measure        [:m 0 0 0 0]                      0]
[voice          [:m 0 0 0 0 0]                    0]
[chord          [:m 0 0 0 0 0 :items 0]           0]
[pitch-c4       [:m 0 0 0 0 0 :items 0 :pitches 0] 0]
[pitch-e4       [:m 0 0 0 0 0 :items 0 :pitches 1] 0]
[pitch-g4       [:m 0 0 0 0 0 :items 0 :pitches 2] 0]
[rest           [:m 0 0 0 0 0 :items 1]           1/2]
```

## 🟡 The Fundamental Insight: Structure → Stream

*"It's just a jump to the left, and then a step to the right" - Riff Raff*

Before diving into examples, let's understand the most important concept that makes timewalking (and transducers generally) so powerful:

### Transducers Transform Structure Into Stream

Your musical piece is a **nested spatial structure**:

```clojure
{:musicians
  [{:name "Violin I"
    :instruments
      [{:staves
          [{:measures
              [{:voices
                  [{:items [pitch pitch rest chord ...]}]}]}]}]}]}
```

This structure represents music as **space** - musicians contain instruments, instruments contain staves, staves contain measures. It's like ink on paper, organized hierarchically.

But music isn't experienced as nested containers - it's experienced as **time**. Events flowing one after another. Note, note, note, chord, rest...

**Timewalking bridges this gap.** It transforms the spatial structure into a temporal stream:

```clojure
;; Structure (nested, spatial)
piece

;; Stream (linear, temporal)
(timewalk piece {})
;; => [piece-event] [measure-event] [voice-event] [pitch-event1] [pitch-event2] [rest-event1] [chord-event1] ...
```

### This Changes Everything

Once you see your data as a stream, your thinking shifts:

**BEFORE (navigating structure)**:
```clojure
;; "Navigate to musician 0, get instrument 0, find staff 0,
;;  iterate measures, check each voice, filter for pitches"
(for [musician (:musicians piece)
      instrument (:instruments musician)
      staff (:staves instrument)
      measure (:measures staff)
      voice (:voices measure)
      item (:items voice)
      :when (pitch? item)]
  item)
```

**AFTER (filtering stream)**:
```clojure
;; "Filter the stream for pitches"
(->> (timewalk piece {})
     (filter pitch??)
     (map item))
```

**The structure hasn't changed.** Your **relationship** to it has. You're no longer navigating a tree - you're filtering a stream.

### Why This Matters for Musical Thinking

Music naturally flows in time. When you think about a melody, you don't think:

> "The musician at index 0, instrument at index 0, staff at index 0, measure at index 5, voice at index 0, items at indices 0 through 7"

You think:

> "These eight notes, flowing from beat 1 of measure 5"

**Timewalking lets you code the way you think about music:**

```clojure
;; Find the first C#4 in the piece
(->> (timewalk piece {})
     (filter pitch??)
     (filter #(= "C#4" (:note (item %))))
     (take 1)
     (first))
```

This is exactly how you'd describe it in words. No tree navigation. No index juggling. Just filtering the temporal stream.

### This Isn't Just About Timewalking

**All transducers work this way** - they transform structure into stream:

```clojure
;; Vector (structure)
[1 2 3 4 5 6 7 8 9 10]

;; Map transducer (transforms structure → stream → structure)
(->> [1 2 3 4 5 6 7 8 9 10]
     (map inc)
     (filter even?)
     (take 5))
;; => (2 4 6 8 10)

;; Same transformation, reusable recipe:
(def process (comp (map inc) (filter even?) (take 5)))
(sequence process [1 2 3 4 5 6 7 8 9 10])
;; => (2 4 6 8 10)
```

The key insight: **the transformation is separate from the data**. The `process` transformation works on any sequential data source.

### What This Means for Development

When you embrace the stream mindset:

1. **You stop thinking about structure** - The hierarchy becomes invisible. You never write "musician 0, instrument 0, staff 0" again.
2. **Code becomes declarative** - "give me pitches above C5" not "for each musician, for each instrument..."
3. **Operations compose naturally** - filter → map → take is a pipeline, not nested loops
4. **Performance comes with the form** - a transducer pipeline processes one item at a time, with no intermediate collections
5. **Reusability emerges** - same transformation works on different pieces

**Before structure → stream thinking:**
```clojure
(defn find-high-notes [piece]
  (let [results (atom [])]
    (doseq [musician (:musicians piece)
            instrument (:instruments musician)
            staff (:staves instrument)
            measure (:measures staff)
            voice (:voices measure)
            item (:items voice)]
      (when (and (pitch? item) (hz>= item "C5"))
        (swap! results conj item)))
    @results))
```

**After structure → stream thinking:**
```clojure
(defn find-high-notes [piece]
  (->> (timewalk piece {})
       (filter pitch??)
       (filter #(hz>= (item %) "C5"))
       (map item)))
```

The second version is:
- **Shorter** (5 lines vs 12)
- **Clearer** (expresses intent directly)
- **Free of mutable state** (no atom, no accumulation by hand)
- **More composable** (can easily add more filters)

Everything that follows builds on this insight. But it all comes back to this: **structure → stream**. Once you see this, transducers become natural.

### Understanding the `->>` Operator (Thread-Last)

Before diving into examples, you need to understand `->>` (pronounced "thread-last"), which appears in every example:

```clojure
;; Instead of nesting functions like this (hard to read):
(take 10 (map #(:note (item %))
              (filter pitch??
                      (timewalk piece {}))))

;; We use ->> to thread the result through each step:
(->> (timewalk piece {})             ; Start with this
     (filter pitch??)                ; Pass result to filter (note: ??)
     (map #(:note (item %)))         ; Pass result to map
     (take 10))                      ; Pass result to take
```

**How it works:** `->>` takes the result of each line and inserts it as the **last argument** of the next function call.

```clojure
;; These are equivalent:
(->> data
     (filter even?)
     (map inc))

;; Expands to:
(map inc (filter even? data))
```

**Why it's useful:** With `->>` you read top-to-bottom, which matches how you think about data transformations: "start with this data, then filter it, then map it, then take some."

**Important Performance Note:** The steps after the timewalk are lazy — `filter`, `map` and `take` realise only as much as you consume. The timewalk at the head of the thread is not:

```clojure
(->> (timewalk piece {})             ; Walks the whole piece and holds every tuple
     (filter pitch??)                ; Lazy (note: ??)
     (map #(:note (item %)))         ; Lazy
     (take 10))                      ; Lazy: realises 10 results, but the walk is already done
```

The 2-arity `(timewalk piece {})` is `(sequence (timewalk {}) [piece])`. `sequence` with a transducer processes its input one element at a time and buffers whatever each element produces. Here there is only one input element, the piece, so the whole walk runs as soon as the sequence is created and every tuple in scope is held in the buffer. `take 10` limits what you receive; it does not shorten the walk.

**Connection to transducers:** The `->>` pattern shows you're doing step-by-step data transformation. Transducers take the same concept and fuse the steps into one. Placing the timewalk itself inside the composition is what lets an early `take` stop the walk, as [Even Better: The Transducer Form](#even-better-the-transducer-form) shows.

## 🟢 Starting Simple: Direct Lazy Sequences

The easiest way to use the `timewalk` function is the **2-arity form** that returns a sequence directly:

```clojure
(timewalk piece {})  ; Empty map means "traverse the entire piece"
```

**For Clojure learners**: A **lazy sequence** is one of Clojure's most powerful features. Instead of computing all results upfront and storing them in memory, a lazy sequence generates values *on demand* as you consume them, which is what lets `filter`, `map` and `take` in the examples below do only the work their results require. The 2-arity timewalk is a convenience, not a lazy producer: it is `(sequence (timewalk options) [piece])`, a sequence over a single input element, the piece. Creating it runs the entire scoped walk and holds all of its tuples, however few you then read. That is entirely acceptable for exploration at the REPL and for small pieces. For large scores, or whenever you want to stop at the first match, use the transducer form introduced in [Even Better: The Transducer Form](#even-better-the-transducer-form).

Let's start with practical examples that show the power of this approach.

### Example 1: Find All Pitches

```clojure
;; Get all pitches in the entire piece (returns lazy sequence)
(->> (timewalk piece {})
     (filter pitch??)
     (map item))  ; Extract just the pitch objects
;; => (#Pitch{:note "C4" :duration 1/4} #Pitch{:note "E4" :duration 1/4} #Pitch{:note "G4" :duration 1/4} ...)  ; Lazy sequence

;; To realize into a vector:
(into []
      (comp (filter pitch??)
            (map item))
      (timewalk piece {}))
;; => [#Pitch{:note "C4" :duration 1/4} #Pitch{:note "E4" :duration 1/4} #Pitch{:note "G4" :duration 1/4} ...]  ; Realized vector
```

### Example 2: Filter by Pitch Range

```clojure
;; Find pitches above middle C (returns lazy sequence)
;; Note: hz>= compares frequencies, works with Pitch objects and pitch strings
(->> (timewalk piece {})
     (filter pitch??)
     (filter #(hz>= (item %) "C4"))  ; Compare by frequency
     (map item))
;; => (#Pitch{:note "C4" :duration 1/4} #Pitch{:note "E4" :duration 1/4} #Pitch{:note "G4" :duration 1/2} ...)  ; Lazy sequence
```

### Example 3: Get Pitch Positions

```clojure
;; Find pitches with their timing (returns lazy sequence)
(->> (timewalk piece {})
     (filter pitch??)
     (map #(hash-map :note (:note (item %))
                     :measure (vpd/measure (vpd %))
                     :position (position %))))
;; => ({:note "C4" :measure 0 :position 0}
;;     {:note "E4" :measure 0 :position 1/4}
;;     {:note "G4" :measure 0 :position 1/2} ...)  ; Lazy sequence
```

### Example 4: Simple Processing Pipeline

```clojure
;; Find pitches, filter range, extract notes (returns lazy sequence)
(->> (timewalk piece {})
     (filter pitch??)  ; Only pitches
     (filter #(hz>= (item %) "C4"))  ; Above middle C
     (map #(:note (item %)))  ; Extract note names
     (take 10))  ; First 10 results - still lazy until consumed
;; => ("C4" "E4" "G4" "C5" "E5" ...)  ; Lazy sequence
```

Notice how each step builds on the previous one, and we always have access to the full tuple `[item vpd position]` for complete context. The result remains lazy until you consume it (e.g., with `doall`, `into []`, or by iterating over it).

## 🟡 The Magic of Composition

*"The transducer will seduce ya" - Frank N. Furter*

Here the power of composition becomes evident. Those steps above can be **composed** using transducers for elegance and efficiency.

**For Clojure learners**: Composing transformations that work independently of their data source is one of the most elegant and reusable patterns in functional programming. Once you understand this pattern through musical examples, you'll recognize it everywhere in idiomatic Clojure code.

### Understanding `comp` - Function Composition

`comp` creates a new function by combining multiple functions together:

```clojure
;; Instead of nesting functions like this:
(take 10 (map #(:note (item %))
              (filter #(hz>= (item %) "C4")
                      (filter pitch?? results))))

;; We can compose them into a single transformation:
(comp (filter pitch??)
      (filter #(hz>= (item %) "C4"))
      (map #(:note (item %)))
      (take 10))
;; This creates one function that does all four steps
```

### Understanding `sequence` - Applying Transformations

`sequence` applies a transformation (transducer) to a collection:

```clojure
;; sequence takes a transformation and a collection
(sequence transformation collection)

;; It's like ->> but more efficient internally - composed transducer, lazy result
(->> collection step1 step2 step3)           ; Creates lazy sequence wrappers between steps
(sequence (comp step1 step2 step3) collection)  ; Single composed transformation, lazy result
```

**For Clojure learners**: The `sequence` function is how you apply a transducer to actual data while getting a lazy sequence back. Think of it this way: `comp` creates a *transformation recipe*, and `sequence` *applies that recipe* to your data, returning a lazy sequence. This separation between defining transformations and applying them is fundamental to functional programming—it lets you build reusable transformation logic that works with any compatible data source. The key insight: your transformation doesn't know or care whether it's processing musical notes, database records, or lines from a file.

**Important**: `sequence` returns a **lazy sequence**, not a fully realized collection. For truly eager, push-based processing with no lazy wrappers at all, use `into` or `transduce`.

### Side-by-Side Comparison

```clojure
;; Traditional approach with ->>
(->> (timewalk piece {})                             ; Returns lazy sequence
     (filter pitch??)                                ; Creates new lazy sequence wrapper
     (filter #(hz>= (item %) "C4"))                  ; Creates another lazy wrapper
     (map #(:note (item %)))                         ; Creates another lazy wrapper
     (take 10))                                      ; Creates final lazy wrapper
;; Total: 4 intermediate lazy sequence wrappers (lightweight, but still overhead)
;; Result: Lazy sequence (pull-based)

;; Transducer approach with sequence and comp
(sequence (comp (filter pitch??)                     ; \
                (filter #(hz>= (item %) "C4"))       ; > Combined into
                (map #(:note (item %)))              ; /  one transformation
                (take 10))                           ; /
          (timewalk piece {}))
;; Total: Composed transformation internally, no intermediate wrappers
;; Result: Lazy sequence (pull-based, but more efficient internally)

;; Truly eager push-based approach with into
(into []
      (comp (filter pitch??)
            (filter #(hz>= (item %) "C4"))
            (map #(:note (item %)))
            (take 10))
      (timewalk piece {}))
;; Total: Zero lazy wrappers between the steps; the input is the 2-arity sequence
;; Result: Realized vector (push-based, immediate evaluation)
```

**Key differences:**
- **Threading `->>`**: Creates lazy sequence wrapper for each step (pull-based)
- **`sequence` with transducers**: Composed transformation, returns lazy sequence (still pull-based, but more efficient)
- **`into` with transducers**: True push-based processing, no lazy wrappers, immediate realization

The differences lie entirely downstream of the walk. In all three, the 2-arity `(timewalk piece {})` has walked the whole piece and buffered every tuple before the first `filter` runs, so `take 10` saves processing in the later steps and none in the traversal. To have `take 10` stop the walk itself, the timewalk goes inside the composition; see [Even Better: The Transducer Form](#even-better-the-transducer-form).

### Performance Comparison: Threading vs Transducers

Here's why transducers tend to be more efficient for complex processing chains:

```clojure
;; Threading approach - creates multiple lazy sequence wrappers
(->> (timewalk large-symphony {})
     (filter pitch??)
     (map #(:note (item %)))
     (take 1000))
;; Memory: Every tuple of the symphony, buffered by the 2-arity walk,
;;         plus 3 intermediate lazy sequence objects
;; Processing: The whole symphony is walked; each later step wraps the previous one

;; Transducer approach - single processing chain, timewalk included
(into []
      (comp (timewalk {})
            (filter pitch??)
            (map #(:note (item %)))
            (take 1000))
      [large-symphony])
;; Memory: The walk's index of the symphony's measures and the 1000 results;
;;         no tuple sequence, no intermediate objects
;; Processing: All steps fused; the walk stops after the 1000th pitch
```

**Why transducers are typically faster:**
- **Threading (`->>`)**: Each step creates a new lazy sequence wrapper
- **Transducers**: All steps combined into one efficient processing function
- **Memory**: Fewer allocations mean less garbage collection pressure
- **CPU**: Direct function composition eliminates wrapper overhead

**Performance gain varies** depending on:
- **Chain length**: More steps = bigger transducer advantage
- **Data size**: Larger sequences benefit more from reduced allocation
- **GC pressure**: Complex applications see bigger gains from reduced intermediate objects

**Why transducers win on the JVM:**
- **No intermediate allocations**: Direct function composition vs lazy sequence wrappers
- **JIT-friendly**: More predictable execution patterns for JVM optimization
- **Cache efficiency**: Single-pass processing vs multiple sequence traversals

### What Are Transducers? A Brief Foundation

Before diving deeper, let's establish what transducers actually are:

**Transducers are transformation functions that are independent of the context in which they're used.**

**For Clojure learners**: Transducers break the coupling between transformation logic and data structures. You write the transformation logic once (filter for pitches, map to frequencies, take 10 results), and that *same code* works with vectors, lazy sequences, channels, streams, or musical scores. This is functional programming at its most powerful—separating *what* you want to do from *where* you're doing it.

**Push vs Pull — A Critical Distinction:**

Timewalk demonstrates a true push-based transducer: it *drives* the reducing function by calling it directly as items are discovered, rather than producing a lazy sequence that gets *pulled* when consumed. This distinction matters:

- **Pull (lazy)**: "I'll generate the next value when you ask for it"
- **Push (transducer)**: "Here's the next value right now — process it or stop"

The push model enables **early termination to stop computation itself**, not just sequence consumption. When timewalk is part of the composition and `take 10` signals termination via `reduced`, timewalk immediately stops traversing the musical hierarchy — no wasted work generating values that will never be used. The 2-arity form does not get this benefit: its walk has finished before any downstream step can return `reduced`.

Think of them as "transformation recipes" that can be applied to any data source:

```clojure
;; This is a transducer - a transformation recipe
(def process-melody
  (comp (filter pitch??)
        (map #(:note (item %)))
        (take 10)))

;; The same recipe works with different data sources:
(sequence (comp (timewalk opts) process-melody) [piece1])  ; Lazy result from piece1
(into [] (comp (timewalk opts) process-melody) [piece2])   ; Vector from piece2
(into [] process-melody some-tuples)                        ; Any collection of tuples
```

**Key insight**: The transformation logic is **separate** from:
- **Input source** (timewalk, vector, list, etc.)
- **Output destination** (lazy sequence, vector, set, etc.)
- **Processing context** (sequential, parallel, streaming, etc.)

This separation enables **maximum reusability and efficiency**.

### Understanding Transducers Through Musical Processing

The examples above demonstrate transducers in action. Here's what's happening:

1. **Familiar operations** - filtering and mapping musical data
2. **Composition with `comp`** - combining operations like chaining effects
3. **Application with `sequence`** - running the combined transformation
4. **Efficiency gain** - no intermediate collections between steps

Transducers are efficient, composable data transformations. They process sequences without creating unnecessary intermediate collections - particularly useful for large musical scores.

The musical context makes it natural because you're always doing multi-step processing: find the pitches, filter by range, apply transposers, convert to MIDI. Transducers just make this efficient and elegant.

```clojure
;; Same thing, but composed into a single transformation
(->> (timewalk piece {})
     (sequence (comp (filter pitch??)
                     (filter #(hz>= (item %) "C4"))
                     (map #(:note (item %)))
                     (take 10))))
```

**Key insight:** This approach combines the best of both worlds:
- **Input**: Still uses the familiar `->>` threading pattern
- **Processing**: `sequence` with `comp` creates a single efficient transformation
- **No intermediate collections downstream**: The composed transducer processes the tuples one at a time
- **Result**: Lazy sequence that can be consumed incrementally

The input, though, is the 2-arity walk's buffer of every tuple in the piece, complete before the composed transformation sees the first.

**Important distinction**: The `->>` here is just for readability - it's equivalent to:
```clojure
(sequence (comp (filter pitch??) ...) (timewalk piece {}))
```

### Even Better: The Transducer Form

**Following Established Clojure Patterns**

The timewalker's dual-arity API follows the exact same pattern as Clojure's core sequence functions:

```clojure
;; Standard Clojure pattern - you already know this
(map inc)           ; Returns transducer
(map inc [1 2 3])   ; Returns lazy sequence

(filter even?)      ; Returns transducer  
(filter even? data) ; Returns lazy sequence

(take 5)           ; Returns transducer
(take 5 data)      ; Returns lazy sequence

;; The timewalker follows the same pattern
(timewalk {})        ; Returns transducer
(timewalk piece {})  ; Returns lazy sequence
```

**Why This Pattern Works**

This isn't a novel design - it's how functional composition works in Clojure. The `timewalk` function has the same two shapes as `map`, `filter`, and `take`, just with musical semantics instead of generic collection operations.

One difference matters. `(map inc coll)` is lazy element by element, because `coll` has many elements. The 2-arity `(timewalk piece {})` is `(sequence (timewalk {}) [piece])`, whose input has one element, the piece; the transducer turns that one element into every tuple in scope, and `sequence` buffers them all at once. The 2-arity form is therefore a convenience for exploration and small pieces, which runs the whole scoped walk when it is created. The transducer form is the one that streams.

**Composition in Action**

The `timewalk` function itself can be part of the composition:

```clojure
;; One-arity form returns a transducer - timewalk becomes part of the composition
(sequence (comp (timewalk {})  ; Transducer form
                (filter pitch??)
                (filter #(hz>= (item %) "C4"))
                (map #(:note (item %)))
                (take 10))
          [piece])
;; => ("C4" "E4" "G4" "C5" "E5" ...)
```

Here `take 10` is inside the same composition as the walk, so after the tenth result it returns `reduced`, the walker sees it and stops traversing. The same holds for `into` and `transduce` over the composition, and for `reduce` over an `eduction` of it: put the timewalk inside, apply the whole to `[piece]`.

This is **composable**, **reusable**, and **efficient** - the hallmarks of good functional design. More importantly, it integrates naturally with the Clojure ecosystem because it follows established conventions.

### Transducer Usage Patterns

The examples above demonstrate several key patterns:

- **Basic operations** - `filter` and `map` work the same way
- **Composition** - `comp` combines operations efficiently
- **Application** - `sequence` runs the transformation
- **Performance** - no intermediate collections
- **Reusability** - same transformation works on multiple pieces

Musical processing naturally involves multi-step operations: find → filter → transform → output. Transducers make this efficient for large scores and reusable across different pieces.

## 🟡 Why Not Just Use `->>` for Everything?

**For Clojure learners**: This is the pragmatic question every functional programmer faces—when does elegance matter enough to justify learning a new abstraction? The answer reveals an important principle: **choose the right tool for the job**. Threading with `->>` is simpler and perfectly adequate for small data and exploratory work. Transducers add complexity but deliver substantial benefits at scale. Learning to recognize this tradeoff—readability vs. performance, simplicity vs. reusability—is part of becoming a mature functional programmer. The musical domain makes this tradeoff especially visible: a string quartet has different needs than a Mahler symphony.

Threading with `->>` is perfectly adequate for:
- **Small pieces** (under 1000 total elements)
- **One-off analysis tasks**
- **Simple transformations**
- **Prototyping and experimentation**

Transducers become essential for:
- **Large orchestral scores** (10,000+ total elements) - memory efficiency matters
- **Real-time processing** - minimal allocation overhead
- **Reusable transformation libraries** - compose once, apply anywhere
- **Memory-constrained environments** - no intermediate collections
- **Performance-critical paths** - MIDI generation, real-time analysis

**Example: When size matters**
```clojure
;; make-transposer (ooloi.shared.ops.pitches) returns a function from
;; pitch string to pitch string, so it is applied to the :note of each pitch
(def up-a-tone (make-transposer :up :major :second))

;; Small chamber piece (1000 total elements) - either approach works fine
(->> (timewalk string-quartet {})
     (filter pitch??)
     (map (comp up-a-tone :note item)))

;; Large symphony (50,000+ total elements) - the transducer form never holds
;; the symphony's tuples, only the transposed note strings it collects
(into []
      (comp (timewalk {})
            (filter pitch??)
            (map (comp up-a-tone :note item)))
      [symphony])
```

### Choosing the Right Consumption Strategy

Once you've filtered and transformed your timewalking stream, you need to consume it. Choose based on your needs:

**When you need all results in a collection:**
- Use `into` for vectors, sets, or maps
- Best for: Building collections, random access, multiple passes

**When you need to aggregate to a single value:**
- Use `transduce` to reduce without building intermediate collections
- Best for: Counting, summing, averaging, finding min/max

**When you need side effects only:**
- Use `run!` to execute a function for effects, returns `nil`
- Best for: MIDI scheduling, logging, database writes

**When you want lazy, on-demand processing:**
- Use `->>` threading or `sequence` for lazy sequences
- Best for: REPL exploration, small pieces, simple transformations
- Over the 2-arity form, the downstream steps are lazy but the walk is not

**When you need reusable transformations:**
- Use 1-arity transducer form to compose once, apply many times
- Best for: Library code, applying same transformation to multiple pieces

**When you need early termination:**
- Put `take` (or `take-while`, `halt-when`) in the same composition as the 1-arity `(timewalk opts)`, applied to `[piece]`
- Critical for performance: this stops the walk itself; a `take` after the 2-arity form only limits the result

## 🔴 Advanced Pattern: Using `reduce` for Stateful Transformations

**For Clojure learners**: This advanced pattern demonstrates a subtle but critical distinction in functional programming - when side effects interact with immutable data structures. The problem reveals why `reduce` exists as a fundamental operation: not just for aggregation, but for threading state through a series of transformations. This pattern appears throughout functional programming whenever you need to transform a data structure based on its own contents - parsing, compilation, state machines, and musical transformations all share this challenge.

When modifying a piece during traversal, each modification creates a new piece object. This makes `run!` unsuitable because subsequent operations reference  the original piece structure, not the updated one.

### The Problem: Side Effects with Stale State

```clojure
;; This DOESN'T WORK - every modification is applied to the original piece
(let [transposer (make-transposer :up :major :third)]
  (run! #(vpd/mutate (conj (vpd %) :note) piece transposer)
        (->> (timewalk piece {})
             (filter pitch??))))
;; Problem: Each modification returns a new piece, which is immediately discarded
;; Every later modification starts again from the original piece
```

### The Solution: Threading State with `reduce`

```clojure
;; This WORKS - piece state threads through each transformation
(let [transposer (make-transposer :up :major :third)
      transpose-pitch (fn [piece result]
                        (vpd/mutate (conj (vpd result) :note) piece transposer))]
  (reduce transpose-pitch
          piece
          (->> (timewalk piece {})
               (filter pitch??))))
;; Success: Each modification receives current piece, returns updated piece
;; Final result is fully transposed piece with all modifications applied
```

`make-transposer` returns a function from pitch string to pitch string, so the VPD is extended with `:note` to address the pitch's note string rather than the Pitch record. `vpd/mutate` takes the path, the piece and the transform, applies the transform to the value at that path and returns the new piece. The VPDs collected from the original piece stay valid throughout, because transposing a note changes no item's place in the structure. Pitches inside chords are included, since `(filter pitch??)` yields them as tuples of their own.

**Key insight**: `reduce` threads the accumulator (the piece) through each step, ensuring every transformation operates on the current state. This pattern is essential whenever modifications depend on previous modifications.

**When to use this pattern:**
- Modifying a piece during traversal (transposition, duration changes, etc.)
- Any transformation where each step depends on the result of previous steps
- Building up complex state through sequential operations

**Performance consideration**: This pattern creates a new piece object for each modification. For large-scale transformations, consider batch operations or alternative strategies.

### Common Patterns Reference

#### Collect into collection (`into`)
```clojure
;; Vector for sequential access
(into [] (comp (timewalk {}) (filter pitch??) (map #(:note (item %)))) [piece])

;; Set for unique values
(into #{} (comp (timewalk {}) (filter pitch??) (map #(:note (item %)))) [piece])
```

#### Reduce to single value (`transduce`)
```clojure
;; Count pitches above middle C
(transduce (comp (timewalk {})
                 (filter pitch??)
                 (filter #(hz>= (item %) "C4"))
                 (map (constantly 1)))
           +
           0
           [piece])
```

#### Side effects only (`run!`)
```clojure
;; Schedule for MIDI playback, no return value needed.
;; run! reduces over the eduction, so each tuple is pushed to the
;; function as the walker finds it and no tuple sequence is built.
(run! schedule-for-midi
      (eduction (comp (timewalk {}) (filter pitch??)) [piece]))
```

#### Lazy processing (`sequence` or `->>`)
```clojure
;; Simple threading for exploration (the 2-arity walk runs in full)
(->> (timewalk piece {}) (filter pitch??) (map #(:note (item %))) (take 10))

;; With the walk inside the composition, take 10 stops it
(sequence (comp (timewalk {}) (filter pitch??) (map #(:note (item %))) (take 10))
          [piece])
```

#### Reusable transformations (1-arity form)
```clojure
;; Define once, apply many times
(def find-high-notes
  (comp (timewalk {}) (filter pitch??) (filter #(hz>= (item %) "C5")) (map item)))

(into [] find-high-notes [piece1])
(into [] find-high-notes [piece2])
```

#### Early termination (`take`)
```clojure
;; The walk stops at the first pitch
(first (into [] (comp (timewalk {}) (filter pitch??) (take 1)) [piece]))
```

### Critical Performance Rule: Stop the Walk, Not Just the Result

**This does not stop the walk:**
```clojure
(->> (timewalk piece {})
     (filter pitch??)
     (take 1)   ; Limits the result to one tuple...
     (first))   ; ...but the 2-arity walk has already covered the whole piece
```

**This does:**
```clojure
(first
  (into []
        (comp (timewalk {})
              (filter pitch??)
              (take 1))   ; Returns reduced after the first match; the walker stops
        [piece]))
```

**Why:** the 2-arity form has walked its whole scope and buffered every tuple before `filter` or `take` see anything, so no downstream step can shorten the traversal. With the 1-arity `(timewalk opts)` in the same composition, `take 1` returns `reduced` after the first match and the walker checks for it after every tuple it emits. For large scores, this is the difference between walking the items up to the first match and walking every item in scope. `sequence` over the same composition stops as well, because `take` returns `reduced` inside the walk; `first` over `(timewalk piece {})`, with or without a lazy `take`, never does.

## 🟠 Understanding Timewalk's Architecture: Push vs Pull

**For Clojure learners**: This architectural distinction affects performance in real-world applications. Understanding push vs. pull models is the difference between processing 100,000 items to find one result versus stopping immediately when found. This choice ripples through your entire application design, affecting memory usage, latency, and scalability.

You've been using two approaches to consume timewalking results:

```clojure
;; Approach 1: The 2-arity form, consumed as a sequence
(->> (timewalk piece {})
     (filter pitch??)
     (take 10))

;; Approach 2: The 1-arity form, part of the composition
(sequence (comp (timewalk {})
                (filter pitch??)
                (take 10))
          [piece])
```

Both return the same ten tuples, but there's a fundamental architectural difference that affects performance and behaviour.

### The Two Paradigms

The walker itself is always push-based: it calls a reducing function with each tuple as it finds it. What differs is who receives the pushes.

**The 2-arity form (a buffer in front of you)**:
```
(timewalk piece {})
  Timewalk ----every tuple in scope----> sequence's buffer      (all at once, when created)
You ----request----> Lazy Seq ----request----> buffer
You <----value------ Lazy Seq <----value------ buffer
```
`(timewalk piece {})` is `(sequence (timewalk {}) [piece])`. `sequence` feeds its single input element, the piece, through the walk, and the walk runs to the end of its scope while every tuple is added to the buffer. You then pull values one at a time, but only from the buffer; the walk is already over.

**The 1-arity form (true transducers)**:
```
Timewalk ----value----> Your Function ----result----> Collection
Timewalk ----value----> Your Function ----STOP!
```
Timewalk pushes values into your composition as it discovers them. You signal when to stop.

### Why This Matters: Early Termination

The distinction becomes significant when finding specific elements:

```clojure
;; Find the first forte passage in a symphony
```

**With the 2-arity form**:
```
╔════════════════════════════════════════════════════════╗
║  Symphony (100,000 elements)                           ║
║  ┌──────────────────────────────────────────────────┐ ║
║  │ Timewalk walks all 100,000 into the buffer...    │ ║
║  │ Element 1 → check → no                           │ ║
║  │ Element 2 → check → no                           │ ║
║  │ Element 3 → check → YES! (take 1)                │ ║
║  │ ✓ Stop consuming... the walk finished long ago   │ ║
║  └──────────────────────────────────────────────────┘ ║
╚════════════════════════════════════════════════════════╝
```

**With the 1-arity form in the composition**:
```
╔════════════════════════════════════════════════════════╗
║  Symphony (100,000 elements)                           ║
║  ┌──────────────────────────────────────────────────┐ ║
║  │ Timewalk pushes items as discovered...           │ ║
║  │ Element 1 → your function → continue             │ ║
║  │ Element 2 → your function → continue             │ ║
║  │ Element 3 → your function → REDUCED!             │ ║
║  │ ✓ Timewalk stops traversing immediately          │ ║
║  └──────────────────────────────────────────────────┘ ║
╚════════════════════════════════════════════════════════╝
```

**The key insight**: With timewalk inside the composition, `take 1` stops *computation itself*. The timewalk stops discovering and processing items the moment it receives a `reduced` signal.

### Timewalk's True Transducer Implementation

Timewalk implements a **push-based producer**. As it discovers musical items during traversal, it immediately calls the reducing function with each `[item vpd position]` tuple. When the rest of your pipeline is that reducing function, there's no intermediate sequence being created and then consumed. Both arities, as defined in `ooloi.shared.ops.timewalk`:

```clojure
(defn timewalk
  ([options]
   (fn [rf]                       ; Returns a transducer
     (fn
       ([] (rf))
       ([acc] (rf acc))
       ([acc piece]               ; Each input element is a piece
        ;; walk! calls (rf acc [item vpd position]) for every item it finds,
        ;; and stops as soon as rf returns (reduced acc)
        (walk! rf acc piece options)))))

  ([piece options]
   (sequence (timewalk options) [piece])))
```

This is the distinction between:
- **Creating then consuming** (the 2-arity form - the walk fills a buffer, you read it)
- **Discovering and processing** (the 1-arity form in a composition - push)

### Three Ways to Consume Timewalking Results

Now you can make informed choices about how to consume timewalking results:

#### 1. Streaming (Never Materialise) - Push-Based

Use when you process each item once and discard it:

```clojure
;; MIDI export - process each note once without storing
(run! #(schedule-for-midi (item %))
      (eduction (comp (timewalk {})
                      (filter pitch??))
                [piece]))
;; Memory: No tuple sequence; each tuple is handed to schedule-for-midi and dropped
;; Returns: nil (pure side effects, no collection built)

;; Count without materializing - aggregate to single value
(transduce (comp (timewalk {})
                 (filter pitch??)
                 (map (constantly 1)))
           +
           0
           [piece])
;; Returns: 42 (count of pitches)
;; Memory: Never builds a collection of tuples

;; Early termination - stops computation immediately
(first
  (into []
        (comp (timewalk {})
              (filter forte-passage??)
              (take 1))
        [piece]))
;; Stops traversing the moment first forte found
```

#### 2. Materialisation (Collect into Vector) - For Random Access

Use when you need to access results multiple times or in random order:

```clojure
;; Collect all pitches for multiple analyses
(def all-pitches
  (into []
        (comp (timewalk {})
              (filter pitch??))
        [piece]))

;; Now you can access randomly
(nth all-pitches 42)
(count all-pitches)
(map analyze all-pitches)  ; Second pass

;; Cost: ~437 bytes per [item vpd position] tuple
;; For 520,000 items: ~244 MB
```

**When to materialise**:
- Need random access to results
- Multiple passes over same data
- Need to count results before processing
- Building lookup tables or indices

#### 3. The 2-arity Form - Simple but Not Streaming

Use when simplicity matters more than optimal performance:

```clojure
;; Simple threading over the 2-arity form
(->> (timewalk piece {})
     (filter pitch??)
     (map #(:note (item %)))
     (take 10))

;; Cost: The whole scoped walk runs, and every tuple is held, before filter
;;       sees the first; take 10 limits the result, not the walk
;; Benefit: Familiar, easy to read, good enough for small pieces
```

**When the 2-arity form is fine**:
- Small pieces (< 1,000 elements)
- Prototyping and experimentation
- REPL exploration
- Simplicity is the priority

### Decision Guide: Which Approach to Use

```clojure
;; Side effects only → True streaming with run! over an eduction
(run! #(schedule-for-midi (item %))
      (eduction (comp (timewalk {}) (filter pitch??)) [piece]))

;; Aggregate to single value → True streaming with transduce
(transduce (comp (timewalk {}) (filter pitch??) (map (constantly 1)))
           +
           0
           [piece])

;; Need random access → Materialise with into
(into [] (comp (timewalk {}) (filter pitch??)) [piece])

;; Finding first match → Early termination
(first (into [] (comp (timewalk {}) (filter target??) (take 1)) [piece]))

;; Simple exploration → The 2-arity form (threading)
(->> (timewalk piece {}) (filter pitch??) (take 5))
```

### Why Timewalk Uses Push Architecture

This architectural choice enables, in the transducer form:

1. **True early termination** - Stop computation itself, not just consumption
2. **Zero intermediate allocation** - No lazy sequence wrappers between steps
3. **Streaming without holding tuples** - The walk keeps an index of the measures in scope, one entry per measure of each staff, built before the first tuple is pushed (`collect-measures` in `ooloi.shared.ops.timewalk`). The tuples themselves are kept only if the consumer keeps them
4. **Composability** - Combine with all Clojure transducer operations

The push-based architecture is what makes timewalk efficient enough for professional orchestral scores while maintaining the elegant compositional properties of functional programming.

### The Practical Takeaway

**For most work**: Put the 1-arity `(timewalk opts)` at the head of a composition and apply it to `[piece]` with `sequence`, `into`, `transduce` or an `eduction`. The performance benefits are substantial and the patterns become natural quickly.

**For exploration**: The 2-arity form with `->>` threading is perfectly fine and more readable for learning. It runs the whole scoped walk when it is created.

**For random access**: Materialise into a vector explicitly when you need it.

The rule is short: when the scope is large or you mean to stop early, the timewalk goes inside the composition.

## 🟢 Efficiency Tips

**For Clojure learners**: These patterns demonstrate doing less work as the ultimate optimisation through early termination and precise scope control.

### The Options Map: Controlling Traversal Scope

Up until now, we've been passing an empty map `{}` as the second parameter to `timewalk`. This tells the timewalker to traverse the entire piece structure. But `timewalk` provides powerful scope control through this options map—you can limit traversal to specific musicians, instruments, staves, measure ranges, or even beat positions within measures.

**Why this matters**: Processing only what you need is the most effective optimization. Rather than filtering results after traversal, you can prevent unnecessary traversal entirely. For large orchestral scores, this can mean the difference between processing 50,000 items and processing 100.

The options, with their defaults (`timewalk` in `ooloi.shared.ops.timewalk`):

| Key | Default | Meaning |
|-----|---------|---------|
| `:boundary-vpd` | `[]` (the piece) | The subtree to walk: a musician, instrument, staff or measure |
| `:start-measure` | `0` | First measure index walked |
| `:start-position` | `0` | First position walked within the start measure |
| `:end-measure` | `nil` (the last measure) | Last measure index walked, inclusive |
| `:end-position` | `nil` (end of measure) | Last position walked within the end measure, inclusive |
| `:grace-end-markers?` | `false` | Emit `[:end :Grace nil]` after each grace group |

Let's explore the scope control options available.

### Limit Scope, Not Just Results

Don't process the entire symphony when you only need one part:

```clojure
;; ❌ INEFFICIENT - Traverses all 29 instruments, then filters
(->> (timewalk large-symphony {})
     (filter #(and (= 0 (vpd/musician (vpd %)))      ; Filter after processing
                   (= 0 (vpd/instrument (vpd %)))))
     (filter pitch??))

;; ✅ EFFICIENT - Only traverses the one instrument you need
(->> (timewalk large-symphony {:boundary-vpd [:m 0 0]})
     (filter pitch??))
```

Scope limiting happens *before* traversal. The timewalker never even looks at the other 28 instruments.

### Combine Scope with Measure Ranges

Working on measures 10-15 of the flute part? Tell the timewalker exactly what you need:

```clojure
;; Just measures 10-15 of the flute (musician 2, instrument 0)
;; Note: end-measure is INCLUSIVE - this processes measures 10, 11, 12, 13, 14, AND 15
(->> (timewalk piece {:boundary-vpd [:m 2 0]
                      :start-measure 10
                      :end-measure 15})
     (filter pitch??))
```

**Performance impact**: This processes ~100 items instead of ~50,000. That's a 500× reduction in work.

**Important**: The `:end-measure` parameter is **inclusive**—it includes the specified measure in the results. Same for `:end-position` below.

### Fine-Grained Position Control

Need items starting from beat 2 of measure 5 through beat 3 of measure 7?

```clojure
;; Process from measure 5, beat 2 through measure 7, beat 3 (INCLUSIVE)
;; In 4/4 time: beat 1=0, beat 2=1/4, beat 3=1/2, beat 4=3/4, end of measure=1
;; In 3/4 time: beat 1=0, beat 2=1/4, beat 3=1/2, end of measure=3/4
;; In 5/8 time: positions would be 0, 1/8, 1/4, 3/8, 1/2, end of measure=5/8
(->> (timewalk piece {:start-measure 5
                      :start-position 1/4  ; Beat 2 in 4/4
                      :end-measure 7
                      :end-position 1/2})  ; Beat 3 in 4/4 (inclusive)
     (filter pitch??))
```

The `:start-position` and `:end-position` parameters use **rationals representing position within the measure**. Position 0 is the downbeat. The measure length equals the time signature duration (see [ADR-0033: Time Signature Architecture](../ADRs/0033-Time-Signature-Architecture.md)): 4/4 = 1 (four quarters), 3/4 = 3/4 (three quarters), 5/8 = 5/8 (five eighths), 6/8 = 6/8 = 3/4 (six eighths), etc. Remember: `:end-position` is **inclusive**, so items at exactly that position are included in the results.

The position window applies to the items directly in each voice, compared by the position at which each starts; `:start-position` applies only in the start measure and `:end-position` only in the end measure. Measures, voices and the boundary object are emitted regardless, and so is everything inside an item that passes, such as the remaining notes of a tuplet that starts at the window's last position.

### Grace End Markers and Grace Positions

A grace group (a `Grace` container) is emitted at its **target position**, the position in the voice to which it is attached, and its notes follow at positions **relative to the group**: the first at 0, each later one offset by the notated durations before it. Inside tuplets, those offsets are scaled by the product of the ratios of the tuplets that contain the group, as every position inside a tuplet is ([ADR-0014: Timewalk §5, Grace Note Position Semantics](../ADRs/0014-Timewalk.md#5-grace-note-position-semantics)):

```clojure
;; A grace group at 1/4 holding D5, E5 and F5, each a thirty-second
[grace  [:m 0 0 0 0 0 :items 1]          1/4]
[d5     [:m 0 0 0 0 0 :items 1 :items 0] 0]
[e5     [:m 0 0 0 0 0 :items 1 :items 1] 1/32]
[f5     [:m 0 0 0 0 0 :items 1 :items 2] 1/16]
;; Inside a triplet (ratio 2/3) the same notes lie at 0, 1/48 and 1/24
```

These positions are structural, not rhythmic: they preserve the notated proportions within the group, not the moment at which the grace notes sound. A downstream step that needs the sounding position computes it, as `position-grace-notes-rhythmically` does in the [Remembered Alterations](#-real-world-example-remembered-alterations) pipeline below.

With `:grace-end-markers? true`, the walker emits `[:end :Grace nil]` after the last tuple of each grace group, so a stateful step can tell where a group ends without knowing its length. The marker is a three-element vector whose item is the keyword `:end` and whose position is `nil`; the `??` predicates return false for it, so `(filter pitch??)` removes it along with every other non-pitch tuple. A pipeline that does not ask for markers never sees one.

### Early Termination

Even with scope limiting, stop as soon as you find what you need, with `take` inside the composition:

```clojure
;; Find first forte passage in violin I, measures 10-50
(first
  (into []
        (comp (timewalk {:boundary-vpd [:m 0 0]
                         :start-measure 10
                         :end-measure 50})
              (filter forte-passage??)
              (take 1))   ; The walk stops when found
        [piece]))
```

**The hierarchy of efficiency**:
1. **Scope limiting** - Don't traverse what you don't need
2. **Measure ranges** - Further narrow the traversal window
3. **Early termination with `take` in the composition** - Stop as soon as goal achieved

## 🔴 Real-World Example: Remembered Alterations

You've now learned the building blocks: lazy sequences, transducers, composition with `comp`, consumption strategies, and push-based architecture. Let's see how these patterns compose into production code that solves a genuinely complex problem.

### The Problem

Determining which accidentals to display is one of the most complex aspects of musical notation—it's the very core of music formatting. The algorithm must balance key signatures, temporal memory, measure boundaries, grace notes, ties, and multi-voice coordination while handling dozens of edge cases and musical conventions.

Using transducers lets you break this complexity into manageable, composable pieces. Each transducer handles one part of the algorithm. Together, they implement the complete solution.

### The Pipeline

This is the pipeline at the core of `accidental-decisions-for-measure` in `ooloi.shared.fmt.remembered-alterations`, which decides the accidentals of one measure of an instrument:

```clojure
(transduce
 (comp
  (timewalk {:boundary-vpd instrument-vpd
             :start-measure measure-index
             :end-measure (inc measure-index)
             :end-position 0
             :grace-end-markers? true})
  (filter grace-pipeline-item?)
  (position-grace-notes-rhythmically piece measure-index)
  (merge-voices)
  (filter pitch??)
  (group-simultaneities)
  (detect-simultaneity-conflicts))
 (completing reducer)
 initial-state
 [piece])
```

`initial-state` is the accumulator, built by `create-initial-state` from what the previous measure handed on, and `reducer` is the two-argument reducing function returned by `process-pitch-tuple`; `completing` adds the one-argument arity that `transduce` calls at the end. The window reaches one position into the next measure, `:end-measure (inc measure-index)` with `:end-position 0`, because grace notes at the very start of the next measure sound at the end of this one, and their accidentals are decided here ([ADR-0035: Grace Notes at Measure Boundaries](../ADRs/0035-Remembered-Alterations.md#grace-notes-at-measure-boundaries)).

Eight pieces—seven transducers and one reducer—collectively implement the accidental algorithm. Each piece is focused and manageable. As the stream flows through the pipeline, each step transforms it for the next: adding data, modifying positions, consolidating voices, filtering, grouping events, all in preparation for the reducer where the decisions are made.

### How Data Flows Through the Pipeline

Let's trace how the stream transforms at each step:

**Step 1: Structure → Stream**
```clojure
(timewalk {:boundary-vpd instrument-vpd ...})
```
Transforms nested piece structure into a linear stream of `[item vpd position]` tuples. All staves and voices become a single temporal sequence. Includes structural markers (Instrument, Measure, Voice) for downstream processing and, because `:grace-end-markers? true` is given, an `[:end :Grace nil]` after each grace group.

**Step 2: Filter for grace pipeline**
```clojure
(filter grace-pipeline-item?)
```
Reduces the stream to items relevant for grace positioning: pitches, grace containers, grace end markers, and the Instrument, Measure and Voice markers. Other musical elements are filtered out early, minimising downstream work.

**Step 3: Transform grace positions**
```clojure
(position-grace-notes-rhythmically piece measure-index)
```
Walks the stream, adjusting rational positions for grace notes from metric to temporal positions. Each grace note's position field is recalculated based on tempo and available space. Grace containers and grace end markers are consumed here, and so are Voice markers, each of which first resets the previous position against which grace notes are placed. Instrument and Measure markers pass through unchanged.

**Step 4: Consolidate voices**
```clojure
(merge-voices)
```
Transparently handles multi-voice measures by consolidating voices across staves when necessary. For measures with 0-1 voices, tuples pass through unchanged (zero-consing). For measures with 2+ voices, tuples are accumulated and sorted by position before emission. Strategy determined per-measure-number using structural markers from the stream.

**Step 5: Filter for pitches**
```clojure
(filter pitch??)
```
Explicit boundary between structural processing and pitch processing. Filters stream to only pitch tuples, removing structural markers that were needed by earlier stages. Clear separation of concerns.

**Step 6: Group simultaneities**
```clojure
(group-simultaneities)
```
Identifies tuples at identical positions and marks them as simultaneities. Singles pass through unchanged. Only operates on pitches at this point.

**Step 7: Enrich groups with metadata**
```clojure
(detect-simultaneity-conflicts)
```
Scans grouped items, adding conflict metadata where needed. Singles still pass through unchanged. The stream is now prepared for the reducer.

**Step 8: Reduce to decisions**
```clojure
(completing (process-pitch-tuple piece key-sig (:prev-final initial-state) measure-index))
```
The stateful reducer. Receives the prepared stream and produces decisions. Threads one state map through each item (remembered alterations, courtesy accidentals shown, tied notes and the decisions so far) and returns it at the end, and `accidental-decisions-for-measure` builds `[partitions-structure decisions tied-target-endpoint-ids]` from it. The partitions structure and the tied endpoint ids are what the next measure starts from.

Notice the pattern: each transducer transforms the stream for the next stage. Structure becomes stream, stream gets filtered for grace processing, grace positions get adjusted, voices get consolidated, stream gets filtered for pitches only, pitches get grouped, groups get enriched, stream gets reduced to decisions. The pipeline has two clear phases: structural processing (steps 1-4) and pitch processing (steps 5-8). No single piece needs to understand the whole problem—each just transforms its input and passes the result along.

### Breaking Down Complexity

Why decompose the algorithm this way? Because the alternative is difficult to manage:

The accidental algorithm is complex because it must handle:

- Multi-staff, multi-voice temporal coordination
- Grace note positioning with collision detection
- Chord conflict detection for courtesy accidentals
- Remembered state updates after each pitch
- Tied note bypass across measure boundaries
- Key signature baseline and deviations
- Cross-octave courtesy accidental logic

**Without transducers**, you'd write one large function with nested conditionals and interleaved concerns—temporal ordering mixed with voice consolidation mixed with grace positioning mixed with state management mixed with decision logic. You'd probably be consing intermediate vectors at each step: one vector for filtered items, another for positioned items, another for merged voices, another for grouped items. Difficult to test, difficult to modify, and wasteful with memory.

**With transducers**, you separate concerns into eight focused pieces. Your mental focus shifts to the logical task at hand—what transformation does this step perform?—rather than nested iterators and recursive descent. The Ooloi transducers achieve this with zero consing in the normal single-voice case:

- `timewalk` handles temporal ordering and structural markers
- `filter grace-pipeline-item?` handles grace pipeline relevance
- `position-grace-notes-rhythmically` handles grace positioning
- `merge-voices` handles multi-voice consolidation (zero-consing for 0-1 voices)
- `filter pitch??` handles pitch pipeline boundary
- `group-simultaneities` handles chord grouping
- `detect-simultaneity-conflicts` handles conflict detection
- `process-pitch-tuple` handles state and decisions

Each piece is independently testable. Each piece is independently understandable. The composition implements the full algorithm.

### Connection to This Guide

Everything you've learned in this guide comes together here:

- **Structure → Stream** - Nested structure becomes temporal event stream
- **Tuples** - `[item vpd position]` carries context through pipeline
- **Filtering** - Select relevant items for processing
- **Composition** - `comp` combines transformations
- **Transduction** - Push-based processing through composed pipeline
- **Decomposition** - Complex algorithm split into focused pieces

**For complete algorithmic details** (baseline initialization, courtesy accidental rules, tied note bypass, French ties, keyless modes, conflict resolution), see [ADR-0035: Remembered Alterations](../ADRs/0035-Remembered-Alterations.md).

## Cross-References

- **Basic usage**: See [🟢 Starting Simple](#-starting-simple-direct-lazy-sequences) for your first timewalking operations
- **Performance**: See [Performance Comparison](#performance-comparison-threading-vs-transducers) for how threading and transducers compare
- **Benchmarks**: See [Timewalk Performance Benchmarks](https://github.com/PeterBengtson/Ooloi-docs/blob/main/READMEs/BENCHMARKS_README.md) for measured traversal times, including sub-millisecond cache refresh and 2-10 microsecond endpoint searches (2025 laptop)
- **🔴 Advanced concurrency**: See [Advanced Concurrency Patterns](ADVANCED_CONCURRENCY_PATTERNS.md) for parallel processing with STM coordination
- **VPD addressing**: See [VPDs Guide](VPDs.md) for understanding the Vector Path Descriptors returned in timewalking tuples
- **Type predicates**: See [Polymorphic API Guide](POLYMORPHIC_API_GUIDE.md) for the type predicates and helper functions used in filtering
- **Frontend rendering**: See [Frontend Architecture Guide](FRONTEND_ARCHITECTURE_GUIDE.md) for how timewalking results feed into the rendering pipeline
- **Architecture**: See [ADR-0014: Timewalk](../ADRs/0014-Timewalk.md) for the architectural decisions behind timewalking

## Further Reading

For more examples and advanced usage patterns, see:
- `ooloi.shared.ops.timewalk-test` for comprehensive test examples

## Envoi

*I’ve seen blue skies<br>
through the tears in my eyes,<br>
And I realise<br>
I’m going home.<br>
—Frank N. Furter*
