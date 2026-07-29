# Previews, AI & What-Ifs

For developers wiring predictive UI and enemy behavior onto abilities — the red band a health bar paints on hover, the greyed button with an honest tooltip, an enemy that thinks before it acts, and the layer that turns a committed activation into on-screen cues on its own. After reading, you'll know the sanctioned consumer doors for each, and which are Blueprint-friendly versus a C++ extension point.

Everything on this page rests on one honest sentence, the same one the concept page states plainly: **previews and AI what-ifs run the action against a temporary, throwaway copy of the game state and discard it afterward.** The framework copies the whole game value, runs the *real* steps against the copy — real rules, real triggers, real target math — reads back the summarized set of changes the run produced, and throws the copy away. The real game is never touched, and live surfaces stay silent for the duration, so nothing on screen ever reacts to a move that did not really happen. That single mechanism feeds three consumers — hover previews, availability checks, and enemy AI. A fourth surface, the automatic playout layer, turns *real* committed changes into cues.

For the model behind all of this, read [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#what-if-runs-previews-and-enemy-ai). This page is the mechanics.

## The automatic playout layer

This is the layer the [presentation page left waiting](../../concepts/tokens-and-cues.md#where-cues-come-from-today): it watches the changes a resolution commits and turns them into the cues that page documented — choosing snap-versus-sequence, batching one step's changes into coherent beats (all of an area blast's damage numbers as one wave, not a stutter of single hits), and, during a hover, feeding the [preview overlays](#hover-previews) instead of the live cue rail.

Nothing you built on the [Tokens & Cues](../../concepts/tokens-and-cues.md) page changes: the same cues, the same handlers, the same capabilities. This layer is simply the "somebody" that now plays them for you during an ability.

**From Blueprint it is observational.** You watch playout happen; you don't drive it. The verbs that open and close a playout grouping belong to the rules engine and are not part of the public surface — a game never calls them. What you *can* reach is a notification when an action is queued and a couple of inspection hooks:

```cpp
// The automatic playout layer for this world. Blueprint: the context defaults to self.
static UPAbProjectionSubsystem* Get(const UObject* WorldContextObject);

// Fires each time the layer queues a played-out action. Observe only.
FPAbOnActionProjected OnActionProjected;

// The most recently played-out action, and a running count this session —
// inspection hooks (handy for tests and diagnostics).
UPAbGenericProjectedAction* LastAction;
int32                       ActionsProjected;
```

!!! tip "Undo waits politely"
    The undo button is honored only once playout has drained — never mid-animation. A rewind never fights the screen: the beats finish, *then* the state rolls back and every surface snaps to the restored truth. The resolution subsystem exposes this as `IsDrainingPlayout()` and `IsUndoRedoAllowed()`; see [reference-targeting.md](reference-targeting.md).

### Specializing how an intent plays out

The unit the layer queues is `UPAbGenericProjectedAction` — a played-out action that fans one cue per affected token and reports done when the last of them finishes. Out of the box its generic body does the right thing (a net stat decrease drives a `Cue.TakeDamage` on the target; a placement change re-snaps the token). You author a subclass when a particular intent needs a playout of its own — a walked move, a fly-move that arcs, a summon that fades in.

!!! note "A move inside an ability snaps unless you specialize it"
    The generic body re-snaps a token to its new cell; it does not fan a `Cue.Move`.
    That is correct for a blink or a shove, and wrong for a walk — so if you want an
    ability's movement to travel the route, subclass for your `Action.Move` intent and
    play a [`Cue.Move`](../tokensystem/reference.md#the-shipped-cue-handlers) carrying the
    path. Outside an ability, you play that cue yourself the same way.

The recipe mirrors the [cue-handler pattern](../tokensystem/reference.md#cue-handlers): subclass, tag it, override the run hook. It self-registers by its class defaults — there is no central table to edit — and the layer picks the most specific subclass whose intent tag matches, falling back to the generic body when none is more specific.

=== "Blueprint"
    1. Create a Blueprint subclass of **Generic Projected Action**.
    2. In class defaults, set **Handled Intent** to your `Action.*` tag (for example `Action.Move.Fly`). This property is declared on the [TokenSystem projected-action base](../tokensystem/reference.md#projected-actions), not on the ability-system subclass — you'll find it under the **Action** category.
    3. Override the **Run On Queue Async** event: call **Begin Parallel** with the number of tokens you'll drive, then loop **Play Cue On Token** once per affected entity. The action proceeds automatically when the last cue reports done.
    4. Read **Mode** to branch: only a *live* action animates; a what-if action draws a preview or nothing. Read **Fanned Cues** if a test needs to confirm what it played.

=== "C++"
    ```cpp
    // A specialized played-out action for one intent. HandledIntent is inherited
    // from the TokenSystem base (UPTkProjectedAction); set it in class defaults.
    UCLASS()
    class UMyFlyMoveAction : public UPAbGenericProjectedAction
    {
        GENERATED_BODY()
    public:
        UMyFlyMoveAction()
        {
            // Self-registers by this tag — most-specific-wins, generic body as floor.
            HandledIntent = FGameplayTag::RequestGameplayTag(TEXT("Action.Move.Fly"));
        }

        // The run hook. Fan your own cues; the base's BeginParallel/ReportDone latch
        // reports the action done when the last token finishes. Read Mode to tell a
        // live beat from a preview.
        virtual void RunOnQueueAsync_Implementation() override;
    };
    ```

The full projected-action surface — `BeginParallel`, `ReportDone`, `PlayCueOnToken`, and the entity/change arrays the layer fills for you — lives in [Projected actions](../tokensystem/reference.md#projected-actions) in the TokenSystem reference. Entity-less flourishes (a bomb arc, a screen banner) do *not* come through this path; a step enqueues those itself.

## Hover previews

Hovering a target answers the ability's pending [decision point](../../concepts/abilities-and-resolution.md#targeting-and-decision-points) on a throwaway copy of the game state. The run's predicted changes feed the [preview surfaces and ghosts](../../concepts/tokens-and-cues.md#previews-ride-the-same-rails) the presentation page left waiting — the health bar that paints the red band it *would* lose, the translucent stand-in of a unit that *would* be summoned. A preview never commits and never animates the live path; it is torn down cleanly when the hover moves on.

You drive it through the **targeting session** — the same reusable object a real targeting UI uses. Two calls do the whole job:

=== "Blueprint"
    1. On **hover-enter**, call the session's **Preview Gate Input** with the caster, the ability, and the hovered target. It runs the what-if and paints the predicted outcome onto the target's preview surfaces. It returns false (and paints nothing) if a resolution is already in flight.
    2. On **hover-leave**, or before the next preview, call **Clear Gate Input Preview**. It tears down *this* session's overlay by its own handle, so a peer surface's preview — an enemy's telegraph, another widget's hover — is never wiped.

=== "C++"
    ```cpp
    // On hover-enter: run the what-if and paint the predicted outcome. Caller first,
    // Ability second, then the hovered target. Returns false if a resolution is busy.
    Session->PreviewGateInput(Caster, MindBomb, HoveredTarget);

    // On hover-leave (and before the next preview): tear down THIS session's overlay
    // only, leaving any peer telegraph untouched.
    Session->ClearGateInputPreview();
    ```

The rest of the targeting-session API — binding it to resolution, submitting a real selection, cancelling — lives in [reference-targeting.md](reference-targeting.md). The preview *surfaces* themselves (the previewable and ghostable [capabilities](../tokensystem/reference.md#capabilities)) are the presentation side, documented in the TokenSystem reference; this page only feeds them.

!!! note "Reading back the predicted changes yourself"
    Sometimes you want the numbers, not just the paint — a "−12 HP" tooltip beside the cursor, a summary line. The automatic playout layer keeps the summarized set of changes the last what-if produced, so you can read it back directly. **Always pair it with its serial**: a run that aborted or produced no net change does *not* overwrite the buffer, so an unchanged serial means the readout is stale and belongs to an earlier run, not this hover.

    ```cpp
    UPAbProjectionSubsystem* Playout = UPAbProjectionSubsystem::Get(this);

    const int32 Before = Playout->GetLastSpeculativeCoalescedSerial();
    Session->PreviewGateInput(Caster, MindBomb, HoveredTarget);
    if (Playout->GetLastSpeculativeCoalescedSerial() != Before)
    {
        // Fresh — this hover produced changes. Read and display them.
        const TArray<FPGeEntityChange> Predicted = Playout->GetLastSpeculativeCoalesced();
    }
    // Unchanged serial => the hover changed nothing; don't show a stale readout.
    ```

### Predicted terrain changes

An ability can change the board itself — the shipped **Effect: Remove Cells** step removes the cells in its target list (see [Programs & Steps](reference-programs.md#the-building-block-steps)). When such an ability is *previewed*, the cells it would carve out are published the same way its entity changes are, so a UI can ghost the hole before the player commits.

```cpp
// The board changes the last what-if run produced — which cells it removed and added.
UFUNCTION(BlueprintPure) FGridStructuralDelta GetLastSpeculativeStructuralDelta() const;

// Its serial. Pair it with the readout exactly as with the entity changes above:
// an unchanged serial means the run changed no terrain, not that it changed none NOW.
UFUNCTION(BlueprintPure) int32 GetLastSpeculativeStructuralSerial() const;

// Fires when a what-if run's board changes settle. Blueprint-assignable.
FPAbOnSpeculativeStructureCoalesced OnSpeculativeStructureCoalesced;
```

!!! note "You draw the ghost"
    The framework publishes *which* cells a preview would remove or add; it does not render a ghost-terrain overlay for you. Paint it yourself through the [grid visualization channels](../gridgraph/reference-visualization.md), the same way you would any other predictive board paint.

## Checking availability

Two different questions gate an ability button, and they cost very different amounts. Ask the cheap one every frame and the expensive one only on click.

**`CanUse` — the per-frame greyed-button test.** It just evaluates the ability's declared use conditions (cooldown ready, enough resource, in the right phase). It is cheap enough to poll every frame and caches internally, so hovering a hotbar of abilities costs almost nothing. When the button is greyed, `ExplainCanUse` runs the same check but reports *which* condition failed, so the tooltip can say "on cooldown" instead of nothing.

**`CanActivate` — the on-click legality check.** It runs a what-if forward to the first decision point to confirm a *legal target actually exists* before you commit the player to targeting. This one spins up a throwaway copy and runs the real steps against it, so it is heavier — call it on click, not per frame.

```cpp
// Cheap, per-frame: do the ability's declared use conditions pass right now?
// Caller first, Ability second — both are FPGeEntityRef, so mind the order.
bool CanUse(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability);

// The diagnostic form: fills OutResults with every use condition's pass/fail and its
// label, so a greyed button can name the exact reason. Returns the same bool CanUse would.
bool ExplainCanUse(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability,
                   TArray<FPAbUseConditionResult>& OutResults) const;

// On click: run a what-if to the first decision point — does a legal target exist?
bool CanActivate(const FPGeEntityRef& Caller, const FPGeEntityRef& Ability);
```

=== "Blueprint"
    1. Bind the ability button's enabled state to **Can Use** (pure, per-frame) — it checks only the declared use conditions.
    2. On a greyed button, call **Explain Can Use** to name the specific failing condition for a tooltip.
    3. On click, call **Can Activate** first; enter targeting only if it returns true, so a "no legal target" ability never drags the player into a dead targeting mode.

=== "C++"
    ```cpp
    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);

    // Per-frame: is the button live? (cheap — just the declared use conditions)
    Button->SetEnabled(Rules->CanUse(Caster, MindBomb));

    // Why is it greyed? Name the failing condition(s) for a tooltip.
    TArray<FPAbUseConditionResult> Reasons;
    Rules->ExplainCanUse(Caster, MindBomb, Reasons);

    // On click: confirm a legal target exists before entering targeting.
    if (Rules->CanActivate(Caster, MindBomb)) { /* enter targeting */ }
    ```

## Enemy AI

An enemy turn is a legible three-beat loop over the same what-if mechanism: the AI **tries** each candidate action on a throwaway copy, **scores** the outcome, **telegraphs** its chosen action through the *same* preview overlay a player hover uses, then **plays** it for real. While it is deliberating, player input is held so the board cannot change under it.

**What each candidate is worth is authored as data.** An entity carries an **AI profile** — a bundle of *considerations*, each measuring one axis of a candidate's outcome — and the framework combines them into a score. That is the path most games should take, and it needs no C++ at all; it has its own page,
[Enemy AI: Considerations & Profiles](reference-ai.md). This page covers the loop that drives it and the C++ seam for taking scoring over entirely.

!!! warning "The *loop* is a C++-only extension point"
    Authoring judgment is data (see [Enemy AI](reference-ai.md)), but driving the loop is not: apart from `Get`, the deliberation driver's verbs are **C++-only**, and the scorer delegate is a non-dynamic delegate over a reference-carrying view, which cannot be bound from Blueprint by construction. The plan it produces (`FPAbAiPlan`) is Blueprint-readable so you can *inspect* a chosen plan, but everything that produces or acts on one is C++.

### The scorer contract

Binding a scorer is how a game takes scoring over completely. Your scorer is a plain function: given one candidate's outcome, return a float where higher is better. It receives a call-scoped view carrying the summarized changes the trial produced, read-only before/after state, and the board as the trial left it — so it can judge magnitude, position, *and* terrain.

**What "unbound" means:** leaving the scorer unbound selects the **data-driven profile scorer**, which scores each entity from its own authored AI profile. Per entity, if that profile is empty it falls back to a content-neutral stand-in (it scores the magnitude of stat decrease inflicted on units other than the caller), so an enemy carrying no profile behaves exactly as it did before profiles existed. An **explicit binding always wins**, for every entity.

```cpp
// A plain C++ view (deliberately NOT a USTRUCT) handed to your scorer for one
// candidate. Its references are valid ONLY for the duration of the scorer call.
struct FPAbScoreContext
{
    FPGeEntityRef                   Caller;         // the deliberating unit
    FPGeEntityRef                   Ability;        // the ability being scored
    const TArray<FPAbTarget>&       Targets;        // this candidate's targets
    int32                           CandidateIndex; // which candidate of this ability
    int32                           SampleIndex;    // which fair-AI sample

    const TArray<FPGeEntityChange>& Coalesced;      // the summarized changes this trial produced
    const FPGeGameState&            ResultantState; // the copy's after-state (read-only)
    const FPGeGameState&            BaselineState;  // the untouched before-state (read-only)

    const FPAbScoreBoardView&       Board;          // the RESULTANT board: connectivity, occupants, paths
    bool                            bHasStructuralDelta; // did THIS trial change the board's shape?
    const FGridStructuralDelta&     StructuralDelta;     // which cells it added / removed
};

// Higher = better. Non-dynamic delegate — C++ only, cannot be bound from Blueprint.
DECLARE_DELEGATE_RetVal_OneParam(float, FPAbCandidateScorer, const FPAbScoreContext& /*Ctx*/);
```

`Board` answers query-shaped questions about the board the candidate would leave behind — is this cell occupied, what does it cost to path there, what is reachable within a budget. There is deliberately **no baseline board**: a structural before-state would cost a real grid copy per trial. Compute anything baseline-dependent about terrain *outside* deliberation, where the live board is the baseline — it is restored exactly between candidates, so a value read before deliberating stays valid across every trial.

```cpp
// A game-specific scorer reads back the trial's changes and weights them by faction.
float ScoreForMyGame(const FPAbScoreContext& Ctx)
{
    float Score = 0.f;
    for (const FPGeEntityChange& Change : Ctx.Coalesced)
    {
        // Reward harm to my enemies, discount self-harm; compare BaselineState vs
        // ResultantState for positional judgment beyond what the change list encodes.
    }

    // Blend in the framework's own reading of magnitude instead of reimplementing it:
    Score += UPAbAiDeliberationSubsystem::ContentNeutralScore(Ctx.Coalesced, Ctx.Caller);
    return Score;
}

// Bind it once. Unbound => each entity is scored by its own AI profile.
UPAbAiDeliberationSubsystem* Ai = UPAbAiDeliberationSubsystem::Get(this);
Ai->SetCandidateScorer(FPAbCandidateScorer::CreateStatic(&ScoreForMyGame));
```

!!! tip "Blend, don't reimplement"
    `ContentNeutralScore` is public precisely so a game scorer wanting *"the framework's reading of magnitude, plus my faction knowledge"* can add it in rather than rewrite it. It is a static C++ function with no Blueprint node.

### The deliberation driver

`UPAbAiDeliberationSubsystem` runs the loop. `Get` is its one Blueprint-visible call; the rest are C++-only:

```cpp
// Resolve the driver (the only Blueprint-visible call on it).
static UPAbAiDeliberationSubsystem* Get(const UObject* WorldContextObject);

// Bind a game-specific scorer; an explicit binding always wins. Unbound, each
// entity is scored from its own AI profile.
void SetCandidateScorer(FPAbCandidateScorer Scorer);

// The framework's content-neutral reading of a candidate's outcome — the per-entity
// fallback when an entity has no AI profile, exposed so a game scorer can blend it.
static float ContentNeutralScore(const TArray<FPGeEntityChange>& Coalesced,
                                 const FPGeEntityRef& Caller);

// Try each of Caller's candidate actions on a throwaway copy, score the outcome, and
// return the best plan. Holds player input while it runs; the hold STAYS held on a
// valid plan so nothing commits before the AI acts (release it with Commit or Abandon).
FPAbAiPlan Deliberate(const FPGeEntityRef& Caller, const TArray<FPGeEntityRef>& Abilities);

// Paint the chosen plan as the enemy-intent telegraph — the same overlay a hover uses.
void TelegraphPlan(const FPAbAiPlan& Plan);

// Clear the telegraph, release the input hold, and play the plan for real. Returns
// true if the action was accepted.
bool CommitPlan(const FPAbAiPlan& Plan);

// Pass: release the input hold without acting.
void AbandonDeliberation();

// True while the AI holds the turn (between Deliberate and Commit/Abandon).
bool IsDeliberating() const;
```

The three beats in order:

```cpp
UPAbAiDeliberationSubsystem* Ai = UPAbAiDeliberationSubsystem::Get(this);

// 1. Try each candidate on a copy, score it, pick the best. Player input is now held.
FPAbAiPlan Plan = Ai->Deliberate(Enemy, EnemyAbilities);
if (Plan.IsValid())
{
    Ai->TelegraphPlan(Plan);   // 2. Paint "here's what I'll do" for a beat.
    // ...let it read...
    Ai->CommitPlan(Plan);      // 3. Clear the telegraph, then play it for real.
}
else
{
    Ai->AbandonDeliberation(); // Nothing worth doing — pass, release the hold.
}
```

The plan the AI settled on is inspectable:

```cpp
// A scored plan the AI chose — BlueprintReadOnly, so a UI can inspect it.
struct FPAbAiPlan
{
    FPGeEntityRef      Caller;   // the deliberating unit
    FPGeEntityRef      Ability;  // the ability it chose
    TArray<FPAbTarget> Targets;  // the targets to answer the first decision point with
    float              Score;    // the winning outcome score
    bool               bValid;   // false = found nothing worth doing (the AI passes)
    bool               bHasGate; // true = CommitPlan must answer a decision point
};
```

!!! warning "These AI types are provisional"
    The AI extension surface — `FPAbAiPlan` and the deliberation driver's verbs — may still change in a future version, including in non-additive ways, while the searching-driver shape settles. Build against it, but don't treat its exact shape as frozen the way the config structs you author abilities against are. (The scorer view `FPAbScoreContext` *is* frozen — additive-only, which is why its board and structural-delta fields were appended at the end.)

### Seeing why the AI chose what it chose

Every candidate the AI scores can be captured with its per-consideration arithmetic, then dumped from the console:

- `PAb.AI.ScoreBreakdown 1` — capture the breakdown (`2` also logs each candidate as it scores; `0`, the default, captures nothing and costs nothing).
- `PAb.AI.DumpLastBreakdown` — print the last deliberation's per-candidate breakdown.

The records are also readable in C++ through `GetLastDeliberationScores()` and `FormatLastDeliberationScores()`, which is what a test asserts against. The setting is sampled once per deliberation, so turn it on *before* the turn you want to inspect. How to read the output is covered in [Tuning: the console loop](reference-ai.md#tuning-the-console-loop).

## Repeat-stable rolls

Random rolls are keyed to the game state and to the step making them, and that one property carries three guarantees:

- **A replay reproduces every roll exactly** — the same state and the same step always draw the same number, so a recorded game plays back identically.
- **Re-trying the same action can't fish for a better roll** — because the key is the same, the roll is the same; there is no re-roll by cancelling and re-issuing.
- **The AI reasons over the odds, never over your upcoming roll.** When the AI scores a candidate, its trial runs sample a *separate* stream — it weighs the outcome distribution without reading the specific roll you are about to get at real activation. That is what keeps a telegraphed enemy move honest: it planned against the odds, not against a peek at the dice.

A step draws its numbers through the context it is handed (see [the execution context](reference-programs.md)); it never pulls from a sequential source of its own, which would be invisible to replay and would make a preview disagree with the real thing.

---

**See also:** [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#what-if-runs-previews-and-enemy-ai) (the model) · [Enemy AI: Considerations & Profiles](reference-ai.md) (authoring enemy judgment as data) · [Tokens & Cues](../../concepts/tokens-and-cues.md#previews-ride-the-same-rails) (the preview surfaces) · [TokenSystem reference](../tokensystem/reference.md#projected-actions) (projected actions and capabilities) · [Targeting & Running Abilities](reference-targeting.md) (the targeting session in full).
