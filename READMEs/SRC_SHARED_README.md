# Ooloi Shared Source Directory

This is the developer reference for the Ooloi shared library — the core of the system that both
frontend and backend build on. It covers architecture, data models, namespace organisation, code
conventions, invariants, and testing. Whether you are working on shared models, backend operations,
or tracing a production code path, this is the starting point.

## Table of Contents

- [Project Architecture](#project-architecture)
- [Directory Structure](#directory-structure)
  - [Key Directories](#key-directories)
- [Data Model](#data-model)
- [Core Data Flow](#core-data-flow)
- [Namespace Organisation: How to Import](#namespace-organisation-how-to-import)
  - [Quick Start](#quick-start)
  - [Most Important Namespaces](#most-important-namespaces)
  - [Frontend and Backend Usage](#frontend-and-backend-usage)
  - [Decision Tree: Which Namespace to Use?](#decision-tree-which-namespace-to-use)
- [Architectural Invariants](#architectural-invariants)
- [Adding New Multimethods](#adding-new-multimethods)
- [Unified gRPC Architecture](#unified-grpc-architecture)
- [Code Conventions](#code-conventions)
- [Testing Infrastructure](#testing-infrastructure)
- [Build & Test Commands](#build--test-commands)
- [Deeper Topics](#deeper-topics)

## Project Architecture

Ooloi is a three-project repository:

```
Ooloi/
├── shared/     # Core models, traits, interfaces, predicates — consumed by both backend and frontend
├── backend/    # Server, piece management, complex operations
└── frontend/   # UI components, rendering, user interaction
```

**The shared project is the core library.** It contains the complete Ooloi data model, all
multimethods, predicates, and operations. Both backend and frontend are consumers — they add
capabilities on top of what shared defines, but the shared code is where all musical and visual
data structures live.

**Combined app entry point**: `shared/src/app/clojure/ooloi/shared/system.clj` is the production
entry point for the combined desktop application. It orchestrates startup, wires all action handlers
(windows, menus, lifecycle events), and connects all component events.
`frontend/src/main/clojure/ooloi/frontend/system.clj` is the frontend-only system used primarily
for testing — it does not wire action handlers or UI lifecycle. When tracing production code paths
for windows, menus, and startup, always look in `shared/system.clj` first.

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
├── srv_client.clj            ; gRPC server/client utilities for combined system testing
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

**⚡ CRITICAL**: This IS the complete Ooloi system. Frontend and backend are consumers that use
this unified data model and API — no separate representations exist.

### Key Directories

**Core System Files:**
- `core.clj` - Complete system namespace; assembles and re-exports the whole system
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
- `grpc/conversion.clj` - gRPC conversion utilities
- `specs/generators.clj` - Test data generation for all models
- `traits/` - All behavioral mixins used across the system

## Data Model

Ooloi uses the following unified tree structure for all musical elements (used identically by
frontend and backend):

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

The entire structure is always a pure tree, making serialization and deserialization
straightforward. Cross-references are implemented as ID references, not pointers. IDs are lazily
assigned to objects as needed.

**Model-specific operations**: Each model file contains the most basic operations for that model,
such as adding and deleting nested items. More complex or abstract operations (formatting,
traversal) are placed in the `ops/` directory.

## Core Data Flow

The shared system files have a strict dependency ordering that must never be violated:

```
predicates.clj → interfaces.clj → models/*.clj → core.clj → api.clj
```

Each layer may only depend on layers to its left:

- `predicates.clj` — pure predicate functions, no model dependencies
- `interfaces.clj` — multimethod declarations using predicates for dispatch; no model implementations
- `models/*.clj` — data structures and model-specific implementations; may use interfaces and predicates
- `core.clj` — assembles the complete system; imports and re-exports everything
- `api.clj` — public API surface for external consumers

**The key constraint**: Files that import `interfaces` or `predicates` directly cannot also import
`core`. Importing `core` would create a circular dependency because `core` imports everything,
including the very files that imported it. This constraint applies to all architecture-sensitive
files — `predicates.clj`, `interfaces.clj`, and any model file use selective imports from specific
namespaces, not `core :refer :all`.

**PieceRefResolver protocol**: In shared contexts, the `PieceRefResolver` protocol requires an
explicit `[ooloi.shared.ops.piece-ref]` require. Without it, the protocol functions exist but
implementations may not be loaded, causing runtime failures. Add this require explicitly in any
shared-layer namespace that resolves piece references.

## Namespace Organisation: How to Import

Ooloi's shared system provides a carefully structured namespace organisation for the complete system
used by both frontend and backend.

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

**Why this works**: The shared `core` namespace IS the complete Ooloi system used identically by
frontend and backend.

The 10% that uses selective imports are architecture-constrained files (`predicates.clj`,
`interfaces.clj`, model files) — they cannot use `core :refer :all` without creating circular
dependencies (see [Core Data Flow](#core-data-flow)).

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
(def my-pitch (create-pitch :note "C4" :duration 1/4))  ; Same function, same Pitch record
(p/pitch? my-pitch)                                     ; Same predicate as backend
(i/get-duration my-pitch)                               ; Same interface as backend
```

**Backend Consumer Pattern:**
```clojure
(ns my-backend-service
  (:require [ooloi.shared.models.musical.pitch :refer [create-pitch]]  ; SAME as frontend
            [ooloi.shared.interfaces :as i]                           ; SAME as frontend
            [ooloi.shared.predicates :as p]))                         ; SAME as frontend

;; Creates identical objects to frontend
(def my-pitch (create-pitch :note "C4" :duration 1/4))  ; SAME function, SAME Pitch record
(p/pitch? my-pitch)                       ; SAME predicate as frontend
(i/get-duration my-pitch)                 ; SAME interface as frontend
```

**Key Point**: Frontend and backend use identical imports and functions. No separate
representations exist.

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

## Architectural Invariants

These constraints must never be violated. They exist to maintain correctness, performance, and
testability across the system.

### VPD Addressing
Vector Path Descriptors (VPDs) are the primary addressing mechanism for musical elements. Always
use precise `boundary-vpd` scope to limit the work a traversal does — never traverse the entire
piece tree when a narrower scope is correct. Broad traversals cause unnecessary work and make
behaviour harder to reason about.

### STM Transactions
All piece mutations must use `dosync` blocks for ACID compliance. Ooloi uses Clojure's Software
Transactional Memory for concurrent piece management. Never mutate piece state outside a `dosync`
block — partial updates will corrupt piece state and produce non-deterministic behaviour.

### Timewalk Results
Every item returned from a timewalk is a `[item vpd position]` tuple. Always use the helper
functions `item`, `position`, and `vpd` to destructure timewalk results — never destructure by
position directly, as the tuple structure may evolve and helpers provide stable access. The
double-question-mark predicates (`pitch??`, `chord??`, etc.) operate on these tuples.

### Multimethod Dispatch
Never add a `:type` parameter or any other magic dispatch parameter to a multimethod for test
convenience. Dispatch must reflect real semantic distinctions in the domain. Test-driven dispatch
hacks leak into production and obscure what the multimethod is actually selecting on.

### Production Code Purity
- Never create test-specific function variants (e.g., `create-real-foo` / `create-test-foo`).
  Tests must exercise production code through production code paths.
- Never adapt production code to accommodate broken test setups. When a test fails because a mock
  is incomplete or a test assumption is wrong, fix the test — never add nil-guards, fallbacks, or
  alternate code paths to production to make the test pass.

## Adding New Multimethods

When adding a new operation to the Ooloi API, follow this four-step procedure in order:

1. **Define in `interfaces.clj`** with proper `:vpd-category` metadata. The metadata ensures VPD
   integration is automatic.
2. **Export through `core.clj`** in alphabetical order among the existing exports.
3. **Export through `api.clj`** in alphabetical order among the existing exports.
4. **Add implementations in appropriate model files** — each model file contains the
   implementations specific to that type.

VPD integration is automatic once the `:vpd-category` metadata is correct. Alphabetical ordering
in steps 2 and 3 is enforced by convention to make the exports easy to scan and update.

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
│   (create-pitch :note "C4" :duration 1/4)  ← SAME function as backend
│   (pitch? some-pitch)                      ← SAME predicate as backend
│
├── Frontend Remote Usage (via gRPC)
│   Frontend ──gRPC──→ Backend ──delegates──→ SAME shared functions
│   (grpc-call :create-pitch {...})          ← Returns SAME Pitch record
│
└── Backend Usage
    (create-pitch :note "C4" :duration 1/4)  ← IDENTICAL to frontend
    (pitch? some-pitch)                ← IDENTICAL to frontend

⚡ KEY: Same data structures, same functions, same everything!
```

## Code Conventions

### Copyright Header

All new source files begin with the following header, placed before the namespace declaration
with one blank line after:

```clojure
;; Copyright © 2024-2026 Peter Bengtson
;;
;; This Source Code Form is subject to the terms of the Mozilla Public License, v2.0.
;; If a copy of the MPL was not distributed with this file, you can obtain one at http://mozilla.org/MPL/2.0/.

(ns ooloi.example.namespace
  ...)
```

### Defrecord Documentation

Document `defrecord` types in the **namespace docstring**, not as comments. Comments are invisible
to documentation generators (Codox); namespace docstrings appear in generated API docs. Include a
"Record Structure" section with field descriptions. See `time_signature.clj` and
`key_signature.clj` as reference examples.

### Namespace Import Strategy

- **90% of code**: Use `[ooloi.shared.models.core :refer :all]` — this gives access to all
  constructors, predicates, and multimethods in one import.
- **Architecture-constrained files** (`predicates.clj`, `interfaces.clj`, model files): Use
  selective imports from specific namespaces. These files cannot use `core :refer :all` without
  creating circular dependencies (see [Core Data Flow](#core-data-flow)).

### Code Comments

- **Functional comments**: Explain non-obvious logic liberally.
- **No test-reference comments**: Never add comments like `; For Test 40` in production code.
  Tests and production code are separate concerns.
- **No temporary comments**: Remove any comments added as implementation scaffolding once the
  implementation is complete.

### Methodical Multimethods

Use Methodical (`m/defmulti` and `m/defmethod`) for defining and implementing multimethods — not
core Clojure's `defmulti`/`defmethod`. Methodical provides better composability and tooling.

### Specter

Using Specter for nested data structure updates is highly encouraged. Specter transforms that
would otherwise require deeply nested `update-in` calls are clearer and less error-prone with
Specter navigators.

### Shared Behaviors

Define shared behaviors (traits, mixins) in `models/core.clj`. Implement model-specific behaviors
in individual model files. Use docstrings everywhere for collaborative development.

## Testing Infrastructure

Ooloi provides two test helper namespaces with macros that handle all lifecycle boilerplate.
**Never use raw `CountDownLatch` + `fx/run-later!` + `.await` patterns** — use
`run-on-fx-thread!` instead.

### `util.frontend` — JavaFX test helpers

Used in `frontend/test/` and `shared/test/app/` (system integration tests).

```clojure
(require '[util.frontend :as th])
```

| Macro / Function | Purpose |
|---|---|
| `run-on-fx-thread! f` | Runs `f` on the JavaFX thread, blocks until complete, returns the value. Replaces all `CountDownLatch` + `fx/run-later!` + `.await` patterns. |
| `with-test-config {overrides}` | Combines platform directory isolation, settings isolation, and locale isolation. Controls `load-defaults` via the overrides map. |
| `with-event-bus` | Creates a live event bus for the duration of body. Required when calling `set-app-setting!` with a value that differs from the stored value. |
| `with-isolated-platform-directory` | Redirects all platform directory access to a temp directory. Included inside `with-test-config`. |
| `with-restored-settings` | Saves and restores settings atoms across the body. Included inside `with-test-config`. |
| `default-settings` | Returns all registry settings at their defaults. Use as base for `load-defaults` mocks. |
| `with-zero-animation-times` | Sets all animation durations to zero for lifecycle tests. |

**Double FX flush** (macOS deferred setup): use two sequential calls — `(th/run-on-fx-thread! (fn [])) (th/run-on-fx-thread! (fn []))`. Never nest `run-on-fx-thread!` calls — that deadlocks.

### `util.server` — gRPC and server test helpers

Used in `shared/test/` and `backend/test/`.

```clojure
(require '[util.server :refer :all])
```

| Macro / Function | Purpose |
|---|---|
| `with-server [s]` / `with-server [s :in-process]` | Starts server, runs body, stops server. Default 100ms startup delay. |
| `with-clients s [[c1 "id1"] [c2 "id2"]]` | Creates and registers multiple clients with automatic cleanup. Clients inherit TLS settings from server. |
| `with-srv-client c` | Binds client to `SRV/*` dynamic context so all `SRV/` calls use it automatically. |
| `with-system [sys {}]` | Full backend Integrant system with automatic halt. |
| `with-combined-system [sys]` | All 9 application components, in-process gRPC transport, headless UI. Use for combined system integration tests. |

**Typical patterns:**

```clojure
;; FX thread — render a spec and inspect the result
(let [root (th/run-on-fx-thread!
             #(cljfx/instance (cljfx/create-component (build-spec))))
  (instance? TabPane root) => true)

;; Settings isolation
(th/with-test-config {:ui/theme :nord-dark}
  (th/with-event-bus
    (settings/get-app-setting :ui/theme) => :nord-dark))

;; gRPC client/server
(with-server [s :in-process]
  (with-clients s [[c "test-user"]]
    (with-srv-client c
      (SRV/create-rest :duration 1/4))))
```

## Build & Test Commands

All lein commands require the correct working directory — always `cd` to the project root before
running lein. Running lein from the wrong directory produces `LOAD FAILURE` errors that resemble
compilation bugs.

```bash
# Shared project (run from shared/)
cd /path/to/Ooloi/shared && lein midje    # All shared tests
cd /path/to/Ooloi/shared && lein build

# Backend project (run from backend/)
cd /path/to/Ooloi/backend && lein midje   # All backend tests
cd /path/to/Ooloi/backend && lein build
cd /path/to/Ooloi/backend && lein codox   # Generate API documentation
cd /path/to/Ooloi/backend && lein coverage # Coverage report
```

All three test suites (shared, backend, frontend) must pass for project success.

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
