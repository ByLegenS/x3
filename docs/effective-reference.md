# effective reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.375.0`**

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

<!-- x3-dist version=v0.375.0 capabilities=668ee945fccc18dca2e9c501a6d66762bf582c357f09c62d0cbce2b4a14133e6 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
