# Stats and Modifiers

For developers building the numbers of their game — health, attack, cost — with
buffs, gear, and auras layered on top. After reading, you'll know how a base stat
plus a stack of modifiers becomes the current value, in what order, and why the
arithmetic comes out identical on every machine and every replay.

## Two layers: base and current

Every stat has two layers.

- The **base stat** is a whole number you author and store on the entity —
  a unit's starting attack, a card's base cost.
- The **current stat** is computed: the base folded through the entity's stack of
  modifiers. It is derived state — never saved, never authored directly.

You mutate base stats and modifiers; you *read* current stats. Stats live under the
`Stat.*` gameplay-tag namespace.

!!! important "The one rule to internalize"
    **Read current stats. Mutate base stats and modifiers.** When anything a current
    stat depends on changes, the framework recomputes it automatically — it tracks
    which values each stat reads, so only the affected stats recompute. You never
    call a "recalculate" function and you never cache the result yourself.

Every base-stat change and every modifier change goes through the command stack, so
all of it undoes, saves, and replays — see
[commands-and-undo.md](commands-and-undo.md). The current values fall out as derived
state, exactly like the reactive pattern in [derived-state.md](derived-state.md).

!!! note "If you've used Unreal Engine's Gameplay Ability System"
    A base value aggregated through a stack of modifiers into a current value will
    feel familiar. The difference here is that the entire stack is plain data on the
    entity — not a runtime-only component — so it undoes, saves, and replays with the
    rest of your game state.

## What a modifier is

A modifier is four things:

