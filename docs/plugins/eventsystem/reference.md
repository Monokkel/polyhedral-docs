# EventSystem API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md).

All public types carry the `PEs` prefix. Signatures below are hand-written to show
the shape of each API; they are illustrative, not source excerpts.

## The event subsystem

`UPEsEventSubsystem` is a `UGameInstanceSubsystem` — there is exactly one per
running game instance. It owns subscriptions, resolves broadcasts to their
subscribers, and hosts the loop and depth safety hooks.

```cpp
// Resolve the subsystem through any world-context object. In Blueprint the context
// defaults to self.
static UPEsEventSubsystem* Get(const UObject* WorldContextObject);
```

## Subscribing to events

A subscription is three things: **which event tag**, **which channel**, and an
**order** that decides both its phase and its place in line. The default
subscribe path creates an **observer** — a listener told what happened after every
rule has run, unable to alter anything.

```cpp
// Subscribe an object to an event on a channel at a given order.
//
// bTransient defaults to true: a pure OBSERVER. It is dispatched after every rules
// listener, is kept out of the interrupt phase, and does not open a reaction window.
// Ideal for UI, sound, logging.
//
// Pass bTransient=false ONLY as the escape hatch for a hand-written rules listener
// ("I own this listener's lifecycle myself"). The durable way to author an
// outcome-affecting listener is an entity-carried trigger, documented in a later
// section.
void Subscribe(UObject* Subscriber, FGameplayTag EventTag, FName Channel,
               int32 Order = 0, bool bTransient = true);

// Stop listening to one event on one channel.
void Unsubscribe(UObject* Subscriber, FGameplayTag EventTag, FName Channel);

// Stop listening to everything, on every channel.
void UnsubscribeAll(UObject* Subscriber);
```

