# ADR-0049: Clef Registry

## Status

Accepted

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Clef Definition Format](#clef-definition-format)
  - [Musical Semantics of the Fields](#musical-semantics-of-the-fields)
  - [Default Clef Registry](#default-clef-registry)
  - [Additional Glyphs for G Clefs](#additional-glyphs-for-g-clefs)
  - [Clef Change Glyphs](#clef-change-glyphs)
  - [User-Defined Clefs](#user-defined-clefs)
  - [Piece Lifecycle](#piece-lifecycle)
  - [Impact on Existing Code](#impact-on-existing-code)
- [Rationale](#rationale)
- [Open Questions](#open-questions)
- [Consequences](#consequences)
- [References](#references)

---

## Context

Ooloi currently uses flat keywords for clefs: `:treble`, `:bass`, `:alto`, `:tenor`, `:soprano`, `:mezzo-soprano`, `:baritone`, `:treble-8va`, `:treble-8vb`, `:treble-15ma`, `:treble-15mb`, `:bass-8va`, `:bass-8vb`, `:bass-15ma`, `:bass-15mb`, `:percussion`, `:tab` (17 keywords). These are defined in `shared/src/main/clojure/ooloi/shared/models/musical/staff.clj` as `valid-clefs`.

Limitations of the current system:

- **No connection to SMuFL glyph names.** The keyword `:treble` carries no information about which glyph to render or which alternatives exist.
- **No user-defined clefs.** The set is hardcoded. A French violin clef (G clef on line 1) or sub-bass clef (F clef on line 5) cannot be represented.
- **No alternative glyph styles.** Users cannot choose a 19th century F clef, a mensural C clef, or any visual variant.
- **Conflated semantics.** The keywords conflate glyph type, staff position, and octave transposition into a single name.
- **Colloquially fuzzy names.** "Treble" and "bass" are universally understood by musicians but have no formal definition in the SMuFL specification.

The clef registry is the first concrete implementation of the general glyph selection mechanism defined in [ADR-0048](0048-SMuFL-Glyph-Selection-Architecture.md). The same pattern will extend to noteheads, rests, accidentals, articulations, time signature digits, and all other notation elements with visual variants.

---

## Decision

The clef registry is a map from keyword to clef definition. It is a standalone data structure --- like the instrument library ([ADR-0045](0045-Instrument-Library.md)) --- that ships with a standard default set, is copied into each new piece, and is user-editable per piece.

### Clef Definition Format

```clojure
{:class              <keyword>         ;; SMuFL class: :clefsG, :clefsF, :clefsC, :clefsPercussion, :clefsTab
 :line               <integer>         ;; Staff line (bottom = 1, top = 5) --- mutually exclusive with :space
 :space              <integer>         ;; Staff space (bottom = 1) --- mutually exclusive with :line
 :pitch              <string>          ;; Pitch of the line/space the clef sits on (ADR-0026 format: "G4", "C4", "F3")
 :additional-glyphs  <set of keywords> ;; Optional: extra SMuFL glyph keywords beyond the class
}
```

Exactly one of `:line` or `:space` is present. `:additional-glyphs` is absent when not needed.

The available glyphs for a clef = (members of `:class` from `classes.json`) ∪ `:additional-glyphs`, filtered by the active font's glyph inventory ([ADR-0048 §Font Availability Filtering](0048-SMuFL-Glyph-Selection-Architecture.md#font-availability-filtering)).

### Musical Semantics of the Fields

**`:class`** --- determines the clef shape family. A G clef (`:clefsG`) marks the line it sits on as G in whatever octave `:pitch` specifies. An F clef (`:clefsF`) marks its line as F. A C clef (`:clefsC`) marks its line as C. The class is the primary classifier and determines which SMuFL glyphs are valid visual alternatives.

**`:line` / `:space`** --- where the clef sits on the staff. Standard 5-line staff: lines are numbered 1 (bottom) to 5 (top), spaces are numbered 1 (between lines 1--2) to 4 (between lines 4--5). Positions outside 1--5 (ledger line territory) are valid for exotic clef placements.

**`:pitch`** --- the sounding pitch of the line or space the clef occupies, in [ADR-0026](0026-Pitch-Representation-and-Operations.md) string format. For standard clefs, this is determined by the clef shape: G clefs mark G4 (or G3 for 8vb, G5 for 8va, etc.), F clefs mark F3 (or F2 for 8vb, etc.), C clefs mark C4. For user-defined clefs, the pitch is explicit --- any glyph on any line with any pitch. The pitch field is essential for MIDI input and for computing the pitch of every staff position.

**`:additional-glyphs`** --- cherry-picks individual SMuFL glyphs that aren't in the primary class but should be available as visual alternatives. This avoids pulling in an entire mixed class (like the `medievalAndRenaissanceClefs` range, which contains G, F, and C clef glyphs together). See [ADR-0048 §The Additional-Glyphs Mechanism](0048-SMuFL-Glyph-Selection-Architecture.md#the-additional-glyphs-mechanism).

### Default Clef Registry

```clojure
{;; G clefs
 :treble      {:class :clefsG :line 2 :pitch "G4"}
 :treble-8va  {:class :clefsG :line 2 :pitch "G5"}
 :treble-15ma {:class :clefsG :line 2 :pitch "G6"}
 :treble-8vb  {:class :clefsG :line 2 :pitch "G3"}
 :treble-15mb {:class :clefsG :line 2 :pitch "G2"}

 ;; C clefs
 :soprano       {:class :clefsC :line 1 :pitch "C4"
                 :additional-glyphs #{:mensuralCclef :chantCclef
                                      :mensuralCclefPetrucciPosLowest}}
 :mezzo-soprano {:class :clefsC :line 2 :pitch "C4"
                 :additional-glyphs #{:mensuralCclef :chantCclef
                                      :mensuralCclefPetrucciPosLow}}
 :alto          {:class :clefsC :line 3 :pitch "C4"
                 :additional-glyphs #{:mensuralCclef :chantCclef :kievanCClef
                                      :mensuralCclefPetrucciPosMiddle}}
 :tenor         {:class :clefsC :line 4 :pitch "C4"
                 :additional-glyphs #{:mensuralCclef :chantCclef
                                      :mensuralCclefPetrucciPosHigh}}
 :baritone      {:class :clefsC :line 5 :pitch "C4"
                 :additional-glyphs #{:mensuralCclef :chantCclef
                                      :mensuralCclefPetrucciPosHighest}}

 ;; F clefs
 :bass      {:class :clefsF :line 4 :pitch "F3"
             :additional-glyphs #{:mensuralFclef :mensuralFclefPetrucci :chantFclef}}
 :bass-8vb  {:class :clefsF :line 4 :pitch "F2"}
 :bass-15mb {:class :clefsF :line 4 :pitch "F1"}
 :bass-8va  {:class :clefsF :line 4 :pitch "F4"}
 :bass-15ma {:class :clefsF :line 4 :pitch "F5"}

 ;; Percussion and tab
 :percussion {:class :clefsPercussion :line 3 :pitch "C4"}
 :tab        {:class :clefsTab        :line 3 :pitch "C4"}}
```

The 17 default entries correspond exactly to the current `valid-clefs` vector. The existing keyword names are preserved --- `:treble`, `:bass`, `:alto`, etc. remain the identifiers. The registry adds structured data behind each keyword without changing the identifiers that the rest of the codebase uses.

### Additional Glyphs for G Clefs

The G clef defaults above do not include `:additional-glyphs` because the base `clefsG` class already contains 13 variants including the octave-transposed glyphs. If historical G clef variants are desired:

```clojure
:treble {:class :clefsG :line 2 :pitch "G4"
         :additional-glyphs #{:mensuralGclef :mensuralGclefPetrucci :medRenGClefCMN}}
```

Whether to include mensural alternatives by default is a house style decision. The default registry takes the conservative position: mensural glyphs are included for C and F clefs (where they are more commonly needed by users working with Renaissance repertoire) but not for G clefs (where the modern glyph is overwhelmingly standard).

### Clef Change Glyphs

When a clef changes mid-staff, it is conventionally drawn smaller. SMuFL provides dedicated glyphs for this: `gClefChange`, `cClefChange`, `fClefChange`. These are derived by convention from the class:

| Class | Change glyph |
|---|---|
| `:clefsG` | `:gClefChange` |
| `:clefsF` | `:fClefChange` |
| `:clefsC` | `:cClefChange` |
| `:clefsPercussion` | (none --- use main glyph at reduced size) |
| `:clefsTab` | (none --- use main glyph at reduced size) |

The derivation is handled by the rendering pipeline, not stored in the clef definition. If the active font lacks the change glyph, the main glyph is rendered at a reduced size.

### User-Defined Clefs

Users can add entries to the piece's clef registry:

```clojure
;; French violin clef (G clef on line 1)
:french-violin {:class :clefsG :line 1 :pitch "G4"}

;; Sub-bass clef (F clef on line 5)
:sub-bass {:class :clefsF :line 5 :pitch "F3"}

;; Baritone F clef (F clef on line 3 --- distinct from baritone C clef)
:baritone-f {:class :clefsF :line 3 :pitch "F3"}

;; Hypothetical: C clef on a space
:c-clef-space-2 {:class :clefsC :space 2 :pitch "C4"}
```

User-defined clefs use the same format, participate in the same glyph selection cascade ([ADR-0048](0048-SMuFL-Glyph-Selection-Architecture.md)), and are fully functional throughout the piece model. Any combination of class, line/space position, and pitch is valid.

### Piece Lifecycle

1. **New piece**: the default clef registry is copied into the piece.
2. **Editing**: the user modifies the piece's clef registry via the clef editor (adding, removing, or redefining entries). Changes affect only this piece.
3. **Instrument assignment**: when an instrument is assigned from the library ([ADR-0045](0045-Instrument-Library.md)), its staff definitions reference clef keywords. These keywords must exist in the piece's clef registry. If a library instrument references a clef keyword that doesn't exist in the piece (e.g., a user deleted `:alto`), the assignment fails with a clear error.
4. **Rendering**: the rendering pipeline ([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)) resolves the clef keyword → clef definition → glyph through the cascade ([ADR-0048](0048-SMuFL-Glyph-Selection-Architecture.md)). The backend uses the resolved glyph's bounding box and anchor data from the font's metadata to compute spatial layout.

### Impact on Existing Code

The clef registry replaces the flat `valid-clefs` vector and `valid-clef-set` in `staff.clj`. All code that currently validates clef keywords against `valid-clef-set` will instead validate against the piece's clef registry. All code that currently uses clef keywords directly for rendering will instead resolve through the glyph selection cascade.

**Files affected** (non-exhaustive):

| Area | Files | Change |
|---|---|---|
| Clef definition | `shared/.../models/musical/staff.clj` | `valid-clefs` → clef registry; `create-staff` default clefs unchanged (still `:treble`) |
| Clef validation | `shared/.../specs/generators.clj` | Validate against registry instead of hardcoded set |
| UI clef selectors | `frontend/.../ui/core/cljfx.clj` | Combo-box items from registry; display names from i18n |
| Instrument library | `backend/resources/.../instrument_library_default.edn` | No change --- clef keywords unchanged |
| i18n translation keys | `shared/resources/i18n/*.po` | No change initially --- keywords unchanged; new keys added for any new clef entries |
| Glyph rendering | Rendering pipeline | Resolve keyword → definition → glyph through cascade |
| Tests | All test files referencing clef keywords | No change to keywords; new tests for registry operations |

---

## Rationale

### Why a registry instead of enriching the keywords

The alternative is to keep flat keywords and add a lookup function that derives the class, line, and pitch from the keyword name. This fails for user-defined clefs (no keyword convention to parse) and for unusual configurations (C clef on a space). The registry makes everything explicit.

### Why copy into each piece

The alternative is a global registry that all pieces share. This fails when a user defines a custom clef for one piece --- it should not appear in other pieces. Copying follows the same pattern as the instrument library: a default set is copied into each new piece and becomes the piece's own.

### Why preserve the existing keyword names

The keywords `:treble`, `:bass`, `:alto`, `:tenor`, etc. are universally understood by musicians. They appear in the instrument library, in test fixtures, in i18n translation files, and throughout the codebase. Renaming them (e.g., `:treble` → `:g-clef`) would require changes across all three projects and all locale files for no functional benefit. The registry makes the keyword an arbitrary identifier --- its meaning is fully defined by the registry entry, not by parsing the keyword string.

### Why `:line` / `:space` instead of a single numeric field

The alternative is a single `:position` field where integers represent lines and half-integers represent spaces (e.g., line 3 = `3`, space between lines 2--3 = `2.5`). This is computationally convenient but musically unintuitive. Musicians think "C clef on line 3" or "clef on space 2", not "position 3.0" vs "position 2.5". Separate `:line` and `:space` keys match the mental model, and the conversion to a numeric position for layout computation is trivial.

---

## Open Questions

1. **Clef keyword naming**: should `:treble` → `:g-clef`, `:bass` → `:f-clef`, `:alto` → `:c-clef`? The registry makes this purely cosmetic --- the keyword is an identifier, and its mapping to glyph class, line, and pitch is explicit. The traditional names are universally understood by musicians. Decision deferred.

2. **Clef editor UI**: a dedicated editor window (like the instrument library editor) for managing the piece's clef registry. Design to be specified separately.

3. **House style persistence**: how house style glyph defaults are stored and shared. This is a cross-cutting concern that applies to all notation elements, not just clefs. Design to be specified separately.

4. **Petrucci position assignments**: the `:additional-glyphs` sets in the default registry match `mensuralCclefPetrucciPosLowest` → soprano (line 1), `PosLow` → mezzo-soprano (line 2), etc. This mapping from position name to staff line is a reasonable starting point but may need refinement based on historical musicological feedback.

---

## Consequences

**Positive**

- Every clef keyword now carries structured data: which glyph class it belongs to, where it sits on the staff, and what pitch it defines. This information was previously implicit.
- Users can define custom clefs with any combination of glyph class, staff position, and pitch. Historical and experimental notation is supported without special-casing.
- The glyph selection cascade ([ADR-0048](0048-SMuFL-Glyph-Selection-Architecture.md)) applies to clefs uniformly: house style defaults, piece overrides, and local overrides all work.
- The existing 17 clef keywords are unchanged. All current code, instrument library entries, i18n translations, and tests continue to work without modification.
- Font availability filtering ensures users only see clef glyphs that the active font provides. Switching to a low-coverage font never produces errors.

**Negative**

- Clef validation becomes registry-dependent: a keyword is valid if and only if the piece's clef registry contains it. Code that previously checked against a compile-time constant now checks against a runtime data structure.
- The piece's clef registry must be serialised and persisted alongside the piece data. This adds a small amount of per-piece data.
- If a user deletes a clef entry from their piece's registry and an instrument in the piece references that clef keyword, the assignment becomes invalid. The UI must prevent or clearly handle this case.

---

## References

### Related ADRs

- [ADR-0048: SMuFL Glyph Selection Architecture](0048-SMuFL-Glyph-Selection-Architecture.md) --- the general mechanism this ADR applies to clefs
- [ADR-0047: Font Management](0047-Font-Management.md) --- font registry, spec-level metadata loading, font availability filtering
- [ADR-0006: SMuFL](0006-SMuFL.md) --- SMuFL as the sole font standard
- [ADR-0045: Instrument Library](0045-Instrument-Library.md) --- instruments reference clef keywords in staff definitions
- [ADR-0026: Pitch Representation and Operations](0026-Pitch-Representation-and-Operations.md) --- string-based pitch format used in `:pitch` field
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) --- resolves clef keyword → glyph at render time
- [ADR-0016: Settings](0016-Settings.md) --- piece settings store piece-level glyph selection overrides

### External References

- [SMuFL Specification: Clefs](https://w3c.github.io/smufl/latest/tables/clefs.html) --- canonical clef glyph table
- [SMuFL classes](https://w3c.github.io/smufl/latest/specification/classes.html) --- glyph classification
