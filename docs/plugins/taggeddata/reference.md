# TaggedData API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Tagged Data](../../concepts/tagged-data.md).

All public types carry the `PTa` prefix. Signatures below are hand-written to
show the shape of each API; they are illustrative, not source excerpts.

!!! note "This plugin describes tagged data; it does not store it"
    TaggedData owns the schema, the primitive wrappers, and the typed-pin
    machinery. The read/write API lives on GameEntity's game-state subsystem —
    see [Tagged data on an entity](../gameentity/reference-entities.md#tagged-data).
    There is no interface, component, or function-library storage API here.

## Schema

`UPTaTaggedDataSchema` is a primary data asset that maps each `Data.*` tag to
the struct type stored under it. It is your project's data dictionary, and it
drives both the typed Blueprint pins and the type check on write.

```cpp
// Look up the struct type mapped to Tag, or null if this schema doesn't map it.
UScriptStruct* FindStructTypeForTag(const FGameplayTag& Tag) const;
```

Its authored properties:

| Property | What it does |
|---|---|
| `TagFilter` | An optional namespace filter. When set, it constrains the tag picker for this schema's entries to that namespace. |
| `Entries` | The tag-to-struct mappings, an array of `FPTaTaggedDataSchemaEntry`. |

You register a schema by adding it to the project [settings](#settings). One
project can have several schema assets, each with its own `TagFilter`, so you
can split your data dictionary by area.

### `FPTaTaggedDataSchemaEntry`

One mapping. It has exactly three fields.

```cpp
struct FPTaTaggedDataSchemaEntry
{
    FGameplayTag   Tag;                // a Data.* tag (leaf tags only in the picker)
    UScriptStruct* StructType;         // the struct expected under that tag
    bool           bAllowSubclasses;   // default false — see below
};
```

| Field | Editor label | Default |
|---|---|---|
| `Tag` | **Tag** | — |
| `StructType` | **Struct Type** | none |
| `bAllowSubclasses` | **Allow Subclasses** | unticked |

### Allow Subclasses

By default an entry matches **exactly**: a key mapped to `FMyBase` accepts
`FMyBase` and refuses any subclass. Ticking **Allow Subclasses** switches that
one tag to base-class matching — and, deliberately, also drops its typed
Blueprint pin down to a generic `FInstancedStruct` pin.

| | Type check on write | Typed-pin node |
|---|---|---|
| Default entry | Exact type identity | A pin of `StructType` |
| **Allow Subclasses** ticked | The incoming struct must be `StructType` or derive from it | A generic `FInstancedStruct` pin |

!!! warning "Both halves move together on purpose"
    A base-typed pin cannot honestly represent a subclass value. The typed
    getter needs an exact type match and would hand you back a
    default-constructed struct; the typed setter is worse — it writes the base
    slice back and destroys the subclass value you authored. Relaxing the type
    check on its own would trade one silent failure for two, so the flag relaxes
    the check and drops the typed pin in one move.

Tick it for keys whose owning system validates by base class ("any policy struct
of this family"). Leave it off otherwise — exact matching is what the typed-pin
guarantee rests on.

## Schema subsystem

`UPTaTaggedDataSchemaSubsystem` is an **engine** subsystem that answers "what
struct type belongs under this tag?" at both runtime and edit time. It consults
native registrations first (below), then the schema assets listed in settings,
in list order. Assets load lazily on first use.

```cpp
// Fetch the subsystem (null-safe; returns nullptr if the engine has none).
static UPTaTaggedDataSchemaSubsystem* Get();

// Resolve a tag to its struct type across native registrations and asset
// schemas; null if no source maps it.
UScriptStruct* FindStructTypeForTag(const FGameplayTag& Tag) const;

// Same lookup, also reporting whether the winning entry is polymorphic.
UScriptStruct* FindStructTypeForTag(const FGameplayTag& Tag, bool& bOutAllowSubclasses) const;

// True if any schema source is available at all. Lets you distinguish
// "this tag has no mapping" (this returns true, the lookup returns null) from
// "no schema loaded yet" (this returns false).
bool HasSchemas() const;

// Drop and reload the schema assets listed in project settings.
void ReloadSchemas();
```

`ReloadSchemas` runs at engine start, lazily on the first lookup, and again
whenever you edit the schema list in Project Settings — you rarely call it
yourself.

!!! note "C++ only, except one"
    `ReloadSchemas` is Blueprint-callable. `Get`, `FindStructTypeForTag`, and
    `HasSchemas` are C++ only, as are the native-fragment functions and the
    write-verdict helper below.

!!! tip "Console command"
    `PTa.ReloadSchemas` reloads the schema **assets** from the console without
    restarting the editor — handy after editing a schema by hand. Native
    registrations are unaffected; they are owned by module startup.

### The write verdict

One function decides whether a struct type is acceptable for a tag, and every
gate in the framework calls it. Call it yourself if you validate a tagged-data
write before submitting it.

```cpp
// True if IncomingType may be stored under Tag. On a false return,
// *OutExpectedType names the type the schema expects (never null).
static bool PassesSchemaOnSet(const FGameplayTag& Tag, const UScriptStruct* IncomingType,
                              const UScriptStruct** OutExpectedType = nullptr);
```

Its answers, in order:

1. **True** if enforcement is off, or no schema subsystem exists.
2. **True** if the tag has no schema mapping — the schema is a partial
   description of your `Data.*` space, not a whitelist.
3. **True** if the type matches exactly, or (for an **Allow Subclasses** entry)
   derives from the expected type.
4. **False** otherwise.

Two things to know when you call it:

- It takes a **type**, not a value, so a caller who hasn't built an
  `FInstancedStruct` yet can check first and skip the allocation.
- It **logs nothing**. Reporting a refusal is the caller's job.
- It does **not** check the `Data.*` namespace rule. That check lives on the
  entity write path — see
  [the write gate](../gameentity/reference-entities.md#the-write-gate).

### Native schema registration

If you ship your own C++ module, you can register the tag-to-struct pairings it
**owns** from module startup instead of asking every game author to transcribe
them into a schema asset. Several of the framework's own plugins do exactly this
for their `Data.*` keys.

```cpp
// In StartupModule — the entries this module owns.
static void RegisterNativeSchemaFragment(
    FName FragmentId, const TArray<FPTaTaggedDataSchemaEntry>& Entries);

// In ShutdownModule — always pair it.
static void UnregisterNativeSchemaFragment(FName FragmentId);

// Resolve against native registrations only, plus a diagnostics count.
static UScriptStruct* FindNativeStructTypeForTag(const FGameplayTag& Tag);
static UScriptStruct* FindNativeStructTypeForTag(const FGameplayTag& Tag, bool& bOutAllowSubclasses);
static int32 GetNativeSchemaFragmentTagCount();
```

The registry is process-wide and consulted **before** asset schemas.
**First registration wins**: a later fragment claiming a tag another fragment
already owns is ignored and logged as a warning at load. Re-registering the same
fragment with identical entries is silent. Entries missing a tag or a struct type
are skipped with a warning.

## Settings

`UPTaTaggedDataSettings` is a developer-settings object, surfaced at
**Project Settings → Plugins → Tagged Data**.

| Setting | Editor label | Default |
|---|---|---|
| `SchemaAssets` | **Schema Assets** — the active schema assets, as soft references | empty |
| `bEnforceSchemaOnSet` | **Enforce Schema On Set** — reject a write whose struct type doesn't match the schema entry for its tag | on |

!!! note "It saves to its own config file"
    These settings live in **`Config/DefaultTaggedData.ini`**, not
    `DefaultGame.ini`. Edit them through Project Settings and they travel with
    the project.

Tags with no schema mapping are never blocked by **Enforce Schema On Set**, so
you can enforce types on your designed keys while leaving room for ad-hoc ones.
The `Data.*` namespace rule is a separate check and is **not** governed by this
toggle — see
[the write gate](../gameentity/reference-entities.md#the-write-gate).

## Primitive wrappers

When you only need a single value, use one of the ready-made wrapper structs
rather than declaring a one-field struct. Each is a `BlueprintType` struct with a
single `Value` field (except `FPTaEnumValue`, noted below).

| Wrapper struct | Wrapped type |
|---|---|
| `FPTaBool` | `bool` |
| `FPTaInt64` | `int64` — use for all whole numbers |
| `FPTaDouble` | `double` — use for all floating-point |
| `FPTaName` | `FName` |
| `FPTaString` | `FString` |
| `FPTaText` | `FText` |
| `FPTaEnumValue` | an enum, as an `EnumType` name plus an `int64` `Value` |
| `FPTaObjectRef` | `TObjectPtr<UObject>` |
| `FPTaClassRef` | `TObjectPtr<UClass>` |
| `FPTaSoftObjectPath` | `FSoftObjectPath` |
| `FPTaSoftClassPath` | `FSoftClassPath` |
| `FPTaTransform` | `FTransform` |

!!! warning "There is no `FPTaInt32`, `FPTaFloat`, or `FPTaVector`"
    Integers are 64-bit only (`FPTaInt64`) and floating-point is double only
    (`FPTaDouble`). Reaching for a 32-bit wrapper that doesn't exist is the usual
    first stumble. For a vector, author your own struct — or store an
    `FPTaTransform` if that's what you actually mean.

There is also **`FPTaTaggedDataEntry`**, which is a pair rather than a wrapper:

```cpp
struct FPTaTaggedDataEntry
{
    FGameplayTag     Tag;    // restricted to Data.*, leaf tags only
    FInstancedStruct Value;
};
```

It's how tagged data is authored as an **array of entries** in the details panel
— entity data-table rows use it for their **Manual Tagged Data** list.

```cpp
// Wrapping and unwrapping a single value.
FInstancedStruct Packed = FInstancedStruct::Make(FPTaDouble{ 2.5 });
// ... store Packed on an entity, later read it back ...
const double Speed = Packed.Get<FPTaDouble>().Value;
```

## Typed Blueprint pins

The editor module owns the **shared machinery** behind every typed tagged-data
node in the framework: literal-tag resolution, the schema-to-struct lookup, the
pin retyping, and the pack/unpack conversion each node expands into.

!!! note "TaggedData ships no tagged-data node of its own"
    The nodes you actually place in a graph belong to GameEntity, because that's
    where tagged data is stored. They are **Get Entity Tagged Data (Typed)**,
    **Get Entity Tagged Data (Typed, Exec)** and **Set Entity Tagged Data
    (Typed)**, all under the **GameEntity | TaggedData** palette category —
    documented in the
    [GameEntity reference](../gameentity/reference-entities.md#the-typed-blueprint-nodes).

Behaviour every typed-pin node inherits from this plugin:

- When the **Tag** pin holds a literal that the schema maps, the **Data** pin
  takes that concrete struct type.
- When the tag is dynamic, unmapped, marked **Allow Subclasses**, or the schema
  hasn't loaded yet, the pin falls back to a generic `FInstancedStruct` — the
  node still works.
- The **Tag** pin is filtered to the `Data.*` namespace.
- Nodes never synchronously load assets while you edit or compile a graph, so a
  schema that isn't in memory can never stall or fail a Blueprint compile. An
  unresolved pin re-resolves once the schema is available.
- The node's title compacts to the tag with its leading `Data.` trimmed, so a
  graph full of them stays readable.

### Deriving your own typed-pin node

`UK2Node_PTaTaggedPinBase` is the abstract base to derive from if you add a
storage system of your own and want it to offer the same typed pins. It gives
you the tag pin, the retyping, the reconstruct callbacks, the tag filter, and two
expansion helpers that spawn the pack/unpack conversion for you; you supply the
function call your node expands into.

!!! note "C++ only"
    Deriving a node is a C++ task in an editor (uncooked) module. There is no
    Blueprint path for authoring Blueprint nodes.

The two wildcard conversion functions those helpers spawn
(`UnpackInstancedStruct` / `PackStructToInstancedStruct` on
`UPTaTaggedDataFunctionLibrary`) are marked internal-use-only and **do not
appear in the Blueprint palette**. They exist as node-expansion targets; you use
them by placing a typed node, not by calling them.

### There is no "Expanded" node family

Earlier versions shipped an **Expanded** node variant that generated one
Blueprint pin per member of the mapped struct. It is **gone**, and the entity
nodes have no equivalent.

To reach a single member today, use an ordinary **Break Struct** after
**Get Entity Tagged Data (Typed)**, or **Make Struct** before
**Set Entity Tagged Data (Typed)**. The typed pin gives you the concrete struct
type, so both nodes resolve their members automatically.

## Related pages

- [Guides](guides.md) — defining a schema, using the wrappers, validating on
  write, and the entity read/write recipes.
- [Tagged Data](../../concepts/tagged-data.md) — the model, the guarantees, and
  the design guidance.
- [Tagged data on an entity](../gameentity/reference-entities.md#tagged-data) —
  the storage API this plugin describes.
