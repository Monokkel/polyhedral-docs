# GameEntity Guides

For developers with the plugin enabled who want to get something done. Each
section is a short, followable recipe; pick the one that matches your task. For
the full type-by-type surface, follow the reference link on each recipe. For the
model behind it all, start with
[Entities Are Data](../../concepts/entities-as-data.md) and
[Stats and Modifiers](../../concepts/stats-and-modifiers.md).

Every recipe uses the game-state subsystem. Get it once from any world context and
hold it for the calls that follow.

=== "Blueprint"
    Call **Get Game State Subsystem** (from the *GameEntity* category), passing a
    world context, and store the result.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);
    ```

## Create entities and read data back

The usual way to make an entity is from a data-table row. You get back an **entity
reference** — a small value you store and resolve through the subsystem; you never
hold a pointer to the entity itself. Creation is a command, so it undoes.

=== "Blueprint"
    1. Call **Create Entity From Row**, passing a data-table row handle. It returns
       an **entity reference**.
    2. Read data back with **Get Stat** (for example `Stat.Health`), **Has Tag**
       (for example `Unit.Enemy`), and the entity **Get Tagged Data** node, all
       taking that reference.
    3. Before reading a stored reference later, branch on **Entity Exists** — an
       entity that was destroyed or undone away simply stops resolving.

=== "C++"
    ```cpp
    // From a data-table row.
    FPGeEntityRef Goblin = State->CreateEntityFromRow(GoblinRow);

    // Read data back through the subsystem, never off a pointer.
    int64 Health   = State->GetStat(Goblin, TAG_Stat_Health);
    bool  bIsEnemy = State->HasTag(Goblin, TAG_Unit_Enemy);
    ```

When many entities share one definition, register it as a **template** once and
create instances from it. Each instance stores only its own overrides and reads
the rest from the template.

=== "Blueprint"
    1. Call **Register Template From Row** once (at load, say) — it returns the
       template's **id**.
    2. Call **Create Entity From Template** with that id for each instance. Use
       **Create Entity From Registry** instead to source the definition from a Data
       Registry item.

=== "C++"
    ```cpp
    // Register the shared definition once...
    FName GoblinId = State->RegisterTemplateFromRow(GoblinRow);

    // ...then spawn instances that delegate unshared reads to it.
    FPGeEntityRef A = State->CreateEntityFromTemplate(GoblinId);
    FPGeEntityRef B = State->CreateEntityFromTemplate(GoblinId);
    ```

See the [Entities reference](reference-entities.md) for every creation entry
point, the template store, and the query API.

## Author entity templates as data-table rows

Designers compose entities declaratively as **entity rows**, with parent-row
inheritance — a child row overrides its parents, so a "fire goblin" is a "goblin"
row plus a few changes rather than a subclass. You fill in a row's fields and the
framework merges parents and children into the entity's tags, stats, and tagged
data for you.

=== "Blueprint"
    1. Create a **Data Table** using the **Entity Row** row structure.
    2. Add a row and fill in its authoring fields:
        - **Parent Entities** — row handles this row inherits from; later entries
          win over earlier ones.
        - **Manual Gameplay Tags** — the entity's starting tags.
        - **Manual Base Stats** — starting base stats (whole numbers).
        - **Manual Tagged Data** — typed data entries keyed by `Data.*` tags.
        - **Manual Granted Mods** — modifiers this entity grants to whatever carries
          it (see the equip recipe below).
        - **Friendly Name** / **Description** — imported into `Data.Name` /
          `Data.Description`.
    3. A derived row struct can add its own properties tagged
       `StartingStat` / `GrantedStat` / `TaggedData` / `StartingTag`; their values
       fold into the same containers automatically on import.

=== "C++"
    ```cpp
    // Entity rows extend the entity struct; author them in a data table and let the
    // import merge parents + manual fields. In code you usually consume the merged
    // result rather than build the row by hand:
    if (const FPGeGameEntityRow* Row = Table->FindRow<FPGeGameEntityRow>(RowName, TEXT("")))
    {
        FPGeGameEntity Merged = Row->ToGameEntity();   // parents + manual fields, resolved
        FPGeEntityRef  Entity = State->CreateEntityFromData(Merged);
    }
    ```

!!! tip "Base stats are whole numbers"
    Author base stats as integers. For fractional flavor — a ×1.5 crit, a −30%
    debuff — author a **modifier**, not a fractional base. See the
    [Stats reference](reference-stats.md) for the row authoring surface in full.

## Read and write tagged data on an entity

On an entity, tagged data is authoritative game state: writes are command-driven
(so undoable and saved), and reads fall back to the template when the entity has
no override of its own.

=== "Blueprint"
    1. Use the entity **Set Tagged Data** node against the entity reference and a
       `Data.*` tag — the write shows up in Undo.
    2. Read with the entity **Get Tagged Data** node: it returns the entity's own
       value, or the template's value when the entity hasn't overridden it.

=== "C++"
    ```cpp
    // Writing entity tagged data routes through the command stack — undoable.
    State->SetTaggedData(Card, PositionTag,
        FInstancedStruct::Make(FBoardPosition{ 3, 2 }));

    // Falls back to the template when this entity has no override.
    FInstancedStruct Raw = State->GetTaggedData(Card, PositionTag);
    ```

!!! note "Setting data also marks the tag"
    Writing tagged data marks the entity with that key tag, so a plain tag query
    answers "does this entity carry `Data.*`?". The typed Blueprint get/set nodes
    and the `Data.*` schema come from
    [TaggedData](../taggeddata/index.md); on entities they route through the
    game-state subsystem. Full API in the
    [Entities reference](reference-entities.md).

## Change a base stat with a working Undo

A base-stat change is one command, so wiring Undo is a single call — you don't
write anything special.

=== "Blueprint"
    1. Call **Modify Stat** (for example `Stat.Health`, delta `-5`) — the change is
       recorded automatically.
    2. On an **Undo** button, **Get Command Stack** and call **Undo**; enable the
       button from **Can Undo**. **Redo** mirrors it.

=== "C++"
    ```cpp
    UPCsCommandStack* Stack = UPCsCommandStack::Get(this);

    State->ModifyStat(Hero, HealthTag, -5);      // recorded on the stack

    if (Stack->CanUndo()) { Stack->Undo(); }     // health back to its previous value
    ```

Use **Set Stat** to write an absolute value instead of a delta. Both create the
stat if it doesn't exist. See the [Commands & replay reference](reference-commands.md)
for the built-in mutator inventory and the [Stats reference](reference-stats.md)
for the stat calls.

## Add and remove modifiers, and read the current stat

You mutate base stats and modifiers; you *read* current stats. The current value
recomputes automatically the moment a modifier is added or removed.

=== "Blueprint"
    1. Read **Get Current Stat** and convert it for display to show the value now.
    2. Call **Add Flat Modifier** (`+2` Attack, source = the sword) — it returns a
       **handle**.
    3. Call **Add Percent Modifier** (`+50`, source = a rage effect).
    4. Read **Get Current Stat** again — it already reflects both.
    5. **Remove Stat Modifier** by handle to take one off; or **Remove Stat
       Modifiers By Source** to clear everything one entity applied.

=== "C++"
    ```cpp
    const double Before = State->GetCurrentStat(Hero, AttackStat).ToDisplay();

    FPGeStatModifierHandle Flat = State->AddFlatModifier(Hero, AttackStat, 2, Sword);
    State->AddPercentModifier(Hero, AttackStat, 50, Rage);

    const double After = State->GetCurrentStat(Hero, AttackStat).ToDisplay();
    // +2 folds in before the ×1.5; After is recomputed for you.

    State->RemoveStatModifier(Flat);                       // by handle
    State->RemoveStatModifiersBySource(Hero, Rage);        // all from one source
    ```

!!! warning "Convert stat values only at the display edge"
    `Get Current Stat` returns an exact fixed-point **stat value**, not a float.
    Keep it exact while you compute; convert with **Stat Value → Display Text** (a
    rounded label) or **Stat Value → Display** (the exact number) only when you
    show it. For the operations, phases, and formula magnitudes a modifier can
    carry, see the [Stats reference](reference-stats.md).

## Grant modifiers from a carried entity (equip an item)

An entity carried in the hierarchy can **grant** modifiers to whatever holds it —
a sword in a weapon slot granting `+2` attack to its wielder. The grants apply and
remove automatically as the item is attached and detached, and equip/unequip
undoes cleanly because it is just a hierarchy change on the stack.

=== "Blueprint"
    1. Author the item's granted modifiers on its **entity row** — fill in
       **Manual Granted Mods** (or a `GrantedStat` property on a derived row).
    2. To equip: call **Add Child**, parenting the item entity under the wielder in
       a slot tag (for example `Slot.Weapon`). The grants apply as part of that
       command.
    3. To unequip: call **Remove Child**. The grants come off automatically.

=== "C++"
    ```cpp
    // The sword entity was created from a row that carries its granted modifiers.
    State->AddChild(Hero, Sword, TAG_Slot_Weapon);   // grants apply — Attack goes up
    // ...later...
    State->RemoveChild(Hero, Sword);                 // grants remove — Attack back down
    ```

!!! tip "Grants can be gated by a condition"
    A grant can require a condition on the carried entity's own tags — including
    which slot it sits in — so a rune grants its bonus only while attuned, or a
    weapon only in a weapon slot. Toggle the tag and the grant reconciles itself.
    The gating and the granted-modifier authoring surface are in the
    [Stats reference](reference-stats.md).

## Set up an entity inside a stat batch

When you make many stat changes at once — spawning a fully-equipped unit — wrap
them in a **stat batch**. Reads stay fresh inside the batch, but the recompute and
the change broadcast defer to the close and fire once as a net change per stat, so
a listener hears one settled result instead of a storm of intermediate values.

=== "Blueprint"
    Call **Begin Stat Batch**, do all your writes (set base stats, add a modifier
    per piece of gear, attach children), then **End Stat Batch**. Everything
    recomputes once at the close. **Modify Stats** applies a whole map of base-stat
    deltas as one batch if that's all you need.

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

A stat batch is a recompute-coalescing scope, separate from an undo transaction.
Pair it with a [transaction](../commandsystem/guides.md#group-changes-into-one-undo-step-with-a-transaction)
when you also want the whole setup to undo as one step. See the
[Stats reference](reference-stats.md).

## Show a stat breakdown in a tooltip

Ask for a stat **breakdown** and you get the ordered, labeled contributions that
produced a current value — the base, then each modifier in apply order — ready to
feed a tooltip like `+3 Strength (Sword), ×1.5 (Rage)`.

=== "Blueprint"
    Call **Get Stat Breakdown**. It returns an array of steps; each step has a
    **Label**, a **Value**, and the **Running** total after it. Bind them into your
    tooltip widget. It is a pure query — safe to call every time the tooltip opens.

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

See the [Stats reference](reference-stats.md) for the breakdown step shape.

## Keep a display in sync by reacting to change events

Reactive code — a health bar, a token, a cache — stays correct by reacting to
**change events**, never by polling. The subsystem broadcasts every mutation
during play *and* during undo, redo, and load, so the same handler keeps you
correct through all of them.

!!! warning "Subscribe to the catch-all plus restore — always both"
    Bind **On Any Entity Changed** (fires for every mutation) *and* **On State
    Restored** (fires once when the whole state is replaced, on load or reset).
    Deriving from only a subset of the typed events is the classic way to desync
    silently.

=== "Blueprint"
    1. On construct, bind the subsystem's **On Any Entity Changed** and **On State
       Restored** events.
    2. In the change handler, branch on **Change Type**; when it is **Current Stat
       Changed** for the entity you watch, read **New Value**, convert at the
       display edge, and update the display.
    3. Branch on **Cause** for feel — animate on **Command**, snap on **Undo**,
       **Redo**, or **Restore**.
    4. In the restore handler, re-read the entity from scratch and rebuild — the
       wholesale path that runs on load.

=== "C++"
    ```cpp
    void UHealthBar::BindTo(UPGeGameStateSubsystem* State)
    {
        // Dynamic multicast delegates — OnChanged and Rebuild must be UFUNCTION()s.
        State->OnAnyEntityChanged.AddDynamic(this, &UHealthBar::OnChanged);
        State->OnStateRestored.AddDynamic(this, &UHealthBar::Rebuild);  // wholesale
    }

    void UHealthBar::OnChanged(const FPGeEntityChange& Change)
    {
        if (Change.EntityRef != Watched) { return; }
        if (Change.ChangeType == EPGeEntityChangeType::CurrentStatChanged)
        {
            const bool bSnap = Change.Cause != EPGeChangeCause::Command;
            SetHealth(Change.NewValue.ToFloat(), bSnap);   // convert only at display
        }
    }
    ```

!!! warning "Listeners are read-only"
    A change-event handler may never submit a command — it runs inside command
    execution, including undo and redo. Reactions that *change* state are game
    rules and belong elsewhere. See
    [Derived State & Events](../../concepts/derived-state.md). The full
    mutation-to-event matrix is in the [Events reference](reference-events.md).

## Record and replay a session

The framework can record the live command timeline to a file and play it back step
by step — the base state plus every committed change, so the replay reproduces the
run exactly.

=== "Blueprint"
    1. At a settled point (no action mid-flight), call **Start Recording** on the
       replay subsystem with a label. It writes to a file under your project's
       saved replays.
    2. Play normally. Call **Request Checkpoint** at meaningful moments if you want
       labeled markers.
    3. Call **Stop Recording** at another settled point to close the file. **Get
       Current File Path** tells you what was written.

=== "C++"
    ```cpp
    UPGeReplaySubsystem* Replay = UPGeReplaySubsystem::Get(this);

    Replay->StartRecording(TEXT("Match 1"));
    // ...play; the live command timeline is captured...
    Replay->StopRecording();

    const FString File = Replay->GetCurrentFilePath();
    // Load and single-step the recording back through the same fixture:
    TSharedRef<FPGeReplayPlayer> Player = Replay->LoadReplay(File);
    ```

!!! note "Recording only happens at settled boundaries"
    Start and stop are legal only when nothing is mid-transaction, and look-ahead
    simulations never leak into a recording. Loading and stepping a replay is a C++
    surface today. The recorder and player API is in the
    [Commands & replay reference](reference-commands.md).
