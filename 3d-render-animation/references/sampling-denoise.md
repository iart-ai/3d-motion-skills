# Render Sampling & Denoising

Cut render times without trading away noise or detail. The core skill is knowing **which sample count actually costs time**, using **adaptive sampling** to spend samples only where needed, and treating the **denoiser as a finisher for the last few percent of noise — never as a crutch** that smears detail into plastic. For animation, the extra job is killing temporal flicker by locking the noise pattern across frames.

Applies when renders are slow, grainy, or fireflies persist; when the denoiser produces a waxy/plastic/painterly look or smears fine detail (hair, fabric, foliage); when animation shimmers, boils, or flickers frame-to-frame; or when tuning AA/camera/light/GI samples in Cycles, Redshift, Octane, or Arnold.

## The sampling hierarchy: AA samples are squared

The single most important cost fact: **anti-aliasing / camera samples multiply everything**. Many renderers nest sampling — total rays ≈ (AA samples) x (per-effect samples). Raising AA from 4 to 8 doesn't double cost; because per-pixel rays scale with AA and each AA sample fires the secondary (diffuse/specular/GI) samples, the effective cost grows roughly with the **square** of the AA/camera sample setting in squared-sampling renderers (classic Arnold/Mental Ray model). Per-light and GI samples are *inside* that loop and far cheaper to raise.

Practical rule by noise type:

- **Noise everywhere / edge aliasing** -> raise **AA / camera samples** (expensive — last resort, raise modestly).
- **Noise in shadows / on a specific light** -> raise that **light's samples** (cheap, targeted).
- **Noise in indirect light / GI bounces / diffuse** -> raise **GI / diffuse / per-effect samples** (cheaper than AA).
- **Fireflies (bright single pixels)** -> clamp indirect (see below), not more AA.

Always push noise down at the **cheapest level that fixes it** before touching AA. In Arnold specifically: keep Camera (AA) as low as the edges allow, then raise Diffuse/Specular/SSS/Volume samples to clear their respective noise — this is dramatically faster than brute-forcing AA.

## Adaptive sampling + noise threshold

Adaptive sampling fires the **max** samples only on noisy pixels and stops early on clean ones (flat walls, sky). This is the highest-leverage speed win and should almost always be on.

- Set a **noise threshold** (the target per-pixel noise level): lower = cleaner + slower. Typical: **0.01** final, **0.05** preview. Cycles default 0.01; Redshift/Octane expose similar thresholds.
- Set **min and max samples**: max is the ceiling for hard pixels; min prevents adaptive from quitting too early on subtle gradients (raise min if smooth areas look blotchy/undersampled).
- The threshold, not the max, controls quality; the max just caps worst-case time. Tune threshold first.

## Clamping fireflies (without nuking highlights)

Fireflies come from rare high-energy indirect samples. Use **indirect clamping** to cap them, leaving direct light untouched so highlights stay bright:

- Cycles: Render > Sampling > Clamping > **Clamp Indirect** (e.g. 10). Leave **Clamp Direct = 0** so direct highlights aren't dimmed.
- Set clamp as high as still kills fireflies; too-low clamp darkens GI and flattens energy.

## The denoiser: a finisher, not a crutch

A denoiser estimates the clean image from a noisy one. Run it on an **already nearly-clean** render (denoise the last few percent of noise) and it's invisible. Run it on a **heavily noisy** render to "save samples" and it produces the telltale failures: **plastic/waxy skin, painterly smearing, loss of fine high-frequency detail (hair, pores, fabric weave, foliage), blotchy low-frequency patches, and temporal boiling in animation.**

Rules:

- Render to a reasonable noise floor **first** (adaptive threshold ~0.01–0.02), then denoise. Do not drop samples drastically and lean on the denoiser.
- **Feed the denoiser AOV/feature guides** — albedo and normal passes. AI denoisers (OptiX, OIDN) use albedo+normal to preserve texture detail and edges; without them they smear. Cycles: enable "Denoising Data" passes / the denoiser's albedo+normal inputs.
- **OIDN vs OptiX:** OIDN (CPU, also GPU now) is generally higher quality / less smeary for finals; OptiX (GPU) is faster for viewport/preview. Use OptiX interactive, OIDN final.
- If detail is being lost, the fix is **more samples before denoise**, a **higher-quality denoiser with feature passes**, or **denoiser strength/blend reduction** — not a stronger denoiser.

## Temporal flicker in animation (the hard part)

Per-frame denoising and per-frame random sampling each cause flicker. Two distinct problems:

