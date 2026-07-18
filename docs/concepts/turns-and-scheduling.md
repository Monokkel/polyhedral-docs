# Turns and Scheduling

For developers structuring a match into rounds, turns, and phases — and deciding who acts next. After reading, you'll know why turn state lives on an entity instead of in a manager, what happens (and in what order) when a turn advances, how a scheduler policy computes the turn order, and how to preview and manipulate that order safely.

## Turn state is an entity

It is tempting to keep the current round and the active side as plain fields on a game controller. Don't. Those fields have no history, so the first time a player undoes a move, they silently drift from the rest of the game — the board rewinds and the turn counter does not.

The framework keeps turn state where every other guarantee already lives: on an entity. A framework-created **turn tracker** entity holds the round number, the turn within that round, a monotonic turn counter that only ever climbs, the current phase, and the active side. Because it is an entity, the whole of turn state undoes, saves, and replays — and rides along into previews and AI look-aheads — exactly like the units on the board — see [Entities Are Data](entities-as-data.md). It *cannot* fall out of step with the game, because it is part of the game.

!!! warning "Don't keep turn state in a manager"
    Round, turn, and active-side fields on a controller or a subsystem look convenient and desync the moment a player undoes. The turn tracker exists so there is nothing left to keep in sync — read the round from the tracker, never from a field you maintain yourself.

The core turn machinery — the tracker and the call that advances it — ships inside the entity system, so you can run turns with nothing else added. Everything else on this page (policies, the projection, the flow driver) is an optional toolkit layered on top of it.

<!-- pluginlink: turnsystem-index -->

## Advancing the turn

Moving the match forward is one **advance**: one call, one undo step, and one fixed sequence of events. Advancing always does the same four things in the same order:

1. **End-of-turn events fire first** — while the ending turn's effects are still in place, so a rule like "at end of turn, if you still have Haste…" can actually see the Haste.
2. **Effects scoped to "until end of turn" expire** — visibly, so any rule that runs next sees them already gone. This is the same expiry the [Stats and Modifiers](stats-and-modifiers.md) page called an end condition.
3. **The turn indices move** — round, turn-in-round, the monotonic counter, phase, and active side update to the new turn.
4. **Start-of-turn events fire into the new turn** — so a start-of-turn reaction can, for instance, apply an effect that lasts *this* turn.

