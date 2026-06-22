# Prompt Templates & Per-Model Settings

## Master prompt skeleton

```
[Subject + key appearance], [single main action/motion], [camera move + speed],
[setting + lighting direction], [style/mood/film-stock].
```

Example:
```
A weathered fisherman in a yellow raincoat, turning slowly to look off-screen left,
slow dolly push-in, on a storm-lit harbor at dusk with hard rim light,
moody anamorphic cinematic look, shallow depth of field.
```

## Character-consistency I2V recipe (any model)

1. In an image model, lock a hero still: correct face, outfit, lighting. Iterate here (cheap).
2. Upload as I2V **start frame**. (Add an **end frame** in Kling/Luma/Runway to bound the shot.)
3. Prompt ONLY motion + camera. Do not re-describe appearance.
4. Reuse the SAME still as a **reference** in every subsequent shot.
5. Lock seed; keep style sentence verbatim across shots.
6. For shot-to-shot: take last frame of shot A → start frame of shot B.

## Per-model camera & motion settings

| Model | Identity lock | Camera control | Motion dial | Negative field | Notes |
|---|---|---|---|---|---|
| **Kling** | Char/face reference, Elements (1.6+), start+end frames | Camera presets (push/pull/pan/tilt/truck/orbit) + motion brush/dynamic masks | Moderate default; stable, tolerates more motion | Yes | Pro mode for fidelity; mask region to localize motion |
| **Runway Gen-3/Gen-4** | Gen-4 References | Camera Control sliders (pan/tilt/zoom/roll/H/V) + motion brush | Keep mid-range; don't stack big camera + big action | No (phrase positively) | Short clips; strong with reference images |
| **Luma Dream Machine** | Keyframes (start, start+end) | Natural-language camera keywords | Implied via keyframe distance | No (phrase positively) | Excellent interpolation/looping between keyframes |
| **Hailuo / MiniMax** | Subject reference (S2V) | Camera-movement phrases in prompt | High by default — dial toward "subtle/slow" | Yes | Best realistic human motion/physics |
| **ComfyUI / WAN** | I2V start frame + ControlNet (pose/depth/flow) | Control via conditioning + prompt | denoise / CFG / motion params | Yes (neg prompt node) | Strongest open I2V; lower CFG if warpy/oversaturated |
| **ComfyUI / LTX** | I2V start frame | Prompt + conditioning | frame/motion params | Yes | Fast — use for cheap iteration before final |

## Camera-move vocabulary (use literally in prompts)

`slow dolly push-in` · `pull back / dolly out` · `pan left/right` · `tilt up/down` ·
`truck left/right (lateral track)` · `orbit / arc around subject` · `crane up` ·
`handheld follow` · `locked-off static shot` · `slow zoom in`. Always add a **speed**
word (slow/gradual/subtle) — bare camera verbs default to abrupt.

## Negative-prompt block (Kling / Hailuo / WAN / LTX)

```
melting, morphing, distorted face, warped features, extra limbs, extra arms,
extra fingers, fused fingers, deformed hands, mutated, duplicate, cloned face,
flicker, strobing, jitter, plastic skin, waxy, low detail, blurry, watermark, text
```

## Positive restatement (Runway / Luma — no negative field)

Fold the protections into the positive prompt:
```
...with anatomically correct hands, five fingers, a stable consistent face,
smooth coherent motion, no warping, steady framing...
```
(Phrasing it positively works better here than pasting a negative list, which is ignored.)

## Adherence checklist (per prompt)

- [ ] One main action only.
- [ ] Positive phrasing (no "no/without/avoid").
- [ ] Key idea stated twice in different words.
- [ ] Order: subject → action → camera → setting → style.
- [ ] Camera verb has a speed word.
- [ ] Appearance lives in the I2V frame, not the text (for I2V).
- [ ] Hands cropped/occluded if present.
- [ ] Face off-axis or moving if it's the main subject.

## Credit budget rule of thumb

```
shots_needed = storyboard_shots
gen_budget   = shots_needed * 3        # first pass rarely lands
iterate at:  low-res / short / fast model (LTX, lower Kling tier)
finalize at: high-res / upscale ONLY the locked take
change ONE variable per generation; log seed + prompt + reference per winning shot
```