1. **Sampling-pattern flicker / boiling.** If the renderer randomizes the noise seed every frame, residual noise *moves* each frame and reads as boiling/shimmer. **Lock the pattern:** disable "**randomize pattern each frame**" / use the **same sampling seed across frames** (Cycles: Sampling > Advanced > "Seed" — keep constant or animate deliberately; some pipelines set a fixed seed). With a locked seed, residual noise sits still and reads as static grain (far less objectionable, and a denoiser removes it cleanly).
2. **Denoiser flicker.** A spatial-only denoiser solves each frame independently, so its smeared regions differ frame-to-frame -> flicker. Use a **temporal denoiser** that looks at neighboring frames (Cycles supports temporal denoising via OIDN with motion vectors + the full frame sequence; Redshift, Arnold (Noice), Octane have temporal denoise modes). Temporal denoise **requires motion vector AOVs** and the surrounding frames, so it runs as a post pass over the rendered sequence, not per-frame in isolation.

Also raise the **noise floor consistency**: render every frame to the same adaptive threshold so per-frame noise level is uniform; a varying noise level itself flickers after denoise.

## Per-engine notes

- **Cycles:** Adaptive Sampling on; Noise Threshold 0.01; Max Samples as ceiling; constant Seed for animation; OIDN with Albedo+Normal passes for final; temporal denoise via OIDN over the sequence with motion vectors; Clamp Indirect for fireflies.
- **Redshift:** Unified Sampling with Adaptive Error Threshold (lower = cleaner); raise per-light samples for shadow noise rather than global; Altus/OptiX/OIDN denoise — use feature passes; temporal denoise mode for animation; Max Subsample/Brute-force GI samples for indirect noise.
- **Octane:** PathTracing/PMC kernel with Adaptive Sampling + Noise Threshold; raise Max Samples; AI denoiser with "Denoise on completion" + albedo/normal; **enable the spectral/AI denoiser's temporal mode** and lock seed for animation.
- **Arnold:** Camera (AA) low; raise Diffuse/Specular/SSS/Volume samples per noise type (squared-AA cost model); Adaptive Sampling with Max Camera (AA) + Adaptive Threshold; **Noice** is Arnold's temporal denoiser (run over the sequence with variance + feature AOVs); OptiX/OIDN for interactive.

## Quick reference

| Symptom | Fix (cheapest first) |
|---|---|
| Noise everywhere / aliased edges | Raise AA/camera samples (expensive — last) |
| Shadow / single-light noise | Raise that light's samples |
| GI / diffuse / indirect noise | Raise GI / diffuse / per-effect samples |
| Fireflies | Clamp Indirect (keep Clamp Direct 0) |
| Slow but clean | Adaptive sampling + raise noise threshold |
| Plastic / smeared detail | More samples before denoise + feed albedo/normal |
| Animation boiling | Lock seed (no per-frame randomize) |
| Animation denoiser flicker | Temporal denoise (motion vectors, over sequence) |

## Gotchas

- AA/camera samples scale cost ~quadratically in squared-sampling renderers; clear noise at the cheapest level first.
- Denoising a heavily-noisy render to save time is the #1 cause of the plastic/painterly look — render to a noise floor first.
- Denoiser without albedo+normal feature passes smears texture and edges; always feed them.
- Per-frame random seed makes residual noise boil; a constant seed makes it sit still and denoise cleanly.
- Spatial-only denoisers flicker in animation; temporal denoise needs motion vectors and the neighboring frames (post pass).
- Clamp Direct > 0 dims real highlights; clamp only Indirect for fireflies.
- A varying adaptive noise level across frames flickers after denoise — render every frame to the same threshold.
- Raising Max Samples without lowering the noise threshold often does nothing; the threshold gates quality.

---

# Settings Tables

## Noise-diagnosis flowchart

```
See noise? Identify WHERE before raising anything.
├─ On geometry edges / everywhere uniformly  -> AA/Camera samples (raise modestly; expensive)
├─ In/near shadows of one light             -> that LIGHT's samples
├─ Soft area-light penumbra                  -> light samples + light size sanity check
├─ In bounced/indirect/diffuse regions       -> GI / diffuse / per-effect samples
├─ Bright isolated dots (fireflies)          -> Clamp Indirect (NOT more AA)
├─ Glossy reflections sparkle                -> specular/reflection samples + roughness clamp
└─ Volumes/fog grainy                        -> volume samples / step size
```

## Cycles settings

| Setting | Preview | Final still | Animation |
|---|---|---|---|
| Adaptive Sampling | On | On | On |
| Noise Threshold | 0.05 | 0.01 | 0.01 (same every frame) |
| Max Samples | 256 | 1024-4096 | 1024-4096 |
| Min Samples | 0 (auto) | raise if blotchy | raise if blotchy |
| Seed | any | any | **constant** (lock) or deliberately animated |
| Clamp Indirect | 10 | 10 (0 = off if no fireflies) | 10 |
| Clamp Direct | 0 | 0 | 0 |
| Denoiser | OptiX (fast) | OIDN + Albedo+Normal | OIDN temporal over sequence |
| Denoising passes | — | Albedo + Normal on | Albedo + Normal + Motion vectors |

