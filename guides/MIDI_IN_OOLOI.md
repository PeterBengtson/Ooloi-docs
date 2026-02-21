# MIDI in Ooloi: Input, Not Output

Ooloi's relationship with MIDI is deliberately asymmetric. The system accepts MIDI
input from keyboards and controllers for note entry. It produces no MIDI output
whatsoever in its unextended form. This is not an omission — it is a decision, and
understanding it clarifies several things about how the system is structured.

---

## Why No MIDI Output

The question arises naturally: if Ooloi supports playback, why doesn't it output
MIDI to drive external synthesisers or software instruments?

The answer is that playback operates at a layer above MIDI. Ooloi's playback plugins
control virtual instruments directly through host parameter automation — the same
mechanism a DAW uses when it moves a fader or selects an articulation on a loaded
instrument. This approach, specified in [ADR-0041 (OVID)](ADRs/0041-Ooloi-Virtual-Instrument-Definition-OVID.md),
gives the playback engine per-note tuning, continuous parameter control, and direct
articulation selection without the constraints MIDI imposes.

Those constraints are not trivial. Igor Engraver — Ooloi's predecessor — achieved
sophisticated microtonal playback by distributing notes across MIDI channels using
register allocation algorithms borrowed from compiler technology: a "DNA soup" of
channel management to squeeze capability out of a 16-channel, 7-bit protocol. Modern
host control eliminates the need for those workarounds. A plugin can tune each note
independently, select articulations through VST3's `IKeyswitchController`, and shape
expression through `INoteExpressionController` — without routing a single byte
through OS-level MIDI.

There is also the question of where playback lives. Ooloi's backend runs as a server —
potentially in the cloud, certainly without audio hardware. All audio processing
happens on the frontend client, which hosts playback plugins locally
([ADR-0027](ADRs/0027-Plugin-Based-Audio-Architecture.md)). MIDI output to external
devices is a client-side concern by definition. If a user needs MIDI output — to drive
a hardware synthesiser, or to route into a DAW — a plugin can provide it. The core
system does not, because the core system's playback requirements are already better
served by direct host control.

---

## What MIDI Input Is For

MIDI input serves one purpose: note entry in Flow Mode.

Flow Mode ([ADR-0032](ADRs/0032-Flow-Mode.md)) is Ooloi's primary input method — a
stateful modal keyboard paradigm where the computer keyboard governs rhythm and
articulation while a MIDI keyboard provides pitch. The combination is fast in a way
neither instrument alone achieves: both hands are productive simultaneously, and the
physical act of playing a pitch on a keyboard is more natural for most musicians than
pressing cursor keys.

The integration is narrow by design. Ooloi accepts:

- **NOTE_ON / NOTE_OFF** — pitch and velocity for note entry
- **NOTE_ON with velocity 0** — treated as NOTE_OFF (standard MIDI convention)
- **Control Change 64** — the sustain pedal, for chord entry or sustain semantics

Everything else is discarded at the input boundary, silently, before it can reach
anything. SysEx, clock, program change, pitch bend, all other control changes — none
of it enters the system. This hard boundary eliminates entire classes of JVM MIDI
edge-case crashes by ensuring no SysEx or malformed message data ever enters the
system. This is not a limitation of the implementation; it is the implementation. MIDI input is a narrow pipe for a specific purpose, not a general
MIDI processing subsystem.

---

## The Technical Foundation

Ooloi uses `javax.sound.midi` as its MIDI API on all platforms. This is the standard
Java MIDI interface — cross-platform, no native dependencies in application code, and
sufficient for input-only use.

