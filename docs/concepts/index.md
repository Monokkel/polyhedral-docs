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
7. **[Events, ordering, and reaction windows](events-and-reactions.md)** — game rules that *respond* to what happens — armor softening a hit, poison triggering at a tag — run in deterministic order at declared moments, and undo as one step with the change that caused them. This is the reaction mechanism the two pages above kept pointing to.
8. **[Turns and scheduling](turns-and-scheduling.md)** — turn state is an entity and turn boundaries are events, so rounds, initiative, and "at the start of my turn" undo, save, and replay like everything else — with pluggable rules for who acts next.
9. **[Grids and occupancy](grids-and-occupancy.md)** — the board is a durable graph of cells; a unit's position is entity data, "who stands where" is derived state, and "where can this unit go" is a per-unit query — never something baked into the map.
10. **[Presentation: Tokens and Cues](tokens-and-cues.md)** — how committed state changes become what the player sees: every entity gets an on-screen stand-in that is always right, and animated playout is layered on top without ever touching game state. This is where all of the above reaches the screen.
11. **[Abilities and step-by-step resolution](abilities-and-resolution.md)** — a unit's actions are themselves entities: data-authored programs of steps, resolved by a rules engine that pauses for choices and responses — and the same steps run against a throwaway copy of the game state to power previews and enemy AI. This is the final capstone: every idea above, composed.

## Where to start

- Want the hands-on version first? The [Getting Started](../getting-started/index.md) tutorial builds a working slice touching most of these ideas, and links back here as each one becomes load-bearing.
- Coming from Unreal's Gameplay Ability System? Start with [Entities](entities-as-data.md), then jump to [Stats and Modifiers](stats-and-modifiers.md) — the base-plus-modifiers model will feel familiar, with one big difference: here the entire stack is plain data that undoes and saves with everything else.
- Coming from GAS's GameplayCues? Jump to [Tokens & Cues](tokens-and-cues.md) — a cue handler will feel like a GameplayCueNotify, with one hard rule added: presentation code never reads or writes game state out of turn, which is what keeps undo and replay exact.
- Building a grid tactics game? Read [Grids & Occupancy](grids-and-occupancy.md) and [Turns & Scheduling](turns-and-scheduling.md) back to back — board and turn order are the two structures such a game stands on, and both undo and replay by construction here.
- Here for spells, attacks, and passives? Read [Abilities and step-by-step resolution](abilities-and-resolution.md) *last* — it is the capstone that composes everything else, and it assumes the vocabulary of [Events](events-and-reactions.md), [Turns](turns-and-scheduling.md), and [Tokens & Cues](tokens-and-cues.md).
