# TaggedData

For developers who want to attach typed data — a position, a loadout, a custom
struct — to their game objects without growing a class hierarchy. This section is
the plugin's API and task reference; after it you'll know the full public surface
and how to use it.

Tagged data is a map from a gameplay tag to a typed struct value: one key, one
struct, and different tags on the same entity can hold entirely different types.
Keys live under the `Data.*` gameplay-tag namespace.

!!! note "Read the concept first"
    This page assumes you know the model. If you don't yet, start with
    [Tagged Data](../../concepts/tagged-data.md) in Core Concepts — it owns the
    "why" and "when". This section owns the "how": the exact API and the tasks
    you'll actually perform.

## What this plugin owns

TaggedData is the framework's foundation layer. It depends on nothing else, and
much of the rest of the framework builds on it: entities store part of their
state as tagged data, granted stat modifiers are carried as tagged data, and
display conventions read from it.

What lives here is the **description** of tagged data, not the data:

- the **schema** — your project's dictionary of which struct type belongs under
  which `Data.*` tag;
- the **primitive wrappers** — ready-made one-field structs so a flag or a count
  doesn't need a struct of its own;
- the **typed-pin machinery** — the shared Blueprint-node base that turns a
  schema entry into a real, typed struct pin.

## Storage lives on entities

There is exactly **one** place tagged data is stored: on an entity, through
GameEntity's game-state subsystem. That write is command-routed, so every value
you store is undoable, saved, replayed, broadcast as a change event, and checked
against the schema — without you wiring any of it.

!!! warning "There is no standalone store"
    Earlier versions let any `UObject` carry tagged data through an interface and
    a drop-in actor component. Both are retired, along with the storage functions
    that resolved between them. They were a second, parallel store with none of
    the guarantees above, so code reaching for them was silently opting out of
    undo, save, and replay. For convenient non-authoritative storage on an actor,
    use an ordinary `UPROPERTY` — keying it with a gameplay tag bought you
    nothing.

So you still read this section to understand and configure tagged data, and to
use the wrappers — but the read/write calls are on
[GameEntity](../gameentity/reference-entities.md#tagged-data).

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPTaTaggedDataSchema` | A data asset mapping each `Data.*` tag to the struct type stored under it. Your data dictionary. |
| `FPTaTaggedDataSchemaEntry` | One mapping: a tag, a struct type, and whether that key accepts subclasses. |
| `UPTaTaggedDataSchemaSubsystem` | The engine subsystem that resolves a tag to its struct type at runtime and edit time, and owns the type-check verdict. |
| `UPTaTaggedDataSettings` | Project settings: which schema assets are active, and whether to type-check on write. |
| `FPTa*` wrapper structs | Small structs wrapping single primitives — bool, 64-bit int, double, name, string, and more. |
| `FPTaTaggedDataEntry` | A tag-plus-value pair, for authoring tagged data as an array in the details panel. |
| `UK2Node_PTaTaggedPinBase` | The shared typed-pin Blueprint-node base the entity nodes derive from. |

## Where to go next

- **[Guides](guides.md)** — task recipes: defining a schema, reading and writing
  entity data, the two ways to remove a key, declaring a polymorphic key, the
  primitive wrappers, and validation on write.
- **[API Reference](reference.md)** — the full public surface, grouped by area,
  with clean signatures and short examples.
- **[Tagged data on an entity](../gameentity/reference-entities.md#tagged-data)** —
  the read/write API and the three typed Blueprint nodes.
- **[First board tutorial](../../getting-started/first-board.md)** — stores a
  custom struct on an entity as part of the core loop, hands-on.
- **[Installation](../../getting-started/installation.md)** — the one-time step
  that wires your schema into project settings.
