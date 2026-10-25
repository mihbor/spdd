---
name: arcadia-logical-architecture
description: Use when authoring or changing the Arcadia Logical Architecture (LA) layer in sysml/arcadia/logical — logical components, ports, interfaces, exchange items, logical functions, and SA-to-LA allocation. Technology-agnostic; not for deployment, code, or file formats.
license: ISC
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
---

# Arcadia Logical Architecture (LA)

LA defines how the system is structured logically: components with responsibilities, ports and interfaces, exchange items, logical functions, and the allocation of SA functions. It remains technology-agnostic.

This is a modeling track, not an SPDD phase. It does not renumber or replace the lifecycle in `AGENTS.md`.

## Inputs

- SA layer: `sysml/arcadia/system-analysis/` — required; read the functions, capabilities, and functional chains.
- OA layer for intent: `sysml/arcadia/operational/`.
- Business scope: `requirements/`, `docs/`.
- Existing model: `sysml/arcadia/logical/` if present. If absent, create only the topic files the scope needs.

## Layer boundary

**In scope**

- Logical components and their responsibilities.
- Logical functions and their allocation to components.
- Ports, interfaces, exchange items, and component connections.
- Logical scenarios for critical chains.
- Allocation of every SA function to at least one logical component.

**Out of scope — defer to PA**

- Language, framework, library, storage engine, file format, rendering technology.
- Deployment nodes, processes, threads, protocols, and runtime topology.
- `src/` modules, file paths, APIs, performance tuning.

If a candidate component is named after a code file or a library, it belongs in PA.

## SysML v2 mapping

| Arcadia element | SysML v2 | Validated pattern |
|---|---|---|
| Logical component | `part def` | `part def TurnEngine;` |
| Responsibility | `doc` + owned functions | `action resolveTurn : ResolveTurn;` |
| Logical function | `action def` | `action def ResolveTurn;` |
| Port | `port def` with `in`/`out` features | `port def CommandSink { in command : PlayerCommand; }` |
| Interface | `connection def` with two typed ends | `connection def CommandLink { end source : CommandSource; end sink : CommandSink; }` |
| Exchange item | `item def` | `item def PlayerCommand;` |
| Connection usage | typed `connection` with both ends bound | `connection link : CommandLink { end source ::> ui.port; end sink ::> engine.port; }` |
| SA function → LA component | allocation def with usages | `allocation def TurnAllocation :> FunctionAllocation { allocate resolveTurn to turnEngine; }` |
| Logical scenario | `action def` + `first/then` or `use case def` | `first receiveCommand then resolveTurn;` |

**Port and interface notes**

- Use `connection def` for component-to-component links and `port def` for interaction points.
- Do not use `interface def` with part-typed ends: the binding validator interprets `end` as a port and fails. Use `connection def`.
- Match port directions. Separate provider/consumer port defs are the validated pattern; connecting a port to another with incompatible `in`/`out` features warns `flow_direction_incompatible`.
- `allocate` needs feature usages, not definitions. Direct `allocate DefA to DefB` passes spec42 but fails the binding gate. Define usages and allocate those.
- Do not use `satisfy` with a part-typed requirement subject and an action witness; it triggers a conforming-types warning.

## Files and names

- Directory: `sysml/arcadia/logical/`.
- Recommended topics: `LogicalComponents.sysml`, `LogicalFunctions.sysml`, `LogicalInterfaces.sysml`, `LogicalScenarios.sysml`, `SystemFunctionAllocation.sysml`.
- One package per file. Package names use `PlanetCivArcadiaLogical<Topic>` and must be unique.
- `private import ScalarValues::*;` plus explicit single-name imports. No cross-package wildcards.
- `doc /* ... */` takes no trailing semicolon. Every definition carries a `Source:` line naming the SA function/capability and the requirement source.
- Avoid reserved declaration names listed in `story-to-sysml` (`allocate`, `first`, `then`, `state`, `transition`, `view`, `case`, `join`, `end`, ...).

## Procedure

1. Run the baseline gates. If the model is red, report the root cause and stop.
2. Read the SA functions and OA intent. Derive candidate components from cohesion, coupling, change rate, and lifecycle boundaries; avoid one component per function.
3. Write component responsibilities and the logical functions each component owns. Name components after their responsibility, not after code files.
4. Define exchange items and ports. Connect components with typed connections; document the direction and purpose of each interface.
5. Allocate every SA function to at least one component in `SystemFunctionAllocation.sysml`. No orphan functions or components.
6. Add logical scenarios for critical chains: turn processing, new game, save/load, world generation, AI opposition, game end.
7. Update `tools/validate-sysml.ps1` `$Order` with OA/SA before LA, and LA dependencies first.
8. Run both gates and report the counts.

## Quality checks

- Every SA function is allocated to at least one logical component.
- Every component has at least one responsibility; every port/interface has a purpose.
- Exchange items are defined as `item def`, not stringly-typed placeholders.
- No technology, framework, deployment, file format, or `src/` path appears in the layer.
- Component boundaries are justified; the layer does not mirror code modules (that is PA).
- Both gates report 0 errors and 0 warnings.

## Validation

```powershell
spec42 check sysml --workspace-root sysml --format json
powershell -File tools/validate-sysml.ps1
```

A green check proves syntax and reference resolution, not method quality. Review the technology-agnostic boundary by hand.

## References

- Upstream: `sysml/arcadia/system-analysis/`, `sysml/arcadia/operational/`.
- Sources: `requirements/`, `docs/`.
- Companion skills: `arcadia-system-analysis`, `arcadia-physical-architecture`, `spec42-check`, `sysmlv2-skill`, `story-to-sysml`.
