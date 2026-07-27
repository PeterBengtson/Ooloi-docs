# ADR-0005: Selection of JavaFX and Skija for the Frontend GUI

## Status

Under implementation

## Context

Ooloi requires a high-performance, cross-platform graphical user interface (GUI) for music notation editing and display. The GUI needs to handle complex musical scores with potentially hundreds of staves and thousands of measures, while providing a responsive and intuitive user experience. Additionally, the chosen technology should integrate well with our Clojure-based backend, support efficient rendering of musical notation, and provide high-quality, consistent printing across all platforms. The ability to produce professional-grade printed output is a critical requirement for a music notation application.

## Decision

We will use JavaFX as the primary GUI framework for Ooloi, with Skija (Java bindings for Skia) for high-performance 2D graphics rendering and printing.

## Rationale

1. Native Performance:
   - JavaFX provides native-like performance across different platforms.
   - Skija, being a Java binding for Skia (used in Chrome and Android), offers high-performance 2D graphics capabilities.

2. Clojure Compatibility:
   - JavaFX can be easily used with Clojure, maintaining language consistency across the project.
   - Existing Clojure wrappers for JavaFX (e.g., cljfx) can simplify development.

3. Cross-platform Support:
   - JavaFX applications can run on Windows, macOS, and Linux with minimal platform-specific code. The authoritative list of `(os, arch)` pairs Ooloi actually ships bundles for is in [ADR-0050: Platform Support Policy](0050-Platform-Support-Policy.md).

4. High-Quality, Consistent Printing:
   - Skija's rendering capabilities extend to printing, ensuring high-quality output.
   - Using the same rendering engine for display and printing guarantees consistency between screen and paper.
   - Cross-platform printing consistency is achievable as Skija handles the rendering independent of the operating system's print drivers.

5. Rich UI Components:
   - JavaFX provides a comprehensive set of UI controls and layouts out of the box.

6. Custom Rendering Capabilities:
   - Skija allows for custom, high-performance rendering of musical notation, crucial for both display and printing in Ooloi.

7. Scalability:
   - The combination of JavaFX and Skija can handle large, complex scores efficiently, both for on-screen display and printing.

8. Modern Architecture:
   - JavaFX's scene graph and property binding system align well with reactive programming models.

9. Community and Ecosystem:
   - Both JavaFX and Skija have active communities and ongoing development.

10. Familiarity:
    - The team has existing experience with Java-based technologies, reducing the learning curve.

11. Integration with Backend:
    - Using JVM-based technologies for both frontend and backend simplifies integration and data exchange.

## Consequences

### Positive

- High-performance rendering of complex musical scores on screen and in print.
- Consistent, high-quality printed output across all supported platforms.
- Consistent development experience with Clojure across frontend and backend.
- Cross-platform compatibility without significant additional effort.
- Flexibility to create custom UI components for specific music notation needs.
- Good integration with our backend architecture.
- Early implementation of printing functionality ensures it's a core, well-integrated feature.

### Negative

- Potential for larger application size due to bundling of JavaFX runtime.
- May require additional effort to achieve platform-specific look and feel if desired.
- Limited web deployment options compared to web-based technologies.
- Potential complexity in handling various paper sizes and printer-specific settings across platforms.

## Implementation Approach

1. Set up a basic JavaFX application structure using cljfx (Clojure JavaFX wrapper).
2. Integrate Skija for custom rendering of musical notation.
3. Implement core UI components using cljfx declarative syntax and JavaFX controls.
4. Develop custom JavaFX components for music-specific elements (e.g., staff, measure, note).
5. Use Skija for rendering detailed musical symbols and notation.
6. Implement reactive UI updates using cljfx's React-inspired declarative model.
7. Develop the printing subsystem early in the project:
   - Implement Skija-based rendering for print output.
   - Ensure consistency between screen display and printed output.
   - Develop a cross-platform print dialog and settings interface.
   - Implement support for various paper sizes and orientations.
8. Optimize rendering performance for large scores using techniques like virtualization.
9. Develop a theming system to allow for customizable appearances.
10. Implement platform-specific optimizations where necessary.
11. Conduct thorough testing of printing functionality across all supported platforms.

## Alternatives Considered

