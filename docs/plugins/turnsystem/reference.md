# TurnSystem API Reference

For developers who want the complete public surface. Each section covers one area
with clean signatures and short usage notes; for step-by-step recipes see the
[Guides](guides.md), and for the model see
[Turns and Scheduling](../../concepts/turns-and-scheduling.md).

This section spans the two plugins the turn story is built from. The turn **core**
types carry the `PGe` prefix and ship with the [GameEntity](../gameentity/index.md)
plugin; the scheduling **toolkit** types carry the `PTs` prefix and ship with
TurnSystem. Signatures below are hand-written to show the shape of each API; they
are illustrative, not source excerpts.

## The turn tracker and the advance

The core turn machinery lives on the [GameEntity](../gameentity/index.md)
subsystem, so these calls work with no other plugin enabled. Turn state is entity
state on a framework-minted **turn tracker**, and an advance is one command-routed
undo step.

`FPGeTurnState` is the tracker's indices — ordinary entity data that saves,
undoes, and replays with the match.

```cpp
struct FPGeTurnState
{
    int32        Round       = 0;   // 1-based round index (0 before the first advance)
    int32        TurnInRound  = 0;   // 1-based turn within the round
    int32        TurnSerial   = 0;   // monotonic across the whole match — the log spine
    FGameplayTag Phase;              // opaque, game-defined
    FGameplayTag ActiveSide;         // opaque side identity — "what a side is" stays yours
};
```

The tracker lifecycle and reads:

```cpp
// Mint the tracker (command-routed; joins an open transaction). InitialState is
// written verbatim; TemplateId optionally template-backs the tracker. Opens NO
// scope and fires NO event — the first advance is "turn 1 begins". Returns the
// existing tracker (dev-warned) if one is already live.
FPGeEntityRef InitializeTurnTracker(const FPGeTurnState& InitialState, FName TemplateId);

// Find the live tracker (invalid ref if none).
FPGeEntityRef ResolveTurnTracker() const;

// Read the tracker's current turn state. False (OutState default) if no tracker.
bool GetTurnState(FPGeTurnState& OutState) const;
```

The advance itself. Each call is **one transaction**, so a single Undo rewinds all
of it, and it does a fixed sequence: end-of-turn events fire, "until end of turn"
effects expire, the indices move to `NextState`, and start-of-turn events fire into
the new turn (the exact ordering is on the
[concept page](../../concepts/turns-and-scheduling.md)).

```cpp
// Advance to NextState. The framework never computes the successor — your policy
// or your own code does, and NextState is written verbatim. Dev-warns if
// TurnSerial does not strictly increase. False if there is no tracker or the
// command stack is unavailable.
bool AdvanceTurn(const FPGeTurnState& NextState, const FPGeTurnFanOut& FanOut);

// A round-boundary advance: the same contract, but it also cycles the round scope
// (closes the outgoing round, opens the incoming one and its first turn). Rounds
// are opt-in — a game that never calls this never has a round scope.
bool AdvanceRound(const FPGeTurnState& NextState, const FPGeTurnFanOut& FanOut);

// Change only the phase within the current turn (no scope cycling; a phase is a
// label, not a boundary). A no-op (NewPhase equals the current phase) returns false.
bool SetTurnPhase(FGameplayTag NewPhase, const FPGeTurnFanOut& FanOut);
```

!!! warning "Gate before advancing — an interrupt is a no-op"
    A turn event carries no pending mutation. An interrupt-preset subscriber still
    runs, but marking the event "interrupted" is a dev-warned no-op — there is
    nothing to cancel once the advance is under way. Decide whether the turn may
    advance *before* you call, in your own end-turn gate. The framework never
    decides who's next, who wins, or whether an advance is allowed; those stay your
    game's.

The fan-out is how the game names *which* units hear an advance's events. The turn
event is always broadcast on the tracker's own channel for board-wide listeners;
the fan-out re-broadcasts it to each named unit's own channel too, so a
per-unit "at the start of my turn" observer fires.

```cpp
struct FPGeTurnFanOut
{
    TArray<FPGeEntityRef> OutgoingListeners;   // receive the End-family events
    TArray<FPGeEntityRef> IncomingListeners;   // receive the Start-family events
};
```

Every turn event delivers the same payload — the tracker plus the state on both
sides of the boundary:

```cpp
struct FPGeTurnEventPayload
{
    FPGeEntityRef Tracker;         // the tracker entity (also the broadcast instigator)
    FPGeTurnState PreviousState;   // "whose turn is ending" for End listeners
    FPGeTurnState State;           // the state after this advance
    FPGeEntityRef ClosingScope;    // the closing turn/round scope (for scope-aware rules)
    FPGeEntityRef OpeningScope;    // the opening turn/round scope
};
```

