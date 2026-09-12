# One configuration, split across files

[The pages](INDEX.md) - [what x3 is](../README.md)

## Splitting the configuration

One `x3.json` is enough for a small repository and wrong for a large one: the
rules that belong to a module end up far from the module, so removing the module
leaves its rules behind, guarding nothing.

```json
{ "include": ["x3/*.json", "apps/*/x3.json"], "language": { "allowed": "en" } }
```

Every command reads the merged result, under one merge law that knows nothing
about any section's schema:

| Both sides are | Result |
|---|---|
| lists | the parts are **added**, root first, then the files in name order |
| objects | merged key by key, recursively |
| anything else | **refused** — the run stops and names both files and the key |

Nothing is silently overwritten: a setting that quietly loses to another file is
a setting whose author believes it is in force. Four more refusals, all
fail-closed: **a pattern that matches no file** (an `include` that does not work
is a set of rules nobody notices is missing); **a part that includes** (parts are
one level deep, so the whole configuration is readable from the root);
**discovery** (parts are declared, never found by scanning — a file dropped into
a folder must not add a rule nobody reviewed); and **a missing section**, still
an error for the command that needs it. Ordering is by file name, so the merged
configuration is the same on every run and machine. A project that does not
split pays nothing.

**A relative path written in the configuration is relative to the
configuration.** `baseline.dir` and `cache.dir` are resolved against the
directory of the root configuration file — the same base `include` patterns
already use, so the whole configuration has exactly one base and a part cannot
introduce a second. An absolute path is left alone, and a path given on the
command line (`-baseline`, `-cache`) is not touched at all: a path typed into a
shell belongs to that shell's directory.

Bound to the working directory instead, the same configuration opens a
different file depending on where it is run from: the baseline is not found and
a frozen debt turns every finding red again, or a new baseline is written into
the root of whatever tree the shell happened to be standing in. Both are silent.
A setting that quietly changes meaning is worse than a wrong answer, because a
wrong answer can be read.

## Pilot: a real `x3.json`

x3 is piloted inside a real production application. Nothing about that
application is encoded in the engine; what follows is its configuration file,
with generic names, as an example of what live guards are actually for. The
pilot's problem is the one every deployment has: a long test or migration run
that starts against a **wrong live environment** wastes an hour and can corrupt
state.

```json
{ "live": { "guards": [
    { "name": "schema-current", "kind": "sql", "policy": "block",
      "dsnEnv": "APP_DATABASE_URL",
      "query": "select max(version)::text from schema_migrations", "equals": "0117" },
    { "name": "catalog-engine-address", "kind": "sql", "policy": "block",
      "dsnEnv": "APP_DATABASE_URL",
      "query": "select engine_ref from capability_catalog where tier = 'standard'",
      "equals": "provider:engine-v3" },
    { "name": "provider-agent-permission", "kind": "http", "policy": "warn",
      "url": "https://api.provider.example/v1/agents/self", "status": 200,
      "headerEnv": { "Authorization": "PROVIDER_TOKEN" },
      "jsonPath": "/agent/permissions/0", "equals": "outbound" } ] } }
```

| Guard | The question | Why that policy |
|---|---|---|
| `schema-current` | is the migration ledger at the schema version this code expects? | `block` — an older schema produces failures that look like code bugs and are not |
| `catalog-engine-address` | does the catalog row for this tier still point at the engine the run assumes? | `block` — a stale row silently routes the whole run somewhere else |
| `provider-agent-permission` | does the provider still grant this agent the permission the run needs? | `warn` — an external provider having a bad minute should not stop local work, but nobody should discover it an hour in |

The gate is then one line, with no shell logic deciding anything:

```
x3 guard -config x3.json -report build/guards.json -- go test ./...
```

**The red that made this worth building.** When the token holds a rotated key the
endpoint answers `401`, and the run says so before anything starts. The token
itself appears nowhere — not in the config, not on stderr, not in the report.
Change that guard's policy to `block` and the same situation stops the run
instead of warning about it; that one word is the whole difference.

<!-- x3-dist version=v0.144.0 capabilities=c7a898f2ed45fa4107ced156c2151290063596e5205ab3e9fac74e3314f2808e template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
