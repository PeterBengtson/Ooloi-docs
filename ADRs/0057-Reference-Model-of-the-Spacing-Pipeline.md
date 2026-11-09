# ADR-0057: Reference Model of the Spacing Pipeline

## Status

Accepted

The decision is made: horizontal spacing is explored, and its implementation checked, on an exact model of the pipeline's Stages 1, 2, 4 and 5, kept in two implementations that agree exactly.

## Table of Contents

- [Context](#context)
  - [One Model for All of Spacing](#one-model-for-all-of-spacing)
  - [What Makes the Model Possible](#what-makes-the-model-possible)
- [Decision](#decision)
  - [What the Model Is](#what-the-model-is)
  - [The Model Follows the Pipeline](#the-model-follows-the-pipeline)
  - [One Model, Two Implementations](#one-model-two-implementations)
  - [The Exploration](#the-exploration)
  - [Every Figure from the Model](#every-figure-from-the-model)
  - [What the Model Is Not](#what-the-model-is-not)
- [Invariants and Property Tests](#invariants-and-property-tests)
- [Alternatives Considered](#alternatives-considered)
- [Consequences](#consequences)
- [Open Items](#open-items)
- [References](#references)

## Context

Horizontal spacing is decided as rules: the forms in which they reach Stage 3, the settings and the stages that read them ([ADR-0055](0055-Horizontal-Spacing.md)), the springs and how Stage 4 places them ([ADR-0037](0037-Measure-Distribution-Optimization.md)), and the geometry the contours are built from ([ADR-0056](0056-Glyph-Geometry-from-Outlines.md)). Whether a rule is right shows only when it meets music. A clearance that looks right between two noteheads may let a sixteenth stand among the dots of the note before it, since nothing there collides; a ledger line held at its minimum may meet a flag that hangs lower than anyone expected. Engravers spend most of their time on points like these, on what makes a line of music look natural, and the eye judges them before anything is measured.

Designing rules in the implementation confuses two questions: whether the rule is right, and whether its implementation is. Answering the first in production code is slow, and a rule found wrong there arrives as a defect with no evident cause. Separating them needs a place where rules can be tried and seen before they are built.

### One Model for All of Spacing

Every horizontal rule, however fine, reaches the springs in one of ADR-0055's four forms: an ideal width, a blocking width, a constraint spanning several columns, or a variant at a break ([The Four Forms](0055-Horizontal-Spacing.md#the-four-forms)). No rule runs as a pass of its own that corrects the layout after the springs have placed it, and no two rules negotiate with each other. With the underlying mathematics right, the finer points of engraving fold into the same model as everything else, rather than becoming a herd of formatters, each correcting what the others did, that nobody can steer. Folding every rule into one model is what makes spacing fast and predictable: the springs are solved once, exactly, and the same piece always gives the same page ([ADR-0037](0037-Measure-Distribution-Optimization.md#layout-is-a-function-of-the-piece)). It is also what makes a model of spacing small enough to build and exact enough to check against.

### What Makes the Model Possible

Ooloi's architecture makes such a model possible, small and exact:

- **The pipeline runs one way.** No stage sends anything back to an earlier one ([ADR-0013](0013-Slur-Formatting.md#the-pipeline-is-one-way-and-what-stage-5-may-change), [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)), so each stage can be reproduced from its inputs alone.
- **Each setting declares the stage that reads it** ([ADR-0055](0055-Horizontal-Spacing.md#spacing-settings)), so a model knows where a changed setting enters, and what it can move.
- **Arithmetic is exact, and the layout is a function of the piece** ([ADR-0037](0037-Measure-Distribution-Optimization.md#layout-is-a-function-of-the-piece)), so two implementations can be compared for equality rather than within a tolerance.
- **Glyph geometry is data**, read once from the fonts' outlines ([ADR-0056](0056-Glyph-Geometry-from-Outlines.md)), so a model sees the ink the implementation will see.

Hegazy and Gourlay's springs are also cheap to compute: a model that recomputes a passage's springs and positions in exact arithmetic does so faster than a slider moves ([Consequences](#consequences)).

## Decision

Horizontal spacing has an executable reference model: exact, divided as the pipeline is, and kept in two implementations that agree exactly. Open spacing questions are explored on it before they are decided, and the Clojure implementation is checked against it. Every figure about spacing and glyph geometry in the ADRs is computed by it.

### What the Model Is

A model of one system. Its inputs are a passage of music, its settings, and either the system's adjustment ratio or its width. From them it computes what the pipeline computes for that system: atoms and their contours, springs, positions, and what Stage 5 draws into the room Stage 4 gives. Its values are the placeholders the ADRs name, until the exploration settles them. It draws from the glyphs' outlines, so a figure shows the ink its numbers were computed against.

### The Model Follows the Pipeline

The model is divided as [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)'s pipeline is, at the same boundaries, and each stage reads only the settings [ADR-0055](0055-Horizontal-Spacing.md#spacing-settings) assigns to it:

| Stage | In the model | Reads |
| --- | --- | --- |
| 1 | Atoms: contours by glyph class, each accidental at its tightest offset, ledger lines at their minimum forms | The forms: the minimum protrusions, the full extension, the opening width, the distances within an atom |
| 2 | The raster and its springs: columns from every staff's onsets; gaps by class; ideal widths; blocking widths from contours compared within each staff, the largest across the staves; constraints spanning several columns, reduced as ADR-0037 reduces them; stretch weights | The duration table, the clearances, the settings of the space around a barline |
| 3 | Left out: the model computes one system, whose ratio or width is given | |
| 4 | Positions, through the springs ([ADR-0037](0037-Measure-Distribution-Optimization.md)) | The ratio or the width; settings read only for the stretch weights, such as the stretch of the space after a barline |
| 5 | Ledger lines lengthened into the room Stage 4 gave; the accidental position; stays nearer its own note | The accidental position; stays nearer its own note and its ratio |
| 6 | Left out | |

Because the model's structure is the implementation's, a rule shown to work in the model has a place waiting for it in the implementation, and a rule that will not fit a stage's inputs and outputs is found before any code depends on it. ADR-0055's four forms hold in the model as in the implementation: a rule the model cannot express in one of them is a design problem, settled in the ADR ([The Four Forms](0055-Horizontal-Spacing.md#the-four-forms)).

### One Model, Two Implementations

- **The reference implementation**, in Python, computes the ADRs' figures and renders them with Skia, the rasteriser Ooloi draws with.
- **The interactive implementation**, in JavaScript, runs in a browser with every setting live. It is self-contained, needing no library and no network, and it is public and free to download. It offers every bundled music font, all of them under the SIL Open Font License ([ADR-0047](0047-Font-Management.md#bundled-fonts)), and November 2, the commercial font Ooloi's documentation uses ([ADR-0048](0048-SMuFL-Glyph-Selection-Architecture.md)). For each it carries only the handful of glyphs its examples draw: their outlines, their profiles and their SMuFL metadata, with the font's engraving defaults. It never carries a font file, nor the outlines of any glyph its examples do not use, which matters most for a commercial font such as November 2.
- **Reference values hold them together.** The reference implementation writes what each stage computes for a set of inputs, as exact fractions, and the interactive implementation is checked against them for equality. The Clojure implementation's tests read the same values; Clojure's ratios are exact, so they too compare for equality.
- **A rule changes everywhere at once.** A decided change goes into the ADR, the reference implementation and the interactive one together, and the checks fail until all of them agree. Two implementations written separately also check each other: a difference between them is a defect in one of them, or an ambiguity in the ADR.

### The Exploration

How a principle is found, and how it becomes a rule. This extends [ADR-0055's exploration](0055-Horizontal-Spacing.md#the-exploration), which settles values.

- **The eye.** A principle begins with the trained eye, the project author's and that of the people following the project. Something looks wrong, or looks natural and organic, and naming why is how the principle is found.
- **The model.** The principle becomes a rule in the model, and the model shows its consequences in every case at once. The eye adjusts the rule against those cases until the result reads as natural. This is the work engravers spend most of their time on, and a model that answers as the settings move is what makes it practical here.
- **The plate.** Printed engraving in the plate-engraving tradition, measured, is part of the same loop: it shows how engravers resolved the cases the eye is weighing, and it settles values ([ADR-0055](0055-Horizontal-Spacing.md#sources-of-truth)).
- **Open questions as switches.** An open question can be tried as a switch in the interactive model, with its alternatives side by side. Once it is decided, the decision goes into the ADR as a rule or a setting, and the switch is removed.
- **More cases before the eye.** A principle found in one passage must hold in others. Presets put many passages in front of the eye, and ADR-0055's invariants, run on the model over generated passages, find the cases nobody thought to try.

**Rhythmic order shows the loop.** With contours alone, a sixteenth after a double-dotted half, stems up then down, may stand with its notehead among the dots and its accidental above the previous notehead, since none of their ink collides. The eye rejects it, and the principle is that a note's dots belong to its width. Applied in the model, the first form of the rule also stopped an accidental tucking under a dotted note, which the eye had accepted. Refined, the rule holds a notehead clear of the previous noteheads and dots and lets its accidental tuck. [ADR-0055](0055-Horizontal-Spacing.md#blocking-widths) states it.

### Every Figure from the Model

Every figure about spacing or glyph geometry in the ADRs is computed by the model. Whenever the model changes, every figure is computed again and compared with the last. A figure that has changed replaces the one its ADR shows, and its caption, the numbers its ADR quotes from it and other ADRs citing those numbers change with it. The ADRs' figures therefore always show the current decisions, as the understanding of engraving deepens.

A figure that exists to show an earlier state or a rejected alternative, such as bounding boxes in place of outlines, takes that state from the model as an option, kept for as long as the figure is. Figures already published on the blog stay as they were published.

### What the Model Is Not

- **Not Ooloi's renderer.** Its figures say so: a conceptual model, not rendered by Ooloi. Ooloi never runs it.
- **Not a source of values.** Its placeholders are named as placeholders; values come from the exploration and its register ([ADR-0055](0055-Horizontal-Spacing.md#the-exploration)).
- **Not the whole pipeline.** It computes one system at a time, without breaking, and of Stage 5 only what spacing needs: ledger lines and accidentals. Slurs, ties and beams are outside it ([ADR-0013](0013-Slur-Formatting.md)).

## Invariants and Property Tests

1. **Parity.** For every case in the reference values, the interactive implementation computes the reference implementation's values exactly, stage by stage. The Clojure implementation's spacing tests hold it to the same values.
2. **A setting moves only what its stage can move.** Changing a setting changes the results of the stage that reads it and of the stages after it, and nothing computed before that stage ([ADR-0055](0055-Horizontal-Spacing.md#spacing-settings)).
3. **Determinism.** The same inputs give the same values in every implementation, on every platform, in every run.
4. **The figures are current.** Computing every figure again, in the reference environment, reproduces the figures the ADRs show. A figure that differs is stale. Renders are compared within one environment, since a different rasteriser build changes pixels without changing anything the model computes.
5. **ADR-0055's invariants hold in the model**, tested over generated passages.

## Alternatives Considered

| Alternative | Why not |
| --- | --- |
| Exploring in the Clojure implementation | Confuses whether a rule is right with whether its implementation is; a wrong rule arrives as a defect; every trial costs a build. |
| Figures alone, rendered one at a time | The eye sees frames rather than consequences, and every question costs a render. |
| An interactive model alone | No reference renders the ADRs' figures with Ooloi's rasteriser, and no second implementation catches the first one's errors. |
| ADR-0055's property tests alone | They state what must never happen, not what a rule produces, and show the eye nothing. |
| A model in floating point | Cannot be compared exactly. Decisions that turn on equality, such as which setting sets a gap, would differ between implementations. |

## Consequences

### Positive

- **Rules are seen before they are built.** Edge cases and general principles surface in the model, not as defects in the implementation.
- **The implementation has an oracle**: exact values for every stage it builds.
- **The ADRs' figures always show the current decisions.**
- **Anyone can explore.** The interactive model is public, and shows what the architecture does as its settings move.
- **The stage boundaries are exercised** before the implementation depends on them.
- **Interactive at the scale of a passage.** Measured in JavaScript, in exact arithmetic, on a 2017 Intel MacBook Pro, four measures on one staff are recomputed in under 2 ms when a Stage 2 setting changes, and in about 0.3 ms when only the ratio does. An Apple M3 is three to five times faster.

### Negative

- **Three implementations to keep in step**: the reference, the interactive model and Ooloi itself. The reference values make every disagreement visible, but each rule is written more than once.
- **The model covers part of the pipeline**: one system, no breaking, and of Stage 5 only ledger lines and accidentals. Conclusions drawn from it reach as far as it does.
- **Placeholders can be mistaken for decisions.** Captions, and the model's own labels, say which values are which.
- **Whatever the interactive model contains is published.**

## Open Items

| Item | Question | Settled by |
| --- | --- | --- |
| M1: The reference values | Their format, which cases they cover at each stage, and how the Clojure implementation's tests read them. | The first Clojure implementation of Stages 1 and 2 |
| M2: Stage 5's reach | Whether slurs, ties and beams join the model once their ADRs specify them. | [ADR-0013](0013-Slur-Formatting.md) and the beaming decision |
| M3: November 2's outlines in public | Whether the outlines of November 2, a commercial font, may be published with the interactive model. | Its publisher, Klemm Music Technology |

## References

### Related ADRs

- [ADR-0055: Horizontal Spacing](0055-Horizontal-Spacing.md) — the forms, stages, settings and exploration the model implements; its figures are the model's
- [ADR-0037: Measure Distribution Optimization](0037-Measure-Distribution-Optimization.md) — springs, Stage 4 and exact arithmetic; the reduction of constraints spanning several columns
- [ADR-0056: Glyph Geometry from Outlines](0056-Glyph-Geometry-from-Outlines.md) — outline profiles; its figures are the model's
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) — the stages the model follows
- [ADR-0013: Slur Formatting](0013-Slur-Formatting.md) — the one-way pipeline and what Stage 5 may change
- [ADR-0047: Font Management](0047-Font-Management.md) — the bundled fonts and their licences
- [ADR-0048: SMuFL Glyph Selection Architecture](0048-SMuFL-Glyph-Selection-Architecture.md) — November 2, the commercial font Ooloi's documentation uses

### External References

- Hegazy, W.A. and Gourlay, J.S. "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, The Ohio State University (1987) — the springs the model computes
- [Bravura](https://github.com/steinbergmedia/bravura) — the font whose outlines the figures are drawn from
- [Skia](https://skia.org) — the rasteriser the reference implementation renders with
