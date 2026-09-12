# arch reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.156.0`**

## component path patterns

| Pattern | Matches |
|---|---|
| `**` | zero or more path elements; at the end of a pattern, **at least one** |
| `*` | a run inside one element, never crossing `/` |
| `?` | one character inside one element |

## arch rule fields

| Field | Required | Meaning |
|---|---|---|
| `name`, `kind` | yes | unique in the file; one of the nine kinds |
| `match` | `deps` only | `import`, `literal` or `symbol` |
| `from`+`deny` / `to`+`allowFrom` | deps | the outward / inward question |
| `pattern`+`owner` / `pattern`+`from` | literal | ownership / prohibition |
| `unknownOwner` | literal | `report` (default) or `ignore` a captured owner no instance carries |
| `marker` | required | the mark every file in `sources` must carry |
| `counterpart`+`requires` | pairing | the file that must name this one |
| `satisfiedBy` | pairing | what else counts as a counterpart |
| `subjects` | pairing | which of the matched files the rule takes as subjects |
| `value`+`allow` | flow | the value to follow, and where it may appear |
| `surface`+`fields`+`carrier` | exposure | where to watch, which names, written how |
| `across`+`minLines` | duplication | the component compared with itself |
| `in`+`terms`+`comments` | vocabulary | the layer, the words, whether prose counts |
| `keys` | containment | the ownership prefix per component |
| `left`+`right`+`compare` | consistency | the two sets and how they must agree |
| `per` | consistency | compare each instance of a component with **itself** |
| `parts`+`join`/`each`, `skip`, `comments`, `strings`, `syntax`, `invoke` | consistency | extractor details |
| `absent`, `relativeTo` | `left-exists-on-disk` | paths meant to be missing; `repo` (default) or `source` |
| `except` | no | `self` only, next to `from` + `deny` |
| `minimum` | no | the fewest subjects the rule must see |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `sources` / `exclude` | no | this rule's file set; its own list replaces the inherited one |

## arch error codes

| Code | Raised by | Meaning |
|---|---|---|
| `forbidden_dependency` | `deps` | a forbidden import edge, or a name a component may not spell or use |
| `foreign_resource` | `deps:literal` | a component spelled a name another owns |
| `escaped_value` | `flow` | the value appeared where it may not |
| `exposed_field` | `exposure` | a hidden name reached the surface |
| `duplicate_body` | `duplication` | the same body in two instances |
| `foreign_term` | `vocabulary` | a layer let through a word it must not know |
| `part_outside_its_root` | `containment` | a part sits outside its component's root |
| `set_mismatch` | `consistency` | the two sets drifted; each difference is named |
| `missing_target` | `consistency` | a value read as a path leads nowhere |
| `missing_marker` | `required` | a file of the class does not carry the mark |
| `missing_counterpart` | `pairing` | no counterpart, or it names nothing from the subject |
| `empty_scope` | every rule | a component, source set or followed field matched nothing |
| `scope_below_minimum` | every rule | fewer subjects than `minimum` |
| `dead_exemption` / `dead_exclusion` / `dead_filter` | escape hatches | an exemption, exclusion or filter that took nothing out |
| `dead_satisfier` | `pairing` | `satisfiedBy` rescued no subject |
| `unjudged_exemption` | any | an exemption names a rule this configuration does not load (warning) |

## how a rule reads a file

| Setting | Where | What it does |
|---|---|---|
| `comments: "exempt"` | `deps:literal`, `vocabulary` | the default: comments are not read |
| `comments: "checked"` | `deps:literal`, `vocabulary` | prose counts too; the name may not even be mentioned |
| `syntax` | the `arch` section | comment syntax per extension; **replaces** the embedded entry, never merges with it |
| `syntax: { ".go": ... }` | the `arch` section | how a comment is written **inside a Go string**, applied per literal |

<!-- x3-dist version=v0.156.0 capabilities=c8bb02269798cd209388b465a9941635adeb7d3cc673c6643bd116b9a446af22 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
