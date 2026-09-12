# Is the release really published

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 published`

```
x3 published [-config <file>] [-out <file>]
```

A release is not the files it writes. A consuming project asks for a **version**,
and a version is a **tag**: the publication is read as `<repository>/<tag>/<file>`
([`x3 update`](update.md#x3-update)). A publication whose tag was never pushed is therefore
a release only on the machine that made it — the pointer announces a version,
every file is in place, every checksum matches, and nothing goes red until
somebody tries to download it.

That has happened twice in this engine's own history, and the second time a person
caught it by hand. No gate asked, because the question is not about files: it is
about a remote, and none of them were speaking to one.

`x3 published` reads one **claim** and measures every repository that has to carry
it:

```json
"published": {
  "claim": { "dir": "../releases", "file": "LATEST" },
  "repositories": [
    { "dir": ".", "remote": "origin" },
    { "dir": "../releases", "remote": "origin" }
  ]
}
```

The claim is the publication's own announcement: a file naming the tag of the
newest release. Each way a repository can fail it is a separate finding, because
each is a separate step somebody can forget:

| Code | What the run found |
|---|---|
| `tag_never_created` | the claim names a tag this repository does not have |
| `tag_not_pushed` | the tag is here and the remote does not have it — nothing can be downloaded from it |
| `tag_differs_on_remote` | here and on the remote, the tag names different commits |
| `tag_off_the_line` | the tagged commit is not an ancestor of `HEAD` — the tag sits on work this line does not carry |
| `tag_is_not_the_release_commit` | the tag names a commit whose claim file announces another version |
| `claim_unreadable` | the announcement itself could not be read |
| `remote_unreachable` | the remote could not be asked |

**The claim is read from the committed `HEAD`, not from the working tree.** A
publish run that has written a new pointer but has not committed it has announced
nothing yet, and a gate that went red between writing and committing would be red
in the ordinary course of every release — which is how a gate stops being read.
Once the release is committed, the announcement is live and the tag is owed.

**A remote that cannot be asked is red, and says so by name.** "I could not
measure" and "I measured, and it is there" must never be the same colour: a run
with no network proves nothing about a publication.

**The repository carrying the claim must be one of the measured ones.** Were it
not, the very tag the pointer names would go unmeasured in the repository that
announces it — so that is a configuration error (exit `2`), not a finding.

An annotated tag and a lightweight one are read the same: `refs/tags/<tag>^{}` is
preferred where the remote offers it, so a project that annotates its tags does
not see every release reported as pointing somewhere else.

**What it does not do.** It pushes nothing, creates no tag, and does not download
the release. It answers one question: *is the version this publication announces
actually obtainable?*

<!-- x3-dist version=v0.129.0 capabilities=f407b41733163dd348e141d538fa5d3f1f5a9f6c3f9f9b5019029a55fdf4d2fc template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
