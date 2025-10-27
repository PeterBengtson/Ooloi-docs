# ADR-0009: Multi-Client Collaboration Support

## Status

Accepted

## Context

Ooloi's architecture—built around immutability, STM transactions, and client-server separation—makes multi-user collaboration architecturally straightforward to implement. While the primary use case remains single-user desktop notation (99.99% of scenarios), specialized scenarios benefit from multi-client access:

- **Classroom environments**: Teacher and students viewing/editing scores simultaneously
- **Remote instruction**: Teacher guiding student work in real-time
- **Ensemble preparation**: Multiple musicians reviewing parts together with synchronized playback

Traditional notation software struggles with collaboration due to mutable state synchronization challenges, requiring complex operational transform algorithms. Ooloi's immutable data structures and STM transactions eliminate these complexities—collaboration becomes a natural consequence of the architecture rather than a bolted-on feature.

## Decision

We will implement multi-client collaboration support in Ooloi by extending our existing architecture with session management, real-time synchronization, and collaborative UI features. This leverages the architectural foundation already in place (immutability, STM, gRPC streaming) to provide reliable multi-user editing when needed, without compromising the primary single-user desktop experience.

## Rationale

1. **Architectural Foundation Already Exists**:
   - Client-server separation designed for separation of concerns naturally supports multiple clients
   - Piece Manager already handles concurrent access through STM—adding clients requires no fundamental changes
   - Server location (local vs. remote) is transparent to the architecture

2. **Immutability Eliminates Collaboration Complexity**:
   - No complex operational transform algorithms needed
   - STM transactions provide automatic conflict resolution with ACID guarantees
   - Immutable data structures make state synchronization straightforward

3. **Efficient Communication Infrastructure**:
   - gRPC streaming already implemented for client synchronization
   - Binary protocol ensures low-latency updates between clients
   - VPDs provide efficient change communication

4. **Natural Extension, Not Retrofit**:
   - Collaboration leverages existing architectural decisions made for correctness and determinism
   - No architectural compromises required—the foundation enables it naturally
   - Adding collaboration features doesn't complicate single-user scenarios

5. **Specialized Use Case Support**:
   - Enables classroom teaching scenarios without requiring dedicated "education mode"
   - Supports remote instruction naturally through same architecture
   - Occasional multi-user needs addressed without architectural overhead

6. **Plugin Ecosystem Benefits**:
   - Plugins work identically in single-user and multi-user contexts
   - No special "collaboration-aware" plugin APIs required

## Consequences

### Positive

- **Architectural validation**: Multi-client support demonstrates correctness of immutability and STM design decisions
- **Educational scenarios**: Enables classroom and remote instruction use cases naturally
- **Zero architectural tax**: Collaboration features don't compromise single-user performance or complexity
- **Simplified implementation**: No operational transform complexity—STM handles conflict resolution automatically
- **Market differentiation**: Reliable multi-user editing in scenarios where competitors struggle

### Negative

- **Session management complexity**: Requires user presence tracking, access control, and invitation mechanisms
- **UI complexity**: Collaborative indicators, remote cursors, and presence display add frontend complexity
- **Infrastructure considerations**: Multi-user scenarios may require remote server hosting (though local remains default)
- **Security requirements**: Access control and data privacy considerations for shared sessions

## Implementation Approach

1. Extend Piece Manager:
   - Implement session management for collaborators.

2. Real-Time Synchronization:
   - Use gRPC streaming for real-time updates between clients and server.
   - Implement a publish-subscribe system for broadcasting changes to all connected clients.

3. Conflict Resolution:
   - Extend the transactional system to handle and resolve conflicts between concurrent edits.
   - Implement Operational Transformation or a similar algorithm to ensure consistency across clients.

4. Change Representation:
   - Use VPDs to efficiently communicate changes between clients and server.
   - Implement a change compression mechanism for bandwidth optimization.

5. Collaborative UI:
   - Develop UI components to show collaborator presence and activities.
   - Implement visual indicators for remote cursor positions and selections.

6. Access Control:
   - Develop a permission system for different levels of access (view, edit, admin).
   - Implement secure invitation and joining mechanisms for collaborative sessions.

7. Offline Support:
   - Implement a change queuing system for offline edits.
   - Develop a robust synchronization mechanism for when clients come back online.

8. Version History:
   - Leverage the undo/redo system to provide a browsable history of changes.
   - Implement features to revert to or branch from previous states of the score.

9. Large Data Handling:
   - Implement intelligent chunking for transferring large scores or changes.

10. Plugin API:
    - Extend the plugin API to allow integration with the collaboration system.
    - Provide hooks for plugins to participate in conflict resolution and change representation.

11. Testing and Validation:
    - Develop a comprehensive test suite simulating various collaborative scenarios.
    - Implement stress testing for concurrent edits on large scores.

## Alternatives Considered

1. Peer-to-Peer Collaboration:
   - Rejected due to complexities in NAT traversal and maintaining consistency across all peers.

2. WebSocket-based Solution:
   - Considered but rejected in favor of gRPC for its better performance and type safety.

3. Third-party Collaboration Platforms:
   - Explored but deemed insufficient for the specialized needs of music notation collaboration.

## Notes

- Regularly benchmark the collaboration system's performance, especially with large scores and many concurrent users.
- Implement analytics to understand collaboration patterns and optimize accordingly.
- Consider implementing "roles" within collaborative sessions (e.g., conductor, section leader).
- Explore integration with version control systems for advanced collaboration workflows.
- Develop clear documentation and tutorials for effective collaborative use of Ooloi.
- Investigate potential for using the collaboration system for live performance applications.
- Consider implementing a "replay" feature to review the collaborative creation process.
- Explore options for audio/video chat integration to enhance remote collaboration.
- Regularly gather user feedback on the collaboration experience and iterate on the implementation.
- Stay informed about advancements in distributed systems and real-time collaboration technologies.

## Related Decisions

- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Separation of concerns architecture that naturally accommodates multiple clients
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Communication protocol providing efficient client-server streaming
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model designed for correctness that naturally enables multi-client coordination
- [ADR-0008: VPDs](0008-VPDs.md) - Addressing system enabling precise change communication across clients
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) - Undo/redo architecture supporting multi-client scenarios
