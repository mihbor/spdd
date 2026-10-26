---
name: arcadia-operational-analysis
description: Use when authoring or changing the Arcadia Operational Analysis (OA) layer in sysml/arcadia/operational — operational actors, entities, activities, capabilities, processes, interaction scenarios, and context. Not for System Analysis, Logical Architecture, or Physical Architecture.
license: ISC
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
---

# Arcadia Operational Analysis (OA)

Model the game domain as it operates independently of the PlanetCiv software. OA answers who acts, what they do, what abilities they need, and how they interact. It never names the software, renderer, storage, framework, or code module.

This is a modeling track, not an SPDD phase. It does not renumber or replace the lifecycle in `AGENTS.md`; architecture findings feed SPDD analysis and prompts, and implementation still goes through `/spdd-generate`.

The Arcadia ladder is OA → SA → LA → PA: each layer refines the previous one and must not leak downstream concerns upward.

If a request says "Arcadia" without naming a layer, start at OA when no `sysml/arcadia/operational/` model exists or when the domain is being re-scoped. Otherwise ask whether to update OA, SA, LA, PA, or review the chain; do not skip OA while the operational scope is missing.

## Inputs

- Business scope: `requirements/Epic-*.md`, `requirements/Story-*.md`, `docs/game-description.md`, `docs/Glossary.md`, `requirements/decisions.md`.
- Existing model: `sysml/arcadia/operational/` — use it as the exemplar and extend it; do not rewrite unrelated packages.
- If the user names a narrower scope (one epic or story), honor it and record the boundary in the package header.

## Layer boundary

**In scope**

- Operational actors and operational entities.
- Operational activities, capabilities, processes, and interaction scenarios.
- Operational context and the interactions between actors/entities.

**Out of scope — defer to SA/LA/PA**

- Map-generation algorithms, rendering, camera, LOD, culling, assets, XR, performance.
- Save formats and serialization, storage, input handling, APIs, modules, frameworks, TypeScript/Three.js.
- Requirement definitions and acceptance criteria; those stay in `requirements/` and `sysml/requirements/`.

Rule of thumb: if the subject is "the software does X", it is not OA. If the subject is "a player or civilization does X", it is OA.

## SysML v2 mapping

| Arcadia element | SysML v2 | Validated pattern |
|---|---|---|
| Operational actor / entity | `part def` | `part def Player;` |
| Composition | nested `part` | `part cities : City[0..*];` |
| Activity / process | `action def` | `action def FoundCity { ... }` |
| Sequencing | `first ... then ...` | `first assess then issueOrders;` |
| Capability | `part def Capability_*` owning its realizing activity | `action exploreAndExpand : ExploreAndExpand;` |
| Capability owner | `dependency` | `dependency from Civilization to Capability_ExploreAndExpand;` |
| Interaction scenario | `use case def` | `subject civilization : Civilization; actor firstParty : Player;` |
| Context interaction | `connection def` + connection usage | `end commander : Player;` then `end commander ::> humanPlayer;` |

Do not use `satisfy` with a part-typed requirement subject and an action witness: `validate-sysml` warns "Bound features should have conforming types". Capabilities are structural here. Requirement satisfaction belongs to the requirements/system seam and follows the `story-to-sysml` skill.

## Files and names

- Directory: `sysml/arcadia/operational/`.
- Current topics, extend rather than duplicate: `OperationalActors.sysml`, `OperationalEntities.sysml`, `OperationalActivities.sysml`, `OperationalCapabilities.sysml`, `OperationalScenarios.sysml`, `OperationalContext.sysml`.
- One package per file. Package names use `PlanetCivArcadiaOperational<Topic>` and must be unique across the workspace.
- `private import ScalarValues::*;` plus explicit single-name cross-package imports. No cross-package wildcards.
- `doc /* ... */` takes no trailing semicolon. Every definition carries a `Source:` line naming the Epic, Story, decision, or reference file it derives from.
- Avoid reserved declaration names: `allocate`, `decide`, `first`, `then`, `state`, `transition`, `view`, `case`, `join`, `actor`, `subject`, `include`, `perform`, `end`. Rename to `allocateResearch`, `endTurn`, etc.

## Procedure

1. Run the baseline gates. If the model is red, report the root cause and stop.
2. Read the scope sources. Inventory actors and entities; decide what is external to the civilization enterprise and what belongs to it.
3. Inventory activities in business language, grouped by capability area. Give every activity a `Source:`.
4. Group activities into capabilities: define `part def Capability_*` owning the aggregate activity, then add `dependency from <owner> to <capability>`.
5. Add processes with `first/then` for the sequences that matter (turn round, settlement wave, war campaign, victory drive).
6. Add interaction scenarios as `use case def` with `subject` and `actor`.
7. Add the operational context part with typed connection usages; omit the software system by design.
8. Update `tools/validate-sysml.ps1` `$Order` dependency-first: actors/entities, activities, capabilities/scenarios, context.
9. Run both gates, fix root causes, and report the counts.

## Traceability (hard matrix)

| Upstream | Downstream | Required |
|---|---|---|
| OA capability | SA capability/function | at least one |
| OA activity | SA function allocation | every activity allocated or explicitly deferred with user consent |
| OA activity | noun in an OA capability | every leaf activity has a home |
| OA capability | SA functional chain/scenario | at least one downstream chain/scenario covers the capability's activities |

- Realize, do not copy: downstream layers allocate/realize OA elements; they do not restate them under new names. A deliberate new element needs a source and a rationale.
- An OA element with no downstream link goes to `spdd/analysis/deferred.md` only with user consent; never drop it silently.

## Quality checks

- Every actor/entity participates in at least one activity or context interaction.
- Every capability owns at least one activity; every leaf activity has a home.
- Every process step resolves to a defined activity; sequences are plausible, not decorative.
- No software nouns, file formats, APIs, rendering, or performance concerns.
- No duplicate semantic elements under different names; no orphan definitions.
- No invented game rules: CtP2 first, then `requirements/decisions.md`; mark deliberate deviations and cite them.
- Both gates report 0 errors and 0 warnings.

## Validation

```powershell
spec42 check sysml --workspace-root sysml --format json
powershell -File tools/validate-sysml.ps1
```

A green check proves syntax and reference resolution, not method quality. Review the layer boundary by hand.

## References

- Existing OA exemplar: `sysml/arcadia/operational/`.
- Sources: `docs/game-description.md`, `docs/Glossary.md`, `requirements/decisions.md`.
- Companion skills: `spec42-check`, `sysmlv2-skill`, `story-to-sysml`.
