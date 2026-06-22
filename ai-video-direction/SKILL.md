---
name: ai-video-direction
description: This skill should be used when the user asks to "make my AI video consistent", "stop my character morphing between shots", "control the camera in Kling/Runway/Luma/Hailuo", "fix melting/extra limbs in AI video", "write a better video prompt", "the motion is frozen / too warpy", "clean up my AI video clip", "fix Topaz ghosting / blur on fast motion", "my interpolation is smearing across a cut", "AI roto is breaking on hair/shadows", "match the color of my AI clip to my edit", "conform fps/resolution of a generated clip", or wants to get controllable, consistent generations out of image-to-video and text-to-video models without burning credits and then upscale, roto, and integrate those clips cleanly into a real NLE/After Effects edit.
version: 0.1.0
---

# AI Video Direction

Get controllable, temporally consistent AI video out of Kling, Runway (Gen-3/Gen-4), Luma Dream Machine, Hailuo (MiniMax), and the open ComfyUI/WAN/LTX stack. The core levers are: anchor with image-to-video so compute goes to motion not appearance, structure prompts for adherence, control camera and motion strength per model, and iterate cheaply before spending on upscales. (Note: as of 2026 the OpenAI Sora app/API is winding down; prefer the models above.)

## When to use

Use when generations suffer from: a character/face/outfit that morphs shot to shot, melting or extra/duplicated limbs, prompts that get ignored, motion that is either frozen (nothing moves) or warps the whole frame, uncontrollable camera, or a credit budget vanishing on bad takes. Applies to both closed APIs and local ComfyUI/WAN/LTX graphs.

## 1. Consistency: anchor with image-to-video (I2V)

The biggest lever for consistency is **starting from a still image (I2V) rather than text-to-video (T2V)**. In T2V the model must invent appearance *and* motion every frame, so identity drifts. With I2V the appearance is locked by the input frame, and the model spends its compute on motion only. Workflow:

1. Generate (or shoot) a clean still of the character/scene in a controllable image model — get face, outfit, lighting right *here*, where iteration is cheap.
2. Feed that still as the I2V start frame. Optionally provide an end frame (Kling, Luma, Runway support start+end keyframes) to constrain the whole shot.
3. Prompt only the **motion and camera**, not the appearance (the frame already encodes appearance).

**Character locking / references**: use the model's reference feature — Runway Gen-4 References, Kling's character/face reference and 1.6+ "Elements", Luma keyframes, Hailuo subject reference (S2V). Reuse the *same* reference image across shots for continuity.

**Do not animate a static face head-on.** Front-on, perfectly still portraits are where melting, eye-warping, and dental horror appear. Give the subject a slight angle, an existing implied motion, or frame so the face is not the dead-center largest element.

**Frame hands at the edges or out of frame.** Hands are the most common artifact source (extra fingers, fusing). Compose so hands are partially cropped, behind objects, or near frame edges where errors are less visible.

**Negative prompts** (where supported — Kling, Hailuo, WAN/ComfyUI): block the known failure modes:
`melting, morphing, distorted face, extra limbs, extra fingers, deformed hands, warping, duplicate, flicker, jitter, plastic skin`.
Runway/Luma have no classic negative field — phrase positively instead ("steady, anatomically correct hands, stable face").

**Multi-shot continuity**: lock seed where available, reuse the same reference image and the same style phrasing verbatim across prompts, keep lighting direction and color description identical, and use the last frame of shot A (or a matched still) as the start frame of shot B.

## 2. Prompt structure for adherence

Models reward clear, ordered, positive prompts. Use this skeleton:

> **[Subject + appearance] + [action/motion] + [camera move] + [setting/lighting] + [style/mood]**

Rules that materially improve adherence:
- **Phrase positively.** Say what should happen, not what should not. "Empty street, no people" still summons people; say "a deserted street at dawn."
- **Repeat the key idea** in different words once. If the camera push is the point, mention it in the camera clause and again in mood ("slow, deliberate push-in"). Restating anchors the model.
- **One main action per shot.** Multiple simultaneous actions dilute adherence and increase morphing. Chain shots instead of cramming.
- **Concrete over adjective soup.** "tracking left as she walks past brick storefronts" beats "cinematic dynamic epic motion."
- Keep prompts moderate length; walls of adjectives get averaged away. Kling and Hailuo respond well to ~1-3 tight sentences.

## 3. Camera and motion-strength control (per model)

Motion strength is a balance: **too much → warping/melting; too little → frozen, photo-like result.** Tune it as a dial, not max-it-out.

- **Kling**: use explicit camera-movement controls/presets (push in, pull out, pan, tilt, truck, orbit) plus prompt verbs. Has a "professional mode" and motion brush / dynamic masks to move *only* a region. Start motion moderate; Kling tends stable, so it can take more motion before warping.
- **Runway (Gen-3/Gen-4)**: use Camera Control (horizontal/vertical/pan/tilt/zoom/roll sliders) for precise moves; keep Motion/Camera intensity mid-range. Gen-4 References for identity. Avoid stacking a big camera move with a big subject action in one short clip.
- **Luma Dream Machine**: rely on **keyframes** (start, and start+end) for control; camera via natural-language "camera" keywords. Luma loops and interpolates between keyframes well — exploit start+end to constrain motion.
- **Hailuo / MiniMax**: strong realistic human motion; use camera-movement instructions in prompt and subject reference (S2V). Tends toward more motion by default — dial prompts toward "subtle, slow" if warping.
- **ComfyUI / WAN / LTX (open)**: control via denoise/CFG, motion/frame settings, and ControlNet-style conditioning (depth, pose, optical flow). WAN gives strong I2V; LTX is fast for cheap iteration. Lower CFG slightly if output is over-saturated or warpy; use a pose/depth control to lock motion paths.

