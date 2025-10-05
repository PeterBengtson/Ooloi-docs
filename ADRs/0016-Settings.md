# ADR-0016: Settings

## Status

Accepted - July 24, 2025

## Context

Ooloi's musical notation system requires extensive configuration capabilities for visual rendering, layout preferences, and performance characteristics. The system needs a systematic approach for managing these configuration attributes across the hierarchical structure of musical entities.

### Current Architecture Analysis

**Musical Entity Hierarchy**: Piece → Musicians → Instruments → Staves → Measures → Voices → Items
**Visual Entity Hierarchy**: Layouts → PageViews → SystemViews → StaffViews → MeasureViews

Each entity type currently uses defrecord slots that fall into two categories:
1. **Machinery Slots**: Essential for engine operation (vectors, IDs, musical structure, core data)
2. **Configuration Attributes**: Display properties, preferences, and rendering parameters

### Architectural Constraints from Previous ADRs

**ADR-0015: No Backend Application Settings**
- Backend does NOT implement application settings component
- Server deployment configuration via environment variables/config files
- User preferences stored and managed entirely by frontend clients
- "Piece-specific settings" explicitly identified as part of piece data, not separate settings

**ADR-0001: Frontend-Backend Separation**
- Backend handles authoritative piece data using STM
- Frontend manages local UI state and preferences
- Clear boundary maintained between musical content and application preferences

### Configuration Requirements

The system needs to support configuration attributes across the entity hierarchy:
- Some attributes are currently direct defrecord slots (`:name`, `:default-clef`, `:num-lines`)
- Others use specialized storage (ChangeSets for time signatures)
- Future needs require a systematic approach for adding new configuration attributes
- Storage efficiency requires a mechanism for default values and optimization

Future configuration needs include:
- **Visual rendering**: beam thickness, staff spacing, note head sizes, line weights
- **Layout preferences**: margins, system breaks, page settings
- **Performance attributes**: invisible items, playback velocities, swing ratios
- **User customizations**: custom symbols, color schemes, accessibility options

### Current Architecture Foundation

**Metadata-Based Method Categorization**:
The backend uses a clean metadata-driven architecture for method dispatch:

- **Direct metadata queries**: All VPD macros use `methods-with-category` for real-time filtering by `:vpd-category` metadata
- **Pure functional approach**: On-demand filtering with `(-> @method-var meta :vpd-category)` queries
- **Nine category system**: `:getters`, `:set-item`, `:set-seq`, `:set-attribute`, `:change-set-item-set`, `:change-set-seq`, `:change-set-item-remove`, `:settings-get`, `:settings-set`

**VPD Dispatch Architecture**:
Each category maps directly to a VPD macro through metadata filtering:
```clojure
(let [getters (methods-with-category :getters)]
(let [setters (methods-with-category :change-set-seq)]
(let [removers (methods-with-category :change-set-item-remove)]
(let [settings-getters (methods-with-category :settings-get)]
(let [settings-setters (methods-with-category :settings-set)]
```

This architecture provides the foundation for Universal Entity Settings through two additional metadata categories (`:settings-get`, `:settings-set`) with complete VPD dispatch support.

### The Uniform API Requirement

The codebase architecture principles establish that uniform interfaces are paramount and hybrid approaches where some functions are in namespace X, others in namespace Y based on implementation details are unacceptable. Any configuration solution must provide consistent API patterns regardless of underlying storage mechanisms.

## Decision

We will implement a **Universal Entity Settings Architecture** that provides configuration capabilities for any entity in Ooloi's musical and visual hierarchies while maintaining storage efficiency and API consistency.

### Core Architecture Principles

1. **Universal Applicability**: Any entity (Piece, Musician, Staff, Pitch, Layout, etc.) can have configuration settings
2. **Lazy Storage**: `:settings` maps created only when non-default values are set
3. **Unified API**: Consistent `get-<setting>`/`set-<setting>` functions regardless of storage mechanism
4. **Default Value System**: Systematic default value management with automatic cleanup
5. **VPD Integration**: Full Vector Path Descriptor support with transaction semantics

### Implementation Strategy

#### Settings-Backed Attribute Functions

