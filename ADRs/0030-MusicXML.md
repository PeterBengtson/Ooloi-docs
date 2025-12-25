# ADR-0030: MusicXML Interoperability (Importer/Exporter)

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
4. [Non-Goals](#non-goals)
5. [Architecture Overview](#architecture-overview)
6. [Plugin Implementation](#plugin-implementation)
7. [Vendor-Specific Handling](#vendor-specific-handling)
8. [Implementation Approach](#implementation-approach)
9. [Success Criteria](#success-criteria)
10. [Consequences](#consequences)

---

## Implementation Rationale (October 19, 2025)

**Development Reality**: MusicXML is not a late-stage interoperability feature or an addon like any other—it is the integral plugin that drives Ooloi's core implementation to feature-completeness.

**MusicXML as Development Driver (October 20, 2025)**:

MusicXML import is the mechanism by which Ooloi reaches feature-completeness. Rather than building all notation elements first and then adding import/export, import drives which notation elements get implemented and when.

This makes MusicXML import unique among plugins:
- **First canonical plugin**: Implemented before full on-screen rendering exists
- **Core development driver**: Defines implementation order (atoms → simple extent → complex extent)
- **Implementation driver**: By the time of open source release, this plugin will have driven implementation of all core notation elements
- **Quality gate**: Round-trip validation of complex works validates architectural soundness before public release

**Rationale for Early Implementation**:

1. **Core Implementation Strategy**: MusicXML import defines which Ooloi elements (pitches, slurs, beams, dynamics, lyrics) get implemented and in what order. Start with atomic elements (no extent), progress to simple spanning elements, finish with complex spanning elements.

2. **Rendering Development Validation**: Plugin-based formatters are difficult to validate fully without real-world test material. Synthetic test fixtures are insufficient for validating professional music engraving against real-world edge cases.

3. **Layout Testing Requires Real Scores**: Symphonies, operas, chamber works, piano repertoire provide edge cases that synthetic generators may not cover (tuplet nesting, cross-staff beaming, extreme dynamics, microtones, percussion notation).

4. **Performance Validation Needs Scale**: Large orchestral scores are required to benchmark timewalk traversal, memory optimization, hash-consing effectiveness. The 647-line timewalk implementation requires real 100k+ event pieces for validation.

5. **Bidirectional Implementation**: Both import and export ship together. Round-trip validation enables verification of semantic preservation.

## MusicXML as External Definition of Done (October 20, 2025)

**MusicXML as Definition of Core Completeness**

When Ooloi can import complex contemporary works from MusicXML and export them accurately for import in other programs, core completeness is achieved. This is the end goal; notational capability is added gradually and the MusicXML plugin evolves during the process.

**Why Complex Notation as the Benchmark**:

1. **Objective Validation**: Not "does it work for our test cases?" but "can it handle what professional engravers actually publish?"

2. **Real-World Complexity Litmus Test**: Contemporary professional notation represents legitimate professional repertoire that notation systems should handle.

3. **Bidirectional Round-Trip Proof**: Import from MusicXML → manipulate in Ooloi → export to MusicXML → open in Finale/Sibelius/Dorico/MuseScore with high fidelity. Measurable and testable.

4. **Architectural Completeness**: Every data structure, transformation, and rendering decision should be production-ready to handle this complexity.

5. **Industry Interoperability**: Implementation works with existing professional music notation software. Real publisher catalogs provide validation material.

**Success Criteria**:
- Import complex contemporary works from MusicXML with minimal semantic loss
- Export to all major notation software with high visual fidelity
- Round-trip validation: import → export → re-import produces structurally identical results
- SHA-256 content verification across round-trip cycle

**Complex Notation as Coverage Proof**:

This complexity level serves as an upper-bound engineering validation. If Ooloi handles the most complex contemporary notation, then it necessarily handles Classical/Romantic repertoire, avant-garde composition, and contemporary commercial notation needs. Complex scores sit at the ceiling; commercial/pop notation requirements are covered well before reaching this benchmark.

This provides a falsifiable milestone that prevents scope drift and "perpetual beta" syndrome. When the most complex notation round-trips successfully, the core architectural work is complete.

**Implementation Priority**: Follows plugin system implementation, as MusicXML will be implemented as first canonical plugin.

**Architectural Dependencies**:
- Windowing System (UI foundation) - Required for visual feedback
- Skija Integration (GPU rendering) - Required for rendering imported scores
- Plugin System Implementation - Required framework for MusicXML as plugin
- MusicXML Import/Export - First canonical plugin demonstrating system
- Rendering Pipeline - Validated using real MusicXML test scores

**Library Integration Strategy**: Leverage existing MusicXML cleaning/repair libraries where beneficial. Do not reinvent validated parsing/repair patterns.

---

## Context

MusicXML remains the de facto interchange format among major notation software (Sibelius, Dorico, Finale, MuseScore) despite verbosity and uneven vendor interpretations. For users to evaluate Ooloi, they need to import their existing files to see how they render. This should not be a one-way operation - users should be able to export files again without loss of content or fidelity.

MusicXML interchange involves common challenges including tie/slur confusion, tuplet encoding variations, voice representation differences, cross-staff beaming complexities, and output determinism. Real-world MusicXML files contain vendor-specific patterns and deviations from the specification that are handled through repair functions.

## Decision

Ship a canonical plugin `ooloi-musicxml` (importer + exporter) with comprehensive roundtrip fidelity, determinism, and vendor-specific repair capabilities. Provide batch catalog tooling and actionable validation reports. Open-source the plugin, golden corpus, and validators.

The plugin will use the existing VPD API to build/traverse Ooloi structures, requiring no backend architectural extensions. This validates that the plugin architecture (ADR-0003) provides sufficient capabilities for complex format interchange.

## Rationale

**Technical foundation:** Ooloi's internal representation is semantic rather than layout-based. This makes generating clean, correct MusicXML straightforward - the export writes what the structure means, not how it was rendered. Import is more complex, requiring vendor-specific repair functions to handle files with vendor-specific patterns or spec deviations.

**Development driver:** Using MusicXML import to drive feature implementation ensures that all core notation elements reach production quality. Import defines implementation order (atomic elements → simple spanning → complex spanning) and provides real-world test material.

**Bidirectional workflow:** Roundtrip fidelity (import → modify → export with semantic preservation) allows users to evaluate Ooloi with existing catalogs. Files can be exported again for use in other software.

**Technical outcome:** MusicXML import drives feature implementation. The semantic IR architecture simplifies export generation. Imported scores provide real-world validation material.

**Development Infrastructure**: MusicXML import provides development infrastructure benefits:
- Test data: Import scores from IMSLP/MuseScore.com for testing
- Examples: Real repertoire for documentation/blog posts
- Load testing: Actual orchestral complexity for performance validation
- Notation validation: Cross-staff beaming, tuplets, etc. from existing corpus

This justifies implementing MusicXML earlier in the development cycle to support layout engine development and provide real-world test material.

## Non-Goals

No general XML framework. No invented layout beyond preserving explicit hints. No support for MusicXML < 3.0.

## Architecture Overview

The plugin follows a three-stage pipeline:

**Import:** MusicXML file → Normalized IR → Ooloi structures via VPD API calls
**Export:** Ooloi structures → Normalized IR → MusicXML file

The intermediate representation (IR) normalizes MusicXML into a consistent internal form before conversion, handling vendor variations and spec deviations.

## Plugin Implementation

MusicXML import/export is implemented as a backend plugin (`ooloi-musicxml`), not as core backend functionality.

**Key properties:**
- Uses existing VPD API exclusively (no backend extensions required)
- Import builds Ooloi structures through VPD API calls
- Export traverses structures using `timewalk` to generate MusicXML
- Vendor-specific handlers stored as plugin configuration data
- Core backend remains format-agnostic

**Dependencies:**
- XML parsing library (e.g., ProxyMusic or JAXB-generated bindings)
- XSD validation library
- Standard JVM libraries

The VPD API provides all required operations: navigation, construction, traversal, and error reporting. This validates that the plugin architecture supports complex format interchange without backend modifications.


## Vendor-Specific Handling

Real-world MusicXML files contain vendor-specific patterns and deviations from the specification. The plugin handles these through:

- **Vendor detection**: Automatic detection from file metadata with confidence scoring
- **Configuration profiles**: Vendor-specific repair sets stored as configuration data
- **Composable repairs**: Independent, testable repair functions applied based on profile
- **User control**: Override detection or customize repair sets

Common patterns requiring repair include tie/slur disambiguation, voice inference, tuplet normalization, cross-staff beaming consistency, and ID stabilization. The repair set is compact (~15-20 functions handle 95%+ of issues).

## Implementation Approach

**Import modes:**
- Strict mode: XSD + semantic validation; fail on ambiguities
- Repair mode: Apply heuristics; emit warnings with precise locations

**Export requirements:**
- Deterministic output (stable element/attribute order, stable IDs)
- Layout preservation policy (preserve explicit hints or omit)
- XSD validation

**Validation:**
- Schema validation (XSD 3.1/4.0)
- Semantic validators (rhythmic sums, tuplet legality, tie/slur graph consistency, etc.)

## Success Criteria

* High round-trip fidelity (≥99%) on test corpus
* Deterministic output (byte-stable exports, SHA-256 verification)
* Meets performance targets for large scores
* Vendor repair functions validated through unit tests and round-trip verification
* Publisher catalog processing produces actionable reports

## Consequences

**Positive:**
- Interoperability with existing notation software through standard format
- Format compatibility enables file exchange with other tools
- **Plugin architecture validation**: Validates VPD API sufficiency for complex format interchange without backend extensions
- **Development infrastructure**: Provides test data, examples, and load validation material
- **Implementation flexibility**: Can implement earlier in development cycle to support layout engine development

**Negative:**
- Sustained engineering investment (validators, corpus, deterministic writer)
- Maintenance for profiles/evolution