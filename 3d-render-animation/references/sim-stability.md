# Sim Stability Checklist

Stabilize exploding, jittering, clipping, or non-deterministic physics simulations (rigid body, cloth/Vellum, pyro/smoke) across Blender, Cinema 4D, and Houdini. The same root causes recur in every solver: too few substeps for the velocities present, collision geometry that is too thin or interpenetrating at frame 1, and missing damping/friction. Fix in order; do not change everything at once.

Applies when a simulation: explodes (bodies launch to infinity or vibrate violently), jitters (objects buzz/vibrate at rest instead of settling), clips/tunnels through collision geometry, sinks into the floor, or produces different results on each bake. Also use when raising substeps made things *worse* instead of better.

## The ordered fix list

Work top to bottom. Stop when stable. Re-bake after each change; never judge stability from a cached old result.

### 1. Scale the scene to real-world units first

Solvers are tuned for meters/kilograms. A "building" modeled at 1 unit tall behaves like a 1m toy: gravity looks weak, objects float and jitter. A character at 100 units acts like a giant: everything explodes. Set the scene to real-world scale (a person ~1.7-1.8 units tall, gravity -9.8 m/s^2). In Houdini check the solver's unit/scale; in Blender set unit scale to 1.0 and model to size; in C4D check Project Settings scale. Wrong scale is the single most common cause of "physics looks wrong" and no amount of substeps fixes it.

### 2. Start the sim settled (no initial interpenetration)

At the first frame nothing should overlap. If two rigid bodies, or a cloth and a collider, share volume at frame 1, the solver applies a huge separation impulse on frame 2 and everything explodes. Before simulating: drop objects so they rest with a small gap (a few mm), or run a few "settle" frames and use that as the start state. For stacked rigid bodies leave a visible seam. For cloth, lift it slightly off the collider. Interpenetration at frame 1 is the #1 cause of explosions.

### 3. Raise substeps to satisfy the CFL condition

CFL (Courant-Friedrichs-Lewy): in one substep an object must not move more than ~one collision-thickness/cell. Fast objects passing thin colliders tunnel because between substep N and N+1 the object teleported past the wall. Increase substeps (per-frame solver iterations) until the fastest object moves <1 thickness per substep. Rule of thumb: double substeps and re-test; if tunneling stops, the budget was too low.

Caveat (critical): raising substeps the *wrong* way explodes cloth/Vellum. If constraint stiffness is high and substeps are raised without lowering the per-substep stiffness response, the solver overshoots. For Vellum, raise **Substeps** and let constraints solve, but if it explodes, *lower stiffness* or raise substeps gradually (e.g. 1→2→4→6) rather than jumping to 20. More substeps = smaller timestep = stiffer effective constraints.

### 4. Fix collision thickness / margins / interpolation

Thin or zero-thickness colliders are tunneled through. Give colliders real thickness:
- Increase collision margin / thickness so there is a buffer the solver can catch.
- Too *large* a margin makes objects float above surfaces with a visible gap — tune down until contact looks right but tunneling stops.
- Enable continuous / interpolated collision detection if available (catches fast objects between substeps).
- Use simplified convex/proxy collision shapes for rigid bodies; never collide against raw high-poly mesh when a box/hull works.

### 5. Add friction and damping

Jitter at rest and slow drift are usually missing friction/damping. Increase friction so resting objects stop sliding; increase linear/angular damping so velocities bleed off instead of oscillating. A small amount of damping kills buzzing without making motion look syrupy. For cloth, increase damping to stop the surface shimmering.

### 6. Tune constraint / solver iteration counts

If constraints (joints, cloth structural links, stacks) stretch or pop, raise constraint iterations (solver iterations), not substeps — iterations resolve constraint accuracy within a substep. Raise gradually; very high iterations slow the bake and can over-stiffen.

### 7. Cache / bake to disk for determinism

Live/interactive sims re-evaluate and can differ run to run, especially with adaptive substeps or scrubbing the timeline. For deterministic, repeatable results: bake to disk (Houdini: write a .sim/.bgeo cache via File cache / DOP I/O; Blender: Bake to disk in cache settings; C4D: cache the dynamics/Bake Object). Always simulate from frame 1 forward — never scrub backward and expect the same result. Disable adaptive substeps if exact reproducibility is needed.

