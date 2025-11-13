# ADR-0035: Remembered Alterations

## Table of Contents
- [Status](#status)
- [Context](#context)
  - [Problem Statement](#problem-statement)
- [Decision](#decision)
  - [Core Algorithm](#core-algorithm)
  - [Data Structure](#data-structure)
  - [Measure Boundary Behavior](#measure-boundary-behavior)
  - [Courtesy Accidentals](#courtesy-accidentals)
  - [House Style Settings](#house-style-settings)
  - [Performance Architecture](#performance-architecture)
  - [Correctness Invariants](#correctness-invariants)
- [Integration with Key Signatures](#integration-with-key-signatures)
- [Instrument-Level Scope](#instrument-level-scope)
- [Architectural Implications](#architectural-implications)
- [Consequences](#consequences)
- [References](#references)

## Status
Accepted

## Context

In music notation, the decision of when to print an accidental is surprisingly complex. It's not simply "show an accidental when the note differs from the key signature." Accidentals have **temporal memory** within measures—once an accidental is used, it affects subsequent notes on the same staff position until the barline. This memory system, called "remembered alterations," is fundamental to readable music notation.

Historical engraving practice established several principles:
- Accidentals apply within the measure in which they appear
- Memory resets at barlines (with exceptions for tied notes)
- Courtesy (cautionary) accidentals clarify ambiguous situations
- House styles vary in how strictly these rules are applied

Modern notation adds further complexity:
- Mid-measure key signature changes
- Polytonal and atonal music (Berg, Schoenberg, Webern)
- Contemporary notation with extensive accidentals
- Cross-octave interactions
- Grace notes and their participation in the memory system

### Problem Statement

Accidental rendering depends on the state of previously encountered pitches. The rules for remembered alterations apply to the musical timeline, not to visual layout. In an instrument with multiple staves and voices, the sequence in which pitches appear rhythmically is not the same as the sequence implied by staff order or voice structure.

The core problem is therefore:

* to define a representation of "remembered accidental state" at each temporal point,
* to determine how that state evolves when a new pitch is encountered, and
* to apply these rules consistently across all staves and voices of an instrument, independent of layout.

This requires a model in which the musical time-order of pitches is explicit, and in which all pitches belonging to the same instrument contribute to a single accidental-memory state.

Ooloi's key signature system ([ADR-0034](0034-Key-Signature-Architecture.md)) provides the **baseline** of which accidentals are "normal" for a given context. The remembered alterations system determines when that baseline has been **overridden** by explicit accidentals in the music, and when those overrides should be made explicit in the notation.

## Decision

### Core Algorithm

The remembered alterations system uses a **wave pattern** where accidental memory flows left-to-right through the music in temporal order:

1. **Start of measure**: Initialize remembered state (setting-dependent)
2. **Process each note**: Compare against current remembered state
3. **Update after each note**: Accidental becomes new remembered state
4. **Measure boundary**: Reset or carry forward (setting-dependent)

**The central decision**: A note requires a printed accidental when its accidental differs from the **current remembered state** for that letter/octave combination.

**Key insight**: This single rule handles both required accidentals (contradicting key signature) and courtesy accidentals (restating key signature after alteration). The complexity lies in what constitutes the "current remembered state."

### Data Structure

Remembered alterations use an optimized structure that minimizes memory usage while enabling efficient lookups:

```clojure
{:default {letter accidental}  ; Baseline from key signature (non-naturals only)
 octave {letter accidental}}   ; Per-octave deviations from default
```

**Example** (C# major with 7 sharps, initially):
```clojure
{:default {"C" :sharp, "D" :sharp, "E" :sharp, "F" :sharp,
           "G" :sharp, "A" :sharp, "B" :sharp}}
```

**After C natural appears in octave 4** (deviation from default):
```clojure
{:default {"C" :sharp, "D" :sharp, "E" :sharp, "F" :sharp,
           "G" :sharp, "A" :sharp, "B" :sharp}
 4 {"C" :natural}}  ; Only the deviation stored
```

**Why this structure:**

1. **Memory efficient**: Default stores 7 entries for key signature; octave deviations only store differences (vs 70 entries for all octaves × all letters)
2. **Starts from key signature baseline**: `:default` key contains non-natural accidentals from key signature
3. **Updates incrementally**: Deviations stored per octave, matching default removes deviation
4. **Octave separation**: Enables different house styles for octave-specific behavior
5. **Absence means natural**: If letter not found in octave or default, assume `:natural`

**Lookup algorithm:**
```clojure
(or (get-in remembered [octave letter])    ; Check octave-specific first
    (get-in remembered [:default letter])  ; Fall back to default
    :natural)                               ; Assume natural if absent
```

**Update algorithm:**
```clojure
(let [default-acc (get-in remembered [:default letter] :natural)]
  (if (= accidental default-acc)
    (update remembered octave dissoc letter)         ; Matches default, remove deviation
    (assoc-in remembered [octave letter] accidental))) ; Store deviation
```

**Octave separation rationale:**
- Some house styles apply accidentals across all octaves
- Others treat each octave independently
- The structure supports both without code changes
- Helper predicates (`altered-in-other-octaves?`) can efficiently check only octaves with explicit deviations
- Cross-octave searches are faster: only check keys present in map, not all 10 octaves

### Measure Boundary Behavior

At measure boundaries, the system must decide what state to carry forward. Two primary behaviors exist:

**Standard (most traditions):**
```clojure
initial-remembered = prev-measure-final
```
The complete final state from the previous measure (baseline + all alterations) becomes the starting point for the new measure. Tied notes naturally carry their accidentals forward.

**French style:**
```clojure
initial-remembered = baseline-from-key
```
Reset completely to the key signature baseline. Tied notes must have their accidental restated at the barline.

**Unified function:**
```clojure
(defn initial-remembered-for-measure
  [key-sig prev-final french-ties?]
  (if french-ties?
    (baseline-from-key key-sig)
    prev-final))
```

**Critical insight**: Regardless of tie style, the **previous measure's final state** must always be available for courtesy accidental detection. In French style, we reset the remembered state but still compare against the previous measure when deciding whether to show courtesy accidentals.

**Example: Tied notes across multiple measures**

C# major, C natural tied through measures 2, 3, 4, then C# in measure 5:

- Measure 1 final: `{:default {"C" :sharp, "D" :sharp, ...}, 4 {"C" :natural}}`
- Measure 2 initial: `prev-final` (C natural deviation preserved)
- Measure 2 final: `{:default {"C" :sharp, "D" :sharp, ...}, 4 {"C" :natural}}` (tie continues)
- Measures 3, 4: Same pattern, C natural deviation flows through
- Measure 5 initial: `{:default {"C" :sharp, "D" :sharp, ...}, 4 {"C" :natural}}`
- C# appears in octave 4: differs from remembered (:natural) → print sharp
- Measure 5 final: `{:default {"C" :sharp, "D" :sharp, ...}}` (C now matches default, deviation removed)

The wave carries correctly regardless of how many measures the tie spans.

### Courtesy Accidentals

Courtesy (cautionary) accidentals occur when a note matches the key signature baseline but differs from what was recently altered. These appear **throughout measures**, not just at barlines.

**Within-measure courtesy accidental** (C# major):
- F# appears (matches key signature)
- F natural appears (contradicts key → prints natural, updates remembered)
- F# appears (matches key signature BUT differs from remembered :natural → prints sharp)

That final F# is a courtesy accidental—technically correct by the key signature, but shown for clarity after the F natural.

**Cross-measure courtesy accidental**: When a note in measure N+1 matches the key signature but differs from the final remembered state in measure N, it may require a courtesy accidental depending on house style settings.

**Cross-octave courtesy accidental** (house style dependent): When an alteration appears in one octave (e.g., C#3), subsequent notes on the same letter in other octaves (e.g., C4 natural) may show courtesy accidentals, optionally parenthesized.

### House Style Settings

House styles vary significantly across publishers, historical periods, and musical genres. The remembered alterations algorithm provides the framework for determining when accidentals are semantically required, but presentation details vary:

- **Measure boundary behavior**: Some traditions (French style) restate accidentals at barlines even for tied notes; others carry the remembered state across
- **Courtesy accidentals**: Whether to show accidentals that match the key signature after recent contradictions
- **Cross-octave behavior**: Whether alterations in one octave affect other octaves, and how to indicate this visually
- **Grace note participation**: Whether grace notes update the remembered alterations state

The algorithm separates the core decision logic from these stylistic variations, enabling the same codebase to handle Baroque figured bass, Viennese atonal music, and contemporary microtonal scores with different house style configurations.

### Performance Architecture

**Solution: Two paths based on voice count**

**Single voice (≤1 voices): Direct transduce (zero allocation)**

```clojure
;; When instrument has single voice or no voices, timewalk yields in temporal order
;; Use transduce directly - no collection, no sorting needed
(transduce
  (comp
    (timewalk {:boundary-vpd instrument-vpd
               :start-measure measure-index
               :end-measure measure-index})
    (filter pitch??))

  (completing
    (fn [[remembered decisions] tuple]
      (let [[new-remembered decision] (make-accidental-decision
                                        remembered tuple piece baseline)]
        [new-remembered (cond-> decisions decision (conj decision))])))

  [initial-remembered []]
  [piece])
```

**Multiple voices: Collect, sort, reduce**

```clojure
;; When instrument has multiple voices, timewalk yields voice-by-voice
;; Must collect all pitches, sort by position, then reduce
(let [all-pitch-tuples (sequence
                         (comp
                           (timewalk {:boundary-vpd instrument-vpd
                                      :start-measure measure-index
                                      :end-measure measure-index})
                           (filter pitch??))
                         [piece])
      sorted-tuples (sort-by position all-pitch-tuples)]

  (reduce
    (fn [[remembered decisions] tuple]
      (let [[new-remembered decision] (make-accidental-decision
                                        remembered tuple piece baseline)]
        [new-remembered (cond-> decisions decision (conj decision))]))
    [initial-remembered []]
    sorted-tuples))
```

**Complete function with branching:**

```clojure
(defn- process-pitch-tuple
  "Reducing function that processes a single pitch tuple.
   Updates remembered state and accumulates decisions."
  [piece baseline]
  (fn [[remembered decisions] tuple]
    (let [[new-remembered decision] (make-accidental-decision
                                      remembered tuple piece baseline)]
      [new-remembered (cond-> decisions decision (conj decision))])))

(defn- can-transduce?
  "Check if we can use transduce path (0 or 1 voice) vs collect/sort/reduce path (2+ voices).
   Short-circuits after finding second voice.

   Returns: true for 0-1 voices (transduce path), false for 2+ voices (reduce path)"
  [piece instrument-vpd]
  (transduce
    (comp (timewalk {:boundary-vpd instrument-vpd})
          (filter voice??))
    (fn
      ([] 0)
      ([n] (<= n 1))
      ([n _] (if (>= n 2) (reduced false) (inc n))))
    [piece]))

(defn accidental-decisions-for-measure
  [piece measure-index instrument-vpd key-sig prev-final]
  (let [baseline (baseline-from-key key-sig)
        initial-remembered (if (french-ties? piece)
                            baseline
                            prev-final)
        reducer (process-pitch-tuple piece baseline)]

    (if (can-transduce? piece instrument-vpd)
      ;; Single voice: transduce directly (zero allocation)
      (transduce
        (comp
          (timewalk {:boundary-vpd instrument-vpd
                     :start-measure measure-index
                     :end-measure measure-index})
          (filter pitch??))
        (completing reducer)
        [initial-remembered []]
        [piece])

      ;; Multiple voices: collect, sort, reduce
      (let [all-pitch-tuples (sequence
                               (comp
                                 (timewalk {:boundary-vpd instrument-vpd
                                            :start-measure measure-index
                                            :end-measure measure-index})
                                 (filter pitch??))
                               [piece])
            sorted-tuples (sort-by position all-pitch-tuples)]
        (reduce reducer [initial-remembered []] sorted-tuples)))))
```

**Tuple preservation:**

Each decision preserves the complete `[item vpd position]` tuple from timewalk:
- Item reference provides the actual pitch object
- VPD uniquely identifies each pitch in the piece hierarchy
- Position enables sorting for temporal ordering

**State threading:**

State flows naturally through sequential measure processing. Each measure receives the previous measure's final state, processes it, and returns the new final state for the next measure.

```clojure
;; Process measures sequentially, threading state
(reduce
  (fn [prev-final measure-index]
    (let [[final-alts decisions]
          (accidental-decisions-for-measure
            piece measure-index instrument-vpd key-sig prev-final)]
      ;; Apply decisions to rendering pipeline
      (apply-accidental-decisions piece decisions)
      ;; Return final state for next measure
      final-alts))
  (baseline-from-key key-sig)  ; Initial state
  (range start-measure end-measure))
```

### Correctness Invariants

The remembered-alterations system maintains the following invariants:

1. **Deterministic evaluation**
   For a given piece state, key-signature baseline, and measure-initial remembered state, the algorithm produces the same accidental decisions regardless of layout, staff assignment, or rendering order.

2. **Position-based ordering**
   All pitches belonging to an instrument are evaluated in strictly increasing temporal order determined by their rhythmic position, independent of voice or staff structure.

3. **Single comparison rule**
   A printed accidental is required exactly when the pitch's accidental differs from the current remembered state for its letter and octave.

4. **Instrument-level memory**
   Remembered state is shared across all staves and voices of an instrument. Alterations from any staff or voice contribute to a single state threaded through the instrument's timeline.

5. **Baseline + deviation model**
   The remembered state at any point is the combination of the key-signature baseline and any per-octave deviations accumulated earlier in the measure. Deviations always override baseline; removal of deviation reverts to baseline.

6. **Measure-boundary rules**
   At measure boundaries, the initial remembered state is either the previous measure's final state or the key-signature baseline, depending on house-style settings. Courtesy-accidental detection always has access to the previous measure's final state.

These invariants hold for any number of staves, any number of voices, and any accidental vocabulary supported by the key-signature system.

## Integration with Key Signatures

Key signatures ([ADR-0034](0034-Key-Signature-Architecture.md)) provide the **baseline** for remembered alterations:

```clojure
(defn baseline-from-key [key-sig]
  ;; Convert key signature to {octave {letter accidental}} structure
  ;; For simple key signatures: expand to all octaves
  ;; For per-octave key signatures: use as-is
  )
```

**Key signature changes mid-measure:**

When a key signature changes at position `[measure beat]`:
1. Current remembered state continues until the change point
2. At change point, new baseline takes effect
3. Remembered state merges: new baseline + current alterations
4. Subsequent notes compare against merged state

**Example** (G major → E minor mid-measure):
- Initial: `{:default {"F" :sharp}}`
- F natural appears in octave 4 before change → updates remembered to `{:default {"F" :sharp}, 4 {"F" :natural}}`
- Key changes to E minor (same signature: F#) → baseline remains `{:default {"F" :sharp}}`
- F# appears in octave 4 after change → compares against remembered (:natural) → prints sharp
- After F# processed: `{:default {"F" :sharp}}` (deviation removed, matches default again)

## Instrument-Level Scope

Remembered alterations operate at the **instrument level**, encompassing all staves and voices of that instrument. All pitches contribute to the shared remembered state based on **rhythmic position** (horizontal placement).

**Rationale**: A performer reading an instrument's notation sees all staves and voices simultaneously. If the piano's right hand has F# at position 1/4 and the left hand has F natural at position 1/2 (on a different staff), the F natural affects the visual memory for subsequent F notes in either staff.

**Multi-staff instruments**: For piano, harp, organ, and choir (SATB on grand staff), all staves share accidental memory. Cross-staff notation works naturally because the entire instrument is scanned together.

**System-level separation**: Instruments in the same system do NOT share accidental memory. An oboe's F# does not affect a flute's F natural, even though they appear visually proximate on the page. The boundary is strictly at instrument level - accidental memory is an instrument property, not a system property or score property.

**Example** (piano cross-staff, G major):
```clojure
;; Piano (two staves):
;; Right hand (treble staff): F# at position 0
;; Left hand (bass staff): F natural at position 1/4
;; Right hand: F at position 1/2

;; After sorting by position: F#(0), F-natural(1/4), F(1/2)

;; Result:
;; Position 0: F# (treble) matches key → no print
;; Position 1/4: F natural (bass) contradicts remembered → print natural
;; Position 1/2: F (treble) contradicts remembered (:natural) → print natural (courtesy)
```

The bass F natural at position 1/4 forces a courtesy natural on the treble F at position 1/2, demonstrating cross-staff memory sharing.


## Architectural Implications

The remembered alterations decisions represent the **musical requirement** for accidentals based on key signature and temporal context. They apply universally to all layouts of this piece. Individual layouts may override these decisions for visual reasons, but the remembered alterations algorithm provides the semantic baseline.

The system produces decisions containing:
- Which pitch requires consideration
- Whether an accidental should be printed
- Whether it's a courtesy accidental
- Whether it should be parenthesized

This separation of semantic decisions from visual presentation enables consistent musical interpretation across all visual representations.

## Consequences

**Positive:**

1. **Unified algorithm** - Single decision rule handles required and courtesy accidentals
2. **House style flexibility** - Settings control behavior without code changes
3. **Multi-voice temporal ordering** - Explicit sorting ensures correct left-to-right processing across all voices
4. **Tuple preservation** - Complete `[item vpd position]` tuples identify each pitch decision
5. **Performance optimized** - O(n log n) complexity acceptable for typical measure sizes
6. **Key signature integration** - Seamless interaction with standard, keyless, custom, per-octave, and microtonal key signatures
7. **French style supported** - Tie behavior configurable
8. **Cross-octave awareness** - Structure enables octave-specific house styles
9. **Musical semantics** - Decisions represent musical requirements, independent of layout choices
10. **State threading** - Pure functional approach with explicit data flow

**Neutral:**

1. **Settings complexity** - Multiple settings needed for different engraving traditions
2. **Sequential dependency** - Each measure depends on previous measure's final state (inherent in the problem domain)
3. **Instrument-level scope** - Scanning all staves of multi-staff instruments (e.g., piano, organ, choir) increases the number of pitches to sort, though still negligible for typical measures

## References

- [ADR-0026: Pitch Representation and Operations](0026-Pitch-Representation-and-Operations.md) (string-based pitch format, memoized `convert` function for parsing)
- [ADR-0034: Key Signature Architecture](0034-Key-Signature-Architecture.md) (provides baseline)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md) (transduce pattern)
- [ADR-0014: Timewalk](0014-Timewalk.md) (temporal traversal)
