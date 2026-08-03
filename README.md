# Liquid Light — LL-1 Overhead Stage

A browser-based liquid light show. Colored dyes are dropped onto a simulated
overhead projector stage, spread outward as circular films, and are stirred by
an incompressible fluid bath underneath. Light passes *through* the film rather
than being emitted by it, so the image behaves like an oil-wheel projection
rather than a glowing particle effect.

Single standalone HTML file. No build step, no dependencies, no network access.
Open it and it runs.

![status](https://img.shields.io/badge/status-working-brightgreen)

---

## Running it

Open `liquid-light.html` in a browser. That's the whole procedure.

It can be served from any static host, opened from `file://`, or embedded in an
iframe. Nothing is fetched at runtime and nothing is stored.

**Requirements:** WebGL2 with a floating-point or half-float renderable color
buffer. That covers Safari 15+, Chrome/Edge 56+, Firefox 51+, and iOS Safari
15+. If neither `RGBA16F` nor `RGBA32F` can be attached to a framebuffer, the
page shows a message instead of a blank canvas.

---

## Using it

| Gesture | Result |
| --- | --- |
| Double-click / double-tap | Drop dye at that point |
| Click and drag | Stir the bath — moves the film without adding dye |
| Drag with **Dragging also paints dye** on | Stir and lay down a trail |

Keyboard: `space` toggles self-dripping, `C` wipes the glass, `H` hides the
console, `F` goes full screen.

The double-tap detector is custom rather than the native `dblclick` event, so
mouse and touch behave identically: two presses within 380 ms and 4.5% of the
canvas width of each other.

---

## Controls

### Dye palette

Six dyes. Click a swatch to select it, then use **Recolor selected dye** to
change it. With **Advance dye each drop** on, successive drops cycle through
the palette; off, every drop uses the selected dye.

A dye's color is what it transmits, so bright saturated dyes read as bright
saturated color and a near-white dye is nearly invisible film.

### Glass & lamp

| Control | What it does |
| --- | --- |
| **Lamp** | Color of the light under the stage. Default is a warm tungsten-halide white. |
| **Clear bath tint** | Tint of the clear suspension the dyes sit in. Near-white by default; push it toward a hue to color the whole field. |
| **Lamp hot spot** | Falloff from the center of the stage to the corners. |
| **Fresnel rings** | Visibility of the concentric grooves moulded into the stage glass. Zero for clean optical glass. |
| **Wet sheen** | Highlight on the film surface, and how strongly the film's curvature focuses the lamp into bright cores. |

### Film on the glass

| Control | What it does |
| --- | --- |
| **Spread** | How hard a drop wets outward. Every drop opens as a circle and slows as it thins. |
| **Merge rate** | How fast touching colors blend into a third. Low keeps each dye beaded and distinct. |
| **Drop size** | Radius of a new bead. |
| **Fade** | Rate at which dye clears off the glass. |

### Currents

| Control | What it does |
| --- | --- |
| **Stir force** | Velocity a drag imparts to the bath. |
| **Swirl** | Vorticity confinement. Low reads as still glass; high marbles the film. |
| **Viscosity** | How quickly the bath calms after being stirred. |
| **Stage tilt** | Lifts one edge of the glass so the film runs. Zero is level. |

---

## Presets worth knowing

**Beaded oil wheel** — Merge 0, Spread 55, Swirl 0, Viscosity 60. Dyes hold
saturated color and only deform where they collide, like immiscible oils on a
clock glass.

**Marbled** — Merge 20, Spread 35, Swirl 55, Stir force 70. Drag continuously;
the film folds into itself and makes the classic Fillmore-era marbling.

**Wash** — Merge 70, Spread 70, Fade 25. Colors bleed into new hues and clear
slowly, leaving a soft moving wash.

**Running glass** — Stage tilt −60, Spread 30, Viscosity 25. The film runs
downhill and pools at the bottom of the field.

---

## Notes

The simulation is unconditionally stable at any control setting — all the
diffusion coefficients are clamped below their stability limits and the
timestep is capped at 1/60 s regardless of frame rate. There is no setting
combination that blows the field up.

Resizing the window reallocates the simulation textures, which clears the
glass. That's deliberate: the fields are aspect-dependent.

For the pass structure, the shader math, and the two physical models that were
tried and abandoned, see [DEVELOPMENT.md](DEVELOPMENT.md).

---

## License

Code is released under the GNU General Public License v3.0. See `LICENSE`.

Any dye palettes, presets, or recorded output distributed alongside it are not
covered by the GPL and remain proprietary unless stated otherwise in that
material's own notice.
