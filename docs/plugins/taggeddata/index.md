# TaggedData

For developers who want to attach typed data — a position, a loadout, a custom
struct — to any Unreal object without growing a class hierarchy. This section is
the plugin's API and task reference; after it you'll know the full public surface
and how to use it.

Tagged data is a map from a gameplay tag to a typed struct value: one key, one
struct, and different tags on the same object can hold entirely different types.
Keys live under the `Data.*` gameplay-tag namespace.

!!! note "Read the concept first"
    This page assumes you know the model. If you don't yet, start with
    [Tagged Data](../../concepts/tagged-data.md) in Core Concepts — it owns the
    "why" and "when". This section owns the "how": the exact API and the tasks
    you'll actually perform.

## Where it fits in the framework

TaggedData is the framework's foundation layer. It depends on nothing else, and
much of the rest of the framework builds on it: entities store part of their
state as tagged data, stat definitions and granted modifiers are carried as
tagged data, and display conventions read from it. You can also adopt it entirely
on its own — it works on any `UObject` with zero ties to the other plugins.

## Two homes, two guarantees

Tagged data lives in two places that behave very differently. Choosing the right
one is the most important decision you'll make with this plugin.

- **Standalone, on any actor or object.** Convenient typed storage. No undo, no
  automatic save, no replay. Reach for it for presentation state or editor-time
  configuration.
- **On an entity.** Part of an entity's authoritative game state. Every write is
  command-driven, so it is undoable, saved, and replayed with the rest of the
  game.

!!! warning "Put game-rules data on entities"
    Anything your rules depend on — anything that must survive undo and save —
    belongs on an entity, not on an actor. The full contrast is on the
    [Tagged Data](../../concepts/tagged-data.md) and
    [Commands & Undo](../../concepts/commands-and-undo.md) concept pages.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `IPTaTaggedDataInterface` | The interface any object implements to become its own tagged-data store. |
| `UPTaTaggedDataComponent` | A drop-in actor component that implements the interface with a backing map. |
| `UPTaTaggedDataFunctionLibrary` | The main static API. Reads and writes tagged data on *any* target, resolving the provider for you. |
| `UPTaTaggedDataSchema` | A data asset mapping each `Data.*` tag to the struct type stored under it. |
| `UPTaTaggedDataSchemaSubsystem` | The engine subsystem that resolves a tag to its struct type at runtime and edit time. |
| `UPTaTaggedDataSettings` | Project settings: which schema assets are active, and whether to validate on set. |
| `FPTa*` wrapper structs | Small structs wrapping single primitives (bool, int, float, name, string, and more). |
| Typed Blueprint nodes | Custom Get/Set nodes that read the schema to give you a real typed struct pin. |

## Standalone or on an entity

The standalone pieces cooperate so you write one call and let it find the store:
the function library checks whether the target implements the interface, falls
back to a component on an actor, and can create that component on demand. See
[Attach tagged data to any actor or object](guides.md#attach-tagged-data-to-any-actor-or-object).

On an entity, the read/write API mirrors the standalone one but is command-routed
and adds template fallback and automatic tag marking. See
[Read and write tagged data on entities](guides.md#read-and-write-tagged-data-on-entities).

## Where to go next

- **[Guides](guides.md)** — task-oriented recipes: attaching data, defining a
  schema, using the primitive wrappers, validating on set, and working with
  entities.
- **[API Reference](reference.md)** — the full public surface, grouped by area,
  with clean signatures and short examples.
- **[First board tutorial](../../getting-started/first-board.md)** — stores a
  custom struct on an entity as part of the core loop, hands-on.
- **[Installation](../../getting-started/installation.md)** — the one-time step
  that wires your schema into project settings.
