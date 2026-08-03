# Liquid Light — Development Notes

Technical reference for `liquid-light.html`. Covers the pipeline, the data
layout, the shader math, the models that were tried and rejected, and where to
change things.

---

## 1. Structure of the file

One HTML file, three sections:

1. **CSS** — the console chrome. Deliberately monochrome; the dye swatches are
   the only color on the panel, and the selected swatch drives the `--accent`
   custom property, so the panel's one accent is always the dye you're about to
   drop.
2. **Markup** — canvas, collapse tab, console, hint, failure message.
3. **Script** — one IIFE. Format probing, shader compilation, FBO allocation,
   simulation step, render, input, console wiring, main loop.

There is no module system and no build. Everything is in scope of the IIFE.

---

## 2. GPU resources

### Format probing

`supports(internalFormat, format, type)` allocates a 4×4 texture, attaches it to
a framebuffer, and checks completeness. `RGBA16F`/`HALF_FLOAT` is preferred;
`RGBA32F`/`FLOAT` is the fallback. Extensions are requested before probing:

- `EXT_color_buffer_float`
- `EXT_color_buffer_half_float` — the one iOS Safari needs
- `OES_texture_float_linear` — only relevant on the 32-bit path

Half-float is preferred not only for bandwidth: in WebGL2, `HALF_FLOAT`
textures are filterable in core, whereas `FLOAT` filtering requires the
extension above. Advection depends on hardware bilinear sampling, so a
non-filterable format would silently degrade to nearest-neighbour smearing.

Shaders are GLSL ES 1.00, not `#version 300 es`. WebGL2 accepts 1.00 sources
while still exposing the sized internal formats, which keeps the code readable
without giving up `RG16F` / `R16F`.

### Textures

| Target | Resolution | Format | Filter | Contents |
| --- | --- | --- | --- | --- |
| `dye` | `DYE_RES` = 1024 | RGBA16F | LINEAR | `rgb` = tint × thickness, `a` = thickness |
| `velocity` | `SIM_RES` = 160 | RG16F | LINEAR | bath velocity, texels/second |
| `pressure` | `SIM_RES` | R16F | NEAREST | Jacobi pressure |
| `divergence` | `SIM_RES` | R16F | NEAREST | ∇·v |
| `curl` | `SIM_RES` | R16F | NEAREST | ∂v_y/∂x − ∂v_x/∂y |

`dye`, `velocity` and `pressure` are double-buffered (`createDouble`) since
every pass reads and writes the same field. `resolution()` fits the requested
short-edge resolution to the canvas aspect, so texels stay square and the
spreading stays circular. Non-square texels would make drops elliptical.

**Dye is stored premultiplied** (`rgb = tint · h`). Every operation applied to
it — advection, diffusion, fading — is linear, and linear operations on
premultiplied color are correct where alpha varies. Storing straight tint
alongside thickness would smear hues at every boundary.

The vertex shader supplies `vUv` plus the four axis neighbours `vL/vR/vT/vB`
from `texelSize`. Passes that need diagonals compute them in the fragment
shader from the same uniform.

---

## 3. Pass order

Per frame, `step(dt)` runs:

```
curl          velocity            -> curl
vorticity     velocity, curl      -> velocity      confinement, scaled by Swirl
tilt          velocity, dye       -> velocity      skipped when tilt == 0
divergence    velocity            -> divergence
clear         pressure            -> pressure      decay by 0.8, warm start
pressure ×22  pressure, divergence-> pressure      Jacobi
gradient      pressure, velocity  -> velocity      projection
advect        velocity            -> velocity      self-advection + viscosity
film          velocity, dye       -> dye           transport + wetting
merge         dye                 -> dye           pigment blending
```

then `render()` composites `dye` to the default framebuffer.

Order matters in two places. Projection must come before advection, or the
velocity field carries dye through a divergent field and mass appears from
nowhere. And `film` must come before `merge`, so that merging operates on dye
that has already moved this frame; the reverse order visibly lags the color
behind the motion.

`dt` is clamped to 1/60 s. On a 120 Hz display the simulation therefore runs at
real time or slower, never faster — a slow frame can't produce a large step
that destabilizes the advection.

---

## 4. The fluid bath

Standard stable-fluids on the velocity field. Semi-Lagrangian advection,
divergence, 22 Jacobi iterations, gradient subtraction. Two departures worth
noting:

**Pressure warm start.** Rather than zeroing pressure each frame, the previous
solution is decayed by 0.8 and used as the initial guess. Jacobi converges
slowly; reusing the previous frame's field gets far closer for free. The decay
prevents stale pressure from accumulating.

**Boundary handling** is in the divergence pass: where a neighbour sample falls
outside `[0,1]`, the center velocity is reflected, giving free-slip walls at
the edge of the stage.

