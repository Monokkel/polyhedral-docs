# TurnSystem Guides

For developers with the plugin enabled who want to get something done. Each
section is a short, followable recipe. For the full type-by-type surface see the
[API Reference](reference.md); for the model behind it all see
[Turns and Scheduling](../../concepts/turns-and-scheduling.md).

## Start a match and advance turns

The flow driver is the quickest way to a working match. It is a plain object you
construct and keep — not a subsystem, so there is no static `Get`; hold a
reference on your game mode. Give it a policy and it seeds the tracker, begins
round 1, and advances on demand, each advance a single undo step.

=== "Blueprint"
    1. **Construct Object from Class** with class **PTs Turn Flow Driver** (or your
       own subclass of it). Store the result on your game mode and call
       **Initialize** on it, passing self as the world context.
    2. Build the shipped side-based policy: **Make** an
       `FPTsSchedulerPolicy_Igougo`, set its **Sides** to `[Side.Player,
       Side.Enemy]`, and wrap it with **Make Instanced Struct**.
    3. Call **Start Match** with that instanced struct. It seeds the tracker and
       begins round 1 — all as one undo step.
    4. On your **End Turn** button, call **Advance To Next Turn**.
    5. On your **Undo** button, **Get Command Stack** and call **Undo** — the whole
       turn rewinds, events and expired effects included.

=== "C++"
    ```cpp
    // Construct the driver once and keep it (e.g. a UPROPERTY on your game mode).
    Driver = NewObject<UPTsTurnFlowDriver>(this);
    Driver->Initialize(this);

    // The shipped side-based preset, configured with an ordered list of side tags.
    FPTsSchedulerPolicy_Igougo Policy;
    Policy.Sides = { PlayerSideTag, EnemySideTag };      // opaque, game-defined
    Driver->StartMatch(FInstancedStruct::Make(Policy));  // seed tracker + begin round 1

    // Later, on "End Turn":
    Driver->AdvanceToNextTurn();                          // one advance, one undo step

    // Undo rewinds the entire turn.
    UPCsCommandStack::Get(this)->Undo();
    ```

!!! tip "Gate the advance, don't cancel it"
    `AdvanceToNextTurn` checks an advance gate first and does nothing if it's
    closed. Put "can this turn end right now?" logic there — override
    `IsAdvanceAllowed` in a C++ subclass, or simply decide in your own code before
    calling. Once an advance runs there is nothing to veto; a turn event is a
    notification, not a proposal.

## Join and leave the turn order

A unit is in the turn order for exactly one reason: it carries a **schedule
entry** under the `Data.Turn.Schedule` key. Writing that entry joins the order;
removing it leaves. There is no membership list to maintain, and death needs no
call at all — destroying the entity takes its entry with it.

=== "Blueprint"
    1. Get the scheduler subsystem (**Get** on `UPTsSchedulerSubsystem`, world
       context self).
    2. On spawn, **Make** the policy's entry struct — for the side-based preset,
       an `FPTsScheduleEntry_Side` with **Side** set to that unit's side — and wrap
       it with **Make Instanced Struct**.
    3. Call **Join Turn Order** with the unit and that entry.
    4. To pull a unit out *without* killing it, call **Leave Turn Order**. On
       death, do nothing.

=== "C++"
    ```cpp
    UPTsSchedulerSubsystem* Scheduler = UPTsSchedulerSubsystem::Get(this);

    FPTsScheduleEntry_Side Entry;
    Entry.Side = PlayerSideTag;
    Scheduler->JoinTurnOrder(Unit, FInstancedStruct::Make(Entry));  // command-routed

    // Optional voluntary exit; death needs no call at all.
    Scheduler->LeaveTurnOrder(Unit);
    ```

!!! note "Joining undoes with the spawn"
    `JoinTurnOrder` and `LeaveTurnOrder` are command-routed writes, like every
    other authoritative change. A unit that entered the order on spawn leaves it
    again cleanly when you undo the spawn — see
    [Commands & Undo](../../concepts/commands-and-undo.md).

