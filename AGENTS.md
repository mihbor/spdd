# SPDD Agent Workflow

This file is the authoritative guide for the workflow shared by the SPDD skills in `.agents/skills/`. Individual `SKILL.md` files define command-specific inputs, outputs, procedures, and guardrails; they must not redefine the lifecycle or its phase numbering.

## Canonical lifecycle

The story step is optional. When used, it is Phase 0 and decomposes a high-level requirement into independently processable stories. Each story then follows the numbered pipeline.

| Stage | Command | Input | Output | Normal handoff |
| --- | --- | --- | --- | --- |
| Phase 0 (optional) | `/spdd-story` | High-level business requirement | `requirements/Epic-*.md` and/or `requirements/Story-*.md` | Process each selected story with `/spdd-analysis` |
| Phase 1 | `/spdd-analysis` | Business requirement or story | `spdd/analysis/*-Analysis-*.md` | `/spdd-reasons-canvas` |
| Phase 2 | `/spdd-reasons-canvas` | Business context or Phase 1 analysis | `spdd/prompt/*.md` containing the REASONS Canvas | `/spdd-generate` after user confirmation |
| Phase 3 | `/spdd-generate` | Structured prompt | Implementation code and verification results | Review the implementation |
| Prompt-first iteration | `/spdd-prompt-update` | Existing prompt and a design or requirement change | Updated prompt | Regenerate affected code with `/spdd-generate` |
| Code-first synchronization | `/spdd-sync` | Existing prompt and implementation changes | Prompt synchronized with actual code | Review the synchronized design and regenerate only when needed |

```text
High-level requirement
  └─ optional /spdd-story → requirements/Epic-*.md or Story-*.md
        ↓
  /spdd-analysis → /spdd-reasons-canvas → /spdd-generate → code
                                                            │
  prompt/design change → /spdd-prompt-update ───────────────┘
  code change → /spdd-sync → prompt (then /spdd-generate when needed)
```

`/spdd-sync` is a reverse synchronization path, not a mandatory fourth implementation phase. Use it when implementation changes before the prompt; use `/spdd-prompt-update` when the prompt or requirements change before implementation.

## Artifact ownership and handoffs

- `requirements/` is the business-level source for epics, stories, scope, and acceptance criteria.
- `spdd/analysis/` contains strategic, codebase-grounded context. `spdd/analysis/deferred.md` is reserved for in-scope items explicitly deferred with user consent.
- `spdd/prompt/` contains the implementation contract in the seven-section REASONS Canvas order: Requirements, Entities, Approach, Structure, Operations, Norms, and Safeguards.
- The implementation code is the executable result. Code-first changes must be reflected back into the relevant prompt through `/spdd-sync`.
- Handoffs must preserve traceability: retain the source requirement, carry its identifier into downstream artifact names when available, and link each generated artifact to the preceding artifact.
- `spdd/done.md` is the implementation trace. `/spdd-generate` opens one section per prompt at invocation (start time) and closes it only after verification (completion time, elapsed time, intended commit message, verification notes); the Commit field is recorded before any commit is created, never backfilled afterwards. Follow-up tweaks requested on work that is not yet signed off / committed do NOT get a new section: update the latest entry instead (keep the earliest Impl start, extend Completed to the latest verification, recompute Elapsed as the diff, amend Prompt/Notes, and set Commit to the actual message once known).

## Cross-skill rules

1. Preserve supplied business requirements verbatim in artifacts that carry them; do not silently invent or discard scope.
2. Keep the thinking level appropriate to the phase: business decomposition in `/spdd-story`, strategic analysis in `/spdd-analysis`, detailed design in `/spdd-reasons-canvas`, and implementation in `/spdd-generate`.
3. Treat the prompt as the implementation contract. When generated code is wrong or incomplete, update the relevant prompt section first and then regenerate the affected code.
4. Keep prompt sections internally consistent. Changes to entities, structure, operations, norms, or safeguards must be checked against related sections.
5. When code changes first, synchronize the actual implementation rather than the originally planned implementation. Do not alter business requirements merely to document a refactoring.
6. Detect and follow the project’s actual stack and architecture. Do not introduce layers, abstractions, or technologies that the requirement and codebase do not justify.
7. Do not create commits unless the user explicitly requests them. When requested, commit related prompt and code changes together.
8. Keep tool output small and execution observable in every phase: never dump raw logs — read slices only (e.g. tail the last N lines), truncate long lines, and strip ANSI escape sequences and carriage returns from captured output; prefer short polls with small outputs over long-wait calls when observing builds, servers, or browser runs; print the local time immediately before launching and immediately after return of long-running steps so durations stay on record.

For command-specific behavior, always follow the applicable skill file in addition to this shared workflow.