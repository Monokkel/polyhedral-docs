# Plugins

For developers ready to go past the concepts and work with a specific plugin's API. Each plugin section has an **overview** (what it's for and when to reach for it), an **API reference** (the full public surface, with hand-written examples), and — where a plugin's surface warrants it — **guides** (task-oriented how-tos).

New to the framework? Read the [Core Concepts](../concepts/index.md) first — the plugin sections assume that mental model and link back to it rather than re-explaining it.

The framework ships as fourteen plugins. They stack: each layer below is usable on its own, and each layer above assumes the ones under it.

## The foundation: Polyhedral Core

**[PolyhedralCore](polyhedralcore/index.md)** is one plugin hosting four independent systems as four separate modules. None of them depends on anything, and everything else in the framework is built on them. Each keeps its own section here:

- **[CommandSystem](commandsystem/index.md)** — the undo/redo/replay engine: every authoritative change goes through a command stack, so the whole game rewinds and replays reliably.
- **[QueueFramework](queueframework/index.md)** — ordered, channel-based processing of async work items: run gameplay steps one at a time, some instant and some spanning real time, with depth-first nesting and single-step debugging. The sequencing primitive the event and presentation layers build on.
- **[TagEvents](tagevents/index.md)** — the lightweight tag-keyed dispatch core: call a handler on an object by gameplay tag, with no delegate wiring or hard reference. The foundation the event system is built on, and usable on its own.
- **[NoiseBasedRandomSeed](noisebasedrandomseed/index.md)** — deterministic, save- and replay-safe random numbers keyed by a *domain* tag: each domain gets its own seed and stream, with sequential draws that advance the stream and stateless keyed draws that don't — so loot rolls and procedural generation reproduce exactly across a reload.

You enable one plugin, `PolyhedralCore`, and get all four. From C++ you depend on the modules individually — the [Polyhedral Core page](polyhedralcore/index.md) covers that distinction, which is the most common first-build stumble.

## Standalone foundations

Three more plugins sit close to the bottom of the stack and are adoptable on their own terms:

- **[TaggedData](taggeddata/index.md)** — the schema that maps each `Data.*` gameplay tag to the struct type stored under it, the primitive wrapper structs, and the typed-pin machinery that turns a mapped tag into a real struct pin in Blueprint. It describes and pins the shape of tagged data; entities are where that data is stored.
- **[Evaluators](evaluators/index.md)** — composable number and true/false calculations authored as data: damage formulas, requirements, and tuning knobs that live in data tables and assets instead of hard-coded branches.
- **[EventSystem](eventsystem/index.md)** — broadcast/subscribe gameplay events on channels, with deterministic ordering and reaction windows: the rules-layer mechanism where armor softens a hit, a passive triggers on a tag, and a whole cascade undoes as one step.

## The board

- **[GridGraph](gridgraph/index.md)** — build the board — square, hex, or freeform; flat or multi-storey — and answer "where is everything?" and "where can this unit go?" The grid is a static graph; per-unit movement and shape queries are computed on top, with terrain-conforming generation and a full grid-visualization layer.
- **[GridCommands](gridcommands/index.md)** — structural edits to the board that ride the command stack: add and remove nodes and edges, retag cells, change heights and movement costs, all as commands that undo, save, and replay. What makes runtime digging, destructible terrain, and an in-game level editor rewindable.

## Entities, presentation, and turns

- **[GameEntity](gameentity/index.md)** — the heart of the framework: the central game state where every unit, item, ability, and status effect lives as a value-struct entity, with template inheritance, a stat + modifier system, change events, and save/replay. Ties the foundations together.
- **[GridEntity](gridentity/index.md)** — where the grid and entity systems meet: an entity's position as command-routed placement (so moves undo, save, and replay), the derived occupancy index (who is on this cell), entity-facing movement grants and reachability, and a grid-aware token that snaps a unit to its cell.
- **[TokenSystem](tokensystem/index.md)** — the presentation layer: every entity gets an on-screen stand-in kept correct through undo and load with no custom code, plus cues and cue handlers for animated playout, capabilities to drive any token, and channels so one entity can drive many surfaces.
- **[TurnSystem](turnsystem/index.md)** — scheduling and flow over the entity system's turn machinery: pluggable scheduler policies (the shipped side-based preset, plus your own initiative/timeline rules), a derived projected turn order, and an optional flow driver — all riding turn state that undoes, saves, and replays because it lives on an entity.

## The capstone

- **[AbilitySystem](abilitysystem/index.md)** — the capstone rules engine: author a unit's actions — spells, attacks, item uses, passives, counters — as data-driven programs of steps that the framework resolves one step at a time, pausing for player choices and for entity-carried triggers (interrupts and reactions). The same steps run against a temporary, throwaway copy of the game state to power target previews, availability checks, and enemy AI.

## Testing your own game

- **[PolyhedralTestKit](polyhedraltestkit/index.md)** — a headless game instance and world in three lines, so your own automation tests can drive the real subsystems — create entities, submit commands, assert that Undo puts everything back — without a running game. C++ only.

## Supporting utilities

- **[TagDebug](tagdebug/index.md)** — a live, tag-driven debug console: author `Debug.*` gameplay tags for typed flags and one-shot actions, read them anywhere to branch on a toggle, and reflect changes into Blueprint event graphs, with optional per-tag persistence. Built on TagEvents.
- **[PolyhedralTooltips](polyhedraltooltips/index.md)** — tooltip and display-data conventions: inline rich-text keywords that pop a tooltip on hover, or imperative show/hide from widget code, with content authored in a data table, per-entry data assets, or live objects, and rendered by your own tooltip widget.

Every plugin in the current framework release has a section above. If you're deciding what to install, the [installation page](../getting-started/installation.md) lists the same set with its dependency requirements.
