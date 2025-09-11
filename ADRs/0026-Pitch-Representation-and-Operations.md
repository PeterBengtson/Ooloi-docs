# ADR-0026: Pitch Representation and Operations

Decision: Adopt string-based pitch representation ("C#4+25") as canonical format with comprehensive transposition operations through a factory-based transposer system supporting both chromatic and diatonic modes.

## Table of Contents

- [Context](#context)
- [Decision](#decision)
- [Rationale](#rationale)
- [Why a Factory](#why-a-factory)
- [Detailed Design](#detailed-design)
- [Challenges Addressed](#challenges-addressed)
- [Caching Strategy](#caching-strategy)
- [Instrument Integration](#instrument-integration)
- [Outcome](#outcome)

## Context

Ooloi requires a pitch representation system that balances computational efficiency with musical correctness. The system must serve two critical needs:

1. **Playback accuracy**: Precise frequency calculations for MIDI output and audio rendering
2. **Notation correctness**: Proper spelling preservation after transposition, including microtonal cent offsets and accidentals for contemporary music

### Design Alternatives Considered

**Igor Engraver Approach**: Used compound pitch objects with separate fields for letter name, accidental, octave, and microtonal offset. While semantically clear, this approach creates overhead for large orchestral scores where thousands of pitches require frequent operations.

**Ooloi's String-Based Approach**: Adopts compact string notation ("C#4+25") as the canonical representation, offering several advantages:

- **Compact for large scores**: Single string per pitch reduces memory overhead in complex works
- **Human readable**: Musicians can directly interpret "Bb4-15" without decoding object fields  
- **Natural parsing integration**: String format works seamlessly with score parsing and caching systems
- **Microtonal support**: Arbitrary cent offsets handle microtonal music requirements

### Musical Requirements

The system must handle diverse musical contexts:

- **Traditional tonal music**: Standard 12-tone equal temperament with double accidentals (C##, Bbb) and precise spelling preservation
- **Contemporary atonal music**: Primarily single accidentals without reference to a specific key
- **Microtonal music**: Arbitrary cent deviations with consistent transposition behavior and MIDI playback support via pitch bend or direct virtual instrument control
- **Instrument transposition**: Both chromatic (frequency-based) and diatonic (letter-preserving) modes

## Decision

Implement comprehensive pitch operations in `ooloi.shared.ops.pitches` with the following core components:

### Primary Operations
- **Normalization**: Canonical string format with validated ranges
- **Conversion**: MIDI number and Hz frequency calculation with memoization
- **Sorting**: Frequency-based pitch ordering
- **Enharmonic comparison**: Equivalence testing across different spellings
- **Transposition**: Factory-based transposer system

### Transposer Factory Signature

```clojure
(make-transposer MODE DIRECTION DELTA & [CENTS])
```

**Parameters**:
- `MODE`: `:diatonic` | `:chromatic` - Controls spelling behavior
- `DIRECTION`: `:up` | `:down` - Transposition direction  
- `DELTA`: Non-negative integer semitones (octaves folded in)
- `CENTS`: Optional integer cents (default 0)

**Mode Semantics**:
- `:chromatic`: Produces canonical sharp spelling for consistent playback math
- `:diatonic`: Preserves letter-name relationships, adds accidentals as needed, no simplification

**Validation**: Invalid inputs throw `ex-info` with descriptive error messages.

## Rationale

### String Representation Benefits

**Efficiency**: Single string representation reduces memory allocation compared to multi-field objects, critical for large orchestral scores with thousands of pitch instances.

**Readability**: Musicians can directly interpret pitch strings during development and debugging without requiring object field inspection.

**Integration**: String format aligns naturally with score file parsing, caching systems, and serialization requirements.

### Two-Mode Transposition System

**Chromatic Mode**: Essential for accurate playback mathematics where enharmonic equivalents (C# = Db) must produce identical frequencies. Guarantees consistent MIDI output regardless of input spelling.

**Diatonic Mode**: Critical for notation correctness where spelling relationships must be preserved. A piece requiring "C## to D##" progression cannot accept chromatic mode's "C# to Eb" output, as this destroys the intended harmonic analysis.

**No Simplification Policy**: Automatic enharmonic simplification (C## → D) would corrupt composer intent. Contemporary music frequently uses extreme accidentals for specific harmonic or analytic purposes that must be preserved exactly.

## Why a Factory

The factory pattern leverages higher-order functions and closures to create efficient, composable transposition operations. Rather than repeatedly passing configuration parameters, the factory generates specialized functions that encapsulate both the transposition logic and their specific musical behavior rules.

### Higher-Order Function Benefits

**Closure Efficiency**: Each generated transposer is a closure that captures its configuration (mode, direction, delta, cents) at creation time. This eliminates parameter passing overhead on every transposition operation.

**Composability**: Transposer functions integrate seamlessly with Clojure's functional composition operators (`comp`, `partial`, threading macros), enabling complex musical transformations.

**Performance**: Pre-configured functions eliminate runtime conditional branching. A diatonic transposer contains only diatonic logic, while a chromatic transposer contains only chromatic logic.

### Musical Context

Consider these distinct musical operations:
- **"Up a major second diatonically"**: C## → D## (preserves letter relationship)
- **"Up a major second chromatically"**: C## → E (frequency-based, canonical spelling)
- **"Down a minor third with 25 cents"**: Microtonal operation with simple cent deviation

Each represents a fundamentally different musical operation, despite similar mathematical foundations.

### Technical Benefits

**Clarity**: Single factory call `(make-transposer :diatonic :up 2)` creates a clear, reusable function rather than requiring repeated parameter passing to a generic operation.

**Reuse**: Orchestra scores apply the same transposition to thousands of notes. Generated functions can be stored and reused across entire movements without reconfiguration overhead.

**Caching**: Factory memoization ensures identical parameter sets reuse the same function object, eliminating redundant closure creation.

**Instrument Integration**: Real instruments store transposition parameters directly (e.g., Bb clarinet: `:diatonic :up 2`). These parameters plug directly into the factory without transformation.

## Detailed Design

### String Format and Parsing

**Canonical Format**: `[A-G][accidentals][octave][cents]`
- Letters: A-G (case insensitive, normalized to uppercase)
- Accidentals: # (sharp), b (flat), multiple allowed (##, bbb)
- Octave: Integer (C0 = 16.35 Hz, middle C = C4, A0 = lowest piano note at 27.5 Hz)
- Cents: Optional +/- prefix with arbitrary range for microtonal precision

**Normalization Process**:
1. Parse components using regex pattern matching
2. Validate ranges: accidentals ≤ 3, octave -1 to 9, cents arbitrary range
3. Canonical case conversion and format standardization

### Conversion Operations

**MIDI Calculation**: 
- Base semitone calculation from C4 = 60
- Accidental contribution (sharp +1, flat -1 per symbol)
- Cent offset normalization and addition
- Memoized for performance on repeated conversions

**Frequency Calculation**:
- Equal temperament: 440 * 2^((midi - 69) / 12)
- Supports arbitrary cent deviations for microtonal accuracy

**Microtonal MIDI Support**:
- Cent deviations enable precise microtonal playback unlike most notation programs
- Implementation options: MIDI pitch bend messages (as used by Igor Engraver) or direct virtual instrument control
- Maintains playback accuracy for composers working with quarter-tones, just intonation, or other microtonal systems

### Sorting Implementation

Frequency-based ordering provides musically intuitive results across enharmonic spellings. Implementation uses MIDI number comparison for efficiency while maintaining precise microtonal ordering.

### Enharmonic Equivalence

Compares normalized frequency values, abstracting away spelling differences. Critical for harmonic analysis and chord recognition where C# and Db must be treated identically.

### Transposition Algorithms

**Chromatic Mode**:
1. Convert input pitch to MIDI + cents
2. Apply semitone + cent delta
3. Convert result to canonical sharp spelling
4. Handle octave overflow and underflow

**Diatonic Mode**:
1. Calculate letter name advancement from semitone count
2. Advance base letter by required steps
3. Calculate chromatic target frequency  
4. Add accidentals to reach target while preserving base letter
5. Maintain exact spelling relationships

## Challenges Addressed

### Diatonic Transposition Complexity

Diatonic transposition must preserve letter-name relationships while reaching correct chromatic targets. Algorithm ensures C## + major second = D## (not Eb), maintaining harmonic analysis integrity.

**Implementation Strategy**: Letter-first approach calculates letter advancement, then adds accidentals to reach chromatic target frequency.

### Microtonal Consistency

Arbitrary cent offsets must transpose predictably across all operations. System ensures +25 cent deviation transposed up one semitone consistently becomes +25 cents in the target pitch.

### Round-Trip Guarantees

**Diatonic Round-Trip**: Bijective on spelling preservation
- Input: C##4 → Up major second → D##4 → Down major second → C##4
- Guarantees exact spelling recovery

**Chromatic Round-Trip**: Bijective on frequency (enharmonic equivalence)
- Input: C##4 → Up major second → E4 → Down major second → D4  
- Guarantees enharmonic equivalence of result

## Caching Strategy

Multi-layer caching system optimizes performance across different usage patterns:

### Primitive Operation Caches
- **Pitch Decomposition**: Memoized parsing of string components
- **Cents Conversion**: Cached cent-to-pitch calculations for common values

### Transposer Function Caches  
Each generated transposer function maintains internal memoization of its outputs, optimizing repeated application to the same pitches.

### Factory Cache
Factory itself memoizes generated functions by parameter tuple `(mode, direction, delta, cents)`, ensuring identical configurations reuse existing function objects.

### Cache Management
`clear-transposition-caches!` utility provides explicit cache clearing for testing scenarios and memory management in long-running applications.

## Instrument Integration

Real-world instruments declare transpositions as parameter tuples that integrate directly with the factory system:

### Example Instrument Configurations

**Bb Clarinet** (sounds major second down):
- Written-to-sounding: `(make-transposer :diatonic :down 2)`
- Sounding-to-written: `(make-transposer :diatonic :up 2)`

**Piccolo** (sounds octave up):
- Written-to-sounding: `(make-transposer :diatonic :up 12)`  
- Sounding-to-written: `(make-transposer :diatonic :down 12)`

**French Horn in F** (sounds perfect fifth down):
- Written-to-sounding: `(make-transposer :diatonic :down 7)`
- Sounding-to-written: `(make-transposer :diatonic :up 7)`

### Direct Parameter Integration

Instrument definitions store transposition parameters as tuples that map directly to factory calls:
```clojure
{:name "Bb Clarinet"
 :written-to-sounding [:diatonic :down 2]
 :sounding-to-written [:diatonic :up 2]}

{:name "French Horn in F"
 :written-to-sounding [:diatonic :down 7]
 :sounding-to-written [:diatonic :up 7]}

{:name "Quarter-tone Trumpet" 
 :written-to-sounding [:chromatic :up 6 50]  ; Up tritone + 50 cents
 :sounding-to-written [:chromatic :down 6 50]}
```

These parameter tuples apply directly to the factory via `(apply make-transposer params)`, enabling clean separation between instrument data and transposition logic.

## Outcome

The pitch representation and operations system provides:

### Minimal API Surface
Simple string format and single factory function reduce cognitive load for musicians and developers working with pitch operations.

### Comprehensive Musical Expression  
Supports traditional tonal music, contemporary atonal compositions, and microtonal works through unified interface without compromising any musical requirement.

### Performance Optimization
Multi-layer caching strategy scales efficiently from individual pitch operations to large orchestral score processing.

### Musical Correctness
Separation of chromatic (playback) and diatonic (notation) modes ensures both frequency accuracy and spelling preservation according to musical context.

### Architectural Stability
Factory-based design provides extensibility for future musical requirements while maintaining backward compatibility and clear separation of concerns.

The system establishes pitch operations as a settled architectural component, enabling confident development of higher-level musical features without revisiting fundamental representation decisions.