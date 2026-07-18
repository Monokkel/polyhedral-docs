# Derived State and Change Events

For developers wiring UI, visuals, or cached lookups to game state. After reading, you'll know which events fire when state changes, the one subscription rule that keeps reactive code correct through undo and load, and what belongs in derived state instead of authoritative state.

## Two kinds of state

Draw a line through everything your game holds:

- **Authoritative state** is what commands mutate and what saves — the entities, tags, base stats, and hierarchy. It is the source of truth.
- **Derived state** is anything you can recompute from it — resolved current stats, lookup caches, the health bar on screen, the token standing on a tile.

Derived state is never saved and never undone. It has no separate history to keep correct. Instead it stays correct by *reacting*: every time authoritative state changes, the change is broadcast, and derived state reconciles to the new truth.

!!! tip "The test for which side something is on"
    If you can rebuild it from authoritative state, it is derived — keep it out of commands and out of save data. If losing it would lose player progress, it is authoritative — it changes through [commands](commands-and-undo.md).

## Every change broadcasts — even during undo and load

The game-state subsystem fires a change event for every authoritative mutation. Crucially, it fires during normal play **and during undo, redo, and load**. Undoing a stat change is, to a listener, just another stat change.

This is what lets reactive code stay simple: it never reasons about *why* state changed. It reconciles to *what state now is*. Health went from 40 to 45 — draw 45. It does not matter whether that was a heal, an undo of damage, or a reload from a save.

## The one subscription rule

There is one catch-all event that fires for *every* change, and one restore event that fires when the whole state is replaced (on load or a wholesale reset). A consumer that must stay correct needs both.

!!! warning "Subscribe to the catch-all plus restore — always both"
    A consumer maintaining derived state subscribes to **`OnAnyEntityChanged`** (the catch-all, fired for every mutation) **and** **`OnStateRestored`** (fired once when state is replaced wholesale, so you can rebuild from scratch).

    The typed events — `OnStatChanged`, `OnTagChanged`, and friends — are conveniences for narrow listeners. Deriving state from only a *subset* of them is the classic way to desync silently: you will miss the one mutation you forgot to subscribe to, and nothing will tell you.

`OnStateRestored` alone is not enough — it fires only at load boundaries, so a listener bound to it sees a correct board on load and drifts everywhere else. The catch-all alone is not enough either — it does not fire for a wholesale replacement the way a rebuild needs. Bind both.

## What a change event carries

The catch-all event hands you a change record (`FPGeEntityChange`) with enough to react without re-querying:

- **The change type** — created, destroyed, tag added, stat changed, child added, and so on.
- **The affected tag or stat**, and the **exact old and new values**. Stat values are an exact fixed-point number type; a comparison on the payload is exact by construction. Convert to float or text only at your own display edge.
- **A cause** (`EPGeChangeCause`): `Command`, `Undo`, `Redo`, or `Restore`.

Use the cause for *presentation* decisions only — "snap the bar instead of animating it when the change came from an undo." Never use it as a game-rules filter; rules do not run from change events at all.

Destroy events carry one extra thing: a final snapshot of the entity, so death presentation can read its closing state after the entity is already gone from the game state. (That snapshot is available to C++ listeners.)

## Listeners are read-only

A change-event listener may never change authoritative state — not directly, and not by submitting a command. This is permanent and the stack enforces it.

!!! warning "Reactions to events are read-only, forever"
    Change events fire during undo and redo as well as forward play. A listener that mutated state would re-drive your rules while the game was trying to undo them. So change events are for **derived state and presentation only**.

State-mutating reactions — "when damaged, retaliate" — are game rules. They belong to the event system's reaction mechanism, documented in a later section, not to change-event listeners.

## The pattern to copy

Every well-behaved consumer has the same shape:

- **On restore**, rebuild wholesale — read the current state and reconstruct your derived structure from nothing.
- **On any change**, update incrementally — adjust just the affected part.
- Keep the derived structure a **pure function of authoritative state**, so both paths land on the same answer.

Here is a health bar that stays correct through play, undo, and load:

=== "Blueprint"
    1. On construct, bind to the game-state subsystem's **On Any Entity Changed** and **On State Restored** events.
    2. In the **On Any Entity Changed** handler: branch on **Change Type**. When it is **Current Stat Changed** for the entity you are watching, read **New Value**, convert it to a float at the display edge, and set the bar.
    3. Branch on **Cause** for the feel: on **Command**, animate; on **Undo**, **Redo**, or **Restore**, snap.
    4. In the **On State Restored** handler: re-read the entity's current stat from scratch and set the bar — the wholesale rebuild path.

=== "C++"
    ```cpp
    void UHealthBar::BindTo(UPGeGameStateSubsystem* State)
    {
        // These are dynamic multicast delegates, so OnChanged and Rebuild
        // must be UFUNCTION()s, bound with AddDynamic.
        State->OnAnyEntityChanged.AddDynamic(this, &UHealthBar::OnChanged);
        State->OnStateRestored.AddDynamic(this, &UHealthBar::Rebuild); // wholesale
    }

    void UHealthBar::OnChanged(const FPGeEntityChange& Change)
    {
        if (Change.EntityRef != Watched) { return; }
        if (Change.ChangeType == EPGeEntityChangeType::CurrentStatChanged)
        {
            const bool bSnap = Change.Cause != EPGeChangeCause::Command;
            SetHealth(Change.NewValue.ToFloat(), bSnap); // convert only at display
        }
    }
    ```

    `Rebuild` re-reads the watched entity's current stat from the subsystem and sets the bar from scratch — the path that runs on load.

The framework's own resolved current stats are maintained exactly this way: recomputed reactively from change events, rebuilt on restore, never saved. It is the same pattern working at scale — see [Stats and Modifiers](stats-and-modifiers.md). Change events fire *from* command execution, which is the other half of the [commands](commands-and-undo.md) story; and the state they describe is the [entity](entities-as-data.md) data itself.

For the complete change-event surface — every event type, the payload fields, and the
[mutation-to-event matrix](../plugins/gameentity/reference-events.md#the-mutation-to-event-matrix) —
see the [GameEntity change-events reference](../plugins/gameentity/reference-events.md#subscribing-to-change-events).

*See it in action: the [first board tutorial](../getting-started/first-board.md) wires a health bar to these events and watches it track through an undo.*
