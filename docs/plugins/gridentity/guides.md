# GridEntity Guides

For developers with the plugin enabled who want to get a unit onto the board and
moving. Each section is a short, followable recipe. For the full type-by-type
surface see the [API Reference](reference.md); for the model behind it all see
[Grids & Occupancy](../../concepts/grids-and-occupancy.md).

Every recipe assumes you already have a [board](../gridgraph/index.md) in the world
and [entities](../gameentity/index.md) in your game state. All the write calls below
route through GameEntity commands, so each one is on the undo stack the moment it
returns.

## Place a unit on the board

The cleanest way to put a unit on a cell is to create it *already there*. The
create-at-cell call seeds the starting cell into the entity's creation, so there is
never an instant where the unit exists but stands nowhere — and the whole thing is
one undoable step.

=== "Blueprint"
    1. Call **Create Entity At Cell** with a template id, the starting **Cell**, and
       a **Facing** rotation. Leave **Grid** unset to use the world's main board.
    2. It returns an **entity reference** already standing on that cell. The
       occupancy index already lists it there, and — if the template names a token
       class — its mini has already spawned on the cell.
    3. Wire an **Undo** button on the command stack. Undoing the creation removes the
       unit *and* clears it from the cell in one step.

=== "C++"
    ```cpp
    // Born placed: the cell is seeded into the create command — no un-placed instant.
    FPGeEntityRef Goblin = UPGxPlacementLibrary::CreateEntityAtCell(
        this, /*TemplateId=*/"Goblin", StartCell, FRotator::ZeroRotator);
    ```

!!! tip "Off-grid pieces use a transform instead"
    For a piece that lives off the board — a freely dragged token, an off-board
    reserve — use **Create Entity At Transform** with a free world transform. It is
    the exact off-grid mirror of create-at-cell, and it is undoable the same way.

## Move units on the board

Repositioning a placed unit is a single call. **Move Entity To Cell** rewrites the
unit's placement to the new cell in one step; the occupancy index re-buckets it, and
the unit's token re-snaps (or, with a move cue, walks there — see below).

=== "Blueprint"
    - Call **Move Entity To Cell** with the unit's **reference** and the target
      **Cell**. Leave **Grid** unset for the main board. One **Undo** puts it back.

=== "C++"
    ```cpp
    // An ordinary command underneath — undoable like everything else.
    UPGxPlacementLibrary::MoveEntityToCell(this, Goblin, TargetCell);
    ```

The move call is deliberately *bare*: it changes position and nothing else. It does
not spend a movement point, because most games want to decide that separately. When
a real player move both repositions the unit **and** spends a point, you want a
single **Undo** to reverse the whole thing — so wrap the two steps in a
[transaction](../../concepts/commands-and-undo.md).

=== "Blueprint"
    1. Command stack &rarr; **Begin Transaction** (`"Move Unit"`).
    2. Call **Move Entity To Cell** to reposition the unit.
    3. Call **Modify Stat** on the game state to spend the movement point (for
       example `Stat.Movement`, delta `-1`).
    4. Command stack &rarr; **Commit Transaction**. One **Undo** now reverses both.

=== "C++"
    ```cpp
    UPCsCommandStack* Stack = UPCsCommandStack::Get(this);

    Stack->BeginTransaction(TEXT("Move Unit"));
    UPGxPlacementLibrary::MoveEntityToCell(this, Goblin, TargetCell);   // reposition
    State->ModifyStat(Goblin, MovementTag, -1);                        // spend a point
    Stack->CommitTransaction();                                        // one undo step
    ```

