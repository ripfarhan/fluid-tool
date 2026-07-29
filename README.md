# LIQUID

Drop an image in, push it around with the cursor.

The GPU fluid solver from the liquid headings on [karamat.work](https://karamat.work), pointed at
arbitrary images instead of text. Real Navier–Stokes, not a noise warp:

```
advect → splat → curl → vorticity confinement → divergence → pressure Jacobi → gradient subtract
```

running every frame in half-float FBOs, with the resulting velocity field used
to displace the image.

## Running it

Double-click `index.html`. That's it — one self-contained file, no build, no
dependencies, no server. Needs a browser with **WebGL2 + `EXT_color_buffer_float`**
(any current Chrome/Edge/Firefox/Safari); it says so plainly if that's missing.

To serve it instead (or to deploy — it's just a static file):

```bash
python -m http.server 4173
```

## Controls

Move the cursor over the image to push real momentum into the fluid, click for a
radial burst. Leave it alone and a slow wandering point keeps it breathing.

**Presets** — 19 of them, from `SIGNATURE` (the exact values the site's headings
use) through `INK BLOOM`, `CHROME`, `SMOKE`, `PLASMA`, `GLASS`, `TAR`, `MIRAGE`,
`OIL SLICK` to `MELTDOWN`. Each is a full state, so switching is deterministic.
**Surprise me** rolls a fresh one inside tasteful ranges.

| group | dials |
| --- | --- |
| Flow | warp (displacement ceiling, % of width), flow follow, vorticity/curl, velocity decay, drift force + angle, sim resolution, pressure iterations |
| Brush | brush size, brush force, dye per stroke, resting swirl, click impulse |
| Dye | dye source (fixed / hue cycle / hue by direction / pick up image colour), colour, hue speed, blend (glow / screen / ink stain / off), intensity, decay, softness |
| Look | chromatic split, ambient wobble + speed, exposure, contrast, saturation, grain, vignette, view (composite / dye field / velocity field) |
| Motion | idle energy, burst hold, idle wander + force + speed, freeze solver |

Double-click any slider row to reset that one dial. Dials the current mode
ignores are dimmed rather than hidden.

Buttons: **Save PNG** (current frame, named after the active preset), **Calm**
(zero the velocity/dye fields without reloading the image), **Reset all**.

Keys: <kbd>space</kbd> freeze · <kbd>C</kbd> calm · <kbd>R</kbd> random ·
<kbd>S</kbd> save PNG · <kbd>V</kbd> record · <kbd>H</kbd> hide panel ·
<kbd>1</kbd>–<kbd>9</kbd> presets.

## Recording

**Record** runs a countdown (default 5s — adjustable 0–15, `off` starts
instantly) so you can get the cursor onto the image, then captures until you
press **Stop**. Only the image is recorded; no UI, no cursor.

| dial | default | notes |
| --- | --- | --- |
| countdown | 5s | pre-roll before capture starts; `0` = immediate |
| frame rate | 60 fps | also 30 / 24 |
| format | auto | prefers MP4/H.264, falls back to WebM/VP9 → VP8 |
| quality | high | bits-per-pixel-per-frame: web 0.06, high 0.15, max 0.3 |

Encoding details that matter:

- Frames come from a separate 2D canvas that composites onto an **opaque**
  background, so images with alpha don't record as black or pick up premultiply
  fringing.
- That canvas holds **one fixed, even-numbered size** for the whole take. H.264
  requires even dimensions, and changing a track's resolution mid-recording
  upsets muxers — so resizing the window mid-take can't corrupt the file.
- `captureStream(0)` + `requestFrame()` paces frames off the render loop rather
  than letting the browser sample whenever it likes.
- MediaRecorder timestamps frames by arrival, so the file's timeline is real
  wall-clock time and **playback speed is always correct** — even if the loop
  can't hold the target rate. When it can't, the status line says so
  (`ran at 34 fps`) instead of pretending.
- The take is assembled on whichever lands last, the flush chunk or `onstop`
  plus a grace period. VP9 can emit its first chunk *after* `onstop`, and
  finalising on `onstop` alone yields an empty file.
- Chunked at 1s (bounded memory), auto-stops at 10 min, and banks what it has if
  the tab is hidden — `requestAnimationFrame` stops when hidden, so continuing
  would only record a freeze frame.
- **WebM caveat:** browsers write MediaRecorder WebM without a duration tag, so
  some editors read the clip as unbounded. MP4 has no such problem, which is why
  `auto` prefers it. The panel says so when you pick WebM.

## The one deliberate difference from the headings

The headings derive their edge sheen from glyph **alpha**, which is fine for text
on transparency but cancels to nothing on an opaque photo. Images use a true
per-channel (RGB) offset along the flow instead — the `chromatic split` dial.

## Notes

- Uploads are downscaled to 1600px on the long edge so large photos don't stall.
- The simulation grid defaults to 144px wide (height follows the image aspect) —
  the fluid is deliberately low-res; it's the displacement that's full-res. The
  `sim resolution` dial goes 64–288 and reallocates the FBOs on the fly.
- Dye carries colour in RGB and coverage in alpha, so a near-black ink reads as
  *thick* rather than as *faded* — that's what makes the `ink` blend work.
- `drift force` is added after the gradient subtract: a uniform push is already
  divergence-free, so it costs nothing and stays stable.
- The brush impulse comes from the pointer's **speed** (uv/second, clamped to 6
  widths/s), not from the raw gap between two pointer events. Feeding the
  per-event delta in directly — as the headings do — makes the force depend on
  event coalescing instead of hand speed: one long frame or a re-entry across
  the canvas lands a ~110× impulse spike that shears the whole field into a
  single gust. Hops over 0.35 widths in one event are treated as a teleport and
  re-seed the stroke rather than being splatted.
- The walls are no-through-flow (the normal velocity component is zeroed in the
  border texel). Without it, momentum driven into an edge is held there,
  `CLAMP_TO_EDGE` sampling feeds it back in, and the projection spreads the
  pile-up across the whole field.
- `pick up image colour` samples a 64px thumbnail read once at load, so the dye
  takes the photo's own colours without ever reading back from the GPU.
- Once an image is up, the dropzone shrinks to a corner chip — still a live drop
  target, no longer occupying the stage.
- Nothing leaves the browser. The image is decoded locally into a canvas.
