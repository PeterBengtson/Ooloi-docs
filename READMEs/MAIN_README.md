# Ooloi: a Body Reimagined

<div align="center">
<img src="/img/main-readme-top.png" alt="Ooloi">
</div>

Ooloi is a modern, open-source music notation software designed to handle complex musical scores with ease. It is designed to be a flexible and powerful music notation software tool providing professional, high-quality results. The core functionality includes inputting music notation, formatting scores and their parts, and printing them. Additional features can be added as plugins, allowing for a modular and customizable user experience.

It leverages Clojure for both the backend and frontend, JavaFX with Skia for high-quality rendering, and gRPC for efficient server-client communication.

For more information, check out our website and blog on [https://ooloi.org](https://ooloi.org), and also [our wiki](https://github.com/PeterBengtson/Ooloi/wiki).

## Table of Contents

1. [Background and Name](#background-and-name)
    - [Nature and Evolution](#nature-and-evolution)
    - [Name and Symbolism](#name-and-symbolism)
2. [Performance & Architecture Advantages](#performance--architecture-advantages)
3. [License](#license)
    - [Core](#core)
    - [Plugins](#plugins)
4. [Contributions](#contributions)
5. [Code of Conduct](#code-of-conduct)
6. [Architecture](#architecture)
7. [Core Concepts Quick Reference](#core-concepts-quick-reference)
8. [Directory Structure](#directory-structure)
9. [Getting Started](#getting-started)

## Background and Name

Ooloi is a modern revival and evolution of Igor Engraver, a pioneering music notation software developed from 1996 to 2001 by Peter Bengtson and a team of 12 spending a total of $7.5 million USD, adjusted for inflation. 

Igor Engraver was built entirely in Common Lisp and was named after the composer Igor Stravinsky, with an additional playful nod to Igor, the assistant from the classic film "Young Frankenstein." The project was innovative for its time, offering advanced features for music notation and engraving. However, it ultimately succumbed to management-imposed feature creep, venture capital policies, and the broader economic impact of the 9/11 attacks, which halted mergers and acquisitions worldwide.

## Performance & Architecture Advantages

Ooloi addresses fundamental challenges that have plagued music notation software for decades through its functional programming architecture. While traditional systems struggle with performance bottlenecks, memory management complexity, and concurrency limitations, Ooloi's design eliminates entire classes of problems:

**Scalability Through Functional Design**: Traditional object-oriented notation software hits performance walls with large orchestral scores, requiring workarounds like reducing visible staves or using special views. Ooloi's immutable data structures and STM-based concurrency enable linear performance scaling, handling complex scores efficiently without architectural limitations.

**Natural Collaboration**: Most notation software struggles with collaborative editing due to mutable state synchronization challenges. Ooloi's functional architecture enables natural multi-user editing through immutable data structures and automatic conflict resolution, eliminating the complex operational transform algorithms required by traditional systems.

**Market Opportunity**: The discontinuation of Finale in 2024 after 35+ years of accumulated technical debt validates the need for modern architectural approaches. Finale's demise, attributed to unmaintainable legacy code and performance limitations, creates space for next-generation systems built on sound functional programming principles.

**Professional Quality**: Building on Igor Engraver's tradition of intuitive input and professional output, Ooloi combines the best lessons from the music notation community while solving core architectural problems that have limited existing systems.

### Nature and Evolution

**Igor Engraver**:
- **Technology**: Built in Common Lisp, leveraging the language's flexibility and power. Single-threaded.
- **Focus**: Aimed at providing comprehensive music notation and engraving capabilities.
- **Challenges**: Faced issues with feature creep and external pressures from venture capital, leading to an unsustainable development cycle.
- **Demise**: The project was discontinued due to a combination of internal and external factors, including the economic fallout from 9/11.

**Ooloi**:
- **Technology**: Utilizes modern technologies such as Clojure for the backend, Skia for rendering, and gRPC for efficient data querying. Massively multi-threaded.
- **Focus**: Emphasizes a modular, scalable architecture with a lean core that can be extended through plugins.
- **Approach**: Open-source core with optional paid plugins, ensuring a sustainable development model and community engagement.
- **Legacy**: Builds on the lessons learned from Igor Engraver, avoiding feature bloat and maintaining a clear focus on core functionalities.

Ooloi is a complete rewrite. There is no legacy code from Igor Engraver in Ooloi. However, many of the underlying ideas are similar: for instance, Igor Engraver was particularly noted for its extremely intuitive and fast input of music which paralleled how a musician works with pen and music paper. Twenty-five years later, we still haven't seen anything on the market that compares to it. The same ideas and goals pervade Ooloi.

### Name and Symbolism

The pre-open-source name "FrankenScore" was a deliberate reference to Igor Engraver, encapsulating both a tribute to its predecessor and a fresh start. It evokes the idea of assembling a powerful, flexible system from various components, much like Dr. Frankenstein's creation. It also subtly acknowledges the playful spirit of the original project's association with "Young Frankenstein" - namely that everyone should have an assistant called Igor.

This new iteration aims to avoid the pitfalls that led to the downfall of Igor Engraver, focusing on a sustainable, community-driven approach to create a robust and extensible music notation software.

By naming the project Ooloi as it goes public and open-source, we signal a new direction that leverages modern technologies and development practices.

<div align="center">
<img src="/img/main-readme-ooloi.png" alt="Ooloi" width="500">
</div>

## License

### Core

Ooloi is licensed under the Mozilla Public License (MPL) 2.0. This means that any modifications to the core code must be shared under the same license. However, plugins developed using the interfaces and APIs provided by Ooloi can be proprietary. See the [LICENSE](LICENSE.md) file for more details.

### Plugins

Plugins for Ooloi can be closed-source and proprietary. Developers are free to create and distribute plugins under their own licensing terms. They can create commercial plugins and sell them without any obligation to open-source the plugin code. We are planning a CRM interface via Cryptolens, so plugin developers have full control of installations through flexible license keys.

## Contributions

We welcome contributions to Ooloi from the community! By contributing to to the core of Ooloi, you agree that your contributions will be licensed under the MPL 2.0. Please see our [CONTRIBUTING.md](CONTRIBUTING.md) file for more details on how to contribute.

## Code of Conduct

We are committed to providing a friendly, safe, and welcoming environment for all. Please read and follow our [Code of Conduct](CODE_OF_CONDUCT.md).

## Architecture

1. **Server-Client Architecture**
   - Backend service handles computations, formatting, and data management
   - Frontend service manages UI and rendering
   - Decoupled design allows for local operation or potential SaaS deployment
   - Utilizes multithreading and multiprocessing for efficiency

2. **Backend (Clojure with JavaFX)**
   - Leverages JVM for performance and interoperability
   - JavaFX for cross-platform server dashboards and debugging tools
   - Implements core logic, data structures, and computational tasks
   - Utilizes Clojure's functional programming paradigms and immutability
   - Handles data persistence and plugin management
   - Leverages Methodical, Potemkin, and Specter for its internal architecture

3. **Frontend (Clojure with JavaFX and Skija)**
   - Native application built with Clojure, not browser-based
   - JavaFX for cross-platform windowing and UI components
   - Skija (Java binding for Skia) for high-performance 2D graphics rendering
   - Capable of handling large, complex scores (e.g., 100 staves or more, thousands of measures or more)

4. **Rendering System**
   - Skija for both on-screen rendering and high-quality print output
   - Implements virtual scrolling or windowing for efficient rendering of large scores
   - Uses caching and dirty region tracking for optimized updates

5. **Data Management**
   - Utilizes Clojure's persistent data structures for efficient updates and undo/redo functionality
   - Implements a transactional system for updates using Clojure's software transactional memory (STM)

6. **Concurrency and Parallelism**
   - Leverages Clojure's Software Transactional Memory (STM) for thread-safe, concurrent data operations
   - Utilizes refs and dosync blocks to ensure atomicity and consistency in updates
   - Implements automatic retry mechanism for handling conflicts in concurrent modifications
   - Achieves high-performance concurrent updates (2017 2,2 GHz 6-Core Intel Core i7 old Macbook Pro):
     - Transactional mode: Over 100,000 updates per second
   - Leverages Clojure's core.async for managing asynchronous operations and parallelization
   - Implements parallel processing for tasks like layout calculations, part extraction, and rendering of complex sections
   - Scales automatically to utilize multiple cores, enhancing performance on modern hardware
   - Provides a simple, intuitive API that hides the complexity of the underlying concurrent operations
   - Utilizes Specter for efficient data transformations, offering 2.5 to 3 times performance improvement over traditional methods

7. **Plugin Architecture**
   - Supports dynamic extensibility and modularity
   - Plugins can be developed in Clojure or other JVM-compatible languages
   - Enables runtime loading and management of plugins

8. **File I/O and Persistence**
   - Custom file format for efficient saving/loading of large scores
   - Uses Nippy for data serialization
   - Optional compression for reduced file sizes

9. **Printing**
   - Utilizes Skija to render print-quality output
   - Integrates with system print services via Java's printing API

10. **Cross-Platform Compatibility**
    - JavaFX ensures consistent behavior across Windows, MacOS, and Linux
    - Skija provides consistent rendering across platforms
    - Uses jpackage for creating native installers for different platforms

11. **Development and Build Tools**
    - Leiningen for dependency management and building
    - Implements automated testing and CI/CD pipelines

This architecture combines the power of Clojure for both backend and frontend development, leveraging native performance and high-quality rendering capabilities. It's designed to handle the complexities of large musical scores while maintaining flexibility through a plugin system and potential for both local and cloud deployments.

The system's concurrency model, powered by Clojure's STM and enhanced by Specter, provides a powerful yet user-friendly foundation. While the underlying mechanisms manage complex, high-performance concurrent operations, the exposed API remains simple and intuitive. This allows developers to harness the full power of the system without needing to manage the intricacies of thread safety, conflict resolution, or optimal data transformations manually. The result is a system that can handle tens of thousands of updates per second with automatic scaling across multiple cores, all while presenting a clean, easy-to-use interface that abstracts away the complexity.

## Core Concepts Quick Reference

| Concept | Description | Key Files | ADR |
|---------|-------------|-----------|-----|
| Vector Path Descriptors (VPDs) | Addressing mechanism for navigating the nested piece structure | [`ops/vpd.clj`](/shared/src/main/clojure/ooloi/shared/ops/vpd.clj), [`VPDs.md`](/guides/VPDs.md) | [ADR-0008](/ADRs/0008-VPDs.md) |
| Pure Tree Structure | Hierarchical representation of musical elements with ID-based references | [`models/musical/piece.clj`](/shared/src/main/clojure/ooloi/shared/models/musical/piece.clj) | [ADR-0010](/ADRs/0010-Pure-Trees.md) |
| **Shared Model Contracts** | Unified data models and interfaces across frontend/backend | [`shared/models/`](/shared/src/main/clojure/ooloi/shared/models/), [`interfaces.clj`](/shared/src/main/clojure/ooloi/shared/interfaces.clj) | [ADR-0023](/ADRs/0023-Shared-Model-Contracts.md) |
| Software Transactional Memory | Thread-safe concurrent operations using Clojure's STM | Used throughout with `ref` and `dosync` | [ADR-0004](/ADRs/0004-STM-for-concurrency.md) |
| Plugin Architecture | Extensible system supporting plugins in any JVM language | Core interfaces in backend | [ADR-0003](/ADRs/0003-Plugins.md) |
| Shared Structure | Handling elements that span across the musical piece tree | [`ops/attachment_resolver.clj`](/shared/src/main/clojure/ooloi/shared/ops/attachment-resolver.clj) | [ADR-0011](/ADRs/0011-Shared-Structure.md) |
| Timewalk/Traversal | Transducer-based traversal of piece structure with temporal coordination | [`ops/timewalk.clj`](/shared/src/main/clojure/ooloi/shared/ops/timewalk.clj) | [ADR-0014](/ADRs/0014-Timewalk.md) |
| SMuFL Fonts | Standard Music Font Layout for notation rendering | Font files in [`SMuFL-fonts/`](/SMuFL-fonts/) | [ADR-0006](/ADRs/0006-SMuFL.md) |

## Directory Structure

```
Ooloi/
├── backend/              ; Backend code
├── frontend/             ; Frontend code
├── shared/               ; Shared model contracts, gRPC layer, and combined app builder
├── skia/                 ; Skia integration code
├── .gitignore
├── docs/                 ; Documentation for the project as a whole
└── README.md             ; Project-wide README
```

## Getting Started

Ooloi is organized into three main components:

### 📁 **[Backend](/backend/)** - Music Notation Engine
The core server that handles musical data, calculations, and business logic.
- **Key Features**: gRPC server, piece management, real-time event streaming, server components
- **Installation**: See [backend/README.md](/backend/README.md) for detailed setup instructions
- **Quick start**: `cd backend && lein deps && lein run`

### 📁 **[Shared](/shared/)** - Model Contracts & gRPC Infrastructure  
Unified data models, interfaces, and Protocol Buffer layer for frontend-backend communication.
- **Key Features**: All core data models, multimethod interfaces, attachment system, timewalk operations, unified gRPC protocol
- **Architecture**: Supports selective frontend imports with backend-dependent modules
- **Documentation**: See [shared/README.md](/shared/README.md) for architecture details

### 📁 **[Frontend](/frontend/)** - User Interface Client
The client application providing the graphical interface for music notation.
- **Key Features**: gRPC client, UI manager, comprehensive configuration system, deployment modes
- **Installation**: See [frontend/README.md](/frontend/README.md) for detailed setup instructions
- **Documentation**: See [frontend/README.md](/frontend/README.md)

For technical architecture details and development roadmap, see:
- [Development Plan](/DEV_PLAN.md) - Complete project roadmap and milestones  
- [Ooloi Server Architectural Guide](/guides/OOLOI_SERVER_ARCHITECTURAL_GUIDE.md) - Comprehensive server architecture analysis and enterprise patterns
- [gRPC Communication and Flow Control Guide](/guides/GRPC_COMMUNICATION_AND_FLOW_CONTROL.md) - Practical communication patterns and collaborative scenarios
- [ADR-0024: gRPC Concurrency and Flow Control Architecture](/ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) - Technical decisions behind communication patterns