New `defsetting` macro in `models.core` generates functions that look identical to existing attribute functions but use map-based storage:

```clojure
;; Existing direct slot attribute
(defattribute Staff :name)          ; Direct field access
;; → get-name, set-name

;; New settings-backed attribute  
(defsetting Staff :beam-thickness 0.5)  ; Map-based with default
;; → get-beam-thickness, set-beam-thickness
```

#### Usage with Full VPD Support
```clojure
;; Direct usage
(set-beam-thickness staff 0.8)

;; VPD usage with automatic STM transaction
(set-beam-thickness [:musicians 0 :instruments 0 :staves 0] piece-ref 0.8)

;; Lazy storage - only non-default values stored
(get-beam-thickness staff)  ; Returns 0.5 (default) without storage overhead
```

### Storage Efficiency

- **Default Values**: Not stored in `:settings` maps, saving memory for 99.9% of entities
- **Lazy Maps**: `:settings` only created when first non-default value is set
- **Automatic Cleanup**: Setting a value to its default removes it from storage
- **Empty Map Removal**: Empty `:settings` maps are completely removed

### API Consistency

Users experience identical interfaces regardless of underlying storage:
```clojure
;; Direct slot attributes (existing)
(get-name staff)              ; Direct field access
(set-name staff "Violin I")   ; Direct field update

;; Settings-backed attributes (new)
(get-beam-thickness staff)    ; Map access with default fallback
(set-beam-thickness staff 0.6) ; Map update with cleanup logic
```

### Default Value Management

**Defaults Registry Architecture**: 

A central, dynamically-populated registry provides default value storage and discoverability:

```clojure
;; Central registry - populated at compile-time by defsetting macro
(defonce defaults-registry (atom {}))
;; Structure: {Staff {:beam-thickness 0.5 :staff-spacing 10.0}
;;            Measure {:beam-thickness 0.8 :width 100.0}}
```


## Rationale

### Alignment with Existing Architecture

**Maintains ADR-0015 Boundaries**: This architecture applies only to piece data entities, not backend application settings. All settings travel with piece data and are serialized with pieces, maintaining the established separation.

**Leverages ADR-0001 Separation**: Settings remain part of musical content managed by the backend, while UI preferences continue to be handled by frontend clients.

**Uses Existing Infrastructure**: VPD dispatch, STM transactions, and multimethod patterns from the current architecture provide the foundation.

### Uniform Interface Achievement

**Single API Pattern**: All settings use identical `get-<setting>`/`set-<setting>` patterns regardless of storage mechanism, satisfying the uniform interface architectural requirements.

**No Cognitive Load**: Users don't need to know whether an attribute is a direct slot or settings-backed - the API is identical.

**Future-Proof**: New configuration attributes can be added without API changes or architectural decisions.

### Storage Optimization

**Memory Efficiency**: 99.9% of entities store no settings data, maintaining compact representation for large pieces with 100,000+ notes.

**Performance**: Direct map access for non-default values, with automatic fallback to defaults.

**Scalability**: Approach scales to any number of setting types without per-entity overhead.

### Development Experience

**Consistent Patterns**: Developers use the same macro and helper functions for all settings across all entity types.

**Automatic VPD Support**: Settings immediately work with Vector Path Descriptors and STM transactions without additional implementation.

**Clear Intent**: `defsetting` declarations make configuration attributes explicit and documented.

## Implementation

The Universal Entity Settings architecture is implemented in `backend/src/main/clojure/ooloi/backend/ops/vectors_and_attributes.clj` with comprehensive test coverage in the corresponding test namespace.

Key implementation aspects:
- `defsetting` macro with runtime validation and defaults registry integration
- `get-setting`/`set-setting` helper functions with lazy storage and cleanup
- Full VPD integration with STM transaction support
- Metadata-based method categorization for dispatch system integration

## Consequences

### Positive

1. **Uniform Configuration API**: Consistent interface across all entity types and setting types
2. **Storage Efficiency**: Minimal memory overhead for entities using default values
3. **Architectural Consistency**: Builds on existing VPD, STM, and multimethod patterns
4. **Development Velocity**: Easy addition of new configuration attributes without architectural decisions
5. **User Experience**: Predictable API behavior regardless of underlying storage mechanisms
6. **Future Extensibility**: Foundation for plugin-defined settings and advanced customization

