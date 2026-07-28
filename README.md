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
<kbd>S</kbd> save · <kbd>H</kbd> hide panel · <kbd>1</kbd>–<kbd>9</kbd> presets.

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
- `pick up image colour` samples a 64px thumbnail read once at load, so the dye
  takes the photo's own colours without ever reading back from the GPU.
- Once an image is up, the dropzone shrinks to a corner chip — still a live drop
  target, no longer occupying the stage.
- Nothing leaves the browser. The image is decoded locally into a canvas.
