# Build Your First Board

For a developer who has the plugins enabled and wants to feel the whole loop in one sitting. By the end you'll have authored two entity templates, spawned them into the game state, attached typed data to one, changed a stat through the command stack with a working Undo, and watched a change event fire — the framework's core loop, end to end.

This tutorial teaches by doing. Each step links out to a concept page the moment that idea starts carrying weight, so you can keep moving now and read the "why" when you want it.

## Watch the walkthrough

!!! info "Video walkthrough in production"
    A narrated video version of this tutorial is being recorded. Until it lands,
    the written steps below are complete and self-contained.

<!-- swap VIDEO_ID and un-comment when published
<iframe width="560" height="315"
    src="https://www.youtube-nocookie.com/embed/VIDEO_ID"
    title="YouTube video player" frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen></iframe>
-->

## What you'll build

A tiny slice of a tactics board: a hero and a goblin, each spawned from a shared
definition, one of them carrying a board position, a hit that lowers health, an
Undo that puts it back, and a listener that keeps a display in sync through all
of it. No grid, no turns, no rendering — just the state layer everything else
sits on.

!!! note "Before you start"
    You need the four Phase 1 plugins enabled and the project compiling. If you
    haven't done that yet, walk through [Installation](installation.md) first.

Every mutation in this tutorial is followable two ways: pick the **Blueprint**
or **C++** tab in each step.

## Step 1 — Author two entity templates

Entities aren't Actors or Blueprint classes — they're plain data records you
create from reusable definitions. Those definitions are authored as data-table
rows. See [Entities](../concepts/entities-as-data.md) for what an entity is and
why keeping game state as data is what makes undo and save work.

1. Create a new **Data Table** and pick **Entity Row** as its row structure.
2. Add a row named `Hero`. Fill in:
    - **Friendly Name** → `Hero`
    - **Manual Base Stats** → `Stat.Health` = `20`, `Stat.Attack` = `5`
    - **Manual Gameplay Tags** → `Unit.Hero`
3. Add a second row named `Goblin`:
    - **Friendly Name** → `Goblin`
    - **Manual Base Stats** → `Stat.Health` = `12`, `Stat.Attack` = `3`
    - **Manual Gameplay Tags** → `Unit.Enemy`

This step is pure data, so it's identical whether you finish the tutorial in
Blueprint or C++. Base stats are authored as whole numbers under the `Stat.*`
tag namespace.

!!! tip "Rows can inherit from rows"
    An entity row can point at parent rows and override just the fields it
    changes — a hundred goblin variants from one base definition. The
    [Entities](../concepts/entities-as-data.md) page covers the inheritance
    model.

## Step 2 — Spawn entities into the game state

All live entities live in one place: the game-state subsystem, on the
GameInstance. You create an entity from a row and get back an **entity
reference** — a lightweight handle you hold instead of a pointer.

=== "Blueprint"
    1. **Get Game State Subsystem** (world context = `self`).
    2. Off it, call **Create Entity From Row**. Set the **Row Handle** to your
       table's `Hero` row.
    3. Promote the returned **Entity Ref** to a variable named `Hero`.
    4. Repeat for the `Goblin` row, storing a `Goblin` reference.

=== "C++"
    ```cpp
    UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

    FDataTableRowHandle HeroRow;
    HeroRow.DataTable = UnitTable;   // your Entity Row data table
    HeroRow.RowName   = TEXT("Hero");

    FPGeEntityRef Hero   = State->CreateEntityFromRow(HeroRow);
    FPGeEntityRef Goblin = State->CreateEntityFromRow(GoblinRow);
    ```

!!! tip "Hold the reference, not the entity"
    Store the `Hero` reference wherever you need it — a widget, a controller.
    Look the entity up through the subsystem each time you read it. References
    stay valid across undo and load; a destroyed entity's reference simply
    stops resolving.

## Step 3 — Store tagged data on an entity

Entities carry more than stats and tags: any typed struct can be attached under
a gameplay tag in the `Data.*` namespace. Give the goblin a board position.
See [Tagged Data](../concepts/tagged-data.md) for the tag-to-struct model and
how the schema turns your struct into a typed Blueprint pin.

First, a small struct to store:

```cpp
USTRUCT(BlueprintType)
struct FBoardPosition
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Column = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Row = 0;
};
```

Then set it on the goblin and read it back:

=== "Blueprint"
    1. Add an entry to your tagged-data schema mapping `Data.Board.Position` to
       the `Board Position` struct. Now the typed nodes give you a real struct
       pin for that tag.
    2. Call the typed **Set Tagged Data** node: **Entity Ref** = `Goblin`, tag =
       `Data.Board.Position`, and fill the **Column** / **Row** pins.
    3. To read it, call the typed **Get Tagged Data** node with the same tag —
       it outputs a `Board Position` directly.

=== "C++"
    ```cpp
    FBoardPosition Start{ /*Column*/ 3, /*Row*/ 2 };
    State->SetTaggedData(Goblin, PositionTag,
        FInstancedStruct::Make(Start));   // PositionTag = Data.Board.Position

    FInstancedStruct Raw = State->GetTaggedData(Goblin, PositionTag);
    if (const FBoardPosition* Pos = Raw.GetPtr<FBoardPosition>())
    {
        // Pos->Column == 3, Pos->Row == 2
    }
    ```

!!! note "Entity data is not the same as actor data"
    The same tagged-data API also works on any Actor or object as convenient
    typed storage. The difference — and why entity tagged data undoes and saves
    while actor tagged data doesn't — is spelled out on the
    [Tagged Data](../concepts/tagged-data.md) page.

## Step 4 — Change a stat, then Undo it

