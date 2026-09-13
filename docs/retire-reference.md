# retire reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.175.0`**

## the retire report fields

| Field | Meaning |
|---|---|
| `groups[].name` | what the report calls this pile; unique, and the name a finding carries |
| `groups[].patterns` | the files counted; the first group that matches a file owns it |
| `groups[].start` | how many stood on the day the promise was made - the denominator |
| `groups[].reason` | why they are going; printed beside the count |
| `exclude` | paths no group counts (fixtures, vendored trees) |
| `policy` | `block` (default), `warn`, or an object keyed by finding code |

<!-- x3-dist version=v0.175.0 capabilities=67925653b79a8165a912b94e81e1a9319d1f1981fe2afdcae635a3269b53ab60 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
