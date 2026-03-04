# Ooloi Guides

This directory contains guides for understanding Ooloi's core concepts and APIs. **These guides also serve as practical tutorials for Clojure programming concepts**, using musical examples to teach functional programming patterns, concurrency, and data transformation techniques.

## Start Here

**[A_GUIDED_PATH_FOR_NEWCOMERS.md](A_GUIDED_PATH_FOR_NEWCOMERS.md)** — If you are new to Ooloi, begin here. This guide organises the complete documentation corpus — ADRs, guides, blog posts, and READMEs — into a nine-phase reading path ordered by conceptual dependency. Each phase builds the vocabulary the next phase requires. Nothing should be opaque when you arrive at it.

The three gateway guides listed in that document — TIMEWALKING_GUIDE, POLYMORPHIC_API_GUIDE, and FRONTEND_ARCHITECTURE_GUIDE — are where earlier foundations converge into practical synthesis. They are not the entry point; they are the destination.

---

## What Each Guide Teaches

| Guide | Ooloi Concepts | Clojure Concepts |
|-------|----------------|------------------|
| **🔵 [FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)** | Window lifecycle, event architecture, rendering pipeline, settings, localisation, collaboration | Declarative UI specs, pub/sub events, JAT threading discipline, cljfx |
| **🟢 [MIDI_IN_OOLOI.md](MIDI_IN_OOLOI.md)** | MIDI input subsystem, Flow Mode integration, device management, event bus `:midi` category | `reify`, SPI pattern, `javax.sound.midi` |
| **🟢 [TIMEWALKING_GUIDE.md](TIMEWALKING_GUIDE.md)** | Temporal coordination, musical traversal | Transducers, lazy sequences, functional composition, threading macros |
| **🟢 [PIECE_PERSISTENCE_GUIDE.md](PIECE_PERSISTENCE_GUIDE.md)** | Save/load workflows, I/O backends | Agents, asynchronous operations, error handling patterns |
| **🟡 [VPDs.md](VPDs.md)** | Path-based navigation, addressing | Vector operations, `get-in`/`update-in` patterns |
| **🟡 [PIECE_MANAGER_GUIDE.md](PIECE_MANAGER_GUIDE.md)** | Piece lifecycle, reference management | STM (refs, dosync), concurrent state management |
| **🟠 [POLYMORPHIC_API_GUIDE.md](POLYMORPHIC_API_GUIDE.md)** | Type system, VPD vs object dispatch, musical API design | Multimethods, hierarchies, polymorphic dispatch, `derive`/`isa?` |
| **🟠 [GRPC_COMMUNICATION_AND_FLOW_CONTROL.md](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** | gRPC communication, collaborative editing patterns | Network programming, distributed state, async operations |
| **🔴 [GRPC_STREAMING_THREADING_GUIDE.md](GRPC_STREAMING_THREADING_GUIDE.md)** | Network transport streaming, backpressure patterns | gRPC threading constraints, HTTP/2 flow control, context propagation |
| **🔵 [OOLOI_SERVER_ARCHITECTURAL_GUIDE.md](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** | Server architecture, distributed systems, enterprise patterns | STM-gRPC integration, concurrent state management, functional architecture |
| **🔴 [ADVANCED_CONCURRENCY_PATTERNS.md](ADVANCED_CONCURRENCY_PATTERNS.md)** | Parallel processing, performance optimization | STM coordination, parallel algorithms, performance tuning |

---

## Available Guides

### Getting Started
- **[A_GUIDED_PATH_FOR_NEWCOMERS.md](A_GUIDED_PATH_FOR_NEWCOMERS.md)** - Nine-phase reading path through the complete documentation corpus, ordered by conceptual dependency

### Frontend Architecture
- **🔵 [FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)** - The central architectural gateway: window lifecycle, event system, rendering pipeline, settings, localisation, and collaboration (comprehensive)
- **🟢 [MIDI_IN_OOLOI.md](MIDI_IN_OOLOI.md)** - MIDI input for note entry, why the core produces no MIDI output, and how events flow to Flow Mode via the frontend event bus (concise)

### Core Architecture
- **🟢 [TIMEWALKING_GUIDE.md](TIMEWALKING_GUIDE.md)** - Temporal piece traversal (beginner-friendly)
- **🟢 [PIECE_PERSISTENCE_GUIDE.md](PIECE_PERSISTENCE_GUIDE.md)** - Asynchronous save/load operations (foundational)
- **🟡 [PIECE_MANAGER_GUIDE.md](PIECE_MANAGER_GUIDE.md)** - Storage and lifecycle management (important concepts)
- **🟠 [POLYMORPHIC_API_GUIDE.md](POLYMORPHIC_API_GUIDE.md)** - Type-driven design foundations (advanced intermediate)
- **🔴 [ADVANCED_CONCURRENCY_PATTERNS.md](ADVANCED_CONCURRENCY_PATTERNS.md)** - Parallel processing with STM (advanced/dangerous)

### Distributed Systems & Server Architecture
- **🟠 [GRPC_COMMUNICATION_AND_FLOW_CONTROL.md](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** - gRPC communication patterns and collaborative scenarios (distributed systems)
- **🔴 [GRPC_STREAMING_THREADING_GUIDE.md](GRPC_STREAMING_THREADING_GUIDE.md)** - gRPC streaming implementation with proper threading (advanced - common pitfall across all languages)
- **🔵 [OOLOI_SERVER_ARCHITECTURAL_GUIDE.md](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Enterprise-grade server architecture and design patterns (comprehensive reference)

### Reference Documentation
- **🟡 [VPDs.md](VPDs.md)** - Vector Path Descriptors reference (core system knowledge)

---

## Related Documentation

- **Architecture Decision Records (ADRs)**: See `/ADRs/` for design decisions
- **API Documentation**: Generated docs in `/backend/docs/`

## Walking the Talk

Quite apart from the Ooloi documentation aspect, the guides also represent my attempt to walk my talk
in this article:

- https://peterbengtson.medium.com/functional-programming-beyond-the-vampire-castle-b601c8faf1cf.

You may also be interested in this article:

- https://peterbengtson.medium.com/the-musical-journey-to-understanding-transducers-building-oolois-piece-walker-e2b015e76fe2.
