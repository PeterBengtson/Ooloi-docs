# 🟢 Timewalking Guide: Musical Traversal with Temporal Coordination

*"Let's do the timewalk again" - The Unconventional Conventionalists*

## Table of Contents

- [Overview](#overview)
- [Prerequisites and Intended Audience](#prerequisites-and-intended-audience)
- [What Timewalking Actually Does](#what-timewalking-actually-does)
- [What Timewalking Returns: The Power of Tuples](#what-timewalking-returns-the-power-of-tuples)
- [🟢 Starting Simple: Direct Lazy Sequences](#-starting-simple-direct-lazy-sequences)
- [🟡 The Magic of Composition](#-the-magic-of-composition)
- [Understanding Timewalk's Architecture: Push vs Pull](#understanding-timewalks-architecture-push-vs-pull)
- [Why Not Just Use `->>` for Everything?](#why-not-just-use-----for-everything)
- [🟡 More Practical Examples](#-more-practical-examples)
- [Core Concepts](#core-concepts)
- [Understanding Scope and Boundaries](#understanding-scope-and-boundaries)
- [Simple Practical Applications](#simple-practical-applications)
- [Core Temporal Coordination](#core-temporal-coordination)
- [Efficiency Tips](#efficiency-tips)
- [Working with Other Systems](#working-with-other-systems)
- [Best Practices](#best-practices)
- [Debugging and Troubleshooting](#debugging-and-troubleshooting)
- [Common Mistakes and Solutions](#common-mistakes-and-solutions)
- [Cross-References](#cross-references)

## Overview

The `timewalk` function provides powerful traversal of musical piece structures with **temporal coordination** across measures. This ensures proper musical time ordering essential for formatting, analysis, and finding musical elements.

This guide demonstrates how to use the `timewalk` function for musical data processing. The examples show practical operations like finding pitches, transposing melodies, and analyzing musical data. As the timewalker supports both direct lazy sequences and transducers, you'll see both approaches throughout.

## Prerequisites and Intended Audience

This guide assumes:
- **Comfortable fluency with basic Clojure syntax and REPL usage**
- **Familiarity with `map`, `filter`, and basic sequence operations**
- **Some exposure to music notation concepts** (measures, voices, pitches)
- **No prior experience with transducers required** - they're explained through musical examples

The guide progresses from basic timewalking to advanced transducer patterns. Each section builds on previous concepts, so working through sequentially is recommended.

## What Timewalking Actually Does

Before diving into examples, here's the key insight:

**Timewalking transforms your piece into a stream of musical events in time order.**

Instead of navigating a tree of musicians → instruments → staves → measures → voices, you get a single stream where everything flows in musical time: all of measure 1, then all of measure 2, then all of measure 3.

**What this means for you:**

- **No manual synchronization**: Want all forte passages? Just filter the stream. No tracking which voice you're in.
- **Natural musical thinking**: "Find the first C# after measure 10" is exactly what you write—no tree navigation.
- **Composable operations**: Layout, harmonic analysis, and data processing all use the same stream patterns.

Think of it like this: A score is a spatial thing (ink on paper, nested data structures). Music is a temporal thing (events in time). Timewalking bridges the gap—it transforms the spatial score into the temporal music.

The rest of this guide shows you how to work with that stream.

## What Timewalking Returns: The Power of Tuples

Every item returned by timewalking is a **three-element tuple** containing everything you need:

```clojure
[item vpd position]
```

- **`item`** - The actual musical object (pitch, chord, rest, measure, etc.)
- **`vpd`** - Vector Path Descriptor - the exact location of this item in the piece
- **`position`** - Rhythmic position within the measure (0, 1/4, 1/2, 3/4, etc.)

This tuple gives you **complete context** for any musical element:

```clojure
;; Example tuple for a pitch
[#Pitch{:note "C4"} [:musicians 0 :instruments 0 :staves 0 :measures 2 :voices 0 :items 1] 1/4]
;;   ^actual pitch    ^exact location in piece structure (staff → measure → voice → item)  ^beat position
```

With this information, you can:
- **Access the item**: Get its properties like `(:note pitch)`
- **Know its location**: Navigate to related elements using the VPD
- **Know its timing**: Understand rhythmic placement for analysis or formatting

### Essential Tuple Helpers and Predicates

Since the `timewalk` function returns `[item vpd position]` tuples, Ooloi provides both helper functions for tuple destructuring and specialized `??` predicates for filtering. These `??` predicates are counterparts to the normal `?` predicates, designed specifically for timewalk tuples and require a 3-element vector as input:

```clojure
;; Core destructuring helpers (from timewalk.clj)
(item result)      ; Extract the musical object (first element)
(vpd result)       ; Extract the location path (second element)  
(position result)  ; Extract the rhythmic position (third element)

;; Specialized ?? predicates - counterparts to normal ? predicates for timewalk tuples
;; The ?? symbolizes "descending into tuple" to check the item type
;; These REQUIRE a 3-element tuple [item vpd position] or will signal an error
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
(pitch?? raw-item)      ; ✗ IllegalArgumentException - expects 3-element tuple
(pitch?? [item vpd pos]) ; ✓ Correct usage with tuple
```

These helpers transform the timewalker from a low-level traversal tool into a readable musical analysis API.

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

**Important Performance Note:** When you see `->>` with lazy sequences (like timewalk returns), **no intermediate collections are created**. Each step processes one item at a time:

```clojure
;; This looks like it creates intermediate collections, but it doesn't
(->> (timewalk piece {})             ; Lazy sequence
     (filter pitch??)                ; Still lazy (note: ??)
     (map #(:note (item %)))         ; Still lazy
     (take 10))                      ; Still lazy, only realizes 10 items
```

The magic happens because the `timewalk` function returns a **lazy sequence**, so no "consing" (creating intermediate lists) occurs until you actually consume the results.

### Thread-Last vs Thread-First

There's also `->` (thread-first) which inserts as the **first argument**:

```clojure
;; ->> (thread-last) - for collections and data transformations
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (map #(:note (item %))))

;; -> (thread-first) - for objects and method calls  
(-> piece
    (get-musician 0)
    (get-instrument 0)
    (:name))
```

**Rule of thumb:** Use `->>` when working with collections (which is almost always with timewalking), use `->` when working with objects or nested data access.

**Connection to transducers:** The `->>` pattern shows you're doing step-by-step data transformation. Transducers take this same concept but make it more efficient by eliminating intermediate collections. The thought process is identical - it's just the implementation that's optimized.

## 🟢 Starting Simple: Direct Lazy Sequences

The easiest way to use the `timewalk` function is the **2-arity form** that returns a lazy sequence directly:

```clojure
(timewalk piece options)
```

Let's start with practical examples that show the power of this approach.

### Example 1: Find All Pitches

```clojure
;; Get all pitches in a staff (voices are now inside measures)
(def staff-vpd [:musicians 0 :instruments 0 :staves 0])

(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (map item))  ; Extract just the pitch objects
;; => [#Pitch{:note "C4"} #Pitch{:note "E4"} #Pitch{:note "G4"} ...]
```

### Example 2: Filter by Pitch Range

```clojure
;; Find pitches above middle C (using frequency comparison)
;; Note: hz>= compares frequencies, works with Pitch objects and pitch strings
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (filter #(hz>= (item %) "C4"))  ; Compare by frequency
     (map item))
;; => [#Pitch{:note "C4"} #Pitch{:note "E4"} #Pitch{:note "G4"} ...]
```

### Example 3: Get Pitch Positions

```clojure
;; Find pitches with their timing
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (map (fn [result]
            {:note (:note (item result))
             :measure (vpd/measure (vpd result))    ; Measure number from VPD
             :position (position result)})))       ; Beat position
;; => [{:note "C4" :measure 0 :position 0}
;;     {:note "E4" :measure 0 :position 1/4}
;;     {:note "G4" :measure 0 :position 1/2} ...]
```

### Example 4: Simple Processing Pipeline

```clojure
;; Find pitches, filter range, extract notes
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)  ; Only pitches
     (filter #(hz>= (item %) "C4"))  ; Above middle C
     (map #(:note (item %)))  ; Extract note names
     (take 10))  ; First 10 results
;; => ["C4" "E4" "G4" "C5" "E5" ...]
```

Notice how each step builds on the previous one, and we always have access to the full tuple `[item vpd position]` for complete context.

## 🟡 The Magic of Composition

*"The transducer will seduce ya" - Frank N. Furter*

Now here's where it gets powerful. Those steps above can be **composed** using transducers for elegance and efficiency.

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

;; It's like ->> but more efficient - no intermediate collections
(->> collection step1 step2 step3)           ; Creates intermediate lists
(sequence (comp step1 step2 step3) collection)  ; No intermediate lists
```

### Side-by-Side Comparison

```clojure
;; Traditional approach with ->>
(->> (timewalk piece {:boundary-vpd staff-vpd})       ; Returns lazy sequence
     (filter pitch??)                                 ; Creates new lazy sequence wrapper
     (filter #(hz>= (item %) "C4"))          ; Creates another lazy wrapper
     (map #(:note (item %)))                          ; Creates another lazy wrapper
     (take 10))                                       ; Creates final lazy wrapper
;; Total: 4 intermediate lazy sequence objects (lightweight, but still overhead)

;; Transducer approach with sequence and comp
(sequence (comp (filter pitch??)                      ; \
                (filter #(hz>= (item %) "C4")) ; > Combined into
                (map #(:note (item %)))               ; /  one transformation
                (take 10))                            ; /
          (timewalk piece {:boundary-vpd staff-vpd}))
;; Total: 1 processing chain - no intermediate lazy objects
```

**Key difference:** 
- **Threading**: Each step wraps the previous result in a new lazy sequence
- **Transducers**: All steps are composed into a single transformation function that processes items one-by-one

Both approaches are lazy and don't realize unnecessary elements, but transducers eliminate the wrapper object overhead.

### Performance Comparison: Threading vs Transducers

Here's why transducers tend to be more efficient for complex processing chains:

```clojure
;; Threading approach - creates multiple lazy sequence wrappers
(->> (timewalk large-symphony {})
     (filter pitch??)
     (map #(:note (item %)))
     (take 1000))
;; Memory: Creates 3 intermediate lazy sequence objects
;; Processing: Each step wraps the previous step in a new lazy sequence

;; Transducer approach - single processing chain
(into [] 
      (comp (filter pitch??)
            (map #(:note (item %)))
            (take 1000))
      (timewalk large-symphony {}))
;; Memory: Single processing chain, no intermediate objects
;; Processing: All steps fused into one efficient transformation
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

**Real-world benchmarks** show significant improvements:
- **Clojure community benchmarks**: ~45% faster execution (109ms → 60ms for multi-step transformations)
- **Memory efficiency**: Eliminates intermediate lazy sequence objects between steps
- **JVM optimization**: Better loop fusion opportunities compared to chained lazy operations
- **Scaling**: Performance gap widens with longer transformation chains

**Why transducers win on the JVM:**
- **No intermediate allocations**: Direct function composition vs lazy sequence wrappers
- **JIT-friendly**: More predictable execution patterns for JVM optimization
- **Cache efficiency**: Single-pass processing vs multiple sequence traversals

### What Are Transducers? A Brief Foundation

Before diving deeper, let's establish what transducers actually are:

**Transducers are transformation functions that are independent of the context in which they're used.**

**Push vs Pull — A Critical Distinction:**

Timewalk demonstrates a true push-based transducer: it *drives* the reducing function by calling it directly as items are discovered, rather than producing a lazy sequence that gets *pulled* when consumed. This distinction matters:

- **Pull (lazy)**: "I'll generate the next value when you ask for it"
- **Push (transducer)**: "Here's the next value right now — process it or stop"

The push model enables **early termination to stop computation itself**, not just sequence consumption. When `take 10` signals termination via `reduced`, timewalk immediately stops traversing the musical hierarchy — no wasted work generating values that will never be used.

This makes timewalk pedagogically valuable for understanding transducers: it shows how they compose transformations of *reducing functions* rather than transformations of *sequences*.

Think of them as "transformation recipes" that can be applied to any data source:

```clojure
;; This is a transducer - a transformation recipe
(def process-melody
  (comp (filter pitch??)
        (map #(:note (item %)))
        (take 10)))

;; The same recipe works with different data sources:
(sequence process-melody (timewalk piece1 opts))  ; Apply to piece1
(sequence process-melody (timewalk piece2 opts))  ; Apply to piece2
(into [] process-melody (timewalk piece3 opts))   ; Collect into vector
(transduce process-melody + 0 some-sequence)          ; Reduce to single value
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
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (sequence (comp (filter pitch??)
                     (filter #(hz>= (item %) "C4"))
                     (map #(:note (item %)))
                     (take 10))))
```

**Key insight:** This approach combines the best of both worlds:
- **Input**: Still uses the familiar `->>` threading pattern
- **Processing**: `sequence` with `comp` creates a single efficient transformation
- **No intermediate collections**: The composed transducer processes items one at a time
- **Result**: Lazy sequence that can be consumed incrementally

**Important distinction**: The `->>` here is just for readability - it's equivalent to:
```clojure
(sequence (comp (filter pitch??) ...) (timewalk piece {:boundary-vpd staff-vpd}))
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
(timewalk {:boundary-vpd staff-vpd})        ; Returns transducer
(timewalk piece {:boundary-vpd staff-vpd})  ; Returns lazy sequence
```

**Why This Pattern Works**

This isn't a novel design - it's how functional composition works in Clojure. The `timewalk` function behaves exactly like `map`, `filter`, and `take`, just with musical semantics instead of generic collection operations.

**Composition in Action**

The `timewalk` function itself can be part of the composition:

```clojure
;; One-arity form returns a transducer - timewalk becomes part of the composition
(sequence (comp (timewalk {:boundary-vpd staff-vpd})  ; Transducer form
                (filter pitch??)
                (filter #(hz>= (item %) "C4"))
                (map #(:note (item %)))
                (take 10))
          [piece])
;; => ["C4" "E4" "G4" "C5" "E5" ...]
```

This is **composable**, **reusable**, and **efficient** - the hallmarks of good functional design. More importantly, it integrates naturally with the Clojure ecosystem because it follows established conventions.

### Transducer Usage Patterns

The examples above demonstrate several key patterns:

- **Basic operations** - `filter` and `map` work the same way
- **Composition** - `comp` combines operations efficiently
- **Application** - `sequence` runs the transformation
- **Performance** - no intermediate collections
- **Reusability** - same transformation works on multiple pieces

Musical processing naturally involves multi-step operations: find → filter → transform → output. Transducers make this efficient for large scores and reusable across different pieces.

## Understanding Timewalk's Architecture: Push vs Pull

*"Yes, Brad, but doesn't it feel **nice**?" - Frank N Furter* 

You've been using two approaches to consume timewalking results:

```clojure
;; Approach 1: Threading with lazy sequences
(->> (timewalk piece {})
     (filter pitch??)
     (take 10))

;; Approach 2: Transducers with sequence
(sequence (comp (filter pitch??)
                (take 10))
          (timewalk piece {}))
```

Both work, but there's a fundamental architectural difference that affects performance and behavior.

### The Two Paradigms

**Pull-based (lazy sequences)**:
```
You ----request----> Lazy Seq ----request----> Timewalk
You <----value------ Lazy Seq <----value------ Timewalk
```
You pull values when needed. Timewalk generates them on demand.

**Push-based (true transducers)**:
```
Timewalk ----value----> Your Function ----result----> Collection
Timewalk ----value----> Your Function ----STOP!
```
Timewalk pushes values as it discovers them. You signal when to stop.

### Why This Matters: Early Termination

Here's where the difference becomes concrete:

```clojure
;; Find the first forte passage in a symphony
```

**With pull-based lazy sequences**:
```
╔════════════════════════════════════════════════════════╗
║  Symphony (100,000 elements)                           ║
║  ┌──────────────────────────────────────────────────┐ ║
║  │ Timewalk generates lazily...                     │ ║
║  │ Element 1 → check → no                           │ ║
║  │ Element 2 → check → no                           │ ║
║  │ Element 3 → check → YES! (take 1)                │ ║
║  │ ✓ Stop consuming... but timewalk doesn't know    │ ║
║  └──────────────────────────────────────────────────┘ ║
╚════════════════════════════════════════════════════════╝
```

**With push-based transducers**:
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

**The key insight**: With push-based transducers, `take 1` stops *computation itself*. The timewalk stops discovering and processing items the moment it receives a `reduced` signal.

### Timewalk's True Transducer Implementation

Timewalk implements a **push-based producer**. As it discovers musical items during traversal, it immediately calls your reducing function with each `[item vpd position]` tuple. This is why transducers with timewalk are genuinely efficient—there's no intermediate sequence being created and then consumed.

```clojure
;; What's actually happening (conceptual)
(defn timewalk [piece options]
  (fn [reducing-fn]  ; Returns a transducer
    (fn [acc item]   ; Your reducing function gets called directly
      ;; Timewalk discovers item → immediately calls your function
      ;; If you return (reduced acc), timewalk stops traversing
      (reducing-fn acc [item vpd position]))))
```

This is the distinction between:
- **Creating then consuming** (lazy sequences - pull)
- **Discovering and processing** (transducers - push)

### Three Ways to Consume Timewalking Results

Now you can make informed choices about how to consume timewalking results:

#### 1. Streaming (Never Materialise) - Push-Based

Use when you process each item once and discard it:

```clojure
;; MIDI export - process each note once
(sequence (comp (timewalk {:boundary-vpd staff-vpd})
                (filter pitch??)
                (map item->midi-event))
          [piece])
;; Memory: Constant (~10 MB regardless of piece size)
;; Processing: Zero intermediate allocation

;; Early termination - stops computation immediately
(first
  (into []
        (comp (timewalk {})
              (filter forte-passage?)
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
        (comp (timewalk {:boundary-vpd staff-vpd})
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

#### 3. Lazy Pull Sequences - Simple but Less Optimal

Use when simplicity matters more than optimal performance:

```clojure
;; Simple threading - creates lazy sequence wrappers
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (map #(:note (item %)))
     (take 10))

;; Cost: Lazy sequence wrapper overhead
;; Benefit: Familiar, easy to read, good enough for small pieces
```

**When lazy sequences are fine**:
- Small pieces (< 1,000 elements)
- Prototyping and experimentation
- REPL exploration
- Simplicity is the priority

### Performance Characteristics: Real Benchmarks

Here are actual numbers from a 1,000-measure orchestral piece (29 instruments, 520,000 pitches):

```
╔═══════════════════════════════════════════════════════════╗
║  Streaming (push-based transducer)                        ║
║  ───────────────────────────────────────────────────────  ║
║  Full traversal:        582 ms (~1M pitches/second)       ║
║  Cache refresh:         1-5 ms (10-50 measure windows)    ║
║  Memory usage:          <10 MB (constant, any size)       ║
║  Streaming export:      1.25 seconds                      ║
║  Typical slur search:   <100 microseconds                 ║
╚═══════════════════════════════════════════════════════════╝

╔═══════════════════════════════════════════════════════════╗
║  Materialisation (into vector)                            ║
║  ───────────────────────────────────────────────────────  ║
║  Overhead:              <5% (just vector allocation)      ║
║  Memory cost:           ~437 bytes per tuple              ║
║  Total for 520K items:  244 MB                            ║
║  Access pattern:        O(1) random access                ║
╚═══════════════════════════════════════════════════════════╝

╔═══════════════════════════════════════════════════════════╗
║  Lazy sequences (pull-based)                              ║
║  ───────────────────────────────────────────────────────  ║
║  Overhead:              Wrapper allocations per step      ║
║  Early termination:     Stops consumption, not compute    ║
║  Good for:              Small pieces, prototyping         ║
╚═══════════════════════════════════════════════════════════╝
```

### Decision Guide: Which Approach to Use

```clojure
;; Export operations → Streaming (push-based)
(sequence (comp (timewalk {}) (map item->midi-event)) [piece])

;; Need random access → Materialise
(into [] (comp (timewalk {}) (filter pitch??)) [piece])

;; Finding first match → Streaming with early termination
(first (into [] (comp (timewalk {}) (filter target?) (take 1)) [piece]))

;; Simple exploration → Lazy sequences (threading)
(->> (timewalk piece {}) (filter pitch??) (take 5))
```

### Why Timewalk Uses Push Architecture

This architectural choice enables:

1. **True early termination** - Stop computation itself, not just consumption
2. **Zero intermediate allocation** - No lazy sequence wrappers between steps
3. **Constant memory streaming** - Process million-note symphonies in <10 MB
4. **Composability** - Combine with all Clojure transducer operations

The push-based architecture is what makes timewalk efficient enough for professional orchestral scores while maintaining the elegant compositional properties of functional programming.

### The Practical Takeaway

**For most work**: Use push-based transducers with `sequence`, `into`, or `transduce`. The performance benefits are substantial and the patterns become natural quickly.

**For exploration**: Lazy sequences with `->>` threading are perfectly fine and more readable for learning.

**For random access**: Materialise into a vector explicitly when you need it.

The architecture is designed so the right choice is usually the simplest one.

## Why Not Just Use `->>` for Everything?

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
;; Small chamber piece (1000 total elements) - either approach works fine
(->> (timewalk string-quartet {})
     (filter pitch??)
     (map (comp (make-transposer :up :major :second) item)))

;; Large symphony (50,000+ total elements) - transducers prevent memory pressure
(into []
      (comp (filter pitch??)
            (map (comp (make-transposer :up :major :second) item)))
      (timewalk symphony {}))
```

### Idiomatic Transducer Usage

Common transducer patterns for musical processing:

#### Pattern 1: Direct with `into`
```clojure
;; Most common: transform and collect into a specific collection type
(into [] 
      (comp (filter pitch??)
            (map #(:note (item %))))
      (timewalk piece {:boundary-vpd staff-vpd}))
;; => ["C4" "E4" "G4" "C5" ...]

;; Into a set for unique notes
(into #{} 
      (comp (filter pitch??)
            (map #(:note (item %))))
      (timewalk piece {:boundary-vpd staff-vpd}))
;; => #{"C4" "E4" "G4" "C5" ...}
```

#### Pattern 2: Direct with `transduce`
```clojure
;; Calculate average pitch frequency
(let [freqs (transduce (comp (filter pitch??)
                             (map #(hz (item %))))
                       conj
                       []
                       (timewalk piece {:boundary-vpd staff-vpd}))]
  (/ (reduce + freqs) (count freqs)))
;; => 415.3 (average frequency in Hz)

;; Count pitches above middle C
(transduce (comp (filter pitch??)
                 (filter #(hz>= (item %) "C4"))
                 (map (constantly 1)))  ; Turn each item into 1
           +
           0
           (timewalk piece {:boundary-vpd staff-vpd}))
;; => 42 (count of high pitches)
```

#### Pattern 3: With `sequence` (when you need a lazy seq)
```clojure
;; When you want lazy evaluation - only what you consume
(sequence (comp (filter pitch??)
                (map #(:note (item %)))
                (take 10))
          (timewalk piece {:boundary-vpd staff-vpd}))
;; => ("C4" "E4" "G4" "C5" "E5" ...)  ; Lazy sequence
```

#### Pattern 4: One-arity transducer form (for reuse)
```clojure
;; Create reusable formatting transformation
(def visible-elements
  (comp (filter visible?)                    ; Only visible elements
        (filter rhythmic-item?)              ; Items that take space
        (map (fn [result]
               {:element (item result)
                :vpd (vpd result)
                :position (position result)
                :layout-width (calculate-width (item result))}))))

;; Use with any piece or section
(into [] visible-elements (timewalk piece1 {:boundary-vpd staff-vpd}))
(into [] visible-elements (timewalk piece2 {:boundary-vpd system-vpd}))
(transduce visible-elements conj [] (timewalk piece3 {:boundary-vpd page-vpd}))

;; Timing transformation for MIDI and playback
(def absolute-time
  "Predefined transducer that converts position to absolute time from piece start.
   This simplified version assumes 4/4 time and constant tempo. Full version handles:
   - All time signatures and meter changes
   - Gradual tempo changes with different curvature algorithms
   - Complex rhythmic subdivisions within measures"
  (map (fn [result]
         (let [measure-num (vpd/measure (vpd result))
               measure-pos (position result)
               ;; Simplified: 4/4 time, 4 beats per measure, 120 BPM
               beats-per-measure 4
               seconds-per-beat 0.5  ; 120 BPM = 0.5 seconds/beat
               absolute-time (+ (* measure-num beats-per-measure seconds-per-beat)
                               (* measure-pos seconds-per-beat))]
           [(item result) (vpd result) absolute-time]))))

;; Examples with output
(sequence (comp (filter pitch??) absolute-time (take 3))
          (timewalk piece {:boundary-vpd staff-vpd}))
;; => ([#Pitch{:note "C4"} [:musicians 0 :instruments 0 :staves 0 :measures 0 :voices 0 :items 0] 0.0]
;;     [#Pitch{:note "E4"} [:musicians 0 :instruments 0 :staves 0 :measures 0 :voices 0 :items 1] 0.5]
;;     [#Pitch{:note "G4"} [:musicians 0 :instruments 0 :staves 0 :measures 0 :voices 0 :items 2] 1.0])

(sequence (comp (filter pitch??) absolute-time)
          (timewalk piece {:boundary-vpd staff-vpd 
                           :start-measure 2 :end-measure 2}))
;; => ([#Pitch{:note "F4"} [:musicians 0 :instruments 0 :staves 0 :measures 2 :voices 0 :items 0] 4.0]
;;     [#Pitch{:note "A4"} [:musicians 0 :instruments 0 :staves 0 :measures 2 :voices 0 :items 1] 4.5]
;;     [#Pitch{:note "C5"} [:musicians 0 :instruments 0 :staves 0 :measures 2 :voices 0 :items 2] 5.0]
;;     [#Pitch{:note "E5"} [:musicians 0 :instruments 0 :staves 0 :measures 2 :voices 0 :items 3] 5.5])
```

#### Pattern 5: Early Termination with `take`
```clojure
;; One of transducers' superpowers: efficient early termination
(into [] 
      (comp (filter pitch??)
            (take 1))  ; Stop after finding the first pitch
      (timewalk piece {:boundary-vpd staff-vpd}))
;; => [#Pitch{:note "C4"}]
;; Stops processing as soon as 1 pitch is found - doesn't traverse the entire piece

;; Find first 5 high notes efficiently in the development section
(into [] 
      (comp (filter pitch??)
            (filter #(hz>= (item %) "C5"))  ; Above C5
            (take 5))  ; Stop after 5 high notes
      (timewalk piece {:boundary-vpd staff-vpd :start-measure 32 :end-measure 64}))
;; Stops immediately after finding 5 qualifying pitches

;; Get first pitch frequency (demonstrates early termination)
(->> (timewalk piece {:boundary-vpd staff-vpd})  ; Returns lazy seq of [item vpd position] tuples
     (filter pitch??)                             ; Filters to only pitch tuples
     (take 1)                                     ; Takes first tuple
     first                                        ; Extracts the tuple - now we have [item vpd position]
     item                                         ; Extracts item from tuple
     hz)                                          ; Gets frequency in Hz
;; => 261.63 (frequency in Hz of first pitch - stops immediately)
```

**Why this matters for musical processing:**

Consider finding the first forte passage in a symphony:

```clojure
;; INEFFICIENT: Traditional threading processes entire 45-minute work
(->> (timewalk symphony {})  ; Processes ALL 100,000+ elements
     (filter forte-passage?)                    ; Scans entire symphony 
     (first))                                   ; Gets first result after full scan
;; Waste: Processed 100,000 items to find 1

;; EFFICIENT: Transducers with early termination stop immediately  
(first
  (into []
        (comp (filter forte-passage?)
              (take 1))                         ; Stops at first match
        (timewalk symphony {})))
;; Smart: Stops after finding 1 forte passage, might process only 50 items
```

For large orchestral scores, this is the difference between processing the entire work vs. stopping at the first relevant element.

```clojure
;; Traditional (inefficient) - processes everything, then takes 1
(->> (timewalk piece {})  ; Processes ENTIRE piece
     (filter pitch??)                         ; Filters ENTIRE result
     (take 1))                              ; Then takes first one

;; Transducer (efficient) - stops as soon as 1 is found
(into [] 
      (comp (filter pitch??)
            (take 1))                       ; Stops immediately after finding 1
      (timewalk piece {}))
```

### Critical Performance Rule: Always Use `take 1` Before `first`

**NEVER do this:**
```clojure
(->> (timewalk piece options)
     (filter some-predicate)
     (first))  ; BAD: Forces realization of entire filtered sequence
```

**ALWAYS do this:**
```clojure
(->> (timewalk piece options)
     (filter some-predicate)
     (take 1)   ; GOOD: Early termination, stops after finding 1
     (first))   ; Extract from single-item sequence
```

**Why this is critical:** `first` forces the realization of the lazy sequence just to get one element. `take 1` maintains laziness and enables early termination, stopping the moment a match is found. For orchestral scores with thousands of elements, this is the difference between processing 1 item vs. thousands.

**Key insight:** Transducers separate the transformation from the collection type, enabling maximum reusability and efficiency.

## 🟡 More Practical Examples

### Example 5: Transpose Pitches

```clojure
;; Transpose all pitches up a major third
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (map (let [transposer (make-transposer :up :major :third)]
            (fn [result]
              [(transposer (item result)) (vpd result) (position result)])))
```

### Example 6: Format Elements for Layout

```clojure
;; RECOMMENDED: Compose timewalk with visible-elements for large orchestral works
(into [] 
      (comp (timewalk {:boundary-vpd staff-vpd :start-measure 10 :end-measure 15})
            visible-elements)       ; Single fused pipeline - no intermediate allocations
      [piece])
;; => [{:element #Pitch{:note "C4"} :vpd [...] :position 0 :layout-width 12.5}
;;     {:element #Chord{:pitches [...]} :vpd [...] :position 1/4 :layout-width 18.0}
;;     {:element #Rest{:duration 1/4} :vpd [...] :position 1/2 :layout-width 8.0} ...]

;; Reusable composed transformation for different sections
(def format-for-layout
  (comp (timewalk {:boundary-vpd staff-vpd})
        visible-elements))

(sequence format-for-layout [piece])  ; Apply to any piece

;; Alternative (less efficient): separate application creates intermediate lazy sequence
(sequence visible-elements          ; Creates intermediate allocation
          (timewalk piece {:boundary-vpd system-vpd :start-measure 0 :end-measure 5}))
```

### Example 7: Complete Processing Chain

Here's a multi-step transformation - find specific pitches, transpose them, collect results:

```clojure
;; Find C4 pitches, transpose up a major third, extract notes
(def transposer (make-transposer :up :major :third))

(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (filter #(= "C4" (:note (item %))))
     (map (fn [result]
            (let [transposed (transposer (item result))]
              {:original "C4"
               :transposed (:note transposed)
               :measure (vpd/measure (vpd result))
               :position (position result)})))
     (take 10))
;; => ({:original "C4" :transposed "E4" :measure 0 :position 0}
;;     {:original "C4" :transposed "E4" :measure 2 :position 1/2} ...)
```

### Example 8: Reusable Transducer Version

```clojure
;; Define transformation once, apply to multiple pieces
(def analyze-c4-transpositions
  (let [transposer (make-transposer :up :major :third)]
    (comp (timewalk {:boundary-vpd staff-vpd})
          (filter pitch??)
          (filter #(= "C4" (:note (item %))))
          (map (fn [result]
                 (let [transposed (transposer (item result))]
                   {:original "C4"
                    :transposed (:note transposed)
                    :measure (vpd/measure (vpd result))})))
          (take 10))))

;; Apply to multiple pieces efficiently
(sequence analyze-c4-transpositions [piece1])
(sequence analyze-c4-transpositions [piece2])
(sequence analyze-c4-transpositions [piece3])
```

The transducer version is **reusable** across multiple pieces and **composable** with other transformations.

## Core Concepts

### Temporal Coordination
Unlike structural traversal, timewalking processes **"measure N across all voices before measure N+1 in any voice"**. This ensures proper musical time ordering:

```clojure
;; CORRECT: Temporal coordination
;; Measure 0: Voice A, Voice B, Voice C
;; Measure 1: Voice A, Voice B, Voice C  
;; Measure 2: Voice A, Voice B, Voice C

;; WRONG: Structural traversal
;; Voice A: Measures 0,1,2,3,4,5...
;; Voice B: Measures 0,1,2,3,4,5...
;; Voice C: Measures 0,1,2,3,4,5...
```

### Boundary-VPD Scope Control
Use `:boundary-vpd` to precisely limit traversal scope:

```clojure
;; Whole piece
{}

;; Single musician
{:boundary-vpd [:musicians 4]}

;; Single instrument
{:boundary-vpd [:musicians 4 :instruments 0]}

;; Single staff (all measures, all voices)
{:boundary-vpd [:musicians 4 :instruments 0 :staves 2]}

;; Single measure across all voices in a staff
{:boundary-vpd [:musicians 4 :instruments 0 :staves 2 :measures 5]}

;; Single voice within a specific measure
{:boundary-vpd [:musicians 4 :instruments 0 :staves 2 :measures 5 :voices 0]}
```

### Start Position Control
Use `:start-measure` and `:start-position` to begin traversal from specific points:

```clojure
;; Start from measure 5 (0-based indexing)
{:boundary-vpd staff-vpd :start-measure 5}

;; Start from measure 3, position 1/2 (middle of measure)
{:boundary-vpd staff-vpd :start-measure 3 :start-position 1/2}

;; Start from beginning of measure 0, position 1/4
{:boundary-vpd staff-vpd :start-measure 0 :start-position 1/4}
```

### End Position Control
Support for stopping traversal at specific points (inclusive):

```clojure
;; Stop at measure 10 (inclusive)
{:boundary-vpd staff-vpd :end-measure 10}

;; Stop at measure 8, position 3/4 (inclusive)
{:boundary-vpd staff-vpd :end-measure 8 :end-position 3/4}

;; Traverse specific range (measures 2-5, both inclusive)
{:boundary-vpd staff-vpd :start-measure 2 :end-measure 5}

;; Single measure: start-measure 5, end-measure 5 captures exactly one measure
{:boundary-vpd staff-vpd :start-measure 5 :end-measure 5}

;; Cross-measure end-position: all of measure 5 + position 0 in measure 6
{:boundary-vpd staff-vpd :start-measure 5 :end-measure 6 :end-position 0}
```

## Understanding Scope and Boundaries

### Boundary Control with VPDs

The `:boundary-vpd` option lets you focus on exactly the part of the piece you need:

```clojure
;; Whole piece
(timewalk piece {})

;; Single musician (all their instruments)
(timewalk piece {:boundary-vpd [:musicians 0]})

;; Single instrument (all its staves)
(timewalk piece {:boundary-vpd [:musicians 0 :instruments 0]})

;; Single staff (all measures, all voices)
(timewalk piece {:boundary-vpd [:musicians 0 :instruments 0 :staves 0]})
```

### Start and End Position Control

Control exactly which measures and positions to process:

```clojure
;; Start from measure 5
(timewalk piece {:boundary-vpd staff-vpd :start-measure 5})

;; Process measures 2 through 8 (inclusive)
(timewalk piece {:boundary-vpd staff-vpd :start-measure 2 :end-measure 8})

;; Items that start from beginning up to and including position 1/2 in measure 3
(timewalk piece {:boundary-vpd staff-vpd
                 :start-measure 3
                 :end-measure 3
                 :end-position 1/2})

;; Find ties across barlines: find the tie endpoint for a C4 pitch starting at measure 4
(let [source-pitch (create-pitch :note "C4" :duration 1/16)]
  (->> (timewalk piece {:boundary-vpd staff-vpd 
                        :start-measure 4
                        :end-measure 5 :end-position 0})
       (filter takes-attachment??)  ; Only items that can have ties attached
       (filter #(enharmonically-equal? source-pitch (item %)))  ; Same pitch
       (take 1)  ; Early termination - only need one match
       (first)))  ; Extract the endpoint item
```

## Simple Practical Applications

### Finding Specific Elements

```clojure
;; Find all measures in a piece
(->> (timewalk piece {})
     (filter measure??)
     (map (fn [result] 
            {:number (vpd/measure (vpd result)) 
             :time-sig (:time-signature (item result))})))

;; Find all rests in a voice
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter rest??)
     (map (fn [result] 
            {:duration (:duration (item result)) 
             :measure (vpd/measure (vpd result)) 
             :position (position result)})))

;; Count pitches per measure
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (group-by #(vpd/measure (vpd %)))  ; Group by measure number
     (map (fn [[measure pitches]] [measure (count pitches)])))
```

### Simple Modifications

```clojure
;; Change all quarter notes to eighth notes
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter #(and (pitch?? %) (= (:duration (item %)) 1/4)))
     (map (fn [result]
            (assoc (item result) :duration 1/8))))

;; Find notes that need accidentals
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (filter #(needs-accidental? (:note (item %))))
     (map (fn [result] {:note (:note (item result)) :measure (vpd/measure (vpd result))})))
```


### Building More Complex Chains

Once you understand the basics, you can build more sophisticated processing:

#### Multi-Step Filtering
```clojure
;; Find loud high notes in a specific range
(sequence (comp (timewalk {:boundary-vpd staff-vpd :start-measure 10 :end-measure 20})
                (filter pitch??)
                (filter #(hz>= (item %) "C5"))  ; Above C5
                (filter #(>= (:velocity (item %) 64) 80))  ; Loud
                (map (fn [result]
                       {:note (:note (item result)) 
                        :measure (vpd/measure (vpd result)) 
                        :position (position result)})))
          [piece])
```

#### Grouping and Counting
```clojure
;; Count note frequencies across measures
(sequence (comp (timewalk {:boundary-vpd staff-vpd})
                (filter pitch??)
                (map #(:note (item %)))
                (map (fn [note] [note 1]))  ; Turn into [note count] pairs
                ;; Could add a custom reducing transducer here for frequencies
                )
          [piece])
```

#### Early Termination for Efficiency
```clojure
;; Find first 5 chords and stop
(sequence (comp (timewalk {:boundary-vpd staff-vpd})
                (filter chord??)
                (take 5)  ; Stop after finding 5 chords
                (map (fn [result] 
                       {:measure (vpd/measure (vpd result)) 
                        :notes (map :note (:pitches (item result)))})))
          [piece])
```

## Core Temporal Coordination

### Why Temporal Order Matters: What Goes Wrong Without It

The timewalker processes **"measure N across all voices before measure N+1 in any voice"**. This temporal coordination prevents serious musical analysis problems.

**WRONG: Structural traversal breaks musical meaning**
```clojure
;; Tree traversal (DON'T DO THIS for musical analysis)
(tree-seq coll? identity piece)
;; Result: Voice A (measures 0,1,2,3,4,5...), then Voice B (measures 0,1,2,3,4,5...)
;; Problem: Can't find what happens at measure 47 across all instruments
;; You get: All of violin I, then all of violin II, then all of viola...

;; Trying to find measure 47 events:
(->> (tree-seq coll? identity piece)
     (filter #(and (measure?? %) (= 47 (measure-number %))))
     (mapcat get-items))
;; Result: Broken - gets measures in voice-by-voice order, not simultaneous events
```

**RIGHT: Temporal coordination preserves musical relationships**  
```clojure
;; Timewalker temporal order
(timewalk piece {})
;; Result: Measure 0 (all voices), Measure 1 (all voices), Measure 2 (all voices)...
;; Benefit: Can analyze harmonic progressions, find tutti passages, locate simultaneous events

;; Finding measure 47 events across all voices:
(->> (timewalk piece {})
     (filter #(= 47 (vpd/measure (vpd %))))  ; All items at measure 47
     (filter pitch??))
;; Result: All pitches happening at measure 47, properly coordinated in time
```

**Real-world consequence:** Without temporal coordination, you can't:
- Generate proper MIDI (events out of time order)
- Analyze harmonic progressions (can't see simultaneous notes)
- Find tutti passages (can't identify when all instruments play together)
- Format scores correctly (measures appear voice-by-voice instead of temporally)

### Practical Benefits

This ordering enables:

```clojure
;; Find all simultaneous events at measure 5
(->> (timewalk piece {})
     (filter #(= (vpd/measure (vpd %)) 5))
     (filter pitch??))

;; Collect pitches with their measure numbers - naturally time-ordered
(->> (timewalk piece {})
     (filter pitch??)
     (map (fn [result]
            {:note (:note (item result))
             :measure (vpd/measure (vpd result))
             :position (position result)})))
;; Already in temporal order - no sorting needed

;; Format measures correctly
(->> (timewalk piece {})
     (filter measure??)
     ;; Measures come in temporal order for proper formatting
     (map format-measure-for-layout))
```

## Efficiency Tips

### Only Process What You Need

```clojure
;; Use scope to limit work
(->> (timewalk piece {:boundary-vpd staff-vpd})  ; Just one voice
     (filter pitch??))

;; Use measure ranges to limit work  
(->> (timewalk piece {:boundary-vpd staff-vpd :start-measure 10 :end-measure 15})
     (filter pitch??))

;; Use take for early termination
(->> (timewalk piece {})
     (filter chord??)
     (take 5))  ; Stops as soon as 5 chords found
```

### Memory Efficiency

```clojure
;; Lazy sequences - only generate what you consume
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (first))  ; Only processes until first pitch found

;; Transducers - no intermediate collections
(sequence (comp (timewalk {:boundary-vpd staff-vpd})
                (filter pitch??)
                (take 10))
          [piece])  ; Processes 10 pitches without creating intermediate lists
```

## Working with Other Systems

### Finding Attachment Endpoints

Ooloi's attachment system uses **endpoint-id** to connect attachments (slurs, hairpins, glissandos) to their target items. The timewalker provides the foundation for implementing `endpoint-item` resolution:

```clojure
;; Find an item by its endpoint-id (for slurs, hairpins, glissandos)
;; ALWAYS use take 1 instead of first to avoid forcing lazy sequence realization
(->> (timewalk piece {:boundary-vpd instrument-vpd})
     (filter #(and (takes-attachment?? %)  ; Pitch or Chord
                   (= endpoint-id (get-endpoint-id (item %)))))
     (take 1)  ; Early termination - stops after finding first match
     (first))  ; Extract from the single-item sequence

;; Even better: Use transducers for maximum efficiency in a specific section
(first
  (into [] 
        (comp (filter #(and (takes-attachment?? %)
                            (= endpoint-id (get-endpoint-id (item %)))))
              (take 1))  ; Stops immediately after finding the endpoint
        (timewalk piece {:boundary-vpd instrument-vpd 
                         :start-measure 12 :end-measure 20})))

;; Example: Resolve slur endpoint
(let [slur (get-attachment start-pitch :slur)
      endpoint-id (:endpoint-id slur)
      result (->> (timewalk piece {:boundary-vpd instrument-vpd})
                  (filter #(and (takes-attachment?? %)
                                (= endpoint-id (get-endpoint-id (item %)))))
                  (take 1)  ; Critical: early termination
                  (first))  ; Extract the actual result
      end-item (item result)
      end-vpd (vpd result)
      end-position (position result)]
  (when end-item
    (println "Slur connects to" (:note end-item) "at measure" (vpd/measure end-vpd))))
```

The temporal coordination ensures proper musical time ordering for attachment resolution.

### Generating Audio/MIDI Output (Conceptual)

Temporal coordination means events are naturally time-ordered for audio output:

```clojure
;; Conceptual: Convert pitches to playback events with timing
;; (Note: hz function takes Pitch object or pitch string, returns frequency in Hertz)
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     absolute-time
     (map (fn [[pitch vpd abs-time]]
            {:frequency (hz pitch)  ; Get frequency in Hz
             :velocity 64
             :time abs-time  ; Already in time order
             :channel 0})))

;; With transducers for efficiency
(sequence (comp (timewalk {:boundary-vpd staff-vpd})
                (filter pitch??)
                absolute-time
                (map (fn [[pitch vpd abs-time]]
                       {:frequency (hz pitch)
                        :velocity 64
                        :time abs-time
                        :channel 0})))
          [piece])
```

### Formatting for Layout

Elements come in temporal order with proper layout data:

```clojure
;; RECOMMENDED: Compose transducers for maximum efficiency with large orchestral works
(into [] 
      (comp (timewalk {:boundary-vpd staff-vpd})
            visible-elements)       ; Single fused pipeline - eliminates intermediate allocations
      [piece])
;; => [{:element #Pitch{} :vpd [...] :position 0 :layout-width 12.5}
;;     {:element #Rest{} :vpd [...] :position 1/4 :layout-width 8.0} ...]

;; Format specific measure range for page layout
(sequence (comp (timewalk {:boundary-vpd staff-vpd :start-measure 10 :end-measure 20})
                visible-elements)   ; Efficient single-pass processing
          [piece])
;; Elements ready for layout engine: visibility, spacing, and positioning included

;; Multi-staff system formatting with reusable transformation
(def system-layout-formatter
  (comp (timewalk {:boundary-vpd system-vpd :start-measure 0 :end-measure 5})
        visible-elements))

(into [] system-layout-formatter [piece])  ; Apply to any piece efficiently
```

## Best Practices

### 1. Choose the Right Arity
```clojure
;; Use 2-arity for simple, direct access
(timewalk piece {:boundary-vpd staff-vpd})

;; Use 1-arity for complex transducer chains
(sequence (comp (timewalk {:boundary-vpd staff-vpd})
                (filter pitch??)
                (map analyze-pitch)
                (take 100))
          [piece])
```

### 2. Leverage Temporal Coordination
```clojure
;; GOOD: Respects musical time
(->> (timewalk piece {:boundary-vpd [] :start-measure 9 :end-measure 9})
     (filter pitch??))

;; AVOID: May miss temporal relationships  
(tree-seq coll? identity piece)  ; Don't do this for musical analysis
```

### 3. Use Appropriate Scope
```clojure
;; Efficient: Limit scope to what you need
(timewalk piece {:boundary-vpd [:musicians 0 :instruments 0]})

;; Inefficient: Searching whole piece when you only need one instrument
(timewalk piece {})
```

### 4. Compose Thoughtfully
```clojure
;; GOOD: Efficient filtering early in chain with early termination
(comp (timewalk {:boundary-vpd staff-vpd})
      (filter pitch??)
      (take 10)           ; Early termination - stops after 10 pitches
      (map analyze-harmony))

;; LESS EFFICIENT: Expensive operations before filtering
(comp (timewalk {:boundary-vpd staff-vpd})
      (map expensive-analysis)
      (filter pitch??))

;; EXCELLENT: Combine filtering with take for maximum efficiency
(->> (timewalk piece {:boundary-vpd staff-vpd})
     (filter pitch??)
     (take 5)  ; Stop after first 5 pitches
     (map item))  ; Extract just the pitch objects
```


## Debugging and Troubleshooting

### Inspecting Traversal Order
```clojure
;; Add debugging to see traversal order
(->> (timewalk piece {:boundary-vpd boundary-vpd})
     (take 20)  ; first 20 items
     (map (fn [[item vpd pos]]
            {:type (type item)
             :vpd vpd
             :position pos
             :measure (vpd/measure vpd)
             :preview (str item)})))
```

### Validating Results
```clojure
;; Ensure temporal ordering is correct
(let [results (timewalk piece {:boundary-vpd staff-vpd})
      measures (->> results (map #(-> % vpd (get 9))))]
  (= measures (sort measures)))
```

## Common Mistakes and Solutions

### ❌ Using `first` Instead of `take 1`
```clojure
;; NEVER do this - forces entire lazy sequence realization
(->> (timewalk symphony {})
     (filter forte-passage?)
     (first))  ; BAD: processes entire symphony to get one result

;; ALWAYS do this - early termination
(->> (timewalk symphony {})
     (filter forte-passage?)
     (take 1)   ; GOOD: stops after finding first match
     (first))   ; Extract from single-item sequence
```

### ❌ Manual Destructuring Instead of Helper Functions
```clojure
;; Hard to read and error-prone
(filter (fn [[item _ _]] (isa? (type item) ::h/Pitch)) results)

;; Clear and maintainable  
(filter pitch?? results)
```

### ❌ Ignoring Temporal Coordination Benefits
```clojure
;; Missing the point - use regular tree traversal instead
(tree-seq coll? identity piece)  ; DON'T do this for musical analysis

;; Leveraging temporal coordination properly
(timewalk piece {})  ; DO this for musical analysis
```


## Cross-References

- **Basic usage**: See [🟢 Starting Simple](#-starting-simple-direct-lazy-sequences) for your first timewalking operations
- **Performance**: See [Performance Comparison](#performance-comparison-threading-vs-transducers) for timing benchmarks
- **Benchmarks**: See [Timewalk Performance Benchmarks](https://github.com/PeterBengtson/Ooloi-docs/blob/main/READMEs/BENCHMARKS_README.md) for comprehensive performance validation showing sub-millisecond cache refresh, constant-memory streaming, and sub-100 microsecond endpoint searches
- **Temporal coordination**: See [Why Temporal Order Matters](#why-temporal-order-matters-what-goes-wrong-without-it) for musical analysis foundations
- **🔴 Advanced concurrency**: See [Advanced Concurrency Patterns](ADVANCED_CONCURRENCY_PATTERNS.md) for parallel processing with STM coordination
- **VPD addressing**: See [VPDs Guide](VPDs.md) for understanding the Vector Path Descriptors returned in timewalking tuples
- **Type predicates**: See [Polymorphic API Guide](POLYMORPHIC_API_GUIDE.md) for the type predicates and helper functions used in filtering
- **Architecture**: See [ADR-0014: Timewalk](../ADRs/0014-Timewalk.md) for the architectural decisions behind timewalking

## Further Reading

For more examples and advanced usage patterns, see:
- `ooloi.shared.ops.timewalk-test` for comprehensive test examples

## Envoi

*I’ve seen blue skies through the tears in my eyes,<br>
And I realise... I’m going home.<br>
—Frank N. Furter*
