# Modes, States, and the Data Model

This is the project-owned Arcadia reference for dynamic behavior and typed exchange data. It is a
short paraphrase of the upstream methodology, not a copy of the upstream skill.

## Mode vs state

Use the **chosen-vs-undergone** test:

- **Mode** = a chosen or commanded functional configuration. The expected function set changes;
  transitions are decisions or functional events. Examples: turn phase, paused, maintenance.
- **State** = a condition the system undergoes. External conditions or physical changes drive it.
  Examples: loading, game over, degraded performance.

Rules:

- Keep modes in a mode machine and states in a state machine. Do not nest modes inside states or
  states inside modes.
- One current mode per machine; multiple machines may run in parallel (for example turn phase and
  pause).
- Every mode must list the leaf functions available in it. A mode with no function list is
  incomplete.
- A state lists the functions available while it holds; a terminal state may list only review,
  save, or presentation functions.
- Record the classification in the definition doc as `Mode:` or `State:`.

## Current PlanetCiv mapping

- Turn phases (`humanTurn`, `aiTurn`, `barbarianTurn`, `turnTransition`) are **modes**: each has a
  different available function set.
- `running` and `paused` are **modes** in a parallel pause machine. The requirement's phrase
  "pause state" describes the behavior; the Arcadia classification is a mode because the player
  commands it.
- `loading`, `activeGame`, and `gameOver` are **states** in the lifecycle machine.
- The requirements model `Story-002-002` encodes turn phases as `state def` in
  `TurnPhaseMachine`. That is a generic state-machine encoding; the Arcadia model classifies them as
  modes and records the 1:1 mapping. Do not rename or duplicate the requirements machine without a
  story amendment.

## Data and exchange items

- Exchange items are `item def`s; ports are typed by them. Do not put structured payloads in bare
  `String` attributes.
- Closed sets become `enum def`s (map size, difficulty, speed, world type, barbarian activity,
  turn phase, command kind).
- Use `attribute` for scalar/enumerated data. Use `ref` for item-typed payloads and collections.
- A payload that is genuinely opaque (for example a serialized save body or a raw browser event)
  must say so in its doc and name the requirement that justifies the opacity.
- Interfaces group exchange items through typed ports and connections; do not hand-specify payload
  fields in the interface itself.
- Reuse an SA item type in LA/PA rather than redefining the same concept. Add only layer-specific
  fields that the layer actually needs.

## Review checks

- No nested mode/state machines; no mixed mode/state machine.
- Every mode has an availability list; every state has a defined exit or terminal condition.
- Every port references an item definition; every interface groups at least one exchange item.
- No stringly-typed command, configuration, save, or victory payload.
- Opaque payloads have a named justification.
- The Arcadia classification and the requirements-model encoding are mapped explicitly.
