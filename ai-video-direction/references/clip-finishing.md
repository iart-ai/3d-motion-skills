# Clip Finishing Pipeline & Tool Settings

Finish raw AI-generated clips (from Kling, Runway, Luma, Hailuo, WAN/LTX, etc.) and integrate them into a real edit alongside footage and motion graphics. Treat AI clips as off-spec source that must be normalized before they touch the timeline. Three problem areas dominate: upscale/interpolation artifacts, AI roto failing on hair and shadows, and "foreign" clips that don't match the timeline (fps, resolution, and above all color).

## Ordered finishing pipeline

```
1. Trim head/tail warp frames (AI clips often have 1-2 bad frames each end).
2. Detect & split at every scene cut / flash / re-render in the NLE.
3. (If needed) stabilize each segment.
4. Interpolate to target fps — PER SEGMENT, never across a cut.
5. Upscale to delivery resolution (not beyond).
6. Light sharpen after upscale.
7. Roto (if compositing): auto body matte → manual edge/hair cleanup → separate shadow matte.
8. Color conform: tag input space → neutralize black/white → match to reference on scopes.
9. Drop into timeline; apply unifying grade across all clips.
10. Run the QC checklist.
```

Rule: interpolate BEFORE upscale (cheaper, fewer amplified errors). Grade LAST.

## 1. Upscale & interpolation artifacts

**Cut before you interpolate.** Frame interpolation (Topaz Chronos / Apollo, AE Pixel Motion, RIFE) assumes continuous motion. Run it across a scene cut and it tries to "tween" two unrelated frames → a smear/ghost on the cut. AI clips often contain implicit hard changes (a flash, a sudden re-render). Detect and split at every cut first, interpolate each continuous segment separately, then reassemble. Topaz has no reliable scene-cut detection — do the cutting in the NLE.

**Pick the model by motion type:**
- Fast, complex motion → use a model tuned for it; avoid optical-flow interpolation that ghosts on fast action. In Topaz, Apollo handles large motion better than Chronos for very fast moves; for upscale on AI/CG-origin footage use a model intended for digital/CG sources rather than one assuming camera-noise (avoid models that hallucinate grain detail on already-clean AI footage).
- Slow/smooth motion → optical-flow interpolation is fine and cleaner.
- **Ghosting/blur on fast motion** = wrong interpolation model or interpolating across a cut. Lower the interpolation factor, switch model, or interpolate only the slow segments.

**Don't over-upscale.** Going 2x is usually clean; 4x on a soft AI source amplifies warping and invents texture. Upscale to delivery resolution, not beyond. Sharpen *after* upscale, lightly.

**Order of operations:** stabilize (if needed) → cut at scene changes → interpolate per segment → upscale → grade. Never interpolate after upscaling (you waste compute and amplify upscale errors).

### Topaz Video AI — model picker

| Source / goal | Suggested approach |
|---|---|
| Fast/large motion interpolation | Apollo (handles big displacement better than Chronos); lower target multiplier if ghosting |
| Slow/smooth interpolation | Chronos / optical-flow interpolation |
| Upscaling clean AI/CG footage | Use a model intended for digital/CG/already-clean sources; AVOID models that assume camera grain/compression (they hallucinate texture on AI footage) |
| Upscaling noisy/compressed source | A denoise+enhance model is fine here |
| Scene cuts present | Topaz has NO reliable cut detection — split in the NLE first, process segments separately |

Settings discipline:
- Prefer 2x upscale; only go higher if delivery demands it. 4x amplifies warp.
- Do not stack heavy sharpen — soft AI footage sharpened hard looks crunchy.
- Interpolation factor: match exactly the source→target ratio; arbitrary high factors invent motion and ghost.

## 2. AI roto on hair, shadows, edges

Auto-roto (Runway, AE Roto Brush 2/3, Resolve Magic Mask) is fast but **breaks predictably on fine hair, motion blur, soft shadows, and low-contrast edges** — the matte flickers, eats wisps of hair, or chases cast shadows as if they were the subject.

