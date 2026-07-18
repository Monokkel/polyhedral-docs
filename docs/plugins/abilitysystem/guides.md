# Ability System Guides

For developers with the plugin enabled who want to put a working ability on a
unit and drive it. Each section is a numbered, followable recipe that opens with
what it accomplishes. For the model behind them read
[Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md);
for the type-by-type surface see the reference pages —
[Programs & Steps](reference-programs.md),
[Targeting & Running Abilities](reference-targeting.md),
[Triggers & Reactions](reference-triggers.md),
[Previews, AI & What-Ifs](reference-previews.md), and
[Configuration, Tags & Tooling](reference-tooling.md).

## Make your first ability

Get a working, undoable, previewable ability onto a unit and fire it — without
authoring a program by hand. This is the 80% path: point the ability's row at a
ready-made program and differentiate it with ordinary stats and tags.

=== "Blueprint"
    1. Author the ability as a data-table row, exactly like any other entity. Add
       a **Tagged Data** entry keyed `Data.Ability.Program` and set its **Program
       Class** to the built-in area-damage program (`UPAbComposite_AoEDamage`).
    2. Add a `Data.Ability.Activation` entry — its presence is what marks the
       entity as something a player can manually activate.
    3. Author the numbers the program reads as ordinary stats and tags on the row:
       the damage stat and the element tag. That is the whole ability, no graph —
       a different area spell is the same program with different numbers.
    4. Grant the row to the caster as a child in an ability slot with the ordinary
       entity-hierarchy call. Because it is entity state, the grant undoes and
       saves with the unit.
    5. On the "cast" input, get the resolution subsystem and call **Try Activate
       Ability** — the first pin is the Caller (the unit acting), the second is the
       Ability (the granted entity). Both pins are the same type, so mind the order.
    6. Read **Get Last Order Resolution**: *committed*, *resolved-but-changed-
       nothing*, or *aborted*. Surface the middle case as "no valid target"
       instead of a dead-feeling click.

=== "C++"
    ```cpp
    // 1. Point the ability's program entry at a ready-made program — by SOFT
    //    class, so the row never hard-loads every program at load time.
    FPAbProgramRef Program;
    Program.ProgramClass = UPAbComposite_AoEDamage::StaticClass();
    // Store Program as the row's Data.Ability.Program value. The damage stat and
    // element tag authored on the row are what make THIS one a fireball — the
    // program is shared; the differentiation lives on the entity.

    // 2. Grant the ability as a child in an ability slot (an undoable
    //    entity-hierarchy change — the grant call lives in the GameEntity section),
    //    then activate. Caller first, Ability second (both FPGeEntityRef).
    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);
    if (Rules->TryActivateAbility(Caster, GrantedAbility))
    {
        // Once it has resolved, tell a real commit from a no-op.
        if (Rules->GetLastOrderResolution() == EPAbOrderResolution::ResolvedEmpty)
        {
            // Ran to completion but committed nothing — e.g. the blast caught
            // no one. The honest "I clicked and nothing happened" diagnostic.
        }
    }
    ```

!!! note "The ready-made library is the starting point"
    Deal damage in an area, apply a status, buff self, summon, move to a cell —
    the built-in programs cover most abilities as a data row plus a reference.
    The full library, and how to author a custom program or a custom step when
    you outgrow it, are in [Programs, Modules & Custom Steps](reference-programs.md).
    To grey the cast button when the ability is not usable and explain why, see
    the availability checks in [Previews, AI & What-Ifs](reference-previews.md).

## Add a target-selection decision point

Make an ability pause and ask the player — or a preview, or the AI — to pick its
target, then answer that request. A decision point is a step that needs a choice
before resolution can continue; when it is reached, resolution suspends and
publishes a **decision request** describing the prompt, the legal candidates, and
how many may be picked.

Two ways to get one into a program: reference a ready-made program that already
contains one (ranged single-target damage picks a target in range), or add a
**Gate: Select Targets** step to a custom program.

