# Targeting & Running Abilities

For developers wiring the player's side of an ability: deciding what each step acts
on, issuing an activation, answering the choices resolution pauses for, and painting
those choices on the board. After reading you'll know the target value type, the
built-in target sources, the target-selection decision point, and the resolution
subsystem's driver surface — plus the reusable targeting session a shipped UI is
built over. For the model behind all of this see
[Abilities and Step-by-Step Resolution](../../concepts/abilities-and-resolution.md#targeting-and-decision-points);
for authoring the programs and steps these targets flow through see
[Programs, Modules & Custom Steps](reference-programs.md).

All public types carry the `PAb` prefix. Signatures below are hand-written to show the
shape of each API; they are illustrative, not source excerpts.

## What an ability acts on

Every target an ability touches — an enemy, a spot on the ground, a grid cell — is one
`FPAbTarget`. A single target carries **all three facets at once**: an entity, a world
location, and a grid cell. It also records a **primary kind** — what it fundamentally
*is* — so a cell target that later learns which entity stands on it is still, at heart,
a cell.

```cpp
struct FPAbTarget   // BlueprintType, display name "Ability Target"
{
    FPGeEntityRef   Entity;                             // the entity facet
    FVector         Location = FVector::ZeroVector;     // the world-location facet
    FGridNodeHandle Cell;                               // the grid-cell facet
    EPAbTargetKind  PrimaryKind = EPAbTargetKind::None; // what this target fundamentally is

    // Read-only bookkeeping: which facets are actually populated. Maintained ONLY by
    // the C++ factories below — never hand-set, or it desyncs from the fields it describes.
    uint8 PopulatedFacets = 0;   // BlueprintReadOnly, inspect-only

    bool HasEntity()   const;    // C++ helpers: is that facet populated?
    bool HasLocation() const;
    bool HasCell()     const;
};

enum class EPAbTargetKind : uint8   // BlueprintType
{
    None,
    Entity,     // a unit / item / ability — anything with an entity id
    Location,   // a point in world space
    Cell,       // a cell on the grid
};
```

Reading a populated facet is free and lossless. Turning a cell into the entities
standing on it is a lossy projection done on demand and cached back onto the target, so
a step that starts from a cell can later act on its occupant without losing the original
cell.

!!! warning "Never build an `FPAbTarget` by hand in Blueprint"
    The facet fields are Blueprint-writable, but the populated-facets mask is
    **read-only** and is only ever set by the target's own C++ factories
    (`MakeEntity` / `MakeLocation` / `MakeCell`). A target assembled from a blank
    struct literal in a Blueprint graph leaves that mask empty — so `HasEntity()`
    reports *false*, the target matches nothing, and every merge, dedup, and
    candidate check silently misbehaves.

    In Blueprint you never construct a target. **You obtain targets from a targeting
    step or a target source** (below) and pass them along. In C++, always go through a
    factory:

    ```cpp
    FPAbTarget OnEnemy = FPAbTarget::MakeEntity(EnemyRef);   // the ONLY sanctioned build path
    FPAbTarget OnCell  = FPAbTarget::MakeCell(CellHandle);   // no Blueprint make-node exists
    FPAbTarget AtPoint = FPAbTarget::MakeLocation(WorldPos);
    ```

Two targets are considered **the same** — for merging into the target list, for
deduplication, and for matching a pick against a decision request's candidates — when
they share a primary kind *and* that kind's defining facet is equal. Caching an extra
facet never changes a target's identity. This logical-identity rule is exactly why a
hand-built target with an empty facet mask fails to match a real candidate.

## Target sources: how a step decides what to act on

A **target source** answers one question for a step: *what does this act on?* It reads
the [context](reference-programs.md) and returns a list of targets — it is stateless,
synchronous, and read-only (a resolver, never a step that changes state). A step's
"source" field simply *is* a target source; a target-selection decision point's
"candidate source" is one too.

The framework ships a set of built-ins. The first six take no configuration; the last
three read a field or two.

| Editor label | Resolves to |
|---|---|
| **Previous (Working List)** | the incoming target list, unchanged — the ~90% default when a step sets no source |
| **Previous Result** | just what the nearest earlier step *found* (its own results), not the whole forwarded list |
| **Previous Outgoing** | the exact list the nearest earlier step *passed on* |
| **Owner** | the ability entity's hierarchical owner (the unit wielding it), falling back to the caller |
| **Caller** | the activating unit, as one entity target |
| **Ability** | the ability entity itself, as one entity target |
| **Lookback (ABM Handle)** | a *specific* earlier step's results or forwarded list, by a stable handle |
| **Reachable Cells** | the mover's cost-aware reachable cells — a move ability's candidate cells |
| **Grid Circle Occupants** | the entities within N grid-hops of an anchor — a ranged attack's candidates |

The three configurable sources:

```cpp
// "Lookback (ABM Handle)" — read one named earlier step's output by its stable handle.
struct UPAbSourceAccessor_Lookback   // an EditInlineNew Source Accessor class
{
    FGuid            Handle;   // which earlier step (the authoring dropdown writes this)
    FString          Label;    // human-readable echo only — NEVER matched, so a rename is safe
    EPAbLookbackList List;     // Result (what it found) or Outgoing (the whole list it forwarded)
};

enum class EPAbLookbackList : uint8   // BlueprintType
{
    Result,     // the targets that step produced
    Outgoing,   // the target list it forwarded downstream
};

// "Reachable Cells" — the mover's reachable set, the candidate source a move decision point points at.
struct UPAbSourceAccessor_ReachableCells
{
    UPAbSourceAccessor* Source;   // optional mover override; unset = the Caller
    /* FEvalMagnitudeCalc */ MaxCost;   // move budget (usually a stat); empty = the whole reachable set
    bool bIncludeStart = true;    // include the mover's own current cell
};

// "Grid Circle Occupants" — the occupants within a hop-radius of an anchor, the
// candidate source a ranged-attack decision point points at.
struct UPAbSourceAccessor_GridCircleOccupants
{
    UPAbSourceAccessor* Source;   // optional anchor override; unset = the incoming list, else the Caller's cell
    int32 Radius = 1;             // radius in grid hops (0 = the anchor cell only)
    /* FEvalMagnitudeCalc */ RadiusOverride;   // optional stat-driven radius; unset = the raw Radius
    bool bIncludeCenter = true;   // include occupants of the anchor cell itself
};
```

The `MaxCost` and `RadiusOverride` fields are Evaluator magnitudes: a move ability's
range or a bolt's radius is normally a *stat*, so one shared program yields a short-range
and a long-range spell just by reading a different number off the entity.

### Writing your own target source

A custom target source is a small subclass — it is a Blueprint-native extension point, so
you can author one in Blueprint or C++. Keep to the contract: read state and return a
list; never change anything.

=== "Blueprint"
    1. Create a Blueprint subclass of the **Source Accessor** class.
    2. Implement the **Resolve Sources** event: read whatever you need from the context
       and return an array of targets. Obtain those targets from the entity library or
       another target source — never build one from a blank struct.
    3. Drop your subclass into any step's **Source** (or a decision point's **Candidate
       Source**) field.

