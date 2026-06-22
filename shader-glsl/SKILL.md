---
name: shader-glsl
description: This skill should be used when the user asks to "write a fragment shader", "make a GLSL gradient/noise/plasma background", "create an image transition (dissolve, displacement, glitch)", "add distortion or chromatic aberration", "build an SDF shape shader", "wire up a Three.js ShaderMaterial with uniforms", or "do GPU post-processing". Covers GLSL fragment shaders, noise/fbm, SDFs, domain warping, transitions, and Three.js integration.
version: 0.1.0
---

# Shader / GLSL

Write GPU fragment shaders for generative motion, gradients, transitions, and post-processing. Fragment shaders run once per pixel in parallel — the most performant way to do full-screen generative motion.

## When to use

- Animated gradient, noise, plasma, or aurora backgrounds.
- Image transitions: dissolve, displacement, glitch, ripple, wipe.
- Distortion, chromatic aberration, generative patterns, SDF shapes.
- Post-processing passes over a rendered scene.

## Fragment shader skeleton

Every fragment shader computes a color for one pixel. Normalize coordinates, aspect-correct, then build color.

```glsl
precision highp float;
uniform float u_time;
uniform vec2  u_resolution;
uniform vec2  u_mouse;

void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution.xy;     // 0..1
  vec2 p  = uv * 2.0 - 1.0;                         // -1..1, centered
  p.x *= u_resolution.x / u_resolution.y;           // aspect-correct
  vec3 col = 0.5 + 0.5 * cos(u_time + p.xyx + vec3(0.0, 2.0, 4.0));
  gl_FragColor = vec4(col, 1.0);
}
```

The cosine-palette line above (Inigo Quilez palettes) is the fastest route to a good-looking animated gradient: `a + b*cos(2π*(c*t + d))` with tunable `a,b,c,d` vec3s.

## Core building blocks

**smoothstep + mix** are the workhorses. `smoothstep(e0, e1, x)` gives a smooth 0→1 ramp; `mix(a, b, t)` linearly blends. Antialias an edge by the width of one pixel:
```glsl
float px = fwidth(d);                    // screen-space derivative
float mask = smoothstep(px, -px, d);     // crisp AA edge from SDF distance d
```

**Hash + value noise** (no textures needed):
```glsl
float hash(vec2 p){ return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }
float noise(vec2 p){
  vec2 i = floor(p), f = fract(p);
  vec2 u = f * f * (3.0 - 2.0 * f);                // smooth interpolation
  return mix(mix(hash(i), hash(i + vec2(1,0)), u.x),
             mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), u.x), u.y);
}
float fbm(vec2 p){                                  // fractal noise, organic
  float v = 0.0, a = 0.5;
  for (int i = 0; i < 5; i++){ v += a * noise(p); p *= 2.0; a *= 0.5; }
  return v;
}
```
Full value/simplex noise and fbm variants are in `references/glsl-cookbook.md`.

**SDF shapes** give resolution-independent crisp geometry. Distance is negative inside, positive outside:
```glsl
float sdCircle(vec2 p, float r){ return length(p) - r; }
float sdBox(vec2 p, vec2 b){ vec2 d = abs(p) - b; return length(max(d,0.0)) + min(max(d.x,d.y),0.0); }
// render: float m = smoothstep(fwidth(d), -fwidth(d), d);
```

**Domain warping** for fluid, marbled looks — feed noise into noise:
```glsl
vec2 q = vec2(fbm(p), fbm(p + vec2(5.2, 1.3)));
float n = fbm(p + 4.0 * q + u_time * 0.1);
```

## Image transitions

Sample two textures and blend per pixel by a progress uniform `u_progress` (0→1).

**Dissolve / noise wipe** — reveal by thresholding noise:
```glsl
float n = noise(uv * 20.0);
float edge = smoothstep(u_progress - 0.05, u_progress, n);
gl_FragColor = mix(texture2D(tex0, uv), texture2D(tex1, uv), 1.0 - edge);
```

