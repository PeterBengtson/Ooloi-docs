# Ooloi Shared Source Directory

This directory contains the **complete Ooloi system** - all data models, business logic, API functions, operations, traits, and interfaces used by both frontend and backend.

## Table of Contents

- [Directory Structure](#directory-structure)
  - [Key Directories](#key-directories)
- [Data Model](#data-model)
- [Class-specific Operations](#class-specific-operations)
- [Programming Paradigm](#programming-paradigm)
  - [Key Concepts](#key-concepts)
- [Constructors, Accessors and Mutators](#constructors-accessors-and-mutators)
  - [Constructors](#constructors)
  - [Accessors and Mutators](#accessors-and-mutators)
- [Core and API Polymorphism](#core-and-api-polymorphism)
- [Parallelism and Thread-Safety](#parallelism-and-thread-safety)
- [Principles and Requirements](#principles-and-requirements)
- [Namespace Organization and Import Guidelines](#namespace-organization-and-import-guidelines)
  - [Quick Start: What to Import](#quick-start-what-to-import)
  - [Most Important Namespaces](#most-important-namespaces)
  - [Frontend and Backend Usage](#frontend-and-backend-usage)
  - [Summary: What Most Developers Need to Know](#summary-what-most-developers-need-to-know)
  - [Shared Namespace Architecture](#shared-namespace-architecture)
  - [Namespace Descriptions](#namespace-descriptions)
  - [Import Guidelines by Usage Pattern](#import-guidelines-by-usage-pattern)
  - [Decision Tree: Which Namespace to Use?](#decision-tree-which-namespace-to-use)
  - [Key Benefits of This Architecture](#key-benefits-of-this-architecture)
  - [Common Mistakes to Avoid](#common-mistakes-to-avoid)
- [Unified gRPC Architecture](#unified-grpc-architecture)
- [Coding Conventions](#coding-conventions)

## Directory Structure

```
shared/src/main/clojure/ooloi/shared/
├── build.clj                 ; Build utilities and packaging
├── core.clj                  ; Combined application entry point
├── hierarchy.clj             ; Shared type hierarchy and dispatch values
├── interfaces.clj            ; Shared multimethod interface contracts
├── predicates.clj            ; Shared predicate functions (musical?, visual?, etc.)
├── proto_conversion.clj      ; Clojure ↔ Protocol Buffer conversion utilities
├── models/                   ; **Complete Data Model**
│   ├── core.clj              ; Model core and macro system
│   ├── changes.clj           ; ChangeSet data structure for time sigs, key sigs, tempos
│   ├── musical/              ; Musical data models
│   │   ├── piece.clj         ; Piece - root musical container
│   │   ├── musician.clj      ; Musician - performer representation
│   │   ├── instrument.clj    ; Instrument - musical instrument definition
│   │   ├── staff.clj         ; Staff - staff representation
│   │   ├── voice.clj         ; Voice - voice within staff
│   │   ├── measure.clj       ; Measure - musical measure
│   │   ├── pitch.clj         ; Pitch - musical pitch
│   │   ├── chord.clj         ; Chord - multiple simultaneous pitches
│   │   ├── rest.clj          ; Rest - musical rest
│   │   ├── tuplet.clj        ; Tuplet - rhythmic grouping
│   │   ├── tremolando.clj    ; Tremolando - rapid repetition
│   │   └── attachments/      ; Musical attachments
│   │       ├── articulation.clj ; Articulation markings
│   │       ├── dynamic.clj   ; Dynamic markings (forte, piano, etc.)
│   │       ├── glissando.clj ; Glissando markings
│   │       ├── hairpin.clj   ; Hairpin crescendo/diminuendo
│   │       ├── ottava.clj    ; Octave displacement markings
│   │       ├── slur.clj      ; Slur markings
│   │       └── tie.clj       ; Tie markings
│   └── visual/               ; Visual layout models
│       ├── layout.clj        ; Overall layout configuration
│       ├── page_view.clj     ; Page layout view
│       ├── system_view.clj   ; System layout view
│       ├── staff_view.clj    ; Staff layout view
│       └── measure_view.clj  ; Measure layout view
├── ops/                      ; **Complete Operations**
│   ├── access.clj            ; Vector/attribute operations
│   ├── attachment_resolver.clj ; Attachment resolution operations
│   ├── changes.clj           ; Change-based operations
│   ├── persistence.clj       ; Persistence operations
│   ├── piece_ref.clj         ; Piece reference operations
│   ├── pitches.clj           ; Pitch normalization and conversion utilities
│   ├── rhythm.clj            ; Duration and rhythm processing utilities
│   ├── text.clj              ; Text processing (pluralization, singularization)
│   ├── timewalk.clj          ; Musical structure traversal and analysis operations
│   └── vpd.clj               ; VPD operations and addressing utilities
├── specs/                    ; Test data generation
│   └── generators.clj        ; Test data generators for all models
└── traits/                   ; **All Behavioral Traits**
    ├── attachment.clj        ; Attachment behavior trait
    ├── has_items.clj         ; Collection behavior trait
    ├── rhythmic_item.clj     ; Rhythmic behavior trait
    ├── takes_attachment.clj  ; Attachment acceptance trait
    └── transposable.clj      ; Transposition behavior trait
```

**⚡ CRITICAL**: This IS the complete Ooloi system. Frontend and backend are consumers that use this unified data model and API - no separate representations exist.

### Key Directories

**Core System Files:**
- `core.clj` - Combined application entry point 
- `hierarchy.clj` - Shared type hierarchy and dispatch values
- `interfaces.clj` - Complete multimethod interface definitions
- `predicates.clj` - All type checking functions

**Complete Data Model:**
- `models/` - All Ooloi data structures (musical and visual)
  - `core.clj` - Model core system and macro generation
  - `musical/` - Complete musical data model (piece, musician, instrument, etc.)
  - `visual/` - Complete visual layout model (layout, views, etc.)

**Complete Operations:**
- `ops/` - All Ooloi operations and algorithms
  - `timewalk.clj` - Musical structure traversal (moved from backend)
  - `attachment_resolver.clj` - Attachment resolution (moved from backend)  
  - All other operations used by both frontend and backend

**System Infrastructure:**
- `proto_conversion.clj` - gRPC conversion utilities
- `specs/generators.clj` - Test data generation for all models
- `traits/` - All behavioral mixins used across the system

## Data Model

Ooloi uses the following unified tree structure for all musical elements (used identically by frontend and backend):

```
Piece
├── musicians
│   └── instruments
│       └── staves
│           └── voices
│               └── measures
│                   └── items (Pitch, Chord, Rest, Tuplet, Tremolando, etc.)
└── layouts
    └── page-views
        └── system-views
            └── staff-views
                └── measure-views
                    ├── glyphs
                    └── curves
```

The entire structure is always a pure tree, making serialization and deserialization straightforward. Cross-references are implemented as ID references, not pointers. IDs are lazily assigned to objects as needed.

**⚡ UNIFIED ARCHITECTURE**: Frontend and backend use this IDENTICAL data model - no separate representations, no conversion layers.

## Class-specific Operations

Each model file contains the most basic operations for that model, such as adding and deleting nested items, e.g., to add a Layout to a Piece. The focus is on brevity and understandability.

More complex or abstract operations, such as formatting a MeasureView for display or traversing musical structures, are placed in the `ops/` directory to keep model files succinct and understandable.

## Programming Paradigm

Ooloi uses a dynamic, functional programming approach to handle complex music notation requirements. At its core, it leverages Clojure's functional programming paradigms, enhanced with the Methodical library, to provide a powerful and flexible system for music notation and manipulation.

### Key Concepts

1. **Functional Programming:** Ooloi uses Clojure to ensure immutability, ease of reasoning about code, and powerful data structures.
2. **Multimethods:** Multimethods provide polymorphic dispatch based on types and other attributes, allowing different behaviors for different types.
3. **Methodical Library:** Methodical extends Clojure's multimethods, providing advanced features such as next-method calls, auxiliary methods (`:before`, `:after`, `:around`), and more.
4. **Clojure Hierarchies:** Hierarchies are used, rather than inheritance, for composability and multiple inheritance reminiscent of CLOS.
5. **Specter Library:** Uses Specter for efficient and expressive updates of arbitrarily nested data structures.

## Constructors, Accessors and Mutators

### Constructors

Constructors are functions that create instances of records with default or specified values. They ensure consistency and encapsulate the creation logic for records. Each model in Ooloi has an associated constructor function.

For the `Pitch` model, the constructor function is `create-pitch`. To create a `Pitch` instance:

```clojure
(create-pitch :note "C4" :duration 1/4)
```

Similar `create-xxxxx` functions are available for all models. **These functions work identically in frontend and backend.**

### Accessors and Mutators

Automatically defined accessors and mutators retrieve and modify attributes of defrecords. For attributes that contain vectors or ChangeSets, standardized accessors and mutators are also available to manipulate their elements. There's a consistent naming scheme:

Simple attributes have the following two methods defined. For an attribute named `start-measure-number`:
- `get-start-measure-number`
- `set-start-measure-number`

Vector attributes have the following methods defined. For an attribute named `staff`:
- `add-staff`
- `get-staff`
- `get-staves`
- `move-staff-down`
- `move-staff-up`
- `remove-staff`
- `set-staff`
- `set-staves`

ChangeSets (used for time signatures, key signatures, tempo changes) have these methods. For an attribute called `tempos`:
- `get-tempo`
- `get-tempos`
- `remove-tempo`
- `set-tempo`
- `set-tempos`

**⚡ UNIFIED API**: All accessors and mutators work identically in frontend and backend - same functions, same data structures, same everything.

## Core and API Polymorphism

The unified system results in clear, succinct code that works identically across frontend and backend:

```clojure
(let [measure1 (-> (create-measure)
                   (add-item (create-pitch :note "C4" :duration 1/4))   ; Twink-
                   (add-item (create-pitch :note "C4" :duration 1/4))   ; -le,
                   (add-item (create-pitch :note "G4" :duration 1/4))   ; twink-
                   (add-item (create-pitch :note "G4" :duration 1/4)))  ; -le

      measure2 (-> (create-measure)
                    (add-item (create-pitch :note "A4" :duration 1/4))  ; litt-
                    (add-item (create-pitch :note "A4" :duration 1/4))  ; -le
                    (add-item (create-pitch :note "G4" :duration 1/2))) ; star

      voice (-> (create-voice :start-measure-number 0)
                (add-measure measure1)
                (add-measure measure2))

      staff (-> (create-staff)
                (add-voice voice))

      instrument (-> (create-instrument :name "Oboe")
                     (add-staff staff))

      musician (-> (create-musician :name "Oboe 1")
                   (add-instrument instrument))

      piece (-> (create-piece)
                (add-musician musician)
                (set-time-signature 0 [4 4])
                (set-key-signature 0 [:c :major]))]

    ; This code works IDENTICALLY in frontend and backend
    ...)
```

**Local Usage Pattern** (direct object manipulation):
- Frontend can use this pattern for local data manipulation
- Backend can use this pattern for direct object operations
- Same functions, same data structures, same results

**Remote Usage Pattern** (VPD addressing):
For remote access (frontend → gRPC → backend), the same operations can use Vector Path Descriptors:

```clojure
;; Local: Direct object reference
(remove-item measure 2)

;; Remote: VPD reference to same operation
(remove-item [:musicians 0 :instruments 0 :staves 0 :voices 0 :measures 1] "PieceID2418" 2)
```

The VPD compact form:
```clojure
(remove-item [:m 0 0 0 0 1] "PieceID2418" 2)
```

**⚡ UNIFIED SYSTEM**: Same API functions work locally (direct objects) and remotely (VPD + piece-id). Frontend can use both patterns depending on deployment mode.

## Unified gRPC Architecture

The shared system provides the complete gRPC infrastructure for frontend-backend communication:

### Universal ExecuteMethod Pattern
- **Single endpoint**: `ExecuteMethod` handles all API calls via dynamic resolution
- **Perfect fidelity**: `OoloiValue` preserves all Clojure data types automatically
- **Zero maintenance**: New API methods work immediately without regeneration

### Consumer Relationship
```
SHARED (Complete Ooloi System)
├── Frontend Local Usage
│   (create-pitch "C4" "1/4")           ← SAME function as backend
│   (pitch? some-pitch)                ← SAME predicate as backend
│
├── Frontend Remote Usage (via gRPC)
│   Frontend ──gRPC──→ Backend ──delegates──→ SAME shared functions
│   (grpc-call :create-pitch {...})     ← Returns SAME Pitch record
│
└── Backend Usage
    (create-pitch "C4" "1/4")           ← IDENTICAL to frontend
    (pitch? some-pitch)                ← IDENTICAL to frontend

⚡ KEY: Same data structures, same functions, same everything!
```

## Parallelism and Thread-Safety

Ooloi utilizes Clojure's Software Transactional Memory (STM) with refs to provide a powerful, thread-safe framework for accessing and mutating complex musical structures. This system forms the foundation of all data operations.

Key aspects:
1. All piece data is stored in refs, allowing for ACID transactions.
2. Updates to pieces are wrapped in dosync blocks, ensuring atomicity and consistency.
3. Automatic retries are handled by the STM when conflicts occur in concurrent modifications.

The system provides:
- Functions for safely updating data within the transactional system.
- Functions to mark measures for later visual recomputation.

Performance metrics (on a 2017 MacBook Pro 2,2 GHz 6-Core Intel Core i7):
- Transactional mode: 100,000+ updates per second (1000 updates in 10 ms).

## Principles and Requirements

Overall Principles and Requirements:
- Maintain polymorphism across all models.
- Keep model-specific definitions close to their respective model files.
- Follow Clojure best practices and idiomatic patterns.
- Prioritize simplicity and avoid unnecessary complexity.
- Design for extensibility and ease of use.
- Maintain separation between musical and visual models.
- Use docstrings everywhere. The documentation tool (Codox) uses them.
- Use STM for managing concurrency, ensuring thread-safe operations and data integrity.
- **Unified data model**: Frontend and backend use identical data structures - no separate representations.

## Namespace Organization and Import Guidelines

Ooloi's shared system provides a carefully structured namespace organization for the complete system used by both frontend and backend.

### Quick Start: What to Import

**For 90% of your code (both frontend and backend), you only need this:**

```clojure
(ns my-namespace
  (:require [ooloi.shared.models.core :refer :all]))
```

This gives you access to:
- **All constructors**: `create-pitch`, `create-chord`, `create-measure`, etc.
- **All predicates**: `pitch?`, `chord?`, `measure?` (raw items), plus `pitch??`, `chord??`, `measure??` (timewalk tuples)
- **All multimethods**: `get-duration`, `add-item`, `set-name`, etc.

**Why this works**: The shared `core` namespace IS the complete Ooloi system used identically by frontend and backend.

**Alternative selective imports (for specific use cases):**

```clojure
(ns my-namespace
  (:require [ooloi.shared.models.musical.pitch :refer [create-pitch]]
            [ooloi.shared.interfaces :as i]
            [ooloi.shared.predicates :as p]))
```

### Most Important Namespaces

#### 1. `ooloi.shared.models.*` - **DATA MODEL CONSTRUCTORS**
- **What it is**: Complete Ooloi data model constructors
- **What it contains**: `create-pitch`, `create-piece`, `create-musician`, etc.
- **When to use**: When creating new musical or visual data structures
- **Frontend usage**: ✅ Same functions as backend
- **Backend usage**: ✅ Same functions as frontend

#### 2. `ooloi.shared.predicates` - **TYPE CHECKING**
- **What it is**: All type checking functions
- **What it contains**: `pitch?`, `chord?`, `measure?` (raw items) and `pitch??`, `chord??` (timewalk tuples)
- **When to use**: For type checking and validation
- **Frontend usage**: ✅ Same predicates as backend
- **Backend usage**: ✅ Same predicates as frontend

#### 3. `ooloi.shared.interfaces` - **UNIFIED API**
- **What it is**: Complete multimethod interface definitions
- **What it contains**: `get-duration`, `add-item`, `set-name`, etc.
- **When to use**: For all data manipulation operations
- **Frontend usage**: ✅ Same API as backend (via local calls or gRPC)
- **Backend usage**: ✅ Same API as frontend

### Frontend and Backend Usage

**Frontend Consumer Pattern:**
```clojure
(ns my-frontend-component
  (:require [ooloi.shared.models.musical.pitch :refer [create-pitch]]  ; Same as backend
            [ooloi.shared.interfaces :as i]                           ; Same as backend
            [ooloi.shared.predicates :as p]))                         ; Same as backend

;; Creates identical objects to backend
(def my-pitch (create-pitch "C4" "1/4"))  ; Same function, same Pitch record
(p/pitch? my-pitch)                       ; Same predicate as backend
(i/get-duration my-pitch)                 ; Same interface as backend
```

**Backend Consumer Pattern:**
```clojure
(ns my-backend-service
  (:require [ooloi.shared.models.musical.pitch :refer [create-pitch]]  ; SAME as frontend
            [ooloi.shared.interfaces :as i]                           ; SAME as frontend  
            [ooloi.shared.predicates :as p]))                         ; SAME as frontend

;; Creates identical objects to frontend
(def my-pitch (create-pitch "C4" "1/4"))  ; SAME function, SAME Pitch record
(p/pitch? my-pitch)                       ; SAME predicate as frontend
(i/get-duration my-pitch)                 ; SAME interface as frontend
```

### Summary: What Most Developers Need to Know

1. **Use shared namespaces**: Import from `ooloi.shared.*` for all functionality
2. **Same everywhere**: Frontend and backend use identical imports and functions  
3. **No separate representations**: One data model, used identically by all projects
4. **Local and remote**: Same API works locally (direct calls) and remotely (gRPC)

### Shared Namespace Architecture

The unified architecture provides a clean structure:

```
shared/predicates.clj → shared/interfaces.clj → shared/models/*.clj → shared/ops/*.clj
                                    ↓
                        Frontend and Backend (identical usage)
```

### Decision Tree: Which Namespace to Use?

1. **Creating data structures?**
   → Use `[ooloi.shared.models.* :refer [create-*]]`

2. **Type checking?**
   → Use `[ooloi.shared.predicates :as p]`

3. **Data manipulation?**
   → Use `[ooloi.shared.interfaces :as i]`

4. **Complex operations?**
   → Use `[ooloi.shared.ops.* :as *]`

5. **Are you frontend or backend?**
   → **Doesn't matter** - use the same shared namespaces!

### Key Benefits of This Architecture

- **Unified Data Model**: Same records, same functions, same everything
- **No Conversion**: Frontend objects ARE backend objects
- **Perfect Fidelity**: `(= frontend-object backend-object) => true`
- **Deployment Flexibility**: Local usage OR remote gRPC, same API
- **Plugin Ecosystem**: Plugins work identically in frontend and backend
- **Zero Maintenance**: No protocol buffer regeneration workflows

### Common Mistakes to Avoid

- **Don't create separate frontend/backend data representations**: Use shared models only
- **Don't convert between "frontend" and "backend" objects**: They're the same objects
- **Don't assume frontend needs different APIs**: Use the same shared interfaces
- **Don't rebuild functionality in projects**: Everything is in shared

## Coding Conventions

- Use Methodical (`m/defmulti` and `m/defmethod`) for defining and implementing multimethods.
- Using Specter is highly encouraged for nested data structure updates.
- Define shared behaviors in `models/core.clj`.
- Implement model-specific behaviors in individual model files.
- Use docstrings everywhere for collaborative development.
- **Unified development**: Write code that works identically in frontend and backend.