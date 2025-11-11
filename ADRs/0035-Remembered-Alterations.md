# ADR-0035: Remembered Alterations

## Table of Contents
- [Status](#status)
- [Context](#context)
- [Decision](#decision)
  - [Core Algorithm](#core-algorithm)
  - [Data Structure](#data-structure)
  - [Measure Boundary Behavior](#measure-boundary-behavior)
  - [Courtesy Accidentals](#courtesy-accidentals)
  - [House Style Settings](#house-style-settings)
  - [Performance Architecture](#performance-architecture)
- [Integration with Key Signatures](#integration-with-key-signatures)
- [Multi-Voice Considerations](#multi-voice-considerations)
- [Implementation Functions](#implementation-functions)
- [Consequences](#consequences)
- [References](#references)

## Status
Proposed

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

Remembered alterations use the structure:

```clojure
{octave {letter accidental}}
```

**Example** (C# major with 7 sharps, after C natural appears in octave 4):
```clojure
{4 {"C" :natural, "D" :sharp, "E" :sharp, "F" :sharp,
    "G" :sharp, "A" :sharp, "B" :sharp}}
```

**Why this structure:**

1. **Starts from key signature baseline**: All letters begin with their key signature accidentals
2. **Updates incrementally**: As notes appear, their alterations replace baseline values
3. **Octave separation**: Enables different house styles for octave-specific behavior
4. **Complete state**: Contains all letters (A-G), not just alterations

**Octave separation rationale:**
- Some house styles apply accidentals across all octaves
- Others treat each octave independently
- The structure supports both without code changes
- Helper predicates (`altered-in-same-octave?`, `altered-in-other-octaves?`) enable house style logic

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

- Measure 1 final: `{4 {"C" :natural, "D" :sharp, ...}}`
- Measure 2 initial: `prev-final` (C natural preserved)
- Measure 2 final: `{4 {"C" :natural, "D" :sharp, ...}}` (tie continues)
- Measures 3, 4: Same pattern, C natural flows through
- Measure 5 initial: `{4 {"C" :natural, "D" :sharp, ...}}`
- C# appears: differs from remembered (:natural) → print sharp

The wave carries correctly regardless of how many measures the tie spans.

### Courtesy Accidentals

Courtesy (cautionary) accidentals occur when a note matches the key signature baseline but differs from what was recently altered. These appear **throughout measures**, not just at barlines.

**Within-measure courtesy accidental** (C# major):
```clojure
;; F# appears (matches key signature)
;; F natural appears (contradicts key → prints natural, updates remembered)
;; F# appears (matches key signature BUT differs from remembered :natural → prints sharp)
```

That final F# is a courtesy accidental—technically correct by the key signature, but shown for clarity after the F natural.

**Cross-measure courtesy accidental**:
```clojure
;; Measure 1: C natural appears
;; Measure 2: C# appears
;; In C# major, C# matches key signature but differs from measure 1 → prints sharp
```

**Cross-octave courtesy accidental** (house style dependent):
```clojure
;; C#3 appears (updates remembered for octave 3)
;; C4 (natural) appears (differs from octave 3 alteration → may print natural with parentheses)
```

**Decision logic**:
```clojure
(or
  ;; Required: differs from current remembered state
  (not= accidental (get-in remembered [octave letter]))

  ;; Courtesy: matches baseline but differs from previous measure
  (and courtesy-setting?
       (= accidental (get-in baseline [octave letter]))
       (not= accidental (get-in prev-final [octave letter])))

  ;; Cross-octave courtesy: altered in other octaves
  (and cross-octave-courtesy?
       (altered-in-other-octaves? remembered octave letter accidental)))
```

### House Style Settings

Multiple settings control accidental presentation:

| Setting | Effect |
|---------|--------|
| `:french-ties?` | Tied notes restate accidentals at barlines |
| `:courtesy-accidentals?` | Show courtesy accidentals at measure boundaries |
| `:parenthesize-courtesy?` | Use parentheses for courtesy accidentals |
| `:courtesy-across-octaves?` | Show courtesy when different octave was altered |
| `:grace-notes-participate?` | Grace notes update remembered alterations |
| `:repeated-notes-style` | `:all`, `:except-repeated`, or `:necessary-only` (Berg Wozzeck example) |

**Rationale**: House styles vary significantly across publishers, historical periods, and musical genres. The algorithm provides the framework; settings control the specific behavior. This separation enables the same codebase to handle Baroque figured bass, Viennese atonal music, and contemporary microtonal scores.

### Performance Architecture

The algorithm processes every pitch in every measure during rendering. Performance is critical.

**Multi-voice temporal ordering requirement:**

Timewalk processes voices sequentially, with position counting restarting at 0 for each voice. For remembered alterations, we need true temporal ordering across all voices (left-to-right reading order).

**Solution: Collect, sort, reduce**

```clojure
(defn accidental-decisions-for-measure
  [piece measure-index staff-vpd key-sig prev-final house-settings]
  (let [baseline (baseline-from-key key-sig)
        initial-remembered (if (:french-ties? house-settings)
                            baseline
                            prev-final)

        ;; Collect all pitch tuples from measure (all voices)
        all-pitch-tuples (sequence
                           (comp
                             (timewalk {:boundary-vpd staff-vpd
                                        :start-measure measure-index
                                        :end-measure measure-index})
                             (filter pitch??))
                           [piece])

        ;; Sort by position for temporal order across voices
        sorted-tuples (sort-by position all-pitch-tuples)]

    ;; Process sorted tuples with reduce
    (reduce
      (fn [[remembered decisions] tuple]
        (let [parsed (parse-pitch (:note (item tuple)))
              decision (make-accidental-decision
                         parsed remembered prev-final baseline house-settings)
              new-remembered (update remembered
                                     (:octave parsed)
                                     assoc
                                     (:letter parsed)
                                     (:accidental parsed))
              ;; Preserve complete tuple for layout storage
              new-decision (assoc decision :tuple tuple)]
          [new-remembered (conj decisions new-decision)]))
      [initial-remembered []]
      sorted-tuples)))
```

**Tuple preservation:**

Each decision preserves the complete `[item vpd position]` tuple from timewalk because:
- **VPD for identification** — uniquely identifies each pitch in the piece hierarchy
- **Item reference** — the actual pitch object with its musical properties
- **Position** — temporal position used for sorting, useful for debugging

**Performance characteristics:**
- O(n) complexity for collection
- O(n log n) for sorting by position
- O(n) for reduce processing
- Overall: O(n log n) dominated by sort
- In practice: n is small (typical measure has 4-20 pitches across all voices)
- No repeated traversals - single collection phase

**Future extensibility:**
The collect-sort-reduce pattern enables future cross-staff remembered alterations (e.g., piano left and right hand sharing accidental memory). Implementation: timewalk at instrument level instead of staff level, sort all pitches from both staves by position, process in unified temporal order. No algorithmic changes required.

**No caching required:**
State flows naturally through sequential measure processing. Each measure receives the previous measure's final state, processes it, and returns the new final state for the next measure. Timewalk's transducer efficiency ensures no unnecessary recomputation - each pitch is visited exactly once per rendering pass.

```clojure
;; Process measures sequentially, threading state
(reduce
  (fn [prev-final measure-index]
    (let [[new-final decisions]
          (accidental-decisions-for-measure
            piece measure-index staff-vpd key-sig prev-final house-settings)]
      ;; Decisions are musical semantics - apply to all layouts
      ;; Layouts can override these decisions as needed
      new-final))
  (baseline-from-key key-sig)  ; Initial state
  (range start-measure end-measure))
```

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
- Initial: `{4 {"F" :sharp, ...}}`
- F natural appears before change → updates remembered
- Key changes to E minor (same signature: F#)
- F# appears after change → compares against remembered (:natural) → prints sharp

## Multi-Voice Considerations

In multi-voice music, all voices contribute to the shared remembered state based on **rhythmic position** (horizontal placement), not voice separation.

**Rationale**: Accidentals affect the staff position visually. A performer reading the staff sees all voices simultaneously. If voice 1 has F# at position 1/4 and voice 2 has F natural at position 1/2, the F natural affects the visual memory for subsequent notes in either voice.

**Critical implementation detail**: Timewalk processes voices sequentially with position counting restarting at 0 for each voice. This means we cannot rely on timewalk's emission order for temporal ordering.

**Solution**: Collect all pitch tuples from the measure, then sort by position to establish true left-to-right reading order across all voices.

**Example** (two voices, G major):
```clojure
;; Voice 1: F# at position 0, C# at position 1/4
;; Voice 2: F natural at position 1/8, C natural at position 1/2

;; After sorting by position: F#(0), F-natural(1/8), C#(1/4), C-natural(1/2)

;; Result:
;; Position 0: F# matches key → no print
;; Position 1/8: F natural contradicts key AND remembered → print natural
;; Position 1/4: C# contradicts key → print sharp
;; Position 1/2: C natural contradicts remembered (C#) → print natural
```

**Timewalk position semantics:**
- Within each voice: positions are correct (0, 1/4, 1/2, 3/4 for quarter notes)
- Across voices: positions restart per voice
- For temporal ordering: collect all tuples, then sort by position field

## Implementation Functions

The remembered alterations system consists of six core functions:

**1. parse-pitch** - Extract `{:letter :accidental :octave}` from pitch string (leverages memoized `convert`)

**2. baseline-from-key** - Convert key signature to `{octave {letter accidental}}` structure

**3. accidental-decisions-for-measure** - Full pipeline with multi-voice temporal ordering, returns `[final-alterations decisions]` where each decision includes complete `[item vpd position]` tuple

**4. make-accidental-decision** - Core logic determining print/courtesy/parenthesized

**5. Helper predicates** - `altered-in-other-octaves?` for cross-octave courtesy detection

Implementation details are specified in issue #123 (Accidental Calculations & Engraving Support).

**Decision structure:**

Each decision returned by `accidental-decisions-for-measure` contains:
```clojure
{:tuple [pitch-obj [:m 4 0 2 3 0 :items 2] 1/4]  ; Complete timewalk tuple
 :print? true                                     ; Should accidental be printed?
 :courtesy? false                                 ; Is this a courtesy accidental?
 :parenthesized? false                            ; Should it be parenthesized?
 :reason :required}                               ; Why: :required, :courtesy-measure, etc.
```

**Musical semantics, not layout:**
These decisions represent the **musical requirement** for accidentals based on key signature and temporal context. They apply universally to all layouts of this piece, regardless of transposition (keys transpose with the music). Individual layouts may override these decisions for visual reasons, but the remembered alterations algorithm provides the semantic baseline.

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
2. **Sorting required** - Must sort by position for temporal ordering (negligible cost for typical measure sizes)
3. **Sequential dependency** - Each measure depends on previous measure's final state (inherent in the problem domain)

**Future Extensions:**

The collect-sort-reduce architecture naturally extends to cross-staff remembered alterations:
- **Piano left/right hand** - Change `boundary-vpd` from staff to instrument level
- **Organ multiple manuals** - Same pattern, instrument-level scope
- **Shared visual context** - Any scope where notation proximity suggests shared accidental memory

No algorithmic changes required - simply adjust the `boundary-vpd` parameter to control scope.

## References

- [ADR-0026: Pitch Representation and Operations](0026-Pitch-Representation-and-Operations.md) (string-based pitch format, memoized `convert` function for parsing)
- [ADR-0034: Key Signature Architecture](0034-Key-Signature-Architecture.md) (provides baseline)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md) (transduce pattern)
- [ADR-0014: Timewalk](0014-Timewalk.md) (temporal traversal)
