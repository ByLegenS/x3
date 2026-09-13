# lang reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.197.0`**

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

<!-- x3-dist version=v0.197.0 capabilities=5a212239948f7f0b9a0d90e324cdd1e34c042af4a9543480a1a81b96b9e111f3 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
