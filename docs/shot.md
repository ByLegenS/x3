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

| Exit | Meaning |
|---|---|
| 0 | every image written |
| 1 | at least one page did not load, answered 400 or more, did not settle, or had more findings than a `-measure` ceiling allows |
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

<!-- x3-dist version=v0.304.0 capabilities=87bbb154d947a1e3765d06dda776c73c339b85c66625b3cf2393ebf3ab19477b template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
