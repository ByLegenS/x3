# The directory a command criterion runs in

[The pages](INDEX.md) - [what x3 is](../README.md)

## The directory a command criterion runs in

Every other criterion kind resolves against the **tree the run was handed**:
`sources`, a `file` path, a `gone` path. A `command` cannot — it is not a path in
the tree, it is a command of the *project*, and a work list written inside
`docs/` may name a test run that covers the whole repository. Anchoring it to
the tree was tried and measured wrong: five control experiments in this
engine's own gate went red, all of them lists in a subdirectory naming commands
about the module above them.

So the command's directory ran wherever the **caller** stood, and that made
almost half a measurement a property of who started x3. Measured on a real
repository, same tree, same binary, **absolute** `-config` and an **absolute**
root on both runs:

| Run from | Findings | Exit |
|---|---|---|
| the project root | 0 | `0` |
| another directory | **24** | `1` |

Twenty-four false reds, and the report said `0 could not be measured` — the
runner had started, found no module where it stood, and its non-zero exit was
read as "the criterion does not hold".

The engine cannot know which of the two a project meant. What it can do is let
the project say, and refuse to be silent when it has not:

```json
"boxes": { "file": "docs/OPEN-WORK.json", "workdir": "." }
```

`workdir` is written **against the tree**, never as a machine path (an absolute
one is exit `2`), so a run is reproducible from any directory — and with an
absolute root it is reproducible from any machine. Every report says where the
commands actually ran, declared or not:

| `workdir` in the report | What it means |
|---|---|
| `.` | the commands ran at the root of the measured tree |
| any other relative path | that is where they ran — with no declaration it is **the caller's directory**, and it changes from caller to caller |
| `(another volume)` / `?` | the caller stood somewhere with no path back to the root, or the directory could not be read |

Two runs of one tree that measured in different places now **differ in their
bytes**, and the field that differs names the reason. A run that declares
`workdir` produces the same bytes from anywhere.

<!-- x3-dist version=v0.219.0 capabilities=d43625b748b998cf7dfd9a15e7cc521aa49e38fea68caca749b54b42fcc4254e template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
