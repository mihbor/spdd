---
name: arcadia-system-analysis
description: Use when authoring or changing the Arcadia System Analysis (SA) layer in sysml/arcadia/system-analysis — system boundary, system functions, system capabilities, functional chains, exchange items, and OA-to-SA allocation. Not for logical components, technology, or code.
license: ISC
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
---

# Arcadia System Analysis (SA)

SA turns the operational view into a black-box definition of what the PlanetCiv system must do for its operational actors. It introduces the system boundary but still avoids logical components, technology, deployment, and code.

This is a modeling track, not an SPDD phase. It does not renumber or replace the lifecycle in `AGENTS.md`.

## Inputs

- OA layer: `sysml/arcadia/operational/` — read the capabilities and activities before deriving functions.
- Business scope: `requirements/`, `docs/game-description.md`, `docs/Glossary.md`, `requirements/decisions.md`.
- Existing model: `sysml/arcadia/system-analysis/` if present. If absent, create only the topic files the scope needs; do not add empty placeholder packages.

## Layer boundary

**In scope**

- System boundary and external actors.
- System functions, system capabilities, functional chains, and scenarios.
- Black-box exchange items and interactions.
- Allocation of operational activities to system functions.

**Out of scope — defer to LA/PA**

- Logical components, component responsibilities, ports/interfaces between internal components.
- Language, framework, library, storage engine, file format, rendering technology, deployment.
- `src/` modules, APIs, performance tuning, asset pipelines.

**Boundary rule.** OA actors realized by the software (`ComputerPlayer`, `BarbarianFaction`) are not external actors in SA: their behavior becomes system functions. The human player and the operating environment (browser, OS, file system) are the external actors. The simulated planet, cities, and AI are inside the system boundary.

## SysML v2 mapping

| Arcadia element | SysML v2 | Validated pattern |
|---|---|---|
| System boundary / external actor | `part def` | `part def SystemUnderStudy;` / `part def HumanPlayer;` |
| System function | `action def` | `action def GenerateWorld { ... }` |
| System capability | `part def Capability_*` owning functions | `action generateWorld : GenerateWorld;` |
| Capability owner | `dependency` | `dependency from SystemUnderStudy to Capability_GenerateWorld;` |
| Functional chain | `action def` + `first ... then ...` | `first generateWorld then startTurnLoop;` |
| Exchange item | `item def` | `item def PlayerCommand;` |
| Black-box interaction | `connection def` + usage | `end sender : HumanPlayer;` then `end sender ::> player;` |
| OA activity → SA function | allocation | `allocation def ExplorationSupport :> OperationalActivitySupport { allocate explorePlanet to supportExploration; }` |
| Requirement trace | doc citation, or `satisfy` only by a conforming part | see `story-to-sysml` |

**Allocation syntax.** `allocate` needs feature usages, not definitions. Direct `allocate DefA to DefB` passes spec42 but fails the binding gate. Define package-level usages (`action generateWorld : GenerateWorld;`) or nested usages and allocate those. Do not name an element `allocate`; `allocateResearch` is safe.

**Function decomposition and allocation.** Mark every system function as **leaf** or **aggregate** before allocating it downstream. A leaf function has no child action usages; an aggregate function exists only to decompose a capability or group and is documentary. Only leaf functions own ports/exchanges and only leaf functions are component-allocation candidates. An OA activity may be realized by an aggregate SA function, but at LA the aggregate's leaf children carry the allocation; never allocate the aggregate itself to a component. See `arcadia-logical-architecture` for the LA-side rule.

**Modes, states, and data.** Read `references/modes-states-and-data.md` before adding behavior or exchange items. Use the chosen-vs-undergone test: modes are commanded functional configurations, states are undergone conditions. Keep them in separate machines, never nested. Every mode lists its available leaf functions. Exchange items are `item def`s with typed attributes/enums; use `ref` for item-typed payloads and mark genuinely opaque payloads with a justification. Reuse SA item types downstream.

**Do not use `satisfy` with a part-typed requirement subject and an action witness**: `validate-sysml` warns "Bound features should have conforming types". Use a conforming part witness, or record the trace in docs and keep the formal link in the requirements/system seam.

## Files and names

- Directory: `sysml/arcadia/system-analysis/`.
- Recommended topics: `SystemContext.sysml`, `SystemFunctions.sysml`, `SystemModesAndStates.sysml`, `SystemCapabilities.sysml`, `FunctionalChains.sysml`, `SystemInterfaces.sysml`, `FunctionalExchanges.sysml`, `OperationalAllocation.sysml`.
- One package per file. Package names use `PlanetCivArcadiaSystem<Topic>` and must be unique.
- `private import ScalarValues::*;` plus explicit single-name imports. No cross-package wildcards.
- `doc /* ... */` takes no trailing semicolon. Every definition carries a `Source:` line.
- Avoid reserved declaration names listed in `story-to-sysml` (`allocate`, `decide`, `first`, `then`, `state`, `transition`, `view`, `case`, `join`, `end`, ...).

## Procedure

1. Run the baseline gates. If the model is red, report the root cause and stop.
2. Read the OA layer and scope. Write the system boundary and external actors; state what is inside and outside.
3. Derive system functions from OA activities and capabilities. Mark each function as leaf or aggregate; decompose only as far as the level needs. Include system-only functions (world generation, presentation, persistence, input, performance) only when they support an OA activity/capability or a documented quality need.
4. Group functions into system capabilities and trace each to an OA capability.
5. Define functional chains for the critical operations: turn processing, new game, save/load, world generation, AI opposition, game end.
6. Define modes and states for the dynamic behavior: turn phases, pause/resume, loading, game over. Keep modes and states in separate machines and list the leaf functions available in each.
7. Define exchange items and black-box interactions with typed attributes/enums. Do not define internal component interfaces yet.
8. Allocate every OA activity to at least one SA function in `OperationalAllocation.sysml`. If an in-scope OA activity is deferred, get user consent and record it in `spdd/analysis/deferred.md`; never drop it silently.
9. Update `tools/validate-sysml.ps1` `$Order` with OA before SA, and SA dependencies first.
10. Run both gates and report the counts.

## Quality checks

- Every OA capability is supported by at least one SA capability/function.
- Every OA activity is allocated to an SA function or explicitly deferred with consent.
- Every SA function is marked leaf or aggregate; aggregate functions are not component-allocation candidates.
- Every mode/state machine is classified (`Mode:` or `State:`), non-nested, and documents available leaf functions.
- Exchange items are typed; structured payloads are not bare strings; opaque payloads are justified.
- Every SA function is observable/testable and traces to an OA activity/capability or a documented quality need.
- No logical components, internal ports, frameworks, file formats, code paths, or deployment.
- AI and barbarians are system functions, not external actors; the human player remains external.
- No requirement duplication; `sysml/requirements/` stays the requirements model.
- Both gates report 0 errors and 0 warnings.

## Validation

```powershell
spec42 check sysml --workspace-root sysml --format json
powershell -File tools/validate-sysml.ps1
```

A green check proves syntax and reference resolution, not method quality. Review the black-box boundary by hand.

## References

- Upstream: `sysml/arcadia/operational/`.
- Behavior and data: `references/modes-states-and-data.md`.
- Sources: `requirements/`, `docs/game-description.md`, `docs/Glossary.md`, `requirements/decisions.md`.
- Companion skills: `arcadia-operational-analysis`, `spec42-check`, `sysmlv2-skill`, `story-to-sysml`.