**Motion brush / masking**: paint motion only where wanted (Kling motion brush, Runway motion brush). Too much painted area → global warp; too little → frozen. Mask the moving subject, leave the background static for stability.

## 4. Credit-efficient iteration

AI video burns budget fast. Discipline:

- **Multiply your estimate by ~3.** First idea rarely lands; budget for ~3 generations per usable shot minimum.
- **Lock the seed and all inputs** once a take is close, then change exactly one variable per generation so you learn what each change does.
- **Test cheap, then upscale.** Iterate at low resolution / short duration / a fast model (LTX, lower-tier Kling/Luma) until the motion and composition are right; only then re-run the final at high res or push through an upscaler. Never iterate at max quality.
- Generate the **shortest clip that proves the motion** (often 5s) before committing to longer.
- Save prompts + seeds + reference images per shot so a winning recipe is reproducible across a sequence.

See `references/prompt-templates.md` for copy-paste prompt templates, per-model setting tables, and negative-prompt blocks.

## 5. Finishing & integration

Generation is only half the job. Raw AI clips arrive off-spec and must be normalized before they drop into an NLE / After Effects timeline next to footage and motion graphics. Treat the clip as foreign source: finish it, then conform it.

**Upscale & interpolation artifacts.** Frame interpolation (Topaz Chronos/Apollo, AE Pixel Motion, RIFE) assumes continuous motion. Run it across a scene cut and it tweens two unrelated frames into a smear — so **split at every cut first and interpolate each continuous segment separately** (Topaz has no reliable cut detection). For fast/large motion use a model built for it (Apollo over Chronos) to avoid ghosting/blur; lower the interpolation factor if it ghosts. Don't over-upscale — 2x is clean, 4x on soft AI source amplifies warping and invents texture. Order: trim head/tail warp frames → cut at scene changes → interpolate per segment → upscale → light sharpen → grade.

**AI roto on hair, shadows, edges.** Auto-roto (Runway, AE Roto Brush, Resolve Magic Mask) breaks predictably on fine hair, motion blur, and soft shadows — flickering mattes that eat wisps of hair or chase cast shadows. Use a hybrid pass: auto matte the solid body, then clean the hair boundary by hand in AE (Refine Edge, feather/choke 1-2px, decontaminate edge colors, small edge blur). **Roto shadows as a separate looser matte** — never inside the body matte, or they flicker.

**Conform foreign clips into the edit.** Normalize three things in order — **fps → resolution → color.** Match the timeline fps by interpolating (retiming), never by dropping/duplicating frames (judder). Upscale offline to sequence resolution rather than letting the NLE scale on the fly at export. Color is the big tell: tag the clip's input color space (often sRGB on a Rec.709 timeline) before grading, neutralize black/white points, then match to a reference frame on scopes (waveform + vectorscope), not by eye. AI clips often read milky/low-contrast — add gentle contrast so they seat next to motion graphics, then apply one unifying grade across the whole timeline.

See `references/clip-finishing.md` for the ordered finishing pipeline, Topaz model picker, AE roto/refine steps, Resolve/NLE color-conform steps, and a pre-export QC checklist.

## Quick reference

| Goal | Lever |
|---|---|
| Lock character identity | I2V start frame + reference feature, reuse same image |
| Stop face melting | Don't animate static front-on face; add slight angle/motion |
| Reduce hand artifacts | Frame hands at edges / cropped / occluded |
| Improve prompt adherence | Subject+motion+camera+style order, positive phrasing, repeat key idea |
| Block melting/extra limbs | Negative prompt (Kling/Hailuo/WAN) or positive restatement (Runway/Luma) |
| Frozen result | Raise motion strength / add explicit camera move |
| Warping result | Lower motion strength, mask motion region, simplify action |
| Multi-shot continuity | Same seed + reference + style text; last frame → next start frame |
| Save credits | ×3 estimate, lock seed, test cheap, upscale last |
| Smear across a cut | Split at cuts first, interpolate each segment separately |
| Ghosting on fast motion | Right interpolation model (Apollo), lower factor |
| Hair/shadows breaking on roto | Auto body matte + manual AE edge cleanup; shadow as separate matte |
| AI clip "pops" in the edit | Conform fps → res → color; tag input space, match on scopes |

## Gotchas

- **T2V for a recurring character is a trap** — always I2V from a locked still.
- Negative words in a **positive** prompt summon the thing ("no rain" → rain). Use the negative field, or rephrase.
- Maxing motion strength does not give "more cinematic" — it gives warping. Mid-range plus a defined camera move reads as cinematic.
- Runway/Luma have **no negative prompt field**; people paste negatives there and they do nothing — phrase positively.
- Front-on static portraits are the #1 melting source; reframe.
- Iterating at full resolution wastes most of the budget; prove the shot cheap first.
- Sora app/API is being wound down in 2026 — don't build a pipeline around it; use Kling/Runway/Luma/Hailuo or the open WAN/LTX stack.

## Reference files

- `references/prompt-templates.md` — Copy-paste prompt skeletons, character-consistency I2V recipe, per-model camera/motion setting tables, and negative-prompt blocks.
- `references/clip-finishing.md` — Ordered finishing pipeline, Topaz model picker, AE roto/refine workflow, Resolve/NLE fps/resolution/color-conform steps, and a pre-export QC checklist.
