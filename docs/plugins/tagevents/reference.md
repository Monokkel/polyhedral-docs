# TagEvents API Reference

For developers who want the complete public surface of the plugin. For the model and
when to use it, see the [overview](index.md); for the broadcast layer built on top,
see [Events & Reactions](../../concepts/events-and-reactions.md).

Signatures below are hand-written to show the shape of each API; they are illustrative,
not source excerpts.

## Raising a tag event

`UTagEventFunctionLibrary` is a Blueprint function library — every entry is static and
callable from both Blueprint and C++.

```cpp
// Raise EventTag on CalledObject. If that object declares a handler for the tag, it is
// invoked with (Caller, PayloadObject) and this returns true; otherwise it's a no-op
// that returns false. Caller defaults to self in Blueprint.
static bool RaiseTagEvent(
    UObject* CalledObject,
    FGameplayTag EventTag,
    UTagEventPayloadObject* PayloadObject,
    UObject* Caller);

// Same, but builds the payload for you from loose data — skip the payload object when
// you only need to pass a struct, some tags, or a few numbers. If Payload, Tags, and
// Stats are all empty, no payload object is created at all.
static bool RaiseTagEventWithPayload(
    UObject* CalledObject,
    FGameplayTag EventTag,
    const FInstancedStruct& Payload,
    const FGameplayTagContainer& Tags,
    const TMap<FGameplayTag, float>& Stats,
    UObject* Caller);
```

Two queries round out the surface:

```cpp
// True if Object declares a handler for EventTag. Use it to branch before raising, or
// to check capability without side effects.
static bool HasTagEvent(UObject* Object, FGameplayTag EventTag);

// The reflected function name a given tag resolves to. You rarely need this — it's for
// tooling and for C++ handlers that must match the generated name (see below).
static FName GetGeneratedFunctionNameForTag(FGameplayTag EventTag);
```

=== "Blueprint"
    Drop **Raise Tag Event With Payload**, set **Event Tag**, and wire the **Called
    Object** (leave **Caller** unconnected to default to self). The payload pins accept
    a struct, a tag container, and a tag→number map; leave them all empty for a bare
    signal. The boolean return tells you whether a handler ran.

=== "C++"
    ```cpp
    // Announce "picked up" on an item, handing it the collector as the caller.
    UTagEventFunctionLibrary::RaiseTagEventWithPayload(
        Item,                                   // the object whose handler runs
        ItemPickedUpTag,                        // which handler, by tag
        FInstancedStruct::Make(FPickupInfo{ Collector }),
        /*Tags=*/{}, /*Stats=*/{},
        /*Caller=*/Collector);
    ```

## The payload object

`UTagEventPayloadObject` is the optional argument a handler receives. It's a
`BlueprintType`, `Blueprintable` `UObject`, so you can subclass it — but for most events
the built-in fields carry everything you need without a subclass.

```cpp
class UTagEventPayloadObject : public UObject   // BlueprintType, Blueprintable, EditInlineNew
{
    // Arbitrary typed payload — any USTRUCT, carried without copying it around.
    FInstancedStruct Payload;

    // A mutable tag bag. A handler may read it, and add or remove tags for whatever
    // reads the payload after it.
    FGameplayTagContainer Tags;

    // A mutable tag->number map, for the small numeric facts an event carries.
    TMap<FGameplayTag, float> Stats;

    // True between a Halt() and its ProceedFromHalt().
    bool bHalted;   // read-only

    // Pause an event chain that is being processed through a queue, then resume it.
    void Halt();
    void ProceedFromHalt();
};
```

The `Tags` and `Stats` maps being *mutable* is what lets a handler shape what later
readers of the same payload see — the mechanism the [reaction window](../../concepts/events-and-reactions.md)
uses to let an interrupt reshape a proposed change before it commits.

!!! note "Halt is for queued chains"
    `Halt` / `ProceedFromHalt` only do something when the event is being run through a
    queue that understands them — for example an asynchronous reaction that must wait
    for a player choice before the chain continues. A plain, immediate `RaiseTagEvent`
    has nothing to pause.

## Declaring a handler

A handler is the function a raised tag event calls. You declare it against a tag; the
framework resolves the tag to that function by a generated name and calls it with a
fixed signature — the caller and the payload.

=== "Blueprint"
    1. In any Blueprint, add a **Tag Event** node (right-click → Tag Events).
    2. Set its **Listening Tag** to the tag it should handle.
    3. Wire the exec output; read the **Caller** and **Payload Object** output pins.

    The node *is* the handler — there's no separate function to name. When
    `RaiseTagEvent` fires that tag on this object, this exec line runs.

=== "C++"
    ```cpp
    // The handler's NAME must be the one the tag generates — declare it with the
    // generated name, not a name of your choosing, or the raise will never find it.
    // The signature is fixed: (UObject* Caller, UTagEventPayloadObject* Payload).
    UFUNCTION()
    void OnItemPickedUp(UObject* Caller, UTagEventPayloadObject* Payload);
    ```

    In practice, C++ handlers are usually declared through the higher-level EventSystem
    subscription API rather than by hand-matching generated names; that API takes care
    of the name for you.

!!! warning "Signatures are validated before dispatch"
    Because dispatch calls a function resolved by name, a handler whose signature
    doesn't match what the caller expects — a stale compiled Blueprint, a name
    collision — would be a memory hazard. The core validates the resolved function's
    parameters against the expected shape *before* invoking it and refuses (with a
    logged error) on any mismatch. A wrong-signature handler fails loudly instead of
    corrupting the call frame.
