# ADR-0016: Settings

## Status

Implemented

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
- Some attributes are currently direct defrecord slots (`:name`, `:clefs`, `:num-lines`)
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

The `defsetting` macro (`ops/access.clj`, alongside `defattribute`) generates functions that look identical to existing attribute functions but use map-based storage with optional validation:

```clojure
;; Existing direct slot attribute — dispatches on the RECORD
(defattribute Staff :name)          ; Direct field access
;; → get-name, set-name

;; Settings-backed attribute — dispatches on the HIERARCHY KEYWORD
(defsetting ::h/Staff :beam-thickness 0.5)  ; Map-based with default
;; → get-beam-thickness, set-beam-thickness

;; Settings with enumerated value validation
(defsetting ::h/Piece :keyless-accidentals :standard
  #{:standard :all-except-repeated :all})

;; Settings with predicate validation
(defsetting ::h/Piece :max-lookahead-measures 50 pos-int?)
(defsetting ::h/Piece :french-ties? false boolean?)
```

**Note the asymmetry in the first argument, because it is easy to get wrong and fails loudly.**
`defattribute` takes the **record** (`Staff`); `defsetting` takes the **hierarchy keyword**
(`::h/Staff`) and *throws* on anything else — it requires a keyword in the
`ooloi.shared.hierarchy` namespace. The difference is not cosmetic: a hierarchy keyword is what
lets a setting be declared against a **trait** as well as a concrete type, so
`(defsetting ::h/Structural :beam-thickness 0.5)` would install the accessors on Piece, Musician,
Instrument, Staff and Layout from one declaration. A record name cannot express that.

The examples above show the macro's **current** arity. They do not show the mandatory category
described below, because the form that argument takes is not yet settled — the validator occupies
the optional fourth position and a Clojure map is itself callable as a predicate, so an options map
placed there would be silently read *as* the validator. The category therefore needs an unambiguous
position or shape, and this ADR specifies the requirement without yet fixing the syntax.

**Category**: every setting declares the category it belongs to, and declaring it is **mandatory** —
`defsetting` throws if it is missing, so a setting cannot come into existence without a place to
appear. The category is what a settings window groups by, one tab per category, and it is declared
rather than inferred because a setting's public interface is generated from its *name*:
`:keyless-accidentals` becomes the `get-keyless-accidentals` and `set-keyless-accidentals` that
clients call. Carrying the grouping in the name instead would corrupt both, giving names that no
longer read as a complete thought where they are called, and groups that depend on a spelling
convention holding. The category is therefore separate data, recorded in the defaults registry
beside the default and the validator.

Requiring it rather than allowing it to default is deliberate. An optional category admits settings
that belong nowhere, and a window must then invent a home for them — a residual tab that accretes
whatever nobody troubled to classify, which is precisely the outcome grouping exists to prevent.
Asking the question at the point of declaration is the cheaper discipline: whoever adds a setting
knows where it belongs, and no window has to guess.

**Within a tab, settings appear in declaration order** — the order in which the `defsetting` forms
are read, neither alphabetical nor incidental — so the author of a group controls how it reads. The
defaults registry must therefore preserve that order.

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

A central, dynamically-populated registry provides default value, validator and category storage with discoverability:

```clojure
;; Central registry - populated at load time by the defsetting macro
(defonce defaults-registry (atom {}))
;; Keyed by the HIERARCHY KEYWORD the setting was declared against — the macro's
;; first argument — never by the record:
;; {::h/Staff {:beam-thickness {:default 0.5 :validator (constantly true)}
;;             :staff-spacing  {:default 10.0 :validator pos?}}
;;  ::h/Measure {:width {:default 100.0 :validator pos-int?}}
;;  ::h/Piece {:keyless-accidentals {:default :standard
;;                                   :validator #{:standard :all-except-repeated :all}
;;                                   :category :accidentals}}}
```

The category rides in the same entry as the default and the validator rather than in a structure of
its own, so a settings window has everything it needs to build itself — what the setting is, what it
may be set to, and which tab it belongs on — from one lookup. Every entry carries one; there is no
uncategorised case to handle.

The registry must also **preserve declaration order**, since that is the order in which a tab
presents its settings. A plain map does not, so the registry is ordered by construction rather than
by a separate index kept alongside it — the frontend settings registry solves the identical problem
by holding a vector of pairs, and re-declaring an existing key replaces its entry in place so that
reloading a namespace does not reorder what it declares.

#### Scope of a Setting

A setting is **strictly local to the entity it is set on**. Reading consults that entity's own
`:settings` map and falls back to the declaration default; it does not consult enclosing entities.
Setting beam thickness on a musician therefore does not change what an enclosed staff reports — the
staff returns the declaration default until a value is set on the staff itself. VPD access behaves
identically, resolving the addressed entity and reading it locally. Note also that a default belongs
to the entity type it was declared for, not to the setting: a setting declared for staves has no
meaning on a piece, and reading it there is a dispatch failure rather than a fallback.

