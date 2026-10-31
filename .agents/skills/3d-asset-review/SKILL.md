---
name: 3d-asset-review
description: Review loop for procedural 3D game assets — compile gate, multi-angle capture, geometric self-test, baseline compare — before user preview.
---

# 3D Asset Review

Use this skill when a new or changed procedural 3D asset is ready for review, before showing it to the user. It runs the full Epic-22 review loop: compile gate, multi-angle capture, geometric self-test, and baseline comparison. It was built from the Settler/Worker feedback rounds, where agent-only front-view inspection missed real geometry bugs the user then caught.

## TRIGGER

- A unit, resource, improvement, or building 3D builder is new or changed
- The user asks for asset feedback or a visual check
- Epic-22 implementation work producing visible geometry

## DO NOT TRIGGER

- 2D sprite or icon work with no 3D geometry involved
- Gameplay logic with no visual change
- Final user sign-off itself — this skill prepares the review; the user gives feedback

## Procedure

### 0. Compile gate

`npx tsc --noEmit` must pass before any capture. Fix type errors first.

### 1. Serve (reuse, short polls)

- Probe the preview page over HTTP first. If nothing listens, launch the vite dev server detached and poll briefly.
- Never start a second server when one is already up.
- Follow AGENTS.md rule 8 throughout: short polls with small outputs, never raw log dumps, timestamps around long-running steps.

### 2. Capture three angles

Drive the preview harness through browser-harness CDP against a debug-enabled browser. The harness supports `?static=1` (no auto-rotate), `?unit=<TYPE_ID>` (registry selection), `?focus=left&dist=0.3` (close-up framing), and `?az=<radians>` (camera orbit: `0` front, `1.6` side, `3.1` back). Capture front, side, and back close-ups via `Page.captureScreenshot` to temp files and Read each image. For tool-bearing units add one tight-front (`dist=0.18`) and judge the weapon class-read there (pattern 14). Front-only inspection is how the pickaxe-through-head and back-kit bugs escaped — all three angles are mandatory.

### 3. Geometric self-test

Load `?selftest=1&unit=<TYPE_ID>`. The page title reports `SELFTEST PASS/FAIL (n checks): <TYPE_ID>` with failures listed in the `#selftest` element and console. The test (`src/rendering/unit-models/selftest.ts`) enforces three rules over builder-tagged meshes (`head`, `headgear`, `body`, `limb`, `accessory`, `tool`, `trim` in `userData.part`):

1. **Attachment** — every headgear, accessory, tool, and limb volume must transitively touch body-anchored volume (trim and head exempt; head anchors). Catches floating straps, belts, and detached limbs.
2. **Head clearance** — no tool volume may intersect the head volume. Catches diagonal tools routed through the head.
3. **Headgear seating** — every headgear volume must touch the head volume. Catches hovering hats and hoods.

New builders must tag every mesh or the test silently skips it — an untagged mesh is a review failure. Investigate every FAIL before presenting; a PASS is required but not sufficient (taper direction and silhouette quality stay human checks).

Fast path: the same verdict runs in seconds without a browser — compile the builders plus a throwaway runner with `tsc` into a temp dir and execute it with `node` (project `node_modules` resolves `three`; delete runner and output afterwards). Use it to iterate geometry cheaply; the in-browser `?selftest=1` title check then only needs to confirm registry wiring, and image budget stays reserved for the visual passes.

### 4. Baseline compare

Capture the same framing with the current 2D sprite beside the model at true game scale. The 3D asset must match-or-beat recognizability; the sprite is the art-direction anchor, not a fallback to beat down.

### 5. Present only when green

`tsc` clean plus self-test PASS plus three inspected angles. The summary states what was checked (angles, self-test result, baseline). Never present an asset the agent has not itself inspected from three angles.

## Session budget

