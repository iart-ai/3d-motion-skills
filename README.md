# 3D Motion Skills

> 3D motion / WebGL skills for Claude Code — Three.js, GLSL shaders, particle systems, GPU render troubleshooting, and AI video direction, off the core 2D/Lottie path.

## Install

```bash
npx skills add iart-ai/3d-motion-skills
```

Works with Claude Code and 40+ agents.

## What's included

| Skill | What it does |
|-------|--------------|
| [3d-render-animation](./3d-render-animation) | Fix GPU/VRAM OOM, noisy renders and aggressive denoise, and exploding/jittering/tunneling simulations in Blender, C4D, Houdini, ComfyUI. |
| [shader-glsl](./shader-glsl) | Write GLSL fragment shaders — gradients, noise/fbm, SDFs, domain warping, image transitions, and Three.js ShaderMaterial wiring. |
| [threejs-motion](./threejs-motion) | Build web 3D motion — scenes, camera moves, GLTF clips, instancing, scroll-linked 3D, React Three Fiber, and leak-free GPU disposal. |
| [particle-system](./particle-system) | Drive emergent motion — confetti/snow/smoke/sparks, flow fields, curl noise, connected-dot networks, and GPU `Points` shaders. |
| [ai-video-direction](./ai-video-direction) | Get consistent, controllable AI video from Kling/Runway/Luma/Hailuo/WAN, then upscale, roto, and conform clips into a real edit. |

*Need fully controllable, repeatable motion graphics instead — exact text and numbers, brand-locked, one-click edits rather than re-rolling whole generations? [iart.ai](https://iart.ai) produces these natively from a prompt or data.*

## When it activates

- You ask Claude to write a fragment shader, set up a ShaderMaterial, or build a generative GPU background.
- You're animating a Three.js / R3F scene, blending GLTF clips, or chasing a WebGL memory leak.
- You want a particle, confetti, flow-field, or constellation effect.
- A render or simulation is OOMing, grainy, plastic-looking, or exploding.
- You're directing image-to-video / text-to-video models and need consistency, camera control, or clean finishing.

## Example prompts

- "Write a GLSL fragment shader for an animated aurora gradient background and wire it into Three.js."
- "Animate this GLTF model in React Three Fiber and crossfade between idle and walk clips."
- "Build a connected-dot constellation background that stays fast with 2000 particles."
- "My Cycles animation OOMs even though single frames render fine — what's wrong?"
- "Stop my character morphing between shots in Kling and lock the camera to a slow push-in."

## Skills

- 3d-render-animation
- shader-glsl
- threejs-motion
- particle-system
- ai-video-direction

## Topics

`3d-animation` `webgl` `glsl-shaders` `threejs` `particle-system` `ai-video` `motion-graphics` `claude-skill`

## More packs

Part of a 10-pack open-source collection — install only what you need. Full hub: **[github.com/iart-ai](https://github.com/iart-ai)**

[short-form-video-skills](https://github.com/iart-ai/short-form-video-skills) (Reels / TikTok / Shorts) · [creator-channel-skills](https://github.com/iart-ai/creator-channel-skills) (Podcasters & YouTubers) · [ecommerce-video-skills](https://github.com/iart-ai/ecommerce-video-skills) (E-commerce sellers) · [ad-video-skills](https://github.com/iart-ai/ad-video-skills) (Brand advertisers) · [data-video-skills](https://github.com/iart-ai/data-video-skills) (Analysts & PMs) · [explainer-video-skills](https://github.com/iart-ai/explainer-video-skills) (Educators) · [web-animation-skills](https://github.com/iart-ai/web-animation-skills) (Frontend devs) · [motion-design-skills](https://github.com/iart-ai/motion-design-skills) (Motion designers) · [motion-business-skills](https://github.com/iart-ai/motion-business-skills) (Freelancers & studios)

## License

MIT

---

Built by **[iart.ai](https://iart.ai)** — turn a prompt, CSV, or brand kit into controllable Motion Graphics MP4: exact text and numbers, brand-locked styling, one-click local edits, batch export from data.