!!! note "The move joins your open transaction"
    Move Entity To Cell already brackets its own internal work, so calling it inside
    a transaction *joins* your transaction rather than starting a separate undo step.
    You get exactly one undo entry for the whole move — the reposition and the point,
    reversed together. Transactions are covered in full in the
    [CommandSystem guides](../commandsystem/guides.md#group-changes-into-one-undo-step-with-a-transaction).

## Query who is on a cell

The reverse question — "who is standing here?" — is answered by the occupancy index.
It reports; it never decides whether a move is legal. Capacity, blocking, and
stacking are *your game's* rules, which you layer on top of what it reports.

=== "Blueprint"
    1. **Get** the occupancy subsystem (its static **Get**).
    2. Call **Get Entities On Cell** with the cell. You get every occupant, in a
       stable order (ascending entity id — creation order).
    3. Read the length, or check factions/tags on the occupants, to enforce your own
       rule — "is this cell already full?", "does an enemy hold it?".

=== "C++"
    ```cpp
    UPGxOccupancySubsystem* Occupancy = UPGxOccupancySubsystem::Get(this);

    // Multi-occupant, deterministic order — the same list live, after undo, in replay.
    TArray<FPGeEntityRef> Here = Occupancy->GetEntitiesOnCell(TargetCell);

    // Your rule, over what the index reports — the index itself never adjudicates.
    const bool bCellIsFull = Here.Num() >= MyCapacityForCell(TargetCell);
    ```

!!! tip "Close the loop with a movement cost modifier"
    Writing a blocking rule twice — as an occupancy query *and* as a movement cost
    modifier that forbids the step — makes the highlight and the legality check agree
    by construction. Granting the shipped faction-block modifier (below) does the
    movement half for you.

## Grant and revoke a movement capability

A unit's movement is assembled from its granted capabilities: **movement sources**
(the ways it can step — the ordinary walk, a cliff-jump, flight) and **cost
modifiers** (what a step costs, or what is forbidden). Each is granted as tagged data
under a `Data.Movement.*` key, tagged with the item that granted it, so revoking
exactly what an item gave — when it is unequipped — is one call, and both undo.

=== "Blueprint"
    1. On equip, call **Grant Movement Source** with a **Cliff Jump** source struct,
       the source key `Data.Movement.Source.CliffJump`, and the boots as the **Grant
       Source** so the grant can be revoked cleanly later.
    2. Call **Compute Entity Reachability** for the unit — the reachable set now
       includes ledges it could not reach before.
    3. On unequip, call **Revoke Movement Grants By Source** with the boots. Recompute
       and the extra reach is gone. **Undo** the unequip and it returns.

=== "C++"
    ```cpp
    // A pair of boots grants a cliff-jump capability, tagged with what granted it.
    FGridSuccessorSource_CliffJump Jump;
    Jump.DefaultJumpHeight = 2;   // may drop or rise up to two levels
    UPGxMovementLibrary::GrantMovementSource(
        this, Unit, PGxMovementTags::TAG_Data_Movement_Source_CliffJump,
        FInstancedStruct::Make(Jump), /*GrantSource=*/Boots);

    // The reach recomputes from the new profile — nothing to invalidate.
    FGridReachability Reach =
        UPGxMovementLibrary::ComputeEntityReachability(this, Unit, Grid, /*MaxCost=*/50);

    // Unequip: revoke exactly what the boots granted — one undo unit.
    UPGxMovementLibrary::RevokeMovementGrantsBySource(this, Unit, Boots);
    ```

!!! note "The default 'can't enter an enemy's cell' rule"
    The one blocking rule most tactical games want is shipped: **Grant Default
    Faction Block** grants a cost modifier that forbids stepping onto a cell held by
    a unit of another faction. Pass the tag namespace whose children name your
    factions. A game wanting a different relationship — pass through allies, swap with
    an ally — grants its own cost modifier instead.

## Show a unit with a grid token

A unit's on-screen mini is a [token](../tokensystem/index.md), and a unit on the
board wants a **grid-aware** token — one that stands on the right cell and walks a
move instead of teleporting. The framework ships that token base; you subclass it and
assign it, and the registry does the rest.

=== "Blueprint"
    1. Create a Blueprint subclass of **PGx Grid Token Actor** (call it
       `BP_GridToken`). Give it a mesh; optionally set **Move Speed**.
    2. In the unit's data-table row, add a **Tagged Data** entry keyed
       `Data.Presentation.TokenClass` whose value is `BP_GridToken`.
    3. Create the unit with **Create Entity At Cell**. The token spawns *on the cell*
       automatically — no positioning code — because it derives its world position
       from the unit's placement.
    4. To make a move *walk* instead of snap, after the move commits build a
       **Cue.Move** carrying the path (from **Build Path From Cells**) and play it
       through the token subsystem. The grid token walks the route.

=== "C++"
    ```cpp
    // The token class is authored on the template (usually in its data-table row).
    // On Create Entity At Cell the registry spawns it already standing on the cell —
    // its world position comes from the unit's placement, so there is no origin-pop.

    // To animate a move, hand a Cue.Move the world-space path and play it:
    TArray<FVector> Path = UPGxPlacementLibrary::BuildPathFromCells(this, PathCells, Grid);

    FPTkMovePayload Move;
    Move.Path = Path;

    FPTkCue Cue;
    Cue.Intent  = PTkTokenTags::TAG_Cue_Move;
    Cue.Payload = FInstancedStruct::Make(Move);

    UPTkCueContext* Ctx = NewObject<UPTkCueContext>(this);
    UPTkTokenSubsystem::Get(this)->PlayCue(UnitRef, Cue, Ctx);   // the token walks the route
    ```

!!! tip "Snapping is automatic; walking is the polish"
    You do not have to play a cue at all — with only the token class assigned, the
    mini is always on the correct cell and re-snaps on every move, undo, and load.
    The move cue is the opt-in animation on top. The full token and cue model is in
    [Tokens & Cues](../../concepts/tokens-and-cues.md) and the
    [TokenSystem guides](../tokensystem/guides.md#play-a-cue-yourself).

## Highlight a selected unit's reachable tiles

To paint a selected unit's reachable tiles on the board, GridEntity ships an optional
feeder that connects a selection to a [grid render surface](../gridgraph/reference-visualization.md).
You bind it to a driver once, then tell it what is selected; it computes the unit's
reachability and drives the highlight, rebuilding it whenever the unit moves or the
board changes.

=== "Blueprint"
    1. **Get** the selection-visualization subsystem and call **Set Driver** with your
       grid visualization driver component (it adopts the driver's grid).
    2. Optionally call **Set Max Move Cost** with the unit's movement budget (scaled
       integer — a "move 5" unit is `50`).
    3. On selection, call **Set Selected Entity** with the unit. The reachable tiles
       light up. On deselection, call **Clear Selection**.
    4. To preview a path, feed the hovered cell with **Set Hovered Cell** — the driver
       reconstructs the path over the shown flood with no recompute.

=== "C++"
    ```cpp
    UPGxSelectionVisualizationSubsystem* Vis = UPGxSelectionVisualizationSubsystem::Get(this);
    Vis->SetDriver(GridDriverComponent);   // bind the render surface + its grid
    Vis->SetMaxMoveCost(50);               // a "move 5" budget

    Vis->SetSelectedEntity(Unit);          // reachable tiles light up
    Vis->SetHoveredCell(HoveredCell);      // paint the path to the hovered cell
    ```

!!! note "Undo updates the highlight for free"
    The feeder is a reactive consumer: it recomputes when the selected unit moves,
    when its capabilities or stats change, or when any unit steps onto or off a cell
    (occupancy shifted). Undo a move and the highlight floods from the restored cell
    on its own — you wire nothing.
