# x3 documentation

One page per capability: the directive line, what it catches, a red and a green
example, and the settings it reads. Every page is generated from the engine's
own capability document in the same run as the binaries, so a page cannot
describe a version that does not exist.

[What x3 is](../README.md) - [the lookup tables](../REFERENCE.md)

**Current version: `v0.74.0`**

| Page | What it covers |
|---|---|
| [Directives in the source](scan.md) | the `//x3:` dictionary, the scopes a directive may sit in, the report and its expectations |
| [Where a directive may sit](layout.md) | the placement the Go formatter writes, measured by the engine itself, so that a formatting run cannot move a directive behind your back |
| [Where a pattern binds](patterns.md) | how `^` and `$` are read against a file, and where a line ends |
| [Inline examples that run](case.md) | an example that calls the declaration it sits on, with a state given to it and an aspect of the result asserted |
| [What an example may name, and what a run says](case-findings.md) | the pool of importable names and the scope it is measured in, the finding codes, the settings and the report |
| [One language outside comments](lang.md) | the dictionary run in reverse: the allowed language, and every token outside it |
| [The shape of the project](arch.md) | the import graph and nine further rule kinds, against the components a project declares |
| [The two sets a rule compares](sets.md) | the consistency kind, the extractors that read each side, the escape hatches they carry, and the engine's own roster |
| [Lists that may only shrink](freeze.md) | a measured set or a number frozen to a baseline that growth turns red |
| [The exported API, which may only grow](surface.md) | a removal or a changed signature is red, and the finding names who breaks |
| [Today's findings, frozen](baseline.md) | adopting a gate on a tree that is not clean yet, without a thousand reds |
| [Changes that must not travel alone](docs.md) | a change under one path that requires a change under another in the same diff |
| [Credentials in the source](secrets.md) | credential formats in any text file, masked in the report that names them |
| [The comment diet](comments.md) | comment blocks over a limit, with the ratio to code kept as a warning |
| [Open work, measured](boxes.md) | every box against the criteria that would prove it done, in both directions |
| [Before a name is removed](holds.md) | which criteria hold a name that is about to be deleted, including the selector patterns a search cannot find |
| [Which hold is really a hold](holds-weight.md) | the place a criterion looks at, weighed against the file being asked, and the record the engine refuses to decide |
| [A name that lives outside the list](holds-elsewhere.md) | the gate scripts and workflow files a name is called from, declared because no engine can guess them |
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
| [Gaps in what a run reaches](gaps-outside.md) | the bounds that begin where the source tree ends: the release it pulls, the toolchain it mutates through, the traffic it records, the live world it asks, and the database it borrows |
| [Control experiments, and the documentation gate](experiments.md) | every capability proven able to go red, and the gate that keeps these pages current |

<!-- x3-dist version=v0.74.0 capabilities=c65cff0a74d72c2baca34ce2677e8892c28679fc77a5e1f4edd66aec4d4138c3 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
