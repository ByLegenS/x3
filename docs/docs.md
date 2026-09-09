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
docs: none - wording of one stderr line; the capabilities document does not quote it
```

The marker is `docs: none` unless the rule says otherwise; in `head` scope it
lives in the commit body, in `working` scope it is passed with `-reason`. **The
marker alone is red** — `exemption_without_reason` is a separate code from the
missing change, because an exemption nobody had to justify becomes the only path
within a month.

<!-- x3-dist version=v0.61.0 capabilities=1f6808cc172ef8b2b2b963daa8347ee49cfc2c182847ac8d118aebcec0a9e1ea template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
