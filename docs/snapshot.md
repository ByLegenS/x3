# A snapshot the cache keeps

[The pages](INDEX.md) - [what x3 is](../README.md)

## A snapshot the cache keeps

A run already records, step by step, the files it read (with their content
digest) and the directories it walked. Union those and you have a picture of
the tree as the last runs saw it. `x3 snapshot` is the question nobody could
ask before: **what has moved since then?**

```
x3 snapshot                       # every remembered path, judged against the disk
x3 snapshot -only changed         # just the ones that moved
x3 snapshot -command case         # another command's cache
x3 snapshot -cache <file> <root>  # a cache file and a tree named outright
```

It prints one line per path and a count:

```
  a.txt: changed
  sub/d.txt: new
  gone.txt: gone
x3 snapshot: 1 changed, 3 unchanged, 1 new, 1 gone, 0 moved across 12 walked directory(ies)
```

Five states, and the third is the one a read-list alone cannot give you:

| State | Means |
|---|---|
| `changed` | no remembered digest of this path matches what is on disk today |
| `unchanged` | one of them does — the gate still has a valid memory of it |
| `new` | a walked directory's *name list* has moved, and no record has ever named this path in it |
| `gone` | a record names it and the disk no longer holds it |
| `moved` | a rename: a `gone` path whose remembered content stands, byte for byte, under another name today — and that other name, when it is `new` |

**No second record is written.** The union was measured before it was designed:
in the pilot the gate cache already held **1 439 of the 1 439** files git
tracks and **197 of the 197** directories they live in, so the snapshot was
already on disk — it simply had no reader. Writing a parallel record would have
meant the same bytes twice and two files free to drift apart.

**It never asks git.** A tree that is not a working copy answers the same way,
and a file nobody tracks is as visible as one everybody does. That is also why
`new` works: a directory is remembered by the digest of its *name list*, so a
file no step will ever open still shows up the moment it lands.

**`new` means "the listing moved", not "nobody read it".** The digest is
*asked*, and a directory whose name list still matches the record has had
nothing arrive in it — so a file no rule will ever open, sitting there since
the first commit, is **not** a change. Until this was asked, it was: every run,
for ever. In the engine's own tree that was one file
(`internal/release/testdata/VERSION`, which `x3 docs` counted as an eighth
change while only seven paths had moved); in the pilot it was **595**, because
a settings edit moves the cache salt, a narrow run then rewrites the record
with only its own steps (see `cache.Save`), and every path the other steps had
read is forgotten at once.

When a listing *has* moved, every unread name in that directory is counted, not
just the one that arrived — which name it was cannot be recovered from a
digest, and counting too few is passing without measuring. The next run records
that listing again, so the over-count lasts exactly one run.

**Each path is weighed in its own terms.** A path the gate reads whole is
judged by its content; one it reads as *code* is not moved by a comment; one it
reads as a *surface* is not moved by a function body. The snapshot is what the
cache keeps, not a photograph of the disk — and what the cache keeps is exactly
what would make a step run again.

One limit, named: a slot holds several states of the same path (see
`cache.Pick`), so a file changed and changed back reads as `unchanged`. That is
the honest answer to "would the gate re-measure this?", not to "was this file
ever touched?".

**A rename is not an edit.** Measured in a real project: a migration file was
renamed to its numbered name with its content untouched, and a counterpart rule
("a migration changed, so the code that enforces it must change too") went red
for a file whose content never moved. The snapshot now matches every `gone`
path against today's tree in the term it was remembered in; a match is `moved`,
on both names. `x3 docs` weighs only the paths whose **content** changed and
says the renames out loud:

```
x3 docs: renamed, content unchanged, not a change: migrations/NEXT.sql
x3 docs: snapshot scope - 1 file(s) changed - 6 rule(s) - 0 block, 0 warn, 0 exempted
```

A rename is still a change to everyone else: `-scope snapshot` keeps both names
in the list `x3 test` and `x3 mutate` read, because a file moved to another
package does move what gets built. Empty content matches nothing — deleting one
empty file while another sits elsewhere is a deletion.

**Version control's own directories are not walked.** A step that walks the
whole tree also recorded `.git/objects/*`, and every commit writes there: in
that project **446 of 450** "changed" paths were git's own objects. Paths with a
`.git`, `.hg`, `.svn`, `.bzr` or `.jj` segment are left out of the snapshot and
out of the walked-directory count (`.github` is not such a segment).

The control experiment (`snapshot rename control experiment`) runs one
counterpart rule (`inside/**` needs `papers/**`) over seven planted records:

| Arm | Result |
|---|---|
| `inside/NEXT.txt` renamed to an unrecorded name, content untouched | `0 file(s) changed`, no BLOCK, both names said |
| the same rename, new name already remembered by another record | `0 file(s) changed`, no BLOCK |
| **red:** the content of `inside/a.txt` changed | `BLOCK ...: missing_counterpart_change` |
| **red:** `inside/NEXT.txt` deleted, its content nowhere else | BLOCK, `changed: inside/NEXT.txt`, no "renamed" |
| a moved `.git/HEAD` and a new `.git/objects/ab/cdef` | `0 file(s) changed` |
| **red:** the same two paths under a plain `git/` directory | `2 file(s) changed` |

<!-- x3-dist version=v0.290.0 capabilities=c138e0be580c1e819f85bea5e238767dc4c28c2e8ff5f7660ec20299173f5652 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
