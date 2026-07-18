# Grids and Occupancy

For developers building a board — square, hex, or freeform; flat or multi-storey —
who need "where is everything?" and "where can this unit go?" answered correctly
through undo, save, and replay. After reading, you'll know how the grid is generated
and what it stores, the difference between a shape query and a movement query, how
per-unit movement is assembled from data, and how entities stand on and move across
the board.

One idea carries this whole page: **the grid is a static stage. Everything per-unit
and per-moment is a query, never a property of the map.** A connection between two
cells means "these cells are linked" — it never means "this unit may pass." Who can
walk where, and for how much, is computed fresh each time you ask, from the unit's
own data. Bake that "may pass" judgement into the map and it stops being right the
moment a unit gains boots, loses a leg, or the board is undone.

## A board is a graph with a topology

A board is a graph: **cells** (the nodes) joined by **connections** (the edges). A
**topology** object owns the shape math — coordinates, which cells are neighbors, and
where each cell sits in the world:

- **Square** — four-way, or eight-way when diagonals are allowed.
- **Hex** — six-way, on axial coordinates.
- **Freeform** — nodes at arbitrary positions with no coordinates at all; connections
  are whatever you wire by hand.

The graph stores only durable facts: the cells, the connections between them, per-cell
heights, and gameplay tags on cells and connections. It is *not* touched as units move.
Structural edits — severing a bridge, opening a permanent shortcut — are rare and
deliberate, not part of ordinary play. That stability is what lets the same board hash,
save, and rewind identically.

!!! note "Freeform boards still answer the important questions"
    A freeform graph has no coordinates, so shape queries that need coordinate math
    (below) don't apply to it — but "which cells connect to this one" and "where can
    this unit reach" work exactly the same, because both are questions about the graph's
    connections, not about coordinates.

<!-- pluginlink: gridgraph-reference -->

## Building the board

Generation is two steps.

First, a **pattern** proposes candidate cells. A pattern is pure shape math — a
rectangle, a filled disk, a ring — with parameters and nothing else, so it lives happily
in a data table or an asset. Patterns compose: you can union, subtract, and intersect
them to carve the exact footprint you want.

Second, **conforming the grid to your level's geometry** reads the actual level *once*.
For each cell it traces the world and: sets the cell's height to the ground under it,
drops cells that hang over holes, cuts connections blocked by a wall or by a step too
tall to climb, and — when you ask for it — discovers stacked floors above the ground,
giving you a multi-storey board. Extra floors are separate **levels** of the board; they
connect only by ramps and by movement abilities like jumping, never by a free vertical
link. After this pass, no query ever touches the level geometry again — the board is a
self-contained value.

=== "Blueprint"
    On the grid component:

    1. Set **Seed Pattern** to a shape — pick **Square Rectangle** and give it a min and
       max coordinate for a 10&times;10 board.
    2. Tick **Conform To Terrain** and open **Conform Settings**: choose the **Floor
       Channel** the ground blocks, the **Wall Channel** that blocks steps between cells,
       and set **Max Levels** above 1 to discover stacked floors.
    3. Click **Generate Grid**. The bake proposes the cells, conforms them to the level,
       and stores the result — it is not recomputed at play time.

=== "C++"
    ```cpp
    // The component hosts the board in the world; the UGridGraph is the board data.
    UGridGraph* Grid = GridComponent->Grid;

    // 1. Propose cells from a pure-shape pattern (a 10x10 square).
    FSquareRectanglePatternData Shape;
    Shape.MinCoord = FIntVector(0, 0, 0);
    Shape.MaxCoord = FIntVector(9, 9, 0);
    Grid->GenerateGridFromPattern(FInstancedStruct::Make(Shape), FIntVector::ZeroValue);

    // 2. Read the level once: set heights, drop holes, cut blocked connections,
    //    and (MaxLevels > 1) discover stacked floors.
    FTerrainConformSettings Settings;
    Settings.FloorChannel = ECC_WorldStatic;   // what standable ground blocks
    Settings.WallChannel  = ECC_WorldStatic;   // what blocks a step between cells
    Settings.MaxLevels    = 3;                  // discover up to three levels
    Grid->ConformToTerrain(Grid->GetAllNodes(), Settings);
    ```

