# Scene skeleton

A complete, working starter scene. Drop in geometry and annotations as needed. This is the structure to follow — don't ship the literal example, build the user's specific subject inside it.

## Single-file HTML template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>Holographic Visualization</title>
<style>
  html, body { margin: 0; padding: 0; overflow: hidden; background: #000; }
  canvas { display: block; }
  @import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;600&display=swap');
  .hud {
    position: fixed; color: #ffb547; font-family: 'JetBrains Mono', monospace;
    font-size: 11px; letter-spacing: 0.1em; pointer-events: none;
    text-shadow: 0 0 8px rgba(255, 181, 71, 0.6);
  }
  .hud.tl { top: 16px; left: 16px; }
  .hud.tr { top: 16px; right: 16px; text-align: right; }
  .hud.bl { bottom: 16px; left: 16px; }
  .hud.br { bottom: 16px; right: 16px; text-align: right; }
  .hud .dim { opacity: 0.5; }
</style>
</head>
<body>
<div class="hud tl">
  <div class="dim">// SYSTEM</div>
  <div id="status">INITIALIZING...</div>
</div>
<div class="hud tr">
  <div class="dim">// TIMESTAMP</div>
  <div id="timestamp">--:--:--</div>
</div>
<div class="hud bl">
  <div class="dim">// SUBJECT</div>
  <div>SUBJECT_NAME_HERE</div>
  <div class="dim" id="coords">X +0.000 Y +0.000 Z +0.000</div>
</div>
<div class="hud br">
  <div class="dim">// CONTROLS</div>
  <div>DRAG • ORBIT</div>
  <div>SCROLL • ZOOM</div>
</div>

<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/",
    "postprocessing": "https://unpkg.com/postprocessing@6.34.2/build/postprocessing.esm.js"
  }
}
</script>

<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import {
  EffectComposer, RenderPass, EffectPass,
  BloomEffect, ChromaticAberrationEffect, NoiseEffect, VignetteEffect
} from 'postprocessing';

const PALETTE = {
  primary: new THREE.Color(0xffb547),
  secondary: new THREE.Color(0xff6a00),
  highlight: new THREE.Color(0xfff2c4),
};

// --- Renderer ---
const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.1;
renderer.setClearColor(0x000000, 1);
document.body.appendChild(renderer.domElement);

// --- Scene & camera ---
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(6, 4, 8);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.minDistance = 3;
controls.maxDistance = 30;

// --- Hologram material factory ---
function makeHologramMaterial(color = PALETTE.primary, intensity = 1.0) {
  return new THREE.ShaderMaterial({
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    side: THREE.DoubleSide,
    uniforms: {
      uTime: { value: 0 },
      uColor: { value: color },
      uIntensity: { value: intensity },
    },
    vertexShader: /* glsl */`
      varying vec3 vNormal;
      varying vec3 vViewDir;
      varying vec3 vWorldPos;
      void main() {
        vec4 mv = modelViewMatrix * vec4(position, 1.0);
        vNormal = normalize(normalMatrix * normal);
        vViewDir = normalize(-mv.xyz);
        vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
        gl_Position = projectionMatrix * mv;
      }
    `,
    fragmentShader: /* glsl */`
      uniform vec3 uColor;
      uniform float uTime;
      uniform float uIntensity;
      varying vec3 vNormal;
      varying vec3 vViewDir;
      varying vec3 vWorldPos;
      void main() {
        float fresnel = pow(1.0 - max(dot(vNormal, vViewDir), 0.0), 2.5);
        float scan = 0.9 + 0.1 * sin(vWorldPos.y * 200.0 + uTime * 1.5);
        float band = smoothstep(0.0, 0.04, fract(uTime * 0.12 - vWorldPos.y * 0.08)) * 0.5;
        float flicker = 0.97 + 0.03 * sin(uTime * 30.0 + vWorldPos.x * 10.0);
        vec3 col = uColor * fresnel * uIntensity * scan * flicker;
        col += uColor * band * fresnel;
        gl_FragColor = vec4(col, fresnel * 0.6);
      }
    `,
  });
}

// --- Wireframe material ---
function makeWireMaterial(color = PALETTE.primary) {
  return new THREE.LineBasicMaterial({
    color, transparent: true, opacity: 0.85, blending: THREE.AdditiveBlending,
  });
}

// --- Build subject (REPLACE with actual subject geometry) ---
const subject = new THREE.Group();
scene.add(subject);

function addHoloMesh(geometry, color = PALETTE.primary, intensity = 1.0) {
  const fill = new THREE.Mesh(geometry, makeHologramMaterial(color, intensity));
  const edges = new THREE.LineSegments(new THREE.EdgesGeometry(geometry, 20), makeWireMaterial(color));
  const group = new THREE.Group();
  group.add(fill); group.add(edges);
  return group;
}

// Example subject — replace with the real one
const placeholder = addHoloMesh(new THREE.IcosahedronGeometry(2, 1));
subject.add(placeholder);