Long review sessions die with provider `400 invalid parameters` once image history piles up (observed: 305 messages, 70 reads, 40+ PNGs). Deleting temp files does not help — context keeps the bytes. Only compaction or a fresh session drops them; fork does not (it copies image history — proven: a fork at the last good turn 400'd again).

- One session per unit, not per Epic. After sign-off, close it and start fresh; carry forward via the pattern library and prompt Operations, never via image history.
- Review fan-out via subagents (verified 2026-10-14, Fighter trial): when a batch queues several visual reviews, run each unit's capture loop in a Task subagent instead of the main session. Subagents start with a fresh context window — captures and image reads live and die inside the subagent, and only its text verdict returns, so main-session image budget stays flat. Give each subagent this skill's procedure plus the unit's builder path, proposed design, and sprite baseline, and demand a text-only return (compile result, self-test verdict + check count, per-angle observations, baseline verdict, APPROVE or concrete fix list — never images). One unit per subagent, launched sequentially: they all drive the same local Chrome tab, so parallel reviewers fight over navigation. Record each returned approval in prompt Operations immediately, same session.
- Batch flow that works: implement the whole family in code, self-test all of them, then visually inspect only the lead unit for sign-off while the rest queue behind it — never present an asset that has not had its own three angles, but code-plus-selftest may run ahead of the visual queue.
- Intermediate fix iterations capture the single relevant angle only. Full front/side/back capture runs only for the final present-gate.
- Baseline compare runs once per unit, not every round.
- Before each capture batch, check session size with `Invoke-RestMethod` (never `curl.exe`). Past ~150 messages or ~400k input tokens, compact first or start a fresh session — auto-compaction does not catch image-budget `400 invalid parameters`:
  ```powershell
  $s = Invoke-RestMethod -Uri "http://127.0.0.1:4096/session/$sessionID" -Method Get
  $m = Invoke-RestMethod -Uri "http://127.0.0.1:4096/session/$sessionID/message" -Method Get
  # if ($s.tokens.input -gt 400000 -or @($m).Count -gt 150) → compact:
  Invoke-RestMethod -Uri "http://127.0.0.1:4096/session/$sessionID/compact" -Method Post -Body '{}' -ContentType "application/json" | Out-Null
  ```
  Compaction keeps old messages durable but replaces active context with a summary plus recent tail; image attachments become textual descriptors, which is exactly the budget being freed. If the session already holds an empty failed turn, compact keeps the poison — start fresh instead (see Guardrails).

## Pattern library (from Settler/Worker/Mounted-Archer/Cossack feedback)

1. Every attachment embedded in body mass — straps, rings, and belts sized to hug surfaces, never floating (Settler belt and bedroll straps).
2. Limbs need shoulder mass — never meet a bare cone apex; use a yoke (Settler arms).
3. Tapers wide at the root — shoulders wider than wrists (Settler arms shipped inverted once).
4. Tools outside the head radius from every angle — vertical side carry beats diagonal routing (Worker pickaxe went through the head).
5. Headgear small and high — never engulf the head; face visible (Settler hood).
6. Trim stays dedicated parts (base ring plus one pad) — never full-body tint; verify against near-black civ colors.
7. One builder module per unit; shared palette and parts in `unit-model-types.ts`; id-to-builder registry plus roster validation in `unit-models/index.ts`.
8. Thin planar features need depth in every projection — a bow (or blade) lying in one plane collapses to a single stick line edge-on; sweep a curve and cant it so the silhouette reads from front, side, and back (Mounted Archer bow).
9. Thin-part joints must overlap, never meet at points — extend string/chord ends into the parts they join so no notch gap shows from any angle (Mounted Archer string tips).
10. Baseline sprite anchors recognizability, not correctness — when the 2D sprite is functionally wrong (sideways bow), deviate deliberately, match its colors and silhouette language, and record why (Mounted Archer forward-facing bow).
11. Seat with overlap margin, never exact boundary touches — Box3 equality fails at float precision (0.89 vs 0.8899999994); bury every seated part at least 0.005 into what it joins (German spike base, Samurai crescent, Pikeman comb).
12. Pose grips clear of torso mass — a hand centered on a vertical pole buries itself in the yoke; re-pose (diagonal trail/shoulder carry) instead of nudging numbers (Pikeman).
13. Signature proportions beat uniform budgets — a pike must read longest even past the height cap; keep the exception thin and document it (Pikeman tip y=1.21 vs dory 1.0).
14. Weapons must read their class at game scale — blades repeatedly author short and read as knives (Cossack shashka 0.22 long, 6:1 aspect). Sabre-class blades need length >= 0.3 rider units with slenderness >= 10:1 (width <= 0.03); polearms keep tip-height ordering (pike longest). Judge on a tight-front capture (`dist=0.18`), never on self-test (it checks clearance/seating, not length) — extend outboard/back within the height budget rather than upward.
15. Design from the unit's role, not its costume — a scout reads through motion and observation (stride pose, telescope sighted with both hands, pointing), not through outfit tweaks. Stacking genre-trope combos misfires regardless of colors (robe plus hood plus staff always reads wizard). When the 2D sprite is itself generic, derive the read from what the unit does.
16. Instruments must connect to the body at both ends — a telescope whose eyepiece floats by the cheek reads as a club being swung. Seat the eyepiece at the eye (kept head-clear per the self-test) and give optics an optical silhouette: narrow at the eye flaring to a wider objective housing, the opposite of a club.
17. Bands need an accompanying volume — a bare torus on the head reads as exactly that. Seat bands on a cap, dome, or hat so the pair reads as headwear; the same holds for any ring-shaped trim or gear.
18. Same-color functional parts merge — thruster bells in armor gray vanish against the pack. Give sub-parts material contrast (near-black housings) plus silhouette separation (splayed outboard, protruding below) so each part reads independently; presence alone is not a read.
19. Orient emissives toward the viewer — downward-facing exhaust glows are edge-on and invisible from gameplay cameras. Tip thrusters, vents, and lenses rearward/upward toward likely camera angles and size them to survive game zoom.
20. Floaters must clear the trim ring — the ground ring reads as a plate anything sitting inside it rests on. Levitating units (hover tanks) need an unmistakable air gap bridged only by their glow wash, never hull-to-ring contact.
21. Parachutes need converging cues — a box pack always reads as a backpack. Canopy (rounded, light canvas, rising behind the shoulders) plus harness risers over the shoulders plus a chest reserve together read parachute; no single cue does it alone.
22. Lifting surfaces must taper — a constant-chord box panel reads as a slab, never a wing (Interceptor deltas shipped slabby, worse than the Fighter's boxes). Author wings as extruded plan shapes with root chord visibly wider than tip chord plus aft sweep (`addTaperedWing`: wide root, narrow tips, swept — deltas get big root chord with strong sweepBack); pass full-span wings straight through the fuselage mass so no mirror gap shows. Judge the taper on side/back close-ups, never front-only (slabs hide head-on).

## Guardrails

- Never silently reinterpret feedback — record agreed unit designs in the STORY-022-004 prompt Operations before rebuilding.
- Record every approval immediately, in the same session that grants it — write the `agreed design (art feedback YYYY-MM-DD)` line plus the `spdd/done.md` trace entry before capturing the next unit. Approvals kept only in image history or uncommitted edits are lost on a `400` session fail and force re-asks.
- On session start (especially a fresh session after a `400`), read the STORY-022-004 prompt Operations first and treat every `agreed design` entry as settled — never re-present or re-ask those units. Carry verdicts forward as text; never re-read old captures.
- Keep the preview harness generic (`?unit=` selection); never fork per-unit pages.
- Keep this skill the single source for review procedure; prompts reference it, they do not duplicate it.
- On provider `400 invalid parameters`, never send `continue` blindly — the failed turn leaves an empty assistant message that poisons history and every retry fails again. Start a fresh session carrying forward only text (pattern library, prompt Operations, prior verdicts as text — never re-read old captures) and read only the missing angle. Fork/revert cannot rescue an oversize image payload, only a poisoned-but-small one.
