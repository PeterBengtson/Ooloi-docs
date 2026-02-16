# ADR-0015: Undo and Redo Architecture in Collaborative Environment

## Status

Proposed

## Context

Ooloi's architecture presents unique challenges for implementing undo/redo functionality across a collaborative, distributed system. The system must coordinate undo/redo operations across multiple concerns:

1. **Musical content changes** (notes, rhythms, attachments) stored on the backend
2. **UI state changes** (themes, layouts, zoom levels) on individual frontend clients  
3. **Application settings** (server configuration, user preferences) with unclear scope and location

The architectural constraints established by previous ADRs create specific requirements:

- **ADR-0001**: Frontend-backend separation with two deployment models (combined desktop application, backend-only server)
- **ADR-0002**: gRPC streaming for real-time collaboration between frontend clients and backend server
- **ADR-0004**: STM-based concurrency for coordinated updates to musical structures
- **ADR-0009**: Multi-user collaborative editing with conflict resolution

### Key Architectural Realities

**Backend (Server):**
- Stores authoritative piece data using STM refs
- Handles all piece modifications with automatic conflict resolution
- Streams piece updates to connected frontend clients via gRPC
- Does NOT store UI preferences or client-specific state

**Frontend (Clients):**
- Receives piece data via gRPC streaming (no local piece storage)
- Manages local UI state (themes, panel layouts, zoom levels)
- Displays piece content received from backend
- Handles UI interactions and forwards piece modifications to backend

**Collaborative Context:**
- Multiple frontend clients can edit the same piece simultaneously
- Backend coordinates all piece changes using STM transactions
- Frontend clients see real-time updates from other collaborators
- Piece modifications must be coordinated; UI changes are client-local

## Decision

We will implement a **three-tier undo/redo architecture** that separates concerns according to the system's architectural boundaries:

### Tier 1: Backend Piece Undo/Redo (Coordinated)
- **Scope**: Musical content changes (notes, rhythms, tempo, key signatures, attachments)
- **Implementation**: STM-based coordinated undo/redo using existing transactional infrastructure
- **Coordination**: Single authoritative undo/redo stack shared across all collaborating clients
- **Storage**: Backend server maintains the undo/redo history in STM refs
- **Distribution**: Undo/redo operations broadcast to all connected clients via gRPC streaming

### Tier 2: Frontend UI Undo/Redo (Local)
- **Scope**: Client-specific UI state (themes, panel arrangements, zoom levels, selection state)
- **Implementation**: Simple local undo/redo stack per frontend client
- **Coordination**: No coordination needed - purely local to each client
- **Storage**: Frontend client memory (not persisted)
- **Distribution**: Not distributed - each client manages its own UI history

### Tier 3: No Backend Application Settings Component
- **Decision**: Backend will NOT implement an application settings component
- **Rationale**: Analysis reveals no legitimate backend application settings that require undo/redo
- **Configuration**: Server deployment configuration handled via environment variables and config files
- **User Preferences**: Stored and managed entirely by frontend clients

### Unified User Experience
- **Single Undo/Redo Interface**: Frontend presents one undo/redo button to users
- **Smart Coordination**: Frontend determines whether to undo local UI change or send undo request to backend
- **Contextual Behavior**: Undo behavior depends on what user action was most recent (UI change vs piece modification)
- **Descriptive Messages**: Undo/redo operations display clear descriptions like "Undo Add Note", "Redo Change Key Signature" rather than generic "Undo"/"Redo"

## Rationale

### STM Provides Natural Collaborative Undo/Redo
Clojure's STM system already provides the foundation for coordinated undo/redo:
- **Atomic Operations**: All piece modifications occur within `dosync` transactions
- **Conflict Resolution**: Automatic retry mechanism handles concurrent modifications
- **Consistency**: Either all changes in a transaction apply, or none do
- **Scalability**: Designed for high-performance concurrent operations (100,000+ transactions/second)

### Clear Separation Prevents Architectural Complexity
Attempting to unify undo/redo across different architectural tiers would require:
- Complex event coordination systems layered on top of STM
- Artificial coupling between UI state and musical content
- Global state management breaking STM paradigm
- Synchronization complexity across network boundaries

### Backend Settings Analysis Reveals Minimal Needs
Examination of potential backend application settings shows:
- **Server configuration** (ports, TLS, capacity limits) → Environment variables/config files
- **User session preferences** → Frontend client storage
- **Piece-specific settings** → Part of piece data, not separate settings
- **Musical algorithm parameters** (like `MAX_LOOKAHEAD_MEASURES`) → Code constants, not user settings

