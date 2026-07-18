# TaggedData Guides

For developers who have the plugin enabled and want to get things done. Each
section below is a short, followable recipe; pick the one that matches your task.
For the full type-by-type surface, see the [API Reference](reference.md). For the
model behind it all, see [Tagged Data](../../concepts/tagged-data.md).

## Attach tagged data to any actor or object

The quickest way to store a typed struct on any object is the function library.
It resolves the provider for you — the object itself if it implements the
interface, otherwise a component on the actor — and can create that component on
demand.

=== "Blueprint"
    1. Drag a **Set Tagged Data** node (from the *TaggedData* category).
    2. Set **Target** to the object or actor. Leave it as `self` to store on the
       current Blueprint.
    3. Pick a **Tag** in the `Data.*` namespace.
    4. Feed the **In Data** pin. Use a typed node (below) to get a real struct
       pin, or pack a value into a generic container first.
    5. Tick **Create Component On Actor If Missing** if the actor has no store
       yet and you want one made automatically.
    6. Read it back with **Get Tagged Data**, which branches **Success** /
       **Failure**.

=== "C++"
    ```cpp
    FBoardPosition Pos{ /*Column*/ 3, /*Row*/ 2 };

    // Store on any object; make a component if the actor has no store yet.
    UPTaTaggedDataFunctionLibrary::TrySetTaggedData(
        Target, PositionTag, FInstancedStruct::Make(Pos),
        /*bCreateComponentOnActorIfMissing=*/ true);

    // Read it back out of the generic container.
    FInstancedStruct Raw;
    if (UPTaTaggedDataFunctionLibrary::TryGetTaggedData(Target, PositionTag, Raw))
    {
        if (const FBoardPosition* Got = Raw.GetPtr<FBoardPosition>())
        {
            // use Got->Column, Got->Row
        }
    }
    ```

!!! tip "Two flavours of get/set"
    The library exposes an exec pair (`GetTaggedData` / `SetTaggedData`, with a
    Success/Failure exec branch) and a simpler bool-returning pair
    (`TryGetTaggedData` / `TrySetTaggedData`). Use whichever reads more clearly
    in your graph or code.

!!! warning "No undo on actor storage"
    Data set this way is convenient typed storage only — it is not undoable,
    saved, or replayed. For anything your game rules depend on, store it on an
    entity instead (see the last guide on this page).

## Give your own class its own tagged-data store

When an object should *be* its own store rather than delegating to a component,
implement the interface directly. This is the right choice for a `UObject` that
isn't an actor, or when you want tagged data to live on the object itself.

=== "Blueprint"
    1. In your Blueprint's **Class Settings**, add the **Ta Tagged Data
       Interface**.
    2. Implement the interface events (Get / Set / Remove / Has / Get All / Set
       All) against your own storage — commonly a `Map` of tag to a generic
       struct container.
    3. Callers using the function library will now find your object directly, no
       component required.

=== "C++"
    ```cpp
    UCLASS()
    class UMyDataObject : public UObject, public IPTaTaggedDataInterface
    {
        GENERATED_BODY()

    public:
        virtual bool GetTaggedData_Implementation(
            const FGameplayTag& Tag, FInstancedStruct& OutData) const override;
        virtual bool SetTaggedData_Implementation(
            const FGameplayTag& Tag, const FInstancedStruct& InData) override;
        // ...Remove / Has / GetAll / SetAll

    private:
        UPROPERTY()
        TMap<FGameplayTag, FInstancedStruct> Store;
    };
    ```

