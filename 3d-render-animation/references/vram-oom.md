# VRAM Out-of-Memory Triage

Diagnose and fix GPU out-of-memory across offline renderers (Cycles, Redshift, Octane) and ComfyUI video diffusion (WAN, LTX). The unifying truth: **VRAM holds the whole working set at once** — geometry + textures + acceleration structures + framebuffers (renderers), or model weights + latents + activations for *every frame in the temporal window* (video models). OOM means that set exceeds the card. Fixes either shrink the set or stream parts of it off-card, each with a specific cost.

Applies to "CUDA out of memory", "out of VRAM", "failed to allocate", driver crashes mid-render, ComfyUI `torch.cuda.OutOfMemoryError` / `Allocation on device`, renders that succeed on single frames but die in animation, or OOM that appears only at the denoise/upscale step.

## First: read the symptom, it names the cause

- **Single frame renders, animation sequence OOMs.** The renderer is *not* releasing the previous frame's data, or motion blur/persistent caches accumulate. Each frame in a sequence can hold deformation caches, sub-frame geometry for motion blur, and the prior frame's buffers. Fix: render frames as separate process invocations (one frame per process), disable persistent data between frames, or reduce motion-blur steps. In Blender, **disable "Persistent Data"** when it inflates memory across frames (it trades VRAM for speed); enable it only if VRAM allows.
- **OOM exactly at the denoise or upscale step, after the main render/sampling finished.** This is the classic peak: the denoiser (OptiX/OIDN) or the upscaler allocates a *new large buffer on top of* the already-resident image/latents. The render "fit" but render + denoiser does not. Fix: use a tiled denoiser, denoise on CPU, lower upscale factor, or free the model before upscaling.
- **OOM growing frame-by-frame in ComfyUI video (WAN/LTX).** Memory scales with the **number of frames (temporal length)**, not just resolution. A 81-frame clip holds ~81x the latent of one frame plus cross-frame attention. Fix: reduce frame count / segment length first — it is the highest-leverage lever for video models.
- **OOM at load before any sampling.** The model weights alone don't fit. Fix: use FP8/GGUF quantized weights, or `--lowvram`/`--novram` to stream weights from system RAM.

## Critical misconceptions that waste hours

- **Out-of-core / "GPU out-of-memory fallback" only offloads TEXTURES, not geometry.** Redshift and Octane out-of-core memory streams *textures* (and sometimes geometry on Redshift, with a heavy speed penalty) from host RAM, but the **BVH/acceleration structure and core scene must still fit in VRAM**. A scene that OOMs on geometry/polycount will still OOM with out-of-core on. Do not expect out-of-core to rescue a high-poly scene.
- **Multi-GPU does NOT pool VRAM.** Two 24 GB cards give 24 GB of usable scene budget, not 48 GB. Each GPU holds a **full copy** of the scene and renders different buckets/tiles/frames. If the scene doesn't fit on one card, adding cards does not help. (NVLink pooling is the rare exception on specific pro cards/workloads and is not available on consumer RTX.)
- **More system RAM is not VRAM.** System RAM only helps via explicit out-of-core/offload paths; it never transparently extends VRAM for the GPU's resident set.

## Universal levers (apply in order of leverage)

1. **Reduce resolution / segment length / batch.** For video models, frame count is usually the biggest win; for stills, resolution. Halving resolution roughly quarters framebuffer/latent memory.
2. **Lower precision.** FP16 over FP32; FP8 or GGUF quantization for diffusion models cuts weight memory ~2-4x with minor quality cost.
3. **Tiling.** VAE tiling (ComfyUI) and tiled rendering/denoising process the image in chunks so peak buffer is per-tile, not full-frame. Trades speed for VRAM.
4. **Stream/offload.** `--lowvram`/`--novram` (ComfyUI), out-of-core textures (Redshift/Octane), CPU denoise. Trades speed for capacity.
5. **Shrink the scene/model.** Decimate geometry, lower texture resolution, prune unused channels, use proxies/instances, smaller model variant.

## Tool-specific flags

