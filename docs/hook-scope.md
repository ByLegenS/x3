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
| `git commit -m "$(cat <<'EOF'` … `EOF` `)"` - one `cat` of a **quoted** here-document inside double quotes, nothing else | passes: the message is plain text | passes |
| the same with an unquoted `<<EOF`, a `$(` inside the body, a second command, or outside double quotes | **refused** | passes |
| `git -C <dir> <subcommand>` | read as `cd <dir> && git <subcommand>` for that one command: the directory must be allowed and the subcommand listed | passes |
| a path naming `~`, `$HOME`, `${X}`, `$env:X` or `%X%` | judged **both as written and opened**; refused if either reading leaves the scope | passes |
| a variable that is not set, or another user's home (`~name`) | **refused**: the hook cannot tell where it points | passes |
| an input that is not JSON | **refused** | passes |
| no scope file, or one that cannot be decoded (an unknown field is an error), on a scoped branch | **every call refused**, saying why | passes |
| the same for a call that carries an `agent_type` | passes, with a **visible warning** naming the file it looked for: the `agents:` list is inside that file, so who is scoped cannot be told | passes |

File paths are read from the input's field **names**, not the tool's name: any
field whose name contains `path`, or ends in `file`, `dir`, `directory`, `folder`,
`cwd`, `root`, `source` or `destination`. A relative path is read from the call's
`cwd`. Every path is first made absolute (`~`, `/c/...` and backslashes all land on
the same path), then each pattern speaks its own form: an **absolute** pattern
(`/x/**`, `C:/x/**`) is matched against the absolute, forward-slash path - inside
the working tree or out - and a **relative** pattern against the path from the
repository root, never reaching outside it. When the `cwd` is not a repository the
`cwd` itself is the root.

**Why two readings of a variable.** A tool may open `$HOME/a.md` or read it as
written; the hook cannot tell which. Judged only as written, `$HOME/a.md` asked from
inside `docs/` looks like `docs/$HOME/a.md` and passes while the tool reads the
home directory; judged only opened, a tool that does not open it reads a file next
to the one allowed. So both must be inside.

**An unreadable scope file is never silent.** On a branch the scope holds, every
call is refused. For a call that carries an `agent_type`, refusing is not narrow:
the hook is registered for every session, and a relative `-file` cannot be found
from a directory that is not the repository - every subagent of every session would
stop. Such a call passes, and the hook prints Claude Code's `systemMessage` (and the
same sentence on stderr) with the path it looked for. An absolute `-file` is read
from every directory.

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
backslash outside quotes is kept as written, not read as a shell escape. A `Glob`
or `Grep` in the **parent** of allowed directories is refused, not narrowed: it
would list the names (`Glob`) or print the lines (`Grep`) of the files beside them
that the scope does not allow - search each allowed directory instead. Whether
Claude Code sends `agent_type` for a subagent's call is not measured here; the
branch is the scope that holds without it. Twenty-six arms run in this engine's
own gate (`agent scope hook control experiment`).

<!-- x3-dist version=v0.339.0 capabilities=32778afbc9ceb1303caf168038bbe0c583c52168a2b336603fedb7c70c33d49b template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
