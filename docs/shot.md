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
| 1 | at least one page did not load, answered 400 or more, or did not settle |
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

**What it does not do yet.** It takes pictures; it does not judge them. A layout
audit on the same browser — sideways overflow, an element with its own scrollbar,
a label broken onto two lines, clipped text — is the next capability, not this one.

<!-- x3-dist version=v0.299.0 capabilities=17f3b80bd2749e826740d8d5966d3ebf204b1bf0081d2f97f25f77059da0a873 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