All four steps run inside a single [transaction](commands-and-undo.md#transactions-one-undo-step-for-a-whole-move), so one **Undo** rewinds the entire advance — the events that fired, the effects that expired, the indices — as a single step.

!!! tip "Gate before advancing; don't try to veto after"
    There is nothing to cancel once an advance is under way — a turn event is a notification, not a proposal, and marking one "interrupted" is a no-op. A rule like "you can't end your turn while burning" belongs *before* the call, in your game's own end-turn gate. Decide whether the turn may advance; then advance it.

=== "Blueprint"
    1. Create a flow driver once (**Construct Object from Class**, or your own subclass of it), keep a reference to it on your game mode, and call **Initialize**.
    2. Build a side-based policy: **Make** the policy struct, set its **Sides** to `[Side.Player, Side.Enemy]`, wrap it with **Make Instanced Struct**, and pass it to **Start Match** — that seeds the tracker and begins round 1, all as one undo step.
    3. On an **End Turn** button, call **Advance To Next Turn**.
    4. On an **Undo** button, get the command stack and call **Undo** — the whole turn rewinds, events and expiries included.

=== "C++"
    ```cpp
    // Create the driver once and keep it (e.g. on your game mode).
    UPTsTurnFlowDriver* Driver = NewObject<UPTsTurnFlowDriver>(this);
    Driver->Initialize(this);

    FPTsSchedulerPolicy_Igougo Policy;
    Policy.Sides = { PlayerSide, EnemySide };            // opaque game-defined side tags
    Driver->StartMatch(FInstancedStruct::Make(Policy));  // seed tracker + begin round 1, one undo step

    // Later, on "End Turn":
    Driver->AdvanceToNextTurn();                          // one advance, one undo step

    UPCsCommandStack::Get(this)->Undo();                  // the whole turn rewinds
    ```

## Turn events reach the right ears

A turn boundary is broadcast as a gameplay event on the machinery from [Events, Ordering, and Reaction Windows](events-and-reactions.md) — so "the turn advanced" is heard exactly like any other event. Each turn event goes out on two kinds of channel:

- **The tracker's own channel**, for board-wide listeners — a turn banner, a combat log, an initiative UI.
- **Each entity your game names for that advance**, on that entity's own channel — because *whose* turn it is is your game's call, so your game supplies the list of units the end-of-turn and start-of-turn events are re-broadcast to. This re-broadcast to a named set of entities is the advance's **fan-out**.

Once an event lands on a unit's own channel, "at the start of my turn" is just an ordinary observer on that unit — the previous page's vocabulary, nothing new here.

=== "Blueprint"
    On a unit's widget or component, **Subscribe to Event** for `Event.Turn.Start` on that unit's own channel. When it fires, flip a "YOUR TURN" banner on. This is an observer: it is told after the fact and changes only presentation.

=== "C++"
    ```cpp
    // Observe start-of-turn on this unit's own channel (a read-only reaction).
    UPEsEventSubsystem::Get(this)->Subscribe(this, TurnStartTag, UnitChannel);
    // In the handler: read the payload's new turn state, show the banner. No state writes.
    ```

An observer only *reflects* the turn. A start-of-turn effect that *changes* state — poison ticking damage at the start of your turn — is a game rule authored on the unit as a trigger, which is documented in a later section.

## Participants join by carrying an entry

An entity is in the turn order for one reason: it carries a **schedule entry** — a small struct, under the `Data.Turn.Schedule` key, whose shape the active policy defines. Writing that entry joins the turn order; removing it leaves. There is no membership list to maintain and nothing to keep in sync with the board: when a unit dies, its entry dies with it, and it drops out of the order automatically.

=== "Blueprint"
    On spawn, **Make** the policy's entry struct (for the side-based preset, an entry carrying `Side = Side.Player`), wrap it with **Make Instanced Struct**, and call **Join Turn Order** on the scheduler subsystem. To pull a unit out without killing it, call **Leave Turn Order**. On death, do nothing — destroying the entity removes its entry for you.

=== "C++"
    ```cpp
    UPTsSchedulerSubsystem* Scheduler = UPTsSchedulerSubsystem::Get(this);

    FPTsScheduleEntry_Side Entry;
    Entry.Side = PlayerSide;
    Scheduler->JoinTurnOrder(Unit, FInstancedStruct::Make(Entry));  // command-routed: undoes with the spawn

    // Optional voluntary exit; death needs no call at all.
    Scheduler->LeaveTurnOrder(Unit);
    ```

Joining and leaving are command-routed like every other write, so a unit that entered the order on spawn leaves it again cleanly when you undo the spawn.

## Scheduler policies decide who acts next

Who acts next is not baked into the framework — it is a **scheduler policy**: a pure rule that reads the participants' schedule entries plus the current turn state and answers the who-acts-next questions — the next actors, the turn state the following advance should write, and which participants belong to a given side. "Pure" means it reads only that state — no wall clock, no hidden variables — so the same board always schedules the same way, live or replayed.

The shipped preset is **side-based** (sometimes called I-go-you-go): each side takes its whole turn, then play passes to the next side, and one full cycle of the sides is a round. You configure it with an ordered list of side tags — `[Side.Player, Side.Enemy]` for a two-sided skirmish, more for a free-for-all — and that is the entire policy.

The same seam expresses far more: an initiative list, a time-unit or charge-gauge timeline where the fastest unit acts next, or anything you author. The side-based preset is not privileged — it plugs into the exact extension point your own policy would, from the same base type.

And because the active policy is itself stored on the turn tracker as ordinary entity data, it saves, undoes, and rides into previews with the match. You can even swap policies mid-match as an undoable change — switch from side-based to a timeline when a "time warp" fires, then undo it and the old order comes back.

<!-- pluginlink: turnsystem-reference -->

<!-- pluginlink: turnsystem-guides -->

## The projected turn order

Every initiative UI wants the same thing: a "next up" strip. The framework gives it to you as the **projected turn order** — ask for the next N actors and the active policy produces them from current state. It is a derived preview, computed on demand and *never stored*. There is no turn-order array to keep synchronized, because there is nothing to synchronize: the answer is recomputed each time you ask.

That is exactly why "haste", "slow", "delay", and "push" need no special API. Each is an ordinary write — a stat modifier, an edit to a schedule entry, a status mark — and the next read of the projection reflects it. A hover preview or an AI look-ahead, running against a temporary copy of the game state that is discarded afterward, sees the reordered strip for free — no scheduler code involved.

!!! warning "Nothing in your game rules may react to the projection changing"
    UI hears about projection changes through a plain notification and re-reads the strip — that is its whole job. But a game *rule* must never fire because the projected order changed. The projection is [derived state](derived-state.md), and reacting to derived state re-deriving is the classic desync trap that page warns about. Rules react to the *state underneath* — the stat write, the joined participant — never to the preview of it.

=== "Blueprint"
    1. Get the scheduler subsystem and call **Get Projected Order** (depth `8`); feed the returned entities into your initiative-strip widget.
    2. Rebuild the strip after each turn advances (call it from your **End Turn** handler, right after **Advance To Next Turn**), or whenever the board changes.
    3. To watch a slow debuff reorder the strip, just apply the debuff as a stat modifier and re-query — under a policy that ranks by a speed stat, the returned order changes with no scheduler code touched. Undo the debuff and it reorders back.

=== "C++"
    ```cpp
    UPTsSchedulerSubsystem* Scheduler = UPTsSchedulerSubsystem::Get(this);

    TArray<FPGeEntityRef> Order;
    Scheduler->GetProjectedOrder(8, Order);   // the next 8 actors, from live state

    // Refresh the strip the moment the order could have changed:
    Scheduler->OnProjectionChanged.AddWeakLambda(this, [this]{ RebuildStrip(); });
    ```

    A slow debuff is an ordinary stat write; under a policy that ranks by a speed stat, the next `GetProjectedOrder` returns the reordered strip. The projection ran no game rules — it just re-read the board.

## The flow driver

The **flow driver** is the optional conductor for a whole match. It asks the projection who is next, routes control — raising an "awaiting input" signal for a player side, handing an AI side to your own deliberation code — and calls the advance. Per-participant control is a data binding on the entity (`Data.Turn.Controller`, marked player or AI), so the driver reads how to route each unit from state, not from a table it maintains.

Its defining property: it owns no game state at all. Every fact it needs — whose turn it is, which side is active, who is on that side — it re-reads from entity state on demand. So a load, an undo, or a preview run can never strand it mid-thought; there is no cursor to restore. If the plain driver does not fit your flow, subclass it, or skip it entirely and drive the core advance yourself.

=== "Blueprint"
    Right after **Start Match** or **Advance To Next Turn**, read **Active Side**, **Active Participants**, and **Controller Of** each participant to decide what happens next: show the end-turn UI for a player side, or kick your own AI for units bound to AI. (The awaiting-input and turn-advanced notifications are C++ hooks — a Blueprint game reads these functions right after advancing, or subclasses the driver in C++ to bind them.)

=== "C++"
    ```cpp
    // A player side is up — raise your input UI. This re-fires after an undo/redo
    // or a load that moves the turn boundary, because the driver re-derives it.
    Driver->OnAwaitingInput.AddWeakLambda(this, [this](FGameplayTag Side)
    {
        ShowEndTurnUI(Side);
    });

    // Route each active participant by its own controller binding:
    for (const FPGeEntityRef& Unit : Driver->ActiveParticipants())
    {
        if (Driver->ControllerOf(Unit) == PTsTags::TAG_Turn_Controller_Ai)
        {
            KickAiFor(Unit);   // your AI pump — the framework never runs it for you
        }
    }
    ```

<!-- pluginlink: turnsystem-reference -->

## What stays yours

The framework supplies the mechanism; the meaning stays your game's. It never interprets a phase or a side — those are opaque tags you define. What remains entirely yours:

- **What a "side" or a "phase" means** — the framework only cycles opaque tags.
- **Who participates** — you write the schedule entries; membership is just which entities carry one.
- **Action budgets** — how many moves or actions a turn grants, and what spends them.
- **When the match starts and ends** — victory, defeat, and the conditions for both.
- **Whether an advance is allowed** — your end-turn gate decides; the advance just carries it out.
