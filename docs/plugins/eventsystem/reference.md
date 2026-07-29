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

A subscription is keyed by three things: **which event tag**, **which channel**,
and **which phase**. Alongside the key it carries an **order** that decides its
place in line and whether it runs before or after a change commits. The default
subscribe path creates an **observer** — a listener told what happened after every
rule has run, unable to alter anything.

```cpp
// Subscribe an object to an event on a channel, at a given phase and order.
//
// bTransient defaults to true: a pure OBSERVER. It is dispatched after every rules
// listener, is kept out of the interrupt phase, and does not open a reaction window.
// Ideal for UI, sound, logging.
//
// Pass bTransient=false ONLY as the escape hatch for a hand-written rules listener
// ("I own this listener's lifecycle myself"). The durable way to author an
// outcome-affecting listener is an entity-carried trigger, documented in a later
// section.
//
// Phase defaults to Main — see "Event phases" below.
void Subscribe(UObject* Subscriber, FGameplayTag EventTag, FName Channel,
               int32 Order = 0, bool bTransient = true,
               EPEsEventPhase Phase = EPEsEventPhase::Main);

// Stop listening to one event on one channel at one PHASE. The phase is part of
// the key, so this removes only the Main-phase entry unless you say otherwise.
void Unsubscribe(UObject* Subscriber, FGameplayTag EventTag, FName Channel,
                 EPEsEventPhase Phase = EPEsEventPhase::Main);

// Stop listening to everything, on every channel, at every phase.
void UnsubscribeAll(UObject* Subscriber);
```

!!! warning "Unsubscribe must name the same phase you subscribed with"
    Phase is part of the subscription key, so an `Unsubscribe` that leaves it at
    the default removes the Main-phase entry and **silently leaves an Intent-phase
    subscription in place**. If you subscribed at `Intent`, unsubscribe at
    `Intent` — or use `UnsubscribeAll`, which clears every phase.

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

### Event phases

**Phase** answers *which broadcast* you are listening to. It is a separate axis
from order: order says **how** a listener runs within one broadcast (before or
after the commit); phase says **which** broadcast it hears at all.

```cpp
enum class EPEsEventPhase : uint8
{
    Main,     // the main window: interrupt phase, commit, reaction phase
    Intent,   // the earlier declaration broadcast, before any proposal exists
};
```

- **`Main`** is the default everywhere — on subscribe, on unsubscribe, on every
  broadcast entry point, and on every query. A game that never mentions phase
  behaves exactly as it did before the axis existed.
- **`Intent`** is an earlier broadcast that fires *before* a proposal or a
  transaction bracket exists — the moment for a rule that needs to weigh in on
  something being *about* to be attempted, and the only phase where a listener may
  legitimately suspend and wait (a "do you want to counterattack?" prompt) before
  the change is even shaped.

Because phase is part of the subscription key, the two phases are genuinely
separate populations: an Intent-only subscriber never makes a Main-phase query
report subscribers, and vice versa.

!!! warning "Blueprint cannot reach the intent phase"
    The **Subscribe to Event** node — the Blueprint path, and the only one that
    writes the tag-named handler for you — has **no Phase pin**. Neither does the
    **Broadcast Event** node. A Blueprint subscription is always `Main` and always
    an observer, and a Blueprint broadcast is always `Main`.

    So all three of these are **C++-only**: broadcasting at `Intent`, subscribing at
    `Intent`, and subscribing authoritatively (`bTransient = false`, the
    rules-listener population). The subsystem's raw `Subscribe` function is
    Blueprint-callable and does expose the phase and transient parameters, but it
    does not generate the tag-named handler the node writes for you, so it is not a
    practical Blueprint route. If your design needs an intent-phase listener, plan
    on C++ — or on the entity-carried trigger surface, which is the framework's
    durable answer for outcome-affecting rules.

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
These are **C++-only**, and each answers for one phase.

