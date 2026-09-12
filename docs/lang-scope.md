# The three scopes of a language run

[The pages](INDEX.md) - [what x3 is](../README.md)

## The three scopes of a language run

**What it catches:** the language rule a project actually wrote, which is
usually narrower than *"every word in every file"* — and the places a word can
hide where no file's content is ever read.

A run asks three separate questions, and each one has its own answer.

### Are string constants part of the rule?

Many projects say: *the message a user sees is written in their language, the
field names around it are not.* Under the default every string constant is held
to the dictionary, which turns every screen of prose red and gets the gate
switched off by the afternoon.

```json
{ "language": { "strings": "any" } }
```

`strings` is the twin of `comments` and reads the same two values. It silences
**strings only** — identifiers stay red. A setting that could silence the
identifiers too would be a short way of switching the gate off.

⚠️ What that costs is named, and it has its own answer: field names written
**inside** a string — the keys of a structured log line — are strings, and
`any` would stop reading them. They are read anyway, by declaration; see [The
field name written inside a string](lang-fields.md#the-field-name-written-inside-a-string).

### Which files are read?

Left out, `sources` is the Go tree. Declared, it is whatever the project says:

```json
{ "language": { "sources": ["**/*.go", "ui/**/*.js"] } }
```

A `.go` file is read with the Go parser; every other file is read as text,
through the comment syntax of its language, so that the same three questions
(identifier · string · comment) can be asked of it. An extension whose comment
syntax is unknown is read as **all code** — the cost of not knowing picks a
direction, and a gate that sees too much says so out loud while one that sees
too little stays quietly green.

### Which names are read?

A file's name and a directory's name appear in no file's content, so content
scanning can never see them — and a directory is named once and colors every
path beneath it.

```json
{ "language": { "names": ["**/*.go", "scripts/**"] } }
```

Left out, no name is read: a project's naming rule is its own, and a tree whose
data files are deliberately named in another language should not turn red for
installing the engine. A directory is read when it **carries** a file whose name
is read, or when its own path matches a pattern — both are needed, because a
directory that holds nothing the run reads still has a name.

### A pattern that matches nothing

Either list may hold a pattern no path answers. That is a claim about the tree
that did not hold, so it is `empty_scope` and red — a misspelled pattern would
otherwise be the quietest way to switch a scope off. A list written as `[]` is
refused with exit `2`: an empty scope is not a narrow scope, it is no scope.

<!-- x3-dist version=v0.152.0 capabilities=99e1a9e5ef7349cef2de389de0c82b8654db18f948462add95ca7aead639768c template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