!!! note "An empty leg is skipped"
    If nothing is subscribed on an event leg, that leg is skipped entirely — which
    keeps a long roguelike pump of hundreds of advances cheap. A start-of-turn
    *effect* that changes state (poison ticking damage) is not an observer but a
    game rule authored on the unit; that authoring surface is documented in a later
    section.

## Scheduler policies

A **scheduler policy** is the open extension point for who-acts-next. It is a
`USTRUCT` deriving from `FPTsSchedulerPolicy`, stored polymorphically as the
tracker's active policy. A policy is a **pure rule**: every method reads only the
view it is handed — the turn state, the participants with their entries, and a
read-only entity handle — and never writes, reads a clock, or touches hidden
state. That purity is what lets the projected order be recomputed on demand and
stay correct in a preview or after an undo.

The base type — override the questions your ordering needs; all base defaults are
inert, so an unconfigured policy simply schedules nothing:

```cpp
struct FPTsSchedulerPolicy   // USTRUCT base; derive your policy from this
{
    // The struct type participants carry under Data.Turn.Schedule (an initiative
    // int, a next-act time, a side marker...). Base default: none.
    virtual UScriptStruct* GetEntryType() const;

    // The next Depth actors, from current state. Base default: empty.
    virtual void Project(const FPTsSchedulerView& View, int32 Depth,
                         TArray<FPGeEntityRef>& OutOrder) const;

    // The turn state the next advance should write, and whether it is a round
    // boundary (drives AdvanceTurn vs AdvanceRound). Return false when there is no
    // successor. Base default: false.
    virtual bool ComputeNextTurnState(const FPTsSchedulerView& View,
                                      FPGeTurnState& OutNext, bool& bOutNewRound) const;

    // The coarse facets — beyond the always-invalidating entries and turn state —
    // whose change should recompute the projection. Base default: none.
    virtual void GetDeclaredReadFacets(FPTsReadFacets& Out) const;

    // Which participants belong to Side — the flow driver's fan-out needs this,
    // since only the policy knows its own entry shape. Base default: empty.
    virtual void GetSideMembers(const FPTsSchedulerView& View, FGameplayTag Side,
                                TArray<FPGeEntityRef>& Out) const;
};
```

What a policy reads — everything arrives in the view; there is nothing else to
consult:

```cpp
// One scheduled participant as the projection sees it: the entity and a read-only
// view of its schedule entry. Transient — a window into live state, never stored.
struct FPTsParticipant
{
    FPGeEntityRef           Ref;
    const FInstancedStruct* Entry = nullptr;   // the policy-declared entry struct
};

// What a policy is handed. Participants arrive pre-sorted by entity serial (the
// deterministic tiebreak). Transient; never a stored type.
struct FPTsSchedulerView
{
    FPGeTurnState                 TurnState;
    TArray<FPTsParticipant>       Participants;
    const UPGeGameStateSubsystem* GameState = nullptr;   // read-only entity queries
};

// A policy's declared read-set: the coarse facets whose change dirties the
// projection. The subsystem matches tags hierarchically.
struct FPTsReadFacets
{
    FGameplayTagContainer Tags;             // tag roots whose add/remove matters
    FGameplayTagContainer Stats;            // stat tags whose value change matters
    FGameplayTagContainer TaggedDataKeys;   // tagged-data keys whose write matters
};
```

The **one shipped preset** is side-based (sometimes called I-go-you-go): each side
takes its whole turn, then play passes to the next side, and one full cycle of the
sides is a round. It orders each side's units by their **entity serial** — not by
any speed stat — so it needs no per-unit configuration beyond side membership.

```cpp
// The side-based entry: a participant's side identity and nothing else.
struct FPTsScheduleEntry_Side
{
    FGameplayTag Side;   // one of the active policy's Sides
};

// The side-based policy. Configure it with an ordered list of side tags.
struct FPTsSchedulerPolicy_Igougo : public FPTsSchedulerPolicy
{
    // Ordered side identities. The successor cycles through these; a wrap past the
    // last side is a round boundary. Empty = schedules nothing.
    TArray<FGameplayTag> Sides;

    // Carriers of any of these (hierarchical) are skipped for the CURRENT turn —
    // the "already acted" / "dead" marks. Your game stamps them scoped to the turn,
    // so they clear automatically next turn.
    FGameplayTagContainer ExcludeMarks;

    // A separate exclusion set for the event audience: end/start events should still
    // reach a unit that already acted, but not a dead one.
    FGameplayTagContainer FanOutExcludeMarks;

    // ...overrides GetEntryType, Project, ComputeNextTurnState, GetDeclaredReadFacets,
    //    and GetSideMembers.
};
```

