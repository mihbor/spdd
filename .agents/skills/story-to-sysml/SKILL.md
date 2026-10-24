---
name: story-to-sysml
description: Convert INVEST user stories in requirements/ into SysML v2 textual requirements in sysml/, one package per story. Use when supplementing a story with requirement defs, pinned evidence usages, satisfy links, or state/connection structure. Assumes the spec42 CLI on PATH for validation (see spec42-check).
license: ISC
allowed-tools: Bash Read Grep Glob
metadata:
  version: "1.0.0"
  author: PlanetCiv contributors
---

# Story → SysML Conversion

## Setup

- One file per story: `sysml/requirements/Story-<module>-<seq>.sysml`, one package per
  file (`PlanetCivEpic<NN><Topic>`). Never repeat a package name across
  files — spec42 does not merge them.
- `private import ScalarValues::*;` always; add stdlib imports
  (`SequenceFunctions`, `ControlFunctions`, `IntegerFunctions`) as needed.

## Rules

- Doc blocks carry Given/When/Then verbatim (SPDD rule 1: preserve business
  requirements). Anything else is labeled derivation-not-story-text.
- Suspicious ease is a story defect. If attributes map 1:1 from identifiers
  in the story, the story is code-shaped — fix the story, do not encode it.
- Name attributes so `owningPart.attribute` is self-describing; never
  transcribe AC verbs into bare booleans. Avoid reserved keywords as
  names entirely (`state`, `transition`, `view`, `case` fail with
  `reserved_declaration_name` — use `game`, `turnTransition`, `sight`,
  `situation`); the error lists the keyword, not the fix. Cross-file
  requirement references must match the def name exactly — the validator
  reports the use site, so check the defining file first.
- Every approximated constraint gets a doc note: what was lost in
  formalization and where enforcement really lives (test, future `verify`).
- A constraint earns its place only if you can imagine it failing.
  Tautological `== true` flags become doc-only requirement defs: every AC
  keeps a model home for the satisfy graph, but only falsifiable bounds,
  counts, pins, machines, and links carry constraints. Prune attributes
  orphaned by dropped constraints. Both subject-without-constraint and
  empty part defs validate clean.
- Story defects get fixed by corroboration count, not hunch: grep the
  corpus for the number, weigh witnesses, prefer the side citing a source
  file (CtP2 reference, decisions.md), fix toward it, and record the
  provenance in both the story and the model doc. Precedent: the +3/+30
  Scientist output (5 witnesses vs 1).
- A satisfy witness may be imported: the link lives in the dependent's
  package, but the usage it binds can come from another file's package
  (e.g. radius visibility satisfied by a growth-package usage).
- Integer math without Real types: percents as `x*250/100`, turn counts
  as `(box-stored)/surplus` (exact when divisible), accumulation as
  `after == before + rate`. Re-derive per file is waste; copy the shape.
- Numbers: story numbers verbatim; invented or story-defined values marked
  as such; derivations shown step by step in plain language, no jargon.
- Story examples pin as spot-checks, never as the whole table: mark which
  examples are pinned and name the authoritative source (goods.txt,
  terrain.txt, BtS table) for exhaustive conformance, which belongs in
  the test that reads that source. Precedent: `Story-003-003` AC2/AC7,
  `Story-003-004` AC5.
- Check expressibility before assuming it: the stdlib has no `pow`, so
  exponential formulas stay as pinned rows rather than calcs.

## Patterns

- Numeric AC tables become pinned part usages with explicit `redefines`
  bindings, plus one constraint over them (`Story-001-002` AC2).
- Collection properties (e.g. uniqueness) become pairwise `forAll` with
  indexed access over a part collection (`Story-001-002` AC3).
- Topological relations become a `connection def` for the relation plus
  count attributes for the universal invariants (`Story-001-002` AC4).
- Lifecycles and staged processes become `state def` states with `item`
  triggers in a state machine (`Story-001-004` LOD, `Story-001-005`
  stages).
- Trace links use a requirement usage plus `by`: `requirement x : Req;`
  then `satisfy x by element;`. Cross-story links live in the dependent
  story's package via explicit `private import OtherPackage::*;`.

## Consumption

- Impact analysis: `powershell -File tools/sysml-dependents.ps1 -Story
  <MMM-SSS>` lists every satisfy link touching a story (requirement,
  satisfying element, owning files). Built on `model-export`
  `satisfySource`/`satisfyTarget` peers — the graph's first reader.

## System seam

- `sysml/system/` mirrors the implementation (package per area, sourced
  with date; tracks code, never constrains it). Story packages import
  system packages, never the reverse. Order system files first for
  `validate-sysml` (it resolves imports incrementally).
- A satisfy whose witness is system-typed against a scaffolding-typed
  subject is the seam made visible: spec42 stays clean,
  `validate-sysml` warns `Bound features should have conforming types`. Expected — do not
  "fix" by retyping; the warning marks where requirement meets code.

## Standing caveats

- After any story edit, grep `sysml/` for the story ID: doc quotes go
  stale and pinned values can drift from amended numbers. Re-validate.
- Validation is not a quality gate: a green check can still encode a bad
  requirement. Requirement quality (business language, no prescribed
  identifiers) is reviewed by hand.
- `spec42 check` proves a constraint well-formed, not that any data
  satisfies it. Enforcement lives in implementation tests, linked back via
  `verify` once test elements exist.
