# The names a template calls

[The pages](INDEX.md) - [what x3 is](../README.md)

### `template` — a markup file and the script that feeds it

**Catches:** a template calling a name its component does not offer. A renamed
method, a value moved into another part, a directive a refactor left behind —
the production build of a template framework says nothing, the button quietly
stops working, and the cause is visible nowhere. No compiler reads both files.

```json
{ "kind": "consistency", "per": "file:stem",
  "sources": ["ui/**/parts/*.html", "ui/**/parts/*.js"],
  "left":  { "from": "template", "select": "calls" },
  "right": { "from": "template", "select": "bindings", "resolve": ["ui"] },
  "compare": "left-subset-of-right" }
```

`calls` reads a markup file: the root identifiers of every directive value and
every `{{ }}` expression. What the **template itself** declares is dropped —
loop locals, arrow-function parameters — because they come from the template and
not from the component, and looking for them in the script is a false red. So
are string literals (the key inside `t('asst.title')` is no identifier), field
names after a dot, and object keys. A commented-out call is not a call, and the
names the framework and the browser bring are known here rather than written
into every project's settings.

`bindings` reads the script beside it: its declarations, its named imports, and
the names its **mixins** and **spreads** bring in. Those are followed, not
listed — `mixins: [base]` is resolved to the file `base` was imported from and
read with the same extractor, two levels deep. A hand-kept list of what a mixin
offers rots in both directions: forgotten it turns red on names that do exist,
stale it stays silent on names that do not.

An absolute import path (`/core/base.js`) is a server path, and what it points at
in the tree is the project's knowledge, not a guess the engine may make:
`resolve` names the roots it is joined to. Where the chain cannot be resolved it
stops, and the names it would have brought turn **red** rather than quietly
passing.

**`per: "file:stem"`** — the pair, not the component. A component pattern cannot
ask this: its star captures the file name **with** its extension, so a template
and the script feeding it become two instances that never meet. The stem groups
them — `parts/chip.html` and `parts/chip.js` are one instance — and each side
reads only the files of its own kind out of the scope they share.

<!-- x3-dist version=v0.249.0 capabilities=13b2a10b301e1e17806104037af6cab6ede1f1713ffc6fec2f0c24691d84d625 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