Hierarchical defaulting — an unset entity inheriting from its container, ending at the piece — is a
natural extension of this design and is deliberately not part of it yet. Two properties would have
to be revisited first, recorded here so the possibility stays open:

- **Storing a value equal to the default removes it.** The cleanup that keeps `:settings` maps small
  leaves "explicitly set" and "never set" indistinguishable in the data. Under inheritance those are
  different intentions — pinning a value so that a later change to an enclosing entity does not move
  it is exactly what the cleanup erases — so reading would need three states where the storage
  affords two, at some cost to the efficiency this ADR prizes.
- **Entities hold no reference to their container.** A cascade can therefore only be driven where
  the path is known, which is VPD access; a direct call on an entity object could not participate.
  The two forms agree exactly today, and inheritance would separate them unless every direct caller
  moved to VPD access.

#### Which settings reach the Piece Window

Whether an entity's settings appear in the structural projection, and whether writing one notifies
subscribers, is decided by that **entity's** `non-structural-fields` declaration and not by the
setting ([ADR-0052](0052-Change-Detection-and-Event-Generation.md) §3a/§3b). The two follow together
by construction: the same declaration the projection strips is the one whose complement fires
`:piece-structure-changed`, so a setting cannot be visible to the window without its change being
announced, nor announced without being visible.

Piece settings are therefore projected, because some of them decide what the Piece Window *displays* —
the numeral settings of [ADR-0054](0054-Automatic-Semantic-Naming-and-Numbering-of-Musicians-and-Instruments.md)
§6 govern every musician header and instrument row in it. Settings on lower entities are not, because
none of them currently does.

**The projection must be dense, and sparse storage is what forces it.** Storage omits any setting at
its default, and removes the `:settings` map once it empties — so a piece whose settings are all
default has no `:settings` key at all, and a setting returned to its default becomes
indistinguishable from one never touched. Projected raw, that tells a reader nothing: absence would
have to be resolved against the declarations, and a client resolving it against *its own* copy of
them would render from a default its build happened to hold rather than the one the piece is actually
governed by. So the projection carries **every declared setting of that entity at its effective
value** — stored value where present, declared default where absent — resolved on the authoritative
side, from the registry, at projection time. This is also consistent by construction: the effective
value is computed by the same stored-else-default rule the getters use, so the projection and
`get-<setting>` cannot disagree.

The shape is a map keyed by setting, each entry a map of its own:

```clojure
{:settings {:numbering-form      {:value :arabic  :default :arabic}
            :numbering-placement {:value :follow  :default :precede}}}
```

`:default` costs nothing to include — densification has just read it to compute the effective value —
and it earns two things. It makes "has this piece overridden the setting?" answerable from the
projection alone, and it puts the **server's** notion of the default behind the Preferences window's
per-field reset, which both decides whether that control appears and supplies the value it writes. A
client built against an older default would otherwise offer to reset a value that already *is* the
current default, and write one the server considers an override.

The nesting is what allows that, and it will earn its keep again: hierarchical defaulting needs a
further key — which level the effective value came from, the third state sparse storage cannot
express. Ordering within the map is irrelevant; order is a property of the declarations.

**The projection describes a setting's state; the registry describes how to present it.** From the
projection a client learns what a setting *is set to* and what it *would be* unset — enough to render
the value and to know whether it has been overridden, with nothing else consulted. From the registry,
which is on its classpath already, it learns category, declaration order, labels, control type and
validator. Only state crosses the wire; presentation never needs to.

**There is a second reason, and it is this ADR's own.** A setting is required to be
indistinguishable from a direct slot — "Users experience identical interfaces regardless of underlying
storage", "users don't need to know whether an attribute is a direct slot or settings-backed", and
Alternative 4 was rejected precisely for splitting the two. But a write to a direct slot of a
structural entity *announces itself*: `set-title` and `set-name` emit `:piece-structure-changed`
([ADR-0052](0052-Change-Detection-and-Event-Generation.md) §3b). So while a settings write stayed
silent, a caller promised an identical interface met a difference in observable consequence — the very
distinguishability this ADR exists to prevent. Making `:settings` structural on the Piece was
therefore required by the uniformity principle, independently of what any window needs.

**That reasoning does not stop at the Piece.** A Staff's `:name` emits, so a Staff's settings should
too, and until they do the principle is unsatisfied there. It is unsatisfied only *potentially*: no
entity below the Piece declares a setting today, so nothing yet exists to be distinguishable from.
The trigger for revisiting is therefore narrower than it looks — not "when a lower setting affects the
Piece Window" but **when any lower entity gains a setting at all**, since at that moment it becomes
distinguishable from its own slots whether or not anything renders from it.

