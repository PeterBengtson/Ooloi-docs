# A Guided Path for Newcomers

This guide exists because the documentation does not naturally sort itself for a first reader.

Architecture Decision Records are numbered chronologically — in the order decisions were made, not in the order a newcomer needs to encounter them. Reading the ADRs in numerical order means confronting sophisticated distributed-systems reasoning before you understand what a `Piece` contains, or encountering the lazy frontend-backend synchronisation architecture before you know what VPDs are. The answer keeps arriving before you have understood the question.

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

The same ADR introduces integer ID references as the mechanism for cross-tree relationships — slurs, ties, dynamics — without abandoning tree purity. The distinction between "pure tree structure" and "ID references for cross-element relationships" is crucial to understand before encountering the timewalker, which must resolve those references in musical time order.

**7. [ADR-0012: Persisting Pieces](../ADRs/0012-Persisting-Pieces.md)**

How the pure tree and its integer ID references are kept intact through serialisation and deserialisation, and, in §4 "Piece Identity: UUID as Primary Key", what identifies a piece: a string UUID embedded in the piece data, assigned when the piece is created and unchanged by renaming, restructuring or new content, so that anything outside the piece can hold it and trust it; a copy made by Save As receives a fresh one. The Piece Manager's registry is keyed by it, and its catalogue maps it to where the piece is stored.

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

**11. [ADR-0023: Shared Model Contracts](../ADRs/0023-Shared-Model-Contracts.md)**

Where that single data model lives. Every `defrecord`, multimethod interface, predicate and trait is in `shared/`, which is the engine; the backend project wraps it with the server, piece management and persistence, the frontend project with the user interface. Because both sides use the same types, a value crosses gRPC and comes back with its type intact, and a plugin defines its models once. Read directly after ADR-0001: it is what "one data model" means in the source tree.

> **What you should understand at the end of Phase 2:** That Clojure's properties are not incidental to the problem. That STM provides ACID correctness for all musical mutations. That "frontend" and "backend" are separable but share a single data model.

---

## Phase 3 — Addressing and Operating on the Model

*How do you name a location in the tree, and how do you work with what you find there?*

This phase is the core operational toolkit. Everything above and below in this reading order uses these concepts.

**12. [ADR-0008: VPDs](../ADRs/0008-VPDs.md)**

Vector Path Descriptors are compact vectors that address locations in the musical hierarchy. `[:m 0 1 0 3 0]` means: musician 0, instrument 1, staff 0, measure 3, voice 0. Read the rationale section — the alternatives considered (string paths, integer indexing, object references) illuminate why VPDs specifically were chosen. VPDs serialise cleanly across gRPC; object pointers do not.

**13. [Guide: VPDs.md](VPDs.md)**

The practical guide to VPDs. Focus on: VPD form vs navigator form, the `vpd/compact` and `vpd/navigator` conversion functions, and the best practices section. The crucial point: compact form is what you write in all application code; navigator form is what Specter uses internally. Conversion is idempotent. You almost never touch navigator form directly.

**14. [Guide: POLYMORPHIC_API_GUIDE.md](POLYMORPHIC_API_GUIDE.md)**

How operations dispatch on first-argument type — VPD vs direct object. This is where the tree structure, VPDs, and STM converge into a single API. The dual-mode design has a concrete reason: VPD operations automatically establish STM transactions and work across gRPC; direct object operations require manual transaction management. The "Twinkle Twinkle Little Star" example that opens the guide, under "The Proof of the Pudding", is worth reading slowly — it shows what the full API looks like in practice, from creating a piece through adding notes and slurs within a single `dosync`.

**15. [Guide: PIECE_MANAGER_GUIDE.md](PIECE_MANAGER_GUIDE.md)**

How pieces are stored, retrieved, and referenced. UUID-based identification in production. The relationship between piece-id, piece-ref, and piece-object. The prerequisites listed in the guide itself confirm this is the correct moment to read it: VPDs and basic Clojure knowledge needed, nothing more.

**16. [ADR-0016: Settings](../ADRs/0016-Settings.md)**

Configuration for any entity in either hierarchy, from a piece to a single pitch. `defsetting` generates `get-` and `set-` accessors indistinguishable from those of a record slot, so the polymorphic API covers settings with no new vocabulary; values are stored in a `:settings` map that holds only what differs from the default, are validated when written, and are addressed by VPD like everything else. Piece settings — accidental behaviour, the default music font and the glyphs drawn from other fonts, staff spacing and the rest — are of this kind.

> **What you should understand at the end of Phase 3:** How to address any element in any piece using a VPD. How the polymorphic API dispatches. How pieces are stored and retrieved. How to read and construct any API call in the codebase.

---

## Phase 4 — Temporal Traversal

*How do you process music in time rather than in tree order?*

**17. [ADR-0014: Timewalk](../ADRs/0014-Timewalk.md)**

