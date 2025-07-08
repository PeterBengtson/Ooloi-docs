# ADR: Separation of Frontend and Backend into Distinct Clojure Applications

## Status

Accepted

## Context

Ooloi is a complex music notation software that requires both a rich user interface for score editing and manipulation, and powerful backend processing for musical logic, formatting, and data management. We need to decide on the overall architecture of the application, specifically whether to combine frontend and backend into a single Clojure application or separate them into distinct applications.

## Decision

We will implement Ooloi as two separate Clojure applications: one for the frontend and one for the backend.

## Rationale

1. Separation of Concerns: 
   - The frontend focuses on user interface and interaction.
   - The backend handles core musical logic, data processing, and persistence.

2. Technology Optimization:
   - Frontend can leverage JavaFX and Skija for high-performance GUI rendering.
   - Backend can focus on efficient data processing without GUI overhead.

3. Scalability:
   - Backend can be scaled independently of the frontend.
   - Allows for future cloud-based deployments if needed.

4. Development Workflow:
   - Frontend and backend teams can work independently.
   - Easier to test and debug each component in isolation.

5. Performance:
   - Reduced memory footprint for each component.
   - Ability to optimize each component for its specific tasks.

6. Flexibility:
   - Potential for multiple frontend clients (desktop, web, mobile) in the future.
   - Easier to replace or upgrade either component independently.

7. Concurrency:
   - Backend can handle complex, long-running tasks without affecting UI responsiveness.

8. Cross-platform Compatibility:
   - Easier to manage platform-specific issues (e.g., GUI on different OS) in the frontend while keeping the backend consistent.

## Consequences

### Positive

- Clear separation of responsibilities between UI and core logic.
- Improved maintainability and testability of each component.
- Potential for better performance and scalability.
- Flexibility for future expansions (e.g., web or mobile clients).

### Negative

- Increased complexity in application architecture.
- Need for inter-process communication (addressed by using gRPC).
- Potential for data synchronization issues between frontend and backend.
- More complex deployment process.

## Implementation Approach

1. Define clear API boundaries between frontend and backend.
2. Implement gRPC for efficient communication between the two applications.
3. Use shared data models (e.g., protobufs) for consistent data representation.
4. Implement a robust error handling and recovery system for inter-process communication.
5. Design a state management system in the frontend to handle backend updates efficiently.
6. Create a development environment that supports running and debugging both applications simultaneously.

## Alternatives Considered

1. Single Clojure Application: Rejected due to concerns about mixing UI and core logic, potential performance issues with large scores, and limited flexibility for future expansion.

2. Web Frontend with Clojure Backend: Rejected for the initial implementation due to performance requirements for complex score rendering, but may be considered for future web-based versions.

## Notes

We will need to carefully monitor the performance and developer experience of this separated architecture. If inter-process communication becomes a bottleneck or significantly complicates development, we may need to reevaluate this decision in the future.