At that moment uniformity and precision will pull against each other: uniformity says emit, while
nothing may consume the event. The question is vacuous today and is left for then, recorded rather
than answered. What must not be forgotten is that it arrives with the first lower-entity setting, and
that hierarchical defaulting and projection have to land together — one keyword per entity, small, but
not automatic.

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

**Zero Runtime Overhead**: The generated setter **closes over** its validator and its default directly, so validating a write consults no registry and performs no lookup. The registry holds the same values for *discovery* — a settings window enumerating what exists, and the projection densifying a piece's settings (ADR-0052 §3a) — not for the write path.

**Consistent Error Messages**: Set validators automatically include valid values in error messages for better developer experience.

### Storage Optimization

**Memory Efficiency**: 99.9% of entities store no settings data, maintaining compact representation for large pieces with 100,000+ notes.

**Performance**: Direct map access for non-default values, with automatic fallback to defaults.

**Scalability**: Approach scales to any number of setting types without per-entity overhead.

**Validator Efficiency**: One validator per setting, closed over by the generated setter and recorded once in the registry — never duplicated per entity.

### Development Experience

**Consistent Patterns**: Developers use the same macro and helper functions for all settings across all entity types.

**Automatic VPD Support**: Settings immediately work with Vector Path Descriptors and STM transactions without additional implementation.

**Clear Intent**: `defsetting` declarations make configuration attributes, defaults, and constraints explicit and documented.

**Reduced Boilerplate**: Validation defined inline with setting declaration, eliminating separate validation functions and :around methods.

## Implementation

The Universal Entity Settings architecture is implemented in `shared/src/main/clojure/ooloi/shared/ops/access.clj` with comprehensive test coverage in the corresponding test namespace.

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
10. **Collaborative Awareness**: A setting write notifies on two channels, and neither does the other's work:
    - `:piece-structure-changed` — because `:settings` is a structural slot, the write announces itself exactly as a rename does, and **every window subscribed to that piece** refetches the snapshot in which the values ride ([ADR-0052](0052-Change-Detection-and-Event-Generation.md) §3a). One event serves every consumer of a setting's value: a Piece Window's numeral labels and a Piece Preferences window's own controls read the same projected entries, so a collaborator's write cannot leave either showing a value the other has already been told has changed.
    - cache invalidation — the visual consequences (re-rendering after a beam-thickness or music-font change) travel out of the formatting pipeline, and only this channel reaches the paintlists.

    A third, payload-bearing per-setting channel was considered and rejected. It would have carried the changed key and its new value so a Preferences window could refresh one control without a refetch — but the refetch happens regardless, driven by the same write, and the projection it returns already carries every setting's value, so the second event would have delivered more precisely what the first had already brought. The cost of the coarser channel is over-signalling, accepted deliberately: a write to any setting refetches the whole projection, which is cheap because piece settings change rarely and the projection carries no measures, voices or items.

    The first channel concerns *what the piece is*, the second *what the music looks like* — the division [ADR-0052](0052-Change-Detection-and-Event-Generation.md) §6 draws for change detection generally. Settings sit deliberately across that line: they are part of what a piece *is*, and many also govern how it is *drawn*. The Piece Preferences window that presents these settings is specified in [ADR-0053](0053-Piece-Window-and-Piece-Preferences.md) §6.

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
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) - How a settings write reaches other clients: `:piece-structure-changed` delivered to that piece's subscribers, and why no separate per-setting event exists
- [ADR-0043: Frontend Settings](0043-Frontend-Settings.md) - Companion system for global application preferences (frontend-only); borrows validation patterns from this ADR
- [ADR-0053: The Piece Window and Piece Preferences](0053-Piece-Window-and-Piece-Preferences.md) - The Piece Preferences window presents these piece settings (§6)
- [ADR-0054: Automatic Semantic Naming and Numbering of Musicians and Instruments](0054-Automatic-Semantic-Naming-and-Numbering-of-Musicians-and-Instruments.md) - Declares three numeral piece settings (form, placement, full stop) that compose a stored number into its displayed label

### Technical Dependencies
- **Methodical**: Multimethod system providing polymorphic dispatch foundation
- **STM**: Transaction system for coordinated updates across entity hierarchy
- **VPD System**: Path descriptor system enabling uniform access patterns
- **Integrant**: Component system managing piece lifecycle and references

### Code References
- `shared/src/main/clojure/ooloi/shared/ops/access.clj`: Settings implementation including `defsetting` macro, validation logic, and helper functions
- `shared/src/main/clojure/ooloi/shared/models/core.clj`: VPD wrapper macros and dispatch generation
- `shared/test/clojure/ooloi/shared/ops/access_test.clj`: Comprehensive test coverage including validation scenarios

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