# The rules an engine carries

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 profile`

**What it catches:** the same rule written from scratch in every repository.

Splitting the configuration answers *where* a rule is written. It does not
answer *why it has to be written again at all*. Measured in one production
repository over four days: its settings grew from 490 to 2 540 lines while its
own scripts lost 5 744 — the growth is melted code, not waste. Then the rules
themselves were counted, 158 of them:

| Class | Share | What a second repository would write |
|---|---:|---|
| general | 19.7% | the same text, character for character |
| shaped | 49.1% | the same rule, with a different path, name or threshold |
| its own | 31.2% | nothing — the rule is about that repository |

Two thirds of a settings file is therefore a copy of a file nobody has written
yet. A line ceiling would not help: it makes *writing no rule* the cheapest way
past the gate. The engine carries the rule sets instead, and a project calls one:

```json
{ "profile": { "use": ["go-monorepo@1"], "with": { "text": ["go", "json", "md"] } } }
```

`x3 profile -library` lists what the engine carries; `x3 profile` prints what is
in force in this project, rule by rule, with the full text of each one. **A call
is not allowed to make a settings file unreadable** — whoever reads the settings
today sees every rule, and that has to survive the call.

### The version is part of the call

`go-monorepo` alone is refused; `go-monorepo@1` is the name. A set called without
a version would change under the project that called it, on the day the engine is
updated, with nothing in the project's own history to show for it.

### Tokens: one rule, many repositories

The shaped class above is half of everything, so the library rules take
**tokens**. A rule declares what it needs and what the project is to write there;
the project fills them in under `with`, and an unfilled token stops the run
(exit `2`) rather than measuring something nobody asked for:

```
x3: "profile" section: with is missing "text" - each file extension this
    repository keeps as text, one check per extension: go, json, md ...
```

**Only declared tokens are filled in.** The engine's own placeholders use the
same spelling (`{file}`, `{unit}`, `{overlay}`), and a library rule may carry
them; a filling that replaced everything in braces would quietly break that rule.

### `each`: the same rule, measured per item

A token can hold a **list**, and then the rule is built once per item. This is
the shape that produced the most duplication when it was counted by hand — the
same rule three, five, six times over, one list item apart:

```json
{ "each": "text", "rule": { "name": "every-{text}-file-decodes-as-utf8",
    "sources": ["**/*.{text}"], "encoding": { "as": "utf-8", "bom": "any" } } }
```

With `"text": ["go", "json"]` the run carries two checks and the project wrote
one word more. The rule's **name** must carry the token, or every copy would land
in the list under one name and only the last would be read.

<!-- x3-dist version=v0.171.0 capabilities=b3ee80044ca2375e9e2d5edb498234a5a45b2570d8c0b521eb7436c22a50a4bd template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
