# Core Concepts

For a developer deciding *how to think* in the Polyhedral Game Framework — before writing much code. After reading this section you'll understand the handful of ideas the whole framework rests on, and why they make undo, save, and replay work by construction rather than by effort.

Each page is self-contained and shows Blueprint and C++ side by side. You can read them in any order, but they're written to build on each other in the sequence below.

## The through-line

The framework is a small stack of ideas, each one resting on the previous:

1. **[Entities are data, not Actors](entities-as-data.md)** — your whole game is one value of plain data, held in a central game state. That single decision is what makes everything below possible.
2. **[Tagged data](tagged-data.md)** — entities (and any object) carry arbitrary typed structs, keyed by gameplay tags, with typed Blueprint pins from a schema.
3. **[Commands and undo](commands-and-undo.md)** — every change to that data goes through one door, the command stack, which is what turns "plain data" into real undo, redo, and replay.
4. **[Derived state and change events](derived-state.md)** — everything else in your app — UI, visuals, caches — is a reaction to those changes, and stays correct through undo and load by following one subscription rule.
5. **[Evaluators](evaluators.md)** — the numbers and rules *inside* the data are composed from reusable pieces, authored in data instead of hard-coded.
6. **[Stats and modifiers](stats-and-modifiers.md)** — base values with buffs, gear, and auras layered on top, computed exactly and identically on every machine. This is where all of the above come together.

## Where to start

- Want the hands-on version first? The [Getting Started](../getting-started/index.md) tutorial builds a working slice touching most of these ideas, and links back here as each one becomes load-bearing.
- Coming from Unreal's Gameplay Ability System? Start with [Entities](entities-as-data.md), then jump to [Stats and Modifiers](stats-and-modifiers.md) — the base-plus-modifiers model will feel familiar, with one big difference: here the entire stack is plain data that undoes and saves with everything else.
