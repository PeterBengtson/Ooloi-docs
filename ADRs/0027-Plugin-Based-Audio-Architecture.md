# ADR-0027: Plugin-Based Audio Architecture

## Table of Contents

- [Status](#status)
- [Context](#context)
  - [Historical Context: Igor Engraver's Approach (1996-2001)](#historical-context-igor-engravers-approach-1996-2001)
  - [Frontend-Backend Audio Separation](#frontend-backend-audio-separation)
  - [Technical Requirements](#technical-requirements)
- [Decision](#decision)
  - [Core Architecture](#core-architecture)
- [Rationale](#rationale)
  - [Frontend-Backend Separation](#frontend-backend-separation-1)
  - [Plugin Architecture Benefits](#plugin-architecture-benefits)
- [Challenges Addressed](#challenges-addressed)
  - [User Experience](#user-experience)
  - [Technical Implementation](#technical-implementation)
  - [Default Experience](#default-experience)
- [Integration Points](#integration-points)
  - [Frontend-Backend Separation (ADR-0001)](#frontend-backend-separation-adr-0001)
  - [Plugin Architecture (ADR-0003)](#plugin-architecture-adr-0003)
  - [gRPC Communication (ADR-0002)](#grpc-communication-adr-0002)
  - [Frontend Development (ADR-0005)](#frontend-development-adr-0005)
- [Alternative Approaches Considered](#alternative-approaches-considered)
  - [Built-in VST/AU Hosting](#built-in-vstau-hosting)
  - [Built-in MIDI Output](#built-in-midi-output)
  - [Hybrid Core/Plugin Audio](#hybrid-coreugin-audio)
  - [Custom Audio Engine](#custom-audio-engine)
- [Success Criteria](#success-criteria)
  - [Technical Metrics](#technical-metrics)
  - [Musical Quality](#musical-quality)
  - [User Experience](#user-experience-1)
- [Future Extensions](#future-extensions)
  - [Advanced Humanisation](#advanced-humanisation)
  - [Professional Workflow Integration](#professional-workflow-integration)
- [Conclusion](#conclusion)

## Status

Accepted - September 13, 2025

## Context

Ooloi requires a sophisticated audio architecture capable of handling advanced musical notation, microtonal compositions, and nuanced humanisation. The system's frontend-backend separation, where backends may run in cloud environments, requires that all audio processing occurs on frontend clients rather than server infrastructure.

### Historical Context: Igor Engraver's Approach (1996-2001)

Twenty-five years ago, Igor Engraver achieved sophisticated music notation playback by seizing complete control of MIDI units. The system employed pitch bend manipulation for microtonal accuracy, distributing notes across MIDI channels using register allocation algorithms borrowed from compiler technology. In chords containing microtonally altered notes, each note played on different channels with dynamic patch changes and no fixed instrument-to-channel relationships - a "DNA soup" of resource allocation that extracted capabilities beyond MIDI's nominal sixteen-channel limit.

Igor incorporated advanced humanisation including realistic instrumental limitations, orchestral section rhythmic hierarchies, and sophisticated performance characteristics such as four-millisecond temporal offsets between soloists and accompaniment. However, these innovations operated within MIDI's protocol constraints: limited channel allocation, 7-bit controller resolution, and complex device-specific compatibility requirements.

Ooloi's plugin-based architecture addresses the same musical intelligence requirements while eliminating the protocol constraints that required Igor's elaborate workarounds.

### Frontend-Backend Audio Separation

**Backend Responsibilities** (ADR-0001: Frontend-Backend Separation):
- Musical data management and storage
- Real-time collaboration coordination
- Paintlist generation and caching
- No audio processing or MIDI handling

**Frontend Client Responsibilities**:
- All audio input and output processing
- MIDI keyboard input handling and translation to musical commands
- Plugin hosting and audio generation
- Local playback coordination and synchronization

**Cloud Deployment Considerations**:
- Backend services may run on cloud infrastructure (AWS, etc.)
- Audio streaming from cloud to clients impractical for latency and bandwidth
- Local client audio processing provides responsive user experience
- MIDI input requires low-latency local processing for practical music entry

### Technical Requirements

**Contemporary Music Support**:
- Arbitrary microtonal deviations with precise control beyond MIDI's 16-channel limitations
- Complex dynamic instructions like "sffzp < f > pp" requiring multiple coordinated parameter changes
- Advanced articulation techniques for contemporary composition and professional sample library integration
- Collaborative microtonal editing with consistent playback across clients

**Professional and Educational Deployment**:
- High-quality orchestral playback for classroom demonstrations without server audio streaming
- Professional-grade mock-ups for composition work using industry-standard tools
- Multi-user synchronized playback with advanced humanisation in classroom environments
- Integration with existing professional music production workflows and sample libraries

**Architectural Consistency**: Following Ooloi's commitment to modern standards (SMuFL fonts, plugin architecture from ADR-0003, contemporary collaboration features), audio functionality should use current professional tools rather than accommodate legacy protocol limitations.

## Decision

The Ooloi core provides no audio or MIDI functionality whatsoever. All audio processing, MIDI input/output, and playback are delegated to frontend plugins. The frontend translates all input (MIDI, screen clicks, scripting, plugins) into API commands for the backend.

### Core Architecture

**Complete Frontend Audio Responsibility**: All audio processing occurs on frontend clients
- **Plugin architecture**: Frontend clients host all audio plugins (VST/AU, MIDI, synthesis, etc.)
- **Local processing**: Audio generation and processing handled locally for responsive performance
- **No server audio**: Backend provides no audio capabilities, suitable for cloud deployment
- **Client independence**: Each client manages its own audio setup and plugin configuration

**Backend Audio Separation**: Backend provides no audio or MIDI processing
- **Musical data only**: Backend handles musical structure, collaboration, and paintlist generation
- **Input agnostic**: Backend receives high-level API commands regardless of input source (MIDI, clicks, scripts, plugins)
- **No audio/MIDI data**: No audio or MIDI data transmitted to or processed by backend
- **Cloud compatibility**: Backend suitable for deployment on cloud infrastructure without audio/MIDI capabilities

**Input Processing**: All input handled locally by frontend
- **Frontend responsibility**: All input devices (MIDI keyboards, mouse, touchscreen) connect to frontend clients
- **Local interpretation**: Clients translate all input types to high-level musical commands locally
- **API communication**: Musical commands sent to backend via gRPC API calls regardless of input source
- **Collaborative updates**: All musical changes propagated to other clients through backend

**Plugin Audio Architecture**: Multiple audio strategies available through frontend plugins
- **VST/AU hosting plugins**: Professional sample library integration on client machines
- **MIDI output plugins**: MIDI device control and external synthesis from clients
- **Direct synthesis plugins**: Built-in audio generation without external dependencies
- **Network audio plugins**: OSC, networked audio, or custom protocols managed by clients

**Default Installation**: Bundled audio plugin for immediate frontend capability
- **Client-side installation**: Default audio plugin installed with frontend client
- **Local audio processing**: Immediate playback capability without backend dependencies
- **Transparent operation**: Users receive audio feedback from local processing
- **Professional upgrade**: Advanced plugins available for enhanced client-side audio

## Rationale

### Frontend-Backend Separation

**Cloud-Compatible Architecture**:
- **Backend deployment flexibility**: Backend services can run on cloud infrastructure (AWS, Azure, etc.) without audio hardware requirements
- **No audio streaming**: Eliminates bandwidth and latency issues associated with streaming audio from servers to clients
- **Local audio performance**: Frontend audio processing provides responsive, low-latency user experience
- **Resource efficiency**: Audio processing distributed across client machines rather than centralized on servers

**Complete Responsibility Separation**: 
- **Backend scope**: Musical data management, collaboration coordination, paintlist generation, and storage
- **Frontend scope**: All audio input/output, MIDI processing, plugin hosting, and playback coordination
- **Clear boundaries**: No overlap between data management (backend) and audio processing (frontend)
- **Independent scaling**: Backend and frontend audio capabilities can scale independently based on requirements

**Client Audio Independence**:
- **Individual configuration**: Each client manages its own audio setup, plugins, and device configuration
- **Personalized workflows**: Musicians can use preferred audio tools without affecting other collaborators
- **Local optimization**: Audio processing optimized for individual client hardware capabilities
- **Offline capability**: Clients can provide audio playback without backend connectivity for local work

### Plugin Architecture Benefits

**Multiple Audio Approaches**: Frontend plugin architecture enables diverse audio strategies:
- **VST/AU hosting**: Professional sample library integration on client machines
- **MIDI output**: Traditional synthesis and external device control from clients
- **Direct synthesis**: Built-in audio generation without external dependencies
- **Custom protocols**: OSC, networked audio, or proprietary approaches managed locally

**Professional Integration**:
- **Industry standards**: VST/AU plugins integrate with established professional audio workflows
- **Local sample libraries**: Direct access to professional orchestral libraries installed on client machines
- **Audio hardware**: Integration with professional audio interfaces and monitoring setups
- **Creative tools**: Synthesis and processing plugins expand beyond traditional notation playback

**Educational and Collaborative Benefits**:
- **Classroom deployment**: Each student client handles its own audio without server audio requirements
- **Synchronized playback**: Coordinated playback across clients using backend timing coordination
- **Individual monitoring**: Each collaborator can use different audio setups for personal monitoring
- **Resource scaling**: Audio processing distributed across participant machines rather than centralized

## Challenges Addressed

### User Experience

**Initial Setup Complexity**: Users require plugin installation for any audio output

**Mitigation**: Default plugin pre-installed during Ooloi installation provides immediate functionality
- **Transparent operation**: Default audio capability appears as built-in feature to users
- **Progressive enhancement**: Advanced plugins available for users with specific requirements
- **Clear documentation**: Installation guides for different audio workflow preferences

**Audio Troubleshooting**: Plugin-based audio moves debugging from core to plugin ecosystem

**Mitigation**: Plugin isolation means audio issues don't affect core notation functionality  
- **Diagnostic tools**: Core system provides plugin health monitoring and diagnostic information
- **Community support**: Plugin-specific troubleshooting handled by plugin developers and user community
- **Fallback options**: Multiple plugin alternatives available if specific plugins fail

### Technical Implementation

**Plugin Interface Complexity**: Designing audio interfaces that support diverse plugin approaches

**Mitigation**: Flexible plugin API accommodates VST/AU hosting, MIDI output, synthesis, and custom approaches
- **Reference implementations**: Canonical plugins demonstrate interface usage patterns
- **Developer documentation**: Comprehensive guides for audio plugin development
- **Community development**: Open plugin architecture encourages third-party innovation

**Performance Coordination**: Managing plugin audio performance without core audio dependencies

**Mitigation**: Plugin architecture handles audio threading and resource management independently
- **Monitoring capabilities**: Core system monitors plugin performance without direct audio involvement
- **Resource isolation**: Plugin failures contained without affecting core notation system
- **Scalability**: Multiple plugins can operate simultaneously without core resource conflicts

### Default Experience

**Out-of-Box Functionality**: Ensuring immediate audio capability without manual configuration

**Mitigation**: Bundled default plugin provides immediate demonstration capability
- **Quality baseline**: Default plugin offers reasonable audio quality for evaluation and basic use
- **Upgrade pathway**: Clear progression from default to professional-grade plugins
- **Educational support**: Default plugin suitable for classroom and demonstration scenarios

## Integration Points

### Frontend-Backend Separation (ADR-0001)

Plugin-based audio architecture reinforces clean frontend-backend separation:
- **Backend responsibilities**: Musical data storage, collaboration coordination, paintlist generation - no audio/MIDI processing
- **Frontend responsibilities**: All audio/MIDI input/output, plugin management, input translation, local playback
- **Cloud deployment**: Backend suitable for cloud infrastructure without audio/MIDI hardware dependencies
- **Network efficiency**: Only musical data and paintlists transmitted between frontend and backend, no audio/MIDI streaming

### Plugin Architecture (ADR-0003)

Frontend audio plugins integrate with established plugin system architecture:
- **Frontend plugin hosting**: Audio plugins managed by frontend clients through same system as functional plugins
- **Local plugin ecosystem**: Plugin installation and management handled on individual client machines
- **Commercial integration**: Audio plugins participate in plugin store while running locally on clients
- **Development consistency**: Same development patterns for audio and functional plugins, with audio plugins running on frontends

### gRPC Communication (ADR-0002)

Audio architecture supports efficient backend communication:
- **Musical command API**: All input types translated to high-level musical commands sent via gRPC
- **No audio/MIDI data**: gRPC communication handles musical structure changes, not raw audio or MIDI data
- **Playback coordination**: Synchronized playback across clients coordinated through backend timing commands
- **Collaboration efficiency**: Musical changes propagated to clients for local audio rendering rather than centralized audio processing

### Frontend Development (ADR-0005)

Frontend audio capabilities integrate with windowing infrastructure:
- **Local plugin management**: Audio plugins loaded and managed within frontend window system
- **Resource coordination**: Audio processing coordinated with frontend UI threading without backend involvement
- **Platform audio**: Native audio capabilities handled by frontend on each target platform
- **User interface**: Audio plugin configuration and management integrated with frontend user interface

## Alternative Approaches Considered

### Built-in VST/AU Hosting

**Approach**: Integrate VST/AU hosting directly into Ooloi core

**Rejection Reasons**:
- **Architectural complexity**: Audio threading and resource management complicate core system design
- **Platform dependencies**: Audio library requirements increase deployment complexity across platforms  
- **Maintenance burden**: Audio troubleshooting becomes core system responsibility
- **Feature constraints**: Single audio approach limits user workflow options

### Built-in MIDI Output

**Approach**: Include MIDI output capabilities in core system

**Rejection Reasons**:
- **Protocol constraints**: MIDI channel and controller limitations reduce musical expression possibilities
- **Implementation overhead**: MIDI device management and channel allocation add complexity
- **User workflow mismatch**: Many users prefer plugin-based audio production workflows
- **Compatibility requirements**: Device-specific MIDI implementations require ongoing maintenance

### Hybrid Core/Plugin Audio

**Approach**: Basic audio capabilities in core with plugin extensions for advanced features

**Rejection Reasons**:
- **Inconsistent user experience**: Different capability levels create confusing audio behavior
- **Development duplication**: Audio features require implementation in both core and plugin systems
- **Architectural compromise**: Partial audio support in core creates unclear separation of responsibilities
- **Maintenance complexity**: Audio bugs potentially affect both core stability and plugin ecosystem

### Custom Audio Engine

**Approach**: Develop proprietary audio synthesis and processing engine

**Rejection Reasons**:
- **Development scope**: Custom audio engine development diverts resources from notation system goals
- **User expectations**: Musicians expect integration with existing professional tools and libraries
- **Market fragmentation**: Proprietary audio format reduces interoperability with professional workflows
- **Ongoing maintenance**: Audio engine maintenance competes with core notation system development

All these alternatives converge on the same principle: playback belongs in plugins, not in the core.

## Success Criteria

### Technical Metrics

- **Latency**: Audio latency under 20ms for real-time input and playback
- **Stability**: Plugin hosting stable across popular VST/AU instruments
- **Resource efficiency**: Multiple plugin instances without performance degradation
- **Compatibility**: Support for major plugin formats across target platforms

### Musical Quality

- **Dynamic interpretation**: Complex dynamic markings rendered with appropriate nuance
- **Microtonal accuracy**: Arbitrary pitch deviations reproduced precisely
- **Orchestral realism**: High-quality orchestral audio output from notation input
- **Educational effectiveness**: Classroom demonstrations with clear, engaging audio quality

### User Experience

- **Setup simplicity**: Default configuration provides immediate high-quality playback
- **Professional integration**: Seamless workflow with existing sample libraries
- **Educational adoption**: Music educators successfully use system for classroom instruction
- **Collaborative effectiveness**: Multi-user synchronized playback works reliably in classroom environments

## Future Extensions

### Advanced Humanisation

- **Machine learning integration**: AI-based performance interpretation for more realistic playback
- **Style libraries**: Historically-informed performance practice for different musical periods
- **Conductor simulation**: Interpretation of tempo and phrasing markings with musical intelligence

### Professional Workflow Integration

- **Multi-track export**: Individual instrument track export for professional audio production integration
- **Mix coordination**: Professional mixing features for orchestral balance and spatial positioning  
- **Synchronization**: Time code synchronization with external audio/video systems

## Related ADRs

- [ADR-0003: Plugin Architecture](0003-Plugins.md) - Plugin system design establishing extensibility foundations
- [ADR-0041: OVID](0041-Ooloi-Virtual-Instrument-Definition-OVID.md) - Virtual instrument definition defining how playback plugins control sample libraries
- [ADR-0044: MIDI Input Library and Boundary Architecture](0044-MIDI-Input-Library-and-Boundary-Architecture.md) - How MIDI input is received at the frontend: `javax.sound.midi`, CoreMidi4J on macOS, and the hard input boundary

## Related Guides

- [MIDI in Ooloi](../guides/MIDI_IN_OOLOI.md) - MIDI input for note entry, the deliberate absence of MIDI output from the core, and extension points for plugins requiring MIDI capabilities

## Conclusion

Plugin-based audio architecture positions Ooloi as a forward-looking professional notation system capable of serving contemporary music, educational applications, and collaborative workflows. By completely separating audio processing to frontend clients through plugins, the architecture eliminates the constraints that plagued Igor Engraver's sophisticated MIDI approach while enabling cloud deployment and professional audio integration.

This architectural approach takes the core innovations behind Igor Engraver's advanced humanisation and microtonal capabilities - the musical intelligence, sophisticated performance characteristics, and understanding of orchestral behavior - and re-implements them for modern technology and collaborative contexts. Where Igor's "DNA soup" required workarounds within MIDI's 16-channel constraints, Ooloi's plugin architecture provides unlimited precision and parameter control. Where Igor's advanced humanisation was constrained by MIDI controller resolution, plugins offer direct access to sample library parameters and articulation systems.

The architectural decision reflects Ooloi's commitment to modern standards and professional capabilities, ensuring that musical notation serves musical requirements rather than being constrained by protocol limitations. Frontend clients handle all audio complexity through flexible plugin architecture, while the backend focuses purely on musical data management and collaboration coordination. The result preserves and extends Igor's pioneering approach to musical intelligence while eliminating the technical constraints that limited its full expression.
