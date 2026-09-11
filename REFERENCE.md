# x3 reference

The lookup tables behind the guides: field names, error codes and exit codes,
one section per table, in the order the guides use them. Nothing here is new.
Every page links to the table it uses, and all of them are generated from one
document in one run, so no two can describe different versions.

[What x3 is](README.md) - [the pages](docs/INDEX.md)

**Current version: `v0.109.0`**

## scan exit codes

| Code | Meaning |
|---|---|
| `0` | green — every directive is well formed and in a legal scope |
| `1` | red — at least one directive failed; each is printed with `file:line` |
| `2` | usage error, or the run could not complete |

## the directive dictionary

| Directive | Valid scopes | Requires |
|---|---|---|
| `//x3:rule:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:guard:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:case: <payload>` | `decl` only | a payload that parses: `in=(...) out=...` |
| `//x3:import: [<name> ]<path>` | `file` only | an import path, and a name that is a plain identifier if one is written |
| `//x3:tags: <tag>[ <tag>...]` | `file` only | one or more build tags; a constraint expression is not a tag |
| `//x3:type: <declaration>` | `file` only | a type declaration, or a method on a type the same file declares |
| `//x3:live` | `decl`, `file`, `pkg` | nothing |
| `//x3:skip:<reason>` | `decl`, `file`, `pkg` | a reason |
| `//x3:allow:<type>:<reason>` | `decl`, `file`, `pkg` | a type **and** a reason |

## scan error codes

| Code | Turns red when |
|---|---|
| `unknown_category` | the type is not in the dictionary — no verifier exists for it |
| `malformed` | a required sub-type or reason is missing, a doubled colon left an empty sub-type, or a `case` payload does not parse |
| `scope_not_allowed` | the type is known and well formed, but not legal in this scope |
| `unattached` | the directive binds to nothing at all |

## the scan report fields

| Field | Notes |
|---|---|
| `version` | schema version; it goes up when a field changes meaning |
| `file`, `line` | relative to the scan root, always `/`-separated |
| `raw` | the directive line exactly as written |
| `category`, `segments`, `payload` | the parsed line; omitted when empty |
| `scope`, `target` | resolved binding; `target` only for `decl` |
| `status`, `code`, `message` | `ok` or `error`; the last two only on `error` |

## expectation fields

| Field | Meaning |
|---|---|
| `name` | required; the red names the expectation that was not met |
| `min` | required, at least 1 — an expectation of zero verifies nothing |
| `paths` | globs **relative to the configuration**; absent means the whole scan |
| `category` | `guard`, `rule`, `case`, ...; absent means any |
| `kind` | the first segment after the category; absent means any |

## case finding codes

| Code | Means |
|---|---|
| `example_failed` | the declaration was called and the result is not what the example says |
| `never_ran` | no verdict was reported for it, it was skipped, or a run that died elsewhere never reached it |
| `crashed` | the run started this example and never came back — the process died inside it |
| `does_not_build` | the example does not compile — charged to its own line when the compiler names one, and to every example in the package when the fault is in the package's own source |
| `malformed` | the payload has no body, does not parse, or holds a proposition that cannot fail |
| `not_a_function` | the example sits above something that cannot be called |
| `in_a_test_file` | the example is in a `_test.go` file, where nothing would run it |
| `wrong_result_count` | the declaration returns a different number of values than the example expects, or a method was given no receiver |
| `dead_type` | a declared type that no example names, reported on the line that declares it |
| `dead_import` | a declared import that no example names — from `case.imports`, asked only of a run whose tree contains the configuration; or from a `//x3:import:` line, asked of the file that carries it |
| `does_not_parse` | a source file the gate could not read at all — the parser stopped, so nothing in that package was measured |

## language settings

