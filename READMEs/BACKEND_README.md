# Ooloi Backend Source Directory

This directory contains the source code for the backend of Ooloi.


## Table of Contents

- [Directory Structure](#directory-structure)
- [Data Model](#data-model)
- [Class-specific Operations](#class-specific-operations)
- [Programming Paradigm](#programming-paradigm)
  - [Key Concepts](#key-concepts)
- [Constructors, Accessors and Mutators](#constructors-accessors-and-mutators)s
  - [Constructors](#constructors)
  - [Accessors and Mutators](#accessors-and-mutators)
- [Parallelism and Thread-Safety](#parallelism-and-thread-safety)
- [Principles and Requirements](#principles-and-requirements)
- [Coding Conventions](#coding-conventions)


## Directory Structure

```
backend/
└── src/
    └── main/
        └── clojure/
            └── ooloi/
                └── backend/
                  ├── api.clj
                  ├── core.clj
                  ├── ops/
                  │   ├── attachment_resolver.clj
                  │   ├── changes.clj
                  │   ├── persistence.clj
                  │   ├── piece_manager.clj
                  │   ├── pitches.clj
                  │   ├── rhythm.clj
                  │   ├── text.clj
                  │   ├── vectors_and_attributes.clj
                  │   ├── vpd.clj
                  │   └── walk.clj
                  └── models/
                      ├── core.clj
                      ├── hierarchy.clj
                      ├── musical/
                      │   ├── attachments/
                      │   │   ├── articulation.clj
                      │   │   ├── dynamic.clj
                      │   │   ├── glissando.clj
                      │   │   ├── hairpin.clj
                      │   │   ├── ottava.clj
                      │   │   ├── slur.clj
                      │   │   └── tie.clj
                      │   ├── chord.clj
                      │   ├── instrument.clj
                      │   ├── measure.clj
                      │   ├── musician.clj
                      │   ├── piece.clj
                      │   ├── pitch.clj
                      │   ├── rest.clj
                      │   ├── staff.clj
                      │   ├── tremolando.clj
                      │   ├── trill.clj
                      │   ├── tuplet.clj
                      │   └── voice.clj
                      ├── traits/
                      │   ├── attachment.clj
                      │   ├── has_items.clj
                      │   ├── rhythmic_item.clj
                      │   ├── takes_attachment.clj
                      │   └── transposable.clj
                      └── visual/
                          ├── layout.clj
                          ├── measure_view.clj
                          ├── page_view.clj
                          ├── staff_view.clj
                          └── system_view.clj
```     
The above diagram doesn't include all models.

## Data Model

Ooloi uses the following tree structure for its main musical elements:

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

The entire structure is always a pure tree, which makes serialisation and deserialisation straightforward. Cross-references are implemented as ID references, not pointers. IDs are lazily assigned to objects as needed.


## Class-specific Operations

Each file also contains the most basic operations for the model, such as adding and deleting nested items, e.g. to add a Layout to a Piece. However, the idea here is brevity and understandability.

More complex or abstract operations, such as formatting a MeasureView for display, are placed in the `ops` directory in order to keep model files succinct and understandable.


## Programming Paradigm

Ooloi uses a dynamic, functional programming approach to handle complex music notation requirements. At its core, it leverages Clojure's functional programming paradigms, enhanced with the Methodical library, to provide a powerful and flexible system for music notation and manipulation.

### Key Concepts

1. **Functional Programming:** Ooloi uses Clojure, a functional programming language, to ensure immutability, ease of reasoning about code, and leveraging powerful data structures.
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

Similar `create-xxxxx` functions are available for all API models.

### Accessors and Mutators

Automatically defined accessors and mutators retrieve and modify attributes of defrecords. For attributes that contain vectors or ChangeSets, standardised accessors and mutators are also available to manipulate their elements. Just as for constructors, there's a consistent naming scheme for accessors and mutators:

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

There are also so-called ChangeSets, used to implement things that start at one point and go on until they are changed.
Examples are key and time signatures and tempo changes, but also instrument changes for a musician. An attribute called
`tempos` using a ChangeSet has the following methods:
- `get-tempo`
- `get-tempos`
- `remove-tempo`
- `set-tempo`
- `set-tempos`

All accessors and mutators are exposed in the `api` namespace. There may of course be other operations specific
to each model, but most operations follow the above pattern. The documentation describes all such operations and their arguments.


## Core and API Polymorphism

The above conventions result in clear, succinct, and understandable backend code like the following:

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

    ...
)
```

Here, we are freely passing instances of Piece, Pitch, Instrument and other models around, as you do in the backend. We have direct access to all objects; they can be directly referenced and manipulated. The polymorphic dispatch will use the type of the first argument to dispatch to the right method. A method that adds something to something else will return a new version of the updated object, just as you would expect.

However, this also means that the above usage pattern operates only locally on the data. If you change something deep inside a piece, only that object will change. But in an immutable data structure such as Piece, updating a nested object requires replacing all its ancestors all the way up to the piece, or the piece won't see the change. 

Moreover, the above code does not establish any transactions for its operations, so it's not inherently threadsafe.

Now, this is all intentional. This way of working is specifically designed for working with backend core code; it's fully expected that the developer writing such code will wrap piece-mutating code in a `dosync` with a `ref` on the piece. The ancestor chain up to the piece may be modified using any methodology the developer sees fit, mostly using Specter or things like `update-in`. Moreover, the Piece Manager offers assistance with reffing and updating pieces.

Thus we can regard code as the above to be an internal and direct form of data manipulation, designed to run exclusively in the Ooloi backend.

However, Ooloi is designed as a server/client architecture. This means that there is a frontend application that knows nothing of the internals of the backend and which doesn't have direct access to any of its data structures. The frontend and backend talk to each other using gRPC. We thus must have some other way of referring to the data structures of a piece which doesn't involve pointers.

Enter VPDs, Vector Path Descriptors. They are vectors describing the _path_ to data relative to a piece. VPDs are essentially Specter navigators or the paths we see in `get-in` and `update-in`.

All API methods that dispatch on the type of their first argument - which is all of them except the constructors - can also take a VPD as their first argument. Thus:

```clojure
(remove-item measure 2)
```

where `measure`is, say, the measure2 of the example above, and which would remove the third item of that measure ("star") can also be invoked:

```clojure
(remove-item [:musicians 0 :instruments 0 :staves 0 :voices 0 :measures 1] "PieceID2418" 2)
```

The VPD replaces the direct object `measure` with a _description_ of where that object is in `piece`, which also must be supplied in all VPD call signatures: the direct object is replaced by the VPD _and_ a piece reference. The piece reference can be a direct piece object, a ref (for backend use), or as in the example above, a string Piece ID. If a string is given, the Piece Manager will be used to retrieve the piece. 

Thus, this method of calling the backend API can be used by the frontend. There's nothing to prevent the backend from using this method, either: fact is, it's often the most practical way of working.

Also, which is important, when using VPDs as the first parameter the operation is wrapped in an STM transaction and Specter will be used to update the piece. Thus, consumers of the API in the backend have a choice between two different methods of data manipulation, and consumers of the API in the frontend have the same expressivity with the added bonus of automatic management of the underlying data structures, in a fully transactional, threadsafe way.

VPDs also have a more compact form. The above can also be written:

```clojure
(remove-item [:m 0 0 0 0 1] "PieceID2418" 2)
```

The frontend window manager translates clicks on the screen to VPDs suitable for passing to the backend. The fact that the frontend always will pass a string piece reference with every call allows the backend to serve multiple pieces simultaneously; there is no backend "current piece".

This means that setting a slur from the first note to the last one in the example above from the frontend simply becomes:

```clojure
(add-attachment [:m 0 0 0 0 0 :items 0] "PieceID2418" "slur" [:m 0 0 0 0 1 :items 1])
```

There's additional magic going on in this particular case (using an :around Methodical method), but that complexity is all abstracted away for the API user.

The end result is a powerful, expressive API that is the same for the backend and the frontend. Moreover, the API is used throughout the entire application:; it's not just something tacked on for the frontend to use.


## Parallelism and Thread-Safety

Ooloi utilizes Clojure's Software Transactional Memory (STM) with refs to provide a powerful, thread-safe framework for accessing and mutating complex musical structures. This system forms the foundation of all data operations in Ooloi.

Key aspects:
1. All piece data is stored in refs, allowing for ACID transactions.
2. Updates to the piece are wrapped in dosync blocks, ensuring atomicity and consistency.
3. Automatic retries are handled by the STM when conflicts occur in concurrent modifications.

The system provides:
- Functions for safely updating data within the transactional system.
- Functions to mark measures for later visual recomputation.

Performance metrics (on a 2017 MacBook Pro 2,2 GHz 6-Core Intel Core i7):
- Transactional mode: 100,000+ updates per second (1000 updates in 10 ms).
These results demonstrate the system's ability to handle a high volume of concurrent updates efficiently, both with and without explicit transaction management.


## Principles and Requirements

Overall Principles and Requirements:
  - Maintain polymorphism across all models.
  - Keep model-specific definitions close to their respective model files.
  - Follow Clojure best practices and idiomatic patterns.
  - Prioritize simplicity and avoid unnecessary complexity.
  - Design for extensibility and ease of use.
  - Maintain separation between musical and visual models.
  - Use docstrings everywhere. The documentation tool (Codox) uses them. They are crucial in a collaborative environment.
  - Use STM for managing concurrency, ensuring thread-safe operations and data integrity.

## Namespace Organization and Import Guidelines

Ooloi uses a carefully structured namespace organization to eliminate circular dependencies while providing a clean, unified API. Understanding when to use which namespace is crucial for maintainable code.

### Quick Start: What to Import

**For 90% of your code, you only need this:**

```clojure
(ns my-namespace
  (:require [ooloi.backend.models.core :refer :all]))
```

This gives you access to:
- **All constructors**: `create-pitch`, `create-chord`, `create-measure`, etc.
- **All predicates**: `pitch?`, `chord?`, `measure?`, etc.
- **All multimethods**: `get-duration`, `add-item`, `set-name`, etc.

**Why this works**: The `core` namespace is the unified entry point that re-exports everything you need from the entire Ooloi system.

### Most Important Namespaces

#### 1. `ooloi.backend.models.core` - **YOUR PRIMARY NAMESPACE**
- **What it is**: The main entry point for all Ooloi functionality
- **What it contains**: All constructors, predicates, and multimethods
- **When to use**: For application code, tests, examples, and most development
- **How to import**: `[ooloi.backend.models.core :refer :all]`

#### 2. `ooloi.backend.models.predicates` - **FOR ARCHITECTURE-CONSTRAINED FILES**
- **What it is**: Type checking functions (pitch?, chord?, measure?, etc.)
- **What it contains**: Only type predicates
- **When to use**: In model files, trait files, or when you can't import core
- **How to import**: `[ooloi.backend.models.predicates :refer [pitch? chord?]]` (selective)

#### 3. `ooloi.backend.models.interfaces` - **FOR ARCHITECTURE-CONSTRAINED FILES**
- **What it is**: Multimethod definitions (method signatures only)
- **What it contains**: get-duration, add-item, set-name, etc.
- **When to use**: In model files, trait files, or when you can't import core
- **How to import**: `[ooloi.backend.models.interfaces :as i]`

#### 4. `ooloi.backend.api` - **LEGACY COMPATIBILITY**
- **What it is**: Legacy namespace for backward compatibility
- **What it contains**: Same as core, but older interface
- **When to use**: Only for testing the API explicitly or legacy code
- **How to import**: `[ooloi.backend.api :as api]`

### When You Can't Use Core

Some files cannot import `core` because it would create circular dependencies:

**Architecture-constrained files:**
- Model implementation files (`models/musical/*.clj`, `models/visual/*.clj`)
- Trait files (`models/traits/*.clj`)
- Some ops files (`ops/timewalk.clj`, `ops/attachment-resolver.clj`)

**In these files, use selective imports:**

```clojure
(ns ooloi.backend.models.musical.pitch
  (:require [ooloi.backend.models.interfaces :as i]
            [ooloi.backend.models.predicates :refer [pitch? rhythmic-item?]]))

;; Use predicates unqualified (clean and readable)
(pitch? item)
(rhythmic-item? item)

;; Use multimethods qualified
(i/get-duration item)
(i/set-duration item new-duration)
```

### Summary: What Most Developers Need to Know

1. **Start with core**: `[ooloi.backend.models.core :refer :all]` for 90% of your code
2. **Use selective imports**: Only in architecture-constrained files
3. **Don't use api**: Core is preferred for new development
4. **When in doubt**: Use core with `:refer :all`

### Core Namespace Architecture

The namespace architecture follows a clean dependency flow:

```
predicates.clj → interfaces.clj → models/*.clj → core.clj → api.clj
```

### Namespace Descriptions

#### `ooloi.backend.models.predicates`
- **Purpose**: Contains all type predicates (pitch?, chord?, measure?, etc.)
- **Dependencies**: Only imports hierarchy (no circular dependencies)
- **Usage**: For type checking and validation

#### `ooloi.backend.models.interfaces`
- **Purpose**: Contains all multimethod definitions (method signatures only)
- **Dependencies**: Imports predicates and hierarchy
- **Usage**: For defining polymorphic operations

#### `ooloi.backend.models.core`
- **Purpose**: Unified namespace re-exporting multimethods, predicates, AND constructors
- **Dependencies**: Imports and re-exports from interfaces, predicates, and all model files
- **Usage**: Primary namespace for application code

#### `ooloi.backend.api`
- **Purpose**: Legacy namespace for backward compatibility
- **Dependencies**: Re-exports from core
- **Usage**: Maintained for backward compatibility, but core is preferred

#### Model Files (`models/musical/*.clj`, `models/visual/*.clj`)
- **Purpose**: Implement multimethods and define constructors
- **Dependencies**: Import interfaces and predicates (NOT core)
- **Usage**: Internal implementation files

### Import Guidelines by File Type

#### **Most Application Code, Tests, and Examples**
Use the core namespace with `:refer :all` for clean, readable code:

```clojure
(ns my-application
  (:require [ooloi.backend.models.core :refer :all]))

;; All functions available unqualified:
(create-pitch :note :C4 :duration 1/4)  ; Constructor
(pitch? some-item)                       ; Predicate
(get-duration some-item)                 ; Multimethod
```

#### **API Tests (testing the API explicitly)**
Use qualified API imports to ensure you're testing the API namespace:

```clojure
(ns api-test
  (:require [ooloi.backend.api :as api]
            [ooloi.backend.models.core :refer [get-musicians get-name]]))

;; Constructors explicitly qualified
(api/create-pitch :note :C4 :duration 1/4)
;; Multimethods from core (selective import)
(get-musicians piece)
```

#### **Files with Architecture Constraints (Model Files, Traits, Some Ops)**
These files CANNOT import core (would create circular dependencies). Use selective imports:

```clojure
(ns ooloi.backend.models.musical.pitch
  (:require [ooloi.backend.models.interfaces :as i]
            [ooloi.backend.models.predicates :refer [pitch? rhythmic-item?]]))

;; Clean, unqualified predicates
(pitch? item)
(rhythmic-item? item)
;; Qualified multimethods
(i/get-duration item)
```

#### **Predicate Test Files**
Test predicates explicitly using qualified imports:

```clojure
(ns predicates-test
  (:require [ooloi.backend.models.core :refer :all]  ; For constructors
            [ooloi.backend.models.predicates :as p])) ; For predicates

;; Constructors unqualified
(create-pitch :note :C4 :duration 1/4)
;; Predicates explicitly qualified
(p/pitch? item)
```

### Decision Tree: Which Namespace to Use?

1. **Are you writing application code, tests, or examples?**
   → Use `[ooloi.backend.models.core :refer :all]`

2. **Are you explicitly testing the API namespace?**
   → Use `[ooloi.backend.api :as api]` with selective core imports

3. **Are you in a model file, trait file, or constrained ops file?**
   → Use selective imports from `interfaces` and `predicates`

4. **Are you testing predicates explicitly?**
   → Use `[ooloi.backend.models.predicates :as p]`

### Key Benefits of This Architecture

- **No Circular Dependencies**: Clean unidirectional dependency flow
- **Unified API**: Core namespace provides access to all functions
- **Selective Imports**: Architecture-constrained files can import only what they need
- **Readable Code**: Unqualified function calls in application code
- **Explicit Testing**: API and predicate tests are explicit about what they're testing
- **Maintainability**: Clear separation of concerns and dependencies

### Common Mistakes to Avoid

- **Never import core in model files**: This creates circular dependencies
- **Don't use `:refer :all` for predicates in predicate tests**: Be explicit about what you're testing
- **Don't mix qualified and unqualified calls**: Pick one style per namespace
- **Don't use api namespace for new code**: Core is preferred for new development

## Coding Conventions

  - Use Methodical (`m/defmulti` and `m/defmethod`) for defining and implementing multimethods.
  - Using Specter is highly encouraged. There is a separate [Specter guide](../../guides/SPECTER.md) to give you some tips.
  - Define shared behaviors in `models/core.clj`.
  - Implement model-specific behaviors in individual model files (e.g., `dynamic.clj`, `slur.clj`).
  - Expose public functions through `api.clj` using `import-vars`.
