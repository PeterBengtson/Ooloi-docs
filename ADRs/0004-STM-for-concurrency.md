# ADR: Implementation of Software Transactional Memory (STM) for Concurrency

Decision: Use Clojure's Software Transactional Memory (STM) for managing concurrent operations in Ooloi, rather than simple atoms.

Context:
- The system needs to handle complex, concurrent modifications to musical structures.
- Musical scores can be large and complex, potentially requiring multiple coordinated updates across different parts of the structure.
- Consistency and atomicity of operations are crucial for maintaining the integrity of musical data.
- The system aims to support high-performance concurrent updates (over 100,000 updates per second).
- Traditional music notation software faces significant concurrency challenges with mutable object graphs, leading to complex synchronization requirements and performance bottlenecks.

Rationale:
1. Coordinated Updates: STM allows for coordinated updates across multiple refs, which is essential for maintaining consistency in complex musical structures. Atoms, while useful for single-point updates, don't provide this level of coordination.

2. Automatic Retry Mechanism: STM provides an automatic retry mechanism for handling conflicts in concurrent modifications. This is superior to manual conflict resolution that would be necessary with atoms.

3. Atomicity: STM ensures that either all changes in a transaction are applied, or none are. This is crucial for operations that need to modify multiple parts of the musical structure atomically.

4. Composability: STM transactions are composable, allowing for the creation of larger atomic operations from smaller ones. This is particularly useful for complex musical operations that may involve multiple steps.

5. Scalability: STM is designed to scale well with multiple cores, automatically leveraging available processing power. This aligns with the system's goal of high-performance concurrent updates.

6. Deadlock Avoidance: STM helps avoid common concurrency pitfalls like deadlocks, which could be more challenging to manage with lower-level concurrency primitives.

7. Elimination of Traditional Threading Issues: Unlike traditional systems that require explicit locking mechanisms, STM eliminates entire classes of concurrency bugs while providing automatic conflict resolution for deterministic behavior.

**What actually distinguishes STM from atoms here is coordination, not throughput.** The scalability points above (5, and the 100,000-updates/second capability) describe what the engine *can* do, not why STM was chosen over an atom — atoms scale just as well for *uncoordinated* writes, and contention is not a concern here in any case (clients are few, writes comparatively rare, which is exactly the regime in which STM's guarantees come for free). STM earns its place wherever consistency must span more than a single write. Today that shows up in two ways: a piece is held in a *single* STM ref — the piece registry itself uses an atom — so coordination currently means composing several mutations within that one ref atomically ([ADR-0045 §"Why atom, not STM ref"](0045-Instrument-Library.md)); and the change-detection funnel already relies on STM's transaction semantics beyond mere atomicity — its coalescing gate is a ref, so it rolls back on retry, and emission is dispatched through an agent held to commit ([ADR-0052 §4](0052-Change-Detection-and-Event-Generation.md)) — neither of which an atom provides. The multi-ref case becomes *literal* only if layouts or musicians are ever promoted to nested refs of their own; choosing STM now keeps that a local change rather than a concurrency-model rewrite. Developer-facing patterns for all of this are in the [Advanced Concurrency Patterns guide](../guides/ADVANCED_CONCURRENCY_PATTERNS.md).

Consequences:
- Pros:
  - Thread-safe operations with automatic conflict resolution.
  - Simplified programming model for complex concurrent operations.
  - Efficient conflict resolution (100,000+ transactions/second tested on 2017 hardware).
  - Better consistency guarantees for complex musical structures.
  - Deterministic behavior regardless of concurrent access patterns.
  - Multi-client access becomes architecturally straightforward when needed.
  - Enables linear performance scaling with processor cores, addressing scalability limitations in existing notation software.
- Cons:
  - Potential performance overhead for simple operations compared to atoms.
  - Learning curve for developers not familiar with STM concepts.

## Related Decisions

- [ADR-0000: Clojure](0000-Clojure.md) - Language choice providing STM as a core concurrency primitive
- [ADR-0009: Multi-Client Collaboration Support](0009-Collaboration.md) - Multi-client features naturally enabled by STM's conflict resolution
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) - Undo/redo architecture leveraging STM for coordinated piece modifications
- [ADR-0025: Server Statistics Architecture](0025-Server-Statistics-Architecture.md) - STM-gRPC transaction monitoring and performance metrics
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - Single-authority model relying on STM for transactional piece modifications
- [ADR-0045: Instrument Library](0045-Instrument-Library.md) - §"Why atom, not STM ref": the per-resource choice — an atom for single-container state (the IL, the piece registry), an STM ref for coordinated piece content
- [ADR-0052: Change Detection and Event Generation](0052-Change-Detection-and-Event-Generation.md) - the change-detection funnel's coalescing gate and agent-deferred emission depend on STM transaction semantics (a ref that rolls back on retry; dispatch held to commit)
