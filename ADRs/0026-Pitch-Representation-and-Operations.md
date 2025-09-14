# ADR-0026: Pitch Representation and Operations

**Decision:** Adopt string-based pitch representation (`"C#4+25"`) as the canonical format with a factory-based transposer system supporting **three lanes**: interval string, fluid keywords, and chromatic. Internally, diatonic transposition uses a direction-aware `(D,A,C)` model that preserves spelling exactly and supports arbitrary multi-accidentals.

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Primary Operations](#primary-operations)
  - [Transposer Factory (Three Lanes)](#transposer-factory-three-lanes)
  - [Diatonic & Chromatic Semantics](#diatonic--chromatic-semantics)
- [Rationale](#rationale)
  - [String Representation Benefits](#string-representation-benefits)
  - [Why a Factory](#why-a-factory)
- [Detailed Design](#detailed-design)
  - [Pitch String Format & Normalization](#pitch-string-format--normalization)
  - [Transposition Lanes & Grammars](#transposition-lanes--grammars)
  - [Musical Examples](#musical-examples)
  - [Error Handling](#error-handling)
- [Challenges Addressed](#challenges-addressed)
- [Caching Strategy](#caching-strategy)
- [Instrument Integration](#instrument-integration)
- [Performance](#performance)
- [Outcome](#outcome)

---

## Context

Ooloi needs pitch operations that are fast, notation-correct, and microtonal-aware:

1. **Playback accuracy**: Precise frequency (Hz) and MIDI calculations (incl. cents).
2. **Notation correctness**: Spelling preserved under diatonic transposition (double/triple/quadruple/etc accidentals allowed), microtonal cent offsets stable.

**Alternatives considered**
- **Compound pitch objects** (e.g., Igor Engraver): semantically clear, but higher memory/CPU overhead in large scores.
- **String canonical representation** (Ooloi): compact, human-readable, cache-friendly, integrates naturally with parsers/serializers.

---

## Decision

Implement `ooloi.shared.ops.pitches` with a canonical **string** pitch format and a **transposer factory** that exposes three input lanes.

### Primary Operations

- **Normalization** → `{:pitch <sharp-canonical>, :octave <int>, :cent-offset <-99..99>}`
- **Conversion** → `:midi` + `:hz` (memoized)
- **Sorting** → by frequency, microtonal-safe
- **Enharmonic comparison** → `enharmonically-equal?`
- **Transposition** → factory returns `(fn [pitch-str] -> pitch-str)`

### Transposer Factory (Three Lanes)

1) **Interval string lane (primary)** — case-sensitive quality  
```clojure
(make-transposer :interval "<Q><N>[+|-]" [:cents C])
;; <Q> ∈ P | M | m | A+ | d+   (one or more A/d allowed)
;; <N> ∈ integer ≥ 1           (1..7 simple; 8=9th; 9=10th; unbounded)
;; [+|-] optional direction    (+ = up; − = down; default up)
````

2. **Fluid keyword lane (order-agnostic; only \:cents takes a number)**

```clojure
(make-transposer [:up|:down] <quality/base/modifiers> <ordinal> [:octave ...] [:cents C])
;; Direction: :up | :down   (default :up)
;; Ordinal (exactly one): :unison | :second | :third | :fourth | :fifth | :sixth | :seventh | :octave | :ninth
;; Extra octaves: add :octave tokens (each adds 7 letter-steps); :ninth ≡ :second + one :octave
;; Quality: base (0 or 1) :perfect | :major | :minor  + modifiers (repeatable) :augmented / :diminished
;; Invalid: :minor with 1/4/5; :perfect with 2/3/6/7
```

3. **Chromatic lane**

```clojure
(make-transposer :chromatic S [:cents C])   ;; S = signed semitones
```

### Diatonic & Chromatic Semantics

**Internal diatonic model:** `(D,A,C)`

* `D` = signed **letter steps** (0=unison, 1=2nd, …; +7 per compound octave). Sign is direction.
* `A` = signed **semitone tweak** relative to Major/Perfect base magnitude for `D`.
  **Rule:** `+1` always **raises** final pitch height, `−1` always **lowers**, regardless of direction.

  * Perfect-type degrees: 1/4/5; Major-type: 2/3/6/7.
* `C` = signed **cents** applied after semitone shift.

**Chromatic:** `S` = signed semitones; spelling uses sharp-canonical normalization.

**Invertibility:** inverse of `(D,A,C)` is `(-D,-A,-C)`.

---

## Rationale

### String Representation Benefits

* **Efficient & compact** for large scores.
* **Readable** for musicians (`"Bb4-15"` is self-explanatory).
* **Serializer/parsers friendly**; microtonal cent offsets embedded.

### Why a Factory

* **Higher-order closures** capture configuration once; application cost is minimal.
* **Composability** with `comp`, threading, etc.
* **Separation of concerns:** interval-string lane (UI-friendly), fluid lane (dialog-friendly), and chromatic lane (playback/MIDI).

---

## Detailed Design

### Pitch String Format & Normalization

`[A-G][accidentals][octave][±cents]`

Examples: `"C4"`, `"F#3"`, `"Ebb5"`, `"G4+25"`, `"A3-50"`.
Normalization constrains cent-offset to `[-99..99]` and yields sharp-canonical pitch class.

### Transposition Lanes & Grammars

* **Interval tokens:** `"M2+"`, `"m3-"`, `"A5+"`, `"d4-"`, `"AA1+"`, `"M9+"`, …
  Case-sensitive quality; optional trailing direction.

* **Fluid keywords:**
  `:up | :down` + base quality `:perfect | :major | :minor` + ordinal
  `:unison|:second|:third|:fourth|:fifth|:sixth|:seventh|:octave|:ninth`
  with repeatable `:octave` and modifiers `:augmented` / `:diminished`.
  Example equivalents:

  ```clojure
  (make-transposer :interval "M2+")        ; same as:
  (make-transposer :up :major :second)
  ```

* **Chromatic:** `(make-transposer :chromatic 3)` ⇒ up 3 semitones.

### Musical Examples

**Common Musical Intervals:**

```clojure
;; Every musician intuitively understands these
(def up-octave (make-transposer :up :octave))
(up-octave "C4")  ;=> "C5"

(def down-fifth (make-transposer :down :perfect :fifth))
(down-fifth "G4") ;=> "C4"

(def major-third-up (make-transposer :up :major :third))
(major-third-up "C4") ;=> "E4"

(def major-sixth-down (make-transposer :down :major :sixth))
(major-sixth-down "A4") ;=> "C4"
```

**Real-World Transposition Scenarios:**

```clojure
;; Transpose melody from C major to F major
(def to-f-major (make-transposer :up :perfect :fourth))
(map to-f-major ["C4" "D4" "E4" "F4" "G4"])
;=> ["F4" "G4" "A4" "Bb4" "C5"]

;; Move bass line down an octave
(def bass-register (make-transposer :down :octave))
(map bass-register ["G3" "F3" "E3" "D3"])
;=> ["G2" "F2" "E2" "D2"]

;; Jazz harmony - chromatic approach notes
(def chromatic-up (make-transposer :chromatic 1))
(chromatic-up "B3") ;=> "C4"  ; leading tone resolution

;; Microtonal fine-tuning
(def slightly-flat (make-transposer :up :perfect :unison :cents -10))
(slightly-flat "A4") ;=> "A4-10"
```

**Advanced Musical Applications:**

```clojure
;; Modal interchange - borrow from parallel modes
(def flatten-third (make-transposer :down :augmented :unison))  ; semitone down
(flatten-third "E4") ;=> "Eb4"  ; major → minor third

;; Compound intervals with threading
(let [up-m7 (make-transposer :up :major :seventh)
      up-m2 (make-transposer :up :major :second)]
  (->> "C4"
       up-m7      ; up M7
       up-m2))    ; then up M2
;=> "D5"  ; net result: major ninth up

;; Efficient bulk transposition with transducers
(into []
      (map (make-transposer :up :perfect :fourth))
      ["C4" "D4" "E4" "F4" "G4"])
;=> ["F4" "G4" "A4" "Bb4" "C5"]  ; to F major
```

**Three Lanes for Different Contexts:**

```clojure
;; Interval strings - compact, precise
(def tritone-up (make-transposer :interval "A4+"))
(def diminished-seventh-down (make-transposer :interval "d7-"))

;; Fluid keywords - crystal clear for UI/dialogue
(def augmented-sixth-up (make-transposer :up :augmented :sixth))
(def minor-tenth-down (make-transposer :down :minor :third :octave))

;; Chromatic - direct for MIDI/playback
(def whole-step-up (make-transposer :chromatic 2))
(def quarter-tone-sharp (make-transposer :chromatic 0 :cents 50))
```

### Error Handling

* **Conflicts** (both `:up` and `:down`) → `ExceptionInfo`.
* **Invalid quality/degree pairs** (`:minor :fourth`, `:perfect :third`) → `ExceptionInfo`.
* **Lane mixing** (`:interval` together with fluid keywords) → `ExceptionInfo`.
* **Type errors** (`:cents` non-integer, chromatic semitones non-integer) → `ExceptionInfo`.

---

## Challenges Addressed

* **Diatonic spelling preservation:** letter-first logic with `(D,A,C)` yields exact spellings (double/triple/quadruple/etc accidentals allowed); **no enharmonic simplification**.
* **Microtonal stability:** cents are applied after the semitone computation and preserved predictably.
* **Round-trip guarantees:**

  * Diatonic: exact spelling round-trip (up then down with inverse `(D,A,C)` returns identical string).
  * Chromatic: round-trip is **enharmonically equal** (canonical sharp spelling each hop).

---

## Caching Strategy

**Multi-layer caches:**

1. **Primitive hot caches** (memoized): pitch decomposition and cents→pitch conversion.
2. **Factory cache** keyed by the **raw arg vector** (not canonicalized):

   * Interval lane: exact token string (case-sensitive).
   * Fluid lane: exact keyword sequence (order preserved).
   * Chromatic lane: `[:chromatic S :cents C]`.
     This avoids parse-normalize overhead on cache hits.
3. **Global output caches** for zero-work application:

   * **Diatonic:** key = `[D A C <pitch-str>]`
   * **Chromatic:** key = `[S C <pitch-str>]`
     Any transposer closure benefits; repeated inputs return instantly.
4. **Per-closure output memo** (pitch→pitch) still present and tunable.

`clear-transposition-caches!` clears factory, global, and primitive caches.

---

## Instrument Integration

Instrument definitions can use **interval tokens** or the **fluid lane**:

```clojure
{:name "B♭ Clarinet"
 :sounding->written [:interval "M2+"]
 :written->sounding [:interval "M2-"]}

{:name "A Clarinet"
 :sounding->written [:interval "m3+"]
 :written->sounding [:interval "m3-"]}

{:name "Horn in F"
 :sounding->written [:up :perfect :fifth]
 :written->sounding [:down :perfect :fifth]}

{:name "Bass Clarinet in B♭"          ; sounding -> written: M9 up (M2 + octave)
 :sounding->written [:up :major :ninth]
 :written->sounding [:down :major :ninth]}

{:name "Quarter-tone Trumpet"
 :sounding->written [:chromatic 6 :cents 50]    ; up tritone + 50c
 :written->sounding [:chromatic -6 :cents -50]}
```

These vectors are applied as `(apply make-transposer params)`.

---

## Performance

On a **2017 MacBook Pro** (2.2 GHz 6-core Intel):

* **1,000,000 transpositions in 5637.002775 ms**

  * ≈ **177,500 ops/sec**
  * ≈ **5.64 µs per transposition**

This includes the global `(D,A,C,pitch)` / `(S,C,pitch)` caches and hot-path memoization. Throughput is sufficient for interactive editing, part extraction, and large batch processes.

---

## Outcome

* **Minimal, expressive API:** string pitch format + single factory with three intuitive lanes.
* **Musical correctness:** exact diatonic spelling, microtonal stability, no unwanted simplification.
* **Scalable performance:** layered caching delivers sub-10 µs ops on older hardware.
* **Extensible architecture:** supports extreme accidentals, compound intervals (e.g., `"M13+"`), and flexible UI (interval tokens or fluid dialogs).

This ADR cements pitch operations as a stable, high-performance foundation for Ooloi’s higher-level notation and rendering features.
