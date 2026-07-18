# GameEntity

For developers building the game-state layer of a turn-based game — the units,
items, abilities, and status effects, and the numbers and rules that move them.
After this section you'll know how to create entities, change their stats and
data, and get undo, save, and replay for the whole board without wiring any of it
yourself.

GameEntity is the heart of the framework. Every piece of game state — a unit, a
card, an ability, a status effect, a zone on the board — is one uniform
value-struct **entity**, and they all live together in a single GameInstance
subsystem: **the game state**. The plugin ties the foundation layers together —
entities store part of themselves as
[tagged data](../taggeddata/index.md), every change to them is a
[command](../commandsystem/index.md), and their stat formulas are
[evaluators](../evaluators/index.md).

!!! note "Read the concepts first"
    This page is the plugin's overview; the guides and reference below assume you
    know the model. If you don't yet, start with
    [Entities Are Data](../../concepts/entities-as-data.md) and
    [Stats and Modifiers](../../concepts/stats-and-modifiers.md) in Core Concepts
    — they own the "why" and "when". This section owns the "how": the exact calls
    and the tasks you'll actually perform.

## Where it fits in the framework

GameEntity sits on top of the foundation plugins and gives them a home. An entity
is a plain **value struct** — no Actor, no per-entity class, no pointers into the
world — stored by value inside the game-state subsystem. Because the whole board
is copyable data with no hidden references, the framework can hash it, save it,
and rewind it reliably.

Two properties fall out of that and shape everything you do here:

- **All authoritative mutation is command-driven.** You never write entity state
  directly; you call a mutator and it routes through the
  [command stack](../commandsystem/index.md). That is what buys undo, redo, and
  replay for the entire game — for free, on every built-in call.
- **Current stats are derived.** The resolved value of a stat is recomputed
  automatically from its base and modifiers whenever an input changes. Derived
  state is never saved and never command-captured; it is rebuilt from the
  authoritative data. See [Derived State & Events](../../concepts/derived-state.md).

## What you get

- **Data-driven content.** Author entities as **data-table rows** with parent-row
  inheritance — a "fire goblin" row points at a "goblin" row instead of
  subclassing. Register a row as a **template** and a hundred live entities share
  one authored definition, each storing only its differences.
- **Undo, redo, and replay for free.** Every create, destroy, tag, stat, modifier,
  data, and hierarchy change is a command, so all of it undoes and redoes as one,
  and a whole session can be recorded and played back.
- **A stat and modifier system.** A whole-number **base stat** folded through a
  stack of **modifiers** (flat, percent, or any evaluator formula) into an exact
  **current stat** — familiar if you've used Unreal's Gameplay Ability System, but
  plain data that saves and replays.
- **A reactive change-event surface.** Every mutation broadcasts a **change
  event**, during play *and* during undo, redo, and load — so UI, visuals, and
  caches stay in sync by reacting instead of polling.
- **Save and replay.** Serialize the whole game state to a save, or record the
  command timeline and replay it step by step.

## When to reach for GameEntity

Reach for [TaggedData](../taggeddata/index.md) on its own when you only need
convenient typed storage on an actor or object — a bit of presentation state, some
editor-time configuration — with no undo and no save.

Reach for GameEntity when you want the whole stack: authoritative game state that
undoes and saves, stats with modifiers, a hierarchy of units and items, and a
change-event feed to drive your presentation. Anything your **rules** depend on
belongs on an entity, not on an actor.

## The pieces at a glance

| Piece | What it is | Reference |
|---|---|---|
| **Entities & templates** | The game-state subsystem, entity references, creation from rows/templates/registry, hierarchy and slots, queries, and entity-owned tagged data. | [Entities reference](reference-entities.md) |
| **Stats & modifiers** | Base vs. current stats, flat/percent/formula modifiers, granted modifiers, stat batches, and breakdowns. | [Stats reference](reference-stats.md) |
| **Change events** | The change-event delegates, the change record payload, and the mutation-to-event matrix reactive code subscribes to. | [Events reference](reference-events.md) |
| **Commands & replay** | The built-in command-driven mutators, and recording and playing back the command timeline. | [Commands & replay reference](reference-commands.md) |

## Where to go next

- **[Guides](guides.md)** — task recipes: creating entities, authoring template
  rows, changing stats with a working Undo, adding modifiers, equipping items,
  stat batches, tooltips, reacting to change events, and recording a replay.
- **[Entities reference](reference-entities.md)** — entities, references,
  templates, data-table rows, and the game-state subsystem's lifecycle, query,
  and hierarchy API.
- **[Stats reference](reference-stats.md)** — the full stat and modifier surface.
- **[Events reference](reference-events.md)** — every change-event delegate and
  what each mutation fires.
- **[Commands & replay reference](reference-commands.md)** — the built-in command
  inventory and the replay recorder.
- **[First board tutorial](../../getting-started/first-board.md)** — creates
  entities, changes a stat, wires a working Undo, and reacts to a change event,
  hands-on.
