# TaggedData Guides

For developers who have the plugin enabled and want to get things done. Each
section below is a short, followable recipe; pick the one that matches your task.
For the full type-by-type surface, see the [API Reference](reference.md). For the
model behind it all, see [Tagged Data](../../concepts/tagged-data.md).

Tagged data is stored on entities, so every read/write recipe here goes through
GameEntity's game-state subsystem. Get it once and hold it.

=== "Blueprint"
    Call **Get Game State Subsystem** (from the *GameEntity* category), passing a
    world context, and store the result.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);
    ```

## Read and write tagged data on an entity

This is the one storage path. Writes are command-driven — undoable, saved, and
replayed — and reads fall back to the entity's template when the entity has no
override of its own.

=== "Blueprint"
    1. Drop a **Set Entity Tagged Data (Typed)** node (palette category
       **GameEntity | TaggedData**) and wire your **Entity Ref** to it.
    2. Pick a **Tag** in the `Data.*` namespace. If the schema maps it, the
       **Data** pin becomes that concrete struct type.
    3. Wire the **Data** pin — a **Make Struct** node is the usual source. The
       node reports **Success**, and the write shows up in Undo.
    4. Read it back with **Get Entity Tagged Data (Typed)**: it outputs the
       typed struct plus a **Found** bool. Use **Get Entity Tagged Data (Typed,
       Exec)** instead when you'd rather branch on **Success** / **Failure** exec
       pins than test a bool.

=== "C++"
    ```cpp
    // Writing entity tagged data routes through the command stack — undoable.
    State->SetTaggedData(Card, PositionTag,
        FInstancedStruct::Make(FBoardPosition{ 3, 2 }));

    // Reads resolve the entity's own value, then fall back to its template.
    FInstancedStruct Raw = State->GetTaggedData(Card, PositionTag);
    if (const FBoardPosition* Pos = Raw.GetPtr<FBoardPosition>())
    {
        // use Pos->Column, Pos->Row
    }

    // Presence check, same fallback.
    const bool bPlaced = State->HasTaggedData(Card, PositionTag);
    ```

!!! warning "A wired Data pin, not a typed-in literal"
    Once a **Tag** resolves to a concrete schema type, the **Data** pin *must* be
    wired — a typed struct pin can't take an inline literal, and the Blueprint
    fails to compile with a message saying so. Feed it a **Make Struct**.

!!! warning "In C++, `SetTaggedData` returns `void`"
    A write the schema gate refuses is logged, but the subsystem's
    `SetTaggedData` has no way to tell you. When you need to know whether the
    write landed, call the bool-returning `TrySetTaggedData` on
    `UPGeGameStateLibrary` instead — it runs the same gate and returns `false` on
    a refusal. (The typed Blueprint nodes already use it; that's where their
    **Success** pin comes from.)

Full API, including the gate and the mirror-tag rule:
[Tagged data on an entity](../gameentity/reference-entities.md#tagged-data).

## Define a schema and get typed Blueprint pins

A schema maps each `Data.*` tag to the struct type stored under it. It buys you
two things: **typed Blueprint pins** on the entity get/set nodes, and a **type
check on write**. This is a one-time authoring step.

=== "Blueprint"
    1. Create a **Tagged Data Schema** data asset.
    2. For each tag you'll store, add an entry: the `Data.*` **Tag** and the
       **Struct Type** you expect there (any `BlueprintType` struct — one of the
       `FPTa*` wrappers, or your own).
    3. Open **Project Settings → Plugins → Tagged Data** and add your schema to
       the **Schema Assets** list. It's saved to project config, so it travels
       with the project.
    4. Now drop a **Set Entity Tagged Data (Typed)** node and pick a mapped tag —
       its **Data** pin becomes that real struct type. The matching **Get Entity
       Tagged Data (Typed)** node outputs the same type.

=== "C++"
    ```cpp
    // With Data.Board.Position mapped to FBoardPosition in the schema, the typed
    // nodes give you an FBoardPosition pin in Blueprint. In C++ you work with the
    // struct type directly and pack it yourself:
    State->SetTaggedData(Card, PositionTag,
        FInstancedStruct::Make(FBoardPosition{ 3, 2 }));
    ```

!!! tip "Fallback is automatic"
    If a tag has no schema entry — or the schema hasn't loaded yet — the typed
    nodes fall back safely to a generic struct container (`FInstancedStruct`).
    Un-schemaed tags still work; you just don't get a pre-typed pin. You can
    start with an empty schema and fill it in as you go.

!!! note "Shipping a module of your own?"
    A C++ module can register the tag-to-struct pairings it *owns* from module
    startup, so game authors never transcribe them into a schema asset. See
    [native schema registration](reference.md#native-schema-registration).

## Declare a key that accepts subclasses

Sometimes a key genuinely holds "any struct of this family" — a policy struct
with several concrete variants, say. Exact type matching would refuse every
subclass, so the schema entry has to say so explicitly.

=== "Blueprint"
    1. In your schema asset, find the entry for that tag.
    2. Set its **Struct Type** to the **base** struct of the family.
    3. Tick **Allow Subclasses**.
    4. That tag's get/set nodes now give you a generic **Instanced Struct** pin
       instead of a typed one. Build the concrete value with **Make Struct**, then
       pack it into an instanced struct to feed the **Data** pin.

=== "C++"
    ```cpp
    // With Allow Subclasses ticked on the entry for this tag, a derived struct
    // is accepted where the base was expected.
    State->SetTaggedData(Unit, PolicyTag,
        FInstancedStruct::Make(FMyConcretePolicy{ /* ... */ }));

    // Read it back as the base and test for the concrete type you want.
    FInstancedStruct Raw = State->GetTaggedData(Unit, PolicyTag);
    if (const FMyConcretePolicy* P = Raw.GetPtr<FMyConcretePolicy>())
    {
        // ...
    }
    ```

!!! warning "You lose the typed pin, and that's deliberate"
    **Allow Subclasses** relaxes the type check *and* drops the typed pin
    together, because a base-typed pin cannot represent a subclass value: the
    typed getter would hand back an empty struct, and the typed setter would
    write the base slice back and destroy your subclass value. Tick it only for
    keys whose consuming system really does validate by base class — exact
    matching is what the typed pin rests on.

## Remove a key: clear an override, or shadow the template

There are **two** removal operations and they mean different things. Picking the
wrong one is a real bug, not a style choice.

| Call | Meaning | After it, a read returns |
|---|---|---|
| **Remove Tagged Data** | *Clear my override* | the template's value, if the template provides one |
| **Shadow Template Tagged Data** | *Absent for me* | nothing — the key reads as missing |

=== "Blueprint"
    1. To discard a per-instance customisation — this card was re-costed and you
       want the authored cost back — call **Remove Tagged Data** with the entity
       reference and the tag.
    2. To make a key genuinely not apply to this one entity, even though its
       template supplies a value — this unit is off the board although its
       template ships placed — call **Shadow Template Tagged Data**.
    3. Both return **false** when there was nothing to do, and both undo cleanly.

=== "C++"
    ```cpp
    // The template says 3x3; this instance was overridden to 1x1.
    // Clear the override: the read goes back to the template's 3x3.
    State->RemoveTaggedData(Unit, FootprintTag);

    // Make the key absent for this instance, template or not.
    State->ShadowTemplateTaggedData(Unit, FootprintTag);
    ```

!!! tip "Undoing a shadow, and lifting one later"
    Undo restores whichever state you had. A later **Set Entity Tagged Data
    (Typed)** lifts a shadow by writing an override; adding the key's marker tag
    back with **Add Tag** lifts it *without* writing one, so the entity returns to
    reading its template's value.

## Use the primitive wrappers for simple values

When all you need is a single value, you don't have to declare a one-field
struct. The plugin ships small wrapper structs for the common primitives, so a
flag or a count is a one-line set.

=== "Blueprint"
    1. Make a **PTa Bool** (or **PTa Int64**, **PTa Double**, **PTa Name**, …)
       and set its **Value**.
    2. Map the tag to that wrapper in your schema, then feed the wrapper straight
       into the **Data** pin of a **Set Entity Tagged Data (Typed)** node.

=== "C++"
    ```cpp
    // A simple flag under Data.Card.Exhausted.
    State->SetTaggedData(Card, ExhaustedTag,
        FInstancedStruct::Make(FPTaBool{ true }));

    const FInstancedStruct Raw = State->GetTaggedData(Card, ExhaustedTag);
    const bool bExhausted = Raw.IsValid() && Raw.Get<FPTaBool>().Value;
    ```

!!! note "Which wrapper for which type"
    Whole numbers use `FPTaInt64` and floating-point uses `FPTaDouble` — there is
    no 32-bit integer or single-precision float wrapper, and no vector wrapper.
    The full list — object and class references, soft paths, transforms — is in
    the [Primitive wrappers](reference.md#primitive-wrappers) reference.

## Validate data types on write

The schema catches the mistake of wiring the wrong struct to a tag. With
validation on, a write whose struct type doesn't match the schema for that tag is
refused and logged instead of silently stored.

=== "Blueprint"
    1. Open **Project Settings → Plugins → Tagged Data**.
    2. Leave **Enforce Schema On Set** enabled (it is on by default).
    3. Write a struct whose type doesn't match a mapped tag — the write is
       refused and a warning is logged naming both the expected and the supplied
       type. Correct the struct type to fix it.

=== "C++"
    ```cpp
    // With Data.Board.Position mapped to FBoardPosition and enforcement on,
    // this write is refused and logged — nothing reaches the command stack.
    State->SetTaggedData(Card, PositionTag,
        FInstancedStruct::Make(FPTaBool{ true }));
    ```

!!! tip "Tags with no mapping are always allowed"
    Validation only applies where the schema actually maps the tag. A `Data.*`
    tag that isn't in any schema is never blocked, so you can enforce types on
    your designed keys while leaving room for ad-hoc ones.

!!! warning "One half of the gate can't be turned off"
    **Enforce Schema On Set** governs the *type* check only. The write path also
    refuses any key outside the `Data.*` namespace, and that check always
    applies. See
    [the write gate](../gameentity/reference-entities.md#the-write-gate) for why.

The gate sits at the write call, never inside the command it submits — so
loading a save, undoing, redoing, and replaying are all exempt. A value that was
legal when it was stored still loads after you tighten the schema.
