# One debt, one file, one writer

[The pages](INDEX.md) - [what x3 is](../README.md)

## The baseline, split

### A segment owns its paths, and writes its own file

```json
{ "baseline": { "dir": "baselines",
                "segments": { "billing": ["apps/billing/**"],
                              "web":     ["services/web/**"] } } }
```

| Finding under | Written to |
|---|---|
| `apps/billing/**` | `baselines/comments.billing.json` |
| `services/web/**` | `baselines/comments.web.json` |
| anything else | `baselines/comments.json` |

Two workers, each in its own region, now write two different files. Git has
nothing to merge.

`segments` is an **object**, so the parts a root configuration `include`s merge
into it key by key: each region declares its own baseline where it declares
everything else about itself. The directory stays single and is still resolved
[beside the root configuration](configuration.md#a-relative-path-is-relative-to-the-configuration)
— the split changes the file **name**, never the base.

### A segment may own a name instead of a place

A path splits a debt by **where** it sits. That is not enough to freeze one
rule: two rules reading the same files land in the same segment, so accepting
one rule's debt swallows the other rule's red. The second axis is the finding's
own `note` — for `arch` the rule's name, for `secrets` the pattern's, for
`syntax` the check's, for `lang` the token, for `boxes` the box's title.

```json
{ "baseline": { "dir": "baselines",
                "segments": { "pairing": { "notes": ["every-source-file-has-a-test"] },
                              "billing": { "paths": ["apps/billing/**"] } } } }
```

A segment written as a plain list is the path axis, exactly as before. Written
as an object it may declare `paths`, `notes`, or **both** — and both then have
to hold: *"this rule's debt, in these files"* is one sentence, not two. A
segment that declares neither is exit `2`; so is a blank name, which would take
no finding and quietly hold nothing.

What this buys: run `-update-baseline` once, keep the segment file for the debt
you accept, and drop the default file. The rule you froze stays quiet and every
other rule is red again — with one file per rule, the debt of each is a thing
you can read, review and delete on its own.

### The split divides the files, not the set

Reading is unchanged: a finding is held if **any** of the files holds it, so
declaring a segment on a tree that already has a baseline keeps every gate
exactly as green as it was. The next `-update-baseline` re-routes each record to
the file that owns it and says so:

```text
x3 comments: 612 finding(s) moved into the segment that owns them;
             the debt did not change, only the file that holds it
```

A move is a third direction next to growth and shrink: the debt neither appears
nor dies, so the growth refusal does not block it and the drop report does not
claim it. It happens once.

| Written this way | Result |
|---|---|
| a name that is not a file name | exit `2` — the name becomes part of a path |
| a segment owning no path | exit `2` — nothing would ever be written to it |
| `segments` without `dir` | exit `2` — the split lives under the directory |
| one path in two segments | exit `2` — a finding has one home, or two runs write it |
| the same id in two files | exit `2` — each run would believe the other froze it |

The last two are the same law seen twice: **one debt, one record, one writer.**
Give it two and the split has bought nothing.

<!-- x3-dist version=v0.127.0 capabilities=2801084864972251f16605de3f015cce95cb887510f91b29286bf29571907c96 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
