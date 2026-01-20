# ADR-0040: Single-Authority State Model

## Status

Accepted

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Core Principle](#core-principle)
  - [Capability Symmetry, Authority Asymmetry](#capability-symmetry-authority-asymmetry)
  - [No Dual State](#no-dual-state)
  - [Operations as the Only Unit of Truth](#operations-as-the-only-unit-of-truth)
  - [Streaming Import](#streaming-import)
  - [Deployment Model](#deployment-model)
  - [Frontend Role Delimitation](#frontend-role-delimitation)
- [Rationale](#rationale)
  - [Why No Dual State](#why-no-dual-state)
  - [Why Operations Not Pieces](#why-operations-not-pieces)
  - [Why Granular Operations for Import](#why-granular-operations-for-import)
  - [Stability as Architectural Property](#stability-as-architectural-property)
- [Consequences](#consequences)
  - [Positive](#positive)
  - [Negative](#negative)
  - [Neutral](#neutral)
- [Alternatives Considered](#alternatives-considered)
- [References](#references)
  - [Related ADRs](#related-adrs)
- [Notes](#notes)

## Context

Ooloi has a three-tier architecture: a shared core containing the complete musical engine, wrapped by thin frontend and backend projects. The shared core includes musical types, VPD operations, STM coordination, timewalker, hash-consing, and all musical logic. Both frontend and backend have full access to this engine.

This creates a design question: given that the frontend possesses the complete engine, what role should it play in piece state management?

Several architectures are possible:

1. **Optimistic local construction**: Frontend builds pieces locally, syncs to backend
2. **Dual state with reconciliation**: Frontend maintains local copy, reconciles with server
3. **Zero-latency editing**: Frontend maintains parallel state, backend confirms asynchronously
4. **Single authority**: Frontend observes and operates on server state via API only

The choice affects stability, complexity, and failure modes.

Existing architectural foundations:

- **Frontend-Backend Separation** (ADR-0001): Establishes the boundary between UI and core logic
- **STM for Concurrency** (ADR-0004): Provides transactional foundation for all piece modifications
- **VPDs** (ADR-0008): Addressing system enabling operation targeting across network boundaries
- **Pure Trees** (ADR-0010): Immutable structure that operations modify
- **Collaborative Sessions** (ADR-0036): Remote connection architecture for peer-to-peer and institutional servers
- **Backend-Authoritative Rendering** (ADR-0038): Establishes backend authority for rendering decisions; backend produces abstract paintlists, frontend executes them

## Decision

We implement a **single-authority state model** where pieces exist only on the server and change only through accepted operations.

### Core Principle

**The piece lives on the server. The frontend is a window and an input device.**

A piece is not an object that moves between locations. It is the accumulated result of operations the server has accepted. This principle governs import, editing, collaboration, and recovery as a single unified model rather than four special cases.

### Capability Symmetry, Authority Asymmetry

The frontend and backend both wrap the same shared core. They possess identical capability but differ in authority.

**The frontend may understand. It may not decide.**

The shared core provides the frontend with:

- **Type literacy**: Understanding what the server sends
- **Local validation**: Reject invalid operations without roundtrip
- **Piece construction**: Create valid pieces using `api/create-piece` and all constructors
- **Full predicate access**: Test pieces using the same validation logic as backend
- **Timewalker**: Traverse and analyse local pieces with full temporal coordination
- **All shared utilities**: Complete access to musical operations, transformations, queries
- **Plugin execution**: Operating on server state via API

The shared core does not provide the frontend with:

- Piece authority (locally constructed pieces are valid data but not authoritative state)
- Wholesale piece transfer to server (no `set-piece`)
- Offline editing with later sync
- Optimistic updates awaiting confirmation

Note: While `set-piece` does not exist, component operations do. The API provides `set-measure`, `add-musician`, `set-pitch`, and similar operations for modifying authoritative state granularly. A locally constructed piece can have its components submitted through these operations — the constraint is granular submission, not prohibition of local construction.

The frontend additionally contains concrete rendering implementation (paintlist interpretation via Skija) that is not part of the shared core. The backend produces abstract paintlists; the frontend decides how to realise them.

### No Dual State

There is no zero-latency editing via local state.

The frontend does not hold authoritative state. It may construct pieces locally (for testing, validation, preparation), but these are data, not authority. The authoritative piece exists only on the server.

### Operations as the Only Unit of Truth

There is no `get-piece` or `set-piece` in the API. This is not a gap but a boundary.

**A piece is the accumulated result of accepted operations.**

This means:

- Import is operation streaming, not blob upload
- Editing is operation submission
- Collaboration is operation coordination
- Recovery is operation replay

All four are the same problem with the same solution: operations against authoritative state.

Note: This is stricter than event sourcing. There is no replay into alternative projections, no competing materialisations, no speculative timelines. Operations are not history to be reinterpreted — they are authority itself.

### Streaming Import

MusicXML import demonstrates the canonical pattern:

**Batch piece transfer** (not supported):

- Build complete local piece → transfer entire piece to server via `set-piece`
- Requires API that transfers authority wholesale
- Authority ambiguous during construction

**Local construction + component submission** (supported):

- Parse XML → build complete local piece → validate/analyse locally with timewalker
- Submit components through granular API operations (`add-musician`, `set-measure`, etc.)
- Local piece useful for validation before submission
- Authority clear: local is data, server is authoritative after operations accepted

**Direct streaming** (supported):

- Parse XML → emit API calls as elements encountered → server builds incrementally
- Peak memory: one piece (server-side) plus parser state
- Errors reported in context (measure 847 fails at measure 847)
- No local piece constructed

Both component submission and direct streaming are valid. The constraint is: pieces reach the server through operations, not through wholesale blob transfer. At 36μs per call, a 50,000-element piece builds via API in ~1.8 seconds.

### Deployment Model

Ooloi runs a local client/server on the same machine. Remote connections (peer-to-peer collaboration, institutional servers) are additive.

**Local backend**:

- Always available
- 36μs in-process roundtrip
- Primary authority for local editing
- Never goes away unless crashed (solution: restart)

**Remote connections** (when established):

- Peer-to-peer collaborative sessions
- Institutional shared servers
- Connection to remote authority for shared pieces

**If remote connection drops**:

- Operation reverts to local backend
- No automatic reconnection or queue replay
- Manual reconnection when desired
- No reconciliation logic — the user decides when to reconnect

**There is no offline mode** in the sense of "editing without any backend." The local backend is always present. Remote is optional. This eliminates distributed systems complexity (partition tolerance, eventual consistency, conflict resolution) for the local case, and handles remote disconnection through reversion rather than queueing.

### Frontend Role Delimitation

The frontend is explicitly:

- A typed observer of authoritative state
- A validator of proposed operations
- A terminal executor of backend rendering decisions (receives abstract paintlists, produces concrete pixels)
- A plugin execution host acting through the API
- Capable of constructing valid pieces for testing, validation, or preparation

The frontend is explicitly not:

- An authority on piece content
- A source of rendering decisions (paintlists come from backend)
- A holder of authoritative state

The abstraction boundary for rendering is the paintlist. The backend produces abstract paintlists specifying what to draw; the frontend interprets them via its chosen technology (currently Skija/JavaFX). A future web frontend could interpret the same paintlists via Canvas or WebGL — the backend would not change. This separation ensures backend rendering logic is testable independently of any concrete graphics implementation.

## Rationale

### Why No Dual State

1. **36μs API roundtrip is already imperceptible to humans** — optimising below this threshold is solving a non-problem
2. **Dual state requires reconciliation logic** — reconciliation logic can fail
3. **One piece, one location, one authority** eliminates sync bugs entirely
4. **Any design introducing parallel state to optimise below 36μs is strictly worse** because it introduces real failure modes to solve a non-problem

### Why Operations Not Pieces

If pieces are always the accumulated result of operations:

- Authority is never ambiguous
- No ID canonicalisation needed (server mints all IDs)
- No conflict semantics for whole-piece replacement
- No version skew risk from divergent shared cores
- Import and editing are the same code path

The absence of `set-piece` isn't a missing feature — it's a design decision. Pieces aren't transferable blobs; they're the server's accumulated state from operations it accepted. Component operations (`set-measure`, `add-musician`, etc.) provide all necessary granularity.

### Why Granular Operations for Import

Local construction before submission is valid and useful:

- Validate complete structure before any server operations
- Use timewalker to analyse and verify musical content
- Catch errors locally rather than mid-submission

The constraint is that submission happens through component operations, not wholesale piece transfer. This ensures:

- Each operation is individually validated
- Server maintains authority throughout
- Partial failures are localised and recoverable

### Stability as Architectural Property

Ooloi's stability derives from the removal of failure modes, not from testing intensity.

| Failure Mode | Elimination Mechanism |
|--------------|----------------------|
| State corruption | Immutable data structures |
| Memory leaks | Garbage collection |
| Race conditions | Software Transactional Memory |
| Deadlocks | Lock-free STM |
| Dangling pointers | Integer ID references (no pointers) |
| Cache staleness | Immutable caching |

These bug classes have zero probability because they are not representable states. This applies equally to backend and frontend because both execute the same shared core under the same constraints.

Because the frontend executes the same core under the same constraints, frontend stability is not a separate achievement — it is inherited.

The single-authority model extends this stability guarantee by eliminating reconciliation as a failure mode. There is no reconciliation logic because there is nothing to reconcile.

## Consequences

### Positive

- **Unified model**: Import, editing, collaboration, recovery are one code path
- **Stability inheritance**: All operations inherit the tested API stability (million operations, zero errors)
- **No reconciliation bugs**: Cannot occur because reconciliation does not exist
- **Bounded memory**: Streaming operations prevent piece duplication
- **Incremental validation**: Errors localised to their source
- **Architectural clarity**: Authority is never ambiguous
- **No distributed systems complexity**: Local deployment eliminates partition tolerance, eventual consistency, and conflict resolution concerns
- **Graceful degradation**: Remote disconnection reverts to fully functional local operation
- **Rendering portability**: Abstract paintlists allow different frontend implementations without backend changes

### Negative

- **Server dependency**: No editing without running backend (restart if crashed)
- **Import speed bound by API**: 50,000 elements ≈ 1.8 seconds (acceptable but not instant)
- **Authority requires API**: Plugins can construct and analyse locally, but authoritative changes require API operations
- **Manual reconnection**: Remote connection recovery is explicit user action, not automatic

### Neutral

- **Frontend sophistication**: Full engine available but authority-limited; developers must understand the distinction
- **Restart as recovery**: Local backend failure recovery is restart
- **Reversion as disconnection handling**: Remote drop reverts to local, user reconnects when ready

## Alternatives Considered

### 1. Wholesale Piece Transfer

Frontend builds pieces locally, submits complete piece to backend via `set-piece`.

**Rejected because**:

- Requires `set-piece` API that transfers authority wholesale
- Creates window where authority is ambiguous
- Conflict semantics needed if server state changed during construction
- ID canonicalisation required on commit

Note: Local construction followed by component submission is supported and useful. Only wholesale piece transfer is rejected.

### 2. Dual State with Reconciliation

Frontend maintains local piece copy, reconciles with server periodically or on conflict.

**Rejected because**:

- Reconciliation logic is a new failure mode
- Introduces distributed systems complexity (partition tolerance, eventual consistency)
- 36μs roundtrip makes the latency benefit negligible
- Solves a non-problem at the cost of real complexity

### 3. Zero-Latency Editing via Local State

Frontend maintains parallel piece state, backend confirms asynchronously.

**Rejected because**:

- Identical problems to dual state
- 36μs is already below human perception threshold
- Two complete pieces in memory
- Reconciliation required when confirmation differs from optimistic update

### 4. Operation Queueing for Offline Support

Queue operations locally when disconnected, replay on reconnection.

**Rejected because**:

- Local backend is always present — "offline" means "backend crashed"
- Queue introduces storage questions (where? persistence?)
- Replay introduces conflict questions (state changed during queue accumulation?)
- Simpler solution: restart crashed backend

## References

### Related ADRs

- **ADR-0001**: Frontend-Backend Separation — establishes the boundary this document sharpens
- **ADR-0004**: STM for Concurrency — provides the transactional foundation
- **ADR-0008**: VPDs — addressing system that enables operation targeting
- **ADR-0009**: Collaboration — collaborative editing features this model supports
- **ADR-0010**: Pure Trees — structure that operations modify
- **ADR-0018**: API-gRPC Interface Generation — the operation-based API this model mandates
- **ADR-0036**: Collaborative Sessions and Hybrid Transport — remote connection architecture
- **ADR-0038**: Backend-Authoritative Rendering — rendering authority; this document addresses state authority

## Notes

The summary formulation:

**Capability symmetry, authority asymmetry.**

The frontend possesses the engine to understand and reason about musical state, but it does not own state. The piece exists in one location (the server), changes through one mechanism (accepted operations), and renders through one pipeline (backend-authoritative paintlists). This is not a limitation but a guarantee: the failure modes of distributed state do not exist because distributed state does not exist.

Cloud-only deployment (no local backend) would require revisiting this ADR, but that is a separate architectural decision.