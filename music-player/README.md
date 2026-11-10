# Tideline Player

A standalone music player, separate from the PS5 savings app. It rebuilds two
mechanics of a streaming web player: the background color that follows the
cover art, and the now-playing bar. Open `index.html` in a browser. Three
synthesized demo tracks load right away, and **+ Add your music** plays your
own files.

## How the original works

Source: the public open.spotify.com web player bundle, captured on 2026-11-10
and analyzed with REA (`analyze-javascript-application`, Evidence
`ev_8e8e280a294a6f83fda3df45762daf8757ef4f19087355f948d48488fb10717f`) plus
direct reading of the minified chunks. Static analysis only: I read the code but
didn't watch it run or log in.

### Cover-art color (observed)

- **The colors come from a server, not the browser.** The client sends image
  URIs to a GraphQL persisted query named `fetchExtractedColors`
  (`{imageUris: [...]}`, batched), with an optional country header.
- **Each image gets three colors:** `colorRaw`, `colorDark` and `colorLight`,
  each a hex value with an `isFallback` flag.
- **Results are cached twice.** An in-memory map is keyed by image URI, and a
  React Query entry `["extracted-color", uri]` is fresh for 6 hours
  (`staleTime: 216e5`) and kept for 24 hours (`gcTime: 864e5`).
- **Failures fall back to a default palette.** On a failed request, or a result
  marked all-fallback, it uses a default triple and reports the miss.
- **Inferred:** the page tints its header gradient with `colorDark`. That's how
  the CSS variables are used; I didn't confirm it per component.

### Now-playing bar (observed)

- **Player state holds an anchor, not a ticking position.** It carries
  `timestamp`, `positionAsOfTimestamp`, `playbackSpeed`, `isPlaying`,
  `isPaused`, `isBuffering` and `duration`, encoded as protobuf.
- **Inferred:** the UI works out the live position as
  `positionAsOfTimestamp + (now − timestamp) × speed` while playing. The fields
  point to this formula, but I didn't find the exact line.
- **Seeking happens on release.** The seek bar (`data-testid="playback-progressbar"`)
  previews while you drag, then on release seeks once to `fraction × duration`
  and logs whether it was a click (`hitSeekToTime`) or a drag
  (`dragSeekToTime`).
- **Hover previews are throttled.** Thumbnails and chapter labels follow a 50 ms
  debounced position.

**Not covered:** the 711 KB vendor chunk was left out after the full-bundle
analysis hit its 5-minute limit. Server-side color extraction is invisible from
the client.

## What this app does differently

| Original | Here | Why |
| --- | --- | --- |
| Server extracts colors | Browser extracts them from a 48×48 canvas, scoring color buckets by count weighted toward saturated mid-tones | No backend |
| Raw / dark / light from the server | `dark` = cover hue at 24% lightness, `light` = 78% lightness | Keeps white text readable on any cover |
| Fixed brand accent | Accent is the cover's `light` color | Its own look |
| Remote playback state | `<audio>` is the source of truth; the anchor model is re-set on every media event and interpolated each frame | Same smooth bar, local playback |

Same as the original: per-URL color cache with a fallback, click vs drag
seeking committed on release, previous restarts the track after 3 s, and
lock-screen controls through the Media Session API. Plus shuffle, repeat,
volume, keyboard control (space, arrow keys on the sliders) and a phone layout.
