# Authoring Abilities: Programs & Steps

For developers building an ability's behavior — from referencing a ready-made
program on a data-table row to writing a bespoke step in Blueprint or C++. After
reading you'll know the shipped program library (the fast path), the anatomy of a
**step**, how steps hand each other a **target list**, how to nest and loop, the
data an ability carries on its row, and the context every custom step is handed.
For the model behind all this, read
[Abilities and Step-by-Step Resolution](../../concepts/abilities-and-resolution.md#a-program-of-steps);
for how an ability is *run*, targeted, and driven, see
[Targeting & Running Abilities](reference-targeting.md).

The AbilitySystem plugin's public types use the `PAb` prefix — `UPAbAbilityModule`,
`FPAbProgramRef`, and so on. Signatures below are hand-written to show the shape of
each API; they are illustrative, not source excerpts.

!!! note "A program is data, not a graph"
    An **ability program** is an ordered list of steps authored as data — the list
    *is* the ability's logic. No event graph runs it. You rarely assemble one by
    hand: most abilities reference a ready-made program straight from their row and
    differ only in the stats and tags the steps read.

## The ready-made program library

The framework ships nine ready-made programs. Reference one by soft class from an
ability's `Data.Ability.Program` entry and you have a working ability — no steps to
wire. What makes two abilities that share a program *different* is the stats and
tags on their rows: a fireball and an ice lance are the same area-damage program
pointed at a different damage stat and element.

| Program (class) | What it does |
|---|---|
| `UPAbComposite_SingleTargetDamage` | Deal a computed amount of damage to a single target. The amount reads the ability's damage stat. |
| `UPAbComposite_AoEDamage` | Deal damage to every unit within a radius. Batched so the whole area settles in one stat recompute. |
| `UPAbComposite_ApplyStatus` | Add a status tag to the target(s). |
| `UPAbComposite_BuffSelf` | Add a computed amount to a stat on the caster. |
| `UPAbComposite_Summon` | Create an entity from a template, optionally parented under the caster. |
| `UPAbComposite_DeclaredDamage` | Open a declaration window *before* the damage lands — a reaction can counter it pre-hit — then deal the damage. The pattern to copy for "react before the hit." |
| `UPAbComposite_MoveToCell` | Present the caster's reachable cells, then move the caster to the chosen one. |
| `UPAbComposite_TargetedDamage` | Present the units in range, then deal damage to the chosen one. Range differs per row via a range stat. |
| `UPAbComposite_ChainStatusThenDamage` | Apply a status, then deal damage — each a nested ready-made program. The worked example of composing programs. |

Two abilities using the same program differ only in the numbers and tags on the
row (the damage stat, the element, the status tag, the radius or range stat, the
summon template). For finer control — *which* stat the damage reads, the element
tag, the status tag — make a small Blueprint subclass of the program and pin those
on it; the shipped C++ form sets only structural fields (scale, batch flag, radius),
because gameplay tags can't be baked into it safely.

=== "Blueprint"
    1. In the ability's row, add a **Tagged Data** entry keyed `Data.Ability.Program`.
    2. Set its **Program Class** to a library program (or to your Blueprint subclass
       of one).
    3. Author the ability's damage amount, radius, and status tag as ordinary stats
       and tags on the row. That is the whole ability — no graph.

=== "C++"
    ```cpp
    // An ability references its program by SOFT class, so a row never hard-loads
    // every program at load time. Point it at a ready-made one:
    FPAbProgramRef Program;
    Program.ProgramClass = UPAbComposite_AoEDamage::StaticClass();

    // The row's damage stat and element tag are what make THIS one a fireball —
    // the program is shared; the differentiation lives on the entity.
    ```

!!! note "The class names are a stable authoring contract"
    A shipped program's class name is part of the marketplace contract: it is never
    renamed out from under the rows that reference it, and the config a program reads
    only ever grows. Authored content does not silently break on a framework upgrade.

## Anatomy of a step

A **step** is one small unit of an ability — find targets, ask for a choice, change
game state, open a response window, show a marker. You make a new kind of step by
subclassing the **Ability Module** class (`UPAbAbilityModule`) and implementing one
body. A step instance carries only the config a designer sets on it; it holds no
per-run state of its own.

Every step exposes the same three authored fields:

```cpp
class UPAbAbilityModule   // the base every step derives from
{
    EPAbOperation Operation;  // how this step's results combine with the target list
    FString       Label;      // an optional human-readable name (debug / dropdowns)
    FGuid         Handle;      // a stable id keying this step's recorded results
};
```

`Handle` is normally left blank — the framework assigns a deterministic one — and
matters only if another step reaches back to read this step's results by handle
(see [Repeat-stable rolls](#repeat-stable-rolls) and cross-step reads below).

### The override points

A step's body goes in one of three places, depending on whether it needs to wait:

```cpp
// The Blueprint sync body. The engine completes the step for you after it returns.
void Execute(UPAbExecContext* Ctx);

// The async entry. Override this (in Blueprint or C++) when the step must WAIT for
// something — a decision point, a window. You then call the context's completion
// yourself when the input arrives.
void ExecuteAsync(UPAbExecContext* Ctx);

// The C++ sync override point. The built-in steps use this and let the base
// complete them. (protected; C++ only.)
void NativeExecute(UPAbExecContext* Ctx);
```

A step never returns its output — it writes results to the context, so synchronous
and waiting steps produce their output identically. See
[Writing a custom step](#writing-a-custom-step) for the full picture.

### Combining results into the target list

Steps hand each other a running **target list**. Each step receives the list
produced so far, contributes its own results, and declares — through its
**Operation** setting — how the two combine:

| Operation | What it does to the target list |
|---|---|
| `Override` | Replace the list with this step's results, verbatim (duplicates preserved). The default for a targeting step. |
| `Ignore` | Leave the list untouched — pass it straight through. The default for a state-changing or display step. |
| `Union` | Merge this step's results into the list (a true set union — deduplicates). |
| `Difference` | Remove this step's results from the list ("cut a hole"). |
| `Intersect` | Keep only the targets in both. |

Two defaults carry most authoring. A **targeting step** defaults to `Override`, so
it replaces the list with whatever it found. A **state-changing step** defaults to
`Ignore` (leave the list untouched), so a "deal damage" step acts on exactly whoever
the targeting steps found and never goes hunting for its own targets — and a second
state-changing step after it still sees the full set.

!!! warning "`Operation` governs the merge, never the targets a step acts on"
    A state-changing step *always* acts on the incoming target list. `Operation`
    only decides what the list looks like for the *next* step. `Override` is a
    verbatim replace, not a set operation — a targeting step emitting three copies
    of the same target threads three copies downstream; only `Union` deduplicates.

## The building-block steps

When you assemble a program by hand, these are the shipped steps you reach for.

| Step (class) | Role | You configure |
|---|---|---|
| `UPAbTargeter_CallerLocation` | targeting | nothing — seeds the list with the caster itself |
| `UPAbTargeter_GridCircle` | targeting | an anchor source, a radius, what to emit |
| `UPAbEffect_StatChange` | state-changing | a stat tag + a flat delta |
| `UPAbEffect_StatChangeByMagnitude` | state-changing | a stat tag + a computed magnitude + a scale |
| `UPAbEffect_TagChange` | state-changing | a tag + add-or-remove |
| `UPAbEffect_Summon` | state-changing | a template id + an optional parent slot |
| `UPAbEffect_MoveAlongPath` | state-changing | nothing — moves the caster to the chosen cell |
| `UPAbDisplay_Marker` | display | an optional marker tag |

The state-changing steps route their mutation through the entity system's
command-routed, *windowed* helpers, so every change is reactable, folds into the
activation's single undo unit, and is undone for free. The zero-config steps are
useful precisely because they read from context: `UPAbTargeter_CallerLocation`
seeds the caster; `UPAbEffect_MoveAlongPath` moves the caster to the destination
cell the target list supplies (resolving a legal path under the caster's own
movement profile — no path means no move, never a teleport).

The two most-configured steps:

```cpp
// A targeting step: every cell within Radius graph-hops of each anchor, emitted as
// cells and/or the units standing on them.
class UPAbTargeter_GridCircle : public UPAbAbilityModule
{
    UPAbSourceAccessor* Source;                          // where to anchor; unset = the incoming target list
    int32 Radius = 1;                                    // reach in graph-hops (0 = the anchor cell only)
    TInstancedStruct<FEvalMagnitudeCalc> RadiusOverride; // optional: read the radius from a stat instead
    bool bIncludeCenter    = true;                       // include the anchor cell(s)
    bool bIncludeCells     = true;                       // emit one target per cell
    bool bIncludeOccupants = true;                       // emit one target per unit on those cells
};

// A state-changing step: change a stat by an amount COMPUTED from a formula that
// reads the caster's / ability's stats — so one step serves many rows.
class UPAbEffect_StatChangeByMagnitude : public UPAbAbilityModule
{
    FGameplayTag StatTag;                                // which stat to change on each target
    TInstancedStruct<FEvalMagnitudeCalc> Magnitude;      // the amount, evaluated once from the caster's context
    float Scale = -1.f;                                  // -1 = damage, +1 = heal / buff
    bool  bBatchRecompute = false;                       // wrap a bulk (area) application in one recompute
    FGameplayTag ElementTag;                             // optional element, carried for resistance / cue selection
};
```

`Source` is a **target source** — a small pluggable resolver ("the caster," "the
previous step's results," "cells reachable from here"); the family is documented in
[Targeting & Running Abilities](reference-targeting.md). `Magnitude` and
`RadiusOverride` are [Evaluators](../evaluators/reference.md) calculations, read
from the caster's stats at run time. Namespace hints like `ElementTag` and
`UPAbDisplay_Marker`'s marker tag are conventions your game branches on — they are
not enforced.

### Assembling a program by hand

A bespoke program is a data-only Blueprint class whose class-defaults list of steps
is the program. The class itself never runs logic.

=== "Blueprint"
    1. Create a Blueprint subclass of the program class (**Ability Program**).
    2. In its class defaults, add three entries to the **Modules** list:
       **Targeter: Caller**, then **Targeter: Grid Circle** (set its **Radius**),
       then **Effect: Stat Change (By Magnitude)** (set its **Stat Tag** and
       **Magnitude**).
    3. Reference this program from an ability row's `Data.Ability.Program` entry,
       exactly as you would a library program.

=== "C++"
    ```cpp
    // A hand-authored area blast: anchor on the caster, gather every unit in a
    // 2-hop circle, deal computed damage to them. (This is close to what
    // UPAbComposite_AoEDamage ships as — shown here to illustrate assembly.)
    class UMyBlast : public UPAbAbilityProgram
    {
        UMyBlast()
        {
            Modules.Add(CreateDefaultSubobject<UPAbTargeter_CallerLocation>(TEXT("Anchor")));

            UPAbTargeter_GridCircle* Circle = CreateDefaultSubobject<UPAbTargeter_GridCircle>(TEXT("Circle"));
            Circle->Radius = 2;
            Circle->bIncludeOccupants = true;
            Circle->bIncludeCells = false;
            Modules.Add(Circle);

            Modules.Add(CreateDefaultSubobject<UPAbEffect_StatChangeByMagnitude>(TEXT("Damage")));
        }
    };
    ```

Follow the target list through those three steps: the caller step seeds it with the
caster and (defaulting to `Override`) makes that the list; the grid-circle step
anchors on that incoming list, then replaces it with the units it finds in the
circle; the damage step leaves the list untouched and deals damage to each of them.

!!! tip "The build-graph alternative"
    A program Blueprint may instead implement a **Build Program** graph that *emits*
    the same list of steps once at first use, for programs assembled from external
    data. It is data-only — it constructs steps, it never runs one — and it is
    mutually exclusive with the class-defaults list: if both are authored, the
    class-defaults list wins.

## Nesting: sub-programs and per-target loops

Two steps run *other* steps, which is how the ready-made library composes larger
abilities out of smaller ones.

```cpp
// A step that runs a whole sub-chain as a single step in the parent program.
class UPAbModule_SubProgram : public UPAbAbilityModule
{
    TSoftClassPtr<UPAbAbilityProgram> Program;  // a shared program to run (wins when set)
    TArray<UPAbAbilityModule*>        InlineModules; // OR a one-off inline sub-chain
};
```

`UPAbModule_SubProgram` (**Sub-Program (Nested)** in the editor) runs a referenced
program — or an inline sub-chain — as one step, then merges the sub-chain's final
target list back into the parent through its own `Operation` (default `Override`).
Its threading stays hidden from the parent, which is exactly what lets a program be
reused as a building block: `UPAbComposite_ChainStatusThenDamage` is nothing but two
sub-program steps, one per nested library program.

`UPAbModule_ForEach` (**For-Each (Nested)**) is its iterating sibling. It runs its
body — the same two shapes, a referenced program or an inline sub-chain — **once per
item** in the incoming target list, each iteration seeded with that single item. Its
default `Operation` is `Ignore` (the per-iteration effects have already changed
state, so it passes the incoming list through). This is what a "for each enemy the
blast struck, do X" ability rides on: a for-each over the struck units whose body is
its own little targeting-plus-effect chain.

## The data an ability carries

An ability carries its authoring data as `Data.Ability.*` tagged-data entries on its
row. The `Data.Ability.Program` entry (above) points at the program; four more shape
how the ability is offered and gated.

```cpp
// Data.Ability.Program — which program drives this entity's activation / triggers.
struct FPAbProgramRef
{
    TSoftClassPtr<UPAbAbilityProgram> ProgramClass;  // the program (Blueprint class); used when set
    TSoftObjectPtr<UPAbProgramAsset>  ProgramAsset;  // a forward data-asset door (see note)
};

// Data.Ability.Intent — the presentation intent the automatic playout layer reads.
struct FPAbAbilityIntent
{
    FGameplayTag Intent;  // an Action.* tag; unset = the generic Action floor
};

// Data.Ability.Activation — its PRESENCE is what makes the entity manually activatable.
struct FPAbActivationSpec
{
    bool         bPipelinedCompletion = false;  // opt-in completion-policy override
    FGameplayTag UICategory;                     // optional UI grouping tag (presentation only)
};

// Data.Ability.UseConditions — the cheap "can this be used right now?" check.
struct FPAbUseConditions
{
    TArray<TInstancedStruct<FEvalConditionCalc>> Conditions;  // implicit AND; empty = always usable
};
```

`FPAbActivationSpec`'s mere *presence* on an entity is what marks it as manually
activatable — its fields are optional refinements. `FPAbUseConditions` is a list of
boolean [Evaluators](../evaluators/reference.md) conditions, combined with implicit
AND ("only in the forest," "not while silenced," "enough mana"); it is what the cheap
per-frame availability check evaluates and can explain, condition by condition — see
[Targeting & Running Abilities](reference-targeting.md). `FPAbAbilityIntent` carries
an `Action.*` tag that the
[automatic playout layer](reference-previews.md#the-automatic-playout-layer) uses to
pick how the activation's changes present. A fifth facet, `Data.Ability.Triggers`,
makes the entity react to events — it belongs to
[Triggers & Reactions](reference-triggers.md#authoring-triggers).

!!! note "One door for programs today"
    `FPAbProgramRef` carries two references: a program **class** and a program
    **asset**. The asset door is a forward-looking anchor — no concrete asset type
    ships yet — so the **Blueprint-class program is the door to author against**.
    `ProgramClass` also wins if both are somehow set, so an existing row's behavior
    can never change by adding an asset beside it.

## Writing a custom step

When the shipped steps aren't enough, write your own: a small Blueprint (or C++)
subclass of the **Ability Module** class with a single body. Put the body in
`Execute` (Blueprint, synchronous — the engine completes the step for you) or
`NativeExecute` (the C++ synchronous equivalent). If the step must *wait* for
something, override `ExecuteAsync` instead and call the context's completion
yourself once the input arrives.

!!! warning "The one authoring rule"
    A step touches the game **only** through the entity system's command-routed calls
    and the context it is handed. Anything else — spawning an actor directly,
    mutating the grid behind the system's back, drawing a number from a sequential
    random source — is invisible to undo and to what-if runs, and will make a preview
    disagree with the real thing. Development builds call a broken rule out loudly at
    run time.

### The context handed to each step

The context (`UPAbExecContext`, threaded through every step as `Ctx`) is that door.
Beyond the entity library it hands back, it *is* the step's surface onto the world.
Its methods group into five families:

```cpp
// — Which run is this? (the real thing, or a what-if) —
bool IsLive();                    // a real, kept activation
bool IsSpeculative();             // a preview / AI trial / availability check — never wait for input here
FPGeEntityRef GetCaller();        // the activating unit
FPGeEntityRef GetAbility();       // the ability entity (may be invalid for a standalone program)
UPGeGameStateSubsystem* GetGameState();  // the entity library — your only authoritative-state door
UEvalContext* GetEvalContext();   // seeded with the caster & ability, for magnitude / condition formulas

// — The target list & this step's results —
TArray<FPAbTarget> GetWorkingList();          // the incoming target list
void AddResult(const FPAbTarget& Target);     // contribute one result
void AddResults(const TArray<FPAbTarget>& Targets);
void SetResult(const TArray<FPAbTarget>& Targets);  // replace this step's result wholesale
void RequestEntityRemoval(const FPGeEntityRef& Entity);  // remove an entity from play (two-phase — see note)
void EnqueueBespokeAction(UPTkProjectedAction* Action);  // an entity-less visual (a projectile arc, a banner)
void CompleteABM();               // finish this step — an async / waiting step calls this itself

// — Wait for a decision (async steps only) —
void RequestDecision(const FPAbDecisionRequest& Request);  // suspend at a decision point
void RunAsyncOrPreview(FPAbAsyncOrPreviewBranch LiveAsync,  // the outcome-affecting async seam;
                       FPAbAsyncOrPreviewBranch SpeculativeSync);  //   BOTH branches are required
UTagEventPayloadObject* GetWindowPayload();  // the payload a window-triggered step runs under

// — Read an earlier step's results (by that step's handle) —
bool GetLookbackResult(const FGuid& StepHandle, TArray<FPAbTarget>& Out);
bool GetLookbackOutgoing(const FGuid& StepHandle, TArray<FPAbTarget>& Out);

// — Repeat-stable rolls —
int32 NextDrawIndex(const FGuid& DrawSite);   // pass your OWN step's Handle
float GetKeyedRandomFloat(const FGuid& Site, int32 Index, const FPGeEntityRef& Target);
int32 GetKeyedRandomIntInRange(const FGuid& Site, int32 Index, int32 Min, int32 Max, const FPGeEntityRef& Target);
bool  GetKeyedRandomBool(const FGuid& Site, int32 Index, float TrueChance, const FPGeEntityRef& Target);
```

A note on each family:

- **Which run is this.** A step that would do something irreversible or player-facing
  should check `IsSpeculative()` first: a what-if run must never stop to wait, and
  most steps simply behave the same in both — but a step that plays a montage or
  waits on the network must not do so during a preview.
- **The target list & results.** Read the incoming list with `GetWorkingList`; act on
  the entities in it; record what you affected with `AddResult`. `RequestEntityRemoval`
  is the sanctioned "a rule genuinely removes this from play" door. `EnqueueBespokeAction`
  is for a visual that has no backing state change (a projectile arc, a screen shake).
- **Wait for a decision.** `RequestDecision` suspends the step at a decision point;
  the built-in target-picking step does this for you — see
  [Targeting & Running Abilities](reference-targeting.md). `RunAsyncOrPreview` is the
  seam for outcome-affecting async work.
- **Read an earlier step's results.** `GetLookbackResult` / `GetLookbackOutgoing`
  fetch a specific earlier step's output by its `Handle`, for a step that must sew
  together results two steps back rather than just consume the running list.
- **Repeat-stable rolls.** Any randomness a step needs comes from here.

!!! danger "Never build an `FPAbTarget` by hand"
    Get targets from the context (`GetWorkingList`, `GetLookbackResult`) or from a
    targeting step — never construct one field-by-field in a Blueprint struct
    literal. A hand-built target leaves its populated-facet bookkeeping unset, so
    downstream membership tests and set operations silently misbehave.

### Repeat-stable rolls

Randomness in a step is **keyed**, not sequential: a roll is a pure function of the
game state, the step making it, and the target it concerns. That is what makes a
replay reproduce every roll exactly, stops a re-cast from fishing for a better one,
and lets an AI trial sample its *own* separate stream. Draw an index for your step,
then read a keyed value with your step's own handle:

```cpp
// A custom state-changing step: a coin-flip strike. Reads the target list, rolls a
// repeat-stable 50/50 per target, and applies damage through the entity library.
void UMyCoinFlipStrike::NativeExecute(UPAbExecContext* Ctx)
{
    UPGeGameStateSubsystem* State = Ctx->GetGameState();  // the only state door
    for (const FPAbTarget& T : Ctx->GetWorkingList())     // act on what targeting found
    {
        if (!T.HasEntity())
        {
            continue;
        }

        const int32 Draw = Ctx->NextDrawIndex(Handle);    // this step's own stable handle
        if (Ctx->GetKeyedRandomBool(Handle, Draw, 0.5f, T.Entity))
        {
            // A command-routed, windowed change — reactable, and undone for free.
            State->ApplyStatChange(T.Entity, DamageStat, -Amount, Ctx->GetCaller());
            Ctx->AddResult(T);                            // record who we actually hit
        }
    }
    // Synchronous: the base completes the step after this returns. A step that must
    // WAIT overrides ExecuteAsync and calls Ctx->CompleteABM() when input arrives.
}
```

!!! warning "Async and standalone caveats"
    - `RunAsyncOrPreview` requires **both** branches — a live async branch and a
      synchronous what-if branch. Leaving the required branch unbound does not fail to
      compile; it fails soft at run time (the step auto-completes and a development
      build warns), so bind both.
    - `RequestEntityRemoval` is **two-phase**: it flags the entity as removed
      immediately, so targeting and effect filters exclude it at once, but the
      structural removal is deferred to a stable settling point. The entity is
      *logically* gone the instant you call it, not structurally gone.
    - Running a single step **standalone** (outside a full program) supports
      **synchronous steps only** — decision-point and window steps run only inside a
      full ability program.

---

Related: [Abilities and Step-by-Step Resolution](../../concepts/abilities-and-resolution.md) ·
[Targeting & Running Abilities](reference-targeting.md) ·
[Triggers & Reactions](reference-triggers.md) ·
[Previews, AI & What-Ifs](reference-previews.md) ·
[Configuration, Tags & Tooling](reference-tooling.md) ·
[Guides](guides.md) · [Ability System overview](index.md)