The fast way to answer it is the framework's **reference targeting driver**: drop
an `APAbTargetingDriver` into the level and call **Bind To Resolution** on it — it
subscribes to decision requests, debug-draws the candidates, and takes
click-to-pick, submit, decline, and cancel. It is example-grade (debug-draw); for
a shipped game, replace it with your own UI (see
[Wire a custom targeting UI](#wire-a-custom-targeting-ui) below). To answer a
request directly:

=== "Blueprint"
    1. Subscribe to **On Decision Requested** on the resolution subsystem — it
       hands over the request: the prompt, the candidate list, and the min/max
       selection count.
    2. Build your chosen targets from the request's **Candidates** — pick them by
       index. Do not construct a target by hand: a raw struct literal leaves the
       target's internal facets unpopulated, so downstream matching misbehaves.
    3. Call **Submit Gate Input**, passing the request's handle and the picked
       candidates. A **Decline Gate** with the same handle is a normal empty
       answer — resolution continues; **Cancel Order** rolls the whole activation
       back. Decline is skip; cancel is undo.
    4. Always read the handle fresh from the live request — from the **On Decision
       Requested** payload or **Get Pending Decision Request**. Never store it.

=== "C++"
    ```cpp
    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);

    // Answer whatever request is live right now — read it fresh; never cache the handle.
    FPAbDecisionRequest Request;
    if (Rules->GetPendingDecisionRequest(Request))
    {
        // Pick from the PUBLISHED candidates. Don't build an FPAbTarget by hand —
        // a raw struct literal leaves its facets unset and dedup silently misbehaves.
        TArray<FPAbTarget> Picked = { Request.Candidates[0] };

        Rules->SubmitGateInput(Request.GateHandle, Picked); // answer
        // Rules->DeclineGate(Request.GateHandle);          // skip — empty, continue
        // Rules->CancelOrder();                            // cancel — roll it all back
    }
    ```

!!! warning "Read the handle fresh — never cache it"
    Each time the same decision point suspends it is issued a brand-new handle; a
    submit against a superseded one is silently rejected. Always take the handle
    from the live **On Decision Requested** broadcast or **Get Pending Decision
    Request** at the moment you answer — a handle you stashed a frame ago may
    already be stale.

For where a request's candidates and the cells an ability would strike get
painted onto the board, see
[painting targets on the grid](reference-targeting.md#painting-targets-on-the-grid).

## Author a reactive trigger

Give an entity a rule that fires in response to something happening — an
**interrupt** that reshapes a change before it lands, or a **reaction** that
answers with changes of its own after it commits. A trigger is a data entry the
entity carries, so who is listening undoes, saves, and replays with the world.

A trigger lives in a `Data.Ability.Triggers` entry (an array — multiple per
entity is native). Each one names an event tag, a **channel** (whose window to
listen on: the carrier's own, its parent's, or an entity a target source
resolves), an **order** whose sign selects the phase, an optional **condition**,
and the **program** to run when it fires.

=== "Blueprint"
    1. On the reacting entity — an armor item, a passive, a status effect — add a
       **Tagged Data** entry keyed `Data.Ability.Triggers`.
    2. Add an **interrupt** that softens incoming damage: set **Event Tag** to
       `Stat.Health`, **Channel** to **Self**, **Order** to `-10` (negative =
       interrupt, pre-commit), an optional **Condition**, and a **Program
       Override** whose program halves the proposed amount. It runs synchronously
       — an interrupt must never pause for input.
    3. Add a **reaction** that hits back: a second entry, **Order** `0`
       (zero-or-positive = reaction, post-commit), and a program that deals the
       counter-swing. A reaction *may* pause for input; when it does, it resolves
       at the acting ability's next natural pause — never mid-step.
    4. Grant the carrier to a unit as a child. Because the trigger is entity data,
       who is listening undoes and saves with the unit. Stacking is by
       composition — many child entities each carrying their own trigger, not one
       array you keep editing.

=== "C++"
    ```cpp
    // An interrupt: armor softens an incoming Health change BEFORE it commits.
    // A negative order selects the interrupt phase; it runs synchronously and
    // must not pause.
    FPAbTriggerSpec Soften;
    Soften.EventTag        = HealthTag;                 // react to changes to this stat
    Soften.Channel         = EPAbTriggerChannel::Self;  // on the carrier's own window
    Soften.Order           = -10;                       // < 0 => interrupt (pre-commit)
    Soften.ProgramOverride = UArmorSoftenProgram::StaticClass();

    // A reaction: retaliate AFTER the hit commits. Zero-or-positive selects the
    // reaction phase, which is allowed to pause for input.
    FPAbTriggerSpec Retaliate;
    Retaliate.EventTag        = HealthTag;
    Retaliate.Channel         = EPAbTriggerChannel::Self;
    Retaliate.Order           = 0;                      // >= 0 => reaction (post-commit)
    Retaliate.ProgramOverride = URetaliateProgram::StaticClass();

    // Both are just entries the entity carries; multiple triggers per entity is native.
    FPAbTriggerList Triggers;
    Triggers.Triggers = { Soften, Retaliate };
    // Store Triggers as the entity's Data.Ability.Triggers value.
    ```

!!! note "Order sign is a phase selector, not just a priority"
    The sign of **Order** picks the phase: negative is an interrupt (before the
    change commits — reshape or veto), zero-or-positive is a reaction (after it
    commits). The *magnitude* is also the within-phase sort tiebreak, so a lower
    reaction order resolves first. A "when hit, retaliate" buff granted onto a
    unit sets **Channel** to **Parent** so it reacts for its owner; "at the start
    of my turn" is simply a trigger bound to a turn event on the unit's own channel.

!!! warning "An interrupt cannot prompt a human"
    An interrupt program runs synchronously — a decision point inside one is a
    loud development-build error, not a supported flow. An interrupt can reshape
    a number but cannot stop and ask a player. If a *player* must choose before
    the hit lands, the source's program must open a **declaration window** (the
    ready-made "declared damage" program is the pattern to copy) — a plain
    interrupt offers automatic responses only.

The full trigger surface — accessor-resolved channels, conditions, how the
child's target list is seeded, and declaration windows — is in
[Triggers & Reactions → Authoring triggers](reference-triggers.md#authoring-triggers).
The model is on the
[concept page](../../concepts/abilities-and-resolution.md#triggers-entity-carried-rules).

## Wire a custom targeting UI

Replace the example driver's debug-draw with your own UMG targeting UI, driving
the exact same selection and preview machinery. The reference driver
(`APAbTargetingDriver`) is a worked example; its real value is the reusable
**targeting session** (`UPAbTargetingSession`) underneath it. You build your
widget over the session's calls and resolution does not change. The session is
*used and composed*, never subclassed.

=== "Blueprint"
    1. Create a **Targeting Session** object (**Construct Object From Class** →
       **Targeting Session**, with your widget as its Outer) and store it. Call
       **Bind To Resolution** once, passing the resolution subsystem — it
       subscribes to the decision-request lifecycle and adopts any request already
       published.
    2. Render from the session's read surface: **Has Active Request** / **Get
       Active Request** (prompt, candidates, min/max), **Get Status Text** for a
       banner, **Is Waiting On Reaction** for the "waiting on a reaction" state,
       and **Get Selected Count** for a running count.
    3. On a candidate click, call **Select Candidate** with its index — a
       single-select decision point submits immediately; a multi-select one toggles
       and auto-submits at the maximum (tunable with **Auto Submit At Max**).
       Confirm a multi-selection manually with **Submit Selection**.
    4. **Decline Pending Gate** answers empty and continues; **Cancel Pending
       Order** rolls the activation back.
    5. On hover-enter, call **Preview Gate Input** (Caller, Ability, the hovered
       target) to paint the predicted outcome; on hover-leave call **Clear Gate
       Input Preview**.
    6. Surface one-shot notices (rejections, confirmations): read **Get Notice
       Serial** and, when it changes, post **Get Last Notice** in the colour that
       **Get Last Notice Level** maps to.

=== "C++"
    ```cpp
    // Build the session once (a plain UObject — used, not subclassed).
    UPAbTargetingSession* Session = NewObject<UPAbTargetingSession>(this);
    Session->BindToResolution(UPAbResolutionSubsystem::Get(this));

    // Render from its read surface whenever a request is live:
    if (Session->HasActiveRequest())
    {
        const FPAbDecisionRequest Req = Session->GetActiveRequest();
        // Draw Req.Candidates / Req.Prompt in your widget;
        // Session->GetStatusText() drives a banner.
    }

    // The input verbs — the same ones the reference driver forwards:
    Session->SelectCandidate(Index);   // single-select submits; multi-select toggles
    Session->SubmitSelection();        // confirm a multi-selection
    Session->DeclinePendingGate();     // skip — resolution continues
    Session->CancelPendingOrder();     // whole-activation rollback

    // Hover preview against a temporary, throwaway copy of the game state:
    Session->PreviewGateInput(Caster, Ability, HoveredTarget);
    Session->ClearGateInputPreview();
    ```

!!! tip "Read the driver for the pattern, not to subclass it"
    The reference driver is example-grade and exists to keep the API honest and to
    be the worked example — read how it forwards key presses to the session, then
    drive the session directly from your own UI. The hover-preview calls paint on
    a throwaway copy of the game state; see
    [hover previews](reference-previews.md#hover-previews).

## Plug in a custom AI scorer

Teach the enemy AI your game's judgment of a good move. The
deliberate → telegraph → commit loop runs out of the box with a content-neutral
default scorer (it rewards the magnitude of harm inflicted on entities other than
the caster); *whose* harm is good is your game's value judgment, and you supply
it by binding a scorer.

This is a **C++ extension point** — there is no Blueprint path. The scorer carries
live references to the game state and is not a dynamic delegate, so it cannot be
authored in Blueprint by construction.

```cpp
UPAbAiDeliberationSubsystem* Ai = UPAbAiDeliberationSubsystem::Get(World);

// Bind your judgment. The scorer is a pure float over one candidate's context;
// higher = better. Unbound, the framework's content-neutral default is used.
Ai->SetCandidateScorer(FPAbCandidateScorer::CreateLambda(
    [](const FPAbScoreContext& Ctx) -> float
    {
        // Ctx.Coalesced      — the candidate's net-change stream (what a preview reads)
        // Ctx.ResultantState — the throwaway copy's post-run state, for positional
        //                      judgment (adjacency, threat range) — valid ONLY during this call
        // Ctx.BaselineState  — the untouched live state, for a before/after comparison
        float Score = 0.f;
        // e.g. reward damage dealt to the player faction, penalise friendly fire.
        return Score;
    }));
```

Each enemy turn is three legible phases — each candidate action is tried on a
temporary, throwaway copy of the game state and scored, then the best is
telegraphed and played for real:

```cpp
// One enemy's turn. Deliberate scores every candidate on a throwaway copy and
// keeps the best plan; while it deliberates, live player input is held so the
// board cannot change under it.
FPAbAiPlan Plan = Ai->Deliberate(Enemy, EnemyAbilityRefs);
if (Plan.IsValid())
{
    Ai->TelegraphPlan(Plan);   // paint the "here's what I'll do" overlay — your turn UI owns the dwell
    Ai->CommitPlan(Plan);      // clear the telegraph, then issue the plan for real
}
else
{
    Ai->AbandonDeliberation(); // nothing worth doing — pass
}
```

!!! note "These AI types may still change"
    `FPAbAiPlan` and the AI-deliberation types are provisional — additive-only as
    a discipline, but they may still change non-additively while the searching
    driver is proven out. The scorer context and the decision-request shapes are
    stable. Because the scorer runs while the throwaway copy is still alive, it can
    judge the resulting *position*, not just the change stream — and the AI samples
    its own separate roll stream, so it reasons over the odds, never the specific
    roll you are about to get.

The full what-if model and the rest of the AI surface are in
[Previews, AI & What-Ifs](reference-previews.md).

## Validate an ability

Catch authoring mistakes before you hit play, and confirm an ability behaves
identically whether it runs for real or as a preview. Two rails do this: a static
**ability linter** flags structural mistakes in a program (a state-changing step
on an override step, a dangling back-reference to an earlier step, a duplicate
handle, a cyclic sub-program), and a per-ability **Validate Ability** editor
action runs the linter *plus* an automated consistency check.

=== "Editor"
    1. In the Content Browser, right-click an ability program asset and choose
       **Validate Ability**.
    2. Read the results in the **Ability Validation** Message Log — the linter
       findings, then the consistency verdict.

=== "C++"
    ```cpp
    // Run the same linter + consistency harness headlessly (e.g. in a test).
    FPAbValidationReport Report = FPAbAbilityValidator::Validate(MyProgramClass);

    if (Report.HasWarningsOrErrors())
    {
        // Report.Lint.Findings   — structural findings from the linter
        // Report.Harness.Status  — Passed / Indeterminate / Diverged / ...
    }
    ```

The consistency verdict has three outcomes worth knowing:

- **Diverged** — the real run and the what-if run of the *same* ability produced
  different results. This is a genuine bug: a step reached around the one
  authoring rule (a raw actor spawn, a sequential random draw, an off-books
  mutation), so its preview will disagree with the real thing. Fix it.
- **Indeterminate** — the ability had nothing to act on without a fixture (no
  target, no enemy to hit), so the harness could not produce a comparison. This
  is **not a failure** — the tool reports Indeterminate rather than a false pass.
  Give it a richer fixture (a seed stat, a target) to exercise a real change.
- **Passed** — the real and what-if runs matched.

!!! note "The linter runs either way"
    The structural linter runs even when the consistency harness reports
    Indeterminate, so a program that needs a fixture still gets its structure
    vetted. The full check-id list, the project-settings knobs, and the native-tag
    reference live in [Configuration, Tags & Tooling](reference-tooling.md).
