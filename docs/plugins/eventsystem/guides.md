# EventSystem Guides

For developers with the plugin enabled who want to get something done. Each
section is a short, followable recipe. For the full type-by-type surface see the
[API Reference](reference.md); for the model behind it all see
[Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md).

## Broadcast an event

Announcing that something happened is one node in Blueprint, or a two-step
Begin/Finish pair in C++. If nothing is listening on that event and channel, the
broadcast costs almost nothing — it checks for subscribers and exits.

=== "Blueprint"
    1. Drop the **Broadcast Event** node.
    2. Set **Event Tag** to what happened — `Event.Trap.Sprung`.
    3. Set **Channel** to where it should be heard. Leave it on the default shared
       channel for "anyone interested," or point it at one entity's own channel to
       address a single unit.
    4. Fill the **Payload** — when the event tag is a literal mapped in your data
       schema, the pin becomes that typed struct; otherwise it's a generic
       instanced struct. Optionally add convenience **Tags** and named **Stats**
       (numbers) for small facts.
    5. Leave **Timing** on **Immediate** so subscribers resolve before execution
       continues, or set **Deferred** to let the current work finish first.

    The node has three exec outputs: **Then** continues immediately, while **After
    Interrupts** and **After All** fire once the corresponding phase of subscribers
    has completed.

=== "C++"
    ```cpp
    // Announce "a trap sprung" on a shared channel. Begin/Finish bracket the send;
    // the two-step shape lets you bind phase delegates in between.
    UPEsBroadcastHandle* Handle = UPEsEventLibrary::BeginBroadcastEvent(
        TrapSprungTag,                 // what happened
        ChannelTag,                    // where — a shared channel here
        FInstancedStruct::Make(FTrapSprungInfo{ TriggeringCell }),
        /*Tags=*/{}, /*Stats=*/{},     // convenience tags and named numbers
        this,                          // the caller
        EPEsBroadcastTiming::Immediate);

    // Optional: react to a phase completing before you queue the calls.
    Handle->OnAfterAll.AddDynamic(this, &UMyRules::OnTrapResolved);  // handler declared UFUNCTION()

    UPEsEventLibrary::FinishBroadcastEvent(Handle);
    ```

    `BeginBroadcastEvent` returns a handle with `bHasSubscribers == false` when
    nobody is listening, so you can skip the work entirely. A `...ByObject`
    variant takes a live `UObject` as the channel and resolves its id for you.

!!! tip "Immediate vs Deferred"
    An **Immediate** broadcast resolves depth-first: the rules that answer it run
    to completion before the code that raised it moves on. A **Deferred** broadcast
    is appended to the end of the queue and runs after the current work finishes.
    Immediate is the default and is what you want when you need the outcome before
    the next line.

## Observe an event

An observer is a listener that only wants to *know* something happened — a HUD
line, a sound, an analytics tap. It runs after every rule and cannot change
anything. This is the default subscribe path.

