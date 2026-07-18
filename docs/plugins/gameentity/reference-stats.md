# API Reference: Stats & Modifiers

For programmers and technical designers wiring stats into gameplay. This page is
the full stat and modifier surface on the GameEntity game-state subsystem
(`UPGeGameStateSubsystem`): reading base and current stats, the modifier system,
granted modifiers, batches, and breakdowns, plus the supporting value types.

New to the model? Read the [Stats & Modifiers concept page](../../concepts/stats-and-modifiers.md)
first. In one breath: an entity's **base stat** is an authored whole number; its
**current stat** is a derived value the framework recomputes for you by folding a
stack of modifiers over the base — you never write the current value, you change
inputs and read the result.

!!! note "Mutation is command-driven; reads are free"
    Every function that *changes* a stat routes through the command stack, so it is
    undoable and replayable (see [Commands & Undo](../../concepts/commands-and-undo.md)).
    Every function that *reads* a stat is a direct, side-effect-free query you can
    call as often as you like. Current-stat reads are cached and flushed on read, so
    a read right after a change always returns the fresh value.

All examples are hand-written. Get the subsystem from any world context:

=== "Blueprint"
    Call **Get Game State Subsystem** (a static node) with a world context, or drag
    off any actor's *self* and search for it.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(WorldContextObject);
    ```

---

## Base vs current stats

A stat is identified by a gameplay tag under the `Stat.*` namespace. It has two
readable values:

- **Base stat** — the authored whole number (`int64`), stored on the entity and
  captured by commands. Base stats are whole by domain rule; you set and adjust them
  directly.
- **Current stat** — the derived result of folding the entity's modifier stack (and
  any static definition pipeline) over the base. Returned as a **stat value**
  (`FEvalStatValue`, an exact fixed-point number type from the
  [Evaluators plugin](../evaluators/index.md)). Never authored, never saved, always
  recomputed — see [Derived state](../../concepts/derived-state.md).

!!! tip "Convert stat values only at the display edge"
    Keep a `FEvalStatValue` exact through all your logic and comparisons. Convert it
    to a float or text only when you put it on screen. The
    [Evaluators reference](../evaluators/reference.md) documents the value type and
    its display helpers; the short version:

    === "Blueprint"
        Use **Stat Value → Display** (double), **Stat Value → String**, or
        **Stat Value → Whole (Rounded)** from the *Evaluators \| Stat Value* category.

    === "C++"
        ```cpp
        FEvalStatValue Hp = State->GetCurrentStat(Unit, TAG_Stat_Health);
        double Shown   = Hp.ToDisplay();      // 12.5
        FString Text   = Hp.ToString();       // "12.5"
        int64 Whole    = Hp.ToWholeRounded(); // 13
        ```

---

## Reading stats

All reads are `BlueprintPure` and return safe defaults for a missing entity or stat.

```cpp
// Base value (whole units). 0 if the stat or entity does not exist.
int64          GetStat(const FPGeEntityRef& Entity, FGameplayTag StatTag) const;

// Does this entity carry a base entry for the stat?
bool           HasStat(const FPGeEntityRef& Entity, FGameplayTag StatTag) const;

// Current (fully modified) value as an exact stat value.
// Zero if the stat is not materialized (no base entry and no modifiers).
FEvalStatValue GetCurrentStat(const FPGeEntityRef& Entity, FGameplayTag StatTag) const;

// Whole-map snapshot of every resolved current stat on the entity.
TMap<FGameplayTag, FEvalStatValue> GetCurrentStats(const FPGeEntityRef& Entity) const;
```

`GetCurrentStat` and `GetCurrentStats` are the values you show the player and feed to
game logic. Use `GetStat` when you specifically need the untouched authored number
(e.g. "restore to full" needs the base max).

=== "Blueprint"
    **Get Current Stat** returns a *Stat Value*; pipe it through **Stat Value →
    Display** to bind it to a text widget.

=== "C++"
    ```cpp
    if (State->GetCurrentStat(Unit, TAG_Stat_Health) <= FEvalStatValue())
    {
        // unit is down
    }
    ```

---

## Changing base stats

Base-stat writes are command-routed and undoable.

```cpp
// Set the base value outright (whole units). Creates the stat if absent.
void  SetStat(const FPGeEntityRef& Entity, FGameplayTag StatTag, int64 Value);

