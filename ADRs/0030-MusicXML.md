# ADR-0030: MusicXML Interoperability Excellence (Importer/Exporter)

## Status

**Accepted** (with plugin architecture implementation)

## Date

September 24, 2025 (Proposed)
October 19, 2025 (Accepted and repositioned)
October 19, 2025 (Updated) - Revised for plugin implementation and compressed timeline

## Table of Contents

1. [Context](#context)
2. [Decision](#decision)
3. [Rationale](#rationale)
4. [Scope](#scope)
5. [Non-Goals](#non-goals)
6. [Architecture Overview](#architecture-overview)
7. [Implementation As Plugin](#implementation-as-plugin)
8. [MusicXML-IR Contract & Invariants](#musicxml-ir-contract--invariants)
9. [Single Library + Vendor Profiles Architecture](#single-library--vendor-profiles-architecture)
10. [Importer Design](#importer-design)
11. [Exporter Design](#exporter-design)
12. [Versioning & Packaging Policy](#versioning--packaging-policy)
13. [Determinism Requirements](#determinism-requirements)
14. [Vendor Profiles](#vendor-profiles)
15. [Common Interop Issues by Vendor](#common-musicxml-interop-issues-by-vendor-field-observations)
16. [Validation & Semantics](#validation--semantics)
17. [CLI & Reports](#cli--reports)
18. [Performance & Scale Targets](#performance--scale-targets)
19. [Performance Gates](#performance-gates)
20. [Implementation Roadmap](#implementation-roadmap)
21. [Repair Function Catalog](#repair-function-catalog)
22. [Risks & Mitigations](#risks--mitigations)
23. [Alternatives Considered](#alternatives-considered)
24. [Success Criteria](#success-criteria)
25. [Consequences](#consequences)

---

## Strategic Repositioning (October 19, 2025)

**Critical Development Reality**: MusicXML import is not a late-stage interoperability feature - it is **essential test infrastructure** required NOW for all subsequent development phases.

**Why This Cannot Wait**:

1. **Rendering Development Validation**: Cannot fully validate plugin-based formatters (Phase 12) without real-world test material. Synthetic test fixtures insufficient for validating professional music engraving against real-world edge cases.

2. **Layout Testing Requires Real Scores**: Symphonies, operas, chamber works, piano repertoire provide edge cases that synthetic generators cannot anticipate (tuplet nesting, cross-staff beaming, extreme dynamics, microtones, percussion notation).

3. **Performance Validation Needs Scale**: Large orchestral scores required to benchmark timewalk traversal, memory optimization, hash-consing effectiveness. Cannot validate 647-line timewalk implementation without real 100k+ event pieces.

4. **Open Source Trust Requirements**: Both import AND export must ship together for transparency and trustworthiness. Round-trip validation demonstrates commitment to interoperability excellence.

5. **Competitive Positioning**: MusicXML excellence establishes Ooloi as interoperable hub from day one, eliminating lock-in fears and enabling risk-free evaluation.

**Implementation Priority**: Phase 11 (following Phase 10 plugin system implementation, as MusicXML will be implemented as first canonical plugin).

**Architectural Dependencies**:
- **Phase 8**: Windowing System (UI foundation) - Required for visual feedback
- **Phase 9**: Skija Integration (GPU rendering) - Required for rendering imported scores
- **Phase 10**: Plugin System Implementation - Required framework for MusicXML as plugin
- **Phase 11**: MusicXML Import/Export - First canonical plugin demonstrating system
- **Phase 12**: Rendering Pipeline - Validated using real MusicXML test scores

**Library Integration Strategy**: Leverage existing MusicXML cleaning/repair libraries where beneficial. Do not reinvent validated parsing/repair patterns.

---

## Context

MusicXML remains the de facto interchange format among major notation software (Sibelius, Dorico, Finale, MuseScore) despite verbosity and uneven vendor interpretations. Publishers and engravers evaluate tools primarily on import/export reliability.

Current implementations suffer systematic failures (tie/slur confusion, tuplet encoding errors, voice loss, cross-staff beaming breaks, non-deterministic output). Vendors often treat MusicXML as a begrudged path rather than a core feature. Ooloi needs a strategic wedge: **MusicXML excellence** lets publishers evaluate Ooloi on real catalog material with minimal risk.

## Decision

Ship a canonical plugin `ooloi-musicxml` (importer + exporter) with best-in-class fidelity, determinism, and speed. Provide batch catalog tooling and actionable validation reports. Open-source the plugin, golden corpus, and validators.

The plugin will use the existing VPD API to build/traverse Ooloi structures, requiring no backend architectural extensions. This validates that the plugin architecture (ADR-0003) provides sufficient capabilities for complex format interchange.

## Rationale

Excellence in MusicXML eliminates lock-in fear via trustworthy round-trip, enabling risk-free trials and immediate value (cleaner files, faster processing). This leverages Ooloi strengths (immutability → deterministic output; VPD → precise errors; hash-consing → fast import; plugin architecture → vendor-specific handling) and positions Ooloi as the interoperable hub.

**Early Development Value**: MusicXML import provides immediate development infrastructure benefits independent of adoption strategy:
- Test data: Import scores from IMSLP/MuseScore.com for testing
- Examples: Real repertoire for documentation/blog posts
- Load testing: Actual orchestral complexity for performance validation
- Notation validation: Cross-staff beaming, tuplets, etc. from existing corpus

This justifies implementing MusicXML earlier than Phase 17 (professional features), potentially as Phase 10.5 (after gRPC hardening, before Skija integration) to support layout engine development.

## Scope

**In scope:** Core CMN (pitches, rests, tuplets, beams, slurs/ties, articulations, dynamics, directions, lyrics, parts, transposing instruments), cross-staff beaming, basic percussion mapping, optional preservation of explicit layout hints.
**Out of scope (v1):** MEI, exotic graphical notation, engraving algorithms, full publisher-specific layout reproduction.

## Non-Goals

No general XML framework. No invented layout beyond preserving explicit hints. No support for MusicXML < 3.0.

## Architecture Overview

Parse/write via a **normalized MusicXML-IR** (small, immutable).

* **Import:** streaming SAX/StAX → IR → normalization → Ooloi objects (heuristics toggleable).
* **Export:** Ooloi → normalized IR → canonical writer (stable ordering/IDs).

## Implementation As Plugin

MusicXML import/export will be implemented as a **backend plugin** (`ooloi-musicxml`), not as core backend functionality.

**Architectural Benefits:**
- Plugin uses existing VPD API (no special backend extensions required)
- Import: Parse MusicXML → call `api/add-musician`, `api/set-measure`, etc.
- Export: Use `timewalk` to traverse piece → generate XML
- Vendor profiles stored as plugin configuration data
- Third parties can build alternative implementations
- Core backend remains format-agnostic

**Plugin Dependencies:**
- Single XML parsing library (e.g., `musicxml-java`, `proxymusic`)
- XSD validation library
- Standard JVM libraries (no exotic dependencies)

**API Surface Required:**
All existing VPD-based operations are sufficient:
- Getters: Navigate piece structure
- Setters: Build Ooloi objects during import
- `timewalk`: Temporal traversal for export
- Transaction boundaries: Atomic import operations
- Error reporting: Structured error codes with VPD loci

**No new backend capabilities required.** The API that exists for manual editing is sufficient for automated import/export.

## MusicXML-IR Contract & Invariants

* Units normalized (durations in ticks; offsets as `(measure, fractional-beat)`).
* Pitch = `{step, alter, octave}`; tuplets reduced to lowest integer ratios.
* Voices are 1-based per staff; anchors are `(measure, offset, staff, voice)`.
* Canonical ordering keys for beams, directions, and notations.
* **Unknown/unsupported preservation channel:** attach typed metadata (original XML + locus) to nearest logical object; round-trip via `<other-*>` where legal; otherwise warn. Enabled by default (`--preserve-unknowns`).

## Single Library + Vendor Profiles Architecture

**Not multiple libraries.** One generic MusicXML parser + configuration-driven vendor-specific repairs.

### Library Selection
Use single JVM-compatible XML parsing library:
- `musicxml-java` (reference implementation)
- `proxymusic` (JAXB-based)
- Or similar generic MusicXML parser

**All vendor handling is post-parse normalization, not different parsing libraries.**

### Vendor Detection

```clojure
;; Auto-detect from file metadata
<identification>
  <encoding>
    <software>Dorico 5.1.20</software>
  </encoding>
</identification>

;; Detection with confidence scoring
(defn detect-vendor [parsed-xml]
  {:vendor (detect-from-software-tag parsed-xml)
   :confidence (if (explicit-tag?) :high :medium)
   :fallback-indicators (analyze-structural-patterns parsed-xml)})
```

### Vendor Profile Structure

Profiles are **configuration data**, not code:

```clojure
;; profiles/sibelius.edn
{:name "Sibelius"
 :repairs #{:tie-as-slur          ; Ties encoded as slurs
            :voice-infer          ; Missing <voice> tags
            :beam-relink          ; Broken beam chains
            :bar-number-offset    ; Anacrusis numbering
            :hidden-rest-spacing} ; Spacing rests
 :heuristics {:tie-slur-disambiguation :prefer-same-pitch
              :voice-inference :by-stem-direction}}

;; profiles/dorico.edn
{:name "Dorico"
 :repairs #{:slur-numbering-stabilize
            :hairpin-endpoint-fix
            :xstaff-anchor-infer
            :tuplet-normalize}
 :heuristics {:cross-staff :staff-of-notation
              :microtones :smufl-with-fallback}}
```

### Repair Pipeline

```clojure
(defn import-with-profile [file]
  (let [parsed (parse-musicxml file)           ; Generic parse
        vendor (detect-vendor parsed)           ; Auto-detect
        profile (load-profile vendor)]          ; Load config
    (-> parsed
        (apply-repairs profile)                 ; Normalize
        (convert-to-ooloi-api-calls))))        ; Build structures
```

**Repairs are composable functions:**
- Each repair is independent, testable function
- Applied in sequence based on profile configuration
- Users can override detected profile or customize repair set

### Why This Architecture

1. **Single parser to maintain** - one dependency, one security surface
2. **Composable repairs** - profiles are data, easy to modify/extend
3. **Transparent behavior** - users can see/customize which repairs apply
4. **Testable in isolation** - each repair function has unit tests
5. **Vendor evolution tracking** - update profiles, not code, when vendors change

## Importer Design

**Modes**

* `--strict`: XSD + semantic validation; fail on ambiguities.
* `--repair`: apply limited heuristics; emit warnings with loci `(measure:beat:staff:voice)`.

**Heuristics (repair mode only)**

* Tie vs slur disambiguation; cross-measure tie reconciliation.
* Tuplet legality normalization (including nested).
* Voice inference where `<voice>` is missing/inconsistent.
* Grace ordering/anchors; ties from grace to main.
* Cross-staff beaming consistency (staff-of-notation vs staff-of-sounding).

**Provenance**
Attach source XML line/indices to created objects for precise diagnostics.

**Parser Safety**

* Disable external entities (XXE).
* Limits on recursion depth, attribute size/count.
* `.mxl` ZIP protections (zip-bomb thresholds, path traversal prevention).
* MIME/extension validation.

## Exporter Design

**Output Requirements**

* Deterministic bytes (stable element/attribute order; stable IDs).
* Layout policy: `--layout=preserve|none`. Preserve only explicit breaks/distances; never invent defaults.

**Domain Handling**

* Transposition: concert/written switch; key signature correction.
* Microtones: strategy `:accidental-text | :smufl | :cents` with fallbacks.
* Percussion: instrument/notehead mappings where representable; warn otherwise.

## Versioning & Packaging Policy

* Default export **3.1** as `.mxl` (uncompressed `mimetype` first).
* Auto-bump to **4.0** only if required; warn when forcing 3.1 would lose fidelity.
* Import validates as **4.0**; accepts 3.0+.

## Determinism Requirements

* Identical input ⇒ identical bytes (incl. whitespace/attribute order).
* **Deterministic packaging for `.mxl`:** `mimetype` first; fixed file order; UTF-8 no BOM; LF line endings; zeroed ZIP timestamps; stable compressor level.
* **Stable IDs:** `{part-index}.{measure-number}.{voice}.{event-seq}` for cross-refs (ties/slurs/spanners).
* Generator metadata PI: exporter version, profile, SHA-256 of logical content.
* `--deterministic` verifies SHA-256 per output.

## Vendor Profiles

Minimal toggles for known dialect deltas: `generic | dorico | sibelius | finale | musescore`.
Document each toggle; default `generic`. Auto-detect profile heuristically (confidence score; allow override).

## Common MusicXML Interop Issues by Vendor (Field Observations)

> Use these heuristics to guide profile toggles, validators, and repairs. Each issue has a corpus example + error code.

### Dorico
* **Slur numbering quirks** in dense overlaps; occasional off-by-one in multi-layer passages → `SLUR_NUMBER_MISMATCH`
* **Tuplet export** prefers explicit `time-modification`; nested tuplets legal but need canonicalization → `TUPLET_NORMALIZE_REQUIRED`
* **Cross-staff beaming** correct semantically but staff-of-notation vs staff-of-sounding can be implicit → `XSTAFF_ANCHOR_INFER`
* **Dynamics/hairpins** anchored as directions; endpoints present but sometimes require stable ID reassignment → `HAIRPIN_ENDPOINT_FIXUP`
* **Microtones** via SMuFL accidentals; fall back to `accidental-text` when font data is missing → `MICROTONE_STRATEGY_SWITCH`

### Sibelius
* **Tie vs slur confusion** in older exports; ties encoded as `<slur>` across measures → `TIE_AS_SLUR`
* **Voices** missing `<voice>` in polyphonic layers; hidden rests used for spacing → `VOICE_MISSING`, `HIDDEN_REST_SPACING`
* **Beams** level/continue markers dropped on cross-staff moves → `BEAM_CHAIN_BROKEN`
* **Grace notes** ordering and tie-to-main ambiguities → `GRACE_ANCHOR_AMBIG`
* **Measure numbering** pickups/anacrusis produce implicit bars; numbering drift → `BAR_NUMBER_OFFSET`

### Finale
* **Tuplets** inconsistent ratio vs duration encoding, especially on copy/paste regions → `TUPLET_RATIO_ILLEGAL`
* **Duplicate IDs** for ties/slurs in long passages → `ID_DUPLICATE`
* **Expressions/dynamics** placed as directions with loose anchors; need re-attachment to rhythmic loci → `DIRECTION_REANCHOR`
* **Percussion maps** pitch/unpitched ambiguity; staff-line semantics lost → `PERC_MAP_AMBIG`
* **Part vs score** layout hints diverge; mixed break directives → `LAYOUT_PART_SCORE_DIVERGENCE`

### MuseScore
* **Rapid format evolution** attribute/element ordering unstable between minor versions → `NONDETERMINISTIC_ORDERING`
* **Hairpins** start/stop present but occasionally missing end IDs after edits → `HAIRPIN_LOST_STOP`
* **Voices/layers** aggressive voice compaction merges lines without `<voice>` → `VOICE_COLLAPSE`
* **Cross-measure ties** ends dropped on repeat boundaries → `TIE_END_MISSING`
* **Courtesy accidentals & enharmonics** encoded as text in edge cases → `ACCIDENTAL_TEXT_FALLBACK`

### Shared Across Vendors
* **Nested/irrational tuplets** need normalization and legality checks → `TUPLET_NORMALIZE_REQUIRED`
* **Repeat/volta graphs** inconsistent with playbook order → `REPEAT_GRAPH_INCOHERENT`
* **Lyrics melismas** extender lines not spanning correctly → `LYRIC_EXTENDER_FIXUP`
* **Microtones** mixed strategies (SMuFL vs text) in the same file → `MICROTONE_MIXED_STRATEGY`

**Detection & Repair (per profile)**

* `sibelius`: `:tie-as-slur :voice-infer :beam-relink :bar-number-offset`
* `finale`: `:tuplet-normalize :id-dedupe :direction-reanchor :perc-map-clarify`
* `dorico`: `:slur-numbering-stabilize :hairpin-endpoint-fix :xstaff-anchor-infer`
* `musescore`: `:id-stabilize :hairpin-stop-recover :tie-end-recover :voice-preserve`

## Validation & Semantics

**Schema:** XSD (3.1/4.0).
**Semantic validators:** rhythmic sums & pickup math; tuplet legality/nesting; tie/slur graph consistency; repeat/volta coherence; lyric melismas/extenders; dynamics/hairpin anchors.
Export must pass schema; `--strict` import rejects invalid files with actionable errors.

## CLI & Reports

```
ooloi mxml import   <in.{xml,mxl}> --out <dir> [--strict|--repair] [--profile <vendor>] [--report report.json]
ooloi mxml export   <in.ool>       --out <file.mxl> --version 3.1|4.0 --layout preserve|none [--profile <vendor>] [--deterministic]
ooloi mxml validate <in.{xml,mxl}> [--strict] --report <report.json>
ooloi catalog       <dir/>         --out <report-dir/> [--strict|--repair] [--profile <vendor>] [--deterministic]
```

**Exit codes:** 0 OK · 1 warnings · 2 schema/semantic errors · 3 internal error.

**Report JSON:** per-file timings; input/output sizes; schema result; semantic issues (machine-readable codes, severity, locus, suggested repair); round-trip summary.
**HTML catalog report:** success rate, time/size deltas, top issues, per-file drill-down.
**Error code taxonomy:** e.g., `TIE_MISSING_NUMBER`, `TUPLET_RATIO_ILLEGAL`, `VOICE_INCONSISTENT`, `CROSS_STAFF_BROKEN`; severities `error|warn|info`.

## Performance & Scale Targets

Linear passes; no quadratic phases. Handle ≥100k events:

* Import ≤10 s (6-core 2019 Intel).
* Export ≤7 s.
* ≤500 MB RSS during processing.
* Deterministic under concurrency (single writer).

### Repair Performance Budget

Vendor-specific repairs must not significantly impact import time:
- Detection: <100ms per file
- Profile loading: <10ms (cached after first load)
- Repair application: <500ms total for all repairs
- Target: <10% overhead vs. generic parse

**Achieved through:**
- Linear-time algorithms (no quadratic phases in repairs)
- Early termination in repair functions
- Profile caching
- Repair composition (apply only configured repairs)

## Performance Gates

Release-blocking checks in CI: byte-stable exports on corpus; time/RSS budgets enforced; per-commit metrics snapshot; fast-fail on regressions.

## Implementation Roadmap

### Revised Timeline: Best-in-Class in 3-4 Weeks

**Key Insight**: The repair set is surprisingly small (~15-20 distinct functions handle 95%+ of issues). Vendor bugs are repetitive and concentrated, not scattered across hundreds of edge cases.

### Week 1: Foundation + High-Impact Repairs
**Plugin Infrastructure:**
- Plugin manifest and lifecycle
- Generic XML parser integration
- Vendor detection from metadata
- Profile loading system

**Top 5 Repairs (handles 70-80% of issues):**
1. Sibelius tie-as-slur disambiguation
2. Finale duplicate ID fixing
3. Voice inference (all vendors)
4. Tuplet normalization (shared issue)
5. Cross-staff beam reconnection

**Testing:** Basic round-trip validation on each repair

**Deliverable:** "Ooloi handles Sibelius ties better than Dorico does"

### Week 2: Core Vendor Intelligence
**Additional 10 Repairs (brings to 95%+ coverage):**
6. Hairpin endpoint resolution
7. Grace note ordering
8. Direction re-anchoring
9. Measure numbering offset
10. Cross-staff anchor inference
11. Lyric extender fixing
12. Slur numbering stabilization
13. Repeat/volta graph validation
14. Percussion map clarification
15. Microtone strategy selection

**Profile Configuration:**
- Complete profiles for Sibelius, Finale, Dorico, MuseScore
- Generic fallback profile
- Profile override and customization

**Testing:** Comprehensive repair test suite with corpus samples

**Deliverable:** "Ooloi is best-in-class for MusicXML import"

### Week 3: Export + Round-Trip Validation
**Export Implementation:**
- Ooloi structure → MusicXML via timewalk
- Deterministic writer (stable ordering, hashed IDs)
- XSD validation
- Layout policy (preserve explicit hints vs. omit)

**Round-Trip Testing:**
- Import → Export → Re-import validation
- SHA-256 content hashing
- Automated corpus testing
- Performance benchmarking

**Deliverable:** "Ooloi proves interoperability superiority"

### Week 4: Integration + Documentation
**CLI Integration:**
```bash
ooloi mxml import <file> [--profile <vendor>] [--strict|--repair]
ooloi mxml export <piece-id> [--version 3.1|4.0] [--layout preserve|none]
ooloi mxml validate <file> --report <output.json>
```

**Documentation:**
- Plugin development guide
- Vendor profile reference
- Known limitations
- Repair function catalog

**Positioning:**
- Blog post with benchmark comparisons
- Example imports showing vendor-specific repairs
- Round-trip SHA-256 verification demos

**Deliverable:** "The wooden horse arrives at court"

### Beyond Week 4: Refinement (Ongoing)
- Field data from user reports
- New vendor pattern detection
- Profile updates for vendor evolution
- Golden corpus expansion
- Community contributions

**Note:** The original ADR's 12-month timeline included extensive publisher partnerships, public benchmarking campaigns, and comprehensive golden corpus curation. Those activities support **positioning and adoption strategy**, not technical capability. The technical superiority (best-in-class import/export) is achievable in 3-4 weeks.

## Repair Function Catalog

### Implementation Complexity Tiers

**Trivial Repairs (1-2 hours each):**
- Duplicate ID deduplication
- Voice inference by stem direction
- Measure numbering offset adjustment
- Accidental text fallback handling
- Non-deterministic ordering stabilization

**Moderate Repairs (3-6 hours each):**
- Tie-as-slur disambiguation (heuristic-based)
- Beam chain reconnection
- Tuplet ratio normalization
- Hairpin endpoint inference
- Grace note ordering resolution
- Direction re-anchoring to rhythmic loci
- Lyric extender span fixing

**Complex Repairs (1-2 days each):**
- Cross-staff beaming consistency
- Voice collapse reconstruction
- Slur numbering in dense overlaps
- Repeat/volta graph coherence validation
- Percussion map semantic clarification

### Estimated Implementation Time

- Trivial repairs: 8 repairs × 1-2 hours = 8-16 hours (1-2 days)
- Moderate repairs: 8 repairs × 3-6 hours = 24-48 hours (3-6 days)
- Complex repairs: 4 repairs × 1-2 days = 4-8 days

**Total core implementation: 8-16 days**

Plus:
- Profile configuration: 1-2 days
- Detection logic: 1 day
- Testing infrastructure: 3-5 days

**Total: ~3 weeks for best-in-class capability**

### Why So Few Repairs Needed

1. **Vendor bugs are repetitive**: Sibelius always encodes ties as slurs; once fixed, works for all Sibelius files
2. **Most files are clean**: PDMX corpus (250k files) shows 0.017% corruption rate
3. **Repairs are composable**: Same function used across multiple profiles with different thresholds
4. **Pareto principle**: 15 repairs handle 95% of issues; long tail doesn't block "best in class"

## Risks & Mitigations

* **Vendor dialect evolution:** small profiles; nightly CI on vendor samples.
* **Corner-case notation:** precise warnings; no silent loss; public feature matrix.
* **Performance drift:** enforce budgets in CI.

## Alternatives Considered

* **MEI** (deferred).
* **Hand-rolled DOM parsers** (rejected; streaming required).
* **Embedded layout engine** (rejected; keep interchange vs presentation separate).

## Success Criteria

* ≥99% round-trip fidelity on golden corpus.
* Deterministic bytes & SHA-256 across runs.
* Meets timing/memory targets.
* Publisher catalog runs produce actionable reports with high pass rates.
* **Vendor repair validation**: Each repair function has:
  - Unit test with broken input sample
  - Expected normalized output
  - Round-trip verification (import → export → re-import)
  - Performance budget compliance (<100ms per repair)
* **Comparative benchmarking**: Import success rate on test corpus:
  - Ooloi: ≥95% (with vendor profiles)
  - Dorico: ~85% (documented baseline)
  - MuseScore: ~80% (documented baseline)
* **Round-trip fidelity**: Content-addressed hashing proves:
  - Import(file) → Export() → Import() produces identical structure
  - SHA-256 verification across round-trip
  - Deterministic output (byte-stable exports)

## Consequences

**Positive:**
- Establishes Ooloi as reference implementation
- Measurable competitive edge
- Network effects
- Showcases architectural strengths
- **Rapid competitive differentiation**: Best-in-class status achievable in 3-4 weeks vs. 12-month expectation
- **Plugin architecture validation**: Proves VPD API sufficient for complex format interchange without backend extensions
- **Development infrastructure**: Immediate value for testing, examples, load validation independent of adoption strategy
- **Phasing flexibility**: Can implement early (Phase 10.5) to support layout engine development, not deferred to Phase 17

**Negative:**
- Sustained engineering investment (validators, corpus, deterministic writer)
- Maintenance for profiles/evolution
- High public expectations
- **Plugin system maturity required**: MusicXML forces early completion of plugin specification (ADR-0003), including lifecycle, API surface, error handling, configuration

**Mitigation:** open corpus/impl; CI with performance gates; clear docs on scope/gaps.