The problem statement alone is worth reading carefully. A polyphonic score requires that all events at measure N are processed before any events at measure N+1, across all voices and all instruments simultaneously. This ordering cannot be derived from the tree structure directly — the tree is spatial, not temporal. The timewalk system makes temporal coordination the foundational abstraction for all musical computation. This is a genuine paradigm shift.

**18. [Guide: TIMEWALKING_GUIDE.md](TIMEWALKING_GUIDE.md)**

The practical guide. The key insight stated near the top: timewalking transforms a piece into a stream of musical events in time order. Rather than navigating musicians → instruments → staves → measures → voices manually, you get a single stream where everything flows in musical time. The guide teaches both the Ooloi API and Clojure transducers simultaneously — if you understand one, you understand the other.

Pay attention to the streaming vs materialisation distinction. Use streaming (constant memory) for cache refresh, endpoint search, and data export. Materialise into a vector only when random access to the full result set is required.

The timewalker returns three-element tuples: `[item vpd position]`. The VPD is the item's address in the hierarchy; the position is its temporal location. Both are essential to operations that must locate an item and refer to it afterwards.

**19. [ADR-0011: Shared Structure](../ADRs/0011-Shared-Structure.md)**

The structure shared here is that of connecting elements: slurs, ties, glissandi and hairpins, which refer across the tree by integer ID and need the items under their span collected before they can be drawn. The ADR settles that this is done through the timewalker, with transducers and boundary VPDs limiting the work to the relevant section, and no intermediate collections. ADR-0013 (Phase 7) builds on it.

> **What you should understand at the end of Phase 4:** That the timewalk is the fundamental operation for anything processing music sequentially — MIDI generation, layout, accidental resolution, tie and slur endpoint resolution. How to compose transducers over the resulting stream. When to stream and when to materialise.

---

## Phase 5 — Network and Deployment Architecture

*How does the frontend communicate with the backend, and why does that not cost anything?*

**20. [ADR-0002: gRPC](../ADRs/0002-gRPC.md)**

Why gRPC with Java interop rather than a native Clojure solution. The architectural principle that matters most: the gRPC interface is 1:1 with `api.clj`. There is no separate "network API" distinct from the local API. The same functions accessible locally are accessible remotely. Hundreds of methods are exposed without manual implementation through the `ExecuteMethod` pattern.

**21. [ADR-0018: API-gRPC Interface and Events](../ADRs/0018-API-gRPC-Interface-and-Events.md)**

The `ExecuteMethod` unified endpoint and `OoloiValue` schema. Dynamic function resolution via `ns-resolve` means new API methods are immediately available to remote callers without regenerating stubs. Hot-installable plugins work for the same reason. The server-to-client event streaming architecture is also introduced here — the bidirectional communication model that makes collaborative consistency possible.

**22. [ADR-0019: In-Process gRPC Transport Optimisation](../ADRs/0019-In-Process-gRPC-Transport-Optimization.md)**

A gRPC call over localhost TCP costs 300 or more microseconds; the same call over the in-process transport costs 4 to 8, by the benchmarks the ADR cites. In combined deployments, gRPC transport is in-process, not networked. The frontend-backend separation is therefore architecturally clean (all mutation goes through the API; the frontend never touches the musical model directly) without performance penalty. This ADR is short and resolves what would otherwise feel like a paradox.

**23. [ADR-0046: Reference-Passing In-Process Transport](../ADRs/0046-Reference-Passing-In-Process-Transport.md)**

Completes the optimisation chain ADR-0019 began. For in-process gRPC, custom marshallers pass Clojure references directly between frontend and backend rather than converting to and from protobuf. The final conversion overhead vanishes. Read directly after ADR-0019 — together they make the case that "gRPC everywhere" is not a performance compromise for the combined-desktop case, it is the baseline.

**24. [ADR-0022: Lazy Frontend-Backend Architecture](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md)**

The complete data synchronisation model. Three layers: events tell clients which local objects are stale; gRPC requests provide fresh data when needed; local shared API provides fast access to cached data. The key insight: invalidation events contain only structural addresses (VPD regions), not content. The frontend knows *that* something has changed and *where*, then requests fresh data on demand. This is lazy evaluation applied to distributed state.

**25. [ADR-0052: Change Detection and Event Generation](../ADRs/0052-Change-Detection-and-Event-Generation.md)**

Where an invalidation comes from. Change is detected at one point, the single write funnel every VPD mutation passes through. A mutation of a structural entity — piece, musician, instrument, staff, layout — emits `:piece-structure-changed`, deferred to commit and coalesced to exactly one event per outermost transaction by `with-coalesced-structural-change`, and the subscribed window refetches the piece's structural snapshot. A change to the music itself emits no structure event; it reaches the frontend as invalidations from the formatting pipeline. The ADR also defines the per-piece dirty flag. Read beside ADR-0022: that ADR says what an event carries, this one says how the backend decides to emit it.

**26. [ADR-0015: Undo and Redo](../ADRs/0015-Undo-and-Redo.md)**

