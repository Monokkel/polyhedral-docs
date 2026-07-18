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

!!! info "Documentation in progress"
    The framework's first four plugins are documented and ready to try today —
    start with **[Getting Started](getting-started/index.md)** and the
    **[Core Concepts](concepts/index.md)** section. More plugin
    reference and guides are being written; check back as the site grows.

## What the framework provides

- **Typed tagged data** on any object — schema-validated, with typed Blueprint nodes
- **Struct-based game entities** — units, items, abilities, and stats with template
  inheritance and data-table authoring
- **A command stack** — every authoritative mutation is undoable, redoable, and replayable
- **Composable evaluators** — data-driven number and condition calculations
- **Reactive derived state** — displays and caches that follow game state through undo and load
- **Stats and modifiers** — base values with buffs, gear, and auras layered on top

## Two ways in

- **[Getting Started](getting-started/index.md)** — the hands-on path.
  [Install the plugins](getting-started/installation.md), then
  [build your first board](getting-started/first-board.md) end to end.
- **[Core Concepts](concepts/index.md)** — the reasoning behind each
  system, starting with why game state is data rather than Actors.

Want to contribute your own guides? See the
[Community Guides](community/index.md) section for the workflow.
