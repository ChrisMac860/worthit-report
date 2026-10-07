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

## Sharing

Below the report, a share panel offers two images, both drawn on-device on
offscreen canvases in the same pixel-art style as the chart:

- **Story** — 1080×1920, opaque. The session card centred on a full-bleed
  dithered background, so it fills a story when shared straight into one.
- **Sticker** — 1080×1080, **transparent** outside the card
  (`renderSticker()`), for laying over your own photo.

Both show the beer count, duration, date, HR start/average/peak (nulls
skipped), the Famous Last Words line if one was recorded, a small stepped HR
trace and a pixel mug — facts only, no praise or claims.

How the share stays one tap:

1. **Everything is prepared before the tap.** As soon as the report renders,
   both images are encoded to PNG and wrapped in `File`s, and the preview
   shows the selected one. `navigator.share()` and `clipboard.write()` need
   the click's transient user activation, and any `await` before them can
   spend it, so the Share button reads "Preparing…" (disabled) until the
   files exist and every click handler then calls the platform API
   synchronously.
2. **Share** passes the file to `navigator.share({files})`, which opens the
   OS share sheet (Instagram, WhatsApp, Messages, …) with the image already
   attached. Cancelling the sheet stays silent.
3. Where the browser cannot share files, the main button says **Save image**
   and downloads the PNG instead. Where it can, **Save** is offered as well.
4. **Copy image** (when the browser supports `ClipboardItem`) puts the PNG on
   the clipboard to paste into a chat or post.

The report URL is never shared: it carries the whole session in its
fragment. A web page also cannot open Instagram's story composer directly
with media pre-loaded (native apps do that with a platform-specific
hand-off); from the share sheet, Instagram receives the image and you pick
Story there. Everything happens on-device — nothing is uploaded anywhere.

## Support

The footer links to the static [support page](support.html) and privacy policy.
The "Buy me a beer" link (`https://buymeacoffee.com/christopheosp`) is styled
natively in the page's own pixel-button chrome rather than using an embed
script. Links only fire when the reader taps them — the report page still
makes zero network requests on load.

## Privacy

The public policy is available at
<https://chrismac860.github.io/worthit-report/privacy.html>.

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
