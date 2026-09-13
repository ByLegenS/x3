# retire reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.196.0`**

## the retire report fields

| Field | Meaning |
|---|---|
| `groups[].name` | what the report calls this pile; unique, and the name a finding carries |
| `groups[].patterns` | the files counted; the first group that matches a file owns it |
| `groups[].start` | how many stood on the day the promise was made - the denominator |
| `groups[].reason` | why they are going; printed beside the count |
| `exclude` | paths no group counts (fixtures, vendored trees) |
| `policy` | `block` (default), `warn`, or an object keyed by finding code |

<!-- x3-dist version=v0.196.0 capabilities=42c9e4d25b581df1ee169b0e61079459d286eff37cd1289ab08a0064586cddf3 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
