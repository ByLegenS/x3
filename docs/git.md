# How the engine reaches git

[The pages](INDEX.md) - [what x3 is](../README.md)

## How the engine reaches git

The engine reads history: what a change touched, which branch it sits on, whether
a tag was pushed, what version a binary should carry. All of that is one external
program, and a program a repository may simply not have.

### One door for git

**Exactly one file runs git.** Everything else asks it. The door is
`internal/changed.Git(root, args...)`: it sets `core.quotepath=false`, bounds the
call with a timeout, runs it unattended, and records the answer in the read
ledger — because git's answer arrives through no file, and a step that asked it
would otherwise be remembered as having read nothing.

Measured before this was true: git was invoked from four separate places. The
copies were born identical and did not stay identical — one carried
`core.quotepath=false` and another did not, so a path with a non-ASCII name was
quoted in one reading and plain in the other; one recorded its answer in the read
ledger and another did not, so a cached step could be green on a question it had
never re-asked. A dependency spread over four call sites cannot be counted,
cannot be given one answer for "is git even here", and cannot be switched off.

**The rule that keeps it that way is a syntax check, not a habit:**

```yaml
syntax:
  checks:
    - comments: exempt
      deny: 'exec\.Command[A-Za-z]*\([^)]*"git"|\[\]string\{"git"'
      exclude: ['**/*_test.go', internal/changed/changed.go]
      name: one-door-runs-git
      reason: git is run through exactly one door; a second call site is a second set of flags and a second answer to "is git even here"
      sources: ['**/*.go']
```

Two escapes are written into it on purpose. A **comment** naming the call is not
a call — the sentence explaining this rule has to name what it forbids. And a
**test** is not a second door: a test that plants its own repository runs
`git init` to *build the input*, which is the opposite of depending on git to
answer a question.

This is a rule any Go repository can write about any external program it would
rather depend on once: change the name, change the excluded file.

### A missing git that says so

**A missing git is an answer, and there is one of it.** When no binary named
`git` is on `PATH`, the door returns one named error and every caller surfaces it
unchanged:

```
git is not on this machine: no binary named git was found on PATH
```

Before, the same absence produced two different answers. `release.Tag` swallowed
the error and returned `"unreleased"` — silent, and indistinguishable from a real
repository that simply has no tag yet. The work-list rule that asks whether a
path was ever in history said something else and said it out loud. Both looked
correct on their own; together they made a machine with no git impossible to tell
apart from a repository with no tag. A checker that answers differently when its
input is missing can be green for the wrong reason.

**The absences stay apart.** A repository with git installed and no tag still
reads `unreleased`. Only the other case — no git at all — collapsed into one
sentence, because it is one fact:

| The tree | git installed | git not on `PATH` |
|---|---|---|
| tagless working copy | `unreleased` | the sentence above |

**The second caller has since left git altogether.** The work-list rule that asks
whether a path was ever in history no longer runs git at all: it reads the
engine's own record instead (*A path that was here once*, below), so the tree it
is pointed at no longer has to be a working copy — and the sentence
`... is not a git working tree` left the engine with it. What that tree is told
now is about its own settings: it declares no cache.

The control experiment runs the *same command on the same tree* twice, with
`env: {PATH: ''}` the only difference — so the answer cannot have come from the
tree.

### A version read from a file

**A repository that already writes its version down should not be asked git for
it.** `release.version` names the file:

```yaml
release:
  version: VERSION
```

With that line the number comes from the file, and the tag becomes a
**confirmation** rather than a source. The chain is four steps and none of them
is silent:

| The tree | The answer |
|---|---|
| file declared, git absent | the file, and the note says no tag confirms it |
| file declared, tree sits past its tag | the file, and the note names what the tree sits at |
| file declared, tree sits **on** a tag that disagrees | **red**, naming both values |
| nothing declared | the tag, exactly as before |
| nothing declared and no git | no answer — `there is no source for this tree's version` |

A tag speaks only for the commit it names. One commit later the tree is something
else and that tag confirms nothing, which is why bumping the file before tagging —
the ordinary order of work — is not a contradiction. On the **same** commit a
disagreement is one, and choosing a side quietly would hide it:

```
VERSION says v1.2.0, the tag says v1.1.9; a file and a tag on the SAME commit
cannot disagree - move the tag or fix the file
```

**The second effect is the one that pays for the work.** `git describe` was being
called with `--dirty`, so the version string changed with the state of the working
tree. That string is embedded in the built binary, and the binary's digest is part
of the gate's cache key — so an edit anywhere moved the key and emptied the cache
for every step, twice, on the way there and back. A version read from a file
carries no `-dirty` suffix and does not move.

`x3 release -dry` answers the question on its own, building nothing:

```
x3 release: v1.2.0, read from VERSION; confirmed by the tag - nothing is built
```

The **version file control experiment** runs four arms on one tree: the file with
`env: {PATH: ''}`, the file with git installed, no declaration with git, and no
declaration without it. The confirmation rule itself is a pure function measured
by inline examples, because the one input a fixture tree cannot be given is a
git tag — planting one would mean running git to test not running git.

### A publish only runs where it is declared

`x3 published` asks git for local and remote tags. That is the one git call that
stays legitimate — but only where a project actually publishes through tags. A
project that does not needs the command to **say it has nothing to do**, not to
fail:

```
x3 published: x3.yaml: this project declares no release; nothing to check
```

Exit `0`. Every git call in that command is born after the configuration is
read, so where nothing is declared git is never asked — measurable, because the
same command with `env: {PATH: ''}` prints the same sentence and never names the
absence of git.

A declaration that **is** written is still measured as before, and one that is
written but cannot be measured is still exit `2` (`claim wants file`). The three
states stay apart: nothing declared is not the same as declared-and-broken, and
neither is the same as green.

<!-- x3-dist version=v0.279.0 capabilities=2d4dbaa4fd0947e09be46fbdc475aa6d2c15d07d65a13b06f643f87f8c988536 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
