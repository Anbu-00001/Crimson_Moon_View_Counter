# Credits

## Runtime dependency

This project has **zero runtime dependencies**. `api/counter.js` and everything
it imports use only Node.js built-ins (`node:http` in the dev server) and the
global `fetch`. Nothing is installed to deploy or serve the counter — that is
a genuine property of this repo, not an oversight in `package.json`.

## Data source

- **[antonkomarev/github-profile-views-counter](https://github.com/antonkomarev/github-profile-views-counter)** (MIT).
  The live view-count data source, read over plain HTTP at request time — no
  code from this project is vendored here. The counter depends on Komarev's
  free hosted service staying up; see the README for how it's used as a floor
  rather than proxied directly.

## Prior art

- **[journey-ad/Moe-Counter](https://github.com/journey-ad/Moe-Counter)** (MIT).
  The closest comparable project (a themeable animated SVG view counter) and
  useful prior art while designing this one. No code from it is used here.

## Tooling / format

- **[piskelapp/piskel](https://github.com/piskelapp/piskel)** (Apache-2.0).
  `piskel_frames/crimson_moon_banner.piskel` uses Piskel's `.piskel` JSON
  container format so the frame sequence can be opened and edited in the
  Piskel editor. No Piskel source code is vendored — only the file format is
  used.

## Build-time only (not shipped, not runtime dependencies)

`piskel_frames/banner_generator.py` — the script that procedurally generates
the 10 frame PNGs — is run manually, offline, and only when regenerating
artwork. It depends on:

- **NumPy** — array math for the palette, gradients, and masks.
- **Pillow** — PNG encoding and palette quantization.
- **SciPy** (`scipy.ndimage`) — distance transforms and morphology for the
  glow/bloom falloff.

None of these are installed by `npm install`, listed in `package.json`, or
required to run, test, or deploy the counter itself.

## No webfont

No web font is used or bundled anywhere in this repo. An earlier version of
the numeral/caption rendering tried a Google Fonts `@import` (Press Start 2P)
and it was removed deliberately: GitHub renders README images inside an
`<img>` via its Camo proxy, which blocks all external resource loads,
including font `@import`s. See `lib/pixelfont.js` and the README for the
bitmap-font replacement.
