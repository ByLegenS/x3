# A skipped test is not a green one

[The pages](INDEX.md) - [what x3 is](../README.md)

## Counting what was skipped

**What it catches:** a unit whose tests all skip. The runner says `ok`, the run
is green, and the green is cached - so it never runs again either. A run that
did not measure cannot be told from one that measured and found nothing.

The shape most often seen: a test that needs a database, an environment
variable that is not set, and a `t.Skip` at the top. On the machine that has
the database the suite measures something; everywhere else it measures nothing
and says the same word.

```json
{ "test": {
    "skips": { "pattern": "^\s*--- SKIP: (\S+)", "name": 1,
               "policy": "block", "max": 0 } } }
```

| Field | What it is |
|---|---|
| `pattern` | the line in which the runner says a test was skipped |
| `name` | the capture group holding the skipped test's name |
| `policy` | `warn` (default) or `block` |
| `max` | in `block`, how many skips are allowed; 0 when not written |
| `listed` | how many names the finding carries; 5 when not written |

The count is printed on **every** run - `x3 test: N test(s) skipped: ...` - and
the report carries `skipped`. A finding is raised only under `block`: a silent
warning is not a warning, but a declaration cannot redden an existing tree by
itself. Under `block` the run is red **and nothing is cached**, including the
units that passed: a green the run did not measure must not be allowed to skip
its next run too.

⛔ **A runner that does not print the skip line reports none.** `go test` names
skipped tests only in its verbose output, so a `skips` pattern under a quiet
runner counts zero forever. There is no dead-declaration finding for this, and
the reason is that the two cases cannot be told apart: a tree with no skips at
all is a legitimate result and a gate that called it a fault would redden every
healthy tree. Prove the reading once by making a test skip on purpose and
watching the count move.

<!-- x3-dist version=v0.127.0 capabilities=2801084864972251f16605de3f015cce95cb887510f91b29286bf29571907c96 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