| Field | Meaning |
|---|---|
| `allowed` | the language outside comments; `en` is the only embedded dictionary, and any other value is an error, not a silent pass |
| `comments` | `any` (default) leaves comments alone; `en` holds them to the dictionary |
| `allow` | project terms no dictionary has. One ASCII word, three letters or more — an entry that could never match is rejected rather than ignored |

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
| `marker` | required | the mark every file in `sources` must carry |
| `counterpart`+`requires` | pairing | the file that must name this one |
| `value`+`allow` | flow | the value to follow, and where it may appear |
| `surface`+`fields`+`carrier` | exposure | where to watch, which names, written how |
| `across`+`minLines` | duplication | the component compared with itself |
| `in`+`terms`+`comments` | vocabulary | the layer, the words, whether prose counts |
| `keys` | containment | the ownership prefix per component |
| `left`+`right`+`compare` | consistency | the two sets and how they must agree |
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

## how a rule reads a file

| Setting | Where | What it does |
|---|---|---|
| `comments: "exempt"` | `deps:literal`, `vocabulary` | the default: comments are not read |
| `comments: "checked"` | `deps:literal`, `vocabulary` | prose counts too; the name may not even be mentioned |
| `syntax` | the `arch` section | comment syntax per extension; **replaces** the embedded entry, never merges with it |

## boxes criteria fields

| `when` | Fields | Holds when |
|---|---|---|
| `file` | `path` | `path` exists |
| `pattern` | `sources`, `match`, `directives` | `match` is found under `sources` |
| `absent` | `sources`, `match`, `directives` | `match` is found **nowhere** under `sources` |
| `sql` | `dsnEnv`, `query`, `equals`, `driver`, `timeoutMs` | the query's first cell equals `equals` |
| `command` | `command`, `args`, `output`, `timeoutMs` | it exits `0` **and** its output meets `output` |
| `manual` | `by`, `seen`, `signed` | `signed` is written |

## boxes criteria written in prose

| `when` | The rest of the line is read as |
|---|---|
| `file` | a path |
| `pattern`, `absent` | a place, then the expression; the place matches the file **and** everything under it |
| `sql` | the query, `==`, the value it must give |
| `command` | arguments appended to `prefix` |
| `manual` | who looks, the separator, what they must see |

## guard exit codes

| Code | Meaning |
|---|---|
| the command's own | the guards allowed the launch |
| `1` | a `block` guard was red, and the command was never started |
| `2` | the configuration or report could not be read or written, or the command could not start |

## guard fields every kind has

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file |
| `kind` | yes | `sql`, `http`, `exec` or `steps` |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `tags` | no | what `-only` and `-skip` select on; a guard with none always runs |
| `timeoutMs` | no | defaults to `10000` (`steps`: `600000`); a dead dependency must not hang the gate forever |

## guard fields for kind sql

| Field | Required | Meaning |
|---|---|---|
| `dsnEnv` | yes | **name** of the variable holding the DSN; the DSN never appears in the file |
| `query` | yes | its first row, first column is the observed value |
| `driver` | no | defaults to `pgx`; a name this binary has not registered is a configuration error (exit `2`) |
| `equals` / `contains` | one of them | what the observed value must be |

## guard fields for kind http

| Field | Required | Meaning |
|---|---|---|
| `url` | yes | the address; the request is always a `GET` |
| `status` | yes | the expected status code |
| `headerEnv` | no | header name → **name** of the variable holding its value |
| `jsonPath` | no | an RFC 6901 JSON Pointer into the body; without it the whole body is the value |
| `equals` / `contains` | no | with only `status`, the status code alone is the assertion |

## guard fields for kind exec

| Field | Required | Meaning |
|---|---|---|
| `command` | yes | executable to run |
| `args` | no | its arguments |
| `equals` / `contains` | no | what its trimmed stdout must be; without either, **exit code 0** is the assertion |

## guard fields for kind steps

