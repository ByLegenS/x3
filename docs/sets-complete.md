# A list that names two names them all

[The pages](INDEX.md) - [what x3 is](../README.md)

## A list that names two names them all

### `left-complete-in-each-group`

```yaml
- name: a-list-naming-two-apps-names-every-app
  kind: consistency
  compare: left-complete-in-each-group
  left:  { from: tree, select: 'dirs:apps/*' }      # the members; any extractor
  right:
    from: regex
    sources: [deploy/*.yaml, .env.example]
    group: line                                       # line | block | region
    forms: ['app-{m}\b', 'apps/{m}/', 'ROLE_{M}=']    # {m} the name, {M} upper case
    comments: exempt
  quorum: 2                                           # default 2
  only: 'x3:only\(([a-z, ]+)\):\s*(\S.*)'             # names, then the reason
  minimum: 20                                         # lists the rule must see
```

**What it catches:** the list that was written when there were two members and
was never touched again. A stop command, a health check, a build loop, a role
variable - each names every application by hand, and the day a third
directory appears, every such list is silently short.

`consistency` cannot ask it. It reads the right side as **one pool**: a
complete line and an incomplete twin pour the same names into it, and the pool
agrees. `per` splits by component, not by line, and refuses a fixed file. This
comparison builds no pool: every group is weighed on its own.

**How it reads.** For every group G it takes the members whose form appears in
G. If at least `quorum` of them do and not all of them, the group is
`incomplete_list`, written on the group's first line and naming the members it
leaves out. A group naming fewer than `quorum` is a mention, not a list:
`run: psql -c 'SELECT * FROM alpha_settings'` is one member and stays green.

```yaml
halt: [web, app-alpha, app-beta]            # RED: incomplete_list, leaves out gamma
halt: [web, app-alpha, app-beta, app-gamma] # green
run: psql -c 'SELECT * FROM alpha_settings' # green: one member, not a list
# x3:only(alpha, beta): gamma has no worker
alive: [app-alpha, app-beta]                # green: marked, with its reason
# x3:only(alpha): kept from an older list
alive: [app-alpha, app-beta]                # RED: dead_marker
```

| `group` | One list is |
|---|---|
| `line` | one line |
| `block` | a run of `- ` items or `KEY=` lines at one indentation, with the deeper lines under each item; a blank line or a shallower line ends it, a comment-only line does not. A `key:` line right above the run opens it, so the finding and the marker sit on the key. A line outside every run is a block of its own |
| `region` | each container `region` (and `until`) draws, exactly as in *The container a value sits in* |

**The marker.** A list kept short on purpose says so, on its first line or the
line above it. It exempts the list only when it names **exactly** the members
the list names; the day the list or the marker changes alone, it is
`dead_marker` and red - and so is a marker standing beside no list, or one
whose reason is blank. The marker's text is taken out before the forms are
read, so its own names never count as a list.

**Two members tell nothing.** With two members every list that counts names
both, so the rule cannot be red. It does not pretend otherwise: when the set is
no larger than `quorum`, the run carries `vacuous_list` as a warning. The rule
earns its keep the day the third member arrives - which is why its control
experiment plants three.

Measured on the control experiment (three planted directories,
`x3.yaml` step `list completeness control experiment`):

| The tree | Result |
|---|---|
| two members, lists naming both | exit `0`, `vacuous_list` warning |
| a third directory, nothing else touched | exit `1`: the one-line list, the stepped block and the `KEY=` block, each by line, each "leaves out gamma" |
| every list completed, the marked line unchanged | exit `0` |
| the marker removed | exit `1`, `incomplete_list` on that line |
| the marker kept to one name | exit `1`, `dead_marker` |
| the engine before this comparison (v0.336.0) | exit `2`: `group` and `forms` are unknown fields |

### The laws

- `group` and `forms` belong to `from: regex` on the **right** side, and only
  beside this comparison; `select`, `parts`, `join`, `each`, `yields`, `holds`
  and `skip` are refused next to them - a group puts no value in a set.
- Every form carries `{m}` or `{M}`; the member is written into it as plain
  text, so a name with a dot matches only a dot.
- `quorum` is 2 or more; `only` has exactly two capture groups. `per` is
  refused: every group is already its own instance.
- A rule whose forms find no member at all is `empty_scope`; `minimum` counts
  **lists** (groups naming at least `quorum` members), so forms that rot to
  nothing cannot run green.
- The answer for each file is cached against the file's bytes, the members and
  the marker pattern: a second run over the same tree reads nothing again
  (`x3 arch: cache 4 hit(s), 0 miss(es)`).

<!-- x3-dist version=v0.340.0 capabilities=7f596239b33d07b08fcfa0550f33f7a3d4eadd50b8ad99f6fce73dc731635008 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
