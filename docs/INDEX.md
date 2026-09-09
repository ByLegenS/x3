# x3 documentation

One page per capability: the directive line, what it catches, a red and a green
example, and the settings it reads. Every page is generated from the engine's
own capability document in the same run as the binaries, so a page cannot
describe a version that does not exist.

[What x3 is](../README.md) - [the lookup tables](../REFERENCE.md)

**Current version: `v0.65.0`**

| Page | What it covers |
|---|---|
| [Directives in the source](scan.md) | the `//x3:` dictionary, the scopes a directive may sit in, the report and its expectations |
| [Inline examples that run](case.md) | an example that calls the declaration it sits on, with a state given to it and an aspect of the result asserted |
| [One language outside comments](lang.md) | the dictionary run in reverse: the allowed language, and every token outside it |
| [The shape of the project](arch.md) | the import graph and nine further rule kinds, against the components a project declares |
| [Lists that may only shrink](freeze.md) | a measured set or a number frozen to a baseline that growth turns red |
| [The exported API, which may only grow](surface.md) | a removal or a changed signature is red, and the finding names who breaks |
| [Today's findings, frozen](baseline.md) | adopting a gate on a tree that is not clean yet, without a thousand reds |
| [Changes that must not travel alone](docs.md) | a change under one path that requires a change under another in the same diff |
| [Credentials in the source](secrets.md) | credential formats in any text file, masked in the report that names them |
| [The comment diet](comments.md) | comment blocks over a limit, with the ratio to code kept as a warning |
| [Open work, measured](boxes.md) | every box against the criteria that would prove it done, in both directions |
| [Files no compiler reads](syntax.md) | JSON, YAML, TOML, SQL and the rest, parsed anyway; a missing parser is red |
| [A change that stays in its lane](scope.md) | a declared lane, and the change that enters it and also reaches outside |
| [Only the tests a change can reach](test.md) | the unit graph, the cache, and what the measurement honestly shows |
| [The test you forgot to write](mutate.md) | the code broken on purpose, and the behaviour no test noticed |
| [What a mutation run breaks, and what it says](mutate-findings.md) | the operators, the text mutations, the fail-closed codes, and the findings a run names |
| [Traffic, written down](record.md) | a run of the application recorded, redacted before it reaches the disk |
| [The recording, sent again](replay.md) | compared field by field, with what is allowed to differ written down |
| [The live world, before the command](guard.md) | `sql`, `http`, `exec` and multi-step trials, each one a warning or a block |
| [The setting on paper against the setting in force](effective.md) | a recorded value compared with the value the running system actually uses |
| [How much of this engine actually runs](adoption.md) | the project measured against the engine's own command table |
| [The version, and how it updates itself](update.md) | the embedded tag, the self-update, the pinned checksum and the minimum version gate |
| [A fresh database for this run](testdb.md) | a template cloned per run, migrated, dropped, and the leftovers collected |
| [Speed, the cache, and what a run leaves behind](speed.md) | measured timings, the incremental cache, and the files the engine reads back |
| [One configuration, split across files](configuration.md) | `include`, how lists and objects merge, and a real `x3.json` from a live project |
| [Releases, and calling the engine from another project](releases.md) | reproducible builds, and the gate script that pins a tag and a checksum |
| [Gaps we know about](gaps.md) | what is not built, said plainly, next to what is |
| [Control experiments, and the documentation gate](experiments.md) | every capability proven able to go red, and the gate that keeps these pages current |

<!-- x3-dist version=v0.65.0 capabilities=30f2211593ea62df95d9a529b650866118e447096978014873bc8ee488525447 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
