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
