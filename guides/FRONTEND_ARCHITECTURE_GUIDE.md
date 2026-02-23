# Ooloi Frontend Architecture Guide

> Together with the [Polymorphic API Guide](POLYMORPHIC_API_GUIDE.md) and the [Timewalking Guide](TIMEWALKING_GUIDE.md), this document forms a central gateway into Ooloi's architecture. The Polymorphic API Guide explains how musical structures are exposed and manipulated through Ooloi's polymorphic command surface. The Timewalking Guide explains how musical structure is traversed and transformed. This guide explains how those structures become an interactive, distributed, GPU‑accelerated application.
>
> This guide walks through the Ooloi frontend architecture: its principles, boundaries, event flow, rendering model, window lifecycle, settings system, localisation discipline, and collaboration context handling.
>
> It is written for architects and experienced developers — including those who are not Clojure programmers — who genuinely want to understand how Ooloi works from the inside. It assumes technical maturity and does not dilute architectural ideas, but it also does not retreat into abstraction. The aim is clarity without distance, and precision without coldness.
>
> This guide is a structural and conceptual exploration of the frontend architecture. It complements the formal ADRs and plugin documentation by focusing on how the pieces fit together, how they behave in practice, and what kind of discipline makes them cohere. Low‑level details are defined formally in the relevant ADRs, which are linked throughout; here we stay at the structural level and examine how those formal decisions become a working, coherent system.

---

## Table of Contents