<!-- pluginlink: gridgraph-guides -->

## Two spatial questions, two tools

"Cells within 2" and "cells this unit can reach" sound alike and are answered by
completely different machinery. Reaching for the wrong one is the classic grid-tactics
bug.

!!! warning "Shape is not range"
    A **pattern** answers *geometric proximity* — "every cell within 2" — by coordinate
    math. Walls, closed doors, and teleporters cannot bend it. That is exactly what you
    want for a blast radius, an aura, or a targeting ring: the shape is the shape.

    **Reachability** answers *movement range* — "the cells this unit can actually get to"
    — by a cost-aware search that respects the grid's connections, the cost of each step,
    and the unit's own abilities. That is what you want for a move preview.

    At a wall the two sets diverge on purpose: a blast still covers the cells behind the
    wall; the reachable set stops at it. Use a pattern for a blast and reachability for
    movement — never the reverse.

=== "Blueprint"
    - **Shape:** set a pattern to **Square Disk** with radius 2, then call **Resolve
      Handles** on the pattern subsystem, passing the grid and the caster's cell — you get
      the blast's cells by coordinate math.
    - **Range:** call **Compute Entity Reachability** for the moving unit — you get only
      the cells it can travel to. Read the result with **Get Reached Nodes**, **Is
      Reachable**, and **Get Cost To Reach**.

=== "C++"
    ```cpp
    // A shape query: every cell within 2 by coordinate math — walls do not bend it.
    FSquareDiskPatternData Blast;
    Blast.Radius = 2;
    Blast.Metric = ESquareDiskMetric::Chebyshev;
    TArray<FGridNodeHandle> BlastCells = UGridPatternSubsystem::Get()->ResolveHandles(
        FInstancedStruct::Make(Blast), Grid, CasterCell);

    // A movement query: where THIS unit can actually reach, respecting connections,
    // costs, and its own abilities. Budget 50 = a "move 5" unit (one step costs 10).
    FGridReachability Reach =
        UPGxMovementLibrary::ComputeEntityReachability(this, Unit, Grid, /*MaxCost=*/50);
    ```

<!-- pluginlink: gridgraph-reference -->

## Movement is per-unit, assembled from data

Because the map never says "this unit may pass," each unit brings its own answer. A
unit's **movement profile** is two lists, composed from data much the way an
[evaluator](evaluators.md) formula is composed from steps rather than written as code:

- **Movement sources** — each one proposes steps the unit *could* take. The ordinary
  walk follows the grid's connections; a cliff-jump invents a hop down a ledge, its reach
  driven by the unit's jump stat; there are gap-crossing, leaping, and flying sources too.
  A unit's candidate steps are the union of its sources.
- **Cost modifiers** — an ordered list that reprices or forbids every proposed step, in a
  fixed order: a terrain surcharge, a hazard aura, "blocked by an enemy occupant." Each
  sees the running cost, so an added surcharge and a multiplier compose predictably; a
  modifier can also block a step outright.

Gaining or losing a capability is granting or revoking one piece. "Boots of leaping"
grant a source; a "curse of slowness" grants a cost modifier. Grants are entity data
written through [commands](commands-and-undo.md), so they undo, save, and replay with
everything else — and revoking exactly what an item granted, when it is unequipped, is a
single call.

Running the query is a pure function. It snapshots the unit's tags and stats once — those
are the same [current stats](stats-and-modifiers.md) resolved elsewhere — reads the live
board through a read-only view, and returns the same answer for the same inputs on every
machine. Costs are whole scaled integers: one straight step is 10, a diagonal is 14. A
"move 5" unit searches on a budget of 50. Integer costs are part of why a replay cannot
drift — there is no floating-point wobble to reorder a tie between two builds.

=== "Blueprint"
    1. On equip, call **Grant Movement Source** with a **Cliff Jump** source, passing the
       boots as the grant source so it can be revoked cleanly later.
    2. Call **Compute Entity Reachability** again — the reachable set now includes ledges
       the unit could not reach before.
    3. On unequip, call **Revoke Movement Grants By Source** with the boots. Recompute, and
       the extra reach is gone. **Undo** the unequip and it returns.

