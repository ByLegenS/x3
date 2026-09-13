# The words a language reserves

[The pages](INDEX.md) - [what x3 is](../README.md)

## Keywords are not foreign words

A file the Go parser cannot read is scanned token by token, and in that reading
every word is a candidate — including the words the **language itself** reserves.
`const`, `async`, `typeof`, `elif`, `varchar`: no dictionary carries them, so
each one arrives as a finding, at every occurrence. Measured on one tree: of
11,625 findings, 12% were exactly this — `const` 1,093 times, `async` 332 — and
under that noise no one reads the two hundred real ones.

So the reading knows what the language it is reading reserves
(`internal/source/keywords.txt`, one block per extension). Go is deliberately
**absent**: its reading goes through the parser and sees only declared names, and
a keyword can never be an identifier there — a list would open an exemption and
catch nothing.

The list is narrow on purpose, in three directions:

- **Only in code.** A keyword written inside a string or a comment is the
  author's word, not the language's, and is held to the dictionary as before.
  Without that cut the list would be an escape hatch: a forbidden word could be
  hidden by declaring it a keyword and writing it in prose.
- **Only the "not in the dictionary" question.** A non-ASCII letter outside a
  comment stays red whatever any list says. That law has no exemptions, and a
  keyword list is not one.
- **Replaces, never merges.** A project's `keywords` for an extension stands in
  for the embedded block rather than adding to it — inheriting half a language
  would make it unreadable which half is in force. Each entry is one ASCII word;
  an extension written without its leading dot stops the run.

```json
{ "language": { "sources": ["ui/**/*.js"],
                "keywords": { ".vue": ["defineProps", "defineEmits"] } } }
```

Control experiment, four directions over one tree, only the list changing:
the language's own keywords leave **1** red identifier (the genuinely foreign
one) while the same tree with `"keywords": { ".js": [] }` — a language declared
to reserve nothing, which is the reading as it was before this capability —
leaves **7**; a keyword in a comment or a string stays red in both; a project
list replaces the embedded one, so its own word goes quiet and the built-in ones
come back red.

<!-- x3-dist version=v0.176.0 capabilities=b557ad04f5f0efc6e52b7370ab28640203fa267a10c6b85b38ee9e370e6da5ff template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
