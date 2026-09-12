# The tree read as a set

[The pages](INDEX.md) - [what x3 is](../README.md)

## The tree itself is a set

```json
{ "kind": "consistency",
  "left":  { "from": "tree", "select": "dirs:apps/*" },
  "right": { "from": "tree", "select": "files:apps/*/gate.ps1" } }
```

Every other extractor reads the **inside** of a file, and the inside of a file
can only be read once the file is there. So the question *"does every directory
of this shape carry these paths"* cannot be asked with them: the rule opens with
the very file it requires, and a directory that carries nothing never becomes a
subject at all. The gate is then green about the only case it existed for.

| `select` | A member is |
|---|---|
| `dirs:<pattern>` | a directory whose path matches |
| `files:<pattern>` | a file whose path matches |

**What enters the set is what the stars caught**, the same rule the other
extractors follow. `dirs:apps/*` yields `alpha`, `beta`; `files:apps/*/gate.ps1`
yields only the apps that have one — so the two sides are comparable and the
finding names the app, not a path. A pattern with no star puts the path itself
in the set, and the question becomes a plain "is it there".

A directory counts when it holds at least one file. Nothing is lost by that: git
cannot record an empty directory, so in a tree under version control "a
directory that exists" and "a directory holding a file" are the same set.

The rule's `sources` does not narrow this side — that narrowing is exactly the
blindness — but the rule's `exclude` still does, and a dead `exclude` pattern is
red as everywhere else. A pattern that matches nothing leaves the side empty,
and an empty side is `empty_scope`, not a tree full of debts: a mistyped pattern
must not be able to write its own mistake onto the code.

<!-- x3-dist version=v0.158.0 capabilities=e84674ef730d4c01bd28143e856bf5a927aed61a6b4eead5472b98cb6d4baaa2 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