**Displacement** — push UVs using a displacement map before sampling:
```glsl
float disp = texture2D(dispTex, uv).r;
vec2 d0 = uv + vec2(disp * u_progress * 0.3, 0.0);
vec2 d1 = uv - vec2(disp * (1.0 - u_progress) * 0.3, 0.0);
gl_FragColor = mix(texture2D(tex0, d0), texture2D(tex1, d1), u_progress);
```

**Glitch** — block-shift rows by time, split RGB channels (chromatic aberration):
```glsl
float row = floor(uv.y * 20.0);
float shift = (hash(vec2(row, floor(u_time * 12.0))) - 0.5) * 0.1 * u_glitch;
vec2 g = uv + vec2(shift, 0.0);
vec3 c;
c.r = texture2D(tex, g + vec2(0.005, 0.0)).r;       // channel offset
c.g = texture2D(tex, g).g;
c.b = texture2D(tex, g - vec2(0.005, 0.0)).b;
gl_FragColor = vec4(c, 1.0);
```

## Three.js ShaderMaterial wiring

```js
import * as THREE from 'three';
const uniforms = {
  u_time:       { value: 0 },
  u_resolution: { value: new THREE.Vector2(innerWidth, innerHeight) },
  u_mouse:      { value: new THREE.Vector2(0, 0) },
};
const material = new THREE.ShaderMaterial({
  uniforms,
  vertexShader: `void main(){ gl_Position = vec4(position, 1.0); }`,
  fragmentShader: FRAG_SRC,            // your GLSL string
});
// Full-screen triangle/quad: a plane that covers clip space.
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material);
const scene = new THREE.Scene(); scene.add(mesh);
const camera = new THREE.Camera();   // no projection needed for clip-space quad
const renderer = new THREE.WebGLRenderer();
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  uniforms.u_time.value = clock.getElapsedTime();
  renderer.render(scene, camera);
});
addEventListener('resize', () => {
  renderer.setSize(innerWidth, innerHeight);
  uniforms.u_resolution.value.set(innerWidth, innerHeight);
});
```
For a full-screen pass with a plain camera, write the vertex shader to output `position` directly and skip projection (as above). For shaders applied to real geometry, pass `vUv` from the vertex shader via `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.0); }`.

## Mobile / performance

- Declare `precision mediump float;` on mobile when highp is not needed; some effects (large coordinates, deep fbm) require `highp`.
- Cap pixel ratio: `renderer.setPixelRatio(Math.min(devicePixelRatio, 2))`. Render to a lower-res target and upscale for heavy shaders.
- Loops must have **constant bounds** in GLSL ES — no dynamic loop counts. Keep fbm octaves ≤ 5–6.
- Avoid `if`/branches in hot paths; prefer `mix`/`step`/`smoothstep`. Minimize `texture2D` calls; avoid dependent texture reads where possible.
- WebGL2/GLSL ES 3.00 enables `texelFetch`, integer ops, and `textureLod`; declare `#version 300 es` and use `in`/`out`/`fragColor`.

## Quick reference

| Goal | Primitive |
|------|-----------|
| Animated gradient | cosine palette `a + b*cos(...)` |
| Organic texture | `fbm(uv * scale + time)` |
| Crisp shape | SDF + `smoothstep(fwidth(d), -fwidth(d), d)` |
| Fluid / marble | domain warp: noise into noise |
| Reveal transition | threshold noise vs `u_progress` |
| Glitch | row hash shift + RGB channel offset |
| AA edge | `fwidth(d)` for screen-space width |

## Reference files

- `references/glsl-cookbook.md` — Full value and simplex noise + fbm implementations, IQ cosine-palette recipes, the complete SDF shape library with boolean ops and rounding, domain warping, all three image transitions (dissolve/displacement/glitch) as complete shaders, Three.js uniform/texture wiring, GLSL ES 3.00 migration, and mobile precision gotchas.
