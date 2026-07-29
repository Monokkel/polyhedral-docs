# Grids and Occupancy

For developers building a board — square, hex, or freeform; flat or multi-storey —
who need "where is everything?" and "where can this unit go?" answered correctly
through undo, save, and replay. After reading, you'll know how the grid is generated
and what it stores, the difference between a shape query and a movement query, how
per-unit movement is assembled from data, how entities stand on and move across
the board, and what happens when the board itself is dug, collapsed, or built up
mid-game.

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
heights, and gameplay tags on cells and connections. It is *not* touched as units move —
a unit's position is data on the unit, never a mark on the map.

The board's *structure*, though, is allowed to change during play: a tunnel is dug, a
bridge collapses, a city grows tile by tile. What keeps such a board correct is not that
edits are rare — it is that **every structural edit is a command**, on the same stack as
everything else, so a change to the board rewinds with the action that caused it. See
[When the board itself changes](#when-the-board-itself-changes) at the end of this page.

!!! note "Freeform boards still answer the important questions"
    A freeform graph has no coordinates, so shape queries that need coordinate math
    (below) don't apply to it — but "which cells connect to this one" and "where can
    this unit reach" work exactly the same, because both are questions about the graph's
    connections, not about coordinates.

For the full graph API — nodes and connections, topologies, and hosting a grid in the world — see [The Grid & Topologies](../plugins/gridgraph/reference-graph.md#the-grid-graph) in the GridGraph reference.

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

Ready to build one? Follow [Build a board](../plugins/gridgraph/guides.md#build-a-board) in the GridGraph guides.

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

For the API behind each: [patterns](../plugins/gridgraph/reference-generation.md#patterns) answer the shape query, and [reachability](../plugins/gridgraph/reference-movement.md#reachability) answers the movement query.

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
        this, Unit, PGxMovementTags::TAG_Data_Movement_Source_CliffJump,
        FInstancedStruct::Make(Jump), /*GrantSource=*/Boots);

    // The reach recomputes from the new profile — no cache to invalidate.
    FGridReachability Reach =
        UPGxMovementLibrary::ComputeEntityReachability(this, Unit, Grid, /*MaxCost=*/50);

    // Unequip: revoke exactly what the boots granted — one undo unit.
    UPGxMovementLibrary::RevokeMovementGrantsBySource(this, Unit, Boots);
    ```

For the full movement API — sources, cost modifiers, and the reachability result — see [Movement & Reachability](../plugins/gridgraph/reference-movement.md#the-movement-profile) in the GridGraph reference.

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

For the full placement API — create-at-cell, move-to-cell, and how a move joins your transaction — see [Placement](../plugins/gridentity/reference.md#placement) in the GridEntity reference; to move units step by step, follow [Move units on the board](../plugins/gridentity/guides.md#move-units-on-the-board) in the guides.

## Bigger than one cell, and pointing somewhere

Two things ride along with placement, and both follow the same rule: they are entity data,
so they undo, save, and replay exactly like position does.

**A footprint** is the set of cells a unit covers — a 2×2 ogre, a 1×3 siege engine. It is
authored on the [template](entities-as-data.md), as offsets from an anchor cell, and every
instance inherits it until one overrides. Because the offsets are relative, a unit that
turns carries its shape around with it.

**Facing** is which way the unit is pointing, stored as a whole number of rotation
**steps** — four on a square board, six on a hex — never as an angle. Whole steps are what
make facing exact: there is no float to drift between two machines or between a live run
and its replay. The space around a facing unit divides into **arcs** — front, flank, rear —
which is all a backstab rule needs to ask.

Turning a unit rotates its footprint offsets by the same steps, and the framework guarantees
the coordinate math and the visible transform agree: rotating the offsets by *k* steps gives
the same cells as yaw-rotating the world positions by *k* step-angles. A rotated shape never
drifts from what the player sees.

Two questions follow from a shape, and they are genuinely different:

- **Does it fit?** Do all the covered cells exist on the board?
- **Does it hold together?** Is the covered set one connected body, or has a wall split it
  so the unit would straddle a chasm?

Both are answered for you, and the framework combines them into a single "can this unit
stand here?" check.

!!! warning "That check is a convention, not a guard"
    The placement calls do **not** run it for you — a game that skips it can place a
    four-cell unit into a cell it does not fit. Movement is the exception: the fit test is
    injected into every movement query automatically, so a unit can never *walk* somewhere
    its shape doesn't go. It is placing and spawning where you have to ask.

A freeform board has no coordinates and a single rotation step, so footprints and facing
don't apply there.

See [Facing & Arcs](../plugins/gridentity/reference-facing.md) and
[Footprints](../plugins/gridentity/reference-footprints.md) in the GridEntity reference.

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

It runs both ways, and the reverse direction is a **set**: a footprinted unit is on every
cell it covers, not just its anchor. Ask "which cells is this entity on?" rather than
assuming one, and "is this entity on this cell?" when that is the actual question — a 2×2
unit answers yes for four different cells.

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

For the full occupancy API — listing a cell's entities and the order they come back in — see [Occupancy](../plugins/gridentity/reference.md#occupancy) in the GridEntity reference.

## When the board itself changes

The stage is static about *passability* — a connection never says "this unit may pass" —
but it is not frozen. Games dig tunnels, drop bridges, and let players build. So the
board's structure gets the same treatment every other piece of game state gets: **it is
changed through commands.**

Adding or removing a cell, adding or removing a connection, tagging either, changing a
cell's height, overriding a connection's cost — each is a command on the same stack as
the entity commands. That single fact is what makes a mutable board safe:

- Undo an action that dug a tunnel *and* walked a unit through it, and tunnel and unit
  rewind together, in the right order. There are no two histories to reconcile.
- A replay reproduces the dug board exactly, because the dig is in the log like anything
  else.
- Everything downstream — occupancy, tokens, visualizers — is told what changed, on the
  way forward and on the way back.

A tool wants one more thing: a whole brush stroke of edits — nine cells dragged over —
should be **one** press of undo, not nine. Wrapping the edits in a transaction gives
exactly that, the same way wrapping a move and its movement-point spend does.

=== "Blueprint"
    - Call **Submit Grid Command** with a structural command to change the board.
    - Wrap several in **Begin Transaction** / **Commit Transaction** on the command
      stack so the whole stroke is one undo step.
    - **Is Board Authoring Allowed** tells a tool button whether a wholesale re-conform
      of the board is legal right now.

=== "C++"
    ```cpp
    // One brush stroke: N edits, one undo unit.
    {
        FPGcGridTransactionScope Stroke(Stack, Grid, TEXT("Dig tunnel"));

        FPGcCmd_AddNode Add;
        Add.bHasCoord = true;
        Add.Coord     = FIntVector(4, 0, 0);
        UPGcGridCommandLibrary::SubmitGridCommand(this, FInstancedStruct::Make(Add));
        // ...more edits...
    }

    Stack->Undo();   // the whole stroke rewinds
    ```

### A unit whose cell no longer exists

Remove a cell someone is standing on and that unit's placement becomes **dangling**: it
names a cell that is not there any more. This is a legal state, not a corruption, and the
framework is careful about it in three separate ways.

**It is always detectable.** Cell handles are never renumbered and never recycled, so a
dangling reference can never quietly come to mean some *other* cell that was created
later. It either names the cell it always named or names nothing — there is no third
possibility, and no version stamp needed to tell them apart.

**Occupancy drops it, and says who was affected.** The "who is on this cell" index
reconciles from the change like it reconciles from everything else: a placement it can no
longer resolve simply leaves the index, and each unit whose cell vanished is *reported* as
displaced. Your rules and your presentation layer can both listen for that.

**Undo heals it.** Undoing the removal makes the old placement valid again, with zero
compensating writes — nobody has to remember where the unit used to be, because nothing
ever overwrote it. That is the payoff for treating the board edit as a command rather than
as a special case.

!!! warning "The framework reports displacement; it never decides where a unit goes"
    Teleport the unit to the nearest safe cell? Kill it? Leave it dangling until the
    player resolves it? Those are your game's rules, and the framework deliberately
    declines to pick one — it hands you the fact that a unit's cell is gone and gets out
    of the way. Write the answer as an ordinary rule that reacts to the displacement, and
    it will undo with everything else.

    GridEntity's cell-removal call wraps the whole thing for you: rules get a chance to
    veto the removal *before* it happens, each affected unit gets a signal after it, and
    the behaviour is identical in live play and inside a what-if run.

For the structural editing API — the command family, brush-stroke transactions, and the
change notification a custom visualizer subscribes to — see the
[GridCommands](../plugins/gridcommands/index.md) plugin section.
