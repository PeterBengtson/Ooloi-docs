# ADR-0006: Adoption of SMuFL (Standard Music Font Layout) for Music Notation

## Status

Accepted

## Context

Ooloi requires a standardised and comprehensive system for rendering musical symbols and notation. The choice of font system affects rendering quality, cross-platform consistency, interoperability with other music software, and the ability to support a wide range of musical notations. Traditionally, music notation software has relied on proprietary or non-standardised font layouts, leading to inconsistencies and limitations in symbol support.

## Decision

We will adopt the Standard Music Font Layout (SMuFL) as the primary system for musical symbol rendering in Ooloi. SMuFL organises musical symbols in fonts according to a published standard, using the Unicode Private Use Area, with accompanying JSON metadata describing glyph bounding boxes, anchor points, and engraving defaults.

Bravura is the canonical default and fallback font. Users may select alternative SMuFL-compliant fonts (November 2, Petaluma, Leipzig, and others) through the font management system described in [ADR-0047: Font Management](0047-Font-Management.md).

## Rationale

1. **Standardisation**: SMuFL provides a standardised way of organising musical symbols in fonts, ensuring consistency across different SMuFL-compliant fonts. Reduces the need for custom glyph mapping.

2. **Comprehensive Coverage**: SMuFL covers a wide range of musical symbols, from common notation to specialised and historical notations, without requiring custom symbol implementations.

3. **Unicode Compliance**: SMuFL uses the Unicode Private Use Area, ensuring Unicode compliance while providing extensive symbol coverage.

4. **Interoperability**: Improves compatibility with other SMuFL-compliant music notation software and fonts. Facilitates data exchange and consistent rendering across platforms.

5. **Metadata Support**: SMuFL fonts ship with extensive metadata — glyph bounding boxes, anchor points, engraving defaults — essential for precise positioning and scaling in the rendering pipeline.

6. **Font Flexibility**: Users can choose from various SMuFL-compliant fonts, allowing stylistic customisation without changing the underlying code. A piece rendered with one SMuFL font can be re-rendered with another; the rendering pipeline reads all metrics from the active font's metadata.

7. **Future-Proofing**: SMuFL is an evolving standard supported by the principal notation software vendors and is likely to expand to cover new notation needs.

8. **Ecosystem**: Growing ecosystem of fonts, tools, and resources. The standard is maintained as a W3C Community Group specification.

## Consequences

### Positive

- Consistent and standardised musical symbol rendering across the application.
- Improved interoperability with other music notation software and standards.
- Reduced development time for symbol mapping and management.
- Flexibility for users to choose different SMuFL-compliant fonts.
- Future-proofed for new musical symbols and notations.
- The rendering pipeline consumes font metrics from standardised metadata rather than hardcoded values, making font switching a data change rather than a code change.

### Negative

- Limited to SMuFL-compliant fonts, which are fewer than traditional music fonts (though the major professional-quality fonts are all SMuFL-compliant).
- Font-specific engraving defaults may vary, requiring the rendering pipeline to handle per-font metric differences gracefully.

## Alternatives Considered

1. **Custom Font System**: Rejected due to increased development overhead and lack of standardisation.

2. **Traditional Music Fonts**: Rejected due to limitations in symbol coverage, lack of standardised metadata, and inconsistent glyph mapping across fonts.

3. **Multiple Font Standards Support**: Considered but rejected as unnecessarily complex. SMuFL has sufficient industry adoption to serve as the sole standard.

## Related Decisions

- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) — frontend rendering framework that displays SMuFL symbols via both Skija (score rendering) and JavaFX (UI controls)
- [ADR-0003: Plugins](0003-Plugins.md) — plugin system that may extend SMuFL-based notation with specialised symbols
- [ADR-0028: Hierarchical Rendering Pipeline](0028-Hierarchical-Rendering-Pipeline.md) — consumer of SMuFL font metrics for spatial calculations
- [ADR-0047: Font Management](0047-Font-Management.md) — font registry component, bundled font strategy, local font discovery, dual Skija/JavaFX registration, piece font manifests, and active font selection

## External References

- [SMuFL Specification](https://www.w3.org/2021/03/smufl14/index.html) — Standard Music Font Layout, W3C Community Group
- [Bravura](https://github.com/steinbergmedia/bravura) — reference SMuFL font, canonical fallback for Ooloi
- [SMuFL metadata specification](https://www.w3.org/2021/03/smufl14/specification/font-specific-metadata.html) — JSON metadata format for glyph bounding boxes, anchors, and engraving defaults

## Notes

- Regularly review and update to the latest SMuFL specification as it evolves.
- Develop guidelines for plugin developers on how to properly use and extend SMuFL-based notation in their plugins.