- **ComfyUI:** `--lowvram` (split model, keep minimal on GPU), `--novram` (maximum offload, slowest), `--medvram`/`--normalvram` (defaults vary by build), `--disable-smart-memory` (force ComfyUI to free models aggressively between runs instead of caching them in VRAM — use when a second workflow OOMs after the first), `--cache-none`. Enable **VAE tiling** node for decode OOM. Use **FP8** model weights and GGUF for big video models.
- **Blender Cycles:** disable **Persistent Data** when animation accumulates memory; lower **Tile Size** is mostly legacy (GPU is full-frame now) but **reduce final-render resolution / use border render**; switch denoiser to **OIDN on CPU** or lower OptiX memory; reduce **Dicing/Subdivision** and **Volume step**; reduce **Motion Blur steps**.
- **Redshift:** raise out-of-core thresholds but remember it offloads textures, not BVH; reduce **Geometry Memory** pressure via proxies; lower **Automatic Memory** / **Bucket size**; **Texture max resolution / mip** override globally downsamples textures.
- **Octane:** out-of-core textures (host memory) — geometry must still fit; reduce **mesh detail / displacement**; lower **render target resolution**; deep image / denoiser buffers are extra — disable if tight.

## Decision tree

```
OOM occurs...
├─ at model/scene LOAD (before sampling/render)
│    -> weights/scene too big: FP8/GGUF, --lowvram/--novram, decimate geo, smaller model
├─ DURING sampling/render, single frame
│    -> resolution/poly too high: lower res, tiling/VAE tiling, lower precision, proxies
├─ at DENOISE or UPSCALE step (render finished, then dies)
│    -> peak buffer spike: tiled/CPU denoise, lower upscale factor, free model before upscale
├─ only in ANIMATION (single frame is fine)
│    ├─ offline renderer -> disable Persistent Data, render 1 frame/process,
│    │                      cut motion-blur steps, clear caches between frames
│    └─ video diffusion  -> REDUCE FRAME COUNT/segment first, then --disable-smart-memory,
│                           VAE tiling, FP8
└─ on a SECOND workflow/scene after the first ran
     -> stale model cached in VRAM: --disable-smart-memory / free models, restart worker
```

## Quick reference

| Lever | Cost | Renderers | Video models |
|---|---|---|---|
| Lower resolution | quality/size | yes | yes |
| Reduce frame/segment count | shorter clip | n/a (per-frame) | biggest win |
| FP8 / GGUF / FP16 | minor quality | partial | yes |
| Tiling / VAE tiling | slower | yes | yes |
| Out-of-core (textures only) | much slower | Redshift/Octane | n/a |
| `--lowvram` / `--novram` | slower | n/a | ComfyUI |
| `--disable-smart-memory` | reload time | n/a | ComfyUI |
| Disable Persistent Data | slower anim | Cycles | n/a |
| CPU/tiled denoise | slower | all | n/a |
| Multi-GPU | none (does NOT add capacity) | bucket/frame split | frame split |

## Gotchas

- Multi-GPU never pools VRAM on consumer cards — adding a card does not let a non-fitting scene render.
- Out-of-core offloads textures, not the BVH/geometry core — a high-poly OOM stays an OOM.
- The denoiser/upscaler is the usual peak; a render that "just fit" dies when denoise allocates on top.
- Video-model VRAM scales with frame count, not only resolution — cut length before fighting resolution.
- ComfyUI caches models in VRAM between runs; a second workflow OOMs until `--disable-smart-memory` or a worker restart.
- Cycles Persistent Data speeds animation but inflates per-frame memory; disable it when animation OOMs.
- Motion blur multiplies resident geometry (sub-frame samples); high blur steps cause animation-only OOM.
- "Failed to allocate" mid-render after success on warm-up often means fragmentation/leak — restart the process to get a clean allocator.

---

# Per-Tool Playbooks

## Blender Cycles

1. **Confirm it's an animation-only OOM:** render frame 1 alone (F12). If it succeeds, the leak is cross-frame.
2. **Disable Persistent Data:** Render Properties > Performance > "Persistent Data" OFF. It caches the whole scene (geometry, BVH) in memory across frames; great for speed, costly for VRAM.
3. **Denoiser memory:** Render Properties > Sampling > Denoise. Switch from OptiX (GPU) to **OpenImageDenoise** and set device to **CPU** if the denoise step is the OOM peak. OptiX denoise allocates a large GPU buffer on top of the framebuffer.
4. **Motion blur:** Render Properties > Motion Blur > reduce sub-steps; high steps replicate deforming geometry in VRAM.
5. **Geometry:** apply Decimate, lower Subdivision render levels, lower Adaptive Subdivision dicing rate (Subdivision modifier > Dicing). Volumes: increase Volume step size (Render > Volumes) to cut volume grid memory.
6. **Textures:** Simplify panel (Render Properties > Simplify) > "Texture Limit" caps max texture size for render — global downsample without editing assets.
7. **Render one frame per process** (kills cross-frame accumulation entirely):
   ```bash
   for f in $(seq 1 240); do
     blender -b scene.blend -f $f
   done
   ```
   Each `-f` invocation is a fresh process with a clean allocator.

