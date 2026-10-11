# Full-length screenshots

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 shot`

A design reviewed page by page needs the same picture of every page, at every
width, in every theme — and taking them by hand is where pages get skipped.
`x3 shot` starts a Chromium-based browser **headless, on a throwaway profile**,
opens each page, and writes one PNG for each page and size.

```text
x3 shot -base http://127.0.0.1/site/ -sizes 1440,390,1440:dark -out shots -index index.html guide/faq.html api/core/index.html
x3 shot -sizes 390 page.html https://example.com/   # a file on disk, or a full address
x3 shot -base http://127.0.0.1/site/ -accept 404 no-such-page.html   # the error page, captured on purpose
```

```text
x3 shot: index-1440.png 1440x1355
x3 shot: index-390.png 390x2540
x3 shot: index-1440-dark.png 1440x1355
x3 shot: guide-faq-390.png 390x1567
x3 shot: 84 image(s) in shots, 0 failed
```

**The length comes from the page, not from the window.** The obvious trick — a
window as tall as the page — changes the page: every `100vh` block grows to the
window's height and the picture ends in empty space. The browser is asked for the
layout's own size (`Page.getLayoutMetrics`) and captures past the visible area
(`captureBeyondViewport`), so a `100vh` block keeps the window's height: 900 on a
desktop width, 844 on a phone width. A page longer than 8000 pixels is captured in
slices and joined, because the browser's texture limit is 16384.

**What one size means.** `-sizes` takes widths, each with an optional theme
(`1440:dark`; light is the default and is emulated too, so the machine's own theme
never leaks in). The theme is `prefers-color-scheme`, which is what a page's own
CSS and script read. A width under **768** is a phone: touch and the `mobile` flag
are emulated with it. The file is the page's path with `/` turned into `-` and the
extension dropped, then the width, then `-dark`: `api/core/index.html` at
`390:dark` is `api-core-index-390-dark.png`. Two pages that would land on the same
file stop the run before anything is opened.

**When a page is ready.** Not after a fixed sleep. A page is captured when its load
event has fired, no request has been open for half a second, `document.fonts.ready`
has resolved, and two animation frames have passed. Lazy images are switched to
eager first, or everything below the first screen would be blank. A page that does
not settle within `-timeout` (a minute) is a failure, not a half-loaded picture.

**What a run says.** One line per image, on stderr, with its size. A page whose
content is wider than the width says so — `content 2000 wide, the page scrolls
sideways` — because a cropped picture would hide exactly that. `-index <file>` also
writes a page of that name next to the images, listing them page by page. Pages are taken in up to
`-tabs` tabs at once (4).

**The browser.** `-browser`, then `X3_BROWSER`, then the usual install places
(Edge and Chrome on Windows and macOS, `chromium` and `google-chrome` on the path
elsewhere). It never runs with your profile and never attaches to a browser you
have open: its profile is a temporary directory, removed when the run ends, and
the browser is closed — politely first, killed if it does not go.

**Signed-in pages: `-cookies`.** A throwaway profile has no session, so a page behind
a sign-in is captured as the sign-in screen. `-cookies` gives the run one, without
touching anyone's browser: the cookies are written into the throwaway profile before
the first page and go away with it.

```text
x3 shot -base http://127.0.0.1:8080/ -cookies ~/.session/app.txt account/index.html
SESSION='sid=…' x3 shot -base http://127.0.0.1:8080/ -cookies env:SESSION account/index.html
```

```text
x3 shot: session cookie(s) sid set for 127.0.0.1:8080, values not shown
x3 shot: account-index-1440.png 1440x2000
```

The **value never sits on the command line**, where the process list and a log would
show it: `-cookies` names a file (one `name=value` per line, `#` for a comment) or
`env:NAME`, a variable holding `name=value; name2=value2`. Only the names are printed;
an error names the entry's number and the cookie's name, never its value; the index
page never sees it. The cookies are set for `-base`'s host only — a run without an
`http(s)` base stops with exit 2, so a session never goes to a host nobody named — and
they are `httpOnly`, so the page's own script cannot read them onto the screen. An
empty file or variable stops the run too: it would capture the sign-in screen and call
it the page. Headers (`Authorization: Bearer …`) are not offered: the debugging
protocol's extra headers go with every request a page makes, to every host it loads
from, and limiting them to one host needs request interception.

