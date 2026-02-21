# ADR-0043: Frontend Settings

## Status

Accepted - February 12, 2026
Updated - February 13, 2026 (Defaults declared in def-app-setting; event bus integration)
Updated - February 16, 2026 (Settings dialog implemented; code references added)

## Table of Contents
- [Context](#context)
  - [Architectural Boundary](#architectural-boundary)
  - [Requirements](#requirements)
- [Decision](#decision)
  - [Architecture Overview](#architecture-overview)
  - [Setting Definitions](#setting-definitions)
  - [Validation Architecture](#validation-architecture)
  - [Translation Integration](#translation-integration)
  - [Storage](#storage)
  - [API](#api)
  - [Settings Registry](#settings-registry)
  - [Settings Dialog](#settings-dialog)
- [Rationale](#rationale)
- [Implementation](#implementation)
- [Consequences](#consequences)
- [Alternatives Considered](#alternatives-considered)
- [Related ADRs](#related-adrs)
- [Code References](#code-references)

## Context

### Architectural Boundary

[ADR-0015](0015-Undo-and-Redo.md) establishes that the backend has no application settings — only piece data. [ADR-0016](0016-Settings.md) implements per-piece, per-entity settings within piece data using `defsetting`, multimethods, VPD dispatch, and STM transactions. These are backend concerns that travel with the piece.

The frontend needs its own global settings system for application-wide preferences that persist across sessions, independent of any piece. Examples: theme selection, UI language, editor preferences.

These two systems are fundamentally different:

| | Backend ([ADR-0016](0016-Settings.md)) | Frontend (this ADR) |
|---|---|---|
| **Scope** | Per-piece, per-entity | Global, app-wide |
| **Storage** | defrecord `:settings` map, serialized with piece | EDN file on disk |
| **Lifecycle** | Lives with piece data | Persists across app sessions |
| **Mechanism** | Multimethods, VPD dispatch, STM | Atom, write-through to disk |
| **Validation** | Set or predicate | Set with tr-keyed choice labels, or predicate |
| **API** | Generated `get-<name>` / `set-<name>` per setting | Generic `get-app-setting` / `set-app-setting!` |
| **UI integration** | None (backend) | Choices map with translated labels for Settings dialog |

### Requirements

1. **Persistence**: Settings survive application restarts
2. **Defaults**: Ship default values as a generated classpath resource, always available
3. **Validation**: Prevent invalid values at write time (enumerated choices or predicate)
4. **Translation**: Setting names, descriptions, and choice labels must be translatable ([ADR-0039](0039-Localisation-Architecture.md))
5. **Introspection**: A registry of all settings enables automatic Settings dialog generation
6. **Simplicity**: No component machinery — settings are global state, an atom is the honest representation
7. **Write-through**: Every change persists immediately — settings change rarely, I/O cost is negligible

## Decision

### Architecture Overview

Frontend settings use a `defonce` atom holding a map of namespaced keywords to values. Settings are lazy-loaded on first access: defaults from a generated classpath resource, overridden by the user's settings file (if it exists). Every change writes the full map to disk immediately and publishes a `:setting-changed` event on the frontend event bus ([ADR-0031](0031-Frontend-Event-Driven-Architecture.md)).

A settings registry (separate from the settings atom) holds metadata about each setting: default value, validator, and choice labels. Default values are declared in `def-app-setting` alongside validation rules. A `lein frontend-settings` build task generates the defaults resource file from these declarations — the generated file is the runtime artifact, not a hand-maintained source of truth. This registry enables the Settings dialog to enumerate settings by category and render appropriate controls.

### Setting Definitions

Each setting is defined using `def-app-setting` and a corresponding `tr-declare`. Default values are declared in `def-app-setting` alongside validation rules — `def-app-setting` is the single source of truth for each setting's default, validation, and UI metadata.

```clojure
(tr-declare
  {:setting.ui.theme            "Theme"
   :setting.ui.theme.desc       "Visual theme for the application"
   :setting.ui.theme.nord-dark  "Dark"
   :setting.ui.theme.nord-light "Light"})

(def-app-setting :ui/theme
  {:default :nord-dark
   :choices {:nord-dark  :setting.ui.theme.nord-dark
             :nord-light :setting.ui.theme.nord-light}})
```

Settings use namespaced keywords with the namespace as category:
- `:ui/theme` — category `:ui`, setting `:theme`
- `:editor/auto-save` — category `:editor`, setting `:auto-save`

Categories map to tabs in the Settings dialog.

### Validation Architecture

Two validation modes, mirroring [ADR-0016](0016-Settings.md)'s pattern:

**Choices map** — for enumerated values. The map keys are the valid values; the map values are `tr` keys for display labels. Validation is automatic (`contains?` on keys). The Settings dialog renders a dropdown.

```clojure
(def-app-setting :ui/theme
  {:default :nord-dark
   :choices {:nord-dark  :setting.ui.theme.nord-dark
             :nord-light :setting.ui.theme.nord-light}})
```

**Validator function** — for open-ended values. A predicate function that returns truthy for valid values. Validator-based settings include an optional `:control` key that declares the UI control type for the Settings dialog:

```clojure
(def-app-setting :user/email
  {:default   ""
   :validator valid-email?
   :control   :text})

(def-app-setting :editor/auto-save
  {:default   true
   :validator boolean?
   :control   :toggle})

(def-app-setting :editor/font-size
  {:default   12
   :validator pos-int?
   :control   :number})
```

**Control type rules:**
- `:choices` present → dropdown (always; `:control` ignored if supplied)
- `:validator` + `:control` → render the declared control
- `:validator` without `:control` → text field (safe default)

Supported control types: `:text`, `:toggle`, `:number`, `:path`. This is a small, closed set — extensible later if needed. Inferring control type from the predicate function (e.g. detecting `boolean?`) is unreliable because function equality is not guaranteed in Clojure; explicit declaration eliminates all ambiguity.

Invalid values throw exceptions with clear error messages, matching [ADR-0016](0016-Settings.md)'s fail-fast behaviour.

**Closed mutation surface:** `set-app-setting!` refuses to set keys that are not registered via `def-app-setting`. This ensures every writable key has validation rules and a corresponding default — no unregistered keys can enter the settings map through the public API.

### Translation Integration

All user-facing strings follow [ADR-0039](0039-Localisation-Architecture.md). Translation keys use a convention derived from the setting keyword:

| Purpose | Convention | Example |
|---------|-----------|---------|
| Setting name | `:setting.<category>.<name>` | `:setting.ui.theme` |
| Description | `:setting.<category>.<name>.desc` | `:setting.ui.theme.desc` |
| Choice label | `:setting.<category>.<name>.<value>` | `:setting.ui.theme.nord-dark` |
| Category tab | `:setting-category.<category>` | `:setting-category.ui` |

All keys must appear as literals in `tr-declare` — no computed keys ([ADR-0039](0039-Localisation-Architecture.md) invariant 7). The `def-app-setting` choices map references the same literal tr keys. At runtime, the settings infrastructure calls `tr` with keys from the registry; the scanner finds these keys in `tr-declare`, not in the `tr` call sites.

### Storage

**Defaults resource:** `shared/resources/app-settings/defaults.edn`
- **Generated** by `lein frontend-settings` from `def-app-setting` declarations — not hand-maintained
- Ships with the application, always available via `io/resource`
- Accessible from shared tests
- Contains the complete default settings map

```edn
;; GENERATED — do not edit. Run: lein frontend-settings
{:ui/theme :nord-dark}
```

**User settings file:** Platform-specific via `platform/get-platform-directory`
- Path: `~/.ooloi/app-settings/settings.edn` (Unix/macOS) or `%APPDATA%/Ooloi/app-settings/settings.edn` (Windows)
- Created on first setting change (not on startup)
- Contains the **complete effective settings map** — not a diff or overlay
- The file is replaced in its entirety on every write

**Merge strategy:** Defaults loaded from resource, then overridden by user file. This merge is necessary to pick up new settings added in future versions — the user file from an older version won't contain them.

**Forward compatibility:** Unknown keys in the user file are preserved unconditionally. Validation applies only to `set-app-setting!` calls, never on load. The load path does not reject or drop any keys, ensuring that a newer version's settings file remains readable by an older version.

**Write-through:** Every `set-app-setting!` call writes the full settings map to disk immediately via atomic write (write to temp file, then rename). Settings change rarely (user toggles a preference); the I/O cost is negligible. The atomic rename guarantees that a crash mid-write cannot corrupt the settings file — the previous version remains intact until the rename completes.

### API

Module: `frontend/src/main/clojure/ooloi/frontend/ui/app_settings.clj`

```clojure
;; Read a setting (lazy-loads on first access)
(get-app-setting :ui/theme)    ; => :nord-dark

;; Write a setting (validates, updates atom, writes to disk, publishes event)
(set-app-setting! :ui/theme :nord-light)

;; Register a setting definition (called at namespace load time)
(def-app-setting :ui/theme
  {:default :nord-dark
   :choices {:nord-dark  :setting.ui.theme.nord-dark
             :nord-light :setting.ui.theme.nord-light}})
```

**Event integration:** Every `set-app-setting!` call publishes a `{:type :setting-changed :key <key> :old-value <old> :new-value <new> :timestamp <ms>}` event to the `:settings` category on the frontend event bus ([ADR-0031](0031-Frontend-Event-Driven-Architecture.md)). The UI Manager subscribes to this category and dispatches reactions — e.g. a theme change triggers `apply-theme-to-all!`.

**Lazy loading:** Settings are loaded on first `get-app-setting` or `set-app-setting!` call, not at an explicit startup point. This eliminates ordering dependencies in the startup sequence. The load itself is a pure EDN file read with no dependency on i18n or other infrastructure — so even if `:ui/language` is an app setting, the i18n system can safely call `get-app-setting` during `set-locale!` without circular dependencies.

**Thread safety:** First access can occur from multiple threads simultaneously (e.g. UI thread + i18n init). The lazy load uses a `delay` to guarantee exactly-once evaluation: the `delay` performs the merge of defaults and user file, and the settings atom is initialized from the realized delay. The `delay` is thread-safe by JVM guarantee — concurrent derefs block until the single evaluation completes. Settings load exactly once with no interleaved partial state.

`def-app-setting` is a function, not a macro. It performs a `swap!` into the registry atom — no code generation is needed (unlike [ADR-0016](0016-Settings.md)'s multimethod generation). A macro could theoretically provide compile-time validation that referenced tr keys exist, but this would create a build-time dependency on the i18n scanner that doesn't exist in that direction. The existing build-time i18n verification already catches missing keys separately.

### Settings Registry

A `defonce` atom holding metadata for all registered settings, including defaults:

```clojure
{:ui/theme    {:default :nord-dark
               :choices {:nord-dark  :setting.ui.theme.nord-dark
                         :nord-light :setting.ui.theme.nord-light}}
 :user/email  {:default   ""
               :validator valid-email?
               :control   :text}
 :editor/auto-save {:default   true
                    :validator boolean?
                    :control   :toggle}}
```

Every `def-app-setting` call populates the registry with the default value, validation rules, and UI metadata. Since defaults and validation come from the same declaration, they cannot diverge.

The registry enables:
- **Settings dialog generation**: enumerate all settings, group by category (keyword namespace), render appropriate controls (dropdown for choices, declared control type for validators)
- **Validation**: `set-app-setting!` looks up the validator/choices from the registry
- **Reset to default**: the Settings dialog reads the `:default` from the registry
- **Defaults resource generation**: `lein frontend-settings` extracts `:default` values from the registry to produce the classpath resource

### Settings Dialog

The settings registry provides everything needed to generate a Settings dialog:

- **Tabs**: one per category (`:ui`, `:editor`, etc.), labels via `tr` with convention `:setting-category.<category>` (e.g. `(tr :setting-category.ui)`)
- **Controls**: deterministic mapping from registry metadata:
  - `:choices` → dropdown (always)
  - `:validator` + `:control :text` → text field
  - `:validator` + `:control :toggle` → toggle switch
  - `:validator` + `:control :number` → numeric stepper
  - `:validator` + `:control :path` → path picker
  - `:validator` + no `:control` → text field (default)
- **Labels**: setting name via `(tr :setting.ui.theme)`, description via `(tr :setting.ui.theme.desc)`
- **Choice labels**: `(tr :setting.ui.theme.nord-dark)` → "Dark"
- **Validation feedback**: immediate validation on input change

The Settings dialog is implemented in `settings_dialog.clj`, consuming the registry to generate the UI automatically. It follows the content builder pattern (ADR-0042): `build-settings-content!` (private) constructs a TabPane with one tab per category, and `show-settings!` (public) handles lifecycle via the UI Manager. Each tab contains a ScrollPane of setting rows — `Tile` + `ComboBox` for choices, `Tile` + `TextField` for validator-based settings — each with a per-field reset button. A unified bottom bar provides Reset All Defaults (scoped to the active tab's category) and OK.

## Rationale

### Single Source of Truth for Defaults

Each setting's default value is declared exactly once, in its `def-app-setting` call. The classpath resource file is generated from these declarations by `lein frontend-settings` — it is a build artifact, not a source of truth. This eliminates the risk of divergence between declaration and resource: there is only one declaration site. This mirrors [ADR-0016](0016-Settings.md)'s pattern where `defsetting` declares the default inline.

### Lazy Loading Eliminates Ordering Dependencies

Settings are loaded on first access, not via an explicit init call. This eliminates startup ordering concerns entirely. In particular, if `:ui/language` is an app setting, the i18n system can read it during `set-locale!` without requiring settings to be loaded before `init-locales!`. The load is a pure EDN file read — no dependency on i18n, JavaFX, or any other infrastructure.

### Separation from Backend Settings

[ADR-0016](0016-Settings.md) provides per-piece, per-entity settings using multimethods, VPD dispatch, and STM transactions. These are fundamentally different concerns: piece settings travel with musical data and use backend machinery; app settings are global user preferences persisted locally. Combining them would obscure both systems and violate the boundary established by [ADR-0015](0015-Undo-and-Redo.md).

### Atom Over Integrant Component

Application settings are inherently global state. Wrapping them in an Integrant component would add dependency injection ceremony without changing their nature. An atom is the honest representation — it is what it is, with no lifecycle pretence.

### Validation Pattern Consistency

The two-mode validation (set-based, predicate-based) mirrors [ADR-0016](0016-Settings.md)'s proven pattern. Frontend adds translated choice labels for UI rendering — a natural extension that the backend doesn't need because it has no user-facing settings UI.

### Translation Key Convention

The `setting.<category>.<name>` convention enables systematic key derivation while keeping all keys as literals in source (satisfying [ADR-0039](0039-Localisation-Architecture.md) invariant 7). The build-time scanner finds keys in `tr-declare` maps; runtime `tr` calls use keys from the registry (variables, silently skipped by scanner — this is the intended data-driven pattern documented in ADR-0039).

## Implementation

The frontend settings system is implemented in `frontend/src/main/clojure/ooloi/frontend/ui/app_settings.clj` with:

- `def-app-setting` — registers setting metadata (default, choices/validator) in the registry
- `get-app-setting` — reads from settings atom; lazy-loads on first call
- `set-app-setting!` — validates, updates atom, writes to disk, publishes `:setting-changed` event

Default values are declared in `def-app-setting` and extracted into `shared/resources/app-settings/settings.edn` by the `lein frontend-settings` build task. User settings are stored in the platform-specific directory via `platform/get-platform-directory "Ooloi" "app-settings"`.

The Settings dialog (`settings_dialog.clj`) consumes the registry to generate UI controls automatically, following the content builder pattern (ADR-0042). `system.clj` maps the `:ui/open-settings` action handler to a one-line delegation to `show-settings!`.

## Consequences

### Positive

1. **Simple and honest**: Global state in an atom — no unnecessary machinery
2. **Persistent**: Settings survive restarts via write-through EDN
3. **Introspectable**: Registry enables automatic Settings dialog generation
4. **Translatable**: Full i18n integration for names, descriptions, and choice labels
5. **Validated**: Invalid values prevented at write time
6. **Consistent**: Validation patterns mirror [ADR-0016](0016-Settings.md) (set/predicate)
7. **Accessible**: Defaults on classpath, available from all test suites
8. **Forward-compatible**: Unknown keys in user file preserved across upgrades
9. **No ordering dependencies**: Lazy loading eliminates startup sequence constraints
10. **Single source of truth**: Default values declared once in `def-app-setting`, resource file generated
11. **Event-driven reactions**: Setting changes publish to frontend event bus, enabling decoupled responses

### Negative

1. **Global mutable state**: Atom is accessible from anywhere — no dependency injection discipline
2. **Full file rewrite**: Every setting change rewrites the entire file (acceptable for small files)
3. **Lazy load timing**: First access incurs file I/O (negligible for small EDN files)

### Mitigations

1. Global state is the honest representation of app settings — injection would add ceremony without benefit
2. Settings files are small (tens of entries); full rewrite is negligible I/O
3. First-access load is a single small file read; subsequent accesses are pure atom derefs

## Alternatives Considered

### Integrant Component

Wrap settings in an Integrant component with init/halt lifecycle.

**Rejected because:**
- Settings are inherently global — component wrapping adds ceremony without changing the nature
- Every module reading a setting would need the component injected as a parameter
- The lifecycle is trivial (load a file, keep an atom) — doesn't warrant component machinery
- Existing pattern (`theme.clj` uses `defonce` atom) works well

### Extend ADR-0016 (Backend Settings)

Add frontend settings to the existing backend settings ADR.

**Rejected because:**
- Fundamentally different systems: per-piece vs global, STM vs atom, VPD vs plain map
- [ADR-0016](0016-Settings.md) explicitly states it applies "exclusively to piece data entities"
- Merging would obscure both systems
- Different validation needs (backend has no UI, frontend needs translated choice labels)

### Property Files or JSON

Use standard Java properties or JSON instead of EDN.

**Rejected because:**
- EDN is native to Clojure — no conversion needed
- EDN preserves keywords, which are the setting keys
- Consistent with existing persistence patterns (window state uses EDN)
- EDN supports all Clojure data types without custom serialization

### Defaults in Separate Resource File

Maintain default values in a hand-edited resource file (`settings.edn`), separate from `def-app-setting` declarations.

**Rejected because:**
- Creates two locations where developers must look to understand a setting (definition + resource file)
- Risk of divergence: a developer adds `def-app-setting` but forgets to update the resource file (or vice versa)
- Generating the resource file from `def-app-setting` declarations eliminates this class of error entirely
- `def-app-setting` with `:default` mirrors [ADR-0016](0016-Settings.md)'s `defsetting` pattern — consistency across the codebase

### Explicit Startup Init

Require `load-app-settings!` call in system.clj startup sequence.

**Rejected because:**
- Creates ordering dependency: settings must load before any consumer
- If `:ui/language` is a setting, i18n init must happen after settings init
- Lazy loading eliminates this constraint entirely
- First-access load is a negligible cost for small EDN files

## Related ADRs

- [ADR-0015: Undo and Redo](0015-Undo-and-Redo.md) — Establishes that backend has no application settings; user preferences managed by frontend
- [ADR-0016: Settings](0016-Settings.md) — Backend per-piece entity settings; validation pattern (set/predicate) borrowed here
- [ADR-0039: Localisation Architecture](0039-Localisation-Architecture.md) — Translation integration, `tr`/`tr-declare` API, literal key constraint
- [ADR-0031: Frontend Event-Driven Architecture](0031-Frontend-Event-Driven-Architecture.md) — Frontend event bus; `set-app-setting!` publishes `:setting-changed` events
- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) — Dialog/window patterns for Settings dialog implementation

## Related Guides

- [ADR-0044: MIDI Input Library and Boundary Architecture](0044-MIDI-Input-Library-and-Boundary-Architecture.md) — Device selection persistence via `def-app-setting` is cited in ADR-0044 as a consequence of the MIDI input architecture
- [MIDI in Ooloi](../guides/MIDI_IN_OOLOI.md) — MIDI device selection is stored as `:midi/input-device-id` in the frontend settings system; the settings dialog auto-generates a dropdown populated with currently available physical input devices

## Code References

- `frontend/src/main/clojure/ooloi/frontend/ui/app_settings.clj` — Settings system implementation
- `frontend/src/main/clojure/ooloi/frontend/ui/settings_dialog.clj` — Settings dialog (content builder pattern, ADR-0042)
- `shared/resources/app-settings/defaults.edn` — Generated defaults resource (produced by `lein frontend-settings`)
- `frontend/src/main/clojure/ooloi/frontend/ui/theme.clj` — Theme module consuming settings
- `shared/src/main/clojure/ooloi/shared/platform.clj` — Platform-specific directory paths
- `frontend/src/main/clojure/ooloi/frontend/ui/persistence.clj` — Existing EDN persistence pattern
