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
- **Palette** — cycle between spectrum, aurora, ember, and orchid
- **Glow** — toggle the bloom
- **Drift** — toggle ambient motion
- **Q** — cycle simulation quality manually (disables the auto-governor)
- **Clear** — wipe the canvas
