# Crimson Moon View Counter

A live, animated GitHub profile view counter: a 10-frame procedurally
generated pixel-art banner where a crimson moon emerges from cloud cover once
per page load, with the running view count inscribed on its flat face.

<div align="center">
  <a href="https://anbu-00001.github.io/Anbu-00001/souls_counter_demo.html" title="Scroll-scrub the reveal frame by frame">
    <img src="https://souls-view-counter.vercel.app/api/counter" alt="Crimson Moon view counter — live example" width="100%" />
  </a>
  <br />
  <sub>Live example — this is the actual deployed endpoint, not a screenshot. Click it to scroll-scrub the reveal.</sub>
</div>

It ships as a serverless function that returns a fresh SVG on every request,
not a static image — that distinction is the entire point, and the first
"Finding" below explains why.

## How it fits together

```
api/counter.js     ── HTTP handler: resolves the count, sets cache headers, returns SVG or JSON
lib/counter.js      ── view-count resolution (Komarev + optional KV)
lib/render.js        ── composites frames + inscribed count into one SVG
lib/pixelfont.js       ── 3x5 bitmap font, drawn as <rect>s
lib/frames.js            ── the 10 frame PNGs, pre-encoded as base64 data URIs
piskel_frames/             ── source art: generator script, .piskel project, raw PNGs
scripts/                     ── dev server + selftest harness, static-snapshot utility
```

Embed it in a README with:

```md
<img src="https://<your-deployment>.vercel.app/api/counter?username=<your-github-username>" width="100%" />
```

## Findings — why it's built this way

These are the non-obvious constraints that shaped this design. Skipping any
one of them is the difference between "counter" and "static image that looks
like a counter."

**A committed SVG can never be a live counter.** GitHub serves any file
checked into a repo as a static asset. The count has to come from a
request-time endpoint — there is no way to make a file in git update itself
per view. `scripts/snapshot.js` exists for local preview only and is
deliberately not what the live badge points at.

**GitHub proxies README images through Camo, and Camo honours upstream cache
directives.** If `/api/counter` doesn't send
`Cache-Control: max-age=0, no-cache, no-store, must-revalidate`, Camo pins the
first response it fetches and every later viewer sees that same frozen image
— the counter looks broken even though the backend is genuinely computing a
fresh number on every request. This header is set in both `api/counter.js`
and `vercel.json` so it's enforced twice.