=== "C++"
    ```cpp
    UCLASS(DisplayName="Source: Allies In Party")
    class UMyPartySource : public UPAbSourceAccessor
    {
        GENERATED_BODY()
    protected:
        virtual TArray<FPAbTarget> ResolveSources_Implementation(UPAbExecContext* Ctx) const override
        {
            TArray<FPAbTarget> Out;
            for (const FPGeEntityRef& Ally : /* read party members from the entity library */)
            {
                Out.Add(FPAbTarget::MakeEntity(Ally));   // factory — never a blank literal
            }
            return Out;   // stateless, synchronous, read-only
        }
    };
    ```

## The target-selection decision point

When an ability needs the player (or a preview, or the AI) to *choose* its targets, the
program includes a **target-selection decision point**. When resolution reaches it, it
gathers a candidate set, suspends, and publishes a [decision request](#the-decision-request)
describing the choice — then waits until an answer arrives.

```cpp
struct UPAbGate_SelectTargets   // a step; editor display name "Gate: Select Targets"
{
    FText               Prompt;         // designer-facing text carried on the request ("Choose a target")
    UPAbSourceAccessor* CandidateSource;// the candidate set; unset = the incoming target list
    int32 MinSelections = 1;            // a non-empty answer must fit [Min, Max]
    int32 MaxSelections = 1;
    /* FEvalMagnitudeCalc */ MinSelectionsOverride;   // optional stat-driven counts; unset = the raw values
    /* FEvalMagnitudeCalc */ MaxSelectionsOverride;
};
```

The chosen targets *replace* the running target list, so a decline — an empty answer —
leaves nothing for the following steps to act on, and the ability harmlessly does nothing.
A decision point that finds **no candidates at all** completes immediately with an empty
result instead of suspending on an unanswerable prompt.

=== "Blueprint"
    1. In the ability's program, add a **Gate: Select Targets** step.
    2. Set its **Prompt** to the player-facing text and its **Candidate Source** to a
       target source (e.g. **Grid Circle Occupants** for "pick a unit in range"). Leave
       Candidate Source empty to choose among whatever the previous steps found.
    3. Set **Min Selections** / **Max Selections** for how many may be picked.
    4. Order the following state-changing steps after it — they act on exactly the picked
       targets.

=== "C++"
    ```cpp
    // Configured on the program's step list, not built at runtime. Conceptually:
    UPAbGate_SelectTargets* Pick = /* the program's authored step */;
    Pick->Prompt        = NSLOCTEXT("Ability", "PickTarget", "Choose a target");
    Pick->CandidateSource = NewObject<UPAbSourceAccessor_GridCircleOccupants>(Program);
    Pick->MinSelections = 1;
    Pick->MaxSelections = 1;
    ```

## Running an ability

`UPAbResolutionSubsystem` is the rules engine that resolves an ability step by step. It is
a world subsystem — one per world — and it is the single door for issuing, querying, and
undoing activations from game code.

```cpp
// Resolve the subsystem through any world-context object (self in Blueprint).
static UPAbResolutionSubsystem* Get(const UObject* WorldContextObject);
```

### Activating

```cpp
// Issue one activation. Returns immediately: true if it was ACCEPTED (its use conditions
// passed and it began resolving), false if refused. Caller FIRST, then Ability.
bool TryActivateAbility(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability);

// How the most recent activation ended. Read right after TryActivateAbility returns.
EPAbOrderResolution GetLastOrderResolution() const;

// Is an activation currently in flight (or a triggered response holding the line open)?
bool IsResolving() const;
```

```cpp
enum class EPAbOrderResolution : uint8   // BlueprintType
{
    None,           // nothing has resolved yet this session
    Committed,      // committed a real, observable change — the ordinary success
    ResolvedEmpty,  // ran to completion but changed NOTHING — the "I clicked and nothing happened" case
    Aborted,        // fizzled: cancelled, replaced, refused, or a misbehaving step
};
```

!!! note "Caller first, then Ability"
    `TryActivateAbility`, `CanUse`, `CanActivate`, and `ExplainCanUse` all take two
    entity references in **Caller-then-Ability** order, and nothing stops you swapping
    them by mistake. The caller is the unit acting; the ability is the granted entity
    being used.

!!! tip "The nothing-happened diagnostic"
    `ResolvedEmpty` is worth surfacing. A targeting step that found no one, or an effect
    that was fully resisted, runs to completion and commits *nothing*. Reading this result
    lets you replace a dead-feeling button press with an honest "no valid target in range".

Only one activation resolves at a time. Issue a new one while an earlier one is still
mid-targeting and — because nothing has committed — it simply *replaces* the old one;
issue it after anything has committed and it is *refused*, because committed state never
silently unwinds. Both outcomes are reported through [`OnOrderAborted`](#reacting-to-resolution).

### Checking whether an ability can be used

```cpp
// Cheap per-frame affordability check — evaluates the ability's declared use conditions.
// No use conditions -> usable. Ideal for a button's enabled state.
bool CanUse(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability);

// The diagnostic form: fills OutResults with every condition's index, label, and pass/fail
// (never short-circuits) so a greyed button can name exactly which condition failed.
bool ExplainCanUse(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability,
                   TArray<FPAbUseConditionResult>& OutResults) const;

// The heavier on-click check: runs a what-if forward to the first decision point to
// confirm a legal target actually exists before committing the player to targeting.
bool CanActivate(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability);
```

```cpp
struct FPAbUseConditionResult   // BlueprintType, read-only
{
    int32   ConditionIndex = INDEX_NONE;  // which declared use condition this is
    FString Label;                        // its debug label, for a tooltip
    bool    bPassed = false;              // did it pass?
};
```

`CanUse` is cheap enough for every frame; `CanActivate` runs a what-if against a
temporary, throwaway copy of the game state, so save it for the moment of the click. Both
are covered in more depth, alongside hover previews and AI, in
[Previews, AI & What-Ifs](reference-previews.md).

=== "Blueprint"
    1. On a "cast" input, get the resolution subsystem and call **Try Activate Ability**
       (Caller, then Ability).
    2. Read **Get Last Order Resolution** — light up a "no valid target" message on the
       *Resolved Empty* case.
    3. Bind the ability button's enabled state to **Can Use**; on a greyed button call
       **Explain Can Use** for the specific failing condition; on click call
       **Can Activate** to confirm a legal target exists.

=== "C++"
    ```cpp
    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);

    if (Rules->TryActivateAbility(Caster, MindBomb))            // Caller, then Ability
    {
        if (Rules->GetLastOrderResolution() == EPAbOrderResolution::ResolvedEmpty)
        {
            // Nothing was in the blast — tell the player why.
        }
    }

    Button->SetEnabled(Rules->CanUse(Caster, MindBomb));        // cheap, per-frame

    TArray<FPAbUseConditionResult> Reasons;                     // why is it greyed?
    Rules->ExplainCanUse(Caster, MindBomb, Reasons);

    if (Rules->CanActivate(Caster, MindBomb)) { /* enter targeting */ }   // on click
    ```

### Undo and redo

```cpp
// Undo / redo one activation, as a single unit (its whole reaction cascade included).
// Refused (returns false) unless both the action and its on-screen playout are idle.
bool TryUndo();
bool TryRedo();

// Whether undo/redo would be accepted right now — a UI-level check for the buttons.
bool IsUndoRedoAllowed() const;
```

One activation is one undo step: undoing removes everything it committed — the effect, any
status it applied, and any response it provoked — together. Undo is honored **politely**:
only once nothing is resolving and the on-screen playout has drained, so a rewind never
fights the screen.

## Driving a decision point

When a decision point suspends, the subsystem publishes exactly **one** decision request
at a time and announces it. Whatever supplies the answer — the player's UI in real play, a
hovered target in a preview, the AI's candidate during deliberation — answers through the
same three verbs.

```cpp
// Answer the published request with the chosen target(s). Rejected (false) if the handle
// is stale, a pick isn't among the candidates, or a non-empty selection breaks [Min, Max].
bool SubmitGateInput(const FGuid& GateHandle, const TArray<FPAbTarget>& SelectedTargets);

// Decline/SKIP the published decision point: a normal EMPTY answer — resolution CONTINUES.
bool DeclineGate(const FGuid& GateHandle);

// CANCEL the whole in-flight activation and roll it back. Returns false if nothing is
// in flight to cancel.
bool CancelOrder();

// Read the currently published request, if any (false when none is pending).
bool GetPendingDecisionRequest(FPAbDecisionRequest& OutRequest) const;
```

!!! note "Decline is a skip; cancel is an undo"
    **Decline** answers *one* decision point with an empty selection and lets resolution
    carry on — "don't move this unit" — and it bypasses the minimum-count constraint by
    design. **Cancel** aborts the *entire* activation and unwinds every committed segment.
    They are not interchangeable: decline ≠ cancel.

### Reacting to resolution

Three Blueprint-assignable, multicast delegates on the subsystem let a UI drive itself by
event instead of polling:

```cpp
// A decision point suspended and its request reached the front of the line — start
// presenting the choice. (This request is always a real, live one, never a what-if's.)
void OnDecisionRequested(const FPAbDecisionRequest& Request);

// An in-flight activation aborted, OR a new activation was refused by the busy policy.
// Culprit is a human-readable "what happened" string.
void OnOrderAborted(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability,
                    EPAbOrderAbortReason Reason, const FString& Culprit);

// The bookend to OnDecisionRequested: the published request went away — clear your UI.
void OnDecisionRequestClosed(const FGuid& GateHandle, EPAbDecisionRequestCloseReason Reason);
```

```cpp
enum class EPAbOrderAbortReason : uint8   // BlueprintType
{
    Cancelled,   // an explicit CancelOrder
    Replaced,    // a new activation superseded this one while it had committed nothing
    Rejected,    // a new activation was refused because committed work held the line
    Aborted,     // a misbehaving step forced an abnormal exit
};
```

Bind `OnDecisionRequested` to raise your targeting UI, `OnDecisionRequestClosed` to tear
it down, and `OnOrderAborted` to surface a legible "cancelled" / "can't do that right now"
message instead of a dead button.

## The decision request

`FPAbDecisionRequest` is the read-only payload a decision point publishes — everything a UI
needs to present the choice, and nothing it can mutate. The engine constructs it; you only
read it.

```cpp
struct FPAbDecisionRequest   // BlueprintType, read-only, display name "Ability Decision Request"
{
    FText              Prompt;           // the designer-facing prompt to show
    EPAbMode           Mode;             // which kind of run is asking (a real activation, or a what-if)
    TArray<FPAbTarget> Candidates;       // the legal choices — the set to pick from
    int32              MinSelections;    // how few may be picked in a non-empty answer
    int32              MaxSelections;    // how many
    FGuid              GateHandle;       // the key you answer with (see the warning below)
    FGuid              OrderId;          // the activation this belongs to (invalid for a standalone response)
    bool               bFromReaction;    // true when the pending choice belongs to a triggered response
};
```

The `bFromReaction` flag marks the "waiting on a reaction" state: the pending choice does
not belong to the ability you just activated but to a triggered response resolving in the
middle of it — a counter, a soak, a prompted counterspell (see
[Triggers & Reactions](reference-triggers.md)). A UI should say so rather than looking
frozen.

!!! warning "Never cache `GateHandle` — always re-read it"
    Each time the same decision point suspends, it is issued a **fresh** `GateHandle`.
    A submit carrying a handle from a superseded suspension is silently rejected. Always
    take the handle from the *live* `OnDecisionRequested` payload (or from
    `GetPendingDecisionRequest` at the moment you answer) — never from a value you stored
    earlier.

When a request closes, `OnDecisionRequestClosed` reports *why*:

```cpp
enum class EPAbDecisionRequestCloseReason : uint8   // BlueprintType
{
    Answered,   // answered with a non-empty selection (SubmitGateInput)
    Declined,   // skipped with an empty answer (DeclineGate) — resolution continued
    Retracted,  // went away unanswered because its activation was cancelled / replaced / aborted
};
```

## The reusable targeting session

You *can* drive resolution with the raw verbs above, but most of the bookkeeping — adopting
the published request, accumulating a multi-selection, validating it against the min/max
before submitting, and posting legible rejection notices — is already written for you in
`UPAbTargetingSession`. It is a render-free object you compose into your UI (it is a
Blueprint value type, used and composed, not subclassed); it holds the whole non-visual
selection lifecycle and talks to the subsystem's public verbs so your widget only has to
draw.

```cpp
class UPAbTargetingSession   // BlueprintType — compose one; do not subclass it
{
    // Bind to the resolution subsystem (null unbinds). Adopts any already-published request.
    void BindToResolution(UPAbResolutionSubsystem* Resolution);

    // Issue an activation with a legible rejection notice instead of a dead button.
    bool RequestActivateAbility(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability);

    // The input path.
    bool SelectCandidate(int32 CandidateIndex);  // single-select submits at once; multi-select toggles
    bool SubmitSelection();                       // submit the accumulated multi-selection
    bool DeclinePendingGate();                    // empty answer — resolution continues
    void CancelPendingOrder();                    // roll the whole activation back

    // Hover preview — paint the predicted outcome of a hovered target, torn down cleanly
    // when the hover moves on. (The what-if machinery is covered on the previews page.)
    bool PreviewGateInput(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability,
                          const FPAbTarget& HoveredTarget);
    void ClearGateInputPreview();

    // State the renderer reads.
    bool HasActiveRequest() const;
    FPAbDecisionRequest GetActiveRequest() const;
    bool IsWaitingOnReaction() const;   // the request belongs to a triggered response
    int32 GetSelectedCount() const;
    FString GetStatusText() const;      // e.g. "TARGETING …" / "WAITING ON A REACTION …"
    FString GetLastNotice() const;                       // last one-shot message
    EPAbTargetingNoticeLevel GetLastNoticeLevel() const; // its severity — map it to a colour
    int32 GetNoticeSerial() const;                       // bumped on each notice; re-post when it changes

    bool bAutoSubmitAtMax = true;   // auto-submit a multi-selection the moment it hits MaxSelections
};

enum class EPAbTargetingNoticeLevel : uint8   // BlueprintType
{
    Info, Success, Warning, Error,
};
```

The session is deliberately render-free: it *classifies* a notice's severity but never
picks a colour or draws anything. Candidate anchors, highlights, and the status banner are
your UI's job, sourced from `GetActiveRequest()`, `GetStatusText()`, and `GetLastNotice()`.

!!! note "The reference driver is an example — replace it"
    The plugin ships `APAbTargetingDriver`, a debug-draw actor that renders the session's
    candidates as spheres, maps keys to the session verbs (click to pick, Enter to submit,
    X to decline, Esc to cancel), and posts notices on screen. It exists to keep the API
    honest and to be a worked example — **your shipped game replaces it with its own UMG UI
    over the same session calls.** Everything real lives in the session; the driver is just
    the first skin over it.

=== "Blueprint"
    1. On your targeting widget, create a **Targeting Session** object and call **Bind To
       Resolution** with the resolution subsystem.
    2. To start an ability, call **Request Activate Ability** (Caller, then Ability).
    3. On **On Decision Requested** (bound via the subsystem), read the session's
       **Get Active Request** and draw its candidates.
    4. On a candidate click, call **Select Candidate**; a single-select choice submits at
       once, a multi-select one accumulates and auto-submits at the maximum.
    5. Wire **Decline Pending Gate** to your skip button and **Cancel Pending Order** to
       your cancel button. Re-post **Get Last Notice** whenever **Get Notice Serial**
       changes.

=== "C++"
    ```cpp
    Session->BindToResolution(UPAbResolutionSubsystem::Get(this));
    Session->RequestActivateAbility(Caster, MindBomb);   // Caller, then Ability

    // On a click over candidate index i:
    Session->SelectCandidate(i);     // submits immediately for a single-select decision point

    Session->DeclinePendingGate();   // skip — resolution continues
    Session->CancelPendingOrder();   // cancel — the whole activation rolls back
    ```

## Painting targets on the grid

On a grid, the cells a decision point offers — and the cells an in-program marker step
telegraphs — are painted through the GridGraph plugin's
[Target visualization channel](../gridgraph/reference-visualization.md#three-independent-channels).
The bridge that connects the two is `UPAbTargetVisualizerFeederSubsystem`. Wire it once and
it **self-drives**: it listens for published decision requests and paints their candidate
cells while the choice is open, clearing them when it closes.

```cpp
class UPAbTargetVisualizerFeederSubsystem   // a world subsystem
{
    static UPAbTargetVisualizerFeederSubsystem* Get(const UObject* WorldContextObject);

    // Wire the render surface (your grid visualization driver) and bind to resolution —
    // that is the whole setup; the feeder drives itself from there.
    void SetDriver(UGridVisualizationDriverComponent* Driver);
    void BindToResolution(UPAbResolutionSubsystem* Resolution);

    void ClearTargets();   // clear the grid target channel by hand, if you ever need to

    // Read-back for a HUD or a test.
    UGridVisualizationDriverComponent* GetDriver() const;
    bool     HasShownTargets() const;
    EPAbMode GetLastShownMode() const;      // whether the last paint was a real run or a preview
    int32    GetLastShownCellCount() const;
    int32    GetPaintSerial() const;        // bumped each time it actually paints
};
```

The feeder paints for the **real activation** and for a **hover preview**, and stays silent
for the game's own behind-the-scenes trials — an AI weighing a move, or a cheap
can-I-use-this check, never flashes a telegraph on screen (and, deliberately, a silent
trial can never *wipe* a live telegraph mid-flight). A game normally does nothing beyond
`SetDriver` + `BindToResolution`; the paint verb itself is a C++-only path the in-program
marker step uses, not something you call.

=== "Blueprint"
    1. Get the **Target Visualizer Feeder** subsystem.
    2. Call **Set Driver** with your grid actor's
       [Grid Visualization Driver](../gridgraph/reference-visualization.md#how-grid-visualization-works)
       component (its profile must include a Target presenter, e.g. **Target – Decal
       Marker**).
    3. Call **Bind To Resolution** with the resolution subsystem. That's the whole wiring —
       candidate cells now paint on the Target channel whenever a decision point is open.

=== "C++"
    ```cpp
    UPAbTargetVisualizerFeederSubsystem* Feeder =
        UPAbTargetVisualizerFeederSubsystem::Get(this);

    Feeder->SetDriver(GridActor->FindComponentByClass<UGridVisualizationDriverComponent>());
    Feeder->BindToResolution(UPAbResolutionSubsystem::Get(this));
    // From here it self-drives: candidate cells paint while a choice is open and clear when it closes.
    ```

For the presenters, themes, and channels that render those cells, see
[Visualization](../gridgraph/reference-visualization.md). For hover previews, availability
what-ifs, and enemy AI, see [Previews, AI & What-Ifs](reference-previews.md). For the
programs, steps, and target-list model these targets flow through, see
[Programs, Modules & Custom Steps](reference-programs.md).
