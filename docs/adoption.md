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

See **the adoption report fields** in the [adoption reference](adoption-reference.md#the-adoption-report-fields).

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

See **adoption finding codes** in the [adoption reference](adoption-reference.md#adoption-finding-codes).

**A settings file that is still JSON without a reason.** Once any part of the
configuration is written in TOML, the ones left in JSON are asked why. The
question is not *"are there two formats"* — one of them may have to stay:
TOML has no `null`, so a file that deletes a field by writing one cannot be
anything but JSON, and the count says how many are held there for that reason.
What it reports is the rest: the files a half-finished migration left behind.
The null is looked for in the VALUE, not in the text — the word inside a string
is not a deletion, and a measure that read the text would sentence that file to
JSON forever.


**A rule may say what it is for.** Any named rule in the settings takes an
optional `why` — one sentence, next to the rule, that `x3 arch -out` carries
into the report:

```toml
[[arch.rules]]
name = 'no-handler-reaches-the-database'
why = 'a handler that queries directly cannot be reused behind a queue'
```

A comment cannot do this job. A comment stays in the file: the engine does not
know it, no report carries it, and nothing can count it. `why` is data, and
being countable is the whole point — `adoption` reports `rules_without_why`,
so a rulebook can watch its own reasons the way it watches anything else.

The field is **optional on purpose, and the pressure comes from the count**.
Made mandatory, a repository with sixty rules writes sixty sentences in one
sitting and most of them are invented. An invented reason is worse than none:
it stops the next person from removing a rule, for a reason that was never
true. Written as a count, the number goes into the report, the project sets the
policy it wants, and the debt comes down as reasons are actually written.

| `dead_policy` | `policy` excepts a code this run does not produce — always a block |

<!-- x3-dist version=v0.273.0 capabilities=a635c171f924344b5f1ed4f6642936fdd96ee6c6e45326f5da82cd46c60ed4a1 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