A cookie named `__Host-…` or `__Secure-…` is written with `Secure`, which the browser
demands for those prefixes; on plain `http` it still reaches `localhost` and
`127.0.0.1`, which the browser counts as a secure origin. Measured: before, the same
file stopped the run with `Invalid cookie fields`.

| Arm (`signed-in screenshot control experiment`) | Measured |
|---|---|
| a page that answers 401 and a 1500-pixel sign-in screen without `sid`, 200 and 2000 pixels with it; the session from a file, `-index` on | exit 0, `private-1440.png 1440x2000`; the value in neither stdout, stderr nor any written file (the test server scans all three) |
| the same page without `-cookies` | exit 1, `1440x1500 - the page answered 401` |
| the session from `env:NAME` | exit 0, `session cookie(s) sid set for 127.0.0.1:…`, value nowhere |
| `-cookies` without `-base` · an empty variable | exit 2 before the browser starts |
| the cookies never set (a planted mutation) | red: both signed-in arms capture the 401 screen |
| the value printed on the cookie line (a planted mutation) | red: stderr carries it and the server's scan names the command's output |
| the value written to a file in the output directory (a planted mutation) | red: the server's scan names that file |

| Exit | Meaning |
|---|---|
| 0 | every image written |
| 1 | at least one page did not load, answered 400 or more, did not settle, had more findings than a `-measure` ceiling allows, failed a step, or showed other text than an `expect` line wants |
| 2 | nothing was taken: no page, a size that cannot be read, or no browser |

| Arm | Measured |
|---|---|
| a 2345-pixel page, at 1440 and at 390 dark | `1440x2345`, `390x2345`; never the window's 900 |
| a 60-pixel sticky header, a `100vh` block and 1200 pixels more | `1440x2160` and `390x2104` (60 + 900 + 1200, 60 + 844 + 1200) |
| a 2000-pixel-wide block at 390 | `content 2000 wide, the page scrolls sideways`; the image is 3376 long, because a phone widens the layout of a page that overflows and its length grows with it |
| the length taken from the window instead of the layout (a planted mutation) | red: the 2345-pixel page and both `100vh` arms never say their length |
| a 3000-pixel window, the "tall window" trick (a planted mutation) | red: the desktop `100vh` arm never says `1440x2160` |
| a page that is not there | exit 1, `the page did not load: net::ERR_FILE_NOT_FOUND` |
| a browser that is not there | exit 2, `no browser at ...`, nothing written |
| 28 pages × {1440, 390, 1440 dark}, 4 tabs | 84 images in 41 s; no browser process left behind |
| `-measure` each of `scrollbar`, `truncation`, `label`, `touch` at 390 on a page carrying one defect of each kind | exit 1, one line each, naming `div#strip`, `p#clip`, `button#save`, `a#close` |
| `-measure overflow` on the 2000-pixel-wide block at 390 | exit 1, the culprit `div` named |
| `-measure touch` on the same page at 1440 | exit 0, silent: not measured above 860 |
| `-measure scrollbar:1` on the same page | exit 0, the strip still named, `within the ceiling of 1` |
| `-measure all` at 390 and 1440 on the same elements done right, with a link in running text, a labelled checkbox, a wrapping sentence in a chip, a hidden button and a visually hidden skip link | exit 0, no finding |
| the same page before the visually-hidden rule | red: `a.sr-only` named as cut-off text and as a tap target |
| a docs site, 7 pages × {1440, 860, 390}, `-measure all` | 21 images; 10 red, all `touch`: two code-language tabs and a breadcrumb link under 44 at 860 and 390; nothing at 1440 |
| the inline-link exemption and the short-label limit removed (a planted mutation) | red: the good page names `main > p > a` (touch) and `main > p > span` (label) |

### The layout audit: `-measure`

A picture hides the defects a reviewer most often misses: a strip that grew its own
scrollbar looks tidy, text cut off with `…` looks designed, and a 20-pixel button
looks fine at a glance. `-measure` asks the page itself, on the same browser, at the
same width, right after the length is taken — no second load, no script file: the
measure is one `Runtime.evaluate` over the debugging protocol, like the settle step.

```text
x3 shot -sizes 1440,860,390 -measure all guide/faq.html
x3 shot -sizes 390 -measure scrollbar:2,touch,label guide/faq.html   # two scrollers pass, the rest none
```

```text
x3 shot: guide-faq-390.png 390x1567
x3 shot: guide-faq-390.png scrollbar: 1 element(s) with their own sideways scrollbar, over the ceiling of 0 - div.table-wrap
x3 shot: guide-faq-390.png touch: 4 tap target(s) smaller than 44x44, over the ceiling of 0 - nav > a.icon, button#menu, input#q, and 1 more
x3 shot: 3 image(s) in shots, 0 failed, 0 answered a status -accept does not name, 1 over a -measure ceiling
```

