# ADR-0048: SMuFL Glyph Selection Architecture

## Status

Accepted

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Spec-Level Metadata](#spec-level-metadata)
  - [The Glyph Selection Mechanism](#the-glyph-selection-mechanism)
    - [Logical Keywords](#logical-keywords)
    - [SMuFL Classes](#smufl-classes)
    - [The Glyph Selection Cascade](#the-glyph-selection-cascade)
    - [Font Availability Filtering](#font-availability-filtering)
  - [Two Selection Patterns](#two-selection-patterns)
  - [The Additional-Glyphs Mechanism](#the-additional-glyphs-mechanism)
  - [SMuFL Glyph Keyword Convention](#smufl-glyph-keyword-convention)
  - [SMuFL Class Inventory](#smufl-class-inventory)
    - [Clef Classes](#clef-classes)
    - [Notehead Set Classes](#notehead-set-classes)
    - [Other Notation Element Classes](#other-notation-element-classes)
    - [Accidental Classes](#accidental-classes)
    - [Time Signature and Tuplet Classes](#time-signature-and-tuplet-classes)
    - [Medieval and Renaissance Ranges](#medieval-and-renaissance-ranges)
  - [Ooloi-Defined Classes](#ooloi-defined-classes)
  - [Font Coverage](#font-coverage)
- [Rationale](#rationale)
- [Consequences](#consequences)
- [References](#references)

---

## Context

[ADR-0006](0006-SMuFL.md) adopts SMuFL as Ooloi's sole font standard for musical symbol rendering. [ADR-0047](0047-Font-Management.md) addresses how fonts are loaded, registered, and kept consistent across rendering paths. This ADR addresses the next question: how does Ooloi exploit SMuFL's glyph vocabulary and classification system to give users control over the visual appearance of notation elements?

Ooloi's domain model uses keywords to identify notation elements: `:treble` for a clef, `:quarter` for a notehead, `:fermata` for an articulation. The visual rendering of each element --- which specific SMuFL glyph to draw --- is currently implicit (hardcoded or assumed). This prevents users from choosing alternative visual styles and prevents house styles from controlling glyph appearance.

SMuFL provides the solution through its classification system. The specification's `classes.json` groups glyphs by function: `clefsG` contains 13 G clef visual variants, `noteheadSetDefault` contains 25 standard noteheads, `rests` contains 22 rest glyphs. These classes are the foundation of a general glyph selection mechanism that applies uniformly to all notation elements with visual variants.

This is a deliberate architectural choice: Ooloi runs *with* SMuFL, not alongside it. Where other applications treat SMuFL as a font encoding detail --- a way to map glyph names to codepoints --- Ooloi lets SMuFL's classification system shape its own architecture. The class registry, the glyph selection cascade, the additional-glyphs mechanism, the font availability filtering --- all are built directly on SMuFL's vocabulary and organisation. When SMuFL defines a class, Ooloi uses that class as the unit of glyph selection. When SMuFL names a glyph, Ooloi uses that name verbatim as a keyword. This alignment means Ooloi benefits automatically from SMuFL's evolution: new classes, new glyphs, and new ranges become available without architectural changes.

---

## Decision

### Spec-Level Metadata

Independent of any specific font, the SMuFL specification defines three JSON files that describe the universal glyph vocabulary and its organisation. These are bundled at `shared/src/main/clojure/ooloi/shared/smufl/metadata/` and loaded once at registry initialisation:

| File | Contents | Role in Glyph Selection |
|---|---|---|
| `glyphnames.json` | Canonical name, Unicode codepoint, and description for every glyph in SMuFL | Authoritative vocabulary. Maps glyph keywords to codepoints for rendering. |
| `classes.json` | Named groupings of functionally related glyphs | Defines which glyphs are valid visual alternatives for each notation element. Drives the selection mechanism. |
| `ranges.json` | Organisational ranges as laid out in the specification | Broader groupings. Source for `:additional-glyphs` cherry-picking (e.g., medieval clefs from the `medievalAndRenaissanceClefs` range). |

Per-font metadata (companion `metadata.json` files) tells the registry what a specific font *can render* and how (metrics, bounding boxes). Spec-level metadata tells the registry what the glyphs *are* and how they relate to each other (classification, vocabulary). The intersection determines what the application can offer the user at any given moment.

### The Glyph Selection Mechanism

Every renderable notation element follows a three-layer architecture:

#### Logical Keywords

The domain-level identifier: `:treble`, `:quarter-rest`, `:fermata`. This is what the piece model stores. It carries musical meaning and is independent of visual rendering.

#### SMuFL Classes

Each logical keyword is associated with a SMuFL class that determines which glyphs are valid visual alternatives. Example: a clef with class `:clefsG` can be rendered using any of the 13 glyphs in the `clefsG` class.

**Unified class registry.** The SMuFL specification splits named glyph groups across two files: `classes.json` (functional groupings like `clefsG`, `rests`, `noteheadSetDefault`) and `ranges.json` (organisational groupings like `timeSignatures`, `tuplets`, `medievalAndRenaissanceClefs`). Ooloi merges both into a single unified class registry at load time. The element definition simply says `:class :timeSignatures` and the registry resolves it regardless of which source file it came from. The class/range distinction is a SMuFL specification organisational detail, not something Ooloi's domain model needs to expose. Ooloi-defined classes (`:clefsPercussion`, `:clefsTab`) are added to the same unified registry.

#### The Glyph Selection Cascade

The cascade determines which specific glyph from the class is actually rendered:

1. **House style default** --- the default glyph for each class (e.g., all G clefs use `:gClef`). Defined in the house style, which ships with sensible defaults.
2. **Piece override** --- this piece uses `:gClef8vbOld` for all G clefs. Stored in piece settings ([ADR-0016](0016-Settings.md)).
3. **Local override** --- this specific staff uses `:mensuralGclef`. Stored on the individual notation element.

Each level overrides the one above. Most users never touch this --- the house style default handles everything. Power users and historical-notation specialists control it precisely.

#### Font Availability Filtering

The choices presented to the user are the intersection of the class membership and the active font's glyph inventory. Glyphs defined in the spec but absent from the active font do not appear as choices. No error, no placeholder --- the font's capabilities are respected silently.

Available glyphs = (members of SMuFL class `union` `:additional-glyphs`) `intersect` active font's glyph inventory.

### Two Selection Patterns

**Pattern 1: Single glyph selection.** The user picks ONE visual variant from a class for a logical element. This is the clef pattern. Example: a treble clef (`:treble`) can be rendered using any glyph in `clefsG` --- `:gClef`, `:gClefReversed`, `:gClef8vbOld`, etc. Articulations, ornaments, fermatas, and individual noteheads follow this pattern.

**Pattern 2: Set selection.** The user switches an entire set at once, replacing all members simultaneously. Example: switching all noteheads from `noteheadSetDefault` to `noteheadSetDiamond` replaces every notehead glyph in the score at once. Time signature digits follow this pattern.

**Patterns are not mutually exclusive.** Noteheads support both: a set selection changes the default for all noteheads in the piece, and individual noteheads can then be overridden via the cascade. A user might switch to diamond noteheads globally (set selection) and then mark specific notes with X noteheads (single glyph override). The cascade handles this naturally --- the set selection operates at the house style or piece level, and the local override operates at the element level.

Both patterns use the same cascade mechanism. The difference is scope: Pattern 1 selects one glyph for one element; Pattern 2 selects a default set for an entire category.

### The Additional-Glyphs Mechanism

SMuFL's class system does not always provide the exact groupings Ooloi needs. The `medievalAndRenaissanceClefs` range contains G, F, and C clef glyphs mixed together --- pulling in the entire range for a C clef definition would also include G and F clef glyphs, which are not valid alternatives.

The solution is `:additional-glyphs`: a set of individual SMuFL glyph keywords that augments the primary class. Each notation element definition may include an `:additional-glyphs` set that cherry-picks specific glyphs from ranges or other classes without pulling in irrelevant members.

The available glyphs for a notation element = (members of primary class from `classes.json`) `union` `:additional-glyphs`, filtered by font availability.

This mechanism is deliberately simple: a flat set of extra glyph keywords, nothing more. There is no class inheritance, no mixed-class inclusion, no recursive expansion.

### SMuFL Glyph Keyword Convention

SMuFL glyph names use lowerCamelCase: `gClef8vb`, `fClef19thCentury`, `mensuralCclefPetrucciPosHigh`. Clojure keywords support this directly --- `:gClef8vb` is a valid keyword. Ooloi uses SMuFL glyph names verbatim as keywords, with no case conversion. This gives a direct, unambiguous mapping to the SMuFL specification.

### SMuFL Class Inventory

The following tables document the SMuFL classes and ranges relevant to Ooloi's glyph selection mechanism, extracted from `classes.json` and `ranges.json` of the SMuFL 1.4 specification.

#### Clef Classes

| Class | Glyphs | Contents |
|---|---|---|
| `clefsG` | 13 | `gClef`, `gClef15mb`, `gClef8vb`, `gClef8va`, `gClef15ma`, `gClef8vbOld`, `gClef8vbCClef`, `gClef8vbParens`, `gClefLigatedNumberBelow`, `gClefLigatedNumberAbove`, `gClefArrowUp`, `gClefArrowDown`, `gClefReversed` |
| `clefsF` | 8 | `fClef`, `fClef15mb`, `fClef8vb`, `fClef8va`, `fClef15ma`, `fClefArrowUp`, `fClefArrowDown`, `fClefReversed` |
| `clefsC` | 7 | `cClef`, `cClef8vb`, `cClefArrowUp`, `cClefArrowDown`, `cClefSquare`, `cClefCombining`, `cClefReversed` |
| `clefs` (general) | 49 | All of the above plus: `fClefTurned`, `gClefTurned`, `unpitchedPercussionClef1/2`, `semipitchedPercussionClef1/2`, `6stringTabClef`, `4stringTabClef`, `schaefferClef/PreviousClef/GClefToFClef/FClefToGClef`, `bridgeClef`, `accdnDiatonicClef`, `gClefChange`, `cClefChange`, `fClefChange`, `clef8`, `clef15`, `clefChangeCombining`, `indianDrumClef` |

Medieval/Renaissance clef glyphs (from `medievalAndRenaissanceClefs` range, accessed via `:additional-glyphs`): `mensuralGclef`, `mensuralGclefPetrucci`, `chantFclef`, `mensuralFclef`, `mensuralFclefPetrucci`, `mensuralCclef`, `chantCclef`, `mensuralCclefPetrucciPosLowest/Low/Middle/High/Highest`, `kievanCClef` (from `kievanSquareNotation` range).

See [ADR-0049](0049-Clef-Registry.md) for how clef classes drive the clef registry.

#### Notehead Set Classes

Each set provides a complete family of noteheads (whole, half, quarter, etc.) in a particular visual style. These support both Pattern 2 (switch the entire set at once) and Pattern 1 (override individual noteheads via the cascade).

| Class | Glyphs | Style |
|---|---|---|
| `noteheadSetDefault` | 25 | Standard noteheads |
| `noteheadSetDiamond` | 14 | Diamond |
| `noteheadSetDiamondOld` | 5 | Old-style diamond |
| `noteheadSetCircleX` | 5 | Circle-X |
| `noteheadSetCircled` | 9 | Circled |
| `noteheadSetHeavyX` | 2 | Heavy X |
| `noteheadSetLargeArrowDown` | 4 | Large arrow down |
| `noteheadSetLargeArrowUp` | 4 | Large arrow up |
| `noteheadSetNamesPitch` | 69 | Pitch names inside noteheads |
| `noteheadSetNamesSolfege` | 24 | Solfege names inside noteheads |
| `noteheadSetPlus` | 5 | Plus-shaped |
| `noteheadSetRoundLarge` | 5 | Round (large) |
| `noteheadSetRoundSmall` | 5 | Round (small) |
| `noteheadSetSacredHarp` | 30 | Shape-note (Sacred Harp) |
| `noteheadSetSlashHorizontalEnds` | 7 | Slash (horizontal ends) |
| `noteheadSetSlashVerticalEnds` | 6 | Slash (vertical ends) |
| `noteheadSetSlashed1` | 4 | Slashed variant 1 |
| `noteheadSetSlashed2` | 4 | Slashed variant 2 |
| `noteheadSetSquare` | 8 | Square |
| `noteheadSetTriangleDown` | 5 | Triangle down |
| `noteheadSetTriangleLeft` | 2 | Triangle left |
| `noteheadSetTriangleRight` | 2 | Triangle right |
| `noteheadSetTriangleUp` | 5 | Triangle up |
| `noteheadSetWithX` | 4 | Standard with X overlay |
| `noteheadSetX` | 6 | X-shaped |

#### Other Notation Element Classes

| Class | Glyphs | Description |
|---|---|---|
| `rests` | 22 | Standard rest glyphs (all durations) |
| `dynamics` | 42 | Dynamic marking glyphs |
| `tuplets` | 11 | Tuplet number glyphs |
| `articulations` | 36 | All articulation glyphs |
| `articulationsAbove` | 18 | Above-staff articulations |
| `articulationsBelow` | 18 | Below-staff articulations |
| `pauses` | 21 | Fermatas and other pause marks |
| `pausesAbove` | 7 | Above-staff pauses |
| `pausesBelow` | 7 | Below-staff pauses |
| `octaves` | 25 | Octave line markings (8va, 15ma, etc.) |
| `ornaments` | 55 | Ornament glyphs |
| `stemDecorations` | 32 | Stem decoration glyphs |
| `figuredBass` | 35 | Figured bass number/accidental glyphs |

#### Accidental Classes

| Class | Glyphs | Description |
|---|---|---|
| `accidentalsStandard` | 12 | Standard 12-EDO accidentals |
| `accidentals` | 461 | All accidentals (master class) |
| `accidentals24EDOArrows` | 12 | Gould arrow quartertone (24-EDO) |
| `accidentals53EDOTurkish` | 8 | Turkish folk music |
| `accidentals72EDOWyschnegradsky` | 22 | Wyschnegradsky (72-EDO) |
| `accidentalsAEU` | 8 | Arel-Ezgi-Uzdilek |
| `accidentalsArabic` | 9 | Arabic accidentals |
| `accidentalsHelmholtzEllis` | 70 | Helmholtz-Ellis just intonation |
| `accidentalsJohnston` | 8 | Johnston just intonation |
| `accidentalsPersian` | 2 | Persian accidentals |
| `accidentalsSagittalAthenian` | 40 | Sagittal Athenian (medium precision) |
| `accidentalsSagittalPromethean` | 94 | Sagittal Promethean (high precision) |
| `accidentalsSagittalPure` | 39 | Sagittal pure |
| `accidentalsSagittalTrojan` | 24 | Sagittal Trojan (12-EDO relative) |
| `accidentalsSims` | 6 | Sims (72-EDO) |
| `accidentalsSteinZimmermann` | 19 | Stein-Zimmermann |
| `accidentalsStockhausen` | 15 | Stockhausen (24-EDO) |

#### Time Signature and Tuplet Classes

These groups are defined in `ranges.json` in the SMuFL specification but are available as classes in Ooloi's unified class registry.

| Class | Glyphs | Description |
|---|---|---|
| `timeSignatures` | 32 | Standard time signature digits, common/cut time, fraction symbols, parentheses |
| `timeSignaturesReversed` | 12 | Reversed time signature digits |
| `timeSignaturesTurned` | 12 | Turned time signature digits |
| `tuplets` | 11 | Tuplet digits 0--9 and colon |

#### Medieval and Renaissance Ranges

These ranges are the primary source for `:additional-glyphs` cherry-picking when a notation element needs historical variants.

| Range | Glyphs | Description |
|---|---|---|
| `medievalAndRenaissanceClefs` | 12 | Mensural, Petrucci, and plainchant clefs |
| `medievalAndRenaissanceRests` | 9 | Medieval/Renaissance rest forms |
| `medievalAndRenaissanceNoteheadsAndStems` | 29 | Historical notehead forms |
| `medievalAndRenaissanceIndividualNotes` | 19 | Individual note forms |
| `medievalAndRenaissanceObliqueForms` | 32 | Oblique ligature forms |
| `medievalAndRenaissancePlainchantSingleNoteForms` | 18 | Plainchant single-note forms |
| `medievalAndRenaissancePlainchantMultipleNoteForms` | 22 | Plainchant multi-note forms |
| `kievanSquareNotation` | 15 | Kievan square notation (includes `kievanCClef`) |

### Ooloi-Defined Classes

SMuFL's `classes.json` does not provide sub-classes for percussion or tab clefs. Ooloi defines two additional classes from the general `clefs` class:

| Ooloi Class | Members | Source |
|---|---|---|
| `:clefsPercussion` | `unpitchedPercussionClef1`, `unpitchedPercussionClef2`, `semipitchedPercussionClef1`, `semipitchedPercussionClef2` | Extracted from the `clefs` general class |
| `:clefsTab` | `4stringTabClef`, `6stringTabClef` (plus `4stringTabClefSerif`, `4stringTabClefTall`, `6stringTabClefSerif`, `6stringTabClefTall` from font-specific optional glyphs where available) | Extracted from the `clefs` general class, extended with optional glyphs |

These are added to the unified class registry alongside classes from `classes.json` and `ranges.json`. They follow the same glyph selection pattern. Additional Ooloi-defined classes will be created as needed for other notation elements where SMuFL's classification granularity is insufficient.

### Font Coverage

Ooloi bundles 11 SMuFL fonts. The tables below also include [November 2](https://www.klemm-music.de/notation/november2/) --- a commercial font (not bundled) that Ooloi recommends as its primary notation font. Notation examples throughout Ooloi's documentation use November 2. It is available separately from Klemm Music Technology.

Glyph counts include both standard glyphs (from `glyphBBoxes`) and optional glyphs (from `optionalGlyphs`). Data extracted from the companion metadata JSON files shipped with each font.

#### Overview

| Font | Total | Style |
|---|---|---|
| **November 2** | **1,465** | **Traditional engraved (commercial --- recommended)** |
| Bravura | 3,416 | Traditional engraved (SMuFL reference) |
| Finale Maestro | 2,728 | Traditional engraved (classic Finale) |
| Petaluma | 1,877 | Handwritten (Real Book jazz) |
| Sebastian | 1,036 | Traditional engraved (distinct character) |
| Leipzig | 749 | Scholarly/musicological |
| Finale Jazz | 713 | Handwritten (bold jazz) |
| Finale Engraver | 534 | Traditional engraved (Urtext influence) |
| Leland | 490 | Traditional engraved (SCORE-inspired) |
| Finale Broadway | 427 | Handwritten (theatre/copyist) |
| Finale Legacy | 414 | Traditional engraved (Petrucci style) |
| Finale Ash | 297 | Handwritten (AshMusic) |

#### Core Notation Classes

Values show glyphs present / class size. Full coverage is highlighted in bold.

| Class (size) | Nov 2 | Bravura | Maestro | Petaluma | Sebastian | Leipzig | Leland | Engraver | Jazz | Legacy | Broadway | Ash |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `clefsG` (13) | 11 | **13** | **13** | **13** | 6 | 9 | 7 | 5 | 3 | 5 | 3 | 3 |
| `clefsF` (8) | **8** | **8** | **8** | **8** | 5 | 6 | 5 | 5 | 3 | 5 | 3 | 3 |
| `clefsC` (7) | 6 | **7** | **7** | **7** | 2 | 5 | 2 | 2 | 1 | 1 | 1 | 1 |
| `clefs` (49) | 37 | 48 | 47 | **49** | 23 | 31 | 23 | 18 | 12 | 16 | 12 | 10 |
| `rests` (22) | 14 | **22** | **22** | **22** | **22** | **22** | 19 | 19 | 15 | 20 | 15 | 14 |
| `articulations` (36) | 28 | **36** | 28 | 28 | 26 | 30 | 26 | 28 | 26 | 28 | 26 | 26 |
| `dynamics` (42) | 40 | 41 | 41 | **42** | **42** | 30 | 34 | 35 | 36 | 33 | 36 | 31 |
| `pauses` (21) | 20 | **21** | 20 | **21** | 19 | 20 | 12 | 4 | 4 | 5 | 5 | 5 |
| `ornaments` (55) | 33 | **55** | **55** | **55** | 53 | 25 | 42 | 11 | 10 | 13 | 10 | 7 |
| `octaves` (25) | 12 | **25** | **25** | **25** | 17 | 23 | 16 | 16 | 12 | 16 | 12 | 9 |
| `stemDecorations` (32) | 28 | **32** | **32** | 27 | 21 | 8 | 6 | 7 | 6 | 7 | 6 | 6 |
| `figuredBass` (35) | 0 | **35** | 32 | 33 | 30 | 25 | 0 | 0 | 0 | 0 | 0 | 0 |
| `accidentalsStandard` (12) | 11 | **12** | **12** | **12** | **12** | **12** | **12** | 7 | 7 | 7 | 7 | 7 |

**Reading the table.** November 2 has strong coverage of clefs, dynamics, pauses, and stem decorations but lacks figured bass and specialist rests. Bravura and Finale Maestro have near-complete coverage across all core classes. Petaluma is surprisingly comprehensive for a handwritten font. Sebastian and Leipzig provide good coverage of the essentials. The Finale handwritten fonts (Jazz, Broadway, Ash) cover the basics but lack specialist glyphs. Leland's metadata reports only 490 glyphs --- likely understating the actual font (MuseScore's default); its coverage of core classes is nonetheless adequate.

#### Notehead Sets

SMuFL defines 25 notehead set classes. Partial sets are usable --- the cascade falls back to the default glyph for any missing duration --- but complete sets give consistent visual results.

| Font | Full sets (of 25) | Partial | Empty | Notes |
|---|---|---|---|---|
| **November 2** | **13** | **7** | **5** | **Standard + variant sets; no names/pitch, Sacred Harp, arrows** |
| Bravura | **25** | 0 | 0 | Complete coverage |
| Finale Maestro | 24 | 1 | 0 | Near-complete |
| Petaluma | 22 | 3 | 0 | Missing: Sacred Harp, names-pitch partial, names-solfege partial |
| Sebastian | 15 | 6 | 4 | Good coverage of standard and variant sets |
| Leland | 4 | 7 | 14 | Default, slash, triangle sets |
| Leipzig | 3 | 7 | 15 | Default, slash sets |
| Finale Engraver | 2 | 15 | 8 | Default partial (14/25), Sacred Harp partial |
| Finale Jazz | 2 | 14 | 9 | Default partial, slash sets |
| Finale Legacy | 2 | 14 | 9 | Default partial, slash sets |
| Finale Broadway | 2 | 14 | 9 | Default partial, slash sets |
| Finale Ash | 0 | 11 | 14 | No complete sets; has basic notehead glyphs only |

**Note on `noteheadSetDefault`.** Only Bravura, Finale Maestro, and Petaluma have all 25 glyphs in `noteheadSetDefault`. Most fonts implement the 5 essential noteheads (whole, half, filled, double whole, noteheadNull) but lack the 20 specialist glyphs (parenthesised, small, large, cross variants). The essential noteheads are present in every bundled font.

#### Accidentals

| Font | Standard (12) | All accidentals (461) | Notes |
|---|---|---|---|
| **November 2** | **11** | **106** | **Good extended coverage; missing 1 standard accidental** |
| Bravura | **12** | 457 | Near-complete (missing 4 of 461) |
| Finale Maestro | **12** | 401 | Extensive microtonal coverage |
| Petaluma | **12** | 162 | Good: standard + Sagittal + quartertone |
| Sebastian | **12** | 49 | Standard + some extended |
| Leipzig | **12** | 38 | Standard + quartertone |
| Leland | **12** | 34 | Standard + quartertone |
| Finale Engraver | 7 | 21 | Missing 5 standard accidentals |
| Finale Jazz | 7 | 20 | Missing 5 standard accidentals |
| Finale Legacy | 7 | 20 | Missing 5 standard accidentals |
| Finale Broadway | 7 | 20 | Missing 5 standard accidentals |
| Finale Ash | 7 | 7 | Standard only (missing 5) |

**Missing standard accidentals in Finale fonts.** Finale Engraver, Jazz, Legacy, Broadway, and Ash all lack 5 of the 12 standard accidentals. These are likely the double sharp/flat and three-quarter-tone variants. Scores using these accidentals with a Finale handwritten font will fall back to Bravura or show reduced choices.

#### Observations

**November 2 (recommended).** With 1,465 glyphs and 13/25 full notehead sets, November 2 provides strong coverage for standard and moderately advanced notation. Its notable gaps are figured bass (0/35) and specialist rests (14/22), but core notation classes --- clefs, dynamics, pauses, articulations, stem decorations --- are well covered. November 2 is a commercial font and is not bundled with Ooloi.

**Tier 1 (full-coverage).** Bravura, Finale Maestro, and Petaluma provide comprehensive coverage suitable for any notation task. Bravura is the most complete, Petaluma is the most complete handwritten font.

**Tier 2 (good coverage).** Sebastian and Leipzig cover all core notation needs and provide reasonable specialist support. Sebastian has notably strong notehead set coverage (15/25 full). Leipzig excels at figured bass and scholarly notation.

**Tier 3 (basic coverage).** Leland, Finale Engraver, Finale Jazz, Finale Legacy, Finale Broadway, and Finale Ash cover the essentials for standard notation but lack specialist glyphs. Users choosing these fonts will see reduced choices in selectors; font availability filtering handles this gracefully.

**Low glyph count is not an error.** When a user switches to Finale Ash (297 glyphs), the glyph selectors shrink to show only what Ash provides. Notation elements whose current glyph selection is not available in the new font fall back through the cascade to the default glyph for their class. If the default glyph is also absent, the element is rendered using a fallback glyph from the same class that the font does provide.

The 11-font bundle provides five distinct traditional engraved aesthetics (Bravura, Finale Maestro, Leland, Sebastian, Finale Engraver), three handwritten styles (Petaluma, Finale Jazz, Finale Broadway), one scholarly style (Leipzig), and two lower-coverage fonts with distinctive character (Finale Ash, Finale Legacy). Users can also load additional SMuFL fonts through the font management UI ([ADR-0047](0047-Font-Management.md)).

---

## Rationale

### Why class-driven selection

The alternative is a hardcoded glyph mapping: `:treble` always renders as `gClef`, with no user choice. This is simple but inflexible. Users working with historical notation, educational materials, or house styles need to control glyph appearance. SMuFL's class system already provides the classification --- we simply use it.

### Why a cascade

A single global glyph choice is too coarse (every piece uses the same glyphs); a per-element choice is too fine (every clef must be individually configured). The three-level cascade gives the right granularity for each use case: house styles set organisation-wide defaults, pieces override where needed, and individual elements can be specialised.

### Why additional-glyphs instead of multiple classes

The initial design considered allowing multiple SMuFL classes per notation element (e.g., `#{:clefsC :medievalAndRenaissanceClefs}`). This was rejected because ranges like `medievalAndRenaissanceClefs` contain mixed glyph types --- G, F, and C clef glyphs together. Including the entire range for a C clef definition would offer G clef glyphs as alternatives, which is musically wrong. Cherry-picking individual glyphs via `:additional-glyphs` avoids this contamination while remaining simple and explicit.

### Why verbatim SMuFL glyph names

Converting `gClef8vb` to `:g-clef-8vb` would be more idiomatic Clojure but would introduce a translation layer between Ooloi's code and the SMuFL specification. Every developer, every debugging session, every spec lookup would require mental case conversion. Using the names verbatim (`:gClef8vb`) gives a direct, unambiguous mapping. Clojure keywords support camelCase; there is no technical reason to convert.

---

## Consequences

**Positive**

- Users can choose alternative visual styles for any notation element through a uniform mechanism. The same cascade applies to clefs, noteheads, articulations, and all other elements.
- House styles become meaningful for glyph selection: an organisation can define defaults that all pieces inherit.
- The glyph selection mechanism is entirely data-driven. Adding support for a new notation element category requires defining its class association and default glyph, not new code.
- Font availability filtering ensures users only see valid choices. Switching to a low-coverage font never produces errors or missing glyphs --- the UI adapts silently.
- Historical notation support (mensural, plainchant, Kievan) is achieved through `:additional-glyphs` without special-casing in the architecture.

**Negative**

- The cascade adds a resolution step to rendering. The cost is negligible (three map lookups) but the code path exists.
- Ooloi-defined classes (`:clefsPercussion`, `:clefsTab`) must be maintained as SMuFL evolves. If a future SMuFL version introduces native sub-classes for these, the Ooloi definitions should be retired.
- The class inventory documented here is a snapshot of SMuFL 1.4. Future SMuFL versions may add, rename, or reorganise classes.

---

## References

### Related ADRs

- [ADR-0006: SMuFL](0006-SMuFL.md) --- adopts SMuFL as the sole font standard
- [ADR-0047: Font Management](0047-Font-Management.md) --- font registry, spec-level metadata loading, bundled font strategy, font availability filtering
- [ADR-0049: Clef Registry](0049-Clef-Registry.md) --- first concrete implementation of this glyph selection architecture, applied to clefs
- [ADR-0016: Settings](0016-Settings.md) --- piece settings store piece-level glyph selection overrides
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) --- resolves logical keyword to glyph at render time by walking the cascade
- [ADR-0045: Instrument Library](0045-Instrument-Library.md) --- references logical clef keywords in staff definitions; unaffected by glyph selection

### External References

- [SMuFL Specification](https://w3c.github.io/smufl/latest/) --- Standard Music Font Layout, W3C Community Group
- [SMuFL classes](https://w3c.github.io/smufl/latest/specification/classes.html) --- glyph classification
- [SMuFL ranges](https://w3c.github.io/smufl/latest/tables/clefs.html) --- canonical glyph tables