1. Electron (Web Technologies):
   - Rejected due to potential performance limitations with large scores and higher resource usage.
   - Concerns about achieving consistent, high-quality printing across platforms.

2. Qt (C++ with Clojure bindings):
   - Rejected due to more complex integration with Clojure and potential licensing concerns.
   - While Qt offers good printing support, integrating it with Clojure would be more challenging.

3. Java Swing:
   - Rejected in favor of the more modern JavaFX framework.
   - Printing in Swing can be more cumbersome and may not provide the same level of consistency across platforms.

4. Pure Web Technologies (ClojureScript):
   - Rejected due to performance concerns for complex score rendering and limitations in controlling print output precisely.
   - May be considered for future web-based versions, but not suitable for the core application where print quality is crucial.

## Related Decisions

- [ADR-0000: Clojure](0000-Clojure.md) - Language choice enabling JavaFX integration through excellent Java interop
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural decision requiring frontend GUI framework choice
- [ADR-0006: SMuFL](0006-SMuFL.md) - Musical symbol standard that will be rendered through JavaFX/Skija
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - JavaFX Application Thread integration with backend event streams using Platform.runLater() patterns
- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) - The UI built on these choices: window specification and lifecycle, the menu system, theming and styling — and §*macOS native integration*, the Cocoa mechanics implementing the native-parity position taken here

## Going Past the Toolkit: Native Platform Integration

A cross-platform toolkit is chosen for reach, and reach is bought with abstraction. Where that abstraction lands short of a platform's own convention, Ooloi's position is to **go to the platform layer rather than approximate**. Parity between supported platforms means parity of *nativeness*, not a uniform lowest common denominator: an Ooloi window on macOS should behave as a Mac application, not as a Java application running on a Mac.

This is the first rationale point above — "native-like performance across different platforms" — taken to its conclusion. Native-*like* is where a toolkit stops; a music notation application that a professional will live inside all day has to go further, and the cost of doing so is bounded and known.

**Today this bites in exactly one place: the macOS application menu** — the bold app-name menu carrying About, Hide, Hide Others, Show All and Quit. JavaFX has no first-class support for it, and its absence is conspicuous to any Mac user. Windows and Linux need no equivalent intervention, their menu conventions being close enough to JavaFX's own model. Three libraries supply what JavaFX does not:

| Library | Version | Why it is here |
|---|---|---|
| **NSMenuFX** (`de.jangassen:nsmenufx`) | 3.1.0 | Replaces JavaFX's auto-generated application menu with a real one, converting JavaFX MenuItems into native `NSMenuItem`s. The de-facto standard for this problem — there is no maintained alternative. |
| **JFA** (`de.jangassen:jfa`) | 1.2.0 | Java Foundation Access: a JNA wrapper over AppKit (`NSApplication`, `NSMenu`, `NSMenuItem`, `NSWorkspace`), mapping Java calls to Objective-C message sends. |
| **JNA** (`net.java.dev.jna:jna`) | 5.7.0 | The native interop layer beneath JFA. Transitive. |

The mechanics — the conversion chain, the native selector each standard item uses, the known NSMenuFX defects and their workarounds, and the two non-obvious JFA interop rules — are specified in [ADR-0042](0042-UI-Specification-Format.md) §*macOS native integration*, beside the menu system they serve.

### The position on an unmaintained dependency

NSMenuFX has had no meaningful commit since February 2021. That is a real liability and it is recorded here rather than discovered later, together with the reason it is an acceptable one.

The realistic prospects, in rough order of likelihood:

1. **JavaFX gains first-class application-menu support** — unlikely. The request has sat in the OpenJFX tracker for years with no sign of work; macOS is a minority target for its maintainers and the application menu is a non-trivial native-integration project.
2. **NSMenuFX is actively maintained again** — also unlikely. The maintainer has moved on.
3. **Fork it** — feasible. The codebase is around a thousand lines. This trades an unmaintained external dependency for a small internal one, with the benefit of fixing defects at source rather than patching around them. The reasonable step if two or three more defects accumulate on the scale of the one already found.
4. **Bypass it entirely using JFA directly** — doable, and the cleanest long-term answer. The bridging infrastructure already exists, having been used to patch NSMenuFX from outside. A full bypass means reimplementing the callback trampoline that turns a native click into a JavaFX event handler, which is the bulk of what NSMenuFX provides.
5. **Move from JNA to Project Panama** — Java 22 stabilised the Foreign Function and Memory API. If option 4 is taken, building the replacement on Panama rather than JNA is the modern choice: JDK-native, no extra dependency, better performance. Independent of NSMenuFX itself, and relevant only when replacing it.

