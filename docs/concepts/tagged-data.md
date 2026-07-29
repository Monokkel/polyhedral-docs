# Tagged Data: Typed Structs Keyed by Gameplay Tags

For developers who want to attach typed data — a position, a loadout, a custom
struct — to game objects without subclass explosions. After reading, you'll know
how tagged data works, how the schema gives you typed Blueprint pins, and what
the framework guarantees about every value you store.

## The model: a tag maps to a typed struct

Tagged data is a map from a gameplay tag to a typed struct value. One key, one
struct. Any struct type works — a primitive wrapper, a position, a whole custom
loadout struct — and different tags on the same entity can hold entirely
different types.

Keys live in the `Data.*` gameplay-tag namespace, so `Data.Card.Cost` might hold
a cost struct while `Data.Terrain` holds a terrain struct on the same tile.

This replaces the reflex to grow a class hierarchy every time an object needs a
new field. Instead of a `FireTile` subclass, a tile carries a `Data.Terrain`
entry.

## One door: tagged data lives on entities

There is exactly one place tagged data is stored — on an
[entity](entities-as-data.md), as part of its authoritative state — and exactly
one way to write it: through the game-state subsystem, which routes the write
through the [command stack](commands-and-undo.md).

That single door is the whole point. Because every write goes through it, every
value you store inherits the same guarantees without you doing anything:

- **Undoable and redoable** — a tagged-data write is a command like any other.
- **Saved and replayed** — it is part of the state that serialises and replays.
- **Correct under speculation** — it behaves properly inside the throwaway state
  copies that power [previews and AI](abilities-and-resolution.md).
- **Broadcast** — it raises a change event, so
  [reactive consumers](derived-state.md) stay current.
- **Type-checked** — it passes the schema gate before it is stored (below).

!!! note "There is no actor-side store"
    Earlier versions of the framework also let *any* `UObject` carry tagged data
    through an interface and a drop-in component. That path was retired: it was a
    second, parallel store that had none of the guarantees above, and code that
    reached for it was silently opting out of undo, save, and replay. If you want
    convenient non-authoritative storage on an actor, use an ordinary
    `UPROPERTY` — you gain nothing by keying it with a gameplay tag.

