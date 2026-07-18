# Polyhedral Game Framework

A collection of modular **Unreal Engine 5** plugins for building turn-based,
board-game-like games — deckbuilders, tactics games, CRPGs — with a C++ core,
Blueprint-heavy content authoring, and strong undo/redo/replay support throughout.

!!! warning "Documentation under construction"
    This site is being actively built out. Structure and content will grow
    plugin by plugin — check back soon.

## What the framework provides

- **Typed tagged data** on any object — schema-validated, with typed Blueprint nodes
- **Struct-based game entities** — units, items, abilities, and stats with template
  inheritance and data-table authoring
- **A command stack** — every authoritative mutation is undoable, redoable, and replayable
- **Composable evaluators** — data-driven float/bool calculations and conditions
- **Tag-keyed gameplay events** with ordering control and reaction windows
- **Grid and turn systems** that bind entities to boards and scheduling policies
- **A data-driven ability system** tying it all together

## Getting started

Guides, tutorials, and per-plugin reference documentation are on their way.
In the meantime, see the [Community Guides](community/index.md) section for
how to contribute your own guides once the site opens up.