## Per-app notes

**Blender** — Rigid Body World: raise **Steps Per Second** (substeps) and **Solver Iterations** in the Rigid Body World panel. Set **Collision Margin** small but nonzero; for thin objects use Mesh collision shape sparingly. Cloth: raise **Quality Steps**, add **Collision** quality steps and **Distance** (thickness), enable **Self Collision**. Always **Bake** the cache to disk; the point cache must start at frame 1. Mind **Unit Scale** in Scene properties.

**Cinema 4D** — Project Settings: raise **Steps per Frame** and **Maximum Solver Iterations** in the Dynamics/Project Dynamics settings. Increase **Collision Margin** and **Bounce/Friction** on the Dynamics Body tag. Use **Cache Object** / Bake to lock results. Soft bodies need higher steps; jitter usually means steps-per-frame too low or scale wrong.

**Houdini** — Bullet (RBD): raise **Substeps** on the solver and **Collision Padding**; prefer convex/box collision shapes over concave. For tunneling, raise substeps and check the object isn't moving faster than its size per substep. **Vellum**: raise **Substeps** and **Constraint Iterations**; if it explodes on higher substeps, lower stiffness or ramp substeps gradually. **Pyro**: tunneling/leaking through collision means **collision resolution too coarse** relative to substeps — raise substeps and/or use a denser collision volume; ensure the collision SDF is thick enough. Use **File Cache** SOP / DOP I/O for deterministic, re-readable sims.

## Quick reference

| Symptom | First suspect | Fix |
|---|---|---|
| Bodies explode/launch off floor | Interpenetration at frame 1 | Add gap, settle first |
| Everything explodes, scene tiny/huge | Wrong scene scale | Model to real-world meters |
| Fast object passes through wall | CFL violated, thin collider | Raise substeps + thickness + continuous collision |
| Cloth/Vellum explodes when substeps raised | Effective stiffness too high | Lower stiffness, ramp substeps gradually |
| Objects buzz/vibrate at rest | Missing friction/damping | Add friction + linear/angular damping |
| Objects float above surface | Collision margin too large | Reduce margin |
| Constraints stretch/pop | Too few iterations | Raise constraint/solver iterations |
| Different result each bake | Live/adaptive sim, scrubbing | Bake to disk, sim from frame 1, disable adaptive |
| Pyro leaks through collider | Coarse collision vs substeps | Raise substeps, denser/thicker collision volume |

## Gotchas

- Raising substeps is the default reflex but it makes cloth/Vellum **worse** if stiffness is high — ramp gradually, lower stiffness.
- Scale errors masquerade as every other problem; check scale before touching solver settings.
- A perfectly tuned solver still explodes if frame 1 has overlap — always settle first.
- Scrubbing the timeline backward corrupts live sims; bake before judging.
- Huge collision margins trade tunneling for floating; there is a sweet spot.
- High-poly concave colliders are slow and unstable; use proxies.

---

# Solver Settings Reference

## CFL condition (the math behind tunneling)

A solver advances time in substeps of `dt = (1 / fps) / substeps`. An object moving at velocity `v` travels `v * dt` per substep. Tunneling/instability happens when:

```
v * dt  >  collision_thickness   (object skips past the collider)
```

Solve for required substeps:

```
substeps  >  v * collision_thickness_in_seconds
substeps  ≈  (v_max / fps) / collision_thickness
```

Practical recipe: find the fastest-moving point (m/s), divide its per-frame travel by the collider's thickness, round up. If the answer is large (e.g. >20), it is cheaper to thicken the collider or enable continuous collision than to brute-force substeps.

## Diagnostic decision tree