**Komarev de-duplicates by socket IP and ignores `X-Forwarded-For`.**
Verified directly: three requests carrying three different `X-Forwarded-For`
values all came back with the same count. That means a serverless function
that *proxies* Komarev — fetching its number server-side on behalf of every
visitor — always looks like a single visitor to Komarev, and its number never
moves. What actually drives Komarev's counter is its own 1x1 `<img>`, fetched
by Camo once per real page view. Measured: 3 Camo fetches → the number went
up by 3; 2 direct server-side fetches from one IP → +0. So this repo reads
Komarev as a *floor* (a number that only ever goes up, sourced independently)
rather than trying to increment it — and if you want a number that advances
on literally every view rather than every distinct visitor, that's what the
optional KV layer is for (see [Deploy](#deploy)).

**Web fonts do not load inside a Camo'd SVG.** GitHub renders the image
inside an `<img>` tag, which blocks all external resource loading — a Google
Fonts `@import` silently fails and falls back to generic monospace, with no
error visible anywhere. `lib/pixelfont.js` draws every glyph as SVG `<rect>`s
instead, which has no external dependency to fail.

**Cross-fading frames (out, then in) causes a black flash every frame.**
The obvious way to animate a frame sequence is to fade the outgoing frame's
opacity to 0 while fading the incoming frame's opacity to 1. But partway
through that transition, *both* frames are below full opacity at once, so
neither fully covers the canvas and the page background shows through for an
instant — once per frame, ten times a reveal. The fix in `lib/render.js` is a
**cumulative** reveal: frame 0 is permanently opaque as a base layer, and
every later frame fades in *on top* and then stays opaque. The canvas is
covered by frame 0 at minimum, always, so there is no moment where nothing is
opaque. (This repo's `npm run check` and the render-timeline check described
under [Verification](#verification) both confirm this holds at every sampled
instant across the 6-second reveal.)

**If both count sources fail at once, the endpoint holds the last known-good
value instead of showing 0.** A counter that 500s or briefly shows `0` reads
as more broken than one that's a few seconds stale. `lib/counter.js` keeps
the last successful count in module scope, which survives for as long as the
serverless instance stays warm. The one case this can't cover: a cold
instance that has never resolved a count yet, hit during a moment when both
Komarev and KV are simultaneously unreachable, has nothing cached and will
show 0. That's an acknowledged gap, not a hidden one.

## Frames

The reveal is 10 hand-tuned-parameters, procedurally generated frames (see
[Bring your own art](#bring-your-own-art)):

<div align="center">
  <img src="piskel_frames/spritesheet_2560x128.png" alt="10-frame Crimson Moon reveal spritesheet" width="100%" />
</div>

## Bring your own art

`piskel_frames/banner_generator.py` is parametric — regenerate your own
sequence rather than reusing these exact frames, which are tuned to one
specific palette and seed. The renderer (`lib/render.js`) doesn't measure
anything about the art at runtime; it trusts a fixed contract instead:

- **256×128 pixels**, exactly. `lib/render.js` hardcodes `ART_W`/`ART_H` and
  the SVG `viewBox`.
- **Fully opaque, every frame, every pixel.** No alpha channel anywhere. The
  cumulative reveal (see [Findings](#findings--why-its-built-this-way)) is
  only flash-free because frame 0 is guaranteed fully opaque — a frame with
  any transparency reintroduces the exact bug that design avoids.
- **A single fixed palette across all frames.** `banner_generator.py`
  quantizes every frame against one hardcoded palette with no dithering, so
  adjacent frames never introduce a color the previous one didn't have —
  that's what keeps the cross-fade from ever looking like it's cutting
  between two different images.
- **A flat, single-color inscription zone: 90×28, centered at (128, 64),**
  that must render as one solid color in every frame where the disc is
  visible enough to read text on. `lib/render.js` reads this as the
  constants `ZONE = { cx: 128, cy: 64, w: 90, h: 28 }` — it does not detect
  or measure the zone from the image, so if your art doesn't keep that
  rectangle flat, the inscribed count will render on top of whatever
  gradient or texture is actually there.

If you regenerate, note that `banner_generator.py`'s `__main__` block writes
frames to a hardcoded path (`/home/claude/work/b{f:02d}.png` in this repo's
copy) — that path is an artifact of the environment it was originally
authored in, not a project convention. Point it at `piskel_frames/` (or
wherever you keep your frames) before running it, then re-run the base64
encode step that produces `lib/frames.js`.

## Local development

```bash
npm run dev        # serve on :3000 — /api/counter and /api/counter?debug=1
npm run check       # selftest: per-view increment, 10 frames, inscription containment, no-store header
npm run snapshot      # regenerate souls_view_counter.svg as a static local preview (NOT used by the live badge)
```

`npm run check` runs the real exported handler against a throwaway local
stand-in for the Upstash REST API, so a passing check is evidence about the
actual deployed code path, not a parallel mock.

## Verification

Beyond `npm run check`, this repo's render logic was additionally checked by
compositing the 10 vendored frame PNGs using the exact opacity math from
`lib/render.js`'s keyframes, sampled at 0.1s steps across the full 6-second
reveal (plus post-reveal holds), against a background color absent from the
art's own palette. Zero background-colored pixels appeared at any sampled
instant — confirming the cumulative-reveal fix in
[Findings](#findings--why-its-built-this-way) actually holds frame-by-frame,
not just by design intent.

## Deploy

```bash
vercel deploy --prod
```

Run from the repo root. That's the whole deploy — there's no build step.

**Per-view vs. per-visitor counting.** Without any extra configuration, the
endpoint reads Komarev only, which counts distinct visitors (see
[Findings](#findings--why-its-built-this-way)). To get a number that advances
on every single view, set up a KV store and provide either:

- `KV_REST_API_URL` + `KV_REST_API_TOKEN` (Vercel KV / Vercel Marketplace
  Upstash integration — these env vars are set automatically if you link a KV
  store to the project in the Vercel dashboard), or
- `UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN` (a standalone Upstash
  Redis database, same REST protocol).

Optional additional env vars:

| Variable | Default | Purpose |
|---|---|---|
| `COUNTER_KEY` | `anbu:profile-views` | KV key the counter increments. Set your own so you don't collide with anyone else's namespace if you share a KV instance. |
| `COUNTER_OFFSET` | `0` | Added to the displayed count — useful if you're migrating from a counter that already had a running total. |
| `COUNTER_CAPTION` | `BEHELD BY WANDERERS` | The caption line drawn above the number. |

The username is **not** an env var — it's a query param, so one deployment
can serve any GitHub username:

```
https://<your-deployment>.vercel.app/api/counter?username=<your-github-username>
```

If you omit `?username=`, it falls back to the original author's username
(`Anbu-00001`) — set yours explicitly.

**Practical ceiling:** Vercel's Hobby plan caps function invocations around
1M/month. Each page view of a README that embeds this is one invocation, so
that's the effective traffic limit before you'd need a paid plan.

## License

[MIT](LICENSE) — see also [CREDITS.md](CREDITS.md) for the data source, prior
art, file-format tooling, and build-time-only dependencies this project
builds on.
