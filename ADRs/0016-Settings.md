# ADR-0016: Settings

## Status

Accepted - July 24, 2025
Updated - November 24, 2025 (Added validation support)
Updated - February 13, 2026 (Event emission for collaborative settings awareness)

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
- **Configuration Integrity**: Invalid settings values must be prevented at write time

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

We will implement a **Universal Entity Settings Architecture** that provides configuration capabilities for any entity in Ooloi's musical and visual hierarchies while maintaining storage efficiency, API consistency, and data integrity through validation.

### Core Architecture Principles

1. **Universal Applicability**: Any entity (Piece, Musician, Staff, Pitch, Layout, etc.) can have configuration settings
2. **Lazy Storage**: `:settings` maps created only when non-default values are set
3. **Unified API**: Consistent `get-<setting>`/`set-<setting>` functions regardless of storage mechanism
4. **Default Value System**: Systematic default value management with automatic cleanup
5. **VPD Integration**: Full Vector Path Descriptor support with transaction semantics
6. **Data Integrity**: Built-in validation prevents invalid settings values at write time

### Implementation Strategy

#### Settings-Backed Attribute Functions

New `defsetting` macro in `models.core` generates functions that look identical to existing attribute functions but use map-based storage with optional validation:

```clojure
;; Existing direct slot attribute
(defattribute Staff :name)          ; Direct field access
;; → get-name, set-name

;; New settings-backed attribute with no validation
(defsetting Staff :beam-thickness 0.5)  ; Map-based with default
;; → get-beam-thickness, set-beam-thickness

;; Settings with enumerated value validation
(defsetting Piece :keyless-accidentals :standard 
  #{:standard :all-except-repeated :all})

;; Settings with predicate validation  
(defsetting Piece :max-lookahead-measures 50 pos-int?)
(defsetting Piece :french-ties? false boolean?)
```

#### Validation Architecture

The validation system provides declarative constraints alongside setting definitions:

**Validator Types**:
- **No validator**: Defaults to `(constantly true)`, accepts any value
- **Set validator**: Validates using `contains?` for enumerated values
- **Function validator**: Validates by calling predicate function

**Validation Execution**:
- Validation occurs in `set-setting` before value storage
- Invalid values throw exceptions with clear error messages
- Set validators include valid values in error messages
- Validation is uniform regardless of direct or VPD usage

**Registry Storage**:

```clojure
;; Registry structure includes both default and validator
{::h/Piece 
 {:keyless-accidentals {:default :standard
                        :validator #{:standard :all-except-repeated :all}}
  :max-lookahead-measures {:default 50
                          :validator pos-int?}
  :french-ties? {:default false
                 :validator boolean?}}}
```

#### Usage with Full VPD Support

```clojure
;; Direct usage with automatic validation
(set-beam-thickness staff 0.8)

;; VPD usage with automatic STM transaction and validation
(set-beam-thickness [:musicians 0 :instruments 0 :staves 0] piece-ref 0.8)

;; Validation failure with clear error message
(set-keyless-accidentals piece :invalid-value)
;; => Exception: Invalid value for setting :keyless-accidentals: :invalid-value. 
;;    Must be one of: #{:standard :all-except-repeated :all}

;; Lazy storage - only non-default values stored
(get-beam-thickness staff)  ; Returns 0.5 (default) without storage overhead
```

### Storage Efficiency

- **Default Values**: Not stored in `:settings` maps, saving memory for 99.9% of entities
- **Lazy Maps**: `:settings` only created when first non-default value is set
- **Automatic Cleanup**: Setting a value to its default removes it from storage
- **Empty Map Removal**: Empty `:settings` maps are completely removed
- **Validator Storage**: Validators stored once in registry, not per-entity

### API Consistency

Users experience identical interfaces regardless of underlying storage or validation:

```clojure
;; Direct slot attributes (existing)
(get-name staff)              ; Direct field access
(set-name staff "Violin I")   ; Direct field update

;; Settings-backed attributes (new)
(get-beam-thickness staff)    ; Map access with default fallback
(set-beam-thickness staff 0.6) ; Map update with validation and cleanup logic
```

### Default Value Management

**Defaults Registry Architecture**: 

A central, dynamically-populated registry provides default value and validator storage with discoverability:

```clojure
;; Central registry - populated at compile-time by defsetting macro
(defonce defaults-registry (atom {}))
;; Structure: {Staff {:beam-thickness {:default 0.5 :validator (constantly true)}
;;                     :staff-spacing {:default 10.0 :validator pos?}}
;;            Measure {:width {:default 100.0 :validator pos-int?}}}
```