!!! tip "The preset is not privileged"
    An initiative list, a time-unit or charge-gauge timeline where the fastest unit
    acts next, or anything you author plugs into the exact same base type. Only
    under a policy that ranks by a speed stat does "slow" or "haste" reorder the
    strip — the shipped side-based preset ranks by entity serial and will not. See
    [Write a scheduler policy](guides.md#write-a-scheduler-policy).

## The scheduler subsystem

`UPTsSchedulerSubsystem` is a `UGameInstanceSubsystem` — one per running game. It
holds the participation helpers, the active policy, and the projected order. It
owns no turn state of its own; it mirrors the turn core reactively, the way a
derived cache mirrors the state it is computed from.

```cpp
// Resolve the subsystem through any world-context object.
static UPTsSchedulerSubsystem* Get(const UObject* WorldContextObject);
```

Participation — thin, command-routed helpers over the entity's `Data.Turn.Schedule`
tagged data. An entity participates if and only if it carries that key.

```cpp
// Add Entity to the turn order by writing its schedule Entry (command-routed).
// Dev-warns (authoring feedback only — it still writes) if Entry's type differs
// from the active policy's entry type. False only if there is no game state or the
// entity does not exist.
bool JoinTurnOrder(FPGeEntityRef Entity, const FInstancedStruct& Entry);

// Remove Entity from the turn order. False if it was not participating.
bool LeaveTurnOrder(FPGeEntityRef Entity);
```

The active policy — stored as ordinary match state on the tracker, so it saves,
undoes, and rides into previews with the match. You can swap it mid-match as an
undoable change.

```cpp
// Set the active policy (command-routed). Dev-warns + returns false if there is no
// tracker or the struct is not an FPTsSchedulerPolicy child. BlueprintCallable.
bool SetActivePolicy(const FInstancedStruct& Policy);

// The active policy instance, or nullptr when there is no tracker or none set. The
// pointer is valid until the next authoritative mutation. C++-only (no Blueprint node).
const FPTsSchedulerPolicy* GetActivePolicy() const;
```

The projected order — **computed on demand and never stored**. Ask for the next N
actors; the active policy produces them from current state.

```cpp
// The next Depth actors per the active policy. Recomputed lazily only when the
// cache is dirty. False (OutOrder empty) when inert — no tracker or no policy.
// BlueprintCallable.
bool GetProjectedOrder(int32 Depth, TArray<FPGeEntityRef>& OutOrder);

// Passthroughs to the active policy over a freshly built view — C++-only (no
// Blueprint node). Both return false when inert.
bool ComputeNextTurnState(FPGeTurnState& OutNext, bool& bOutNewRound) const;
bool GetSideMembers(FGameplayTag Side, TArray<FPGeEntityRef>& OutMembers) const;
```

The projection-changed notice:

```cpp
// Fires on the first invalidation after a clean compute; listeners re-query. A
// plain multicast delegate, NOT a gameplay event.
FSimpleMulticastDelegate OnProjectionChanged;
```

!!! warning "`OnProjectionChanged` is C++-only — Blueprint must poll"
    This is a plain C++ multicast delegate; **Blueprint cannot bind it**. A
    Blueprint UI re-reads `GetProjectedOrder` after each advance (and after any
    board change the order depends on) instead. And nothing in your game *rules*
    may react to it at all — the projected order is
    [derived state](../../concepts/derived-state.md), and reacting to derived state
    re-deriving is a desync trap. UI re-reads a preview; rules react to the state
    underneath it.

## The flow driver

`UPTsTurnFlowDriver` is the optional match conductor. It is a game-instantiated
`UObject` (`Blueprintable`), **not a subsystem** — there is no static `Get`. You
construct one, keep a reference (e.g. on your game mode), and call `Initialize`.
Its defining property: it **owns no game state and re-derives its next move** from
entity state on demand, so a load, an undo, or a preview run can never strand it
mid-thought.

```cpp
// Bind the driver to a world and subscribe it as a reactive consumer so it
// re-derives after a restore. Idempotent. BlueprintCallable.
void Initialize(UObject* WorldContext);

// Start the match as ONE transaction: seed the tracker, set the active policy, and
// take the first advance ("round 1 begins"). One undo returns to pre-match. False
// if a match is already running or the policy is not an FPTsSchedulerPolicy child.
// BlueprintCallable.
bool StartMatch(const FInstancedStruct& Policy);

// Advance one leg: check the advance gate, ask the policy for the successor, build
// the fan-out from side membership, and call the core AdvanceTurn / AdvanceRound.
// False when the gate is closed, inert, or the policy reports no successor.
// BlueprintCallable.
bool AdvanceToNextTurn();
```

Convenience reads — every fact is read from entity state on demand, so these are
always current, including right after an undo/redo:

```cpp
bool         IsMatchRunning() const;                    // a tracker exists with readable state
bool         GetTurnState(FPGeTurnState& OutState) const;
FGameplayTag ActiveSide() const;                        // the current turn's side (invalid if none)
TArray<FPGeEntityRef> ActiveParticipants() const;       // the active side's members (the fan-out audience)
FGameplayTag ControllerOf(FPGeEntityRef Participant) const;  // its Data.Turn.Controller binding kind
```

The routing surface — how the driver hands control out. These are the same kind of
plain C++ delegate as the projection notice, for the same reason: a routing signal
is not a gameplay event, so no rule may react to it.

```cpp
// prev + new turn state on every advance.
FPTsOnTurnAdvanced  OnTurnAdvanced;
// the active side when a player side is up to act — raise your input UI here.
FPTsOnAwaitingInput OnAwaitingInput;
```

!!! warning "The routing delegates are C++-only — Blueprint must poll"
    `OnTurnAdvanced` and `OnAwaitingInput` are plain C++ multicast delegates;
    **Blueprint cannot bind them**. A Blueprint game reads the convenience reads
    right after `StartMatch` / `AdvanceToNextTurn` — check `ActiveSide`, walk
    `ActiveParticipants`, and branch on `ControllerOf` each — to decide what to do
    next: show end-turn UI for a player side, kick your own AI for units bound to
    AI. A C++ game may bind the delegates instead, or subclass the driver.

To shape the driver's behaviour, subclass it and override the two protected
extension points (C++):

```cpp
// The advance gate. Default true. Override with your "can this turn end?" logic —
// idle checks, match-over, an end-turn gate.
virtual bool IsAdvanceAllowed() const;

// The fan-out for an advance. The default fills OutgoingListeners from the current
// side's members and IncomingListeners from the next side's, via the policy.
virtual FPGeTurnFanOut BuildFanOut(const FPGeTurnState& Current, const FPGeTurnState& Next) const;
```

Per-participant control is a **data binding** on the entity, so the driver reads
how to route each unit from state rather than a table it maintains:

```cpp
// Written as Data.Turn.Controller tagged data on a participant (an ordinary,
// command-routed entity tagged-data write — see the GameEntity plugin). ControllerOf
// reads back the Controller tag; an invalid tag means unbound.
struct FPTsControllerBinding
{
    FGameplayTag Controller;   // PTsTags::TAG_Turn_Controller_Player / _Ai / a game-defined leaf
};
```

## Tags

The plugin ships a small set of native gameplay tags. The `Data.Turn.*` keys are
tagged data on entities or the tracker; the `Turn.Controller.*` leaves are the two
canonical controllership kinds. "What a side is" and the AI pump stay your game's —
the plugin ships only the seam.

TurnSystem tags (`PTsTags` namespace):

| Tag | Role |
|---|---|
| `TAG_Data_Turn_Schedule` | Participation key. An entity is in the turn order iff it carries this tagged data; the value's struct type is the active policy's entry type. |
| `TAG_Data_Turn_SchedulerPolicy` | The active policy, stored as match state in the tracker's tagged data. |
| `TAG_Data_Turn_Controller` | A participant's controllership binding (an `FPTsControllerBinding` value). |
| `TAG_Turn_Controller_Player` | Controllership kind: a player side. The driver raises awaiting-input for it. |
| `TAG_Turn_Controller_Ai` | Controllership kind: an AI side. Marks the hand-off to your deliberation code — the framework never runs the AI pump for you. |

The turn **core** contributes its own tags from the [GameEntity](../gameentity/index.md)
plugin, including `Turn.Tracker` (the tracker's identifying tag),
`Data.Turn.State` (where `FPGeTurnState` lives), the `Scope.Turn` / `Scope.Round`
boundary scopes, and the `Event.Turn.*` window tags (`Start`, `End`, `Round.Start`,
`Round.End`, `Phase`) the advance broadcasts.
