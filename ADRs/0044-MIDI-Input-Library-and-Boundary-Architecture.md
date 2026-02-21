# ADR-0044: MIDI Input Library and Boundary Architecture

**Status:** Accepted
**Date:** 2026-02-21

---

## Table of Contents

- [Context](#context)
- [Decisions](#decisions)
  - [1. javax.sound.midi as the Single MIDI API](#1-javaxsoundmidi-as-the-single-midi-api)
  - [2. CoreMidi4J as macOS SPI Provider](#2-coremidi4j-as-macos-spi-provider)
  - [3. Hard Input Boundary Filter as Architectural Invariant](#3-hard-input-boundary-filter-as-architectural-invariant)
- [Alternatives Considered](#alternatives-considered)
  - [RtMidi via JNI](#rtmidi-via-jni)
  - [Platform Conditionals in Application Code](#platform-conditionals-in-application-code)
  - [Permissive Input Filtering](#permissive-input-filtering)
  - [Built-in MIDI Output](#built-in-midi-output)
- [Consequences](#consequences)
- [Related ADRs](#related-adrs)

---

## Context

[ADR-0027](0027-Plugin-Based-Audio-Architecture.md) establishes that the Ooloi core
provides no audio or MIDI output. [ADR-0032](0032-Flow-Mode.md) establishes that MIDI
keyboard input is used for pitch entry in Flow Mode. This ADR records the decisions
governing how that MIDI input is received, on which platform APIs it rests, and what
the system accepts at its input boundary.

The scope is narrow and deliberate: MIDI input only, for the purpose of note entry.
No MIDI output, no clock, no SysEx processing, no general MIDI routing. These
constraints are not implementation details — they are the decisions this ADR
documents.

---

## Decisions

### 1. `javax.sound.midi` as the Single MIDI API

Ooloi uses `javax.sound.midi` as its MIDI API on all platforms. Application code
calls only this interface — no platform-specific MIDI code appears anywhere in the
codebase.

Platform differences are handled at the SPI (Service Provider Interface) layer, not
in application code. On macOS, CoreMidi4J registers as the SPI provider and answers
`javax.sound.midi` calls in place of the JDK bridge. On Windows and Linux, the JDK's
own providers answer those same calls. From the application's perspective, the API is
identical everywhere.

This is not merely a convenience. It is the architectural invariant that makes
platform differences invisible above the SPI layer. Any future platform adaptation —
a new JVM target, a corrected JDK provider, a different native library — is handled
by registering a new SPI provider, not by branching application code.

### 2. CoreMidi4J as macOS SPI Provider

On macOS, the JDK's built-in CoreMIDI bridge has documented deficiencies: unreliable
hot-plug notification, SysEx handling problems, and thread-safety issues under load.
These are not hypothetical — they are known failure modes that affect MIDI input
reliability in practice.

CoreMidi4J addresses these deficiencies by replacing the JDK bridge at the SPI layer.
It registers via `META-INF/services/javax.sound.midi.spi.MidiDeviceProvider`, making
the replacement transparent. Application code is unaffected. On Windows and Linux,
CoreMidi4J is present on the classpath but inactive — the JDK providers take
precedence, and the library contributes nothing and costs nothing.

CoreMidi4J is added as a dependency in both `frontend/project.clj` and
`shared/project.clj`. The frontend project does not produce a standalone application —
it retains the dependency for development and testing. The shared dependency covers
the combined application build, which is the actual desktop artifact. No conditional
loading, no platform detection, no application-level branching.

### 3. Hard Input Boundary Filter as Architectural Invariant

The MIDI `Receiver` implementation accepts exactly three message types:

- `NOTE_ON` with velocity > 0 — note entry
- `NOTE_ON` with velocity = 0 — treated as `NOTE_OFF` (standard MIDI convention)
- `NOTE_OFF` — note release
- `Control Change CC64` — sustain pedal

Everything else is discarded silently before it reaches any other part of the system.
SysEx, clock, program change, pitch bend, all other control changes — discarded at
the boundary, unconditionally, without logging.

This is an architectural invariant, not a filtering heuristic. The JVM MIDI stack has
known crash vectors around malformed SysEx messages and unexpected message types from
misbehaving devices. By discarding everything outside the accepted set before it
touches anything, these crash vectors are eliminated structurally — not mitigated by
error handling downstream.

The boundary is also the definition of scope. MIDI input in Ooloi exists for one
purpose: note entry in Flow Mode. The accepted message set is exactly what that
purpose requires. A plugin that needs additional CC messages, or any other MIDI data,
implements its own `Receiver` and its own boundary logic. The core's boundary does
not expand to accommodate plugin requirements.

---

## Alternatives Considered

### RtMidi via JNI

RtMidi is a cross-platform C++ MIDI library with Java bindings via JNI. It addresses
the same macOS deficiencies as CoreMidi4J and provides a unified API across platforms.

**Rejected because:** For an input-only use case, the JNI surface area is unnecessary
overhead. CoreMidi4J solves the macOS problem as a pure SPI replacement, with no
native code exposed to application code and no JNI layer to maintain. RtMidi's
advantages (broader platform support, output capability) are irrelevant to this
scope. The lighter solution is the correct one when it is sufficient.

### Platform Conditionals in Application Code

Detecting the platform at runtime and calling platform-specific MIDI APIs directly —
CoreMIDI on macOS, Windows Multimedia on Windows, ALSA on Linux.

**Rejected because:** This would require three separate code paths, each with its own
failure modes and maintenance burden. The `javax.sound.midi` + SPI approach is
precisely designed to avoid this: platform differences belong at the provider layer,
not in application logic. Introducing platform conditionals would undermine the
architectural principle that makes the SPI mechanism valuable.

### Permissive Input Filtering

Accepting all `ShortMessage` types and passing them to the event bus, letting
downstream consumers filter what they need.

**Rejected because:** The known crash vectors in the JVM MIDI stack are in message
reception, not in downstream processing. SysEx messages in particular — which are not
`ShortMessage` instances — can cause failures if not handled correctly. Permissive
filtering defers the problem rather than eliminating it. More fundamentally, a
permissive boundary would invite scope expansion: once the bus carries arbitrary MIDI
messages, other subsystems are tempted to consume them, and the narrow purpose of the
input subsystem erodes. The hard boundary enforces scope structurally.

### Built-in MIDI Output

Including MIDI output capability in the core input subsystem, or providing it as a
core feature alongside input.

**Rejected by ADR-0027**, which establishes that the core provides no MIDI output.
Restated here for completeness: playback in Ooloi operates through direct host
parameter control via OVID ([ADR-0041](0041-Ooloi-Virtual-Instrument-Definition-OVID.md)),
not through MIDI routing. MIDI output to external devices is a plugin concern. The
core input subsystem has no output responsibility.

---

## Consequences

**Single API surface.** All MIDI interaction in application code goes through
`javax.sound.midi`. Future developers encounter one API regardless of platform.
Platform-specific behaviour is invisible above the SPI layer.

**macOS reliability.** CoreMidi4J's SPI replacement eliminates the JDK bridge's
known deficiencies on macOS. Hot-plug, SysEx handling, and thread safety behave
correctly without application-level workarounds.

**Structural crash elimination.** The hard input boundary removes entire classes of
JVM MIDI edge-case failures. Malformed or unexpected messages from misbehaving devices
cannot reach the event bus or any downstream system.

**Scope enforcement.** The accepted message set defines the scope of MIDI input in
the core. Extensions require plugins, not modifications to the core boundary. This
makes scope expansion visible and deliberate.

**Plugin extension point.** A plugin requiring additional MIDI data implements its
own `Receiver` with its own boundary policy and publishes to the frontend event bus
([ADR-0031](0031-Frontend-Event-Driven-Architecture.md)). The core's boundary does
not need to change.

**Shutdown latency.** The hot-plug polling loop (3-second interval) means component
shutdown may wait up to 3 seconds for the polling thread to exit. This is accepted.

---

## Related ADRs

- [ADR-0027: Plugin-Based Audio Architecture](0027-Plugin-Based-Audio-Architecture.md) —
  Core provides no audio or MIDI output; all audio delegated to frontend plugins
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) —
  Event bus that accepted MIDI messages are published to
- [ADR-0032: Flow Mode](0032-Flow-Mode.md) —
  The input paradigm that consumes MIDI pitch events; context for the accepted message set
- [ADR-0041: Ooloi Virtual Instrument Definition (OVID)](0041-Ooloi-Virtual-Instrument-Definition-OVID.md) —
  Why playback uses host control rather than MIDI output
- [ADR-0043: Frontend Settings](0043-Frontend-Settings.md) —
  Device selection persistence via `def-app-setting`