// Add Delta to the base value. Creates the stat (at 0) if absent. Returns the new base.
int64 ModifyStat(const FPGeEntityRef& Entity, FGameplayTag StatTag, int64 Delta);

// Remove the base entry entirely. Returns false if it was not present.
bool  RemoveStat(const FPGeEntityRef& Entity, FGameplayTag StatTag);
```

To apply several base-stat deltas as one undo-friendly, single-recompute unit, see
[`ModifyStats`](#stat-batches).

!!! note "A windowed variant exists for reactive changes"
    Alongside the direct `ModifyStat`, the subsystem also offers a *windowed* stat
    change (`ApplyStatChange`) that lets other entities react to — and even reshape —
    the change before and after it commits. That reaction machinery belongs to the
    event system's reaction mechanism, documented in a later section; the
    functions on this page are the direct, reaction-blind path.

---

## The stat modifier

A modifier is authored as a **spec** and stored on the entity with a runtime
identity. You mostly build specs; the framework wraps each one when you add it.

### `FPGeStatModifierSpec` — the authored shape

| Field | Type | Meaning |
|---|---|---|
| `TargetStat` | `FGameplayTag` (`Stat.*`) | Which stat this modifier contributes to. |
| `Op` | `EMagnitudeOp` | How its value folds into the running total: `+`, `−`, `×`, `÷`, `Override` (`=`), `Min`, `Max`. |
| `Magnitude` | `TInstancedStruct<FEvalMagnitudeCalc>` | The **value source** — a constant, or an evaluator magnitude (see [below](#using-an-evaluator-magnitude-as-a-value-source)). |
| `Phase` | `FGameplayTag` (`StatPhase.*`) | Ordering bucket; unset uses the default phase. |
| `Order` | `int32` | Sort order within the phase; ties break by insertion order. |
| `SourceTag` | `FGameplayTag` | Semantic source id (the ability/effect identity), for bulk removal and grouping. |
| `DisplayName` | `FText` | Player-facing label; survives the source entity being destroyed. |

The `Op` values come from the [Evaluators plugin](../evaluators/reference.md).
Because `Override`, `×`, `Min`, and `Max` are order-sensitive, the phase/order fields
decide the outcome — see [ordering](#modifier-ordering-and-phases).

### `FPGeStatModifier` — one live modifier

The stored entry is the spec plus its runtime identity:

- `Spec` — the authored `FPGeStatModifierSpec`.
- `Id` — a stable `FGuid` assigned when the modifier is added.
- `SourceRef` — the applying entity (the spell, item, aura…). May be invalid or
  point at an entity that has since died; modifiers outlive their source.

How long a modifier lasts is set through the
[end-condition helpers](#modifier-lifetime-and-end-conditions), not by writing raw
fields.

### `FPGeStatModifierHandle` — the reference you keep

Adding a modifier returns an opaque handle you use to update or remove it later.

```cpp
struct FPGeStatModifierHandle
{
    FPGeEntityRef Owner;  // the entity the modifier is on
    FGameplayTag  Stat;   // the stat it targets
    FGuid         Id;     // the modifier's identity

    bool IsValid() const; // true only when all three are set
};
```

An invalid handle is what every failed add returns — check `IsValid()` before
trusting it.

---

## Adding and removing modifiers

```cpp
// Add a fully-authored modifier. SourceRef identifies the applier (may be invalid).
// Returns an invalid handle on failure.
FPGeStatModifierHandle AddStatModifier(const FPGeEntityRef& Entity,
    const FPGeStatModifierSpec& Spec, FPGeEntityRef SourceRef);

// Sugar: a flat additive modifier of Amount whole units (+5 armor, −6 damage).
FPGeStatModifierHandle AddFlatModifier(const FPGeEntityRef& Entity,
    FGameplayTag Stat, int64 Amount, FPGeEntityRef SourceRef);

// Sugar: a multiplicative percent modifier — Percent 50 → ×1.5, −30 → ×0.7.
FPGeStatModifierHandle AddPercentModifier(const FPGeEntityRef& Entity,
    FGameplayTag Stat, int32 Percent, FPGeEntityRef SourceRef);

// Remove one modifier by handle. False if it was not found.
bool  RemoveStatModifier(const FPGeStatModifierHandle& Handle);

