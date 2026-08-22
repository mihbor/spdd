# SPDD Agent Skills

This repository contains Agent Skill definitions for Structured Prompt-Driven Development (SPDD). The skills were derived from the command definitions in [gszhangwei/open-spdd](https://github.com/gszhangwei/open-spdd) and adapted for use as project-scoped skills.

## Included skills

| Skill | Purpose |
| --- | --- |
| `spdd-story` | Decompose requirements into INVEST-compliant epics and stories. |
| `spdd-analysis` | Analyze requirements in the context of the codebase and identify concepts, strategy, risks, and gaps. |
| `spdd-reasons-canvas` | Turn business context into an implementation-ready REASONS Canvas prompt. |
| `spdd-generate` | Generate and verify code from a structured SPDD prompt. |
| `spdd-prompt-update` | Update an existing prompt while preserving its structure and traceability. |
| `spdd-sync` | Synchronize implementation changes back into the prompt. |

## Adaptations from OpenSPDD

The original SPDD workflow and REASONS Canvas methodology are retained. The skill definitions were adapted at a high level in the following ways:

- Converted the upstream command-style Markdown files into Agent Skill bundles under `.agents/skills/<skill>/SKILL.md`. The optional `agents/openai.yaml` files provide Codex-specific UI metadata.
- Generalized implementation guidance beyond Java/Spring conventions. Skills now detect and follow the project’s actual stack, architecture boundaries, framework metadata, dependency wiring, and error-handling approach where applicable.
- Refined requirements artifacts: epics and child stories use distinct identifiers and files, analysis artifacts use story or ticket identifiers with a fallback, and deferred in-scope work requires explicit user consent and a pending-items ledger.
- Strengthened workflow safeguards: generated prompts must remain complete and internally consistent, implementation work is checked with applicable build and lint commands, and prompt/code commits are created only when explicitly requested.
- Clarified platform-neutral specification rules, including when to use diagrams, how to preserve existing prompt structure, and how to maintain traceability between requirements, design, and implementation.

## Agent compatibility

The `SKILL.md` files use the shared Agent Skills layout and can be discovered from `.agents/skills` by Codex, OpenCode, and Junie in their supported project contexts. Invocation syntax varies by agent: Codex supports `$spdd-analysis`, OpenCode uses its native skill mechanism, and Junie supports `/spdd-analysis` commands and `$spdd-analysis` references.

The `agents/openai.yaml` files are optional Codex-specific metadata and are ignored by agents that do not use them. The instruction body is portable in principle, but individual tool names and workflow commands may require adjustment for the host agent.

The detailed behavior and guardrails are defined in the individual `SKILL.md` files.

## Attribution and license

This work is an adaptation of [OpenSPDD](https://github.com/gszhangwei/open-spdd), an MIT-licensed project by gszhangwei. This repository is also distributed under the MIT License; see [LICENSE](LICENSE).