// --- Grid floor ---
const gridMat = new THREE.ShaderMaterial({
  transparent: true, depthWrite: false, blending: THREE.AdditiveBlending, side: THREE.DoubleSide,
  uniforms: { uColor: { value: PALETTE.primary }, uTime: { value: 0 } },
  vertexShader: `
    varying vec3 vWorldPos;
    void main() {
      vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 uColor;
    varying vec3 vWorldPos;
    float grid(vec2 p, float s) {
      vec2 g = abs(fract(p * s - 0.5) - 0.5) / fwidth(p * s);
      return 1.0 - min(min(g.x, g.y), 1.0);
    }
    void main() {
      float major = grid(vWorldPos.xz, 1.0);
      float minor = grid(vWorldPos.xz, 5.0) * 0.25;
      float falloff = 1.0 - smoothstep(0.0, 25.0, length(vWorldPos.xz));
      float g = max(major, minor) * falloff;
      gl_FragColor = vec4(uColor * g, g * 0.4);
    }
  `,
});
const floor = new THREE.Mesh(new THREE.PlaneGeometry(60, 60), gridMat);
floor.rotation.x = -Math.PI / 2;
floor.position.y = -2.5;
scene.add(floor);

// --- Postprocessing ---
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new EffectPass(camera,
  new BloomEffect({ intensity: 1.0, luminanceThreshold: 0.25, radius: 0.6 }),
  new ChromaticAberrationEffect({ offset: new THREE.Vector2(0.0008, 0.0008) }),
  new NoiseEffect({ premultiply: true }),
  new VignetteEffect({ darkness: 0.6, offset: 0.3 }),
));

// --- HUD updates ---
const statusEl = document.getElementById('status');
const timestampEl = document.getElementById('timestamp');
const coordsEl = document.getElementById('coords');
setTimeout(() => { statusEl.textContent = 'NOMINAL'; }, 1200);

// --- Animation loop ---
const clock = new THREE.Clock();
function animate() {
  const t = clock.getElapsedTime();
  subject.rotation.y = t * 0.1;
  camera.position.y += Math.sin(t * 0.3) * 0.001;

  // update uniforms
  scene.traverse(o => {
    if (o.material && o.material.uniforms && o.material.uniforms.uTime) {
      o.material.uniforms.uTime.value = t;
    }
  });

  // hud
  const now = new Date();
  timestampEl.textContent = now.toISOString().split('T')[1].split('.')[0];
  coordsEl.textContent = `X ${camera.position.x.toFixed(3)} Y ${camera.position.y.toFixed(3)} Z ${camera.position.z.toFixed(3)}`;

  controls.update();
  composer.render();
  requestAnimationFrame(animate);
}
animate();

// --- Resize ---
addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  composer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
```

## Adding annotations

For technical callouts pointing to features on the subject:

```js
function makeAnnotation(worldPos, labelText) {
  const div = document.createElement('div');
  div.className = 'annotation';
  div.style.cssText = `
    position: fixed; color: #ffb547; font-family: 'JetBrains Mono', monospace;
    font-size: 10px; pointer-events: none; white-space: nowrap;
    border-left: 1px solid #ffb547; padding-left: 8px;
    text-shadow: 0 0 6px rgba(255, 181, 71, 0.5);
  `;
  div.innerHTML = `<span style="opacity:0.5">[${labelText.code}]</span> ${labelText.label}`;
  document.body.appendChild(div);

  return { worldPos, div };
}

const annotations = [
  makeAnnotation(new THREE.Vector3(0, 2, 0), { code: 'A-01', label: 'PRIMARY NODE' }),
  // ...
];

// In animate loop, project each:
const v = new THREE.Vector3();
for (const a of annotations) {
  v.copy(a.worldPos).project(camera);
  const x = (v.x * 0.5 + 0.5) * window.innerWidth;
  const y = (-v.y * 0.5 + 0.5) * window.innerHeight;
  a.div.style.transform = `translate(${x + 40}px, ${y}px)`;
  a.div.style.opacity = v.z < 1 ? 1 : 0;
}
```

For thicker callout LINES from feature to label, draw an SVG overlay or use `Line2`/`LineMaterial` from `three/addons/lines/`.

## When using imported GLB models

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const draco = new DRACOLoader();
draco.setDecoderPath('https://unpkg.com/three@0.160.0/examples/jsm/libs/draco/');
const loader = new GLTFLoader();
loader.setDRACOLoader(draco);

loader.load('model.glb', (gltf) => {
  gltf.scene.traverse(child => {
    if (child.isMesh) {
      const holoFill = child.clone();
      holoFill.material = makeHologramMaterial(PALETTE.primary, 0.8);

      const edges = new THREE.LineSegments(
        new THREE.EdgesGeometry(child.geometry, 25),
        makeWireMaterial(PALETTE.primary)
      );
      edges.position.copy(child.position);
      edges.rotation.copy(child.rotation);
      edges.scale.copy(child.scale);

      child.parent.add(holoFill);
      child.parent.add(edges);
      child.parent.remove(child);
    }
  });
  subject.add(gltf.scene);
});
```
