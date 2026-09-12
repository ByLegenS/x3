# guard reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.159.0`**

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
| `equals` / `contains` / `notContains` / `sameRowsAs` | one of them | the first three say what the observed value must, or must not, be; the last compares row **sets** against a second connection (§ [Two schemas, one question](docs/guard-rows.md#two-schemas-one-question)) |

## guard fields for kind http

| Field | Required | Meaning |
|---|---|---|
| `url` | yes | the address; the request is always a `GET` |
| `status` | yes | the expected status code |
| `headerEnv` | no | header name → **name** of the variable holding its value |
| `jsonPath` | no | an RFC 6901 JSON Pointer into the body; without it the whole body is the value |
| `equals` / `contains` / `notContains` | no | with only `status`, the status code alone is the assertion |

## guard fields for kind exec

| Field | Required | Meaning |
|---|---|---|
| `command` | yes | executable to run |
| `args` | no | its arguments |
| `equals` / `contains` / `notContains` | no | what its trimmed stdout must, or must not, say; without any of them, **exit code 0** is the assertion |

## guard fields for kind steps

| Field | Required | Meaning |
|---|---|---|
| `steps` | yes | run **in order**; the first that does not hold ends the trial and names itself |
| `workspace` | no | a temporary working area: `copy` (required within it), `remove`, `write` |
| `equals` / `contains` / `notContains` | no | what the **last** step's output must, or must not, say; without any of them, every step holding is the assertion |

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
| `measured` | the wrapped command's output weighed against `live.command`; absent only when `live.unweighed` excuses it, or when the run wraps nothing (`live.unwrapped`) |
| `measured.status` `error` | the run could not be measured: the copy outgrew its cap, or the output announced the runner had nothing to run (`live.blind`) |
| `guards[].held` | this red is in the baseline: measured, failing, and frozen for today |
| `summary.baselined` | how many reds the baseline held; they are neither passes nor blocks |
| `startedAt` | present **only** with `-stamp` |

<!-- x3-dist version=v0.159.0 capabilities=7f149416d4d1ff326e5dfdfc03d02ef69f7f13faf78170206823bb1ba5536ceb template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
