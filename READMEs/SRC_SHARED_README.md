# Ooloi Shared Source Directory

This directory contains the **complete Ooloi system** - all data models, business logic, API functions, operations, traits, and interfaces used by both frontend and backend.

## Table of Contents

- [Directory Structure](#directory-structure)
  - [Key Directories](#key-directories)
- [Data Model](#data-model)
- [Namespace Organization: How to Import](#namespace-organization-how-to-import)
  - [Quick Start](#quick-start)
  - [Most Important Namespaces](#most-important-namespaces)
  - [Frontend and Backend Usage](#frontend-and-backend-usage)
  - [Decision Tree: Which Namespace to Use?](#decision-tree-which-namespace-to-use)
- [Unified gRPC Architecture](#unified-grpc-architecture)
- [Deeper Topics](#deeper-topics)
- [Coding Conventions](#coding-conventions)

## Directory Structure

```
shared/src/main/clojure/ooloi/shared/
├── api.clj                   ; Public API namespace (external consumers)
├── build.clj                 ; Build utilities and packaging
├── core.clj                  ; Complete system namespace (internal usage)
├── hierarchy.clj             ; Shared type hierarchy and dispatch values
├── interfaces.clj            ; Shared multimethod interface contracts
├── platform.clj              ; Platform-specific utilities (JVM, native, etc.)
├── predicates.clj            ; Shared predicate functions (musical?, visual?, etc.)
├── srv_client.clj            ; Server/client utilities for testing
├── grpc/                     ; gRPC infrastructure
│   ├── conversion.clj        ; Clojure ↔ Protocol Buffer conversion
│   └── ...                   ; gRPC client/server components
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
│   │   ├── grace.clj         ; Grace - ornamental notes
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
  - `timewalk.clj` - Musical structure traversal
  - `attachment_resolver.clj` - Attachment resolution
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
│           └── measures
│               └── voices
│                   └── items (Pitch, Chord, Rest, Tuplet, Tremolando, Grace, etc.)
└── layouts
    └── page-views
        └── system-views
            └── staff-views
                └── measure-views
                    ├── glyphs
                    └── curves
```

The entire structure is always a pure tree, making serialization and deserialization straightforward. Cross-references are implemented as ID references, not pointers. IDs are lazily assigned to objects as needed.

**Model-specific operations**: Each model file contains the most basic operations for that model, such as adding and deleting nested items. More complex or abstract operations (formatting, traversal) are placed in the `ops/` directory.

## Namespace Organization: How to Import

Ooloi's shared system provides a carefully structured namespace organization for the complete system used by both frontend and backend.

### Quick Start

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
- **What it contains**: `create-pitch`, `create-piece`, `create-musician`, etc.
- **When to use**: When creating new musical or visual data structures
- **Frontend usage**: ✅ Same functions as backend
- **Backend usage**: ✅ Same functions as frontend

#### 2. `ooloi.shared.predicates` - **TYPE CHECKING**
- **What it contains**: `pitch?`, `chord?`, `measure?` (raw items) and `pitch??`, `chord??` (timewalk tuples)
- **When to use**: For type checking and validation
- **Frontend usage**: ✅ Same predicates as backend
- **Backend usage**: ✅ Same predicates as frontend

#### 3. `ooloi.shared.interfaces` - **UNIFIED API**
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

**Key Point**: Frontend and backend use identical imports and functions. No separate representations exist.

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

## Deeper Topics

For detailed information on specific architectural topics, see these guides:

- **[POLYMORPHIC_API_GUIDE.md](../../../guides/POLYMORPHIC_API_GUIDE.md)** - Complete guide to Ooloi's dual-dispatch polymorphic API
  - VPD vs object dispatch patterns
  - Constructors, accessors, and mutators
  - Threading macros and return value guarantees
  - VPD addressing for remote operations

- **[VPDs.md](../../../guides/VPDs.md)** - Vector Path Descriptors usage patterns
  - Compact vs navigator forms
  - Addressing across the musical tree
  - Remote operation patterns

- **[TIMEWALKING_GUIDE.md](../../../guides/TIMEWALKING_GUIDE.md)** - Musical structure traversal
  - Replaces recursive descent with transducer-based temporal coordination
  - Practical timewalk usage
  - Transducer composition patterns
  - Real-world examples

- **[OOLOI_SERVER_ARCHITECTURAL_GUIDE.md](../../../guides/OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Server architecture
  - STM-based piece management
  - Concurrency patterns
  - Thread-safety guarantees

## Coding Conventions

- Use Methodical (`m/defmulti` and `m/defmethod`) for defining and implementing multimethods.
- Using Specter is highly encouraged for nested data structure updates.
- Define shared behaviors in `models/core.clj`.
- Implement model-specific behaviors in individual model files.
- Use docstrings everywhere for collaborative development.
- **Unified development**: Write code that works identically in frontend and backend.
