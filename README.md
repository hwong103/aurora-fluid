# Aurora Fluid

A GPU-accelerated fluid simulation in a single HTML file. No frameworks, no
build step, no external libraries — just WebGL and GLSL.

Drag the mouse or a finger to push colorful dye through the fluid.

## How it works

A Navier–Stokes solver runs entirely on the GPU. Every step is a fragment
shader pass over ping-ponged float framebuffers:

1. **Curl** — measure the local vorticity of the velocity field.
2. **Vorticity confinement** — push energy back into the small eddies that
   semi-Lagrangian advection smears away.
3. **Divergence** — measure how much the field is compressing.
4. **Pressure** — 20 Jacobi iterations solve for the pressure that makes the
   flow divergence-free.
5. **Gradient subtract** — project the velocity field back onto the
   incompressible subspace.
6. **Advect** — move velocity and dye along the flow, with dissipation.
7. **Bloom** — bright-pass, then a separable gaussian, which produces the glow.
8. **Composite** — tone-map, vignette, and dither.

The simulation grid (128) and dye grid (512) are deliberately much coarser
than the display. That is the single biggest performance lever and it is
visually indistinguishable for a flow this diffuse.

The dye is advected with a **MacCormack** scheme rather than plain
semi-Lagrangian advection. Plain semi-Lagrangian is stable but blurs: a thin
filament of dye diffuses away within a few frames. MacCormack advects forward,
advects that result backward, and applies the second-order correction, clamped
to the neighbourhood the sample came from. It costs two extra passes and buys
roughly 40% more visible detail in the dye.

Colour is handled carefully in two places. The palette walks the full hue
circle rather than a single family, so strokes read as a spectrum rather than
various shades of blue. And the tone-map is driven by luminance rather than
applied per channel, so bright dye desaturates gradually instead of clipping
to flat white and taking its hue with it.

## Compatibility

WebGL2 is used when available and WebGL1 otherwise, so the demo runs on older
mobile browsers. Render targets are `RGBA16F` when
`EXT_color_buffer_float`/`EXT_color_buffer_half_float` is present, and fall
back to `RGBA8` when it is not.

## Why finger drags used to turn white

Not clipping — hue mixing. Two separate things were pushing strokes toward
white, and only one of them is a magnitude problem.

**The real cause: the colour wheel was being averaged.** Splat footprints
overlap heavily. A slow drag stacks roughly a dozen splats inside a single
texel, and the old code advanced the hue by ~0.06 *per splat*, freely across
the whole wheel. So every individual texel on the stroke was summing nearly the
entire colour wheel. Summing wheel-wide hues lifts R, G and B together, the
stroke converges on grey, and the tone-map renders grey as white. The field
value there was enormous — max channel 23.8 — and completely colourless.

The fix ties the sweep to the gesture: `newStroke()` re-anchors the hue when a
stroke begins, and the per-splat step drops from ~0.06 to ~0.02, so overlapping
splats stay in one arc of the wheel and the stroke keeps its colour. Measured
on a 390x844 viewport, the minimum saturation anywhere along a slow drag went
from 0.001 to 0.776.

**The secondary cause: a per-channel ceiling.** The soft knee that was already
in the splat shader compressed each channel toward the ceiling independently.
A bright channel and a dim one were squeezed toward the same number, which
drains the saturation out of exactly the stroke the ceiling was meant to
protect. The ceiling now scales the whole RGB vector by one factor, so the
channel ratios — and the hue — survive untouched. A ceiling can bound
magnitude; it can never fix hue.

**Measured on a 390x844 phone viewport, slow one-finger drag:**
white pixels 32.9% -> 0.57%, saturation 41.7% -> 69.1%, and the number of
distinct hues on screen went up, because strokes stay separate colours instead
of merging into one white smear. Worst case across every gesture tested
(slow, fast flick, finger held in one spot, two-finger) is 4.5% white.

## Multi-touch

Any number of fingers work at once, up to whatever the hardware reports. Three
details make it hold up:

- **One stroke, many fingers.** The colour sweep is anchored when the first
  finger lands, not per finger. Ten fingers anchoring separately restarted the
  sweep ten times, so the hand averaged ten unrelated hues into grey.
- **The hue sweep is per frame, not per splat.** Divided by the live finger
  count, so the colour a stroke travels through in a second is the same whether
  one finger is down or ten.
- **Dye strength is normalised by finger count**, as `1/sqrt(n)`. Ten fingers
  should paint ten times the area at the same intensity, not ten times the
  pigment.

`getCoalescedEvents` is used where available, so a 120Hz touch reports every
sample rather than one per frame. Above two simultaneous fingers the MacCormack
correction is skipped -- see below.

## Why the page used to flash white

MacCormack advection was the cause, and it was not obvious. The correction
samples four neighbours and clamps the result to their range, and that clamp is
a strong mixing operator. Under heavy input it repeatedly averaged neighbouring
texels until R, G and B converged. The field then sat pinned at the dye ceiling
with no hue left -- measured at a white frame: peak 1.096, minimum saturation
0.000, 10,434 colourless texels -- and the tone-map rendered that as a white
page.

The correction is now blended in at 0.22 rather than applied at full strength,
and skipped entirely above two fingers. Measured on the deployed build with ten
fingers over 5.5 seconds: frames above white luma 27/46 before, 0/46 after.

Two related fixes came out of the same investigation. Ambient drift now deposits
22% of the dye a stroke does, because at full strength it repeatedly drove
texels to the ceiling and went colourless the same way. And the quality
governor samples the GPU once every 30 frames instead of every frame, because a
per-frame `readPixels` stalls the pipeline enough to show as a hitch.

## Performance

Four quality tiers scale dye, simulation, and bloom resolution together with
pressure-iteration count, curl strength, and bloom iterations. A governor
watches the real GPU cost of each frame and moves between tiers to hold a 60fps
budget, with hysteresis and asymmetric cooldowns so it settles instead of
oscillating.

Measuring GPU cost takes a deliberate `readPixels` of one pixel after
rendering. WebGL commands are queued asynchronously, so timing the JavaScript
around the draw calls measures almost nothing — in practice about 0.09ms. The
one-pixel read forces a pipeline flush, so the elapsed time includes the work
the GPU actually did.

Pressing the **Q** button takes manual control and disables the governor.

## Local development

```sh
npx wrangler pages dev . --profile personal
```

Or serve the directory with any static server — there is nothing to compile.

## Deploy

```sh
npm run deploy
```

## Controls

- **Drag** anywhere to push dye through the fluid
- **spectrum** — the active dye palette; tap to change (spectrum, aurora, ember, orchid)
- **Flow: gentle** — how fast and how hard colour arrives; cycles
  gentle → normal → wild. `gentle` is the default: a stroke eases in and
  lingers instead of snapping to full strength.
- **Glow** — the soft halo around the dye; on by default
- **Auto** — slow background motion; on by default
- **Detail: auto** — quality follows your device's speed; tap to set it by hand
- **Erase** — clear the canvas

Every button names its own state, and every one has a label for screen
readers, so nothing depends on hovering for an explanation.