**Hybrid workflow (best results):**
1. Get a rough fast matte where the auto tool is strong (Runway / Roto Brush on the solid body).
2. Bring it into After Effects and **clean the edges manually**: Refine Edge / Refine Soft Matte brush only along the hair boundary; feather and choke the matte; add a small edge blur to hide stair-stepping.
3. **Handle shadows separately** — do not let the body matte include the cast shadow; roto the shadow as its own looser matte (or recreate it as a soft drawn shape) so it doesn't flicker.
4. For flickering mattes across frames, smooth the matte over time (reduce per-frame jitter) and check every Nth frame, fixing keyframes by hand at the worst frames.

**Edge cleanup specifics:** decontaminate edge colors (remove the old background's color fringe), choke the matte 1-2px inward on hard edges, keep more softness on hair. A matte that is too sharp screams "composited."

### After Effects — roto & edge cleanup

1. **Roto Brush 2/3** for the solid body; propagate, then scrub to find break frames.
2. **Refine Edge tool** strictly along hair/fur boundaries (paint only the wispy zone).
3. **Refine Soft/Hard Matte** effect: feather, choke (1-2px inward on hard edges), reduce chatter.
4. **Decontaminate Edge Colors** to kill the old background fringe.
5. Add a tiny **edge blur** to hide stair-stepping; keep more softness on hair than on body.
6. **Shadows**: roto/recreate as a SEPARATE, looser, soft matte — never inside the body matte.
7. **Temporal flicker**: smooth matte over time; hand-fix keyframes on the worst frames; check every Nth frame.

Alt fast path: rough matte in Runway (auto) → export with alpha → finish edges in AE as above.

## 3. Normalizing foreign clips into the edit

AI clips arrive off-spec. Conform three things, in this order: **fps → resolution → color.** Color is where most AI clips betray themselves next to motion graphics.

**fps conform:** AI clips are often 24/25/30 or an odd rate. Match the timeline rate. If the clip rate differs, **interpolate (retime) to the timeline fps** — do NOT just drop/duplicate frames (judder), and don't let the NLE silently conform with frame-blend off. Decide between true frame-rate conversion vs. interpreting footage to a new rate; for motion-matched cuts, interpolate.

**Resolution conform:** upscale (see section 1) to the sequence resolution; never let the NLE scale up a small clip on the fly at export (soft + slow). Match pixel aspect (square) and check for added letterbox/padding.

**Color conform (the big one):** AI clips frequently come in a different gamma/color space (often sRGB-ish, sometimes with a slight contrast/saturation bias) and **clash with graded footage and motion graphics.** Steps:
1. **Set the correct input color space / tag** so the NLE interprets it right (don't let a Rec.709 timeline read an sRGB clip raw, or it shifts).
2. **Neutralize**: match black point and white point first (levels/lift-gamma-gain), then white balance.
3. **Match to a reference frame** from the surrounding footage — use scopes (waveform + vectorscope), not eyeballs. Line up blacks, mids, highlights, then hue.
4. For dropping next to **motion graphics**, ensure consistent gamma so the AI clip's midtones don't sit darker/lighter than the graphics. A common tell: AI clip looks slightly milky/low-contrast — add a gentle contrast/black-level to seat it.
5. Apply a final unifying grade across the whole timeline so AI and non-AI share one look.

### Color conform — Resolve / NLE

1. **Input transform / clip color space**: tag the AI clip's actual space (commonly sRGB / Rec.709-ish). On a Rec.709 timeline, an untagged sRGB clip shifts — set the input color space or a CST/IDT so it's interpreted correctly BEFORE grading.
2. **Scopes on**: Waveform (luma), Parade (RGB balance), Vectorscope (hue/sat).
3. **Neutralize**: set black point (lift) and white point (gain) to match a reference frame from surrounding footage; then balance mids (gamma) and white balance.
4. **Match**: pull a still from an adjacent real/MoGraph shot; line up blacks → whites → mids → hue against it on scopes, not by eye.
5. **Seat next to motion graphics**: ensure midtone gamma matches the graphics; AI clips often read milky/low-contrast — add a gentle S-curve / black-level.
6. **Unifying grade**: one look node/adjustment layer across the whole timeline so AI and non-AI share grain, contrast, and color cast.

### fps / resolution conform

| Issue | Right move | Wrong move (avoid) |
|---|---|---|
| Clip fps ≠ timeline | Retime with motion interpolation (optical flow / Timewarp / Resolve Optical Flow) | Drop/duplicate frames (judder) |
| Interpreting vs converting | Interpolate for motion-matched cuts; reinterpret only if exact same content at new rate | Silent NLE conform with frame-blend off |
| Resolution < sequence | Upscale offline to sequence res (section 1) | On-the-fly upscale at export (soft, slow) |
| Pixel aspect / padding | Force square pixels; crop added letterbox/padding | Leaving baked-in black bars |

## QC checklist before export

Run every clip through this before final render:

- [ ] fps matches timeline; retime used interpolation (no judder/duplicated frames).
- [ ] Resolution matches sequence; not on-the-fly upscaled at export.
- [ ] No interpolation smear across cuts (each continuous segment interpolated separately).
- [ ] No ghosting/blur on fast motion (right model, sane interpolation factor).
- [ ] Not over-upscaled (no invented texture/amplified warp).
- [ ] Roto edges clean on hair; shadows handled separately; matte not flickering.
- [ ] No background color fringe on roto edges (decontaminated).
- [ ] Input color space tagged correctly; clip not gamma-shifted.
- [ ] Blacks/whites/mids matched to surrounding footage on scopes.
- [ ] AI clip seats next to motion graphics (contrast/gamma consistent).
- [ ] Unifying grade applied across timeline.
- [ ] Audio sync / clip length correct; no trailing AI-warp frames at head/tail.

## Quick reference

| Problem | Cause | Fix |
|---|---|---|
| Smear across a cut | Interpolated over a scene change | Cut first, interpolate each segment |
| Ghosting on fast motion | Wrong interpolation model / too high factor | Switch model (Apollo for big motion), lower factor |
| Mushy 4x upscale | Over-upscaling soft AI source | Upscale only to delivery res; sharpen lightly after |
| Hair eaten by matte | Auto-roto on fine edges | Hybrid: auto body + manual Refine Edge in AE |
| Shadow flickering in matte | Body matte chasing shadow | Roto shadow as separate looser matte |
| Composited-looking edge | Sharp matte + color fringe | Choke + feather + decontaminate edge color |
| Clip juddering | fps mismatch, frame drop/dup | Retime with interpolation to timeline fps |
| Clip "pops" / looks milky | Wrong color space, gamma shift | Tag input space, match on scopes, add contrast |

## Common AI-clip tells to neutralize

- Slight low-contrast / milky blacks → add black level + contrast.
- Oversaturated push → desaturate gently, check vectorscope.
- Gamma sits brighter than graphics → match midtone gamma to the MoGraph.
- Warped head/tail frames → trim.
- Too-clean look vs grainy plate → add matching grain in the unifying grade.

## Gotchas

- **Interpolating across a cut is the #1 smear cause** — Topaz won't warn you; split at cuts yourself.
- AI footage is already clean — upscale models that "add detail/grain" hallucinate noise; pick CG/digital-source models.
- Roto **shadows are not the subject** — separate matte, or they flicker and crawl.
- Letting the NLE auto-conform fps with frame blending off produces judder silently.
- An untagged sRGB AI clip on a Rec.709 timeline shifts gamma/contrast — fix the tag before grading.
- Matching color by eye fails; use waveform + vectorscope against a reference frame.
- AI clips often have 1-2 warped frames at head/tail — trim them before integrating.
