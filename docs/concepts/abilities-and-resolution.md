# Abilities and Step-by-Step Resolution

For developers turning entities, events, and turns into playable actions — spells, attacks, item uses, and the passives and counters that answer them. After reading, you'll know how an ability is authored as data on an entity, how the framework's rules engine resolves it one step at a time — pausing for player choices and opponent responses — and how the very same steps, run against a temporary throwaway copy of the game state, power target previews, what-if displays, and enemy AI.

This is the final capstone. It pays off every IOU the earlier pages left open: the [entity-carried rules](events-and-reactions.md) the events page promised, the [start-of-turn triggers](turns-and-scheduling.md) the turns page deferred, and the [prediction driver](tokens-and-cues.md) the presentation page left "waiting to be fed."

Throughout, one running example: **Mind Bomb** — an area blast that lets the caster pick a zone, damages each enemy in it, applies a poison that resists can soften, and can be countered before it lands. Every section below is one more thing Mind Bomb needs.

## An ability is an entity

An ability is not a special class. It is an ordinary [game entity](entities-as-data.md): template-backed, created from a data-table row, and granted to a unit as a child in an ability slot — the same [hierarchy and slots](entities-as-data.md#the-hierarchy-and-slots) machinery a sword or a held card uses, no new concept. Mind Bomb's cooldown, its rank, its damage are ordinary stats; granting it and revoking it are ordinary undoable changes.

What *makes* an entity an ability is the data it carries under `Data.Ability.*`: a reference to its program of steps, the conditions under which it may be used, and any triggers it listens on. Those three faces — an ability you activate, a trigger that fires in response, an always-on passive — are not exclusive kinds. An entity simply has whichever entries it carries, and *any* entity can carry them: a poison status effect carries a trigger the same way an ability does, and the always-on passive face is just the granted modifiers from [Stats and Modifiers](stats-and-modifiers.md) — nothing new to learn.

=== "Blueprint"
    1. Author Mind Bomb as a data-table row, exactly like any other entity — its `Data.Ability.Program`, a `Data.Ability.Activation` marker (its presence is what makes the entity manually activatable), and any `Data.Ability.UseConditions`.
    2. Grant it to the caster as a child in an ability slot with the ordinary hierarchy call. Because it is entity state, the grant undoes and saves with the unit.
    3. On the "cast" input, get the resolution subsystem and call **Try Activate Ability** — Caller is the caster, Ability is the granted Mind Bomb.
    4. Read the activation's result with **Get Last Order Resolution**: it returns *committed*, *ran-but-changed-nothing*, or *aborted*. Light up a "no valid target" message on the middle case — the honest answer to "I clicked and nothing happened."

=== "C++"
    ```cpp
    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);

    // One activation. Returns immediately: true if the action was accepted
    // (its use conditions passed and it began resolving), false if refused.
    if (Rules->TryActivateAbility(Caster, MindBomb))
    {
        // Once it has resolved, this tells a real commit from a no-op —
        // the "I clicked and nothing happened" diagnostic.
        if (Rules->GetLastOrderResolution() == EPAbOrderResolution::ResolvedEmpty)
        {
            // e.g. nothing was in the blast — tell the player why.
        }
    }
    ```

## A program of steps

An ability's behavior is an **ability program**: an ordered list of **steps**, authored as data. No event graph runs the ability — the program *is* the ability's logic. Each step does one small thing: find targets, ask for a choice, change game state, open a response window, show a marker.

The steps hand each other a running **target list**. Each step receives the list produced so far, contributes its own results, and declares how the two combine — replace the list, merge into it, or leave it untouched. Leaving it untouched is the default for state-changing steps, so a "deal damage" step acts on exactly whoever the targeting steps found and never goes hunting for its own targets. A later step can also reach back and read a specific earlier step's results directly.

For Mind Bomb that is four steps: anchor on the caster's location, gather every enemy in the blast radius, deal the damage, apply the poison. You rarely wire that by hand, though — the framework ships **ready-made programs** (deal damage in an area, apply a status, summon, move to a cell) that most abilities reference straight from their row. A fireball and Mind Bomb are the *same* area-damage program pointed at different damage stats and element tags; authoring one is a data row plus a reference, not a graph. Bespoke programs stay fully supported — the ready-made library is reuse, not a ceiling.

=== "Blueprint"
    1. In Mind Bomb's row, add a **Tagged Data** entry keyed `Data.Ability.Program`.
    2. Set its **Program Class** to the built-in area-damage program.
    3. Author the damage amount, radius, and poison tag as ordinary stats and tags on the row. That is the whole ability — no graph. A different area spell is the same program with different numbers.

=== "C++"
    ```cpp
    // An ability references its program by SOFT class, so a row never hard-loads
    // every program at load time. Point it at a ready-made one:
    FPAbProgramRef Program;
    Program.ProgramClass = UPAbComposite_AoEDamage::StaticClass();

    // The row's damage stat and element tag are what make THIS one Mind Bomb —
    // the program is shared, the differentiation lives on the entity.
    ```

## How an ability resolves

The rules engine that resolves an ability step by step — the **resolution subsystem** — runs the program in order. Three properties carry the whole page:

- **One activation is one undo step.** Everything the program commits, *including every reaction it sets off*, folds into a single transaction — the same [cascade rule](events-and-reactions.md#a-cascade-is-one-undo-step) the events page established. One press of Undo removes Mind Bomb, the damage, the poison, and any retaliation it provoked, together.
- **One activation resolves at a time.** Issue a new intent while Mind Bomb is still mid-targeting and it simply *replaces* the old one — nothing had committed yet. Issue it after anything has committed and it is *refused*: committed state never silently unwinds, so the player cancels explicitly instead.
- **State commits first; the screen plays it out after.** Between pauses, resolution is instant and invisible — Mind Bomb takes every enemy from full to wounded in one commit — and the animation is layered on afterward. That is exactly the [two-tier model](tokens-and-cues.md#state-commits-instantly-the-screen-plays-it-out) the presentation page opened with. Cancelling mid-way rolls the whole activation back cleanly.

!!! note "The nothing-happened diagnostic"
    After an activation resolves, a game can ask how it ended — committed with a real change, *resolved but changed nothing*, or aborted. The middle case is the one worth surfacing: a targeting step that found no one, or an effect that was fully resisted, will run to completion and commit *nothing*. Reading that result is how you replace a dead-feeling button click with "no valid target in range."

## Targeting and decision points

Resolution runs until it reaches a **decision point** — a step that needs a choice before it can go on: pick a target, choose a cell, confirm Mind Bomb's zone. There it suspends and publishes a **decision request** describing what is being chosen, the legal candidates, and the selection constraints (how many may be picked).

Whatever supplies the answer is pluggable — the player's UI in real play, a hovered target in a preview, the AI's candidate during its deliberation. One request shape, three suppliers. The framework ships a **reference targeting driver** that is clearly example-grade: it debug-draws the candidates and takes click-to-pick, submit, decline, and cancel. Its real value is the reusable **targeting session** underneath it — a shipped game replaces the debug driver with its own UMG targeting UI over the *same* session calls, changing nothing about how resolution works.

Declining a decision point is a normal, empty answer: resolution continues (Mind Bomb with no zone chosen simply does nothing). Cancelling is different — it aborts the whole activation and rolls it back. On a grid, the candidate cells and the cells Mind Bomb would strike are painted through the [grid visualization channels](../plugins/gridgraph/reference-visualization.md) the GridGraph section already documented — this is the driver that section said would arrive to feed its target channel.

=== "Blueprint"
    1. Subscribe to **On Decision Requested** (or let the reference targeting driver do it for you) — it hands over the prompt, the candidate list, and the min/max selection count.
    2. On a click, call **Select Candidate**. A single-select choice submits immediately; a multi-select one accumulates and auto-submits once it reaches the maximum.
    3. **Decline Pending Gate** answers empty and lets resolution continue; **Cancel Pending Order** rolls the whole activation back. Decline is skip; cancel is undo.

=== "C++"
    ```cpp
    // Answer the published decision request directly — exactly what the reference
    // driver and any custom UMG targeting UI both call under the hood:
    Rules->SubmitGateInput(Request.GateHandle, ChosenTargets); // pick target(s)
    Rules->DeclineGate(Request.GateHandle);                    // skip — empty, continue
    Rules->CancelOrder();                                      // cancel — roll it all back
    ```

## Triggers: entity-carried rules

This is the payoff the events page promised. It taught reaction windows and said the durable way to author outcome-affecting listeners was still to come — this is it. A **trigger** is a data entry on an entity that says: *which event tag* to react to (a stat, a tag, a declared moment), *whose window* to listen on (my own, my owner's, or someone a target source resolves), *an order*, an optional condition, and *which program to run* when it fires.

Because the trigger is entity data, *who is listening* undoes, saves, and replays with the world — the events page's "[rules listeners are part of game state](events-and-reactions.md#observers-and-rules-listeners-are-different-populations)" made concrete. And the order's sign picks the phase, exactly as it did on the events page:

- **Negative order is an interrupt** — it runs *before* the change commits and can reshape the proposed amount or veto it outright. An interrupt must run without pausing; if its program tries to stop for a choice, the framework says so loudly in a development build rather than letting it stall the hit.
- **Zero-or-positive order is a reaction** — it runs *after* the change commits and answers with changes of its own. A reaction *may* pause for input, and if it does, it resolves at the acting ability's next natural pause — never mid-step. So an opponent's retaliate passive can never wedge Mind Bomb between its steps; it waits for a place its author already allowed a pause.

Some responses have to happen *before* the effect, though — the counterspell moment. For that, an ability opens a **declaration window** on purpose ("I am about to cast Mind Bomb"), giving an opponent a chance to respond before it lands. Be clear on the boundary: a plain windowed stat change offers *automatic* interrupts only. If a *player* must choose before the hit, the source has to be a program that opens a declaration window — a reshape-the-number interrupt cannot prompt a human.

Stacking is by composition, not by editing an array: a "when hit, retaliate" buff granted onto a unit is a *child entity* carrying its own trigger, the same granted-modifier pattern from Stats and Modifiers. Many sources, many child entities, each with its own trigger. And the turns page's loose end closes here in one line: **"at the start of my turn"** is just a trigger bound to a turn event on the unit's own channel.

=== "Blueprint"
    1. On the reacting entity — an armor item, a passive, a status effect — add a **Tagged Data** entry keyed `Data.Ability.Triggers`.
    2. Add an **interrupt** that softens incoming damage: set **Event Tag** to `Stat.Health`, **Channel** to **Self**, **Order** to `-10` (negative = interrupt, pre-commit), an optional **Condition**, and a **Program Override** that halves the proposed amount. It runs synchronously — no pauses.
    3. Add a **reaction** that hits back: a second trigger, **Order** `0` (post-commit), and a program that deals the counter-swing. A reaction is allowed to pause for input; when it does, it resolves at the acting ability's next natural pause.

=== "C++"
    ```cpp
    // An interrupt: armor softens incoming damage. Negative order = pre-commit,
    // and it runs synchronously — an interrupt never pauses.
    FPAbTriggerSpec Armor;
    Armor.EventTag        = HealthTag;                  // react to changes to this stat
    Armor.Channel         = EPAbTriggerChannel::Self;   // on MY own window
    Armor.Order           = -10;                        // interrupt phase (sign picks it)
    Armor.ProgramOverride = UArmorSoften::StaticClass();

    // A reaction: retaliate after being hit. Zero-or-positive = post-commit.
    FPAbTriggerSpec Retaliate;
    Retaliate.EventTag        = HealthTag;
    Retaliate.Channel         = EPAbTriggerChannel::Self;
    Retaliate.Order           = 0;                       // reaction phase
    Retaliate.ProgramOverride = URetaliateSwing::StaticClass();

    // Both are just data entries the entity carries — they undo, save, and
    // replay with it. Multiple triggers per entity is native.
    FPAbTriggerList Triggers;
    Triggers.Triggers = { Armor, Retaliate };
    ```

## What-if runs: previews and enemy AI

Here is the honest story, stated plainly: **previews and AI what-ifs run the action against a temporary, throwaway copy of the game state and discard it afterward.** That is the whole mechanism, and there is nothing more to explain.

It works because [the entire game is one value of plain data](entities-as-data.md) — the payoff paragraph the entities page promised, now cashed in. The framework copies that value, runs the *real* steps against the copy (real rules, real triggers, real target math), reads back the summarized set of changes the run produced, and throws the copy away. The real game is never touched, and live listeners stay silent for the duration, so no on-screen surface reacts to a move that did not really happen. Three consumers ride that one mechanism:

- **Hover previews.** Hovering an enemy answers Mind Bomb's pending decision point on a throwaway copy; the predicted deltas feed the [preview surfaces and ghosts](tokens-and-cues.md#previews-ride-the-same-rails) the presentation page left waiting — the health bars that paint the red band they *would* lose.
- **Availability.** The cheap per-frame button check just evaluates the ability's declared use conditions — and can name exactly *which* condition failed, so a greyed button gets a real tooltip. The heavier on-click check runs a what-if forward to the first decision point, confirming a legal target actually exists before committing the player to targeting.
- **Enemy AI.** The AI tries each candidate action on a throwaway copy, scores the outcome, telegraphs its chosen action through the *same* preview overlay a player hover uses, then plays it for real. While it is deliberating, player input is held so the board cannot change under it. The deliberate-telegraph-commit loop runs out of the box with a content-neutral scorer; teaching the AI your game's own judgment of a good move is a small piece of C++.

!!! tip "Repeat-stable rolls"
    Random rolls are keyed to the game state and to the step making them. So a replay reproduces every roll exactly, re-trying the same action can't fish for a better one, and the AI samples its *own* separate stream — it reasons over the odds, never over the specific roll you are about to get.

=== "Blueprint"
    1. Bind the ability button's enabled state to **Can Use** (pure, per-frame) — it just checks the declared use conditions.
    2. On a greyed button, call **Explain Can Use** to name the specific failing condition for a tooltip ("on cooldown", "not enough mana").
    3. On click, call **Can Activate** — it runs a quick what-if to the first decision point to confirm a legal target exists.
    4. On hover-enter, call the targeting session's **Preview Gate Input** with the hovered target to paint the predicted outcome; on hover-leave, call **Clear Gate Input Preview**.

=== "C++"
    ```cpp
    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);

    // Per-frame: is the button live? (cheap — just the declared use conditions)
    Button->SetEnabled(Rules->CanUse(Caster, MindBomb));

    // Why is it greyed? Name the failing condition(s).
    TArray<FPAbUseConditionResult> Reasons;
    Rules->ExplainCanUse(Caster, MindBomb, Reasons);

    // On click: a what-if to the first decision point — does a legal target exist?
    if (Rules->CanActivate(Caster, MindBomb)) { /* enter targeting */ }

    // On hover: paint the predicted outcome through the targeting session,
    // and tear it down cleanly when the hover moves on.
    Session->PreviewGateInput(Caster, MindBomb, HoveredTarget);
    Session->ClearGateInputPreview();
    ```

## From committed change to the screen

The presentation page left one loose end: it said you play cues yourself today, and that turning committed changes into cues *automatically* was work the ability system does. With the ability system in play, that upgrade arrives. An **automatic playout layer** watches the changes a resolution commits and turns them into the cues the presentation page documented — choosing snap-versus-sequence, batching one step's changes into coherent beats (all of Mind Bomb's damage numbers as one wave, not a stutter of single hits), and, during a hover, feeding the preview overlays instead of the live cue rail.

Nothing you built on the [tokens page](tokens-and-cues.md) changes: same cues, same handlers, same capabilities. This layer is simply the "somebody" that now plays them for you during an ability.

!!! tip "Undo waits politely"
    The undo button is honored once playout has drained — never mid-animation. So a rewind never fights the screen: the beats finish, *then* the state rolls back and every surface snaps to the restored truth.

## Authoring and safety rails

Programs are **data-only Blueprint classes**. The class-defaults list of configured steps *is* the program; the class never runs logic of its own. (There is an alternative — a build graph that *emits* the same list once at init — but the two are mutually exclusive, and the class-defaults list wins if both are authored.) A program has no variables, no event bindings, no latent nodes. Custom steps and custom target sources, when you need them, are small Blueprint subclasses with a single body to implement.

One rule governs a step's body, and everything else depends on it: **a step touches the game only through the entity system's command-routed calls and the context it is handed.** Anything else — spawning an Actor directly, mutating the grid behind the system's back, drawing a random number from a sequential source — is invisible to undo and to what-if runs, and will make a preview disagree with the real thing. The framework backs the rule with rails, not walls:

- A static **ability linter** flags structural mistakes before you ever hit play.
- A per-ability **Validate Ability** editor action runs the linter *plus* automated consistency checks; if the real run and a what-if run of the same ability disagree, it reports a genuine **Diverged**, and an ability that has nothing to act on without a fixture reports **Indeterminate** rather than a false pass.
- Development builds call out a broken authoring rule loudly at run time.

!!! note "The config you author against is stable"
    The data structs a project authors against ship on the marketplace, so they are a versioned contract: changes to them are additive-only. Authored content never silently breaks on a framework upgrade.

=== "Blueprint"
    1. Create a Blueprint subclass of the **Ability Module** class — this subclass is one step.
    2. Implement the **Execute** event: read the running target list, do your one thing through the entity library's command-routed calls, and write your results back. The engine completes the step for you when Execute returns.
    3. If the step must *wait* for something (a bespoke pause), override **Execute Async** instead and call the context's completion when the input arrives.

=== "C++"
    ```cpp
    UCLASS(DisplayName="Effect: Chill")
    class UMyChillEffect : public UPAbAbilityModule
    {
        GENERATED_BODY()
    protected:
        // The C++ sync override point. The base completes the step after this returns.
        virtual void NativeExecute(UPAbExecContext* Ctx) override
        {
            UPGeGameStateSubsystem* State = Ctx->GetGameState(); // the only state door
            for (const FPAbTarget& T : Ctx->GetWorkingList())    // act on what targeting found
            {
                if (T.HasEntity())
                {
                    // A command-routed, windowed change — reactable, and undoable.
                    State->ApplyStatChange(T.Entity, MoveSpeedTag, -2, Ctx->GetCaller());
                }
            }
            // No completion call here — the base does it for a synchronous step.
            // A step that waits overrides ExecuteAsync and calls Ctx->CompleteABM() itself.
        }
    };
    ```

## Where this goes next

The AbilitySystem plugin section owns the mechanics this page kept at arm's length: the exact editor nodes, the full step-and-program reference, the trigger-authoring recipes, the targeting-session API and its grid wiring, the hover-preview calls, the automatic playout layer, and the AI hooks. It is documented in the [AbilitySystem plugin section](../plugins/abilitysystem/index.md).

Once the Getting Started arc lands, it will wire a first ability end-to-end — grant it, activate it, add a trigger, and preview it on hover — as a single worked slice, tying every idea on this page back to a running game.
