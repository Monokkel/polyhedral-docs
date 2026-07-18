# API Reference: Entities & Templates

For developers who want the complete public surface for creating, reading,
referencing, and organising entities. Entities are plain value data owned by one
central subsystem, all mutation is command-routed, and reads are direct and
free — see [Entities Are Data, Not Actors](../../concepts/entities-as-data.md)
for the model.

This page covers the entity data shapes, entity references, the template store,
data-table rows, and the game-state subsystem's lifecycle, query, hierarchy, and
tagged-data API. Stats and modifiers have [their own page](reference-stats.md);
the change events every mutation fires are documented in
[Change Events](reference-events.md). All public types carry the `PGe` prefix,
and the signatures below are hand-written to show each API's shape — they are
illustrative, not source excerpts.

## The game-state subsystem

`UPGeGameStateSubsystem` is a `UGameInstanceSubsystem` — exactly one per running
game instance. It owns every live entity by value and is the single door to
entity state: you create, read, and mutate entities through it, never through a
pointer to an entity.

=== "Blueprint"
    Use **Get Game State Subsystem** (from `UPGeGameStateLibrary`) or the
    subsystem's own **Get** node — both resolve through any world-context object.

=== "C++"
    ```cpp
    // Resolve through any world-context object.
    static UPGeGameStateSubsystem* Get(const UObject* WorldContextObject);
    ```

!!! note "Every mutation routes through the command stack"
    Each create, destroy, tag, hierarchy, and tagged-data function below submits
    a command, so it participates in undo, redo, save, and replay. Read
    functions do not — they are direct lookups. See
    [CommandSystem](../commandsystem/index.md) and the
    [built-in command inventory](reference-commands.md).

## The entity data shapes

Two struct types describe an entity's readable shape. You rarely construct or
mutate them by hand — the subsystem accessors are the everyday API — but you
receive them from reads such as `GetEntityData`.

`FPGeGameEntity` is the **definitional shape**: the containers an entity or a
template carries.

```cpp
struct FPGeGameEntity          // "Game Entity"
{
    FGameplayTagContainer                  GameplayTags;   // the tags it carries
    TMap<FGameplayTag, int64>              BaseStats;      // authored base stats
    TMap<FGameplayTag, FInstancedStruct>   TaggedData;     // typed data by key
};
```

`FPGeLiveEntity` is a **live instance**: it adds the runtime facts a spawned
entity accumulates on top of its definition.

```cpp
struct FPGeLiveEntity : FPGeGameEntity      // "Live Game Entity"
{
    FName                     TemplateId;   // which template it was made from (None = standalone)
    TMap<FGameplayTag, ...>   CurrentStats; // resolved current stats — derived, never saved
    TMap<FGameplayTag, ...>   StatModifiers;// runtime modifier stacks
    FPGeEntityRef             Parent;       // its parent in the hierarchy
    TArray<FPGeEntityRef>     Children;     // its direct children
};
```

- **Base stats** are whole numbers (`int64`) in display units. Their resolved
  **current** values (with modifiers folded in) are a separate, derived, exact
  fixed-point kind — read those with the stat API, never off `BaseStats`
  directly. See [Stats & Modifiers](reference-stats.md).
- **`CurrentStats`** and the dependency bookkeeping behind them are *derived
  state*: recomputed reactively from change events and rebuilt on load, never
  captured by a command and never in save data. See
  [Derived State and Change Events](../../concepts/derived-state.md).

!!! tip "Template fallback for reads"
    A live entity's `TemplateId` links it to a registered template. Where the
    entity has no local value for a tagged-data key, reads *delegate* to that
    template. So a hundred goblins can share one definition and store only their
    differences. Fallback applies to tagged data (below); base stats and tags
    are copied into the instance at creation.

## Entity references

You never hold an entity itself — you hold an `FPGeEntityRef`, a small,
copyable handle you can store anywhere and resolve through the subsystem on
demand.

```cpp
struct FPGeEntityRef             // "Entity Ref"
{
    FPGeEntityRef();             // an invalid (unset) reference
    explicit FPGeEntityRef(const FGuid& Id);

    bool IsValid() const;        // true once it names a real id (not yet resolved)
    void Invalidate();           // reset to the invalid state
    const FGuid& GetId() const;  // the underlying stable id

    bool operator==(const FPGeEntityRef&) const;   // identity comparison
    bool operator!=(const FPGeEntityRef&) const;
    // Hashable — usable directly as a TMap / TSet key.
    friend uint32 GetTypeHash(const FPGeEntityRef&);

    FString ToString() const;    // log-readable form
};
```

