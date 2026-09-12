# lang reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.147.0`**

## language settings

| Field | Meaning |
|---|---|
| `allowed` | the language outside comments; `en` is the only embedded dictionary, and any other value is an error, not a silent pass |
| `comments` | `any` (default) leaves comments alone; `en` holds them to the dictionary |
| `strings` | `en` (default) holds every string constant to the dictionary; `any` leaves them alone |
| `sources` | which files are read; left out, the Go tree |
| `names` | which paths have their **name** read; left out, no name is read |
| `allow` | project terms no dictionary has. One ASCII word, three letters or more — an entry that could never match is rejected rather than ignored |

<!-- x3-dist version=v0.147.0 capabilities=ec7474ac3ae437e0e7b4c481e6241021d5e7b6a9a998088c3a6178cf6d8f00af template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