## Show the initiative strip

An initiative UI wants the "next up" order. Ask the scheduler for the next N
actors and feed them to your widget. The order is **computed on demand and never
stored**, so there is nothing to keep in sync — you just re-read it whenever it
could have changed.

The change-notice hook (`OnProjectionChanged`) is a **C++-only delegate**; a
Blueprint game cannot bind it. So a Blueprint UI **polls**: re-query the order
right after each advance, and after any board change that could reorder it.

=== "Blueprint"
    1. Get the scheduler subsystem and call **Get Projected Order** with a depth
       (say `8`). Feed the returned entities into your initiative-strip widget.
    2. Rebuild the strip after each turn advances — call this straight from your
       **End Turn** handler, right after **Advance To Next Turn** — and whenever
       the board changes in a way the order depends on.

=== "C++"
    ```cpp
    UPTsSchedulerSubsystem* Scheduler = UPTsSchedulerSubsystem::Get(this);

    TArray<FPGeEntityRef> Order;
    Scheduler->GetProjectedOrder(8, Order);   // the next 8 actors, from live state

    // In C++ you may bind the change notice instead of polling:
    Scheduler->OnProjectionChanged.AddWeakLambda(this, [this]{ RebuildStrip(); });
    ```

!!! warning "The projection is a preview — no rule may react to it"
    Re-reading the strip is a UI job. A *game rule* must never fire because the
    projected order changed — that is [derived state](../../concepts/derived-state.md),
    and reacting to it re-deriving is a desync trap. Rules react to the *state
    underneath* — the stat write, the joined participant — never to the preview.

The shipped side-based preset orders each side's units by their entity serial, so
"reorder the strip with a haste stat" is **not** something it does. That example
holds only under a policy that ranks by a speed stat — which you would write
yourself (see the next recipe). Under such a policy, applying the stat modifier
and re-querying returns the reordered strip with no scheduler code touched.

## Write a scheduler policy

Who acts next is not baked into the framework — it is a **scheduler policy**, and
the side-based preset is not privileged. Your own policy plugs into the exact same
extension point, from the same base type. A policy is a **pure rule**: it reads
the participants' schedule entries plus the current turn state and answers the
who-acts-next questions. "Pure" means it reads only that — no wall clock, no
hidden variables — so the same board always schedules the same way, live or
replayed.

A policy is a `USTRUCT` deriving from `FPTsSchedulerPolicy`. Give it whatever
configuration it needs (kept to plain tags and containers so it saves and undoes
as ordinary match state), declare the shape of the entry its participants carry,
and override the questions you care about:

```cpp
// The entry each participant carries under Data.Turn.Schedule.
USTRUCT()
struct FInitiativeEntry
{
    GENERATED_BODY()
    int32 Initiative = 0;   // higher acts earlier
};

// A simple "highest initiative acts next" policy.
USTRUCT()
struct FMyInitiativePolicy : public FPTsSchedulerPolicy
{
    GENERATED_BODY()

    // Participants carry FInitiativeEntry.
    UScriptStruct* GetEntryType() const override
    {
        return FInitiativeEntry::StaticStruct();
    }

    // Produce the next Depth actors from current state. The view hands you the
    // participants (already sorted by entity serial for a deterministic tiebreak)
    // with their entries. Read them; never write.
    void Project(const FPTsSchedulerView& View, int32 Depth,
                 TArray<FPGeEntityRef>& OutOrder) const override
    {
        TArray<FPTsParticipant> Ranked = View.Participants;
        Ranked.StableSort([](const FPTsParticipant& A, const FPTsParticipant& B)
        {
            return A.Entry->Get<FInitiativeEntry>().Initiative
                 > B.Entry->Get<FInitiativeEntry>().Initiative;
        });

        for (int32 i = 0; i < Ranked.Num() && OutOrder.Num() < Depth; ++i)
        {
            OutOrder.Add(Ranked[i].Ref);
        }
    }

    // The turn state the next advance should write. Return false when there is no
    // successor (no participants). Set bOutNewRound at a round boundary.
    bool ComputeNextTurnState(const FPTsSchedulerView& View,
                              FPGeTurnState& OutNext, bool& bOutNewRound) const override
    {
        TArray<FPGeEntityRef> Order;
        Project(View, 1, Order);
        if (Order.Num() == 0) { return false; }

        OutNext = View.TurnState;
        OutNext.TurnSerial += 1;           // the serial must strictly increase
        bOutNewRound = false;
        return true;
    }
};
```

