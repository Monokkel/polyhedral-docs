# GridGraph Guides

For developers with the plugin enabled who want to get a board on screen and answer
spatial questions about it. Each section is a short, followable recipe. For the model
behind them read [Grids & Occupancy](../../concepts/grids-and-occupancy.md); for the
full type-by-type surface see the reference pages —
[Graph & topologies](reference-graph.md),
[Generation & patterns](reference-generation.md),
[Movement & reachability](reference-movement.md), and
[Visualization](reference-visualization.md).

## Build a board

A board is built **once**, at edit time. The grid component hosts it in the world; you
give it a shape, optionally conform it to your level's geometry, then bake. The bake is
authoritative — it is serialized and reloaded verbatim, never recomputed at play time.

=== "Blueprint"
    1. Add a **Grid Graph** component to your board actor. It holds a **Grid** — open
       it and pick a **Topology**: **Square** (tick **Allow Diagonals** for eight-way),
       **Hex**, or **Freeform**.
    2. Set **Seed Pattern** to a shape — choose **Square Rectangle** and give it a min
       and max coordinate for a 10&times;10 board. (Optional: set an **Exclude Pattern**
       to punch out declarative holes like water or a wall line.)
    3. To follow your level's geometry, tick **Conform To Terrain** and open **Conform
       Settings**: choose the **Floor Channel** that standable ground blocks, the
       **Wall Channel** that blocks steps between cells, and set **Max Levels** above 1
       to discover stacked storeys. Leave conform off for a purely abstract board.
    4. Tick **Is Main Grid** so the board registers as the world's board.
    5. Click **Generate Grid**. The bake proposes the cells, conforms them to the level
       once, and stores the result.

=== "C++"
    ```cpp
    // The component hosts the board in the world; the UGridGraph is the board data.
    UGridGraph* Grid = GridComponent->Grid;

    // 1. Propose cells from a pure-shape pattern — a 10x10 square.
    FSquareRectanglePatternData Shape;
    Shape.MinCoord = FIntVector(0, 0, 0);
    Shape.MaxCoord = FIntVector(9, 9, 0);
    Grid->GenerateGridFromPattern(FInstancedStruct::Make(Shape), FIntVector::ZeroValue);

    // 2. Conform the cells to the level once: set each cell's height to the ground,
    //    drop cells over holes, cut connections blocked by a wall or too-tall a step,
    //    and (MaxLevels > 1) discover stacked storeys. An edit-time bake.
    FTerrainConformSettings Settings;
    Settings.FloorChannel = ECC_WorldStatic;   // what standable ground blocks
    Settings.WallChannel  = ECC_WorldStatic;   // what blocks a step between cells
    Settings.MaxLevels    = 3;                  // discover up to three storeys
    Grid->ConformToTerrain(Grid->GetAllNodes(), Settings);
    ```

!!! note "Conform is a one-time bake"
    Conforming reads the level exactly once, at bake time, and stores the result — a
    self-contained board value. Play never re-runs it. Extra storeys are separate
    **levels** of the board; they connect only by ramps and by movement abilities like
    jumping, never by a free vertical link. After the bake no query ever touches the
    level geometry again.

## Query the grid

Reading the board is free — a plain lookup, no command and no undo cost. Three common
reads:

=== "Blueprint"
    - **Get Neighbors** on the grid returns the cells directly connected to one cell.
    - Coordinate and world: **Get Node At** (a coordinate &rarr; its cell), **Get World
      Position** (a cell &rarr; where it sits in the world), and **World To Coord** (a
      world position &rarr; the nearest coordinate). The grid *component* also offers
      **Get Nearest Node** — the nearest existing cell to a world position.
    - Cells in a coordinate range: set a pattern to **Square Disk** with radius 2, then
      call **Resolve Handles** on the pattern subsystem, passing the grid and the center
      cell — you get every cell within 2 by coordinate math.

=== "C++"
    ```cpp
    // The cells directly connected to one cell.
    TArray<FGridNodeHandle> Around = Grid->GetNeighbors(Cell);

    // Coordinate <-> world.
    FGridNodeHandle At    = Grid->GetNodeAt(FIntVector(3, 4, 0));
    FVector         World = Grid->GetWorldPosition(At);
    FIntVector      Coord = Grid->WorldToCoord(World);   // nearest coordinate

    // Every cell within 2 of a center, by coordinate math (a pattern).
    FSquareDiskPatternData Disk;
    Disk.Radius = 2;
    Disk.Metric = ESquareDiskMetric::Chebyshev;   // square area; Manhattan for a diamond
    TArray<FGridNodeHandle> Within2 = UGridPatternSubsystem::Get()->ResolveHandles(
        FInstancedStruct::Make(Disk), Grid, Center);
    ```

!!! note "Freeform boards have no coordinates"
    A freeform board places nodes at arbitrary positions with no coordinate system, so
    the coordinate reads and coordinate patterns above don't apply to it. But **Get
    Neighbors** and reachability work exactly the same, because both are questions about
    the grid's connections, not about coordinates.

## Compute a unit's reachable set

A move preview is a **reachability** query: flood outward from the unit's cell,
spending a budget, and collect every cell reachable within it. Costs are whole scaled
integers — one straight step is **10** (a diagonal is **14** under the standard
diagonal-cost modifier) — so a "move 5" unit searches on a budget of **50**.

=== "Blueprint"
    1. Call **Compute Reachability** from the unit's cell with **Max Cost** `50`.
    2. Read the result: **Get Reached Nodes** (every reachable cell), **Is Reachable**
       (one cell), and **Get Cost To Reach** (the cheapest cost to a cell).
    3. **Reconstruct Path** from the result to any reached cell for the step-by-step
       route.

