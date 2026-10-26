---
name: arcadia-layer-review
description: Use when reviewing an Arcadia layer or the OA→SA→LA→PA chain in sysml/arcadia — boundary leaks, traceability gaps, orphan elements, naming/package consistency, and green spec42/validate-sysml gates.
license: ISC
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
---

# Arcadia Layer Review

Review an Arcadia model for method fidelity, traceability, and model hygiene. This skill reports findings; it does not rewrite the model or requirements unless the user explicitly asks.

## Scope

- One layer (`operational`, `system-analysis`, `logical`, `physical`) or the whole chain.
- Before committing a layer change, after a layer is added, or when the model is "green but wrong".

Behavior/data classification details: `../arcadia-system-analysis/references/modes-states-and-data.md`.

## Gates first

```powershell
spec42 check sysml --workspace-root sysml --format json
powershell -File tools/validate-sysml.ps1
python tools/validate-arcadia-allocation.py
python tools/validate-arcadia-traceability.py
```

If either spec42 or validate-sysml is red, stop: report the first root cause and its `file:line`. Do not draw findings from a broken model. The allocation audit is a method gate; a red audit is a finding even when the syntax gates are green.

## Layer boundary checklist

| Layer | Must contain | Must not contain |
|---|---|---|
| Operational (OA) | actors, entities, activities, capabilities, processes, interaction scenarios, context | software, rendering, storage, file formats, modules, frameworks, performance |
| System Analysis (SA) | system boundary, external actors, system functions/capabilities, functional chains, exchange items, OA allocation | logical components, internal component ports, technology, deployment |
| Logical (LA) | logical components, responsibilities, logical functions, ports/interfaces, exchange items, SA allocation | technology, frameworks, deployment, file formats, `src/` paths |
| Physical (PA) | physical components, deployment, physical interfaces, LA allocation, implementation mapping | invented modules, changed game rules, secrets, absolute paths |

**Method-specific check:** OA actors realized by the software (`ComputerPlayer`, `BarbarianFaction`) must become SA system functions, not be reintroduced as external actors. The human player remains external.

## Traceability matrix

| Upstream | Downstream | Required |
|---|---|---|
| OA capability | SA capability/function | at least one |
| OA activity | SA function allocation | every activity allocated or explicitly deferred with user consent |
| SA capability | functional chain or scenario | every leaf function owned by the capability appears in at least one chain/scenario |
| SA leaf function | LA component allocation | exactly one behavioral component |
| SA aggregate function | no component allocation | decomposition only; children carry behavior |
| SA functional exchange | LA component exchange | every cross-component FE realized by exactly one CE |
| LA component | PA component allocation | every component realized by a behavioral PA component, never a node |
| PA component | `src/` / config or planned story | every component grounded; technical-only components explicit |

Also check:

- Every definition has a `Source:` line naming the upstream Epic/Story/decision/code path.
- Names are stable across layers; no renamed duplicate of the same concept.
- Each layer uses the correct package prefix (`PlanetCivArcadiaOperational*`, `...System*`, `...Logical*`, `...Physical*`); names are unique workspace-wide.
- Every new file is in `tools/validate-sysml.ps1` `$Order` dependency-first.
- Layer directories match `sysml/arcadia/<layer>/` and the topic files in the layer skill.

## Realization, not copy-paste

- A downstream element that elaborates an upstream element carries an allocation/realization link and a `Source:` line; it is not a renamed copy.
- A deliberate structural addition (LA routing function, PA runtime component) is allowed only with a rationale and a source.
- An upstream element with no downstream link is a finding unless it is in `spdd/analysis/deferred.md` with user consent.
- The traceability report (`docs/arcadia-traceability.md`) is the evidence; a green syntax gate does not substitute for it.

## Allocation legality

- Only leaf functions allocate. An aggregate function allocated to a component is a finding: remove the allocation, keep its leaf children.
- Each leaf function has exactly one behavioral owner. Two owners is an error, not a trade-off.
- Every LA-specific leaf function (routing/aggregation/orchestration) is allocated exactly once.
- Every component owns at least one function unless it is a documented aggregate component.
- Every cross-component functional exchange has exactly one component exchange; intra-component exchanges need none.
- Every PA component has `Nature: behavior` or `Nature: node`; a behavioral LA component never allocates to a node.
- A PA component with no LA counterpart is either a documented technical/node component or a finding.
- A PA allocation with several targets has a written rationale; otherwise it is a review finding.

When the SA layer uses the functional-exchange catalog fallback, verify each LA connection usage cites exactly one FE and each FE is cited once. A connection with neither an FE nor a documented deferral is a finding.

## Model hygiene

- No orphan capabilities, functions, or components that should participate in a chain.
- No duplicate semantic elements under different names.
- No aggregate function allocation; no leaf function with multiple owners.
- No function allocated to a PA node component; every PA component has a nature.
- No functional exchange without a component exchange or a documented deferral.
- No nested modes/states or mixed mode/state machine; every mode/state is classified and lists its available functions.
- No stringly-typed structured exchange payload; opaque payloads are justified.
- No process history, review notes, or implementation logs in the model.
- No tautological constraints or decorative `first/then` chains.
- No reserved declaration names (`allocate`, `decide`, `first`, `then`, `state`, `transition`, `view`, `case`, `join`, `end`, ...).
- No `satisfy` with a part-typed requirement subject and an action witness.
- No `interface def` with part-typed ends; use `connection def`.
- No `allocate DefA to DefB`; allocation targets must be feature usages.

## Useful inventory commands

```powershell
Get-ChildItem sysml\arcadia -Recurse -Filter *.sysml | Select-Object FullName
rg -n "^package |part def Capability_|action def |use case def |allocation def |connection def " sysml/arcadia
```

Keep output small: read slices and summarize counts, do not dump whole models.

## Report format

| Severity | Finding | Evidence | Fix |
|----------|---------|----------|-----|
| Blocker / Major / Minor | [what is wrong and why it matters] | `file:line` | [smallest correct fix] |

End with:

```
Verdict: PASS | PASS WITH FINDINGS | BLOCKED
Gates: spec42 0 errors / 0 warnings; validate-sysml N files, 0 / 0
```

## Guardrails

- Review only: do not edit the model, requirements, or code unless the user asks.
- Do not "fix" a red gate by deleting valid structure; identify the root cause.
- Do not add review findings as model content; route requirement gaps through the SPDD/story amendment path.
- Do not treat a green gate as a method-quality verdict; boundary and traceability checks are manual.
- Keep tool output and the final report concise.
