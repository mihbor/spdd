---
name: spec42-check
description: Validate SysML v2 textual models with the Elan8 Spec42 CLI (doctor, check, explain-diagnostic, model-export). Use when editing or reviewing .sysml/.kerml files under sysml/, or when a model diagnostic code needs a stable fix strategy. Same engine as the Spec42 VS Code extension; CLI JSON output is the contract for automation.
license: ISC
allowed-tools: Bash Read Grep Glob
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
  upstream: https://github.com/elan8/spec42
  upstream-version-tested: "0.57.0"
---

# Spec42 Model Validation

Validate `.sysml` / `.kerml` files with the Spec42 CLI. There is no MCP
server or HTTP API — every surface calls the same CLI.

## Binary

`spec42` on `PATH` (`%LOCALAPPDATA%\spec42\bin`, v0.57.0). Refresh from
`elan8/spec42` releases (`win32-x64` archive) when upgrading.

```powershell
spec42 --version
```

## Workflow (in order)

1. **Doctor** — stdlib, config dirs, library paths. Run first when
   `unresolved_*` or `missing_library_*` diagnostics appear.

    ```powershell
    spec42 doctor
    spec42 doctor --format json
    ```

2. **Check** — validation report. Always pass `--workspace-root` when
   validating a single file inside a multi-file project.

    ```powershell
    spec42 check sysml/requirements/Story-001-001.sysml
    spec42 check sysml --workspace-root sysml --format json
    ```

   Gate on `summary.error_count`. Use per-file `diagnostics[].code`,
   not message text, to decide fixes.

3. **Explain** — stable fix strategy for a diagnostic code.

    ```powershell
    spec42 explain-diagnostic --code unresolved_type_reference --format json
    spec42 explain-diagnostic --code <code> --path <file> --line <n> --format json
    ```

4. **Model export** — read-only resolved projection (relationship
   families, resolved expressions, connector topology). Branch on
   `projection.schema_version`; `--max-nodes` bounds output and
   `projection.truncation` reports what was cut.

    ```powershell
    spec42 model-export sysml/requirements/Story-001-001.sysml --max-nodes 500 --format json
    ```

Global flags: `--library-path`, `--config`, `--stdlib-path`,
`--no-stdlib`.

## Rules

- Validate after every substantive model edit; do not commit models
  with `error_count > 0`.
- `doctor` before rewriting imports when many diagnostics are
  `unresolved_type_reference`, `unresolved_import_target`, or
  `missing_library_context`.
- Do not "fix" parse errors by deleting valid spec-aligned structure
  without re-running `check`.
- Do not commit secrets or absolute local paths into shared models.
- Where Spec42 and the OMG Pilot Implementation disagree, the Pilot
  is the binding semantic gate (see `sysmlv2-skill`).
- Name attributes so `owningPart.attribute` is self-describing
  (`scene.hasDirectionalLight`, not `scene.directionalPresent`).
  Never transcribe AC verbs into bare booleans (`isRendered`,
  `ambientPresent`); name the observable fact (what, where, unit —
  e.g. `sceneVisibleInCanvas`, `measuredFramesPerSecond`).
- Value bindings inside a usage need an explicit `redefines` clause
  (`attribute generatedTileCount redefines Grid::generatedTileCount
  = 5120;`); bare `attribute x = v;` fails with
  `implicit_redefinition_without_operator`.
- Traceability uses a requirement *usage* plus `by`, not a typed def:
  `requirement frustumCulling : AC1_...;` then `satisfy frustumCulling
  by mainPipeline;`. The `satisfy name : ReqDef { subject ... }` shape
  does not resolve (`unresolved_reference`). Upstream reference:
  `elan8/sysml-examples` webshop.
- Model shape: one package per story file (`PlanetCivEpic01Grid`,
  `...Culling`, ...), never the same package name in two files —
  spec42 does not merge them. Cross-story links go in the dependent
  story's package via explicit `private import OtherPackage::*;`
  (e.g. LOD and Culling both satisfy grid's pinned `smallGrid`).
- `attribute def X;` without `:> Base` is NOT flagged (negative-tested,
  zero diagnostics) — grep manually. Same for `explain-diagnostic` gaps:
  some error codes return `catalog: null` (e.g.
  `missing_import_visibility`).
- spec42 0.57 has no `verification case` / `analysis case`
  (negative-tested: `recovered_package_body_element`; `use case` alone
  validates). Do not model test links until tool support lands — record
  enforcement in doc notes instead. Re-test each spec42 upgrade.
- Binding gate: OMG Pilot via DeciSym `sysmlv2-validator`
  (`%LOCALAPPDATA%\sysmlv2-validator\validate-sysml.cmd`, Maven 3.9.16
  in `%LOCALAPPDATA%\maven`). It resolves imports incrementally in
  argument order — pass files dependency-first (010 before 004, 012
  before 006), never rely on directory scan order. A namespace error
  that spec42 does not report is usually a forward reference: reorder,
  don't rewrite.

## References

- Upstream agent guide: https://github.com/elan8/spec42/blob/main/docs/user/AI-ASSISTANTS.md
- Repo conventions pattern: https://github.com/elan8/spec42/blob/main/.github/copilot-instructions.md
- Companion skill: `sysmlv2-skill` (syntax, patterns, error catalog)