Make it the active policy through the scheduler (a command-routed, undoable
write), then everything else — the projection, the flow driver — reads from it:

```cpp
UPTsSchedulerSubsystem::Get(this)->SetActivePolicy(FInstancedStruct::Make(FMyInitiativePolicy{}));
```

!!! note "A policy is pure, and reads only the view"
    Everything a policy needs arrives in the `FPTsSchedulerView` — the turn state,
    the participants with their entries, and a read-only handle to the entity
    library for declared-facet queries. Never read a wall clock, a global, or a
    derived cache; same view in, same answer out. That is what lets the projected
    order be recomputed on demand and stay correct in a preview or after an undo.

!!! tip "Declare what dirties the projection"
    If your ordering depends on a facet *other* than the schedule entries and turn
    state — a speed stat for an initiative timeline, say — override
    `GetDeclaredReadFacets` to name it. The scheduler then knows to recompute the
    projection when that facet changes, so an initiative UI refreshes on a speed
    edit without you wiring it by hand.

## Drive turns without the toolkit

The flow driver, the scheduler, and the policies are all optional. The turn
*core* — the tracker and the advance — lives in the entity system, so you can run
turns with nothing from this plugin at all. This is the right choice for a fixed
two-player alternation, a solitaire loop, or any flow simple enough that a
scheduler is overkill.

You compute the next turn state yourself and hand it to `AdvanceTurn`. The
framework never decides who's next — your code does — and writes exactly the state
you give it.

=== "Blueprint"
    1. On match start, get the game-state subsystem and call **Initialize Turn
       Tracker**, passing an initial `FPGeTurnState` (round 0, serial 0) and an
       optional template id. This mints the tracker; it opens no scope and fires no
       event.
    2. On **End Turn**, **Make** the next `FPGeTurnState` — bump the serial, flip
       the active side, advance the round when you wrap — and call **Advance Turn**
       with it and a fan-out naming the units that should hear the end/start
       events.
    3. One **Undo** rewinds the whole advance.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    // Mint the tracker once. No scope, no event — the first advance is "turn 1 begins".
    FPGeTurnState Initial;                       // round 0, serial 0
    State->InitializeTurnTracker(Initial, NAME_None);

    // On "End Turn": you compute the successor (the framework never does).
    FPGeTurnState Current;
    State->GetTurnState(Current);

    FPGeTurnState Next = Current;
    Next.TurnSerial  += 1;                        // must strictly increase
    Next.ActiveSide   = (Current.ActiveSide == PlayerSideTag) ? EnemySideTag : PlayerSideTag;

    // Name who hears the end/start turn events (your call — membership is policy).
    FPGeTurnFanOut FanOut;
    FanOut.OutgoingListeners = UnitsOnSide(Current.ActiveSide);   // hear Turn.End
    FanOut.IncomingListeners = UnitsOnSide(Next.ActiveSide);      // hear Turn.Start

    State->AdvanceTurn(Next, FanOut);            // one transaction, one undo step
    ```

!!! note "Rounds are opt-in"
    Call `AdvanceRound` instead of `AdvanceTurn` at a round boundary and the
    advance also cycles the round scope. A game that never calls it never has a
    round scope — you only pay for what you use.
