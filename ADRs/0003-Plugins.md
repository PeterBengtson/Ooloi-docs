# ADR-0003: Integration of Plugins as a Core Architectural Component

## Status

Accepted

## Table of Contents

- [Context](#context)
- [Decision](#decision)
- [Rationale](#rationale)
- [Consequences](#consequences)
  - [Positive](#positive)
  - [Negative](#negative)
- [Implementation Approach](#implementation-approach)
  - [Core Plugin Architecture](#core-plugin-architecture)
  - [Kinds of Plugin](#kinds-of-plugin)
  - [Undo: Frontend Plugins Get It Free, Backend Plugins Register It](#undo-frontend-plugins-get-it-free-backend-plugins-register-it)
  - [Hot Plugin Installation Architecture (Enabled by Unified gRPC)](#hot-plugin-installation-architecture-enabled-by-unified-grpc)
  - [Separation, Security, and Dynamic Loading](#separation-security-and-dynamic-loading)
  - [Standard Plugin Development Infrastructure](#standard-plugin-development-infrastructure)
- [Plugin System Architecture](#plugin-system-architecture)
- [Alternatives Considered](#alternatives-considered)
- [Plugin Configuration Architecture](#plugin-configuration-architecture)
  - [Backend Plugin Settings](#backend-plugin-settings)
  - [Frontend Plugin Settings](#frontend-plugin-settings)
  - [UI Responsibility](#ui-responsibility)
- [Notes](#notes)
- [Related Decisions](#related-decisions)

## Context

Ooloi is designed with a philosophy of a minimal, efficient core surrounded by a rich ecosystem of plugins. This approach aims to create a flexible and extensible music notation software that can cater to a wide range of user needs and specialized use cases. While the core application should provide essential functionality for music notation, we recognize that different users may have unique requirements that are best served through a plugin architecture. Additionally, we want to foster a community of developers who can contribute to and extend Ooloi's capabilities, while also allowing for commercial opportunities.

## Decision

We will implement a robust plugin system as a central architectural component of Ooloi, allowing for significant extension of functionality through user-created, third-party, or commercial plugins. The core application will be designed to be minimal and efficient, with plugins providing additional features such as sequencing, MIDI support, specialized notations (e.g., guitar tablature, jazz notation, drum set notation), and more. The plugin system will support multiple programming languages to maximize interoperability and developer flexibility.

## Rationale

1. Core Design Philosophy:
   - A minimal core ensures stability, performance, and ease of maintenance.
   - Plugins allow the application to be highly customizable without bloating the core.

2. Extensibility:
   - Plugins enable Ooloi to cater to niche use cases and specific user needs.
   - Specialized features (e.g., tablature, jazz notation) can be added without affecting core users.

3. Commercial Opportunities:
   - Allowing closed-source and commercial plugins creates incentives for high-quality, specialized development.
   - Potential for a marketplace ecosystem, benefiting both developers and users.

4. Community Engagement:
   - Encourages the development of a vibrant ecosystem around Ooloi.
   - Allows third-party developers to contribute to the platform and potentially profit from their work.

5. Modularity:
   - Promotes a modular architecture, improving maintainability of the core application.
   - Allows for easier testing and isolation of features.

6. Customization:
   - Users can tailor Ooloi to their specific workflows and requirements.

7. Rapid Feature Development:
   - New features can be developed and released as plugins without waiting for core application release cycles.

8. Performance Optimization:
   - Users can choose to load only the plugins they need, potentially improving performance.

9. Separation of Concerns:
   - Keeps the core application focused on fundamental music notation tasks.

10. Future-Proofing:
    - As music notation evolves, new notations or techniques can be added via plugins without major core changes.

11. Interoperability:
    - Leveraging the JVM allows plugins to be written in multiple programming languages (e.g., Java, Kotlin, Scala, Clojure).
    - Developers can use their preferred language and tools, lowering the barrier to entry for plugin creation.
    - Enables integration with existing libraries and frameworks from various JVM languages.

## Consequences

### Positive

- Highly flexible and customizable platform for users.
- Potential for a rich ecosystem of free, open-source, and commercial plugins.
- Improved maintainability of the core application.
- Easier to cater to specialized or niche requirements.
- Potential for community-driven growth and feature development.
- Commercial opportunities for developers, potentially driving innovation.
- Broad developer base due to multi-language support for plugins.
- **Zero-downtime plugin installation** enabled by universal gRPC architecture.
- **Hot plugin deployment** without server restarts or schema regeneration.
- **Infinite extensibility** for plugin data types and API methods.

### Negative

- Increased complexity in application architecture.
- Need for careful API design to ensure stability and backwards compatibility across multiple languages.
- Potential for conflicts between plugins or with core functionality.
- Security considerations for running third-party code, especially closed-source.
- Performance overhead of plugin management system.
- Possible fragmentation of user experience if core features are neglected in favor of plugins.
- Additional complexity in managing plugins written in different languages.

## Implementation Approach

### Core Plugin Architecture

1. Design a robust and well-documented plugin API that can be easily used from multiple JVM languages.
2. Implement a plugin manager to handle loading, enabling, disabling, and updating plugins, regardless of their implementation language.
3. Define clear boundaries between core functionality and plugin-extensible areas.
4. Develop a sandboxing mechanism to ensure plugins can't compromise system security or stability.
5. Create a standardized way for plugins to integrate with the UI, add menu items, and extend existing functionality.
6. Implement version checking to ensure compatibility between plugins and the core application.

**Plugins are not all of one kind, and the difference is architectural.** Most of the examples
above extend the *score*, and reach it through the polymorphic API — an import format builds and
traverses piece structure with the same operations any other caller uses
([ADR-0030](0030-MusicXML.md)). Others extend the *application* rather than the score: pure Clojure
loaded into the backend process, reached by direct call, using no API and carrying nothing across a
wire. A destination for the failure record
([ADR-0017](0017-System-Architecture.md) §Surfacing Unexpected Runtime Failures) would be of that
second kind — it would attach to one existing function and touch no piece data at all. Which kind a
plugin is determines what governs it: the first is bound by the API and by the piece it operates
on, the second only by the process it is loaded into.

### Kinds of Plugin

Two accepted distinctions decide what a plugin can do: **which side it runs on**, and **whether it
is written in Clojure or reaches Ooloi through the JVM API**.

| | **Clojure** | **Any other JVM language** |
|---|---|---|
| **Backend** | Runs in the backend process, with no wire between it and the score, and is free of the restriction to use the API only: it may call the polymorphic API in either form — VPD or object — or any other function or method in the codebase. It is indistinguishable from core code. | Reaches Ooloi through the API, via a Java-facing interface. Being in the same JVM as the score, both signature forms are open to it — the VPD form and the object form. |
| **Frontend** | Works exactly as its backend counterpart, and is free of the same restriction: it may call any function or method in the frontend codebase and present whatever GUI it likes. `SRV/` is its route to the backend. | Reaches Ooloi through the API. `SRV/` is its only route to the backend. |

**The frontend is not the UI side.** A file converter is naturally a frontend plugin: importing
MusicXML means reading a file on the user's machine, so the plugin reads it there and then issues
`SRV/` calls that build the piece on the backend in one transaction. Nothing about it is a matter of
presenting UI, and either language serves: the Clojure one has the wider freedom, the other is
restricted to `SRV/` for the crossing, and both do the same job. Which side a plugin belongs on
follows as much from where the data it needs lives as from what the plugin does.

**The two axes govern different things.** The language axis decides how far a plugin reaches into
the process it is loaded into — not how fast it runs. The Java-facing interface is an ordinary JVM
call and every JVM language is compiled and optimised alike, so a plugin written in Kotlin or Scala
executes at the same speed as one written in Clojure, on either side.

The side axis decides how a plugin talks to the score. On the backend the score is in the same
process; on the frontend it is not, and `SRV/` is the route to it. That does not make a frontend
plugin the lesser thing. The frontend holds the same data model and the same `api/` operations as
the backend — the same records, with perfect fidelity — so a plugin may build and transform real
piece structure there, freely, and compute as heavily as it likes while doing so. What it may not do
is *decide*: those local mutations are provisional, and only what crosses through `SRV/` becomes
musical truth ([ADR-0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)).
The distinction is authority, not capability. That crossing is also where undo is captured, so a
frontend plugin's edits become undoable
steps with no work on its part, while a backend plugin reaches the score below the boundary and
registers its own
(§[Undo](#undo-frontend-plugins-get-it-free-backend-plugins-register-it)). Being written in Clojure
buys a frontend plugin nothing with respect to the piece — object pointers cannot cross a network,
so VPD-only is a property of the wire rather than of the language.

**How a plugin is engaged is a separate question from either axis, and is specified where each
contract lives.** A plugin taking part in notation formatting declares hooks at the pipeline stages
it participates in, and is registered against the musical elements it formats
([ADR-0028](0028-Hierarchical-Rendering-Pipeline.md)), which admits no architectural distinction
between core and extended notation, chords and beams being plugins themselves. A plugin reached by
menu item, click or keyboard shortcut contributes command descriptors and is rendered alongside core
commands ([ADR-0042](0042-UI-Specification-Format.md) §Command Descriptors). Participation is a
contract rather than a location: either may be written in Clojure or in any other JVM language, on
either side.

### Undo: Frontend Plugins Get It Free, Backend Plugins Register It

**A plugin runs on one side of the frontend/backend boundary or the other, and the two are
different things** — a distinction this ADR already draws for configuration, where a backend
plugin stores its settings as piece settings and a frontend plugin keeps a local settings file
(§[Plugin Configuration Architecture](#plugin-configuration-architecture)). Undo is the second
place the distinction bites, and there the two sides are opposite.

**A frontend plugin gets undo for free.** The frontend is terminal: a piece exists only on the
backend ([ADR-0040](0040-Single-Authority-State-Model.md)), so a frontend plugin can reach it
only by sending an operation to the backend. Every such operation crosses the gRPC boundary,
which is exactly where Ooloi captures undo automatically
([ADR-0015](0015-Undo-and-Redo.md)) — so the plugin's edits become undoable steps with no work
on its part and no awareness that undo exists. This is the same path a user's own gesture takes,
which is why the two are indistinguishable once they arrive.

**A backend plugin must register its own.** It runs inside the backend process and reaches the
score through the polymorphic API directly, which puts its mutations *below* that boundary, so
nothing captures them on its behalf. The confinement is deliberate rather than an oversight: the
formatting pipeline also writes to a piece through the API, and a boundary that captured every
such write would put an incremental reflow on the undo stack.

Registration is one call. The plugin takes the piece value before its work and after it, and
calls `push-undo!` on the Backend Undo Manager with a description key of its own and closures
over those two snapshots — the shape the Instrument Library uses for its own edits. Persistent
data structures make holding both snapshots cheap. Because the plugin operation was invoked by a
client, it runs inside that request and the entry acquires the caller's identity automatically,
so it behaves like every other entry in a collaborative session: another client can undo it, and
the Edit menu labels it with the plugin's own description.

Two cases need nothing on either side. A plugin that only reads a piece — analysis, export,
playback — records no step; and one that *creates* a piece rather than editing an open one
records none either, an imported score being a document arriving, closed rather than undone,
exactly as with New and Open.

**A plugin's entries are labelled without its participation, and a key it invents is its own to
translate.** The capture boundary derives a label from the operation it invoked — `:undo.<op>`,
declared and translated for every operation the API exposes — so a frontend plugin's lone call
reads as a named step in the reader's own language with nothing done on its part, and a batch sent
without a name takes the generic `:undo.atomic`. A plugin that supplies its own `:undo-key` on an
`SRV/atomic` batch, which is the right thing to do because a batch is one gesture and deserves one
name, thereby leaves the set of keys Ooloi declares. Nothing resolves that key on the plugin's
behalf: `tr` renders the conspicuous `[MISSING: …]` placeholder for a key with no entry
([ADR-0039](0039-Localisation-Architecture.md) §Forward Compatibility), so the plugin ships the key
in its own catalogues, in each language it supports, as it would any other user-facing string. The
same applies to the description key a backend plugin hands to `push-undo!`.

### Hot Plugin Installation Architecture (Enabled by Unified gRPC)

**Zero-Downtime Plugin System**: Ooloi's unified Clojure-aware gRPC architecture enables hot plugin installation capabilities:

**Plugin Installation Process**:
```bash
# Current Architecture Problem (Eliminated)
1. Plugin installs new models → hardcoded protobuf generation fails
2. Server restart required → minutes of downtime
3. Schema regeneration → complex build pipeline
4. All clients must reconnect → user disruption

# Unified Architecture Solution
1. Plugin installs: (defrecord CustomNotation [...])
2. Unified OoloiValue handles new types immediately
3. API methods discovered dynamically at runtime  
4. Perfect type fidelity preserved automatically
# Result: Zero downtime, seamless installation
```

**Technical Implementation**:
- **Dynamic Model Discovery**: Unified `OoloiValue` protobuf message handles any plugin data structure
- **Runtime API Registration**: New plugin API methods discovered via dynamic function resolution
- **Type Fidelity Preservation**: Ratios, keywords, custom types maintain semantics across network
- **No Schema Changes**: Static unified schema never needs regeneration

#### Namespace Naming Requirement for Plugin Defrecords

**A Clojure plugin's namespaces must be named with hyphens, never with literal underscores.**

A defrecord crosses the wire as its Java class name, and Clojure *munges* a hyphen in a namespace
name into an underscore in the compiled package and file name: the namespace
`my.plugin.custom-notation` produces the class `my.plugin.custom_notation.CustomNotation`.
Reconstruction reverses that. `resolve-defrecord-constructor`, in
`shared/src/main/clojure/ooloi/shared/grpc/clojure_conversion.clj`, demunges the package segment
back into a namespace name before requiring it and resolving `map->CustomNotation`.

Munging is not invertible. Given only `my.plugin.custom_notation`, nothing distinguishes a
hyphenated namespace from one containing a literal underscore, so the demunge necessarily assumes
the hyphen. A plugin that names its namespaces with underscores will have its records arrive as
plain maps instead of records — and *silently*, because an unresolvable constructor falls back to
the map form by design. That fallback is deliberate and valuable: it lets data from an unknown
plugin travel intact rather than failing. It also means a naming mistake degrades quietly rather
than announcing itself, which is why the requirement is stated here rather than left to be
discovered.

**Scope: this applies to Clojure defrecords only.** The `:defrecord-val` wire path is entered
solely for values satisfying `record?`. A plugin written in Java, Kotlin, Scala or any other JVM
language sends its data as maps and collections, which round-trip with perfect fidelity and
without any namespace resolution whatsoever. Record *reconstruction* is a Clojure-defrecord
facility; the rest of the plugin interface remains language-agnostic as described above.

**Plugin Use Cases Enabled**:
- **Streaming Data**: MIDI, audio analysis, real-time collaboration data
- **Custom Notation**: Microtonal systems, extended techniques, cultural notation
- **Domain Extensions**: Analysis tools, educational features, composition AI
- **Drawing Operations**: Custom curves, graphics, dynamic visual elements

### Separation, Security, and Dynamic Loading

Ooloi loads plugin code into a running process. That is what makes hot installation possible, and it
is also the whole of the security problem.

**What the JVM offers for loading.** Code enters a running JVM through a ClassLoader, which delegates
to its parent for classes it does not itself define, so loaded code can be scoped rather than merged
into one flat namespace — two plugins may carry different versions of the same library without
either seeing the other's. There is no explicit unload: code goes away when its ClassLoader becomes
unreachable and is collected. Discovery has a standard mechanism in `ServiceLoader`, which finds
implementations of an interface written in any JVM language without the host knowing their names in
advance. Clojure adds runtime namespace loading and by-name var resolution on top. Which of these
Ooloi uses, and how, belongs with the plugin design work rather than here; what is settled is that
the wire imposes no obstacle — a unified `OoloiValue` message and by-name method dispatch mean a
plugin needs no schema regeneration and no restart ([ADR-0002](0002-gRPC.md), and
§[Hot Plugin Installation Architecture](#hot-plugin-installation-architecture-enabled-by-unified-grpc)
above).

**What the JVM no longer offers for isolation.** A separate ClassLoader separates *names* and
*lifetimes*; it grants no permissions and withholds none. The facility that once withheld them is
gone — the SecurityManager was deprecated by JEP 411 and permanently disabled by JEP 486, so on a
current JVM it cannot be installed at all, and with it went every in-process policy over filesystem,
network and reflection. Two further capabilities are absent for related reasons: a thread cannot be
forcibly stopped, so a plugin that will not return cannot be made to; and the heap is accounted
process-wide with no per-ClassLoader quota, so a plugin's memory use can be neither attributed nor
capped. What the JVM does still enforce is strong encapsulation of its own internals — reflective
access into the JDK is denied by default, and Ooloi opens nothing.

**What remains is the process.** Real confinement of untrusted code is a process-level matter, and
therefore the operating system's rather than the JVM's. Ooloi's frontend/backend separation is a
process boundary, but it is drawn for authority over musical truth rather than for containment: it
governs what a plugin can do to a score, not what it can do to the machine it runs on. How plugins
are to be confined is forthcoming.

### Standard Plugin Development Infrastructure

7. Develop guidelines and documentation for plugin developers, including best practices for both open-source and commercial plugins, with examples in multiple languages.
8. Create a system for managing plugin dependencies that works across different JVM languages.
9. Implement a mechanism for plugins to store and retrieve their own configuration data.
10. Develop a testing framework for plugins to ensure quality and compatibility, supporting multiple languages.
11. Create a plugin marketplace or repository for sharing, discovering, and purchasing plugins.
12. Implement licensing and validation mechanisms for commercial plugins.
13. Develop a clear policy on plugin licensing, including guidelines for commercial plugins.
14. Create language-specific wrappers or SDKs to simplify plugin development in popular JVM languages.

## Plugin System Architecture

1. Plugin Interface:
   - Define a clear interface that all plugins must implement, with bindings for multiple JVM languages.
   - Include methods for initialization, shutdown, and version information.

2. Extension Points:
   - Identify key areas in the application where plugins can extend functionality.
   - Examples: custom note heads, new layout algorithms, additional file format support, MIDI integration, specialized notation systems.

3. Event System:
   - Implement a robust event system that plugins can hook into, accessible from all supported languages.
   - Allow plugins to register for and respond to application events.

4. Resource Management:
   - Provide mechanisms for plugins to load and manage their own resources (e.g., images, fonts).

5. Configuration:
   - Allow plugins to define their own configuration options.
   - Integrate plugin configurations into the main application settings.

6. Lifecycle Management:
   - Implement clear lifecycle hooks for plugin initialization, activation, deactivation, and uninstallation.

7. Licensing System:
   - Develop a flexible licensing system that can accommodate both free and commercial plugins.
   - Implement secure validation for commercial plugin licenses.

8. Language Interoperability:
   - Develop a common interface layer that allows seamless integration of plugins written in different JVM languages.
   - Provide language-specific wrappers to simplify plugin development in each supported language.

## Alternatives Considered

1. Monolithic Application:
   - Rejected due to lack of flexibility and potential for bloat.

2. Scripting Language Integration:
   - Considered but deemed insufficient for complex extensions.
   - May be implemented alongside the plugin system for simpler customizations.

3. Microservices Architecture:
   - Rejected as overly complex for a desktop application, though some concepts may be applied to the plugin system.

4. Open-Source Only Plugins:
   - Rejected as it would limit commercial opportunities and potentially reduce the quality and variety of available plugins.

5. Single-Language Plugin System:
   - Rejected in favor of multi-language support to maximize developer engagement and leverage existing JVM ecosystem.

## Plugin Configuration Architecture

### Backend Plugin Settings

Backend plugins store configuration as piece settings (ADR-0016):
- Settings declared using `defsetting` with `:plugin/` namespace
- Settings travel with piece data (collaboration, version control, undo)
- No separate plugin settings files for backend plugins
- Example: `(defsetting ::h/Piece :plugin/musicxml/format-version "4.0" #{"3.0" "3.1" "4.0"})`

> **How a plugin's settings UI is generated is UNDECIDED — this is a requirement, not yet a design.** The need is real and specific: a plugin's `defsetting` declarations live in *backend plugin code*, so unlike the core piece settings — which sit in `shared/` and are therefore on the frontend's classpath already ([ADR-0053](0053-Piece-Window-and-Piece-Preferences.md) §6) — a plugin's declarations are not available to the frontend locally. Something must carry them across.
>
> **The hard part is not transport but content**, and it is what makes this a requirement rather than a design. Whatever carries a declaration across must carry enough to build a control from, and a `defsetting` declaration does not uniformly contain that: a **set** validator enumerates its legal values, from which a control type and a precise error message both follow, while an **arbitrary predicate** yields neither — `pos-int?` says nothing about whether to render a spinner or a text field, nor what to tell the user when they type something else. Any scheme that ships declarations must therefore either constrain what a plugin may declare or carry presentation alongside the constraint.
>
> Alternatives not yet weighed: plugins shipping their declarations in shared code so the frontend reads them locally as it does the core ones; a plugin supplying its own window spec, which ADR-0042's backend-described dialog capability would already transport; or a settings-description call, which would have to answer the content question above before it could be specified.
>
> Values are a settled matter either way, and only the *declarations* are at issue: a plugin setting's value reaches the frontend in the structural projection like any other, since `:settings` is a structural slot ([ADR-0052](0052-Change-Detection-and-Event-Generation.md) §3a).

**Benefits:**
- Settings persist with pieces (sharing, version control)
- Automatic undo/redo support
- Collaboration-friendly (settings shared across clients)
- Unified Settings window (core + plugin settings together)

### Frontend Plugin Settings

Frontend plugins use local settings files:
- Stored in `~/.ooloi/frontend/plugins/{plugin-id}/settings.edn`
- Simple EDN maps for UI preferences (not musical decisions)
- File-based persistence, not part of piece data
- Example: `{:toolbar-position :left :icon-size :large}`

**Rationale:**
- UI preferences are client-specific, not piece-specific
- No need to serialize/share UI customizations
- Simpler implementation for local-only settings

### UI Responsibility

Frontend provides all UI surfaces:
- **Core operations**: Key signatures, time signatures (frontend windows)
- **Backend operations**: Import/export parameters (frontend windows)
- **Settings window**: Auto-generated from metadata

Backend plugins expose:
- Settings schema (via `defsetting`)
- API operations (via polymorphic API)
- No UI descriptions

**Future Extensibility:**
Custom backend-described windows may be added if plugins require UI beyond settings and standard operations. Initial implementation focuses on settings-based configuration which covers the majority of plugin needs.

## Notes

- Regularly review and update the plugin API to ensure it meets developer needs across all supported languages.
- Consider implementing a plugin certification process to ensure quality and security, especially for commercial plugins.
- Monitor performance impacts of plugins and provide tools for users to identify problematic plugins, regardless of their implementation language.
- Develop clear guidelines on what types of functionality should be plugins vs. core features.
- Consider implementing a telemetry system (opt-in) to understand plugin usage and performance.
- Plan for internationalization support in plugins from the outset.
- Ensure the plugin system is designed with future web or mobile versions of Ooloi in mind.
- Develop clear policies and guidelines for commercial plugin development and distribution.
- Regularly assess the balance between core features and plugin-provided functionality to ensure the base application remains robust and useful.
- Provide comprehensive documentation and examples for plugin development in each supported language.
- Consider hosting workshops or webinars to encourage plugin development in various JVM languages.

## Related Decisions

- [ADR-0000: Clojure](0000-Clojure.md) - Language choice providing JVM compatibility for multi-language plugin support
- [ADR-0006: SMuFL](0006-SMuFL.md) - Musical notation standard that plugins might extend with specialized symbols
- [ADR-0016: Settings](0016-Settings.md) - Universal entity settings architecture used for backend plugin configuration
- [ADR-0027: Plugin-Based Audio Architecture](0027-Plugin-Based-Audio-Architecture.md) - Audio/MIDI processing as plugins rather than core features
- [ADR-0041: OVID](0041-Ooloi-Virtual-Instrument-Definition-OVID.md) - Virtual instrument definition as plugin resources
- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) - Format for plugins to define UI (windows, preferences) without frontend dependencies
