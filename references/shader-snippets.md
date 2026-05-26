# Shader snippets

Ready-to-paste GLSL for the holographic look. All snippets assume `uniform float uTime;` and `uniform vec3 uColor;` are available.

## Fresnel emissive (volumetric fill)

```glsl
// vertex
varying vec3 vNormal;
varying vec3 vViewDir;
varying vec2 vUv;
varying vec3 vWorldPos;

void main() {
  vec4 mv = modelViewMatrix * vec4(position, 1.0);
  vNormal = normalize(normalMatrix * normal);
  vViewDir = normalize(-mv.xyz);
  vUv = uv;
  vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
  gl_Position = projectionMatrix * mv;
}
```

```glsl
// fragment
uniform vec3 uColor;
uniform float uTime;
uniform float uIntensity;
varying vec3 vNormal;
varying vec3 vViewDir;
varying vec2 vUv;
varying vec3 vWorldPos;

void main() {
  float fresnel = pow(1.0 - max(dot(vNormal, vViewDir), 0.0), 2.5);

  // horizontal scanlines
  float scan = 0.9 + 0.1 * sin(vWorldPos.y * 200.0 + uTime * 1.5);

  // slow scan band sweeping vertically
  float band = smoothstep(0.0, 0.04, fract(uTime * 0.12 - vWorldPos.y * 0.1)) * 0.4;

  // flicker (very subtle)
  float flicker = 0.97 + 0.03 * sin(uTime * 30.0 + vWorldPos.x * 10.0);

  vec3 col = uColor * fresnel * uIntensity * scan * flicker;
  col += uColor * band * fresnel;

  gl_FragColor = vec4(col, fresnel * 0.6);
}
```

Use with: `transparent: true`, `blending: THREE.AdditiveBlending`, `depthWrite: false`, `side: THREE.DoubleSide`.

## Glowing wireframe

```glsl
// fragment for LineBasicMaterial replacement
uniform vec3 uColor;
uniform float uTime;
varying vec3 vWorldPos;

void main() {
  float pulse = 0.85 + 0.15 * sin(uTime * 2.0 + vWorldPos.y * 3.0);
  gl_FragColor = vec4(uColor * pulse, 1.0);
}
```

Pair with `LineMaterial` from `three/addons/lines/LineMaterial.js` for variable line width.

## Grid floor

```glsl
// fragment for a large PlaneGeometry under the subject
uniform vec3 uColor;
uniform float uTime;
varying vec3 vWorldPos;

float grid(vec2 p, float scale) {
  vec2 g = abs(fract(p * scale - 0.5) - 0.5) / fwidth(p * scale);
  float line = min(g.x, g.y);
  return 1.0 - min(line, 1.0);
}

void main() {
  float major = grid(vWorldPos.xz, 1.0);
  float minor = grid(vWorldPos.xz, 10.0) * 0.3;

  float dist = length(vWorldPos.xz);
  float falloff = 1.0 - smoothstep(0.0, 30.0, dist);

  float g = max(major, minor) * falloff;
  gl_FragColor = vec4(uColor * g, g * 0.5);
}
```

## Edge-only fresnel (when EdgesGeometry is too noisy)

For dense organic meshes where extracted edges look messy, use this instead — pure fresnel with no fill:

```glsl
// fragment
uniform vec3 uColor;
varying vec3 vNormal;
varying vec3 vViewDir;

void main() {
  float rim = pow(1.0 - max(dot(vNormal, vViewDir), 0.0), 4.0);
  if (rim < 0.15) discard;
  gl_FragColor = vec4(uColor * rim * 2.0, rim);
}
```

## Vertex displacement for "data corruption" pulse

Optional. Apply briefly when the user triggers an event (e.g. clicking a component):

```glsl
// vertex addition
uniform float uGlitch; // 0..1, animate from 0 → 1 → 0 over ~200ms

vec3 p = position;
float noise = sin(p.y * 50.0 + uTime * 100.0) * cos(p.x * 30.0);
p += normal * noise * uGlitch * 0.02;
gl_Position = projectionMatrix * modelViewMatrix * vec4(p, 1.0);
```
