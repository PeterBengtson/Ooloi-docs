# ADR: Implementation of Real-Time Collaboration in Ooloi

## Status

Accepted

## Context

Real-time collaboration on musical scores is a highly requested feature that is currently not well-supported by existing music notation software. Musicians, composers, and educators need the ability to work simultaneously on the same piece, seeing each other's changes in real-time, similar to collaborative document editing platforms. Ooloi's architecture provides a unique opportunity to implement this feature effectively.

## Decision

We will implement real-time collaboration as a core feature of Ooloi, leveraging our existing architecture and technologies to provide a seamless, efficient, and reliable collaborative editing experience.

## Rationale

1. Existing Architecture Advantages:
   - Separation of frontend and backend allows for a cloud-hosted backend that can manage concurrent edits.
   - The Piece Manager can handle multiple concurrent connections to the same piece.

2. Transactional Model:
   - Ooloi's existing transactional system with automatic retries is well-suited for handling concurrent edits and resolving conflicts.

3. Efficient Communication:
   - gRPC's binary protocol ensures low-latency communication between clients and the server, crucial for real-time collaboration.
   - Nippy's efficient serialization can be used for larger data transfers when saving and loading music files.

4. Immutable Data Structures:
   - Clojure's immutable data structures simplify the implementation of conflict resolution and change merging.

5. Market Differentiation:
   - Implementing this highly requested feature will significantly differentiate Ooloi from competitors.

6. Extensibility:
   - The collaboration system can be designed to work with Ooloi's plugin architecture, allowing for custom collaborative features.

## Consequences

### Positive

- Unique feature offering in the music notation software market.
- Enhanced user experience for collaborative composition and education.
- Leverages and validates Ooloi's core architectural decisions.
- Potential for new use cases and market opportunities (e.g., remote music education, distributed orchestras).

### Negative

- Increased complexity in backend logic to manage concurrent edits.
- Potential for increased server infrastructure costs.
- Need for additional security measures to manage access control and data privacy.

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