### Negative

1. **Implementation Complexity**: Additional abstraction layer over direct field access
2. **Performance Overhead**: Map lookups for non-default values vs direct field access
3. **Default Value Architecture**: Additional architectural decision required for default management
4. **Testing Complexity**: Must verify both direct usage and VPD dispatch patterns
5. **Migration Effort**: Existing configuration attributes may need migration to new pattern

### Mitigations

1. **Performance Monitoring**: Measure and optimize map lookup performance in realistic scenarios
2. **Incremental Implementation**: Deploy settings pattern incrementally, maintaining existing attributes
3. **Comprehensive Testing**: Automated testing for storage efficiency, API consistency, and VPD integration
4. **Clear Documentation**: Guidelines for when to use settings vs direct slots
5. **Developer Tools**: Tooling to verify storage optimization and detect configuration issues

## Alternatives Considered

### Alternative 1: Dedicated Slots for All Configuration

**Approach**: Add dedicated defrecord slots for each configuration attribute
**Rejection Reasons**:
- Violates uniform interface principle (different APIs for different attributes)
- Creates storage overhead for all entities regardless of usage
- Requires defrecord modifications for each new configuration attribute
- No systematic approach to default values

### Alternative 2: Global Configuration Registry

**Approach**: Separate configuration system outside of piece data
**Rejection Reasons**:
- Violates ADR-0015 decision against backend application settings
- Breaks piece data completeness (pieces wouldn't contain their configuration)
- Creates complex synchronization between piece data and configuration state
- Inconsistent with established piece-as-complete-data architecture

### Alternative 3: Trait-Based Configuration Only

**Approach**: Configuration attributes defined only by traits, not specific models
**Rejection Reasons**:
- Many configuration attributes are model-specific (beam thickness only applies to staves)
- Creates artificial abstraction where none is needed
- Complicates default value resolution
- Reduces clarity of which entities support which settings

### Alternative 4: Two-Tier Attribute System

**Approach**: Separate macros and APIs for direct slots vs settings-backed attributes
**Rejection Reasons**:
- Violates CLAUDE.md uniform interface requirement
- Creates cognitive load for users (must know which API to use)
- Complicates VPD integration (different dispatch patterns)
- Reduces flexibility for migrating between storage approaches

## References

### Related ADRs
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural boundaries maintained by settings-in-piece-data approach
- [ADR-0008: Vector Path Descriptors](0008-VPDs.md) - VPD integration providing uniform access patterns
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) - Establishes that backend has no application settings, only piece data

### Technical Dependencies
- **Methodical**: Multimethod system providing polymorphic dispatch foundation
- **STM**: Transaction system for coordinated updates across entity hierarchy
- **VPD System**: Path descriptor system enabling uniform access patterns
- **Integrant**: Component system managing piece lifecycle and references

### Code References
- `backend/src/main/clojure/ooloi/backend/ops/vectors_and_attributes.clj`: Settings implementation including `defsetting` macro and helper functions
- `backend/src/main/clojure/ooloi/backend/models/core.clj`: VPD wrapper macros and dispatch generation
- `backend/test/clojure/ooloi/backend/ops/vectors_and_attributes_test.clj`: Comprehensive test coverage

## Notes

This architecture decision establishes the foundation for comprehensive configuration management within Ooloi's piece data while maintaining the storage efficiency critical for large musical compositions.

The approach leverages existing architectural patterns rather than introducing new abstractions, ensuring consistency with established development practices and performance characteristics.

The settings system applies exclusively to piece data entities, maintaining the clear separation established by ADR-0015 between musical content (backend responsibility) and application preferences (frontend responsibility).

Future enhancements may include:
- Plugin-defined settings for extensible customization
- Settings validation and constraints for data integrity
- Settings migration tools for version compatibility
- Advanced default value inheritance patterns
- Settings export/import for user preference management

The implementation should prioritize storage efficiency monitoring and comprehensive testing to validate the approach's effectiveness for large-scale musical compositions.