### Frontend-Backend Boundary Alignment
The three-tier approach aligns perfectly with existing architectural boundaries:
- **Backend responsibility**: Musical data integrity and coordination
- **Frontend responsibility**: User interface and local preferences  
- **gRPC boundary**: Clean separation maintained without artificial unification

## Implementation Approach

### Backend Implementation
```clojure
;; Extend existing STM-based piece management with descriptive messages
(defrecord UndoRedoEntry [previous-content current-content description timestamp])
(defrecord PieceState [content undo-stack redo-stack])

(defn record-piece-change 
  "Records a piece change with descriptive message for undo/redo display"
  [piece-ref new-content description]
  (dosync
    (let [current-state @piece-ref
          old-content (:content current-state)
          undo-entry (->UndoRedoEntry old-content new-content description (System/currentTimeMillis))]
      (alter piece-ref 
             #(-> %
                  (assoc :content new-content)
                  (update :undo-stack conj undo-entry)
                  (assoc :redo-stack []))))))

(defn undo-piece-change [piece-ref]
  (dosync
    (let [current-state @piece-ref
          last-change (peek (:undo-stack current-state))]
      (when last-change
        (alter piece-ref 
               #(-> %
                    (assoc :content (:previous-content last-change))
                    (update :undo-stack pop)
                    (update :redo-stack conj last-change)))))))

;; Broadcast undo operations with descriptions to all connected clients
(defn broadcast-undo [piece-id description]
  (doseq [client (get-connected-clients piece-id)]
    (send-undo-notification client piece-id description)))
```

### Frontend Implementation
```clojure
;; Local UI undo/redo stack with descriptions
(defrecord UIUndoEntry [previous-state current-state description timestamp])
(defrecord UIState [theme layout zoom-level panels undo-stack redo-stack])

(defn unified-undo []
  "Single undo function that handles both UI and piece changes with descriptions"
  (cond
    (recent-ui-change?) 
    (let [last-ui-change (peek (:undo-stack @ui-state-atom))]
      (when last-ui-change
        (show-status-message (str "Undo " (:description last-ui-change)))
        (undo-ui-change)))
    
    (recent-piece-change?) 
    (let [last-piece-change (get-last-piece-change)]
      (when last-piece-change
        (show-status-message (str "Undo " (:description last-piece-change)))
        (send-undo-request-to-backend)))
    
    :else (no-op)))

;; UI state management with descriptive messages
(defn record-ui-change [old-state new-state description]
  (let [undo-entry (->UIUndoEntry old-state new-state description (System/currentTimeMillis))]
    (swap! ui-state-atom 
           #(-> %
                (merge new-state)
                (update :undo-stack conj undo-entry)
                (assoc :redo-stack [])))))

;; Menu generation for undo/redo items
(defn generate-undo-menu []
  (let [ui-undo (peek (:undo-stack @ui-state-atom))
        piece-undo (get-last-piece-change)]
    (cond
      (and ui-undo piece-undo)
      (if (> (:timestamp ui-undo) (:timestamp piece-undo))
        (str "Undo " (:description ui-undo))
        (str "Undo " (:description piece-undo)))
      
      ui-undo (str "Undo " (:description ui-undo))
      piece-undo (str "Undo " (:description piece-undo))
      :else "Undo")))
```

### gRPC Service Extensions
```protobuf
service OoloiService {
  // Existing piece operations...
  
  // Undo/redo operations with descriptive messages
  rpc UndoPieceChange(UndoRequest) returns (UndoRedoResponse);
  rpc RedoPieceChange(RedoRequest) returns (UndoRedoResponse);
  
  // Real-time streaming for collaborative updates including undo/redo descriptions
  rpc StreamPieceUpdates(StreamRequest) returns (stream PieceUpdate);
}

message UndoRedoResponse {
  PieceUpdateResponse piece_update = 1;
  string description = 2;  // e.g., "Add Note", "Change Key Signature"
  int64 timestamp = 3;
}

message PieceUpdate {
  string piece_id = 1;
  bytes piece_data = 2;
  string change_description = 3;  // Description of what changed for collaborative awareness
  string user_id = 4;            // Who made the change
}
```

## Consequences

### Positive

