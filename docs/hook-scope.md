# A scope an agent cannot leave

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 hook scope`

An agent asked to write about one part of a repository should not be able to read
the rest of it: what it cannot see, it cannot leak. `x3 hook scope` is a
`PreToolUse` hook for Claude Code. It runs before every tool call, and when the
session is **scoped** it refuses any call that reaches a path the scope file does
not allow. A gate measures what was written; this hook stops the read before it
happens, so the two are separate nets.

```json
{ "hooks": { "PreToolUse": [ { "matcher": "Read|Edit|Write|MultiEdit|NotebookEdit|Glob|Grep|Bash|PowerShell",
  "hooks": [ { "type": "command",
    "command": "x3 hook scope -file docs/writer-scope.yaml -branch \"work/docs*\"" } ] } ] } }
```

```yaml
# docs/writer-scope.yaml - globs from the repository root
allow: [docs/**, web/pages/**, locales/en.json]
commands: [x3 boxes, x3 gate -region docs, git status, git add docs/, git commit]
agents: [writer]
```

**When the scope holds.** The hook reads the branch from the `.git/HEAD` of the
call's `cwd` (a worktree's `gitdir:` file is followed; git is never started). The
scope holds when the branch matches one of the `-branch` globs (comma separated),
or when the hook input carries an `agent_type` that the file's `agents:` lists -
then on any branch. Otherwise the hook says nothing and the call goes on.

| The call | Scoped | Not scoped |
|---|---|---|
| a file tool (`Read`, `Edit`, `Write`, `NotebookEdit`, an MCP tool...) on a path inside `allow` | passes | passes |
| the same tool outside `allow`, or outside the working tree (another worktree, the main checkout) | **refused**, naming the path | passes |
| `Grep` with no `path`, `Glob` whose pattern starts with a wildcard | **refused**: it searches the whole tree | passes |
| any tool but `Read`, `Glob`, `Grep` on the scope file itself | **refused**: a session that edits its scope widens it | passes |
| a shell command not starting with a `commands:` prefix | **refused**: what it reads cannot be told | passes |
| a listed command with a path argument outside `allow` | **refused** | passes |
| a command line with `$`, a backtick, `( ) { }`, a here-document | **refused**: only the running shell knows what it names | passes |
| an input that is not JSON | **refused** | passes |
| no scope file, or one that cannot be decoded (an unknown field is an error) | **every call refused**, saying why | passes |

File paths are read from the input's field **names**, not the tool's name: any
field whose name contains `path`, or ends in `file`, `dir`, `directory`, `folder`,
`cwd`, `root`, `source` or `destination`. A relative path is read from the call's
`cwd`; a path outside the working tree is matched in its absolute, forward-slash
form, so the scope can still allow one by writing it that way.

**The shell is denied by default.** A command line is split at `;`, `&&`, `||`,
`|`, `&` and newlines, and every simple command in it must be one of three things:
a command the scope lists (matched word by word; a last word ending in `/` is a path
prefix, so `git add docs/` allows `git add docs/a.md`), a `cd` (followed: later
paths are read from the new directory, and the directory itself must be allowed),
or a filter behind a pipe that reads only the pipe (`head`, `tail`, `grep`, `sort`,
`wc`, `Select-Object`...). Even a listed command has its arguments read: a word
that looks like a path (a slash, a leading `.` or `~`, a file extension, a
wildcard) or that **exists on disk** is a path and must be allowed; so is every
redirection target, and the path half of `revision:path`. A word with a space that
does not exist (a commit message) is not a path.

```text
$ x3 hook scope -file scope.yaml -branch "work/docs*" < read-src.json
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",...}}
x3 hook scope: Read reaches src/b.go; branch work/docs-a may reach only docs/** (scope.yaml)
```

The verdict is the JSON on stdout, and the same sentence is printed to stderr. The
hook **always exits 0**: an exit of 2 would stop every call of every session the
moment a registration or a directory went wrong. A registration missing `-file` or
`-branch` exits 1, which Claude Code shows without stopping the call. `-input
<file>` reads the input from a file instead of stdin, `-dir` is the directory an
input without `cwd` is read from, and `-log <file>` appends every raw input, one
line each - the way to see which fields a live session really sends.

**What it does not do.** A listed command is trusted with what it names: listing
`git show` lets the session read any revision, and listing `git switch` lets it
leave the branch, and the scope with it. Symbolic links are not resolved. A
backslash outside quotes is kept as written, not read as a shell escape. Whether
Claude Code sends `agent_type` for a subagent's call is not measured here; the
branch is the scope that holds without it. Eighteen arms run in this engine's
own gate (`agent scope hook control experiment`).

<!-- x3-dist version=v0.295.0 capabilities=84ff292206cc313338dc9e485b28d297be4e6745e8256bfe4076bebfe1d41a0f template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
