# ADR-0055: Horizontal Spacing

## Status

Accepted

The architecture is decided: the forms in which horizontal engraving rules reach Stage 3, what Stages 1 and 2 compute and in which form, and the invariants that hold between them. Values and several policies are open. Each is an entry in [Open Items](#open-items), which is amended in place as [the exploration](#the-exploration) settles it. One open item reaches beyond values: a stretchability per class of gap would change ADR-0037's interface ([H11](#h11-elasticity-by-gap-class)).

Revised October 2026: everything at a barline belongs to a measure, the barline in the form it takes at a system's end and any clef, key or time signature change included, so the postamble holds only cautionary key and time signatures; what a repeat barline adds mid-system is no-break material, which ADR-0037's interface carries for each boundary ([The Bar Boundary](#the-bar-boundary)). Stage 2 records provenance, and spacing can be inspected ([Inspection](#inspection)). [H7](#h7-grace-notes) has a starting point for grace notes. A barline drag has two modes ([User Inputs](#user-inputs)). [H20](#h20-drags-as-edits) is settled, and [H19](#h19-order-of-kinds-at-one-onset) and [H21](#h21-barline-drag-modes) are added.

Revised October 2026: Stage 5 adjusts elements in three ways: within an envelope the contours reserved, from a minimum form into space Stage 4 has given, and, for loose elements, into whatever room exists. Ledger lines are no longer an exception to the envelope rule but the first case of a minimum form, a pattern any element whose comfortable form is wider than its acceptable minimum may declare ([Atoms and Contours](#atoms-and-contours), [Formatter Conditions and Plugins](#formatter-conditions-and-plugins)). [H22](#h22-minimum-forms) is added. Removing a column transfer is undone with the edit that caused it ([H14](#h14-removal-of-column-transfers)).

## Table of Contents

- [Context](#context)
  - [Scope](#scope)
  - [What Horizontal Spacing Must Achieve](#what-horizontal-spacing-must-achieve)
  - [How Engravers Reason](#how-engravers-reason)
  - [Sources of Truth](#sources-of-truth)
- [Decision](#decision)
  - [The Four Forms](#the-four-forms)
  - [Stages 1 and 2](#stages-1-and-2)
  - [Atoms and Contours](#atoms-and-contours)
  - [The Raster and Gap Classes](#the-raster-and-gap-classes)
  - [The Bar Boundary](#the-bar-boundary)
  - [Ideal Widths](#ideal-widths)
  - [Blocking Widths](#blocking-widths)
  - [Constraints Spanning Several Columns](#constraints-spanning-several-columns)
  - [Formatter Conditions and Plugins](#formatter-conditions-and-plugins)
  - [Variants at a Break](#variants-at-a-break)
  - [Elasticity by Gap Class](#elasticity-by-gap-class)
  - [User Inputs](#user-inputs)
  - [Inspection](#inspection)
  - [Invariants and Property Tests](#invariants-and-property-tests)
- [The Exploration](#the-exploration)
- [Open Items](#open-items)
- [Consequences](#consequences)
- [Alternatives Considered](#alternatives-considered)
- [Prior Art](#prior-art)
- [References](#references)

## Context

### Scope

[ADR-0028](0028-Hierarchical-Rendering-Pipeline.md) defines the six-stage rendering pipeline. [ADR-0037](0037-Measure-Distribution-Optimization.md) begins where Stage 3 receives a summary of each measure stack. It treats every gap between neighbouring columns as a spring with an ideal width and a blocking width, fits the springs of each candidate system, chooses the breaks, and has Stage 4 apply each system's adjustment ratio to its gaps.

This ADR specifies how Stages 1 and 2 produce those springs from the music: what an atom's horizontal geometry is, how columns and gaps are formed, how each gap's ideal and blocking widths are derived, which rules become constraints spanning several columns, and which material differs at a break. It ends at ADR-0037's [Interface Contract](0037-Measure-Distribution-Optimization.md#interface-contract) and does not restate the model beyond it.

Read first, in ADR-0037: [Gaps as Springs](0037-Measure-Distribution-Optimization.md#gaps-as-springs), [Interface Contract](0037-Measure-Distribution-Optimization.md#interface-contract), [Constraints Spanning Several Columns](0037-Measure-Distribution-Optimization.md#constraints-spanning-several-columns), and its [design principle](0037-Measure-Distribution-Optimization.md#design-principle-no-engraving-rules-after-stage-2): Stages 3 and 4 contain no engraving rules, and every rule about horizontal space reaches them as an ideal width, a blocking width, a constraint spanning several columns or a variant at a break. This ADR is where those rules live.

Terms used throughout:

- **Atom**: everything at one rhythmic position in one staff that moves as a unit ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)): the noteheads of every voice at that onset, their stems and flags, accidentals, dots, articulations and the like. Its geometry is relative to its **origin**.
- **Column**: a position in a measure stack at which any staff has an atom, or which material without duration occupies, such as a barline.
- **Gap**: the space between two neighbouring columns of a stack; a spring in ADR-0037's sense, with an **ideal width** `w` and a **blocking width** `b`.
- **Adjustment ratio** `r`, **stretchability** `σ`, **ordinary**, **prestretched**, **kink**, **fixed space**: as in ADR-0037.
- **Minimum form**: the least horizontal extent an element can acceptably take, held in its atom's contour when its comfortable form is wider ([Atoms and Contours](#atoms-and-contours)).
- **Staff space**: the unit of every width in this ADR, at the system's staff size. All widths are exact rationals.

### What Horizontal Spacing Must Achieve

Spacing must reflect duration, but not in proportion to it. A whole note does not take four times the space of a quarter: linear spacing opens gaps so wide that the eye loses the line, and equal spacing destroys the rhythm the notation exists to show. Engraving practice grows the space by a step for each doubling of duration, a roughly logarithmic relationship. A half note is not twice a quarter; it is a different kind of event, and its space has to say so. Whether the step is a constant amount or a constant factor is part of [H1](#h1-the-duration-table).

The symbols are physical. Noteheads, stems, accidentals, dots and articulations have extent, and no two may collide. Legibility needs more than the absence of collision: notation can be collision-free and still too cramped to read. And an accidental must belong unmistakably to its note.

When spacing fails, it fails in specific ways. A note carrying an accidental looks late, because the accidental was given room at the note's expense. An accidental drifts toward the previous note, and which note it belongs to becomes ambiguous. Dense passages collapse into clusters. None of this is cosmetic: spacing is how a reader sees the rhythm.

The literature describes results, not procedures. Gould (2011) and Ross (1970) record proportions, clearances and conventional arrangements, but no algorithm; font geometry, kerning, extreme durations and dotted values are left to the engraver's judgement. A program has to make explicit what the tradition leaves tacit, and fitting the result to the width of a system is a further problem, which ADR-0037 solves.

### How Engravers Reason

Before measuring anything, an engraver makes a few judgements that govern everything after. They are seldom stated, but they can be read from engraved music. This ADR formalises them. Where a claim about practice is untested, it is marked as a hypothesis for [the exploration](#the-exploration).

1. **The dominant rhythmic value.** A passage has a prevailing pulse, in quarters or eighths or sixteenths, around which it is read; it sets the visual tempo of the page. Here it is the anchoring duration of the duration table ([H2](#h2-the-anchoring-duration)).
2. **A comfortable norm.** The engraver imagines the passage as it would look if nothing were tight and nothing forced. It is neither a minimum nor the result of a constraint, but the reference against which every later compromise is judged. Here it is each gap's ideal width, the rest length of its spring.
3. **Hierarchy.** Long values read as anchors and short values as motion. This is why spacing grows more slowly than duration. Here it is the shape of the duration table ([H1](#h1-the-duration-table)).
4. **Physical reality intrudes.** Accidentals, dots, ledger lines and articulations may need more room than the norm gives, or limit how far it can be compressed, and the compromise is negotiated from the norm. Here it is the blocking width, and the springs of ADR-0037.
5. **Sacrifice locally; keep the hierarchy as long as possible.** When space is short, the engraver compresses where it costs least and flattens the spacing uniformly only under severe constraint. *Hypothesis.* How compression is shared is ADR-0037's [fitting rule](0037-Measure-Distribution-Optimization.md#fitting-rules); whether the hierarchy the eye reads is the distance between notes or the white space between them is open ([H16](#h16-distance-or-white-space)); whether a bar's long values should give room to its very short ones is [H18](#h18-locality-within-a-bar).
6. **Connecting elements adapt.** Slurs, ties and beams are drawn to the spacing; they do not determine it. Here, Stage 5 places elements within the space Stage 4 gives and never changes note spacing ([ADR-0013](0013-Slur-Formatting.md)). Anything an element needs horizontally is declared before Stage 3.
7. **Small things give way first.** In a tight passage the engraver shortens ledger lines and tucks accidentals closer to their notes before giving up the rhythm; in a loose one they take their comfortable forms. Shortened ledger lines are documented practice; how far accidentals give way is a *hypothesis*. Here it is the minimum form, held in the contour, which Stage 5 expands into whatever space the system has been given ([Atoms and Contours](#atoms-and-contours)).

### Sources of Truth

Three authorities answer different questions:

- **SMuFL metadata**: can this exist without collision? The bounding boxes, cut-outs, anchors and engraving defaults of the active font ([ADR-0006](0006-SMuFL.md), [ADR-0047](0047-Font-Management.md)), read as exact rationals.
- **The engraving literature**, Gould (2011) and Ross (1970): would this be considered correct engraving? Proportions, clearances and conventions. Values recorded from them here are provisional until verified against the books, with page references ([H15](#h15-verification-against-gould-and-ross)).
- **Printed editions in the plate-engraving tradition**: does it look as engravers made it? They are the arbiter wherever the first two leave a choice ([The Exploration](#the-exploration)). The output of other notation programs is not a reference.

Collision-free but cramped is unprofessional; well proportioned but colliding is impossible. Both must hold.

## Decision

### The Four Forms

Every horizontal rule takes one of four forms, each produced before Stage 3:

| Form | What it carries | Produced in | Specified in |
|---|---|---|---|
| Ideal width | Duration, through the duration table, with optical corrections; or an ideal clearance after material without duration | Stage 1 (rhythm, atoms), Stage 2 (per gap) | [Ideal Widths](#ideal-widths), [The Raster and Gap Classes](#the-raster-and-gap-classes) |
| Blocking width | The distance between neighbouring atoms' contours with minimum clearances, the largest across the staves | Stage 1 (contours), Stage 2 (per gap) | [Atoms and Contours](#atoms-and-contours), [Blocking Widths](#blocking-widths) |
| Constraint spanning several columns | A minimum distance over several gaps | Declared in Stage 1 or formed in Stage 2; reduced in Stage 2 | [Constraints Spanning Several Columns](#constraints-spanning-several-columns) |
| Variant at a break | Material that differs where a system starts or ends, or exists only mid-system | Before Stage 3 | [The Bar Boundary](#the-bar-boundary), [Variants at a Break](#variants-at-a-break) |

Elasticity is not a fifth form: it is a parameter of ADR-0037's model, which may differ by class of gap ([Elasticity by Gap Class](#elasticity-by-gap-class)). Nor is a minimum form: it is how an element enters the contour, and so the blocking width, when it can be drawn tighter than it would like to be ([Atoms and Contours](#atoms-and-contours)). User interventions enter in one of the four forms ([User Inputs](#user-inputs)). A horizontal rule that fits none of them is a design problem to be resolved here, not a pass to be added after Stage 2.

### Stages 1 and 2

**Stage 1** runs per measure, in parallel, and is cached by content ([ADR-0029](0029-Global-Hash-Consing.md)). It does the heavy geometric work and no spacing arithmetic. It:

- resolves column assembly and forms atoms ([Atoms and Contours](#atoms-and-contours));
- computes each atom's origin and contours, and the attachment points and contours of material without duration;
- records, for each element with a minimum form, both its minimum and its comfortable form; the contour holds the minimum ([Atoms and Contours](#atoms-and-contours));
- records each staff's rhythm: every note and rest, with its onset and duration as exact rationals, tuplets and dots included;
- records what optical corrections need: stem directions, beams, and the widths of noteheads and rests;
- declares, as geometry, the constraints spanning several columns that its own content implies: which atoms or glyphs, and of what kind (ties, lyrics, elements that reserve room, plugin reservations);
- supplies the geometry of the decorations a measure needs when it starts a system: courtesy accidentals for tied-to notes, tie continuations, a start repeat that may move there; and of its closing barline both as it ends a system and as it stands mid-system ([The Bar Boundary](#the-bar-boundary)).

Stage 1's results depend on the measure's content, the active font and the settings that change glyph geometry. They never depend on the duration table, the anchoring duration, the clearance table, spacing distances such as the minimum tie length, `σ`, or anything in another measure, with one exception: a measure draws before its closing barline a clef change that takes effect at the start of the next measure, so it reads the clef the next measure begins with, and that clef is part of its cache key ([The Bar Boundary](#the-bar-boundary)). That is what keeps Stage 1 parallel, and its caches valid when any spacing parameter changes.

**Stage 2** runs per measure stack, in two steps. First, for each stack, it:

1. merges the staves' onsets into the raster of columns and forms the gaps ([The Raster and Gap Classes](#the-raster-and-gap-classes));
2. computes each gap's ideal width ([Ideal Widths](#ideal-widths));
3. computes each gap's blocking width, forms the constraints between atoms of one staff that are not neighbours, and turns the constraints Stage 1 declared into distances ([Blocking Widths](#blocking-widths), [Constraints Spanning Several Columns](#constraints-spanning-several-columns));
4. applies measure-width overrides, those set by barline drags among them ([ADR-0037](0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms), [User Inputs](#user-inputs));
5. reduces the constraints that lie within the stack, by ADR-0037's procedure.

Then, across neighbouring stacks, it:

6. reduces the constraints that cross a barline, over the gaps of every stack they span;
7. computes each stack's gutter, and the no-break width of its closing boundary, from Stage 1's geometry ([Variants at a Break](#variants-at-a-break));
8. applies the user's column transfers, and the transfers barline drags make between neighbouring stacks ([User Inputs](#user-inputs));
9. produces ADR-0037's stack summaries, and keeps every gap's `w` and `b`, with their provenance, for Stage 4 and for [inspection](#inspection).

**Provenance.** For every blocking width, Stage 2 records what set it: the staff or row of text and the pair of glyph classes that meet, or the constraint spanning several columns that raised it, with its `r_D`. For every ideal width it records how it was formed: the table entries it lies between, the controlling duration, each optical correction, and any transfer. Nothing in the pipeline reads provenance, so it changes no width (Invariant 9).

Beyond its own stack, Stage 2 reads only finished output of earlier work: the anchoring duration, where it is derived from the music ([H2](#h2-the-anchoring-duration)); the previous stack's closing barline, where the leading gap begins and whose start repeat the gutter reserves at a system start ([The Bar Boundary](#the-bar-boundary)); and, in its second step, the first step's gaps of the stacks a constraint spans. All three are forward dependencies; nothing flows back.

A change to the duration table, the anchoring duration, the clearance table, a spacing distance or `σ` re-runs Stage 2 and Stage 3. It re-runs no Stage 1 geometry.

### Atoms and Contours

**Column assembly.** Before any spacing, Stage 1 resolves the horizontal arrangement within each atom:

- seconds within a chord, displaced to the other side of the stem;
- voices at the same onset whose noteheads would collide, offset or, at a unison where the noteheads allow, shared;
- stacked accidentals;
- dots, aligned;
- arpeggio signs;
- articulations, on their side.

All of it is local to one atom in one staff, which is why it belongs to Stage 1 and runs in parallel. The rules are engraving rules ([H5](#h5-column-assembly)). This ADR fixes only where they run and what they produce: the atom's geometry, and from it its contours.

**Origins.** Springs attach at origins ([ADR-0037](0037-Measure-Distribution-Optimization.md#gaps-as-springs), after Hegazy and Gourlay), so an origin is the point the rhythm aligns.

- A note atom's origin is the left edge of its principal notehead column: the noteheads where an undisplaced note would sit, on the stem's normal side (left of an up-stem, right of a down-stem; for a stemless chord, the side its stem would take). Where voices at one onset are offset, it is the principal column of the voice that is not displaced; column assembly decides which that is ([H5](#h5-column-assembly)).
- A rest's origin is the left edge of the rest.
- Material without duration, such as a barline, a clef, a key or time signature or a breath mark, has two attachment points: the left and right edges of its ink. Its own width is a gap of fixed space between them, in ADR-0037's sense: a gap of zero ideal width, which never stretches or shrinks. A barline's thickness, for example, is fixed space within the stack that owns the barline.

**Contours.** Each atom has a left and a right profile. For every horizontal band of height, the profiles give the leftmost and the rightmost ink in that band, relative to the origin.

- Profiles are built from the glyphs' SMuFL bounding boxes, refined by their cut-outs (the `cutOutNE`, `cutOutSE`, `cutOutSW` and `cutOutNW` anchors, which mark rectangular regions at a box's corners that hold no ink), with stems and ledger lines as rectangles.
- In Bravura, 165 glyphs carry cut-outs, among them the common accidentals: the sharp has all four, the flat its two on the right.
- Bands are kept per glyph class (notehead, accidental, dot, stem, flag, ledger line, articulation and so on): where two classes share a height, each keeps its own extreme, because clearances depend on the pair of classes that meet.
- Profiles are step functions with rational breakpoints.
- An atom in a staff drawn at reduced size contributes profiles scaled by that staff's size.
- An element with a minimum form contributes its minimum form ([What Stage 5 may change](#atoms-and-contours), below).

Contours replace ADR-0028's four-directional extents. A box overstates the blocking width wherever a glyph's corner is empty: the notch of an accidental, the open side of a flag, a displaced notehead. Each overstatement makes a gap prestretched that need not be, and the spacing turns irregular for geometry no reader sees. LilyPond computes the minimum distance between neighbouring columns from horizontal skylines, profiles of the same kind ([Prior Art](#prior-art)).

**Ledger lines** are part of the atom and its contours. SMuFL's `legerLineExtension` sets how far a ledger line projects beyond its notehead: in Bravura 0.4 staff spaces on each side, so a ledger line under a black notehead 1.18 staff spaces wide is 1.98 wide. At close spacing the ledger lines of neighbouring notes would touch, and engravers shorten them. A ledger line therefore has a minimum form, the shortest acceptable ledger line, from which the blocking width is computed; Stage 5 lengthens it into the space Stage 4 gives, up to its full extension ([H6](#h6-ledger-lines)).

**Immutability.** Atoms are cached with their contours. Stage 4 positions them; Stage 5 adjusts elements to the space Stage 4 gives and never moves an atom or a column. It may adjust geometry the earlier stages produced, such as a stem length or an articulation's offset, within declared ranges, in one bounded pass that is not recursive and never moves the rhythmic raster ([ADR-0013](0013-Slur-Formatting.md#the-pipeline-is-one-way-and-what-stage-5-may-change)). What ADR-0013 leaves open is how that reconciles with ADR-0028's immutability of atoms. Its account is that a beamed note's stem length is a Stage 5 output rather than a property of the atom; whether that is exactly right, and where the boundary falls for unbeamed stems, is not settled ([ADR-0013](0013-Slur-Formatting.md#whether-stage-5-modification-touches-atom-internal-geometry)).

**What Stage 5 may change.** No column moves after Stage 4, so every horizontal adjustment Stage 5 makes is of one of three kinds, and each element's formatter declares which applies to it ([Formatter Conditions and Plugins](#formatter-conditions-and-plugins)):

- **Within a reserved envelope.** An adjustment bounded by a declared range stays, band by band, within the horizontal extent Stage 1's contours reserved for it. The contour reserves the envelope of the range: for a stem, every height its tip and any flag can reach; for an articulation, every position it can take.
- **From a minimum form into given space.** An element whose comfortable form is wider than the least it can acceptably take declares both. Its contour holds the minimum form, so the blocking widths reflect what the music needs at its tightest. Stage 5 expands it toward its comfortable form only into space Stage 4 has given, checked against everything already placed. A tight system gets the minimum form and a loose one the comfortable form, and nothing flows back. Ledger lines are of this kind, lengthening toward their full extension, and so is an accidental taking its comfortable distance from its notehead where the system leaves room. The minimum and comfortable forms are geometry, set by column assembly and glyph metrics, never by the clearance table or any other spacing parameter, so Stage 1 stays independent of them (Invariant 4).
- **Into whatever room exists.** An element whose condition is loose, such as a text box, reserves nothing, and Stage 5 places it wherever the space allows, checked against everything already placed.

Where two expansions compete for the same space, as the ledger lines of neighbouring notes do, or an accidental and the previous note's ledger line, they share it by a fixed rule, so the result is deterministic whatever order Stage 5 meets them in ([H22](#h22-minimum-forms)).

This is not an exception to the rule that no column moves: a minimum form never asks for room, it only takes room the springs have already given. A tight system loses nothing it needed, and a loose one gains the comfortable forms without a pass back to Stage 3.

Stage 1 can reserve envelopes and minimum forms only if the declared ranges and forms are available to it, which is a requirement on the Slur and Tie Geometry ADR and on beaming.

### The Raster and Gap Classes

**Columns.** Stage 2 merges every staff's onsets into one ordered sequence of rational positions; each distinct onset is a column, tuplets exact. Material without duration that occupies horizontal space also forms columns: barlines; clef, key and time signature changes; breath marks and caesuras. Such a column may share an onset with a note column, as a clef change does with the note it precedes; it then comes first. Where several share an onset they come in a fixed order, provisionally a breath mark or caesura, then a clef change, then a key and a time signature change, then any grace notes, then the note column, though a clef change may instead follow the grace notes ([H19](#h19-order-of-kinds-at-one-onset)). Whether grace notes form columns of their own is [H7](#h7-grace-notes).

**Gaps** run from one column's right attachment point (for an atom, its origin) to the next column's origin or left attachment point. Each gap has a **class**, given by the columns at its ends, which decides how its widths are derived:

- **A gap leaving a column with duration** carries that column's duration space, up to the next column with a later onset ([Ideal Widths](#ideal-widths)). Whether a barline is such a column, so that a bar's last note has its space up to the barline, is [H9](#h9-the-last-note-before-a-barline). The gap's blocking width is the contour distance with the minimum clearance between the glyph classes that meet.
- **A gap leaving material without duration** runs from its right attachment point to the next origin. Its ideal width is the contour distance with the ideal clearance between the glyph classes that meet, and its blocking width the contour distance with the minimum clearance, both from the clearance table ([H4](#h4-clearances)). Gould's space after a barline, as recorded, is of this kind: 1.0 ideally and 0.5 at least, between the barline and the next ink. Material without duration therefore adds its own width and the clearance after it: a clef change before a note lengthens the bar by both.
- **Every gap between columns at different onsets** has a blocking width of at least a minimum separation ([H4](#h4-clearances)), so that two onsets never print at the same place, even where no staff has atoms at both.
- **The class also selects the gap's elasticity** ([Elasticity by Gap Class](#elasticity-by-gap-class)).

**Staves are compared only with themselves.** A contour meets only contours of the same staff, or of the same row of text, such as a lyric line or a line of dynamics. Material that crosses between staves contributes to the contours of every staff it reaches; Stage 1 records its geometry with its measure, and Stage 2 assigns it to the staves.

**Bar rests** are not on the raster. A bar rest constrains its measure's whole span, a constraint across all of the stack's gaps for that staff, and is placed in Stage 5, centred in the space its measure has been given. The width of a multi-bar rest is [H13](#h13-bar-rests-and-multi-bar-rests).

### The Bar Boundary

Material at a barline has up to three forms, as a discretionary break in TeX has: what is printed mid-system, what ends a system there, and what starts the next. ADR-0037 maps the end of a system to the postamble, the start to the preamble and gutter, and what exists only mid-system to the no-break width of the boundary ([ADR-0037](0037-Measure-Distribution-Optimization.md#interface-contract)). Everything else at a barline belongs to a measure in every form:

- **A stack owns its closing barline**, of whatever kind, in the form it takes at a system's end: the gap from its last column to the barline, and the barline's own width as fixed space. Every measure has one, and it is part of the measure's width wherever the measure falls, so the postamble reserves nothing for it. No barline is wider at a system's end than mid-system. The last measure of a piece, and often of a section, ends with a final or a double barline, but that is the measure's own barline, the same wherever the measure falls.
- **What a repeat barline adds mid-system is no-break material.** A combined repeat `:||:` ends a system as its end part, and a start repeat at the boundary as a plain barline. The start part, which mid-system stands at the boundary, is no-break material of that boundary: fixed space, counted only when the boundary falls inside a system. At a break it moves to the start of the next system, where the gutter reserves its full width, less anything the stack already holds ([Variants at a Break](#variants-at-a-break)).
- **A stack owns its leading gap**, from the previous stack's closing barline (its right attachment point) to the stack's first column. Stage 2 reads that barline's type and contour from the previous stack's Stage 1 output.
- **Changes at a boundary belong to measures.** A clef change that takes effect at the start of a measure is drawn before the barline by the measure before it, which includes it on examining the next measure; at a system's end it is the cautionary clef, the same glyph. A key or time signature change is drawn after the barline by the measure it begins, as a column before its first note, just as a change within a measure is a column of it. Each is part of its measure's width in every position, together with the space that separates it from the barline. At a break, the postamble adds cautionary key and time signatures after the barline, and the next system's preamble shows the clef and the key in force, omitting a key or time signature change that the system's first measure draws itself.
- **At a system start there is no barline before the stack.** The preamble and gutter stand in its place, and the leading gap begins where they end.

Two things remain, both [H10](#h10-no-break-material-and-the-system-start). Whether house style omits a key change's cancellation naturals at a system start is open; if it does, they are no-break material by the same mechanism, since the measure is then narrower at a system start than mid-system. How the space at a system start is divided between the preamble, the gutter and the leading gap is open too.

### Ideal Widths

**The duration table** is house-style data: a sorted map from the duration ratio `ρ = d / d_ref` to a width in staff spaces, both rationals.

- Between entries, the width is linear in `ρ`; beyond the first and last entries, it is their value.
- Entries may sit at any duration, dotted and tuplet values included, so a value can be set by measurement wherever it matters. Elsewhere, interpolation supplies it.
- Every positive rational duration therefore has an ideal width, exact and identical on every platform. No logarithm is computed and no floating point is used ([ADR-0037](0037-Measure-Distribution-Optimization.md), exact arithmetic).

Interpolating linearly between powers of two approximates a logarithmic table closely:

- **With a constant step of 1.2 staff spaces per doubling**, the largest difference is about 0.10 staff spaces (0.103), near 1.44 times a power of two. A dotted value, at 1.5 times, differs by 0.10.
- **Under a rule of √2 per doubling**, the largest difference is 1.8 per cent of the width: 0.06 between a quarter of 3.5 and a half.

An entry at the dotted value removes the difference wherever measurement shows it matters. The values themselves are [H1](#h1-the-duration-table).

**The anchoring duration** `d_ref` fixes where the table sits. A table is only a shape: how much more room a quarter gets than an eighth. Something has to fix its position in staff spaces, and since the table is indexed by `ρ`, changing `d_ref` moves every duration along it. How `d_ref` is set is [H2](#h2-the-anchoring-duration). Four things are decided now:

- It is set before Stage 2, from the music or by the user.
- It never depends on systems, which are chosen from the widths it produces.
- It may be derived from a reduction over Stage 1's output across a section or a piece, a forward dependency.
- Where it changes, every stack that uses it is re-spaced, and ADR-0037's [re-optimisation after an edit](0037-Measure-Distribution-Optimization.md#exact-re-optimisation-after-an-edit) falls back to a full pass. The user may pin it, and may begin a new section with a value of its own.

**The controlling duration.** For a gap leaving a column `c` with duration, the controlling duration `d_ctrl` is the shortest duration among the notes and rests, in any staff of the stack, that sound at `c`: those beginning at or before `c` and ending after it. Grace notes are never among them, since their metric duration is zero; if they form columns of their own, those columns take their widths from a grace table ([H7](#h7-grace-notes)). With `c′` the next column with a later onset:

```
w = space(d_ctrl) × (t(c′) − t(c)) / d_ctrl  +  optical corrections
```

A long value therefore spans the shorter values sounding with it. Take a half note in one staff against a quarter and two eighths in another. The gaps are the quarter's space, then an eighth's, then an eighth's: with a quarter at 3.5 and an eighth at 2.5, the half spans 8.5 staff spaces, not the 5.0 it would have alone. The rule is LilyPond's ([Prior Art](#prior-art)). In some combinations it spaces equal values unequally, which is [H17](#h17-the-controlling-duration-rule).

**Optical corrections** are additive adjustments to a gap's ideal width, for configurations the eye misjudges. Candidates:

- Stem directions: an up-stem followed by a down-stem gets extra space, and a down-stem followed by an up-stem less.
- The width of the left atom's notehead or rest compared with a normal notehead: a rest is narrower.

A correction may depend on durations, stem directions, beams and the widths of noteheads and rests, never on attachments (Invariant 2). Each staff contributes its own. Which corrections apply, how large they are, how the staves' corrections combine into a gap's single ideal width, and which gap carries a correction when a staff's pair of atoms spans several gaps, are [H3](#h3-optical-corrections).

**Attachments never change duration space.** Accidentals, articulations, ledger lines and displaced noteheads change contours, and so blocking widths, never the ideal width of a gap leaving a column with duration, nor a controlling duration. (A gap after material without duration is a clearance by nature, and its ideal follows the ink.) An accidental therefore takes room only when its gap's blocking width exceeds the width its spring gives at the system's ratio, and then exactly as much as the collision requires ([ADR-0037](0037-Measure-Distribution-Optimization.md#gaps-as-springs)). This is the engraver's rule, that everything belonging to a note fits within its allocation, made exact, and it is why a note with an accidental does not look late. A dot is different: it changes the duration, and with it the ideal width; its ink, like any other, changes only the contour.

```clojure
(defn duration-space
  "Ideal space, in staff spaces, for duration d under anchoring duration
   d-ref. table is a sorted map from ρ = d / d-ref to width, both rationals
   (house-style data, H1): linear in ρ between entries, the end values beyond
   them. Exact."
  [table d d-ref]
  (let [rho (/ d d-ref)
        [k0 v0] (first (rsubseq table <= rho))
        [k1 v1] (first (subseq table >= rho))]
    (cond
      (nil? k0) v1
      (nil? k1) v0
      (= k0 k1) v0
      :else (+ v0 (* (- rho k0) (/ (- v1 v0) (- k1 k0)))))))

(defn gap-ideal
  "Ideal width of the gap leaving a column with duration: the space of the
   controlling duration d-ctrl, in proportion to the time dt to the next
   column with a later onset, plus the gap's optical corrections (H3)."
  [table d-ref d-ctrl dt correction]
  (+ (* (duration-space table d-ctrl d-ref) (/ dt d-ctrl))
     correction))
```

### Blocking Widths

**Between two atoms of one staff at neighbouring columns**, the blocking width is the least distance between their origins at which no ink of the left atom, with the minimum clearance for the pair of glyph classes that meet, reaches ink of the right atom at the same height. Bands of height count as meeting when they come within a vertical clearance of each other, so that ink at slightly different heights is still kept apart. Ink with a minimum form counts in that form.

**Atoms that are not neighbours.** An atom's ink can reach past its neighbour where the neighbour has none at that height, as a high articulation can over a low note. For every atom, Stage 2 also constrains each earlier atom of the same staff whose right profile can still reach it past the atoms between. It walks back until none can, as LilyPond does ([Prior Art](#prior-art)). These are constraints spanning several columns.

**Clearances** form a table by pair of glyph classes, in staff spaces, for legibility and not merely for avoiding collision ([H4](#h4-clearances)). Each pair has a minimum, used for every blocking width, and, where a gap's ideal width is a clearance, an ideal value as well. Two further values complete it: the vertical clearance between bands, and the minimum separation between columns at different onsets.

**A gap's blocking width** is the largest over the staves, and rows of text, that have atoms at both of its columns, and at least the minimum separation between onsets. A staff with an atom at the left column and none at the right meets its next atom further on. That distance spans several gaps and is a constraint ([Constraints Spanning Several Columns](#constraints-spanning-several-columns)).

```clojure
(defn- bands-meet?
  [a b vpad]
  (and (< (:y0 a) (+ (:y1 b) vpad))
       (< (:y0 b) (+ (:y1 a) vpad))))

(defn blocking-width
  "The least distance between the origins of two atoms of one staff at which
   no ink of the left atom, with its clearance, reaches ink of the right atom.
   right-profile is the left atom's: bands {:y0 :y1 :x :class}, x its
   rightmost ink in the band, one band per glyph class at each height.
   left-profile is the right atom's, x its leftmost ink, negative where ink
   lies left of its origin. Elements with a minimum form appear in that form.
   clearance gives the minimum clearance for a pair of glyph classes (H4);
   vpad is the vertical clearance within which bands at different heights
   still meet. Exact."
  [right-profile left-profile clearance vpad]
  (reduce max 0N
          (for [r right-profile
                l left-profile
                :when (bands-meet? r l vpad)]
            (+ (:x r) (clearance (:class r) (:class l)) (- (:x l))))))
```

### Constraints Spanning Several Columns

Stage 1 declares, as geometry, those its own content implies. Stage 2 forms those between atoms of one staff, since only it knows the columns, turns every declared constraint into a distance from the spacing parameters, and reduces them all to blocking widths by ADR-0037's [procedure](0037-Measure-Distribution-Optimization.md#constraints-spanning-several-columns). They come from:

- **Atoms of one staff that are not neighbouring columns**: either other staves separate them by further columns, or an earlier atom's ink reaches past its neighbour. The distance comes from their contours, as for a blocking width. In polyphonic music this is the commonest case.
- **Ties.** A tie needs at least the minimum tie length, a piece setting, between its noteheads.
- **Hairpins, glissandi and tremolos between two notes.** Each needs a minimum length between its ends.
- **Lyrics.** Neighbouring syllables of one line need their minimum gap, or room for a hyphen. How a syllable aligns with its note is [H12](#h12-elements-that-reserve-room).
- **Elements whose formatter condition reserves horizontal room** ([Formatter Conditions and Plugins](#formatter-conditions-and-plugins)).
- **Material crossing between staves.**
- **Bar rests**, across their measure's whole span.

**Constraints across a barline**, such as a tie, a melisma or a word, are reduced in Stage 2's second step, over the finished gaps of every stack they span. They are applied in a fixed order across the piece, by first gap and then by last, as ADR-0037 orders overlapping constraints within a stack, so every stack sees the same result. A constraint can span several barlines; it then couples every stack it spans.

If a boundary such a constraint crosses becomes a break, the reduction is conservative, as ADR-0037 states. There the constraint may need less, as a lyric extender does, or what it needs at the system's end and at the next system's start is carried by the postamble and the gutter. A split tie's two halves, a hyphen at a system's end and a syllable that overhangs a system's start are of this kind ([H12](#h12-elements-that-reserve-room)).

### Formatter Conditions and Plugins

An element's formatter declares its placement condition as data: for example, "directly above or below the note" for a staccato, or "loosely adjacent" for a text box. One declaration serves two phases.

- **Spacing, Stages 1 and 2.** The condition decides which form the element takes:
  - a condition that travels with the note makes the element part of the atom's contour, on the side Stage 1 has fixed;
  - a condition with a minimum form makes the element part of the atom's contour in that form, and declares the comfortable form Stage 5 may expand it toward;
  - a condition that must not run into a neighbour becomes a constraint spanning several columns;
  - a loose condition reserves nothing.
- **Placement, Stage 5.** The solver places the element within the space Stage 4 has given, satisfying the condition, and never moves an atom or a column ([ADR-0013](0013-Slur-Formatting.md)). An element that is part of an atom's contour may move only within the extent its contour reserved. An element with a minimum form may expand only into space Stage 4 has given, sharing contested space by the fixed rule of [H22](#h22-minimum-forms). An element that reserved nothing may go wherever the space allows, checked against everything already placed.

Stage 5 can only use room that exists. Anything an element must have, it declares in the spacing phase; anything it would merely like, it may declare as a comfortable form, and receives where the system allows.

ADR-0028's Stage 1 [atom hook](0028-Hierarchical-Rendering-Pipeline.md) produces atoms, with their contours and any minimum forms, and may also declare constraints spanning several columns. A plugin participates in both phases through its hooks; it cannot add a pass after Stage 2.

### Variants at a Break

ADR-0037 specifies their structure: the preamble and postamble computed on demand and cached by combination, and the gutter and no-break widths known before breaking. Their widths are computed from the same contours and clearances as everything else:

- **Preamble**: the clef and the key signature in force, with the clearances between them, as the largest across the staves. A key or time signature change the system's first measure begins with is that measure's, and the preamble omits it ([The Bar Boundary](#the-bar-boundary)). Whether the preamble carries space after its ink is [H10](#h10-no-break-material-and-the-system-start).
- **Postamble**: the cautionary key and time signatures, after the closing barline. The barline, and a cautionary clef before it, belong to the last measure ([The Bar Boundary](#the-bar-boundary)), so the postamble holds nothing for them.
- **Gutter**: computed by Stage 2 from Stage 1's geometry: courtesy accidentals for tied-to notes, tie continuation arcs, and a start repeat that a break moves to the system's start, the start part of a combined repeat or a start repeat at the boundary, at its full width; each for the room it needs beyond what the stack already holds, as ADR-0037 specifies.
- **No-break width**: computed by Stage 2 for each stack's closing boundary, from Stage 1's geometry: what the closing barline adds mid-system beyond the form it takes at a system's end ([The Bar Boundary](#the-bar-boundary)), and cancellation naturals where house style omits them at a system start ([H10](#h10-no-break-material-and-the-system-start)). ADR-0037 counts it only at the boundaries inside a system ([ADR-0037](0037-Measure-Distribution-Optimization.md#interface-contract)).

### Elasticity by Gap Class

ADR-0037's baseline elasticity applies to every gap: stretch `σ × w`, shrink `max(0, w − b)`. A class of gap may need elasticity of its own. A barline gap that yields "only in cramped conditions" is the first candidate, and it is a matter of shrinking: a shrink weight per class is ADR-0037's alternative 3, generalised from duration to class. A stretchability per class is possible too, but it changes ADR-0037's interface: each prestretched pair carries its own `σ`, and the ordinary sum becomes `Σ σ_j × w_j`. Until [H11](#h11-elasticity-by-gap-class) is settled, the baseline applies to every class.

### User Inputs

**Column drags.** The user sees the columns on the page and can drag them. A drag is stored as a transfer of ideal width between the column's two neighbouring gaps: the gap on the side the column moves toward loses `Δ`, and the gap on the other side gains it.

- **The transfer.** Under the baseline elasticity, with both gaps ordinary, moving a column by `δ` at ratio `r` takes `Δ = δ / (1 + σr)` when the system is stretched and `Δ = δ / (1 + r)` when it is compressed. In general, as when a neighbouring gap is prestretched, `Δ` is the least transfer that moves the column by `δ` under ADR-0037's springs, found exactly.
- **The clamp.** A drag is clamped at the current ratio, so that neither gap becomes narrower than its blocking width. At `r = −1` every gap is at its blocking width and no column can move.
- **What changes.** While both gaps' ideal widths stay at or above their blocking widths, the stack's summary is unchanged: a transfer leaves `Σ w`, `Σ` shrink and `Σ` stretch where they were. Every candidate system's ratio, and so the breaks, stay as they were, and only the dragged column moves.
- **When the summary changes.** Under stretching, the clamp can let the losing gap's ideal width fall below its blocking width, while its width at the current ratio stays above it. This system's ratio is still unchanged, but the gap is now prestretched and the stack's summary changes, so the breaks may change. It is a real request for more room.

Each transfer is attached to its column, identified by measure, rhythmic position and kind, since a clef change can share an onset with a note, and for a grace column by its offset as well, since a grace group has several columns at one onset ([H19](#h19-order-of-kinds-at-one-onset)). Stage 2 applies it after the reductions, and clamps it there so that no ideal width falls below zero, ADR-0037's precondition: the gaps it was made against may have changed since. A reduction raises blocking widths only, so every constraint holds whatever the ideal widths, and a transfer can neither break one nor change what a reduction reserved. A transfer persists until the user removes it, or the engine does: when its column no longer exists, or when its measure is reformatted ([H14](#h14-removal-of-column-transfers)).

```clojure
(defn drag->transfer
  "The ideal width moved between a column's neighbouring gaps when the user
   drags it by delta (positive to the right) in a system with adjustment
   ratio r: the left gap gains it and the right gap loses it. Both gaps
   ordinary, baseline elasticity with stretchability sigma. nil at r = -1,
   where no column can move. The caller clamps delta so that neither gap
   becomes narrower than its blocking width."
  [delta r sigma]
  (cond
    (>= r 0) (/ delta (+ 1 (* sigma r)))
    (> r -1) (/ delta (+ 1 r))
    :else nil))
```

**Barline drags.** A barline belongs to the measure it closes ([The Bar Boundary](#the-bar-boundary)). Dragging it makes one of two edits, and what persists is that edit, not a width: at another system's ratio the measure's width differs. ADR-0037 specifies both ([Editorial Control Mechanisms](0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms)).

- **Mode (i): a transfer between the measure and the next.** The drag transfers ideal width between measure N and measure N+1, distributed over each measure's gaps, computed exactly against the current ratio as a column transfer is, so the barline lands under the pointer and the system's ratio is unchanged. It is one transfer attached to the barline, identified by measure, not two overrides, so that the coupling survives later edits. It is clamped where the gaps of the measure that loses width reach their blocking widths at the current ratio, and Stage 2 applies and clamps it as it does a column transfer. The two measures' summaries change individually, so the breaks may change after release.
- **Mode (i) at a system's end.** Where N+1 is on the next system, the drag changes that system's width instead, and the barline follows the pointer. The width is anchored to the measure that ends the system, and has no effect if that measure no longer ends one, so the system stays frozen whatever the setting ([ADR-0037](0037-Measure-Distribution-Optimization.md#frozen-systems)).
- **Mode (ii): the measure's width.** The drag sets N's ideal-width override, solved exactly so that the barline lands under the pointer after the system refits. The system's other gaps give or take room through their springs, the next measure is not edited, and the breaks may change. At a system's end, where the barline cannot move, the measure changes width by the distance dragged and its start moves.

While a drag lasts its system is frozen; whether the freeze remains afterwards is a setting, off by default ([ADR-0037](0037-Measure-Distribution-Optimization.md#frozen-systems)). Both modes are also reachable from the keyboard in fixed steps, each an ordinary edit. Which mode is the default, and how the other is chosen, is [H21](#h21-barline-drag-modes).

**Measure-width overrides**, such as "stretch this measure", are specified by ADR-0037's [Width Overrides](0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms). **The anchoring duration** may be pinned, and a new section begun with a value of its own ([H2](#h2-the-anchoring-duration)). **Element offsets**, such as dragging a dynamic or a text, are Stage 5 placements relative to the element's anchor. They are not spacing inputs.

**Drags are edits.** Every update of a drag is a backend edit, followed by a fetch of the paintlists it invalidates, as every visible change is under [ADR-0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md); no frontend draws a preview of a drag. Updates therefore arrive at the cadence of the pipeline's batched formatting ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md#client-server-event-coordination)) and of invalidation delivery ([ADR-0031](0031-Frontend-Event-Driven-Architecture.md)), not at the pointer's: roughly six to ten updates a second, which is sufficient, so drags have no regime of their own ([H20](#h20-drags-as-edits)). Each update carries the value for the pointer's current position, never an increment, and one update is in flight at a time. Each is a transaction, and the whole drag is one undo step, under ADR-0015's rule for continuous gestures ([ADR-0015](0015-Undo-and-Redo.md#continuous-gestures-one-undo-step)).

### Inspection

Inspection shows the springs and why each has its widths. Its overlays are drawn from the cached results of Stages 2 to 4, with Stage 2's provenance ([Stages 1 and 2](#stages-1-and-2)). The backend generates them as paintlists, as it does everything that is drawn ([ADR-0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)); the frontend only requests and draws them.

- **The raster lane** shows each gap's spring, free or held; what holds a held one, from provenance; the ratio at which it is released, its kink; and each gap's share of the system's adjustment.
- **A lane per staff and per row of text** shows, for each pair of consecutive atoms, the distance given, the least distance needed and the ideal the pair would have alone, marked slack, tight or violated. Invariant 1 excludes a violation; the mark exists so that a failure is seen.
- **Minimum forms** are marked where Stage 5 drew an element short of its comfortable form, with how much of the way it got, so that a ledger line or accidental cut back by a tight system is visible as such.
- **The system's width as a function of `r`**, with its kinks and the tolerance.

In a full score, lanes are shown only for the staves that bind, or on demand. A layer for engravers states the same things in engraving terms, without `r`: how much room a gap has given up or gained, what holds it, and at what looseness it would be released. Line style distinguishes the gap classes: duration springs, clearance springs, fixed space, and grace springs if grace notes form columns ([H7](#h7-grace-notes)). Inspection never offers to disable the reduction of constraints spanning several columns ([ADR-0037](0037-Measure-Distribution-Optimization.md#constraints-spanning-several-columns)).

### Invariants and Property Tests

The boundary between Stages 1–2 and Stage 3 is specified by properties, tested generatively over random measures and random stacks. They are what lets the exploration change values without changing the architecture.

1. **No collision at any feasible ratio, before or after Stage 5.** For any rational `r ≥ −1`, placing the columns by ADR-0037's `g_j(r)` leaves no two contours of one staff or row of text overlapping with their minimum clearances, whether or not their atoms are neighbours, and no two columns at different onsets closer than the minimum separation. The same holds for what Stage 5 draws: an adjustment within its declared range keeps an atom's ink inside its contours; an element Stage 5 expands from its minimum form stays within the space Stage 4 gave, clear of everything already placed; and a loose element is placed clear of everything already placed ([Atoms and Contours](#atoms-and-contours)). The test draws the adjustments from their declared ranges, and the expansions from the full range between minimum and comfortable forms.
2. **Attachments never change duration space.** With durations and stem directions fixed, adding or removing accidentals, articulations or ledger lines, or displacing noteheads, changes no controlling duration and no ideal width of a gap leaving a column with duration. A grace group, if grace notes form columns ([H7](#h7-grace-notes)), may divide the duration space before its main note among its own gaps, but never changes its total.
3. **The duration table is monotone and sub-linear.** Its width never decreases as the duration grows, and its width divided by duration never increases.
4. **Stage 1 is independent of spacing parameters.** Changing the duration table, the anchoring duration, the clearance table, a spacing distance or `σ` changes no Stage 1 output, minimum and comfortable forms included.
5. **Determinism.** Identical content gives identical contours, summaries and gap lists, whatever the order in which staves, measures and stacks are processed. Stage 5's expansions of minimum forms are identical whatever the order in which it meets them ([H22](#h22-minimum-forms)).
6. **Scale invariance.** Widths are in staff spaces at the system's staff size; changing that size changes none of them. A staff drawn at reduced size contributes contours scaled by its ratio, and text sized in absolute units is converted at the system's size.
7. **Drag neutrality.** A transfer that keeps both gaps' ideal widths at or above their blocking widths leaves the stack's `:min`, `:natural`, `:ideal`, `:ordinary-ideal` and `:prestretched` unchanged.
8. **Sound reduction.** After Stage 2, every constraint spanning several columns holds at every `r ≥ −1`.
9. **Provenance is inert.** Recording provenance or not changes no width, summary, ratio or break.
10. **Minimum forms never ask for room.** Replacing an element's comfortable form by a wider one, with its minimum form unchanged, changes no width, summary, ratio or break; only what Stage 5 draws.

## The Exploration

The architecture is fixed so that the exploration can only change values and inputs, never add a pass. It is how the open items close.

- **Parameters are data.** The duration table, the anchoring rule, the clearance table, the optical corrections, the elasticities and the minimum and comfortable forms live in a house-style structure. Each value records its source, its date and its status: provisional, verified against the literature, or measured.
- **The reference is printed engraving.** Editions in the plate-engraving tradition are measured: the staff space from the staff lines; the distance from one notehead's left edge to the next, and the white space between them; each for every duration, in loose and in tight systems of the same edition. Each measurement records the edition, page and system. The output of other notation programs is not used.
- **The procedure** is measurement, then a value in the house-style data, then the register entry updated with its date and evidence. The property tests hold throughout.

What it decides is listed in [Open Items](#open-items). Several items share evidence. The same measurements of loose and tight systems settle the duration table ([H1](#h1-the-duration-table)), the behaviour of very short values ([H8](#h8-very-short-values-under-stretching)), what the eye keeps under compression ([H16](#h16-distance-or-white-space)) and how far small things give way ([H22](#h22-minimum-forms)).

## Open Items

| ID | Item | Form | Stage | Decided by |
|---|---|---|---|---|
| [H1](#h1-the-duration-table) | The duration table | Ideal width | 2 | Measurement |
| [H2](#h2-the-anchoring-duration) | The anchoring duration | Ideal width | Before 2 | Reasoning, then measurement |
| [H3](#h3-optical-corrections) | Optical corrections | Ideal width | 2 | Measurement |
| [H4](#h4-clearances) | Clearances | Blocking width, ideal width | 2 | Literature and measurement |
| [H5](#h5-column-assembly) | Column assembly | Contour | 1 | Literature |
| [H6](#h6-ledger-lines) | Ledger lines | Contour (minimum form) | 1, 5 | Literature and measurement |
| [H7](#h7-grace-notes) | Grace notes | Raster, ideal width, contour | 1–2 | Measurement |
| [H8](#h8-very-short-values-under-stretching) | Very short values under stretching | Ideal width | 2 | Measurement |
| [H9](#h9-the-last-note-before-a-barline) | The last note before a barline | Gap class | 2 | Measurement |
| [H10](#h10-no-break-material-and-the-system-start) | No-break material and the system start | Variant at a break | 2–3 | Design, with ADR-0037 |
| [H11](#h11-elasticity-by-gap-class) | Elasticity by gap class | Parameter | 2–4 | Measurement, with ADR-0037 |
| [H12](#h12-elements-that-reserve-room) | Elements that reserve room | Constraint | 1–2 | Design and measurement |
| [H13](#h13-bar-rests-and-multi-bar-rests) | Bar rests and multi-bar rests | Constraint | 1–2 | Literature |
| [H14](#h14-removal-of-column-transfers) | Removal of column transfers | User input | 2 | Design |
| [H15](#h15-verification-against-gould-and-ross) | Verification against Gould and Ross | All | — | Literature |
| [H16](#h16-distance-or-white-space) | Distance or white space | ADR-0037 fitting rule | 3–4 | Measurement |
| [H17](#h17-the-controlling-duration-rule) | The controlling-duration rule | Ideal width | 2 | Reasoning and measurement |
| [H18](#h18-locality-within-a-bar) | Locality within a bar | Ideal width | 2 | Measurement |
| [H19](#h19-order-of-kinds-at-one-onset) | Order of kinds at one onset | Raster | 2 | Literature |
| [H20](#h20-drags-as-edits) | Drags as edits | User input | — | Settled: design, with ADR-0015, ADR-0028 and ADR-0038 |
| [H21](#h21-barline-drag-modes) | Barline drag modes | User input | — | Design |
| [H22](#h22-minimum-forms) | Minimum forms | Contour | 1, 5 | Literature and measurement |

### H1: The duration table

The table's entries, dotted and tuplet entries included. Two starting points:

- **As recorded from Gould and Ross**, provisional ([H15](#h15-verification-against-gould-and-ross)): sixteenth 1.3–1.5, eighth 2.5, quarter 3.5, half 5.0, whole 4.8–5.0.
- **LilyPond's default**, with an anchoring duration of an eighth ([Prior Art](#prior-art)): thirty-second 1.5, sixteenth 1.8, eighth 2.4, quarter 3.6, half 4.8, whole 6.0.

The recorded values are not self-consistent:

- A rule of √2 per doubling, a constant factor, fits the eighth, quarter and half. It would give 1.75 for the sixteenth and 7.0 for the whole note.
- A constant step per doubling, the logarithmic form LilyPond and Gourlay use, fits those three as well, and fits the recorded sixteenth better.
- At 1.3–1.5 the sixteenth sits at or near its collision limit: Bravura's black notehead is 1.18 wide, leaving 0.1–0.3 of air.
- The recorded whole note, at 4.8, is smaller than the half note at 5.0, which the table cannot hold (Invariant 3). The observation belongs to [H9](#h9-the-last-note-before-a-barline).

These recorded values presuppose a prevailing value of about a quarter ([H2](#h2-the-anchoring-duration)).

### H2: The anchoring duration

Whether `d_ref` is fixed, set by the user, or derived from the music; over what scope (bar, section, piece); and by what statistic. What is already fixed is listed under [Ideal Widths](#ideal-widths).

Starting point: LilyPond's. It takes the shortest duration that starts in each measure, uses the most frequent of these across the piece, and never goes above a dotted eighth (`base-shortest-duration`, 3/16 by default), with sections the user can begin.

This item needs reasoning in depth before measurement:

- A fixed anchor leaves fast music at its collision limit at its ideal widths, with no room to compress and a cost that cannot see the crowding.
- An anchor derived from the music couples a section, so one edit can re-space it.
- The engraver's dominant value is a pulse, whereas LilyPond's statistic is the most common shortest value. Which of the two the anchor follows is part of this item.

### H3: Optical corrections

Which corrections apply and how large they are; how each staff's corrections combine into a gap's single ideal width; and which gap carries a correction when a staff's pair of atoms spans several gaps.

Starting points from LilyPond:

- **The correction for opposite stem directions.** It is 0.5 staff spaces for stems that overlap completely and is scaled by how far they overlap. It is halved before a barline, replaced by a correction of its own under a kneed beam, and omitted after an unbeamed note with a long flag. LilyPond's source cites Wanske for it.
- **A correction for stems in the same direction**, 0.25. LilyPond omits it when accidentals stick out of the right-hand note; under Invariant 2 that condition cannot be adopted as it stands.
- **The adjustment of the ideal width by the actual width of the left notehead or rest.**
- **Combining staves.** LilyPond averages the staves' ideal distances, takes the largest of their minimums, and keeps the result at least 0.3 above that minimum.

### H4: Clearances

The table of clearances between pairs of glyph classes, minimum and ideal; the vertical clearance between bands; and the minimum separation between columns at different onsets. Starting value: after a barline, 1.0 ideally and 0.5 at least (Gould, as recorded). The literature describes accidentals largely through examples rather than measurements, especially in chords.

### H5: Column assembly

The rules for seconds in chords, offsets between voices (including which voice is displaced), shared unisons, stacking accidentals, aligning dots and placing arpeggio signs. They may warrant a decision of their own. This ADR fixes that they run in Stage 1 and produce the atom's geometry, including, for an accidental, its minimum and comfortable distances from its notehead ([H22](#h22-minimum-forms)).

### H6: Ledger lines

The shortest acceptable ledger line, which is the ledger line's minimum form and from which blocking widths are computed, and how Stage 5 lengthens ledger lines into the space Stage 4 gives, up to their full extension. Neighbouring ledger lines that compete for the same space share it by [H22](#h22-minimum-forms)'s rule.

### H7: Grace notes

Whether grace notes are part of the main note's atom or form columns of their own, and their spacing. The starting point:

- **Columns of their own, timed backwards from the main note.** A grace note's offset is its position within its group as the timewalk gives it, counted from zero by written durations ([ADR-0014](0014-Timewalk.md) §5), minus the group's total written duration: minus its own written duration and those of the grace notes after it, so that the group ends at its main note. Inside tuplets the timewalk scales those positions by the product of the ratios of the tuplets that contain the group (`walk-item!` in `ooloi.shared.ops.timewalk`), and the total is taken on the same scale. Stage 1 derives the offsets, and they order raster columns and nothing else. The positions [ADR-0035](0035-Remembered-Alterations.md) gives grace notes for accidentals and playback depend on tempo, and are never used for spacing.
- **Width from the preceding gap.** The group's ideal width is taken out of the duration space of the gap before its main note, at most all of it, so that the total does not change (Invariant 2) and the bar lengthens only where blocking widths require it. A group last in its measure takes it from the bar's last gap, which runs to the barline ([H9](#h9-the-last-note-before-a-barline)).
- **At the start of a measure.** A group at position 0 of a measure after the first is displayed after the barline, though it sounds at the end of the previous measure ([ADR-0035](0035-Remembered-Alterations.md)). What precedes it is a clearance, after the barline or after a change at the start of the measure, or the preamble and gutter at a system start, so the stack holds no duration space before it, and the group adds its width as material without duration does. Taking the width from the previous measure's last gap would cross the stack boundary, against [The Bar Boundary](#the-bar-boundary) and H9's starting point.
- **Elasticity.** Grace gaps have no stretchability, so a group keeps its spacing as the system loosens. That changes ADR-0037's interface, and belongs to [H11](#h11-elasticity-by-gap-class).
- **Widths.** Grace notes take no part in the controlling duration ([Ideal Widths](#ideal-widths)); grace columns take their widths from a grace table, starting from LilyPond's values, 1.6 and 0.8 in place of 2.0 and 1.2.
- **After-graces.** A grace group always precedes an item, so after-graces attach to the following column rather than to a fixed fraction of the previous note's duration. Where engravers place them is a question for the plates.
- **Across staves**, grace columns merge by offset, so groups with different written durations share no columns, and their last notes need not align. This too is a question for the plates.

This keeps what LilyPond's `strict-grace-spacing` intends, that grace notes do not lengthen the bar, while keeping their columns in the raster, so that their accidentals count and their room is checked ([Prior Art](#prior-art)).

### H8: Very short values under stretching

Values whose ideal width lies below their blocking width are prestretched. They stay at their blocking width as the system stretches, until their kink, while everything else gains space. Whether a run of thirty-seconds should stay packed in this way, or the table should keep very short values above their blocking widths, is a matter of the table ([H1](#h1-the-duration-table)), settled by measurement.

The tolerance bounds how far a system stretches. At tolerance 200, ADR-0037's documented starting point, no system chosen within the tolerance has `r` above about 1.26, so in such a system a gap whose kink lies above that is never released ([ADR-0037](0037-Measure-Distribution-Optimization.md#calibrating-the-adjustment-ratio)). A frozen system, and a system of a part broken without the tolerance because no segmentation of it met it, can stretch further. With an eighth as the anchoring duration and LilyPond's table, beamed thirty-seconds have an ideal width of 1.5. Before a note carrying a double flat the kink is about 1.77 even with no clearance, and with provisional clearances of 0.2 the gaps before flats (1.31) and sharps (1.44) are held too. The anchoring duration moves this ([H2](#h2-the-anchoring-duration)): with a sixteenth, a thirty-second is 1.8 and the double flat's kink lies between 1.14 and 1.58, so the gap is held unless its two clearances come to less than about 0.11; with a thirty-second, it is 2.4, and every such gap is released.

### H9: The last note before a barline

There are two options:

- **Up to the barline.** The bar's last note has its space up to the barline, with the barline's clearance after it. LilyPond measures to the barline, and this ADR's gap classes state it as the starting point.
- **Up to the next bar's first note**, with the barline placed within that space. This would carry part of the last note's space into the next stack's leading gap, across the stack boundary, which [The Bar Boundary](#the-bar-boundary) would then have to allow.

The recorded observation that a whole note alone in its bar is spaced like a half note bears on the choice.

### H10: No-break material and the system start

Everything at a barline belongs to a measure in every form ([The Bar Boundary](#the-bar-boundary)): the barline in the form it takes at a system's end, a clef change drawn before it, and a key or time signature change drawn after it. The time signature at the start of the piece is the first measure's in the same way. What exists only mid-system is no-break material in TeX's sense.

- **Repeat barlines: decided.** What a repeat barline adds mid-system beyond the form it takes at a system's end, the start part of a combined repeat or a start repeat at the boundary, is fixed space whose width depends only on its boundary. Each stack carries its closing boundary's no-break width, and the DP subtracts their sum over a segment's interior boundaries from the available width, a prefix sum, so the DP stays exact and each evaluation O(1) ([ADR-0037](0037-Measure-Distribution-Optimization.md#interface-contract)). At a system start, the gutter reserves the start repeat's full width, less anything the stack already holds.
- **Cancellation naturals: conditional on house style.** A key change shows cancellation naturals mid-system. At a system start, house style may omit them, since the previous system's cautionary has shown them; the measure is then narrower at a system start than mid-system. If house style omits them, they are no-break material by the same mechanism: the measure enters its stack without them, and they are the no-break width of the boundary before it. If it keeps them, they are part of the measure in every position. Which it does is open.
- **The space at a system start**, and how it is divided between the preamble, the gutter and the leading gap. ADR-0037's preamble includes the space after the key signature, which the leading gap also provides. Starting point: the preamble ends at its ink, and the leading gap supplies the space after it; the gutter reserves any difference between the leading gap's system-start and mid-system forms.

### H11: Elasticity by gap class

Whether any class of gap, the barline gap first, needs elasticity of its own:

- **On the shrinking side**, a weight per class depends on ADR-0037's alternative 3.
- **On the stretching side**, a stretchability per class changes ADR-0037's interface.
- **Drags.** A column dragged between gaps of different classes no longer leaves the stack's stretch unchanged.

Two further candidates:

- **The clearance gap after material without duration.** Under the baseline it stretches by `σ` times its ideal clearance and shrinks with everything else, so the space after a barline or a clef change grows as a system loosens. A measure-width override scales it too ([ADR-0037](0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms)).
- **The grace class**, with no stretchability ([H7](#h7-grace-notes)). A gap with `σ = 0` has no kink, so a held grace gap is never released, and ADR-0037's `add-kinks` would have to pass over it as it passes over a gap of zero ideal width. Under stretching, the span from the note before a grace group to its main note then grows by `σ` times the group's ideal width less than the same span without the group, unless the rest of the preceding gap takes over the group's stretch.

The share bars of the inspection view, each gap's share of the system's adjustment ([Inspection](#inspection)), are the instrument for judging them.

### H12: Elements that reserve room

Several questions belong here:

- Which elements reserve horizontal room by default: dynamics, text, chord symbols, fingerings, figured bass.
- The minimum lengths of hairpins, glissandi and tremolos between notes.
- The room for laissez-vibrer ties.
- The minimum gaps of lyrics, and how a syllable aligns with its note.
- What a split tie, a hyphen at a system's end or a syllable overhanging a system's start needs from the postamble and gutter.

### H13: Bar rests and multi-bar rests

The space a bar rest needs within its measure, and the width of a multi-bar rest.

### H14: Removal of column transfers

What counts as reformatting a measure, so that the engine removes its transfers. This is specified with the editing commands. Removing a transfer is a change to layout data made in response to an edit, so it is undone with that edit, as ADR-0037 requires of every such change ([ADR-0037](0037-Measure-Distribution-Optimization.md#stability-term)).

### H15: Verification against Gould and Ross

The values recorded here as Gould's and Ross's have not been checked against the books: the duration table in [H1](#h1-the-duration-table), the barline clearance in [H4](#h4-clearances), and the orders [H19](#h19-order-of-kinds-at-one-onset) records from Gould pp. 7, 93 and 436, which were read in excerpts and are provisional until checked against the book. Each needs a page reference before it is called verified.

### H16: Distance or white space

The two candidates for sharing compression behave differently:

- **ADR-0037's baseline** shrinks each gap in proportion to its room above its minimum. White space then shrinks fairly evenly, and the ratios between notehead distances flatten.
- **Its alternative 3 with `v ≡ 1`** keeps the distance ratios exactly until gaps block, and squeezes the white space around short notes hardest.

Which one engravers kept is measurable in printed editions: the same values in loose and tight systems of one edition, with distance and white space each measured. The choice belongs to ADR-0037; the evidence comes from here.

### H17: The controlling-duration rule

The shortest note sounding at a column sets its gap's space. Where another staff's onsets fall inside a value, the rule can space equal values unequally.

Take a dotted quarter and an eighth against two quarters, with a quarter at 3.5 and an eighth at 2.5. The gap after the second quarter's onset is controlled by that quarter until the eighth begins (1.75), and by the eighth after it (2.5). The two quarters therefore get 3.5 and 4.25.

Whether that is right, or each staff's own rhythm should be weighed, is open. LilyPond shares the rule.

### H18: Locality within a bar

ADR-0037 asks who pays when a bar needs more room than its proportions give, such as a run of thirty-seconds between a half note and a quarter. Its answer is that, under one adjustment ratio, every gap in the system pays.

Locality would have the long values of the same bar give room to the short ones before any fitting. That is a rule on ideal widths within a bar, formed in Stage 2. Whether it is needed is settled by measurement. The anchoring duration ([H2](#h2-the-anchoring-duration)) is a different matter: an anchor per bar would widen the whole bar, long values included.

### H19: Order of kinds at one onset

Columns that share an onset come in a fixed order, so that a column transfer can name its column ([User Inputs](#user-inputs)). The provisional order is a breath mark or caesura, then a clef change, then a key signature change, then the grace notes by offset, if they form columns ([H7](#h7-grace-notes)), then the note column; a clef change may instead follow the grace notes, as below. A time signature changes only at the start of a measure ([ADR-0033](0033-Time-Signature-Architecture.md)), where it follows the key signature; a clef change there is drawn before the barline ([The Bar Boundary](#the-bar-boundary)).

What the literature reached so far says, the Gould pages read in excerpts and provisional ([H15](#h15-verification-against-gould-and-ross)):

- **A breath mark first.** Gould places it just before the following note or barline (p. 436), and LilyPond's horizontal order before a clef (`break-align-orders` in `scm/define-grobs.scm`). Placed there, as a column of its own at the following onset, it closes the space of the note before it and lengthens the bar by its width and the clearance after it. Placed instead at the end of the note it follows, it travels with that note, as part of its atom's contour ([Formatter Conditions and Plugins](#formatter-conditions-and-plugins)), within that note's space, and takes room only where it would collide. Ooloi supports both; which applies, by house style or for each breath mark, is not yet specified.
- **A clef before a key signature**, as at a barline: Gould (p. 93) and LilyPond's order agree.
- **A clef before or after grace notes.** A clef before the grace notes is the default of the MNX format, which records a clef after the grace notes, and a clef within a run of them, as cases of their own. One engraved example in Gould (p. 7) places a grace note before a clef change. The two orders read differently: a clef before the grace notes applies to them, while one after them applies only from the main note, so the grace notes are read in the clef in force before it. Ooloi supports both; how a score records which order a clef change takes is not yet specified.
- **Not found**: where a key signature goes relative to grace notes, and the order of a breath mark and a caesura at one point. Gould has been read only in parts.

These are starting points; the literature and printed engravings decide ([Sources of Truth](#sources-of-truth)).

### H20: Drags as edits

**Settled.** Every update of a drag is a backend edit ([User Inputs](#user-inputs)), so a drag updates at the cadence of the pipeline's batched formatting, about 100 ms ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md#client-server-event-coordination)), and of invalidation delivery, about 50 to 100 ms ([ADR-0031](0031-Frontend-Event-Driven-Architecture.md)), not at the pointer's: roughly six to ten updates a second. That cadence is sufficient, and drags have no regime of their own. The documented latency target for a visible edit, under 150 ms at the 95th percentile ([ADR-0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)), sets no rate.

Each update is a transaction, and the drag is one undo step however many transactions it takes, under ADR-0015's rule for continuous gestures ([ADR-0015](0015-Undo-and-Redo.md#continuous-gestures-one-undo-step)): the gesture carries an identifier, each of its transactions amends the entry its first one pushed, and another client's entry coming on top splits it in two. Each update carries the value for the pointer's current position, never an increment, and one update is in flight at a time, since the server runs a client's requests concurrently.

### H21: Barline drag modes

Which of a barline drag's two modes ([User Inputs](#user-inputs)), the transfer between the measure and the next or the measure's own width, a drag takes by default, and how the other is chosen: a modifier key or a toggle.

### H22: Minimum forms

Which elements besides ledger lines have a minimum form, what each element's minimum and comfortable forms are, and the fixed rule by which expansions that compete for the same space share it ([Atoms and Contours](#atoms-and-contours)).

- **Ledger lines**: the shortest acceptable length, and the full extension SMuFL's `legerLineExtension` gives ([H6](#h6-ledger-lines)).
- **Accidentals**: the least and the comfortable distance from an accidental to its notehead, set by column assembly ([H5](#h5-column-assembly)). Whether accidentals in a stack also have a compact and a comfortable arrangement, or only their distance from the noteheads gives way, is part of this item.
- **Other candidates**: dots and their distance from the notehead, articulation offsets beside the note, and the arpeggio sign's distance from the chord.
- **Sharing.** Competing expansions, such as the ledger lines of neighbouring notes, or an accidental and the previous note's ledger line, share the free space by a rule that depends only on the geometry, never on the order in which Stage 5 meets them (Invariant 5). Starting point: each expansion receives an equal share of the free space, and whatever one of them cannot use, because it reaches its comfortable form first, goes to the other.

The same measurements of loose and tight systems that settle [H1](#h1-the-duration-table) show how far ledger lines and accidentals give way in printed engravings.

## Consequences

**Positive:**

1. **One place for every rule.** Every horizontal rule is located by its form and the stage that produces it, and Stages 3 and 4 stay free of engraving rules.
2. **A safe exploration.** Values change; passes do not. The property tests guard the invariants while values are tuned.
3. **Exact geometry.** Contours give blocking widths without spurious prestretched gaps, and an accidental takes exactly the room its collision needs.
4. **Stage 1 stays heavy, parallel and cached.** None of its work depends on spacing parameters, so tuning them never repeats it.
5. **Determinism.** Tables of rationals and linear interpolation replace logarithms and floating point.
6. **Drags rarely reflow.** A dragged column moves alone, and the breaks hold, unless a gap's ideal width falls below its blocking width.
7. **Plugins are first-class in both phases.** They reserve room in Stage 1 and place within it in Stage 5.
8. **Spacing can be inspected.** Provenance and the overlays show why every gap has its widths, and change none of them.
9. **Adaptive without feedback.** Minimum forms let a tight system take compact forms and a loose one comfortable forms, with no pass back to Stage 3: the blocking widths reflect what the music needs at its tightest, and Stage 5 spends only room the springs have already given.

**Negative:**

1. **Contours cost more than boxes**, in computation and in memory in Stage 1. They are cached with the atoms.
2. **An anchor derived from the music couples a section.** If one is chosen, an edit can re-space the section, and re-optimisation after a local edit falls back to a full pass.
3. **Constraints across barlines couple every stack they span** in Stage 2's second step.
4. **Interpolation is approximate.** Linear interpolation approximates a logarithmic shape between entries, by about 0.10 staff spaces at the starting values. More entries reduce it.
5. **Nothing is validated yet.** The values are unvalidated until the exploration. This ADR fixes forms, not appearance.
6. **No-break material adds a value per stack.** Each stack carries its closing boundary's no-break width, and Stage 3 a prefix sum of them; whether cancellation naturals join the repeat barlines there waits on house style ([H10](#h10-no-break-material-and-the-system-start)).
7. **Drags follow the pipeline, not the pointer.** Every update is an edit, so a drag updates at the cadence of batched formatting and invalidation delivery, roughly six to ten times a second, which is sufficient ([H20](#h20-drags-as-edits)). The drag is one undo step, split only where another client's edit comes between its updates ([ADR-0015](0015-Undo-and-Redo.md#continuous-gestures-one-undo-step)).
8. **Minimum forms are drawn short of comfort where the system is tight.** A system the breaker has compressed shows shortened ledger lines and tucked accidentals rather than asking for more room. That is the engraver's own trade, but the cost function does not see it; the inspection view marks it ([Inspection](#inspection)).

**Changes to other decisions:**

- **ADR-0028:**
  - Its Stage 1 four-directional extents, measured from an atom's centre, are replaced by contours relative to the origin defined here.
  - Ideal and blocking widths are formed in Stage 2, from Stage 1's rhythm and contours.
  - The atom hook may declare constraints spanning several columns, and minimum forms.
  - A measure reads the clef the next measure begins with, which it draws before its closing barline ([The Bar Boundary](#the-bar-boundary)).
- **ADR-0037:**
  - Its `ADR-00XX` references point here.
  - Where it places ideal widths, blocking widths or the gutter in Stage 1, they become Stages 1–2: Stage 1 supplies the geometry, Stage 2 the widths. This affects Ideal Width Semantics, The Gutter, The Preamble, Fitting Rules (including its note on who pays) and How Complete Information Enables Optimality.
  - Semantic accidentals enter its widths through the atoms' contours, and so through blocking widths, not through ideal widths.
  - A measure's closing barline, in the form it takes at a system's end, and a clef change drawn before it, are part of the measure's width, so its postamble holds cautionary key and time signatures only.
  - What a repeat barline adds mid-system is no-break material: each stack carries its closing boundary's no-break width, and the available width of a system subtracts those of its interior boundaries ([The Bar Boundary](#the-bar-boundary)).
  - Its preamble shows the clef and the key in force, and omits a key or time signature change the system's first measure draws itself.
  - A barline drag either transfers ideal width between its measure and the next or sets its measure's width override ([User Inputs](#user-inputs)).
- **ADR-0015:** a continuous gesture, such as a drag, is one undo step however many transactions it takes ([ADR-0015](0015-Undo-and-Redo.md#continuous-gestures-one-undo-step)).
- **ADR-0013:** Stage 5 places elements within the space Stage 4 gives ([ADR-0013](0013-Slur-Formatting.md#the-pipeline-is-one-way-and-what-stage-5-may-change)). Every adjustment it makes inside an atom either stays within the envelope the atom's contours reserved or, for an element with a minimum form, expands only into space Stage 4 has given, so the declared ranges and the minimum and comfortable forms must be available to Stage 1 ([Atoms and Contours](#atoms-and-contours)). How the adjustments reconcile with ADR-0028's immutability of atoms is [open in ADR-0013](0013-Slur-Formatting.md#whether-stage-5-modification-touches-atom-internal-geometry).

## Alternatives Considered

1. **Bounding boxes.** Four extents per atom, measured from its centre. Rejected: they overstate blocking widths wherever a glyph's corner is empty, and create prestretched gaps from geometry no reader sees.
2. **Accidentals adding to the ideal width.** Rejected: the note looks late, and the rhythm the ideal width encodes is distorted. Under springs an accidental takes room only when its collision requires it.
3. **Clearance rules without geometry.** Fixed clearances by element type serve common cases, and fail in dense accidentals and in dotted textures of several voices. Rejected as the sole basis. Clearances are kept, applied between contours.
4. **An anchoring duration per system.** Rejected: systems are chosen from the widths it would produce.
5. **Horizontal adjustment in Stage 5.** Rejected:
   - A column belongs to every staff of its stack, and Stage 5 runs per musician without coordination.
   - Widening one gap means narrowing others or overflowing the system, a second fitting outside Stage 3's cost.
   - The collision guarantee holds only at the widths Stages 3 and 4 computed.

   Minimum forms are not this: they move no column and widen no gap, but let an element take room a gap already has.
6. **Absolute offsets for dragged columns.** An offset added after fitting collides or jumps when an edit elsewhere changes the system's ratio. Rejected in favour of transfers of ideal width.
7. **Logarithms in floating point.** Rejected: results would differ between platforms, against ADR-0037's determinism.
8. **Comfortable forms in the contour.** Reserving every element at its comfortable form would make blocking widths reflect a comfort the music does not need, so tight systems would be refused room they could have had, and more gaps would be prestretched. Rejected in favour of minimum forms.

## Prior Art

- **LilyPond** (source, as read in October 2026):
  - `Spacing_options::get_duration_space` in `lily/spacing-options.cc` gives a duration `d` the space `(2.0 + log₂(d / d_ref)) × 1.2` staff spaces when `d ≥ d_ref`, and grows linearly below it. Its comment cites Gourlay's "Spacing a Line of Music".
  - `Spacing_spanner::calc_common_shortest_duration` in `lily/spacing-spanner.cc`:
    - takes the shortest duration starting in each measure;
    - uses the most frequent of these;
    - never goes above `base-shortest-duration`, 3/16 by default (`scm/define-grobs.scm`).
  - `set_column_rods` in the same file sets the minimum distance from each column to every earlier column whose overhang can still reach it.
  - In `lily/spacing-basic.cc`, a musical column's spring takes its length from the shortest duration playing there, in proportion to the time to the next column, and stretches in proportion to the space between bare noteheads.
  - `Note_spacing::get_spacing` in `lily/note-spacing.cc`:
    - takes a gap's minimum from horizontal skylines;
    - adjusts its ideal by the actual width of the left notehead or rest;
    - measures to the barline when one follows (`space-to-barline`).
  - `Note_spacing::stem_dir_correction` in the same file adds space for an up-stem followed by a down-stem and removes it for the reverse, citing Wanske, p. 138:
    - the default `stem-spacing-correction` is 0.5, scaled by how far the stems overlap;
    - the default `same-direction-correction` is 0.25.
  - `merge_springs` in `lily/spring.cc` combines the staves' springs by averaging their ideal distances, taking the largest minimum, and keeping the ideal at least 0.3 above it.
  - `Separation_item::set_distance` in `lily/separation-item.cc` turns the distance between neighbouring skylines into a minimum distance between columns.
  - Grace notes:
    - `Moment` has a main part and a grace part (`lily/include/moment.hh`). Grace music starts at minus its own length in the grace part and runs forward to the main note's moment (`lily/grace-music.cc`, `lily/grace-iterator.cc`).
    - By default each grace time step has a musical column of its own in the main spacing. The springs between grace columns come from `GraceSpacing`, with values of 1.6 and 0.8 in place of 2.0 and 1.2, under a source comment that grace notes should not stretch much (`lily/spacing-basic.cc`). Grace notes are ignored when the shortest duration is found (`lily/spacing-spanner.cc`).
    - `strict-grace-spacing` makes grace columns float: the main notes are spaced first, and the grace notes are placed to the left of their main notes (`lily/spacing-loose-columns.cc`). The snippet that documents it, from version 2.25.81, records two open issues: that accidentals are then ignored (issue 6876), and that LilyPond does not check whether the grace notes have room (issue 2630, which reports collisions with what precedes them).
    - `\afterGrace` places grace notes after a fraction of the main note's duration, 3/4 by default (`afterGraceFraction` in `ly/music-functions-init.ly`).
- **Hegazy and Gourlay** (1987): springs that pass through boxes and attach to the left edges of noteheads, blocking widths, and prestretched springs ([ADR-0037](0037-Measure-Distribution-Optimization.md#the-spring-model)). Their example is a pair of flats that stops a spring from shrinking as the measure is compressed.
- **Gourlay, "Spacing a Line of Music"** (1987): cited by Hegazy and Gourlay for the logarithmic dependence of ideal spacing on duration, and by LilyPond's source. Not consulted directly.
- **Wanske, *Musiknotation*** (1988): cited by LilyPond's source for the stem-direction correction. Not consulted directly.
- **SMuFL**: glyph bounding boxes, the cut-out anchors and the engraving defaults, among them `legerLineExtension`. Bravura's metadata supplies the figures quoted here.

## References

- [ADR-0006: SMuFL](0006-SMuFL.md)
- [ADR-0013: Slur Formatting](0013-Slur-Formatting.md) (what Stage 5 may adjust)
- [ADR-0014: Timewalk](0014-Timewalk.md) (grace note positions within a group)
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) (a drag as one undo step)
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md)
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) (invalidation delivery)
- [ADR-0033: Time Signature Architecture](0033-Time-Signature-Architecture.md)
- [ADR-0034: Key Signature Architecture](0034-Key-Signature-Architecture.md)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) (which accidentals Stage 1 draws; grace notes at the start of a measure)
- [ADR-0037: Measure Distribution Optimization](0037-Measure-Distribution-Optimization.md) (including the undoing of layout changes made in response to an edit)
- [ADR-0038: Backend-Authoritative Rendering and Terminal Frontend Execution](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md) (drags as edits; inspection as paintlists)
- [ADR-0047: Font Management](0047-Font-Management.md)
- [ADR-0048: SMuFL Glyph Selection Architecture](0048-SMuFL-Glyph-Selection-Architecture.md)
- Gould, E. *Behind Bars: The Definitive Guide to Music Notation*, Faber Music (2011)
- Ross, T. *The Art of Music Engraving and Processing*, Hansen Books (1970)
- Hegazy, W.A. and Gourlay, J.S. "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, The Ohio State University (1987)
- Gourlay, J.S. "Spacing a Line of Music", Technical Report OSU-CISRC-10/87-TR35, The Ohio State University (1987)
- Wanske, H. *Musiknotation: Von der Syntax des Notenstichs zum EDV-gesteuerten Notensatz*, Schott (1988)
- SMuFL specification, W3C Music Notation Community Group: https://www.w3.org/2021/03/smufl14/
- LilyPond source: `lily/spacing-options.cc`, `lily/spacing-spanner.cc`, `lily/spacing-basic.cc`, `lily/note-spacing.cc`, `lily/spring.cc`, `lily/separation-item.cc`, `lily/spacing-loose-columns.cc`, `lily/include/moment.hh`, `lily/grace-music.cc`, `lily/grace-iterator.cc`, `ly/music-functions-init.ly`, `scm/define-grobs.scm`
- LilyPond Snippets, "Positioning grace notes with floating space" (2.25.81 and later), https://lilypond.org/doc/v2.27/Documentation/snippets/rhythms-_002d-positioning-grace-notes-with-floating-space; LilyPond issues 6876, https://gitlab.com/lilypond/lilypond/-/issues/6876, and 2630, https://gitlab.com/lilypond/lilypond/-/issues/2630
- MNX, W3C Music Notation Community Group, rhythmic position object: https://w3c-cg.github.io/mnx/docs/mnx-reference/objects/rhythmic-position/