| Measure | A finding is | Why this definition |
|---|---|---|
| `overflow` | an element whose right edge passes the width while its parent's does not, outside any clipping or scrolling ancestor — counted only when the page itself scrolls sideways | names the culprit, not every descendant it drags along; a page that overflows but whose culprit cannot be found still says `content N wide, no single element found` |
| `scrollbar` | an element with `overflow-x: auto` or `scroll` whose content is wider than its box (`scrollWidth > clientWidth`) | horizontal only: a sideways strip inside a page is what a phone user does not discover; a vertical scroller is often a deliberate panel |
| `truncation` | an element carrying its own text, clipped by `text-overflow: ellipsis` or `overflow-x: hidden`/`clip`, whose content is wider than its box | the text node must be the element's own child, so a clipping container around images or other boxes is not text being cut |
| `label` | a control (`button`, `summary`, `[role=button]`, a link that is not inline, any `inline-block`/`inline-flex`/`inline-grid` box such as a badge or chip) whose text is at most three words and 40 characters, on one logical line, rendered on more than one line | a short label is read as one unit; a sentence that wraps, a link wrapping inside running text, or text with a line break of its own is normal flow. Lines are counted from the text's own boxes, so an icon beside the text is not a second line |
| `touch` | at widths up to **860**: a visible `a[href]`, `button`, `input`, `select`, `textarea`, `summary` or `[role=button]` smaller than **44×44** | 44 is WCAG 2.5.5 and Apple's minimum. 860, not the phone threshold 768, because a tablet held upright (768–834) is used with a finger. Exempt: a link inline in running text (WCAG's own exception), a checkbox or radio whose label enlarges the target |

A hidden element is never a finding: no box, `visibility: hidden`, or the
visually-hidden pattern a screen reader reads (a box of one pixel or less, or
`clip: rect(0 0 0 0)`) — its text is clipped on purpose and it is no target on
screen. Measured: before this rule, every docs page of the pilot reported its skip
link and its icon-only tab labels as cut-off text and tiny tap targets. Each finding
names its element by a short selector: the nearest `#id`, otherwise up to three
`tag.class` steps. A line names the first three and counts the rest.

**The ceiling.** A measure written alone (`touch`) allows no finding: these are
defects, and one is enough to look at. `name:N` lets N findings pass — a project that
has decided its code blocks scroll writes `scrollbar:3` and names the decision where
it writes the flag. A finding within its ceiling is still printed (`within the
ceiling of 3`), so an allowance never turns into blindness. `all` turns on all five
with no allowance. Without `-measure` nothing is measured and nothing changes: the
sideways-overflow line is still printed, and still informs without turning red.

**What it does not measure.** Text clipped vertically (`line-clamp`, a fixed height),
two elements overlapping, colour contrast, and anything inside an `iframe`.

### Steps in one tab: `-steps`

Some states have no address: a dialog that opens on a click, a card that appears
after a choice, and the defect that lives only in in-page navigation — open record
A, then B, then A again without a reload, and B's form still shows A's values. One
picture per address cannot show any of them. `-steps <file>` runs a list on every
page **in one tab, in order**, and takes images only where it says `shot`:

```text
# in-page navigation A -> B -> A
click #rec-a
wait #form
expect #name = Alpha
shot a
click #rec-b
expect #name = Beta
shot b
click #rec-a
read #name
```

```text
x3 shot -sizes 1440 -steps records.steps -base http://127.0.0.1:8080/ records.html
x3 shot: records-1440 expect #name = "Alpha": "Alpha"
x3 shot: records-a-1440.png 1440x1253
x3 shot: records-1440 expect #name = "Beta": it shows "Alpha"
x3 shot: records-b-1440.png 1440x1253
x3 shot: records-1440 read #name: "Alpha"
x3 shot: 2 image(s) in shots, 0 failed, 0 answered a status -accept does not name, 1 expect line(s) not met
```

| Step | Does |
|---|---|
| `click <selector>` | scrolls the element to the middle and clicks it with the mouse, as a person would — an element under an overlay gets no click |
| `wait <selector>` | until the element exists and is shown, or `-timeout` |
| `fill <selector> = <text>` | sets an input, textarea or select and fires `input` and `change` |
| `read <selector>` | prints the element's text (an input's value), whitespace collapsed |
| `expect <selector> = <text>` | the same, and the run is red when the text is another |
| `shot <name>` | an image named `<page>-<name>-<width>.png`, at full length |
| `reload` | loads the page again in the same tab, and waits for it to settle |

