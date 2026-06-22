# Three.js Motion (3D / WebGL)

Build real-time 3D motion for the web — animated scenes, camera moves, GLTF playback, instancing, scroll-linked 3D, and React Three Fiber — optimized for smooth 60fps and leak-free GPU disposal. For web developers building 3D hero sections, product viewers, and interactive backgrounds who need them to stay fast and not leak VRAM.

## When it activates

- "Build a 3D hero section / WebGL background" or "animate a Three.js scene."
- "Play or crossfade GLTF animation clips" or "render thousands of objects with InstancedMesh."
- "Move/lerp the camera or link it to scroll" or "set up React Three Fiber motion."
- "Fix a Three.js memory leak" / "dispose geometries, materials, textures" / "stop VRAM growing in R3F."

## Example prompts

- "Build a scroll-linked Three.js scene where the camera flies along a curve as the user scrolls."
- "Render 5000 instanced cubes and animate each one's matrix per frame."
- "My R3F app's VRAM keeps growing on every route change — find and fix the leak."

## What's inside

- `SKILL.md` — minimal scene + render loop, GLTF AnimationMixer crossfade, camera lerp/scroll motion, InstancedMesh, R3F with `useFrame`, performance tips, and the critical GPU-disposal section.
- `references/r3f-and-perf.md` — full crossfade/one-shot handling, useFrame patterns, drei ScrollControls, per-frame instancing, adaptive resolution, and postprocessing (bloom, DOF).
- `references/resource-disposal.md` — complete GPU disposal utility, EffectComposer teardown, R3F leak cases with the useMemo+cleanup pattern, a `renderer.info` leak-test harness, and HMR safety.

---
Part of **[3D Motion Skills](../)** · Built by **[iart.ai](https://iart.ai)** — controllable Motion Graphics MP4 from a prompt or data.