```cpp
// True if a broadcast of (EventTag, Channel, Phase) would reach anyone at all —
// observer or rules listener. Includes global-channel subscribers (see below).
bool HasSubscribers(FGameplayTag EventTag, FName Channel,
                    EPEsEventPhase Phase = EPEsEventPhase::Main) const;

// True only if it would reach at least one RULES LISTENER. The windowed-change
// entry points use this so a pure observer never forces a reaction window open.
// Also global-channel-aware.
bool HasNonTransientSubscribers(FGameplayTag EventTag, FName Channel,
                                EPEsEventPhase Phase = EPEsEventPhase::Main) const;

// The stored list for exactly (EventTag, Channel, Phase) — store truth, and
// deliberately NOT global-channel-aware. For debugging and tests, not decisions.
const FPEsSubscriptionList* GetSubscribers(FGameplayTag EventTag, FName Channel,
                                           EPEsEventPhase Phase = EPEsEventPhase::Main) const;

// The list a broadcast would actually deliver to: the channel's subscribers merged
// with the global channel's, in final dispatch order, deduplicated by subscriber id
// (a listener on both keys is called once). True when the list is non-empty.
bool BuildDispatchList(FGameplayTag EventTag, FName Channel,
                       TArray<FPEsSubscriberEntry>& OutEntries,
                       EPEsEventPhase Phase = EPEsEventPhase::Main) const;
```

!!! note "Two of these answer for the union, one for the store"
    `HasSubscribers` and `HasNonTransientSubscribers` answer *"would a broadcast
    reach anyone?"*, so they count global-channel subscribers too — that is what
    makes any cheap "nobody is listening, skip the work" exit honest.
    `GetSubscribers` deliberately does not: it reports exactly what is stored under
    one key, which is what you want when you are debugging *where* a subscription
    landed. When you want the real delivery list, use `BuildDispatchList`.

Blueprint-facing debug accessors:

```cpp
// Every (EventTag, Channel, Phase) key currently in the store.
TArray<FPEsSubscriptionKey> GetAllSubscriptionKeys() const;

// The sorted subscriber entries stored for one event+channel, MAIN PHASE ONLY —
// this debug accessor has no phase parameter. For an intent-phase list, or for the
// list a broadcast really delivers to, use BuildDispatchList in C++.
TArray<FPEsSubscriberEntry> GetSubscriberEntries(FGameplayTag EventTag, FName Channel) const;

// A display string ("ClassName (ObjectName)") for a subscriber id, or the raw id.
FString GetSubscriberDisplayName(FName SubscriberId) const;
```

`FPEsSubscriptionKey` names one event on one channel; `FPEsSubscriberEntry`
describes one listener in the sorted list:

```cpp
struct FPEsSubscriptionKey   // BlueprintType
{
    FGameplayTag        EventTag;
    FName               Channel;
    EPEsEventPhase      Phase = EPEsEventPhase::Main;   // part of the key
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
    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate,
    EPEsEventPhase Phase = EPEsEventPhase::Main);

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
    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate,
    EPEsEventPhase Phase = EPEsEventPhase::Main);

// Channel is a raw FName — used by the entity window channels, where the channel is
// derived from an entity rather than a tag or object. C++ only.
static UPEsBroadcastHandle* BeginBroadcastEventOnNamedChannel(
    FGameplayTag EventTag, FName Channel,
    const FInstancedStruct& Payload, const FGameplayTagContainer& Tags,
    const TMap<FGameplayTag, float>& Stats, UObject* Caller,
    EPEsBroadcastTiming Timing = EPEsBroadcastTiming::Immediate,
    EPEsEventPhase Phase = EPEsEventPhase::Main);
```

A broadcast reaches the subscribers registered for *its* phase, and only those.

!!! note "C++ only: the intent phase, on both ends"
    The Blueprint **Broadcast Event** node has no Phase pin either, so a Blueprint
    broadcast is always `Main`. Between that and the subscribe node, the intent
    phase is **entirely a C++ surface** — you can neither raise one nor listen to
    one from Blueprint.

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

### The global channel

One channel is reserved: **`Channel.Global`**. It is the framework's answer to
"I want to hear *everything*" — a combat log, an analytics tap, a debug overlay
that should not have to enumerate every channel in the game.

```cpp
// The reserved global channel's name. C++-only; in Blueprint use the
// Channel.Global gameplay tag directly on the subscribe node.
static FName UPEsEventSubsystem::GetGlobalTopicName();
```

It is a **receive-side mirror**, and that asymmetry is the whole thing to
understand:

- **Subscribing to `Channel.Global`** means every broadcast on *any* channel is
  additionally delivered to you. Subscribing is the entire opt-in; broadcasters do
  nothing and know nothing about it.
- **Broadcasting *to* `Channel.Global`** does **not** fan out to every channel. It
  reaches only the listeners who subscribed to `Channel.Global` itself, exactly
  like a broadcast on any other channel.

