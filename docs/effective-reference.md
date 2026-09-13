# effective reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.175.0`**

## effective check fields

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | unique within the file |
| `policy` | no | `warn` or `block`; **defaults to `block`** |
| `attempts` | no | retries while the comparison disagrees; defaults to `1` |
| `retryDelayMs` | no | wait between attempts; defaults to `250` |
| `recorded` | yes | one reading: the setting as it was written down |
| `effective` | yes | one or more readings: the setting as it is in force; all must equal `recorded` |

## effective reading fields

| Field | Meaning |
|---|---|
| `label` | the name this source carries in the report |
| `map` | value mapping applied before the comparison; a value the map does not mention is compared as it came |
| ~~`equals`~~ / ~~`contains`~~ | **rejected here** — a reading has no expectation of its own; its expectation is the other readings |

<!-- x3-dist version=v0.175.0 capabilities=67925653b79a8165a912b94e81e1a9319d1f1981fe2afdcae635a3269b53ab60 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