Velocity is in *texels per second*, which is why advection multiplies by
`texelSize`. This is what lets the velocity field be sampled at dye resolution
without rescaling — the same numbers mean the same physical speed at either
resolution.

---

## 5. The film — dye transport and wetting

This is the pass that took the most iteration and the one to be careful about.

Current model, in `filmP`:

```glsl
coord = vUv - dt * velocity * texelSize;     // carried by the bath
c     = sample(coord);
s     = 3×3 binomial blur around coord, offsets 3 texels;
k     = clamp(spread * clamp(c.a, 0, 1.5) * dt, 0, 0.22);
out   = (c + (s - c) * k) * dissipation;
```

Wetting is **thickness-weighted isotropic diffusion**. Thick film diffuses
fast, thin film barely moves, so a bead opens outward as a circle and slows as
it thins — a drop wetting a plate and settling. Three properties matter:

- It conserves dye, so it cannot manufacture material.
- Diffusion strictly *removes* high-frequency content, so it cannot manufacture
  detail either.
- `k` is clamped to 0.22, well under the 0.25 explicit-diffusion stability
  limit for this kernel, so no control setting can destabilize it.

The blur kernel is the 3×3 binomial (center 4, edges 2, corners 1, ÷16) at a
3-texel offset. The corner taps matter: a 4-point cross kernel is anisotropic,
favors the texel axes, and lets an expanding rim finger into diamond-shaped
noise. The offset of 3 texels rather than 1 sets the wetting length scale well
above the noise floor, which is what makes the expansion read as smooth.

### Rejected model 1: film-pressure advection

Earlier versions used the physical thin-film wetting velocity

```
v_s = −k·h·∇h
```

applied to the dye transport only — deliberately outside the incompressible
projection, because that projection would otherwise cancel exactly the radial
expansion being asked for. Mass was corrected with a continuity factor
`(1 − dt·∇·v_s)`.

This is the more physically faithful model and it produces a genuine contact
line and rim. It also fails in practice. Where the spreading flow converges
just inside the rim, material piles up; the pile-up steepens its own gradient,
which strengthens the flow that created it. Any texel-scale variation grows
into visible grain within a second, and the `|∇h|²` term contributes
`∇·v_s < 0` everywhere there is a gradient, so the runaway is not even
conditional. Clamping the continuity factor bounds the rate but not the
direction — it just saturates.

Abandoned in favor of diffusion, which trades the sharp contact line for
unconditional smoothness. The sharpness is recovered optically instead (§6).

### Rejected model 2: subtractive absorbance stacking

Dye was originally stored as per-channel optical density, with the display
doing Beer-Lambert transmission `lamp · exp(−A)`. Drops added absorbance, so
overlapping dyes stacked like gel filters and mixed subtractively — correct
physics, and yellow over cyan really does give green.

It also drives every complementary pair to black: red dye blocks green and
blue, cyan dye blocks red, and stacked they block everything. Real oil-wheel
projections don't do this because the oils are immiscible and sit *beside* each
other rather than stacking.

Replaced by thickness-weighted tint averaging, where meeting dyes produce a
third hue and the darkest possible result is the darkest dye in the palette.

### The merge pass

Uniform 3×3 binomial blur, mixed by `amount`:

```js
amount = (0.002 + merge^1.6 * 0.45) * (dt * 60)
```

The floor is small but nonzero to keep boundaries from aliasing. An earlier
version mapped low settings to a *negative* amount — anti-diffusion, to
simulate immiscibility by sharpening. It sharpened noise just as effectively as
edges and was a significant source of grain. Immiscibility is now expressed by
the absence of diffusion rather than by its reversal.

`spread` and `merge` are both diffusion, distinguished by weighting: spread
scales with thickness and only moves thick beads, merge is uniform and blends
hue everywhere. That's why low merge with high spread gives expanding beads
that keep their color.

---

## 6. The display pass

The stage is transmissive: a lamp field beneath Fresnel glass, filtered by the
film.

```
hotspot   1 - hot · smoothstep(0.05, 0.95, r)
grooves   mix(sin(2πφ), sawtooth(φ), 0.30),  φ = r · 34
light     lamp · hotspot · (1 + rings · grooves)
caustic   light *= clamp(1 - ∇²h · (18 + 42·sheen), 0.55, 1.9)
tint      c.rgb / max(h, 0.045)
          saturation ×1.30 about luminance
          normalized to peak channel, floor 0.72
coverage  dens = smoothstep(0.012, 0.20, h)
color     light · mix(white, tint, dens · 0.94) · bath
specular  + sheen · 0.30 · (n·l)^32,  n from the thickness gradient
edge tint + sheen · 0.12 · |∇h| · cool blue
falloff   × smoothstep(0.95, 0.30, r)
```

