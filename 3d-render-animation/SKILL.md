---
name: 3d-render-animation
description: This skill should be used when the user asks to "fix GPU out of memory / CUDA out of memory", "Cycles / Redshift / Octane runs out of VRAM", "ComfyUI WAN / LTX video out of memory", "why does my animation OOM but single frames render", "fix grainy / noisy renders", "set up adaptive sampling", "stop the denoiser making things look plastic / painterly", "fix denoiser flicker or temporal boiling in animation", "tune Cycles / Redshift / Octane / Arnold samples", "fix my simulation exploding", "rigid bodies fly off the floor", "cloth jittering", "Vellum exploding when I raise substeps", "pyro tunneling through collision", or "why is my sim non-deterministic" — covering GPU/VRAM OOM, render sampling & denoise, and simulation stability in Blender, Cinema 4D, Houdini, and ComfyUI.
version: 0.1.0
---

# 3D Render & Animation Troubleshooting

A hub for three recurring failure classes in 3D and motion-graphics production: GPU memory running out, renders that are noisy or smeared by the denoiser, and simulations that explode, jitter, tunnel, or refuse to reproduce. Each domain has a deep playbook in `references/`. Read this overview to route to the right one, then open that file for exact settings, decision trees, and per-tool flags.

## Pick the domain

- **GPU / VRAM out of memory** — "CUDA out of memory", "out of VRAM", "failed to allocate", driver crash mid-render, ComfyUI `torch.cuda.OutOfMemoryError`, renders that succeed on single frames but die in animation, OOM only at the denoise/upscale step, or video models (WAN/LTX) OOMing as frame count grows. Read `references/vram-oom.md`.
- **Render sampling & denoise** — slow/grainy renders, fireflies, denoiser producing a plastic/waxy/painterly look or smearing fine detail, animation that boils/shimmers/flickers, or tuning AA/camera/light/GI samples in Cycles, Redshift, Octane, or Arnold. Read `references/sampling-denoise.md`.
- **Simulation stability** — sims that explode (bodies launch off the floor, violent vibration), jitter at rest, tunnel/clip through colliders, sink into the floor, or give different results each bake; cloth/Vellum exploding when substeps are raised; pyro leaking through collision; non-deterministic bakes in Blender, Cinema 4D, or Houdini. Read `references/sim-stability.md`.

## Domain summaries

### VRAM out of memory

VRAM must hold the whole working set at once — geometry + textures + acceleration structures + framebuffers (renderers), or model weights + latents + activations for every frame in the temporal window (video models). OOM means that set exceeds the card; every fix either shrinks the set or streams part of it off-card at a cost.

Read the symptom first, it names the cause: OOM at *load* means weights/scene are too big (FP8/GGUF, `--lowvram`/`--novram`, decimate). OOM during a *single frame* means resolution/poly is too high (lower res, tiling, lower precision). OOM at the *denoise/upscale* step is the classic peak buffer — denoise on CPU/tiled or free the model before upscaling. OOM only in *animation* (single frames are fine) is cross-frame accumulation — disable Cycles Persistent Data, render one frame per process; for video diffusion, cut frame count first. OOM on a *second* workflow is a stale cached model — `--disable-smart-memory` or restart the worker.

Costly misconceptions: out-of-core offloads textures, not the BVH/geometry core, so a high-poly OOM stays an OOM; multi-GPU does NOT pool VRAM (each card holds a full copy); system RAM never transparently extends VRAM. Levers in order of leverage: reduce resolution / frame count → lower precision → tiling → stream/offload → shrink the scene. Full per-tool flags, a VRAM-budget worksheet, and a frame-count-vs-VRAM table are in `references/vram-oom.md`.

### Render sampling & denoise

Cut render time without trading away noise or detail. AA / camera samples are the expensive multiplier — they scale cost roughly quadratically in squared-sampling renderers (Arnold-style), so clear noise at the *cheapest level that fixes it* first: light samples for shadow noise, GI/diffuse samples for indirect noise, indirect clamping for fireflies (never Clamp Direct), and AA only as a last resort. Adaptive sampling with a noise threshold (≈0.01 final, 0.05 preview) is the highest-leverage speed win; the threshold gates quality, the max only caps worst-case time.

Treat the denoiser as a finisher for the last few percent of noise, never a crutch — denoising a heavily noisy render is the #1 cause of the plastic/painterly look and lost detail. Always feed it albedo + normal feature passes. For animation, two distinct flickers: lock the sampling seed so residual noise sits still instead of boiling, and use a temporal denoiser (OIDN temporal, Noice, Redshift/Octane temporal modes) that needs motion-vector AOVs and runs as a post pass over the whole sequence. Render every frame to the same noise threshold so the noise level is uniform. Exact per-engine settings tables (Cycles, Redshift, Octane, Arnold), a noise-diagnosis flowchart, the temporal-denoise pipeline, and a pre-render checklist are in `references/sampling-denoise.md`.

### Simulation stability

The same root causes recur across rigid body, cloth/Vellum, and pyro in every solver: wrong scene scale, interpenetration at frame 1, too few substeps for the velocities present, thin colliders, and missing friction/damping. Fix in order and change one thing per bake. The ordered list: (1) scale the scene to real-world meters/kilograms — wrong scale masquerades as every other problem and no substep count fixes it; (2) start settled with no overlap at frame 1 — interpenetration is the #1 explosion cause; (3) raise substeps to satisfy CFL (an object must not move more than one collision-thickness per substep), but ramp gradually for cloth/Vellum because more substeps = stiffer effective constraints and can explode it; (4) fix collision thickness/margins and enable continuous collision; (5) add friction and damping to kill jitter and drift; (6) raise constraint/solver iterations (not substeps) for stretchy joints; (7) bake to disk and sim from frame 1 forward for determinism, disabling adaptive substeps.

CFL math, starting values per parameter, a full diagnostic decision tree, and exact setting names/locations for Blender, Cinema 4D, and Houdini (including Vellum and pyro) are in `references/sim-stability.md`.

## Cross-domain notes

- The denoise/upscale step appears in two domains: it is the usual VRAM *peak* and the usual cause of the *plastic look* — a render that "just fit" can die there, and a render denoised too aggressively looks waxy. When both memory and quality are in play, render to a clean noise floor at a manageable resolution, then denoise/upscale as a separate freed-model pass.
- "Works on a single frame, fails in animation" recurs in both VRAM (cross-frame accumulation) and sampling (per-frame seed/denoiser flicker) — identify whether the failure is a *crash* (VRAM) or a *visual flicker* (sampling) before routing.
- For deterministic, repeatable output across renders and sims, bake/cache to disk and always advance from frame 1 forward; never scrub backward and trust the result.
