# lang reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.213.0`**

## language settings

| Field | Meaning |
|---|---|
| `allowed` | the language outside comments; `en` is the only embedded dictionary, and any other value is an error, not a silent pass |
| `comments` | `any` (default) leaves comments alone; `en` holds them to the dictionary |
| `strings` | `en` (default) holds every string constant to the dictionary; `any` leaves them alone |
| `sources` | which files are read; left out, the Go tree |
| `names` | which paths have their **name** read; left out, no name is read |
| `allow` | project terms no dictionary has. One ASCII word, three letters or more — an entry that could never match is rejected rather than ignored |
| `keywords` | a language's reserved words, per extension. Written, the list **replaces** the embedded one for that extension; written empty, that language has no reserved words |

<!-- x3-dist version=v0.213.0 capabilities=f60b889e641897e52daf3f936af2ca5e03d88f986b31c320450e70c499a0c20a template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
