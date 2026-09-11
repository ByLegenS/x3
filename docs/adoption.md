# How much of this engine actually runs

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 adoption`

**What it catches:** an engine half used and nobody noticing — a capability no
gate script runs, a section holding a single token rule, a checksum vouching for
a binary the version gate already refuses, and a pile of test files the inline
examples were supposed to replace.

```
x3 adoption [-config <file>] [-out <file>] [dir]
```

**It is a mirror, not a fence.** Findings default to `warn`, so the run stays
green and stops nobody: no project has to use every command, but a project that
is not using one should be able to see that. `"policy": "block"` gives it teeth,
and an [object](adoption-policy.md#one-known-debt-without-a-permanently-red-gate) gives them to
every code but the one you are working on.

**The list is the engine's own.** The commands measured are the entries of the
binary's dispatch table, read at run time. A copy of that list kept beside the
project would go stale the day a command is added, and a stale list reports its
own blindness as full coverage — the measure fails exactly where it is needed.

```json
{ "adoption": {
    "runners": ["check.ps1"],
    "invoke": ["[$]bin ['\"]?([a-z]+(?::[a-z]+)?)"],
    "tests": ["**/*_test.go"],
    "token": 2, "split": 300,
    "exempt": { "lang": "one language only; there is no prose to gate" },
    "policy": "warn" } }
```

See **the adoption report fields** in [REFERENCE.md](../REFERENCE.md#the-adoption-report-fields).

The spelling this project calls the engine by is declared too, and the line
between a real call and a sentence about one is drawn where the match begins —
[How the engine is called](adoption-invoke.md#how-the-engine-is-called).

### Green

```
  COMMANDS     [##################..]  20/22 run, 2 exempt
  SECTIONS     1 in the configuration, 0 barely in force
  TEST FILES   0 standing against 0 inline example(s)

  EXEMPT, WITH A REASON (2): lang - version

x3 adoption: 22 command(s) - 1 section(s) - 0 block, 0 warn, 0 allowed
```

The same tree with one finding excepted — green, and the debt still on the page:

```
  POLICY EXCEPTIONS (1) - each one named, with what it touched:
    tests_remain     -> warn   1 finding(s)  the migration is under way

ALLOW tests_remain
        1 test file(s) still stand against 0 inline example(s); the engine's
        claim is that the second replaces the first
x3 adoption: 23 command(s) - 1 section(s) - 0 block, 0 warn, 1 allowed
```

### Red

```
BLOCK docs section_token
        1 rule(s) in force in the configuration, and the token ceiling is 2;
        at this size a section is an example, not an audit
BLOCK tests_remain
        1 test file(s) still stand against 0 inline example(s); the engine's
        claim is that the second replaces the first
BLOCK dead_pin
        update.pin carries 1 checksum(s) below min_version v0.10.0 (v0.9.0);
        the version gate already refuses those binaries, so the record vouches
        for nobody
```

**"The section exists" is not a measure.** A section putting one rule in force
and a section putting thirty in force would otherwise read the same, and depth
is counted the way the engine counts it: `policy: "warn"` is not in force.

**But the shape of a section is not its weight.** Sections do not all carry
their work in the same place, and one counter asked of all three is always wrong
about the third:

| `kind` | Where the weight is | Weighed against `token` |
|---|---|---|
| `rules` | named rules declared in the configuration | yes |
| `directives` | `//x3:` directives in the tree carrying the section's name | yes |
| `settings` | one mechanism — a pattern list, an env variable, a ceiling | no |

Counting the *paths* inside a section instead calls a run command with three
settings and one nested object "two names, written, not working" — and says the
same of a section whose tree holds a thousand examples, three lines under the
`DIRECTIVES` count of those thousand. A settings section stays on the page and
the report names its `kind`; only the ceiling goes quiet, and nothing is lost: a
rule list that holds nothing is refused by its own command at load time.

**An exemption carries a reason, and a dead one speaks.** A command exempted and
then actually run is a finding; a name exempted that the engine does not have
stops the run with exit `2`, because an exemption for nothing hides the day the
name changed. The sections describing the engine's own workings — `x3`,
`update`, `baseline`, `cache`, and `adoption` itself — are counted in neither
direction: they put no check in force.

### Findings

See **adoption finding codes** in [REFERENCE.md](../REFERENCE.md#adoption-finding-codes).

<!-- x3-dist version=v0.110.0 capabilities=8e148dba0bfdd51abe71c339e3f254364ebfc4f3e0ba2c90a6e0d9518d73398d template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