!!! tip "Or just drop in the component"
    If your class is an actor, you usually don't need to implement anything — add
    a `UPTaTaggedDataComponent` (it's a spawnable component) and it provides the
    store for you. The function library finds it automatically.

## Define a schema and get typed Blueprint pins

A schema maps each `Data.*` tag to the struct type stored under it. It buys you
two things: **typed Blueprint pins** on the custom nodes, and optional
**validation on set**. This is a one-time authoring step.

=== "Blueprint"
    1. Create a **Tagged Data Schema** data asset.
    2. For each tag you'll store, add an entry: the `Data.*` **Tag** and the
       **Struct Type** you expect there (any `BlueprintType` struct — one of the
       `FPTa*` wrappers, or your own).
    3. Open **Project Settings → Plugins → Tagged Data** and add your schema to
       the **Schema Assets** list. It's saved to project config, so it travels
       with the project.
    4. Now drop a typed **Set Tagged Data** node and pick a mapped tag — its
       value pin becomes that real struct type. The matching **Get** node outputs
       the same type.

=== "C++"
    ```cpp
    // With Data.Board.Position mapped to FBoardPosition in the schema, the typed
    // nodes give you an FBoardPosition pin in Blueprint. In C++ you work with the
    // struct type directly and pack/unpack the generic container yourself:
    FInstancedStruct Raw =
        FInstancedStruct::Make(FBoardPosition{ 3, 2 });
    UPTaTaggedDataFunctionLibrary::TrySetTaggedData(Target, PositionTag, Raw);
    ```

!!! tip "Fallback is automatic"
    If a tag has no schema entry — or the schema hasn't loaded yet — the typed
    nodes fall back safely to a generic struct container (`FInstancedStruct`).
    Un-schemaed tags still work; you just don't get a pre-typed pin. You can
    start with an empty schema and fill it in as you go.

The typed nodes also come in an **Expanded** variant that breaks the struct into
one pin per field, and an **Exec** variant with Success/Failure branches. See
[Typed Blueprint nodes](reference.md#typed-blueprint-nodes) for the full set.

## Use the primitive wrappers for simple values

When all you need is a single value, you don't have to declare a one-field
struct. The plugin ships small wrapper structs for the common primitives, so a
flag or a count is a one-line set.

=== "Blueprint"
    1. Make a **PTa Bool** (or **PTa Int64**, **PTa Double**, **PTa Name**, …)
       and set its **Value**.
    2. Feed it into a typed **Set Tagged Data** node whose tag is mapped to that
       wrapper in the schema — or pack it into a generic container for an
       untyped set.

=== "C++"
    ```cpp
    // A simple flag under Data.Card.Exhausted.
    UPTaTaggedDataFunctionLibrary::TrySetTaggedData(
        Card, ExhaustedTag, FInstancedStruct::Make(FPTaBool{ true }));

    FInstancedStruct Raw;
    if (UPTaTaggedDataFunctionLibrary::TryGetTaggedData(Card, ExhaustedTag, Raw))
    {
        const bool bExhausted = Raw.Get<FPTaBool>().Value;
    }
    ```

!!! note "Which wrapper for which type"
    Whole numbers use `FPTaInt64`; floating-point uses `FPTaDouble`. The full
    list — object and class references, soft paths, transforms, and more — is in
    the [Primitive wrappers](reference.md#primitive-wrappers) reference.

## Validate data types on set

Schema validation catches the mistake of wiring the wrong struct to a tag. With
it on, setting a struct whose type doesn't match the schema for that tag is
rejected and logged instead of silently stored.

=== "Blueprint"
    1. Open **Project Settings → Plugins → Tagged Data**.
    2. Leave **Enforce Schema On Set** enabled (it is on by default).
    3. Set data whose type doesn't match a mapped tag — the set fails and a
       warning is logged; correct the struct type to fix it.

=== "C++"
    ```cpp
    // With Data.Board.Position mapped to FBoardPosition and enforcement on,
    // this set is rejected because the struct type doesn't match:
    UPTaTaggedDataFunctionLibrary::TrySetTaggedData(
        Target, PositionTag, FInstancedStruct::Make(FPTaBool{ true }));
    ```

!!! tip "Tags with no mapping are always allowed"
    Validation only applies where the schema actually maps the tag. A `Data.*`
    tag that isn't in any schema is never blocked, so you can enforce types on
    your designed keys while leaving room for ad-hoc ones.

## Read and write tagged data on entities

On an entity, tagged data is authoritative game state. The read/write calls look
like the standalone ones, but they go through the command stack — so they're
undoable, saved, and replayed — and add template fallback plus automatic tag
marking. Entity tagged data is reached through the GameEntity game-state
subsystem rather than the function library (GameEntity is documented in its own
section).

=== "Blueprint"
    1. Get the game-state subsystem and hold an **entity reference** to the entity
       you want to write.
    2. Call the entity's typed **Set Tagged Data** node against that reference —
       the write is command-driven and shows up in Undo.
    3. Read with the entity typed **Get** node: it returns the entity's own
       value, or falls back to the template's value when the entity hasn't
       overridden it.

=== "C++"
    ```cpp
    // Writing entity data routes through the command stack — undoable.
    State->SetTaggedData(Card, PositionTag,
        FInstancedStruct::Make(FBoardPosition{ 3, 2 }));

    // Falls back to the template when this entity has no override.
    FInstancedStruct Raw = State->GetTaggedData(Card, PositionTag);
    ```

!!! note "Two behaviours unique to entities"
    - **Template fallback** — a live entity with no override for a tag reads the
      value from its template, so many entities can share one authored value.
    - **Tag marking** — setting tagged data also marks the entity with that tag,
      so a plain tag query answers "does this entity have `Data.*`?".

    See [Entities as Data](../../concepts/entities-as-data.md) for the entity
    model, and [Derived State & Events](../../concepts/derived-state.md) for how
    reactive code (health bars, tooltips) should watch for these writes instead
    of polling.
