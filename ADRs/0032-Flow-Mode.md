# ADR-0032: Ooloi Flow Mode

**Status:** ACCEPTED
**Date:** 2025-10-20
**Phase:** Phase 14 – Input System

---

## Context

### The Lost Paradigm

Igor Engraver (1996-2009) pioneered a revolutionary input method: **stateful modal keyboard entry with real-time WYSIWYG feedback**. Unlike modern notation software that requires explicit tool selection or popover invocation for every operation, Igor's Flow Mode used persistent modal states where typing `.` would activate staccato mode that remained active across multiple note entries until explicitly changed.

**Core characteristics:**
- **Persistent modal states**: Type `ffzpp<` → creates that exact dynamic with crescendo start
- **Expanding elements**: Slurs and crescendos automatically extend as subsequent notes are added
- **Three-stage workflow**: Position cursor → Select pitch → Press Enter to create
- **Combinatorial articulations**: `>` + `.` = accent + staccato stacked on same note
- **Intuitive commands**: Musical symbols map naturally to keystrokes – low learning curve
- **Real-time WYSIWYG**: Every keystroke updates visual score immediately (not compile-based like LilyPond)
- **MIDI integration**: Pitch entry via MIDI keyboard while computer keyboard handles rhythm/articulation

### What Happened to Flow Mode

