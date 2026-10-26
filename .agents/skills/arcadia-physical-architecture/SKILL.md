---
name: arcadia-physical-architecture
description: Use when authoring or changing the Arcadia Physical Architecture (PA) layer in sysml/arcadia/physical — physical components, deployment, physical interfaces, LA-to-PA allocation, and implementation mapping to src/. Ground every element in actual code or a planned story.
license: ISC
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
---

# Arcadia Physical Architecture (PA)

PA is the implementation-facing architecture: the real runtime and deployment parts, their interfaces, and how logical components are realized in the repository. Every PA element must be grounded in actual code/config or explicitly marked as planned.

This is a modeling track, not an SPDD phase. It does not renumber or replace the lifecycle in `AGENTS.md`. PA models the implementation; it does not implement it.

## Inputs

- LA layer: `sysml/arcadia/logical/` — required; read components and allocations.
- Codebase: `src/`, `package.json`, `vite.config.ts`, `tsconfig.json`.
- Code mirror: `sysml/system/GameFoundation.sysml`, `sysml/system/PresentationContract.sysml`.
- Requirements for planned or absent components; `docs/`.
- Existing model: `sysml/arcadia/physical/` if present.

## Layer boundary

**In scope**

- Physical components and deployment nodes.
- Runtime topology, processes, and physical interfaces/protocols.
- Implementation artifacts and their mapping to `src/` or configuration.
- Allocation of every logical component to at least one **behavioral** physical component; node components are resource hosts, never function-allocation targets.

**Out of scope**

- Changing game rules or requirements to fit a refactor.
- Inventing modules, files, or services that do not exist.
- Editing application code (implementation goes through SPDD `/spdd-generate`).
- Secrets, absolute local paths, or environment-specific values in shared models.

## SysML v2 mapping

| Arcadia element | SysML v2 | Validated pattern |
|---|---|---|
| Physical component / node | `part def` with a `Source:` path | `part def RendererSubsystem;` |
| Deployment | `part def` context with parts and connections | `part client : BrowserClient;` |
| Physical interface | `port def` + `connection def` | `connection def FrameLink { end source : FrameSource; end sink : FrameSink; }` |
| LA component → PA component | allocation def with usages | `allocation def RendererRealization :> ComponentAllocation { allocate worldRenderer to rendererSubsystem; }` |
| Implementation artifact | `doc` citation to `src/` or config | `Source: src/rendering/...` |
| Code mirror | `sysml/system/` tracks implementation | keep it consistent |

**Grounding rules**

- Read the actual code before writing a PA element. No invented modules.
- Each physical component's `doc` names its real implementation path (`src/...`) or says `Planned` with the story ID and date.
- Allocate every LA component to a physical component. No orphan logical components.
- If PA and `sysml/system/` disagree, code is authoritative: update `sysml/system/` per the `story-to-sysml` system seam, then PA.
- `src/` remains the source of truth for implementation; PA tracks and explains it, never constrains it.
- Use the validated port/connection pattern; do not use `interface def` with part-typed ends or `connect` between mismatched port directions.
- `allocate` needs feature usages, not definitions; direct `allocate DefA to DefB` fails the binding gate.

**Behavioral vs node components (hard rule)**

- Every PA component is classified **behavior** or **node** (resource/implementation/tooling). Record the classification in the component doc as `Nature: behavior` or `Nature: node`; the file header may provide a default for a homogeneous file, but every node must be explicit.
- Behavioral components carry functions and are the only valid targets of LA-component realization. Node components provide resources, hosts, storage, or tooling and are never targets of a behavioral LA component.
- Deployment is a hosting relation, not containment. Do not model a behavioral component as a child of a node and do not mix behavior and node in one composition hierarchy.
- A technical PA component with no LA counterpart (for example a development toolchain) is allowed only when it is explicitly classified node/tooling and is grounded in `Deployment.sysml` or `ImplementationMapping.sysml`.
- When one LA component is realized by several PA components, add a one-line rationale to the allocation definition and treat the split as a review finding until justified.
- Physical links/paths route component exchanges; they do not replace them.

## Files and names

- Directory: `sysml/arcadia/physical/`.
- Recommended topics: `PhysicalComponents.sysml`, `Deployment.sysml`, `PhysicalInterfaces.sysml`, `LogicalComponentAllocation.sysml`, `ImplementationMapping.sysml`.
- One package per file. Package names use `PlanetCivArcadiaPhysical<Topic>` and must be unique.
- `private import ScalarValues::*;` plus explicit single-name imports. No cross-package wildcards.
- `doc /* ... */` takes no trailing semicolon. Every definition carries a `Source:` line with the code path or planned story.
- Avoid reserved declaration names listed in `story-to-sysml`.

## Procedure

1. Run the baseline gates. If the model is red, report the root cause and stop.
2. Read the LA layer, the code tree, `package.json`, `vite.config.ts`, and `sysml/system/`.
3. Inventory real runtime/deployment parts and their responsibilities; map each to code/config or mark it planned.
4. Define physical interfaces and connections with the validated port/connection pattern.
5. Allocate every LA component to behavioral PA components in `LogicalComponentAllocation.sysml`. Node components are never targets.
6. Record implementation mapping in `doc` blocks (component → paths), and keep `sysml/system/` consistent with `src/`.
7. Update `tools/validate-sysml.ps1` `$Order` with OA → SA → LA before PA, and PA dependencies first.
8. Run both gates and report the counts.

## Quality checks

- Every PA element is grounded: real path/config or a planned story.
- Every PA component is classified `Nature: behavior` or `Nature: node`; every LA component is allocated to at least one behavioral PA component.
- Every PA component is grounded and participates in at least one allocation or deployment connection; technical-only components are explicitly node/tooling.
- `sysml/system/` and PA agree with `src/`.
- No game-rule changes, invented modules, secrets, or absolute local paths.
- Both gates report 0 errors and 0 warnings.

## Validation

```powershell
spec42 check sysml --workspace-root sysml --format json
powershell -File tools/validate-sysml.ps1
```

A green check proves syntax and reference resolution, not that PA matches the code. Review the mapping against `src/` by hand.

## References

- Upstream: `sysml/arcadia/logical/`.
- Behavior and data: `../arcadia-system-analysis/references/modes-states-and-data.md`.
- Code mirror: `sysml/system/`.
- Sources: `src/`, `package.json`, `vite.config.ts`, `tsconfig.json`.
- Companion skills: `arcadia-logical-architecture`, `spec42-check`, `sysmlv2-skill`, `story-to-sysml`.
