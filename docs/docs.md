# Changes that must not travel alone

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 docs`

**What it catches:** a change that travelled alone — code without its
documentation, a migration without its release note.

```
x3 docs [-config <file>] [-out <file>] [-scope auto|working|head] [-reason <text>] [dir]
```

```json
{ "docs": { "rules": [
    { "name": "code-changes-carry-documentation",
      "when": ["internal/**", "cmd/**"], "then": ["docs/**"] } ] } }
```

That is this repository's own section, and its own gate runs this command
against itself on every `check.ps1`.

### What counts as changed

| `-scope` | Reads |
|---|---|
| `auto` (default) | the working tree when it is dirty, the last commit when it is clean |
| `working` | `git status`, untracked files included; a rename counts as its new name |
| `head` | the files in `HEAD`, with the commit body as the place a reason may live |

The default is not a convenience: checking the last commit while the tree is
dirty would count documentation that has not been written yet. **A directory
that is not a repository is red**, not green.

### Exemption, with a reason

```
docs: none (code-changes-carry-documentation) - wording of one stderr line; the capabilities document does not quote it
```

The marker **names the rule it excuses**: `docs: none (<rule>)` unless the rule
says otherwise (`exempt`). In `head` scope it lives in the commit body, in
`working` scope it is passed with `-reason`. **The marker alone is red** —
`exemption_without_reason` is a separate code from the missing change, because an
exemption nobody had to justify becomes the only path within a month.

The rule's name is in the marker because a commit body answers **one** gate. The
person writing it saw one rule turn red and wrote a reason for that rule; a
shared marker takes that one sentence and silences every other rule as well —
including rules the writer never saw. Measured in this repository: with a shared
marker, the `docs gate` step's own control experiment (a rule whose counterpart
directory cannot exist, which must exit `1`) exited `0` instead. The exemption
had excused the experiment.

<!-- x3-dist version=v0.118.0 capabilities=70f1f255387ba0e0b3e37f307a3ec5c030d0ad4c3514eac7aef65a5cead99c77 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
