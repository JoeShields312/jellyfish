# Medusa — a real-time jellyfish simulation

An interactive jellyfish scene that runs entirely in the browser. No dependencies, no build step, no assets — each version is a single self-contained HTML file.

There are two renderers:

| File | Renderer | Notes |
|---|---|---|
| `index.html` | **WebGL2** | The main version. Raymarched volumetric bells, GPU-drawn tentacles, HDR post-processing. |
| `canvas.html` | Canvas 2D | The earlier version. Same behaviour model, flat-shaded 3D projection instead of volumes. |

## Run it

Open `index.html` in a browser. That's it — it is a static file.

If you'd rather serve it (recommended while editing), any static server works:

```bash
npx serve .
# or
python3 -m http.server 8000
```

**Browser requirements:** `index.html` needs WebGL2 (current Chrome, Edge, Firefox, and Safari 15+). If WebGL2 is unavailable it shows an error message instead of a blank screen. `canvas.html` uses `ctx.filter` for bloom and depth of field, which Safari supports poorly, so use a Chromium-based browser or Firefox for that one.

A discrete GPU is recommended for `index.html`. The **Quality** slider scales the internal render resolution if it runs slowly.

## Deploy to Vercel

1. Push this folder to a GitHub repository.
2. In Vercel: **Add New → Project**, and import the repository.
3. Framework Preset: **Other**. Leave the build command and output directory empty.
4. Deploy.

`index.html` is served at the root URL and `canvas.html` at `/canvas.html`. No configuration file is required.

## Controls

The panel in the lower right collapses from its header.

**Water** — Population, Current, Viscosity, Eddies
**Animal** — Pulse rate, Pulse depth, Wander, Tangle
**Medium** — Density, Scatter, Tissue, Emission
**Camera** — Shutter (motion blur), Bloom, Shafts, Grain, Quality

## How it works

### Bells: raymarched volumes (`index.html`)

Each animal is drawn as a screen-space billboard. For every pixel it covers, the fragment shader casts a camera ray through a bounding sphere and marches through an analytic density field, accumulating radiance front to back with Beer–Lambert extinction.

The density field is built from anatomy rather than a mesh: a hollow shell with mesogloea that thickens toward the apex, a scalloped margin, radial and ring canals, four gonads, oral arms, and eight rhopalia. Each sample does a short secondary march toward the key light for single scattering, which gives self-shadowing and a forward-scatter glow when an animal is backlit.

This is an approximation, not a path tracer: one scattering bounce, a three-tap shadow march, and procedural noise for tissue texture.

### Tentacles

The tentacle simulation runs on the **CPU in JavaScript**; only the rendering is on the GPU.

- Verlet chains in camera space, with a fixed world-unit rest length.
- An asymmetric distance constraint: stiff against stretching, nearly slack against compression, so tentacles can go limp and bunch.
- A bending-resistance term so slack chains form smooth curves instead of zigzagging.
- Per-node sampling of a fluid field that includes small-scale eddies. Without structure at tentacle scale the forces along a filament are effectively uniform, and a uniform force translates a chain rather than bending it.
- Inertial coupling to the body's actual acceleration, so each jet visibly drags the tentacles.
- Tentacle–tentacle contact via a 3D spatial hash (repulsion at short range, weak cohesion at mid range).

Tentacle geometry is rebuilt each frame as quads with a Gaussian cross-section in the fragment shader.

### Swimming

Bells pulse with a fast contraction and slow relaxation. Thrust scales with the square of the contraction rate, and inter-pulse timing is resampled every cycle with occasional coasting beats so animals drift out of phase. The visible bell deformation is deliberately small and smooth; the sharp contraction curve drives the physics only.

The surrounding water is an analytic field (summed sinusoids plus eddies), not a fluid solver.

### Orientation

Each animal's local frame is composed analytically from yaw, pitch, and roll. An earlier version derived it by crossing the swim axis with a reference vector and switching references near a threshold, which flipped the whole animal in a single step; the analytic form has no such branch.

### Rendering pipeline

Scene → HDR float targets (when `EXT_color_buffer_float` is available, otherwise 8-bit) → temporal accumulation for motion blur → threshold and separable blur for bloom → composite with a filmic tonemap, vignette, and grain. Physics steps at a fixed 120 Hz, independent of display rate.

## Known limitations

- The WebGL version has not been profiled across a range of GPUs. Integrated graphics may need the Quality slider lowered.
- Biology and fluid dynamics are simplified for visual plausibility, not modelled accurately.
- `canvas.html` predates several changes made to `index.html` (volumetric anatomy, 3D tentacles, tentacle contact) and is kept as a lighter-weight alternative.

## Built with

Developed iteratively with assistance from Claude (Anthropic).