Cycles paths: Render Properties > Sampling (threshold, max/min, seed); Render Properties > Sampling > Denoise (denoiser, passes); Sampling > Advanced > Clamping (Direct/Indirect); View Layer Properties > Passes > Data > Vector (motion vectors for temporal denoise).

## Redshift settings

| Setting | Preview | Final | Animation |
|---|---|---|---|
| Unified Min Samples | 4 | 16 | 16 |
| Unified Max Samples | 64 | 256-1024 | 256-1024 |
| Adaptive Error Threshold | 0.05 | 0.01 | 0.01 (uniform) |
| Per-light Samples | default | raise for shadow noise | raise for shadow noise |
| GI (Brute Force) rays | low | raise for indirect noise | raise |
| Denoiser | OptiX | OptiX/OIDN + features | Temporal denoise mode |

Redshift: lower the Adaptive Error Threshold to clean up globally; prefer raising the noisy *light's* samples over global max samples. Enable feature/AOV passes for the denoiser. Use the temporal denoise option for sequences.

## Octane settings

| Setting | Preview | Final | Animation |
|---|---|---|---|
| Kernel | PathTracing | PathTracing/PMC | PathTracing |
| Max Samples | 400 | 2000-8000 | 2000-8000 |
| Adaptive Sampling | On | On | On |
| Noise Threshold | 0.05 | 0.01-0.02 | uniform across frames |
| GI Clamp | ~1-4 (firefly control) | tune | tune |
| Denoiser | AI denoiser (preview) | AI denoiser on completion + albedo/normal | AI denoiser **temporal mode** |
| Static Noise / Seed | — | — | lock seed (no per-frame randomize) |

Octane: turn OFF random seed per frame for animation (Render Settings > "Static noise" behavior) so residual grain doesn't crawl; enable the denoiser's temporal mode and feed it the AI light buffer/albedo where available.

## Arnold settings

Arnold uses a squared-sampling model: total rays per effect ≈ (Camera AA)^2 x (effect samples)^2. Keep Camera (AA) as low as edges allow.

| Setting | Preview | Final | Notes |
|---|---|---|---|
| Camera (AA) Samples | 3 | 3-6 | Squared cost — raise last |
| Diffuse Samples | 2 | 2-4 | Raise for diffuse/GI noise |
| Specular Samples | 2 | 2-4 | Raise for glossy noise |
| SSS Samples | 2 | 2-4 | Raise for skin noise |
| Volume Samples | 2 | 2-4 | Raise for fog/volume noise |
| Adaptive Sampling | On | On | Max Camera (AA) + Adaptive Threshold ~0.015 |
| Clamp | Clamp AA + Indirect | Clamp AA + Indirect | Tame fireflies; affects indirect |
| Denoiser | OptiX/OIDN (interactive) | **Noice** (temporal, over sequence) | Noice needs variance AOV + features |

Arnold workflow: set Camera AA to the minimum that resolves edges, then raise per-effect (Diffuse/Specular/SSS/Volume) samples until each region's noise clears. Enable Adaptive Sampling with a Max Camera (AA) ceiling. For animation, render the **variance AOV** and feature AOVs, then run **Noice** across the frame sequence for temporal-stable denoise.

## Temporal denoise pipeline (general)

1. Render the full frame sequence to a consistent noise floor (same adaptive threshold every frame).
2. **Lock the sampling seed** so residual noise is static, not boiling.
3. Output the required AOVs: **motion vectors** (for warping neighbor frames), **albedo**, **normal**, and (Arnold) **variance**.
4. Run the temporal denoiser as a **post pass over the sequence** (OIDN temporal, Redshift temporal, Octane temporal, Arnold Noice) so it can blend information across neighboring frames.
5. Review for residual flicker; if present, raise per-frame samples (lower threshold) rather than denoiser strength.

## Pre-render checklist

- [ ] Adaptive sampling ON; noise threshold set (0.01 final / 0.05 preview).
- [ ] Noise diagnosed by location; cheapest sample level raised first; AA/Camera kept minimal.
- [ ] Clamp Indirect set for fireflies; Clamp Direct left at 0.
- [ ] Denoiser fed Albedo + Normal feature passes.
- [ ] Render reaches a clean-ish noise floor BEFORE denoise (denoiser is a finisher).
- [ ] Animation: seed locked (no per-frame randomize).
- [ ] Animation: motion-vector AOV enabled; temporal denoise run over the sequence.
- [ ] Animation: every frame uses the same noise threshold (uniform noise level).