!!! warning "The global channel is not a send-to-everyone"
    Reading the name as "broadcast here and everyone hears it" is the mistake this
    channel invites, and it fails quietly: your event reaches the handful of global
    listeners and nobody on the channel they actually care about. If you want an
    event heard broadly, broadcast it on the channel that describes it and let
    interested parties subscribe — including, if they wish, via the global mirror.

The mirror respects the other two key axes. It merges **per phase**: a global
subscriber registered at `Main` is folded only into `Main` dispatches. And it
deduplicates by subscriber id, so a listener subscribed to *both* a specific
channel and the global channel is still called exactly once per broadcast, at its
earliest-sorting position. Ordering is the single comparator applied to the merged
list; an exact tie puts the channel's own entry ahead of the global one.

The order-preset map and the two safety caps are authored in **Project Settings
&rarr; Plugins &rarr; Event System** (`UPEsEventSettings`):

- **`OrderPresets`** — the `EventOrder.* → int` map that `ResolveOrderPreset` reads.
- **`DefaultSiteRepetitionCap`** — the loop terminator (see
  [Loop and depth safety](#loop-and-depth-safety)).
- **`MaxNestedBroadcastDepth`** — the recursion backstop for Immediate nesting.

!!! warning "A preset can never buy interrupt capability — keep them all >= 0"
    It is tempting to author a negative order preset ("Interrupt.Early = -100") and
    expect a subscriber using it to run pre-commit. It cannot. The preset map is
    read only by the ad-hoc subscribe path, which produces **observers**, and an
    observer is clamped out of the interrupt phase by construction — so a negative
    preset value is rewritten to `0` with a warning and never reaches the store.
    The listener still runs; it just runs where every observer runs, after every
    rule.

    Keep every shipped preset at zero or above, and treat presets as what they are:
    a way for designers to sequence *reactions* by name. Interrupt-phase
    subscription comes only from the authoritative path, which does not consult the
    preset map at all.

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
    FGameplayTag   EventTag;   // what happened
    FName          Channel;    // where
    UObject*       Caller;     // who raised it
    EPEsEventPhase Phase;      // which broadcast this is (Main or Intent)
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
the cap it stops that listener and raises a hook. The count is kept per
listener **per phase**, so the same subscriber hearing both an intent and a main
broadcast of one event burns two independent budgets rather than one shared one.

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
// vetoes, use an entity-carried trigger (see the AbilitySystem reference).
void SubscribeToEntityWindow(UObject* Subscriber, const FPGeEntityRef& Entity,
                             FGameplayTag EventTag, int32 Order = 0);
void UnsubscribeFromEntityWindow(UObject* Subscriber, const FPGeEntityRef& Entity,
                                 FGameplayTag EventTag);
```

The entity-carried listener surface that *can* veto or reshape a window is the
AbilitySystem plugin's [Authoring triggers](../abilitysystem/reference-triggers.md#authoring-triggers).

The richer-return C++ siblings of these two calls — the ones that tell you whether
a change actually committed, and whether it is still waiting to be judged — are
documented with the entity system, in
[Windowed stat and tag changes](../gameentity/reference-stats.md#windowed-stat-and-tag-changes).

### The window payload

Every windowed change broadcasts a typed struct recording the immutable facts of
the proposal. All of them share one base, so a reactor reads *who* and *to whom*
once, whatever kind of change opened the window:

```cpp
struct FPGeWindowPayloadBase   // BlueprintType, read-only
{
    FPGeEntityRef Target;       // the entity taking the change (invalid for a
                                // window whose subject is not an entity)
    FPGeEntityRef Instigator;   // the cause — attacker, trap, aura (may be invalid)
    // plus a state anchor the framework stamps for you; never set it yourself
};

struct FPGeStatChangeWindowPayload : public FPGeWindowPayloadBase
{
    FGameplayTag StatTag;        // the changed stat (also the event tag)
    int64        OriginalDelta;  // the delta as proposed, before any interrupt shaped it
};

struct FPGeTagChangeWindowPayload : public FPGeWindowPayloadBase
{
    FGameplayTag Tag;            // the changed tag (also the event tag)
    bool         bAdd = true;    // true = adding, false = removing
};
```

Note the **duality** on a stat window: the immutable original delta rides the
typed payload as an `int64`, while the *live, reshapeable* magnitude rides the
broadcast payload's `Stats` map as a float under the stat's tag. An interrupt
shapes the float; the `int64` is the record of what was originally proposed.

### Opening a window of your own (C++)

If you are writing a plugin or a system with its own kind of authoritative change —
not a stat, not a tag — you can open a real reaction window around it instead of
inventing a parallel mechanism. Two C++-only entry points on the game-state
subsystem wrap a change of yours in the same machinery:

```cpp
// Open a veto-only window on EntityRef's own channel, keyed by EventTag, around
// CommitAction. C++ only — there is no Blueprint node for this.
bool OpenWindowedChange(const FPGeEntityRef& EntityRef, FGameplayTag EventTag,
                        const FInstancedStruct& WindowPayload,
                        const FPGeEntityRef& Instigator,
                        TFunction<void()> CommitAction,
                        bool& bOutCommitted, uint64& OutHeldEpisodeId);

// The same window on an arbitrary named channel, for a change whose subject is not
// an entity at all (a structural edit to the board, say). C++ only.
bool OpenWindowedChangeOnChannel(FName Channel, FGameplayTag EventTag,
                                 const FInstancedStruct& WindowPayload,
                                 const FPGeEntityRef& Instigator,
                                 TFunction<void()> CommitAction,
                                 bool& bOutCommitted, uint64& OutHeldEpisodeId);
```

The shape is the same one `Apply Stat Change` uses: interrupt subscribers run
first and may **veto** (there is no magnitude to shape here), `CommitAction` runs
only if nobody vetoed, and reaction subscribers run afterwards against committed
state. With no rules listener on that channel there is a fast path that just runs
`CommitAction` directly, so an unwatched window costs what the raw write costs.

Three rules make it work:

- **`CommitAction` must be command-routed.** That is what folds your change into
  the window's transaction so the whole cascade undoes as one. It runs at most
  once, and it must be safe to re-enter. A **no-op `CommitAction` is legitimate**
  — a window that only *declares* an intention has nothing to commit.
- **Derive your payload from `FPGeWindowPayloadBase`.** The framework fills in the
  identity fields and stamps the state anchor reactors need. A payload that
  doesn't derive from it still opens its window, but its reactors lose that
  anchoring.
- **These are for genuine framework windows,** not a general "make any write
  windowed" facility. Reach for one when a change of yours deserves the same
  veto-and-react treatment a stat change gets.

Both return `false` only for an invalid entity, tag or channel, an unset
`CommitAction`, or an unavailable command stack.

### When a change has not been judged yet

`OutHeldEpisodeId` is how a window reports that its answer is not ready.

- **Zero** means the change settled synchronously: by the time the call returned,
  it had been committed, vetoed, or fizzled.
- **Non-zero** means the change is **held**. A listener in the intent phase
  suspended — it is asking the player something, or waiting on its own work — so
  the change has not committed and will commit (or not) later. The value is the
  identity of the episode holding it, and that episode closes exactly when the
  change resolves.

A caller that must wait registers on it:

```cpp
// Resume when the held change finally resolves. C++-only.
UPEsEventSubsystem::Get(this)->RegisterEpisodeCloseCallback(HeldEpisodeId,
    [this]{ ContinueAfterTheChangeResolved(); });
```

Reporting an episode rather than a bare "it's pending" bool is what lets the
waiter be any depth of caller — you do not have to be the code that opened the
window to wait on it.

!!! warning "Not-committed does not mean vetoed"
    The stat window's richer C++ variant also reports `bOutResolved`, and reading
    it is not optional. `bOutResolved == false` means the window's work was still
    queued when the call returned — the change has simply **not been judged yet**,
    and will commit shortly with nobody waiting on it.

    So **only `!bOutCommitted && bOutResolved` is a veto.** Treating a merely
    unresolved change as a rejected one is the bug this flag exists to prevent: an
    effect's result set would silently depend on whether unrelated content happened
    to have an intent-phase listener registered. Check `bOutResolved` before you
    conclude a target was unaffected. Note that an unresolved change is *not* a
    held one — there is no episode to wait on, because nothing is holding it.

!!! warning "Windowed changes are forward-only"
    Never call `ApplyStatChange` / `ApplyTagChange` from a command's apply/undo path
    or from a change-event listener — those are read-only by construction. A windowed
    change is a live gameplay action, and its whole reaction cascade folds into one
    undo step. The plain, window-blind changes (Modify Stat / Add Tag) live with
    [GameEntity](../gameentity/reference-stats.md); their change events are in the
    [change-events reference](../gameentity/reference-events.md).