=== "C++"
    ```cpp
    // A pair of boots grants a cliff-jump capability, tagged with what granted it.
    FGridSuccessorSource_CliffJump Jump;
    Jump.DefaultJumpHeight = 2;   // may drop or rise up to two levels
    UPGxMovementLibrary::GrantMovementSource(
        this, Unit, TAG_Data_Movement_Source_CliffJump,
        FInstancedStruct::Make(Jump), /*GrantSource=*/Boots);

    // The reach recomputes from the new profile — no cache to invalidate.
    FGridReachability Reach =
        UPGxMovementLibrary::ComputeEntityReachability(this, Unit, Grid, /*MaxCost=*/50);

    // Unequip: revoke exactly what the boots granted — one undo unit.
    UPGxMovementLibrary::RevokeMovementGrantsBySource(this, Unit, Boots);
    ```

<!-- pluginlink: gridgraph-reference -->

## Standing on the board: placement

A unit's position is not a transform on an actor — it is **placement**: tagged data on
[the entity](entities-as-data.md), under `Data.Placement.Cell`, written through commands
like every other change to game state. Because position is ordinary entity data, moving a
unit undoes, saves, and replays with no extra machinery ([tagged data](tagged-data.md) and
[commands](commands-and-undo.md) do all the work).

Two conveniences keep that clean:

- **Create a unit born placed.** The create-at-cell call seeds the placement into the
  entity's creation, so there is never an instant where the unit exists but stands nowhere.
- **Move a unit** with a single move-to-cell call, which repositions it in one undo step.

Off-grid pieces — a freely dragged token, an off-board reserve — use a free world
transform the same way, through the matching create-at-transform and move-to-transform
calls. When a grid isn't named, these default to the world's main board, which is the
model to build against.

The on-screen mini follows automatically — that is a job for the presentation layer, not
something you wire per move; see [Tokens & Cues](tokens-and-cues.md).

=== "Blueprint"
    - **Create Entity At Cell** with a template and a starting cell returns an **entity
      reference** already standing on the board.
    - **Move Entity To Cell** repositions it. Wrap it in a transaction if the move also
      spends a movement point, so one **Undo** reverses the whole move.

=== "C++"
    ```cpp
    // Born placed: the cell is seeded into the create command — no un-placed instant.
    FPGeEntityRef Goblin = UPGxPlacementLibrary::CreateEntityAtCell(
        this, /*TemplateId=*/"Goblin", StartCell, FRotator::ZeroRotator);

    // Moving is an ordinary command — undoable like everything else.
    UPGxPlacementLibrary::MoveEntityToCell(this, Goblin, TargetCell);
    ```

<!-- pluginlink: gridentity-reference -->

<!-- pluginlink: gridentity-guides -->

## Occupancy: who is on this cell

The reverse question — "who is standing here?" — is a **derived index**, not stored truth.
Its source of truth is each entity's command-routed placement, so it is rebuilt from
change events exactly like every other reactive consumer ([derived state](derived-state.md)
at board scale). It is correct across undo and redo with nothing of its own to capture: it
simply reconciles to whatever the entities now say.

The index is multi-occupant, and a cell's entities come back in a deterministic order —
by ascending entity id, i.e. creation order — so a query gives the same list in live play,
after an undo, and in a replay. And it only ever *reports*. It never decides whether a move
is legal.

!!! tip "Occupancy reports; your game adjudicates"
    Capacity, blocking, and stacking are your game's rules, not the index's. Write them two
    ways that reinforce each other: as queries over occupancy ("is this cell already full?")
    and as movement cost modifiers that forbid a step ("can't enter a cell an enemy holds"),
    which closes the loop with the movement profile above.

=== "Blueprint"
    - Call **Get Entities On Cell** on the occupancy subsystem to list a cell's occupants.
    - **Undo** a move and query again — the index already agrees with the rewound board.

=== "C++"
    ```cpp
    UPGxOccupancySubsystem* Occupancy = UPGxOccupancySubsystem::Get(this);

    // Who is standing on the target cell? (Deterministic order, multi-occupant.)
    TArray<FPGeEntityRef> Here = Occupancy->GetEntitiesOnCell(TargetCell);

    Stack->Undo();   // the unit is back on its old cell; the same query now agrees
    ```

<!-- pluginlink: gridentity-reference -->