## Rationale

### Alignment with Existing Architecture

**Maintains ADR-0015 Boundaries**: This architecture applies only to piece data entities, not backend application settings. All settings travel with piece data and are serialized with pieces, maintaining the established separation.

**Leverages ADR-0001 Separation**: Settings remain part of musical content managed by the backend, while UI preferences continue to be handled by frontend clients.

**Uses Existing Infrastructure**: VPD dispatch, STM transactions, and multimethod patterns from the current architecture provide the foundation.

### Uniform Interface Achievement

**Single API Pattern**: All settings use identical `get-<setting>`/`set-<setting>` patterns regardless of storage mechanism or validation requirements, satisfying the uniform interface architectural requirements.

**No Cognitive Load**: Users don't need to know whether an attribute is a direct slot or settings-backed, or what validation applies - the API is identical and validation is automatic.

**Future-Proof**: New configuration attributes can be added without API changes or architectural decisions.

### Data Integrity

**Declarative Validation**: Constraints defined alongside default values, making requirements explicit and discoverable.

**Fail-Fast Behavior**: Invalid values rejected at write time with clear error messages, preventing invalid state propagation.

**Zero Runtime Overhead**: Validation logic generated once at compile time, stored in registry for runtime use.

**Consistent Error Messages**: Set validators automatically include valid values in error messages for better developer experience.

### Storage Optimization

**Memory Efficiency**: 99.9% of entities store no settings data, maintaining compact representation for large pieces with 100,000+ notes.

**Performance**: Direct map access for non-default values, with automatic fallback to defaults.

**Scalability**: Approach scales to any number of setting types without per-entity overhead.

**Validator Efficiency**: Single validator function stored in registry, not duplicated per entity.

### Development Experience

**Consistent Patterns**: Developers use the same macro and helper functions for all settings across all entity types.

**Automatic VPD Support**: Settings immediately work with Vector Path Descriptors and STM transactions without additional implementation.

**Clear Intent**: `defsetting` declarations make configuration attributes, defaults, and constraints explicit and documented.

**Reduced Boilerplate**: Validation defined inline with setting declaration, eliminating separate validation functions and :around methods.

## Implementation

The Universal Entity Settings architecture is implemented in `backend/src/main/clojure/ooloi/backend/ops/access.clj` with comprehensive test coverage in the corresponding test namespace.

Key implementation aspects:
- `defsetting` macro with optional validation parameter, runtime validation, and defaults registry integration
- `get-setting`/`set-setting` helper functions with validation, lazy storage, and cleanup
- Support for both set-based and predicate-based validation
- Full VPD integration with STM transaction support
- Metadata-based method categorization for dispatch system integration
- Comprehensive specs for all validation-related types

## Consequences

### Positive

1. **Uniform Configuration API**: Consistent interface across all entity types and setting types
2. **Storage Efficiency**: Minimal memory overhead for entities using default values
3. **Architectural Consistency**: Builds on existing VPD, STM, and multimethod patterns
4. **Development Velocity**: Easy addition of new configuration attributes without architectural decisions
5. **User Experience**: Predictable API behavior regardless of underlying storage mechanisms
6. **Future Extensibility**: Foundation for plugin-defined settings and advanced customization
7. **Data Integrity**: Invalid settings prevented at write time with clear error messages
8. **Declarative Constraints**: Validation requirements explicit and co-located with definitions
9. **Reduced Boilerplate**: Eliminated separate validation functions and :around methods
10. **Collaborative Awareness**: When a piece setting changes, the backend emits a piece-setting-changed event via the backend event router (ADR-0031). The frontend reacts only when a piece settings window is open — refreshing stale values so a collaborating user does not work on outdated settings. Visual consequences of piece setting changes (e.g. re-rendering after beam thickness changes) are handled separately by the backend's own cache-invalidation events, not by the settings event.

### Negative

1. **Implementation Complexity**: Additional abstraction layer over direct field access
2. **Performance Overhead**: Map lookups and validation for non-default values vs direct field access
3. **Default Value Architecture**: Additional architectural decision required for default management
4. **Testing Complexity**: Must verify both direct usage and VPD dispatch patterns, plus validation behavior
5. **Migration Effort**: Existing configuration attributes may need migration to new pattern
6. **Validation Performance**: Every setter call incurs validation overhead (mitigated by fast validator execution)

