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

<!-- x3-dist version=v0.161.0 capabilities=dc3e9670ba9497349615241cc730ebb0d3de55e98954f4e755b74dcc5cc9ebff template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