=== "Blueprint"
    - Hold an **entity reference** and use **Set Entity Tagged Data (Typed)**
      against it — the write is command-driven and shows up in Undo.
    - **Get Entity Tagged Data (Typed)** returns the entity's own value, or the
      template's value when the entity hasn't overridden it. Both nodes live under
      the **GameEntity | TaggedData** palette category.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    // Writing entity data routes through the command stack — undoable.
    State->SetTaggedData(Card, TAG_Data_Card_Cost,
        FInstancedStruct::Make(FCardCost{ 5 }));

    // Falls back to the template when this entity has no override.
    FInstancedStruct Raw = State->GetTaggedData(Card, TAG_Data_Card_Cost);
    ```

## The schema: typed pins and a type check

The **schema asset** (`UPTaTaggedDataSchema`) maps each tag to the struct type
you expect to find there. It is your project's data dictionary, and it powers two
things.

**Typed Blueprint pins.** The custom Blueprint Get and Set nodes read the schema
at edit time and give you a real, typed struct pin — your `FCardCost` pin, not a
generic container. If a tag has no schema entry, the nodes fall back safely to a
generic struct container (`FInstancedStruct`), so an un-schemaed tag still works.

**A type check on write.** Setting a struct whose type doesn't match the schema
is rejected and logged rather than silently stored — a cheap guard against wiring
the wrong struct to a tag. A tag the schema doesn't map always passes: the schema
is a partial description of your data space, not a whitelist.

=== "Blueprint"
    - Define a small struct (say `FCardCost` with an `Energy` field) and mark it
      **BlueprintType**.
    - Add an entry to the schema asset mapping `Data.Card.Cost` to `FCardCost`.
    - Drop a **Set Entity Tagged Data (Typed)** node — its **Data** pin is now a
      real `FCardCost`. Read it back with **Get Entity Tagged Data (Typed)**,
      which outputs a `FCardCost` directly.

    Once a tag resolves to a concrete schema type, the **Data** pin must be
    *wired* — you can't type a literal into a struct pin.

=== "C++"
    ```cpp
    // A tiny custom struct — no new actor class, just data.
    FCardCost Cost{ /*Energy=*/ 3 };

    State->SetTaggedData(Card, TAG_Data_Card_Cost,
        FInstancedStruct::Make(Cost));

    // Read it back out of the generic container.
    FInstancedStruct Raw = State->GetTaggedData(Card, TAG_Data_Card_Cost);
    if (const FCardCost* Got = Raw.GetPtr<FCardCost>())
    {
        const int32 Energy = Got->Energy;
    }
    ```

### Two checks, and only one of them is optional

The write gate actually asks two questions, and they are not the same kind of
question.

| Check | Can you turn it off? |
|---|---|
| Does the struct type match the schema entry for this tag? | Yes — **Enforce Schema On Set** in project settings. |
| Is the key inside the `Data.*` namespace? | **No.** |

The namespace rule is not a typing preference, it's an invariant. Tagged data is
readable only while the entity carries the matching marker tag (see below), and
tags outside `Data.*` are managed by other systems that have no reason to know
about tagged data — a value stored under, say, a `Slot.*` key would have its
marker stripped by ordinary slot bookkeeping and become permanently unreadable
while still sitting in the save file. So the namespace check always applies, even
with enforcement off.

### Keys that deliberately hold subclasses

Type matching is **exact** by default: an entry mapped to `FMyBase` accepts
`FMyBase` and refuses a subclass. That exactness is what the typed pin rests on.

Some keys genuinely want a base-class constraint instead — a key holding "any
scheduler policy," for example. A schema entry can declare that by ticking
**Allow Subclasses**, which switches that one tag to base-class matching *and*
gives it a generic `FInstancedStruct` pin instead of a typed one.

!!! warning "Those two effects move together on purpose"
    A base-typed pin cannot honestly represent a subclass value. The typed getter
    would hand you back an empty struct, and the typed setter would write the base
    slice back — quietly destroying the subclass value you authored. Relaxing the
    type check alone would trade one silent failure for two, so the flag relaxes
    the check and drops the typed pin together.

### Primitive wrappers

When all you need is a single value, you don't have to declare a one-field
struct. The plugin ships small wrapper structs for the common primitives — a
bool, an integer, a float, a name, a string, and friends — so a simple flag or
count is a one-line set.

## Template fallback, and how to say "not for me"

An entity delegates a tagged-data read to its
[template](entities-as-data.md) when it has no override of its own. A hundred
cards share one authored "cost" from their template until one of them is
individually re-costed.

That raises a question with a subtle answer: how does *one* instance say a key
its template provides does not apply to it at all? "This copy lost its ability."
"This unit is no longer on the board, even though its template ships placed."

There are two different operations, and picking the wrong one is a real bug:

| Call | Meaning |
|---|---|
| **Remove Tagged Data** | *Clear my override* — the read falls back to the template again. |
| **Shadow Template Tagged Data** | *Absent for me* — the key reads as missing, even though the template provides a value. |

Both undo cleanly. Reach for the first when you're discarding a per-instance
customisation, and the second when the key genuinely should not apply to this
entity.

### The marker tag

Setting tagged data also marks the entity with that tag, so "does this entity
have `Data.Card.Cost`?" is answerable with an ordinary tag query — the presence
of the data and the presence of the tag stay in sync.

That marker is load-bearing, not cosmetic: **it is what makes the value
readable.** Shadowing a template value works by removing the marker, and adding a
marker tag back is therefore a data change, broadcast as one. Two consequences
for your code:

- Removing a marker tag through the raw tag API is refused — it would strand the
  value, readable by nothing and still in your save file. Use the tagged-data
  calls above, which name the route you want.
- Adding a marker tag for a key the template provides *restores* the value, and
  raises a tagged-data change event so occupancy, UI, and every other
  [derived consumer](derived-state.md) reconcile.

## Design guidance

Tagged data is easy to over-use. A little discipline keeps it a strength.

!!! tip "Treat the schema as your data dictionary"
    - Prefer a few well-named structs over a scatter of scalar keys. A single
      `Data.Card` struct beats five loose `Data.Card.*` numbers.
    - Treat the schema as your project's designed data surface — the catalogue of
      what data means — not a dumping ground for one-off values.

The [schema](../plugins/taggeddata/reference.md#schema), the
[schema subsystem](../plugins/taggeddata/reference.md#schema-subsystem), and every
[project setting](../plugins/taggeddata/reference.md#settings) are documented in the
[TaggedData plugin reference](../plugins/taggeddata/reference.md); the read/write
API and its Blueprint nodes live in the
[GameEntity reference](../plugins/gameentity/reference-entities.md#tagged-data).
Wiring the schema into your project settings is a one-time
[installation step](../getting-started/installation.md).

!!! note "See it in action"
    The [first board tutorial](../getting-started/first-board.md) stores its first
    custom struct on an entity — a good place to try this hands-on.