### Mitigations

1. **Performance Monitoring**: Measure and optimize map lookup and validation performance in realistic scenarios
2. **Incremental Implementation**: Deploy settings pattern incrementally, maintaining existing attributes
3. **Comprehensive Testing**: Automated testing for storage efficiency, API consistency, validation, and VPD integration
4. **Clear Documentation**: Guidelines for when to use settings vs direct slots, and how to define validators
5. **Developer Tools**: Tooling to verify storage optimization and detect configuration issues
6. **Validator Efficiency**: Use `(constantly true)` default to minimize overhead for unconstrained settings

## Alternatives Considered

### Alternative 1: Dedicated Slots for All Configuration

**Approach**: Add dedicated defrecord slots for each configuration attribute

**Rejection Reasons**:
- Violates uniform interface principle (different APIs for different attributes)
- Creates storage overhead for all entities regardless of usage
- Requires defrecord modifications for each new configuration attribute
- No systematic approach to default values or validation

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
- Complicates default value resolution and validation logic
- Reduces clarity of which entities support which settings

### Alternative 4: Two-Tier Attribute System

**Approach**: Separate macros and APIs for direct slots vs settings-backed attributes

**Rejection Reasons**:
- Violates CLAUDE.md uniform interface requirement
- Creates cognitive load for users (must know which API to use)
- Complicates VPD integration (different dispatch patterns)
- Reduces flexibility for migrating between storage approaches

### Alternative 5: Separate Validation Layer

**Approach**: Define validators separately from setting declarations, using :around methods

**Rejection Reasons**:
- Scatters validation logic away from setting definitions
- Requires verbose boilerplate (validation function, validation set, :around method)
- Disrupts reading flow when scanning setting declarations
- Makes it unclear which settings have validation and which don't
- More difficult to discover validation requirements programmatically

### Alternative 6: Runtime Spec Validation

**Approach**: Use clojure.spec for settings validation with instrumentation

**Rejection Reasons**:
- Requires separate spec definitions away from setting declarations
- Instrumentation overhead in production (or disabled validation if not instrumented)
- Less clear error messages compared to custom validation
- Overkill for simple enumeration and predicate validation
- Adds dependency on spec system for basic data integrity

## References

### Related ADRs
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Architectural boundaries maintained by settings-in-piece-data approach
- [ADR-0008: Vector Path Descriptors](0008-VPDs.md) - VPD integration providing uniform access patterns
- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) - Establishes that backend has no application settings, only piece data
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - Backend event router delivers piece-setting-changed events to the frontend event bus for collaborative settings awareness
- [ADR-0043: Frontend Settings](0043-Frontend-Settings.md) - Companion system for global application preferences (frontend-only); borrows validation patterns from this ADR

### Technical Dependencies
- **Methodical**: Multimethod system providing polymorphic dispatch foundation
- **STM**: Transaction system for coordinated updates across entity hierarchy
- **VPD System**: Path descriptor system enabling uniform access patterns
- **Integrant**: Component system managing piece lifecycle and references

### Code References
- `backend/src/main/clojure/ooloi/backend/ops/access.clj`: Settings implementation including `defsetting` macro, validation logic, and helper functions
- `backend/src/main/clojure/ooloi/backend/models/core.clj`: VPD wrapper macros and dispatch generation
- `backend/test/clojure/ooloi/backend/ops/access_test.clj`: Comprehensive test coverage including validation scenarios

## Notes

This architecture decision establishes the foundation for comprehensive configuration management within Ooloi's piece data while maintaining the storage efficiency critical for large musical compositions and ensuring data integrity through declarative validation.

The approach leverages existing architectural patterns rather than introducing new abstractions, ensuring consistency with established development practices and performance characteristics.

The settings system applies exclusively to piece data entities, maintaining the clear separation established by ADR-0015 between musical content (backend responsibility) and application preferences (frontend responsibility).

The validation system eliminates the verbose pattern of separate validation functions and :around methods, replacing it with inline declarative validation that makes constraints explicit and discoverable. This reduces code volume while improving clarity and maintainability.

Future enhancements may include:
- Plugin-defined settings for extensible customization
- Advanced validation combinators for complex constraints
- Settings migration tools for version compatibility
- Advanced default value inheritance patterns
- Settings export/import for user preference management
- Validation error recovery and suggestion mechanisms

The implementation should prioritize storage efficiency monitoring, validation performance testing, and comprehensive testing to validate the approach's effectiveness for large-scale musical compositions.