Here's the payoff of keeping state as data: **every** mutation you've called so
far already went through the command stack, so it's all undoable without you
writing a single command. See [Commands & Undo](../concepts/commands-and-undo.md)
for the one door all state changes pass through and what it guarantees.

Land a hit on the goblin, then take it back:

=== "Blueprint"
    1. Call **Modify Stat**: **Entity Ref** = `Goblin`, **Stat Tag** =
       `Stat.Health`, **Delta** = `-8`.
    2. Read it with **Get Stat** (`Stat.Health`) — it now reads `4`.
    3. **Get Command Stack** and call **Undo**.
    4. **Get Stat** again — back to `12`. Wire **Can Undo** / **Can Redo** to
       your buttons' enabled state.

=== "C++"
    ```cpp
    State->ModifyStat(Goblin, HealthTag, -8);   // goes through the command stack
    int64 After = State->GetStat(Goblin, HealthTag);   // 4

    UPCsCommandStack* Commands = UPCsCommandStack::Get(this);
    Commands->Undo();
    int64 Restored = State->GetStat(Goblin, HealthTag);   // 12 again
    ```

!!! tip "Reads are free; only writes go through the stack"
    `GetStat` and friends read the state directly. Only the mutating calls are
    recorded, so undo/redo costs you nothing on the read path.

!!! note "Group several changes into one Undo"
    Wrap a multi-step move (spend a resource, move, deal damage) in a
    **transaction** — one undo step for the player's whole action. The
    [Commands & Undo](../concepts/commands-and-undo.md) page shows how, and when
    you'd write a brand-new kind of command yourself.

## Step 5 — React to a change event

Your board display shouldn't poll the goblin's health every frame — it should
be told when it changes. Every state change broadcasts a change event, during
normal play **and** during undo, redo, and load, so a listener that reacts to
events stays correct through all of them. The one subscription rule that keeps
it correct is on the [Derived State & Events](../concepts/derived-state.md) page.

=== "Blueprint"
    1. From the game-state subsystem, bind an event to **On Any Entity Changed**.
    2. In the handler, read the change's **Change Type** and **Entity Ref**;
       when it's a stat change on a unit you display, refresh that unit's
       health bar.
    3. Also bind **On State Restored** so a fresh load rebuilds the display from
       scratch.

=== "C++"
    ```cpp
    State->OnAnyEntityChanged.AddDynamic(this, &UMyBoardView::HandleChange);
    State->OnStateRestored.AddDynamic(this, &UMyBoardView::RebuildAll);

    // UFUNCTION handler
    void UMyBoardView::HandleChange(const FPGeEntityChange& Change)
    {
        if (Change.ChangeType == EPGeEntityChangeType::StatChanged)
        {
            RefreshHealthBar(Change.EntityRef);
        }
    }
    ```

Now repeat Step 4's hit-then-Undo and watch the display follow both directions
by itself — the same listener handles the change and its reversal.

!!! tip "The event carries what you need"
    A change event includes the affected entity and stat, the exact old and new
    values, and a cause (normal play, undo, redo, restore). Use the cause for
    presentation choices — snap instead of animate on an undo — never as a
    gameplay filter. The [Derived State & Events](../concepts/derived-state.md)
    page covers the full payload.

## Step 6 — Add your first buff

One more step ties it all together. Give the hero a sword that grants **+2
Attack**. A base stat plus a stack of modifiers resolves to the current value,
and the framework recomputes it for you whenever anything it depends on changes.
See [Stats & Modifiers](../concepts/stats-and-modifiers.md) for how the stack
resolves — and, from there, [Evaluators](../concepts/evaluators.md) for building
a modifier's value from a data-driven formula instead of a constant.

=== "Blueprint"
    1. Spawn a `Sword` entity (a `Create Entity` call is enough for now).
    2. Call **Add Flat Modifier**: **Entity Ref** = `Hero`, **Stat** =
       `Stat.Attack`, **Amount** = `2`, **Source Ref** = the sword.
    3. Read **Get Current Stat** (`Stat.Attack`) — it now resolves to `7`.
    4. Undo, and it drops back to `5`. The modifier is authoritative state too.

=== "C++"
    ```cpp
    FPGeEntityRef Sword = State->CreateEntity();
    FPGeStatModifierHandle Buff =
        State->AddFlatModifier(Hero, AttackTag, /*Amount*/ 2, Sword);

    FEvalStatValue Attack = State->GetCurrentStat(Hero, AttackTag);   // resolves to 7
    ```

!!! note "Current stats are an exact number type"
    `GetCurrentStat` returns a **stat value** — an exact fixed-point number, so
    fractional buffs stack the same on every machine and every replay. Convert
    it to a float or text only at the display edge (a value-to-text node in
    Blueprint, its display conversion in C++). The reasoning is on the
    [Stats & Modifiers](../concepts/stats-and-modifiers.md) page.

## Where to go next

You've touched the whole core loop. To go deeper on any piece:

- **[Entities](../concepts/entities-as-data.md)** — templates, references, and
  the parent/child hierarchy (equipment, hands, granted abilities).
- **[Tagged Data](../concepts/tagged-data.md)** — the schema as your project's
  data dictionary, and standalone use on plain actors.
- **[Commands & Undo](../concepts/commands-and-undo.md)** — transactions and
  authoring a custom command that undoes itself correctly.
- **[Derived State & Events](../concepts/derived-state.md)** — the subscription
  rule that keeps every reactive consumer correct through undo and load.
- **[Stats & Modifiers](../concepts/stats-and-modifiers.md)** and
  **[Evaluators](../concepts/evaluators.md)** — buffs, gear, auras, and
  data-driven formulas.

<!-- pluginlink: gameentity-reference -->
<!-- pluginlink: taggeddata-reference -->
