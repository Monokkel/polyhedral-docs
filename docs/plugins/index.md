# Plugins

For developers ready to go past the concepts and work with a specific plugin's API. Each plugin section has an **overview** (what it's for and when to reach for it), **guides** (task-oriented how-tos), and an **API reference** (the full public surface, with hand-written examples).

New to the framework? Read the [Core Concepts](../concepts/index.md) first — the plugin sections assume that mental model and link back to it rather than re-explaining it.

## Available now

- **[TaggedData](taggeddata/index.md)** — attach typed structs to any object or entity, keyed by gameplay tags, with typed Blueprint pins driven by a schema. The one plugin you can adopt entirely on its own.
- **[CommandSystem](commandsystem/index.md)** — the undo/redo/replay engine: every authoritative change goes through a command stack, so the whole game rewinds and replays reliably.
- **[Evaluators](evaluators/index.md)** — composable number and true/false calculations authored as data: damage formulas, requirements, and tuning knobs that live in data tables and assets instead of hard-coded branches.
- **[GameEntity](gameentity/index.md)** — the heart of the framework: the central game state where every unit, item, ability, and status effect lives as a value-struct entity, with template inheritance, a stat + modifier system, change events, and save/replay. Ties the other three plugins together.
- **[EventSystem](eventsystem/index.md)** — broadcast/subscribe gameplay events on channels, with deterministic ordering and reaction windows: the rules-layer mechanism where armor softens a hit, a passive triggers on a tag, and a whole cascade undoes as one step.
- **[TagEvents](tagevents/index.md)** — the lightweight tag-keyed dispatch core: call a handler on an object by gameplay tag, with no delegate wiring or hard reference. The foundation the event system is built on, and usable on its own.
- **[QueueFramework](queueframework/index.md)** — ordered, channel-based processing of async work items: run gameplay steps one at a time, some instant and some spanning real time, with depth-first nesting and single-step debugging. The sequencing primitive the event and presentation layers build on.
- **[GridGraph](gridgraph/index.md)** — build the board — square, hex, or freeform; flat or multi-storey — and answer "where is everything?" and "where can this unit go?" The grid is a static graph; per-unit movement and shape queries are computed on top, with terrain-conforming generation and a full grid-visualization layer.
- **[TokenSystem](tokensystem/index.md)** — the presentation layer: every entity gets an on-screen stand-in kept correct through undo and load with no custom code, plus cues and cue handlers for animated playout, capabilities to drive any token, and channels so one entity can drive many surfaces.

More plugin sections are being added as the site grows.
