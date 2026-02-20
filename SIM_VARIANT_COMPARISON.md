# Black Hole Simulator Variant Comparison

This document compares:

- `BlackHole_simulation_do_not_change.html`
- `BlackHole_simulation_gemini.html`
- `BlackHole_simulation_codex.html`

## 1) High-Level Architecture

| Area | do_not_change | gemini | codex |
|---|---|---|---|
| Scene layout | Two-scene split (`sceneBack` + `sceneFront`) | Raymarch quad + one particle scene | Room scene + raymarch quad + back/front particle scenes |
| Post pipeline | Back render -> lensing ShaderPass -> front render -> bloom | Raymarch render -> particles render -> bloom | Room render -> raymarch render -> particles back -> lensing ShaderPass -> particles front -> bloom |
| Main design style | Post-lensing centric | In-shader lensing centric | Hybrid (raymarch + post-lensing + split particles) |

## 2) Lensing Model Differences

| Variant | How lensing is done | Strength behavior | Notes |
|---|---|---|---|
| do_not_change | Separate `LensingShader` post pass warps full frame | `strength` directly scales post warp | Clean and simple, but can over-blacken if center sampling is unstable |
| gemini | No lens post pass; raymarch integration bends rays (`uLensStrength` in acceleration) | Strength changes ray integration in disk/background shading | Physically coherent feel for disk, but no separate screen-space warp layer |
| codex | Both raymarch bend + post lens pass | Strength drives raymarch and lens post path | Most flexible; requires careful guards to avoid black expansion artifacts |

## 3) Particle System Differences

| Variant | Pass model | Occlusion model | Result profile |
|---|---|---|---|
| do_not_change | Front/back split via `uIsFront` and depth mask | Depth split only | Strong wrap layering, weaker hard-core occlusion |
| gemini | Single particle pass | Analytic black-hole shadow ray test | Stable center occlusion, less wrap control |
| codex | Front/back split + extra occlusion checks | Depth split + analytic shadow + screen-space lens shadow | Best control, but easiest to overtune |

## 4) Room / Background Differences

| Variant | Room type | Lighting style |
|---|---|---|
| do_not_change | Real mesh room (walls/floor/pillars) | Scene light sources + lens warp on back scene |
| gemini | Procedural room coloring in raymarch shader | Emissive/procedural shading only |
| codex | Real mesh room + raymarch contributions | Strongest lighting flexibility, more moving parts |

## 5) UI and Defaults

## Controls present per file

| Control | do_not_change | gemini | codex |
|---|---|---|---|
| Time Scale | Yes | Yes | Yes |
| Lensing Strength | Yes | Yes | Yes |
| Core Brightness | No (`Disk Lumens` instead) | Yes | Yes |
| Gas Brightness | No | Yes | Yes |
| Show Particles | No | Yes | Yes |
| Black Hole Mass | No (radius via expansion only) | Yes | Yes |
| Auto Camera Orbit | No | Yes | Yes |
| Simulate Growth | Yes | Yes | Yes |

## Default alignment findings

- `do_not_change`: some shader defaults can differ from GUI defaults until first frame update.
- `gemini`: previously had `uSpinSpeed` and particle brightness default mismatches (first-frame mismatch risk).
- `codex`: currently most aligned between UI defaults and initial uniforms.

## 6) Known Risks by Variant

- `do_not_change`: post-lens center handling can create hard black growth if `dist` handling is not guarded.
- `gemini`: single-pass particles are simpler but less controllable for front/back wrap composition.
- `codex`: hybrid stack is strongest but has the highest tuning complexity; black-growth artifacts appear if lens radius semantics are inconsistent.

## 7) Recommended “Best of All Worlds” Merge Direction

1. Keep `codex` as base (it already has the broadest feature set and UI control surface).
2. Keep raymarch gravity bend logic from `gemini` style as the primary physical lens response.
3. Keep split particle layering from `do_not_change`/`codex` for believable over/under wrap.
4. Keep real room geometry from `do_not_change`/`codex`.
5. In post-lensing, separate radii semantics:
   - `coreRadius` for physical bend scale
   - `shadowRadius` for black-core protection and occlusion mask
6. Enforce non-darkening post-lens blending so lensing warps light but does not invent extra black area.
7. Lock UI defaults and uniform initializers to identical values in all variants to avoid first-frame visual mismatches.

## 8) Suggested Next Integration Pass

If we proceed from here, the next step should be:

1. Create one target file (likely `BlackHole_simulation_codex.html`).
2. Port only the lensing pieces we prefer from `gemini` into codex (not full file copy).
3. Validate with 3 camera scenarios:
   - edge-on disk view
   - near top-down view
   - low core brightness view
4. Tune only three knobs in order:
   - `shadowRadius` multiplier
   - post-lens deflection floor
   - particle lens-mask radius
