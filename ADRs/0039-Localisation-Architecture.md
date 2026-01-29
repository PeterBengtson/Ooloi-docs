# ADR-0039: Localisation Architecture

## Table of Contents
- [Status](#status)
- [Context](#context)
  - [Architectural Boundary](#architectural-boundary)
  - [Current State](#current-state)
- [Decision](#decision)
  - [Frontend-Only Localisation](#frontend-only-localisation)
  - [Single Translation API](#single-translation-api)
  - [PO Files as Translator Interface](#po-files-as-translator-interface)
  - [Distribution Model](#distribution-model)
  - [Runtime Loading and Caching](#runtime-loading-and-caching)
- [Key Design](#key-design)
  - [Key Structure and Context](#key-structure-and-context)
  - [Literal-Only Constraint](#literal-only-constraint)
  - [Named Parameters](#named-parameters)
  - [Plural Forms](#plural-forms)
- [Plugin Localisation](#plugin-localisation)
- [Build-Time Verification](#build-time-verification)
  - [Canonical Completeness (Hard Gate)](#canonical-completeness-hard-gate)
  - [Non-English Coverage (Soft)](#non-english-coverage-soft)
- [Forward Compatibility](#forward-compatibility)
- [Implementation Strategy](#implementation-strategy)
  - [PO File Parsing: Library-Based](#po-file-parsing-library-based)
  - [Translation API: Custom Implementation](#translation-api-custom-implementation)
  - [Named Parameter Substitution: Custom Implementation](#named-parameter-substitution-custom-implementation)
  - [Key Extraction: Adapted from Pottery](#key-extraction-adapted-from-pottery)
  - [Caching: Custom Implementation](#caching-custom-implementation)
  - [Plural Rule Resolution: Lookup Table](#plural-rule-resolution-lookup-table)
  - [Build-Time Verification: Custom Leiningen Task](#build-time-verification-custom-leiningen-task)
  - [Dependency Summary](#dependency-summary)
- [Invariants](#invariants)
- [Out of Scope](#out-of-scope)
- [Consequences](#consequences)
- [References](#references)

## Status

Accepted

## Context

### Architectural Boundary

Ooloi's client-server architecture establishes a clear boundary for localisation: the backend is a deterministic engine that must never produce locale-dependent output. Collaboration correctness, protocol stability, and reproducible behaviour all depend on this separation.

The backend emits:
- Stable English strings in exceptions and error results
- Structured data (maps, enums, identifiers)
- Event payloads with language-neutral content

These are protocol artifacts. They may appear in logs, be matched against programmatically, or be used for debugging. They are not user-facing messages and do not enter the translation system.

When the frontend needs to display something to the user in response to a backend state, it interprets that state and emits its own translated message. The backend string is a stable contract; the translation key is frontend-owned.

### Current State

Ooloi currently has zero user-facing strings. This is the ideal adoption point for localisation infrastructure: the non-string policy can be established as an invariant rather than retrofitted onto existing code. Translator ergonomics is a first-class architectural concern from the outset.

## Decision

### Frontend-Only Localisation

All localisation occurs in the frontend. The backend never localises and never emits user-facing strings.

This preserves:
- **Determinism**: Same input produces same output regardless of locale
- **Protocol stability**: Backend contracts don't change with UI language
- **Collaboration correctness**: All clients see semantically identical data
- **Separation of concerns**: Presentation logic stays in the presentation layer

### Single Translation API

Every user-facing string passes through a single translation function:

```clojure
(tr :menu.file.open)
(tr :dialog.export.warning {:filename "score.ooloi"})
```

Where:
- First argument is a literal keyword (the translation key)
- Optional second argument is a map of named parameters

No other mechanism produces user-visible text. No string concatenation, no inline literals, no positional formatting.

### PO Files as Translator Interface

Translators work exclusively with PO (Portable Object) files—the standard format established by GNU gettext in 1995 and used across thousands of projects since.

**Why PO files:**

PO is the lingua franca of software translation. Professional translators and volunteer contributors alike know this format. It enables a mature ecosystem of specialised tools:

- **Poedit**: Cross-platform desktop editor with translation memory, quality checks, and cloud sync
- **Lokalize**: KDE's translation tool with terminology management and diff views
- **Weblate**, **Pontoon**, **Crowdin**: Web-based platforms for collaborative translation projects
- **GT4T**, **OmegaT**: CAT (Computer-Assisted Translation) tools that integrate with PO workflows
- **msgfmt**, **msgmerge**: GNU command-line tools for validation and merging

Translators never touch Clojure, EDN, JavaFX, or repository internals. They receive PO files, edit them in familiar tools, and return updated PO files.

**Message structure:**

```po
msgctxt "menu.file.open"
msgid "Open…"
msgstr ""

msgctxt "dialog.export.warning"
msgid "Exporting will overwrite %{filename}."
msgstr ""
```

- `msgctxt`: The translation key (context), matching the keyword used in code
- `msgid`: Canonical English text
- `msgstr`: Target translation (empty in source file, filled by translator)

Parameters use named placeholders (`%{filename}`), never positional (`%s`, `%1$d`).

### Distribution Model

Localisation files are loaded from two locations:

1. **Bundled canonical locale**: `en-GB.po` ships inside the application JAR, immutable and version-matched
2. **External directory**: Additional locales and overrides, platform-specific location via `get-platform-directory`

```clojure
;; External locale directory
(get-platform-directory "Ooloi" "i18n")
;; Windows: %APPDATA%/Ooloi/i18n
;; Unix/macOS: ~/.ooloi/i18n
```

```
# Bundled in JAR (read-only)
resources/i18n/
  en-GB.po        ; canonical source (for reference)
  en-GB.edn       ; pre-compiled cache (for runtime)

# External (user-managed, platform-specific)
# Windows: %APPDATA%/Ooloi/i18n/
# Unix/macOS: ~/.ooloi/i18n/
  sv.po           ; Swedish
  de.po           ; German
  en-US.po        ; American English
  en-GB.po        ; optional override of bundled canonical
  .cache/
    sv.edn        ; auto-generated runtime cache
    de.edn
    en-US.edn
```

**Canonical locale:** The canonical source language is UK English (`en-GB`), not generic "en". This is the baseline for all translations and the fallback when a key is missing in other locales. American English (`en-US`) is a separate translation like any other, differing in spelling (colour/color, localisation/localization), punctuation conventions, and occasional terminology.

**Rationale:**

- Canonical locale always present—no missing-file failure mode
- New translations do not require a new Ooloi release
- Community-contributed locales can be distributed independently
- Translators can test updates without rebuild cycles
- The localisation layer survives forward UI evolution
- External `en-GB.po` can override bundled version for testing or customisation

### Runtime Loading and Caching

**Bundled canonical locale:**

The canonical `en-GB.po` is compiled to `en-GB.edn` during the build process. Both files ship in the JAR—the PO for reference, the EDN for runtime use. No parsing occurs at startup for the default locale.

**External locales:**

At first use of an external locale:

1. Load bundled UK English cache (always present, pre-compiled)
2. Check external directory for override (`i18n/en-GB.po`)—if present and newer than bundled, parse and cache
3. For the selected locale:
   - If compiled cache exists and is newer than source PO → load cache
   - Otherwise → parse PO, compile to EDN cache in external directory, load
4. If locale file is missing → fall back to UK English

**Cache format:**

```clojure
{:plural-rule "(n != 1)"  ; original expression for reference
 :messages {:menu.file.open "Öppna…"
            :dialog.export.warning "Export kommer att skriva över %{filename}."
            :files {:one "fil" :other "filer"}}}
```

**Plural rule resolution:**

The `Plural-Forms` expression from the PO header is resolved to a pre-defined Clojure function via lookup table, not runtime compilation. The deployed application uses `jlink` for a minimal JVM runtime without the Clojure compiler, so `eval` is unavailable.

Gettext plural patterns are well-documented and finite—approximately 15-20 patterns cover all real-world languages:

```clojure
(def plural-rules
  {"(n != 1)"       (fn [n] (if (= n 1) 0 1))       ; English, German, Swedish, etc.
   "(n > 1)"        (fn [n] (if (> n 1) 1 0))       ; French, Brazilian Portuguese
   "(n == 0 || n == 1)"  ...                        ; etc.
   })
```

If an unknown pattern appears (unlikely), the system logs a warning and falls back to treating all forms as singular. This is a visible degradation, not a crash.

## Key Design

### Key Structure and Context

Translation keys are architectural identifiers encoding UI structure, not linguistic constructs:

```
menu.file.open
menu.file.save-as
dialog.export.warning
dialog.export.success
palette.noteheads.title
collaboration.invite.pending
error.file.not-found
```

The key directly becomes the `msgctxt` in the PO file. This allows translators to:
- Disambiguate identical English strings appearing in different contexts
- Understand UI placement (menu vs dialog vs tooltip vs error)
- Detect insufficient context and flag issues

### Literal-Only Constraint

Translation keys in frontend code must be literal keywords, never computed:

```clojure
;; Allowed
(tr :menu.file.open)
(tr :dialog.export.warning {:filename name})

;; Forbidden
(tr (keyword (str "menu." section "." action)))
(tr (get key-map some-key))
```

This constraint enables deterministic key extraction. A simple grep or static parse of the codebase produces the complete set of translation keys. Computed keys would require runtime tracing or program analysis, neither of which provides completeness guarantees.

### Named Parameters

Parameters use named placeholders exclusively:

```clojure
(tr :export.files-written {:count 3 :destination "/path"})
```

```po
msgctxt "export.files-written"
msgid "Wrote %{count} files to %{destination}."
msgstr "Skrev %{count} filer till %{destination}."
```

Named parameters allow translators to reorder elements for grammatical correctness without code changes. Swedish might need "Till %{destination} skrevs %{count} filer." Positional formatting (`%1$s`, `%2$d`) makes this fragile and error-prone.

### Plural Forms

Plural handling uses standard gettext conventions. The PO header declares the plural rule:

```po
"Plural-Forms: nplurals=2; plural=(n != 1);\n"
```

Plural messages use indexed forms:

```po
msgctxt "files.selected"
msgid "file"
msgid_plural "files"
msgstr[0] "fil"
msgstr[1] "filer"
```

The translation API accepts a count parameter:

```clojure
(tr :files.selected {:n 1})   ; → "fil"
(tr :files.selected {:n 5})   ; → "filer"
```

At load time, the `Plural-Forms` expression is matched against a lookup table of pre-defined functions (see Runtime Loading and Caching). Runtime plural selection is a simple function call.

**Why gettext plurals over separate keys:**

- Translators expect this mechanism; it's standard in their tools
- Languages with complex plural rules (Russian: 4 forms; Arabic: 6 forms) are handled correctly
- Plural logic stays in the translation layer, not scattered through code

## Plugin Localisation

Plugins ship their own PO files in a dedicated directory within the plugin distribution:

```
my-plugin/
  plugin.edn
  i18n/
    en-GB.po
    sv.po
```

Plugin translation catalogs are namespaced by plugin identifier. No collision with core keys or other plugins is possible:

```clojure
;; Core translation
(tr :menu.file.open)

;; Plugin translation (hypothetical API)
(tr :plugin.my-plugin/palette.title)
;; or
(plugin-tr :my-plugin :palette.title)
```

The exact API is implementation detail, but the architectural requirement is clear: plugin catalogs are isolated namespaces, not merged into a global soup where naming discipline is the only defence against collision.

Plugins use the same PO format, same tooling, same workflow. A translator working on a plugin sees exactly the same file structure as core localisation.

## Build-Time Verification

### Canonical Completeness (Hard Gate)

At build time:

1. Extract all `tr` keys from frontend source (possible because keys are literals)
2. Parse `en-GB.po` and extract all `msgctxt` values
3. Assert: every key in code exists in `en-GB.po`

Build fails if the canonical UK English catalog is incomplete. This prevents shipping UI elements without English definitions.

Implementation is straightforward: grep for `(tr :` patterns, parse the literal keywords, compare against PO contents. The literal-only constraint makes this reliable.

### Non-English Coverage (Soft)

For other locales:

- Missing keys are allowed (fall back to UK English at runtime)
- Coverage percentage can be reported
- Missing keys do not block build or execution

This preserves forward compatibility: an old `sv.po` continues to work when new UI strings are added in a release. Partial translations degrade gracefully rather than failing.

## Forward Compatibility

**Invariant:** A new Ooloi version adding new UI strings must not break existing translations.

Mechanism:

- Missing key in locale → return UK English text
- Missing key in UK English → return conspicuous placeholder (`[MISSING: key.name]`) and log warning, never crash
- Malformed PO file → log warning with line number and error details, fall back to en-GB for entire locale
- Never mix broken and working entries from same file

This ensures:

- Old locale files continue working with new releases
- Partial translations are usable, not fatal
- No localisation-layer failures during upgrades
- Translators can update at their own pace after a release

## Implementation Strategy

This architecture requires several distinct capabilities. The implementation strategy balances library reuse (for complex parsing) against custom code (for our specific constraints):

### PO File Parsing: Library-Based

**Decision:** Use [Potentilla](https://github.com/soberlemur/potentilla) (com.soberlemur:potentilla) for PO file parsing.

**Rationale:**
- PO format is complex: multiline strings, escape sequences, continuation syntax, various comment types
- Gettext plural forms header parsing requires language-specific knowledge
- Library handles edge cases we'd miss writing from scratch (~400-600 lines of complex parsing code)
- ANTLR-based parser: battle-tested parser technology with robust error handling
- Based on jgettext: production-proven code used by Zanata translation platform for years
- Modernized fork: updated dependencies, better test coverage, modular design
- Production-validated: used by autopo (AI-powered PO file manager, actively maintained Jan 2026)
- Minimal JVM interop cost with simple Java API

**Maintenance status (Jan 2026):**
- Version 0.0.3 (April 2025) - dormant for 9 months but stable
- Validated by autopo v1.0.8 (Jan 26, 2026) using it in production
- Based on mature jgettext codebase with years of production use
- Early version number reflects modernization work, not parser maturity

**Scope:** Potentilla handles:
- PO/POT file format parsing with ANTLR grammar
- Multiline string handling and all escape sequences (`\n`, `\t`, `\"`, `\\`)
- UTF-8 BOM handling and CRLF/LF line ending normalization
- Plural forms header extraction (all languages including Russian 4-form, Arabic 6-form)
- Malformed input error recovery and reporting
- Comment types (translator, source reference, general)

### Translation API: Custom Implementation

**Decision:** Build custom `tr` function implementing our specific constraints.

**Rationale:**
- Existing libraries use positional parameters (`%s`, `{0}`) — we require named (`%{param}`)
- Our API must enforce literal keyword keys — no library does this
- We need tight integration with our EDN cache format
- Simple implementation: keyword lookup, named parameter substitution, plural selection

**Signature:**
```clojure
(tr :translation.key)                          ; Simple lookup
(tr :translation.key {:param "value"})         ; Named parameters
(tr :translation.key {:n 5})                   ; Plural forms
```

### Named Parameter Substitution: Custom Implementation

**Decision:** Custom string substitution replacing `%{name}` with actual values.

**Rationale:**
- No existing library supports this syntax
- Straightforward: regex or simple parser
- ~50 lines of code

**Implementation:**
```clojure
;; Input: "Export will overwrite %{filename}."
;; Params: {:filename "score.ooloi"}
;; Output: "Export will overwrite score.ooloi."
```

### Key Extraction: Custom Implementation

**Decision:** Custom source code scanner with literal-only verification.

**Rationale:**
- Simple pattern: scan for `(tr :keyword)` using regex or parser
- Literal-only enforcement: reject computed keys at build time
- ~100 lines: file traversal + keyword extraction + verification
- Build-time completeness check: all keys exist in `en-GB.po`

### Caching: Custom Implementation

**Decision:** Custom EDN cache generation and loading.

**Rationale:**
- Our cache format is simple: `{:messages {:key "text" ...} :plural-rule "(n != 1)"}`
- Potentilla gives us parsed PO data, we serialize to EDN
- Platform-specific external directory requires custom path logic
- Cache invalidation based on file timestamps (enables hot reload without restart)

**Cache structure:**
```clojure
{:plural-rule "(n != 1)"
 :messages {:menu.file.open "Open…"
            :dialog.export.warning "Export will overwrite %{filename}."
            :files {:one "file" :other "files"}}}
```

### Plural Rule Resolution: Lookup Table

**Decision:** Pre-defined function lookup table, not runtime compilation.

**Rationale:**
- Deployed app uses jlink minimal JVM without Clojure compiler
- Gettext plural patterns are finite (~15-20 covering all languages)
- Safe: unknown pattern logs warning, falls back to singular form

**Example:**
```clojure
(def plural-rules
  {"(n != 1)"              (fn [n] (if (= n 1) 0 1))       ; English, German, Swedish
   "(n > 1)"               (fn [n] (if (> n 1) 1 0))       ; French, Portuguese
   "(n % 10 == 1 && ...)"  (fn [n] ...)                    ; Russian (4 forms)
   ...})
```

### Build-Time Verification: Custom Leiningen Task

**Decision:** Custom build task extracting and verifying keys.

**Rationale:**
- Simple: grep source for `(tr :keyword)` patterns
- Parse literal keywords (reject non-literals)
- Compare against `en-GB.po` msgctxt values
- Hard gate: build fails if canonical catalog incomplete

### Dependency Summary

**Added to frontend/project.clj:**
```clojure
[com.soberlemur/potentilla "0.0.3"]  ; PO file parsing (ANTLR-based, Java interop)
```

**Custom implementation:**
- `ooloi.frontend.i18n.core` — `tr` function, named parameters, plural selection
- `ooloi.frontend.i18n.cache` — EDN caching, external directory, timestamps
- `ooloi.frontend.i18n.build` — Key extraction, verification, completeness check
- `ooloi.frontend.i18n.plural` — Plural rule lookup table
- `ooloi.frontend.i18n.po` — Thin wrapper around potentilla Java API

**Why this split:**
- Library for complex parsing (saves ~400-600 lines, handles edge cases with ANTLR)
- Custom for our constraints (named parameters, literal keys, our cache format)
- Clear ownership: we control the API, rely on library for format complexity
- Testable: mock potentilla output, test our logic independently
- Low risk: production-validated parser, minimal interop cost

## Invariants

These are architectural constraints, not conventions. Violating any of them breaks determinism, translator usability, or both.

1. **Backend never emits localised strings.** Backend outputs are protocol artifacts: stable English for logs, exceptions, and programmatic matching.

2. **Frontend contains zero inline user-facing strings.** Every user-visible string passes through `tr`.

3. **All localisation via a single API.** One function, one mechanism, no alternatives.

4. **Named parameters only.** No positional formatting (`%s`, `%d`, `%1$s`).

5. **No sentence assembly from fragments.** Messages are complete translatable units, not concatenated pieces.

6. **No localisation logic in protocols, persistence, or collaboration paths.** These layers are locale-agnostic.

7. **Translation keys are literal.** No computed keys, enabling deterministic extraction.

8. **PO files are the sole translator interface.** Translators never edit code, EDN, or internal formats.

## Out of Scope

This ADR addresses **string translation only**. Related concerns handled separately:

**Date/time/number formatting:**
- Locale-specific formatting ("3,14" vs "3.14", date order variations, thousand separators)
- Separate concern requiring different infrastructure (not gettext-based)
- Future ADR if needed for user-configurable preferences

**RTL language support (Arabic, Hebrew):**
- Pure rendering concern, handled by JavaFX automatically
- Translation infrastructure is direction-agnostic
- Strings should not contain directional markup (Unicode bidi control characters)
- Text rendering engine handles bidi algorithm

**Currency/unit conversion:**
- "5 miles" vs "8 kilometers" requires domain logic, not translation
- Out of scope for string localisation
- Application-specific logic if needed

## Consequences

**Positive:**

1. **Clean adoption point** — Zero existing strings means the policy is established, not retrofitted
2. **Translator-native workflow** — PO files and standard tooling, no Ooloi-specific learning curve
3. **Guaranteed baseline** — Canonical UK English bundled in JAR, no installation failure modes
4. **Zero startup overhead** — Bundled locale is pre-compiled; no parsing for default case
5. **Independent release cycles** — Non-English translations ship separately from application releases
6. **Forward compatibility** — New versions don't break existing translations
7. **Deterministic extraction** — Literal-only keys enable reliable build-time verification
8. **Plugin parity** — Plugins use identical localisation infrastructure
9. **Grammatical flexibility** — Named parameters allow reordering across languages
10. **Correct plural handling** — Gettext mechanism handles all language complexity
11. **No migration required** — Design is complete from the start

**Neutral:**

1. **PO parsing for external locales** — One-time cost per locale at first use, cached thereafter; bundled canonical is pre-compiled
2. **External directory** — User creates platform-specific `i18n/` directory for additional locales; not required for basic operation
3. **Two-file workflow** — Translators work with PO, runtime uses EDN cache (but this is invisible to translators)

**Negative:**

1. **Literal-only constraint** — Some dynamic UI patterns require workarounds (acceptable trade-off for extraction reliability)

## Related ADRs

- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) - Uses translation keys for all UI strings as specified in this ADR

## References

- GNU gettext manual: https://www.gnu.org/software/gettext/manual/
- PO file format specification: https://www.gnu.org/software/gettext/manual/html_node/PO-Files.html
- Poedit: https://poedit.net/
- Lokalize: https://apps.kde.org/lokalize/
- Potentilla: https://github.com/soberlemur/potentilla (ANTLR-based PO parser)
- Autopo: https://github.com/soberlemur/autopo/ (production validation of potentilla)
- Maven Central: https://central.sonatype.com/artifact/com.soberlemur/potentilla
- `ooloi.shared.platform` namespace for cross-platform directory conventions