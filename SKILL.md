---
name: holographic-3d-viz
description: Build interactive 3D web visualizations with a scientific holographic aesthetic — glowing wireframes, technical annotations, HUD overlays, scan effects — rendered in Three.js. Use this whenever the user asks for a "hologram," "holographic," "scientific visualization," "technical render," "blueprint-style 3D," "HUD-style 3D," or describes a subject they want shown as a glowing wireframe/x-ray/diagnostic projection on a dark background. Trigger on requests for engineering schematics, anatomical, molecular, mechanical, aerospace, or architectural subjects rendered in this style, even if the word "hologram" isn't used explicitly.
---

# Holographic 3D Visualization

Build interactive Three.js scenes that look like scientific holograms: precise, technical, hyperrealistic — not toy animations. The goal is the visual language of a Hollywood mission-control display or an engineering diagnostic console, not a neon arcade game.

## Aesthetic targets

The look this skill produces:

- Single dominant emissive color (user-specified; default amber/orange `#ffb547` with secondary `#ff6a00`) against true black
- Glowing wireframe geometry with subtle internal volumetric fill, not flat shaded surfaces
- Scanline overlay, faint grid floor, occasional horizontal "data band" sweeps
- Technical annotations: callout lines, measurement ticks, coordinate readouts, telemetry text in a monospace font
- Subtle chromatic fringing, light grain, gentle bloom — restrained, not Instagram-glow
- Slow, deliberate motion. Holograms drift and rotate; they don't bounce or wobble

What this skill explicitly avoids:

- Saturated rainbow colors, glossy plastic PBR materials, cartoon outlines
- Heavy bloom that blows out detail
- Fast cuts, easing-bounce, anything that reads as "motion graphics demo"
- Solid opaque meshes — holograms should feel semi-transparent and layered

## Tech stack

Use this stack unless the user requests otherwise:

- **Three.js** (latest, via ES modules / importmap from `unpkg` or `jsdelivr`)
- **postprocessing** library (pmndrs) for `EffectComposer`, `UnrealBloomPass` or `SelectiveBloomEffect`, `ChromaticAberrationEffect`, `NoiseEffect`, `VignetteEffect`
- **GLTFLoader** + DRACO/Meshopt for any imported geometry
- **OrbitControls** with damping for camera
- Optional: `lil-gui` for live tuning during development (strip before final delivery unless asked)
- No build step required — deliver a single HTML file with importmap when possible

## Implementation recipe

Follow this pipeline. Each step matters; skipping any of them produces the "clunky" look the user wants to avoid.

### 1. Renderer & color pipeline

```js
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.1;
renderer.setClearColor(0x000000, 1);
```

Black is true `#000000`. Do not use dark blue or charcoal — it kills the contrast that sells the hologram.

### 2. Geometry strategy

Holograms read as holograms because you can see through them. For every solid mesh, render two passes:

- **Wireframe layer**: `LineSegments` built from `WireframeGeometry(geo)` or, better, `EdgesGeometry(geo, thresholdAngle)` for clean topology. Use a custom `ShaderMaterial` or `LineBasicMaterial` with the hologram color, additive blending.
- **Volumetric fill layer**: the same geometry as a `MeshBasicMaterial` (no lighting needed — holograms are emissive) with `transparent: true`, `opacity: 0.05–0.12`, `blending: THREE.AdditiveBlending`, `depthWrite: false`, and `side: THREE.DoubleSide`. This gives the glassy interior glow.

For imported GLB models: traverse the scene, replace materials, build a parallel `LineSegments` mesh from `EdgesGeometry` for each.

If geometry is too dense for `EdgesGeometry` to look clean (e.g. high-poly organic models), fall back to a fresnel-edge shader instead — `pow(1.0 - dot(normal, viewDir), 3.0)` driving emission.

### 3. Fresnel rim shader (the single biggest realism unlock)

This is what makes the surface read as a projected light field instead of flat geometry. Apply to the volumetric fill mesh:

```glsl
// vertex
varying vec3 vNormal;
varying vec3 vViewDir;
void main() {
  vec4 mv = modelViewMatrix * vec4(position, 1.0);
  vNormal = normalize(normalMatrix * normal);
  vViewDir = normalize(-mv.xyz);
  gl_Position = projectionMatrix * mv;
}

// fragment
uniform vec3 uColor;
uniform float uIntensity;
varying vec3 vNormal;
varying vec3 vViewDir;
void main() {
  float fresnel = pow(1.0 - max(dot(vNormal, vViewDir), 0.0), 2.5);
  gl_FragColor = vec4(uColor * fresnel * uIntensity, fresnel);
}
```

Use additive blending. Rim glow at silhouette edges does the heavy lifting.

### 4. Scanlines & scroll

Add a horizontal scanline effect via fragment shader on the fill material, or as a full-screen postprocessing pass. The scanlines should:

- Be subtle (5–10% modulation, not zebra stripes)
- Scroll slowly upward (a few pixels per second)
- Occasionally have a single brighter "scan band" that sweeps top-to-bottom every 4–8 seconds

```glsl
float scan = 0.92 + 0.08 * sin(vUv.y * 800.0 + uTime * 2.0);
float band = smoothstep(0.0, 0.05, fract(uTime * 0.15 - vUv.y)) * 0.3;
color *= scan;
color += uColor * band;
```

