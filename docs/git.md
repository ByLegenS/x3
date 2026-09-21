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

**The two absences stay two.** A repository with git installed and no tag still
reads `unreleased`; a directory that is not a working copy is still told it is
not a working copy. Only the third case — no git at all — collapsed into one
sentence, because it is one fact:

| The tree | git installed | git not on `PATH` |
|---|---|---|
| tagless working copy | `unreleased` | the sentence above |
| not a working copy | `... is not a git working tree` | the sentence above |

The control experiment runs the *same command on the same tree* twice, with
`env: {PATH: ''}` the only difference — so the answer cannot have come from the
tree.

<!-- x3-dist version=v0.275.0 capabilities=a186d7b62d4c0d1683260322b5bc10fbb601b9da7bb6918567062dbf9145dd7c template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