Igor Engraver died due to business failures (management-imposed feature creep, venture capital pressures, 9/11's economic impact halting acquisitions) – not technical inadequacy. The software was praised as revolutionary by early adopters.

**Research conducted October 2025** (documented in `research/OOLOI_FLOW_MODE.md`) reveals a stunning finding:

**No music notation software has implemented Igor's modal paradigm in 23 years (2001-2025).**

Comprehensive research across 532 sources confirms:
- **Dorico's popovers**: Explicitly non-persistent, opposite philosophy
- **Sibelius keypad**: Tool-switching, not modal states
- **Finale Simple Entry**: Limited tool persistence, not true stateful modes
- **Denemo**: Vim-inspired but non-WYSIWYG, different paradigm entirely
- **All other software**: Palette-based, touch/handwriting, or search-driven interfaces

**Academic gap**: Zero papers document Igor's modal paradigm – the knowledge effectively died with the company (2001-2004) before academic attention turned to notation interfaces.

**Professional community gap**: Forums extensively discuss keyboard workflow optimization but show **zero requests for "modal," "stateful," or "persistent mode" features** – users lack the vocabulary to conceptualize this solution despite demonstrating behaviors that align with its benefits.

### Why the 23-Year Gap Exists

1. **Market timing**: Igor died just as Finale/Sibelius achieved dominance with GUI-based approaches
2. **Platform evolution**: 2010s mobile/web shift prioritized touch/gesture input incompatible with modal states
3. **Design philosophy**: Modern HCI emphasizes discoverability over efficiency (palettes vs. modes)
4. **Documentation loss**: Igor's implementation details never published academically or preserved
5. **Conceptual gap**: Musicians/engravers don't think in "modal editing" terms despite valuing the same benefits programmers get from vim/emacs

### The Two-Mode Reality

Modern notation software provides **only one input paradigm**: point-and-click with palette-based tool selection. This is functional but slow for professional users entering large amounts of notation daily.

Ooloi will support **two fundamentally different input modes**:

1. **Ooloi Click Mode** – Mouse-driven, palette-based, tool-switching (standard modern approach)
2. **Ooloi Flow Mode** – Modal, stateful keyboard entry with real-time WYSIWYG feedback (Igor's lost paradigm)

Click Mode is **simplistic but necessary** – it provides discoverability and works for casual users. Flow Mode is where professional speed and efficiency live.

---

## Decision

**Ooloi will implement Flow Mode to revive Igor Engraver's stateful modal keyboard input paradigm as a competitive differentiator.**

This is not innovation – it's **memory + implementation**. We are restoring a proven design that worked, users loved, and no competitor has copied in 23 years.

### Why This Decision Matters

**Genuine competitive advantage**: Research validates this is a **unique market position** in 2025. Not incremental improvement over competitors' versions – there are no competitors' versions to improve upon.

**Professional user need**: Composers, arrangers, engravers work with notation daily. Speed matters enormously. A 5-10x improvement in input speed is compelling enough to justify learning a new tool.

**Low learning curve**: Unlike vim's arbitrary commands, Flow Mode is **intuitive and self-evident**:
- Type `ffzpp<` → get that exact dynamic with crescendo
- Type `.` → staccato mode stays active
- Type `>` → accent mode stays active
- Press Enter → create note with all active modes

Musical notation itself suggests the keystrokes – no artificial memorization required.

---

## What We Remember (Partial Knowledge)

The following keyboard commands are confirmed from user memory of Igor Engraver:

**Articulations:**
- `.` – Staccato mode (remains active)
- `'` – Staccatissimo wedge mode (remains active)
- `-` – Tenuto mode (remains active)
- `>` – Accent mode (remains active or cycles through accent variants)
- Heavy accent mode (marcato – looks like A without crossbar)
- **Stacking rules**: Articulations can stack in pairs, triples, or more (with logical exclusions)
  - Triple combinations: marcato + tenuto + staccato, tenuto + staccato + accent
  - Pair combinations: `-` + `.`, `-` + `>`, `.` + `>`, `'` + `>`, heavy accent + most others
  - Mutually exclusive pairs: `.` ↔ `'` (staccato ↔ staccatissimo), `'` ↔ `-` (staccatissimo ↔ tenuto)
  - System renders combined glyphs with proper vertical spacing

**Structural Elements:**
- `T` – Ties next note
- `S` – Starts slur (expands as subsequent notes added)
- `<` – Starts crescendo (expands as you continue)
- `/` – Beam operations (cut and join beams at various levels)
- Tuplet mode toggle key (mode remains active)

**Dynamics:**
- Type literal text: `ffzpp` creates that exact dynamic marking
- Case sensitivity unknown (requires documentation)

**Text Entry:**
- `Space` – Enters lyrics for "current verse"
- Multiple verse switching mechanism unknown (requires documentation)

**Note Creation (Three-Stage Workflow):**
1. **Position**: Click in score to place cursor (no note created)
2. **Select**: Cursor Up/Down keys to choose pitch
3. **Create**: Enter key creates note at current pitch with all active modes
- `Del` key removes entry at cursor position

**Enharmonic Operations:**
- Cycle through all possible spellings (standard → esoteric order)
- Works on both single notes and entire chords

**Selection and Batch Operations:**
- `Shift + navigation` – Selects range of notes
- Articulations and slurs can apply to entire selected range as unit

**Visual Feedback:**
- Mode indicators showing what's currently toggled on
- Editing cursor positioned between notes or at entry points
- Automatic cursor advancement to next measure after entry
- Real-time visual feedback for all modal state changes

### Archaeology Required

**Critical gaps** requiring Igor Engraver documentation from Magnus Johansson:

1. **State machine details**: How long modes persist, how they exit, which modes can stack
2. **Complete key binding map**: All articulations, dynamics, structural elements, text entry
3. **Conflict resolution**: How `>` resolves between accent and crescendo
4. **MIDI integration specifics**: Velocity mapping, sustain pedal behavior, chord entry method
5. **Expanding element mechanics**: Slur start/end/extension, crescendo behavior, lyrics melismas
6. **Mode combination rules**: Which modes can coexist, visual feedback for stacked modes

Without this documentation, we must **reconstruct through archaeology** – examining any remaining Igor materials, interviewing former users, and designing missing pieces based on musical logic and ergonomic principles.

---

## Consequences

### Positive

**Unique competitive position**: 23-year market gap validated through comprehensive research. First-mover advantage in uncontested space.

**Professional speed gains**: 5-10x faster input than Click Mode for experienced users. Write music at the speed of thought.

**Low learning curve**: Intuitive, self-evident commands. Musical notation itself suggests keystrokes. Discovery through doing, not arbitrary memorization.

**Muscle memory**: Consistent key bindings become automatic, like vim for programmers but without vim's learning wall.

**Accessibility**: Keyboard-only workflow reduces RSI from mouse use.

**MIDI integration**: Both hands productive simultaneously – MIDI keyboard for pitch, computer keyboard for rhythm/articulation.

**Former Igor users**: Potential user base who remember the paradigm and will recognize its return.

**Academic opportunity**: First to formally document this HCI paradigm – no papers exist. Potential for publication and recognition.

**Communication focus**: Accurately describing Flow Mode's historical context (disappeared 2001, no modern equivalent) and genuine capabilities (5-10x speed improvement, intuitive learning curve) to help users understand why this matters.

**Open source advantage**: Community can extend/customize key bindings, share binding schemes. Flow Mode becomes freely available power-user feature that competitors cannot easily replicate.

### Negative

**Implementation complexity**: Sophisticated state machine for modal interactions, expanding elements (slurs that grow), MIDI integration, three-stage workflow with cursor system.

**Archaeology required**: Must reconstruct Igor's complete implementation details from incomplete documentation, user memory, and logical inference.

**Competitive moat barrier**: The 23-year gap suggests high barriers (complexity, conceptual gap, platform conflicts) – implementation will be challenging.

### Neutral

**Optional feature**: Click Mode remains fully functional and primary entry method. Flow Mode is opt-in power-user feature.

**Phased implementation**: Click Mode implemented first (validates architecture). Flow Mode layered on top after basic rendering works.

**Customization boundaries**: Must decide between vim approach (full rebinding) or fixed bindings with limited customization. Design decision deferred until implementation phase.

**Mobile implications**: Flow Mode fundamentally keyboard-centric. Touch screens will use Click Mode only.

---

## Implementation Strategy

### Parallel Development

Flow Mode implementation advances in lockstep with rendering (Phase 12) and MusicXML import (Phase 11), ensuring each capability level works end-to-end:

**Level 1**: Basic notes
- Import → Render with flags → Click to place → Flow Mode pitch selection + Enter

**Level 2**: Articulations
- Import → Render markings → Click to apply → Flow Mode toggles (`.` for staccato, `>` for accent)

**Level 3**: Beams
- Import → Render beams → Click operations → Flow Mode automatic (replace flags with beams)

**Level 4**: Slurs/Ties
- Import → Render curves → Click-drag → Flow Mode expanding elements

### Success Criteria

Flow Mode is successful when:

1. **Speed**: Experienced user can input music 5-10x faster than Click Mode
2. **Adoption**: Professional users prefer Flow Mode after learning period
3. **Muscle memory**: Common operations become automatic
4. **MIDI integration**: Combined keyboard + MIDI workflow feels natural
5. **Expanding elements**: Slurs/crescendos extend predictably and intuitively
6. **Low error rate**: Modal state clear enough that errors are rare
7. **Undo safety net**: Mistakes easily corrected, encouraging experimentation

### Modern Improvements Over Igor

1. **Enhanced mode indicator HUD**: Clear visual display of all active modes without obstructing score view
2. **Advanced cursor system**: Smoother animation and better visual feedback via modern GPU rendering
3. **Customizable bindings**: User-configurable key mappings (Igor likely had fixed bindings)
4. **Mode presets**: Save/load mode configurations for different workflows
5. **Undo/redo integration**: Every modal action undoable with full state restoration
6. **Tutorial mode**: Interactive learning system with on-screen hints

---

## Strategic Reality

**What was expected**: "We'll need to innovate beyond competitors' improvements on Igor."

**What research revealed**: "No one copied it. We just need to remember how Igor worked."

This transforms Flow Mode from an innovation challenge into a **memory + implementation challenge**. Every detail remembered (Enter creates notes, Shift selects ranges, `/` cuts beams) is competitive intelligence that no competitor has because they never experienced Igor.

**The competitive moat isn't innovation – it's memory + implementation.**

Flow Mode is a major USP simply by existing, not by being better than competitors' versions, because there are no competitors' versions.

The 23-year market gap is a gift:
1. **No moving target** – Competitors aren't iterating on this paradigm
2. **Proven design** – Igor's approach worked; users loved it; business failures killed it, not UX problems
3. **Low bar** – Don't need to be revolutionary, just competent at implementing what Igor proved worked
4. **Immediate differentiation** – Day one release with Flow Mode = genuinely unique in 2025 market

### Radical Transparency and Open Documentation

**This ADR and accompanying research are published openly before implementation.** Other notation software projects could read this documentation and implement Flow Mode before Ooloi releases. This is acceptable and part of Ooloi's Radical Transparency approach.

**Why this is fine:**

1. **Benefit to musicians**: If Dorico, MuseScore, or others implement Flow Mode first, professional users benefit immediately – that's a win regardless of who delivers it
2. **Open source advantage persists**: Even if others implement it, Ooloi's version will be freely available, community-extensible, and modifiable – commercial software cannot match this
3. **Not traditional competition**: Ooloi doesn't operate on secrecy and surprise. Documenting solutions publicly contributes to the field even if others benefit
4. **Implementation barrier is real**: The 23-year gap suggests genuine complexity. Documentation alone may not be sufficient for rapid implementation
5. **Multiple implementations benefit everyone**: Different approaches to Flow Mode would validate the paradigm and create a rising tide for modal notation input

**The value isn't in hiding the idea – it's in executing it well and making it freely available.**

---

## Tradeoffs and Considerations

**Complexity vs. Speed**: Flow Mode's implementation complexity is justified by professional speed gains. The 5-10x improvement in input speed for daily users makes the development investment worthwhile.

**Learning Curve Reality**: While lower than vim/emacs, there is still a learning period. However, Click Mode remains available – users can adopt Flow Mode gradually as they learn commands.

**Discoverability vs. Efficiency**: Flow Mode optimizes for efficiency over discoverability. This is acceptable because Click Mode serves the discoverability need.

**Documentation Archaeology**: Incomplete Igor documentation is a risk, but partial knowledge + logical musical inference + user testing can fill gaps. The paradigm's core principles are well-understood.

**Platform Constraints**: Flow Mode won't work on mobile/tablets (keyboard-centric). This is acceptable – Click Mode serves those platforms. Desktop is where professionals do serious notation work.

---

## Open Questions (Deferred to Implementation)

1. **Mode coexistence**: Should Flow Mode and Click Mode be completely separate, or can user click during Flow Mode?

2. **Rare elements**: How to handle notation not easily keyboard-accessible (stopped horn, custom lines)? Fall back to palette or obscure key combos?

3. **Customization boundaries**: Allow full rebinding (vim approach) or fixed bindings with limited customization (Emacs approach)? How to share/distribute binding schemes?

4. **Conflict resolution**: When multiple interpretations exist (e.g., `>` for accent vs. crescendo), what's the precedence?

---

## Related ADRs

**Foundational:**
- **ADR-0001: Frontend-Backend Separation** – Flow Mode state management on frontend, backend authoritative for piece data
- **ADR-0031: Frontend Event-Driven Architecture** – Modal state changes integrate with event system, expanding elements supported

**Integration:**
- **ADR-0022: Event-Driven Data Synchronization** – Flow Mode note creation triggers backend updates, invalidation events refresh display
- **ADR-0028: Hierarchical Rendering Pipeline** – Backend computes all layout, Flow Mode just creates/modifies musical elements

**Supporting:**
- **ADR-0014: Timewalk** – Three-stage workflow (position → select → create) uses timewalk for cursor positioning and element resolution
- **ADR-0017: Integrant Component Lifecycle** – Flow Mode state management as Integrant component with proper lifecycle

---

## References

**Research:**
- Comprehensive competitive analysis documented in `research/OOLOI_FLOW_MODE.md`
- 532 sources validate 23-year market gap (2001-2025)
- Academic literature search (ISMIR, NIME, SMC, TENOR, ACM CHI) – zero papers on modal notation input

**Historical:**
- Igor Engraver (1996-2009) – original implementation, contemporary documentation fragments
- Former Igor beta testers – user memory of keyboard commands and workflow

**Design Philosophy:**
- Vim/Emacs modal editing paradigms applied to music notation domain
- Low learning curve through intuitive symbol-to-keystroke mapping
- Professional speed optimization over casual user discoverability