In Blueprint you don't call these directly — the **Subscribe to Event** node wraps
the subscribe path and writes the handler for you, always as an observer. A
`...ByObject` node variant takes a live `UObject` as the channel. The node also
exposes an **Order Preset** pin; see [Ordering and channels](#ordering-and-channels).

!!! note "The channel is an FName in C++, a tag in Blueprint"
    C++ `Subscribe` takes the channel as an `FName`. The Blueprint node takes it as
    a gameplay **tag**; bridge them with `Channel.GetTagName()`. A shared channel is
    just a tag under the `Channel.*` namespace; an entity's own channel is an FName
    from `UPGeGameStateSubsystem::GetEntityWindowChannel(Entity)` (see
    [Windowed changes](#windowed-changes-entity-system)).

### The handler and the subscriber id

When an event dispatches, the framework calls a handler **named for the event
tag** on the subscriber. In Blueprint the Subscribe node generates that handler.
In C++ you declare it yourself, and its name is not yours to choose:

```cpp
// The handler for a C++ subscriber. Its NAME is generated deterministically from
// the event tag by the dispatch core — __TagEvent__<sanitized tag>__<checksum> —
// and its signature is fixed. A hand-named function will never be found and never
// fire.
UFUNCTION()
void __TagEvent__Event_Trap_Sprung__1A2B3C4D(UObject* Caller, UTagEventPayloadObject* Payload);
```

Rather than transcribe the checksum, ask the dispatch core for the exact name and
assert it at startup:

```cpp
// Blueprint-pure getter (on the TagEvents function library):
FName Name = UTagEventFunctionLibrary::GetGeneratedFunctionNameForTag(EventTag);

// C++ equivalent:
FName Name = TagEvents::MakeGeneratedFunctionName(EventTag);
```

Every subscriber should also return a **stable id**, so a rules-listener
subscription can be re-derived after a save or replay. Implement
`IPEsEventSubscriberInterface`:

```cpp
class IPEsEventSubscriberInterface
{
    // A stable, deterministic id for this subscriber. Entity-backed objects return
    // their entity id. An object that does not implement this falls back to its
    // object name, with a warning that the subscription will not survive a save.
    FName GetSubscriberId() const;   // BlueprintNativeEvent
};
```

The handler-naming rule and the `UTagEventPayloadObject` base both come from the
[TagEvents](../tagevents/index.md) dispatch core.

### Inspecting subscriptions

Read-only queries — for window-opening decisions, tooling, and debug overlays.

```cpp
// True if anyone at all (observer or rules listener) is listening.
bool HasSubscribers(FGameplayTag EventTag, FName Channel) const;

// True only if at least one RULES LISTENER is listening. The windowed-change entry
// points use this so a pure observer never forces a full reaction window to open.
bool HasNonTransientSubscribers(FGameplayTag EventTag, FName Channel) const;
```

Blueprint-facing debug accessors:

```cpp
// Every (EventTag, Channel) pair currently in the store.
TArray<FPEsSubscriptionKey> GetAllSubscriptionKeys() const;

// The sorted subscriber entries for one event+channel — in dispatch order.
TArray<FPEsSubscriberEntry> GetSubscriberEntries(FGameplayTag EventTag, FName Channel) const;

// A display string ("ClassName (ObjectName)") for a subscriber id, or the raw id.
FString GetSubscriberDisplayName(FName SubscriberId) const;
```

`FPEsSubscriptionKey` names one event on one channel; `FPEsSubscriberEntry`
describes one listener in the sorted list:

```cpp
struct FPEsSubscriptionKey   // BlueprintType
{
    FGameplayTag EventTag;
    FName        Channel;
};

struct FPEsSubscriberEntry   // BlueprintType, read-only
{
    FName  SubscriberId;   // stable id of the listener
    int32  Order = 0;      // < 0 = interrupt phase, >= 0 = reaction phase
    bool   bTransient = false;   // true = observer (sorts after every rules listener)
};
```

!!! note "Entries are already in dispatch order"
    The list is sorted: rules listeners before observers, then by order (lower
    first), and equal orders broken by **entity age — older first**. Reading the
    entries gives you the exact sequence a broadcast will follow.

## Broadcasting an event

A broadcast is a **two-step** send: `BeginBroadcastEvent` resolves the subscribers
and builds the payload; you may bind phase delegates on the returned handle; then
`FinishBroadcastEvent` queues the subscriber calls. The Blueprint **Broadcast
Event** node performs both steps and exposes the phases as exec pins.

```cpp
// Step 1. Resolve subscribers and build the payload. If nobody is listening the
// returned handle has bHasSubscribers == false — a cheap early-out.
static UPEsBroadcastHandle* BeginBroadcastEvent(
    FGameplayTag EventTag,
    FGameplayTag Channel,
    const FInstancedStruct& Payload,
    const FGameplayTagContainer& Tags,
    const TMap<FGameplayTag, float>& Stats,
    UObject* Caller,
    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate);

// Step 2. Queue the subscriber calls. No-op if the handle has no subscribers.
static UPEsBroadcastHandle* FinishBroadcastEvent(UPEsBroadcastHandle* Handle);
```

Two more entry points address the channel differently:

```cpp
// Channel is a live UObject; its subscriber id becomes the channel.
static UPEsBroadcastHandle* BeginBroadcastEventByObject(
    FGameplayTag EventTag, UObject* Channel,
    const FInstancedStruct& Payload, const FGameplayTagContainer& Tags,
    const TMap<FGameplayTag, float>& Stats, UObject* Caller,
    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate);

// Channel is a raw FName — used by the entity window channels, where the channel is
// derived from an entity rather than a tag or object. C++ only.
static UPEsBroadcastHandle* BeginBroadcastEventOnNamedChannel(
    FGameplayTag EventTag, FName Channel,
    const FInstancedStruct& Payload, const FGameplayTagContainer& Tags,
    const TMap<FGameplayTag, float>& Stats, UObject* Caller,
    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate);
```

!!! warning "Begin and Finish are a matched pair"
    `BeginBroadcastEvent` builds the handle but queues nothing; `FinishBroadcastEvent`
    is what actually dispatches. Always call both. The two-step shape exists so you
    can bind the handle's phase delegates in between — bind them *before* Finish, or
    an Immediate broadcast may resolve before your delegate is attached.

### Timing

```cpp
enum class EPEsBroadcastTiming : uint8
{
    Immediate,   // depth-first: subscribers resolve before the caller continues (default)
    Deferred,    // appended to the end of the queue; runs after current work finishes
};
```

### The broadcast handle

`UPEsBroadcastHandle` is what `BeginBroadcastEvent` returns. It carries the
payload, whether anyone is listening, and the phase delegates.

```cpp
class UPEsBroadcastHandle : public UObject   // BlueprintType
{
    bool bHasSubscribers = false;         // false = nobody listening; skip the work
    UPEsEventPayload* PayloadObject;      // the payload (null when no subscribers)

    // Fires after all interrupt subscribers (order < 0) have completed.
    FPEsOnBroadcastPhaseComplete OnAfterInterrupts;   // BlueprintAssignable
    // Fires after ALL subscribers have completed.
    FPEsOnBroadcastPhaseComplete OnAfterAll;          // BlueprintAssignable

    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate;
    int32 BroadcastDepth = 0;             // 0 = top-level; deeper = nested in a subscriber
};

// Pull the payload object off a handle (also the node's Payload Object output).
static UPEsEventPayload* GetPayloadFromHandle(UPEsBroadcastHandle* Handle);
```

`OnAfterInterrupts` is the checkpoint the windowed-change entry points use: the
real command is submitted there, after interrupts have shaped or vetoed the
proposal, and reactions run from the queue afterward.

## Ordering and channels

A listener's integer order decides its phase and place: **negative** is the
interrupt phase (before commit), **zero and positive** the reaction phase (after
commit), and within a phase lower runs first. Observers always run last, after
every rules listener, regardless of order.

Designers pick orders by name using presets under the `EventOrder.*` gameplay-tag
namespace, mapped to integers in project settings.

```cpp
// Resolve a preset tag to its integer order. Returns OrderOverride when the preset
// is empty or unmapped.
static int32 ResolveOrderPreset(FGameplayTag OrderPreset, int32 OrderOverride = 0);

// Resolve a UObject to its channel FName (its subscriber id, with a name fallback).
static FName ResolveObjectChannel(UObject* ChannelObject);
```

A **channel** is either a shared channel — a tag under `Channel.*` that any
interested rule watches — or one entity's own channel, so a rule can listen to a
single unit without hearing every unit in the game.

The order-preset map and the two safety caps are authored in **Project Settings
&rarr; Plugins &rarr; Event System** (`UPEsEventSettings`):

- **`OrderPresets`** — the `EventOrder.* → int` map that `ResolveOrderPreset` reads.
- **`DefaultSiteRepetitionCap`** — the loop terminator (see
  [Loop and depth safety](#loop-and-depth-safety)).
- **`MaxNestedBroadcastDepth`** — the recursion backstop for Immediate nesting.

!!! tip "Ties break by entity age"
    When two rules listeners share the same order, the older entity acts first.
    Entity age is a pure function of game state, so the sequence is identical in
    live play, after an undo, and in a replay. Use it only for stability — when the
    sequence matters, encode it with distinct order values.

## The payload

The payload is a `UPEsEventPayload`, built on the TagEvents base
`UTagEventPayloadObject`. It carries the event's facts and is what an interrupt
reshapes.

```cpp
class UPEsEventPayload : public UTagEventPayloadObject   // BlueprintType
{
    // From UTagEventPayloadObject:
    FInstancedStruct           Payload;   // the typed struct of event-specific facts
    FGameplayTagContainer      Tags;      // convenience tags; subscribers may add/remove
    TMap<FGameplayTag, float>  Stats;     // named numbers — the reshapeable magnitudes
    // Halt() / ProceedFromHalt() — pause and resume the chain for async work.

    // Added by EventSystem:
    FGameplayTag  EventTag;   // what happened
    FName         Channel;    // where
    UObject*      Caller;     // who raised it
};
```

The **`Stats`** map is where a windowed change puts its proposed magnitude, under
the stat's tag, as a float — an interrupt reads and rewrites that entry to shape
the amount. Read the typed `Payload` struct with the **Get Typed Payload** node in
Blueprint, or in C++:

```cpp
// Extract the typed struct from a payload object (backs the typed getter node).
static bool GetPayloadInstancedStruct(UTagEventPayloadObject* PayloadObject, FInstancedStruct& OutData);
```

## Interrupting a broadcast

An interrupt in the pre-commit phase can cancel the change outright — mark the
broadcast interrupted and every remaining subscriber is skipped.

```cpp
// Mark the broadcast interrupted. Subsequent subscribers do not run; a windowed
// change reads this mark and does not commit.
static void InterruptEvent(UTagEventPayloadObject* PayloadObject);
```

- On a **stat** window, an interrupt may either *shape* the magnitude (rewrite the
  `Stats` entry) or *cancel* it (call `InterruptEvent`).
- On a **tag** window, there is no magnitude — an interrupt may only cancel
  ("immune to poison").

## Loop and depth safety

Because reactions can cause reactions, the framework bounds runaway chains two
ways.

The **per-listener repetition cap** is the game-rules loop terminator. It counts
how many times each listener fires for the same event within one causal chain; at
the cap it stops that listener and raises a hook.

```cpp
// Fires the first time a listener would fire past DefaultSiteRepetitionCap within
// one causal chain. Bind your loop-resolution rule here.
FPEsOnSiteRepetitionCapReached OnSiteRepetitionCapReached;   // BlueprintAssignable
//   (FName SubscriberId, FGameplayTag EventTag, FName Channel, int32 DispatchCount)

// Valid only inside the handler above: raise the cap and let this chain continue.
void ExtendSiteRepetitionCap(int32 NewCap);
```

The **nested-broadcast depth cap** is a separate, higher recursion backstop for
stack safety. When Immediate nesting reaches `MaxNestedBroadcastDepth`, the
broadcast is demoted to Deferred and a hook fires.

```cpp
// Fires when nesting hits MaxNestedBroadcastDepth; the broadcast is demoted to Deferred.
FPEsOnMaxBroadcastDepthReached OnMaxBroadcastDepthReached;   // BlueprintAssignable
//   (int32 CurrentDepth)

// Current nesting depth (0 when idle) and the handle being dispatched right now.
int32 GetCurrentBroadcastDepth() const;
UPEsBroadcastHandle* GetActiveBroadcastHandle() const;
```

!!! note "Two different backstops"
    The repetition cap is your **balance-safe loop knob** — catch mutually-triggering
    rules and decide the outcome. The depth cap is a pure **stack-safety** limit; keep
    it high and treat it as a last resort. A resource-terminated loop should exhaust
    itself well below either.

## Windowed changes (entity system)

The windowed-change entry points ship with the **[GameEntity](../gameentity/reference-stats.md)**
plugin — they change entity state — but they open their reaction windows on this
plugin's machinery. A windowed change broadcasts on the target entity's own
channel, keyed by the changed stat or tag, and brackets the real command around
its subscribers. They are documented here because reaction windows are an event
concept; the resolved stat and tag state is documented with GameEntity.

```cpp
// Windowed stat change. Opens a stat-change window on the target's own channel,
// keyed by StatTag. Interrupts (order < 0) may shape the magnitude or veto; the
// real command commits after the interrupt phase; reactions (order >= 0) run
// post-commit. Delta is whole units. Returns false only if the entity/stat is
// invalid or the stack is unavailable — a vetoed change still returns true.
bool ApplyStatChange(const FPGeEntityRef& Entity, FGameplayTag StatTag, int64 Delta,
                     const FPGeEntityRef& Instigator);

// Windowed tag change. Same shape, but there is no magnitude — an interrupt may
// only VETO the add/remove. Returns false for a no-op (adding a tag already
// present, removing an absent one) or an invalid target.
bool ApplyTagChange(const FPGeEntityRef& Entity, FGameplayTag Tag, bool bAdd,
                    const FPGeEntityRef& Instigator);

// The channel FName for an entity's windows (its stable id). Feed this to Subscribe
// or a broadcast to address one entity's window.
static FName GetEntityWindowChannel(const FPGeEntityRef& Entity);

// Subscribe to an entity's window as an OBSERVER (always transient). Sees the
// proposal but cannot veto or reshape it. To author a listener that shapes or
// vetoes, use an entity-carried trigger — documented in a later section.
void SubscribeToEntityWindow(UObject* Subscriber, const FPGeEntityRef& Entity,
                             FGameplayTag EventTag, int32 Order = 0);
void UnsubscribeFromEntityWindow(UObject* Subscriber, const FPGeEntityRef& Entity,
                                 FGameplayTag EventTag);
```

<!-- pluginlink: abilitysystem-reference-triggers -->

The typed struct on a window broadcast records the immutable facts of the
proposal. Note the **duality**: the immutable original delta rides the typed
payload as an `int64`, while the *live, reshapeable* magnitude rides the payload's
`Stats` map as a float under the stat's tag. An interrupt shapes the float; the
`int64` is the record of what was originally proposed.

```cpp
struct FPGeStatChangeWindowPayload   // BlueprintType, read-only
{
    FPGeEntityRef Target;         // the entity taking the change
    FPGeEntityRef Instigator;     // the cause (may be invalid)
    FGameplayTag  StatTag;        // the changed stat (also the event tag)
    int64         OriginalDelta;  // the delta as proposed, before any interrupt shaped it
};

struct FPGeTagChangeWindowPayload   // BlueprintType, read-only
{
    FPGeEntityRef Target;
    FPGeEntityRef Instigator;
    FGameplayTag  Tag;            // the changed tag (also the event tag)
    bool          bAdd = true;    // true = adding, false = removing
};
```

!!! warning "Windowed changes are forward-only"
    Never call `ApplyStatChange` / `ApplyTagChange` from a command's apply/undo path
    or from a change-event listener — those are read-only by construction. A windowed
    change is a live gameplay action, and its whole reaction cascade folds into one
    undo step. The plain, window-blind changes (Modify Stat / Add Tag) live with
    [GameEntity](../gameentity/reference-stats.md); their change events are in the
    [change-events reference](../gameentity/reference-events.md).
