# A Guided Path for Newcomers

This guide exists because the documentation does not naturally sort itself for a first reader.

Architecture Decision Records are numbered chronologically — in the order decisions were made, not in the order a newcomer needs to encounter them. Reading ADR-0001 through ADR-0044 in sequence means confronting sophisticated distributed-systems reasoning before you understand what a `Piece` contains, or encountering the lazy frontend-backend synchronisation architecture before you know what VPDs are. The answer keeps arriving before you have understood the question.

This path reorganises the material by conceptual dependency. Each phase builds the vocabulary the next phase requires. If you follow it, nothing should be opaque when you arrive at it.

---

## Contents

- [Phase 0 — Problem and Motivation](#phase-0--problem-and-motivation)
- [Phase 1 — The Data Model](#phase-1--the-data-model)
- [Phase 2 — Language and Foundational Choices](#phase-2--language-and-foundational-choices)
- [Phase 3 — Addressing and Operating on the Model](#phase-3--addressing-and-operating-on-the-model)
- [Phase 4 — Temporal Traversal](#phase-4--temporal-traversal)
- [Phase 5 — Network and Deployment Architecture](#phase-5--network-and-deployment-architecture)
- [Phase 6 — Musical Semantics](#phase-6--musical-semantics)
- [Phase 7 — The Rendering Pipeline](#phase-7--the-rendering-pipeline)
- [Phase 8 — The Frontend](#phase-8--the-frontend)
- [Phase 9 — Advanced and Specialised Topics](#phase-9--advanced-and-specialised-topics)
- [The Three Gateway Documents](#the-three-gateway-documents)
- [What the Numeric ADR Order Conceals](#what-the-numeric-adr-order-conceals)

---

## Phase 0 — Problem and Motivation

*Before any architecture: why does this exist, and what is it actually solving?*

**1. [ooloi.org Overview and Technical Comparison](https://www.ooloi.org/technical-comparison.html)**

The public-facing website covers the lineage of notation software — Finale, Sibelius, LilyPond, Igor Engraver, Dorico — and what structural failures each generation inherited or addressed. The "Technical Comparison" section is particularly useful: it names the problems Ooloi is built to solve at the architectural level (conflation of musical meaning and graphical layout; heuristic layout passes; loss of determinism under change; implicit models of musical time) and explains why those problems have persisted for decades. This is the fastest route to understanding what correctness means in this domain.

**2. [MAIN_README.md](../READMEs/MAIN_README.md)**

Orientation: which technologies, what three-project structure, what the system is for. Short; read completely before anything else.

**3. [Blog: "Is Ooloi Over-Engineered?"](https://www.ooloi.org/home/is-ooloi-over-engineered)**

The single most useful motivational document in the corpus. It makes the case for STM, GPU rendering, gRPC, and plugin-first design not as ambition but as engineering necessity — because music notation is quadratic and cubic complexity, not linear like text. The post explains why the domain's inherent difficulty demands this level of architecture, and why "over-engineering" is therefore a category error. Anyone who will later ask why Ooloi is so complicated should read this first.

**4. [Blog: "Computer Science for Musicians"](https://www.ooloi.org/home/computer-science-for-musicians)**

Written for Roland Gurt, a musician who asked what "functional programming" and "transducers" meant. Explains immutability, determinism, and why those concepts are specifically correct for the notation problem — not as programming fashion, but as domain requirements. Not a tutorial; a framing document. It also states plainly that music notation is in the same general complexity class as compiler construction and symbolic mathematics systems. That statement matters for calibrating expectations.

> **What you should understand at the end of Phase 0:** What makes notation a genuinely hard computational problem. Why mutable state is particularly damaging in this domain. Why determinism is the correct north star rather than a luxury.

---

## Phase 1 — The Data Model

*What does music look like as data in Ooloi?*

**5. [SRC_SHARED_README.md](../READMEs/SRC_SHARED_README.md)**

Read this before any ADR. It contains the essential tree diagram:

```
Piece
├── musicians
│   └── instruments
│       └── staves
│           └── measures
│               └── voices
│                   └── items (Pitch, Chord, Rest, Tuplet, …)
└── layouts
    └── page-views
        └── system-views
            └── staff-views
                └── measure-views
                    ├── glyphs
                    └── curves
```

Every other concept in the system is ultimately about traversing, addressing, or transforming nodes in this tree. The separation between the *musical hierarchy* (musicians through items) and the *visual hierarchy* (layouts through curves) is structural — musical meaning and graphical placement are never conflated.

**6. [ADR-0010: Pure Trees](../ADRs/0010-Pure-Trees.md)**

Why a pure tree rather than an object graph with back-references? Short; read completely. The answer is that immutable pure trees have trivially correct serialisation, no aliasing bugs, and no hidden coupling between distant parts of the score.

**7. [ADR-0012: Persisting Pieces](../ADRs/0012-Persisting-Pieces.md)**

Introduces integer ID references as the mechanism for cross-tree relationships — slurs, ties, dynamics — without abandoning tree purity. The distinction between "pure tree structure" and "ID references for cross-element relationships" is crucial to understand before encountering the timewalker, which must resolve those references in musical time order.

> **What you should understand at the end of Phase 1:** The complete shape of a `Piece` in memory. That it is a pure tree. That cross-element references use integer IDs, not pointers. That serialisation is trivially correct as a consequence of this structure.

---

## Phase 2 — Language and Foundational Choices

*Why Clojure? Why STM?*

**8. [ADR-0000: Clojure](../ADRs/0000-Clojure.md)**

The reasoning behind the language choice. Notable not because the conclusion is surprising, but because it articulates which properties of Clojure are load-bearing for this domain: immutable data structures, STM, REPL-driven development, JVM compatibility, and macro-based extensibility. Establishes the vocabulary the rest of the ADRs use.

**9. [ADR-0004: STM for Concurrency](../ADRs/0004-STM-for-concurrency.md)**

Why Software Transactional Memory rather than atoms, locks, or actors? The key argument: music notation requires coordinated updates across multiple parts of the structure atomically. STM is not a performance choice here; it is a correctness choice. The 100,000+ transactions per second benchmark on 2017 hardware establishes that correctness costs nothing in this context — the bottleneck is network, rendering, or I/O, never the transaction mechanism itself.

**10. [ADR-0001: Frontend-Backend Separation](../ADRs/0001-Frontend-Backend-Separation.md)**

The three-project structure — `shared/`, `backend/`, `frontend/` — and the two deployment modes (combined desktop application, backend-only server). Short. The critical point to absorb: frontend and backend share one data model. There are no separate frontend representations of a `Pitch` or a `Piece`. When the combined application runs, frontend and backend are in the same JVM process.

> **What you should understand at the end of Phase 2:** That Clojure's properties are not incidental to the problem. That STM provides ACID correctness for all musical mutations. That "frontend" and "backend" are separable but share a single data model.

---

## Phase 3 — Addressing and Operating on the Model

*How do you name a location in the tree, and how do you work with what you find there?*

This phase is the core operational toolkit. Everything above and below in this reading order uses these concepts.

**11. [ADR-0008: VPDs](../ADRs/0008-VPDs.md)**

Vector Path Descriptors are compact vectors that address locations in the musical hierarchy. `[:m 0 1 0 3 0]` means: musician 0, instrument 1, staff 0, measure 3, voice 0. Read the rationale section — the alternatives considered (string paths, integer indexing, object references) illuminate why VPDs specifically were chosen. VPDs serialise cleanly across gRPC; object pointers do not.

**12. [Guide: VPDs.md](VPDs.md)**

The practical guide to VPDs. Focus on: VPD form vs navigator form, the `vpd/compact` and `vpd/navigator` conversion functions, and the best practices section. The crucial point: compact form is what you write in all application code; navigator form is what Specter uses internally. Conversion is idempotent. You almost never touch navigator form directly.

**13. [Guide: POLYMORPHIC_API_GUIDE.md](POLYMORPHIC_API_GUIDE.md)**

How operations dispatch on first-argument type — VPD vs direct object. This is where the tree structure, VPDs, and STM converge into a single API. The dual-mode design has a concrete reason: VPD operations automatically establish STM transactions and work across gRPC; direct object operations require manual transaction management. The "Twinkle Twinkle Little Star" example near the end is worth reading slowly — it shows what the full API looks like in practice, from creating a piece through adding notes and slurs within a single `dosync`.

**14. [Guide: PIECE_MANAGER_GUIDE.md](PIECE_MANAGER_GUIDE.md)**

How pieces are stored, retrieved, and referenced. UUID-based identification in production. The relationship between piece-id, piece-ref, and piece-object. The prerequisites listed in the guide itself confirm this is the correct moment to read it: VPDs and basic Clojure knowledge needed, nothing more.

> **What you should understand at the end of Phase 3:** How to address any element in any piece using a VPD. How the polymorphic API dispatches. How pieces are stored and retrieved. How to read and construct any API call in the codebase.

---

## Phase 4 — Temporal Traversal

*How do you process music in time rather than in tree order?*

**15. [ADR-0014: Timewalk](../ADRs/0014-Timewalk.md)**

The problem statement alone is worth reading carefully. A polyphonic score requires that all events at measure N are processed before any events at measure N+1, across all voices and all instruments simultaneously. This ordering cannot be derived from the tree structure directly — the tree is spatial, not temporal. The timewalk system makes temporal coordination the foundational abstraction for all musical computation. This is a genuine paradigm shift.

**16. [Guide: TIMEWALKING_GUIDE.md](TIMEWALKING_GUIDE.md)**

The practical guide. The key insight stated near the top: timewalking transforms a piece into a stream of musical events in time order. Rather than navigating musicians → instruments → staves → measures → voices manually, you get a single stream where everything flows in musical time. The guide teaches both the Ooloi API and Clojure transducers simultaneously — if you understand one, you understand the other.

Pay attention to the streaming vs materialisation distinction. Use streaming (constant memory) for cache refresh, endpoint search, and data export. Materialise into a vector only when random access to the full result set is required.

The timewalker returns three-element tuples: `[item vpd position]`. The VPD is the item's address in the hierarchy; the position is its temporal location. Both are essential to operations that must locate an item and refer to it afterwards.

> **What you should understand at the end of Phase 4:** That the timewalk is the fundamental operation for anything processing music sequentially — MIDI generation, layout, accidental resolution, tie and slur endpoint resolution. How to compose transducers over the resulting stream. When to stream and when to materialise.

---

## Phase 5 — Network and Deployment Architecture

*How does the frontend communicate with the backend, and why does that not cost anything?*

**17. [ADR-0002: gRPC](../ADRs/0002-gRPC.md)**

Why gRPC with Java interop rather than a native Clojure solution. The architectural principle that matters most: the gRPC interface is 1:1 with `api.clj`. There is no separate "network API" distinct from the local API. The same functions accessible locally are accessible remotely. Hundreds of methods are exposed without manual implementation through the `ExecuteMethod` pattern.

**18. [ADR-0018: API-gRPC Interface and Events](../ADRs/0018-API-gRPC-Interface-and-Events.md)**

The `ExecuteMethod` unified endpoint and `OoloiValue` schema. Dynamic function resolution via `ns-resolve` means new API methods are immediately available to remote callers without regenerating stubs. Hot-installable plugins work for the same reason. The server-to-client event streaming architecture is also introduced here — the bidirectional communication model that makes collaborative consistency possible.

**19. [ADR-0019: In-Process gRPC Transport Optimisation](../ADRs/0019-In-Process-gRPC-Transport-Optimization.md)**

The 36-microsecond roundtrip figure. In combined deployments, gRPC transport is in-process, not networked. The frontend-backend separation is therefore architecturally clean (all mutation goes through the API; the frontend never touches the musical model directly) without performance penalty. This ADR is short and resolves what would otherwise feel like a paradox.

**20. [ADR-0022: Lazy Frontend-Backend Architecture](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md)**

The complete data synchronisation model. Three layers: events tell clients which local objects are stale; gRPC requests provide fresh data when needed; local shared API provides fast access to cached data. The key insight: invalidation events contain only structural addresses (VPD regions), not content. The frontend knows *that* something has changed and *where*, then requests fresh data on demand. This is lazy evaluation applied to distributed state.

> **What you should understand at the end of Phase 5:** That frontend and backend share one data model with no conversion overhead. That gRPC is the communication mechanism but carries no overhead in combined deployments. That event-driven invalidation is how collaborative consistency is maintained without polling. The first concrete production application of this pattern to a global singleton (rather than piece data) is the Instrument Library — covered in Phase 9.

---

## Phase 6 — Musical Semantics

*The domain-specific decisions that determine what notes mean.*

**21. [ADR-0026: Pitch Representation and Operations](../ADRs/0026-Pitch-Representation-and-Operations.md)**

String-based canonical form: `"C#4"`, `"Bb3-75"` (75 cents below B♭3), `"C###4"`. Pitches are stored as sounding pitches throughout the system. Key signatures do not alter stored pitch values — they are purely presentational constructs that guide the engraving engine. The ADR treats round-trip integrity under diatonic transposition carefully, because it is harder to guarantee than it initially appears.

**22. [ADR-0033: Time Signature Architecture](../ADRs/0033-Time-Signature-Architecture.md)**

Composite metres, irrational time signatures, the descriptor string format. Establishes rational arithmetic as the internal representation — no floating-point approximations anywhere in the temporal model.

**23. [ADR-0034: Key Signature Architecture](../ADRs/0034-Key-Signature-Architecture.md)**

Standard (major, minor, modal), keyless, mixed-accidental, per-octave variation (Bartók-style), and microtonal key signatures. The critical architectural point: key signatures guide when accidentals are printed; they do not alter stored pitches. This separation between sounding pitch (always stored exactly) and notated pitch (determined at engraving time) is what makes the next ADR possible.

**24. [ADR-0035: Remembered Alterations](../ADRs/0035-Remembered-Alterations.md)**

The first "impossible problem turned straightforward" result. Read the problem statement carefully. Accidentals have temporal memory within measures; that memory applies to the musical timeline, not to visual order; multi-staff instruments require a single shared accidental state across all staves. The solution is deterministic via the timewalker. The same input always produces the same accidental decisions, regardless of layout, staff count, or rendering order.

This is worth understanding thoroughly. It is the first empirical confirmation that the architecture resolves problems the field has long treated as inherently heuristic. The ADR establishes that five user-configurable settings cover the full domain of legitimate taste decisions in accidental rendering. That small configuration surface is a consequence of the algorithm's completeness: where the correct answer can be computed, configuration is unnecessary.

> **What you should understand at the end of Phase 6:** How pitches are represented and why key signatures are presentational. That remembered alterations are deterministically computed via the timewalker — no approximations; adjustments, where supported, are for taste rather than necessity.

---

## Phase 7 — The Rendering Pipeline

*The crown of the architecture. Where all prior foundations become consequence.*

**25. [Blog: "The Rendering Pipeline: Ooloi's Core Architecture"](https://www.ooloi.org/home/the-rendering-pipeline-oolois-core-architecture)**

Peter's own description of ADR-0028 in prose, written when the specification was complete. Read this before the ADR itself; it gives the conceptual shape — the fan-out/fan-in pattern, the separation of connecting from non-connecting elements, the plugin hooks at each stage — without the engineering detail.

**26. [ADR-0028: Hierarchical Rendering Pipeline](../ADRs/0028-Hierarchical-Rendering-Pipeline.md)**

Six stages:

- **Stage 1** (fan-out): Atom formation and collision detection. Each measure independently computes its spatial requirements. Results are immutable and persist until content changes.
- **Stage 2** (fan-in): Vertical reconciliation. The MeasureStackFormatter collects widths and generates result rasters with minimum, ideal, and gutter widths per measure stack.
- **Stage 3** (single): System breaking. Dynamic programming over the measure sequence. Determines optimal system breaks and scale factors.
- **Stage 4** (fan-out): Atom positioning. Applies scale factors from Stage 3; computes absolute coordinates.
- **Stage 5** (fan-out per musician): Spanners and margins. Ties, slurs, beams, hairpins, and glissandi are drawn once final positions are known. Gutter decorations (courtesy accidentals, tie continuations) are added here. System heights become definitive at this stage.
- **Stage 6** (single): Page breaking. Dynamic programming over the system sequence using actual heights from Stage 5.

The gutter model is the detail that enables Stage 3 to be provably optimal: every measure carries both its normal width and the additional space it needs when appearing at system start, so Stage 3 has complete information when making break decisions.

Plugin hooks exist at every stage. Core notation elements and plugin-defined elements use identical interfaces.

**27. [ADR-0037: Measure Distribution Optimisation](../ADRs/0037-Measure-Distribution-Optimization.md)**

The second "impossible problem turned straightforward" result. The Knuth-Plass algorithm — TeX's paragraph-breaking algorithm from 1981, well-known in typesetting circles — applies directly to measure distribution once Stages 1–2 have resolved vertical coordination and collision detection. The problem that had appeared intractable turns out to be textbook dynamic programming on a one-dimensional sequence with separable costs.

The ADR makes the key point explicitly: the algorithm is not novel; its applicability is what the architecture creates. Without the staged pipeline's separation of vertical coordination and collision detection from the distribution decision, the problem resists this formulation — mutable state creates feedback loops, and coupled evaluation of horizontal and vertical concerns prevents the clean one-dimensional reduction. Ooloi's pipeline creates both preconditions.

[Blog post "Twice"](https://www.ooloi.org/home/twice) captures the significance: two problems the industry treats as inherently heuristic — requiring manual correction, special cases, user-facing knobs to tune approximations — collapsed into straightforward algorithms. Same architectural properties both times: immutable data, rational arithmetic, explicit stage boundaries, semantic determinism before layout.

**28. [ADR-0038: Backend Authoritative Rendering and Terminal Frontend Execution](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)**

The frontend is *terminal*: it executes rendering decisions but never renegotiates, refines, or reinterprets them. The litmus test: discard all frontend rendering state, regenerate from backend → identical output. This is the property that makes distributed collaboration and multiple frontend implementations consistent without complex synchronisation.

**29. [ADR-0013: Slur Formatting](../ADRs/0013-Slur-Formatting.md)**

Stage 5 in practice. Point collection via timewalking; shape determination via hull and Bézier; variable-thickness rendering following copper-plate engraving aesthetics. The problem statement is worth reading even if you are not implementing spanners, because it shows how the pipeline's prior stages provide complete information to Stage 5: atom positions, slur start and end points, items under the slur's span — all resolved before Stage 5 begins.

> **What you should understand at the end of Phase 7:** The six pipeline stages and what each one produces. Why Stages 1–2 create the preconditions for Stage 3's optimality. Why the frontend is terminal. Why Knuth-Plass applies here when it does not apply in other notation systems.

---

## Phase 8 — The Frontend

*How users interact with it, and how the UI is structured.*

**30. [FRONTEND_README.md](../READMEs/FRONTEND_README.md)**

The component overview: event-bus, ui-manager, grpc-clients, event-router, fetch-coordinator. Short; establishes vocabulary before the architecture guide.

**31. [ADR-0031: Frontend Event-Driven Architecture](../ADRs/0031-Frontend-Event-Driven-Architecture.md)**

Three event layers: the frontend event bus (category-based pub/sub backed by the shared Claypoole thread pool), the backend event router (categorises and batches backend events for the bus), and JavaFX (UI input only). The threading model and handler isolation guarantee that a slow subscriber cannot block the publisher, and that one handler's failure does not affect others.

**32. [ADR-0039: Localisation Architecture](../ADRs/0039-Localisation-Architecture.md)**

GNU gettext `.po` files, `tr-declare` as first-class mechanism for translation key visibility, instant locale switching via event-driven architecture. Short. The critical architectural point: locale is application state, not startup configuration.

**33. [ADR-0042: UI Specification Format](../ADRs/0042-UI-Specification-Format.md)**

Pure-data UI specification. cljfx specs, setting declarations, command descriptors. The per-window reactive renderer pattern — the piece window as pilot implementation. The absolute invariant: the UI Manager manages Stages (outer window shell); window files manage content nodes (inner reactive content). These responsibilities never overlap.

**34. [ADR-0043: Frontend Settings](../ADRs/0043-Frontend-Settings.md)**

Lazy loading, atomic file writes, closed mutation surface. Short. Read in conjunction with ADR-0042.

**35. [Guide: FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)**

Read this after the individual ADRs, so that specific concepts arrive with context. The guide synthesises window lifecycle, event architecture, the rendering pipeline's frontend side, fetch coordination, localisation, and collaboration.

Pay particular attention to the section addressed to "Frontend Framework Developers." It explains directly why this architecture inverts React instincts: the frontend holds no semantic authority; that subordination is precisely what makes distributed collaboration, transport independence, and first-class plugin composition possible without architectural compromise. The guide also explains why UI structure is expressed as pure data rather than imperative widget construction — plugins must be able to describe UI across process and network boundaries, which requires data, not objects.

> **What you should understand at the end of Phase 8:** The event architecture. How windows are lifecycle-managed. Why UI structure is expressed as pure data. Why plugins can contribute windows, settings, and commands without importing JavaFX.

---

## Phase 9 — Advanced and Specialised Topics

*Read in any order after Phase 8, driven by interest.*

These documents are not required for understanding Ooloi's architecture. They provide depth in specific areas.

**[ADR-0003: Plugins](../ADRs/0003-Plugins.md)** — The minimal core / plugin ecosystem design philosophy. Why commercial closed-source plugins are intentionally supported. Why core notation elements and plugin elements use identical interfaces.

**[ADR-0027: Plugin-Based Audio Architecture](../ADRs/0027-Plugin-Based-Audio-Architecture.md)** — Why MIDI output is architecturally obsolete and what replaces it. Igor Engraver's "DNA soup" of MIDI workarounds, and why modern virtual instruments make a clean break possible. The OVID (Ooloi Virtual Instrument Definition) concept.

**[ADR-0036: Collaborative Sessions and Hybrid Transport](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)** — How a standalone application transitions dynamically to host or guest mode without restart. The precondition that all pieces must be closed before backend switching, which eliminates race conditions entirely without requiring epoch-tagging mechanisms.

**[ADR-0040: Single Authority State Model](../ADRs/0040-Single-Authority-State-Model.md)** — The formal statement of state ownership rules across the system.

**[ADR-0045: Instrument Library](../ADRs/0045-Instrument-Library.md)** — The first non-piece singleton in the architecture: a server-side instrument registry with optimistic locking, lazy frontend caching, and invalidate-only event synchronisation. Concrete proof that the single-authority model scales beyond piece data to any global state. The bundled default library covers the full orchestral repertoire from Bach to Messiaen, together with the mechanisms for users to extend it permanently with instruments of their own.

**[ADR-0030: MusicXML](../ADRs/0030-MusicXML.md)** — Import and export as a first-class plugin, preserving musical meaning rather than graphical approximation.

**[Guide: INTEGRANT_COMPONENTS.md](INTEGRANT_COMPONENTS.md)** — Integrant component lifecycle, the three-project wiring asymmetry, the combined system dependency graph, startup sequence, component wiring checklist (including two unintuitive requirements), and the full testing macro reference. Essential reading before adding any new component to the system.

**[Guide: ADVANCED_CONCURRENCY_PATTERNS.md](ADVANCED_CONCURRENCY_PATTERNS.md)** — STM edge cases, commutative operations, coordination under load.

**[Guide: PIECE_PERSISTENCE_GUIDE.md](PIECE_PERSISTENCE_GUIDE.md)** — Asynchronous save and load via Clojure agents, multiple I/O backends.

**Research: GEOMETRY_OF_THE_UNWRITTEN.md** — The theoretical basis for whitespace-first spanner placement, which treats available space as a first-class geometric object rather than reasoning from obstacles. More advanced; relevant primarily to contributors implementing connecting elements.

---

## The Three Gateway Documents

The Ooloi blog identifies three documents as the main conceptual gateways into the system's internals. Having followed Phases 0–7, you will arrive at all three with full context:

- **[Guide: TIMEWALKING_GUIDE.md](TIMEWALKING_GUIDE.md)** — Teaches temporal stream processing and Clojure transducers through musical examples. The transformation of spatial structure (the score) into temporal flow (the music) is the central concept.
- **[Guide: POLYMORPHIC_API_GUIDE.md](POLYMORPHIC_API_GUIDE.md)** — Teaches the operational model: tree, VPDs, STM, and API dispatch working together. The practical entry point for writing code.
- **[Guide: FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)** — Teaches the complete frontend model and its relationship to the backend. The structural argument for terminal execution is made here in full.

The guides are not the entry point. They are where earlier foundations converge into practical synthesis. A newcomer who arrives at them in numeric order will find them opaque. A newcomer who follows this path will find them clear.

---

## What the Numeric ADR Order Conceals

The chronological ordering reflects when decisions were made, not their conceptual difficulty. The earliest decisions (ADR-0000 through ADR-0014) establish the foundations — and they happen also to be the most readable documents in the corpus. They answer "why Clojure," "why a pure tree," "why VPDs," "why timewalking." These are exactly the documents a newcomer most needs first.

The documents hardest to read without prior context — ADR-0022 (lazy frontend-backend synchronisation), ADR-0024 (gRPC flow control), ADR-0025 (server statistics), ADR-0036 (collaborative sessions) — sit in the middle of the numerical range. They are detailed implementation decisions for contributors working on those specific systems. They can be deferred indefinitely by anyone whose primary interest is in the musical domain, the rendering pipeline, or the frontend architecture.

The ADRs with the highest numbers are often the most sophisticated: ADR-0035 (remembered alterations), ADR-0037 (measure distribution), ADR-0038 (terminal frontend execution). These are some of the most important documents in the corpus. They require the full context of everything before them to be understood. Following this path ensures you arrive at them prepared.

---

*This guide was produced by systematic retrieval across the complete Ooloi documentation corpus. The reading order reflects conceptual dependencies, not chronology.*
