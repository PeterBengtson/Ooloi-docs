# ADR-0022: Lazy Frontend-Backend Architecture

## Status

Accepted 2025-08-06

## Table of Contents

- [Context](#context)
  - [Architectural Vision and Historical Foundation](#architectural-vision-and-historical-foundation)
  - [Collaborative Scaling Challenge](#collaborative-scaling-challenge)
  - [Data Flow and User Interaction Requirements](#data-flow-and-user-interaction-requirements)
- [Decision](#decision)
- [Core Architecture](#core-architecture)
  - [Communication Patterns](#communication-patterns)
  - [Foundational Principles](#foundational-principles)
  - [Performance Architecture](#performance-architecture)
- [User Interaction Architecture](#user-interaction-architecture)
  - [VPD-Based Frontend Object Hierarchy](#vpd-based-frontend-object-hierarchy)
  - [Hit-Testing and Click Resolution](#hit-testing-and-click-resolution)
  - [Clean/Dirty State Integration](#cleandirty-state-integration)
- [Windowing System Integration](#windowing-system-integration)
  - [Viewport Management](#viewport-management)
  - [Event Processing Pipeline](#event-processing-pipeline)
  - [UI Component Architecture](#ui-component-architecture)
- [Collaborative Architecture](#collaborative-architecture)
  - [Natural Multi-Client Support](#natural-multi-client-support)
  - [Event Subscription Management](#event-subscription-management)
  - [Collaborative Update Handling](#collaborative-update-handling)
- [Platform Independence](#platform-independence)
  - [MVC Internalization](#mvc-internalization)
  - [Frontend Technology Flexibility](#frontend-technology-flexibility)
- [Rationale](#rationale)
  - [Lazy vs Eager Computation](#lazy-vs-eager-computation)
  - [API Operations vs Event Notifications](#api-operations-vs-event-notifications)
  - [Backend Authority vs Distributed Computation](#backend-authority-vs-distributed-computation)
- [Consequences](#consequences)
  - [Positive Outcomes](#positive-outcomes)
  - [Trade-offs and Risks](#trade-offs-and-risks)
  - [Mitigation Strategies](#mitigation-strategies)
- [Example Scenarios](#example-scenarios)
  - [Typical Piece Opening and Navigation Workflow](#typical-piece-opening-and-navigation-workflow)
  - [User Adds Articulation](#user-adds-articulation)
  - [Collaborative Editing Session](#collaborative-editing-session)
- [Related ADRs](#related-adrs)

## Context

Ooloi's frontend-backend separation ([ADR-0001](0001-Frontend-Backend-Separation.md)) requires explicit data synchronization patterns between any frontend implementation and the Clojure backend. The system must handle real-time collaborative editing, large orchestral scores (30-40 staves), responsive user interactions, and complex UI requirements while maintaining clean architectural boundaries.

### Architectural Vision and Historical Foundation

This lazy event-driven architecture builds on patterns proven in [Igor Engraver](https://en.wikipedia.org/wiki/Igor_Engraver) (1996), but represents **fundamental architectural evolution** for collaborative distributed systems. Igor demonstrated successful viewport-driven computation and invalidation cycles within Common Lisp's MVC environment, but operated in single-user desktop contexts where MVC infrastructure was provided "for free" by the GUI framework.

**The Translation Challenge**: Ooloi operates across diverse frontend technologies (JavaFX, web browsers, mobile platforms) where **nothing can be assumed** about viewport handling, event management, or rendering capabilities. MVC mechanisms that Igor inherited from desktop environments have been **deliberately internalized into Ooloi's architecture** to ensure any frontend technology can implement the lazy patterns successfully.

**Scaling Through Systematic Laziness**: Ooloi operates under fundamentally different scaling constraints—distributed collaboration, massive orchestral scores, network latency—that **demand comprehensive lazy evaluation** at every architectural layer to surmount problems that would be intractable with eager computation approaches.

### Collaborative Scaling Challenge

Traditional music notation software operates in single-user desktop contexts where eager computation is feasible. Ooloi faces **multiplicative complexity**:

- **4 users × 120-page score** = potential for 480 pages of redundant computation without laziness
- **Network bandwidth explosion**: Eager updates generate megabytes of unused layout data
- **Cross-platform deployment**: Mobile devices, web browsers, desktop applications with varying performance characteristics
- **Real-time collaboration**: Multiple simultaneous users with immediate feedback requirements

### Data Flow and User Interaction Requirements

**Interactive UI Requirements**:
- Users click on rendered notation elements (notes, symbols, measures) requiring mapping to backend operations
- Frontend must identify clicked elements using cached backend identification data
- Immediate visual feedback before backend confirmation
- Graceful handling of backend operation failures

**Synchronization Complexity**:
- Frontend layout objects must stay synchronized with backend musical data using VPD-based identification
- Backend layout calculations may invalidate multiple UI regions requiring coordinated updates
- Updates may propagate across staves, systems, pages, or entire layouts
- Network latency requires asynchronous update patterns with robust error handling

## Decision

We will implement a **lazy, event-driven data synchronization architecture** that enables real-time collaborative editing of large orchestral scores through API operations and async event notifications, with comprehensive user interaction support through VPD-based frontend object hierarchies.

## Core Architecture

### Communication Patterns

Ooloi operates through two complementary communication mechanisms:

**Local API Operations (Shared Functions)**
- Client queries local cached data using shared API functions:
  - `(get-layout musical-item)` → entire score layout from local cache
  - `(get-page-view layout page-index)` → page data from local cache
  - `(get-system-view page-view system-index)` → system data from local cache  
  - `(get-staff-view system-view staff-index)` → staff data from local cache
  - `(get-measure-view staff-view measure-index)` → `MeasureView{glyphs, curves}` from local cache
- Same shared API functions work locally (frontend cache) and remotely (backend authority)

**gRPC API Operations (Network Requests)**
- Client mutations: `(add-articulation measure-vpd piece-id :staccato)` via ExecuteMethod
- Fresh data requests: Request updated shared models when local cache is dirty
- All network operations use ExecuteMethod via gRPC for consistent interface

**Event Notifications (Asynchronous)**
- Server notifies all clients when visual hierarchy objects change:
  - Layout invalidated: `"layout 'score' has changed"`
  - Page invalidated: `"page 5 has changed"`
  - System invalidated: `"system 2 on page 3 has changed"`
  - Staff invalidated: `"staff 1 in system 2 has changed"`
  - Measures invalidated: `"measures 125-127 in staff 1 have changed"`
- Events **mark local cache dirty** - clients know their local data is stale
- Events contain **only identifiers** of changed objects - no details about what aspect changed  
- Clients request fresh data via gRPC when they need to render dirty objects
- Triggered by any client's musical mutations that affect visual hierarchy

**Key Insight**: Events tell clients *which local objects are now stale*, gRPC requests provide *fresh data to update local cache*, local API provides *fast access to current cache*.

### Foundational Principles

**Backend Authority, Frontend Rendering**:
- **Backend** holds all musical data and performs all layout calculations
- **Frontend** renders using backend-computed drawing instructions (glyphs, curves)
- **Platform Independence**: Backend generates abstract drawing descriptions without knowledge of frontend rendering technology
- **No bitmap transfer**: Backend never generates or transfers graphical bitmaps - only vector drawing instructions
- **Renderer-specific caching**: Frontend rendering systems (JavaFX, Skija) may add their own bitmap caching independently
- **Separation of concerns**: Ooloi business logic manages vector data, rendering systems manage bitmap optimization

**Comprehensive Lazy Evaluation**:
- **Computation Timing**: Backend recalculates affected layouts on 100ms raster after musical changes
- **Data Transfer Laziness**: Layout recalculations trigger invalidation events, but visual data only transferred on client request
- **Memory Laziness**: Frontend caches visual data selectively and discards under memory pressure
- **Cache Disposability**: All cached visual representations can be safely discarded - always re-downloadable
- **Traversal Laziness**: Timewalk transducers with boundary-vpd pruning for O(log n) efficiency
- **Rendering Laziness**: Frontend only processes visible viewport content and only requests needed data

### Performance Architecture

**Two-Tier Performance System**:
- **Path A - Clean Data (95% of requests)**: 0.1ms response time for cached drawing instructions
- **Path B - Dirty Data (5% of requests)**: Raster interval delay plus computation time for uncached layout data

**Raster Deduplication**:
Backend uses configurable raster (e.g., 100ms) for computation-heavy operations while keeping normal API requests immediate. Multiple clients requesting identical data receive single computation distributed to all requesters.

**Scalability Characteristics**:
- **Score complexity independence**: 120 pages performs identically to 12 pages
- **Collaborative user independence**: 4 users don't multiply computation cost
- **Network efficiency**: Single note change → 4KB invalidation broadcast instead of megabytes of layout data

## User Interaction Architecture

### VPD-Based Frontend Object Hierarchy

**Shared Model Architecture**: Frontend uses the same models as backend (per ADR-0023) but maintains only the visual hierarchy elements it needs for current rendering. Frontend downloads visual hierarchy `Layout→PageView→SystemView→StaffView→MeasureView` incrementally and on-demand.

**Incremental Visual Data Management**:
- **Partial hierarchy**: Frontend maintains only visual elements for current and nearby viewports
- **On-demand loading**: Request specific PageView, SystemView, or MeasureView objects as needed
- **Selective caching**: Cache only the visual hierarchy elements required for user interaction
- **Viewport-driven requests**: Download visual data based on what user is viewing or likely to view
- **Lazy expansion**: Fetch additional visual elements when user scrolls or navigates
- **Conservative caching**: Retain visual representations for smooth navigation unless compelling reason to discard
- **Selective eviction**: Remove visual data only for piece closure or significant memory pressure
- **LRU strategy**: When cleanup required, remove least recently used visual elements from non-active pieces

**Frontend UI State Requirements**:
- **VPD linking**: Each displayed object must maintain VPD identification for backend operations
- **Cache state tracking**: Objects must track whether their visual data is clean, dirty, or being requested
- **Partial hierarchy navigation**: Handle cases where parent/child visual objects may not be loaded
- **Geometric bounds**: Hit-testing requires geometric information for user interaction
- **Backend authority**: Frontend never modifies musical model objects directly

**Architectural Principles**:
- Frontend maintains **UI coordination state** around shared model objects
- **VPD + item-id combination** uniquely identifies any musical element for backend operations  
- Frontend **never performs musical logic** - pure rendering and interaction coordination
- **Backend-computed drawing instructions are cached** until invalidated by events
- **Shared models ensure type fidelity** between frontend and backend representations

### Hit-Testing and Click Resolution

**User Interaction Flow**: User clicks → Frontend hit-testing → VPD resolution → Backend API call

**Hit-Testing Architecture Requirements**:
- **Geometric intersection**: Frontend must determine which visual elements contain the click point
- **VPD resolution**: Hit elements must be mapped to their corresponding VPDs for backend operations
- **Clean state dependency**: Interactive elements only available when visual data is current
- **Element identification**: Each renderable element must maintain backend identification metadata
- **Efficient spatial queries**: Hit-testing must scale efficiently with score complexity

**Click-to-Backend Resolution**:
- **Element selection**: Frontend identifies clicked element using geometric bounds
- **VPD extraction**: Selected element provides VPD for backend API calls  
- **Item identification**: Specific musical items within measures require unique identifiers
- **Operation dispatch**: VPD + item-id enables targeted backend mutations
- **State validation**: Frontend must verify data currency before allowing operations

**Architectural Constraints**:
- **No musical logic**: Frontend performs pure geometric-to-logical mapping
- **Backend authority**: All musical operations routed through backend API
- **Version synchronization**: Operations blocked on stale frontend data
- **Clean cache requirement**: Hit-testing only functions with current visual data

### Clean/Dirty State Integration

**Cache State Architecture**:
Frontend maintains cache state for all visual elements to coordinate rendering and user interaction with the lazy data model.

**State Management Approaches**:
- **Present and clean**: Visual data exists locally and is current
- **Missing or invalidated**: Visual element set to nil, :dirty, or removed entirely
- **Requesting**: Fresh data requested from backend, loading indicators active

**Implementation Flexibility**:
- **Dirty marking**: Can be implemented as boolean flags, nil values, or special :dirty markers
- **Cache removal**: Invalid visual representations can be discarded rather than marked
- **Lazy population**: Visual hierarchy elements loaded and cached on-demand

**Rendering Coordination**:
- **Clean state rendering**: Use local shared API calls for immediate access to cached visual data
- **Dirty state handling**: Visual placeholders while requesting fresh data via gRPC
- **Requesting state feedback**: Loading indicators during backend computation and cache updates
- **State-dependent optimization**: Only request fresh data via gRPC for visible dirty elements

**Local vs Network API Usage**:
- **Clean cached data**: Use local shared API functions (e.g., `get-measure-view`) for instant access
- **Dirty cached data**: Request fresh models via gRPC, then update local cache
- **Post-update access**: Return to local shared API functions for subsequent operations

**User Interaction State Integration**:
- **Interaction availability**: Elements only clickable in clean state
- **Operation blocking**: User operations disabled during dirty/requesting states
- **Visual feedback**: Clear indicators of element states for user awareness
- **Graceful degradation**: Appropriate fallbacks when backend operations fail

**Event-Driven Cache Management**:
- **Invalidation processing**: Backend events invalidate relevant frontend cache elements (mark dirty, set nil, or remove)
- **Viewport-aware response**: Clients only request fresh data for invalidated elements **currently visible in viewport**
- **Closed layout handling**: Elements in non-open layouts marked invalid for next time they're opened
- **Open but non-visible handling**: Elements in open layouts but outside current viewport marked invalid, no immediate data request
- **Visible element handling**: Only invalidated elements currently visible in viewport trigger immediate data requests
- **Deferred updates**: Invalid elements (closed layouts or non-visible areas) remain invalid until user navigates to them

## Windowing System Integration

### Viewport Management

**Viewport Architecture**: Windowing system (JavaFX + AtlantaFX) coordinates multiple independent viewports using lazy patterns for efficient rendering in large scores.

**Viewport State Requirements**:
- **Visible element tracking**: Each viewport must know which visual elements are currently displayed
- **Cache state coordination**: Viewport must track clean/dirty state of visible elements
- **Pending request management**: Coordination of backend data requests with visibility
- **Hierarchy navigation**: Efficient determination of viewport contents using VPD addressing

**Visibility Calculation Architecture**:
- **Efficient traversal**: Use timewalk with boundary-vpd pruning for O(log n) visibility determination
- **Geometric intersection**: Determine which visual elements intersect viewport bounds
- **Hierarchical optimization**: Leverage VPD structure to eliminate unnecessary traversals
- **Scalable performance**: Visibility calculations independent of total score complexity

**Lazy Scrolling Architecture**:
- **Incremental loading**: Download visual hierarchy elements as they become visible
- **Selective refresh**: Request fresh data only for dirty elements that become visible
- **On-demand expansion**: Fetch additional visual elements when user scrolls into new regions
- **Performance independence**: Scrolling performance decoupled from total score size
- **Memory efficiency**: Maintain minimal working set of visible and nearby elements
- **Predictive loading**: Pre-fetch visual data for likely scroll destinations

**Multi-Viewport Coordination**:
- **Independent state**: Each viewport maintains separate visibility and cache state
- **Event filtering**: Viewports receive only relevant invalidation events
- **Resource sharing**: Multiple viewports can share cached backend data
- **Efficient updates**: Coordinate updates across viewports without redundant computation

### Event Processing Pipeline

**Window-Level Event Coordination**: Window manager coordinates event processing across multiple viewports and UI components.

**Event Distribution Architecture**:
- **Event reception**: Window manager receives all invalidation events from backend
- **Viewport determination**: Identify which viewports contain affected visual elements
- **Targeted updates**: Route events only to viewports displaying affected content
- **Hierarchy processing**: Handle events at appropriate levels (measure, system, page, layout)

**Multi-Viewport Event Handling**:
- **Parallel processing**: Each viewport processes relevant events independently
- **Cache coordination**: Synchronize cache state updates across related viewports
- **Selective refresh**: Only request fresh data for visible affected elements in each viewport
- **Resource optimization**: Share computation results across viewports when possible

**Event Type Processing**:
- **Measure invalidation**: Target specific viewports displaying affected measures
- **System invalidation**: Update all viewports containing measures in affected systems
- **Page invalidation**: Refresh viewports displaying affected page content
- **Layout invalidation**: Coordinate full layout refresh across all relevant viewports

### UI Component Architecture

**Component Integration Requirements**: UI components must integrate with lazy architecture for consistent performance and data synchronization.

**Docking System Architecture**:
- **Independent viewports**: Each docked pane maintains separate viewport state and event subscriptions
- **Layout-specific events**: Panes receive filtered events relevant to their displayed content
- **Cache isolation**: Each pane manages independent cache state for its visual content
- **Coordinated updates**: Related panes coordinate updates while maintaining independence

**Modal Window Integration**:
- **Lazy data loading**: Windows request only necessary data for their specific editing context
- **Event subscription**: Windows subscribe to events affecting their displayed elements
- **VPD-scoped operation**: Window operations use VPD addressing for backend modifications
- **State synchronization**: Window content stays synchronized with backend changes

**Context Menu Architecture**:
- **VPD-based options**: Menu options determined by VPD type of clicked elements
- **Element-specific operations**: Different menu structures for notes, measures, systems, etc.
- **Backend operation routing**: Menu actions use VPD addressing for consistent backend calls
- **Dynamic content**: Menu options reflect current element state and permissions

**Component State Management**:
- **Shared lazy patterns**: All UI components use consistent cache state and event handling
- **VPD addressing**: Components use VPD system for element identification and operations
- **Event filtering**: Components receive only events relevant to their displayed content
- **Backend coordination**: All component operations route through consistent backend API

## Collaborative Architecture

### Natural Multi-Client Support

**Collaboration as Architectural Consequence**: The lazy event-driven architecture naturally enables collaboration without special "collaborative features" - multiple clients simply use the same patterns.

**Single-Client Operations**:
1. User clicks note → Frontend resolves VPD → API call `(add-articulation note-vpd piece-id :staccato)`
2. Backend updates musical data → Backend broadcasts event: `{:type :piece-measures-invalidated :measures [127]}`
3. Same client receives event → Marks measure 127 dirty → Requests fresh data when visible

**Multi-Client Operations** (identical pattern):
1. User A clicks note → Frontend resolves VPD → API call `(add-articulation note-vpd piece-id :staccato)`
2. Backend updates musical data → Backend broadcasts event to **ALL subscribed clients**: `{:type :piece-measures-invalidated :measures [127]}`
3. Users A, B, C receive same event → Each marks measure 127 dirty → Each requests fresh data when visible

**Key Insight**: Collaboration requires no special code - it's simply multiple clients receiving the same events and using the same lazy data retrieval patterns.

### Event Subscription Management

**Automatic Server Events**: All clients automatically receive server-level events upon connection
```clojure
;; Server events broadcast to all connected clients
{:type :server-shutdown :message "Maintenance in 5 minutes"}
{:type :server-maintenance :message "Backup in progress"}
```

**Manual Piece Subscriptions**: Clients explicitly subscribe to piece-specific events
```clojure
;; Client subscribes when opening a piece
(execute-method :subscribe-to-piece {:piece-id "symphony-no-5"})

;; Client receives piece-specific invalidation events
{:type :piece-measures-invalidated :piece-id "symphony-no-5" :measures [127]}

;; Client unsubscribes when closing piece  
(execute-method :unsubscribe-from-piece {:piece-id "symphony-no-5"})
```

**Server-Side Subscription Management**: Backend maintains subscription registry for targeted event routing

**Registry Architecture**:
- **Client tracking**: Map piece IDs to sets of subscribed client identifiers
- **Event routing**: Direct events only to clients subscribed to affected pieces
- **Dynamic management**: Support subscription and unsubscription during client sessions
- **Cleanup integration**: Automatic subscription removal when clients disconnect

**Subscription Lifecycle**:
- **Connection establishment**: Automatic server event subscription for all clients
- **Piece opening**: Explicit subscription when clients open specific pieces
- **Piece closing**: Explicit unsubscription when clients close pieces
- **Disconnection cleanup**: Automatic removal from all piece subscriptions

### Collaborative Update Handling

**Multi-Viewport Collaboration**: Different users viewing different sections receive targeted updates based on their viewport contents and subscription state.

**Collaborative Scenario Architecture**:
- **Independent viewports**: Multiple users view different score sections simultaneously
- **Targeted notifications**: Users receive events only for pieces they have open
- **Viewport-aware updates**: Client response depends on whether affected elements are visible
- **Selective data requests**: Fresh data requested only when affected content is currently displayed

**Update Distribution Pattern**:
1. **Single user operation**: Any client performs musical modification via backend API
2. **Backend processing**: Backend updates musical data and identifies affected visual elements  
3. **Event broadcast**: Backend sends invalidation events to all clients subscribed to affected piece
4. **Client filtering**: Each client processes events based on their current viewport contents
5. **Lazy refresh**: Clients request fresh data only for affected elements that are currently visible

**Viewport-Specific Response Patterns**:
- **Visible affected elements**: Mark dirty, request fresh data immediately
- **Non-visible affected elements**: Mark dirty, defer data request until visible
- **Non-relevant content**: No action required for elements outside client's layouts

**Conflict Resolution Through Backend Authority**:
- **No conflict resolution needed**: Backend processes mutations sequentially
- **Last-write-wins**: Backend STM handles concurrent mutations using established precedence
- **Optimistic UI updates**: Frontend shows immediate feedback, corrected by backend events if needed
- **Version tracking**: Frontend prevents operations on stale data using version stamps

**Collaborative Efficiency**:
- **Event deduplication**: Single backend event notifies all clients simultaneously
- **Raster synchronization**: Multiple clients requesting same data get shared computation
- **Viewport independence**: Users in different score sections don't interfere with each other
- **Network optimization**: 4KB event broadcast instead of megabytes per client

## Platform Independence

### MVC Internalization

**The Igor Engraver Context**: In 1996, Igor Engraver operated within Common Lisp's desktop environment where viewport management, invalidation handling, and event coordination were provided by the GUI framework.

**The Ooloi Challenge**: Operating across diverse frontend technologies (JavaFX, web browsers, mobile platforms), Ooloi cannot assume MVC infrastructure exists. Rather than requiring frontends to implement complex MVC patterns, Ooloi **internalizes coordination mechanisms**:

- **Viewport Management**: Backend calculates viewport intersections using VPD-bounded timewalk
- **Invalidation Logic**: Backend handles cascade analysis and dirty/clean state tracking  
- **Event Coordination**: Event notification system eliminates complex frontend event management
- **Cache Management**: Backend provides clean/dirty specifications any frontend can implement

### Frontend Technology Flexibility

**Universal Compatibility**: The lazy event-driven patterns work across any frontend technology:

**Clojure/JavaFX Desktop**: 
- Native desktop performance with Skija rendering
- JavaFX/Skija may cache rendered bitmaps independently for performance
- Direct STM integration for viewport state management
- Efficient Java interop for GUI components

**Web Frontend**:
- Browser-based client using HTML Canvas or WebGL
- Browser may cache rendered elements independently
- Same VPD-based object hierarchy implemented in JavaScript
- WebSocket transport for event streaming

**Mobile Platforms**:
- Native iOS/Android clients with platform-specific rendering
- Platform rendering systems handle bitmap caching independently
- Touch interaction patterns using same VPD resolution
- Offline capability with sync-when-connected patterns

**Future Technologies**:
- VR/AR clients with 3D spatial rendering
- Voice-controlled interfaces using VPD addressing
- Specialized display technologies (e-ink, projection, etc.)
- Each platform manages rendering optimization independently from Ooloi business logic

**Rendering Architecture Separation**:
- **Ooloi level**: Manages shared models (MeasureView{glyphs, curves}) and vector drawing instructions only
- **Rendering level**: Platform-specific systems (JavaFX, Skija, Canvas, Metal, etc.) handle bitmap generation and caching
- **Clear boundary**: No bitmap data crosses the Ooloi business logic layer - only vector descriptions
- **No bitmap storage**: Piece and layout representations in Ooloi contain zero bitmap data - purely vector/semantic
- **Bitmap generation on demand**: All bitmaps generated by frontend rendering systems from vector instructions when needed

## Rationale

### Lazy vs Eager Computation

**Why Lazy**:
- **99% of layout work deferred** until actually needed for visible content
- **Collaborative efficiency**: Multiple users don't trigger redundant computation
- **Predictable performance**: Response time independent of total score size  
- **Resource optimization**: CPU cycles spent only on viewport content

**vs Eager**: Would compute all layouts immediately after any change, prohibitive for large orchestral scores with collaborative editing requiring hundreds of concurrent layout calculations.

### API Operations vs Event Notifications

**Why Separate Communication Mechanisms**:
- **Decoupled communication**: Data requests independent of invalidation notifications
- **Network efficiency**: Invalidations broadcast tiny payloads, data flows only when needed
- **Lazy triggering**: Clients decide when they actually need fresh data
- **Collaborative scalability**: One user's edit notifies all users without forcing computation

**vs Traditional Request-Response**: Single-phase systems couple mutation with data retrieval, forcing computation of invisible layouts and preventing efficient collaborative distribution.

### Backend Authority vs Distributed Computation

**Why Backend Authority**:
- **Single source of truth**: Prevents collaborative consistency issues
- **Frontend simplicity**: No need to duplicate complex musical algorithms across technologies
- **Platform independence**: Abstract drawing instructions work across frontend technologies
- **Conflict-free collaboration**: Sequential backend processing eliminates merge conflicts

**vs Distributed**: Multiple frontends with musical logic would require complex state synchronization protocols and risk musical consistency violations.

## Consequences

### Positive Outcomes

1. **Massive Score Scalability**: 120-page orchestral works perform identically to simple pieces - 99% of computation deferred
2. **Sub-Millisecond Responsiveness**: 95% of user interactions receive 0.1ms response via cached data retrieval  
3. **Collaborative Efficiency**: Multiple users in different sections don't trigger redundant computation
4. **Network Optimization**: Single musical change generates 4KB broadcast instead of megabytes per client
5. **Universal Frontend Compatibility**: Internalized MVC enables compatibility across frontend sophistication levels
6. **Natural Collaboration**: Multi-user support emerges from architecture without special collaborative features

### Trade-offs and Risks

1. **Two-Tier Complexity**: System has distinct performance modes requiring careful UI state management
2. **Configurable Latency Floor**: Dirty data requests wait for raster pulse - cannot achieve sub-raster intervals
3. **Network Dependency**: Frontend becomes non-functional if invalidation stream fails
4. **Cache Invalidation Precision**: Must accurately identify all affected measures across layouts
5. **Cold-Start Performance**: First-time layout opening requires computing everything from scratch
6. **Frontend Complexity**: VPD-based object hierarchies require careful implementation

### Mitigation Strategies

- **Deployment-specific tuning**: Raster timing optimized per deployment (20ms in-process, 100ms SaaS)
- **Graceful degradation**: Frontend continues with stale data if invalidation stream fails, with user indicators
- **Connection resilience**: Automatic reconnection with state resynchronization
- **Memory management**: LRU eviction under memory pressure, safe cache disposal since all visual data is re-downloadable
- **Performance monitoring**: Built-in metrics track two-tier response characteristics
- **Frontend scaffolding**: Reference implementations and libraries for common VPD patterns

## Example Scenarios

### Typical Piece Opening and Navigation Workflow

**Opening a Piece**:
1. User opens piece: no visual data downloaded initially
2. User opens layout (full score, violin part, etc.): still no download
3. User navigates to visible content: frontend checks local cache for needed visual elements
4. Missing elements trigger gRPC requests for required PageView, SystemView, MeasureView objects
5. Available elements render immediately (0.1ms), missing elements show placeholders until fetched

**Progressive Navigation**:
1. User scrolls to new page: frontend calls `(get-page-view layout 15)` **locally**
2. If clean and cached: immediate rendering (0.1ms)
3. If missing or dirty: gRPC fetch from backend, update local cache, then render
4. User continues scrolling: pattern repeats for newly visible elements
5. Previously visited pages render instantly from local cache

**Large Navigation Jumps**:
1. User jumps 20 pages ahead: most visual elements not in local cache
2. Frontend requests needed elements via gRPC for new viewport
3. Progressive rendering as elements arrive from backend
4. Optional: pre-fetch adjacent pages based on navigation patterns

**Memory Management Triggers**:
- **Piece closing**: Discard all visual representations for closed pieces
- **Memory pressure**: LRU eviction of visual elements from non-active pieces/layouts
- **Long sessions**: Periodic cleanup of unused visual data
- **NOT for casual scrolling**: Keep visual data available for smooth navigation

### User Adds Articulation
1. Frontend calls `(add-articulation note-vpd piece-id :staccato)` **via gRPC** to backend
2. Backend modifies musical data, queues layout recalculation for next 100ms raster
3. On 100ms raster: Backend recalculates affected MeasureView data
4. Backend broadcasts: `{:type :piece-measures-invalidated :piece-id "symphony" :measures [127]}`
5. All clients invalidate local measure 127 cache (set to nil, :dirty, or remove)
6. **Viewport-aware client responses**:
   - Client viewing pages 12-13 with measure 127 visible: request fresh data **via gRPC**, update local cache
   - Client with same layout open but viewing page 8: invalidate measure 127 cache, no immediate data request
   - Client with layout closed: invalidate measure 127 cache, no immediate data request
7. When clients later navigate to measure 127: `(get-measure-view staff-view 127)` triggers gRPC request for fresh data

### Collaborative Editing Session
1. Four users open same symphony: conductor (full score), violinist (part), pianist (reduction), composer (editing)
2. Composer changes instrument name in system → backend broadcasts system invalidation event
3. Conductor and violinist see system update in their viewports, pianist unaffected
4. Violinist adds dynamics to measure → backend broadcasts measure invalidation event
5. All users viewing that measure see update, others remain unaffected
6. Single backend maintains consistency, all frontends stay synchronized through events

## Related ADRs

- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Establishes separation requiring this synchronization
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Communication protocol enabling event streaming
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model for viewport state management
- [ADR-0008: VPDs](0008-VPDs.md) - Hierarchical addressing enabling efficient traversal and user interaction
- [ADR-0014: Timewalk](0014-Timewalk.md) - Efficient traversal supporting lazy evaluation and viewport management
- [ADR-0018: API-gRPC Interface and Events](0018-API-gRPC-Interface-and-Events.md) - API methods and event broadcasting used in communication
- [ADR-0023: Shared Model Contracts](0023-Shared-Model-Contracts.md) - Multi-project architecture enabling frontend-backend integration
- [ADR-0024: gRPC Concurrency and Flow Control Architecture](0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) - Event streaming flow control for collaborative synchronization
- [ADR-0025: Server Statistics Architecture](0025-Server-Statistics-Architecture.md) - Collaborative session analytics and performance monitoring
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Complete implementation of the event-driven synchronization architecture specified here
- [ADR-0038: Backend-Authoritative Rendering and Terminal Frontend Execution](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md) - Backend-authoritative rendering with GPU-accelerated frontend execution implementing the lazy rendering patterns specified here
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) - Single-authority model enabling lazy frontend architecture