1. **Leverages Existing STM Infrastructure**: No additional concurrency complexity required
2. **Clean Architectural Separation**: Maintains frontend/backend boundaries established by ADR-0001
3. **Optimal Performance**: STM provides automatic optimization for collaborative scenarios
4. **Simplified Development**: No artificial unification complexity
5. **Scalable Collaboration**: STM-based approach proven to handle 100,000+ concurrent operations
6. **Consistent User Experience**: Single undo interface despite underlying architectural separation
7. **Reduced Complexity**: Eliminates need for backend application settings component

### Negative

1. **Frontend Coordination Logic**: Frontend must intelligently route undo/redo operations
2. **Network Dependency**: Piece undo/redo requires backend connectivity
3. **State Synchronization**: Frontend must track what operations are local vs backend
4. **Testing Complexity**: Must test coordination across network boundaries

### Mitigations

1. **Clear State Tracking**: Frontend maintains clear distinction between local and remote operations
2. **Offline Resilience**: UI undo/redo works offline; piece undo/redo queued until reconnection
3. **Comprehensive Testing**: Integration tests covering collaborative undo/redo scenarios
4. **Developer Tools**: Clear debugging for undo/redo operation routing

## Alternatives Considered

### Alternative 1: Unified Global Undo/Redo
**Approach**: Single undo/redo stack handling all changes (UI + piece content)
**Rejection Reasons**:
- Requires complex event coordination on top of STM
- Breaks architectural separation established by ADR-0001
- Introduces artificial coupling between UI state and musical content
- Performance overhead of coordinating UI changes across network

### Alternative 2: Piece-Only Settings (Everything in Piece Data)
**Approach**: Store all "settings" within individual pieces
**Rejection Reasons**:
- Confuses piece content with application configuration
- No settings actually belong to pieces vs UI preferences vs deployment config
- Would duplicate UI preferences across all pieces unnecessarily

### Alternative 3: Event Sourcing on Top of STM
**Approach**: Add event sourcing layer for unified undo/redo
**Rejection Reasons**:
- Adds complexity without benefits (STM already provides event coordination)
- Duplicates STM's built-in transaction and retry mechanisms
- Performance overhead of additional abstraction layer

### Alternative 4: Backend Application Settings Component
**Approach**: Implement settings component for backend configuration
**Rejection Reasons**:
- Analysis reveals no legitimate backend settings requiring undo/redo
- Server configuration belongs in deployment config, not application state
- User preferences belong in frontend, not backend
- Would create artificial coordination complexity

## References

### Related ADRs
- [ADR-0000: Clojure](0000-Clojure.md) - Language providing STM foundation for coordinated undo/redo
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural boundaries defining undo/redo scope separation
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Communication protocol for coordinating undo/redo across clients
- [ADR-0004: STM for Concurrency](0004-STM-for-concurrency.md) - Concurrency model enabling collaborative undo/redo
- [ADR-0009: Collaboration](0009-Collaboration.md) - Collaborative editing context requiring coordinated undo/redo
- [ADR-0016: Settings](0016-Settings.md) - Settings architecture implementing piece-specific configuration identified in this analysis

### Technical Considerations
- STM performance characteristics (100,000+ transactions/second on modest hardware)
- gRPC streaming capabilities for real-time undo/redo coordination
- Frontend state management patterns for UI undo/redo
- Network resilience patterns for offline undo/redo operations

## Notes

This architecture decision removes the "Application Settings" component from the backend development queue. The analysis demonstrates that no legitimate backend application settings exist that would require the complexity of a dedicated component.

However, the analysis identified that **piece-specific settings** (configuration attributes that travel with piece data) do have legitimate use cases. The implementation of this piece-specific settings architecture is detailed in [ADR-0016: Settings](0016-Settings.md), which provides a comprehensive solution for configuration attributes across all musical and visual entities within pieces.

The three-tier approach provides clean separation while maintaining the user experience expectation of unified undo/redo. The STM-based backend tier leverages Clojure's proven concurrency model, while the frontend tier handles local concerns without artificial coordination complexity.

Future enhancements could include:
- Persistent undo/redo history across application restarts
- Advanced undo/redo visualization showing collaborative change attribution
- Selective undo/redo for specific types of changes
- Integration with version control systems for long-term history management

The implementation should monitor performance characteristics in collaborative scenarios and user experience feedback to validate the three-tier approach effectiveness.