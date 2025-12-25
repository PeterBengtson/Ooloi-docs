# ADR: Adoption of SMuFL (Standard Music Font Layout) for Music Notation

## Status

Accepted

## Context

Ooloi requires a standardized and comprehensive system for rendering musical symbols and notation. The choice of font system impacts rendering quality, cross-platform consistency, interoperability with other music software, and the ability to support a wide range of musical notations. Traditionally, music notation software has often relied on proprietary or non-standardized font layouts, leading to inconsistencies and limitations in symbol support.

## Decision

We will adopt the Standard Music Font Layout (SMuFL) as the primary system for musical symbol rendering in Ooloi.

## Rationale

1. Standardization:
   - SMuFL provides a standardized way of organizing musical symbols in fonts, ensuring consistency across different SMuFL-compliant fonts.
   - Reduces the need for custom glyph mapping, simplifying development and maintenance.

2. Comprehensive Coverage:
   - SMuFL covers a wide range of musical symbols, from common notation to specialized and historical notations.
   - Ensures Ooloi can handle diverse notation needs without requiring custom symbol implementations.

3. Unicode Compliance:
   - SMuFL uses the Unicode Private Use Area, ensuring Unicode compliance while providing extensive symbol coverage.

4. Interoperability:
   - Improves compatibility with other SMuFL-compliant music notation software and fonts.
   - Facilitates easier data exchange and consistent rendering across different platforms and applications.

5. Future-Proofing:
   - As an evolving standard supported by major players in the music notation industry, SMuFL is likely to remain relevant and expand to cover new notation needs.

6. Font Flexibility:
   - Users can choose from various SMuFL-compliant fonts, allowing for stylistic customization without changing the underlying code.

7. Metadata Support:
   - SMuFL fonts come with extensive metadata, useful for precise positioning and scaling of musical elements.

8. Community and Ecosystem:
   - Growing ecosystem of tools, fonts, and resources around SMuFL.

9. Performance:
   - Standardized layout can lead to optimizations in rendering and caching.

10. Simplified Development:
    - Reduces the need for maintaining custom font mapping systems, allowing focus on other aspects of notation rendering.

## Consequences

### Positive

- Consistent and standardized musical symbol rendering across the application.
- Improved interoperability with other music notation software and standards.
- Reduced development time for symbol mapping and management.
- Flexibility for users to choose different SMuFL-compliant fonts.
- Future-proofed for new musical symbols and notations.
- Potential for improved performance in symbol rendering and layout.

### Negative

- Limited to SMuFL-compliant fonts, which may be fewer in number compared to traditional music fonts.
- Potential learning curve for developers not familiar with SMuFL specifications.
- May require adaptation of existing musical data or import/export functions to align with SMuFL standards.

## Implementation Approach

1. Select a default SMuFL-compliant font for Ooloi (e.g., Bravura, Petaluma).
2. Implement a SMuFL font loading and management system.
3. Develop a rendering engine that utilizes SMuFL font metadata for accurate symbol placement and scaling.
4. Create a mapping system between Ooloi's internal musical representations and SMuFL codepoints.
5. Implement support for loading and using alternative SMuFL fonts.
6. Develop utilities for accessing and using SMuFL metadata (e.g., glyph bounding boxes, anchor points).
7. Ensure the plugin system can interact with and extend SMuFL-based notation rendering.
8. Implement export functions that maintain SMuFL compliance for interoperability.
9. Create documentation for developers on working with SMuFL in Ooloi.
10. Develop a testing suite to ensure accurate and consistent SMuFL-based rendering across different fonts and platforms.

## Alternatives Considered

1. Custom Font System:
   - Rejected due to increased development overhead and lack of standardization.

2. Traditional Music Fonts:
   - Rejected due to limitations in symbol coverage and lack of standardization across fonts.

3. Multiple Font Standards Support:
   - Considered but deemed unnecessarily complex and potentially inconsistent.

## Related Decisions

- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) - Frontend rendering framework that will display SMuFL symbols
- [ADR-0003: Plugins](0003-Plugins.md) - Plugin system that may extend SMuFL-based notation with specialized symbols

## Notes

- Regularly review and update to the latest SMuFL specification as it evolves.
- Consider contributing to the SMuFL standard if Ooloi requires symbols not currently in the specification.
- Develop guidelines for plugin developers on how to properly use and extend SMuFL-based notation in their plugins.
- Implement a system for users to easily install and manage additional SMuFL-compliant fonts.
- Monitor the SMuFL ecosystem for new fonts and tools that could enhance Ooloi's capabilities.
- Consider developing tools to help convert non-SMuFL musical data to SMuFL-compliant formats.
- Ensure that the implementation allows for efficient rendering of complex scores with numerous SMuFL symbols.
- Investigate the possibility of creating a SMuFL-compliant font specific to Ooloi for any custom symbols or notations.
- Plan for internationalization, ensuring SMuFL implementation works well with various language scripts and text directions.
- Consider implementing a fallback system for cases where a required symbol might not be available in the chosen SMuFL font.