// Swap a live modifier's spec in place, keeping its identity and ordering slot.
// NewSpec must keep the handle's TargetStat.
bool  UpdateStatModifier(const FPGeStatModifierHandle& Handle,
    const FPGeStatModifierSpec& NewSpec);

// Bulk removal. Both return the number removed.
int32 RemoveStatModifiersBySource(const FPGeEntityRef& Entity, const FPGeEntityRef& SourceRef);
int32 RemoveStatModifiersBySourceTag(const FPGeEntityRef& Entity, FGameplayTag SourceTag);

// Copy of one stat's modifier list, in insertion order.
TArray<FPGeStatModifier> GetStatModifiers(const FPGeEntityRef& Entity, FGameplayTag StatTag) const;
```

`AddFlatModifier` and `AddPercentModifier` are the everyday path: whole numbers in,
no evaluator wiring, no fixed-point scaling to think about. Reach for
`AddStatModifier` when the value must come from an evaluator magnitude, or when you
need to set the phase, order, source tag, or display name.

Use `RemoveStatModifiersBySource` to clean up "everything this aura applied" when the
aura ends, and `RemoveStatModifiersBySourceTag` to strip every modifier sharing an
effect identity regardless of who applied it.

=== "Blueprint"
    1. **Add Flat Modifier** — target entity, `Stat.Armor`, `5`, source entity.
    2. Keep the returned **Stat Modifier Handle** in a variable.
    3. **Remove Stat Modifier** with that handle when the effect ends.

=== "C++"
    ```cpp
    FPGeStatModifierHandle Buff =
        State->AddFlatModifier(Unit, TAG_Stat_Armor, 5, SourceItem);
    // ... later ...
    State->RemoveStatModifier(Buff);
    ```

---

## Modifier ordering and phases

Current stats are computed by folding every modifier over the base value one at a
time, so for non-commutative operations (`Override`, `×`, `Min`, `Max`) the order is
the result. Order is decided in three tiers:

1. **Phase** — the modifier's `Phase` tag (`StatPhase.*`) resolves to an integer sort
   order; lower phases fold first.
2. **Order** — the modifier's `Order` field, within its phase.
3. **Insertion order** — a stable tiebreak for equal phase and order.

Phases are named and ranked in project settings, so designers speak in tags
("additive before multiplicative") rather than magic numbers:

- `UPGeStatSettings::PhasePresets` maps each `StatPhase.*` tag to its integer sort
  order (lower folds first).
- `UPGeStatSettings::DefaultPhase` is used by any modifier whose `Phase` is unset or
  has no preset entry.

A stat's static definition pipeline (if any) always folds first, before any runtime
modifier — see [stat definitions](#stat-definitions-and-settings).

---

## Using an evaluator magnitude as a value source

A modifier's `Magnitude` is a `TInstancedStruct<FEvalMagnitudeCalc>` — the same
composable calculation used everywhere in the [Evaluators plugin](../evaluators/index.md).
It can be a constant, a pipeline, or a read of another stat. The Evaluators section
documents the full magnitude catalog; GameEntity adds two calcs that read from an
entity resolved through the evaluation context:

**`FPGeMagCalc_EntityStat`** — reads a stat off a context entity.

| Field | Type | Meaning |
|---|---|---|
| `SourceTag` | `FGameplayTag` (`Eval.Source.*`) | Which context entity to read — `Eval.Source.Owner` (the modified entity) or `Eval.Source.Instigator` (the modifier's `SourceRef`). |
| `Stat` | `FGameplayTag` (`Stat.*`) | The stat to read. |
| `Kind` | `EPGeStatValue` | `Base` or `Current`. |

**`FPGeCondCalc_EntityHasTag`** — a condition (for gated magnitudes) that passes when
a context entity has all (or, with `bRequireAll` false, any) of a tag set.

Both are automatically dependency-tracked: a modifier whose magnitude reads
`Owner`'s Strength, or is gated on the `Owner` having `State.Enraged`, recomputes on
its own the moment that input changes. You wire the dependency simply by reading —
you never register or invalidate anything by hand.

```cpp
// "+ (Owner's current Strength)" to Attack.
FPGeStatModifierSpec Spec;
Spec.TargetStat = TAG_Stat_Attack;
Spec.Op         = EMagnitudeOp::Add;
Spec.Magnitude.InitializeAs<FPGeMagCalc_EntityStat>();
FPGeMagCalc_EntityStat& Mag = Spec.Magnitude.GetMutable<FPGeMagCalc_EntityStat>();
Mag.SourceTag = EvalTags::TAG_Source_Owner;
Mag.Stat      = TAG_Stat_Strength;
Mag.Kind      = EPGeStatValue::Current;