=== "C++"
    ```cpp
    // A "move 5" unit: one straight step costs 10, so its budget is 50.
    FGridReachability Reach = Grid->ComputeReachability(StartCell, /*MaxCost=*/50);

    // Read it back through the library helpers (the result is keyed by cell handle).
    TArray<FGridNodeHandle> Reachable = UGridBlueprintLibrary::GetReachedNodes(Reach);
    bool bCanReach = UGridBlueprintLibrary::IsReachable(Reach, TargetCell);

    int32 Cost;   // scaled: 10 per straight step
    UGridBlueprintLibrary::GetCostToReach(Reach, TargetCell, Cost);

    // The cheapest route to a cell (Source-first, both endpoints included).
    TArray<FGridNodeHandle> Path = UGridBlueprintLibrary::ReconstructPath(Reach, TargetCell);
    ```

!!! note "Per-unit movement rules live with the entity"
    This plain flood walks the grid's connections at a flat cost. A real unit brings its
    own **movement sources** (an ordinary walk, a cliff-jump, flying) and its own **cost
    modifiers** (an ordered list — a terrain surcharge, a hazard, "blocked by an enemy")
    — together its **movement profile**. Those are assembled from the unit's data by the
    GridEntity plugin, documented in its own section; the entity-aware reachability call
    is covered in [Movement & reachability](reference-movement.md). The budget rule is
    unchanged: 10 per step, 50 for "move 5".

## Show a shape versus a range

The classic grid-tactics bug is answering a shape question with a movement tool, or the
reverse. The two are different machinery on purpose.

!!! warning "Shape is not range"
    A **pattern** answers *geometric proximity* — every cell within 2 — by coordinate
    math. Walls, closed doors, and teleporters cannot bend it: the shape is the shape.
    That is exactly right for a blast, an aura, or a targeting ring.

    **Reachability** answers *movement range* — the cells a unit can actually get to —
    by a cost-aware search that respects the grid's connections and the unit's movement
    rules. That is right for a move preview.

    At a wall the two sets diverge on purpose: the blast still covers the cells behind
    the wall; the reachable set stops at it. Use a pattern for a blast and reachability
    for movement — never the reverse.

=== "Blueprint"
    - **Shape (a blast):** set a pattern to **Square Disk**, radius 2, and **Resolve
      Handles** at the caster's cell — the blast's cells, by coordinate math.
    - **Range (a move):** **Compute Reachability** from the unit's cell with its budget
      — only the cells it can travel to.

=== "C++"
    ```cpp
    // A shape query: every cell within 2, by coordinate math. Walls do not bend it.
    FSquareDiskPatternData Blast;
    Blast.Radius = 2;
    Blast.Metric = ESquareDiskMetric::Chebyshev;
    TArray<FGridNodeHandle> BlastCells = UGridPatternSubsystem::Get()->ResolveHandles(
        FInstancedStruct::Make(Blast), Grid, CasterCell);

    // A movement query: where the unit can actually reach. "Move 5" -> budget 50.
    FGridReachability Reach = Grid->ComputeReachability(UnitCell, /*MaxCost=*/50);
    TArray<FGridNodeHandle> MoveCells = UGridBlueprintLibrary::GetReachedNodes(Reach);
    ```

## Visualize the grid

Drawing is a separate layer from the queries. A **Grid Visualization Driver** component
takes a node set — or a reachability result — and runs a **visualization profile**, a
data asset whose lists of presenters (a decal fill, an outline, a path arrow) decide
how it looks. Stacking presenters is how you combine, say, a fill and an outline in one
highlight.

=== "Blueprint"
    1. Add a **Grid Visualization Driver** component to the grid actor and set its
       **Profile** to a **Grid Visualization Profile** asset. Add presenters to the
       profile's **Area Visualizers** (e.g. a decal fill plus an outline) and **Path
       Visualizers** (e.g. an arrow).
    2. To draw a **move range**: compute reachability, then call **Show Area From
       Reachability** with the result — the reachable cells light up.
    3. To draw a **path**: with the range already shown, call **Set Hovered Cell** with
       the cell under the cursor — the driver traces the route to it over the flow field
       it already has, no recompute. (Or call **Show Path** with an explicit ordered
       path from **Reconstruct Path**.)
    4. **Clear Area** takes the whole highlight down; **Clear Path** removes just the
       route.

=== "C++"
    ```cpp
    // The driver draws whatever node set you hand it, using its profile's presenters.
    UGridVisualizationDriverComponent* Viz = /* the driver component on the grid actor */;

    // Draw the move range (this also stores its flow field for hover paths).
    FGridReachability Reach = Grid->ComputeReachability(UnitCell, /*MaxCost=*/50);
    Viz->ShowAreaFromReachability(Reach);

    // Draw the route to a hovered cell — reconstructed over the stored flow field.
    Viz->SetHoveredCell(HoveredCell);

    // Or draw an explicit path.
    TArray<FGridNodeHandle> Path = UGridBlueprintLibrary::ReconstructPath(Reach, HoveredCell);
    Viz->ShowPath(Path);

    Viz->ClearArea();   // take the whole highlight down
    ```

!!! tip "The driver draws what you hand it"
    It never reaches for a selection or hover source of its own, so a game controller,
    an AI, or the entity layer all drive it the same way by calling **Show Area** /
    **Show Path**. A game with no game-entity layer can drive it straight from a
    reachability flood or a hand-authored set.