**The escape hatch is what bounds the risk.** Any NSMenuFX defect is patchable at the Cocoa level from outside the library, by sending Objective-C messages directly and by replacing an item's native target and action with a direct pair. This has already been done once, for a real defect. Ooloi is therefore not held hostage to the library's state of repair for any individual menu item, which is what turns an unmaintained dependency from a structural risk into a maintenance cost.

**The position, then:** stay on NSMenuFX while it works, patch specific defects at the Cocoa level as they arise, and escalate to option 3 or 4 only if the patch surface grows. This is where every serious JavaFX-on-macOS application stands; NSMenuFX is the de-facto standard precisely because no better option exists.

## Theme Implementation

**AtlantaFX Dark Theme**: Modern, futuristic dark theme providing sci-fi aesthetic for professional music notation interface

- **Version**: 2.1.0 (integrated in frontend/project.clj)
- **Theme Direction**: Dark themes with custom extensions for organic, futuristic appearance inspired by Butler's sci-fi aesthetic
- **Design Philosophy**: Modern, clean interface that feels both professional and forward-looking
- **Benefits**: 
  - Reduced eye strain during long composition sessions
  - Modern flat design language consistent across platforms
  - Extensive component styling with dark mode optimizations
  - Professional appearance suitable for commercial music software
- **Integration**: Direct JavaFX theme integration through AtlantaFX, minimal code changes required
- **Extensibility**: AtlantaFX provides foundation for custom styling and component extensions
- **Colour System**: AtlantaFX implements the GitHub Primer colour system with semantic (functional) tokens (`-color-fg-default`, `-color-accent-muted`, `-color-danger-emphasis`, etc.) that resolve to different concrete values in dark vs light mode. All frontend CSS must use these tokens exclusively — no hardcoded hex/rgb values. Scale tokens (`-color-base-N`) are raw palette values and must not be used.
- **Icons**: Ikonli Material Design 2 (`ikonli-javafx` + `ikonli-material2-pack` 12.4.0) provides font-based icons via `FontIcon` nodes. Icons are referenced by string literals and inherit colour from parent control severity classes.

## Copper Plate Aesthetic Principle

Skija stroke caps and joins are **ROUND by default** across the entire rendering system. Element-specific geometry may reduce cap radius but may not introduce geometric points unless explicitly required by house style.

This follows the copper plate engraving tradition (Henle, Bärenreiter, Peters). The burin entering and leaving a copper plate naturally rounds; this rounding is both physically necessary in engraving and aesthetically correct. Sharp terminations look fragile and mechanical; rounded ends look confident and crafted.

The default applies everywhere Skija draws curves, line endings, or corners:

- Slur and tie endpoints
- Hairpin tips
- Bracket corners (tuplet brackets, ottava brackets, system brackets)
- Rehearsal mark box corners
- Any line termination or corner Ooloi generates

Each element type has its own cap radius or corner radius setting. See [ADR-0013](0013-Slur-Formatting.md) for the slur-specific treatment.

## Notes

- Regular performance benchmarking should be conducted, especially for large score rendering and printing.
- Consider creating an abstraction layer over JavaFX to ease potential future GUI framework changes.
- Investigate options for reducing application size, such as custom JavaFX runtime builds.
- Stay updated with JavaFX and Skija developments to leverage new features and improvements.
- Develop comprehensive documentation for custom UI components and rendering techniques, including printing-specific details.
- Plan for potential future web deployment by designing components with possible web adaptation in mind.
- Establish a comprehensive test suite for printing functionality, covering various score complexities, paper sizes, and printer types.
- Consider partnering with printing experts or music publishers to validate the quality and consistency of printed output.
- Develop custom AtlantaFX theme extensions to achieve the desired sci-fi aesthetic while maintaining professional usability.
