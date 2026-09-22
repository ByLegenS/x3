# A configuration that cannot call git

[The pages](INDEX.md) - [what x3 is](../README.md)

## A configuration that cannot call git

Taking a dependency out of the code does not take it out of the gate. One
`run:` line puts it back, and nobody sees it arrive. A project can say so:

```yaml
gate:
  forbid:
    - git
  steps:
    - name: the tag this binary carries
      trials:
        - run: go version -m {bin}
          say: read from the binary, not from the repository
          want: 0
```

A step whose **command position** is a forbidden program refuses the whole
settings file, by name, before a single step runs:

```
x3.yaml: "gate" section: step "a step that calls git" runs "git", and this
configuration forbids it: a dependency the code no longer carries can come
back through a single run: line, and nobody would see it arrive
```

| | |
|---|---|
| Opt-in | a project that writes no `forbid:` is not touched by one |
| Command position only | `grep -c "git tag" x3.yaml` names git and calls grep; it passes |
| The program, not the spelling | `git`, `/usr/bin/git` and `Git.EXE` are one name |
| After multiplication | a step `{each}` produced, and a trial that borrowed the step's default `run:`, are real calls and are read |

`state:` commands are read the same way. What a step's command does *further
down* — a shell it starts, a script it hands over to — is outside this
reading, and the rule says so rather than pretending otherwise.

<!-- x3-dist version=v0.288.0 capabilities=b7fca59edbf65759483bfdca34f14aeafbe84562986ae2f4e8a4b427249da8fb template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
