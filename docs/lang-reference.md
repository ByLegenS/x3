# lang reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.190.0`**

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

<!-- x3-dist version=v0.190.0 capabilities=0baf44f282fed2c99d88b24a56fbcc96dc3aaa6477ac5d753eddf755e2ee5fc1 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
