# Polyhedral Game Framework

A collection of modular **Unreal Engine 5** plugins for building turn-based,
board-game-like games — deckbuilders, tactics games, CRPGs — with a C++ core,
Blueprint-heavy content authoring, and strong undo/redo/replay support throughout.

!!! tip "Start here → [Getting Started](getting-started/index.md)"
    New to the framework? [Install the plugins](getting-started/installation.md)
    and [build your first board](getting-started/first-board.md) — author
    entities, spawn them, change a stat, and undo it: the whole core loop in one
    sitting. Prefer the model first? Read the
    [Core Concepts](concepts/index.md) section.

!!! info "Every plugin has a section"
    The whole framework is documented: **[Getting Started](getting-started/index.md)**,
    the **[Core Concepts](concepts/index.md)** that carry the model, and a
    **[per-plugin section](plugins/index.md)** with overview, guides, and full API
    reference for each one.

## What the framework provides

- **Struct-based game entities** — units, items, abilities, and stats with template
  inheritance and data-table authoring
- **A command stack** — every authoritative mutation is undoable, redoable, and replayable
- **Typed tagged data** — schema-validated typed structs keyed by gameplay tags, with
  typed Blueprint pins
- **Stats and modifiers** — base values with buffs, gear, and auras layered on top
- **Composable evaluators** — data-driven number and condition calculations
- **Reactive derived state** — displays and caches that follow game state through undo and load
- **Gameplay events and reaction windows** — where armour softens a hit and a passive
  fires back, with deterministic ordering
- **Grids and occupancy** — square, hex, or freeform boards, per-unit movement and
  reachability, multi-cell footprints and facing, and a board that can be *dug into* at
  runtime without forfeiting undo
- **Turn scheduling** — pluggable scheduler policies over turn state that undoes and replays
- **Tokens and cues** — the presentation layer that stays correct through undo and load
- **A data-driven ability rules engine** — actions authored as programs of steps, resolved
  one step at a time, and re-run against a throwaway copy of the state to power previews
  and enemy AI

## Two ways in

- **[Getting Started](getting-started/index.md)** — the hands-on path.
  [Install the plugins](getting-started/installation.md), then
  [build your first board](getting-started/first-board.md) end to end.
- **[Core Concepts](concepts/index.md)** — the reasoning behind each
  system, starting with why game state is data rather than Actors.

Want to contribute your own guides? See the
[Community Guides](community/index.md) section for the workflow.