- A **target stat** — which stat it changes.
- An **operation** — how it combines: `+` `−` `×` `÷` `override` `min` `max`.
- A **value source** — a constant, or any evaluator magnitude (so "+1 attack per
  level" is an authored formula, not code). See [evaluators.md](evaluators.md).
- An **ordering** — a **phase** tag, then an order number, then insertion order.

Ordering is what makes stacking predictable. Phases give you the classic structure
— additive modifiers before multiplicative ones before final overrides — and your
project defines the phase set under the `StatPhase.*` namespace. Within a phase,
the order number decides; ties fall back to the order things were added.

### A modifier authored as data

A modifier's specification is plain data, identical whether you author it in a data
table or build it in code. Here is one whose value source is a small pipeline:

```text
Stat Modifier Spec        # authored in a data table
  Target Stat : Stat.Attack
  Op          : +          # add this modifier's result to Attack
  Phase       : StatPhase.Additive
  Order       : 0
  Display Name: "Sharpened"
  Magnitude   : Pipeline
      Base 0
      + Constant 2                        # a flat +2 ...
      + Entity Stat (Owner -> Stat.Level) # ... plus one per owner level
```

## Modifiers are real state

Modifiers are authoritative state, not transient decoration:

- **Add** one and you get back a **handle** that names it.
- **Update** or **remove** it later by that handle.
- **Bulk-remove** by the entity that applied it, or by a source tag —
  "remove everything this poison applied" is a single call.

All of these go through the command stack, so applying a buff and removing it are
both undoable steps.

=== "Blueprint"

    1. Read **Get Current Stat** and convert it for display to show the value now.
    2. Call **Add Flat Modifier** (+2 Attack, source = the sword).
    3. Call **Add Percent Modifier** (+50%, source = a rage effect).
    4. Read **Get Current Stat** again — it already reflects both.
    5. **Undo** removes the last modifier. Each add is its own command; wrap both in
       one transaction if you want a single undo step.

=== "C++"

    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    const double Before = State->GetCurrentStat(Hero, AttackStat).ToDisplay();

    State->AddFlatModifier(Hero, AttackStat, 2, Sword);      // +2
    State->AddPercentModifier(Hero, AttackStat, 50, Rage);   // +50%

    const double After = State->GetCurrentStat(Hero, AttackStat).ToDisplay();
    // The flat +2 folds in before the ×1.5; After is recomputed for you.
    ```

The `+2`-then-`×1.5` order above comes from insertion order within one phase. When
ordering matters, assign each modifier a phase rather than relying on it.

## The numbers are exact

Current stats are held as a **stat value** — an exact fixed-point number type that
carries three guard decimal places. Fractional stacking (a ×1.5 on a ×1.1 on an
odd base) therefore comes out to the same bits on every machine, every undo, and
every replay. There is no float drift to desync a multiplayer match or a replay.

!!! warning "Convert to float or text only at the display edge"
    Keep the exact stat value while you compute. Convert it to a float or a string
    only when you are about to show it — a HUD label, a tooltip, a log line. Base
    stats are whole numbers by design; author fractional flavor as a modifier (a
    `×1.5`) or choose finer units (tenths of a point) rather than storing a fraction
    in the base.

=== "Blueprint"

    Do not display a stat value straight from the map — it is scaled for precision.
    Convert at the UI edge:

    - **Stat Value → Display Text** for a rounded whole-number label (`"13"`).
    - **Stat Value → Display** for the exact number, if you need to compute further.

=== "C++"

    ```cpp
    const FEvalStatValue Attack = State->GetCurrentStat(Hero, AttackStat);

    const FString Label = Attack.ToDisplayText();  // "13" — rounded, for a HUD
    const double  Exact = Attack.ToDisplay();      // 12.5 — exact, for more math
    ```

## Modifiers that expire on their own

Beyond removing a modifier by hand, a modifier can carry an **end condition** —
"until this encounter ends," "while this item stays equipped." Expiry is always
driven by a game event: an entity ending, a child leaving a container, a state
changing. It is never driven by wall-clock time, so a paused, undone, or replayed
game expires buffs at exactly the right moment every time.

## Granted modifiers from carried entities

An entity carried in the hierarchy can grant modifiers to whatever holds it — a
sword in an equipment slot granting `+2` attack to its wielder, a passive granted
to a hero. The grants apply and remove **automatically** as the entity is attached
and detached, and each grant can be gated by a condition on the carried entity's
own tags — including which slot it occupies — so a sword grants its bonus only
while it sits in a weapon slot, or a rune only while it is attuned. Toggle one of
those tags and the grant reconciles on its own.

Because grants ride the parent/child hierarchy, equipping and unequipping "just
works" and undoes cleanly — see [entities-as-data.md](entities-as-data.md) for the
hierarchy itself.

## Stat batches for bulk changes

When you make many changes at once — spawning a fully-equipped unit — wrap them in
a **stat batch**. Inside the batch, reads stay fresh, but the recompute and the
change broadcast are deferred to the moment the batch closes, then emitted once as
a net change per stat. Instead of a storm of intermediate values, a listener hears
a single settled result.

=== "Blueprint"

    Call **Begin Stat Batch**, do all your writes (set base stats, add a modifier
    for every piece of gear), then **End Stat Batch**. Everything recomputes once at
    the close.

=== "C++"

    ```cpp
    State->WithStatBatch([&]
    {
        State->SetStat(Hero, HealthStat, 30);
        State->AddFlatModifier(Hero, AttackStat, 2, Sword);
        State->AddFlatModifier(Hero, ArmorStat, 3, Shield);
        // ...build the rest of the unit here...
    });   // one recompute, one net change event per stat
    ```

## Breakdowns for tooltips

Ask for a stat **breakdown** and you get the ordered, labeled contributions that
produced a current value: the base, the static definition, then each modifier in
apply order. That list is the direct feed for a tooltip like
`+3 Strength (Sword), ×1.5 (Rage)`.

=== "Blueprint"

    **Get Stat Breakdown** returns an array of steps. Each step has a **Label**, a
    **Value**, and the **Running** total after it — bind them straight into your
    tooltip widget.

=== "C++"

    ```cpp
    TArray<FEvalCollectorStep> Steps;
    if (State->GetStatBreakdown(Hero, AttackStat, Steps))
    {
        for (const FEvalCollectorStep& Step : Steps)
        {
            // Step.Label, Step.Value, Step.Running -> one tooltip line each.
        }
    }
    ```

## Where to go next

- The value source of a modifier is an [evaluator](evaluators.md) magnitude.
- Current stats are the built-in example of [derived state](derived-state.md).
- Modifier and base-stat changes are [command-driven](commands-and-undo.md).
- Granted modifiers ride the [entity hierarchy](entities-as-data.md).
- See a buff added end to end in the
  [first board tutorial](../getting-started/first-board.md).

For the complete stat and modifier API — every function, the data-table authoring
surface, and the settings that define your phases — see the
[GameEntity stats &amp; modifiers reference](../plugins/gameentity/reference-stats.md).
