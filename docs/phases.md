# Where a run's wall clock goes

[The pages](INDEX.md) - [what x3 is](../README.md)

## Where a run's wall clock goes

A total tells you a run was slow. It does not tell you which part was slow, and
a gate that runs *no step at all* can still be slow — in which case the total is
measuring everything except the work.

```
x3 gate -region defter -phases
```

prints one line beside the summary, on stderr:

```
-- phases: startup 591ms, config 590ms, workers 0ms, cache-read 5ms, baseline 1ms,
   regions 7ms, build 0ms, sums 274ms, scopes 97ms, steps 56ms, cache-write 11ms,
   reach 0ms, rest 0ms (total 1.64s); 2 configuration read(s), 3 tree walk(s)
```

`startup` is measured from the moment the process reaches the engine, not from
the moment the command starts: the binary loading, an alias being resolved and
the flags being parsed all happen before any command can start a clock, and a
clock that starts inside the command reports nothing about them. `rest` is the
wall clock no phase claimed — a growing `rest` says the line itself has a hole
in it.

**Two of those numbers are not seconds, and that is the point.** How long a
phase takes depends on the machine, the disk and the size of the repository, so
it cannot be tied to a threshold. How many times the configuration was parsed
from scratch, and how many times the tree was walked, are properties of the
*mechanism*: a second full read inside one run means a path that does not
remember its answer, and that is the same defect on every machine.

### Work a run remembers instead of repeating

Two memos are opened explicitly, by the command, and closed when it ends.

`source.Memoize` remembers the *listing* of a tree walk — never the contents,
because handing the same bytes to two rules would let one change what the other
reads. `config.Memoize` remembers a *configuration reading*: the root file, its
ancestors and every part it declares, merged, with the profiles expanded. The
key carries the overlay a run put on top, because the same file under a
different overlay is a different configuration.

Neither is opened by a command that *writes* what it then reads. A command that
edits a configuration and reads it back would be handed the remembered older
copy, and a stale configuration is a silent green.

Measured in the pilot on 2026-09-21, on a gate run that ran zero steps:

| | before | after |
|---|---:|---:|
| `x3 gate -region <one> -phases` wall | 16.5 s | 1.69 s |
| `-- phases` total | 15.31 s | 1.64 s |
| `reach` (verb coverage line) | 6.25 s | 0 ms |
| `regions` (region resolution) | 3.72 s | 7 ms |
| `x3 gate -regions` (prints a list, runs nothing) | 11.3 s | 1.21 s |

Nothing about the work changed; the same answers are computed once instead of
five times. The region resolution used to walk the tree and re-parse the
configuration once per question — `Known`, `Spread`, `Reach`, `Shadowed`, and
once more per step for the declaration check — while the entry gates parsed the
whole configuration twice before the command even began.

<!-- x3-dist version=v0.286.0 capabilities=99636eca84d9bd691f9aeace2584836190f9cd88dde014db1ac9207b05ef7db0 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