```
Is the sim deterministic (same result each bake)?
├─ No  → bake to disk, sim from frame 1 forward, disable adaptive substeps. Then continue.
└─ Yes
   Does it explode (objects launch / vibrate violently)?
   ├─ Yes
   │   ├─ Is scene scale real-world (person ~1.8 units, g = -9.8)? → if no, FIX SCALE FIRST.
   │   ├─ Any overlap/interpenetration at frame 1? → add gap / settle first.
   │   └─ Did it explode only after raising substeps (cloth/Vellum)? → lower stiffness, ramp substeps gradually.
   └─ No
      Does a fast object pass through a collider (tunneling)?
      ├─ Yes → raise substeps to satisfy CFL, add collision thickness/margin, enable continuous/interpolated collision, use convex proxy.
      └─ No
         Do objects jitter/buzz at rest or drift slowly?
         ├─ Yes → add friction, add linear+angular damping, raise solver iterations.
         └─ No
            Do constraints/joints stretch or pop?
            └─ Yes → raise constraint/solver ITERATIONS (not substeps).
```

## Starting values (then tune)

| Parameter | Conservative start | When to raise |
|---|---|---|
| Rigid body substeps / steps-per-second | 10 (Blender SPS 60), C4D 5-10 steps/frame, Houdini 10 | Tunneling, fast motion |
| Solver / constraint iterations | 10 | Stretchy constraints, soft stacks |
| Cloth quality steps | 5-10 | Self-intersection, jitter |
| Vellum substeps | 1 → 2 → 4 → 6 (ramp) | Penetration; STOP if it explodes |
| Vellum constraint iterations | 50-100 | Stretchy/loose cloth |
| Collision margin/thickness | small nonzero (~mm at real scale) | Tunneling (raise), floating gap (lower) |
| Linear/angular damping | 0.04-0.1 | Buzzing, endless drift |
| Friction | 0.5 | Sliding that should stop |

## Per-app exact locations

### Blender
- Rigid Body: Scene Properties → **Rigid Body World** → *Steps Per Second*, *Solver Iterations*. Cache → *Bake*.
- Per body: Physics → Rigid Body → *Collision* → *Margin*, *Shape* (use Convex Hull / Box, not Mesh, when possible); *Sensitivity → Margin*.
- Cloth: Physics → Cloth → *Quality Steps*; → Collisions → *Quality*, *Distance* (thickness), *Self Collision → Distance*.
- Determinism: every cache panel → *Bake* (writes to disk). Scene → Units → *Unit Scale* = 1.0.

### Cinema 4D
- Global: Project Settings → **Dynamics** → *Steps per Frame*, *Maximum Solver Iterations per Step*, *Error Threshold*.
- Per object: Dynamics Body tag → *Collision* → *Margin*, *Bounce*, *Friction*; *Force* → *Linear/Angular Damping*.
- Determinism/cache: Dynamics Body tag → *Cache* → *Bake Object* (or Bake All). Project Settings scale for unit sanity.
- Soft bodies: raise Steps per Frame substantially; jitter almost always = too few steps or wrong scale.

### Houdini
- Bullet RBD: on the **RBD Bullet Solver** / *Bullet Solver* DOP → *Substeps*, *Collision Padding*; geometry collision shapes set to *Convex Hull*/*Box* on the RBD packed prims.
- Min substeps on the **DOP Network** root (Substeps) cascades to all solvers.
- Vellum: **Vellum Solver** → *Substeps*, *Constraint Iterations*, *Smoothing Iterations*; constraint stiffness on the Vellum Constraints SOP. Explosion on raised substeps → lower stiffness or ramp.
- Pyro: **Pyro Solver** substeps; collision via *Collision Source* / SDF — ensure SDF division size is fine enough and the collision volume is thick; leaking = coarse collision relative to substeps. Use **Sourcing** volumes at adequate resolution.
- Determinism: **File Cache** SOP (or DOP I/O) to write .bgeo.sc / .sim sequence; read it back. Avoid scrubbing during sim; simulate frame 1 → end.

## General cautions

- Change ONE variable per bake so the cause is attributable.
- High substeps + high iterations + concave mesh collision = very slow bakes; reach for proxies and continuous collision first.
- Adaptive substeps trade speed for non-determinism — turn off for final renders.
- When all else fails, simplify: collide a box against a box, confirm stability, then add complexity back.
