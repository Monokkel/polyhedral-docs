# Triggers & Reactions

For developers authoring the rules that *answer* an action — armor that softens a
blow, a passive that hits back when its owner is struck, a counterspell that stops a
cast before it lands. After reading, you'll know how to author a **trigger** as data
on an entity, how the sign of a trigger's order decides whether it reshapes a change
*before* it commits or answers *after*, and how a **declaration window** lets a player
respond before an effect resolves.

This page is the mechanics behind the concept page's
[entity-carried triggers](../../concepts/abilities-and-resolution.md#triggers-entity-carried-rules).
It is also the durable authoring surface the events model pointed forward to: an
ordinary event subscription creates a read-only
[observer](../../concepts/events-and-reactions.md#observers-and-rules-listeners-are-different-populations),
but a rule that *changes or blocks* what happens is authored as a trigger instead.
Everything on this page resolves down to the events, reaction windows, and
deterministic ordering documented in the
[EventSystem reference](../eventsystem/reference.md#windowed-changes-entity-system) —
read that first if the window vocabulary here is new.

Signatures below are hand-written to show the shape of each API; they are
illustrative, not source excerpts.

## Authoring triggers

A trigger is not code and not a runtime registration — it is a **data entry an entity
carries**, under the `Data.Ability.Triggers` tagged-data key. Its value is a list of
trigger specs, so one entity can carry several triggers at once. Each spec answers a
handful of questions:

| Field | What it says |
|---|---|
| **Event Tag** | Which window to listen on — a stat or tag for a native change window (`Stat.Health`), or an `Event.Window.*` moment for a [declaration window](#declaration-windows). |
| **Channel** | *Whose* window: **Self** (my own — "when I take damage"), **Parent** (my owner's — how a granted buff reacts for the unit it sits on), or a window **resolved from a target source** (my summoner, the party leader). |
| **Order** | An integer whose **sign picks the phase** and whose magnitude breaks ties — see [Interrupt vs reaction](#interrupt-vs-reaction-the-orders-sign). |
| **Phase** | *Which broadcast* of that window to subscribe to — **Main** (the default: everything else on this page) or **Intent** — see [The intent phase](#the-intent-phase). |
| **Condition** | An optional gate evaluated when the trigger fires. Empty means it always fires. |
| **Auto Target** | What the reacting program starts aimed at — nothing, the entity that *caused* the change, the entity that *took* it (the default), or the reactor itself. |
| **Speculation Policy** | What a preview or an AI trial should *assume* this trigger does, when answering it would need input from another player — see [What a preview assumes](#what-a-preview-assumes-about-your-trigger). |
| **Program Override** | The [ability program](reference-programs.md) to run when it fires. Leave it unset to run the entity's own `Data.Ability.Program`; set it so a passive can react with something other than its active use. |

Because the trigger is entity data, **who is listening undoes, saves, and replays with
the world** — the events page's promise that
[rules listeners are part of game state](../../concepts/events-and-reactions.md#observers-and-rules-listeners-are-different-populations),
made concrete. Grant the entity that carries the trigger and the listener appears;
revoke it and the listener is gone. Both are ordinary, undoable entity changes.

=== "Blueprint"
    Author the trigger on the entity that should *react* — an armor item, a passive, a
    status effect — right on its data-table row:

    1. Add a **Tagged Data** entry keyed `Data.Ability.Triggers`.
    2. Add an element to its **Triggers** array — one element per trigger.
    3. Fill the element in: set **Event Tag** to the stat/tag or `Event.Window.*` moment
       to watch, **Channel** to **Self** / **Parent** / the target-source case, **Order**
       (its sign picks the phase — see below), an optional **Condition**, an **Auto
       Target** mode, and a **Program Override**.
    4. Grant that entity to a unit with the ordinary hierarchy call. The subscription
       comes to life from the granted state — there is no separate "register" step.

=== "C++"
    ```cpp
    // The spec. Every field is editable in Blueprint too — a trigger is pure data.
    struct FPAbTriggerSpec   // DisplayName "Ability Trigger Spec"
    {
        FGameplayTag                         EventTag;        // the window to listen on
        EPAbTriggerChannel                   Channel;         // whose window
        UPAbSourceAccessor*                  ChannelAccessor; // resolves the entity for the target-source channel
        int32                                Order;           // sign picks the phase; magnitude is the tiebreak
        EPEsEventPhase                       Phase;           // Main (default) or Intent
        TInstancedStruct<FEvalConditionCalc> Condition;       // optional gate; empty = always fires
        EPAbTriggerAutoTarget                AutoTarget;       // what the reacting program starts aimed at
        EPAbSpeculationPolicy                SpeculationPolicy;// what a preview assumes this trigger does
        TSoftClassPtr<UPAbAbilityProgram>    ProgramOverride;  // run this instead of the entity's own program
    };

    // The value stored under Data.Ability.Triggers. One entity, many triggers.
    struct FPAbTriggerList   // DisplayName "Ability Trigger List"
    {
        TArray<FPAbTriggerSpec> Triggers;
    };
    ```

    The channel choices and the auto-target seeds are small enums:

    ```cpp
    enum class EPAbTriggerChannel : uint8
    {
        Self,               // my own window — "when I take damage"
        Parent,             // my owner's window — a granted buff reacting for its unit
        EntityFromAccessor, // a window resolved by a target source — "my summoner"
    };

    enum class EPAbTriggerAutoTarget : uint8
    {
        None,           // start empty — the program's first step finds its own targets
        PayloadSource,  // start aimed at whoever CAUSED the change
        PayloadTarget,  // start aimed at whoever TOOK the change (the default)
        Reactor,        // start aimed at the entity carrying the trigger
    };
    ```

!!! note "The `EntityFromAccessor` channel uses a target source"
    When the channel is the target-source case, the entity whose window to watch is
    resolved by a **target source** — the same modular sourcing atoms an ability uses to
    find targets, documented on
    [Targeting & running abilities](reference-targeting.md). The channel is resolved when
    the trigger is granted (and again if the carrier's owner changes); a source used here
    must read game state only, never the live Actor world.

## Interrupt vs reaction: the order's sign

A trigger's **Order** is not just a priority number. Its *sign* selects which of a
[reaction window's two phases](../../concepts/events-and-reactions.md#reaction-windows)
the trigger runs in; its *magnitude* is only the tiebreak that orders triggers within
the same phase.

- **Negative order is an interrupt.** It runs *before* the change commits and can
  reshape the proposed amount — armor halving incoming damage — or veto it outright.
  An interrupt **must run without pausing**: its program cannot stop for a player
  choice, and a development build calls it out loudly rather than letting it stall the
  hit.
- **Zero or positive is a reaction.** It runs *after* the change commits, sees the new
  real state, and answers with changes of its own — "when hit, retaliate." A reaction
  **may pause for input**, and if it does, it resolves at the acting ability's next
  natural pause — never mid-step. An opponent's retaliate passive can therefore never
  wedge your ability between two of its steps; it waits for a place your ability's
  author already allowed a pause.

Within a phase, a lower order resolves first, and exact ties break by entity age —
older first — so the sequence is identical in live play, after an undo, and in a
replay. (This is the same
[deterministic ordering](../../concepts/events-and-reactions.md#ordering-is-deterministic-always)
the events page established; the trigger's order feeds straight into it.)

Mirror the concept page's worked example: an armor plate and a thorns aura, both
authored on the defender.

=== "Blueprint"
    1. **Armor (interrupt).** Add a trigger: **Event Tag** `Stat.Health`, **Channel**
       **Self**, **Order** `-10` (negative → pre-commit), an optional **Condition**, and
       a **Program Override** that halves the proposed amount. It runs synchronously —
       no pauses.
    2. **Retaliate (reaction).** Add a second trigger: **Event Tag** `Stat.Health`,
       **Channel** **Self**, **Order** `0` (zero → post-commit), and a **Program
       Override** that deals the counter-swing. A reaction is allowed to pause for input;
       when it does, it resolves at the acting ability's next natural pause.

=== "C++"
    ```cpp
    // Armor: a pre-commit interrupt. Negative order, and it never pauses.
    FPAbTriggerSpec Armor;
    Armor.EventTag        = HealthTag;                 // react to changes to this stat
    Armor.Channel         = EPAbTriggerChannel::Self;  // on MY own window
    Armor.Order           = -10;                       // interrupt phase (the sign picks it)
    Armor.ProgramOverride = UArmorSoften::StaticClass();

    // Retaliate: a post-commit reaction. Zero-or-positive order; may pause for input.
    FPAbTriggerSpec Retaliate;
    Retaliate.EventTag        = HealthTag;
    Retaliate.Channel         = EPAbTriggerChannel::Self;
    Retaliate.Order           = 0;                      // reaction phase
    Retaliate.ProgramOverride = URetaliateSwing::StaticClass();

    // Both are entity data — they undo, save, and replay with the entity that carries them.
    FPAbTriggerList Triggers;
    Triggers.Triggers = { Armor, Retaliate };
    ```

Every reaction a trigger sets off folds into the *same* undo step as the action that
caused it: **responses nest, newest first, inside the action that caused them**, and
one press of Undo removes the whole exchange together — the hit, the retaliation, and
anything they set in motion. That is the same
[cascade-is-one-undo-step](../../concepts/events-and-reactions.md#a-cascade-is-one-undo-step)
rule the events page established.

## The intent phase

**Order** decides *when within a broadcast* a trigger runs. **Phase** decides *which
broadcast it subscribes to at all* — a second, independent axis.

| Phase | The moment it fires |
|---|---|
| **Main** (default) | The window everything above describes: the interrupt phase, then the commit, then the reaction phase. Every trigger authored before this field existed behaves exactly this way. |
| **Intent** | A **pre-proposal** broadcast, fired against fully committed state *before* any proposal or transaction for the change exists — "I am about to change this stat", not "here is the proposed number". |

The intent phase exists to answer one need: a response that must happen **before** a
change lands *and* needs to ask a person. Because the intent broadcast commits
nothing and holds nothing open, a trigger listening on it behaves differently from a
Main-phase one:

- **Negative order on the intent phase** is a synchronous shaper or veto of the
  *declaration* itself.
- **Zero-or-positive order on the intent phase** *may pause for input* — a
  counterattack prompt, resolved before the original change is even proposed.

That is the whole authoring cost: the same trigger spec you already write, plus this
one field. The emitting side authors nothing — an ordinary windowed change already
broadcasts its intent.

=== "Blueprint"
    1. Author the trigger exactly as you would any reaction: **Event Tag**
       `Stat.Health`, **Channel** **Parent**, **Order** `0`, a **Program Override**
       that opens the counterattack prompt.
    2. Set **Phase** to **Intent**. That is the only difference — the reaction now
       resolves before the incoming change is proposed rather than after it commits.

!!! note "Main is still the right default"
    Use the intent phase when the *ordering* genuinely matters — the response has to
    land before the change. A retaliation, a bleed, an on-hit counter and every other
    "answer what just happened" rule belongs on **Main**, where it sees the real
    committed state.

## What a preview assumes about your trigger

A [preview or AI trial](reference-previews.md) runs the real triggers, which raises a
question a real activation never has to answer: what happens when a reaction *would*
stop and ask a player who is not the one previewing? A what-if run cannot prompt
them, so it has to assume something — and **Speculation Policy** is where the
trigger's author says what.

| Value | What a what-if run assumes |
|---|---|
| **Assume Taken** (default) | The reactor spends the reaction: the attack of opportunity connects, the trap fires. The worst case, and the conservative reading for AI threat scoring — an enemy weighing a move sees the danger rather than wishing it away. |
| **Assume Declined** | The reactor lets it pass — an optimistic preview. |
| **Evaluator Scored** | Present in the dropdown, and currently resolves exactly as **Assume Taken**. |

This is consulted **only** inside a speculative run, and only when the reaction would
suspend for a choice belonging to another controller. A live reaction always waits for
the real answer; nothing here changes what actually happens in play.

## Declaration windows

An interrupt reshapes a *number*. Some responses have to happen before the effect and
involve a *choice* — the counterspell moment, where a player decides whether to stop a
cast. For that, an ability opens a **declaration window** on purpose: a step in its
program that announces "I am about to do this" and gives opponents a chance to respond
*before* the effect lands.

A declaration window is a **step** you place in an ability program — the built-in
**Window: Declaration** module. It broadcasts on the caster's own window channel, so a
trigger listening on that `Event.Window.*` moment fires while the declaring ability is
paused. If nobody is listening, the window costs nothing and the ability runs straight
through.

```cpp
// A program step that opens a response window at an authored point.
class UPAbWindow_Declaration : public UPAbAbilityModule   // DisplayName "Window: Declaration"
{
    // The declared moment. By convention its tag lives under Event.Window.*
    // (a trigger's Event Tag matches it exactly).
    FGameplayTag EventTag;

    // The numbers on the table while the window is open — e.g. { Stat.Health: -40 }
    // for a declared 40 damage. Interrupts may shape these; a responder may zero
    // them out, which fizzles the declared effect.
    TMap<FGameplayTag, float> DeclaredMagnitudes;
};
```

=== "Blueprint"
    1. In the ability's [program](reference-programs.md), add a **Window: Declaration**
       step at the point the effect should be *declared* — after targeting, before the
       damage step.
    2. Set its **Event Tag** to an `Event.Window.*` moment (e.g. `Event.Window.Spell`)
       and fill **Declared Magnitudes** with the numbers on the table.
    3. On the responding entity, author a trigger whose **Event Tag** is that same
       moment. An interrupt trigger shapes or cancels automatically; a trigger that must
       *prompt a player* runs as a reaction that opens its own decision point.

!!! warning "Know the boundary: automatic interrupts vs. a player's choice"
    A plain windowed stat change (the direct **Apply Stat Change** from the
    [events model](../../concepts/events-and-reactions.md#reaction-windows)) offers
    **automatic interrupts only** in its main phase — a rule may reshape or veto the
    number, but a Main-phase interrupt cannot stop to ask a human. If a *player* must
    choose before the hit lands, there are exactly two doors: the source is an ability
    whose program opens a **declaration window**, or the responder listens on
    [the intent phase](#the-intent-phase). Both work for the same reason — each stays
    open, committing nothing, until the response resolves. Reach for a declaration
    window when the *caster* wants to announce a specific moment ("I am casting Mind
    Bomb"), and for the intent phase when the *responder* wants to answer any change
    of a given kind before it is even proposed.

## The window payload

While a declaration window is open, the response reads and rewrites a shared
**window payload** — the facts of the declared moment. Unlike a trigger spec, this
struct is not a details-panel row; every field is read and written **at runtime from an
event-handler graph** (or C++ handler) reacting to the window. It is the single object
an interrupt or a prompted response uses to reshape — or cancel — what is about to
happen.

```cpp
struct FPAbWindowPayload   // DisplayName "Ability Window Payload"
{
    FPGeEntityRef Source;         // the entity declaring the moment (the caster)
    FPGeEntityRef Target;         // the primary declared target, if there is one yet
    FGameplayTag  AffectedStat;   // the stat the declared effect concerns
    FGuid         OrderId;        // identity of the declaring activation
    EPAbMode      Mode;           // how the declaring run is executing (live, preview, ...)
    bool          bCancelled;     // set this to COUNTER the declared effect
    FPGeEntityRef RedirectTarget; // set this to send the declared effect somewhere else
};
```

The two writable levers are the story: set **`bCancelled`** to counterspell the effect
outright, or set **`RedirectTarget`** to make it land on someone else. The declared
*numbers* live separately, in the window broadcast's reshapeable magnitudes (the
`DeclaredMagnitudes` a responder shapes in place), so a generic "amplify" or "ward"
adjusts damage without ever touching this struct. Branching on **`Mode`** lets a
response behave differently during a live cast versus a
[preview or what-if run](reference-previews.md).

!!! note "A documented extension point"
    `FPAbWindowPayload` is meant to be **subclassed** by a project that needs to carry
    extra facts through its own windows — it is constructed by the framework at runtime,
    not serialized into your assets. Like the trigger and step config on this page, it is
    a stable contract: changes to it are additive-only, so content authored against it
    keeps working across framework upgrades.

## How trigger sources stack

A unit rarely carries every trigger on itself. The framework's answer to "this unit has
armor *and* thorns *and* a start-of-turn regen" is **composition, not array editing**:
each source is a separate child entity — the armor item, the thorns buff, the regen
passive — and each carries *its own* trigger. Granting the buff onto the unit is the
same granted-modifier pattern from
[Stats and Modifiers](../../concepts/stats-and-modifiers.md); a buff that reacts *for
its owner* simply uses the **Parent** channel.

You never call a subscribe or register function, and you never edit a global listener
list. **The live subscriptions are derived from entity state automatically** — grant a
child that carries a trigger and the listener exists; revoke it and the listener is
gone; undo the grant and the listener comes back. The subsystem that maintains those
derived subscriptions is internal plumbing you do not touch; the entire authoring
surface is the trigger spec and the entity hierarchy.

!!! tip "\"At the start of my turn\" is just a trigger"
    A start-of-turn passive is not a special mechanism. It is a trigger whose **Event
    Tag** is a turn event and whose **Channel** is **Self** — the same shape as armor,
    pointed at a different moment. Turn boundaries are gameplay events broadcast on the
    unit's own channel (see [Turns and Scheduling](../../concepts/turns-and-scheduling.md)),
    so "when my turn begins" and "when I take damage" are the same kind of rule.

## See also

- [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#triggers-entity-carried-rules)
  — the concept and the running Mind Bomb example this page implements.
- [Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md) — the
  interrupt/reaction phases, deterministic ordering, and cascade rule triggers build on.
- [EventSystem reference](../eventsystem/reference.md#windowed-changes-entity-system) —
  the windowed changes a native trigger listens on, and the observer path for
  display-only listeners.
- [Programs & Steps](reference-programs.md) — the ability
  program a trigger runs, and where the **Window: Declaration** step lives.
- [Targeting & running abilities](reference-targeting.md) — the target sources a
  target-source channel resolves against.
