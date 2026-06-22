# 3D Render & Animation Troubleshooting

A routing hub for the three recurring failure classes in 3D and motion-graphics production: GPU memory running out, renders that come out noisy or smeared by the denoiser, and simulations that explode, jitter, tunnel, or refuse to reproduce. For 3D artists and TDs working in Blender, Cinema 4D, Houdini, and ComfyUI who want the exact settings and decision trees, not generic advice.

## When it activates

- "Fix CUDA / GPU out of memory" — Cycles/Redshift/Octane out of VRAM, or ComfyUI WAN/LTX video OOM.
- "Why does my animation OOM but single frames render fine?"
- "Fix grainy/noisy renders" or "stop the denoiser making things look plastic/painterly."
- "My simulation is exploding" — rigid bodies fly off the floor, cloth/Vellum jitters, pyro tunnels through collision, or bakes are non-deterministic.

## Example prompts

- "My ComfyUI WAN video OOMs as frame count grows but works on a single frame — how do I fix it?"
- "Set up adaptive sampling in Cycles so my animation stops boiling without doubling render time."
- "My Vellum cloth explodes when I raise substeps — what order do I fix things in?"

## What's inside

- `SKILL.md` — domain router plus summaries of all three playbooks and cross-domain notes.
- `references/vram-oom.md` — per-tool flags, a VRAM-budget worksheet, and a frame-count-vs-VRAM table.
- `references/sampling-denoise.md` — per-engine sampling tables, a noise-diagnosis flowchart, and the temporal-denoise pipeline.
- `references/sim-stability.md` — CFL math, starting values, a diagnostic decision tree, and exact setting names for Blender, C4D, and Houdini.

---
Part of **[3D Motion Skills](../)** · Built by **[iart.ai](https://iart.ai)** — controllable Motion Graphics MP4 from a prompt or data.
