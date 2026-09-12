# The version, and how it updates itself

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 version`

```
x3 version
```

Prints the release tag embedded at build time and exits `0` — one line, nothing
else, so a gate can compare it without parsing. A binary not produced by a
release run prints `unreleased`; an untagged binary is not a published one, and
a gate that pins versions should treat it as red.

## `x3 update`

**What it catches:** nothing — it removes the downloader every consuming project
would otherwise write for itself. The binary replaces itself with a published
one, after verifying its SHA256.

```
x3 update [-config <file>] [-version <tag>] [-source <address|dir|owner/repo>] [-check]
```

The order is fixed: resolve the tag (`-version`, else the release pointer); find
the checksum this platform's binary must have (from
[`update.pin`](#updatepin--the-checksum-the-project-itself-vouches-for) when the
project wrote one, else the release's `SHA256SUMS.txt`); download; compute and
compare; put it in place.

**A sum that does not match stops before step 5 and the running binary is left
exactly as it was**, as is a release listing no binary for this platform — both
exit `1` and name the file left in place. Not being able to *reach* the source
exits `2`: "the release refused me" and "the network refused me" are not the
same event and must not wear the same colour.

**The layout.** Every source, remote or local, is read as `<source>/<ref>/<file>`:
`main/LATEST` (one line, the newest tag), `<tag>/SHA256SUMS.txt`, and
`<tag>/x3-<goos>-<goarch>[.exe]`. `LATEST` is written by the same run that builds
the binaries — a separate step would drift, and a pointer naming a release nobody
published is the quietest way to break an update.

**Where it downloads from**, first answer wins: `-source`, `X3_UPDATE_SOURCE`,
`update.source`, then the engine's own public repository. Outside in, on
purpose: pointing one run at a mirror should not require editing a tracked file.
A source may be an address, an `owner/repository` shorthand, or **a local
directory** — the directory case is not a test fixture but how an air-gapped or
mirrored environment publishes the same three files onto a share. An existing
directory wins over the shorthand.

**`-check` changes nothing**: it prints the published tag and exits `1` if that
tag is newer than the running binary. The tag goes to stdout on a line of its
own; everything written for a human goes to stderr.

**Replacing a running file.** The new bytes are written next to the target
first, because a rename is only atomic within one filesystem. On Windows a
running executable cannot be overwritten but can be renamed, so the sequence is
write, move the old one aside, put the new one in place — and if the last step
fails the old name is given back.

```json
{ "update": { "source": "owner/repository", "latestRef": "main", "timeoutMs": 120000 } }
```

An unknown key in it is an error, not a silent skip.

### `update.pin` — the checksum the project itself vouches for

`SHA256SUMS.txt` ships **inside the release it describes**. It catches a
truncated download, a mirror that fell behind, a corrupted file. It cannot catch
a compromised release: whoever can replace the binary can replace the list
beside it. **A release that vouches for itself is not a supply chain guarantee.**

```json
{ "update": { "pin": { "v0.33.0": {
      "x3-windows-amd64.exe": "e2c2bd46...",
      "x3-linux-amd64":       "b21f4cc9..." } } } }
```

**When a pin is written, `SHA256SUMS.txt` is not read at all.** The stderr line
says which authority it obeyed, so a run never leaves that ambiguous. The pin is
keyed by **release tag**, not by binary name alone, which is what lets the engine
tell two refusals apart:

| Situation | Result |
|---|---|
| the tag is pinned and the bytes match | installed |
| the tag is pinned and the bytes differ | exit `1`, binary left in place, message names `update.pin` |
| the tag is not in `pin` | exit `1`, naming the tags that *are* pinned |
| the tag is pinned but not for this platform | exit `1` — a machine nobody pinned gets no weaker guarantee |
| no `pin` at all | `SHA256SUMS.txt` decides, exactly as before |

Because an unpinned tag is refused, a pinned project does not follow `LATEST` by
accident: a new release enters the project the day somebody writes its checksum
down. A malformed pin is a **configuration error** (exit `2`) rather than a
mismatch — reported as a mismatch, a mistyped checksum would leave a project
unable to update and unable to see why.

### The minimum version gate

```json
{ "x3": { "min_version": "v0.30.0" } }
```

This runs **before every command**:

```
x3: RED - this binary is v0.29.0, the project requires v0.30.0 or newer
	run: x3 update
```

`x3 update` is the one command exempt — it is the answer the gate points at, and
a red with no way out is a wall, not a gate. **An untagged binary satisfies
nothing.** A `git describe` suffix is ignored (`v0.30.0-3-gabc1234` counts as
`v0.30.0`, since those commits come after the tag). A requirement that is written
must parse: no file and no section means no requirement, but a value that is not
a release tag is an error — a misspelled requirement silently ignored leaves its
author believing a gate is running.

<!-- x3-dist version=v0.122.0 capabilities=15ca9e0e91f95ecffad1e8ad49dbf97d72b4af46193928ec5bd51865cf8290e0 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
