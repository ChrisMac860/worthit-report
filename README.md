# worthit-report

Static "Send to Phone" report page for the **Worth it?** Garmin watch app.

This is a single self-contained `index.html` — no build step, no server, no
external requests on load (no CDNs, no analytics, no remote fonts). It
renders a session report entirely from data encoded in the URL fragment.

## Design

The page is styled as an arcade score/attract screen rather than a generic
card layout:

1. **Pixel-faithful HR chart** — drawn at a small internal resolution with
   `imageSmoothingEnabled=false`, coordinates snapped to an integer grid,
   a stepped (not smoothed) trace, square markers, and a marked peak;
   upscaled to the page with `image-rendering: pixelated`.
2. **Arcade score layout** — `grid-template-columns: 1fr auto` rows, labels
   left, tabular numbers right.
3. **One oversized, off-centre hero stat** — the session's beer count,
   rendered far larger than everything else, like an arcade `SCORE` line.
4. **Bitmap type for headings and the hero number** — *Press Start 2P*
   (SIL OFL 1.1), fetched once at build time from Google Fonts and embedded
   inline as a base64 `@font-face` (~16 KB). Body copy stays on the system
   monospace stack; no font is ever fetched over the network at runtime.
5. **Bevelled, hard-cornered chrome** — inset `box-shadow` highlights/shadows
   instead of borders-with-radius; no rounded corners, no drop shadows.
6. **Restrained scanlines + phosphor glow** — a faint fixed scanline overlay
   and amber text-shadow on the hero number only, disabled entirely under
   `prefers-reduced-motion` and `prefers-contrast: more`.
7. **`box-shadow` pixel sprites** for the mug icon on the support button,
   in place of an emoji (which renders differently per platform).
8. **A dither tile** (a small repeating conic-gradient checker) anywhere
   shading is needed, instead of a smooth gradient.
9. **An 8px spacing grid with deliberate asymmetry**, plus a two-column
   layout above 600px width.

## Instagram sticker

The header's "Sticker for Instagram" button renders a 1080×1080
**transparent-background** PNG on an offscreen canvas (`renderSticker()`),
reusing the same pixel-art approach as the chart: chunky, `imageSmoothingEnabled=false`
blocks scaled from a 108×108 design grid. It shows the beer count, duration,
date, HR start/average/peak (nulls skipped), the Famous Last Words line if
one was recorded, a small stepped HR trace, and a pixel mug — facts only, no
praise or claims.

On tap: `canvas.toBlob` → `File` → if `navigator.canShare` accepts the file,
`navigator.share()` hands it straight to the OS share sheet (Instagram
accepts a shared PNG into a Story's sticker tray there). Otherwise the PNG
downloads directly and the page shows a three-line note ("Saved to your
photos. Instagram → your story → sticker tray → add from photos.") along
with a preview of the generated image. Everything happens on-device —
nothing is uploaded anywhere.

## Support

The footer's "Buy me a beer" link (`https://buymeacoffee.com/christopheosp`)
is the page's **only outbound link**, styled natively in the page's own
pixel-button chrome rather than the official embed script. It only ever
fires if the reader taps it — the page still makes zero network requests on
load.

## Privacy

The session payload lives **only in the URL fragment** (the part after `#`).
Browsers never send the fragment to the server on a page load or navigation,
so your session data never reaches GitHub, GitHub Pages, or any other
server — it stays on your phone, parsed and rendered locally by JavaScript
already loaded in the page. Nothing is logged, stored remotely, or
transmitted anywhere.

## Live page

https://chrismac860.github.io/worthit-report/

Append `?demo` to the URL (with no fragment) to preview the page with a
built-in sample payload. Append `?selftest` to run the built-in parser
self-test and print PASS/FAIL results at the top of the page.

## Wire format

The watch app builds a compact, URL-safe positional string (`[0-9A-Za-z.~_-]`,
no encoding needed) and opens:

```
https://chrismac860.github.io/worthit-report/#<payload>
```

Payload grammar:

```
v1~<sa>.<en>~<b1>.<b2>...~<dt>.<bpm>.<dt>.<bpm>...~<hs>.<ha>.<hp>~<ts>.<te>~<pr>~<prev>|<prev>...
```

Sections are separated by `~`, in this order: version, times, beers,
hrTrace, hrStats, temp, prediction, previous.

- **version** — always `v1`.
- **times** — `sa.en`: `sa` = session start epoch seconds, `en` = end epoch
  seconds. `en` may be `-` if the session auto-closed without a manual end;
  the page then uses the last beer's time as the end.
- **beers** — `.`-separated absolute epoch seconds, ascending. The first
  beer equals `sa`.
- **hrTrace** — flat alternating `dt,bpm` pairs, where `dt` is the number of
  seconds since the *previous* point (the first `dt` is seconds since `sa`).
  May be empty (no HR history recorded).
- **hrStats** — `hs.ha.hp`: start/average/peak bpm. Any value may be `-`.
- **temp** — `ts.te`: start/end temperature in tenths of °C. `-` when the
  watch has no temperature sensor (most Garmin watches don't); the page
  hides the temperature section entirely when both are `-`.
- **prediction** — `0` = "I'll be grand", `1` = "Absolutely not", `-` = not
  asked or skipped.
- **previous** — `|`-separated summaries of earlier sessions, newest first,
  up to 5. Each summary is `sa.en.beers.ha.hp.pr`. May be empty.

Any field may be `-`, meaning null. The page is defensive: malformed or
missing fragments show a friendly "Couldn't read this report" message
instead of a blank page or a JavaScript error.

## Tone

The report states facts only. It never praises or shames drinking, never
claims anything is healthy or unhealthy, never claims heart-rate changes
were *caused* by drinking, and never estimates intoxication or fitness to
drive. Temperature is presented as supporting data only.