1. [The Frontend Is Not a Viewer](#1-the-frontend-is-not-a-viewer)
2. [Opening a Window (Minimal Example)](#2-opening-a-window-minimal-example)
3. [The Window Lifecycle Invariant](#3-the-window-lifecycle-invariant)
4. [Pure Builders and Materialisation](#4-pure-builders-and-materialisation)
   - [4.1 Two-Level Abstraction](#41-two-level-abstraction)
   - [4.2 Why This Separation Exists](#42-why-this-separation-exists)
   - [4.3 cljfx Spec Over Java Interop](#43-cljfx-spec-over-java-interop)
   - [4.4 Custom cljfx Component Functions](#44-custom-cljfx-component-functions)
   - [4.5 Per-Window Reactive Renderer](#45-per-window-reactive-renderer)
5. [Event Architecture](#5-event-architecture)
6. [The JAT Boundary](#6-the-jat-boundary)
7. [Rendering Pipeline](#7-rendering-pipeline)
8. [Fetch Coordination and Viewport Logic](#8-fetch-coordination-and-viewport-logic)
9. [Localisation Architecture](#9-localisation-architecture)
10. [Settings System](#10-settings-system)
11. [Collaboration and Transport Switching](#11-collaboration-and-transport-switching)
12. [Testing Model](#12-testing-model)
13. [Architectural Invariants](#13-architectural-invariants)
14. [Concluding Perspective](#14-concluding-perspective)

---

## 1. The Frontend Is Not a Viewer

In architectural terms, this type of frontend is sometimes called “terminal.” That word can sound dismissive, as if the frontend were merely a thin wrapper around something more important. That is not what it means here.

Terminal means that the frontend stands at the end of a decision pipeline whose authority resides elsewhere.

Ooloi’s backend is the single semantic authority for musical state. It owns pitch representation, time signatures, key signatures, remembered alterations, layout computation, and the full rendering pipeline. When a piece changes, the backend determines what that change *means* and how it affects the structure of the score.

The frontend executes those decisions — and execution in this context is serious work.

The frontend is a computationally strong system in its own right. It:

* Maintains multi‑layer caches of rendering data.
* Builds GPU command buffers for complex vector graphics.
* Performs parallel computation for picture construction and data preparation.
* Transforms UI gestures into granular backend API calls.
* Manages distributed event streams and collaboration state.

It may process vast amounts of data. It may parallelise aggressively. It may prioritise, batch, and schedule work. None of this makes it authoritative — but it does make it powerful.

Authority, in Ooloi, is defined by who decides the shape of musical truth.

The frontend does **not** hold semantic authority.

It may construct local copies of musical data, transform them, prepare command payloads, batch mutations, or even perform optimistic UI updates in order to interact smoothly with the backend. It may mutate these local representations freely. But those mutations are provisional and have no authoritative status.

Only the backend decides whether a mutation becomes part of musical truth.

If the frontend is discarded entirely and rebuilt from backend state, the musical result must be identical. This is the litmus test that governs the entire architecture (see [ADR‑0038](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)).

The distinction is subtle but central:

* The backend decides.
* The frontend computes, executes, renders, and interacts.

That separation makes determinism, distributed collaboration, and multi‑mode transport possible without architectural compromise. It ensures that rendering, caching, and interaction complexity never erodes semantic closure.

The remainder of this guide explores how that principle is made real — through lifecycle control, event architecture, rendering boundaries, threading discipline, localisation invariants, and transport independence.

### A Note for Frontend Framework Developers

Developers coming from frameworks such as React, Vue, or other DOM‑centric systems may initially experience this architecture as unusually disciplined, even rigid. Modern web frameworks often encourage local component state, cascading updates, and implicit call chains through a hierarchy. Over time, that flexibility can drift into entangled state flows that are difficult to reason about.

Ooloi deliberately chooses a different balance.

The goal is not to demote the frontend or constrain frontend developers. It is to make large‑scale musical determinism possible. Scores are not ephemeral UI state; they are structured, interdependent semantic systems. A single pitch change can affect accidentals, key context, layout, spacing, and engraving decisions across large regions.

Allowing arbitrary local mutation in the UI would compromise reproducibility and distributed collaboration. Instead, Ooloi channels all semantic change through a single authoritative backend, while giving the frontend significant computational responsibility for rendering, caching, scheduling, and interaction.

If anything, this increases the frontend's architectural importance. It must remain disciplined precisely because it is powerful.

There is a further reason for this discipline that is easy to overlook: **plugins are first‑class citizens in Ooloi.**

A backend plugin — running in the server process, potentially across a network — must be able to declare that it has an associated window, dialog, settings panel, or toolbar contribution. It must be able to describe the contents of that UI, its validation rules, its localisation keys, and its event subscriptions. And it must do all of this without ever touching a JavaFX object.

This is why UI structure is expressed as pure data (cljfx specs, setting declarations, command descriptors, localisation keys). Data can travel across the wire. Data can be validated, composed, and projected by infrastructure that the plugin author never sees. An imperative widget API cannot do any of this safely.

The same principle ripples through the entire frontend:

* A plugin declares settings via `def-app-setting` — they appear automatically in the settings dialog, validated and localised.
* A plugin declares commands via data descriptors — they appear in menus with correct keyboard shortcuts and localisation.
* A plugin declares event subscriptions by category — they receive events through the same bus as core code.
* A plugin describes a window as a cljfx spec — it is materialised, lifecycle‑managed, and geometry‑persisted by the same UI Manager.

None of this requires the plugin to import JavaFX classes, manipulate scene graphs, or manage threads. The frontend infrastructure handles materialisation. The plugin speaks in data.

This is what the apparent rigidity protects. It is not rigidity for its own sake. It is the structural precondition for a plugin system where third‑party extensions operate at the same level of architectural integrity as core code — across process boundaries, across network boundaries, and across trust boundaries.

The result is a system where:

* Semantic truth is centralised and deterministic.
* Rendering and interaction are high‑performance and parallel.
* Collaboration modes do not alter structural guarantees.
* Plugins participate as peers, not as guests.

This architecture preserves clarity at scale.

---

## 2. Opening a Window (Minimal Example)

Architectural discussions are clearer when grounded in something concrete. So we will begin with the simplest visible act a frontend can perform: opening a window.

In Ooloi, even this small step reflects several invariants: lifecycle authority, localisation discipline, and separation between pure description and materialisation.

### 2.1 Localisation First

There are no hardcoded user-facing strings in the frontend. This is not a stylistic preference; it is an architectural rule defined in [ADR‑0039](../ADRs/0039-Localisation-Architecture.md). The full localisation architecture is described in Section 9; here we introduce only what is needed to read the first example.

Every visible string is declared and resolved through translation keys.

In many places keys appear *indirectly* (e.g. dialog specs, command descriptors, notification specs). To keep build-time verification strict even when static analysis cannot “see” the eventual `tr` call site, the frontend uses `tr-declare` — a no-op key declaration function consumed by the verification tooling.

```clojure
(tr-declare
  {:hello.window/title "Hello"
   :hello.label/text  "Hello, Ooloi"})
```

The UI does not embed literal text. It references keys.

### 2.2 A Minimal Window via the UI Manager

Before looking at the example, one architectural point must be absolutely clear:

Ooloi uses **cljfx specifications** to describe UI structure.

UI content is not constructed imperatively and not mutated through widget references. It is expressed as immutable Clojure data using cljfx’s declarative model (`:fx/type`, `:children`, `:style-class`, `:text-key`, etc.). That data is then materialised into real JavaFX objects on the JavaFX Application Thread.

This is not an aesthetic choice. It is foundational to the architecture:

* It keeps UI description pure and testable.
* It allows structural reasoning about UI state.
* It keeps localisation, settings, and plugin-driven composition uniform.
* It prevents hidden state drift through ad-hoc widget mutation.

Window lifecycle is handled by the UI Manager, but window *content* is described using cljfx specs.

One detail matters for correctness: the UI Manager owns window lifecycle. Frontend code supplies a **plain map with `:window/*` metadata and a materialised content node** as an event; it does not construct `Stage` or `Scene` directly, and it does not call `show-window!` directly ([ADR‑0042](../ADRs/0042-UI-Specification-Format.md)).

A minimal example illustrates the pattern:

```clojure
(tr-declare
  {:hello.window/title "Hello"
   :hello.label/text  "Hello, Ooloi"})

(defn- hello-content-spec []
  {:fx/type :v-box
   :children [{:fx/type :label
               :text-key :hello.label/text}]})

(defn show-hello! [ui-manager]
  (eb/publish! (:event-bus ui-manager) :window-requests
    [{:type             :window-open-requested
      :window/id        :hello
      :window/title-key :hello.window/title
      :window/content   (cljfx/instance
                          (cljfx/create-component
                            (hello-content-spec)))}]))
```

This shows the intended layering:

1. Window identity and lifecycle metadata are `:window/*` keys.
2. Content is a pure cljfx description produced by a private spec function.
3. `cljfx/create-component` + `cljfx/instance` materialise the description into a JavaFX Node before it is placed in the event.
4. The event is published to `:window-requests`; the UI Manager subscriber handles Stage creation.
5. User-facing strings are translation keys (resolved at materialisation time by the i18n layer, per [ADR‑0039](../ADRs/0039-Localisation-Architecture.md)).

The pattern is consistent whether a window is simple or complex. For windows that need imperative wiring — an action handler on a specific button, a focus request after construction — the private spec function may use `ext-on-instance-lifecycle` to attach behaviour at node creation time ([ADR‑0042](../ADRs/0042-UI-Specification-Format.md)). Section 4 covers this in full.

### 2.3 What Happens Internally

When `:window-open-requested` arrives at the UI Manager's `:window-requests` subscriber, it calls `show-window!` internally. That triggers a concrete, disciplined sequence:

1. **Singleton behaviour**: if a matching window already exists (as defined by `window-id-match?`), the existing `Stage` is brought to the front and returned.
2. **Geometry restoration**: if the window participates in persistence (`:window/persist?` not false), any persisted geometry is merged into the spec, overriding default `:x`, `:y`, `:width`, `:height`, and `:scale` when applicable (and also scroll position, if applicable).
3. **Window construction**: `register-window!` builds the `Stage` from the spec.
4. **macOS ownership**: on macOS, if a menu-bar host stage exists, the new window is owned by it so the system menu bar is inherited.
5. **Lifecycle wiring**:

   * Native close requests (the window “X”) are intercepted and routed through `close-window!`.
   * Geometry listeners are attached and debounced persistence is scheduled after move/resize.
   * The stage is registered under `:window/id`.
6. **Showing**: unless `:ui-mode` is `:headless`, the stage is shown.
7. **Event publication**: a `:window-opened` event is published to the `:window-lifecycle` category.

Even this minimal path demonstrates three recurring principles:

* Lifecycle authority is centralised.
* UI structure is declared as data.
* Visible strings are represented as localisation keys.

From here, we can expand the example — adding interaction, backend calls, and rendering — while preserving these invariants.

### 2.4 Confirmation and Custom Components

Windows are the general lifecycle mechanism. Three other pieces complete the common interaction vocabulary.

**Confirmation dialogs.** `show-confirmation!` in `ooloi.frontend.ui.core.confirmation-dialog` is the sole dialog helper. It materialises a standard cljfx `:alert` spec via `cljfx/create-component` + `cljfx/instance`, blocks synchronously on the JAT via `.showAndWait`, and returns `true` if the user confirmed. No custom dialog infrastructure is needed beyond this.

**Notifications.** The UI Manager provides a non-blocking notification overlay through convenience functions: `show-info-notification!`, `show-warning-notification!`, `show-error-notification!`, and `show-success-notification!`. Notifications auto-dismiss after a configurable delay, stack vertically in a corner of the screen, and are backed by AtlantaFX `Notification` controls materialised via the `ooloi-notification` custom component function. Application code calls the convenience wrappers; the overlay lifecycle is managed entirely by the UI Manager.

**Custom cljfx component functions.** Buttons, labels, scroll panes, menus, and notifications are expressed as custom cljfx component functions in `ooloi.frontend.ui.core.cljfx` (`ooloi-button`, `ooloi-ok-button`, `ooloi-cancel-button`, `ooloi-button-bar`, etc.). These are pure functions from props maps to cljfx description maps — not imperative builders. They handle localisation key resolution internally. Section 4.4 describes the full inventory and the architectural reason this mechanism exists.

The pattern is the same throughout: specs are pure data, materialisation happens at a controlled boundary, and every user-facing string resolves through a localisation key ([ADR‑0039](../ADRs/0039-Localisation-Architecture.md)).

---

## 3. The Window Lifecycle Invariant

The window lifecycle sits at the structural core of the frontend. It is a quiet agreement the system makes with itself about how windows come into being, how they behave, and how they disappear.

In Ooloi, window creation, showing, hiding, and closing are centralised in the UI Manager. The intention is simple: geometry persistence, event publication, macOS ownership rules, and headless compatibility should happen automatically, without each developer having to remember them every time. By gathering lifecycle responsibilities in one place, the system takes care of these details quietly and consistently.

The invariant can be stated precisely ([ADR‑0042](../ADRs/0042-UI-Specification-Format.md)):

1. **Never construct `Stage` directly in application code.**
2. **Never close a window by calling `.close` on a `Stage`.**
3. **All window open and close requests are published to `:window-requests` on the frontend event bus.** The UI Manager is the sole performer of the actual lifecycle operations.
4. **Headless mode must behave identically at the lifecycle level.**

This discipline exists so that:

* Geometry persistence (position, size, scale) is merged and stored automatically.
* Native close requests are intercepted and normalised.
* Window lifecycle events (`:window-opened`, etc.) are published consistently.
* On macOS, ownership is attached correctly so the system menu bar behaves as expected.

Centralising lifecycle control ensures that window behaviour remains deterministic, testable, and transport‑independent. Geometry persistence, event publication, macOS ownership rules, and headless compatibility happen automatically and consistently, so developers can focus on intent rather than remembering infrastructure rules.

When lifecycle control is centralised, the rest of the system can reason about windows structurally rather than defensively. That makes the frontend calmer, not stricter.

### 3.1 Compact Reference: Window Lifecycle and Custom Components

This is not a full API reference, but the most commonly used frontend entry points are:

**Window lifecycle** — publish to `:window-requests` on the frontend event bus:

```clojure
(eb/publish! (:event-bus ui-manager) :window-requests
  [{:type :window-open-requested  :window/id :my-window ...}])

(eb/publish! (:event-bus ui-manager) :window-requests
  [{:type :window-close-requested :window/id :my-window}])
```

**Confirmation dialogs**

* `show-confirmation!` (`ooloi.frontend.ui.core.confirmation-dialog`) — materialises a cljfx `:alert`, blocks on the JAT via `.showAndWait`, and returns `true` if the user confirmed.

**Notifications (UI Manager)**

* `show-info-notification!`
* `show-warning-notification!`
* `show-error-notification!`
* `show-success-notification!`

**Custom cljfx component functions** — see Section 4.4 for the full inventory.

**Common window spec keys**

The recognised `:window/*` keys for `:window-open-requested` events:

| Key | Purpose |
|-----|---------|
| `:window/id` | Unique identifier for registry tracking and geometry persistence |
| `:window/content` | Materialised JavaFX Node for the window's main content |
| `:window/menu-bar` | JavaFX MenuBar to attach above the content |
| `:window/title-key` | i18n key resolved into the Stage title (use instead of raw `:title`) |
| `:window/style` | Stage style: `:decorated` (default), `:undecorated`, `:transparent`, `:utility` |
| `:window/persist?` | Whether to save and restore window geometry across sessions (default `true`) |
| `:window/resizable?` | Whether the window can be resized by the user (default `true`) |
| `:window/default-position` | Default `{:x n :y n}` if no persisted state exists |
| `:window/default-size` | Default `{:width n :height n}` if no persisted state exists |

Unknown `:window/*` keys throw at construction time. Non-`:window/*` keys (`:width`, `:height`, etc.) pass through to the Stage directly.

All of these respect localisation ([ADR‑0039](../ADRs/0039-Localisation-Architecture.md)), lifecycle invariants, and JAT constraints.

If you are reaching for raw JavaFX constructs in application code, you are probably bypassing an existing abstraction.

---

## 4. Pure Builders and Materialisation

Section 2 showed the basic pattern: a private spec function returns a cljfx description map, a public `show-*!` function materialises it and hands the Node to `show-window!`. This section examines why that pattern exists, how it extends to the full range of UI construction, and where its firm boundaries lie.

### 4.1 Two-Level Abstraction

The architecture's essential abstraction is not “builder function versus inline map.” It is:

1. Declarative UI specification (pure data).
2. Materialisation on the JavaFX Application Thread.

Whether the specification is produced inline or by a dedicated private function is an implementation choice. In practice, the established convention is a private `*-spec` function that returns a cljfx description map, with a public `show-*!` function that materialises it via `cljfx/create-component` + `cljfx/instance` and hands the Node to `show-window!`. This convention enables testing in the majority of cases and keeps complex specifications manageable — but the architecture's invariant is the spec-to-materialisation boundary, not the presence of a function.

### 4.2 Why This Separation Exists

The separation between declarative specification and materialisation is one of the quiet structural pillars of the frontend.

At first glance it can look like a stylistic preference: describe UI as data, then build it later. In Ooloi it is far more consequential than that. It is what allows the frontend to remain deterministic, testable, plugin‑compatible, and transport‑independent while still being computationally strong.

**Testing.**
When UI structure is expressed as pure data, most of it can be constructed and inspected without launching JavaFX at all. Builder functions can be evaluated in isolation. Specs can be validated. Structural invariants can be asserted. Around 80% of frontend logic becomes ordinary functional code, not thread‑bound widget manipulation. The JavaFX Application Thread (JAT) is only required at the final step: materialising nodes and attaching them to a scene.

**JAT discipline.**
JavaFX enforces a single UI thread. By keeping specification pure and deferring all object creation to clearly defined materialisation boundaries — `cljfx/create-component` + `cljfx/instance` for content, `show-window!` for lifecycle — the architecture makes the JAT boundary explicit. There is one bridge (`fx/run-later!`, see Section 6), and it is visible. This clarity prevents accidental blocking, hidden cross‑thread mutation, and subtle race conditions.

**Backend‑authoritative rendering ([ADR‑0038](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)).**
The backend produces authoritative paintlists. The frontend turns those into GPU pictures. Because UI description and rendering preparation are separated from semantic authority, the frontend can aggressively cache, batch, parallelise, and discard derived state without risking semantic drift. If all frontend state is dropped and reconstructed from backend data, the visible result must be identical. The spec → materialisation model reinforces that guarantee.

**Plugin compatibility.**
Plugins operate at the same declarative level as core code. They contribute specs, commands, settings, and UI fragments as data. They do not receive privileged access to mutable widget graphs. This keeps extension boundaries clean. A plugin can describe what it wants; the frontend infrastructure decides how and when that description becomes concrete JavaFX objects. Crucially, because specs are pure data, a backend plugin running in a remote server process can describe a window or dialog to the frontend across the wire — the spec is serialised, transmitted, and materialised without the plugin ever importing a JavaFX class. An imperative widget API would make this impossible. The declarative model is therefore not a stylistic choice; it is the structural foundation of Ooloi's plugin architecture.

**Transport independence.**
Whether the backend runs in‑process, over gRPC to a local server, or across a network to a collaboration host ([ADR‑0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)), the frontend’s responsibility is the same: render what it receives, express user intent as API calls, and react to invalidation events. Because semantic authority is never embedded in widget state, switching transport modes does not require architectural reinterpretation.

Taken together, this separation produces a frontend that is both powerful and calm. It can mutate local copies, prepare payloads, stage optimistic updates, and manage large rendering caches — yet its structural contract remains simple: describe, materialise, execute, discard, regenerate. The musical truth remains elsewhere, and that clarity keeps the whole system coherent.

### 4.3 cljfx Spec Over Java Interop

The declarative separation described above has a corollary rule codified in [ADR‑0042](../ADRs/0042-UI-Specification-Format.md): **always prefer cljfx spec to Java interop.**

cljfx specs are pure Clojure data — maps and keywords. They compose naturally, test without a running JavaFX toolkit, and follow the single consistent materialisation path described in this section. Java interop bypasses all of this: it is imperative, JavaFX Application Thread-bound, untestable as data, and adds noise that obscures intent.

The rule is concrete: if a layout property — size, padding, alignment, spacing, max-width, style — can be expressed as a cljfx spec key, it must be. Only reach for interop when cljfx has no equivalent.

```clojure
;; ❌ Wrong — interop for a property cljfx handles declaratively
{:fx/type cljfx/ext-on-instance-lifecycle
 :on-created (fn [node] (.setMaxWidth node 500.0))
 :desc {:fx/type :v-box ...}}

;; ✅ Correct — pure spec, testable without JavaFX
{:fx/type :v-box
 :max-width 500.0
 :children [...]}
```

**Where interop is legitimate** — two narrow cases:

* `ext-on-instance-lifecycle :on-created` for one-time property bindings that have no cljfx spec equivalent (e.g. JavaFX property bindings with `.bind`).
* Wiring action handlers on pre-existing JavaFX nodes that the renderer cannot manage (e.g. `setOnAction` on a materialised control referenced by a variable).

The windowing infrastructure itself (`window.clj`, `ui_manager.clj`) necessarily creates Stages and Scenes — that is not interop in the sense of this rule; it is the materialisation layer itself.

Everything else is spec.

### 4.4 Custom cljfx Component Functions

The plugin architecture described in Section 1 requires that UI descriptions remain pure data, portable across process and network boundaries. [ADR‑0042](../ADRs/0042-UI-Specification-Format.md) specifies the mechanism. This creates a specific challenge: how does a backend plugin specify a button with a localised label when `tr` (the translation function) lives on the frontend?

The answer is a library of **custom cljfx component functions** in `ooloi.frontend.ui.core.cljfx`.

cljfx supports functions as `:fx/type` values. Each Ooloi custom component function receives an enriched props map containing Ooloi-specific keys such as `:text-key` that are not part of cljfx's standard vocabulary. It strips those keys, resolves them to standard cljfx values — calling `tr` for text keys — passes all other props through unchanged, and returns a standard cljfx description map.

```clojure
;; The function is a pure map → map transformation
(defn ooloi-button [{:keys [text-key] :as props}]
  (-> (dissoc props :text-key)
      (assoc :fx/type :button
             :text    (tr text-key)
             :min-width 90.0)))

;; Usage — keyword-only spec, serialisable over gRPC
{:fx/type ooloi-button
 :text-key :piece-window.piece-settings-button}
```

Resolution happens at render time on the frontend, where the locale is known. The spec author — whether frontend code or a backend plugin — writes `:text-key :some.key`. The infrastructure calls `tr` at materialisation.

**Why this matters architecturally.** This is the mechanism that makes the plugin claim in Section 1 concrete. A backend plugin can include `{:fx/type 'ooloi.frontend.ui.core.cljfx/ooloi-button :text-key :common.save}` in a cljfx spec sent over gRPC. The symbol resolves on the frontend; `tr` runs on the frontend. The plugin itself never imports JavaFX or i18n infrastructure.

**Custom components are pure and testable.** Because they are ordinary functions from maps to maps, they can be evaluated and inspected without launching JavaFX:

```clojure
(ooloi-button {:text-key :button.ok})
;; => {:fx/type :button :text "OK" :min-width 90.0}
```

**Atomic components** handle a single element with Ooloi-enriched keys:

| Component | Ooloi keys | Output |
|-----------|-----------|--------|
| `ooloi-button` | `:text-key` | `:button` with min-width 90px |
| `ooloi-ok-button` | `:text-key` (default `:button.ok`) | `:button` with `:default-button true` |
| `ooloi-cancel-button` | `:text-key` (default `:button.cancel`) | `:button` with `:cancel-button true` |
| `ooloi-label` | `:text-key` | `:label` |
| `ooloi-menu-item` | `:text-key` | `:menu-item` |
| `ooloi-menu` | `:text-key` | `:menu` |
| `ooloi-command-item` | `:descriptor`, `:state` | `:menu-item` with resolved text and `:disable` |
| `ooloi-notification` | `:text-key`, `:type`, `:icon`, `:opacity` | AtlantaFX `Notification` node |

**Composite components** encode Ooloi's layout conventions, not just atomic elements:

| Component | Encapsulated layout |
|-----------|---------------------|
| `ooloi-button-bar` | Right-aligned HBox with a spacer Region; consistent button padding |
| `ooloi-vscroll-pane` | Optionally titled ScrollPane with a muted border; title style via `TITLE_4` |

These ensure consistent spatial rhythm throughout the application without repeating layout logic in every module.

### 4.5 Per-Window Reactive Renderer

Every application window uses a **per-window reactive renderer** ([ADR‑0042](../ADRs/0042-UI-Specification-Format.md)). The renderer manages the content Node reactively; the UI Manager manages the Stage. These two responsibilities are absolute and never overlap. One-time `cljfx/create-component` + `cljfx/instance` materialisation without a renderer is reserved exclusively for `show-confirmation!`, which materialises a blocking cljfx `:alert` spec and returns synchronously via `.showAndWait`. Confirmation dialogs bypass `show-window!` entirely and have no lifecycle after dismissal. All other windows use `cljfx/mount-renderer`.

The architecture divides responsibility clearly:

* The **UI Manager** manages the Stage: creation, registry, geometry persistence, lifecycle events.
* A **cljfx renderer** manages the content Node: reactive diffs and patches driven by a private state atom.

These two responsibilities never overlap.

```
  fresh state atom (allocated per call)
       │
       │  cljfx/mount-renderer watches
       ▼
  cljfx renderer
       │  :middleware — wrap-map-desc (atom value → spec)
       │  :fx.opt/map-event-handler — dispatch-fn
       │
       │  cljfx/instance
       ▼
  JavaFX Node
       │  :window/content in :window-open-requested event
       ▼
  UI Manager (:window-requests subscriber)
  Stage → window-registry
```

Each call to `piece-window-content` allocates a fresh, independent state atom. This means any number of piece windows can be open simultaneously — each has fully isolated state, its own renderer, and its own lifecycle. When the window closes, a one-shot `:window-lifecycle` subscriber calls `cljfx/unmount-renderer`, releasing the atom watch.

The renderer is created with two configurations:

* **`:middleware`** — `cljfx/wrap-map-desc` transforms each new atom value into a cljfx description by calling the window's spec function with the current state.
* **`:fx.opt/map-event-handler`** — routes all map-form `:on-action` events through a single dispatcher. When a cljfx element carries `{:on-action {:ooloi/event :some-command}}` (a map, not a function), cljfx appends the JavaFX event object and calls this handler on the JAT.

After initialisation, `cljfx/mount-renderer` keeps the renderer watching the atom. `cljfx/instance` extracts the JavaFX Node, which is published as `:window/content` in a `:window-open-requested` event. From that point, `swap!`ing the state atom causes the renderer to diff and patch the live scene graph without recreating the Stage.

**Two-tier event routing.** Map-form events are pure data — serialisable, inspectable, loggable. The `:fx.opt/map-event-handler` dispatcher routes them:

* **Module-internal events** (navigation, selection, drag-and-drop reactions, backend data updates) are handled entirely within the window module.
* **Application-level commands** (quit, show another window) bubble out through an injected `dispatch-fn` to `system.clj`'s action handlers. The `dispatch-fn` is injected by the caller — the window module holds no direct reference to application state or action tables. This keeps each window module independently testable.

---

## 5. Event Architecture

If lifecycle defines how UI elements come into existence, the event architecture defines how the system moves.

Ooloi’s frontend does not rely on implicit callback chains or cascading component updates. It uses explicit event publication, invalidation signals, and pull-based refresh. The result is a system that can scale from a single local window to a distributed collaborative session without changing its structural guarantees.

### 5.1 Three Event Systems

There are three distinct event layers, each with a different responsibility.

**1. JavaFX input events.**
Mouse movement, key presses, window resize events — these originate in JavaFX. They are immediate, local, and imperative. They are converted into frontend intents as quickly as possible.

**2. Backend Event Router ([ADR‑0031](../ADRs/0031-Frontend-Event-Driven-Architecture.md)).**
Semantic mutations do not propagate through UI state. They are sent to the backend through the polymorphic API. The backend processes them and publishes structured events describing what became stale. These are not UI instructions; they are invalidation signals.

**3. Frontend Event Bus.**
The frontend event bus is a category-based pub/sub mechanism. All frontend coordination travels through it: window lifecycle requests and results, settings changes, app lifecycle signals, collaboration status, and backend invalidations. Application code publishes requests; the UI Manager subscribes, performs the work, and publishes results. No subsystem reaches sideways into another through direct function calls on the UI Manager — the bus is the interface. Window modules subscribe to the categories they own: `:window-lifecycle` to unmount their renderer on close, and `:app-settings` to react to settings changes such as locale — posting `(fx/run-later! #(swap! *state identity))` so the renderer re-evaluates the spec after the UI Manager has already applied the change.

The separation is deliberate. Input, semantic mutation, and UI reaction are not collapsed into one mechanism. Each layer speaks in its own vocabulary.

```
  ┌─────────────────────────────────────┐
  │  1. JavaFX Input Events             │
  │     gestures, keys, resize          │
  │     vocabulary: user intent         │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │  2. Backend Event Router            │
  │     semantic processing             │
  │     vocabulary: VPD staleness       │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │  3. Frontend Event Bus              │
  │     lifecycle, settings, rendering  │
  │     vocabulary: categories          │
  └─────────────────────────────────────┘
```

### 5.2 Invalidation-Based Synchronisation

A key principle is that events signal *staleness*, not structure.

When something changes — a pitch is altered, a measure is inserted, layout recomputed — the backend does not send a pre-rendered UI delta. It emits an event that tells the frontend which VPD region (Vector Path Descriptor hierarchy) is now invalid.

The frontend reacts by pulling fresh authoritative data.

This pull-based model has several consequences:

* The frontend never trusts cached semantic structure beyond its validity scope.
* Multiple invalidations can be coalesced before repaint.
* Collaboration becomes a transport concern, not an architectural fork.
* Determinism is preserved because rendering is always derived from authoritative state.

The pattern is simple but powerful:

1. Something changes.
2. An invalidation event is published.
3. The frontend marks affected regions stale.
4. Fresh data is fetched.
5. Rendering data is regenerated.

No component assumes it “knows” the final structure based on partial updates.

### 5.3 Example Flow

Consider a concrete interaction: the user changes a pitch.

1. **User interaction.**
   A key press is captured by JavaFX and translated into a frontend command.

2. **Backend mutation.**
   The command is sent through the polymorphic API. The backend applies semantic rules: accidentals, key signature context, remembered alterations, layout consequences.

3. **Invalidation event.**
   The backend publishes an event describing which structural regions are stale.

4. **Frontend reaction.**
   The Rendering Data Manager marks corresponding paintlists invalid and schedules a fetch.

5. **Fetch and repaint.**
   Fresh paintlists are requested. GPU pictures are regenerated from authoritative data. The viewport repaints.

```
  Key press ──▶ Command ──▶ Polymorphic API
                                  │
                    ┌─────────────▼──────────────┐
                    │   BACKEND (authority)       │
                    │   Semantic rules applied    │
                    │   (accidentals, key context,│
                    │    layout consequences)     │
                    └─────────────┬──────────────┘
                                  │ invalidation
                    ┌─────────────▼──────────────┐
                    │   FRONTEND                  │
                    │   RDM: mark stale           │
                    │     → fetch paintlists      │
                    │     → regen GPU pictures    │
                    │     → repaint (JAT)         │
                    └────────────────────────────┘
```

At no point does the frontend attempt to predict engraving side effects. It performs heavy computation — caching, picture generation, parallel batching — but the semantic decision has already been made upstream.

Because events are explicit and category-based, the same mechanism works when the mutation originates from another collaborator across the network. The frontend does not distinguish between local and remote invalidation. The transport layer changes; the architectural contract does not.

The event system therefore acts as connective tissue. It keeps subsystems decoupled, makes invalidation explicit, and allows rendering to remain a pure derivation of authoritative state.

---

## 6. The JAT Boundary

JavaFX has a simple rule: all UI object creation and mutation must happen on a single thread — the JavaFX Application Thread (JAT).

Ooloi does not fight this rule. It makes it visible.

Rather than letting thread boundaries leak unpredictably into application code, the frontend architecture draws a clear line between pure computation and UI materialisation. Almost everything can happen off the JAT. The final step — touching JavaFX objects — happens in one controlled place.

### 6.1 The Sole Bridge: `fx/run-later!`

The primary bridge between background work and UI execution is `fx/run-later!`, a thin wrapper around `Platform/runLater`.

Its presence is intentional. When you see `fx/run-later!`, you know two things immediately:

1. Work is crossing into the UI thread.
2. Whatever happens inside must be fast and non-blocking.

There are no hidden crossings. No implicit UI mutations from worker pools. No accidental background writes into scene graphs. The bridge is explicit.

This design has a psychological benefit as well as a technical one: it keeps developers aware of when they are touching real UI state. The JAT is not a background detail. It is a boundary with meaning.

### 6.2 What Never Runs on the JAT

The JAT is for materialisation and short-lived UI updates. It is not for computation.

Heavy work runs elsewhere:

* Backend communication (including gRPC calls in collaborative mode).
* Rendering data preparation and picture construction.
* Cache management and invalidation bookkeeping.
* File I/O and persistence.
* Layout recalculation triggered by backend invalidation.

When invalidation events arrive (Section 5), the frontend may perform substantial parallel processing before repaint. None of that blocks the JAT. Only the final step — attaching new pictures to the scene graph — crosses the boundary.

The effect is a UI that remains responsive even while large orchestral scores are being recomputed or streamed from a remote backend.

### 6.3 Threading Model in Practice

Putting the pieces together:

1. **User input** occurs on the JAT.
2. The intent is converted into a backend API call.
3. Backend work happens off the UI thread.
4. Invalidation events are published.
5. Rendering data is recomputed in worker pools.
6. `fx/run-later!` schedules the minimal UI update required.

This rhythm — compute off-thread, materialise on-thread — is steady throughout the system.

Because UI structure is described as data (Section 4), most frontend logic can execute without touching JavaFX at all. The JAT becomes a thin materialisation layer rather than a central execution engine.

The result is a frontend that feels immediate while remaining architecturally composed. Threading follows a consistent, system-wide pattern rather than being redefined by each subsystem. Once that pattern is understood, it fades into the background and simply supports the work.

---

## 7. Rendering Pipeline

Rendering is where Ooloi’s frontend does some of its heaviest work.

The backend decides what the score *is*. The frontend decides how to turn that decision into pixels — efficiently, incrementally, and at scale.

The rendering pipeline is therefore not decorative. It is a disciplined transformation layer that converts authoritative structural descriptions into GPU-ready pictures.

### 7.1 Backend Paintlist Authority

The backend produces **paintlists** — structured, hierarchical drawing instructions derived from semantic state ([ADR‑0038](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)).

A paintlist is not a bitmap and not a UI fragment. It is a deterministic description of what must be drawn:

* Glyph placements
* Staff lines
* Beams and slurs
* Accidental symbols
* Layout-derived spacing
* Structural groupings

These paintlists are authoritative. They represent the backend’s engraving decisions.

The frontend does not reinterpret them. It does not reflow spacing. It does not “fix” layout heuristically. It consumes them.

This separation is crucial: engraving logic lives in one place only.

### 7.2 The Rendering Data Manager

Between paintlists and the screen sits the **Rendering Data Manager (RDM)**.

The RDM has one responsibility: manage derived rendering state.

It:

* Tracks which VPD regions are valid or stale.
* Maintains caches of paintlists and GPU pictures.
* Coordinates background recomputation.
* Schedules minimal UI updates.

When invalidation events arrive (Section 5), the RDM does not immediately redraw everything. It marks affected regions stale and determines what needs to be fetched or regenerated.

This is where parallelism becomes practical. Large orchestral scores can be split into regions. Only what is visible — or about to become visible — is prioritised. Work is batched and scheduled off the JAT.

The RDM is computationally intense but semantically humble. It derives everything from backend authority.

### 7.3 Two-Level Cache

Rendering uses two distinct cache layers:

1. **Paintlists (authoritative input)**
   Cached as structured drawing instructions received from the backend.

2. **GPU pictures (derived output)**
   Cached as Skia/JavaFX picture objects ready for fast redraw.

The distinction matters.

Paintlists represent engraving decisions. GPU pictures represent a performance optimisation.

If GPU pictures are discarded, they can be regenerated from paintlists. If paintlists are invalidated, they are refetched from the backend. At no point does the frontend invent structural data.

This layered caching enables:

* Fast scrolling
* Zoom-level independence
* Incremental redraw
* Memory proportional to viewport, not score size

The frontend may hold large caches, but every layer is derived and therefore disposable.

```
                    Rendering Data Manager
  ┌─────────┐   ┌─────────────┐   ┌─────────────┐   ┌──────────┐
  │         │   │ Paintlists  │   │     GPU     │   │          │
  │ Backend │──▶│  (cache 1)  │──▶│   Pictures  │──▶│ Viewport │
  │         │   │             │   │  (cache 2)  │   │          │
  └─────────┘   └──────┬──────┘   └──────┬──────┘   └──────────┘
                       │                 │
                  authoritative      derived,
                  refetch from       disposable,
                  backend if         regenerate
                  invalidated        from cache 1
```

### 7.4 The Litmus Test

A simple test governs the rendering pipeline:

> If all frontend rendering state is discarded and rebuilt from backend state, the visible result must be identical.

This is not theoretical. It is a practical invariant.

Because paintlists are authoritative and GPU pictures are derived, the frontend can:

* Drop caches aggressively under memory pressure.
* Switch transport modes (local ↔ remote) without reinterpretation.
* Reconstruct state after collaboration merges.

Determinism is preserved because rendering is always a pure transformation of authoritative data.

The rendering pipeline therefore embodies the frontend’s role precisely:

* It is powerful.
* It is parallel.
* It is performance-critical.
* And it never becomes the source of musical truth.

---

## 8. Fetch Coordination and Viewport Logic

Rendering and invalidation describe *what* must be redrawn. Fetch coordination and viewport logic decide *when* and *how much* of the score is actually brought into memory and prepared for display.

Large orchestral scores are not small documents. A full piece may contain thousands of measures, multiple staves per system, and deep structural hierarchies. Treating the score as a single monolithic drawable surface would quickly exhaust memory and destroy responsiveness.

Ooloi instead treats the score as a structured, navigable space.

### 8.1 VPD Hierarchy as Address Space

The Vector Path Descriptor (VPD) hierarchy functions as an address system for rendering regions.

A VPD does not merely identify an object; it locates a region within the musical structure. Systems, measures, staves, voices, and graphical groupings all live within this hierarchical coordinate system.

When the backend publishes an invalidation event (Section 5), it does so in terms of VPD regions. The frontend does not receive a pixel rectangle to repaint. It receives a structural address that describes which musical region has changed.

This allows fetch coordination to remain semantic rather than geometric.

* A single note change invalidates the smallest containing region.
* A layout recomputation may invalidate entire systems.
* A global change (e.g. key signature) may invalidate large swathes.

Because invalidation is structural, the frontend can reason about scope precisely.

### 8.2 Lazy Fetch and Priority

The frontend does not eagerly fetch every invalid region.

Instead, it maintains a prioritised queue driven by viewport relevance:

1. Regions currently visible on screen.
2. Regions just outside the viewport (anticipatory fetch).
3. Distant regions (background priority).

When the user scrolls, zooms, or jumps to another location, priority shifts accordingly.

This design ensures that:

* Visible content is always prepared first.
* Scrolling remains smooth.
* Memory consumption remains proportional to what the user can actually see.

The backend may be capable of describing the entire score at once. The frontend chooses not to materialise it all simultaneously.

### 8.3 Viewport-Proportional Memory

Memory usage scales with the viewport, not with total score size.

This principle has practical consequences:

* GPU pictures for off-screen regions can be evicted.
* Paintlists for distant regions can be dropped or deprioritised.
* Re-fetching is cheaper than retaining everything indefinitely.

Because all rendering state is derived (Section 7), eviction is safe. The frontend can always reconstruct what it needs from backend authority.

The combination of VPD-based addressing and viewport-driven prioritisation means that even extremely large scores behave like local, bounded scenes.

### 8.4 Scroll, Zoom, and Recomposition

Viewport changes — scroll position, zoom level, window resize — do not mutate semantic structure. They change perspective.

The frontend reacts by:

* Determining which VPD regions are newly visible.
* Scheduling fetches for missing paintlists.
* Regenerating GPU pictures at the appropriate scale.

Zoom independence is particularly important. A zoom change may require regeneration of GPU pictures (because scale affects picture resolution), but it does not require semantic recomputation in the backend. The two concerns remain separate.

This reinforces a recurring architectural theme:

* Semantic change originates in the backend.
* Perspective change originates in the frontend.
* Rendering bridges the two without collapsing them.

### 8.5 Collaboration Context

In collaborative mode ([ADR‑0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)), fetch coordination behaves the same way. Invalidation events may originate from remote mutations, but they still arrive as structural VPD addresses.

The frontend does not care whether a region became stale because of a local edit or a collaborator’s action. It marks it stale, reprioritises if necessary, and fetches authoritative data.

Transport switching therefore does not require a second fetch model. The same coordination logic applies whether the backend is in-process or remote.

The result is a frontend that scales in three dimensions simultaneously:

* Large documents.
* High zoom levels.
* Distributed collaboration.

Fetch coordination and viewport logic make that scaling predictable rather than accidental.

---

## 9. Localisation Architecture

Localisation is part of the frontend's structural discipline.

The rule is simple and strict: **no hardcoded user-facing strings**. Every visible string is represented by a translation key and resolved through the localisation subsystem ([ADR‑0039](../ADRs/0039-Localisation-Architecture.md)).

This is not primarily about translating the UI later. It is about keeping the UI describable, verifiable, and composable — especially once plugins and generated UI (such as the settings dialog) enter the picture.

### 9.1 Translation API: `tr` and `tr-declare`

The translation function is `tr`. It takes a keyword and returns the current-locale string.

Some keys are referenced indirectly (inside specs, command descriptors, notification specs, settings metadata). To keep build-time verification strict even when static analysis cannot see a direct `tr` call, the frontend uses `tr-declare`.

`tr-declare` is intentionally a no-op at runtime. Its role is to make the set of expected keys explicit to the verification tooling.

There is another important advantage: `tr-declare` allows the developer to provide the canonical UK English string directly alongside the code. Static analysis of `(tr ...)` call sites cannot recover the intended source-language text; it can only detect the key. By declaring keys with their real strings, the authoritative wording stays close to the implementation.

Even though the localisation pipeline can generate TODO entries in `.po` files automatically, keeping the intended English text next to the code has practical benefits:

* The developer sees the real wording immediately.
* Review and refactoring remain contextual rather than file-driven.
* The `.po` file is not the only place where meaning lives.

For this reason, it is increasingly good practice to declare keys explicitly even when they are also referenced directly through `tr`. The build tooling will handle undeclared keys, but disciplined declaration keeps language and intent structurally aligned with the code.

```clojure
(tr-declare
  {:button.ok           "OK"
   :button.cancel       "Cancel"
   :dialog.confirm.title "Confirm"})
```

### 9.2 Where Keys Appear

Localisation keys show up in several places that matter architecturally.

**Window titles.**
Window specs carry `:window/title-key`. The window subsystem resolves it through `tr` when building the JavaFX stage.

**Dialog titles and button labels.**
Dialogs follow the same principle: the dialog subsystem resolves `:window/title-key`, and button helpers resolve OK/Cancel keys consistently (with declared defaults).

**Notifications and status UI.**
Notifications accept either raw text or a `:text-key`. When a key is provided, it is resolved through `tr` and combined with any dynamic details.

**Generated UI (settings dialog).**
The settings dialog is derived from the settings registry ([ADR‑0043](../ADRs/0043-Frontend-Settings.md)). That registry carries localisation keys for category names, setting descriptions, and choice labels. Because the UI is generated, key discipline is what prevents "surprise English" from leaking into the application.

The essential idea is that every string has a stable identity, whether translated immediately or not.

### 9.3 Runtime Initialisation

Localisation is initialised during frontend startup by the UI Manager.

In broad strokes:

* locales are initialised,
* a locale is selected (typically based on OS locale),
* translation resources are loaded and cached.

Because this happens before windows are shown, window titles and early startup UI (including the splash label and menu items) can be resolved through the same mechanism from the first frame.

### 9.4 GNU gettext and the `.po` Ecosystem

Ooloi uses the GNU gettext model and standard `.po` files for translations.

This is not an invention. It is a mature, widely adopted standard with decades of tooling behind it. Translators are already familiar with it. Established tools such as Poedit, Weblate, Transifex, Crowdin, Lokalise, POEditor, and OmegaT all understand `.po` files natively.

That choice has architectural implications.

* Translation files are plain text and diff‑friendly.
* The workflow is compatible with existing translator pipelines.
* No proprietary format or custom editor is required.
* Plugins can ship their own `.po` resources using the same mechanism.

Because `.po` is an externalised, standard format, localisation remains a first‑class concern without becoming a custom subsystem that Ooloi must maintain indefinitely.

### 9.5 Build-Time Verification

[ADR‑0039](../ADRs/0039-Localisation-Architecture.md) defines the verification model.

At build time, the tooling checks that:

* every referenced key exists in the translation files,
* no keys are misspelled or orphaned,
* declared keys (`tr-declare`) match what the UI expects.

This is what makes the "no hardcoded strings" invariant enforceable.

Localisation is therefore not a convention maintained by habit. It is a property upheld by tooling.

### 9.6 Default Language and Locale Model

Ooloi's source language is **UK English**. The strings provided in `tr-declare` are written in UK English and form the canonical wording of the application.

This has a structural consequence: American English is treated as a translation, just like German, French, or Swedish. There is no special case for US spelling or phrasing. If the locale is `en_US`, it resolves through the same `.po` mechanism as any other locale.

This keeps the model conceptually clean:

* There is one authoritative source language in the codebase.
* All other language variants — including American English — are translations.
* Locale switching never changes architectural behaviour; it only changes resolved strings.

Because the canonical wording lives alongside the code (via `tr-declare`), the source language remains visible during development rather than being buried in translation files.

### 9.7 Practical Guidance

A few habits make localisation frictionless:

* Introduce keys early, even for temporary UI.
* Use `tr-declare` for keys that are only referenced indirectly.
* Prefer key-based UI specs (`:window/title-key`, dialog button keys, notification keys) rather than embedding literals.

Once you work this way for a little while, it becomes natural. The benefit is that UI composition remains clean, build verification remains strict, and the system can generate UI without accidentally hardcoding language.

---

## 10. Settings System

Application settings in Ooloi are not an afterthought and not a collection of ad-hoc preference reads scattered across the UI. They form two independent systems serving different scopes:

* **Frontend app settings** — global application preferences (theme, UI language, editor preferences). Declared on the frontend with `def-app-setting`, stored in an EDN file, accessed directly through an in-process atom API. Described in full below.
* **Piece settings** — configuration that travels with piece data (beam thickness, staff spacing, accidental behaviour). Declared on the backend with `defsetting`, stored in STM-managed piece state, accessed exclusively through the polymorphic API over gRPC. The frontend reads and writes piece settings via `SRV/` calls — no direct access.

These two systems are architecturally independent. Piece settings concern musical semantics and belong to the backend. Frontend app settings concern application behaviour and belong to the frontend. Mixing the two would compromise the backend-authoritative invariant (Section 13.1).

The remainder of this section covers frontend app settings, governed by [ADR‑0043](../ADRs/0043-Frontend-Settings.md). The central idea: settings are declared once, validated centrally, stored consistently, and exposed through a uniform API. UI surfaces for editing those settings are derived from that declaration rather than hand-crafted repeatedly.

### 10.1 `def-app-setting`

Settings are defined declaratively using `def-app-setting`.

A setting definition includes:

* A key (e.g. `:ui/theme`).
* A default value.
* Validation rules.
* Optional metadata (category, description key, choice map, etc.).

This does several important things at once.

First, defaults are structural, not implicit. The system always knows what the baseline configuration is, even before any persisted user configuration is loaded.

Second, validation is explicit. A setting can define strict validation (reject invalid values), permissive modes, or transformation logic. Invalid state is not allowed to drift silently into the application.

Third, all settings are registered in a central registry. This registry is what allows automatic dialog generation (see below), uniform persistence, and event publication when settings change.

The frontend reads settings through a small API (`get-app-setting`, etc.) rather than directly touching storage. Mutation flows through a controlled path so that validation and event publication happen consistently.

### 10.2 Automatic Settings Dialog

Because settings are declared structurally, the settings dialog does not need to be manually assembled field by field.

Instead:

* Settings are grouped by category.
* Categories map to tabs in the dialog.
* Choice maps become ComboBoxes.
* Boolean settings become checkboxes.
* Text and validated settings become TextFields.

The settings window (`settings_window.clj`) follows the content builder pattern ([ADR‑0042](../ADRs/0042-UI-Specification-Format.md)). `show-settings!` manages lifecycle through the UI Manager; `show-window!` ensures the window is a singleton — opening it twice brings the existing instance to the front. Internally, the window is a TabPane with one tab per settings category namespace, each tab holding a ScrollPane of setting rows. Controls derive from registry metadata: a `:choices` entry becomes a ComboBox inside an AtlantaFX Tile; a validator-equipped text setting becomes a TextField that commits on Enter or focus loss. Each field has a per-field reset button; a "Reset All to Defaults" button at the bottom resets every setting in the active tab category after confirmation.

The result is a dialog that is consistent by construction:

* Every visible string is a localisation key (Section 9).
* Validation logic lives with the setting definition.
* New settings appear automatically when declared.

There is no duplication between "settings storage" and "settings UI." The dialog is a projection of the registry.

### 10.3 Settings Events and Live Updates

When a setting changes, the system publishes an event through the frontend event bus (Section 5).

This allows subsystems to react declaratively.

For example:

* Changing `:ui/theme` triggers a theme reapplication across all windows.
* Notification positioning can be recalculated.
* Rendering parameters can be adjusted.

The UI Manager already exposes operations such as `apply-theme-to-all!` and `refresh-menu-text!`. These are invoked in response to structured events rather than direct cross-component calls.

This reinforces the broader architectural pattern:

* State change is explicit.
* Effects are triggered via event subscription.
* No subsystem reaches sideways to mutate another's internals.

### 10.4 Persistence and Scope

Settings persistence is handled centrally.

On startup:

* Defaults are established.
* Persisted values are loaded.
* Validation ensures that only acceptable values enter runtime state.

When settings are mutated:

* The new value is validated.
* The registry is updated.
* An event is published.
* Persistence is updated.

The backend does not own application settings. It owns piece data. This boundary is intentional. Settings concern application behaviour (theme, UI preferences, notification placement), not musical semantics.

Keeping this separation clean avoids a subtle but common architectural drift where UI configuration becomes entangled with domain state.

### 10.5 Architectural Role

The settings system may appear modest compared to rendering or collaboration, but it plays a stabilising role.

It ensures that:

* Configuration is predictable.
* UI generation is systematic.
* Validation is centralised.
* Changes propagate cleanly through the event system.

In a system that emphasises determinism and explicit structure, settings follow the same discipline. They are declared, validated, projected, and reacted to — never improvised.

---

## 11. Collaboration and Transport Switching

Collaboration in Ooloi grows directly out of the same architectural core ([ADR‑0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)). It is expressed through a stable frontend–backend contract that remains consistent across deployment topologies.

The guiding principle is straightforward: transport may vary, but architectural discipline does not.

Whether the backend runs:

* in-process (standalone desktop mode),
* as a local server instance, or
* remotely across the network in a collaborative session,

the frontend’s structural responsibilities remain the same.

### 11.1 Hybrid Transport Model ([ADR‑0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md))

[ADR‑0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md) defines a hybrid transport architecture in which the frontend communicates with the backend through a stable API surface, regardless of deployment topology.

From the frontend’s perspective, a backend exposes the polymorphic API, emits invalidation events, and returns authoritative paintlists. The transport layer may be:

* direct function calls (in-process), or
* gRPC over loopback or network.

The frontend architecture remains unchanged across these forms.

This continuity has immediate consequences:

* Collaboration uses the same rendering path as standalone editing.
* Event handling remains unified.
* Fetch coordination (Section 8) behaves identically.
* Caching and invalidation rules remain intact.

Transport becomes a deployment property, while the architectural contract remains stable.

### 11.2 Dynamic Server Lifecycle

In collaborative scenarios, a backend server may be started, connected to, disconnected from, or replaced during runtime.

The UI Manager and transport layer handle these transitions explicitly:

* Establishing secure connections.
* Switching transport adapters.
* Updating collaboration state indicators.
* Publishing connection lifecycle events on the frontend event bus.

Throughout these transitions, the frontend continues to operate under the same structural guarantees.

Windows behave the same way.
Invalidation remains pull-based.
Rendering continues to derive from authoritative paintlists.
Settings remain local application configuration.

Collaboration introduces additional UI elements — presence indicators, connection status, and related affordances — but these extend the interface without altering its core structure.

### 11.3 UI Invariance Across Modes

A useful architectural test is this:

> If a user ignored the status indicators, would editing and navigation feel structurally different between standalone and collaborative sessions?

They should not.

Editing, rendering, scrolling, zooming, and settings follow the same rules. Only the physical location of semantic authority changes.

This invariance is what makes the hybrid model coherent.

Because:

* semantic authority is centralised in the backend,
* invalidation is structural and explicit,
* rendering is purely derived,

switching between local and remote backends does not require reinterpretation of frontend state.

The frontend continues to perform the same role: a computationally strong execution layer at the end of a deterministic decision pipeline.

Where that pipeline runs may vary. The discipline of the frontend remains constant.

---

## 12. Testing Model

The frontend architecture is deliberately structured so that most of it can be tested without launching a graphical application.

This is not an afterthought. It follows directly from the separation between pure specification and materialisation (Section 4), and from the explicit JAT boundary (Section 6).

Testing in Ooloi’s frontend happens at several distinct layers.

### 12.1 Headless Mode

The UI Manager supports a `:headless` mode. In this mode:

* `show-window!` does not display real JavaFX stages.
* Lifecycle events are still published.
* Geometry persistence logic still executes.
* Registration and deregistration still occur.

This allows lifecycle behaviour to be exercised in tests without requiring a visible UI. The same structural invariants apply; only the materialisation side effect is suppressed.

Headless mode ensures that window management, event publication, and persistence logic are not coupled to a running desktop environment.

### 12.2 Pure Builder Testing

Because UI structure is expressed as data (cljfx specs), most builder functions can be evaluated as ordinary pure functions.

You can:

* Construct a window or dialog spec.
* Inspect its structure.
* Assert the presence of required `:window/*` keys.
* Verify that translation keys are used instead of literals.
* Check that settings-derived UI fragments are generated correctly.

None of this requires the JavaFX Application Thread.

In practice, a large proportion of frontend logic — often around 80% — lives in this pure layer. It can be tested with standard Clojure tooling, without mocks of scene graphs or synthetic UI drivers.

### 12.3 Integration Testing

Integration tests operate at the boundary between pure description and materialisation.

These tests:

* Cross the JAT boundary intentionally.
* Materialise real stages or dialogs.
* Simulate user interaction where necessary.
* Verify event publication and lifecycle wiring.

Because lifecycle entry points are centralised (`show-window!`, `close-window!`, dialog helpers), integration testing can focus on those choke points rather than attempting to observe arbitrary widget state.

The explicit boundaries make integration testing narrower and more predictable.

### 12.4 Invariants as Tests

Some of the most important guarantees in the frontend are architectural invariants rather than behavioural details.

These include:

* All user-facing strings resolve through localisation keys ([ADR‑0039](../ADRs/0039-Localisation-Architecture.md)).
* No JavaFX mutation occurs outside the JAT boundary.
* All window lifecycle operations pass through the UI Manager.
* Rendering is always derived from authoritative backend state ([ADR‑0038](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)).
* Invalidation is pull-based rather than push-structural ([ADR‑0031](../ADRs/0031-Frontend-Event-Driven-Architecture.md)).

Many of these are enforced structurally (through centralised entry points and build-time verification). Others are expressed as explicit tests.

The goal is not to achieve exhaustive UI automation. It is to make architectural drift difficult.

When structure is clear, testing becomes calmer. You test pure functions where possible, integration points where necessary, and invariants continuously.

---

## 13. Architectural Invariants

Architectural invariants are the quiet rules that allow the frontend to remain powerful without becoming unstable. They are not slogans, and they are not aesthetic preferences. They are structural commitments that make large‑scale determinism, collaboration, and rendering performance coexist without friction.

Each of the following invariants appears throughout this guide. Gathered together, they form the backbone of the frontend.

### 13.1 Backend Authority

All musical truth lives in the backend.

The frontend may prepare mutations, stage optimistic updates, transform local copies, and batch API calls — but only the backend decides what a change *means*. Rendering, layout, accidentals, spacing, and structural relationships are never inferred in the UI.

If the frontend is destroyed and rebuilt from backend state, the musical result must be identical.

This is the anchor that makes everything else possible.

### 13.2 Terminal Rendering

The frontend is the execution layer at the end of the decision pipeline.

It performs heavy computation: caching, picture construction, viewport coordination, GPU submission, parallel preparation. None of that grants semantic authority. Rendering is always a transformation of authoritative paintlists ([ADR‑0038](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)).

The frontend may be computationally strong. It remains structurally subordinate to backend semantics.

### 13.3 Pull-Based Invalidation

Events signal staleness, not structure.

When something changes, the backend emits invalidation events describing which structural regions are affected ([ADR‑0031](../ADRs/0031-Frontend-Event-Driven-Architecture.md)). The frontend marks those regions stale and pulls fresh authoritative data.

This keeps update flow explicit, coalescible, and transport-independent. It prevents hidden semantic drift through partial UI updates.

### 13.4 Explicit JAT Boundary

All JavaFX object creation and mutation occur on the JavaFX Application Thread.

Heavy computation does not.

The bridge between background work and UI materialisation is explicit (`fx/run-later!`). Because the boundary is visible, threading remains predictable. Rendering preparation, backend communication, and cache management happen off-thread. Materialisation happens deliberately.

Responsiveness emerges from that discipline.

### 13.5 UI Manager Owns Lifecycle

Window creation, closing, geometry persistence, and lifecycle event publication pass through the UI Manager.

Application code does not construct `Stage` directly. It does not close windows by reaching into JavaFX objects. It expresses intent by publishing to `:window-requests` on the frontend event bus; the UI Manager subscribes, performs the structured work, and publishes the result to `:window-lifecycle`.

This centralisation ensures:

* Geometry persistence (position, size, scale, scroll state) is consistent.
* macOS ownership rules are respected automatically.
* Headless mode behaves identically at the lifecycle level.
* Lifecycle events are always published.

Windows behave predictably because their lifecycle is not fragmented.

### 13.6 Pure Specification → Materialisation

UI structure is described as immutable cljfx data.

Materialisation into JavaFX objects happens at a controlled boundary.

This separation enables:

* Pure functional testing of most frontend logic.
* Plugin composition without privileged widget access.
* Clear transport switching between local and remote backends ([ADR‑0036](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)).
* Structural reasoning about UI state.

The declarative layer remains inspectable. The imperative layer remains contained.

### 13.7 Derived Caches

All frontend caches are derived.

Paintlists are authoritative input. GPU pictures are derived output. Viewport-specific state is derived perspective.

Because nothing in the frontend is the origin of musical truth, caches can be evicted, recomputed, and reprioritised safely. Memory scales with viewport rather than score size. Collaboration does not require reinterpretation.

Derived state can always be discarded and rebuilt without loss.

### 13.8 No Hardcoded User-Facing Strings

Every visible string is represented by a localisation key and resolved through the localisation subsystem ([ADR‑0039](../ADRs/0039-Localisation-Architecture.md)).

Keys are declared (`tr-declare`) and verified at build time. UK English is the canonical source language. All other locales — including American English — are translations resolved through standard GNU gettext `.po` files.

This keeps UI generation consistent, plugin contributions safe, and language visible alongside code rather than hidden in resource files.

### 13.9 No Hardcoded Colours

All colours in the frontend use AtlantaFX semantic tokens or `Styles` constants. Literal hex values, `rgb()`/`rgba()` functions, and JavaFX `Color` constants are prohibited in theme-dependent UI code.

```clojure
;; ✅ Correct — semantic token adapts to light and dark themes
{:fx/type :label :style "-fx-text-fill: -color-fg-muted;"}

;; ❌ Wrong — breaks in one theme
{:fx/type :label :style "-fx-text-fill: #888888;"}
```

The reason is architectural, not cosmetic. UI specifications are pure data that cross boundaries — plugins send cljfx specs over gRPC. A hardcoded colour survives transport but breaks under theme switching. A semantic token survives both because it resolves at render time against the active theme's stylesheet.

AtlantaFX follows the GitHub Primer semantic colour model. Tokens describe purpose, not appearance: `-color-fg-muted` is "de-emphasised foreground text," not "medium grey." The same token resolves to different concrete values in dark and light mode automatically.

Two narrow exceptions exist: pre-theme rendering (the splash screen, which displays before any AtlantaFX stylesheet is loaded) and theme-aware transparency overlays whose blending cannot be expressed as a single semantic token. Both are documented explicitly and not imitable by general UI code.

---

Taken together, these invariants create a frontend that is simultaneously:

* Deterministic.
* High-performance.
* Testable.
* Transport-independent.
* Extensible.

None of them exists in isolation. Each reinforces the others. Backend authority makes pull-based invalidation meaningful. Declarative specification makes JAT discipline practical. Derived caches make collaboration scalable. Localisation keys make generated UI predictable. Semantic colour tokens make theme switching transparent.

The result is coherence rather than rigidity.

When these invariants are respected, the frontend remains calm even as the musical structures it renders become large, distributed, and complex.

## 14. Concluding Perspective

A frontend like this does not emerge from taste alone. It grows out of constraints: musical determinism, distributed collaboration, GPU‑accelerated rendering, transport switching, plugin composition, localisation discipline. Each constraint could have been handled locally. Instead, they were gathered into a single structural idea: clarity of authority and clarity of boundaries.

What you have seen in this guide is a set of agreements rather than a collection of isolated tricks.

* The backend decides what music *is*.
* The frontend renders, computes, schedules, and interacts.
* Invalidation is explicit.
* Rendering is derived.
* Lifecycle is centralised.
* Strings have identity.
* Colours are semantic.
* UI is specified as data, not built as objects.
* Transport does not alter structure.

Individually, none of these ideas is revolutionary. Together, they create a system that remains calm under scale — large scores, high zoom levels, parallel rendering, remote collaborators.

If you are an architect, you may recognise a familiar pattern: authority placed deliberately in one layer, computation distributed elsewhere, and boundaries made visible rather than implicit. If you are a frontend developer, you may notice that this model asks for discipline — but it also offers something in return: predictability at scale.

The frontend is structured because it is strong; its discipline exists to support its computational weight. It performs substantial work — rendering, caching, batching, coordination — precisely because semantic truth has already been decided upstream.

And perhaps that is the deeper theme of Ooloi’s frontend architecture.

It trusts structure.

It trusts that clarity at the boundary produces freedom inside the layer.

When those boundaries are respected, the frontend can be ambitious without becoming entangled. It can scale without becoming fragile. It can collaborate without splitting into modes.

That is the spirit in which this guide should be read.

If you continue into the ADRs, you will find the colder, formal articulation of each rule. Here, the intention has been to show how those rules live together — how they become a working, coherent system.

---

For deeper formal detail, see:

* [ADR‑0031: Frontend Event‑Driven Architecture](../ADRs/0031-Frontend-Event-Driven-Architecture.md)
* [ADR‑0036: Collaborative Sessions and Hybrid Transport](../ADRs/0036-Collaborative-Sessions-and-Hybrid-Transport.md)
* [ADR‑0038: Backend‑Authoritative Rendering](../ADRs/0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)
* [ADR‑0039: Localisation Architecture](../ADRs/0039-Localisation-Architecture.md)
* [ADR‑0042: UI Specification Format](../ADRs/0042-UI-Specification-Format.md)
* [ADR‑0043: Frontend Settings](../ADRs/0043-Frontend-Settings.md)

## Related Guides

* **[Polymorphic API Guide](POLYMORPHIC_API_GUIDE.md)** — How musical structures are exposed and manipulated through Ooloi's polymorphic command surface
* **[Timewalking Guide](TIMEWALKING_GUIDE.md)** — How musical structure is traversed and transformed
* **[gRPC Communication and Flow Control](GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** — Frontend–backend communication patterns
* **[MIDI in Ooloi](MIDI_IN_OOLOI.md)** — MIDI input for note entry, why the core produces no MIDI output, and how MIDI events flow through the frontend event bus to Flow Mode
* **[Server Architectural Guide](OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** — Complementary server architecture that the frontend executes against
* **[VPDs](VPDs.md)** — The address system used for rendering regions and invalidation

---
