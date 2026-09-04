# worthit-report

Static "Send to Phone" report page for the **Worth it?** Garmin watch app.

This is a single self-contained `index.html` — no build step, no server, no
external requests (no CDNs, no fonts, no analytics). It renders a session
report entirely from data encoded in the URL fragment.

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