=== "Blueprint"
    1. Drop the **Subscribe to Event** node, usually in a widget's construct or an
       actor's begin-play.
    2. Set **Subscriber** to self (the default), **Event Tag** to the event you
       care about, and **Channel** to match the broadcast.
    3. Pick an **Order Preset** (see [Order reactions](#order-reactions-with-presets)),
       or leave it empty and set a raw **Order**. For an observer the order only
       decides its place among *other* observers — it always runs after every rule.
    4. The node generates a handler event for you; read the payload there and do
       your presentation work.

=== "C++"
    ```cpp
    // A HUD widget listens on the same channel. bTransient defaults to true, so
    // this is a pure observer: dispatched after every rule, and unable to veto or
    // reshape anything.
    UPEsEventSubsystem::Get(this)->Subscribe(
        this,                          // the subscriber
        TrapSprungTag,                 // the event tag
        ChannelTag.GetTagName());      // channel as an FName (see note)
    ```

    The subscriber's handler is a `UFUNCTION` bound by the event tag — the same
    mechanism the Blueprint node uses. See
    [Subscribing to events](reference.md#subscribing-to-events) for how the handler
    is named and how to give a subscriber a save-stable id.

!!! note "Tag channels and FName channels"
    The Blueprint subscribe node takes the channel as a gameplay **tag**; the C++
    `Subscribe` takes it as an **FName**. Bridge them with `Channel.GetTagName()`.
    An entity's own channel is an FName you fetch from
    `UPGeGameStateSubsystem::GetEntityWindowChannel(Entity)`.

## React to a change with a reaction window

This is the recipe for a rule that *changes* an outcome — armor that halves an
incoming hit, an immunity that vetoes a status. The key idea from the concept
page: the window belongs to *taking* the change, not to its cause. A spell, a
trap, and a hazard that all deal damage open the same window on the target, keyed
by the same stat — so "armor softens incoming damage" is authored **once**, on
the defender, and every source of damage flows through it.

A windowed change has two phases:

- **Interrupts** run *before* the change commits, in-line. They reshape the
  proposed amount, or cancel it outright. They finish instantly.
- **Reactions** run *after* the change commits and see the new, real state. They
  answer with changes of their own — "when damaged, retaliate."

Order decides the phase: a **negative** order is the interrupt phase, **zero and
positive** the reaction phase.

### The cause: open the window

Where you would make an unconditional change, make the windowed one instead.

=== "Blueprint"
    Blueprint authors the *cause* and can *observe* the window; it does not write
    interrupts.

    1. Where you would call **Modify Stat** for a plain change, call **Apply Stat
       Change** instead — target, `Stat.Health`, `-10`, and the instigator. This
       opens the reaction window on the target.
    2. To *watch* the window from Blueprint, drop **Subscribe to Entity Window** on
       the target for the `Stat.Health` tag and raise a "damage incoming" banner.
       This is an observer: it sees the proposal but cannot change the number.

    Authoring Blueprint interrupts and triggered reactions — the rules that
    actually reshape or answer a windowed change — is part of the entity-carried
    trigger surface, documented in a later section. <!-- pluginlink: abilitysystem-guides-author-a-trigger -->

=== "C++"
    ```cpp
    // The cause. A spell, a trap, and a hazard all call this — the window is on
    // TAKING the change, so the armor below is authored once. With the interrupt
    // in place, a -10 proposal commits as -5. Delta is whole units (int64).
    State->ApplyStatChange(Defender, HealthTag, -10, Attacker);
    ```

### The interrupt: reshape the proposal (worked armor example)

The one hand-written path to a rules listener today is a C++ subscription at a
negative order with `bTransient = false`. That flag is the escape hatch — "I own
this listener's lifecycle myself." (The durable, entity-carried way to author
outcome-affecting listeners is documented in a later section; this is the manual
path.) <!-- pluginlink: abilitysystem-guides-author-a-trigger -->

```cpp
// On the defender, author an interrupt at order -10. A negative order puts it in
// the pre-commit phase. bTransient=false lands it among the rules listeners, where
// it can reshape the proposal. The channel is the defender's OWN channel.
UPEsEventSubsystem::Get(this)->Subscribe(
    this,
    HealthTag,                                              // the stat's tag is the event tag
    UPGeGameStateSubsystem::GetEntityWindowChannel(Defender),
    /*Order=*/-10,
    /*bTransient=*/false);
```

The handler is where armor does its work. Two things to get right:

- The proposed amount rides the payload's **numbers map** (`Stats`) under the
  stat's tag, as a **float**. Read it, halve it, and write it back; the commit
  uses the shaped value.
- The handler is a `UFUNCTION` **named by the event tag**, not a name you choose.
  The dispatch core generates the name deterministically from the tag, so a
  hand-named function never fires. See the note below.

```cpp
// Runs before the change commits. Halving the proposed -10 turns the hit into -5.
// The function name is the generated name for HealthTag (see note) — this is NOT
// an arbitrary handler name.
UFUNCTION()
void __TagEvent__Stat_Health__A1B2C3D4(UObject* Caller, UTagEventPayloadObject* Payload)
{
    const float Proposed = Payload->Stats.FindRef(HealthTag);
    Payload->Stats.Add(HealthTag, Proposed * 0.5f);   // armor softens the hit

    // To cancel the change outright instead of shaping it:
    //   UPEsEventLibrary::InterruptEvent(Payload);
}
```

!!! warning "A C++ handler is named by the event tag, not by you"
    Dispatch resolves the handler by reflection on a `UFUNCTION` whose name is
    generated from the event tag — `__TagEvent__<sanitized tag>__<checksum>`, with
    signature `(UObject* Caller, UTagEventPayloadObject* Payload)`. There is no API
    to bind an arbitrarily-named function or a plain delegate. Ask the dispatch
    core for the exact name rather than transcribing the checksum by hand:
    `UTagEventFunctionLibrary::GetGeneratedFunctionNameForTag(EventTag)` (Blueprint
    pure) or `TagEvents::MakeGeneratedFunctionName(EventTag)` in C++ — assert it
    matches your function's name at startup. This is why the durable authoring path
    is the entity-carried trigger surface, which writes the handler for you. See
    [TagEvents](../tagevents/index.md).

!!! note "Tag changes can only be vetoed"
    The stat window shapes a number. A **tag** window has no magnitude — an
    interrupt on **Apply Tag Change** may only cancel the change via
    `InterruptEvent` ("immune to poison" vetoes `Status.Poisoned` before it lands).
    The reaction phase — "when poisoned, do X" — runs after the tag commits.

!!! tip "A whole cascade is one undo step"
    A reaction can cause more windowed changes, which cause more reactions. The
    interrupt's reshaping, every reaction, and the reactions' own windowed changes
    fold into a **single** undo step: one press of Undo removes the hit *and* the
    retaliation it set in motion, together. You never peel a combat exchange apart
    one internal step at a time. The full change and change-event picture is in the
    [GameEntity change-events reference](../gameentity/reference-events.md).

## Order reactions with presets

A listener's integer order decides both its phase and its place in line: negative
is the interrupt phase, zero and positive the reaction phase, and lower values run
first within a phase. When the sequence *matters* — this shield resolves before
that thorns aura — encode it with distinct order values.

=== "Blueprint"
    Use a named **Order Preset** instead of a bare number. Presets live under the
    `EventOrder.*` gameplay-tag namespace and map to integers in **Project Settings
    &rarr; Plugins &rarr; Event System**.

    1. On the **Subscribe to Event** node, set **Order Preset** to a named preset —
       e.g. an early-interrupt preset for armor, a late-reaction preset for
       cleanup. With a literal preset chosen, the raw Order pin hides itself.
    2. To tune the actual numbers your presets resolve to, edit the **Order
       Presets** map in project settings — no Blueprint changes needed.

=== "C++"
    ```cpp
    // C++ passes the integer order directly. Negative = interrupt, >= 0 = reaction.
    UPEsEventSubsystem::Get(this)->Subscribe(Self, EventTag, Channel, /*Order=*/-10, /*bTransient=*/false);

    // To honor a designer-authored preset from C++, resolve it first:
    const int32 Order = UPEsEventLibrary::ResolveOrderPreset(ArmorPreset, /*OrderOverride=*/-10);
    UPEsEventSubsystem::Get(this)->Subscribe(Self, EventTag, Channel, Order, /*bTransient=*/false);
    ```

!!! tip "Ties are stable, but don't lean on them"
    When two rules listeners share the same order, the tie breaks by **entity age —
    the older entity acts first**. Entity age is a pure function of game state, so
    the sequence is identical in live play, after an undo, and in a replay, on every
    machine. That tiebreak exists to make equal orders *stable*, not to express
    intent — when the sequence matters, say so with distinct order values.

## Break a runaway loop

Because reactions can cause reactions, mutually-triggering rules are legal to
author: "when A is hit, hit B" together with "when B is hit, hit A." Left
unchecked that is an infinite loop. The framework does not crash and does not
silently swallow it: it counts how many times each listener fires for the same
event within one causal chain, and once a listener passes a configurable cap it
stops *that* listener and raises a hook your game answers.

=== "Blueprint"
    1. On the event subsystem, bind **On Site Repetition Cap Reached**. It fires the
       first time a listener would re-fire past the cap within one causal chain.
    2. In the handler, decide the outcome: leave the listener stopped and the loop
       fizzles, or call **Extend Site Repetition Cap** with a higher number to let
       this chain keep going. You can also turn the moment into a scripted outcome —
       a stalemate, a draw.

=== "C++"
    ```cpp
    // Bind once, e.g. at setup.
    UPEsEventSubsystem::Get(this)->OnSiteRepetitionCapReached.AddDynamic(
        this, &UMyRules::OnLoopCapped);

    // OnLoopCapped is declared UFUNCTION() in the header — dynamic delegates bind by name.
    void UMyRules::OnLoopCapped(FName SubscriberId, FGameplayTag EventTag,
                                FName Channel, int32 DispatchCount)
    {
        // Let this exchange bounce a few more times before it stops...
        UPEsEventSubsystem::Get(this)->ExtendSiteRepetitionCap(DispatchCount + 3);
        // ...or do nothing, and the loop fizzles here.
    }
    ```

!!! note "The cap is a safety net, not a balance knob"
    The default cap (a project setting) is deliberately generous — a loop that runs
    out of resources should exhaust itself long before it. Reach for **Extend Site
    Repetition Cap** only inside the cap-reached handler; the extension lasts for
    the rest of that causal chain. There is a separate, higher recursion backstop
    (`OnMaxBroadcastDepthReached`) for stack safety; see
    [Loop and depth safety](reference.md#loop-and-depth-safety).
