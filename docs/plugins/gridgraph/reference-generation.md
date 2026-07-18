# Generation & Patterns Reference

For developers building a board from data: the pattern types that describe a shape,
the subsystem that turns a shape into cells, and the two-step bake that generates a
grid and conforms it to your level. For the mental model behind all of this see
[Grids & Occupancy](../../concepts/grids-and-occupancy.md); for step-by-step recipes
see the [Guides](guides.md); for the graph, nodes, and topologies these produce see
the [Graph Reference](reference-graph.md).

Unlike most plugins in the framework, GridGraph's public generation types are mostly
**unprefixed** — `FSquareRectanglePatternData`, `UGridPatternSubsystem`,
`FTerrainConformSettings` — so that is what you type. Signatures below are hand-written
to show the shape of each API; they are illustrative, not source excerpts.

## Patterns

A **pattern** is pure coordinate shape math — a rectangle, a filled disk, a ring, a
line — expressed as parameters and nothing else. Because a pattern is just data it
lives happily in a data table, a save game, or an asset, and because it is pure
coordinate math it is **edge-agnostic**: it never reads the grid's connections, so a
wall, a closed door, or a teleporter cannot bend it. That is exactly what you want for
a blast radius, an aura, or a targeting shape — the shape is the shape. (One shipped
pattern breaks this rule on purpose; see [Graph Ring](#graph-ring-the-exception).)

Every shape derives from one tiny base struct, and you carry a shape around inside an
`FInstancedStruct` so any use site can offer a dropdown of every registered pattern
type:

```cpp
// The USTRUCT base every pattern shape derives from. Carry one inside an
// FInstancedStruct; a Data.* struct picker then offers every pattern type at the
// use site. It has no fields of its own — the shape's parameters live on the subclass.
struct FGridPatternData { };
```

### Resolving a pattern to cells

`UGridPatternSubsystem` is an engine subsystem — one per process, reached statically —
that turns a pattern plus an anchor into cells. It offers two resolve paths, and the
one you pick depends on whether you have a topology or a live grid in hand:

```cpp
// One engine subsystem; resolve it statically.
static UGridPatternSubsystem* Get();

// Coordinate math only — needs just a topology, not a built grid. Returns the shape's
// cells relative to Anchor. The canonical (origin) shape is computed once and cached,
// then translated by Anchor on return, so the same shape checked from many tiles
// (a sight circle from every candidate move cell) is nearly free.
TSet<FIntVector> ResolveCoords(const FInstancedStruct& Pattern,
                               const UGridTopology* Topology, FIntVector Anchor);

// Against a real grid — returns node handles. A coordinate pattern resolves its coords
// then looks them up as handles (cells not present on the grid are dropped); a graph
// pattern (Graph Ring) runs its connection-aware resolver directly instead.
TArray<FGridNodeHandle> ResolveHandles(const FInstancedStruct& Pattern,
                                       const UGridGraph* Grid, FGridNodeHandle AnchorNode);

// True if this pattern can run on that topology class (accepts any if the pattern
// registered no compatibility check).
bool IsCompatibleWithTopology(const FInstancedStruct& Pattern,
                              TSubclassOf<UGridTopology> TopologyClass) const;

// Drop every cached canonical shape. Rarely needed — only after a change that alters
// what a pattern means for a topology.
void ClearCache();
```

In Blueprint the same three resolvers are the nodes **Resolve Coords**, **Resolve
Handles**, and **Is Compatible With Topology** on the pattern subsystem.

!!! note "Coords need a topology; handles need a grid"
    `ResolveCoords` only needs a topology to do its coordinate math, so you can resolve
    a shape before any grid exists — it is how a seed pattern proposes a board.
    `ResolveHandles` needs a built grid because it returns handles to real nodes, and it
    silently drops cells the shape produced that are not on the grid. A grid-only pattern
    (Graph Ring) has no coordinate form at all and returns empty from `ResolveCoords`.

!!! tip "Adding your own pattern type in C++"
    Declare a `USTRUCT` deriving from `FGridPatternData`, then register a resolver for
    it from your module's startup with `RegisterHandler(StructType, Handler)` (balance
    it with `UnregisterHandler` on shutdown). A handler supplies either a coordinate
    resolver (cacheable) or a graph resolver (for connection-aware shapes), or both.
    Built-in shapes register themselves the same way.

## The shipped shapes

The framework ships a catalogue of shapes grouped by topology. Each is a plain
parameter struct deriving from `FGridPatternData`; the field comments below note the
defaults.

### Square

```cpp
// Inclusive, axis-aligned box of square-grid cells.
struct FSquareRectanglePatternData : FGridPatternData
{
    FIntVector MinCoord;   // default (0,0,0)
    FIntVector MaxCoord;   // default (9,9,0) — a 10x10 board
};

// Filled disk. Chebyshev metric -> a square; Manhattan metric -> a diamond.
struct FSquareDiskPatternData : FGridPatternData
{
    int32             Radius;       // default 3
    ESquareDiskMetric Metric;       // Manhattan | Chebyshev (default Chebyshev)
    int32             ExtraLayers;  // extrude the shape across this many extra levels (default 0)
};

enum class ESquareDiskMetric : uint8
{
    Manhattan,   // |dx| + |dy| <= Radius — a diamond
    Chebyshev,   // max(|dx|, |dy|) <= Radius — a square
};

// Arms radiating from the anchor along the cardinal and diagonal directions.
struct FSquareStarPatternData : FGridPatternData
{
    int32 NumArms;    // 1..8 (default 4)
    int32 ArmLength;  // steps per arm (default 3)
    int32 ArmWidth;   // perpendicular thickness of each arm (default 1)
};
```

### Hex

Hex coordinates are axial `(Q, R, Layer)`.

```cpp
// Rectangular region of hex cells in axial space.
struct FHexRectanglePatternData : FGridPatternData
{
    FIntVector MinCoord;   // default (0,0,0)
    FIntVector MaxCoord;   // default (9,9,0)
};

// Coordinate ring: every hex exactly Distance away (6*Distance cells, or 1 at 0).
// Pure coordinate math — it does NOT read connections. For a connection-aware ring
// use Graph Ring below.
struct FHexRingPatternData : FGridPatternData
{
    int32 Distance;   // default 1
};

// Classic radius-N hexagon, centered on the anchor.
struct FHexagonPatternData : FGridPatternData
{
    int32 Radius;        // default 3
    int32 ExtraLayers;   // extrude across this many extra levels (default 0)
};
```

### Any topology

```cpp
// A line from the anchor along Direction for Length steps (the anchor is included).
// Uses the topology's own offset math, so it runs on a square or a hex grid alike.
struct FCoordLinePatternData : FGridPatternData
{
    FIntVector Direction;   // default (1,0,0)
    int32      Length;      // default 3
};
```

### Graph Ring (the exception)

Every other shipped shape is pure coordinate math. **Graph Ring is the deliberate
exception**: it is connection-aware. It finds every node exactly `Distance` *connections*
from the anchor by a breadth-first walk of the grid's edges, so a wall or a severed
connection shortens it, exactly the way movement would.

```cpp
// Every node exactly Distance CONNECTIONS from the anchor, found by walking the grid's
// edges (breadth-first). Connection-aware: a wall or a cut edge changes the result.
// Needs a real grid, so it can never be a seed pattern, and it resolves ONLY through
// ResolveHandles — ResolveCoords returns empty for it.
struct FGraphRingPatternData : FGridPatternData
{
    int32 Distance;   // graph hops from the anchor (default 1)
};
```

!!! warning "Graph Ring is not Hex Ring"
    They read alike but answer different questions. **Hex Ring** is geometric — every
    cell at a coordinate distance, walls ignored — and works with no grid. **Graph Ring**
    is a reachability-style hop count that respects connections and requires a built grid.
    Reach for Hex Ring (or Square Disk) for a targeting shape; reach for Graph Ring only
    when you specifically want the ring to stop at walls. For true movement range, use
    reachability instead — see the [Movement Reference](reference-movement.md).

### Blueprint escape hatch

When a shape needs logic the shipped structs do not cover, author it in Blueprint. The
pattern subsystem resolves it through the same paths, and the result is cached per asset.

```cpp
// A pattern whose logic lives in a Blueprint asset. Soft reference: the asset loads on
// first resolve, then the result is cached by asset path.
struct FBlueprintPatternData : FGridPatternData
{
    TSoftObjectPtr<UGridPatternObject> PatternAsset;
};

// Subclass this in Blueprint, implement the events, save it as a data asset, and point
// an FBlueprintPatternData at it. One asset = one configuration (it is not instanced);
// for per-instance parameters make several assets or wrap it in a composite.
class UGridPatternObject : public UObject   // Abstract, Blueprintable
{
    // Return the cells this pattern occupies relative to the origin; the subsystem
    // translates the result by the caller's anchor and caches the canonical shape.
    TArray<FIntVector> ResolveCoordsAtOrigin(const UGridTopology* Topology) const;

    // Optional. Restrict the pattern to specific topology classes (default: any).
    bool IsCompatibleWithTopology(TSubclassOf<UGridTopology> TopologyClass) const;
};
```

## Composing patterns

Patterns compose. `FCompositePatternData` is itself a pattern that folds a list of child
patterns together through set operations, letting you carve an exact footprint — a
rectangle minus a disk for a room with a round pit, two rings unioned for a double
border, and so on. Composites nest freely.

```cpp
// One step in a composite: a child pattern, how it folds in, and a shift applied to
// the child before it is combined.
struct FCompositePatternOp
{
    FInstancedStruct   Pattern;       // any FGridPatternData (including another composite)
    EGridPatternBoolOp Op;            // default Union
    FIntVector         LocalOffset;   // shift the child by this before combining (default zero)
};

// Fold children in order: the first entry SEEDS the accumulator, each subsequent entry
// combines into it. Compatible with a topology iff every non-empty child is.
struct FCompositePatternData : FGridPatternData
{
    TArray<FCompositePatternOp> Operations;
};

// How a child combines with the running set.
enum class EGridPatternBoolOp : uint8
{
    Union,      // add the child's cells to the set
    Subtract,   // remove the child's cells from the set
    Intersect,  // keep only cells the child also produced
    Xor,        // keep cells in exactly one of the two (symmetric difference)
};
```

!!! note "Order matters, and the first op is the seed"
    The first entry's `Op` is irrelevant — it seeds the accumulator. Every entry after
    it applies to the running result, so `Subtract` before you have unioned anything
    removes from an empty set. Author a composite as "start with this shape, then carve."

## Generating a grid

Generation is two calls on `UGridGraph`: propose cells from a pattern, then bake the
level into them.

```cpp
// 1. Clear the grid and rebuild its node set from a resolved pattern, anchored at Anchor.
//    An empty or invalid pattern yields an empty grid.
void GenerateGridFromPattern(const FInstancedStruct& Pattern, FIntVector Anchor);

// 2. Read the level and bake the result into those cells: set heights, drop cells over
//    holes, cut connections a wall or a too-tall step blocks, and (MaxLevels > 1)
//    discover stacked levels. Handles is the candidate cells to conform — typically
//    every node. Returns a summary of what happened.
FTerrainConformResult ConformToTerrain(const TArray<FGridNodeHandle>& Handles,
                                       const FTerrainConformSettings& Settings,
                                       UObject* WorldContextOverride = nullptr);
```

In the editor you rarely call these by hand. `UGridGraphComponent` exposes the whole bake
as a single **Generate Grid** button:

```cpp
// The edit-time authoring surface on UGridGraphComponent.
FInstancedStruct        SeedPattern;      // the shape that proposes the board's cells
FInstancedStruct        ExcludePattern;   // optional: cells carved back out (declarative holes)
bool                    bConformToTerrain;
FTerrainConformSettings ConformSettings;  // consulted only when bConformToTerrain

// The one button. Gathers (Seed - Exclude) plus any subgrid contributions, regenerates
// the node set, conforms it when bConformToTerrain, and serializes the result. Baking
// unchanged geometry is deterministic — identical handle indices — so hand-tweaks keyed
// to node handles survive a re-bake. (Runs in the editor.)
void GenerateGrid();
```

=== "Blueprint"
    On the grid component:

    1. Set **Seed Pattern** to a shape — e.g. **Square Rectangle** with min `(0,0,0)`
       and max `(9,9,0)` for a 10&times;10 board.
    2. Optionally set **Exclude Pattern** to punch declarative holes (water, a chasm)
       out of the seed region.
    3. Tick **Conform To Terrain** and fill in **Conform Settings** (below).
    4. Click **Generate Grid**. The bake proposes the cells, conforms them, and stores
       the result — it is not recomputed at play time.

=== "C++"
    ```cpp
    UGridGraph* Grid = GridComponent->Grid;

    // Propose a 10x10 board from a pure-shape pattern.
    FSquareRectanglePatternData Shape;
    Shape.MinCoord = FIntVector(0, 0, 0);
    Shape.MaxCoord = FIntVector(9, 9, 0);
    Grid->GenerateGridFromPattern(FInstancedStruct::Make(Shape), FIntVector::ZeroValue);

    // Bake the level into it.
    FTerrainConformSettings Settings;
    Settings.FloorChannel = ECC_WorldStatic;
    Settings.WallChannel  = ECC_WorldStatic;
    Settings.MaxLevels    = 3;   // discover up to three stacked levels
    Grid->ConformToTerrain(Grid->GetAllNodes(), Settings);
    ```

## Conforming to terrain

Conforming is the world-aware half of generation: it reads your level's geometry once
and bakes it into the grid. It is driven entirely by `FTerrainConformSettings`.

```cpp
struct FTerrainConformSettings
{
    // --- Floor discovery: which cells exist and how tall they stand ---
    ECollisionChannel FloorChannel;    // what standable ground blocks (default WorldStatic)
    float MaxSlopeAngle;               // steeper than this is not floor (degrees, default 50)
    float Clearance;                   // headroom a cell needs to be standable (cm, default 200)
    float TraceStartHeight;            // trace window: begin this far above the cell (cm, default 1000)
    float TraceDistance;               // ...and reach this far down (cm, default 5000)
    int32 MaxLevels;                   // 1 = single floor; >1 discovers stacked levels

    // --- Connection pass: which links survive between neighbouring cells ---
    bool  bConformEdges;               // run the gates below (default true)
    float MaxStepHeight;               // a taller rise/drop severs the link (cm, default 44)
    ECollisionChannel WallChannel;     // what a blocking wall sits on (default WorldStatic)
    float BlockerTraceHeight;          // wall trace height above the higher cell (cm, default 100)
    bool  bAllowDiagonalCornerCutting; // square grids only; off by default
    float MaxCornerStep;               // height spread a diagonal corner tolerates (cm, default 44)

    // The level band height. Read-only: resolved from the topology, never set by hand.
    float H;
};
```

For each candidate cell the conform step traces straight down in the **floor channel**
and sets the cell's height to the ground it finds. A cell whose column finds no walkable
surface is a **hole** and its cell is dropped. `MaxSlopeAngle` rejects surfaces too steep
to stand on; `Clearance` rejects a surface with no headroom above it (a crawlspace under a
slab is not a floor).

The connection pass then decides, for each pair of neighbouring cells, whether their link
survives. Two independent gates apply: the **step gate** severs a link whose height
difference exceeds `MaxStepHeight` (too tall to step — that gap belongs to a jump), and the
**wall gate** traces the **wall channel** at `BlockerTraceHeight` above the higher of the
two cells (knee-high cover passes below the trace and keeps the link; a full wall reaches
it and severs it).

`ConformToTerrain` returns an `FTerrainConformResult` summarising the pass — cells
conformed, holes pruned, extra levels created, connections kept and severed — useful for
diagnostics and tests.

!!! note "Extra floors are levels, joined only by ramps and abilities"
    With `MaxLevels > 1`, a single downward trace discovers every standable surface stacked
    in a column — a walkway over a chasm, a second storey — and each becomes its own
    **level** of the board. Stacked levels get **no free vertical link**: you reach an upper
    level only by a ramp (a run of cells that climbs) or by a movement ability like jumping,
    resolved per unit at query time. `MaxLevels == 1` is a single flat slice.

!!! warning "Conforming is an edit-time bake, not a runtime step"
    The bake runs when you click **Generate Grid** and its result — heights, connections,
    which cells exist — is serialized with the level. **`BeginPlay` never re-conforms.** At
    play time no query ever touches the level geometry again; the board is a self-contained
    value that hashes, saves, and rewinds identically. Do not think of conforming as a
    per-frame or load-time cost — it happens once, at author time.

!!! tip "Regions can discover differently"
    A mostly-flat map with one multi-storey building should not multi-level-trace
    everywhere (props and boulders would sprout spurious floors). A subgrid can carry a
    discovery override so its footprint discovers levels differently from the rest of the
    board, while the connection rule stays global so the boundary between the two wires
    seamlessly. See the [Graph Reference](reference-graph.md) for subgrids.

### Re-conforming at runtime

The bake is authoritative, but occasionally the world changes at runtime — a wall falls,
terrain deforms — and the serialized grid needs to catch up. Two entry points on
`UGridGraph` re-run the conform against the current geometry; both are the exception, not
the norm.

```cpp
// HANDLE-STABLE (the default). Refresh heights and connections over the existing cells.
// Never changes which cells exist and never touches the occupancy index — so anything
// keyed to node handles stays valid. Use when a wall fell but the floor plan is unchanged.
FTerrainConformResult ReconformHandleStable(const TArray<FGridNodeHandle>& Handles,
                                            const FTerrainConformSettings& Settings,
                                            UObject* WorldContextOverride = nullptr);

// FULL-REBUILD (structural). Regenerate from BaseCoords — cells may appear or vanish —
// then re-snap the occupancy index onto the surviving nodes. The result's
// OccupantsResnapped reports how many pieces were moved or dropped. Use when the set of
// standable cells itself changed.
FTerrainConformResult ReconformFullRebuild(const TSet<FIntVector>& BaseCoords,
                                           const FTerrainConformSettings& Settings,
                                           UObject* WorldContextOverride = nullptr);
```

Both refuse to run — mutating nothing, reporting `bRefusedConformGuard` — while a
conform-unsafe read of the graph is in progress, so a re-conform can never tear the board
out from under a query. The component also exposes a **Reconform** button that re-runs the
conform over the current cells after a level tweak has drifted the bake; a stale bake is
flagged by a validation warning on the component.

## Boundary outlines and curves

Two small pure libraries turn a set of cells or a path into geometry a presenter can draw.
Both are value functions — no actors, no spawning — so they are the shared intermediate the
[Visualization Reference](reference-visualization.md) renders on top of.

`UGridBoundaryLibrary` outlines a set of cells into closed loops:

```cpp
// Outline a cell set into its closed boundary loops, per floor. Each loop is the ordered
// world-space corners walked around it (the first corner is not repeated). A solid region
// yields one loop; a region with a hole yields two (outer + inner); platforms on different
// floors each yield their own. Pure function of (grid, cells) — feed it a reachable set to
// draw a movement-range outline.
static TArray<FGridBoundaryLoop> ExtractBoundaryLoops(
    const UGridGraph* Grid, const TSet<FGridNodeHandle>& ReachableNodes);
```

`UGridCurveLibrary` builds a framed, arc-length-parameterized centerline — the shared
"draw along a curve" intermediate that a path ribbon, an outline ribbon, or a mesh sweep
consumes. An open movement path and a closed outline loop both reduce to one
`FGridCurveData` (a list of stations, each carrying position, tangent, surface normal, and
cumulative distance):

```cpp
// Build a framed centerline from ordered positions. Up is the surface normal each frame
// anchors to (grid-up for flat ground); bClosed treats the points as a loop. Do not repeat
// the first point for a closed loop.
static FGridCurveData BuildCurveFromPositions(const TArray<FVector>& Positions,
                                              FVector Up, bool bClosed = false);

// Sweep a flat ribbon strip along a curve into a triangle strip. UVs are arc-length
// parameterized, so an animated stripe runs at a constant world-space size and speed
// regardless of the path's length or curvature.
static FGridRibbonMesh BuildFlatRibbonMesh(const FGridCurveData& Curve,
                                           float Width, float StripePeriod,
                                           float NormalOffset = 2.f);
```

The rendering side — the components that place these meshes in the world and drive their
materials — lives in the [Visualization Reference](reference-visualization.md). For the
overall plugin tour, start at the [GridGraph overview](index.md).