On macOS, the JDK's built-in CoreMIDI bridge has historically suffered from hot-plug
unreliability, SysEx handling problems, and thread-safety issues under load.
[CoreMidi4J](https://github.com/DerekCook/CoreMidi4J) addresses this by registering
as a Java SPI (Service Provider Interface) replacement for the macOS MIDI provider.
Application code calls `javax.sound.midi` as normal; on macOS, CoreMidi4J answers
instead of the JDK bridge. On Windows and Linux, the JDK's own providers handle
everything. On modern Linux distributions (ALSA/PipeWire), input-only use is stable
and requires no additional configuration. No platform conditionals appear in
application code.

The alternative — a native JNI library like RtMidi — would solve the same macOS
problem with considerably more native surface area. For an input-only use case,
the tradeoff is straightforward: CoreMidi4J's SPI approach is lighter, less
invasive, and sufficient.

---

## How Events Flow

A MIDI message that passes the input filter becomes a map and is published to the
frontend event bus ([ADR-0031](ADRs/0031-Frontend-Event-Driven-Architecture.md)) under
the `:midi` category. The Receiver implementation — the only code that touches
`javax.sound.midi` objects directly — does nothing else:

```clojure
(reify Receiver
  (send [_ midi-message _timestamp]
    (when (instance? ShortMessage midi-message)
      (when-let [event (handle-message midi-message)]
        (eb/publish! event-bus :midi [event]))))
  (close [_] nil))
```

The filter runs first. The event bus call is non-blocking. The MIDI callback thread
never touches the JavaFX scene graph. UI updates, if any, happen on the JavaFX
Application Thread via the normal `fx/run-later!` discipline, downstream of the
event bus — not here.

Event shapes are simple:

```clojure
;; Note entry
{:type :midi/note  :on? true  :pitch 60  :velocity 80  :channel 0}

;; Sustain pedal
{:type :midi/sustain  :down? true  :value 127}
```

Flow Mode subscribes to the `:midi` category and translates these events into gRPC
API calls. The backend never sees MIDI data — it receives musical commands, the same
commands that Click Mode produces through mouse interaction. This preserves the
backend's transport-agnostic contract: musical intent, not input modality, crosses
the gRPC boundary.

---

## Device Management

Ooloi detects MIDI input devices at startup, persists the user's selection across
launches, and rescans periodically to handle devices connected or disconnected after
startup.

The selected device is stored as a frontend app setting — `:midi/input-device-id` —
under the settings system described in [ADR-0043](ADRs/0043-Frontend-Settings.md).
The settings dialog auto-generates a dropdown populated with currently available
physical input devices. Virtual and loopback ports are excluded by heuristic (names
containing "Virtual", "Through", "IAC", "Bus", and similar markers). When only one
unambiguous physical device is present and no preference has been saved, it is
selected automatically.

If the selected device disappears — disconnected mid-session — the preference is
retained and a disconnected status is published on the event bus. When the device
reappears, connection is restored automatically.

---

## Extension Points

A plugin that requires MIDI output — to drive hardware, or for any other purpose —
can implement its own MIDI sender. The frontend event bus is the natural integration
point: subscribe to the relevant event categories, translate to MIDI, send via
`javax.sound.midi.MidiSystem`. The core system's input infrastructure does not need
to be involved.

A plugin that requires additional MIDI control change messages beyond CC64 — modulation,
expression, breath, or any other — would implement its own `Receiver` and publish the
relevant events on the bus. Extending the input surface beyond the core's narrow
scope is a plugin decision, not a core decision.

The core stays narrow because note entry is the only thing MIDI input needs to do.

---

## Related ADRs

- [ADR-0027: Plugin-Based Audio Architecture](ADRs/0027-Plugin-Based-Audio-Architecture.md) —
  Why the core has no audio or MIDI output; all playback delegated to frontend plugins
- [ADR-0031: Frontend Event-Driven Architecture](ADRs/0031-Frontend-Event-Driven-Architecture.md) —
  The event bus that MIDI events flow through
- [ADR-0032: Flow Mode](ADRs/0032-Flow-Mode.md) —
  The input paradigm that consumes MIDI pitch events
- [ADR-0041: Ooloi Virtual Instrument Definition (OVID)](ADRs/0041-Ooloi-Virtual-Instrument-Definition-OVID.md) —
  How playback plugins control virtual instruments without MIDI output
- [ADR-0043: Frontend Settings](ADRs/0043-Frontend-Settings.md) —
  Device selection persistence and the settings system
- [ADR-0044: MIDI Input Library and Boundary Architecture](ADRs/0044-MIDI-Input-Library-and-Boundary-Architecture.md) —
  The formal decisions behind this guide: `javax.sound.midi`, CoreMidi4J, and the hard input boundary filter