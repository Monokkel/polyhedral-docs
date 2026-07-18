# TaggedData API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Tagged Data](../../concepts/tagged-data.md).

All public types carry the `PTa` prefix. Signatures below are hand-written to
show the shape of each API; they are illustrative, not source excerpts.

## Interface

`IPTaTaggedDataInterface` is implemented by any object that stores its own tagged
data. All methods are Blueprint-native events, so from C++ you call them through
the generated `Execute_` wrappers (for example
`IPTaTaggedDataInterface::Execute_GetTaggedData(Object, Tag, OutData)`), which
works whether the interface is implemented in C++ or in Blueprint.

```cpp
// Get the struct stored under Tag. Returns true if found.
bool GetTaggedData(const FGameplayTag& Tag, FInstancedStruct& OutData) const;

// Add or replace the struct stored under Tag. Returns true on success.
bool SetTaggedData(const FGameplayTag& Tag, const FInstancedStruct& InData);

// Remove the entry for Tag. Returns true if something was removed.
bool RemoveTaggedData(const FGameplayTag& Tag);

// True if an entry exists for Tag.
bool HasTaggedData(const FGameplayTag& Tag) const;

// Copy every entry into OutData.
void GetAllTaggedData(TMap<FGameplayTag, FInstancedStruct>& OutData) const;

// Replace the whole store at once.
void SetAllTaggedData(const TMap<FGameplayTag, FInstancedStruct>& InData);
```