| Field | Required | Meaning |
|---|---|---|
| `steps` | yes | run **in order**; the first that does not hold ends the trial and names itself |
| `workspace` | no | a temporary working area: `copy` (required within it), `remove`, `write` |
| `equals` / `contains` | no | what the **last** step's output must be; without either, every step holding is the assertion |

## the guard report fields

| Field | Notes |
|---|---|
| `guards[].status` | `pass`, `fail` (it ran and disagreed) or `error` (it could not run); both non-`pass` values are red |
| `guards[].policy` | the policy applied to **this** guard — always present, so the report explains its own decision |
| `expected` / `observed` / `detail` | what was wanted, what was seen, why it was red; secrets already redacted |
| `summary` | `pass` + `warned` (red under `warn`) + `blocked` (red under `block`) |
| `skipped` | the guards a selection left out, by name; absent when nothing was dropped |
| `decision` | `launch` or `blocked` |
| `exit` | the command's exit code. **Absent when `decision` is `blocked`** — that absence is the proof the command never ran |
| `measured` | the wrapped command's output weighed against `live.command`; absent only when `live.unweighed` excuses it |
| `startedAt` | present **only** with `-stamp` |

## effective check fields

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `attempts` | no | retries while the comparison disagrees; defaults to `1` |
| `retryDelayMs` | no | wait between attempts; defaults to `250` |
| `recorded` | yes | one reading: the setting as it was written down |
| `effective` | yes | one or more readings: the setting as it is in force; all must equal `recorded` |

## effective reading fields

| Field | Meaning |
|---|---|
| `label` | the name this source carries in the report |
| `map` | value mapping applied before the comparison; a value the map does not mention is compared as it came |
| ~~`equals`~~ / ~~`contains`~~ | **rejected here** — a reading has no expectation of its own; its expectation is the other readings |

## the adoption report fields

| Field | Meaning |
|---|---|
| `runners` | the gate scripts that call the engine; declared, not discovered |
| `invoke` | how this project spells a call — one capture group, the command name |
| `sources` / `exclude` | where `//x3:` directives are counted (default `**/*.go`) |
| `tests` | the files whose number is supposed to be falling |
| `token` | the ceiling under which a section is an example rather than an audit |
| `split` | the line count past which a configuration wants `include` |
| `exempt` | command → **reason**; a reason is required and a dead one is a finding |
| `policy` | `warn` (default), `block`, or an object keyed by finding code |

## adoption finding codes

| Code | Meaning |
|---|---|
| `command_unused` | no runner calls it and no exemption says why not |
| `section_token` | a section puts `token` or fewer rules (or directives) in force |
| `dead_exemption` | exempted, and run anyway |
| `tests_remain` | test files still stand where inline examples were meant to be |
| `dead_pin` | `update.pin` vouches for a release below `x3.min_version` |
| `config_one_file` | the configuration passed `split` lines and declares no `include` |
| `dead_policy` | `policy` excepts a code this run does not produce — always a block |

## testdb settings

| Field | Required | Meaning |
|---|---|---|
| `adminDsnEnv` | yes | **name** of the variable holding the maintenance DSN. Point it at a maintenance database, never at the template: a template with an open connection cannot be cloned |
| `driver` | no | defaults to `pgx`; an unregistered name is a configuration error (exit `2`) |
| `prefix` | no | defaults to `x3test_`, and it is the **authority boundary** — nothing outside it is listed or dropped, so an empty prefix is rejected |
| `template` | no | without it an empty database is created and the migration hook does the work |
| `dsnEnv` | no | the variable the new DSN is exported as; defaults to `X3_TESTDB_DSN` |
| `maxAgeMinutes` | no | age past which a leftover is stale; defaults to `120` |
| `migrate` | no | `command`, `args`, `timeoutMs`, run after creation with the DSN in the environment |

<!-- x3-dist version=v0.109.0 capabilities=bf57ad24b30d732a9c603eb0a18c1076ec3086ba0bfe52fe1ecfbec19053907a template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
