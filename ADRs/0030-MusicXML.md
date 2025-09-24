# ADR-0030: MusicXML Interoperability Excellence (Importer/Exporter)

## Status

Proposed

## Date

September 24, 2025

## Table of Contents

1. [Context](#context)
2. [Decision](#decision)
3. [Rationale](#rationale)
4. [Scope](#scope)
5. [Non-Goals](#non-goals)
6. [Architecture Overview](#architecture-overview)
7. [MusicXML-IR Contract & Invariants](#musicxml-ir-contract--invariants)
8. [Importer Design](#importer-design)
9. [Exporter Design](#exporter-design)
10. [Versioning & Packaging Policy](#versioning--packaging-policy)
11. [Determinism Requirements](#determinism-requirements)
12. [Vendor Profiles](#vendor-profiles)
13. [Common Interop Issues by Vendor](#common-musicxml-interop-issues-by-vendor-field-observations)
14. [Validation & Semantics](#validation--semantics)
15. [CLI & Reports](#cli--reports)
16. [Performance & Scale Targets](#performance--scale-targets)
17. [Performance Gates](#performance-gates)
18. [Implementation Roadmap](#implementation-roadmap)
19. [Risks & Mitigations](#risks--mitigations)
20. [Alternatives Considered](#alternatives-considered)
21. [Success Criteria](#success-criteria)
22. [Consequences](#consequences)

---

## Context

MusicXML remains the de facto interchange format among major notation software (Sibelius, Dorico, Finale, MuseScore) despite verbosity and uneven vendor interpretations. Publishers and engravers evaluate tools primarily on import/export reliability.

Current implementations suffer systematic failures (tie/slur confusion, tuplet encoding errors, voice loss, cross-staff beaming breaks, non-deterministic output). Vendors often treat MusicXML as a begrudged path rather than a core feature. Ooloi needs a strategic wedge: **MusicXML excellence** lets publishers evaluate Ooloi on real catalog material with minimal risk.

## Decision

Ship a canonical plugin `ooloi-musicxml` (importer + exporter) with best-in-class fidelity, determinism, and speed. Provide batch catalog tooling and actionable validation reports. Open-source the plugin, golden corpus, and validators.

## Rationale

Excellence in MusicXML eliminates lock-in fear via trustworthy round-trip, enabling risk-free trials and immediate value (cleaner files, faster processing). This leverages Ooloi strengths (immutability → deterministic output; VPD → precise errors; hash-consing → fast import; plugin architecture → vendor-specific handling) and positions Ooloi as the interoperable hub.

## Scope

**In scope:** Core CMN (pitches, rests, tuplets, beams, slurs/ties, articulations, dynamics, directions, lyrics, parts, transposing instruments), cross-staff beaming, basic percussion mapping, optional preservation of explicit layout hints.
**Out of scope (v1):** MEI, exotic graphical notation, engraving algorithms, full publisher-specific layout reproduction.

## Non-Goals

No general XML framework. No invented layout beyond preserving explicit hints. No support for MusicXML < 3.0.

## Architecture Overview

Parse/write via a **normalized MusicXML-IR** (small, immutable).

* **Import:** streaming SAX/StAX → IR → normalization → Ooloi objects (heuristics toggleable).
* **Export:** Ooloi → normalized IR → canonical writer (stable ordering/IDs).

## MusicXML-IR Contract & Invariants

* Units normalized (durations in ticks; offsets as `(measure, fractional-beat)`).
* Pitch = `{step, alter, octave}`; tuplets reduced to lowest integer ratios.
* Voices are 1-based per staff; anchors are `(measure, offset, staff, voice)`.
* Canonical ordering keys for beams, directions, and notations.
* **Unknown/unsupported preservation channel:** attach typed metadata (original XML + locus) to nearest logical object; round-trip via `<other-*>` where legal; otherwise warn. Enabled by default (`--preserve-unknowns`).

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

## Performance Gates

Release-blocking checks in CI: byte-stable exports on corpus; time/RSS budgets enforced; per-commit metrics snapshot; fast-fail on regressions.

## Implementation Roadmap

### Phase 1 (Months 1–3): Foundation
* Streaming parser with hardened IO security
* MusicXML-IR implementation 
* Canonical writer with stable ordering
* Golden corpus repository with CI integration

### Phase 2 (Months 2–4, overlapping): Vendor Intelligence
* Vendor failure pattern cataloging
* Error code taxonomy implementation
* Minimal vendor profiles
* Repair rules for top 20 common issues

### Phase 3 (Months 4–7): Core Compliance
* 95%+ MusicXML 3.1 element coverage
* Full semantic validator suite
* Round-trip fidelity testing harness
* Schema-compliant export guarantee
* Performance tuning with hash-consing on import

### Phase 4 (Months 6–9): Real-World Validation
* Publisher trial partnerships
* Batch processing tools and HTML reporting
* Golden corpus expansion with field data
* Differential testing against major tools

### Phase 5 (Months 8–12): Polish and Positioning
* UX refinement and documentation
* Auto-detection and profile suggestion
* Public benchmarking and advocacy
* Community building and testimonials

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

## Consequences

**Positive:** establishes Ooloi as reference implementation; measurable competitive edge; network effects; showcases architectural strengths.
**Negative:** sustained engineering investment (validators, corpus, deterministic writer); maintenance for profiles/evolution; high public expectations.
**Mitigation:** open corpus/impl; CI with performance gates; clear docs on scope/gaps.