Undo is divided by architectural boundary. Backend undo keeps one stack per resource — each piece, the Instrument Library — shared by every client subscribed to it; each frontend keeps its own stack for local UI state; and the backend has no application settings to undo at all. The user sees one Undo and one Redo, routed by comparing timestamps. Undo capture keys on the same outermost-transaction boundary as ADR-0052's coalescing, so a gesture composed into one `SRV/atomic` batch is one undo step. The section ["Continuous Gestures: One Undo Step"](../ADRs/0015-Undo-and-Redo.md#continuous-gestures-one-undo-step) covers a drag committed as it moves: every update is a transaction of its own, carries the gesture's identifier, and amends the entry the first update pushed. Operational transformation is a permanent non-goal, and the ADR says why.

> **What you should understand at the end of Phase 5:** That frontend and backend share one data model with no conversion overhead. That gRPC is the communication mechanism but carries no overhead in combined deployments. That event-driven invalidation is how collaborative consistency is maintained without polling. That a change is detected in one place, reported once per transaction, and undone as one step per gesture. The first concrete production application of this pattern to a global singleton (rather than piece data) is the Instrument Library — covered in Phase 9.

---

## Phase 6 — Musical Semantics

*The domain-specific decisions that determine what notes mean.*

**27. [ADR-0026: Pitch Representation and Operations](../ADRs/0026-Pitch-Representation-and-Operations.md)**

String-based canonical form: `"C#4"`, `"Bb3-75"` (75 cents below B♭3), `"C###4"`. Transposition is through transposer factories, chromatic or diatonic, and instrument definitions carry their transposition from sounding to written pitch as such a specification. Diatonic transposition never simplifies spelling: double and triple accidentals are preserved exactly. The ADR treats round-trip integrity under diatonic transposition carefully, because it is harder to guarantee than it initially appears.

**28. [ADR-0033: Time Signature Architecture](../ADRs/0033-Time-Signature-Architecture.md)**

Composite metres, irrational time signatures, the descriptor string format. Establishes rational arithmetic as the internal representation — no floating-point approximations anywhere in the temporal model.

**29. [ADR-0034: Key Signature Architecture](../ADRs/0034-Key-Signature-Architecture.md)**

Standard (major, minor, modal), keyless, mixed-accidental, per-octave variation (Bartók-style), and microtonal key signatures. Pitches are stored as sounding pitches throughout the system, so key signatures are purely presentational constructs that guide the engraving engine. The critical architectural point: key signatures guide when accidentals are printed; they do not alter stored pitches. This separation between sounding pitch (always stored exactly) and notated pitch (determined at engraving time) is what makes the next ADR possible.

**30. [ADR-0035: Remembered Alterations](../ADRs/0035-Remembered-Alterations.md)**

The first "impossible problem turned straightforward" result. Read the problem statement carefully. Accidentals have temporal memory within measures; that memory applies to the musical timeline, not to visual order; multi-staff instruments require a single shared accidental state across all staves. The solution is deterministic via the timewalker. The same input always produces the same accidental decisions, regardless of layout, staff count, or rendering order.

This is worth understanding thoroughly. It is the first confirmation that the architecture turns a question of judgement into a computation: the ADR states that its algorithm needs no heuristics or special-case logic beyond house-style settings. It establishes that five user-configurable settings cover the full domain of legitimate taste decisions in accidental rendering. That small configuration surface is a consequence of the algorithm's completeness: where the correct answer can be computed, configuration is unnecessary.

> **What you should understand at the end of Phase 6:** How pitches are represented and why key signatures are presentational. That remembered alterations are deterministically computed via the timewalker — no approximations; adjustments, where supported, are for taste rather than necessity.

---

## Phase 7 — The Rendering Pipeline

*The crown of the architecture. Where all prior foundations become consequence.*

