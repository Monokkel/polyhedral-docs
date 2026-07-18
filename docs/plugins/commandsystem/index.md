# CommandSystem

For developers who want the undo, redo, and replay guarantees the framework
advertises — and want to know what to call to get them. After this section you'll
know the full command-stack API, how to group changes into one undo step, and how
to write a command that reverses itself correctly.

Every authoritative change to game state goes through a **command** submitted to a
single **command stack**; because there's exactly one door writes pass through,
the framework can record and reverse them. That mental model — the "why", the one
door, the rationale for the authoring rules — lives on the
[Commands & Undo](../../concepts/commands-and-undo.md) concept page. This section
owns the mechanics: the exact functions, the transaction API, the custom-command
surface, and the correctness harness.

!!! note "Read the concept first"
    If you haven't yet, start with
    [Commands & Undo](../../concepts/commands-and-undo.md) in Core Concepts. It
    explains reads-are-free / writes-are-commands and the reasoning behind the
    four authoring rules. Everything below assumes that model.

## Where it fits in the framework

The command stack is a **GameInstance subsystem** — one stack per running game.
It has no dependencies on the other plugins; it's a standalone undo/redo engine
that any state owner can route its mutations through. The rest of the framework is
built on top of it: entity state, stats, modifiers, and entity-owned tagged data
are all mutated by command, so they undo, save, and replay as one.

Reads never touch the stack. You look up an entity, a stat, or a tag whenever you
like — only *writes* go through the door.

## Most changes are already commands

Reach for a custom command rarely. The framework ships a large inventory of
ready-made, command-driven mutators — create or destroy an entity, add or remove a
tag, modify a stat, add or remove a modifier, write entity tagged data, and more
(these live with the GameEntity plugin, documented in its own section). Call one
and the change is on the undo stack the moment it returns, with nothing extra to
wire up.

You write a **custom command** only when you add a *new kind of authoritative
state* that the framework doesn't already manage — a scoreboard, a custom turn
tracker, a bespoke resource pool. When that day comes, the
[custom-command guide](guides.md#write-a-custom-command-in-c) walks the four rules
and the [Blueprint path](guides.md#write-a-blueprint-command) covers the Payload
rule.

!!! warning "Reactive listeners never write"
    Code that watches state change — a health bar, a cache, an on-screen token —
    may never submit a command in response. A listener runs *inside* command
    execution, including during undo and redo, so a write from there would
    re-drive your rules while they were being reversed. The stack rejects it
    loudly in development builds. See
    [Derived State & Events](../../concepts/derived-state.md).

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPCsCommandStack` | The stack itself — a GameInstance subsystem. Submit, undo, redo, transactions, queries, events. |
| `FPCsCommand` | The base command struct. A custom C++ command derives from it and implements `Apply` / `Undo`. |
| `FPCsCommandContext` | The per-run context handed to a command — how it reaches its target subsystem. |
| `FPCsCompositeCommand` | The grouping a transaction produces: several commands that undo as one step. |
| `UPCsBlueprintCommand` | The base class you subclass to author a command in Blueprint. |
| `FPCsCmd_Blueprint` | The bridge that runs a Blueprint command; carries its `Payload` across the undo/redo cycle. |
| Conformance harness | A public test surface that exercises your command against the correctness rules and fails the build if it breaks one. |

## Where to go next

- **[Guides](guides.md)** — task recipes: submitting a change and wiring Undo /
  Redo buttons, grouping changes into a transaction, aborting on failure, writing
  a custom command in C++ or Blueprint, and validating one with the harness.
- **[API Reference](reference.md)** — the full public surface, grouped by area,
  with clean signatures and short examples.
- **[Commands & Undo](../../concepts/commands-and-undo.md)** — the model and the
  rationale for the four rules.
- **[First board tutorial](../../getting-started/first-board.md)** — makes a stat
  change through the stack and wires a working Undo button, hands-on.
- **[TaggedData](../taggeddata/index.md)** — the "two homes" split (standalone
  storage vs. command-driven entity storage) that this guarantee sits behind.
