# ADR-0039: Localisation Architecture

## Status

Implemented

## Table of Contents
- [Context](#context)
  - [Architectural Boundary](#architectural-boundary)
  - [Adoption Point](#adoption-point)
- [Decision](#decision)
  - [Frontend-Only Localisation](#frontend-only-localisation)
  - [Source Language Scope: Strings, Not Identifiers](#source-language-scope-strings-not-identifiers)
  - [Single Translation API](#single-translation-api)
  - [PO Files as Translator Interface](#po-files-as-translator-interface)
  - [Distribution Model](#distribution-model)
  - [Currently Bundled Locales](#currently-bundled-locales)
  - [Runtime Loading and Caching](#runtime-loading-and-caching)
- [Key Design](#key-design)
  - [Key Structure and Context](#key-structure-and-context)
  - [Every Key Is Declared](#every-key-is-declared)
  - [Two Extraction Mechanisms](#two-extraction-mechanisms)
  - [Declaration Use Cases](#declaration-use-cases)
  - [Named Parameters](#named-parameters)
  - [Interpolated Names](#interpolated-names)
  - [Plural Forms](#plural-forms)
- [Plugin Localisation](#plugin-localisation)
- [Build-Time Verification](#build-time-verification)
  - [Two Operational Modes](#two-operational-modes)
  - [Canonical Completeness (Hard Gate)](#canonical-completeness-hard-gate)
  - [Plural Integrity (Hard Gate, Both Modes)](#plural-integrity-hard-gate-both-modes)
  - [Non-English Coverage](#non-english-coverage)
  - [What Verification Cannot See](#what-verification-cannot-see)
  - [Orphaned Keys](#orphaned-keys)
- [Forward Compatibility](#forward-compatibility)
- [Implementation Strategy](#implementation-strategy)
  - [PO File Parsing: Library-Based](#po-file-parsing-library-based)
  - [Translation API: Custom Implementation](#translation-api-custom-implementation)
  - [Named Parameter Substitution: Custom Implementation](#named-parameter-substitution-custom-implementation)
  - [Key Extraction: Custom Implementation](#key-extraction-custom-implementation)
  - [Locale Loading: Direct Parsing](#locale-loading-direct-parsing)
  - [Plural Rule Resolution: Lookup Table](#plural-rule-resolution-lookup-table)
  - [Build-Time Verification: Custom Leiningen Task](#build-time-verification-custom-leiningen-task)
  - [Dependency Summary](#dependency-summary)
- [Invariants](#invariants)
  - [1. Backend never emits localised strings](#1-backend-never-emits-localised-strings)
  - [2. Frontend contains zero inline user-facing strings](#2-frontend-contains-zero-inline-user-facing-strings)
  - [3. All localisation through a single API](#3-all-localisation-through-a-single-api)
  - [4. Named parameters only](#4-named-parameters-only)
  - [5. No sentence assembly from fragments](#5-no-sentence-assembly-from-fragments)
  - [6. No localisation logic in protocol, persistence, or collaboration paths](#6-no-localisation-logic-in-protocol-persistence-or-collaboration-paths)
  - [7. Every key is declared](#7-every-key-is-declared)
  - [8. PO files are the sole translator interface](#8-po-files-are-the-sole-translator-interface)
  - [9. Every bundled catalogue is grammatically correct for every value of n](#9-every-bundled-catalogue-is-grammatically-correct-for-every-value-of-n)
- [Documentation](#documentation)
- [Out of Scope](#out-of-scope)
- [Consequences](#consequences)
- [Related ADRs](#related-adrs)
- [References](#references)

## Context

### Architectural Boundary

Ooloi's client-server architecture establishes a clear boundary for localisation: the backend is a deterministic engine that must never produce locale-dependent output. Collaboration correctness, protocol stability, and reproducible behaviour all depend on this separation.

The backend emits:
- Stable English strings in exceptions and error results
- Structured data (maps, enums, identifiers)
- Event payloads with language-neutral content

These are protocol artifacts. They may appear in logs, be matched against programmatically, or be used for debugging. They are not user-facing messages and do not enter the translation system.

When the frontend needs to display something to the user in response to a backend state, it interprets that state and emits its own translated message. The backend string is a stable contract; the translation key is frontend-owned.

### Adoption Point

Localisation is established before the user-facing strings exist, rather than retrofitted onto them. That ordering is what allows the non-string policy to be an invariant the backend is built under, instead of a cleanup applied to code that already violates it, and it makes translator ergonomics a first-class concern from the outset rather than a late accommodation.

## Decision

### Frontend-Only Localisation

All localisation occurs in the frontend. The backend never localises and never emits user-facing strings.

This preserves:
- **Determinism**: Same input produces same output regardless of locale
- **Protocol stability**: Backend contracts don't change with UI language
- **Collaboration correctness**: All clients see semantically identical data
- **Separation of concerns**: Presentation logic stays in the presentation layer

### Source Language Scope: Strings, Not Identifiers

The canonical source language is UK English (see [Distribution Model](#distribution-model)), and
its authority runs over **resolved strings** — the text a user reads, declared in `tr-declare`
and carried by the catalogues. It does not extend to source code identifiers: record names, API
function names, keywords and namespaces follow the vocabulary of the formats and systems Ooloi
interoperates with.

The two vocabularies may therefore differ on the same object, and where they do the difference is
deliberate. A stave is the clearest case: every catalogue calls it a *stave*, `en_US` calls it a
*staff*, and the model record is `Staff` carrying an `add-staff` operation on the API.

**Identifiers are an interface, not prose.** Ooloi resolves API methods by name at runtime rather
than generating a schema ([ADR-0018](0018-API-gRPC-Interface-and-Events.md)), so a function name
is a literal that a caller in any JVM language writes out, and plugins register into that same
namespace ([ADR-0003](0003-Plugins.md)). Those callers arrive from a field whose terminology is
predominantly American: MusicXML's element is `<staff>`, SMuFL's glyphs are `staff5Lines` and
`staffLineThickness`, and both are vocabularies Ooloi already consumes verbatim. An identifier
anglicised against them would oblige every plugin author and every format bridge to translate it
back, permanently.

Nothing is lost at the user's end, because the localisation layer is the mechanism for choosing
the word. *Stave* and *staff* are one concept in two dialects, and the dialect is resolved per
locale exactly as every other word's is. Renaming the identifier would move that choice out of
the layer built to make it.

The same boundary governs spelling. Code keeps the spelling of what it calls —
`-fx-background-color`, `-color-fg-default`, `javafx.scene.control.Dialog` — rather than being
anglicised into names that no longer match their targets.

**Data is not an identifier.** English text stored as data rather than as a name — the bundled
instrument names of [ADR-0045 §Instrument Names and Language](0045-Instrument-Library.md#instrument-names-and-language)
— is read by users, so it follows the source language.

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

1. **Bundled locales**: All Ooloi-provided locales ship inside the application JAR, version-matched. Canonical filename format is underscore (`en_GB.po`, `sv_SE.po`). On first run, bundled files are copied to the platform directory so users can customise them.
2. **External directory**: User-managed locales and overrides, platform-specific location via `get-platform-directory`. Both underscore and dash filenames are accepted here; all are normalised to dash-form keys (`:en-GB`, `:sv-SE`) internally.

```clojure
;; External locale directory
(get-platform-directory "Ooloi" "i18n")
;; Windows: %APPDATA%/Ooloi/i18n
;; Unix/macOS: ~/.ooloi/i18n
```

```
# Bundled in JAR (read-only, underscore filenames)
resources/i18n/
  en_GB.po        ; canonical source — always present
  sv_SE.po        ; Swedish (example)
  de_DE.po        ; German (example)
  ...             ; all Ooloi-provided locales

# External (user-managed, platform-specific)
# Windows: %APPDATA%/Ooloi/i18n/
# Unix/macOS: ~/.ooloi/i18n/
  en_GB.po        ; user override of bundled canonical (presence prevents sync overwrite)
  sv_SE.po        ; user override or community locale
  my_locale.po    ; any additional locale
```

**Canonical locale:** The canonical source language is UK English (`en-GB`), not generic "en". This is the baseline for all translations and the fallback when a key is missing in other locales. American English (`en-US`) is a separate translation like any other, differing in spelling (colour/color, localisation/localization), punctuation conventions, and occasional terminology.

**Rationale:**

- Canonical locale always present—no missing-file failure mode
- New translations do not require a new Ooloi release
- Community-contributed locales can be distributed independently
- Translators can test updates without rebuild cycles
- The localisation layer survives forward UI evolution
- External `en_GB.po` can override bundled version for testing or customisation

### Currently Bundled Locales

The set of bundled locales is auto-discovered at runtime from `shared/resources/i18n/` and is not enumerated in code. The current bundled set is:

| Code | Language | Native name | Status |
|---|---|---|---|
| `en-GB` | English (United Kingdom) | English | Canonical source — human-written |
| `cs-CZ` | Czech | Čeština | AI-translated |
| `da-DK` | Danish | Dansk | AI-translated |
| `de-DE` | German | Deutsch | AI-translated |
| `el-GR` | Greek | Ελληνικά | AI-translated |
| `en-US` | English (United States) | English | AI-translated |
| `es-ES` | Spanish | Español | AI-translated |
| `fi-FI` | Finnish | Suomi | AI-translated |
| `fr-FR` | French | Français | AI-translated |
| `hu-HU` | Hungarian | Magyar | AI-translated |
| `is-IS` | Icelandic | Íslenska | AI-translated |
| `it-IT` | Italian | Italiano | AI-translated |
| `ja-JP` | Japanese | 日本語 | AI-translated |
| `ko-KR` | Korean | 한국어 | AI-translated |
| `nb-NO` | Norwegian Bokmål | Norsk bokmål | AI-translated |
| `nl-NL` | Dutch | Nederlands | AI-translated |
| `pl-PL` | Polish | Polski | AI-translated |
| `pt-BR` | Portuguese (Brazil) | Português (Brasil) | AI-translated |
| `pt-PT` | Portuguese (Portugal) | Português (Portugal) | AI-translated |
| `sv-SE` | Swedish | Svenska | AI-translated |
| `uk-UA` | Ukrainian | Українська | AI-translated |
| `zh-CN` | Chinese (Simplified) | 简体中文 | AI-translated |

**Translation quality caveat.** UK English is the sole default and the only human-written locale. All other locales — including American English — are AI-generated translations and require review by native speakers before they should be considered authoritative. Translation quality varies by language; in particular, languages with rich morphology (e.g. Icelandic, Hungarian, Finnish) or non-Latin scripts may need closer revision than languages closer to the source. Localisation is a natural community-driven contribution area — corrections to existing locales and submissions of new locales are welcome.

The list above is a snapshot; the source of truth is the bundled directory. New locales added there are picked up automatically at startup.

### Runtime Loading and Caching

**Bundled locales:**

All Ooloi-provided locales ship in the JAR. At application startup (`init-locales!`) they are discovered via Java NIO `FileSystems/newFileSystem`, which handles both `file:` (development) and `jar:` (packaged) URI schemes uniformly — no manifest or hardcoded list required. In development mode, `en_GB.po` is lazy-loaded on the first `tr` call.

**Startup load sequence (`init-locales!`):**

1. Discover all `.po` files in the bundled `resources/i18n/` classpath directory via NIO scan
2. Load each bundled locale into `catalogs`
3. Sync: for each bundled file, copy it to the platform directory if the file is absent — never overwrite an existing file (presence alone protects user modifications). The copy is **atomic**, staged in a sibling temp file and moved into place, which matters more here than for a file that is overwritten: because presence alone decides whether to copy, a partial file would satisfy that test on every subsequent start and never be copied again, leaving the locale truncated with nothing able to repair it. Staged, the destination is complete or absent, and absent is the state the check knows how to act on
4. Scan the platform-specific directory for external `.po` files
5. Load each external locale (platform overrides bundled for the same key):
   - If parsing succeeds: add to `catalogs` (overwriting any bundled entry for that key)
   - If malformed: log error, skip locale, continue (falls back to `:en-GB` at runtime)
6. Keep all successfully parsed catalogs in memory for instant locale switching

**Return value:**

```clojure
{:loaded        #{:en-GB :sv-SE ...}   ; all successfully loaded locale keys
 :bundled       #{:en-GB :sv-SE ...}   ; locales discovered from JAR resources
 :external      #{:sv-SE ...}          ; locales loaded from platform directory
 :notifications [{:type :warning       ; deferred UI notifications (e.g. duplicate files)
                  :message "..."}]}
```

Parsing is fast enough for startup, and keeping all catalogs in memory enables instant locale switching.

**Error handling:** Malformed PO files return an error map from the parser. The loader catches this during step 3, logs the error, and excludes that locale from the available catalog. At runtime, attempts to use the failed locale fall back to `en-GB`.

**Locale selection:**

Locale loading and selection are separate:
- `init-locales!` — loads all available catalogs (bundled + external)
- `set-locale!` — selects which locale to use from loaded catalogs

```clojure
(init-locales!)                        ; Load all available
(set-locale! (platform/get-os-locale)) ; Auto-select based on OS
```

**Locale switching:** Switching locales is instant (all catalogs pre-loaded). UI updates are driven by the UI Manager. Windows register their private state atoms via `ui-manager/register-renderer!` in their `:window-opened` lifecycle handler and do not subscribe to locale changes directly — a window declares what it is, and the UI Manager iterates the registry generically.

**Locale change cascade.** The UI Manager's `:app-settings` subscription receives `:setting-changed {:key :ui/locale}`, calls `tr/set-locale!` synchronously on the event-bus thread — switching the active catalog, so that all subsequent `tr` calls return new-locale text — and then posts a single `fx/run-later!` to the JavaFX Application Thread. That one runnable does two things, in order:

1. **Menu text.** `refresh-dynamic-items!` re-resolves every menu bar from the `::menu-name-key` and `::static-text-key` properties stored on the JavaFX menu nodes at construction time — the macOS global bar, and on Windows and Linux each piece window's own bar. The macOS application menu's About, Settings and Quit items are re-resolved alongside them.

2. **One pass over the window registry.** For each registered window, in order:
   - **Title** — when the registry entry holds a `:title-key`, the Stage title is recomposed through `tr`, with any title decorators re-prepended. An entry whose `:title-key` has been cleared — a window titled from data, such as a piece window showing a piece's name — is deliberately left alone.
   - **Content** — `(swap! *state identity)` causes the window's cljfx renderer to re-evaluate its spec function. The `swap!` changes nothing in the atom; the re-evaluation is the point. Every `tr` call inside the spec runs again, and cljfx diffs the result against the current scene graph and patches only the nodes that changed.
   - **Re-fit** — a non-resizable window is sized to its content, so it is then `sizeToScene`d. Translated strings differ in length, and a stage still sized for the previous locale would clip the taller or wider content and ellipsise its labels. The re-fit is deferred a pulse so the re-render lands first; a resizable window keeps whatever size the user chose.

Title, content and re-fit are one pass rather than three, so a surface added to the cascade is added in one place.

**Locale reactivity is total, and enforced rather than remembered.** Every user-visible string in Ooloi follows a locale change: window content through the renderer, window titles through the registry, menu text through the stored menu-node properties, and formatter-resolved labels through the `:locale` cache-buster. The surfaces that do *not* change are the ones that cannot meaningfully do so, each for a stated reason, and they are enumerated in the Frontend Architecture Guide rather than left to be rediscovered.

What makes this durable is that the last mechanism is not a convention. A formatter that resolves a caller-supplied `:text-key` asserts that a `:locale` accompanies it, and asserts that a `:raw-text` does not — see [ADR-0042](0042-UI-Specification-Format.md) §Custom cljfx Component Functions. A call site that would have produced a label frozen in the previous language does not render incorrectly; it does not run. The consequence for anyone building a new surface is that locale reactivity is no longer something to remember to arrange: it is the only thing the formatters permit.

Content nested inside custom component functions requires an additional mechanism — cljfx skips re-invoking a component whose props are unchanged, and a `:text-key` keyword does not change with the locale. See the `:locale` cache-buster in [ADR-0042](0042-UI-Specification-Format.md) and its usage guidance in the Frontend Architecture Guide.

**Renderer spec is the locale-reactivity boundary.** Only content the renderer re-evaluates on `swap!` gets updated. Content built by one-time `cljfx/create-component` + `cljfx/instance` at window creation time without a mounted renderer is constructed once and never revisited. See [ADR-0042](0042-UI-Specification-Format.md) for the invariant.

**In-memory format:**

```clojure
{:plural-rule "(n != 1)"
 :messages {:menu.file.open "Öppna…"
            :dialog.export.warning "Export kommer att skriva över %{filename}."
            :file.count ["fil" "filer"]}}  ; plural forms as vector
```

**Plural rule resolution:**

The `Plural-Forms` expression from the PO header is resolved to a pre-defined Clojure function via lookup table, not runtime compilation. The deployed application uses `jlink` for a minimal JVM runtime without the Clojure compiler, so `eval` is unavailable.

Gettext plural patterns are well-documented and finite—approximately 15-20 patterns cover all real-world languages (documented in [GNU gettext manual section 11.2.6](https://www.gnu.org/software/gettext/manual/html_node/Plural-forms.html) and the [CLDR plural rules](https://cldr.unicode.org/index/cldr-spec/plural-rules)):

```clojure
(def plural-rules
  {"(n != 1)"       (fn [n] (if (= n 1) 0 1))       ; English, German, Swedish, etc.
   "(n > 1)"        (fn [n] (if (> n 1) 1 0))       ; French, Brazilian Portuguese
   "(n%10==1 && n%100!=11 ? 0 : ...)" ...           ; Russian (complex)
   })
```

If an unknown pattern appears (unlikely), the system falls back to treating all forms as singular. This is a visible degradation, not a crash.

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

### Every Key Is Declared

The invariant is absolute, and it is about **declaration**, not about construction: **every key that can reach `tr` must appear as a literal keyword in the source** — in a `tr` call, or in a `tr-declare` map. That is what makes build-time verification possible, because the scanner's union of extracted and declared keys is checked against `en_GB.po`.

**How a key is assembled at the call site is beside the point.** Constructing one at runtime is ordinary and widespread — from a setting's namespace and name, from a clef or an interval, from a menu's name — and it is legitimate exactly when every value it can produce is declared:

```clojure
;; Legitimate: the call site computes, and the declaration enumerates.
(tr-declare {:clef.treble "Treble" :clef.bass "Bass" :clef.alto "Alto"})

(defn clef-label [clef]
  (tr (keyword (str "clef." (name clef)))))     ; scanner skips this; the declaration covers it
```

```clojure
;; A violation: nothing declares what this can produce, so no catalogue can contain it
;; and no translator can ever be offered it.
(tr (keyword (str "menu." section "." action)))
```

The two examples are the same construction. What separates them is whether a `tr-declare` enumerates the results — which is why the rule cannot be stated as a prohibition on how the keyword is built.

The failure mode this prevents is a key that exists only at runtime: absent from every catalogue, so it renders as its own name, and invisible to the build because nothing named it. A concatenating call site whose results are declared has none of those properties.

### Two Extraction Mechanisms

Keys appear as literals through two mechanisms. Both are extracted by the build-time scanner; their union must match the `.po` file exactly.

**`tr` calls** — the scanner extracts literal keyword arguments:

```clojure
(tr :menu.file.open)
(tr :dialog.export.warning {:filename name})
```

When `tr` receives a non-literal argument (a variable, a function call), the scanner silently skips it. This is not a violation — data-driven infrastructure legitimately passes keywords through variables. The key itself was a literal at its declaration site.

**`tr-declare` maps** — declares keys with their canonical English text:

```clojure
(tr-declare {:menu.file.open "Open…"
             :menu.file.save "Save"
             :menu.file.close "Close"})
```

`tr-declare` takes a literal map of keyword → default English string. It is a no-op at runtime. The build-time scanner extracts the keys for `.po` file verification and uses the default strings as `msgid` when auto-adding missing entries. Non-literal arguments to `tr-declare` are scanner errors.

### Declaration Use Cases

`tr-declare` serves two purposes:

1. **Indirect keys** — keys that reach `tr` through data structures (dialog specs, command descriptors, notification specs). The key is a literal at the construction site but a variable at the `tr` call site. Declaration makes these keys visible to the scanner.

2. **Key documentation** — declaring canonical English text alongside direct `tr` usage. A `tr-declare` at the top of a file provides a manifest of all translation keys that file uses, with their English text visible without opening the `.po` file. Files using this pattern have both `tr-declare` and `tr` calls for the same keys; the scanner deduplicates.

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

### Interpolated Names

A parameter whose value is **itself a translated string** is a harder case than a filename or a count, because two catalogue entries meet inside one rendered sentence and the grammar of the join belongs to the reader's language. The Edit menu is the standing example: `menu.edit.undo-item` is `"Undo %{name}"`, and `%{name}` arrives already translated, as `"Reorder Instruments"` or `"Hinzufügen des Musikers"`.

**Nothing that belongs to the interpolated string is hardcoded around the placeholder.** A verb glued into the wrapper for one caller's benefit is inherited by every other caller, and reaches the ones that already carry their own.

**`%{name}` sits where the target language's grammar puts it.** Not where English puts it. It follows the verb in English, French, Swedish and Chinese; precedes it in German, Dutch, Hungarian, Japanese and Korean; and is wrapped by it in Finnish, whose particle verb splits around its object — `Tee %{name} uudelleen`. A translator moves the placeholder freely, which is the whole reason named parameters exist.

**The name is attached grammatically, never with punctuation.** No colon, and no quotation marks in English. A language quotes the interpolation where its own grammar requires it: Hungarian's possessive would otherwise compete with the wrapper's — `„%{name}” visszavonása` — and the quotation marks make the inner phrase a citation so the two stop clashing.

**An interpolated string that appears in more than one grammatical slot must work in every one of them, uninflected.** An undo description is read after the verb in the Edit menu and alone as a row in an edit-history window. It is therefore a nominative citation phrase, and is never inflected into the case a wrapping verb governs: doing so would make one slot right and the other wrong, and no single catalogue entry could serve both. Where a language's wrapper is a verb, the description is a noun phrase rather than a bare infinitive, so that the two do not stack — German names the action, `Hinzufügen des Musikers`, rather than repeating a command, `Musiker hinzufügen`.

Case *inside* an interpolated string is a separate matter and unaffected: its own verb governs its own object, so Polish `Dodaj muzyka` keeps its accusative.

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

**Invariant: plural shape is per-locale, not uniform.**

A given `msgctxt` may be declared with `msgid_plural` in some locale PO files and without it in others. The decision belongs to the locale's grammar, not to the key schema. The runtime `tr` API tolerates both shapes transparently:

- A locale whose catalog entry is a single string → straight `%{name}` substitution.
- A locale whose catalog entry is a vector of forms → plural-rule selection by `:n`, then `%{name}` substitution.

This invariant overrides the strict gettext convention that `msgid_plural` is part of the key schema and must be uniform across locales. In Ooloi the source language (English) decides nothing about whether other languages need plural inflection for a given message — the grammar of each target language decides for itself. Concrete consequences:

- If the **English source** (en_GB) has no singular/plural distinction for a string — e.g. "%{n} connected Ooloi." where "Ooloi" is invariant and "connected" is past-participle invariant — en_GB carries `msgid` + `msgstr` only. No `msgid_plural`. No `msgstr[0]/[1]` duplication.
- If a **target locale** (e.g. Swedish "1 ansluten Ooloi" vs "2 anslutna Ooloi") does grammatically distinguish, that locale's PO file carries `msgid` + `msgid_plural` + `msgstr[0..N-1]` with one form per the locale's declared `nplurals`. Every form must be grammatically correct on its own.
- If a locale (e.g. Dutch "%{n} verbonden Ooloi", invariant in numbered-noun construction; Hungarian "%{n} csatlakoztatott Ooloi", singular noun always with numerals; CJK languages, `nplurals=1`) doesn't grammatically distinguish, that locale also carries `msgid` + `msgstr` only. No padding with identical plural forms.

The CANONICAL files in this repository ship out of the box for every locale — grammatically correct for every value of `n`, in every language we support. A human translator may refine wording. A human translator MUST NOT have to add plural-form structure to make a locale grammatical. If a locale's PO file is missing plural shape, that means the locale's grammar genuinely doesn't need it; if a translator believes otherwise, they fix the wording, not the structure.

## Plugin Localisation

Plugins ship their own PO files in a dedicated directory within the plugin distribution:

```
my-plugin/
  plugin.edn
  i18n/
    en_GB.po
    sv_SE.po
```

Plugin translation catalogs are namespaced by plugin identifier. No collision with core keys or other plugins is possible:

```clojure
;; Core translation
(tr :menu.file.open)

;; Plugin translation — the same function, a namespaced key
(tr :plugin.my-plugin/palette.title)
```

**The same `tr`, never a sibling of it.** A `plugin-tr` or any other locale-aware second entry point is excluded by [Invariant 3](#3-all-localisation-through-a-single-api): one function, one mechanism, no alternatives. A plugin's strings are user-visible text like any other, and there is no second path by which user-visible text is produced.

**Namespacing is the requirement; the namespaced keyword is the authoring surface.** Whether the key is stored namespaced or flattened into a dotted form inside the catalogue is an internal matter that does not reach the plugin author, who writes the namespaced keyword either way. That separation is what lets the catalogues keep the dotted convention core uses while collision isolation remains structural rather than a matter of naming discipline — the requirement being that plugin catalogs are isolated namespaces, not merged into a global soup.

Plugins use the same PO format, same tooling, same workflow. A translator working on a plugin sees exactly the same file structure as core localisation.

**Loading timing:** Plugin translation catalogs load at plugin load time, before any plugin UI renders. This ensures translations are available when the plugin's first UI elements are created.

## Build-Time Verification

### Two Operational Modes

Build-time verification operates in two modes depending on context:

**Normal Mode** (development, `lein i18n`):
- Extracts all `tr` keys and `tr-declare` keys from source
- Auto-adds missing keys to `en_GB.po`:
  - Keys with `tr-declare` defaults: fully populated (msgid and msgstr from default text)
  - Keys from bare `tr` calls only: `[TODO: Translation needed]` placeholder as msgstr
- Warns about TODO entries and orphaned keys
- Tolerant of *incompleteness*, so development can iterate rapidly
- **Fails on defects that are never legitimate**: any plural-integrity violation (see below), a duplicate key, a non-literal `tr-declare` argument, or a catalogue that parses to nothing

**Strict Mode** (CI/build, `lein i18n :strict true`):
- Extracts all `tr` keys from source
- Reports missing keys as errors (does not modify file)
- Fails build if TODO entries exist (incomplete translations)
- Fails build if catalog is incomplete
- Everything normal mode fails on, it fails on too
- Hard gate preventing incomplete artifacts

The distinction between the modes is **completeness, not correctness**. A missing key or a TODO placeholder is a legitimate work-in-progress state, so only strict mode rejects it. A plural entry that disagrees with its own catalogue's rule is never a legitimate state at any point in development, so both modes reject it.

**Development workflow:**
Developers run `lein i18n` during active development as new UI strings are added. Missing keys with `tr-declare` defaults are auto-populated with canonical English text; keys from bare `tr` calls get TODO placeholders. The build pipeline then verifies completeness—running `lein i18n :strict true` to ensure all TODO placeholders have been replaced with actual translations before creating artifacts.

### Canonical Completeness (Hard Gate)

At build time (strict mode):

1. Extract all `tr` keys and `tr-declare` keys from frontend source
2. Parse `en_GB.po` and extract all `msgctxt` values
3. Assert: the union of extracted + declared keys matches `en_GB.po` exactly
4. Assert: no TODO placeholders remain in catalog

Build fails if the canonical UK English catalog is incomplete or contains TODO entries. This prevents shipping UI elements without complete English translations.

Implementation: Parse source with Clojure reader, extract literal keywords from `tr` calls and `tr-declare` maps, compare union against PO contents. Deterministic and reliable.

**Build Integration:**
- Verification runs automatically during `lein build`
- Always uses strict mode in build pipeline
- Positioned before uberjar creation (fails fast)
- Colored terminal output for clear visibility

### Plural Integrity (Hard Gate, Both Modes)

Completeness is checked against `en_GB.po` alone. Plural integrity is checked against **every catalogue in the i18n directory**, each held to its own declared `Plural-Forms` rule. Every locale is examined independently because plural shape is per-locale (see [Plural Forms](#plural-forms)): a key may carry plural forms in one catalogue and a single string in another, so a check driven from the English source would miss a defect introduced on a key whose English is simple.

Four checks, all implemented in `ooloi.shared.i18n.verify`:

| Check | Function | Catches | Failure mode if unchecked |
|---|---|---|---|
| Plural arity | `detect-plural-arity-mismatches` | an entry with a different number of `msgstr[N]` forms than its own header declares | crash, or a silently wrong form |
| Rule resolution | `detect-unknown-plural-rules` | a `Plural-Forms` rule with no implementation in `tr/plural-rules` | that locale renders form 0 for **every** count, silently |
| Rule range | `detect-plural-range-violations` | a header disagreeing with itself — its rule selecting a form index its own `nplurals` forbids | over-indexes even a correctly sized entry |
| Count dropped | `detect-count-dropping-forms` | a form the rule selects for more than one count that states no count | a selection of 21 asks "Delete this stave?"; the correct forms sit unreachable |

None of the four is visible to the PO parser or to the loader — each is valid PO syntax — so they surface only in use, as a wrong form or as no form at all. That is why they are build-time gates rather than runtime concerns, and why they fail in both modes.

The count-dropped check names no locales at all. Its criterion is a property of each form's **domain** — the set of counts the catalogue's own rule sends to it:

> A form selected for exactly one count may omit the number. Every form with a wider domain must state it.

That single property covers every case. `(n != 1)` gives English form 0 the domain {1} and nothing else, which is why the English singular legitimately reads "Delete this stave?". Ukrainian and Icelandic give form 0 the domain {1, 21, 31, …}, so it must state the count. A single-form locale gives its only form *every* count, making it the most obvious violation of the lot rather than a special case. And a locale added later with a Slavic-shaped rule is covered without being named.

The domain runs from n=1 rather than n=0. No rule Ooloi ships gives 0 a form of its own, and including it would hand the `n > 1` locales (`fr_FR`, `pt_BR`) a form-0 domain of {0, 1} — flagged for a count it lacks only at zero, which is not a count any confirmation or undo label is raised for.

**What this check is not.** The presence of a count placeholder is a lint, not a proof. A form may carry `%{n}` and still be wrong in agreement — Ukrainian once carried the genitive plural, the five-and-above form, in the two-to-four slot, so three staves asked for *'ці 3 нотних станів'*. The placeholder was present throughout. Grammatical correctness within a form remains a human judgement; see [Invariants](#invariants) item 9 for the standard it is held to.

### Non-English Coverage

Non-English catalogues are gated on **structure always, and on completeness only where they ship as a set**:

- An **external** catalogue may omit any key; it falls back to UK English at runtime
- Coverage percentage can be reported
- Plural-integrity violations block the build, in both modes — see above
- The **bundled** catalogues must hold the same key set as `en_GB.po` to be packaged

Forward compatibility is what the first rule protects, and it is a **runtime** property of **external** catalogues: an old `sv.po` in the platform directory continues to work when new UI strings are added in a release, and a translator updates at their own pace afterwards. Partial translations degrade gracefully rather than failing. A partial translation is a legitimate state; a structurally broken one is not.

**The bundled catalogues are a different case, because they ship together.** They are not a set of independent translations arriving at their own speed; they are one artefact, built in one pass, and a key added to `en_GB` is translated into the rest before that artefact is made. [Canonical Completeness](#canonical-completeness-hard-gate) gates `en_GB.po` against the source, and `detect-locale-key-gaps` gates the rest against `en_GB.po` — an error under strict mode, which the build runs, and a warning naming the divergent locales in normal mode, so that work in progress is visible without being blocked.

The failure this closes is quiet rather than loud, which is why discipline alone did not hold it. A key that was **never** translated falls back to English and reads as a string nobody has reached yet — the soft gate working as intended. A key that **was** translated and whose name then changed in code and `en_GB` alone falls back the same way, and is indistinguishable from the first: German menus silently render English, with nothing to mark them. No test fails, because every test pins `:en-GB`; no marker appears, because the fallback is by design. The gate is on the bundled set specifically because that is the only place the two cases can be told apart — by construction, they are complete or they are wrong.

This gates packaging and nothing else. It never runs against the platform directory, never sees an external catalogue, and does not touch the runtime fallback chain in [Forward Compatibility](#forward-compatibility).

Coverage is a report rather than a gate. `calculate-coverage` computes a locale's percentage against `en_GB.po`. It is a library function and the verification task does not call it: a coverage figure informs a translation effort rather than deciding a build.

### What Verification Cannot See

Verification works by extracting `tr` keys from source and comparing them against the catalogues. Everything it can check follows from a key existing. **A string that never became a key is therefore invisible to it**, and that is precisely the shape of the worst violation of this architecture:

```clojure
;; Passes every build check. There is no key, so there is nothing to check.
(show-error-notification! mgr (str "An unexpected error occurred: " detail))
```

No key is extracted, no catalogue entry is missing, no plural rule is broken. The build is green and the string is untranslatable in all twenty-two locales. The same hole covers an inline literal and a sentence assembled from fragments — Invariants 2 and 5 — which is why those invariants cannot be enforced by the build task and must be enforced by a test.

**The enforcing observation is that a translated string changes when the locale changes.** Produce the same output twice, under two different locales, and compare:

- concatenation and inline literals produce **identical** text both times, because no catalogue was consulted
- a genuine `tr` call produces **different** text, because it was

A single-locale assertion cannot distinguish the two: a hardcoded English prefix satisfies it exactly as well as a correct implementation does.

Two consequences worth stating, because both are easy to mistake for test defects:

- **The comparison also fails when the catalogue sweep was skipped.** Under Forward Compatibility below, a key missing from a locale returns the UK English text — so an untranslated key produces identical output in both locales and is indistinguishable from concatenation. This is correct behaviour: the test asserts that the string is *translated*, and a key present in one catalogue only is not.
- **The second locale must be asserted, not assumed.** `set-locale!` falls back to `:en-GB` when the requested locale is unavailable, and returns the locale actually selected. An unasserted fallback silently compares English against English and fails for a reason unrelated to the subject.

`guides/INTEGRANT_COMPONENTS.md` §Proving a string is translated, not concatenated carries the test pattern and its worked examples.

### Orphaned Keys

Keys in catalog but not in source are reported as warnings, never errors. Rationale:
- Features may be temporarily removed
- Keys may return in future iterations
- Maintaining historical translations is valuable
- Translators prefer stable keys over churn

## Forward Compatibility

**Invariant:** A new Ooloi version adding new UI strings must not break existing translations.

Mechanism:

- Missing key in locale → return UK English text
- Missing key in UK English → return conspicuous placeholder (`[MISSING: key.name]`), never crash
- Malformed PO file → fall back to en-GB for entire locale
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

**Decision:** Custom source code scanner with two extraction mechanisms.

**Rationale:**
- Two patterns: `(tr :keyword)` for direct usage, `(tr-declare {...})` for indirect keys
- Non-literal `tr` arguments silently skipped (tolerance for data-driven infrastructure)
- Non-literal `tr-declare` arguments are errors (map must be a literal)
- Union of extracted + declared keys verified against `en_GB.po`
- Declared defaults used as both `msgid` and `msgstr` when auto-adding missing entries
- Build-time completeness check: all keys exist in `en_GB.po`

### Locale Loading: Direct Parsing

**Decision:** Parse PO files directly at runtime, no caching layer.

**Rationale:**
- PO parsing (via potentilla) is fast enough for startup
- Keeping all catalogs in memory eliminates cache invalidation complexity
- Simpler architecture: fewer moving parts, no timestamp checking
- External locales loaded once at startup for instant switching

**In-memory structure:**
```clojure
{:plural-rule "(n != 1)"
 :messages {:menu.file.open "Open…"
            :dialog.export.warning "Export will overwrite %{filename}."
            :file.count ["file" "files"]}}  ; plural forms as vector
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

**Decision:** Custom build task with dual-mode operation.

**Implementation** (`shared/src/main/clojure/ooloi/shared/i18n/verify.clj`):
- Parse source with Clojure reader (not regex)
- Extract literal keywords from `tr` calls and `tr-declare` maps; skip a non-literal `tr` argument, and reject a non-literal `tr-declare` map
- Compare against `en_GB.po` msgctxt values
- Auto-add mode: append missing keys (tr-declare defaults as msgstr, TODO for bare tr keys)
- Strict mode: fail on missing keys or TODO entries
- Plural integrity across every catalogue in the directory, failing in both modes (see [Plural Integrity](#plural-integrity-hard-gate-both-modes))
- Colored terminal output (ANSI codes)
- Multiple source directory support
- Exclusion patterns with wildcard support (`:exclude ["**/i18n/tr.clj"]` default)
- Configurable via command-line arguments

**Rationale:**
- Reader-based parsing: more reliable than regex, handles all Clojure syntax
- Two modes: rapid development vs. build quality gates
- Auto-add reduces friction during active development
- Strict mode enforces completeness in CI/production builds
- Colored output improves developer experience
- Integrated into build pipeline prevents incomplete artifacts
- Exclusion patterns prevent false positives from implementation files (tr.clj itself uses parameter names in delegation)

### Dependency Summary

**Added to shared/project.clj:**
```clojure
[com.soberlemur/potentilla "0.0.3"]  ; PO file parsing (ANTLR-based, Java interop)
```

**Custom implementation:**
- `ooloi.shared.i18n.tr` — `tr` function, `tr-declare` declaration function, named parameters, plural selection, locale management (consumes potentilla output)
- `ooloi.shared.i18n.verify` — Key extraction from `tr` calls and `tr-declare` maps, verification, auto-add mode with defaults, strict mode, colored output

**Why this split:**
- Library for complex parsing (saves ~400-600 lines, handles edge cases with ANTLR)
- Custom for our constraints (named parameters, literal keys, in-memory catalog format)
- Clear ownership: we control the API, rely on library for format complexity
- Testable: mock potentilla output, test our logic independently
- Low risk: production-validated parser, minimal interop cost

## Invariants

These are architectural constraints, not conventions. Violating any of them breaks determinism, translator usability, or both.

Each invariant carries its own heading deliberately. A run of numbered items embeds as a single vector averaged across all of them, and no individual constraint then has enough signal to be retrieved — a defect measured on this very section, where a query paraphrasing invariant 9 almost verbatim returned nothing at all.

### 1. Backend never emits localised strings

Backend outputs are protocol artifacts: stable English for logs, exceptions, and programmatic matching. Localisation happens only in the frontend, so the same input produces the same backend output regardless of locale.

### 2. Frontend contains zero inline user-facing strings

Every user-visible string passes through `tr`. No string literal reaches a label, a menu item, a window title, or a notification without a translation key behind it.

Neither this invariant nor Invariant 5 is reachable by the build task — a literal produces no key to extract, so nothing is missing and nothing fails. Both are held by test instead; see [What Verification Cannot See](#what-verification-cannot-see).

### 3. All localisation through a single API

One function, one mechanism, no alternatives. There is no second path by which user-visible text can be produced — no string concatenation, no formatting helper, no locale-aware sibling of `tr`.

### 4. Named parameters only

Substitution is by name — `%{name}`, `%{n}`, `%{instrument}` — never by position. No `%s`, `%d`, or `%1$s`. A translator may reorder the parameters in a sentence without knowing the order the caller passed them.

### 5. No sentence assembly from fragments

Messages are complete translatable units, never concatenated pieces. A sentence built from parts cannot be reordered, inflected, or agreed by the translator, because no one place holds the whole of it.

### 6. No localisation logic in protocol, persistence, or collaboration paths

These layers are locale-agnostic. A piece saved by one user and opened by another under a different locale is the same piece; nothing about language survives into the file, the wire format, or a collaboration session.

### 7. Every key is declared

Every translation key appears as a literal keyword in source — in a `tr` call or in a `tr-declare` map — because a key that cannot be found by reading the source cannot be extracted, verified, or offered to a translator.

The invariant governs declaration, not construction. A call site may assemble a key at runtime from a setting's name, a clef, an interval or a menu's name, and many do; what it may not do is produce a value that no `tr-declare` enumerates. See [Every Key Is Declared](#every-key-is-declared).

### 8. PO files are the sole translator interface

Translators never edit code, EDN, or internal formats. Everything a translator needs — the message, its context, its plural forms, and the comments explaining what each plural slot covers — lives in the `.po` file itself.

### 9. Every bundled catalogue is grammatically correct for every value of `n`

A shipped locale is correct out of the box, for one item and for twenty-one, without a translator having to supply plural structure to make it so. A human translator may refine wording; a human translator must never have to repair grammar.

This is enforced rather than intended. The four plural-integrity checks — `detect-plural-arity-mismatches`, `detect-unknown-plural-rules`, `detect-plural-range-violations` and `detect-count-dropping-forms` — fail the build in both normal and strict mode, over every catalogue in the i18n directory rather than over `en_GB.po` alone. See [Plural Integrity](#plural-integrity-hard-gate-both-modes).

What enforcement cannot reach is agreement within a form: a plural form may carry its count placeholder and still inflect the noun wrongly. That residue is a human judgement, and this invariant is the standard it is held to.

## Documentation

Translator-facing documentation is the [Translator's Guide](../guides/TRANSLATORS_GUIDE.md). The PO format itself needs no explaining from us — it is documented by the GNU gettext manual and understood by every standard PO editor — but three things are Ooloi's own and a translator cannot deduce them: where the catalogues live and how the startup sync between the bundled set and the user's folder behaves, the musical vocabulary a general translator renders wrongly, and the conventions governing a translated string interpolated into another (§[Interpolated Names](#interpolated-names)). The guide also states plainly that the shipped translations are machine-generated and provisional, which is what invites the correction they need.

That guide is written for a musician rather than an implementer, and carries its own reasoning rather than delegating it here: a translator who follows a link into this document lands among protocol boundaries and does not come back.

Developer-facing explanation lives in the Frontend Architecture Guide — Section 9 for the translation API, the key conventions and the locale-change cascade, and Section 4.4 for the formatters that resolve keys. This ADR is the specification; the guides teach its use.

## Out of Scope

This ADR addresses **string translation only**. Related concerns handled separately:

**Date/time/number formatting:**
- Locale-specific formatting ("3,14" vs "3.14", date order variations, thousand separators)
- Separate concern requiring different infrastructure (not gettext-based)
- Future ADR if needed for user-configurable preferences

**RTL language support (Arabic, Hebrew, Persian, Urdu):**
- Translation infrastructure is direction-agnostic: a catalogue in an RTL language loads and resolves like any other
- Strings should not contain directional markup (Unicode bidi control characters)
- Text rendering handles the bidi algorithm, so the strings themselves display correctly
- **Interface mirroring does not follow from that and is not implemented.** Mirroring a JavaFX scene graph requires setting node orientation, which nothing in Ooloi sets, so menus, dialogs and panels stay laid out left-to-right whatever the locale. An RTL catalogue is therefore usable but the result is not yet right, and the gap is a UI concern rather than a localisation one

**Currency/unit conversion:**
- "5 miles" vs "8 kilometers" requires domain logic, not translation
- Out of scope for string localisation
- Application-specific logic if needed

**Merging a newer bundled catalogue into a user's own:**
- An external catalogue replaces the bundled one for its locale rather than merging with it, so a user's file that predates a version carries none of the keys that version added, and those strings fall back to English until the user takes them in by hand
- Merging them per key is a candidate change and is deliberately left open here rather than decided
- The precedent sits in the same product: the Instrument Library merges user data over the bundled set by `:id`, with an explicit `:excluded` set for deliberate removals, so local adjustment survives while the bundle keeps improving underneath it. The same shape applies to a publisher who wants house wording in the interface
- Two properties would follow directly — keys added by a later version appear, and a string cannot be lost by being absent from the user's file. A third does not: distinguishing a translation the user *edited* from one they merely still hold requires retaining the previous bundled version and comparing three ways, which merge-on-load alone does not provide
- Until this is settled, the [Translator's Guide](../guides/TRANSLATORS_GUIDE.md) tells translators to compare their catalogue against the shipped one after upgrading

## Consequences

**Positive:**

1. **Clean adoption point** — the policy was established before the strings existed, so it is an invariant the code was written under rather than a cleanup applied to code that had already broken it
2. **Translator-native workflow** — PO files and standard tooling, no Ooloi-specific learning curve
3. **Guaranteed baseline** — Canonical UK English bundled in JAR, no installation failure modes
4. **Minimal startup overhead** — PO parsing is fast; all catalogs loaded once at startup
5. **Independent release cycles** — Non-English translations ship separately from application releases
6. **Forward compatibility** — New versions don't break existing translations
7. **Deterministic extraction** — Literal-only keys enable reliable build-time verification
8. **Plugin parity** — Plugins use identical localisation infrastructure
9. **Grammatical flexibility** — Named parameters allow reordering across languages
10. **Correct plural handling** — Gettext mechanism handles all language complexity
11. **No migration required** — Design is complete from the start

**Neutral:**

1. **PO parsing at startup** — One-time cost; all catalogs kept in memory for instant locale switching
2. **External directory** — User creates platform-specific `i18n/` directory for additional locales; not required for basic operation

**Negative:**

1. **Literal-only constraint** — Some dynamic UI patterns require workarounds (acceptable trade-off for extraction reliability)

## Related ADRs

- [ADR-0042: UI Specification Format](0042-UI-Specification-Format.md) - Uses translation keys for all UI strings as specified in this ADR
- [ADR-0018: API, gRPC Interface and Events](0018-API-gRPC-Interface-and-Events.md) - Runtime name resolution makes API function names an interoperability surface, which is why [Source Language Scope](#source-language-scope-strings-not-identifiers) stops at the string layer
- [ADR-0003: Plugins](0003-Plugins.md) - Plugins in any JVM language address the API by name
- [ADR-0045: Instrument Library](0045-Instrument-Library.md) - Bundled instrument names are data read by users, so they follow the source language

## References

- GNU gettext manual: https://www.gnu.org/software/gettext/manual/
- PO file format specification: https://www.gnu.org/software/gettext/manual/html_node/PO-Files.html
- Poedit: https://poedit.net/
- Lokalize: https://apps.kde.org/lokalize/
- Potentilla: https://github.com/soberlemur/potentilla (ANTLR-based PO parser)
- Autopo: https://github.com/soberlemur/autopo/ (production validation of potentilla)
- Maven Central: https://central.sonatype.com/artifact/com.soberlemur/potentilla
- `ooloi.shared.platform` namespace for cross-platform directory conventions