**31. [Blog: "Six Degrees of Unification"](https://www.ooloi.org/home/six-degrees-of-unification)**

Peter's own description of the six-stage pipeline in prose. Read this before the ADR itself; it gives the conceptual shape — the fan-out/fan-in pattern, system and page breaking as dynamic programming over stable inputs, connecting elements resolved against fixed atom positions — without the engineering detail.

The earlier post, ["The Rendering Pipeline: Ooloi's Core Architecture"](https://www.ooloi.org/home/the-rendering-pipeline-oolois-core-architecture) (23 September 2025), is the historical record of the pipeline as first specified, in five stages. It is superseded by the six-stage design; read it as history, not as a description of the current pipeline.

**32. [ADR-0028: Hierarchical Rendering Pipeline](../ADRs/0028-Hierarchical-Rendering-Pipeline.md)**

Six stages:

- **Stage 1** (fan-out): Atom formation. Each measure forms its atoms and their contours, the outlines Stage 2 measures. Results are immutable and persist until the measure's content changes, or the clef the next measure begins with, which the measure draws before its closing barline, or a font or a setting that changes glyph geometry, such as a ledger line's minimum protrusion.
- **Stage 2** (fan-in): Raster and springs. The staves' rhythms are merged into one raster of columns, every gap between columns becomes a spring with an ideal and a blocking width, and each measure stack is summarised for Stage 3: its minimum, natural and ordinary ideal widths as sums, its prestretched gaps, its gutter, and its no-break width, what the boundary after it adds only when that boundary falls inside a system.
- **Stage 3** (single): System breaking. Dynamic programming over the measure sequence. Determines optimal system breaks and each system's adjustment ratio.
- **Stage 4** (fan-out): Atom positioning. Fills each system's width through its gaps' springs: at the system's adjustment ratio under compression, and under stretching with the room shared out by the stretch dial, which never moves a break; computes absolute coordinates.
- **Stage 5** (fan-out per musician): Spanners and margins. Ties, slurs, beams, hairpins, and glissandi are drawn once final positions are known. Gutter decorations (courtesy accidentals, tie continuations) are added here. System heights become definitive at this stage.
- **Stage 6** (single): Page breaking. Dynamic programming over the system sequence using actual heights from Stage 5.

The gutter model is the detail that enables Stage 3's breaks to be provably optimal for its cost function, given its inputs: every measure stack carries its summary, the additional space it needs when appearing at system start, and the no-break width of the boundary after it — the start part of a repeat barline, and cancellation naturals where house style omits them at a system start, counted only when the boundary falls inside a system — and the preamble and postamble of every candidate system are known before breaking, so Stage 3 has complete information when making break decisions.

Plugins participate at two points: the atom hook in Stage 1, which produces an atom's contours and cached paintlist, and any constraints spanning several columns that the element implies, and the spanner hook in Stage 5, which generates connecting elements from final atom positions. Core notation elements and plugin-defined elements use identical interfaces.

**33. [ADR-0037: Measure Distribution Optimisation](../ADRs/0037-Measure-Distribution-Optimization.md)**

The second result of the same kind as ADR-0035's. Once Stages 1–2 have resolved vertical coordination and collision detection, measure distribution is a segmentation problem in the Knuth–Plass family — TeX's paragraph-breaking algorithm, published in 1981. Stage 3's cost is Knuth–Plass's demerits, from the badness of each system's adjustment ratio; its glue is Hegazy and Gourlay's springs (1987), each gap carrying an ideal width and a blocking width, the distance at which the atoms on either side collide. In the ADR's words, "a spring is glue with a blocking width" ([Relationship to Knuth-Plass](../ADRs/0037-Measure-Distribution-Optimization.md#relationship-to-knuth-plass)). The baseline cost is not the full Knuth–Plass cost model, and the rest of it is available as extensions. The reduced problem is textbook dynamic programming on a one-dimensional sequence with separable costs.

The ADR is explicit that the algorithm is not novel, and its Prior Art section traces optimal breaking in music back through earlier systems. What the pipeline provides is the reduction: by separating vertical coordination and collision detection from the distribution decision, it leaves Stage 3 a one-dimensional sequence of stable scalar inputs, on which the DP runs once per pass. User control enters the same model as input rather than as correction: a dragged barline either transfers ideal width between its measure and the next or sets its measure's width, and a frozen system keeps its breaks ([Editorial Control Mechanisms](../ADRs/0037-Measure-Distribution-Optimization.md#editorial-control-mechanisms), [Frozen Systems](../ADRs/0037-Measure-Distribution-Optimization.md#frozen-systems)).

[Blog post "Twice"](https://www.ooloi.org/home/twice) presents the two results together, and the architectural properties they share: immutable data, rational arithmetic, explicit stage boundaries, semantic determinism before layout.

**34. [ADR-0055: Horizontal Spacing](../ADRs/0055-Horizontal-Spacing.md)**

How Stages 1 and 2 produce the springs ADR-0037 fits. Every horizontal engraving rule takes one of four forms — an ideal width, a blocking width, a constraint spanning several columns, or a variant at a break — and reaches Stage 3 in that form; a rule that fits none of them is a design problem, never a pass added after Stage 2. Atoms have origins, where the springs attach, and contours built from SMuFL bounding boxes and cut-outs; ideal widths come from an exact duration table, interpolated between its entries, in which space grows far more slowly than duration; blocking widths come from the contours and a clearance table. Its Context section asks for four sections of ADR-0037 to be read first — Gaps as Springs, the Interface Contract, Constraints Spanning Several Columns and its design principle — which is why it follows here. The architecture is decided; many values are open, each an entry in its Open Items, to be settled by measuring printed engravings.

**35. [ADR-0038: Backend Authoritative Rendering and Terminal Frontend Execution](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)**

The frontend is *terminal*: it executes rendering decisions but never renegotiates, refines, or reinterprets them. The litmus test: discard all frontend rendering state, regenerate from backend → identical output. This is the property that makes distributed collaboration and multiple frontend implementations consistent without complex synchronisation. The ADR also specifies how that execution is done without the window ever stuttering: three cache levels — paintlists, Skija Pictures, raster tiles — a score viewport built of tiles that JavaFX composites, so its UI can overlap the score, and a threading model in which nothing that renders runs on the JavaFX Application Thread.

**36. [ADR-0013: Slur Formatting](../ADRs/0013-Slur-Formatting.md)**

Stage 5 in practice. Point collection via timewalking; shape determination via hull and Bézier; variable-thickness rendering following copper-plate engraving aesthetics. The problem statement is worth reading even if you are not implementing spanners, because it shows how the pipeline's prior stages provide complete information to Stage 5: atom positions, slur start and end points, items under the slur's span — all resolved before Stage 5 begins.

Read its section ["The Pipeline Is One-Way, and What Stage 5 May Change"](../ADRs/0013-Slur-Formatting.md#the-pipeline-is-one-way-and-what-stage-5-may-change) closely. Stage 5 sends nothing back upstream; it may make bounded, non-recursive adjustments to what earlier stages produced, such as a stem length or an articulation's offset, but it never changes note spacing. The whitespace available to a slur is computed from Stage 4's geometry, and Stage 5's authority to adjust that geometry is what lets it enlarge the whitespace when a slur needs it. ADR-0055 §[Atoms and Contours](../ADRs/0055-Horizontal-Spacing.md#atoms-and-contours) states the horizontal side of the same rule: an element expands only into space the springs have already given.

> **What you should understand at the end of Phase 7:** The six pipeline stages and what each one produces. How Stages 1–2 turn engraving rules into springs, and why that creates the preconditions for Stage 3's optimality. Why the frontend is terminal. How Stages 1–2 reduce measure distribution to the Knuth–Plass problem formulation over Hegazy and Gourlay's springs.

---

## Phase 8 — The Frontend

*How users interact with it, and how the UI is structured.*

**37. [FRONTEND_README.md](../READMEs/FRONTEND_README.md)**

The component overview: event-bus, ui-manager, grpc-clients, event-router, fetch-coordinator, frontend-undo-manager. Short; establishes vocabulary before the architecture guide.

**38. [ADR-0005: JavaFX and Skija](../ADRs/0005-JavaFX-and-Skija.md)**

JavaFX for the user interface and Skija, the Java bindings for Skia, for 2D rendering and for printing, so that screen and paper come from one rendering engine. Two later sections are worth knowing early: the position on native platform integration (where the toolkit falls short of a platform's convention, as with the macOS application menu, Ooloi goes to the platform layer rather than approximate it), and the copper-plate principle that strokes end round by default throughout.

**39. [ADR-0031: Frontend Event-Driven Architecture](../ADRs/0031-Frontend-Event-Driven-Architecture.md)**

Three event layers: the frontend event bus (category-based pub/sub backed by the shared Claypoole thread pool), the backend event router (categorises and batches backend events for the bus), and JavaFX (UI input only). The threading model and handler isolation guarantee that a slow subscriber cannot block the publisher, and that one handler's failure does not affect others.

**40. [ADR-0039: Localisation Architecture](../ADRs/0039-Localisation-Architecture.md)**

GNU gettext `.po` files, `tr-declare` as first-class mechanism for translation key visibility, instant locale switching via event-driven architecture. Short. The critical architectural point: locale is application state, not startup configuration. The [Translator's Guide](TRANSLATORS_GUIDE.md) is the other side of this ADR — not part of this reading path, since it assumes no Clojure, but worth knowing exists before you add a user-facing string.

**41. [ADR-0042: UI Specification Format](../ADRs/0042-UI-Specification-Format.md)**

Pure-data UI specification. cljfx specs, setting declarations, command descriptors. The per-window reactive renderer pattern, which every application window uses. The absolute invariant: the UI Manager manages Stages (outer window shell); window files manage content nodes (inner reactive content). These responsibilities never overlap.

**42. [ADR-0043: Frontend Settings](../ADRs/0043-Frontend-Settings.md)**

Lazy loading, atomic file writes, closed mutation surface. Short. Read in conjunction with ADR-0042.

**43. [ADR-0053: Piece Window and Piece Preferences](../ADRs/0053-Piece-Window-and-Piece-Preferences.md)**

The central workspace, where Phases 5 and 8 meet in one component. A Piece Window is a live view of, and workspace over, one piece the backend owns; it holds no authoritative state and passes ADR-0038's litmus test. It is operated by direct manipulation, with no modes, and raises a dialog only to confirm an irreversible loss. Its two panes, Musicians and Layouts, are assembled by drag and drop, and each gesture is composed into one `SRV/atomic` batch — one `:piece-structure-changed` event, one undo step. The ADR also covers the window title with its dirty and shared marks, the per-piece Piece Preferences window, and the windows a piece owns, which close with it and reopen with it.

**44. [ADR-0054: Automatic Semantic Naming and Numbering of Musicians and Instruments](../ADRs/0054-Automatic-Semantic-Naming-and-Numbering-of-Musicians-and-Instruments.md)**

Numbering as a consequence of the gesture rather than a task it leaves behind. When an instrument is dropped, cloned, deleted or reordered, the frontend composes the numbering with the structural edit and submits them together; the API itself never renumbers anything. Instruments are counted piece-wide by exact name, transposition included, so a Horn in F and a Horn in D are never numbered together, and a musician takes its number from its main instrument, which is why the first flautist can be Flute 1 and Piccolo 2 at once. A compact example of domain knowledge doing the user's clerical work.

**45. [Guide: FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)**

Read this after the individual ADRs, so that specific concepts arrive with context. The guide synthesises window lifecycle, event architecture, the rendering pipeline's frontend side, fetch coordination, localisation, and collaboration.

Pay particular attention to the section addressed to "Frontend Framework Developers." It explains directly why this architecture inverts React instincts: the frontend holds no semantic authority; that subordination is precisely what makes distributed collaboration, transport independence, and first-class plugin composition possible without architectural compromise. The guide also explains why UI structure is expressed as pure data rather than imperative widget construction — plugins must be able to describe UI across process and network boundaries, which requires data, not objects.

> **What you should understand at the end of Phase 8:** The event architecture. How windows are lifecycle-managed. Why UI structure is expressed as pure data. Why plugins can contribute windows, settings, and commands without importing JavaFX.

---

## Phase 9 — Advanced and Specialised Topics

*Read in any order after Phase 8, driven by interest.*

These documents are not required for understanding Ooloi's architecture. They provide depth in specific areas.

**[ADR-0003: Plugins](../ADRs/0003-Plugins.md)** — The minimal core / plugin ecosystem design philosophy. Why commercial closed-source plugins are intentionally supported. Why core notation elements and plugin elements use identical interfaces.

**[ADR-0027: Plugin-Based Audio Architecture](../ADRs/0027-Plugin-Based-Audio-Architecture.md)** — The decision that the Ooloi core has no audio or MIDI functionality whatsoever: playback, audio processing, and MIDI input and output are delegated to frontend plugins, and the backend receives only API commands, whatever the input source, so it can be deployed without audio capability, in the cloud as anywhere else. The context section recounts Igor Engraver's "DNA soup" of MIDI channel allocation, and why plugin hosting on the client removes the protocol constraints that required it.

**[ADR-0041: Ooloi Virtual Instrument Definition (OVID)](../ADRs/0041-Ooloi-Virtual-Instrument-Definition-OVID.md)** — How playback plugins control virtual instruments: declarative YAML files mapping a canonical technique taxonomy onto each library's own controls, listing selection mechanisms in order of preference, requiring loudness and onset calibration, and supplying fallback generators for missing articulations. Control stays inside the plugin host; Ooloi outputs no MIDI. OVID generalises Igor Engraver's Synth Matrices, user-editable descriptions of MIDI synthesisers, to virtual-instrument libraries. Read after ADR-0027.

**[ADR-0032: Flow Mode](../ADRs/0032-Flow-Mode.md)** — Stateful, modal keyboard input, carried over from Igor Engraver: a mode such as staccato stays active across entries until changed, slurs and hairpins extend as notes are added, and a MIDI keyboard supplies pitch while the computer keyboard handles rhythm and articulation. Flow Mode is the primary input method; Click Mode, driven by the mouse, exists for familiarity and for placing esoteric elements.

**[ADR-0044: MIDI Input Library and Boundary Architecture](../ADRs/0044-MIDI-Input-Library-and-Boundary-Architecture.md)** — How MIDI input for note entry is received: `javax.sound.midi` as the single API on every platform, CoreMidi4J as the service provider on macOS, and a hard boundary that accepts note-on, note-off and the sustain pedal (CC64) and discards everything else before it reaches the system. MIDI input exists for note entry in Flow Mode and for nothing else; a plugin that needs more implements its own receiver.

**[Guide: MIDI_IN_OOLOI.md](MIDI_IN_OOLOI.md)** — Describes the design ADR-0044 and ADR-0032 decide, which is not yet built: why there is no MIDI output, what MIDI input is for, how its events flow, device management, and the extension points for plugins.

**[ADR-0009: Collaboration](../ADRs/0009-Collaboration.md)** — The decision that multi-client editing follows from the architecture already in place — client-server separation, STM transactions, gRPC streaming — rather than being built beside it, and that it costs single-user work nothing. Short. ADR-0036 specifies the sessions themselves.

**[ADR-0036: Collaborative Sessions and Hybrid Transport](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)** — How a standalone application transitions dynamically to host or guest mode without restart. Switching is asymmetric: before connecting to a remote backend, every local piece must be closed, because local pieces are the user's own unsaved work; disconnecting from a remote backend, or moving from one remote host to another, has no precondition, because a guest holds only a view of the host's work.

**[ADR-0020: TLS Infrastructure and Deployment Architecture](../ADRs/0020-TLS-Infrastructure-and-Deployment-Architecture.md)** — TLS for the gRPC server and client: off by default and enabled by one switch, with self-signed certificates generated in pure Java when none exist, custom certificates for production, and one-way TLS on the client with automatic certificate discovery.

**[ADR-0021: Authentication](../ADRs/0021-Authentication.md)** — Pluggable authentication: JWT tokens carried in gRPC metadata, issued by providers that are chosen by configuration and managed as Integrant components, so that one protocol serves deployments from a single teacher's machine to an institution's directory service.

**[ADR-0040: Single Authority State Model](../ADRs/0040-Single-Authority-State-Model.md)** — The formal statement of state ownership rules across the system.

**[ADR-0045: Instrument Library](../ADRs/0045-Instrument-Library.md)** — The first non-piece singleton in the architecture: a server-side instrument registry with optimistic locking, lazy frontend caching, and invalidate-only event synchronisation. Concrete proof that the single-authority model scales beyond piece data to any global state. The bundled default library covers the full orchestral repertoire from Bach to Messiaen, together with the mechanisms for users to extend it permanently with instruments of their own.

**[Guide: INSTRUMENT_LIBRARY_CATALOGUE.md](INSTRUMENT_LIBRARY_CATALOGUE.md)** — The bundled library listed in full: 315 instruments in eight families, in up to four language editions (Italian, German, French, English), 1,176 entries in all, with each transposing instrument's interval given relative to written pitch. A reference rather than a reading.

**[ADR-0030: MusicXML](../ADRs/0030-MusicXML.md)** — Import and export as a first-class plugin, preserving musical meaning rather than graphical approximation.

**[ADR-0006: SMuFL](../ADRs/0006-SMuFL.md)** — SMuFL as the sole font standard for musical symbols: glyphs at standard code points, with JSON metadata describing their bounding boxes, anchors and engraving defaults. Bravura is the default and fallback font, and eleven SMuFL fonts are bundled. That metadata is also what ADR-0055's contours are built from. Short; read before the three ADRs that follow.

**[ADR-0047: Font Management](../ADRs/0047-Font-Management.md)** — Font registry with dual Skija/JavaFX registration, bundled-first loading, local font discovery across the OS font directories, exact version matching for pieces (the TeX model: a piece stores the font version it was laid out against, and opening it later requires that exact version to reproduce the layout), and multi-user font consistency for collaborative sessions. Read together with ADR-0006, ADR-0048 and ADR-0049 — they form the font-and-glyph cluster.

**[ADR-0048: SMuFL Glyph Selection Architecture](../ADRs/0048-SMuFL-Glyph-Selection-Architecture.md)** — The general mechanism for selecting notation glyphs from the SMuFL specification. Logical keywords resolve to SMuFL classes, which resolve to glyphs through a three-level cascade (house style → piece → local). Each glyph comes from the piece's default music font or, where the user has chosen, from another SMuFL font, and is offered only from fonts that provide it. The additional-glyphs mechanism lets individual glyphs be added to a class without pulling in a whole mixed range. ADR-0049 is the first concrete implementation of this architecture.

**[ADR-0049: Clef Registry](../ADRs/0049-Clef-Registry.md)** — The first application of ADR-0048. Clefs become a registry mapping keywords to structured definitions (class, line or space, pitch, optional additional glyphs), with 17 defaults covering standard practice and full support for user-defined clefs (French violin clef, sub-bass clef, mensural C clefs, anything else). A flat keyword such as `:treble` conflates glyph, staff position and octave transposition in one name, and a fixed set of such keywords can express no clef outside it; factoring the name into orthogonal fields makes any clef a definition rather than a new keyword, while the default keywords keep their names.

**[ADR-0050: Platform Support Policy](../ADRs/0050-Platform-Support-Policy.md)** — The authoritative answer to "what does Ooloi run on?" macOS on Apple Silicon only, Windows and Linux on `x86_64`, with Linux `aarch64` planned. The reasoning section is the most interesting part: the four-case Rosetta matrix, the pathological behaviour of JVMs under binary translation (every JIT output invalidates Rosetta's translation cache), the build-on-target discipline that follows from `jlink`, and Apple's deprecation timeline. Read when you need to understand why Ooloi's distribution looks the way it does.

**[ADR-0017: System Architecture](../ADRs/0017-System-Architecture.md)** — Integrant for component lifecycle: every component has an `init-key` and a `halt-key!`, the deployment mode is chosen by configuration rather than code, start-up failures carry classified exit codes, and a partly started system is cleaned up. It also specifies how a failure on a thread nobody is waiting on is surfaced rather than lost. Read before the Integrant guide.

**[Guide: INTEGRANT_COMPONENTS.md](INTEGRANT_COMPONENTS.md)** — Integrant component lifecycle, the three-project wiring asymmetry, the combined system dependency graph, startup sequence, component wiring checklist (including two unintuitive requirements), and the full testing macro reference. Essential reading before adding any new component to the system.

**[Guide: OOLOI_SERVER_ARCHITECTURAL_GUIDE.md](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** — The backend server as a whole: deployment scenarios, the single message type that carries every call, STM transactions inside the server process, real-time event streaming, connection management and error handling.

**[Guide: GRPC_COMMUNICATION_AND_FLOW_CONTROL.md](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** — Why API requests run without flow control while event streaming needs it, the queue-based delivery that provides it, and how STM and gRPC complement each other. The guide-level companion to ADR-0024.

**[Guide: GRPC_STREAMING_THREADING_GUIDE.md](GRPC_STREAMING_THREADING_GUIDE.md)** — Why server streaming that works over the in-process transport can fail over the network, where HTTP/2 flow control and Netty's threading are enforced, and the drainer pattern that respects them: per-client queues, backpressure, graceful disconnect, and client registration in the connection registry, one atom.

**[Guide: ADVANCED_CONCURRENCY_PATTERNS.md](ADVANCED_CONCURRENCY_PATTERNS.md)** — STM fundamentals (why atoms and transients cannot coordinate updates, and why `alter` is never used), then parallel measure formatting with coordinated updates, and patterns for collecting and integrating the results. A composed operation that must reach clients as one change goes through `with-coalesced-structural-change`, or `SRV/atomic` from a client, as ADR-0052 specifies.

**[ADR-0007: Nippy](../ADRs/0007-Nippy.md)** — Nippy as the serialisation format, with custom handlers for Ooloi's types, and the reason reads may run concurrently while writes are deliberately serialised. Short.

**[ADR-0029: Global Hash-Consing](../ADRs/0029-Global-Hash-Consing.md)** — Identical immutable musical objects share one instance: a pitch, rest or chord carrying only articulations, and the articulations themselves. Anything in a relationship (an object with an endpoint ID), or carrying any attachment other than an articulation, is left unshared. Memory falls, and a piece read back from disk shares them again, since each thawed object passes through the same caching constructor.

**[ADR-0051: Filesystem Operations — Real and Virtual](../ADRs/0051-Filesystem-Operations-Real-and-Virtual.md)** — The backend contract beneath Open, Save, Save As and the piece picker. No operating-system path crosses the boundary: directories and files are named by opaque tokens the backend mints, so path disclosure and traversal are impossible by construction. A small operation set serves real storage and virtual hierarchies alike, over either transport, and reports failure as typed data.

**[Guide: PIECE_PERSISTENCE_GUIDE.md](PIECE_PERSISTENCE_GUIDE.md)** — How a piece is saved and opened. A save takes its snapshot the moment it is asked for and queues the write on a single-thread save queue, so writes happen one at a time in the order asked while editing continues; opens run concurrently on the shared pool, bounded so that a core is always left for the interface. Failures in either direction are typed data. Also the three I/O backends and how hash-consing survives the round trip.

---

## The Three Gateway Documents

The Ooloi blog identifies three documents as the main conceptual gateways into the system's internals. Having followed Phases 0–8, you will arrive at all three with full context:

- **[Guide: TIMEWALKING_GUIDE.md](TIMEWALKING_GUIDE.md)** — Teaches temporal stream processing and Clojure transducers through musical examples. The transformation of spatial structure (the score) into temporal flow (the music) is the central concept.
- **[Guide: POLYMORPHIC_API_GUIDE.md](POLYMORPHIC_API_GUIDE.md)** — Teaches the operational model: tree, VPDs, STM, and API dispatch working together. The practical entry point for writing code.
- **[Guide: FRONTEND_ARCHITECTURE_GUIDE.md](FRONTEND_ARCHITECTURE_GUIDE.md)** — Teaches the complete frontend model and its relationship to the backend. The structural argument for terminal execution is made here in full.

The guides are not the entry point. They are where earlier foundations converge into practical synthesis. A newcomer who arrives at them in numeric order will find them opaque. A newcomer who follows this path will find them clear.

---

## What the Numeric ADR Order Conceals

The chronological ordering reflects when decisions were made, not their conceptual difficulty. The earliest decisions (ADR-0000 through ADR-0014) establish the foundations — and they happen also to be the most readable documents in the corpus. They answer "why Clojure," "why a pure tree," "why VPDs," "why timewalking." These are exactly the documents a newcomer most needs first.

The documents hardest to read without prior context — ADR-0022 (lazy frontend-backend synchronisation), ADR-0024 (gRPC flow control), ADR-0025 (server statistics), ADR-0036 (collaborative sessions) — sit in the middle of the numerical range. They are detailed implementation decisions for contributors working on those specific systems. They can be deferred indefinitely by anyone whose primary interest is in the musical domain, the rendering pipeline, or the frontend architecture.

The later ADRs are often the most sophisticated: ADR-0035 (remembered alterations), ADR-0037 (measure distribution), ADR-0038 (terminal frontend execution), ADR-0055 (horizontal spacing). These are some of the most important documents in the corpus. They require the full context of everything before them to be understood. Following this path ensures you arrive at them prepared.
