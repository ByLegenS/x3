# The setting on paper against the setting in force

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 guard:effective`

**What it catches:** a setting whose two lives have drifted apart. One is the
**record** — a row in a table, a field in a remote endpoint. The other is what is
**in force** — the value the running process actually loaded, and the value the
provider actually applies. They drift quietly, because every side is internally
consistent.

```
x3 guard:effective [-config <file>] [-out <file>] [-stamp]
```

Unlike `x3 guard` this launches nothing, so stdout is free for the report.

| Checks | Exit | What it means |
|---|---|---|
| every source agrees | `0` | the setting on paper is the setting in force |
| divergent, all `policy: warn` | `0` | printed as `WARN`, the run is not stopped |
| at least one divergent `policy: block` | `1` | the record and the world disagree |
| a source could not be read at all | as above | **red** — an unknown answer is not an answer |

## Effective checks in `x3.json`

Built out of the same three source kinds; what changes is the *role* a source
plays — one is the record, the rest are the world.

```json
{ "effective": { "checks": [
    { "name": "assistant-model", "policy": "block",
      "attempts": 3, "retryDelayMs": 500,
      "recorded": { "label": "database", "kind": "sql", "dsnEnv": "APP_DSN",
                    "query": "select model from settings where id = 1" },
      "effective": [
        { "label": "process", "kind": "http", "url": "${APP_BASE}/internal/settings",
          "status": 200, "jsonPath": "/model" },
        { "label": "provider", "kind": "http", "urlEnv": "PROVIDER_SETTINGS_URL",
          "status": 200, "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
          "jsonPath": "/model", "map": { "engine-2-2026-01-31": "engine-2" } } ] } ] } }
```

Validation is strict and up front, the same fail-closed rules the `live` section
has.

### Fields a check has

See **effective check fields** in [REFERENCE.md](../REFERENCE.md#effective-check-fields).

**Retries exist because the world lags the record** — a process reloads a moment
after the row changes. One attempt is the default precisely so a retry is a
deliberate statement about how long the lag may be, never a way to wait out a
red.

### Fields a reading has

A reading is a `sql`, `http` or `exec` source, and every field documented under
[Live guards in `x3.json`](guard.md#live-guards-in-x3json) applies unchanged. Two are
added and two are **not allowed**:

See **effective reading fields** in [REFERENCE.md](../REFERENCE.md#effective-reading-fields).

`map` is what makes two spellings of the same setting comparable. A value the
map does not cover is *not* an error — it goes into the comparison unchanged, so
an incomplete mapping produces an explainable red, never a false green, and the
report keeps the raw value next to the mapped one.

## The effective report

```json
{ "version": 1, "checks": [
    { "name": "platform", "policy": "block", "status": "fail", "attempts": 1,
      "recorded": { "label": "record", "kind": "exec", "value": "windows" },
      "effective": [ { "label": "world", "kind": "exec", "value": "amd64" } ],
      "detail": "record says \"windows\", but world says \"amd64\"" } ],
  "summary": { "pass": 0, "warned": 0, "blocked": 1 } }
```

`status` is `pass`, `fail` (the sources disagreed) or `error` (a source could not
be read); both non-`pass` values are red. `attempts` says how many the answer
actually needed — a `2` here says the world was late, not wrong. Each reading
carries its `label`, `kind`, compared `value`, the `raw` value when a mapping
changed it, and `error` when it could not be read. `startedAt` appears **only**
with `-stamp`.

Secrets follow the guards' law, and `TestEffectiveSecretNeverLeaves` holds it
for this report specifically.

<!-- x3-dist version=v0.111.0 capabilities=f088e41a540b9aad743af8a7320405094bc2ed69445df36ccf6c69eaf20d959e template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
