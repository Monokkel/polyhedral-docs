# Getting Started

For a developer trying the Polyhedral Game Framework for the first time. This
section takes you from an empty project to a working slice of game state — data
entities you can spawn, mutate, undo, and react to.

## What this framework is

A set of modular Unreal Engine 5.8 plugins for turn-based, board-game-like games
— deckbuilders, tactics games, CRPGs. It keeps your game's state as plain data
rather than a web of Actors, which is what lets the whole game undo, redo, save,
and replay reliably. You write your rules and content on top; the framework
handles the state layer underneath.

## Who it's for

Developers building rule-driven, turn-based games who want:

- Strong **undo / redo / replay** without hand-rolling it per feature.
- **Data-driven content** — units, items, abilities, and stats authored in data
  tables and assets, with Blueprint-heavy authoring and a C++ core.
- Systems that stay **modular** — adopt tagged data on its own, or take the
  whole entity-and-stats stack.

## What you'll build

Following this section, you'll stand up a tiny tactics-board slice: two entities
spawned from shared definitions, typed data attached to one, a stat change
driven through the command stack, a working Undo, and a listener that keeps a
display in sync through all of it. No rendering or turn logic — just the state
layer everything else builds on.

## Prerequisites

- Unreal Engine 5.8 and a project you can build.
- Comfort with either Blueprint or C++ — every step is shown both ways.
- No prior knowledge of the framework. Concepts are introduced as you meet them.

## The path

1. **[Installation](installation.md)** — add and enable the four Phase 1
   plugins, do the one-time schema setup, and confirm it compiles.
2. **[Build Your First Board](first-board.md)** — the flagship tutorial: author
   entity templates, spawn them, store tagged data, make a command-driven stat
   change with a working Undo, and observe a change event.

## Where to go next

Once the tutorial clicks, the **Core Concepts** section explains the "why" behind
each piece. A good reading order:

- **[Entities](../concepts/entities-as-data.md)** — why game state is data, not
  Actors.
- **[Tagged Data](../concepts/tagged-data.md)** — typed structs keyed by
  gameplay tags.
- **[Commands & Undo](../concepts/commands-and-undo.md)** — the one door every
  state change passes through.
- **[Derived State & Events](../concepts/derived-state.md)** — how the rest of
  your app reacts to state.
- **[Evaluators](../concepts/evaluators.md)** and
  **[Stats & Modifiers](../concepts/stats-and-modifiers.md)** — data-driven
  numbers, buffs, and gear.
