# A baseline belongs to the root it measured

[The pages](INDEX.md) - [what x3 is](../README.md)

## A baseline belongs to the root it measured

**What it catches:** a run pointed at a subtree refreshing a baseline
written against the whole tree.

Every path in a baseline, and every pattern a segment owns, is written against
the root the run measured. Point the same command at a subtree and none of them
mean the same thing any more: the recorded paths no longer match the ones the
run produces, and the engine's own law — *what the baseline holds and the run no
longer finds is debt that was paid* — turns that mismatch into a refresh that
**deletes the whole list**. Measured: a run narrowed to one region of a split
baseline reported every record of every other region as `dead_baseline`, emptied
all four files, and exited `0`.

So the root is recorded next to the command, and read back the same way:

```json
{ "version": 1, "command": "comments", "root": ".", "count": 12, "findings": [] }
```

A run that measures a different root is refused before it judges anything
(exit `2`), and the files are not touched. The law is the **declared**
baseline's: its place and its ground both come from the configuration, and a
configuration describes one project rooted at its own directory. A file named
with `-baseline` is exempt, because that flag is a caller pointing somewhere
deliberately — holding one baseline against several trees on purpose is a
measurement, not an accident. The root is written relative to the
configuration file's own directory, so the same tree yields the same word from
any shell, and no local path is ever recorded in a file the project commits.

A baseline written before this was recorded carries no root. It is still read —
an older file is not a broken one — and the root is stamped the first time the
baseline is refreshed. Until that refresh, the narrowed run above is still
possible, so refresh once from the root the gate runs at.

<!-- x3-dist version=v0.127.0 capabilities=2801084864972251f16605de3f015cce95cb887510f91b29286bf29571907c96 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
