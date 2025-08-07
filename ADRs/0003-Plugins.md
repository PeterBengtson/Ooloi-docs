# ADR: Integration of Plugins as a Core Architectural Component

## Status

Accepted

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

### Hot Plugin Installation Architecture (Enabled by Universal gRPC)

**Revolutionary Zero-Downtime Plugin System**: Ooloi's universal Clojure-aware gRPC architecture enables unprecedented hot plugin installation capabilities:

**Plugin Installation Process**:
```bash
# Current Architecture Problem (Eliminated)
1. Plugin installs new models → hardcoded protobuf generation fails
2. Server restart required → minutes of downtime
3. Schema regeneration → complex build pipeline
4. All clients must reconnect → user disruption

# Universal Architecture Solution
1. Plugin installs: (defrecord CustomNotation [...])
2. Universal OoloiValue handles new types immediately
3. API methods discovered dynamically at runtime  
4. Perfect type fidelity preserved automatically
# Result: Zero downtime, seamless installation
```

**Technical Implementation**:
- **Dynamic Model Discovery**: Universal `OoloiValue` protobuf message handles any plugin data structure
- **Runtime API Registration**: New plugin API methods discovered via dynamic function resolution
- **Type Fidelity Preservation**: Ratios, keywords, custom types maintain semantics across network
- **No Schema Changes**: Static universal schema never needs regeneration

**Plugin Use Cases Enabled**:
- **Streaming Data**: MIDI, audio analysis, real-time collaboration data
- **Custom Notation**: Microtonal systems, extended techniques, cultural notation
- **Domain Extensions**: Analysis tools, educational features, composition AI
- **Drawing Operations**: Custom curves, graphics, dynamic visual elements

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
