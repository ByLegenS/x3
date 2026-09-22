# One configuration, split across files

[The pages](INDEX.md) - [what x3 is](../README.md)

## Splitting the configuration

One `x3.yaml` is enough for a small repository and wrong for a large one: the
rules that belong to a module end up far from the module, so removing the module
leaves its rules behind, guarding nothing.

```yaml
{ include: ["x3/*.yaml", "apps/*/x3.yaml"], language: { allowed: en } }
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
fail-closed: **a pattern that matches no file**; **a part that includes** (parts
are one level deep, so the whole configuration is readable from the root);
**discovery** (parts are declared, never found by scanning — a file dropped into
a folder must not add a rule nobody reviewed); and **a missing section**, still
an error for the command that needs it. Ordering is by file name, so the merged
configuration is the same on every run and machine; a project that does not split
pays nothing.

**Settings are written in YAML, and nothing is converted.** Every file the
configuration is read from — the root, a part named by `include`, an overlay
ledger — is YAML, and the engine carries one parser. A second written form puts
a step between what a person wrote and what the engine read, and everything that
step drops (a comment, the order of keys, an empty value) is part of what the
person meant. With no `-config` a command reads `x3.yaml`, then `x3.yml`.

YAML pays where settings are read by people: comments, a multi-line form for a
pattern JSON would make you escape, and the empty value an overlay deletes a
field with. It also reads a word as a type — `no`, `yes`, `on` and `~` are not
the strings they look like — so a value that must stay text is quoted.

**A relative path written in the configuration is relative to the
configuration.** `baseline.dir` and `cache.dir` are resolved against the
directory of the root configuration file — the same base `include` patterns
already use, so the whole configuration has exactly one base and a part cannot
introduce a second. An absolute path is left alone, and a path given on the
command line (`-baseline`, `-cache`) is not touched at all: a path typed into a
shell belongs to that shell's directory.

Bound to the working directory instead, the same configuration opens a different
file depending on where it is run from: the baseline is not found and a frozen
debt turns every finding red again, or a new baseline is written into the root
of whatever tree the shell stood in. Both are silent, and a setting that quietly
changes meaning is worse than a wrong answer, because a wrong answer can be read.

## A configuration that cannot be read says so

The configuration is resolved **before the verb**, and a file that does not
parse ends the run at once with the parser's own place:

```
$ x3 store
x3: x3.yaml cannot be read: yaml: line 7: mapping values are not allowed in this context
$ echo $?
2
```

Exit is `2`, not `1`: nothing was measured. Before this, every reader of a broken
configuration withdrew in silence — the alias table found no alias, the region
list found no region — and what was left of the answer was `unknown command:
store ... (regions here: none is declared)`. Each silence was right on its own;
together they sent the reader looking for a declaration that was not missing.
Measured on the pilot: a single plain scalar carrying `: ` produced exactly that
message, and the real line was printed only by an unrelated command.

A configuration that is **absent** is not an error — commands run in trees that
declare none. One that exists and cannot be opened is reported, because there
the file is there and the answer is not.

## Pilot: a real `x3.yaml`

x3 is piloted inside a real production application. Nothing about that
application is encoded in the engine; what follows is its configuration file,
with generic names, as an example of what live guards are actually for. The
pilot's problem is the one every deployment has: a long test or migration run
that starts against a **wrong live environment** wastes an hour and can corrupt state.

```yaml
live:
  guards:
    - { name: schema-current, kind: sql, policy: block, dsnEnv: APP_DATABASE_URL,
        query: "select max(version)::text from schema_migrations", equals: "0117" }
    - { name: catalog-engine-address, kind: sql, policy: block, dsnEnv: APP_DATABASE_URL,
        query: "select engine_ref from capability_catalog where tier = 'standard'",
        equals: "provider:engine-v3" }
    - { name: provider-agent-permission, kind: http, policy: warn, status: 200,
        url: "https://api.provider.example/v1/agents/self",
        headerEnv: { Authorization: PROVIDER_TOKEN },
        jsonPath: /agent/permissions/0, equals: outbound }
```

| Guard | The question | Why that policy |
|---|---|---|
| `schema-current` | is the migration ledger at the schema version this code expects? | `block` — an older schema produces failures that look like code bugs and are not |
| `catalog-engine-address` | does the catalog row for this tier still point at the engine the run assumes? | `block` — a stale row silently routes the whole run somewhere else |
| `provider-agent-permission` | does the provider still grant this agent the permission the run needs? | `warn` — an external provider having a bad minute should not stop local work, but nobody should discover it an hour in |

The gate is then one line, with no shell logic deciding anything:

```
x3 guard -config x3.yaml -report build/guards.json -- go test ./...
```

**The red that made this worth building.** When the token holds a rotated key the
endpoint answers `401`, and the run says so before anything starts. The token
itself appears nowhere — not in the config, not on stderr, not in the report.
Change that guard's policy to `block` and the same situation stops the run
instead of warning about it; that one word is the whole difference.

<!-- x3-dist version=v0.289.0 capabilities=e2b1c902dafbfc124d29f232a1f3e1807c6357f33259deee7c9a3811eaf626d6 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
