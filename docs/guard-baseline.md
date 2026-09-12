# The reds a live gate starts with

[The pages](INDEX.md) - [what x3 is](../README.md)

## The reds a live gate starts with

A project adopting `x3 guard` on a system that has been running for years does
not get a green first run, and a gate that is red on the day it arrives is a
gate somebody switches off. `guard` reads the same **finding baseline** the tree
commands read:

```
x3 guard -baseline baselines/guard.json -update-baseline   # freeze what fails now
x3 guard -- <command>                                      # only new failures speak
```

A held red stays **in the report** - `held: true`, counted in `summary.baselined`
- and it is printed as `HELD`. It is not removed, because `guards[]` is a list of
measurements rather than a list of findings, and a record taken out of it could
not be told from a guard that never ran. A held red does not block, so the
wrapped command launches; a red the baseline never saw still blocks, and a record
the run no longer produces is `dead_baseline`, exactly as everywhere else.

Warnings never enter, for the engine's own reason: an observation is not a debt.

**A refresh wraps no command.** `-update-baseline` together with a command is
refused with exit `2`. Either it would quietly not run what it was handed - and
the caller would believe the tests ran - or it would run a gate whose reds had
just been frozen, in the same breath. The dead-expectation question is not asked
of a refresh either: a run that wraps nothing neither meets `live.command` nor
contradicts it.

<!-- x3-dist version=v0.157.0 capabilities=0d6774f63a7d6ac7b8ab85705df08fe31d30b6f17310f404154c63efb3690e4d template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