Blueprint helpers on `UPGeGameStateLibrary`:

```cpp
// True if the reference names an id (does not check the entity still exists —
// use EntityExists for that).
static bool IsEntityRefValid(const FPGeEntityRef& EntityRef);

// Wrap an existing FGuid you already hold into a reference.
static FPGeEntityRef MakeEntityRefFromGuid(const FGuid& Guid);
```

!!! warning "The reference contract"
    - **`IsValid` is not existence.** `IsValid` only asks whether the handle
      names an id. To ask whether the entity is still in the game state, call
      [`EntityExists`](#existence-and-queries).
    - Ids are assigned in **creation order** and are **never reused**. A stale
      reference can stop resolving, but it can never silently rebind to a
      different entity.
    - References survive undo and load. The data behind one is rebuilt; the
      handle keeps pointing at the same entity. Always handle the
      "no longer exists" branch when you read.
    - There is deliberately no way to mint a reference to a *new* entity out of
      thin air — creating the entity is what assigns its id. Use the create
      functions below.

## Creating entities

Every create function returns the new entity's reference (invalid on failure)
and routes through the command stack, so creation itself undoes and replays.

```cpp
// An empty entity — no tags, stats, or data yet.
FPGeEntityRef CreateEntity();

// Copy the given definitional data into a brand-new entity (a fresh id is
// assigned; no template link).
FPGeEntityRef CreateEntityFromData(const FPGeGameEntity& EntityData);

// Create from a data-table row. With bRegisterAsTemplate, the row is also
// registered as a template and the entity links to it for tagged-data fallback;
// otherwise the row's full data is copied in with no link.
FPGeEntityRef CreateEntityFromRow(FDataTableRowHandle RowHandle,
                                  bool bRegisterAsTemplate = false);

// Create from a Data Registry item (which must already be cached — see the
// async node below). Same bRegisterAsTemplate meaning as CreateEntityFromRow.
FPGeEntityRef CreateEntityFromRegistry(FDataRegistryId RegistryId,
                                       bool bRegisterAsTemplate = false);

// Create from an already-registered template. Tags and base stats are copied
// in; tagged data is left empty so reads delegate to the template until you set
// an override.
FPGeEntityRef CreateEntityFromTemplate(FName TemplateId);

// Like CreateEntityFromTemplate, but seed per-instance tagged-data overrides
// into the same create — the entity is born already carrying them (one create
// event, no in-between state). Each seeded key is also mirrored as a tag.
FPGeEntityRef CreateEntityFromTemplateWithData(
    FName TemplateId, const TMap<FGameplayTag, FInstancedStruct>& InitialTaggedData);
```

=== "Blueprint"
    Use **Create Entity From Row** for the common designer path — pass a
    data-table row handle and receive an entity reference. **Create Entity From
    Template** and **Create Entity From Template With Data** cover the shared-
    definition path.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    FPGeEntityRef Goblin = State->CreateEntityFromRow(GoblinRow);
    if (!Goblin.IsValid())
    {
        // Row missing or malformed — nothing was created.
    }
    ```

### Creating from a Data Registry (async)

A Data Registry item must be cached before `CreateEntityFromRegistry` can use
it. The async Blueprint node loads it on demand first, then creates the entity.

=== "Blueprint"
    Use **Create Entity From Registry (Async)**. It exposes two exec outputs:

    - **On Succeeded** — carries the new entity reference.
    - **On Failed** — the item could not be acquired.

=== "C++"
    ```cpp
    // In C++, acquire the item through UDataRegistrySubsystem first, then call
    // CreateEntityFromRegistry once it is cached.
    ```

## Destroying entities

```cpp
// Remove an entity and broadcast its destruction. Returns false if the ref
// doesn't resolve. The entity's own children are orphaned (detached), not
// destroyed. Command-routed, so an undo restores it whole.
bool DestroyEntity(const FPGeEntityRef& EntityRef);
```

Destruction fires an `EntityDestroyed` change carrying a final snapshot of the
gone entity for death presentation — see
[Change Events](reference-events.md#the-change-record).

## Existence and queries

Reads are direct lookups — no command, no cost to undo.

```cpp
// Does an entity with this ref currently exist?
bool EntityExists(const FPGeEntityRef& EntityRef) const;

// Copy an entity's full live data out. Returns false if it doesn't exist.
bool GetEntityData(const FPGeEntityRef& EntityRef, FPGeLiveEntity& OutEntity) const;

// C++ direct read — nullptr if not found. Do not hold the pointer across a
// mutation; re-resolve from the ref.
const FPGeLiveEntity* FindEntity(const FPGeEntityRef& EntityRef) const;

// Every live entity, and how many there are.
TArray<FPGeEntityRef> GetAllEntities() const;
int32                 GetEntityCount() const;

// Find entities by tag.
TArray<FPGeEntityRef> FindEntitiesWithTag(FGameplayTag Tag) const;
TArray<FPGeEntityRef> FindEntitiesMatchingQuery(const FGameplayTagQuery& Query) const;
```

## Tags on an entity

Gameplay tags are part of an entity's authoritative state; mutating them is
command-routed, reading them is free.

```cpp
// Add / remove a tag. Each returns false on a no-op (entity missing, or the tag
// was already present / already absent).
bool AddTag(const FPGeEntityRef& EntityRef, FGameplayTag Tag);
bool RemoveTag(const FPGeEntityRef& EntityRef, FGameplayTag Tag);

// Reads.
bool HasTag(const FPGeEntityRef& EntityRef, FGameplayTag Tag) const;
bool HasAnyTags(const FPGeEntityRef& EntityRef, const FGameplayTagContainer& Tags) const;
bool HasAllTags(const FPGeEntityRef& EntityRef, const FGameplayTagContainer& Tags) const;
FGameplayTagContainer GetTags(const FPGeEntityRef& EntityRef) const;   // empty if not found
```

Adding and removing a tag each fire a tag-changed event — see the
[mutation-to-event matrix](reference-events.md#the-mutation-to-event-matrix).

## Tagged data on an entity

Each entity can hold arbitrary typed structs keyed by a `Data.*` gameplay tag.
This is the same tagged-data model the [TaggedData plugin](../taggeddata/index.md)
defines, applied to entities and routed through commands. See
[Tagged Data](../../concepts/tagged-data.md) for the concept.

```cpp
// Set / remove a typed value keyed by a Data.* tag. Set overwrites any existing
// value; Remove returns false if there was nothing to remove.
void SetTaggedData(const FPGeEntityRef& EntityRef, FGameplayTag Tag, const FInstancedStruct& Value);
bool RemoveTaggedData(const FPGeEntityRef& EntityRef, FGameplayTag Tag);

// Reads. Get returns an empty struct if absent; both resolve template fallback.
FInstancedStruct GetTaggedData(const FPGeEntityRef& EntityRef, FGameplayTag Tag) const;
bool             HasTaggedData(const FPGeEntityRef& EntityRef, FGameplayTag Tag) const;
```

Bool-returning helpers on `UPGeGameStateLibrary` back the typed Blueprint nodes:

```cpp
// True (and fills OutData) if the entity has data for Tag — from a live override
// or delegated from its template.
static bool TryGetTaggedData(const UObject* WorldContextObject, const FPGeEntityRef& EntityRef,
                             const FGameplayTag& Tag, FInstancedStruct& OutData);

// True on success.
static bool TrySetTaggedData(const UObject* WorldContextObject, const FPGeEntityRef& EntityRef,
                             const FGameplayTag& Tag, const FInstancedStruct& InData);
```

=== "Blueprint"
    The typed **Get / Set Tagged Data** nodes give you a real typed struct pin
    for a known `Data.*` key, rather than a raw `Instanced Struct` — the same
    typed-pin pattern the [TaggedData plugin](../taggeddata/index.md) documents,
    resolved against the schema at compile time.

=== "C++"
    ```cpp
    // Store a typed struct on an entity.
    State->SetTaggedData(Card, TAG_Data_Cost, FInstancedStruct::Make(FMyCost{ 3 }));

    FInstancedStruct Data = State->GetTaggedData(Card, TAG_Data_Cost);
    if (const FMyCost* Cost = Data.GetPtr<FMyCost>()) { /* ... */ }
    ```

!!! note "Tagged data mirrors a tag"
    Setting tagged data under key `Data.X` also gives the entity the `Data.X`
    **tag**, so `HasTag(Data.X)` and tag queries see it. Removing the data drops
    that mirrored tag again (unless the entity's template still supplies a value
    for the key). This keeps "has this data" answerable through the ordinary tag
    query API.

### Reserved tag namespaces

A few gameplay-tag namespaces have a defined meaning for entities:

- **`Data.Name`** / **`Data.Description`** — the conventional tagged-data keys
  for an entity's display name and description text.
- **`Slot.*`** — hierarchy slot assignments (below).
- **`Scope.*`** — marks an entity as a lifetime *scope* (below).

## Hierarchy and slots

Entities form a parent/child tree, and each child sits in a named **slot**
(a `Slot.*` tag). One structure expresses equipment on a unit, cards in a hand,
abilities granted to a hero, and so on. The hierarchy is authoritative state —
it undoes, saves, and replays with everything else.

```cpp
// Attach Child under Parent in SlotTag (a Slot.* tag). Detaches Child from any
// current parent first. Returns false on failure.
bool AddChild(const FPGeEntityRef& ParentRef, const FPGeEntityRef& ChildRef, FGameplayTag SlotTag);

// Detach a child from its parent; clears its Slot.* tags.
bool RemoveChild(const FPGeEntityRef& ParentRef, const FPGeEntityRef& ChildRef);

// Move a child to a different slot within its current parent.
bool SetChildSlot(const FPGeEntityRef& ChildRef, FGameplayTag NewSlotTag);

// Reads.
FPGeEntityRef         GetParent(const FPGeEntityRef& EntityRef) const;   // invalid if none
bool                  HasParent(const FPGeEntityRef& EntityRef) const;
TArray<FPGeEntityRef> GetChildren(const FPGeEntityRef& EntityRef) const;
TArray<FPGeEntityRef> FindChildrenMatchingQuery(const FPGeEntityRef& EntityRef,
                                                const FGameplayTagQuery& Query) const;
```

Hierarchy changes fire `ChildAdded` / `ChildRemoved` events (and tag events for
the slot tags) — see the
[matrix](reference-events.md#the-mutation-to-event-matrix).

## Templates

A **template** holds definitional data once so many entities can share it. Live
entities record their `TemplateId` and delegate tagged-data reads to it.
Templates are definitional data, not part of undo history.

```cpp
// Register (or, with bAllowOverwrite, replace) a template. Idempotent when the
// content is identical; returns false on an id clash with different content
// unless bAllowOverwrite is set.
bool  RegisterTemplate(FName TemplateId, const FPGeGameEntity& EntityData, bool bAllowOverwrite = false);

// Convenience registrations. Each returns the assigned template id.
FName RegisterTemplateFromRow(FDataTableRowHandle RowHandle);          // "DataTable:<Table>:<Row>"
FName RegisterTemplateFromRegistry(FDataRegistryId RegistryId);        // "Registry:<Type>:<Item>"

// Reads.
bool  GetTemplate(FName TemplateId, FPGeGameEntity& OutEntity) const;  // false if unknown
bool  TemplateExists(FName TemplateId) const;
```

The `bRegisterAsTemplate` flag on `CreateEntityFromRow` /
`CreateEntityFromRegistry` registers and links in one step.

## Authoring rows

Designers author templates as data-table rows of `FPGeGameEntityRow`, which
extends the entity shape with declarative composition. A row merges everything
it inherits and imports into the plain entity containers; call `ToGameEntity()`
(or let the create/register functions do it) to get the clean result.

```cpp
struct FPGeGameEntityRow : FPGeGameEntity        // "Entity Row"
{
    TArray<FDataTableRowHandle> ParentEntities;  // rows to inherit; later entries override earlier
    FText FriendlyName;                          // imported to the Data.Name tagged-data key
    FText Description;                            // imported to Data.Description

    // "Manual" authoring lists, merged into the entity containers:
    FGameplayTagContainer       ManualGameplayTags;  // starting tags
    TArray<FPGeStatEntry>       ManualBaseStats;      // starting base stats (tag + whole value)
    TArray<FPTaTaggedDataEntry> ManualTaggedData;     // starting tagged data
    TArray<FPGeStatEntry>       ManualGrantedMods;    // granted stat modifiers

    FPGeGameEntity ToGameEntity() const;             // the merged, clean result
};
```

**Parent-row inheritance.** `ParentEntities` points a row at other rows whose
merged data it should inherit; later entries win over earlier ones, and the
row's own values win over all of them. Composing a "fire goblin" from a "goblin"
base is pointing one row at another — not writing a subclass.

**Meta-driven imports.** You can add your own `UPROPERTY` fields to a derived
row struct and tag them so their values fold into the standard containers
automatically. Four markers are recognised:

| Marker | Feeds into |
|---|---|
| `StartingStat` | base stats |
| `GrantedStat` | granted stat modifiers |
| `TaggedData` | tagged data (keyed by the marker's tag) |
| `StartingTag` | gameplay tags |

This lets a strongly-typed row (your own hero or item struct) expose friendly,
named fields that still resolve down to the same tags, stats, and data every
entity is made of. `FPGeStatEntry` is the simple `{ Tag, Value }` pair used by
the stat lists.

!!! tip "Whole-unit stats"
    Authored base stats are whole display units. A fractional value entered in a
    row is reported as an import problem rather than silently truncated, so bad
    data surfaces at author time. Fractions only ever arise in the *derived*
    fold — see [Stats & Modifiers](reference-stats.md).

## Scopes and lifetimes

A **scope** is an ordinary entity that happens to carry a `Scope.*` tag; its
reference *is* the scope instance. Scopes are how a modification can be given a
lifetime — "until the encounter ends", "until this card leaves your hand" —
instead of lasting forever. The full lifetime model (end conditions and how they
attach to modifiers) is introduced in
[Stats & Modifiers](../../concepts/stats-and-modifiers.md); this section is the
entity-level surface.

```cpp
// Open a scope: create an entity tagged ScopeTag (must be under Scope.*),
// optionally template-backed and optionally nested under a parent scope.
FPGeEntityRef OpenScope(FGameplayTag ScopeTag, FName TemplateId, const FPGeEntityRef& ParentScope);

// Resolve the currently-open scope instance for a tag (the most recent still-open
// one). Authoring names scope tags; this binds a tag to an instance.
FPGeEntityRef ResolveOpenScope(FGameplayTag ScopeTag) const;
bool          IsOpenScope(const FPGeEntityRef& ScopeRef) const;

// Close a scope: everything given that scope's lifetime expires, as one undo
// unit. Returns false if the ref isn't an open scope.
bool CloseScope(const FPGeEntityRef& ScopeRef);
```

An **end condition** is the single idea a designer works with — "how long does
this last" — as a plain value you build with a helper and hand to a stamping
function.

```cpp
struct FPGeEndCondition { /* Kind + anchor entities */ };

enum class EPGeEndConditionKind : uint8
{
    Permanent,        // never expires (the default and the escape hatch)
    UntilAnchorEnds,  // expires when an anchor scope closes, or an anchor entity is destroyed
    UntilLeaves,      // expires when one entity stops being a descendant of another
};

// Builders — pick one, then pass the result to a Stamp* function.
FPGeEndCondition UntilScopeCloses(const FPGeEntityRef& Scope) const;
FPGeEndCondition UntilDestroyed(const FPGeEntityRef& Anchor) const;
FPGeEndCondition UntilLeaves(const FPGeEntityRef& Container, const FPGeEntityRef& Resident) const;
FPGeEndCondition UntilUnequipped(const FPGeEntityRef& Equipment) const;
FPGeEndCondition Permanent() const;
```

You attach an end condition when you apply a tag, tagged data, a child link, or
a stat modifier, so the mutation cleans itself up when its condition triggers.
Stamping a **stat modifier** under an end condition is covered in
[Stats & Modifiers](reference-stats.md); the tag / tagged-data / child stamps
follow the same `Stamp…(target, …, EndCondition)` shape and produce ordinary,
undoable, event-firing mutations.

## Related pages

- [Stats & Modifiers reference](reference-stats.md) — base stats, current stats,
  modifiers, and stamping under lifetimes.
- [Change Events reference](reference-events.md) — the events every mutation
  above fires, and how to subscribe.
- [Built-in commands reference](reference-commands.md) — the command each
  mutation submits.
- [GameEntity overview](index.md) — where this plugin sits and how to start.
