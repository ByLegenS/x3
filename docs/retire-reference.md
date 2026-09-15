# retire reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.232.0`**

## the retire report fields

| Field | Meaning |
|---|---|
| `groups[].name` | what the report calls this pile; unique, and the name a finding carries |
| `groups[].patterns` | the files counted; the first group that matches a file owns it |
| `groups[].start` | how many stood on the day the promise was made - the denominator |
| `groups[].reason` | why they are going; printed beside the count |
| `exclude` | paths no group counts (fixtures, vendored trees) |
| `policy` | `block` (default), `warn`, or an object keyed by finding code |

<!-- x3-dist version=v0.232.0 capabilities=cd152fab91681cf14f364a8be9adb26569a95605d58e2e1aba63de4fcbd479fd template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