!!! tip "Prefer the function library for callers"
    You rarely call the interface directly. The [function library](#function-library)
    resolves the right provider for any target and calls these methods for you.
    Implement the interface when you're building a *store*; call the library when
    you're a *consumer*.

## Component

`UPTaTaggedDataComponent` is a drop-in actor component that implements
`IPTaTaggedDataInterface` with a backing `TMap<FGameplayTag, FInstancedStruct>`.
It's marked as a spawnable component, so you can add it to any actor in the editor
or at runtime. Its map is editable in the details panel, letting you author
tagged data directly on a placed actor.

Beyond the interface methods, it exposes direct map access for C++:

```cpp
// Read-only view of the backing map.
const TMap<FGameplayTag, FInstancedStruct>& GetTaggedDataMap() const;

// Mutable access when you need to edit the map in bulk.
TMap<FGameplayTag, FInstancedStruct>& GetMutableTaggedDataMap();
```

!!! note "Created on demand"
    You don't have to add the component by hand. When you set data through the
    function library with `bCreateComponentOnActorIfMissing = true` and the actor
    has no store yet, the library adds one for you at runtime.

## Function library

`UPTaTaggedDataFunctionLibrary` is the main static API — the one most consumers
use. Every function takes a `Target` object and resolves the tagged-data provider
in a fixed order:

1. If `Target` implements `IPTaTaggedDataInterface`, use `Target` itself.
2. Otherwise, if `Target` is an actor, search its components for one that does.
3. On a set, optionally create a `UPTaTaggedDataComponent` on the actor if none
   was found.

In Blueprint, `Target` defaults to `self`, and the tag pins are restricted to the
`Data.*` namespace.

```cpp
// Exec-style get: Result is a Success/Failure exec branch in Blueprint.
static void GetTaggedData(UObject* Target, const FGameplayTag& Tag,
    ETaggedDataResult& Result, FInstancedStruct& OutData);

// Exec-style set. When bCreateComponentOnActorIfMissing is true and Target is an
// actor with no store, a component is created for it.
static void SetTaggedData(UObject* Target, const FGameplayTag& Tag,
    const FInstancedStruct& InData, ETaggedDataResult& Result,
    bool bCreateComponentOnActorIfMissing = false);

// Remove the entry for Tag; Result reports Success/Failure.
static void RemoveTaggedData(UObject* Target, const FGameplayTag& Tag,
    ETaggedDataResult& Result);

// Pure check for the presence of an entry.
static bool HasTaggedData(UObject* Target, const FGameplayTag& Tag);

// Copy every entry on the target's provider into OutData.
static void GetAllTaggedData(UObject* Target,
    TMap<FGameplayTag, FInstancedStruct>& OutData);

// Replace the target provider's entire store.
static void SetAllTaggedData(UObject* Target,
    const TMap<FGameplayTag, FInstancedStruct>& InData);

// Simple bool-returning get — no exec branch. Returns true if found.
static bool TryGetTaggedData(UObject* Target, const FGameplayTag& Tag,
    FInstancedStruct& OutData);

// Simple bool-returning set. Returns true on success.
static bool TrySetTaggedData(UObject* Target, const FGameplayTag& Tag,
    const FInstancedStruct& InData, bool bCreateComponentOnActorIfMissing = false);
```

**`ETaggedDataResult`** is the result enum used by the exec-style functions; in
Blueprint it drives the Success/Failure exec pins.

```cpp
enum class ETaggedDataResult : uint8
{
    Success,
    Failure,
};
```

```cpp
// Typical C++ usage: pack a struct, set it, read it back.
FInstancedStruct Data = FInstancedStruct::Make(FPTaInt64{ 42 });
UPTaTaggedDataFunctionLibrary::TrySetTaggedData(Actor, CountTag, Data,
    /*bCreateComponentOnActorIfMissing=*/ true);

FInstancedStruct Out;
if (UPTaTaggedDataFunctionLibrary::TryGetTaggedData(Actor, CountTag, Out))
{
    const int64 Count = Out.Get<FPTaInt64>().Value;
}
```

!!! warning "Standalone storage has no undo"
    These functions operate on actor/object storage, which is not undoable,
    saved, or replayed. For authoritative game state, write through an entity
    instead — see [Read and write tagged data on entities](guides.md#read-and-write-tagged-data-on-entities).

## Schema

`UPTaTaggedDataSchema` is a data asset (a primary data asset) that maps each
`Data.*` tag to the struct type stored under it. The schema drives typed
Blueprint pins and optional set-time validation.

```cpp
// Look up the struct type mapped to Tag, or null if the tag isn't in this schema.
UScriptStruct* FindStructTypeForTag(const FGameplayTag& Tag) const;
```

Its authored properties:

- **`TagFilter`** — an optional namespace filter. When set, it constrains the tag
  picker for this schema's entries to that namespace.
- **`Entries`** — the tag-to-struct mappings, an array of
  `FPTaTaggedDataSchemaEntry`.

**`FPTaTaggedDataSchemaEntry`** is one mapping:

```cpp
struct FPTaTaggedDataSchemaEntry
{
    FGameplayTag  Tag;         // a Data.* tag
    UScriptStruct* StructType; // the struct expected under that tag
};
```

You register a schema by adding it to the project [settings](#settings). One
project can have several schema assets, each with its own `TagFilter`, so you can
split your data dictionary by area.

## Schema subsystem

`UPTaTaggedDataSchemaSubsystem` is an engine subsystem that answers "what struct
type belongs under this tag?" at both runtime and edit time. It consults native
registrations first (see below) and then the loaded schema assets, loading assets
lazily on first use.

```cpp
// Fetch the subsystem.
static UPTaTaggedDataSchemaSubsystem* Get();

// Resolve a tag to its struct type across native registrations and asset
// schemas; null if no source maps it.
UScriptStruct* FindStructTypeForTag(const FGameplayTag& Tag) const;

// True if any schema source is available at all. Lets you distinguish
// "tag has no mapping" (this is true, the lookup is null) from
// "no schema loaded yet" (this is false).
bool HasSchemas() const;

// Drop and reload the schema assets listed in project settings.
void ReloadSchemas();
```

!!! note "Advanced: native schema registration"
    If you ship your own C++ module, you can register the tag-to-struct pairings
    it owns from module startup instead of hand-authoring them into a schema
    asset. Native registrations are consulted before asset schemas, and the first
    registration to claim a tag keeps it (a later duplicate is ignored and
    logged). Pair a registration in startup with its removal in shutdown, keyed
    by a fragment id you choose.

    ```cpp
    static void RegisterNativeSchemaFragment(
        FName FragmentId, const TArray<FPTaTaggedDataSchemaEntry>& Entries);
    static void UnregisterNativeSchemaFragment(FName FragmentId);

    // Resolve against native registrations only, and a diagnostics count.
    static UScriptStruct* FindNativeStructTypeForTag(const FGameplayTag& Tag);
    static int32 GetNativeSchemaFragmentTagCount();
    ```

## Settings

`UPTaTaggedDataSettings` is a developer-settings object, surfaced at
**Project Settings → Plugins → Tagged Data** and saved to project config.

- **`SchemaAssets`** — the list of active schema assets (soft references). Each
  can carry its own `TagFilter`.
- **`bEnforceSchemaOnSet`** — when true (the default), setting a struct whose type
  doesn't match the schema for its tag is rejected and logged instead of stored.
  Tags with no schema mapping are always allowed.

## Primitive wrappers

When you only need a single value, use one of the ready-made wrapper structs
rather than declaring a one-field struct. Each is a `BlueprintType` struct with a
single `Value` field (except `FPTaEnumValue`, noted below).

| Wrapper struct | Wrapped type |
|---|---|
| `FPTaBool` | `bool` |
| `FPTaInt64` | `int64` (use for whole numbers) |
| `FPTaDouble` | `double` (use for floating-point) |
| `FPTaName` | `FName` |
| `FPTaString` | `FString` |
| `FPTaText` | `FText` |
| `FPTaEnumValue` | an enum, as an `EnumType` name plus an `int64` `Value` |
| `FPTaObjectRef` | `TObjectPtr<UObject>` |
| `FPTaClassRef` | `TObjectPtr<UClass>` |
| `FPTaSoftObjectPath` | `FSoftObjectPath` |
| `FPTaSoftClassPath` | `FSoftClassPath` |
| `FPTaTransform` | `FTransform` |

There is also **`FPTaTaggedDataEntry`**, a `{ FGameplayTag Tag; FInstancedStruct
Value; }` pair. It's handy when you want to author tagged data as an array of
entries rather than a map in the details panel.

```cpp
// Wrapping and unwrapping a single value.
FInstancedStruct Packed = FInstancedStruct::Make(FPTaDouble{ 2.5 });
// ... store Packed, later read it back ...
const double Speed = Packed.Get<FPTaDouble>().Value;
```

## Typed Blueprint nodes

The editor module adds custom Blueprint nodes that read the schema at edit time
to give you a **real typed struct pin** instead of a generic container. They come
in Get and Set forms, pure and exec, plus an Expanded variant that breaks the
struct into per-field pins. All of them target any object through the function
library, and their tag pins are restricted to the `Data.*` namespace.

Common behaviour across every node:

- When the **Tag** is a literal mapped in the schema, the struct pin takes that
  concrete type.
- When the tag is dynamic, unmapped, or the schema isn't loaded, the pin falls
  back to a generic struct container (`FInstancedStruct`) — the node still works.
- They never synchronously load assets while you edit the graph, so they're safe
  during Blueprint compilation.

| Node | Form | What it gives you |
|---|---|---|
| **Get Tagged Data (Typed)** | Pure | A typed struct output plus a **Found** bool. Compact node. |
| **Get Tagged Data (Typed, Exec)** | Exec | A typed struct output with **Success** / **Failure** exec branches. |
| **Set Tagged Data (Typed)** | Exec | A typed struct input, a **Create Component On Actor If Missing** option, and a **Success** output. |
| **Get Tagged Data (Expanded)** | Pure | One output pin per field of the mapped struct, plus a **Found** bool — no break node needed. |
| **Get Tagged Data (Expanded, Exec)** | Exec | Per-field output pins with **Success** / **Failure** exec branches. |
| **Set Tagged Data (Expanded)** | Exec | One input pin per field of the mapped struct, a create-on-demand option, and a **Success** output. |

!!! tip "Typed vs Expanded"
    Use a **Typed** node when you want to pass the whole struct around as one
    pin. Use an **Expanded** node when you just want to read or write a couple of
    the struct's fields inline, without a make/break node. Both fall back to the
    generic container when a tag isn't mapped.

!!! note "These are the standalone nodes"
    These nodes write actor/object storage through the function library. The
    entity equivalents — command-driven, undoable — live with GameEntity and are
    documented in its own section. The read/write shape is the same; the
    guarantees differ, as covered in
    [Tagged Data](../../concepts/tagged-data.md).
