# Entities Are Data, Not Actors

For developers used to building gameplay state out of Actors and Components.
After reading, you'll know what a game entity is, how to create and reference
one, and why keeping all game state in plain data is what makes undo, save, and
replay work.

## Everything is an entity

A game entity is a record of plain data: gameplay tags, stats, typed data
entries, and links to other entities. Every entity in your game is stored by
value inside one central place — the game-state subsystem, a GameInstance
subsystem that owns the whole board.

An entity is not an Actor. It holds no pointers to world objects, and there is
no per-entity class to subclass. A unit, an item, an ability, a status effect,
a zone on the board — all of them are the same uniform shape. What makes a
goblin a goblin and a sword a sword is the tags and data it carries, not a place
in a class hierarchy.

This is the opposite of the Actor-and-Component habit. You do not reach for
inheritance to add a new kind of thing; you author different data.

The "typed data entries" above are covered on their own page — see
[Tagged Data](tagged-data.md).

## Creating and reading an entity

You create entities through the game-state subsystem, usually from a data-table
row. Reading data back also goes through the subsystem — you never dereference a
pointer to the entity itself.

=== "Blueprint"
    - Get the game-state subsystem from any world context.
    - Call **Create Entity From Row**, passing a data-table row handle. It
      returns an **entity reference**.
    - Read values back with **Get Stat** (for example `Stat.Health`) and
      **Has Tag** (for example `Unit.Enemy`), passing that reference.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    // Create a live entity from a data-table row.
    FPGeEntityRef Goblin = State->CreateEntityFromRow(GoblinRow);

    // Read data back through the subsystem, not off a pointer.
    int64 Health   = State->GetStat(Goblin, TAG_Stat_Health);
    bool  bIsEnemy = State->HasTag(Goblin, TAG_Unit_Enemy);
    ```

Creating an entity is a change to game state, so it goes through the command
stack — which is what gives you undo, redo, and replay for free. You don't have
to think about that here; see [Commands: One Door to Game State](commands-and-undo.md)
for how it works.

## Holding an entity: references

You never hold a pointer to an entity. You hold an **entity reference** — a
small value you can store on a widget, a controller, or another entity — and you
resolve it through the subsystem each time you need the data.

This indirection is what makes the whole system robust:

!!! warning "The entity-reference contract"
    - A reference stays valid across undo and load. The data behind it is
      rebuilt; the reference still points at the same entity.
    - If the entity is destroyed (or undone out of existence), its reference
      simply stops resolving. Always handle the "no longer exists" branch.
    - IDs are assigned in creation order and are **never reused**. A stale
      reference can fail to resolve, but it can never silently rebind to a
      different entity.

=== "Blueprint"
    - Store the **entity reference** as a variable on your widget or controller.
    - When you refresh, branch on **Entity Exists**.
    - On the true branch, read the data you need; on the false branch, hide or
      clear the display — the entity is gone.

=== "C++"
    ```cpp
    // The widget stores only the lightweight reference.
    FPGeEntityRef TrackedUnit;

    void UHealthWidget::Refresh()
    {
        UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

        if (!State->EntityExists(TrackedUnit))
        {
            // Destroyed, or undone away — stop drawing it.
            SetVisibility(ESlateVisibility::Collapsed);
            return;
        }

        const int64 Health = State->GetStat(TrackedUnit, TAG_Stat_Health);
        // ... update the bar ...
    }
    ```

## Templates: shared definitional data

Most entities share their definition with many others. A **template** carries
that shared data once. A live entity records which template it came from and
falls back to it for anything it hasn't overridden.

So a hundred goblins share a single goblin definition. Each live goblin stores
only its differences — its current health, a status effect it happens to be
under — and reads everything else from the template.

Designers author templates as **data-table rows**, with parent-row inheritance:
a child row overrides its parents, and stats, tags, and typed data are all
authored directly in the row. Composing a "fire goblin" from a "goblin" base is
a matter of pointing one row at another, not writing a subclass.

The full authoring surface — [row property imports](../plugins/gameentity/reference-entities.md#authoring-rows),
[registry-backed creation](../plugins/gameentity/reference-entities.md#creating-entities), and
[entity queries](../plugins/gameentity/reference-entities.md#existence-and-queries) — is covered in the
[GameEntity plugin reference](../plugins/gameentity/reference-entities.md).

## The hierarchy and slots

Entities form a parent/child hierarchy, and each child sits in a named **slot**.
Slot tags live under the `Slot.*` namespace.

This one structure expresses a lot of gameplay:

- equipment slotted onto a unit,
- cards held in a hand,
- abilities granted to a hero.

The hierarchy is authoritative game state, exactly like tags and stats. It
undoes, saves, and replays with everything else — reparenting a card back into
a deck is just another change on the stack.

## What lives in entity state — and what never does

Entity state is for the facts your game rules act on and that must survive undo
and save: tags, base stats, typed data, and hierarchy links.

Entity state never holds pointers to world objects, UObjects, Actors, or
anything that can't be copied as plain data. On-screen representation is not
stored on the entity at all — visuals are a *projection* of state, driven by
change events, and are rebuilt from data rather than saved. See
[Derived State and Change Events](derived-state.md).

## Why data-only state matters

Because the entire game is one value of plain data, the framework can copy it,
hash it, save it, and rewind it reliably. That is the foundation everything else
stands on.

!!! tip "The payoff"
    Undo, redo, save, and replay all work because game state is copyable data
    with no hidden pointers. The same property lets more advanced features —
    like the ability system (documented in
    [Abilities and step-by-step resolution](abilities-and-resolution.md#what-if-runs-previews-and-enemy-ai))
    simulating an action in advance without touching the real game — reuse the
    exact same machinery.

!!! note "See it in action"
    The [first board tutorial](../getting-started/first-board.md) creates its
    first entities from data-table rows — start there if you'd rather learn by
    doing.
