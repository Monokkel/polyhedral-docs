# Events, Ordering, and Reaction Windows

For developers whose game rules need to *respond* — armor that softens a hit, a passive that triggers when poison lands, a trap that fires when a unit steps on it — with responses that stay correct through undo, save, and replay. After reading, you'll know how gameplay events are broadcast and heard, how a reaction window lets a rule reshape or answer a change before and after it commits, and why the order reactions run in is never left to chance.

## Two kinds of "something happened"

The two earlier pages on reactive code both stopped at the same doorway. [Derived State and Change Events](derived-state.md) showed how a health bar or a cache reacts to state changing — and made the rule permanent that such a listener is **read-only**. [Commands and Undo](commands-and-undo.md) showed that a listener can never submit a command in response. Both pointed forward for the case they deliberately left out: rules that *change* state when something happens. This page is that mechanism.

The distinction is the whole foundation, so hold it firmly. There are two separate broadcasts, for two separate jobs:

| | Change event | Gameplay event |
|---|---|---|
| Says | "state changed — reconcile your display" | "this happened — other rules may act" |
| Who listens | derived state and presentation | game rules |
| May change state? | Never — read-only | Yes; that is the point |

A **change event** reflects what already happened, so your display can catch up. A **gameplay event** — an *event* for short — is a declared moment other rules may act on. One system reflects; the other participates. Never blur them: a change-event listener that tried to run a rule would re-drive your game during an undo. Rules live here instead.

!!! note "If you've used GAS"
    A gameplay event replaces the ad-hoc delegate and callback chains you would otherwise wire by hand between systems. The win here is not familiarity — it is that the order responders run in is deterministic, and a whole chain of reactions undoes as one step.

## An event is a tag on a channel

A gameplay event has three parts:

- A **gameplay tag** names *what* happened — `Event.Trap.Sprung`.
- A **channel** names *where* it was heard. A channel is either a shared channel that any interested rule can watch, or one specific entity's own channel — "this unit took damage" is broadcast on that unit's channel, so a rule can listen to *one* unit without hearing every unit in the game.
- A **payload** carries the facts: a typed struct, plus a bag of convenience tags and named numbers for the small stuff.

Subscribing, then, is three things: which event tag, which channel, and an **order** (covered below). Here is a trap announcing itself, and a HUD line listening for it.

=== "Blueprint"
    1. Drop the **Broadcast Event** node. Set its Event Tag to `Event.Trap.Sprung`, leave Channel on the default shared channel, and fill in the payload struct (which cell, which trap).
    2. On a HUD widget, drop **Subscribe to Event** for the same `Event.Trap.Sprung` on the same channel. Its handler reads the payload and prints a line to the combat log.

    That HUD subscription is an *observer* — the next section explains why that matters.

=== "C++"
    ```cpp
    // Announce "a trap sprung" on a shared channel. Begin/Finish bracket the
    // send; the two-step shape lets extra delegates bind in between.
    UPEsBroadcastHandle* Handle = UPEsEventLibrary::BeginBroadcastEvent(
        TrapSprungTag,               // what happened
        ChannelTag,                  // where — a shared channel here
        FInstancedStruct::Make(FTrapSprungInfo{ TriggeringCell }),
        /*Tags=*/{}, /*Stats=*/{},   // convenience tags and named numbers
        this);
    UPEsEventLibrary::FinishBroadcastEvent(Handle);

    // A HUD widget listens on the same channel. bTransient defaults to true,
    // so this subscription is a pure observer (next section).
    UPEsEventSubsystem::Get(this)->Subscribe(
        this, TrapSprungTag, ChannelTag.GetTagName());
    ```

    The subscriber is a `UObject` that returns a stable id (via `IPEsEventSubscriberInterface`); the framework calls a handler on it named for the event tag. In Blueprint the **Subscribe to Event** node writes that handler for you.

A broadcast can resolve immediately and depth-first — the rules that answer it run to completion before the code that raised it moves on — or be queued to run after the current work finishes. That choice is a timing flag on the broadcast; the default is immediate.