State->AddStatModifier(Unit, Spec, SourceAura);
```

---

## Granted modifiers

A carried entity can hand modifiers to whatever owns it — the classic "equip this
sword, gain +2 Attack while it's in a weapon slot". These are authored on the
*granting* entity's template as the tagged-data entry `Data.Stat.GrantedMods`
(`FPGeGrantedStatMods`) and applied to the **parent** automatically and
transactionally whenever the hierarchy changes.

```cpp
struct FPGeGrantedStatMods
{
    TArray<FPGeGrantedModGroup> Groups;
};

struct FPGeGrantedModGroup
{
    // When this group is active. Empty = active whenever the entity has a parent.
    FGameplayTagQuery ActiveCondition;

    // The modifiers applied to the PARENT while the group is active.
    TArray<FPGeStatModifierSpec> Mods;
};
```

!!! important "The gate is a tag query on the carrier, not an evaluator condition"
    `ActiveCondition` is a **gameplay-tag query matched against the carried (child)
    entity's own tags — including its slot tags** — not a general evaluator
    condition. Use it for "only while equipped in a `Slot.Weapon`" or "only while this
    item carries `State.Attuned`". The grants reconcile when the child is
    parented/reparented, when its slot changes, and when a tag the query references is
    added or removed while it stays parented. The framework applies and removes the
    grants for you with the granting child recorded as each modifier's `SourceRef`, so
    they clean up on their own when the item is unequipped.

---

## Modifier lifetime and end conditions

By default a modifier is permanent — it lasts until you remove it by handle or by
source. To make one expire on a game event instead, describe *when it should end* and
let the framework tear it down at the right moment. Build an **end condition**, then
stamp the modifier with it:

```cpp
// End-condition builders (BlueprintPure).
FPGeEndCondition Permanent() const;                                  // never expires (default)
FPGeEndCondition UntilScopeCloses(const FPGeEntityRef& Scope) const; // ends when that scope closes
FPGeEndCondition UntilDestroyed(const FPGeEntityRef& Anchor) const;  // ends when that entity is destroyed
FPGeEndCondition UntilLeaves(const FPGeEntityRef& Container,
    const FPGeEntityRef& Resident) const;                           // ends when Resident leaves Container
FPGeEndCondition UntilUnequipped(const FPGeEntityRef& Equipment) const; // ends when Equipment is unparented

// Add a modifier that expires when its end condition fires.
FPGeStatModifierHandle StampModifier(const FPGeEntityRef& Target,
    const FPGeStatModifierSpec& Spec, const FPGeEntityRef& SourceRef,
    const FPGeEndCondition& EndCondition);
```

`StampModifier` behaves like `AddStatModifier` but binds the modifier's teardown to
the condition, all inside the same undo unit as the change that triggers it.

```cpp
// "+3 Attack until this encounter ends."
State->StampModifier(Hero, RageSpec, SourceAbility,
    State->UntilScopeCloses(EncounterScope));
```

---

## Stat batches

A stat batch coalesces derived-stat recompute and current-stat change broadcasts. Do
a burst of base-stat and modifier changes inside a batch and every affected current
stat recomputes **once** at the end, broadcasting its net change a single time (a
change that nets to zero broadcasts nothing at all).

A batch is a recompute-coalescing scope — it is **not** a command transaction (the
undo unit), and reads stay fresh inside it (a current-stat read mid-batch still
returns an up-to-date value). Batches are nestable and reference-counted; only the
outermost close flushes.

```cpp
// Blueprint scope form — must be balanced.
void BeginStatBatch();
void EndStatBatch();
bool IsStatBatchOpen() const;

