# Ox Alpha — Meadow v4: Design Document

A single-file, dependency-free (Three.js CDN only) procedural meadow.
No external assets: every texture, mesh, and color is generated at
load time. Target: 60fps on discrete GPU, ~40fps integrated.

## 1. Core Architecture

Single HTML file, ES modules via importmap (three@0.160).
Pipeline: procedural textures (canvas) → terrain (displaced plane)
→ instanced grass blades (GPU wind) → alpha-card flowers →
sky dome + fog + sprite clouds → ACES tonemapping.

## 2. Terrain

- **Mesh**: `PlaneGeometry(140, 140, 240, 240)` rotated flat, Y-displaced by `groundH(x,z)`.
- **Height function** = 3 Gaussian hills (control placement & scale)
  + 5-octave value-noise fBm (large forms) + 3-octave fBm (detail)
  − a Gaussian-carved winding path (`sin`-based meander).
  Everything is analytic → grass/flowers/flattening can query it exactly.
- **Material**: `MeshStandardMaterial` with:
  - **Albedo**: 512² tileable canvas — base `#8aa25a`, 60 soft radial
    blobs (warm/cool alternating), 2600 wrap-drawn grass strokes
    (light + dark), 300 straw strokes.
  - **Bump map**: same stroke generator in grayscale, bumpScale 0.6.
  - **Vertex colors**: HSL hue ~0.24 with two low-frequency fBm layers
    modulating lightness → macro patchiness that breaks tiling.
  - **Detail overlay** (shader injection): second albedo texture sampled
    at *irrational* repeat scales (1/61.3, 1/47.7) and rotated 0.37 rad,
    multiplied at 30% strength. Irrational ratio + rotation = no visible
    tile rhythm at any distance.

### Tileability trick
All canvas noise uses `makePeriodicNoise` — value noise on a wrapping
lattice — so fBm textures tile perfectly. Strokes use a `wrapStroke`
helper that redraws each stroke at ±S offsets (9-cell check) so marks
crossing the border wrap seamlessly.

## 3. Grass (the star)

Two-tier system: **real geometry near, texture far.**

### Blade geometry
Custom BufferGeometry: 4-segment strip, tapering width
(`W=0.016`, quadratic taper), baked forward bend (`t²·0.18`).
≈ 9 tris/blade.

### Instancing
`InstancedMesh`, 220,000 blades, `frustumCulled=false`.
Placement: uniform disc r=40 around camera (`sqrt` for area-uniform),
rejection-sampled: skip if outside terrain or if a 3-octave fBm
patchiness mask < 0.28 (creates natural bare patches).
Per blade: random tilt ±14° on X/Z (breaks grid regularity, lets
blades shade each other), random Y-rotation, height 0.16–0.36,
width 0.8–1.3×. Root Y = groundH − 0.02 (sunk to hide gaps).

### Per-instance attributes
- `aPhase` (random 0–1): wind phase + per-blade color variation.
- `aOrigin` (vec3): world root position → distance fade + wind coherence.

### Wind (vertex shader, injected via onBeforeCompile)
sway = sin(t·1.7 + origin.x·0.35 + origin.z·0.5)·0.5
+ sin(t·2.3 + phase·13)·0.2 // secondary flutter
gust = 0.6 + 0.4·sin(t·0.5 + origin·low-freq) // slow gust front
bend = sway·gust·(uv.y)²·0.45 // quadratic: root fixed, tip moves
Two frequencies (1.7Hz sway + 2.3Hz flutter) × gust modulation ×
height² mask = waves rolling across the field instead of uniform
wobble.

### Distance fade
`fade = 1 − smoothstep(28, 46, dist(camera, aOrigin))` — blades
shrink into the ground beyond 28m, fully gone at 46m, where the
terrain texture takes over seamlessly.

### Color (fragment shader)
Vertical gradient `mix(base #2E4D1A, tip #9EC74D, vHeight^0.8)` —
dark saturated roots, warm yellow-green sunlit tips — plus per-blade
variation `0.85 + 0.3·fract(phase·7.13)`. Critical rule learned the
hard way: **blades must be darker/greener than the terrain base.**
Pale blades on green ground read as twigs/ants.

## 4. Flowers

18 clusters of 3–7, tight 1.4m scatter. Per flower:
- Stem: thin cylinder (4-seg), 0.18–0.40m, slight random lean.
- Head: two **crossed alpha cards** (0.34m plane, painterly canvas
  texture — 6–9 soft radial-gradient petals + heart glow) rotated 90°
  from each other → volume from any angle. `MeshBasicMaterial`,
  `depthWrite:false`, so alpha edges never z-fight the grass.
- Heads sit *inside* the grass height zone (stem ≤ blade height) —
  flowers poking out of turf, not floating on it.

### Interaction
Raycast pointer → plane. While mouse-down, nearby heads accumulate
`push`; each frame heads rotate by push, push decays ×0.92 (spring
back), plus idle bob `sin(t·1.8 + x·3)·0.06`.

## 4.5. Butterflies

16 animated butterflies fluttering near ground level (0.45–1.35m above `groundH`):
- **Textures**: Canvas-generated dual-wing patterns in 4 color schemes (Monarch Orange, Swallowtail Yellow, Sky Blue Morpho, Soft Pink).
- **Hinged Wing Mesh**: Two wing planes pivoted at $X=0$ for realistic 16–24 Hz sinusoidal flapping.
- **3D Flight Path**: Smooth Lissajous flutter trajectories near the ground, dynamically facing their direction of travel with ascent/descent pitch.

## 5. Atmosphere

- **Sky**: back-side sphere, 3-stop gradient shader (horizon #D8E8EA →
  mid #9FD0E8 → zenith #3A78C8), pow 0.65 curve for a soft horizon band.
- **Fog**: `FogExp2(0xd8e8ea, 0.011)` — tinted exactly to the sky
  horizon color so terrain melts into sky with no visible seam.
- **Clouds**: 4 groups of 6–10 sprites (radial-gradient puff texture),
  stretched 1.4:0.75, varied opacity, drifting +0.03 x/frame, wrap at ±150.
- **Lighting**: hemisphere (sky #BCD8F0 / ground #7A8A50, 0.58) +
  warm sun `#FFDDB0` intensity 2.0 at (24,18,10), 2048 PCF-soft shadows
  (50m ortho box, bias −0.0004, radius 3) + faint cool fill from opposite.
- **Post**: ACES Filmic tonemapping, exposure 1.25, plus a CSS inset
  vignette (`box-shadow: inset 0 0 120px rgba(19,30,34,0.28)`).

## 6. Performance Notes

- ~2M grass tris is the main cost; scale `BLADES` (220k → 150k) for
  integrated GPUs.
- All textures generated once at startup (~50ms).
- Wind is pure vertex-shader math — zero per-frame CPU cost.
- Flower interaction is the only CPU-side per-frame loop (~80 heads).

## 7. Tuning Cheat-Sheet

| | Change |
|---|---|
| Denser/thinner grass | `BLADES`, field radius `r * 40` |
| Taller/shorter turf | blade `h = 0.16 + rand*0.20` |
| Stronger wind | `bend ... * 0.45` in grass vertex shader |
| Grass extends farther | smoothstep(28, 46, d) + blade field radius |
| Patchier meadow | fbm threshold `0.28` in placement loop |
| Time of day | sun color/position + hemisphere colors + fog color |
| More flowers | cluster count `cl < 18` |