### 5. Grid floor & coordinate frame

A faint grid plane below the subject sells the "scientific instrument" frame. Use a shader-based grid (not a `GridHelper` — too pixel-crisp) with:

- Major lines every 1.0 unit, minor every 0.1
- Falloff to black with distance (`smoothstep(maxDist, 0.0, length(vWorldPos.xz))`)
- Same hologram color, ~30% opacity

Optionally add three thin RGB-coded axis lines at origin (but tint them all to your hologram color for monochrome consistency, unless the user wants the classic X/Y/Z RGB convention).

### 6. Annotations & HUD

This is what separates "wireframe model" from "scientific hologram." Include at least some of:

- Callout lines from key features to text labels (use `Line2`/`LineMaterial` for screen-space-thick lines)
- Monospace text labels (use `troika-three-text` or HTML overlays positioned via `Vector3.project`)
- Corner HUD elements: timestamp, coordinate readout, "SYSTEM: NOMINAL" style telemetry that updates
- Measurement ticks along key dimensions with values
- Subtle data streams: scrolling hex/binary text in margins

Recommended font: `JetBrains Mono`, `IBM Plex Mono`, or `Space Mono` loaded from Google Fonts.

### 7. Postprocessing chain

Order matters. Apply in this sequence:

1. `RenderPass` (main scene)
2. `SelectiveBloomEffect` — threshold ~0.3, intensity 0.8–1.2, radius 0.5. Selective so HUD text stays crisp.
3. `ChromaticAberrationEffect` — offset `(0.0008, 0.0008)`. Just enough to feel like a real lens.
4. `NoiseEffect` — `premultiply: true`, opacity 0.05–0.1. Film grain.
5. `VignetteEffect` — `darkness: 0.6`, `offset: 0.3`. Pulls eye to center.

Skip SSAO, DoF, and SSR — they fight the wireframe aesthetic.

### 8. Motion

- Subject rotates slowly on Y axis (default ~0.1 rad/sec), pausable on hover
- Camera orbits via `OrbitControls` with `enableDamping: true`, `dampingFactor: 0.05`
- Subtle camera "breathing" via `camera.position.y += sin(t * 0.3) * 0.02`
- Any state changes (parts highlighting, annotations appearing) ease over 600–1000ms with `easeInOutCubic`, never bounce

### 9. Interactivity defaults

Ship with at least:

- Orbit / zoom / pan (limit zoom range so the user can't fly inside)
- Hover on subject parts → highlight that part (bump fresnel intensity, show its label)
- Optional toggle controls for: wireframe-only, exploded view, annotations on/off, scanlines on/off

## Subject-specific guidance

The skill works for any subject. For each, the wireframe density and annotation style should match the subject's domain:

- **Aerospace/vehicles**: blueprint feel — straight callouts to numbered components, dimension lines, cross-sections
- **Anatomical/biological**: organic curves, fresnel-heavy, labeled with Latin/medical terms
- **Molecular/chemical**: ball-and-stick or surface-mesh, bond labels, energy readouts
- **Mechanical/engineering**: tight wireframe, tolerance annotations, exploded views
- **Architectural**: floor-plate isolation, level callouts, structural emphasis

If the user provides a GLB/GLTF model, use it directly. If not, build the geometry procedurally from primitives — for technical subjects this often looks better than a stock model anyway, because the topology is clean.

## Default color palette

Unless the user specifies otherwise, use this amber palette (matches mission-control / Iron Man HUD references):

```js
const PALETTE = {
  primary:    new THREE.Color(0xffb547),  // main hologram color
  secondary:  new THREE.Color(0xff6a00),  // accent / alerts
  highlight:  new THREE.Color(0xfff2c4),  // hover / focus
  dim:        new THREE.Color(0x4a2a00),  // background grid
};
```

For other palettes the user might request: cyan `#00d4ff` (Tony Stark), green `#39ff88` (Matrix/terminal), red `#ff2d4a` (alert/warning). Keep palette monochromatic within each scene.

## Deliverable format

Default to a single self-contained `index.html` file with:

- Importmap pointing to `three`, `three/addons/`, and `postprocessing` from a CDN
- All shaders inline as template strings
- All geometry either procedural or referenced from an external `.glb` URL
- Full-viewport canvas, no scrollbars, no margins
- A small instructions overlay (top-right corner, monospace, fades out after 3s)

If the user wants a React Three Fiber version instead, swap the recipe to use R3F + drei (`<Edges>`, `<Environment>`, `<Bloom>` from `@react-three/postprocessing`) but keep all the same aesthetic rules.

## Common failure modes to avoid

- Wireframe lines too thick → looks like a cartoon. Use thin lines, let bloom do the glow.
- Too many colors → kills the hologram illusion. Stay monochrome per scene.
- Surfaces fully opaque → looks like a painted model. Always keep fill opacity below ~0.15.
- Bloom maxed out → blows out detail. The viewer should still see the wireframe topology clearly.
- Spinning too fast → reads as a logo turntable. Default to slow rotation; let user-driven orbit do the work.
- Forgetting annotations → 90% of the "scientific" feel comes from labels, ticks, and HUD elements. Don't ship without them.

See `references/shader-snippets.md` for ready-to-paste shader code and `references/scene-skeleton.md` for a complete starter scene.
