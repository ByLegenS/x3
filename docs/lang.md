# One language outside comments

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 lang`

**What it catches:** a second language outside the comments — the author's
mother tongue leaking into identifiers, log lines and error messages.

**The dictionary runs in reverse.** There is no list of forbidden words; such a
list can only cover the language somebody thought to write down. What is known
is the **allowed** language, and every token outside it is red.

```
x3 lang [-config <file>] [-out <file>] [-baseline <file>] [-update-baseline] [dir]
```

### What is checked

| Read | Not read |
|---|---|
| the package name | comments (unless `comments: "en"`) |
| every **declared** identifier — function, type, variable, constant, field, parameter, result, label, import alias | the **use** of a name declared elsewhere |
| every string constant, struct tags included (unless `strings: "any"`) | import paths |
| the **field names** of a structured log call, wherever `fields` says they sit | the values next to those field names |

The asymmetry is deliberate: a name is spelled once where it is declared, and a
name declared elsewhere — `fmt.Fprintf`, `pgx.Connect` — is not yours to spell.

Which files are read, whether their strings are read at all, and whether the
**names** of files and directories are read are three separate questions; see
[The three scopes of a language run](lang-scope.md#the-three-scopes-of-a-language-run).

### The token rule

Text splits on anything that is not a letter and at case boundaries, with runs
of capitals kept together: `JSONPath` → `json` + `path`, `TOTAL` → `total`. A
run of capitals stays whole on purpose — split letter by letter, a foreign word
in capitals would dissolve into fragments and slip through. Fragments under
three letters are not read. Each remaining word must be in the embedded
dictionary or in `language.allow`. On top of that, one absolute rule: **any
non-ASCII letter outside a comment is red**, and `allow` cannot excuse it.

### `language` in `x3.json`

```json
{ "language": { "allowed": "en", "comments": "any", "strings": "any",
                "sources": ["**/*.go", "ui/**/*.js"], "names": ["**/*.go"],
                "allow": ["cfg", "ctx", "dsn", "omitempty"] } }
```

See **language settings** in the [lang reference](lang-reference.md#language-settings).

A seventh field, `fields`, says which strings are field names and therefore
identifiers; it has its own page — [The field name written inside a
string](lang-fields.md#the-field-name-written-inside-a-string).

No `language` section is not an error; the default is `en` / `any` / `en` / the
Go tree / no names / no field names. A section that *is* written and is wrong
stops the run — fail-closed.

### What a run looks like

```
sample.go:8:2: not_in_dictionary: notaword (identifier)
sample.go:14:18: non_ascii_letter: é (string)
x3 lang: 1 file(s) - 2 finding(s) - dictionary "en"
```

The report carries the same findings sorted by file and line, with no timestamp.
`code` is the stable part — `not_in_dictionary`, `non_ascii_letter` or
`empty_scope`; `where` is `identifier`, `string`, `field`, `comment`, `name` or
`scope`.

### The embedded dictionary

141,848 words are compiled into the binary (`internal/lang/english.txt`). It is
generated from the **English Speller Database** (ESDB, formerly SCOWL) at
<https://app.aspell.net/create>, size 70, US spelling, diacritics stripped, with
the `hacker` list included — which is why `http`, `auth` and `err` are already
words. Its licence requires the notice to travel with any copy:

> Copyright 2000-2026 by Kevin Atkinson
>
> Permission to use, copy, modify, distribute, and sell any part of the English
> Speller Database (ESDB, previously known as SCOWLv2), or word lists created
> from it, is hereby granted without fee, provided that the above copyright
> notice appears in all copies and that both the above copyright notice and this
> notice appear in supporting documentation. Kevin Atkinson makes no
> representations about the suitability of this database for any purpose. It is
> provided "as is" without express or implied warranty.

Do not edit the file by hand. A word that belongs to your project belongs in
`language.allow`.

<!-- x3-dist version=v0.147.0 capabilities=ec7474ac3ae437e0e7b4c481e6241021d5e7b6a9a998088c3a6178cf6d8f00af template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