After every click, fill and reload the run waits for two animation frames and for
the network to be quiet, so a read sees what the click fetched. The selector may hold
spaces; `fill` and `expect` split selector from text at ` = `. A list without a
`shot` line stops the run (exit 2): nothing would be captured. A failing step stops
that page's list — the next steps would run on a page nobody knows — and is red.

**No secret in a step.** `fill` and `read` refuse a `type=password` field: the value
would be in the step file, and a read would print it. A filled value is never
printed either — the line names the selector only.

| Arm (`screenshot steps control experiment`) | Measured |
|---|---|
| a page whose second record keeps the first record's value, A → B → A | exit 1, `expect #name = "Beta": it shows "Alpha"` |
| the same page fixed | exit 0, `expect #name = "Beta": "Beta"` |
| a dialog opened by `showModal()` on a click | exit 0, `read dialog[open] h2: "Edit name"`; the image shows it open |
| `fill` on a password field | exit 1, `#pw is a password field`; the value in no output |
| a file that is not a step list | exit 2, the line named, before the browser starts |

### Inner scrolling: `-inner-scroll`

Many applications keep the document one window tall and scroll inside a panel. The
layout size is then the window's, and the image stops at 900 pixels with the lower
cards cut. With `-inner-scroll`, the largest element that scrolls vertically (at
least half the window wide and tall) is followed: the window grows by its hidden
length until nothing is hidden, the image is taken, and the window goes back. A
fixed-height box that does not grow with the window is left as it is — growing the
window for it would only add empty space under every `100vh` block. It is off by
default: a page without it costs nothing more.

| Arm | Measured |
|---|---|
| a 60-pixel header over a panel of `calc(100vh - 60px)` holding 3000 pixels | `1440x3060` with the flag, `1440x900` without |
| the 2345-pixel page and the `100vh` page with the flag | unchanged: `2345`, `2160` |

### The network record: `-network` and `-self-signed`

`-network` writes, next to each image, `<image>.network.txt`: one tab-separated line
per request the page made since the previous image — start and end in milliseconds,
bytes on the wire, status, cache state (`network`, `304`, `memory`, `disk`,
`worker`), protocol, depth, type, address, and initiator — and one summary line:

```text
x3 shot -sizes 1440 -network -steps reload.steps -base http://127.0.0.1:8080/ chain.html
x3 shot: chain-first-1440.png network: 5 request(s), 1.1 KB, chain depth 3, last finished at 62 ms; cache: network 5; protocol: http/1.1 5
x3 shot: chain-second-1440.png network: 5 request(s), 0.7 KB, chain depth 3, last finished at 17 ms; cache: network 2, 304 3; protocol: http/1.1 5
```

**The chain depth is counted by initiator, not by time.** The document is 0, what it
asks for is 1, what that asks for is 2. Two requests one after the other may not
have waited for each other; a module imported by a module has to. The depth is the
longest such chain — the number of round trips nothing can overlap. With `-steps`,
a `reload` between two `shot` lines splits the record, so the second image carries
the second load alone and its times start at its own first request.

`-self-signed` accepts a self-signed certificate, so a local `https` server — the only
way a browser speaks h2 — can be measured. Every page must be on this machine
(`127.0.0.1`, `localhost`, `::1`); any other host stops the run before the browser
starts, because switching certificate checks off for a remote host opens the
measurement to whoever sits in between.

| Arm (`screenshot network record control experiment`) | Measured |
|---|---|
| a document asking for module 1, which imports 2, which imports 3 | `chain depth 3` |
| the same page loaded again (`reload`), every module with an `ETag` and `no-cache` | `304 3` |
| a local h2 server with a self-signed certificate, `-self-signed` | exit 0, `protocol: h2` |
| the same server without it | exit 1, `ERR_CERT_AUTHORITY_INVALID` |
| `-self-signed` with a page on another host | exit 2 before the browser starts |
| a `__Host-sid` session on a plain-http `127.0.0.1` page | exit 0, the signed-in page, its value in no output and no file |

<!-- x3-dist version=v0.342.0 capabilities=5da92501c0522b7dd930197f3dbaf5aacfa8cc55706c450814a9df2a48292932 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
