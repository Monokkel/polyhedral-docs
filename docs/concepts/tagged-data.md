# Tagged Data: Typed Structs Keyed by Gameplay Tags

For developers who want to attach typed data — a position, a loadout, a custom
struct — to game objects without subclass explosions. After reading, you'll know
how tagged data works, how the schema gives you typed Blueprint pins, and when to
put data on an entity versus a plain actor.

## The model: a tag maps to a typed struct

Tagged data is a map from a gameplay tag to a typed struct value. One key, one
struct. Any struct type works — a primitive wrapper, a position, a whole custom
loadout struct — and different tags on the same object can hold entirely
different types.

Keys live in the `Data.*` gameplay-tag namespace, so `Data.Card.Cost` might hold
a cost struct while `Data.Terrain` holds a terrain struct on the same tile.

This replaces the reflex to grow a class hierarchy every time an object needs a
new field. Instead of a `FireTile` subclass, a tile carries a `Data.Terrain`
entry.

## Two places tagged data lives

Tagged data has two homes, and they behave very differently.

**Standalone, on any object or actor.** Three cooperating pieces let any UObject
carry tagged data:

- an interface (`IPTaTaggedDataInterface`) any object can implement,
- a drop-in component (`UPTaTaggedDataComponent`) you add to an actor, and
- a function library that reads and writes tagged data on *any* target.

The function library resolves the provider for you. If the target implements the
interface, it uses that; otherwise, if the target is an actor, it looks for the
component; and on a set it can create a component on demand. You call one
`SetTaggedData` / `GetTaggedData` pair and let it find the right store.

**On entities.** Tagged data is one of the containers that make up an
[entity's authoritative state](entities-as-data.md). Here it is part of the game
board — not a convenience bolted onto an actor.

## The rule that matters: entity data vs actor data

The two homes carry different guarantees, and choosing the right one is the most
important decision on this page.

!!! warning "Where you store it changes what it guarantees"
    - **Entity tagged data is authoritative game state.** Every write goes
      through the command stack, so it is undoable, saved, and replayed with the
      rest of the game. See
      [Commands: One Door to Game State](commands-and-undo.md).
    - **Actor or component tagged data is just convenient typed storage.** It has
      none of those guarantees — no undo, no automatic save, no replay.

    Put anything your game rules depend on — anything that must survive undo and
    save — on an entity. Reserve actor storage for presentation or editor-time
    configuration.

## The schema: typed pins and validation

The **schema asset** (`UPTaTaggedDataSchema`) maps each tag to the struct type
you expect to find there. It powers two conveniences.

**Typed Blueprint pins.** The custom Blueprint Get and Set nodes read the schema
at edit time and give you a real, typed struct pin — your `FCardCost` pin, not a
generic container. If a tag has no schema entry, the nodes fall back safely to a
generic struct container (`FInstancedStruct`), so an un-schemaed tag still works.

**Optional set-time validation.** With validation enabled in your project
settings, setting a struct whose type doesn't match the schema is rejected
instead of silently stored — a cheap guard against wiring the wrong struct to a
tag.

=== "Blueprint"
    - Define a small struct (say `FCardCost` with an `Energy` field) and mark it
      **BlueprintType**.
    - Add an entry to the schema asset mapping `Data.Card.Cost` to `FCardCost`.
    - Drop the typed **Set Tagged Data** node — its value pin is now a real
      `FCardCost`. Read it back with the typed **Get** node, which outputs a
      `FCardCost` directly.

=== "C++"
    ```cpp
    // A tiny custom struct — no new actor class, just data.
    FCardCost Cost{ /*Energy=*/ 3 };

    // Store it on any actor; a component is created on demand.
    UPTaTaggedDataFunctionLibrary::TrySetTaggedData(
        Tile, TAG_Data_Card_Cost, FInstancedStruct::Make(Cost),
        /*bCreateComponentOnActorIfMissing=*/ true);

    // Read it back out of the generic container.
    FInstancedStruct Raw;
    if (UPTaTaggedDataFunctionLibrary::TryGetTaggedData(Tile, TAG_Data_Card_Cost, Raw))
    {
        const FCardCost& Got = Raw.Get<FCardCost>();
    }
    ```

### Primitive wrappers

When all you need is a single value, you don't have to declare a one-field
struct. The plugin ships small wrapper structs for the common primitives — a
bool, an integer, a float, a name, a string, and friends — so a simple flag or
count is a one-line set.

## Tagged data on entities

On an entity, reads and writes look the same, but the entity model adds two
behaviours worth knowing.

**Template fallback.** When a live entity has no override for a tag, the read
falls through to its [template](entities-as-data.md). A hundred cards share one
"cost" value from their template until one of them is individually re-costed.

**Tag marking.** Setting tagged data also marks the entity with that tag. So
"does this entity have `Data.Card.Cost`?" is answerable with an ordinary tag
query — the presence of the data and the presence of the tag stay in sync.

=== "Blueprint"
    - Use the entity typed **Set Tagged Data** node against an **entity
      reference** — the write is command-driven and undoable.
    - The entity typed **Get** node returns the entity's own value, or the
      template's value when the entity hasn't overridden it.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    // Writing entity data routes through the command stack — undoable.
    State->SetTaggedData(Card, TAG_Data_Card_Cost,
        FInstancedStruct::Make(FCardCost{ 5 }));

    // Falls back to the template when this entity has no override.
    FInstancedStruct Raw = State->GetTaggedData(Card, TAG_Data_Card_Cost);
    ```

Reactive code — health bars, tooltips, board visuals — watches for these writes
rather than polling; see [Derived State and Change Events](derived-state.md).

## Design guidance

Tagged data is easy to over-use. A little discipline keeps it a strength.

!!! tip "Treat the schema as your data dictionary"
    - Prefer a few well-named structs over a scatter of scalar keys. A single
      `Data.Card` struct beats five loose `Data.Card.*` numbers.
    - Treat the schema as your project's designed data surface — the catalogue of
      what data means — not a dumping ground for one-off values.

The complete [node inventory](../plugins/taggeddata/reference.md#typed-blueprint-nodes),
the [schema subsystem](../plugins/taggeddata/reference.md#schema-subsystem), and every
[project setting](../plugins/taggeddata/reference.md#settings) are documented in the
[TaggedData plugin reference](../plugins/taggeddata/reference.md). Wiring the schema
into your project settings is a one-time
[installation step](../getting-started/installation.md).

!!! note "See it in action"
    The [first board tutorial](../getting-started/first-board.md) stores its first
    custom struct on an entity — a good place to try this hands-on.