// One-shot: several base-stat deltas as a single batch. Each entry is undoable.
void ModifyStats(const FPGeEntityRef& Entity, const TMap<FGameplayTag, int64>& Deltas);
```

=== "Blueprint"
    Wrap the run in **Begin Stat Batch** / **End Stat Batch**, or use **Modify Stats**
    with a map of stat → delta when you only need to bump base stats.

=== "C++"
    Prefer the RAII/lambda forms — they close the batch even on an early return:

    ```cpp
    // Lambda form.
    State->WithStatBatch([&]
    {
        State->ModifyStat(Unit, TAG_Stat_Health, -6);
        State->AddFlatModifier(Unit, TAG_Stat_Armor, 2, SourceItem);
    }); // one recompute, one net broadcast here

    // Or a scope guard.
    {
        FPGeStatBatchScope Batch(State);
        State->ModifyStat(Unit, TAG_Stat_Health, -6);
        // ...
    } // flushes on scope exit
    ```

---

## Breakdowns

For tooltips and inspector UI, `GetStatBreakdown` traces how a current stat reached
its value: the base, then the static definition pipeline, then each runtime modifier
in the exact order it was applied (labeled by its `DisplayName`, or its `SourceTag`
when no display name is set), then the final value.

```cpp
bool GetStatBreakdown(const FPGeEntityRef& Entity, FGameplayTag StatTag,
    TArray<FEvalCollectorStep>& OutSteps) const;
```

It is a pure query — it writes no state and broadcasts nothing — and returns false
with an empty array when the stat is not materialized. Each `FEvalCollectorStep`
carries:

| Field | Type | Meaning |
|---|---|---|
| `Kind` | `EEvalStepKind` | `Base`, `Step`, or `Final`. |
| `Label` | `FString` | The step's display label. |
| `Op` | `uint8` | The operation for this step (an `EMagnitudeOp`). |
| `Value` | `float` | The step's contribution. |
| `Running` | `float` | The running total after this step. |
| `Source` | `UObject*` | The contributing source, when known. |

!!! note "Breakdown numbers are for display only"
    Step values are floats captured at a display boundary — perfect for a "+5 armor
    (Shield), ×1.5 (Rage)" tooltip. The authoritative value is always
    [`GetCurrentStat`](#reading-stats); do not drive game logic off breakdown floats.

---

## Stat definitions and settings

A **stat definition** gives a stat a default base and an optional always-on
calculation. It is stored as the tagged-data entry `Data.Stat.Definition`
(`FPGeStatDefinition`) on an entity template — any template carrying that entry *is* a
stat definition, and the subsystem indexes it at startup. A stat with no definition
is just a plain tag with a base number.

```cpp
struct FPGeStatDefinition
{
    FGameplayTag           Stat;          // the stat this row defines (Stat.*)
    int64                  DefaultBase;   // base used when an entity has no entry (whole units)
    FEvalMagnitudePipeline StaticPipeline; // folded first, before any runtime modifier
};

// C++ lookup of a stat's shared definition; nullptr for a plain stat with no definition.
const FPGeStatDefinition* GetStatDefinition(FGameplayTag Stat) const;
```

The `StaticPipeline` is where definitional math lives (a derived stat that is always
"Strength ÷ 2", say); it runs ahead of the runtime modifier stack for every entity
that has the stat.

### Project settings

Configure the system under **Project Settings → Plugins → Game Entity Stats**
(`UPGeStatSettings`):

| Setting | Meaning |
|---|---|
| `StatDefinitionTables` | Data tables of stat-definition entity rows, registered as templates and indexed at startup. |
| `PhasePresets` | Maps each `StatPhase.*` tag to its integer sort order (lower folds first). |
| `DefaultPhase` | The phase used by modifiers with an unset or unrecognized `Phase`. |

The relevant tag roots are `Stat.*` (stats), `StatPhase.*` (ordering phases),
`Data.Stat.Definition` / `Data.Stat.GrantedMods` (the tagged-data keys above), and
`Eval.Source.*` (the context entities a magnitude reads from).

---

## See also

- [Stats & Modifiers concept](../../concepts/stats-and-modifiers.md) — the mental model.
- [Derived state](../../concepts/derived-state.md) — why current stats are never saved.
- [Evaluators plugin](../evaluators/index.md) and its [reference](../evaluators/reference.md) — magnitudes, pipelines, and the stat value type.
- [Entity reference](reference-entities.md) — the lifecycle, hierarchy, and tagged-data API.
- [Command reference](reference-commands.md) — the built-in stat commands and replay.
- [CommandSystem plugin](../commandsystem/index.md) — the undo/redo substrate behind every write.
