# AI Video Direction

Get controllable, temporally consistent AI video out of Kling, Runway (Gen-3/Gen-4), Luma Dream Machine, Hailuo (MiniMax), and the open ComfyUI/WAN/LTX stack — then upscale, roto, and conform those clips cleanly into a real NLE / After Effects edit. For creators and editors who want consistency and clean integration without burning credits on bad takes.

## When it activates

- "Make my AI video consistent" or "stop my character morphing between shots."
- "Control the camera in Kling/Runway/Luma/Hailuo" or "fix melting / extra limbs."
- "Write a better video prompt" or "the motion is frozen / too warpy."
- "Fix Topaz ghosting on fast motion," "AI roto is breaking on hair/shadows," or "match the color / conform fps of my generated clip."

## Example prompts

- "Stop my character's face from melting in Runway and keep her identity consistent across three shots."
- "Write a Kling prompt with a slow push-in and subtle motion that won't warp the frame."
- "My interpolated AI clip smears across a cut and the color pops in my edit — walk me through the finishing pipeline."

## What's inside

- `SKILL.md` — I2V consistency anchoring, prompt structure for adherence, per-model camera/motion control, credit-efficient iteration, and the finishing/integration pipeline (upscale, roto, fps/resolution/color conform).
- `references/prompt-templates.md` — copy-paste prompt skeletons, the character-consistency I2V recipe, per-model camera/motion setting tables, and negative-prompt blocks.
- `references/clip-finishing.md` — the ordered finishing pipeline, Topaz model picker, AE roto/refine workflow, Resolve/NLE color-conform steps, and a pre-export QC checklist.

---
Part of **[3D Motion Skills](../)** · Built by **[iart.ai](https://iart.ai)** — controllable Motion Graphics MP4 from a prompt or data.
