# ADR-0026: Pitch Representation and Operations

**Decision:** Adopt string-based pitch representation as the canonical format for Ooloi, with comprehensive operations for conversion, sorting, and transposition.

## Table of Contents

- [Context](#context)
- [Core Decision: String-Based Pitch Representation](#core-decision-string-based-pitch-representation)
- [What the Representation Can Express](#what-the-representation-can-express)
  - [Basic Musical Pitches](#basic-musical-pitches)
  - [Complex Accidentals](#complex-accidentals)
  - [Microtonal Precision](#microtonal-precision)
  - [Extreme Ranges and Edge Cases](#extreme-ranges-and-edge-cases)
  - [Complex Combinations](#complex-combinations)
- [Format Specification](#format-specification)
  - [Grammar](#grammar)
  - [Normalisation Rules](#normalisation-rules)
  - [Examples of Normalisation](#examples-of-normalisation)
- [Basic Operations](#basic-operations)
  - [Conversion to Playback Values](#conversion-to-playback-values)
  - [Frequency-Based Sorting](#frequency-based-sorting)
  - [Enharmonic Comparison](#enharmonic-comparison)
- [Transposition: The Sophisticated Operation](#transposition-the-sophisticated-operation)
  - [Real Musical Scenarios](#real-musical-scenarios)
  - [The Three-Lane System](#the-three-lane-system)
  - [Advanced Musical Examples](#advanced-musical-examples)
  - [Internal Mathematics: The (D,A,C) Model](#internal-mathematics-the-dac-model)
  - [Round-Trip Guarantees](#round-trip-guarantees)
- [Implementation & Performance](#implementation--performance)
  - [Error Handling](#error-handling)
  - [Multi-Layer Caching Strategy](#multi-layer-caching-strategy)
  - [Performance Benchmarks](#performance-benchmarks)
  - [Instrument Integration](#instrument-integration)
- [Outcome](#outcome)
  - [Delivered Capabilities](#delivered-capabilities)
  - [Architectural Benefits](#architectural-benefits)
  - [Foundation for Higher-Level Features](#foundation-for-higher-level-features)

---

## Context

Ooloi requires pitch operations that are fast, notation-correct, and microtonal-aware:

1. **Playback accuracy**: Precise frequency (Hz) calculations including microtonal cent offsets
2. **Notation correctness**: Spelling preserved under diatonic transposition with support for multiple accidentals
3. **Performance**: Efficient operations for large orchestral scores
4. **Integration**: Seamless interaction with parsers, serializers, and caching systems

**Note on audio architecture**: As established in [ADR-0027 Plugin-Based Audio Architecture](0027-Plugin-Based-Audio-Architecture.md), Ooloi core provides no audio or MIDI functionality. All audio processing occurs in frontend plugins. The pitch operations focus on frequency (Hz) calculations for audio rendering.

**Historical note on A4 = 440 Hz standard**: The A4 = 440 Hz reference significantly predates MIDI (1834 Scheibler recommendation, 1955 ISO standardization vs. early 1980s MIDI development). Scientific pitch notation with A4 designation also predates MIDI by decades. MIDI adopted these existing musical conventions rather than establishing them. The A4 reference in frequency calculations reflects fundamental musical mathematics, not MIDI dependency.

**Backend/Frontend API separation**: The `ops.pitches` namespace serves both backend (Hz calculations) and frontend contexts. The `convert` function uses direct cent-to-Hz calculation without any MIDI number intermediary, providing clean backend-focused conversion (pitch string → Hz only). Frontend components that require MIDI number conversion can calculate this from the Hz values when interfacing with MIDI plugins.

This maintains strict frontend-backend separation whilst providing shared pitch representation with no vestigial MIDI calculations.

**Alternatives considered:**
- **Compound pitch objects** (e.g., Igor Engraver style): semantically clear but higher memory/CPU overhead
- **String canonical representation** (Ooloi choice): compact, human-readable, cache-friendly

---

## Core Decision: String-Based Pitch Representation

Use string-based pitch representation as the canonical format throughout Ooloi. All pitch operations accept and return strings, with internal conversion to components only when required for arithmetic.

---

## What the Representation Can Express

### Basic Musical Pitches
```
"C4"     ; Middle C
"A4"     ; Concert A (440 Hz)
"F#3"    ; F sharp, 3rd octave
"Bb5"    ; B flat, 5th octave
```

### Complex Accidentals
```
"C###4"   ; C triple-sharp
"Dbbb3"   ; D triple-flat  
"F####3"  ; F quadruple-sharp
"Abbbb2"  ; A quadruple-flat
```

### Microtonal Precision
```
"A4+25"     ; A4 plus 25 cents (quarter-tone sharp)
"C4-50"     ; C4 minus 50 cents (quarter-tone flat)  
"G4+150"    ; G4 plus 150 cents (1.5 semitones)
"Bb3-75"    ; Bb3 minus 75 cents
```

### Extreme Ranges and Edge Cases
```
"C-1"       ; Sub-sub-contra octave
"G9"        ; Highest MIDI range
"C0"        ; Sub-contra octave (16.35 Hz)
"B11"       ; Beyond MIDI range
```

**Practical Range:**
- **Lower bound**: No hard limit (tested to octave -38)
- **Upper bound**: No hard limit (tested to octave 1033)  
- **MIDI coverage**: C-1 to G9 (MIDI 0-127)
- **Cents**: Arbitrarily large offsets are normalised

### Complex Combinations
```
"A4+1234567"   ; Normalises to "F#1033+67"
"G2-48000"     ; Normalises to "G-38"
"B#3"          ; Normalises to "C4" 
"Cb4"          ; Normalises to "B3"
```

---

## Format Specification

### Grammar
```
pitch ::= [A-G][accidentals][octave][±cents]

accidentals ::= [#]* | [b]*     // Sharps or flats, not mixed
octave      ::= -?\d+           // Any integer (negative allowed)
cents       ::= [+-]\d+         // Optional cent offset
```

### Normalisation Rules

1. **Sharp-canonical form**: All pitches normalise to natural notes with sharps only
2. **Cent constraints**: Cent offsets constrained to -99 to +99 range
3. **Octave adjustment**: Large cent offsets adjust octave accordingly
4. **No mixed accidentals**: Sharps and flats cannot be combined (validation error)

Canonicalisation (sharp-only) is applied in conversion, sorting, and chromatic transposition. Diatonic transposition never simplifies spelling — double/triple accidentals are preserved exactly.

### Examples of Normalisation
```clojure
"Dbb3"     => {:pitch "C", :octave 3, :cent-offset 0}
"C4+150"   => {:pitch "C#", :octave 4, :cent-offset 50}  
"C4-150"   => {:pitch "A#", :octave 3, :cent-offset 50}
"B#3"      => {:pitch "C", :octave 4, :cent-offset 0}
"C4+1200"  => {:pitch "C", :octave 5, :cent-offset 0}
```

---

## Basic Operations

### Conversion to Playback Values

**Essential for audio rendering:**

```clojure
(convert "A4")
=> {:pitch "A", :octave 4, :cent-offset 0, :hz 440.0,
    :original-letter "A", :original-accidentals "",
    :original-octave 4, :original-cent-offset 0}

(convert "A4+50")
=> {:pitch "A", :octave 4, :cent-offset 50, :hz 452.89,
    :original-letter "A", :original-accidentals "",
    :original-octave 4, :original-cent-offset 50}

(convert "Bb3")  ; Normalizes to A# but preserves original B-flat spelling
=> {:pitch "A#", :octave 3, :cent-offset 0, :hz 233.08,
    :original-letter "B", :original-accidentals "b",
    :original-octave 3, :original-cent-offset 0}

(convert "C4+250")  ; Large cent offset normalizes but preserves original
=> {:pitch "D", :octave 4, :cent-offset 50, :hz 293.66,
    :original-letter "C", :original-accidentals "",
    :original-octave 4, :original-cent-offset 250}
```

**Return structure:**

Normalized values (for frequency calculations and enharmonic equivalence):
- `:pitch` - Normalized note name using only sharps (e.g., "C#", "F", "A#")
- `:octave` - Integer octave number (normalized when cents/accidentals overflow)
- `:cent-offset` - Integer cent offset (-99 to +99, normalized from input)
- `:hz` - Frequency in Hertz

Original values (preserving notation intent):
- `:original-letter` - Original input letter ("A" through "G") before normalization
- `:original-accidentals` - Original input accidentals ("#", "##", "b", "bbb", or "") before normalization
- `:original-octave` - Original input octave number before normalization
- `:original-cent-offset` - Original input cent offset before normalization

**Use cases for original values:**

The `original-*` fields preserve the written notation before normalization, essential for musical contexts where the composer's or musician's notation intent matters. Example use cases include:

- **Accidental tracking**: Distinguishing Bb from A# in remembered alteration systems (see [ADR-0035 Remembered Alterations](0035-Remembered-Alterations.md))
- **Score analysis**: Preserving harmonic spelling for theoretical analysis (e.g., augmented sixth vs. minor seventh)
- **Notation display**: Maintaining original enharmonic spelling in UI and debugging
- **Historical accuracy**: Preserving the notation as written in source material

**Design benefits:**
- Results memoized with LRU cache (10,000 entries) for performance
- Direct cent-to-Hz calculation eliminates MIDI number intermediary
- Handles extreme ranges with floating-point precision
- Clean mathematical model: pitch → cents → Hz
- Preserves original notation intent alongside normalized values
- Direct integration with synthesizer and audio systems

### Frequency-Based Sorting

**Accurate ordering including microtones:**

```clojure
(sort-pitches ["C4" "A4" "G4"]) 
=> ["C4" "G4" "A4"]

(sort-pitches ["C4" "C4+10" "C4-10"]) 
=> ["C4-10" "C4" "C4+10"]

(sort-pitches ["A4" "C4"] :descending) 
=> ["A4" "C4"]
```

### Enharmonic Comparison

**Lightweight equivalence checking:**

```clojure
(enharmonically-equal? "C#4" "Db4")  => true
(enharmonically-equal? "B#3" "C4")   => true  
(enharmonically-equal? "F##3" "G3")  => true
(enharmonically-equal? "C4" "D4")    => false
```

**Performance characteristics:**
- Uses normalisation only (no Hz calculation overhead)
- Essential for chord recognition and harmonic analysis

---

## Transposition: The Sophisticated Operation

### Real Musical Scenarios

**B♭ Clarinet (sounds a major second lower):**
```clojure
;; Concert pitch to written pitch (up M2)
(def to-written (make-transposer :interval "M2+"))
(to-written "B3")   => "C#4"
(to-written "Eb4")  => "F4"
(to-written "F#4")  => "G#4"

;; Written pitch to concert pitch (down M2)  
(def to-concert (make-transposer :interval "M2-"))
(to-concert "C#4")  => "B3"
(to-concert "F4")   => "Eb4"
```

**Horn in F (sounds a perfect fifth lower):**
```clojure
;; Concert to written (up P5)
(def horn-to-written (make-transposer :interval "P5+"))
(horn-to-written "C4")  => "G4"
(horn-to-written "F4")  => "C5"

;; Written to concert (down P5)
(def horn-to-concert (make-transposer :interval "P5-"))  
(horn-to-concert "G4") => "C4"
```

**Chromatic transposition for MIDI/playback:**
```clojure
;; Up three semitones (minor third)
(def chrom-up-3 (make-transposer :chromatic 3))
(chrom-up-3 "C4")  => "D#4"   ; Sharp spelling from normalisation

;; Check enharmonic equivalence
(enharmonically-equal? (chrom-up-3 "C4") "Eb4")  => true
```

**Microtonal adjustments:**
```clojure
;; Quarter-tone sharp
(def quarter-up (make-transposer :interval "P1" :cents 50))
(quarter-up "C4")     => "C4+50"
(quarter-up "C4+50")  => "C#4"    ; Rolls over to next semitone

;; Fine tuning for historical temperaments  
(def well-tempered-c# (make-transposer :chromatic 1 :cents -5))
(well-tempered-c# "C4") => "C#4-5"
```

### The Three-Lane System

The transposer factory provides three distinct input methods for different contexts:

#### Lane 1: Interval String (Primary)
**Compact, case-sensitive quality notation:**

```clojure
;; Syntax: "<Quality><Number>[Direction]"
(make-transposer :interval "M2+")    ; Major second up
(make-transposer :interval "m3-")    ; Minor third down  
(make-transposer :interval "P5+")    ; Perfect fifth up
(make-transposer :interval "A4+")    ; Augmented fourth up
(make-transposer :interval "d7-")    ; Diminished seventh down
(make-transposer :interval "AA1+")   ; Double-augmented unison up
(make-transposer :interval "M9+")    ; Major ninth up
```

**Quality codes:**
- `P` = Perfect (unison, 4th, 5th, octave)
- `M` = Major (2nd, 3rd, 6th, 7th)  
- `m` = minor (2nd, 3rd, 6th, 7th)
- `A` = Augmented (repeatable: `AA`, `AAA`, etc.)
- `d` = diminished (repeatable: `dd`, `ddd`, etc.)

#### Lane 2: Fluid Keywords (Natural Language)
**Order-agnostic, dialog-friendly syntax:**

```clojure
;; Same as "M2+"  
(make-transposer :up :major :second)

;; Same as "m6-"
(make-transposer :down :minor :sixth)

;; Compound intervals with extra octaves
(make-transposer :up :major :second :octave)     ; Major ninth  
(make-transposer :down :perfect :fourth :octave :octave)  ; Down 11th

;; Quality modifiers
(make-transposer :up :augmented :fourth)         ; Tritone up
(make-transposer :down :major :diminished :seventh)  ; Diminished major 7th down
```

**Available keywords:**
- **Direction**: `:up`, `:down` (default: `:up`)
- **Ordinals**: `:unison`, `:second`, `:third`, `:fourth`, `:fifth`, `:sixth`, `:seventh`, `:octave`, `:ninth`
- **Base quality**: `:perfect`, `:major`, `:minor`  
- **Modifiers**: `:augmented`, `:diminished` (repeatable)
- **Cents**: `:cents <integer>`

**Validation:**
- Exactly one ordinal must be present
- Base quality must match the degree (1/4/5 → `:perfect`; 2/3/6/7 → `:major` | `:minor`)
- `:augmented` / `:diminished` are additive and repeatable
- Reject lane mixing and duplicate/conflicting directions

#### Lane 3: Chromatic (Semitone-Based)
**Direct semitone transposition with canonical spelling:**

```clojure
(make-transposer :chromatic 3)        ; Up 3 semitones
(make-transposer :chromatic -7)       ; Down 7 semitones  
(make-transposer :chromatic 0 :cents 25)  ; Up 25 cents only
```

### Advanced Musical Examples

**Modal interchange (borrowing from parallel modes):**
```clojure
;; Flatten the third (major → minor)
(def flatten-third (make-transposer :chromatic -1))  ; Down 1 semitone
(flatten-third "E4")  => "D#4"  ; Major third becomes minor third

;; Using fluid syntax for clarity
(def minor-third-down (make-transposer :down :augmented :unison))  
(minor-third-down "E4")  => "D#4"
```

**Compound intervals with threading:**
```clojure
(let [up-m7   (make-transposer :up :major :seventh)
      up-m2   (make-transposer :up :major :second)]
  (->> "C4"
       up-m7      ; C4 → B4 (major 7th up)
       up-m2))    ; B4 → C#5 (major 2nd up)
=> "C#5"         ; Net result: major 9th up
```

**Efficient bulk transposition with transducers:**
```clojure
;; Transpose melody from C major to F major
(into []
      (map (make-transposer :up :perfect :fourth))
      ["C4" "D4" "E4" "F4" "G4"])
=> ["F4" "G4" "A4" "Bb4" "C5"]
```

### Internal Mathematics: The (D,A,C) Model

**Diatonic transposition preserves letter relationships:**

For interval string and fluid keyword lanes, the system uses an internal `(D,A,C)` model:

- **D** = signed letter steps (0=unison, 1=2nd, 7=octave, -1=2nd down)
- **A** = signed semitone adjustment relative to Major/Perfect base
  - **+1 always raises** final pitch height, **-1 always lowers**, regardless of direction  
- **C** = signed cents applied after semitone calculation

**Example: Major second up from B3**
1. `D=1` (B→C letter step)
2. `A=0` (major 2nd is base interval)  
3. Target letter: C4
4. Base interval B→C = 1 semitone
5. With A=0 adjustment = 1 semitone  
6. B3 + 1 semitone = C4... but we need C#4
7. System calculates: B3 (2320¢) + 200¢ = 2520¢ = C#4

**Critical feature: No enharmonic simplification**
- Double/triple/quadruple accidentals preserved exactly
- `"B##3"` → `"C###4"` (not `"D4"`) when appropriate
- Letter-first logic maintains musical spelling relationships

### Round-Trip Guarantees

**Diatonic transposition:**
```clojure
;; Exact spelling preservation
(let [up   (make-transposer :interval "M6+")
      down (make-transposer :interval "M6-")]
  (-> "C4" up down))  => "C4"   ; Exact round-trip

(let [up   (make-transposer :up :major :third)  
      down (make-transposer :down :major :third)]
  (-> "Ebb4" up down))  => "Ebb4"   ; Preserves double-flat
```

**Chromatic transposition:**
```clojure
;; Enharmonically equivalent round-trip
(let [up   (make-transposer :chromatic 5)
      down (make-transposer :chromatic -5)]
  (enharmonically-equal? "C4" (-> "C4" up down)))  => true
  ;; Note: May not preserve exact spelling due to canonical normalization
```

---

## Implementation & Performance

### Error Handling

**Validation catches common mistakes:**

```clojure
;; Lane conflicts
(make-transposer :interval "M2+" :down)  ; ExceptionInfo
(make-transposer :up :down :major :second)  ; ExceptionInfo

;; Invalid quality/degree combinations  
(make-transposer :minor :fourth)  ; ExceptionInfo - minor not valid on perfect degrees
(make-transposer :perfect :third)  ; ExceptionInfo - perfect not valid on major degrees

;; Type errors
(make-transposer :chromatic "2")  ; ExceptionInfo - must be integer
(make-transposer :interval :M2)   ; ExceptionInfo - must be string  
```

### Multi-Layer Caching Strategy

**Optimised for repeated operations:**

1. **Factory cache**: Keyed by raw argument vector (no canonicalisation overhead)
2. **Global output caches**:
   - Diatonic: `[D A C pitch-str]` → output
   - Chromatic: `[S C pitch-str]` → output  
3. **Primitive hot caches**: Pitch decomposition and conversion (memoized)
4. **Shared global access**: Individual transposer functions delegate to shared global caches using their configuration parameters as part of the cache key

**Cache keys:**
- Diatonic global cache: `[D A C pitch-str]` (shared by all diatonic transposers)
- Chromatic global cache: `[S C pitch-str]` (shared by all chromatic transposers)  
- Factory cache: raw argument vector
- Conversion cache: pitch string

**Cache management:**
```clojure
(clear-transposition-caches!)  ; Clears all transposition-related caches
(clear-convert-cache!)         ; Clears pitch conversion cache
```

### Why a Factory

* **Higher-order closures** capture configuration once; application cost is minimal.
* **Composability** with `comp`, threading, etc.
* **Separation of concerns:** interval-string lane (UI-friendly), fluid lane (dialog-friendly), and chromatic lane (playback/MIDI).

**Benefits:**
- Closures capture configuration once → minimal per-call cost
- Naturally composable with `comp`, threading, transducers
- Supports multiple syntaxes (interval strings, keywords, chromatic) without duplicating logic

### Performance Benchmarks

**2017 MacBook Pro (2.2 GHz 6-core Intel):**
- **1,000,000 transpositions in 5637.002775 ms**
- **≈ 177,500 operations/second**  
- **≈ 5.64 µs per transposition**

Includes global caching and hot-path memoization. Performance scales well for interactive editing, part extraction, and batch processes.

### Instrument Integration

**Transposition specifications in instrument definitions:**

```clojure
{:name "B♭ Clarinet"
 :sounding->written [:interval "M2+"]
 :written->sounding [:interval "M2-"]}

{:name "Horn in F"  
 :sounding->written [:up :perfect :fifth]
 :written->sounding [:down :perfect :fifth]}

{:name "Bass Clarinet in B♭"
 :sounding->written [:up :major :ninth]     ; M2 + octave
 :written->sounding [:down :major :ninth]}

{:name "Quarter-tone Trumpet"
 :sounding->written [:chromatic 6 :cents 50]    
 :written->sounding [:chromatic -6 :cents -50]}
```

These vectors are applied as `(apply make-transposer params)`.

**Integration with musical processing**: Pitch operations are extensively used within the timewalking system ([ADR-0014 Timewalk](0014-Timewalk.md)) for musical analysis, frequency calculations, and cross-measure processing.

---

## Outcome

### Delivered Capabilities

- **Minimal, expressive API**: Single string format + factory with three intuitive input lanes
- **Musical correctness**: Exact diatonic spelling preservation, microtonal stability, no unwanted simplification  
- **Scalable performance**: Sub-10 µs operations with layered caching on older hardware
- **Universal expressiveness**: Extreme accidentals, compound intervals, unlimited range
- **Integration ready**: Natural fit with parsers, serializers, instrument definitions

### Architectural Benefits

- **Compact representation**: Strings significantly more memory-efficient than compound objects
- **Human readable**: Musicians can read `"Bb4-15"` directly in debugging and logs  
- **Cache-friendly**: String keys enable efficient memoization strategies
- **Composable**: Transposers work naturally with `comp`, threading macros, transducers
- **Extensible**: Three-lane factory accommodates different UI contexts and use cases

### Foundation for Higher-Level Features

This ADR establishes pitch operations as a stable, high-performance foundation for:
- Part extraction and transposition
- Score analysis and harmonic functions
- Audio rendering systems via frequency calculations
- Notation layout and formatting
- Interactive editing and playback features

The string-based representation decision enables all of Ooloi's sophisticated musical capabilities whilst maintaining simplicity and performance at the core.

## See Also

- [ADR-0023: Shared Model Contracts](0023-Shared-Model-Contracts.md) - Unified data model architecture
- [ADR-0033: Time Signature Architecture](0033-Time-Signature-Architecture.md) - String-based time signature representation following same pattern
- [ADR-0034: Key Signature Architecture](0034-Key-Signature-Architecture.md) - Key signatures using pitch representation for tonic specification
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) - Accidental display logic leveraging pitch string parsing
