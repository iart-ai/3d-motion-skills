# Shader / GLSL

Write GPU fragment shaders for generative motion, gradients, transitions, and post-processing — the most performant way to do full-screen generative motion, since fragment shaders run once per pixel in parallel. For web and creative developers who want noise, SDFs, domain warping, and clean Three.js integration without trial-and-error.

## When it activates

- "Write a fragment shader" or "make a GLSL gradient / noise / plasma / aurora background."
- "Create an image transition" — dissolve, displacement, glitch, ripple, wipe.
- "Add distortion or chromatic aberration" or "build an SDF shape shader."
- "Wire up a Three.js ShaderMaterial with uniforms" or "do GPU post-processing."

## Example prompts

- "Write a GLSL fragment shader for a domain-warped marble background and hook it into a full-screen Three.js quad."
- "Make a dissolve transition between two textures driven by a progress uniform."
- "Build an SDF shader with crisp antialiased edges using fwidth."

## What's inside

- `SKILL.md` — fragment-shader skeleton, cosine palettes, hash/value noise + fbm, SDF shapes, domain warping, the three image transitions, ShaderMaterial wiring, and mobile/performance gotchas.
- `references/glsl-cookbook.md` — full value and simplex noise + fbm, IQ palette recipes, the complete SDF shape library with boolean ops, all three transitions as complete shaders, Three.js uniform/texture wiring, and GLSL ES 3.00 migration.

---
Part of **[3D Motion Skills](../)** · Built by **[iart.ai](https://iart.ai)** — the AI motion agent for editable, on-brand motion graphics.
