# ADR-0057: Reference Model of the Spacing Pipeline

## Status

Accepted

The decision is made: the pipeline's Stages 1 to 5, as far as horizontal spacing, casting off and beams reach, are explored and specified on an exact model before they are built, and their implementation is checked against it. The model is kept in two implementations that agree exactly.

## Table of Contents

- [Context](#context)
  - [One Model for All of Spacing](#one-model-for-all-of-spacing)
  - [What Makes the Model Possible](#what-makes-the-model-possible)
- [Decision](#decision)
  - [What the Model Is](#what-the-model-is)
  - [The Model Follows the Pipeline](#the-model-follows-the-pipeline)
  - [One Model, Two Implementations](#one-model-two-implementations)
  - [Specified Before It Is Built](#specified-before-it-is-built)
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

The same holds beyond the springs. Where the breaks fall is judged across a page, and so is whether identical preambles stand identical down it ([ADR-0058](0058-Variants-at-a-Break.md)). A beam's ends are placed by tables, Ross's sit, straddle and hang, which say where each end stands against the staff lines; their consequences across a passage, in every font and as a system is stretched, show only when they meet music. And a stage specified in prose is specified only as far as the prose is unambiguous, which shows only when something computes it.

### One Model for All of Spacing

Every horizontal rule, however fine, reaches the springs in one of ADR-0055's four forms: an ideal width, a blocking width, a constraint spanning several columns, or a variant at a break ([The Four Forms](0055-Horizontal-Spacing.md#the-four-forms)). No rule runs as a pass of its own that corrects the layout after the springs have placed it, and no two rules negotiate with each other. With the underlying mathematics right, the finer points of engraving fold into the same model as everything else, rather than becoming a herd of formatters, each correcting what the others did, that nobody can steer. Folding every rule into one model is what makes spacing fast and predictable: the springs are solved once, exactly, and the same piece always gives the same page ([ADR-0037](0037-Measure-Distribution-Optimization.md#layout-is-a-function-of-the-piece)). It is also what makes a model of spacing small enough to build and exact enough to check against.

### What Makes the Model Possible

Ooloi's architecture makes such a model possible, small and exact:

- **The pipeline runs one way.** No stage sends anything back to an earlier one ([ADR-0013](0013-Slur-Formatting.md#the-pipeline-is-one-way-and-what-stage-5-may-change), [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)), so each stage can be reproduced from its inputs alone.
- **Each stage after the springs reads finished output.** Stage 3 receives the records as numbers only ([ADR-0058](0058-Variants-at-a-Break.md#invariants-and-property-tests), Invariant 6), and Stage 5 places beams from the positions Stage 4 gives ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md#pipeline-stage-5-spanners-and-margins-fan-out-per-musician)), so casting off and beams join the model without reaching back into the stages before them.
- **Each setting declares the stage that reads it** ([ADR-0055](0055-Horizontal-Spacing.md#spacing-settings)), so a model knows where a changed setting enters, and what it can move.
- **Arithmetic is exact, and the layout is a function of the piece** ([ADR-0037](0037-Measure-Distribution-Optimization.md#layout-is-a-function-of-the-piece)), so two implementations can be compared for equality rather than within a tolerance.
- **Glyph geometry is data**, read once from the fonts' outlines ([ADR-0056](0056-Glyph-Geometry-from-Outlines.md)), so a model sees the ink the implementation will see.

Hegazy and Gourlay's springs are also cheap to compute: a model that recomputes a passage's springs and positions in exact arithmetic does so faster than a slider moves ([Consequences](#consequences)).

## Decision

The pipeline has an executable reference model: exact, divided as the pipeline is, and kept in two implementations that agree exactly. It reaches as far as horizontal spacing, casting off and beams ([What the Model Is Not](#what-the-model-is-not)). Open questions within that reach are explored on it before they are decided, each stage it covers is specified on it before Ooloi builds that stage, and the Clojure implementation is checked against it. Every figure about spacing, casting off, beams and glyph geometry in the ADRs is computed by it.

### What the Model Is

A model of a passage laid out in systems. Its inputs are a passage of music, on one staff or several and in one voice or several on each, its settings, and the system width, from which Stage 3 casts the passage off into systems. A view of a single system takes that system's adjustment ratio or width instead, and leaves Stage 3 out. From its inputs the model computes what the pipeline computes: atoms and their contours, springs and records, breaks and each system's adjustment ratio, positions, and what Stage 5 draws into the room Stage 4 gives, beams among it. Its values are the placeholders the ADRs name, until the exploration settles them. It draws from the glyphs' outlines, so a figure shows the ink its numbers were computed against.

### The Model Follows the Pipeline

The model is divided as [ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)'s pipeline is, at the same boundaries, and each stage reads only the settings [ADR-0055](0055-Horizontal-Spacing.md#spacing-settings) assigns to it:

| Stage | In the model | Reads |
| --- | --- | --- |
| 1 | Atoms: column assembly, with the noteheads of voices at one onset offset or shared ([ADR-0055 H5](0055-Horizontal-Spacing.md#h5-column-assembly)); contours from the outline profiles, each accidental at its tightest offset, ledger lines at their minimum forms, and the forms beaming declares for beamed notes ([ADR-0055](0055-Horizontal-Spacing.md#atoms-and-contours)); the first atom's system-start contour, and the minimum forms of continuations ending at it | The forms: the minimum protrusions, the full extension, the opening width, the distances within an atom; the rules of column assembly |
| 2 | The raster and its springs: columns from the onsets of every voice in every staff; gaps by class; ideal widths; blocking widths from contours compared within each staff, the largest across the staves; constraints spanning several columns, reduced as ADR-0037 reduces them; stretch weights. The system as `start[s]`, its interiors and mids, and `end[t]`, with the preamble table's entries ([ADR-0058](0058-Variants-at-a-Break.md)) | The duration table, the clearances, the settings of the space around a barline, the distances inside the blocks |
| 3 | Casting off: [ADR-0037](0037-Measure-Distribution-Optimization.md#core-dynamic-programming-structure-stage-3)'s dynamic programme over the records, each candidate system its start record, its stacks' interiors, the mids of its inner barlines and its end record ([ADR-0058](0058-Variants-at-a-Break.md#what-stage-3-receives)), with ADR-0058's early-termination bound; the breaks, and each system's adjustment ratio. Left out in the view of a single system | The system width; the records and nothing else; the cost's parameters ([ADR-0037](0037-Measure-Distribution-Optimization.md#optimization-characterization)) |
| 4 | Positions: blocks and rows at their fixed offsets, springs at each system's ratio ([ADR-0037](0037-Measure-Distribution-Optimization.md)) | The ratio or the width; settings read only for the stretch weights, such as the stretch of the space after a barline |
| 5 | Ledger lines lengthened into the room Stage 4 gave; the accidental position; stays nearer its own note; the blocks at a break, courtesy accidentals and continuations drawn; beams, their ends placed against the staff lines as the beaming rules say, and the stem lengths they give | The accidental position; stays nearer its own note and its ratio; the settings the beaming decision assigns to Stage 5 |
| 6 | Left out | |

Because the model's structure is the implementation's, a rule shown to work in the model has a place waiting for it in the implementation, and a rule that will not fit a stage's inputs and outputs is found before any code depends on it. ADR-0055's four forms hold in the model as in the implementation: a rule the model cannot express in one of them is a design problem, settled in the ADR ([The Four Forms](0055-Horizontal-Spacing.md#the-four-forms)). Column assembly is one test of this: an arrangement of voices that needs to know the spacing does not fit Stage 1.

### One Model, Two Implementations

- **The reference implementation**, in Python, computes the ADRs' figures and renders them with Skia, the rasteriser Ooloi draws with.
- **The interactive implementation**, in JavaScript, runs in a browser with every setting live. It is self-contained, needing no library and no network, and it is public and free to download. It offers every bundled music font, all of them under the SIL Open Font License ([ADR-0047](0047-Font-Management.md#bundled-fonts)). For each it carries only the handful of glyphs its examples draw: their outlines, their profiles and their SMuFL metadata (bounding boxes and anchors), with the font's engraving defaults, and it loads a font's profiles only when the font is first used. It never carries a font file, nor the outlines of any glyph its examples do not use.
- **Reference values hold them together.** The reference implementation writes what each stage computes for a set of inputs, as exact fractions, and the interactive implementation is checked against them for equality. The Clojure implementation's tests read the same values; Clojure's ratios are exact, so they too compare for equality.
- **A rule changes everywhere at once.** A decided change goes into the ADR, the reference implementation and the interactive one together, and the checks fail until all of them agree. Two implementations written separately also check each other: a difference between them is a defect in one of them, or an ambiguity in the ADR.

### Specified Before It Is Built

A stage the model covers is specified on it before the Clojure implementation builds it. The ADRs state its rules; the model shows them computing what they mean, in every case the exploration puts before it; the reference values record what the stage computes for a set of cases, and are the contract the implementation's tests hold it to. A stage is ready to be built when its ADR, both implementations and the reference values agree. A rule the implementation finds missing or ambiguous is not settled in production code: it goes into the ADR and both implementations of the model, as any decided change does.

### The Exploration

How a principle is found, and how it becomes a rule. This extends [ADR-0055's exploration](0055-Horizontal-Spacing.md#the-exploration), which settles values.

- **The eye.** A principle begins with the trained eye, the project author's and that of the people following the project. Something looks wrong, or looks natural and organic, and naming why is how the principle is found.
- **The model.** The principle becomes a rule in the model, and the model shows its consequences in every case at once. The eye adjusts the rule against those cases until the result reads as natural. This is the work engravers spend most of their time on, and a model that answers as the settings move is what makes it practical here. The model is flexible, and anything that helps the eye judge an open question can be added to it for a while: a control, an example, two rules side by side.
- **The plate.** Printed engraving in the plate-engraving tradition, measured, is part of the same loop: it shows how engravers resolved the cases the eye is weighing, and it settles values ([ADR-0055](0055-Horizontal-Spacing.md#sources-of-truth)).
- **More cases before the eye.** A principle found in one passage must hold in others. Presets put many passages in front of the eye, on one staff and on several, in one voice and in several, and ADR-0055's invariants, run on the model over generated passages, find the cases nobody thought to try. For what differs at a break, the presets begin with the cases [ADR-0058](0058-Variants-at-a-Break.md#the-exploration) names, Ravel's *Ondine* first.
- **Casting off.** A passage cast off into systems is judged as a page is: where the breaks fall, whether preambles stand identical down the page ([ADR-0058](0058-Variants-at-a-Break.md#the-exploration)), and which adjustment ratios a passage's systems take, so that a rule is weighed at the ratios music gives as well as at the extremes of the gauge.
- **The table.** Where a source states a rule as a table, as Ross's tables of sit, straddle and hang do for a beam's ends, the model computes every case the table covers and is checked against it for equality ([Invariant 6](#invariants-and-property-tests)), and draws every case as the source prints it, so that the eye checks the transcription against the page. Where a source contradicts itself, each reading is an option of the model, side by side, until the eye and the plates decide between them.
- **The gauge is a drag.** Moving a system's ratio moves its noteheads as a column drag does, so every discrete Stage 5 decision, a beam's quantised slant among them, is seen changing as the ratio moves. Whether Stage 5 updates such decisions during a drag or holds them until it ends ([ADR-0013](0013-Slur-Formatting.md#discrete-stage-5-decisions-during-a-drag)) is explored there.

**Rhythmic order shows the loop.** With contours alone, a sixteenth after a double-dotted half, stems up then down, may stand with its notehead among the dots and its accidental above the previous notehead, since none of their ink collides. The eye rejects it, and the principle is that a note's dots belong to its width. Applied in the model, the first form of the rule also stopped an accidental tucking under a dotted note, which the eye had accepted. Refined, the rule holds a notehead clear of the previous noteheads and dots and lets its accidental tuck. [ADR-0055](0055-Horizontal-Spacing.md#blocking-widths) states it.

### Every Figure from the Model

Every figure about spacing, casting off, beams or glyph geometry in the ADRs is computed by the model. Whenever the model changes, every figure is computed again and compared with the last. A figure that has changed replaces the one its ADR shows, and its caption, the numbers its ADR quotes from it and other ADRs citing those numbers change with it. The ADRs' figures therefore always show the current decisions, as the understanding of engraving deepens.

A figure that exists to show an earlier state or a rejected alternative, such as bounding boxes in place of outlines, takes that state from the model as an option, kept for as long as the figure is. Figures already published on the blog stay as they were published.

A figure an ADR needs before the model can compute it is a mock-up, and its caption says so; the model replaces it once it can, as [ADR-0058](0058-Variants-at-a-Break.md#the-exploration)'s figures are replaced.

### What the Model Is Not

- **Not Ooloi's renderer.** Its figures say so: a conceptual model, not rendered by Ooloi. Ooloi never runs it.
- **Not a source of values.** Its placeholders are named as placeholders; values come from the exploration and its register ([ADR-0055](0055-Horizontal-Spacing.md#the-exploration)).
- **Not the implementation.** It specifies what each stage computes, not how Ooloi computes it. The piece model, caching and incremental recomputation ([ADR-0037](0037-Measure-Distribution-Optimization.md#caching-and-incremental-recompute)), and the stages' fan-out and threading ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)), are the implementation's, and change no value the model computes.
- **Not the whole pipeline.** Of Stage 5 it computes only what spacing and beams need: ledger lines, accidentals, the blocks at a break, tie continuations and glissando ends at a system start ([ADR-0058](0058-Variants-at-a-Break.md)), and beams. Slurs, other ties and the rest of Stage 5's spanners are outside it ([ADR-0013](0013-Slur-Formatting.md)), and so is page breaking in Stage 6.

## Invariants and Property Tests

1. **Parity.** For every case in the reference values, the interactive implementation computes the reference implementation's values exactly, stage by stage. The Clojure implementation's tests hold it to the same values.
2. **A setting moves only what its stage can move.** Changing a setting changes the results of the stage that reads it and of the stages after it, and nothing computed before that stage ([ADR-0055](0055-Horizontal-Spacing.md#spacing-settings)).
3. **Determinism.** The same inputs give the same values in every implementation, on every platform, in every run.
4. **The figures are current.** Computing every figure again, in the reference environment, reproduces the figures the ADRs show. A figure that differs is stale. Renders are compared within one environment, since a different rasteriser build changes pixels without changing anything the model computes.
5. **The pipeline's invariants hold in the model**, for the stages it covers: ADR-0055's, [ADR-0058](0058-Variants-at-a-Break.md#invariants-and-property-tests)'s and [ADR-0037](0037-Measure-Distribution-Optimization.md#stability-guarantees)'s, tested over generated passages.
6. **Tabulated rules hold exactly.** For every case a table adopted in an ADR covers, such as Ross's sit, straddle and hang for a beam's ends, the model computes the table's entry exactly, at the dimensions the table is stated for. The cases are enumerated, not sampled.

## Alternatives Considered

| Alternative | Why not |
| --- | --- |
| Exploring in the Clojure implementation | Confuses whether a rule is right with whether its implementation is; a wrong rule arrives as a defect; every trial costs a build. |
| Stages specified in the ADRs alone | Prose specifies a stage only as far as it is unambiguous, and nothing shows where it is not until the implementation computes it; the ambiguity then arrives as a defect. |
| Figures alone, rendered one at a time | The eye sees frames rather than consequences, and every question costs a render. |
| An interactive model alone | No reference renders the ADRs' figures with Ooloi's rasteriser, and no second implementation catches the first one's errors. |
| ADR-0055's property tests alone | They state what must never happen, not what a rule produces, and show the eye nothing. |
| A model in floating point | Cannot be compared exactly. Decisions that turn on equality, such as which setting sets a gap, would differ between implementations. |

## Consequences

### Positive

- **Rules are seen before they are built.** Edge cases and general principles surface in the model, not as defects in the implementation.
- **Stages are specified before they are built.** The Clojure implementation of a stage begins from rules already seen to compute what they mean, and from the values it must reproduce.
- **The implementation has an oracle**: exact values for every stage it builds.
- **Casting off is judged as a page** before the breaker is built.
- **Tabulated rules are checked in full**: every case a table covers is computed and compared.
- **The ADRs' figures always show the current decisions.**
- **Anyone can explore.** The interactive model is public, and shows what the architecture does as its settings move.
- **The stage boundaries are exercised** before the implementation depends on them.
- **Interactive at the scale of a passage.** Measured in JavaScript, in exact arithmetic, on a 2017 Intel MacBook Pro, four measures on one staff are recomputed in under 2 ms when a Stage 2 setting changes, and in about 0.3 ms when only the ratio does. An Apple M3 is three to five times faster.

### Negative

- **Three implementations to keep in step**: the reference, the interactive model and Ooloi itself. The reference values make every disagreement visible, but each rule is written more than once. Every stage the model covers adds to this, and to the time it takes to compute the reference values, the checks and the figures again after a change.
- **The model covers part of the pipeline**: Stages 1 to 5 for a passage, of Stage 5 only ledger lines, accidentals, the blocks at a break, the continuations at a system start and beams, and no page breaking. Conclusions drawn from it reach as far as it does.
- **A table can be transcribed wrongly.** The check compares the model with the table as transcribed; only the eye, comparing the drawn cases with the source, checks the transcription.
- **Placeholders can be mistaken for decisions.** Captions, and the model's own labels, say which values are which.
- **Whatever the interactive model contains is published.**

## Open Items

| Item | Question | Settled by |
| --- | --- | --- |
| M1: The reference values | Their format, which cases they cover at each stage, and how the Clojure implementation's tests read them. | The first Clojure implementation of Stages 1 and 2 |
| M2: Stage 5's reach | Whether slurs, and ties other than the continuations at a system start, join the model once their ADRs specify them. | [ADR-0013](0013-Slur-Formatting.md) and the Slur and Tie Geometry ADR |
| M3: The form of the beaming tables | Whether the positions of a beam's ends enter the beaming rules as Ross's tables, held as data, or as a criterion that reproduces his tables exactly at the dimensions they are stated for and extends to other beam and staff-line thicknesses, which are settings of the piece ([ADR-0047](0047-Font-Management.md#per-font-metadata)). | The beaming decision |
| M4: Interactivity at the scale of a page | Whether casting off a passage of several systems, in exact arithmetic, recomputes as fast as a slider moves, as a single system does ([Consequences](#consequences)). | Measurement of the interactive implementation's Stage 3 |
| M5: Independence of the two implementations | Whether the interactive implementation is written from the ADRs rather than from the reference implementation, so that a difference between them can reveal an ambiguity in an ADR as well as a defect in one of them ([One Model, Two Implementations](#one-model-two-implementations)). | The next rewrite of a stage in either implementation |

## References

### Related ADRs

- [ADR-0055: Horizontal Spacing](0055-Horizontal-Spacing.md) — the forms, stages, settings and exploration the model implements; column assembly (H5) and the forms beaming declares to Stage 1; its figures are the model's
- [ADR-0037: Measure Distribution Optimization](0037-Measure-Distribution-Optimization.md) — springs, Stage 3's dynamic programme and its cost, Stage 4 and exact arithmetic; the reduction of constraints spanning several columns
- [ADR-0056: Glyph Geometry from Outlines](0056-Glyph-Geometry-from-Outlines.md) — outline profiles; its figures are the model's
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) — the stages the model follows
- [ADR-0013: Slur Formatting](0013-Slur-Formatting.md) — the one-way pipeline, what Stage 5 may change, and discrete Stage 5 decisions during a drag
- [ADR-0047: Font Management](0047-Font-Management.md) — the bundled fonts and their licences; engraving values as settings of the piece
- [ADR-0058: Variants at a Break](0058-Variants-at-a-Break.md) — a system's start and end as barline records, what Stage 3 receives and the early-termination bound, which the model computes; its figures are to be the model's

### External References

- Hegazy, W.A. and Gourlay, J.S. "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, The Ohio State University (1987) — the springs the model computes
- Ross, T. *The Art of Music Engraving and Processing* (1970) — the tables of sit, straddle and hang the model's beams are checked against
- [Bravura](https://github.com/steinbergmedia/bravura) — the font whose outlines the figures are drawn from
- [Skia](https://skia.org) — the rasteriser the reference implementation renders with