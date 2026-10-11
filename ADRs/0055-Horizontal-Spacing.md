# ADR-0055: Horizontal Spacing

## Status

Accepted

The architecture is decided: the forms in which horizontal engraving rules reach Stage 3, what Stages 1 and 2 compute and in which form, and the invariants that hold between them. Values and several policies are open. Each is an entry in [Open Items](#open-items), which is amended in place as [the exploration](#the-exploration) settles it. One open item reaches beyond values: how material at a barline that exists only mid-system is counted ([H10](#h10-no-break-material-and-the-system-start)) changes ADR-0037's interface when it is settled.

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
| Variant at a break | Material that differs where a system starts or ends | Before Stage 3 | [The Bar Boundary](#the-bar-boundary), [Variants at a Break](#variants-at-a-break) |

Elasticity is not a fifth form: it is a parameter of ADR-0037's model, which may differ by class of gap ([Elasticity by Gap Class](#elasticity-by-gap-class)). User interventions enter in one of the four forms ([User Inputs](#user-inputs)). A horizontal rule that fits none of them is a design problem to be resolved here, not a pass to be added after Stage 2.

### Stages 1 and 2

**Stage 1** runs per measure, in parallel, and is cached by content ([ADR-0029](0029-Global-Hash-Consing.md)). It does the heavy geometric work and no spacing arithmetic. It:

- resolves column assembly and forms atoms ([Atoms and Contours](#atoms-and-contours));
- computes each atom's origin and contours, and the attachment points and contours of material without duration;
- records each staff's rhythm: every note and rest, with its onset and duration as exact rationals, tuplets and dots included;
- records what optical corrections need: stem directions, beams, and the widths of noteheads and rests;
- declares, as geometry, the constraints spanning several columns that its own content implies: which atoms or glyphs, and of what kind (ties, lyrics, elements that reserve room, plugin reservations);
- supplies the geometry of the decorations a measure needs when it starts a system: courtesy accidentals for tied-to notes, tie continuations, a start repeat that may move there.

Stage 1's results depend on the measure's content, the active font and the settings that change glyph geometry. They never depend on the duration table, the anchoring duration, the clearance table, spacing distances such as the minimum tie length, `σ`, or anything in another measure. That is what keeps Stage 1 parallel, and its caches valid when any spacing parameter changes.

**Stage 2** runs per measure stack, in two steps. First, for each stack, it:

1. merges the staves' onsets into the raster of columns and forms the gaps ([The Raster and Gap Classes](#the-raster-and-gap-classes));
2. computes each gap's ideal width ([Ideal Widths](#ideal-widths));
3. computes each gap's blocking width, forms the constraints between atoms of one staff that are not neighbours, and turns the constraints Stage 1 declared into distances ([Blocking Widths](#blocking-widths), [Constraints Spanning Several Columns](#constraints-spanning-several-columns));
4. applies measure-width overrides ([ADR-0037](0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms));
5. reduces the constraints that lie within the stack, by ADR-0037's procedure.

Then, across neighbouring stacks, it:

6. reduces the constraints that cross a barline, over the gaps of every stack they span;
7. computes each stack's gutter from Stage 1's geometry ([Variants at a Break](#variants-at-a-break));
8. applies the user's column transfers ([User Inputs](#user-inputs));
9. produces ADR-0037's stack summaries, and keeps every gap's `w` and `b` for Stage 4.

Beyond its own stack, Stage 2 reads only finished output of earlier work: the anchoring duration, where it is derived from the music ([H2](#h2-the-anchoring-duration)); the previous stack's closing barline, where the leading gap begins ([The Bar Boundary](#the-bar-boundary)); and, in its second step, the first step's gaps of the stacks a constraint spans. All three are forward dependencies; nothing flows back.

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

Contours replace ADR-0028's four-directional extents. A box overstates the blocking width wherever a glyph's corner is empty: the notch of an accidental, the open side of a flag, a displaced notehead. Each overstatement makes a gap prestretched that need not be, and the spacing turns irregular for geometry no reader sees. LilyPond computes the minimum distance between neighbouring columns from horizontal skylines, profiles of the same kind ([Prior Art](#prior-art)).

**Ledger lines** are part of the atom and its contours. SMuFL's `legerLineExtension` sets how far a ledger line projects beyond its notehead: in Bravura 0.4 staff spaces on each side, so a ledger line under a black notehead 1.18 staff spaces wide is 1.98 wide. At close spacing the ledger lines of neighbouring notes would touch, and engravers shorten them. The blocking width is therefore computed with the shortest acceptable ledger line, and how long each is drawn is a Stage 5 adjustment to the space Stage 4 gives ([H6](#h6-ledger-lines)).

**Immutability.** Atoms are cached with their contours. Stage 4 positions them; Stage 5 adjusts elements to the space Stage 4 gives and never moves an atom or a column ([ADR-0013](0013-Slur-Formatting.md)). Whether Stage 5 may change an atom's internal geometry, such as a stem length, is ADR-0013's open question. Such a change does not affect spacing as long as it stays within the atom's contours.

### The Raster and Gap Classes

**Columns.** Stage 2 merges every staff's onsets into one ordered sequence of rational positions; each distinct onset is a column, tuplets exact. Material without duration that occupies horizontal space also forms columns: barlines; clef, key and time signature changes; breath marks and caesuras. Such a column may share an onset with a note column, as a clef change does with the note it precedes; it then comes first. Whether grace notes form columns of their own is [H7](#h7-grace-notes).

**Gaps** run from one column's right attachment point (for an atom, its origin) to the next column's origin or left attachment point. Each gap has a **class**, given by the columns at its ends, which decides how its widths are derived:

- **A gap leaving a column with duration** carries that column's duration space, up to the next column with a later onset ([Ideal Widths](#ideal-widths)). Whether a barline is such a column, so that a bar's last note has its space up to the barline, is [H9](#h9-the-last-note-before-a-barline). The gap's blocking width is the contour distance with the minimum clearance between the glyph classes that meet.
- **A gap leaving material without duration** runs from its right attachment point to the next origin. Its ideal width is the contour distance with the ideal clearance between the glyph classes that meet, and its blocking width the contour distance with the minimum clearance, both from the clearance table ([H4](#h4-clearances)). Gould's space after a barline, as recorded, is of this kind: 1.0 ideally and 0.5 at least, between the barline and the next ink. Material without duration therefore adds its own width and the clearance after it: a clef change before a note lengthens the bar by both.
- **Every gap between columns at different onsets** has a blocking width of at least a minimum separation ([H4](#h4-clearances)), so that two onsets never print at the same place, even where no staff has atoms at both.
- **The class also selects the gap's elasticity** ([Elasticity by Gap Class](#elasticity-by-gap-class)).

**Staves are compared only with themselves.** A contour meets only contours of the same staff, or of the same row of text, such as a lyric line or a line of dynamics. Material that crosses between staves contributes to the contours of every staff it reaches; Stage 1 records its geometry with its measure, and Stage 2 assigns it to the staves.

**Bar rests** are not on the raster. A bar rest constrains its measure's whole span, a constraint across all of the stack's gaps for that staff, and is placed in Stage 5, centred in the space its measure has been given. The width of a multi-bar rest is [H13](#h13-bar-rests-and-multi-bar-rests).

### The Bar Boundary

Material at a barline has up to three forms, as a discretionary break in TeX has: what is printed mid-system, what ends a system there, and what starts the next. ADR-0037 maps the end of a system to the postamble, the start to the preamble and gutter, and the mid-system form to the barline's width within its measure ([ADR-0037](0037-Measure-Distribution-Optimization.md#prior-art)). This ADR assigns the material at a boundary to those forms:

- **A stack owns its closing barline** in mid-system form: the gap from its last column to the barline, and the barline's own width as fixed space. As ADR-0037 states, the postamble reserves only what the barline's system-end form adds.
- **A stack owns its leading gap**, from the previous stack's closing barline (its right attachment point) to the stack's first column. Stage 2 reads that barline's type and contour from the previous stack's Stage 1 output.
- **Changes at a boundary are no-break material.** A clef change, drawn before the barline, and key and time signature changes, drawn after it, together with the space that separates them from the barline, exist in their mid-system form only when the boundary is not a break. At a break they are replaced: by the cautionaries at the end of the system, in the postamble, and by the clef, key and time signature at the start of the next, in the preamble. They belong to the boundary, not to either stack.
- **At a system start there is no barline before the stack.** The preamble and gutter stand in its place, and the leading gap begins where they end.

How no-break material is counted only at boundaries that are not breaks, and how the space at a system start is divided between the preamble, the gutter and the leading gap, is [H10](#h10-no-break-material-and-the-system-start). Until it is settled, ADR-0037 has no place for no-break material, and the bar boundary cannot be implemented.

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

**The controlling duration.** For a gap leaving a column `c` with duration, the controlling duration `d_ctrl` is the shortest duration among the notes and rests, in any staff of the stack, that sound at `c`: those beginning at or before `c` and ending after it. With `c′` the next column with a later onset:

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

**Between two atoms of one staff at neighbouring columns**, the blocking width is the least distance between their origins at which no ink of the left atom, with the minimum clearance for the pair of glyph classes that meet, reaches ink of the right atom at the same height. Bands of height count as meeting when they come within a vertical clearance of each other, so that ink at slightly different heights is still kept apart.

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
   lies left of its origin. clearance gives the minimum clearance for a pair
   of glyph classes (H4); vpad is the vertical clearance within which bands
   at different heights still meet. Exact."
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
  - a condition that must not run into a neighbour becomes a constraint spanning several columns;
  - a loose condition reserves nothing.
- **Placement, Stage 5.** The solver places the element within the space Stage 4 has given, satisfying the condition, and never moves an atom or a column ([ADR-0013](0013-Slur-Formatting.md)). An element that is part of an atom's contour may move only within the extent its contour reserved. An element that reserved nothing may go wherever the space allows, checked against everything already placed.

Stage 5 can only use room that exists. Anything an element must have, it declares in the spacing phase.

ADR-0028's Stage 1 [atom hook](0028-Hierarchical-Rendering-Pipeline.md) produces atoms, with their contours, and may also declare constraints spanning several columns. A plugin participates in both phases through its hooks; it cannot add a pass after Stage 2.

### Variants at a Break

ADR-0037 specifies their structure: the preamble and postamble computed on demand and cached by combination, and the gutter known before breaking. Their widths are computed from the same contours and clearances as everything else:

- **Preamble**: the clef and key signature, with the clearances between them, as the largest across the staves. Whether it includes a time signature that takes effect at the system's first stack, and whether it carries space after its ink, is [H10](#h10-no-break-material-and-the-system-start).
- **Postamble**: the cautionaries, the cautionary clef before the closing barline and the key and time signatures after it, and what the barline's system-end form adds.
- **Gutter**: computed by Stage 2 from Stage 1's geometry: courtesy accidentals for tied-to notes, tie continuation arcs, and a start repeat that moves to the system's start, each for the room it needs beyond what the stack already holds, as ADR-0037 specifies.

### Elasticity by Gap Class

ADR-0037's baseline elasticity applies to every gap: stretch `σ × w`, shrink `w − b`. A class of gap may need elasticity of its own. A barline gap that yields "only in cramped conditions" is the first candidate, and it is a matter of shrinking: a shrink weight per class is ADR-0037's alternative 3, generalised from duration to class. A stretchability per class is possible too, but it changes ADR-0037's interface: each prestretched pair carries its own `σ`, and the ordinary sum becomes `Σ σ_j × w_j`. Until [H11](#h11-elasticity-by-gap-class) is settled, the baseline applies to every class.

### User Inputs

**Column drags.** The user sees the columns on the page and can drag them. A drag is stored as a transfer of ideal width between the column's two neighbouring gaps: the gap on the side the column moves toward loses `Δ`, and the gap on the other side gains it.

- **The transfer.** Under the baseline elasticity, with both gaps ordinary, moving a column by `δ` at ratio `r` takes `Δ = δ / (1 + σr)` when the system is stretched and `Δ = δ / (1 + r)` when it is compressed. In general, as when a neighbouring gap is prestretched, `Δ` is the least transfer that moves the column by `δ` under ADR-0037's springs, found exactly.
- **The clamp.** A drag is clamped at the current ratio, so that neither gap becomes narrower than its blocking width. At `r = −1` every gap is at its blocking width and no column can move.
- **What changes.** While both gaps' ideal widths stay at or above their blocking widths, the stack's summary is unchanged: a transfer leaves `Σ w`, `Σ` shrink and `Σ` stretch where they were. Every candidate system's ratio, and so the breaks, stay as they were, and only the dragged column moves.
- **When the summary changes.** Under stretching, the clamp can let the losing gap's ideal width fall below its blocking width, while its width at the current ratio stays above it. This system's ratio is still unchanged, but the gap is now prestretched and the stack's summary changes, so the breaks may change. It is a real request for more room.
- **Barlines.** Dragging a barline moves width between two stacks. It is a measure-width change, and may change the breaks.

Each transfer is attached to its column, identified by measure, rhythmic position and kind, since a clef change can share an onset with a note. Stage 2 applies it after the reductions, and clamps it there so that no ideal width falls below zero, ADR-0037's precondition: the gaps it was made against may have changed since. A reduction raises blocking widths only, so every constraint holds whatever the ideal widths, and a transfer can neither break one nor change what a reduction reserved. A transfer persists until the user removes it, or the engine does: when its column no longer exists, or when its measure is reformatted ([H14](#h14-removal-of-column-transfers)).

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

**Measure-width overrides**, such as "stretch this measure", are specified by ADR-0037's [Width Overrides](0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms). **The anchoring duration** may be pinned, and a new section begun with a value of its own ([H2](#h2-the-anchoring-duration)). **Element offsets**, such as dragging a dynamic or a text, are Stage 5 placements relative to the element's anchor. They are not spacing inputs.

### Invariants and Property Tests

The boundary between Stages 1–2 and Stage 3 is specified by properties, tested generatively over random measures and random stacks. They are what lets the exploration change values without changing the architecture.

1. **No collision at any feasible ratio.** For any rational `r ≥ −1`, placing the columns by ADR-0037's `g_j(r)` leaves no two contours of one staff or row of text overlapping with their minimum clearances, whether or not their atoms are neighbours, and no two columns at different onsets closer than the minimum separation.
2. **Attachments never change duration space.** With durations and stem directions fixed, adding or removing accidentals, articulations or ledger lines, or displacing noteheads, changes no controlling duration and no ideal width of a gap leaving a column with duration.
3. **The duration table is monotone and sub-linear.** Its width never decreases as the duration grows, and its width divided by duration never increases.
4. **Stage 1 is independent of spacing parameters.** Changing the duration table, the anchoring duration, the clearance table, a spacing distance or `σ` changes no Stage 1 output.
5. **Determinism.** Identical content gives identical contours, summaries and gap lists, whatever the order in which staves, measures and stacks are processed.
6. **Scale invariance.** Widths are in staff spaces at the system's staff size; changing that size changes none of them. A staff drawn at reduced size contributes contours scaled by its ratio, and text sized in absolute units is converted at the system's size.
7. **Drag neutrality.** A transfer that keeps both gaps' ideal widths at or above their blocking widths leaves the stack's `:min`, `:natural`, `:ideal`, `:ordinary-ideal` and `:prestretched` unchanged.
8. **Sound reduction.** After Stage 2, every constraint spanning several columns holds at every `r ≥ −1`.

## The Exploration

The architecture is fixed so that the exploration can only change values and inputs, never add a pass. It is how the open items close.

- **Parameters are data.** The duration table, the anchoring rule, the clearance table, the optical corrections and the elasticities live in a house-style structure. Each value records its source, its date and its status: provisional, verified against the literature, or measured.
- **The reference is printed engraving.** Editions in the plate-engraving tradition are measured: the staff space from the staff lines; the distance from one notehead's left edge to the next, and the white space between them; each for every duration, in loose and in tight systems of the same edition. Each measurement records the edition, page and system. The output of other notation programs is not used.
- **The procedure** is measurement, then a value in the house-style data, then the register entry updated with its date and evidence. The property tests hold throughout.

What it decides is listed in [Open Items](#open-items). Several items share evidence. The same measurements of loose and tight systems settle the duration table ([H1](#h1-the-duration-table)), the behaviour of very short values ([H8](#h8-very-short-values-under-stretching)) and what the eye keeps under compression ([H16](#h16-distance-or-white-space)).

## Open Items

| ID | Item | Form | Stage | Decided by |
|---|---|---|---|---|
| [H1](#h1-the-duration-table) | The duration table | Ideal width | 2 | Measurement |
| [H2](#h2-the-anchoring-duration) | The anchoring duration | Ideal width | Before 2 | Reasoning, then measurement |
| [H3](#h3-optical-corrections) | Optical corrections | Ideal width | 2 | Measurement |
| [H4](#h4-clearances) | Clearances | Blocking width, ideal width | 2 | Literature and measurement |
| [H5](#h5-column-assembly) | Column assembly | Contour | 1 | Literature |
| [H6](#h6-ledger-lines) | Ledger lines | Contour | 1, 5 | Literature and measurement |
| [H7](#h7-grace-notes) | Grace notes | Ideal width, contour | 1–2 | Measurement |
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

The rules for seconds in chords, offsets between voices (including which voice is displaced), shared unisons, stacking accidentals, aligning dots and placing arpeggio signs. They may warrant a decision of their own. This ADR fixes that they run in Stage 1 and produce the atom's geometry.

### H6: Ledger lines

The shortest acceptable ledger line, from which blocking widths are computed, and how Stage 5 lengthens ledger lines within the space Stage 4 gives, up to their full extension.

### H7: Grace notes

Whether grace notes are part of the main note's atom or form columns of their own, and their spacing. LilyPond gives them columns of their own, with values of their own: 1.6 and 0.8 in place of 2.0 and 1.2.

### H8: Very short values under stretching

Values whose ideal width lies below their blocking width are prestretched. They stay at their blocking width as the system stretches, until their kink, while everything else gains space. Whether a run of thirty-seconds should stay packed in this way, or the table should keep very short values above their blocking widths, is a matter of the table ([H1](#h1-the-duration-table)), settled by measurement.

### H9: The last note before a barline

There are two options:

- **Up to the barline.** The bar's last note has its space up to the barline, with the barline's clearance after it. LilyPond measures to the barline, and this ADR's gap classes state it as the starting point.
- **Up to the next bar's first note**, with the barline placed within that space. This would carry part of the last note's space into the next stack's leading gap, across the stack boundary, which [The Bar Boundary](#the-bar-boundary) would then have to allow.

The recorded observation that a whole note alone in its bar is spaced like a half note bears on the choice.

### H10: No-break material and the system start

How no-break material at a boundary is counted only when the boundary is not a break, and how the space at a system start is divided between the preamble, the gutter and the leading gap. ADR-0037 adds space at a system start but cannot remove any. Its preamble includes the space after the key signature, which the leading gap also provides, and it omits time signatures.

Starting point:

- **Fixed space.** No-break material is fixed space whose width depends only on its boundary. The DP subtracts its sum over a segment's interior boundaries from the available width, a prefix sum, so the DP stays exact and each evaluation O(1).
- **The preamble.** It ends at its ink, and the leading gap supplies the space after it. The gutter reserves any difference between the leading gap's system-start and mid-system forms.
- **The time signature.** The preamble includes a time signature that takes effect at the system's first stack, and the time signature at the start of the piece.

The alternative is a second, system-start summary for every stack. It keeps the gaps around a boundary elastic, at the cost of a larger change to ADR-0037's interface. Either way ADR-0037 changes.

### H11: Elasticity by gap class

Whether any class of gap, the barline gap first, needs elasticity of its own:

- **On the shrinking side**, a weight per class depends on ADR-0037's alternative 3.
- **On the stretching side**, a stretchability per class changes ADR-0037's interface.
- **Drags.** A column dragged between gaps of different classes no longer leaves the stack's stretch unchanged.

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

What counts as reformatting a measure, so that the engine removes its transfers. This is specified with the editing commands.

### H15: Verification against Gould and Ross

The values recorded here as Gould's and Ross's have not been checked against the books: the duration table in [H1](#h1-the-duration-table) and the barline clearance in [H4](#h4-clearances). Each needs a page reference before it is called verified.

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

## Consequences

**Positive:**

1. **One place for every rule.** Every horizontal rule is located by its form and the stage that produces it, and Stages 3 and 4 stay free of engraving rules.
2. **A safe exploration.** Values change; passes do not. The property tests guard the invariants while values are tuned.
3. **Exact geometry.** Contours give blocking widths without spurious prestretched gaps, and an accidental takes exactly the room its collision needs.
4. **Stage 1 stays heavy, parallel and cached.** None of its work depends on spacing parameters, so tuning them never repeats it.
5. **Determinism.** Tables of rationals and linear interpolation replace logarithms and floating point.
6. **Drags rarely reflow.** A dragged column moves alone, and the breaks hold, unless a gap's ideal width falls below its blocking width.
7. **Plugins are first-class in both phases.** They reserve room in Stage 1 and place within it in Stage 5.

**Negative:**

1. **Contours cost more than boxes**, in computation and in memory in Stage 1. They are cached with the atoms.
2. **An anchor derived from the music couples a section.** If one is chosen, an edit can re-space the section, and re-optimisation after a local edit falls back to a full pass.
3. **Constraints across barlines couple every stack they span** in Stage 2's second step.
4. **Interpolation is approximate.** Linear interpolation approximates a logarithmic shape between entries, by about 0.10 staff spaces at the starting values. More entries reduce it.
5. **Nothing is validated yet.** The values are unvalidated until the exploration. This ADR fixes forms, not appearance.
6. **The bar boundary waits on H10**, which changes ADR-0037's interface ([H10](#h10-no-break-material-and-the-system-start)).

**Changes to other decisions:**

- **ADR-0028:**
  - Its Stage 1 four-directional extents, measured from an atom's centre, are replaced by contours relative to the origin defined here.
  - Ideal and blocking widths are formed in Stage 2, from Stage 1's rhythm and contours.
  - The atom hook may declare constraints spanning several columns.
  - Its remaining references to a stack's largest column ratio, to scale factors and to `actual = ideal × scale_factor` predate ADR-0037's springs, and give way to them.
- **ADR-0037:**
  - Its `ADR-00XX` references point here.
  - Where it places ideal widths, blocking widths or the gutter in Stage 1, they become Stages 1–2: Stage 1 supplies the geometry, Stage 2 the widths. This affects Ideal Width Semantics, The Gutter, The Preamble, Fitting Rules (including its note on who pays) and How Complete Information Enables Optimality.
  - Semantic accidentals enter its widths through the atoms' contours, and so through blocking widths, not through ideal widths.
  - [H10](#h10-no-break-material-and-the-system-start) changes its available width and its preamble when it is settled.
- **ADR-0013:** Stage 5 places elements within the space Stage 4 gives, as ADR-0013 now states. Whether it may change an atom's internal geometry remains ADR-0013's open question.

## Alternatives Considered

1. **Bounding boxes.** Four extents per atom, as in ADR-0028 before this ADR. Rejected: they overstate blocking widths wherever a glyph's corner is empty, and create prestretched gaps from geometry no reader sees.
2. **Accidentals adding to the ideal width.** Rejected: the note looks late, and the rhythm the ideal width encodes is distorted. Under springs an accidental takes room only when its collision requires it.
3. **Clearance rules without geometry.** Fixed clearances by element type serve common cases, and fail in dense accidentals and in dotted textures of several voices. Rejected as the sole basis. Clearances are kept, applied between contours.
4. **An anchoring duration per system.** Rejected: systems are chosen from the widths it would produce.
5. **Horizontal adjustment in Stage 5.** Rejected:
   - A column belongs to every staff of its stack, and Stage 5 runs per musician without coordination.
   - Widening one gap means narrowing others or overflowing the system, a second fitting outside Stage 3's cost.
   - The collision guarantee holds only at the widths Stages 3 and 4 computed.
6. **Absolute offsets for dragged columns.** An offset added after fitting collides or jumps when an edit elsewhere changes the system's ratio. Rejected in favour of transfers of ideal width.
7. **Logarithms in floating point.** Rejected: results would differ between platforms, against ADR-0037's determinism.

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
  - Grace notes are spaced by `GraceSpacing`, with values of 1.6 and 0.8 in place of 2.0 and 1.2.
- **Hegazy and Gourlay** (1987): springs that pass through boxes and attach to the left edges of noteheads, blocking widths, and prestretched springs ([ADR-0037](0037-Measure-Distribution-Optimization.md#the-spring-model)). Their example is a pair of flats that stops a spring from shrinking as the measure is compressed.
- **Gourlay, "Spacing a Line of Music"** (1987): cited by Hegazy and Gourlay for the logarithmic dependence of ideal spacing on duration, and by LilyPond's source. Not consulted directly.
- **Wanske, *Musiknotation*** (1988): cited by LilyPond's source for the stem-direction correction. Not consulted directly.
- **SMuFL**: glyph bounding boxes, the cut-out anchors and the engraving defaults, among them `legerLineExtension`. Bravura's metadata supplies the figures quoted here.

## References

- [ADR-0006: SMuFL](0006-SMuFL.md)
- [ADR-0013: Slur Formatting](0013-Slur-Formatting.md) (what Stage 5 may adjust)
- [ADR-0014: Timewalk](0014-Timewalk.md)
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md)
- [ADR-0029: Global Hash-Consing](0029-Global-Hash-Consing.md)
- [ADR-0033: Time Signature Architecture](0033-Time-Signature-Architecture.md)
- [ADR-0034: Key Signature Architecture](0034-Key-Signature-Architecture.md)
- [ADR-0035: Remembered Alterations](0035-Remembered-Alterations.md) (which accidentals Stage 1 draws)
- [ADR-0037: Measure Distribution Optimization](0037-Measure-Distribution-Optimization.md)
- [ADR-0047: Font Management](0047-Font-Management.md)
- [ADR-0048: SMuFL Glyph Selection Architecture](0048-SMuFL-Glyph-Selection-Architecture.md)
- Gould, E. *Behind Bars: The Definitive Guide to Music Notation*, Faber Music (2011)
- Ross, T. *The Art of Music Engraving and Processing*, Hansen Books (1970)
- Hegazy, W.A. and Gourlay, J.S. "Optimal line breaking in music", Technical Report OSU-CISRC-8/87-TR33, The Ohio State University (1987)
- Gourlay, J.S. "Spacing a Line of Music", Technical Report OSU-CISRC-10/87-TR35, The Ohio State University (1987)
- Wanske, H. *Musiknotation: Von der Syntax des Notenstichs zum EDV-gesteuerten Notensatz*, Schott (1988)
- SMuFL specification, W3C Music Notation Community Group: https://www.w3.org/2021/03/smufl14/
- LilyPond source: `lily/spacing-options.cc`, `lily/spacing-spanner.cc`, `lily/spacing-basic.cc`, `lily/note-spacing.cc`, `lily/spring.cc`, `lily/separation-item.cc`, `scm/define-grobs.scm`