Three deliberate choices:

**Coverage is remapped, not proportional.** `dens = smoothstep(0.012, 0.20, h)`
means the film reads as fully tinted while any real thickness remains, fading
only when nearly gone. An earlier `1 − exp(−1.9h)` tied opacity directly to
thickness, so a drop washed out toward the white lamp exactly as it spread —
the drops went grey. The remap also re-sharpens the physically soft diffusion
edge into a crisp circular boundary. **Soft in the simulation, defined in the
projection.** This is what buys back the contact line lost in §5.

**Chroma recovery.** Averaging tints pulls toward neutral wherever colors meet,
so the display pushes saturation back out by 1.30× about luminance and
normalizes to the peak channel. Complementary pairs still neutralize — that is
what mixed pigment does — but everything else stays vivid.

**The tint divisor has a floor of 0.045.** Unpremultiplying with a smaller
guard divides two near-zero half-floats in the empty regions and produces
colored hash. The floor sits below the coverage threshold, so the guarded
region is invisible anyway.

The caustic term (`1 − ∇²h`) brightens where the film is domed and darkens at
steep rims, which is what oil on a projector actually does to transmitted
light. Its gain is deliberately modest; it multiplies second derivatives, so it
is the fastest way to reintroduce sparkle if turned up.

---

## 7. Input

Pointer events throughout, so mouse, pen and touch share one path.
`setPointerCapture` on pointerdown keeps a drag alive when the cursor leaves the
canvas — an early version also listened for `pointerleave`, which cancelled
drags mid-stroke.

Double-tap: two pointerdowns within 380 ms and 0.045 UV of each other. Native
`dblclick` is suppressed rather than used, so touch and mouse are identical.

Drag imparts `(600 + force·5400) × Δuv` texels/second. The x component is
multiplied by aspect ratio so a diagonal drag pushes at the angle drawn.

A drop is two passes: a dye bead at `0.55 × dropRadius`, and one radial impulse
pass. The impulse shader computes an exact outward direction field around the
point — an earlier version approximated it with eight discrete splats at
randomized angles, which was visibly lopsided.

---

## 8. Tuning reference

| Constant | Location | Effect |
| --- | --- | --- |
| `SIM_RES = 160` | fbo setup | Velocity detail. 256 gives finer turbulence at real cost — every Jacobi iteration pays. |
| `DYE_RES = 1024` | fbo setup | Film detail. The main memory and fill consumer. |
| `pressureIters: 22` | config | Incompressibility. Below ~15, dye visibly compresses; above ~30, no visible gain. |
| `spread × 42` | `step()` | Wetting rate. Clamped at `k = 0.22`, so raising it only makes the slider top out sooner. |
| `texelSize × 3.0` | `filmP` | Wetting length scale. Lower is grainier, higher is mushier. |
| `swirl × 46` | `step()` | Vorticity confinement gain. |
| `0.22` clamp on `k` | `filmP` | Stability margin. Do not raise past 0.25. |
| `smoothstep(0.012, 0.20, h)` | `displayP` | Coverage curve. Widen the range to make drops fade more gradually. |
| `1.30` saturation | `displayP` | Chroma recovery. Above ~1.6, blends posterize. |

---

## 9. Known limitations

- **Resize clears the glass.** The fields are aspect-fitted and reallocated on
  resize. Preserving state would mean resampling five textures through a blit
  pass.
- **Complementary dyes neutralize.** Inherent to tint averaging. True
  per-pigment subtractive mixing would need a Kubelka-Munk model over separate
  pigment channels — realistic, considerably more expensive, and it brings back
  the darkening the current model exists to avoid.
- **No state persistence.** No presets are saved. `config` is a plain object;
  serializing it to a URL hash or to artifact storage is straightforward.
- **No output capture.** `preserveDrawingBuffer` is off for performance, so
  `toDataURL` returns black. Capture needs either that flag or a
  `readPixels` immediately after render.
- **Single dye layer.** Dyes cannot sit above one another; there is one film.
  Layered wheels would need a stack of dye textures composited in order.

---

## 10. Possible directions

- **Audio reactivity.** An `AnalyserNode` driving Spread, Swirl and drop
  triggers from onsets; the pass structure already isolates each as a scalar.
- **MIDI control.** Web MIDI mapping CCs onto `config` — the sliders are the
  only writers, so a second writer needs no restructuring.
- **Multi-projector output.** Rendering the display pass to several viewports
  with different lamp fields, for multi-screen or edge-blended projection.
- **Preset recall with interpolation.** Crossfading between two `config`
  objects over a musical duration, rather than jumping.
- **Ambient dripping patterns.** The current self-drip is uniform random;
  seeded rhythmic or spatial patterns would suit a performance context better.
