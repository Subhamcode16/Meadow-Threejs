# MASTER PROMPT — "Procedural Meadow" (single-file Three.js)

Build a complete, runnable, single HTML file (Three.js 0.160 via
importmap CDN, OrbitControls from examples/jsm). ZERO external assets:
every texture generated on a canvas at load, every mesh procedural.
The result must look like a painterly, wind-swept summer meadow with
rolling hills, dense real grass, wildflowers, drifting clouds, and
atmospheric depth.

## TERRAIN
1. PlaneGeometry 140×140, 240×240 segments, rotated flat.
2. Height = analytic function `groundH(x,z)`: three Gaussian hills
   (heights ~5.5/4/3, spreads 45–70) + 5-octave value-noise fBm·3.2
   + 3-octave fBm·0.5 detail, minus a winding path carved as
   0.7·exp(−(z − sin-meander(x))²/8). All queries exact — reuse for
   grass/flower placement.
3. Material: MeshStandardMaterial, roughness 1.
   - Albedo 512² TILEABLE canvas: base #8aa25a; 60 soft radial
     gradients alternating warm rgba(130,140,60) / cool rgba(60,95,45);
     2600 short grass strokes (light rgba(150,170,80) and dark
     rgba(35,60,25)) drawn with a wrap helper that redraws strokes at
     ±512 offsets so borders tile. Repeat 24×24, max anisotropy.
   - Bump map: same stroke technique in grayscale, bumpScale 0.6.
   - Vertex colors: per-vertex HSL(0.24 ± low-freq fbm·0.05, 0.55,
     lightness·(0.95 + macro fbm·0.25)) → macro patchiness.
   - ANTI-TILING: inject a second copy of the albedo texture via
     onBeforeCompile, sampled at irrational scales (1/61.3, 1/47.7)
     and rotated 0.37 rad in the vertex shader, multiplied into
     diffuse at 30% strength in map_fragment.

## GRASS — two-tier: real blades near, texture far
1. Blade geometry: custom BufferGeometry strip, 4 segments, width
   0.016 with quadratic taper, baked tip bend t²·0.18, ~9 tris.
2. InstancedMesh, 220,000 instances, frustumCulled false.
   Placement: uniform disc radius 40 around camera (sqrt-random),
   reject samples outside terrain or where fbm(x·0.35) < 0.28
   (bare patches Random tilt ±0.25 rad X/Z, random Y rotation,
   height 0.16–0.36, sink root 0.02 below groundH.
3. Instanced attributes: aPhase (float 0–1), aOrigin (vec3 root).
4. Wind via onBeforeCompile vertex injection, uniform uTime:
   bend = [sin(t·1.7 + ox·0.35 + oz·0.5)·0.5 + sin(t·2.3+phase·13)·0.2]
        · [0.6 + 0.4·sin(t·0.5 + ox·0.08 + oz·0.11)]   // gust front
        · uv.y² · 0.45                                  // tip-only
   applied to x (and z at 35%). CRITICAL: distance fade
   transformed *= 1 − smoothstep(28, 46, distance(cameraPosition, aOrigin))
   so blades shrink into the ground where the terrain texture takes over.
5. Fragment injection: gradient mix(vec3(0.18,0.30,0.10),         // dark roots
   vec3(0.62,0.78,0.30), pow(vHeight, 0.8)) with per-blade variation
   ·(0.85 + 0.3·fract(phase·7.13)). RULE: blades must be darker and
   greener than the ground — pale blades on green terrain read as twigs.
6. DoubleSide, roughness 1.

## FLOWERS
18 clusters, each 3–7 flowers scattered in 1.4m. Stem: 4-seg cylinder
0.008–0.012 radius, 0.18–0.40 tall, slight lean. Head: TWO crossed
planes (0.34m, 90° apart) with 128² canvas textures — 6–9 petals as
soft radial gradients around a glowing heart, 4 color variants
(white daisy / red poppy / yellow buttercup / violet).
MeshBasicMaterial, transparent, depthWrite:false, DoubleSide.
Heads must sit INSIDE grass height (not above it).

## INTERACTION
Pointer raycast to ground plane: while mouse is down, nearby flower
heads accumulate push; heads rotate by push each frame, push decays
×0.92 (spring back), plus idle bob sin(t·1.8 + x·3)·0.06.

## SKY & ATMOSPHERE
1. Sky: 400-radius back-side sphere with a 3-stop gradient shader:
   horizon #D8E8EA → mid #9FD0E8 → zenith #3A78C8, with a pow(0.65)
   curve on the mix factor for a soft wide horizon band.
2. Fog: FogExp2 with color EXACTLY matching the sky horizon color
   (#D8E8EA), density 0.011 — terrain melts into sky with no seam.
3. Clouds: 4 groups of 6–10 sprites using a radial-gradient puff
   texture (white core → transparent), stretched 1.4:0.75, random
   opacity 0.5–0.85, drifting slowly +x, wrapping at ±150.
4. Lighting: HemisphereLight(sky #BCD8F0, ground #7A8A50, 0.58) +
   warm sun DirectionalLight #FFDDB0 intensity 2.0 at (24,18,10),
   castShadow, 2048 map, PCFSoftShadowMap, ortho box 50, bias −0.0004,
   radius 3 + a faint cool fill directional from the opposite side.
5. Renderer: ACESFilmicToneMapping, exposure 1.25, outputColorSpace
   sRGB. Add a CSS inset vignette on the body:
   box-shadow: inset 0 0 120px rgba(19,30,34,0.28).
6. Camera: elevated position (~18–22 units up, 25–35 back from
   center), OrbitControls with damping, target near meadow center.

## VALIDATION CHECKLIST (verify before finishing)
- Grass reads as a continuous turf surface, NOT individual pale
  blades — if blades look like twigs/ants, they are too pale, too
  big, or too sparse: darken them, shrink W, raise count.
- No visible texture tiling rhythm anywhere on terrain.
- Grass→texture handoff at 28–46m is invisible (blades shrink, pop).
- Gust waves visibly roll across the field.
- Flowers sit inside the grass, stems hidden by blades.
- Sky→terrain transition has zero hard edge (fog color == horizon).
- Single file, opens offline except the Three.js CDN import.

## TUNING KNOBS (document in code comments)
BLADES count · blade field radius · blade height range · wind bend
strength (0.45) · fade smoothstep(28,46) · bare-patch fbm threshold
(0.28) · flower cluster count · sun position/color (time of day).

