# Plugins

For developers ready to go past the concepts and work with a specific plugin's API. Each plugin section has an **overview** (what it's for and when to reach for it), **guides** (task-oriented how-tos), and an **API reference** (the full public surface, with hand-written examples).

New to the framework? Read the [Core Concepts](../concepts/index.md) first — the plugin sections assume that mental model and link back to it rather than re-explaining it.

## Available now

- **[TaggedData](taggeddata/index.md)** — attach typed structs to any object or entity, keyed by gameplay tags, with typed Blueprint pins driven by a schema. The one plugin you can adopt entirely on its own.
- **[CommandSystem](commandsystem/index.md)** — the undo/redo/replay engine: every authoritative change goes through a command stack, so the whole game rewinds and replays reliably.
- **[Evaluators](evaluators/index.md)** — composable number and true/false calculations authored as data: damage formulas, requirements, and tuning knobs that live in data tables and assets instead of hard-coded branches.

More plugin sections are being added as the site grows.