For the full subscribe-and-broadcast API — channels, order, the payload, and timing — see the [EventSystem reference](../plugins/eventsystem/reference.md#subscribing-to-events).

## Observers and rules listeners are different populations

This is the most important split on the page, and it is not a naming quibble — the two kinds of listener live in different places and can do different things.

The ordinary subscribe path — and *every* Blueprint subscribe node — creates an **observer**. An observer is told what happened *after* the rules have finished, always runs after every rule, and cannot cancel or alter anything. That is exactly right for UI, sound, and logging: a health bar, a "trap sprung!" toast, an analytics line. It reflects the outcome; it does not shape it.

A **rules listener** — one that can change or block what happens — is a different animal. It is part of game state itself: the listening lives on an [entity](entities-as-data.md) as data, so *who is listening* undoes, saves, and replays with the world, exactly like the unit's health does. You don't *register* a rules listener at runtime the way you hook up a UI widget — you *author* one onto an entity, and the framework derives the live subscriptions from that entity state.

!!! warning "Observers can never affect the outcome"
    If your listener only needs to *know* something happened, an ordinary subscription is perfect. But if it needs to block or reshape what happens — halve incoming damage, veto a status — an ad-hoc subscription cannot do it, no matter what order you give it. Observers are dispatched after every rule and are locked out of the pre-commit phase by construction. Outcome-affecting rules are authored as entity-carried triggers; that full authoring surface is documented in [Abilities and step-by-step resolution](abilities-and-resolution.md#triggers-entity-carried-rules). The next section shows the one hand-written path that exists today.

## Reaction windows

Here is where a rule reshapes an outcome. Some changes are **windowed**: instead of applying unconditionally, they open a moment for rules to respond. The stat and tag changes from earlier pages come in both flavours — the direct **Modify Stat** / **Add Tag** apply with no window, while their windowed siblings **Apply Stat Change** and **Apply Tag Change** open one. A window has two phases:

- **Interrupts** run *before* the change commits, in-line. They can reshape the proposed amount — armor halves the incoming damage — or cancel the change outright — immune-to-poison vetoes the tag before it lands. They finish instantly, before anything else happens.
- **Reactions** run *after* the change commits. They see the new, real state and answer with changes of their own — "when damaged, retaliate."

The crucial design property: **the window belongs to *taking* the change, not to the cause.** A spell, a trap, and an environmental hazard that all deal damage open the same window on the target, keyed by the same stat. So "armor softens incoming damage" is authored *once*, on the defender, and every source of damage flows through it. If nothing is listening on that entity for that stat, a windowed change costs exactly what a direct one costs — the window is free until someone uses it.

An interrupt reads the proposed amount off the event's payload, reshapes it, and writes it back; the commit then uses the shaped value.

=== "Blueprint"
    Blueprint authors the *cause* and *observes* the window; it does not write interrupts.

    1. Where you would call **Modify Stat** for an unconditional change, call **Apply Stat Change** instead — target, `Stat.Health`, `-10`, and the instigator. This opens the reaction window on the target.
    2. To *watch* that window from Blueprint, drop **Subscribe to Entity Window** on the target for the `Stat.Health` tag and raise a "Damage incoming" banner. This is an observer: it sees the proposal but cannot change the number.

    Authoring Blueprint interrupts and triggered reactions — the rules that actually reshape or answer a windowed change — is part of the ability system's trigger surface, documented in [Abilities and step-by-step resolution](abilities-and-resolution.md#triggers-entity-carried-rules).

=== "C++"
    ```cpp
    // On the defender, author an interrupt at order -10. A negative order puts it
    // in the pre-commit phase. bTransient=false is the escape hatch — "I own this
    // listener's lifecycle myself." (The framework's durable way to author
    // outcome-affecting listeners, entity-carried triggers, is documented in a
    // later section; this is the one hand-written path today.)
    UPEsEventSubsystem::Get(this)->Subscribe(
        this,
        HealthTag,                                            // the stat's tag is the event tag
        UPGeGameStateSubsystem::GetEntityWindowChannel(Defender), // the defender's own channel
        /*Order=*/-10,
        /*bTransient=*/false);
    ```

    ```cpp
    // Runs before the change commits. The proposed delta rides the payload's
    // numbers map under the stat's tag; halving it turns a -10 hit into -5.
    // A C++ handler's NAME is generated from the event tag, not chosen by you —
    // this signature is illustrative; the EventSystem guides show how it's named
    // (and the trigger surface that writes it for you).
    void UArmor::HandleHealthWindow(UObject* Caller, UTagEventPayloadObject* Payload)
    {
        const float Proposed = Payload->Stats.FindRef(HealthTag);
        Payload->Stats.Add(HealthTag, Proposed * 0.5f);   // armor softens the hit
        // To cancel the change outright instead:
        //   UPEsEventLibrary::InterruptEvent(Payload);
    }
    ```

    ```cpp
    // The cause. A spell, a trap, and a hazard all call this — the window is on
    // TAKING the change, so the armor above is authored once. With that interrupt
    // in place, a -10 proposal commits as -5.
    State->ApplyStatChange(Defender, HealthTag, -10, Attacker);
    ```

Apply Stat Change is the windowed sibling of the direct base-stat change you met on [Stats and Modifiers](stats-and-modifiers.md#two-layers-base-and-current) — same change, now with a moment for rules to respond around it.

For the step-by-step how-to — opening a window with Apply Stat Change and shaping it with an interrupt — see [React to a change with a reaction window](../plugins/eventsystem/guides.md#react-to-a-change-with-a-reaction-window) in the EventSystem guides.

## Ordering is deterministic — always

A listener's integer **order** decides both its phase and its place in line:

- **Negative** order is the interrupt phase — before the change commits.
- **Zero and positive** is the reaction phase — after the change commits.
- **Observers** run after everything above, always last.

Within a phase, lower orders run first. When two rules-carrying listeners share the same order, the tie breaks by **entity age — the older entity acts first**. Entity age is a pure function of game state, so the tiebreak is too: the same board reacts in the same sequence in live play, after an undo, and in a replay, on every machine. There is no hidden dependence on load order or object addresses.

!!! tip "Order values are the design knob; the tiebreak is only for stability"
    When the sequence of reactions *matters* to your design — this shield resolves before that thorns aura — say so by giving the listeners distinct order values. The entity-age tiebreak exists to make otherwise-equal ties *stable* and replay-safe, not to express intent. Don't lean on "the older unit happens to go first" as a rule; encode the rule in the orders.

## A cascade is one undo step

Reactions can cause more windowed changes, which can cause more reactions. A hit triggers a retaliation; the retaliation is itself a windowed change that a third rule answers. The whole chain — the interrupt's reshaping, every reaction, and the reactions' own windowed changes — folds into a single undo step. One press of **Undo** removes the hit *and* the retaliation *and* everything they set in motion, together, because the framework brackets the entire cascade as one [transaction](commands-and-undo.md#transactions-one-undo-step-for-a-whole-move). You never peel a combat exchange apart one internal step at a time.

## Loops end by rule, not by crash

Because reactions can cause reactions, mutually-triggering rules are legal to author: "when A is hit, hit B" together with "when B is hit, hit A." Left unchecked that is an infinite loop. The framework does not crash and does not silently swallow it. It counts how many times each listener fires for the same event within one causal chain, and once a listener passes a configurable cap, it stops that listener and raises a hook your game answers — let the loop continue this once, let it fizzle, or turn it into a scripted outcome (a stalemate, a draw). An infinite loop becomes a designed knob.

=== "Blueprint"
    1. On the event subsystem, bind **On Site Repetition Cap Reached** — it fires the first time a listener would re-fire past the cap within one causal chain.
    2. In the handler, decide the outcome: leave the listener stopped (the loop fizzles), or call **Extend Site Repetition Cap** with a higher number to let this chain keep going.

=== "C++"
    ```cpp
    // Bind once, e.g. at setup.
    UPEsEventSubsystem::Get(this)->OnSiteRepetitionCapReached.AddDynamic(
        this, &UMyRules::OnLoopCapped);

    // OnLoopCapped is declared UFUNCTION() in the header — dynamic delegates bind by name.
    void UMyRules::OnLoopCapped(FName SubscriberId, FGameplayTag EventTag,
                                FName Channel, int32 DispatchCount)
    {
        // Let this exchange bounce a few more times before it stops.
        UPEsEventSubsystem::Get(this)->ExtendSiteRepetitionCap(DispatchCount + 3);
        // Or do nothing, and the loop fizzles here.
    }
    ```

## Where this goes next

Two systems build directly on this machinery. Turn boundaries — "at the start of my turn," "at end of round" — are gameplay events broadcast on exactly the channels and phases described here; see [Turns and Scheduling](turns-and-scheduling.md). And the durable way to author entity-carried rules listeners — triggers that ride on a unit, an item, or an ability and undo and save with it — is the ability system's trigger surface, documented in [Abilities and step-by-step resolution](abilities-and-resolution.md#triggers-entity-carried-rules). Everything on that surface resolves down to the events, windows, and ordering on this page.