## Redshift

1. **Understand out-of-core:** Redshift streams **textures and (with penalty) geometry** from host RAM, but the **ray-tracing acceleration structure must fit in VRAM**. Geometry-bound OOM is not solved by out-of-core.
2. **Memory settings:** System tab > "Automatic Memory Management" on; if still OOM, manually lower the "Geometry Memory (MB)" and "Texture Cache" reservations so other allocations have room, OR raise out-of-core texture budget to spill more to host.
3. **Texture downsample:** Memory > "Max Texture Size" / mip bias to globally reduce texture footprint.
4. **Proxies:** convert heavy meshes to Redshift Proxies and instance them — instances share one geometry copy in VRAM.
5. **Bucket size:** lower bucket size reduces per-bucket framebuffer allocation slightly (minor lever).
6. **Multi-GPU reminder:** each GPU loads the full scene; adding GPUs speeds buckets but does not raise the per-card scene ceiling.

## Octane

1. Out-of-core moves **textures** to host RAM (Preferences > Out-of-core). **Geometry still must fit in VRAM.**
2. Reduce mesh detail and **displacement** subdivision level — displacement explodes geometry and BVH size.
3. Lower render-target resolution; deep-image and AOV buffers add VRAM — disable unused AOVs.
4. The denoiser (AI/spectral) needs extra buffers; if OOM hits at denoise, disable it or lower resolution.
5. Multi-GPU: full scene per card; no pooling on consumer GPUs.

## ComfyUI — WAN / LTX video diffusion

1. **Frame count is the dominant lever.** Memory roughly scales linearly with temporal length plus cross-frame attention overhead. Halve the frame count or render in overlapping segments and stitch.
2. **Launch flags** (set on the python launch line):
   ```bash
   python main.py --lowvram             # split model; keep minimum on GPU
   python main.py --novram              # maximum offload to system RAM (slowest)
   python main.py --disable-smart-memory # free models between runs instead of caching in VRAM
   python main.py --cache-none           # don't cache nodes' outputs
   ```
   Use `--disable-smart-memory` specifically when the *first* workflow runs but a *second* OOMs — ComfyUI keeps the prior model resident otherwise.
3. **FP8 / GGUF weights:** load FP8 or GGUF-quantized WAN/LTX checkpoints; cuts weight VRAM ~2-4x. Use a CLIP/text-encoder offload node to keep the encoder off-GPU during sampling.
4. **VAE tiling:** add the "VAE Decode (Tiled)" node — the VAE decode of a long video latent is a frequent OOM peak; tiling decodes in chunks. Lower tile size if still OOM.
5. **Resolution + segment:** drop to a supported lower resolution (e.g. 480p generation then upscale separately in a second pass with the model freed).
6. **Two-pass to dodge the upscale peak:** generate at low res, **free the diffusion model** (or restart with smart-memory disabled), then upscale in a separate workflow so the upscaler doesn't allocate on top of resident weights.
7. **torch allocator:** `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` reduces fragmentation OOM on long runs.

## VRAM-budget estimation worksheet

Approximate resident set before rendering:

```
Renderers (still):   geometry+BVH  +  textures(resident)  +  framebuffer(res x channels x AOVs)
                     +  denoiser buffer (≈ another framebuffer)   <- the peak
Video diffusion:     model weights  +  (latent_per_frame x frame_count)  +  attention activations
                     +  VAE decode buffer (peak at decode)
```

Rules of thumb:
- Denoiser/upscaler peak ≈ add one more full framebuffer/latent on top — budget for it.
- Doubling resolution ≈ 4x framebuffer/latent.
- Doubling frame count ≈ ~2x latent + super-linear attention growth.
- FP32 -> FP16 ≈ half weight/activation memory; FP16 -> FP8 ≈ half again.

## Frame-count vs VRAM (video diffusion, illustrative)

| Frames | Relative latent memory | Typical mitigation |
|---|---|---|
| 1 (image) | 1x | none needed |
| 16 | ~16x + attention | FP8 |
| 49 | ~49x + attention | FP8 + VAE tiling |
| 81 | ~81x + attention | FP8 + VAE tiling + --lowvram or segmenting |
| 121+ | very high | segment + stitch; reduce resolution |

Numbers are relative, not absolute — actual VRAM depends on model, resolution, and dtype. The point: frame count drives the curve.
