# ADR-0041: Ooloi Virtual Instrument Definition (OVID)

## Table of Contents

- [Status](#status)
- [Context](#context)
  - [Relationship to Plugin-Based Audio Architecture](#relationship-to-plugin-based-audio-architecture)
  - [The Vendor Fragmentation Problem](#the-vendor-fragmentation-problem)
  - [Historical Context: From MIDI Workarounds to Host Control](#historical-context-from-midi-workarounds-to-host-control)
  - [The Calibration Problem](#the-calibration-problem)
- [Decision](#decision)
  - [Core Architecture](#core-architecture)
  - [Design Principles](#design-principles)
- [Specification](#specification)
  - [Schema Structure](#schema-structure)
  - [Canonical Technique Taxonomy](#canonical-technique-taxonomy)
  - [Selection Mechanisms](#selection-mechanisms)
  - [Calibration Requirements](#calibration-requirements)
  - [Fallback Generators](#fallback-generators)
  - [Library Manifests](#library-manifests)
- [Bundled OVID Files](#bundled-ovid-files)
- [Reference Implementations](#reference-implementations)
  - [Spitfire BBCSO Discover — Violins 1](#spitfire-bbcso-discover--violins-1)
  - [VSL Synchron Elite Strings — Violins](#vsl-synchron-elite-strings--violins)
- [Rationale](#rationale)
  - [Why YAML](#why-yaml)
  - [Why Canonical Taxonomy](#why-canonical-taxonomy)
  - [Why Mandatory Calibration](#why-mandatory-calibration)
  - [Why Host Control Over MIDI](#why-host-control-over-midi)
- [Challenges Addressed](#challenges-addressed)
  - [Vendor Fragmentation](#vendor-fragmentation)
  - [Version Drift](#version-drift)
  - [Missing Articulations](#missing-articulations)
  - [Missing Instruments](#missing-instruments)
  - [Cross-Library Consistency](#cross-library-consistency)
- [Integration Points](#integration-points)
  - [ADR-0027: Plugin-Based Audio Architecture](#adr-0027-plugin-based-audio-architecture)
  - [ADR-0003: Plugin Architecture](#adr-0003-plugin-architecture)
  - [Notation Semantics in Core](#notation-semantics-in-core)
  - [Future: Humanisation and Performance Interpretation](#future-humanisation-and-performance-interpretation)
- [Alternative Approaches Considered](#alternative-approaches-considered)
  - [Per-Plugin Hardcoded Mappings](#per-plugin-hardcoded-mappings)
  - [MIDI-Based Control](#midi-based-control)
  - [Vendor-Specific APIs Only](#vendor-specific-apis-only)
  - [No Specification](#no-specification)
- [Success Criteria](#success-criteria)
  - [Technical Metrics](#technical-metrics)
  - [Ecosystem Metrics](#ecosystem-metrics)
- [Future Extensions](#future-extensions)
- [Conclusion](#conclusion)
- [Appendix A: Vendor Control Surface Research](#appendix-a-vendor-control-surface-research)
- [References](#references)

## Status

Accepted

## Context

### Relationship to Plugin-Based Audio Architecture

ADR-0027 establishes that Ooloi's core provides no audio or MIDI functionality whatsoever. All audio processing, playback, and virtual instrument control are delegated to frontend plugins. This architectural boundary creates a requirement: plugins that host virtual instruments need a standardised way to control them.

ADR-0027 also establishes that MIDI input (from keyboards and controllers) is handled by the frontend, translated to API calls, with the backend receiving only high-level musical commands. The backend never sees MIDI data in either direction: not from input devices, not for playback control.

OVID addresses the playback control requirement. It defines how playback plugins communicate with virtual instruments using modern host control mechanisms rather than OS-level MIDI routing.

### The Vendor Fragmentation Problem

Professional virtual instrument libraries use inconsistent control mechanisms:

- **Spitfire** (BBCSO, Abbey Road, Symphonic): Technique Selector with host automation, movable keyswitches, Tightness/Attack parameters
- **Orchestral Tools** (SINE): Articulation Lists with keyswitch/CC/program change, Tightness and Legato options
- **Vienna** (Synchron Player): Dimension Trees with Remote Control, Start Offset, Humanize, Time-Stretch parameters
- **EastWest** (Opus): Multi-articulation instruments with keyswitches and Program Change
- **Audio Modeling** (SWAM): Fully parameter-driven with dynamics, expression, vibrato, portamento time exposed as automatable parameters

Each vendor uses different terminology for equivalent concepts ("Tightness" vs "Start Offset" vs "Attack"), different selection mechanisms (parameters vs keyswitches vs dimension paths), and different default calibrations (loudness levels, onset timing). A playback plugin without an abstraction layer must implement vendor-specific code paths for every supported library.

### Historical Context: From MIDI Workarounds to Host Control

Igor Engraver (1996-2001) achieved sophisticated playback by seizing complete control of MIDI units. The system employed pitch bend manipulation for microtonal accuracy, distributing notes across MIDI channels using register allocation algorithms borrowed from compiler technology. In chords containing microtonally altered notes, each note played on different channels with dynamic patch changes—a "DNA soup" of resource allocation that extracted capabilities beyond MIDI's nominal sixteen-channel limit.

Critically, Igor managed device-specific capabilities through **Synth Matrices**—text files that described each synthesiser's characteristics: available patches, controller mappings, pitch bend ranges, channel limitations, and device-specific behaviours. Users could edit these files to add support for new synthesisers or refine existing descriptions. Synth Matrices were shareable; a user who configured a new device could distribute their matrix file to others.

OVID generalises the Synth Matrix concept from MIDI synthesisers to virtual instrument libraries. The underlying principle remains: declarative text files describing device capabilities, user-manageable, and distributable. What changes is the control surface (VST3/AU host mechanisms rather than MIDI) and the scope (articulation-level detail rather than patch-level).

Igor's approach was constrained by MIDI's protocol limitations: sixteen channels, 7-bit controller resolution, device-specific compatibility requirements. Modern plugin hosting eliminates these constraints:

- **Unlimited precision**: Per-note tuning without channel allocation
- **Direct parameter access**: Articulation selection without external MIDI routing
- **Per-note expression**: Where plugins support it, tuning/volume/timbre can vary per note
- **Articulation advertisement**: Some plugin formats allow hosts to discover available techniques

VST3 and AU provide different implementations of these capabilities. VST3 offers IKeyswitchController and INoteExpressionController; AU has different mechanisms. OVID abstracts over format differences, describing capabilities rather than interface specifics.

OVID leverages these modern capabilities while preserving both the musical intelligence and the user-manageable device description architecture that made Igor's approach effective. Unlike Igor, OVID does not output MIDI; playback goes directly to virtual instruments via host control. Users who need MIDI output to external hardware would require a dedicated plugin—which could reimplement Igor's sophisticated allocation strategies or use simpler fixed-channel mapping.

### The Calibration Problem

Virtual instrument libraries ship with inconsistent calibration:

**Loudness variation**: Libraries from different vendors—and even different products from the same vendor—use different reference levels. Combining Spitfire strings with Vienna brass produces unpredictable balance without manual adjustment.

**Onset timing variation**: Sample start points differ by articulation and vendor. A legato transition might have 80ms of pre-attack; a staccato might be precisely aligned. Without compensation, rhythmic placement becomes inconsistent.

**Articulation availability**: Not all libraries provide all articulations. BBCSO Discover lacks native trills; Synchron Elite Strings includes col legno battuto. A playback system must handle missing articulations gracefully.

OVID addresses calibration through mandatory loudness normalisation, onset alignment parameters, and fallback generators for missing articulations.

## Decision

We adopt a **declarative YAML-based definition** for virtual instrument control that:

1. **Defines instrument capabilities** in human-readable, version-controllable files
2. **Uses canonical technique names** abstracting over vendor-specific terminology
3. **Specifies selection mechanisms** via host control (parameters, articulation selection, note expression)
4. **Requires calibration data** for loudness normalisation and onset alignment
5. **Provides fallback definitions** for synthesising missing articulations
6. **Excludes OS-level MIDI routing**—all control stays within the plugin host; no external MIDI devices or DAW routing required

### Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Playback Plugin                             │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────────┐   │
│  │   OVID      │    │  Selection   │    │  Virtual          │   │
│  │   Files     │───▶│  Engine      │───▶│  Instrument       │   │
│  │   (YAML)    │    │              │    │  (VST3/AU)        │   │
│  └─────────────┘    └──────────────┘    └───────────────────┘   │
│         │                  │                      │             │
│         ▼                  ▼                      ▼             │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────────┐   │
│  │  Canonical  │    │  Calibration │    │  Host Control:    │   │
│  │  Taxonomy   │    │  (Loudness,  │    │  - Parameters     │   │
│  │  Mapping    │    │   Onset)     │    │  - Event Bus      │   │
│  └─────────────┘    └──────────────┘    └───────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Design Principles

**Declarative over procedural**: OVID files describe what exists and how to access it, not how to implement playback logic.

**Canonical abstraction**: Vendor-specific names map to a standard taxonomy. The playback engine works with `legato_portamento`; the OVID file translates this to Synchron's `["Legato","Portamento"]` dimension path.

**Priority-ordered selection**: Multiple selection mechanisms listed in preference order. The engine uses the first supported method.

**Graceful degradation**: Missing articulations trigger fallback generators rather than failure.

**No OS-level MIDI routing for playback**: Playback control stays within the plugin host via parameter automation and host-internal event delivery to virtual instruments. Ooloi does not output MIDI. Users need not configure external MIDI devices, DAW routing, or inter-application MIDI.

**MIDI output would require a plugin**: If a user wanted MIDI output to external hardware synthesisers, they would need a dedicated plugin. Such a plugin could be simple (fixed channel per instrument) or sophisticated (Igor-style dynamic channel allocation with pitch bend for microtonality). OVID does not address MIDI output; it addresses control of virtual instruments within the plugin host.

**MIDI input for note entry is separate**: The frontend accepts MIDI input from keyboards and controllers for note entry purposes, likely handled by a plugin. This MIDI input is translated to API calls; the backend never sees MIDI data. The distinction is: MIDI *input* (note entry) flows through the frontend and becomes semantic commands; playback *output* goes to virtual instruments via host control, not MIDI.

**Format-neutral capabilities**: OVID describes capabilities (parameter automation, articulation selection, per-note expression) rather than format-specific interfaces. VST3 and AU provide different implementations of these capabilities; OVID abstracts over them.

**Schema examples are illustrative**: The YAML structures in this ADR demonstrate the specification's intent. Exact field names, nesting, and syntax may evolve during implementation. Normative schema definitions will be maintained alongside the implementation.

## Specification

### Schema Structure

```yaml
library: <vendor>/<product>           # e.g., spitfire/bbcso-discover
player: <engine>                      # spitfire|sine|synchron|opus|kontakt|sfz|swam
plugin:
  format: VST3                        # or AU
  identifier: <plugin-id>             # host plugin id
  version: ">=1.0.0"                  # minimum supported version

instrument:
  id: <unique-identifier>             # e.g., "vl1-section"
  name: <display-name>                # e.g., "Violins 1"
  family: <instrument-family>         # strings|woodwinds|brass|percussion|keyboards
  range: [<low-pitch>, <high-pitch>]  # playable range, e.g., [G3, C#7]

calibration:
  tuning_hz: <reference-pitch>        # typically 440.0
  loudness_target_lufs: <target>      # reference level, typically -23.0
  gain_db: <offset>                   # library-level offset after measurement
  mic: { preset: <name> }             # or explicit multi-mic blend
  onset_defaults_ms:                  # fallback onset compensation
    legato: <ms>                      # typically -60 to -90
    long: <ms>                        # typically -20 to -40
    short: <ms>                       # typically 0 to -10

controls:                             # automatable parameters
  - id: <canonical-name>              # e.g., dynamics, expression, tightness
    map: { type: param, name: <vendor-name> }

articulations:
  - tech: <canonical-technique>       # from taxonomy
    native: true|false|maybe          # availability status
    select:                           # priority-ordered selection methods
      - { type: param, name: <name>, value: <value> }
      - { type: keyswitch, pitch: <pitch> }
      - { type: dimension, path: [<path>] }
    onset_ms: <compensation>          # articulation-specific timing
    controls: { <id>: <value> }       # articulation-specific parameter values
    fallback:                         # if native: false
      generator: <generator-type>
      <generator-specific-params>
```

### Canonical Technique Taxonomy

The taxonomy provides vendor-agnostic names for articulations, mapped to notation semantics. Playback plugins work with canonical names; OVID files provide the vendor-specific translation.

**Sustained Techniques**

| Canonical Name | Notation Equivalent | Description |
|----------------|---------------------|-------------|
| `long` | Default sustained | Basic sustain without specific articulation |
| `long_senza_vib` | "senza vib." | Sustain without vibrato |
| `long_con_vib` | "con vib." | Sustain with vibrato |
| `long_molto_vib` | "molto vib." | Sustain with intense vibrato |
| `long_sul_tasto` | "sul tasto" | Bowed over fingerboard (strings) |
| `long_sul_pont` | "sul ponticello" | Bowed near bridge (strings) |
| `long_con_sord` | "con sordino" | With mute |
| `long_flautando` | "flautando" | Flute-like tone (strings) |
| `long_harmonics` | Harmonic notation | Natural or artificial harmonics |

**Short Techniques**

| Canonical Name | Notation Equivalent | Description |
|----------------|---------------------|-------------|
| `staccato` | Dot | Standard short note |
| `staccatissimo` | Wedge | Very short, accented |
| `spiccato` | Context-dependent | Bounced bow (strings) |
| `pizzicato` | "pizz." | Plucked (strings) |
| `pizzicato_snap` | Bartók pizz. | Snap pizzicato |
| `col_legno_battuto` | "col legno battuto" | Struck with wood of bow |
| `col_legno_tratto` | "col legno tratto" | Drawn with wood of bow |

**Legato Techniques**

| Canonical Name | Notation Equivalent | Description |
|----------------|---------------------|-------------|
| `legato_normal` | Slur | Standard legato transition |
| `legato_portamento` | Glissando line + "port." | Pitch slide between notes |
| `legato_glissando` | Glissando line | Measured pitch slide |
| `legato_fingered` | Slur (strings context) | Finger-only legato |
| `legato_bowed` | Slur (strings context) | Bow-change legato |

**Tremolo and Trill Techniques**

| Canonical Name | Notation Equivalent | Description |
|----------------|---------------------|-------------|
| `tremolo` | Tremolo strokes | Unmeasured bow tremolo |
| `trill_half` | tr + ♭/♮ | Trill to note half-step above |
| `trill_whole` | tr + ♮/♯ | Trill to note whole-step above |

**Wind-Specific Techniques**

| Canonical Name | Notation Equivalent | Description |
|----------------|---------------------|-------------|
| `flutter_tongue` | Flutter notation | Flatterzunge |

**Brass-Specific Techniques**

| Canonical Name | Notation Equivalent | Description |
|----------------|---------------------|-------------|
| `muted_straight` | "straight mute" | Standard straight mute |
| `muted_cup` | "cup mute" | Cup mute |
| `muted_harmon` | "harmon mute" | Harmon/wah-wah mute |
| `stopped` | + symbol | Hand-stopped (horn) |
| `rip` | Rip notation | Upward glissando |
| `fall` | Fall notation | Downward glissando |

### Selection Mechanisms

Selection methods listed in priority order. The playback engine attempts each method until one succeeds.

**Parameter Selection**

```yaml
select:
  - { type: param, name: "Technique", value: "Long" }
```

Sets a plugin parameter to a specific value. Most reliable when available.

**Keyswitch Selection**

```yaml
select:
  - { type: keyswitch, pitch: "C-2" }
```

Sends a note event to the plugin via the host's internal event delivery to trigger articulation selection. The pitch uses standard notation (C-2 = MIDI note 0). This is not MIDI output; the event stays within the plugin host.

**Dimension Selection (Synchron-specific)**

```yaml
select:
  - { type: dimension, path: ["Legato", "Portamento"] }
```

Navigates Synchron's dimension tree structure.

**Articulation Index Selection**

```yaml
select:
  - { type: articulation_index, index: 3 }
```

Uses articulation index where the plugin advertises available articulations to the host (VST3's IKeyswitchController or equivalent). Not all plugins support this.

### Calibration Requirements

**Loudness Normalisation**

All OVID files must specify:

```yaml
calibration:
  loudness_target_lufs: -23.0    # EBU R128 reference
  gain_db: 0.0                   # measured offset for this library
```

The `loudness_target_lufs` establishes a common reference level (EBU R128 recommends -23 LUFS for broadcast; this provides adequate headroom for orchestral dynamics). The `gain_db` offset compensates for measured deviation from target.

**Measurement procedure (approximate)**: Load instrument, set to `long` articulation at mezzo-forte dynamic, play sustained reference material, measure integrated LUFS per ITU-R BS.1770. Record offset required to reach target. This procedure provides a useful approximation; exact calibration varies with mic position, dynamic layer, round-robin variation, and release characteristics. The goal is cross-library consistency within practical tolerances, not laboratory precision.

**Onset Alignment**

Articulations have different attack characteristics. OVID compensates through:

1. **Plugin controls** where available (Tightness, Start Offset, Attack)
2. **Per-articulation onset_ms** values specifying track delay compensation

```yaml
calibration:
  onset_defaults_ms:
    legato: -70      # legato transitions need significant pre-roll
    long: -30        # sustained notes have moderate attack
    short: 0         # shorts typically aligned to transient

articulations:
  - tech: legato_portamento
    onset_ms: -85    # override default for this specific articulation
```

Negative values indicate the sample starts before the notated onset; the playback engine advances the trigger time accordingly.

**Limitations**: A single static offset per articulation is an approximation. Legato timing in particular depends on interval, tempo, and performance scripting. The onset_ms value provides a useful baseline; future refinements may allow tempo-dependent or interval-dependent adjustments.

### Fallback Generators

When `native: false`, the OVID file specifies a fallback generator:

**Trill Generator**

```yaml
fallback:
  generator: trill
  interval: m2|M2|m3           # interval above written note
  rate_hz_by_dynamic:          # trill speed varies with dynamic
    p: 8
    mf: 10
    ff: 12
  phase_jitter_ms: 5           # optional humanisation
```

Synthesises alternation between written note and upper auxiliary at tempo/dynamic-scaled rate.

**Tremolo Generator**

```yaml
fallback:
  generator: tremolo
  rate_hz: 12                  # unmeasured default
  amplitude_variance: 0.1      # bow pressure variation
```

Synthesises unmeasured bow tremolo through rapid note retriggering with amplitude shaping.

**Portamento Generator**

```yaml
fallback:
  generator: portamento_glide
  time_ms: 140                 # glide duration
  curve: "ease-in-out"         # pitch curve shape
```

Synthesises pitch glide between notes using pitch bend or per-note tuning.

**Glissando Generator**

```yaml
fallback:
  generator: glissando
  mode: chromatic|diatonic|continuous
  time_ms: 200
```

Synthesises measured pitch slide, either as discrete steps or continuous bend.

**None (Explicit Absence)**

```yaml
fallback:
  generator: none
```

Indicates no fallback available. Used when synthesis would be musically inappropriate (e.g., flutter tongue on strings).

### Library Manifests

Individual OVID files describe instruments. Library manifests describe what instruments exist within a library, enabling the playback engine to determine availability without scanning individual files.

**The coverage problem**: Libraries vary dramatically in instrument coverage. BBC Symphony Orchestra Discover provides 34 instruments with a polished, coherent sound but lacks contrabassoon, bass clarinet, harp, piano, keyboard percussion, and various auxiliary winds. Virtual Playing Orchestra covers 168 patches including these missing instruments but with uneven recording quality. A score requiring contrabassoon cannot play back on BBCSO Discover alone.

**Manifest structure**:

```yaml
# Library manifest: spitfire/bbcso-discover/manifest.yaml
library: spitfire/bbcso-discover
name: "BBC Symphony Orchestra Discover"
vendor: spitfire
player: spitfire
plugin:
  format: VST3
  identifier: "Spitfire BBC Symphony Orchestra"
  version: ">=1.6"

calibration_defaults:
  tuning_hz: 440.0
  loudness_target_lufs: -23.0

instruments:
  strings:
    - { id: vl1-section, name: "Violins 1" }
    - { id: vl2-section, name: "Violins 2" }
    - { id: va-section, name: "Violas" }
    - { id: vc-section, name: "Cellos" }
    - { id: cb-section, name: "Basses" }
  woodwinds:
    - { id: fl-solo, name: "Flute" }
    - { id: fl-section, name: "Flutes a2" }
    - { id: ob-solo, name: "Oboe" }
    - { id: ob-section, name: "Oboes a2" }
    - { id: cl-solo, name: "Clarinet" }
    - { id: cl-section, name: "Clarinets a2" }
    - { id: bn-solo, name: "Bassoon" }
    - { id: bn-section, name: "Bassoons a2" }
    # Note: No bass clarinet, contrabassoon, or auxiliary winds
  brass:
    - { id: hn-solo, name: "Horn" }
    - { id: hn-section, name: "Horns a4" }
    - { id: tpt-solo, name: "Trumpet" }
    - { id: tpt-section, name: "Trumpets a2" }
    - { id: tbn-solo, name: "Trombone" }
    - { id: tbn-section, name: "Trombones a3" }
    - { id: tba-solo, name: "Tuba" }
  percussion:
    - { id: timp, name: "Timpani" }
    # Note: No keyboard percussion, harp, or piano
```

**Fallback chains**: Users can configure fallback chains between libraries:

```yaml
# User configuration: fallback-chains.yaml
fallback_chains:
  - primary: spitfire/bbcso-discover
    fallbacks:
      - sfz/virtual-playing-orchestra
```

When a score requires contrabassoon and the primary library lacks it, the playback engine consults the fallback chain and uses VPO's contrabassoon. This formalises the pattern users already employ manually: "BBCSO Discover for sound quality, VPO for coverage."

**Instrument identity mapping**: Different libraries use different instrument identifiers. The manifest enables mapping:

```yaml
# In VPO manifest
instruments:
  woodwinds:
    - { id: cbn-solo, name: "Contrabassoon", aliases: [contrabassoon, contra-bassoon, kontrafagott] }
```

The playback engine matches score instrument requirements against available instruments using both primary IDs and aliases.

## Bundled OVID Files

Ooloi bundles OVID files (YAML definitions) for two recommended free sample libraries. Users must separately download and install the actual libraries:

**BBC Symphony Orchestra Discover** (free download from Spitfire Audio): High-quality recordings in a small footprint (~700MB download). The bundled OVID files cover all 34 instruments with calibration data and articulation mappings. Recommended as the primary library for users beginning with orchestral playback.

**Virtual Playing Orchestra** (free SFZ library): Comprehensive coverage including instruments absent from BBCSO Discover. The bundled OVID files provide fallback coverage for contrabassoon, bass clarinet, harp, keyboard percussion, and auxiliary instruments.

Once users install both libraries, complete orchestral playback works immediately at zero cost. Users begin with BBCSO Discover for superior sound quality, with VPO configured as a fallback chain for missing instruments. As users acquire additional commercial libraries, they can add community-contributed or vendor-provided OVID files and reconfigure their fallback preferences.

The bundled OVID files serve as reference implementations demonstrating OVID conventions and calibration procedures.

## Reference Implementations

### Spitfire BBCSO Discover — Violins 1

```yaml
library: spitfire/bbcso-discover
player: spitfire
plugin:
  format: VST3
  identifier: "Spitfire BBC Symphony Orchestra"
  version: ">=1.6"

instrument:
  id: "vl1-section"
  name: "Violins 1"
  family: strings
  range: [G3, C#7]

calibration:
  tuning_hz: 440.0
  loudness_target_lufs: -23.0
  gain_db: 0.0
  mic: { preset: "Mix 1" }
  onset_defaults_ms:
    legato: -70
    long: -30
    short: 0

controls:
  - { id: dynamics, map: { type: param, name: "Dynamics" } }
  - { id: expression, map: { type: param, name: "Expression" } }
  - { id: reverb, map: { type: param, name: "Reverb" } }
  - { id: tightness, map: { type: param, name: "Tightness" } }

articulations:
  - tech: long
    native: true
    select:
      - { type: param, name: "Technique", value: "Long" }
      - { type: keyswitch, pitch: "C-2" }
    onset_ms: -25

  - tech: spiccato
    native: true
    select:
      - { type: param, name: "Technique", value: "Spiccato/Staccato" }
      - { type: keyswitch, pitch: "C#-2" }
    controls: { tightness: 0.6 }
    onset_ms: 0

  - tech: pizzicato
    native: true
    select:
      - { type: param, name: "Technique", value: "Pizzicato" }
      - { type: keyswitch, pitch: "D-2" }

  - tech: tremolo
    native: true
    select:
      - { type: param, name: "Technique", value: "Tremolo" }
      - { type: keyswitch, pitch: "D#-2" }

  - tech: trill_half
    native: false
    fallback:
      generator: trill
      interval: m2
      rate_hz_by_dynamic: { p: 8, mf: 10, ff: 12 }

  - tech: trill_whole
    native: false
    fallback:
      generator: trill
      interval: M2
      rate_hz_by_dynamic: { p: 8, mf: 10, ff: 12 }

  - tech: legato_normal
    native: false
    fallback:
      generator: none

  - tech: legato_portamento
    native: false
    fallback:
      generator: portamento_glide
      time_ms: 140
      curve: "ease-in-out"
```

### VSL Synchron Elite Strings — Violins

```yaml
library: vsl/synchron-elite-strings
player: synchron
plugin:
  format: VST3
  identifier: "Vienna Synchron Player"
  version: ">=1.3"

instrument:
  id: "vl1-section"
  name: "Violins"
  family: strings
  range: [G3, C#7]

calibration:
  tuning_hz: 440.0
  loudness_target_lufs: -23.0
  gain_db: +1.5
  mic:
    mix:
      - { name: "Close", gain_db: -3 }
      - { name: "Main/Decca", gain_db: 0 }
  onset_defaults_ms:
    legato: -80
    long: -35
    short: -5

controls:
  - { id: dynamics, map: { type: param, name: "Velocity X-Fade" } }
  - { id: expression, map: { type: param, name: "Volume" } }
  - { id: start_offset, map: { type: param, name: "Start Offset" } }
  - { id: humanize, map: { type: param, name: "Humanize" } }

articulations:
  - tech: long
    native: true
    select:
      - { type: dimension, path: ["Type", "Sustain"] }
      - { type: keyswitch, pitch: "C-2" }
    onset_ms: -30

  - tech: tremolo
    native: true
    select:
      - { type: dimension, path: ["Type", "Tremolo"] }

  - tech: trill_half
    native: true
    select:
      - { type: dimension, path: ["Type", "Trills", "m2"] }

  - tech: trill_whole
    native: true
    select:
      - { type: dimension, path: ["Type", "Trills", "M2"] }

  - tech: legato_normal
    native: true
    select:
      - { type: dimension, path: ["Legato", "Normal"] }

  - tech: legato_portamento
    native: true
    select:
      - { type: dimension, path: ["Legato", "Portamento"] }

  - tech: col_legno_battuto
    native: true
    select:
      - { type: dimension, path: ["Type", "Col Legno Battuto"] }

  - tech: pizzicato
    native: true
    select:
      - { type: dimension, path: ["Type", "Pizzicato"] }

  - tech: spiccato
    native: true
    select:
      - { type: dimension, path: ["Type", "Spiccato"] }

  - tech: flutter_tongue
    native: false
    applies_to: [woodwinds, brass]
    fallback:
      generator: none
```

## Rationale

### Why YAML

**Human-readable**: Musicians and developers can inspect and modify OVID files without specialised tooling.

**Version-controllable**: Plain text enables standard Git workflows for tracking library updates, community contributions, and vendor version changes.

**Community-contributable**: Lower barrier to contribution than code. A user with a library can document its capabilities without understanding playback engine internals.

**Tooling ecosystem**: Widespread YAML support in editors, validators, and programming languages.

**Proven precedent**: Igor Engraver's Synth Matrices demonstrated that user-manageable text files describing device capabilities enable community support for diverse hardware without centralised development. OVID continues this architecture with a modern format.

### Why Canonical Taxonomy

**Abstraction over chaos**: Vendors use inconsistent terminology. Spitfire's "Long" is Vienna's "Sustain" is Orchestral Tools' "Long Notes." The taxonomy provides a stable interface.

**Notation semantics linkage**: Canonical names map to notation concepts. The playback engine translates a tremolo marking to `tremolo`; the OVID file translates `tremolo` to vendor-specific selection.

**Extensibility**: New techniques add to the taxonomy without breaking existing OVID files.

### Why Mandatory Calibration

**Predictable cross-library results**: Without calibration, combining libraries produces inconsistent balance and timing. Mandatory calibration ensures playback quality regardless of which libraries are installed.

**Measurable quality baseline**: Calibration data enables automated testing. A harness can verify that playback meets loudness and timing specifications.

**Reduced user burden**: Users should not need to manually adjust gain and delay compensation per library. OVID calibration data automates this.

### Why Host Control Over External MIDI

**Scope**: This section addresses playback control of virtual instruments, not note entry (MIDI input from keyboards) and not MIDI output to external hardware. MIDI input for note entry is accepted by the frontend and translated to API calls. MIDI output to external synthesisers is not provided by OVID; it would require a separate plugin.

**No MIDI output**: Ooloi does not output MIDI. Playback goes to virtual instruments via host control mechanisms (parameter automation, host-internal event delivery). This eliminates the configuration burden of OS-level MIDI routing.

**Higher resolution**: Plugin parameters offer greater precision than MIDI's 7-bit controllers.

**No channel limits**: MIDI's 16-channel ceiling required Igor's elaborate allocation algorithms. Host control to virtual instruments has no such constraint.

**Per-note expression**: Where plugins support it (VST3's INoteExpressionController, MPE-capable plugins), tuning/volume/timbre can vary per note—difficult or impossible with traditional channel-wide MIDI.

**Articulation discovery**: Some plugin formats allow hosts to query available articulations. This enables validation and potentially automatic mapping.

**MIDI output remains possible via plugins**: Nothing prevents creation of a MIDI output plugin for users who need external hardware control. Such a plugin could implement simple fixed-channel allocation or sophisticated Igor-style dynamic routing. This is outside OVID scope.

## Challenges Addressed

### Vendor Fragmentation

OVID files isolate vendor-specific knowledge. The playback engine works with canonical techniques; OVID translates to vendor implementations. Adding support for a new library requires only an OVID file, not engine modifications.

### Version Drift

Libraries update, changing parameter names and keyswitch assignments. OVID files can specify minimum versions and be updated independently of the playback engine. Version tracking fields (`library_version`, `content_revision`) enable managing this evolution.

### Missing Articulations

Fallback generators provide graceful degradation. A score using trills plays correctly whether the library has native trills (using them) or not (synthesising them). The user experience remains consistent.

### Missing Instruments

Library manifests and fallback chains address instrument-level absence. A score requiring contrabassoon plays correctly whether the primary library includes it (using the primary) or not (falling back to an alternative library). Users configure their preferred fallback order; the playback engine handles routing transparently.

### Cross-Library Consistency

Calibration data normalises loudness and timing. A project using Spitfire strings with Vienna brass produces balanced, rhythmically aligned playback without manual adjustment.

## Integration Points

### ADR-0027: Plugin-Based Audio Architecture

OVID operates entirely within the plugin layer established by ADR-0027. It specifies how playback plugins control virtual instruments, not how the core handles audio (which it does not).

### ADR-0003: Plugin Architecture

OVID files are resources consumed by playback plugins. The plugin architecture's resource management, versioning, and distribution mechanisms apply. Community-contributed OVID files can be distributed through the plugin ecosystem.

### Notation Semantics in Core

The core's semantic representation of articulations (staccato dots, tremolo strokes, trill markings) maps to canonical technique names. This mapping is the interface between notation semantics and playback realisation.

### Future: Humanisation and Performance Interpretation

OVID controls *what* sounds play. A future humanisation specification will control *how* they play—tempo variation, dynamic shaping, rubato, ensemble timing. Humanisation plugins will consume OVID-controlled instruments, adding interpretation layers. OVID must be stable before humanisation can be specified.

## Alternative Approaches Considered

### Per-Plugin Hardcoded Mappings

**Approach**: Each playback plugin implements vendor-specific control code directly.

**Rejection**: Does not scale. Every new library requires plugin code changes. Community contribution requires programming expertise. No separation between "what exists" (OVID files) and "how to use it" (engine logic).

### MIDI-Based Control

**Approach**: Use standard MIDI for articulation selection and expression.

**Rejection**: ADR-0027's reasoning applies. MIDI's channel limits, controller resolution, and external routing complexity are the constraints Igor Engraver worked around. Modern host control supersedes these workarounds.

### Vendor-Specific APIs Only

**Approach**: Implement against each vendor's API directly without abstraction.

**Rejection**: No interoperability. Users cannot mix libraries without understanding each API. No common vocabulary for articulation selection. Playback code becomes unmaintainable.

### No Specification

**Approach**: Let plugin developers implement control however they choose.

**Rejection**: Inconsistent user experience. Quality varies by plugin author. No community contribution path. Each plugin reinvents library support.

## Success Criteria

These criteria represent goals for a mature implementation rather than acceptance thresholds. Actual results depend on plugin behaviour, library characteristics, and measurement conditions.

### Technical Goals

- **Cross-library loudness consistency**: Target ±2-3 dB perceived balance across calibrated libraries under typical conditions. Exact LUFS deviation depends on measurement programme, mic position, and dynamic layer.
- **Onset alignment**: Improved rhythmic placement compared to uncalibrated playback. Static offsets provide useful approximation; complex legato transitions may require future refinement.
- **Selection reliability**: Articulation selection should succeed when the target articulation exists and the OVID file correctly describes the selection mechanism.
- **Fallback quality**: Synthesised articulations (trills, tremolo, portamento) should be musically acceptable for playback purposes, recognising they cannot match native samples in all contexts.

### Ecosystem Goals

- **Bundled library functionality**: BBCSO Discover + VPO fallback chain provides playback for standard orchestral scores without additional configuration
- **Library coverage**: OVID files available for major orchestral libraries (BBCSO, Synchron, Berlin, Hollywood, SWAM)
- **Community contribution**: Third-party OVID files accepted and maintained
- **Notation coverage**: Most standard articulation markings mapped to canonical techniques
- **Fallback chain reliability**: Instrument routing between libraries transparent to user; mixed-library playback maintains reasonable calibration consistency

## Future Extensions

**Calibration harness**: Automated tool that loads instruments, plays reference material, measures loudness and onset timing, and generates calibration values for OVID files.

**Community registry**: Central repository for OVID files with version tracking, validation, and search by library/instrument/technique.

**Extended taxonomy**: Contemporary extended techniques, non-Western instruments, electronic/synthesis articulations.

**Humanisation integration points**: Defined interfaces for humanisation plugins to modify OVID-controlled playback.

**Machine learning assistance**: ML-based calibration refinement using perceptual metrics rather than purely technical measurement.

## Related Guides

- [ADR-0044: MIDI Input Library and Boundary Architecture](0044-MIDI-Input-Library-and-Boundary-Architecture) — Confirms that MIDI output is rejected for the core input subsystem; plugin extension point for MIDI output described
- [MIDI in Ooloi](../guides/MIDI_IN_OOLOI.md) — Why the core produces no MIDI output (playback delegates to host parameter automation via OVID instead), and how a plugin requiring MIDI output would be built using the frontend event bus as the integration point

## Conclusion

OVID provides the abstraction layer between Ooloi's notation semantics and the fragmented landscape of virtual instrument control. By defining capabilities declaratively in human-readable files, using a canonical technique taxonomy, requiring calibration data, specifying fallback generators, and enabling library manifests with fallback chains, OVID enables consistent, high-quality playback across diverse libraries without vendor-specific code in the playback engine.

Ooloi bundles OVID files for BBC Symphony Orchestra Discover and Virtual Playing Orchestra. Once users download and install these free libraries, complete orchestral playback works immediately at zero cost. As users acquire additional commercial libraries, the same infrastructure scales to support professional-grade sample collections.

The specification operates entirely within the plugin architecture established by ADR-0027. It addresses the immediate requirement—controlling virtual instruments—while establishing a foundation for future humanisation and performance interpretation specifications.

OVID embodies Ooloi's architectural philosophy: solve the problem correctly at the right layer, use modern capabilities rather than legacy workarounds, and enable community contribution through well-defined interfaces.

## Appendix A: Vendor Control Surface Research

This appendix documents the control mechanisms available for major virtual instrument platforms, providing the empirical foundation for OVID design decisions. The research focuses primarily on VST3 implementations; AU behaviour may differ in specifics while providing equivalent capabilities.

### VST3-Specific Interfaces

These interfaces are VST3-specific. AU provides different mechanisms for similar capabilities; OVID abstracts over format differences at the definition level.

**Parameter automation**: All VST3 plugins expose automatable parameters. OVID leverages this for articulation selection and expression control.

**IKeyswitchController**: VST3 interface that advertises a plugin's available articulations/keyswitches to the host, enabling technique selection without external MIDI routing. Not all plugins implement this interface.

**INoteExpressionController**: Provides per-note tuning, volume, and timbre control where implemented. Essential for microtuning and per-note expression shaping. Support varies by plugin.

### Spitfire Audio (BBCSO, Abbey Road, Symphonic)

**Technique selection**: Via Technique Selector panel with host automation support. Setting "Disable Host Automation" to off enables parameter control.

**Keyswitch access**: Movable keyswitches accessible via VST3 event bus, not requiring external MIDI devices.

**Tightness/Attack**: Automatable parameters for adjusting sample start points, particularly useful for tightening short articulations.

**Typical coverage**: Longs, shorts (staccato/spiccato), pizzicato, tremolo. Higher editions add trills, legatos, additional articulations.

### Orchestral Tools (SINE)

**Articulation Lists**: Managed via keyswitch, CC, or program change. SINE exposes these through standard host mechanisms.

**Tightness**: Sample-start control in Articulation Options, automatable.

**Legato options**: Multiple legato modes available in Articulation Options.

**Berlin Strings coverage**: Multiple legato modes, tempo-synced runs, natural volume balancing, comprehensive articulation sets.

### Vienna Symphonic Library (Synchron Player)

**Dimension Trees**: Hierarchical articulation organisation with Remote Control via keyswitch, CC, program change, or direct parameter automation.

**Start Offset**: Critical for onset alignment, automatable on Perform tab.

**Humanize, Time-Stretch**: Additional automatable parameters for performance shaping.

**Synchron Elite Strings coverage**: Trills, tremolo, portamento legato, col legno (battuto), snap pizzicato, comprehensive articulation sets.

### EastWest (Opus)

**Multi-articulation instruments**: Articulation switching via keyswitches or Program Change (addressable by index without external MIDI routing).

**Performance controls**: Automatable in Opus engine.

**Hollywood coverage**: Flutter-tongue, trills, comprehensive woodwind and brass articulations with dynamic control.

### Audio Modeling (SWAM)

**Fully parameter-driven**: Dynamics, expression, vibrato, portamento time, growl, and other controls exposed as automatable plugin parameters.

**MPE support**: Optional MPE for per-note expression where desired.

**Ideal OVID fit**: Direct parameter control without keyswitch complexity.

### SFZ Players (VPO, etc.)

**Switch opcodes**: Articulations defined via `sw_last`, `sw_default`, etc.

**Limited host parameters**: Most SFZ players expose few automatable parameters; selection primarily via note events delivered through the host to the plugin.

**OVID approach**: Use event-based selection within the plugin host. No MIDI output involved.

## References

- [ADR-0027: Plugin-Based Audio Architecture](0027-Plugin-Based-Audio-Architecture.md) — Establishes audio belongs in plugins
- [ADR-0003: Plugin Architecture](0003-Plugin-Architecture.md) — Plugin system design
- Igor Engraver Synth Matrices (1996-2001) — Precursor device description architecture
- [EBU R128](https://tech.ebu.ch/docs/r/r128.pdf) — Loudness normalisation recommendation
- [ITU-R BS.1770](https://www.itu.int/rec/R-REC-BS.1770) — Loudness measurement algorithms
- [VST3 SDK Documentation](https://steinbergmedia.github.io/vst3_doc/) — IKeyswitchController, INoteExpressionController
- [BBC Symphony Orchestra Discover](https://www.spitfireaudio.com/bbc-symphony-orchestra-discover) — Free library with bundled OVID files (primary)
- [Virtual Playing Orchestra](https://virtualplaying.com/virtual-playing-orchestra/) — Free library with bundled OVID files (fallback coverage)
- Spitfire Audio user manuals — Technique Selector, host automation, Tightness controls
- Vienna Synchron Player manual — Dimension Trees, Remote Control, Start Offset
- Orchestral Tools SINE documentation — Articulation Options, Tightness
- EastWest Opus documentation — Program Change articulation selection