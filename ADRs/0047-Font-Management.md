# ADR-0047: Font Management

## Status

Accepted

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Font Registry Component](#font-registry-component)
  - [Dual Registration](#dual-registration)
  - [Metadata Loading](#metadata-loading)
    - [Per-Font Metadata](#per-font-metadata)
    - [Spec-Level Metadata](#spec-level-metadata)
  - [Bundled Fonts](#bundled-fonts)
  - [Local Font Discovery](#local-font-discovery)
  - [User-Supplied Fonts](#user-supplied-fonts)
  - [Active Font Selection](#active-font-selection)
  - [Piece Font Manifest and Version Matching](#piece-font-manifest-and-version-matching)
  - [UI Integration](#ui-integration)
- [Rationale](#rationale)
- [Consequences](#consequences)
- [References](#references)

---

## Context

[ADR-0006](0006-SMuFL.md) establishes SMuFL as Ooloi's sole font standard for musical symbol rendering. This ADR addresses the concrete question of how SMuFL fonts are loaded, managed, and kept consistent across the two rendering paths (Skija for score rendering, JavaFX for UI controls) and across multiple clients in a collaborative session.

The rendering pipeline is backend-authoritative (ADR-0038). The backend computes layout — glyph positions, stem lengths, beam angles, spacing — from a specific font version's metadata. The frontend renders the resulting paintlists using the corresponding typeface. If the two sides disagree about which font is active, or use different versions of the same font, the result is misaligned notation. Font management exists to prevent this.

Two distinct rendering paths consume the same font data. The Skija GPU rendering layer draws musical notation in the score; JavaFX draws SMuFL glyphs in UI controls — clef selectors, symbol palettes, toolbars, and any other control that displays a musical symbol. Both paths must use the same font from the same source, governed by the same metadata, at all times. The font registry exists to enforce this invariant.

---

## Decision

### Font Registry Component

The font registry is an Integrant component with a standard lifecycle. Its state is a map of registered fonts keyed by font family name. Each entry contains:

| Field | Type | Description |
|---|---|---|
| `:font-bytes` | `byte[]` | Raw font file data |
| `:skija-typeface` | `Typeface` | Skija typeface, created via `Typeface/makeFromData` |
| `:javafx-family` | `String` | Family name as returned by `Font.loadFont` |
| `:metadata` | `map` | Parsed SMuFL JSON: glyphs, anchors, bounding boxes, engraving defaults |
| `:source` | keyword | One of `:bundled`, `:local`, `:user-supplied` |
| `:version` | `String` | Font version string extracted from the font's name table |

The component initialises by loading all bundled fonts, then running the local discovery scan. It halts by releasing Skija typeface resources.

### Dual Registration

Each font is registered with both rendering systems from the same source bytes:

1. **Skija**: `Typeface/makeFromData` creates a typeface from the byte array. The rendering pipeline references this typeface when drawing musical symbols in the score.
2. **JavaFX**: `Font.loadFont` on an `InputStream` over the same bytes registers the font family with JavaFX's internal font system. UI controls reference the font by family name.

Both registrations happen in a single function. There is no path by which one rendering system can see a font the other cannot.

### Metadata Loading

The font registry consumes two distinct layers of SMuFL metadata: spec-level metadata that defines the universal vocabulary and classification of all SMuFL glyphs, and per-font metadata that describes which glyphs a specific font provides and their metrics.

#### Per-Font Metadata

Each SMuFL font ships with a companion `metadata.json` file conforming to the SMuFL specification. The registry parses this JSON at registration time and stores the result alongside the font. The parsed metadata includes:

- **Glyph names → codepoints**: the mapping from canonical SMuFL names to Unicode Private Use Area codepoints.
- **Glyph bounding boxes**: per-glyph extents in staff spaces, consumed by the rendering pipeline for spatial calculations.
- **Anchor points**: stem attachments, optical centres, and other positioning data.
- **Engraving defaults**: staff line thickness, stem width, beam spacing, and other font-specific defaults the rendering pipeline uses when no user override exists.

The rendering pipeline reads all glyph metrics from the registry. There is no separate metadata loading path.

#### Spec-Level Metadata

Independent of any specific font, the SMuFL specification defines three JSON files that describe the universal glyph vocabulary and its organisation. These are bundled at `shared/src/main/clojure/ooloi/shared/smufl/metadata/` and loaded once at registry initialisation (they are font-independent and never change between fonts):

| File | Contents | Purpose |
|---|---|---|
| `glyphnames.json` | Canonical name, Unicode codepoint, and description for every glyph in the SMuFL specification | Authoritative glyph vocabulary. The rendering pipeline, UI controls, and any component that references a SMuFL glyph by name resolves it through this file. |
| `classes.json` | Named groupings of functionally related glyphs (e.g., `clefsG` contains all G clef variants, `noteheadSetDefault` contains the standard notehead set) | Glyph classification. Drives user-facing glyph selection: when a user chooses an alternative visual variant for a notation element, the class determines which glyphs are valid alternatives. |
| `ranges.json` | Named ranges of glyphs as they appear in the SMuFL specification layout, with descriptions (e.g., `medievalAndRenaissanceClefs`, `rests`, `noteheads`) | Broader organisational groupings. Used alongside classes to discover related glyphs across historical periods and notation systems. |

**Why both layers are necessary.** Per-font metadata tells the registry what a specific font *can render* and how (metrics, bounding boxes). Spec-level metadata tells the registry what the glyphs *are* and how they relate to each other (classification, vocabulary). The intersection — glyphs that exist in the spec AND are present in the active font — determines what the application can offer the user at any given moment.

**Glyph classification drives the glyph selection mechanism.** The classes from `classes.json` define which glyphs are valid visual alternatives for each notation element. The full glyph selection mechanism — the three-level cascade (house style → piece → local), two selection patterns (single glyph vs. set), the `:additional-glyphs` mechanism for historical variants, and the complete class inventory — is specified in [ADR-0048: SMuFL Glyph Selection Architecture](0048-SMuFL-Glyph-Selection-Architecture.md).

**Font availability filtering.** Not every SMuFL font implements every glyph. When the user is presented with visual alternatives for a notation element, the choices are the intersection of the element's SMuFL class membership and the active font's glyph inventory (from its per-font metadata). Glyphs defined in the spec but absent from the active font simply do not appear as choices. No error, no placeholder — the font's capabilities are respected silently.

### Bundled Fonts

Ooloi ships with eleven SMuFL-compliant notation fonts as JAR resources in `shared/resources/smufl-fonts/`, each with its companion SMuFL metadata JSON. Companion text fonts for lyrics, expression markings, and other text-alongside-music live in `shared/resources/music-text-fonts/`.

**Bundled notation fonts:**

| Font | Style | Developer | License |
|---|---|---|---|
| Bravura (v1.392) | Traditional engraved (SMuFL reference) | Steinberg | SIL OFL |
| Finale Maestro | Traditional engraved (classic Finale) | MakeMusic | SIL OFL |
| Finale Engraver | Traditional engraved (Urtext influence) | MakeMusic | SIL OFL |
| Finale Legacy | Traditional engraved (Petrucci style) | MakeMusic | SIL OFL |
| Leland (v0.80) | Traditional engraved (SCORE-inspired) | MuseScore | SIL OFL |
| Sebastian (v1.35) | Traditional engraved (distinct character) | Florian Kretlow & Ben Byram-Wigfield | SIL OFL |
| Petaluma (v1.065) | Handwritten (Real Book jazz) | Steinberg | SIL OFL |
| Finale Jazz | Handwritten (bold jazz) | MakeMusic | SIL OFL |
| Finale Broadway | Handwritten (theatre/copyist) | MakeMusic | SIL OFL |
| Finale Ash | Handwritten (legendary AshMusic) | MakeMusic | SIL OFL |
| Leipzig (v5.2.96) | Scholarly/musicological | RISM Digital | SIL OFL |

Bravura is the canonical fallback: it is always available, always loads first, and serves as the default active font.

Bundled fonts are the tested, known-good pairing with the rendering pipeline. A specific version of each font is validated against the layout engine before each Ooloi release. Upgrading a bundled font is an application release, not a user action — the TeX model applies here. Knuth does not let users swap in a different Computer Modern and expect the same line breaks; the font and the layout engine are a tested pair.

### Local Font Discovery

After loading bundled fonts, the registry scans for locally installed SMuFL-compliant fonts. Font files and SMuFL metadata JSON are stored in separate locations on all platforms — the OS font system knows nothing about SMuFL metadata. The scanner must search both sets of directories and correlate results by font family name. The set of platforms Ooloi actually ships for is defined in [ADR-0050: Platform Support Policy](0050-Platform-Support-Policy.md); the tables below cover all three supported OSes.

**Font file locations (standard OS font directories):**

| Platform | User | System |
|---|---|---|
| **macOS** | `~/Library/Fonts/` | `/Library/Fonts/` |
| **Windows** | `%LOCALAPPDATA%\Microsoft\Windows\Fonts\` | `C:\Windows\Fonts\` |
| **Linux** | `~/.local/share/fonts/` | `/usr/share/fonts/`, `/usr/local/share/fonts/` |

**SMuFL metadata locations (separate, per the SMuFL specification):**

| Platform | User | System |
|---|---|---|
| **macOS** | `~/Library/Application Support/SMuFL/Fonts/<name>/<name>.json` | `/Library/Application Support/SMuFL/Fonts/<name>/<name>.json` |
| **Windows** | `%LOCALAPPDATA%/SMuFL/Fonts/<name>/<name>.json` | `%COMMONPROGRAMFILES%/SMuFL/Fonts/<name>/<name>.json` |
| **Linux** | `$XDG_DATA_HOME/SMuFL/Fonts/<name>/<name>.json` | `$XDG_DATA_DIRS/SMuFL/Fonts/<name>/<name>.json` |

A font file found in an OS font directory with matching metadata in a SMuFL directory is a fully discoverable SMuFL font. A font file without metadata but matching a known SMuFL font family name (e.g., Bravura, Petaluma, Leland) is flagged as a potential SMuFL font but cannot be fully registered without its metrics. Metadata found without a corresponding font file is ignored.

When the scan finds a font already present in the registry (same family name), the registry compares version strings. If the local version differs from the bundled version, the registry **does not substitute it**. On first detection, it posts a **permanent alert notification** informing the user that a different version was found locally. Clicking the notification opens the font management UI, where the user chooses which version to use.

The notification persists until resolved. If the user **dismisses** the notification without making a choice, it reappears at the next startup — the unresolved version difference has not gone away. If the user **opens the font management UI and makes an explicit version choice**, that choice is persisted and the notification does not reappear. The font management UI remains accessible at any time for the user to change the decision later.

Discovery without silent substitution. Nothing changes without an explicit user decision, and decisions once made are respected permanently.

### User-Supplied Fonts

The font management UI allows the user to load additional SMuFL fonts from arbitrary file paths. These go through the same registration pipeline: load bytes, register with Skija and JavaFX, parse metadata, validate SMuFL compliance, add to registry. No system installation is required — Skija loads typefaces from byte arrays, not from the OS font system.

The user's active font selection is persisted in application settings and restored at startup. If a previously selected user-supplied font is no longer available at the stored path, the registry falls back to Bravura and posts a notification.

### Active Font Selection

The registry maintains a single active notation font. All rendering — backend layout computation and frontend drawing — uses the active font's metadata and typeface. Switching the active font triggers a full re-render: the backend recomputes layout with the new font's metrics and the frontend redraws with the new typeface.

The active font selection is stored in application settings (see ADR-0016 / ADR-0043). The setting stores the font family name and source, not a file path — the registry resolves the font at startup.

### Piece Font Manifest and Version Matching

Each piece records the notation font used for its layout: font family name and exact version string. This is a **hard constraint**, not advisory metadata.

When a client opens a piece, the registry checks whether the exact font version is available locally. If the version does not match exactly, the piece **cannot be rendered**. The client receives a clear error identifying the required font and version, not a degraded or approximate rendering.

This is the only honest position given the rendering architecture. The backend computes layout — glyph positions, stem lengths, beam angles, spacing — from a specific font version's metrics. A different version of the same font may have different stem widths, altered anchor points, or changed bounding boxes. Rendering with a mismatched version produces subtly incorrect output with no visible cause. Refusing to render is better than rendering incorrectly.

In a multi-user environment where multiple clients connect to the same server, this means all clients working on a piece must have the exact font version that piece was created with. This is a deliberate constraint: the font and the layout are a tested pair, and the pair must match across all participants. If a client lacks the required version, the font management UI provides the path to acquire and register it.

### UI Integration

A JavaFX control displaying a SMuFL glyph sets its text content to the appropriate Unicode Private Use Area codepoint and its font family to the active font's JavaFX family name from the registry. No image files, no SVG paths, no special rendering. It is a character in a font.

When the active font changes, every UI control displaying SMuFL glyphs updates because the font family resolves to the new typeface. The mechanism is JavaFX's standard font resolution, not a custom invalidation system.

This applies to clef selectors, key signature displays, note value selectors, articulation palettes, and any other UI element that shows a musical symbol. The same glyph the Skija score renderer draws appears in the UI through JavaFX's normal text layout — one font, two rendering paths, one source of truth.

---

## Rationale

### Why bundled-first

The rendering pipeline is backend-authoritative (ADR-0038). The backend computes glyph metrics, bounding boxes, and anchor points from SMuFL metadata to produce paintlists; the frontend renders those paintlists using the corresponding typeface. If the two sides disagree about which font is active — or use different versions of the same font — the result is misaligned notation. Bundling a specific, tested version of each font and loading it by default eliminates this class of defect.

### Why discovery without silent substitution

A system-installed font might be a newer version with different stem widths, altered anchor points, or additional glyphs. Silently preferring it over the bundled version produces layout drift with no visible cause — the user sees different output and has no way to diagnose why. The notification pattern gives the user the information and the choice without risking invisible rendering changes.

### Why exact version matching for pieces

A font version is not an aesthetic preference — it is a layout parameter. Bravura 1.392 and Bravura 1.400 may differ in stem widths, anchor point positions, bounding box extents, or engraving defaults. The backend computes layout from these values. If a client renders the resulting paintlist with a different font version, glyph positions will not match the computed layout. The misalignment may be subtle — a stem slightly too long, a beam at a slightly wrong angle — but it is real, and there is no way for the user to diagnose it.

The alternative — "best effort" rendering with a close-enough version — was rejected. It trades a clear, actionable error ("you need Bravura 1.392") for silently incorrect output. In a multi-user environment, it would mean different participants see different renderings of the same piece with no indication that anything is wrong. Exact matching is strict, but it is the only position consistent with the backend-authoritative rendering model.

### Why dual registration

Skija and JavaFX maintain separate font systems. Skija loads typefaces explicitly from data; JavaFX resolves fonts by family name from its internal registry. Both need access to the same font for different purposes. Registering both from the same source bytes in a single operation guarantees they are always in agreement.

### Why not automatic OS font installation

We considered having Ooloi install its bundled SMuFL fonts into the operating system's font directories so they would be available to other applications. This was rejected.

Ooloi does not need OS-installed fonts for its own rendering. The registry loads fonts from JAR resources as byte arrays — `Typeface/makeFromData` for Skija, `Font.loadFont` on an InputStream for JavaFX. Both are process-scoped and require no system installation. The bundled-first architecture is entirely self-contained.

Java provides no API for OS-level font installation. The entire operation is platform-specific: a file copy to `~/Library/Fonts/` on macOS (which auto-detects the new font), a file copy plus registry entry plus a `AddFontResource` Win32 call via JNA on Windows, and a file copy plus `fc-cache` on Linux. SMuFL metadata JSON must be written to a separate set of platform-specific directories defined by the SMuFL specification — there is no API for this either, only a filesystem convention. Three platform code paths, each with different post-copy requirements, maintained indefinitely.

The benefit — making bundled SMuFL fonts available to other applications — does not justify the cost. Users who want SMuFL fonts system-wide can install them through the normal OS mechanisms (Bravura ships with a `.pkg` installer for macOS; other fonts are drag-and-drop). Ooloi should not take on responsibility for fonts it places into the OS: version management across Ooloi upgrades, cleanup on uninstall, and potential conflicts with user-installed versions of the same font all become Ooloi's problem for no gain to Ooloi's own rendering.

---

## Consequences

**Positive**

- Rendering is deterministic: the same font version is used for layout computation and drawing, across backend and frontend, regardless of what the user has installed locally.
- SMuFL glyphs appear in both the score and UI controls from the same font, with no image assets or special rendering paths.
- Users can choose alternative SMuFL fonts through the font management UI. The aesthetic choice is theirs; the rendering invariant is maintained by the architecture.
- Local font discovery respects the user's agency — newer versions and alternative fonts are surfaced, not ignored — without introducing silent rendering changes. Dismissed notifications reappear until the user makes an explicit choice; resolved choices are permanent.
- In multi-user environments, exact font version matching guarantees all participants see identical rendering. There is no class of "works on my machine" font drift.
- The font registry is a clean Integrant component with no global state; it can be started and stopped in tests.

**Negative**

- Upgrading bundled fonts requires an Ooloi release. Users who want a newer version of a bundled font before the next release must use the local font path and explicitly opt in.
- The local font scan adds startup time proportional to the number of fonts installed on the system. The scan should be bounded (check known directories, match known family names or metadata files, do not parse every installed font).
- JavaFX's `Font.loadFont` registers fonts globally for the JVM lifetime. If multiple font versions of the same family are loaded (bundled and user-supplied), JavaFX's family name resolution may be ambiguous. The registry must manage this — either by loading only the active version, or by qualifying family names.
- Exact version matching means a client cannot open a piece created with a font version it does not have. This is intentional friction: the user is told exactly what is needed and can acquire it through the font management UI. The alternative — silent misrendering — is worse.

---

## References

### Related ADRs

- [ADR-0006: SMuFL](0006-SMuFL.md) — adopts SMuFL as the sole font standard; this ADR implements the management layer
- [ADR-0048: SMuFL Glyph Selection Architecture](0048-SMuFL-Glyph-Selection-Architecture.md) — class-driven glyph selection mechanism built on the spec-level metadata this ADR loads
- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) — frontend rendering framework; Skija draws the score, JavaFX draws the UI
- [ADR-0003: Plugins](0003-Plugins.md) — plugin system that may extend SMuFL-based notation with specialised symbols
- [ADR-0016: Settings](0016-Settings.md) / [ADR-0043: Frontend Settings](0043-Frontend-Settings.md) — active font selection stored in application settings
- [ADR-0017: System Architecture](0017-System-Architecture.md) — Integrant component lifecycle
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) — consumer of font metrics for spatial calculations
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) — notification system for local font discovery alerts
- [ADR-0038: Backend-Authoritative Rendering and Terminal Frontend Execution](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md) — rendering invariant: backend layout and frontend drawing must agree on the active font

### External References

- [SMuFL Specification](https://www.w3.org/2021/03/smufl14/index.html) — Standard Music Font Layout, W3C Community Group
- [Bravura](https://github.com/steinbergmedia/bravura) — reference SMuFL font, canonical fallback for Ooloi
- [SMuFL metadata specification](https://www.w3.org/2021/03/smufl14/specification/font-specific-metadata.html) — JSON metadata format for glyph bounding boxes, anchors, and engraving defaults