# 🌐 Translator's Guide

*For the musician who speaks the language.*

## Table of Contents

- [Quick Start](#quick-start)
- [Who This Is For](#who-this-is-for)
- [The Two Surfaces](#the-two-surfaces)
- [Why Your Language May Read Oddly](#why-your-language-may-read-oddly)
- [What Ooloi Asks of a Translation](#what-ooloi-asks-of-a-translation)
- [How Ooloi Chooses Its Words](#how-ooloi-chooses-its-words)
- [Where the Files Are](#where-the-files-are)
- [Anatomy of an Entry](#anatomy-of-an-entry)
- [Plurals](#plurals)
- [Names Inside Sentences](#names-inside-sentences)
- [Musical Terminology](#musical-terminology)
- [Choosing an Editor](#choosing-an-editor)
- [Adding a Language Nobody Has Done Yet](#adding-a-language-nobody-has-done-yet)
- [Sending Your Work Back](#sending-your-work-back)
- [Cross-References](#cross-references)

## Quick Start

To fix a phrase that grates, without reading the rest of this guide:

1. Open your language's catalogue — `~/.ooloi/i18n/sv_SE.po` on macOS and Linux, `%APPDATA%\Ooloi\i18n\sv_SE.po` on Windows.
2. **Search the file for the wrong wording itself.** That is how you find the entry; you do not need to know its name or guess at the English.
3. Edit the `msgstr` line, and only that line. Leave `msgctxt` and `msgid` exactly as they are.
4. Keep every `%{placeholder}` — you may move it to wherever your language puts it, but it must still be there.
5. Save, restart Ooloi, and look at it.

That is the whole cycle. Everything below explains the parts you will meet when you go further.

## Who This Is For

Ooloi's interface ships in twenty-two locales, covering twenty languages — British and American English are two locales of one language, as are European and Brazilian Portuguese. This guide is for the person who makes one of them good.

You need three things: the language, a musician's ear, and a text editor. You do not need to program, and nothing in this guide asks you to.

The musician's ear is the part that cannot be delegated. Software translation is mostly a matter of register and consistency, but notation software is a matter of vocabulary that only musicians hold. Whether a *Notenzeile* or a *Notensystem* is the thing a violinist reads from, whether *pauta* or *pentagrama* belongs in a Portuguese menu, whether your language's word for *voice* means a singer or a strand of polyphony — these are questions a general translator gets wrong and a musician gets right without thinking.

## The Two Surfaces

This guide is about **interface localisation**: menus, dialogs, buttons, settings, notifications — the words Ooloi uses to talk to you. Those live in one catalogue per locale, and correcting them is a contribution to everyone who uses Ooloi in your language.

Instrument names are a **separate surface**, and the difference is not merely one of file format.

|  | Interface strings | Instrument names |
|---|---|---|
| What they are | Menus, dialogs, settings, messages | The names of instruments in a score |
| How many languages | 22 locales | 4 score languages: English, German, French, Italian |
| Chosen by | Your interface language | The score's language, set independently |
| Where you edit them | The catalogue file | The Instrument Library window, inside Ooloi |
| Who gets your work | Everyone, once you send it back | The Ooloi holding the library — normally your own |

**Interface language and score language are independent, deliberately.** A Swedish conductor can have a Swedish interface and Italian instrument names on the page, which is usually exactly right: the interface is a tool one person uses, while the score is a document other musicians read.

The instrument library carries four score languages — 287 instruments in each of English, German, French and Italian, plus 28 whose names are the same everywhere. That is not a gap waiting for volunteers. It is the set of languages orchestral scores conventionally use.

**The library belongs to the Ooloi that holds it, which is why you edit it in the window rather than in a file.** On an ordinary desktop that is your own copy, and your adjustments are yours: they layer over the names Ooloi ships with, while the ones you have not touched keep improving as Ooloi does. Nobody is waiting for you to send them in.

Connected to someone else's Ooloi — a publisher's server, say — the library is *theirs*. It is shared by everyone who connects, and only the operator can change it, which is exactly how a publisher keeps their house naming across everyone working on their scores.

Two consequences worth knowing before you start renaming things:

- **An instrument added to a piece is copied into it.** Renaming an instrument in the library changes what future scores get, and leaves existing scores alone — which is right, since a score is a document and its instrument names should not shift under it because a library changed.
- **An outright error is still worth reporting.** A French name no French musician would use is wrong for everyone, not just for you, and belongs in the bundled library rather than in your local adjustments.

Some other things a user reads are not translated at all, by either mechanism: piece titles, filenames, and anything else a user typed.

## Why Your Language May Read Oddly

The interface translations Ooloi ships today were produced by AI and reviewed by one person, who does not speak most of the languages involved. They are a starting point, not an authority.

That is a deliberate choice and worth explaining, because it determines what we are asking of you.

The alternative was to ship in English, or in the two or three languages someone could vouch for, and to add the rest as volunteers appeared. That would mean a Ukrainian musician opening Ooloi for the first time and finding it in English — and having no reason to come back and translate a program they have already decided is not for them. A provisional translation everywhere beats a good translation in three places and nothing in nineteen: the Ukrainian musician gets a Ukrainian Ooloi on first launch, finds the eight things that are wrong with it, and is annoyed enough to fix them. That is a far better position to start from.

So the honest statement is this: **where a native-speaking musician disagrees with what Ooloi says, they are right.** Both halves of that matter. A native speaker is authoritative about idiom and register; a musician is authoritative about which of your language's words names the thing on the page. You are not editing a professional translation. You are replacing a machine's guess with knowledge.

Three weaknesses recur, and they are worth knowing before you start:

**Terminology.** A machine does not know that a stave and a system are different things, and it will cheerfully use one word for both. It does not know which of your language's several words for *key* means the tonality and which means the piano key. Where Ooloi's vocabulary is wrong, it is usually wrong consistently, which at least makes it easy to find and fix everywhere at once.

**Register.** Menu items are terse and imperative; explanatory text is not. A machine translating a menu item often produces a sentence, and translating a sentence often produces something clipped. If a menu item in your language reads like a paragraph, that is this failure.

**Idiom.** Some strings are grammatical and still wrong — constructions no native speaker would choose, or that follow English word order in a language that puts things elsewhere. These are the hardest for anyone but a native speaker to see, and the most valuable to fix.

## What Ooloi Asks of a Translation

**One term per concept, everywhere.** This matters more than any individual phrase being elegant. If your language has two acceptable words for a stave, pick one and use it throughout. A catalogue that uses both invites the next reader to assume a distinction is intended, and to preserve a difference that was only ever inconsistency.

This is about *vocabulary*, not about surface forms. In an inflected language the chosen term will properly appear in several shapes according to case, number or gender, and that is correct grammar rather than inconsistency. Nobody should flatten a declension to make a file look uniform.

When you do change a term, change it everywhere in the same pass.

**Musical Italian stays Italian.** Dynamics — *pp*, *p*, *mp*, *mf*, *f*, *ff*, *sfz*, *fp* — are never translated in any language. Neither are traditional tempo markings: *Allegro*, *Andante*, *Adagio*. These are international, and a musician reading your language expects them exactly as they are. The labels *around* them are translated: the field called "Tempo marking" gets your language's words, and what a user types into it is their business.

**Follow your language's own capitalisation.** English capitalises interface text more than most languages do. Use whatever your language does in its own software, which for many languages means sentence case, and for German means capitalised nouns wherever they fall.

**Keep menu items short.** A menu item that wraps or truncates is worse than one that is slightly less precise. There is no fixed budget, but the English is a reasonable yardstick: a translation running to twice its length will probably show.

## How Ooloi Chooses Its Words

Every user-visible string in Ooloi is a **key** rather than literal text. Instead of the word "Open", the program holds `menu.file.open`, and asks the localisation layer what that means in the current language. The layer looks the key up in a **catalogue** — one file per locale — and returns whatever it finds.

Those catalogues are [GNU gettext](https://www.gnu.org/software/gettext/) `.po` files. That format is a deliberate choice and not an obvious one for a Clojure program, so the reasoning is worth stating:

- It is thirty years old and completely stable. A catalogue written today will still open in a decade.
- Every translation tool on earth reads it. That is the decisive point: it means you are not obliged to use anything we built. Whatever you already use, it handles `.po`.
- It is plain text, so it survives version control, diffs sensibly, and can be edited with nothing more than a text editor.
- It carries the two mechanisms a program actually needs — per-message **context**, so the same English word can be translated differently in different places, and per-language **plural rules**, which are far more varied than English suggests.

**British English is the source language.** Every catalogue is a translation of `en_GB.po`, which is the only file written rather than translated. American English is itself a translation, and differs in more than spelling: en-GB says *stave* throughout, en-US says *staff*.

**An untranslated key falls back to English.** If your catalogue has no entry for a key, Ooloi shows the English text rather than failing or leaving a blank. The consequence is a good one: you may translate as much or as little as you like, in any order, and Ooloi always runs.

## Where the Files Are

**The first time Ooloi runs, it writes all twenty-two catalogues into your own user folder.** That folder is where you work:

| Platform | Folder |
|---|---|
| macOS, Linux | `~/.ooloi/i18n` |
| Windows | `%APPDATA%\Ooloi\i18n` |

Inside you will find `en_GB.po`, `sv_SE.po`, `ja_JP.po` and the rest. Open the one for your language, edit it, restart Ooloi, and your changes are live. There is no build step and no tooling to install.

The catalogues that ship *inside* the application live in the source tree at `shared/resources/i18n`. Those are the ones a release carries. You do not edit them directly; the copies in your user folder are what Ooloi actually reads.

Three rules govern the relationship, and the last two surprise people:

**Your copy wins.** When Ooloi starts, it loads the catalogues bundled inside the application and then loads whatever it finds in your folder, and yours replaces the bundled one for that locale.

**Your copy replaces the bundled one entirely — it is not merged with it.** If you delete entries from your file, those strings are gone from your language and fall back to English, even though the bundled catalogue still has them. So edit entries; do not remove them. If you send in a catalogue with entries missing, the build will reject it — a submitted catalogue must hold the same set of keys as `en_GB.po`.

**Your copy is never overwritten, which also means it is never updated.** Ooloi copies a catalogue into your folder only if it is not already there. This guarantees your work is never destroyed by an upgrade — the more important guarantee of the two — but it has a consequence: after upgrading Ooloi, your file is still the old one, and it wholly replaces the newer bundled version. Translation improvements shipped in that release will not appear, and any strings the release added will show English. **After upgrading, compare your file against the shipped one and take in what is new.** Keeping a copy of the file you started from makes that comparison straightforward. Merging newly added strings into an existing catalogue automatically is something Ooloi may grow later; until it does, the comparison is manual.

That last point is also why an unexpected English string has two possible explanations. If it appears in a phrase you have already translated, something is wrong and it is worth reporting. If it appears after you upgrade, in a part of Ooloi that is new, it is this: a string your catalogue does not have yet.

## Anatomy of an Entry

A catalogue begins with a header describing the file. The line that matters to you is the last one:

```po
msgid ""
msgstr ""
"Project-Id-Version: Ooloi 0.1.0\n"
"Last-Translator: \n"
"Language-Team: \n"
"Language: sv-SE\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Plural-Forms: nplurals=2; plural=(n != 1);\n"
```

`Plural-Forms` declares how many plural forms your language has and how to choose between them. It is explained under [Plurals](#plurals).

Everything after the header is entries, and every entry has the same shape:

```po
#: ../frontend/src/main/clojure/ooloi/frontend/ui/app/commands.clj
msgctxt "menu.file.open"
msgid "Open…"
msgstr "Öppna…"
```

| Line | What it is | Do you touch it? |
|---|---|---|
| `#:` | Where in the program this string is used | No — it is regenerated |
| `msgctxt` | The **key**. The program asks for this | **Never.** Change it and the string vanishes from your language |
| `msgid` | The British English source text | **Never.** It is not displayed; it identifies the entry |
| `msgstr` | Your translation | **Yes. This line is yours** |

When you are correcting an existing catalogue, `msgstr` is the only thing you ever edit. `msgctxt` is the entry's identity — the program looks the string up by it and nothing else, so changing it removes the string from your language as surely as deleting the entry. `msgid` is not used at runtime, but it is how the entry is matched against the English source, and editing it will simply be undone the next time the catalogues are regenerated. (Creating a *new* language is the one exception, and is covered [below](#adding-a-language-nobody-has-done-yet).)

**Placeholders must survive.** Some strings contain `%{something}`, which the program replaces at runtime with a name, a number or a filename:

```po
msgctxt "notification.open.file-missing"
msgid "'%{name}' no longer exists."
msgstr "»%{name}« finns inte längre."
```

The placeholder must appear in your translation, spelled exactly as in the `msgid` — but it may go **wherever your language puts it**. That freedom is the whole reason Ooloi uses named placeholders rather than numbered ones, and moving them is expected rather than merely tolerated. Dropping one, or renaming it, leaves a user looking at a gap where a filename should be.

If moving a placeholder puts it at the **start** of a sentence, remember that what arrives there is a value Ooloi supplies, and Ooloi will not capitalise it for you. Where your language would capitalise a sentence-initial word, prefer a construction that does not begin with the placeholder.

**Long strings are split across lines.** The parser joins them; the line breaks are not part of the text:

```po
msgctxt "about.text"
msgid ""
"Ooloi is a music notation program. "
"It is free software."
msgstr ""
"Ooloi är ett notskrivningsprogram. "
"Det är fri programvara."
```

Note the trailing space inside the first fragment. Without it the joined string reads *programvara.Det*.

**Quotation marks and backslashes are escaped** as `\"` and `\\`. A newline inside a string is `\n`.

## Plurals

English has two forms — *one stave*, *two staves* — and most software is written as though every language were the same. Yours may have one form, two, or three, chosen by rules that have nothing to do with English's.

Your catalogue's header declares which:

```po
"Plural-Forms: nplurals=3; plural=(n==1 ? 0 : n%10>=2 && n%10<=4 && (n%100<10 || n%100>=20) ? 1 : 2);\n"
```

An entry that counts something then carries one translation per form:

```po
msgctxt "il.undo.add-instruments"
msgid "Add %{names}"
msgid_plural "Add %{n} Instruments"
msgstr[0] "Dodaj %{names}"
msgstr[1] "Dodaj %{n} instrumenty"
msgstr[2] "Dodaj %{n} instrumentów"
```

**Every form must be a complete, grammatical phrase on its own.** The program picks exactly one of them and shows it; the others are never seen. A form left as a copy of another is a form that will be wrong for every count that selects it.

**How many forms, and what each covers:**

| Locale | Forms | What the forms mean |
|---|---|---|
| `ja_JP`, `ko_KR`, `zh_CN` | 1 | No grammatical plural; one form serves every count |
| `en_GB`, `en_US`, `da_DK`, `de_DE`, `el_GR`, `es_ES`, `fi_FI`, `hu_HU`, `it_IT`, `nb_NO`, `nl_NL`, `pt_PT`, `sv_SE` | 2 | Form 0 for *n* = 1; form 1 for everything else |
| `fr_FR`, `pt_BR` | 2 | **Form 0 covers 0 *and* 1**; form 1 covers 2 and above |
| `is_IS` | 2 | **Form 0 covers 1, 21, 31, 41 … but not 11.** Form 1 everything else |
| `cs_CZ` | 3 | Form 0 for 1; form 1 for 2–4; form 2 for 5 and above |
| `pl_PL` | 3 | Form 0 for 1; form 1 for 2–4 and 22–24, 32–34 …; form 2 for the rest |
| `uk_UA` | 3 | **Form 0 for 1, 21, 31 … but not 11**; form 1 for 2–4 and 22–24 …; form 2 for the rest, including 0 |

The bolded rows are where translators are most often caught out. French and Brazilian Portuguese count zero as singular — *0 fichier*, not *0 fichiers*. Icelandic and Ukrainian select their first form for 21 and 31 as readily as for 1, and specifically not for 11, so a form written as though it only ever meant "one" will appear attached to twenty-one.

**Plural shape belongs to your language, not to English.** A key may carry plural forms in your catalogue and a single string in another, and that is correct rather than an inconsistency to be tidied. English may have one invariant phrase where your language inflects with number; then your catalogue needs the plural forms and English does not. The reverse also happens. Follow your own grammar and disregard the shape of the English entry.

## Names Inside Sentences

A few strings have another translated string dropped into them, and these need more care than an ordinary placeholder, because two pieces of your language meet inside one sentence and the join is yours to get right.

The Edit menu is the standing example. Ooloi holds a template —

```po
msgctxt "menu.edit.undo-item"
msgid "Undo %{name}"
```

— and `%{name}` arrives already translated, as *Reorder Instruments* or *Add Musician*. Four rules govern it.

**Put `%{name}` where your language puts it.** English follows the verb; several languages do not. Undo and redo can differ within one language, so they are listed separately:

| | Undo | Redo |
|---|---|---|
| **Name follows the verb** | en, fr, es, it, pt, sv, da, nb, is, pl, cs, uk, el, **fi** | en, fr, es, it, pt, sv, da, nb, is, pl, cs, uk, el |
| **Name precedes the verb** | de, nl, hu, ja, ko | de, nl, hu, ja, ko |
| **Verb wraps around the name** | — | **fi** — `Tee %{name} uudelleen` |
| **No space between verb and name** | zh | zh |

Finnish is the reason the two directions are listed apart: its undo is `Peru %{name}`, verb first, while its redo is a particle verb that splits around its object.

**Use the verb Ooloi has chosen for your language.** These are the words a musician's own operating system uses in its Edit menu — but the platforms do not agree with each other. Apple's German is *Widerrufen* where Windows says *Rückgängig*; Apple's Dutch is *Herstel* where Windows says *Ongedaan maken*. Ooloi carries one word per language rather than one per platform, so a choice was necessary, and it follows Apple's:

| | Undo | Redo | | | Undo | Redo |
|---|---|---|---|---|---|---|
| en | Undo | Redo | | hu | Visszavonás | Ismétlés |
| de | Widerrufen | Wiederholen | | is | Afturkalla | Endurtaka |
| fr | Annuler | Rétablir | | it | Annulla | Ripristina |
| es | Deshacer | Rehacer | | ja | 取り消す | やり直す |
| sv | Ångra | Gör om | | ko | 실행 취소 | 실행 복귀 |
| da | Fortryd | Gentag | | nl | Herstel | Opnieuw |
| nb | Angre | Gjør om | | pl | Cofnij | Przywróć |
| fi | Peru | Tee `%{name}` uudelleen | | pt | Desfazer | Refazer |
| cs | Odvolat | Opakovat | | uk | Відмінити | Повторити |
| el | Αναίρεση | Επανάληψη | | zh | 撤销 | 重做 |

*These twenty rows cover the twenty languages: en serves en-GB and en-US, pt serves pt-PT and pt-BR.* This table was compiled from the platform's own localised documentation where the platform offers the language, and from the prevailing local convention where it does not — Icelandic is the one such case here, macOS not being localised into it. That makes it firmer ground than most of what a catalogue currently contains, but if it is wrong in your language it is still wrong, and worth saying so.

**No colons, and no quotation marks unless your grammar needs them.** The name attaches grammatically rather than with punctuation. Hungarian is the one language here that does need quotes, and for a specific reason: its wrapper ends in a possessive, and the inserted phrase carries a possessive of its own, so the two collide — *Zenész hozzáadása visszavonása*. Quoting makes the inner phrase a citation and the collision disappears: `„%{name}” visszavonása`. Use that remedy where your language has the same problem, and not otherwise.

**The inserted name appears in two places, so it must work in both.** The same phrase that follows the verb in the Edit menu also stands alone as a row in the edit-history window. It is therefore written in its plain, uninflected, dictionary form, and never bent into whatever case the surrounding verb would normally govern — that would fix one place and break the other.

This has a consequence in languages where the wrapper is a verb: the inserted phrase should **name the action** rather than command it. German *Musiker hinzufügen* inside *%{name} widerrufen* produces two stacked infinitives; *Hinzufügen des Musikers* does not, and also reads correctly as a history row, where an imperative would look like an instruction rather than a record of something done.

## Musical Terminology

The tables below are working vocabulary rather than settled authority: like the rest of the catalogues, most of it was machine-produced and has not been checked by a native-speaking musician. Treat it as the first thing to verify in your language, not as a ruling.

**The stave.** Ooloi's most-used noun, and the one most often mistranslated:

| Locale | Term | | Locale | Term |
|---|---|---|---|---|
| `cs_CZ` | notová osnova | | `it_IT` | rigo |
| `da_DK` | nodesystem | | `ja_JP` | 譜表 |
| `de_DE` | Notensystem | | `ko_KR` | 보표 |
| `el_GR` | πεντάγραμμο | | `nb_NO` | notesystem |
| `en_GB` | stave | | `nl_NL` | notenbalk |
| `en_US` | staff | | `pl_PL` | pięciolinia |
| `es_ES` | pentagrama | | `pt_BR` | pauta |
| `fi_FI` | nuottiviivasto | | `pt_PT` | pauta |
| `fr_FR` | portée | | `sv_SE` | notrad |
| `hu_HU` | vonalrendszer | | `uk_UA` | нотний стан |
| `is_IS` | nótnastrengur | | `zh_CN` | 五线谱 |

A stave and a **system** are different things. A system is the group of staves sounding together on one line of the score, read simultaneously — it may be bracketed or not, and may consist of a single stave. Several languages build both words on the same root, and where yours does, take care that Ooloi's *Stave View* and *System View* do not end up with the same name.

**Intervals**, which appear in transposition:

| English | German | French | Italian | Spanish |
|---|---|---|---|---|
| unison | Prime | unisson | unisono | unísono |
| second | Sekunde | seconde | seconda | segunda |
| third | Terz | tierce | terza | tercera |
| fourth | Quarte | quarte | quarta | cuarta |
| fifth | Quinte | quinte | quinta | quinta |
| sixth | Sexte | sixte | sesta | sexta |
| seventh | Septime | septième | settima | séptima |
| octave | Oktave | octave | ottava | octava |
| major | groß | majeur | maggiore | mayor |
| minor | klein | mineur | minore | menor |
| perfect | rein | juste | giusta | justa |
| augmented | übermäßig | augmenté | aumentata | aumentada |
| diminished | vermindert | diminué | diminuita | disminuida |

**Never translated:** dynamics (*pp* through *ff*, *sfz*, *fp*) and traditional tempo markings (*Allegro*, *Andante*, *Adagio*). These are Italian in every language.

## Choosing an Editor

A `.po` file is plain text, and a text editor is a perfectly good tool for the job — Ooloi's catalogues are a few hundred entries, not tens of thousands. If you are comfortable in one, you need nothing else.

A dedicated PO editor adds things worth having on a longer session: it shows you which entries are untranslated, keeps plural forms in separate fields rather than as raw `msgstr[0]` lines, and will not let you break the file's syntax. Widely used ones include **Poedit** (macOS, Windows, Linux; free, with a paid tier adding machine-translation assistance), **Lokalize** (Linux, macOS, Windows; free and open source), **Gtranslator** (Linux; free and open source), and **OmegaT** (anywhere Java runs; free and open source, aimed at professional translators working across many files). **Weblate** is a web platform rather than a desktop application, and suits a group working together more than an individual.

Rather than trust any feature list, including this one, **run three checks on whatever you choose**, using a copy of your own catalogue:

1. **Open the file and find any entry. Can you see its `msgctxt`?** Every one of Ooloi's entries carries a context, and it is the entry's identity. An editor that hides it is inconvenient; an editor that discards it on save destroys the file.
2. **Find an entry with plural forms. Are all of them editable?** Some tools show only the first.
3. **Change one string, save, and compare against the copy.** What must survive is every context, every entry, every plural form, every placeholder and the translator comments. Tools legitimately differ in how they wrap long lines or update the header — that is not damage. Entries vanishing, contexts disappearing or plural forms collapsing is.

The third check takes a minute and is the one that matters. Any tool that passes it is fine.

## Adding a Language Nobody Has Done Yet

Ooloi's twenty-two locales are not a fixed list, and **adding a new one does not usually require a new version of Ooloi**. The exception is a language whose plural rule Ooloi does not yet recognise; step 2 explains how to tell.

1. **Copy `en_GB.po`** in your language folder to a new file named for your locale — `nl_BE.po`, `ca_ES.po`, `et_EE.po`.
2. **Set the header.** `Language:` to match your locale, and `Plural-Forms:` to your language's rule.

   **Ooloi recognises these seven rules and no others, and matches them as exact text.** Copy one of these expressions character for character:

   ```
   No plural — Chinese, Japanese, Korean, Turkish, Vietnamese
   nplurals=1; plural=0;

   Two forms, singular at 1 — most European languages
   nplurals=2; plural=(n != 1);

   Two forms, singular at 0 and 1 — French, Brazilian Portuguese
   nplurals=2; plural=(n > 1);

   Icelandic-style
   nplurals=2; plural=(n%10!=1 || n%100==11);

   Czech-style
   nplurals=3; plural=(n==1) ? 0 : (n>=2 && n<=4) ? 1 : 2;

   Polish-style
   nplurals=3; plural=(n==1 ? 0 : n%10>=2 && n%10<=4 && (n%100<10 || n%100>=20) ? 1 : 2);

   Ukrainian-style — also Russian, Belarusian, Serbian, Croatian
   nplurals=3; plural=(n%10==1 && n%100!=11 ? 0 : n%10>=2 && n%10<=4 && (n%100<10 || n%100>=20) ? 1 : 2);
   ```

   A rule Ooloi does not recognise is not reported to you — it is ignored, and **every count in your language then renders the first form**, silently. So if your language needs a rule that is not in this list, the catalogue is still worth writing, but say so when you send it in: the rule has to be added to Ooloi itself, and that part genuinely does need a new version.

3. **Empty the `msgstr` lines.** Copying `en_GB.po` leaves English text sitting in every translation slot, so nothing is marked as untranslated and your editor's view of what remains to be done will show nothing at all. Emptying them — `msgstr ""` — fixes that: every key stays where it is, so the file remains complete, and an empty translation falls back to English at runtime exactly as a missing one would.

4. **Translate the `msgstr` lines.** You can do a hundred and leave the rest; the untranslated ones show English.
5. **Add plural forms where your language needs them and English does not.** This is the step that copying `en_GB.po` cannot do for you. English carries plural forms on some twenty entries, but your language may inflect with number where English does not — the phrase counting connected collaborators is one such, and in the three-form languages the derived score title is another. For any entry whose text changes with a count in your language, add a `msgid_plural` line (repeating the English) and replace the single `msgstr` with `msgstr[0]`, `msgstr[1]` and so on, one per form your header declares. **Nothing checks this for you**: an entry left with a single form is valid, and simply shows that one form for every count.
6. **Restart Ooloi.** Your language appears in the language list in Settings, named in its own language.

**One current limit.** Ooloi does not mirror its interface for right-to-left languages. The text of a translation displays correctly, but menus, dialogs and panels stay laid out left-to-right, so Hebrew, Arabic, Persian and Urdu are not yet properly supported. The catalogue you write would not be wasted, but the result will not look right until Ooloi handles direction.

## Sending Your Work Back

Improvements to the interface catalogues are only worth making once. If you have corrected your language, send the file back so that the next person to install Ooloi gets your version rather than the machine's. (Instrument library adjustments are a different matter — as described under [The Two Surfaces](#the-two-surfaces), those belong to the Ooloi that holds the library.)

Open an issue on [Ooloi's repository](https://github.com/PeterBengtson/Ooloi) with the file attached, or send a pull request if you are comfortable doing so. What helps most in the accompanying note:

- **Which locale**, and whether you are a native speaker.
- **Any term you changed throughout** — "I replaced *notsystem* with *notrad* everywhere" tells the maintainer that the change is deliberate and pervasive, rather than an inconsistency to be reconciled.
- **Anything you were unsure about.** A question recorded is worth more than a guess silently committed, and other translators hit the same questions.

**Licence.** Contributions to Ooloi, translations included, are under the [Mozilla Public License 2.0](https://www.mozilla.org/MPL/2.0/), as described in [CONTRIBUTING.md](https://github.com/PeterBengtson/Ooloi/blob/main/CONTRIBUTING.md).

**Credit.** Every catalogue has `Last-Translator` and `Language-Team` fields in its header. If you would like your work attributed, put your name there; it travels with the file.

Keep your own copy. As explained under [Where the Files Are](#where-the-files-are), an upgrade will not overwrite it, and you will want it for comparison.

## Cross-References

- **[ADR-0039: Localisation Architecture](../ADRs/0039-Localisation-Architecture.md)** — the full specification: how keys are structured, how the catalogues are verified at build time, and the invariants the system holds to. Written for implementers rather than translators, and not required reading for anything in this guide.
- **[Bundled Instrument Library](INSTRUMENT_LIBRARY_CATALOGUE.md)** — every instrument Ooloi ships with, grouped by family, with clefs and transpositions.
- **[GNU gettext manual](https://www.gnu.org/software/gettext/manual/gettext.html)** — the authority on the `.po` format itself, including the plural-form expression for a long list of languages.
