# The Grid & Topologies

For developers who want the public surface of the board itself — the graph that
stores your cells and connections, the topology objects that give it a shape, and
the actors and subsystems that host one grid in the world. For the model behind it
all, read [Grids & Occupancy](../../concepts/grids-and-occupancy.md); for the
generation pipeline (patterns, conforming to terrain) see
[Generation](reference-generation.md); for movement and reachability see
[Movement](reference-movement.md); and for drawing a grid see
[Visualization](reference-visualization.md).

Public types here are named with a `Grid` word in the type name — `UGridGraph`,
`FGridNodeHandle`, `USquareTopology` — rather than a short code prefix. The
signatures below are hand-written to show the shape of each API; they are
illustrative, not source excerpts, and omit engine macros and reflection metadata.

## The grid graph

`UGridGraph` is the board as pure data: a `UObject` that holds the cells, the
connections between them, and the durable facts attached to each. It knows nothing
about the world or the screen — a topology object supplies the shape math, and a
[hosting component](#hosting-a-grid-in-the-world) plugs it into a level. Everything
it stores is serialized, so a baked board reloads verbatim.

What lives on the graph:

- **Cells** — the nodes, each identified by a [handle](#node-and-edge-handles).
- **Connections** — the directed links between cells (what `GetNeighbors` reports).
- **Per-cell height** — a small vertical offset that sits a cell on the actual
  ground under it.
- **Cell and edge tags** — gameplay tags on cells and on individual connections.
- **Edge cost overrides** — an optional per-connection step cost.
- A **transient occupancy index** of live world objects — explicitly *not* durable
  game state; see [its own section](#the-transient-occupancy-index).

```cpp
class UGridGraph : public UObject
{
    // The shape policy — square, hex, or freeform. See Topologies below.
    UGridTopology* Topology;

    int32 GetNodeCount() const;
    bool  IsValidNode(FGridNodeHandle Handle) const;
};
```

!!! note "The graph is a static stage — about passability"
    Units move by writing their own placement, never by touching the board (see
    [Grids & Occupancy](../../concepts/grids-and-occupancy.md)): a connection
    means "these cells are linked", never "this unit may pass", so nothing
    per-unit or per-moment is ever stored here. The *structure* itself can change
    during play — dig a tunnel, drop a bridge — and when it does it changes
    through [GridCommands](../gridcommands/index.md), which routes every edit
    through the command stack. What keeps a board hashing, saving, and rewinding
    identically is that every change to it is a command, not that changes are
    rare.

## Node and edge handles

A cell is referred to by an `FGridNodeHandle` — a thin wrapper over an integer
index. Handles are **local to one grid**: the same index in a different grid is a
different cell. A default-constructed handle is invalid.

```cpp
struct FGridNodeHandle
{
    int32 Index = INDEX_NONE;    // the cell's id within its grid

    bool IsValid() const;        // false for a default / cleared handle
    bool operator==(const FGridNodeHandle& Other) const;
    // hashable — usable as a TMap/TSet key
};
```

!!! note "Handles are never renumbered and never recycled"
    A cell's handle is allocated once and belongs to that cell for the life of the
    board. Removing a cell does not free its index for reuse, and nothing at
    runtime renumbers the board — so a handle held elsewhere (a unit's placement,
    a saved selection, an entry in the undo history) either still names the cell
    it always named, or names a cell that no longer exists. It can never quietly
    come to mean a *different* cell.

    That is what makes a stale reference **permanently detectable**: `IsValidNode`
    is a complete test, forever, and undoing the removal makes the very same
    handle live again. It is also why nothing here needs generation counters or
    handle versions. The one operation that *does* renumber is a full regenerate,
    which is [bake-time only](#building-and-editing-structure).

A connection is identified by an `FGridEdgeKey` — an ordered `(From, To)` pair. Edge
keys are **directional**: tagging or costing `(A, B)` does not touch `(B, A)`. Most
edit calls take a `bBothDirections` flag (default `true`) so you can treat a
connection as two-way without spelling out both directions.

```cpp
struct FGridEdgeKey
{
    FGridNodeHandle From;
    FGridNodeHandle To;
    // hashable and equality-comparable
};
```

## Building and editing structure

Most boards are built once, from a pattern, at edit time — that whole pipeline
(patterns, conforming to terrain, the **Generate Grid** button) lives in
[Generation](reference-generation.md). The calls below are the low-level
construction surface: use them to bake a board, or to author a freeform graph by
hand.

```cpp
// --- Regenerate the whole graph (BAKE TIME ONLY — see the warning below) ---

// Clear and rebuild from an explicit coordinate set. Each coord becomes a cell,
// wired to its already-placed coord-neighbors by the topology.
void GenerateGridFromCoords(const TSet<FIntVector>& Coords);

// Clear and rebuild from a pattern resolved at Anchor. (Patterns: see Generation.)
void GenerateGridFromPattern(const FInstancedStruct& Pattern, FIntVector Anchor);

// --- Add and remove individual cells ---

// Allocate a bare cell with a fresh index (no coordinate — for freeform graphs).
FGridNodeHandle AddNode();

// Return the cell at Coord, or create one there and wire it to its coord-neighbors.
// Invalid handle if the topology has no coordinates (freeform).
FGridNodeHandle GetOrCreateNodeAt(FIntVector Coord);

// Remove a cell and everything attached to it — its connections, tags, and height.
void RemoveNode(FGridNodeHandle Handle);

// --- Add and remove connections ---

// Link two cells (idempotent). bBothDirections adds the reverse link too.
void AddEdge(FGridNodeHandle From, FGridNodeHandle To, bool bBothDirections = true);

// Sever a connection and drop its tag / cost data.
void RemoveEdge(FGridNodeHandle From, FGridNodeHandle To, bool bBothDirections = true);

// Remove every cell, connection, and attached datum. (Bake time only.)
void ClearGrid();
```

All of these are Blueprint-callable.

!!! warning "These mutate the graph directly — they do not undo"
    Nothing here is routed through the command stack, so a call made during play
    does not undo, save, or replay, and raises no
    [structural-change notification](#the-structural-change-notification) for
    visualizers and occupancy to reconcile against. They are the *apply surface*
    the command family drives, not a gameplay API.

    **To change a board at runtime, submit a command instead.** Every edit above
    has a counterpart in the [GridCommands](../gridcommands/index.md) plugin —
    add/remove a cell, add/remove a connection, tags, heights, cost overrides —
    which applies it through the command stack so the change rewinds with
    everything else. See [its guides](../gridcommands/guides.md) for the call
    sequence.

!!! warning "The regenerate calls are bake time only"
    `GenerateGridFromCoords`, `GenerateGridFromPattern`, and `ClearGrid` clear the
    board and **renumber every handle**. That is fine for initial generation and
    an editor re-bake, and catastrophic on a live board: every handle held
    anywhere else — unit placements, saved selections, the undo history — silently
    starts meaning a different cell. Do not call them at runtime. The runtime
    equivalent of "re-read the level and update the board" is a
    [re-conform](reference-generation.md#re-conforming-at-runtime), which is a
    coordinate diff that keeps surviving cells' handles.

## The structural-change notification

When the board's structure changes through the command family, the graph
publishes a **structural delta** describing exactly what changed. This is the
signal anything that mirrors the board — a visualizer, the entity occupancy
index, a navigation cache — keeps itself current from.

```cpp
// One added or removed cell. For a removed cell the coordinate is the one it had;
// for an added cell it is read from the live board.
struct FGridStructuralNodeChange
{
    FGridNodeHandle Handle;
    FIntVector      Coord;
    bool            bHasCoord;
};

// Everything that changed inside one change scope.
struct FGridStructuralDelta
{
    TArray<FGridStructuralNodeChange> AddedNodes;    // cells that came into existence
    TArray<FGridStructuralNodeChange> RemovedNodes;  // cells that ceased to exist
    TArray<FGridEdgeKey>              AddedEdges;    // connections added
    TArray<FGridEdgeKey>              RemovedEdges;  // connections severed on their own
    TArray<FGridNodeHandle>           SurfaceChangedNodes;  // surviving cells whose tags or height changed
    TArray<FGridEdgeKey>              SurfaceChangedEdges;  // surviving connections whose tags or cost changed
};

// Subscribe from C++.
FOnGridStructureChanged UGridGraph::OnStructureChanged;   // void(const FGridStructuralDelta&)
```

`FGridStructuralDelta` and `FGridStructuralNodeChange` are Blueprint-readable
value types, so a Blueprint function can take one as a parameter — but the
delegate itself is native.

!!! note "C++ only — and Blueprint has a bridge"
    `OnStructureChanged` is a plain C++ multicast delegate, deliberately not
    Blueprint-assignable: the payload is a rich value struct and every consumer of
    it is native. Blueprint-side, the
    [grid component](#hosting-a-grid-in-the-world) forwards a structural change
    into its own **On Grid Rebuilt** event, so a Blueprint visualizer that already
    rebuilds on that event keeps working with no changes.

Four properties of the notification decide how you write a handler:

- **It fires on undo too.** The delta always describes *the change that just
  happened*, in whichever direction: undoing an add reports a **removed** cell,
  undoing a removal reports an **added** one. Derived state cannot tell the
  difference and must not need to.
- **A removed cell implies its connections.** Connections that vanished because
  their cell vanished are *not* listed in `RemovedEdges` — they are implied by
  `RemovedNodes`. Only connections severed on their own appear there.
- **Surface changes name the affected cell or connection, not the field.** You
  are told *what* changed, and re-read the current value.
- **Batching is forward-only.** See the warning below.

!!! warning "One notification forward, N back"
    A brush stroke of N commands wrapped in a
    [transaction scope](../gridcommands/reference.md#the-transaction-scope)
    raises **one** batched notification when it is submitted — but **N** when it
    is undone, and **N** when it is redone. The grouping is applied on the forward
    pass only.

    That is harmless to a handler written as an **idempotent reconciler** — given
    the delta, make your mirror agree with the board, and be safe to run again —
    which is how the shipped consumers are written. It will quietly break a
    handler that does per-notification bookkeeping: a version counter, one
    animation per notification, an append-only change log. Don't count
    notifications; reconcile from them.

This is a different signal from `OnGridRebuilt`, and mixing them up is the easy
mistake:

| Signal | Meaning | Fires for |
|---|---|---|
| `OnStructureChanged` | *These specific things changed* — reconcile incrementally. | Every command-routed structural edit, on both apply and undo, and the runtime full re-conform. |
| `OnGridRebuilt` | *Everything changed, resync from scratch.* | The editor bake, an editor rebuild, and the handle-stable re-conform. |

## Core queries

Read-only questions about the graph's structure. These are about the *connections*,
never about a particular unit — so they work identically on a freeform graph that
has no coordinates at all.

```cpp
// Every cell in the graph.
TArray<FGridNodeHandle> GetAllNodes() const;

// The cells directly connected to Handle (its neighbors).
TArray<FGridNodeHandle> GetNeighbors(FGridNodeHandle Handle) const;

// Cells within [MinDist, MaxDist] connection-hops of Center (a breadth-first flood).
TArray<FGridNodeHandle> GetNodesInRange(FGridNodeHandle Center, int32 MinDist, int32 MaxDist) const;

// Cells at exactly Distance hops from Center — the hollow ring.
TArray<FGridNodeHandle> GetRing(FGridNodeHandle Center, int32 Distance) const;

// Resolve a baked offset template around a center cell (see the note below).
TArray<FGridNodeHandle> ResolveOffsetPattern(FGridNodeHandle Center, const FGridOffsetPattern& Pattern) const;
```

`GetNodesInRange` and `GetRing` measure **connection hops** — they respect where the
graph is linked, but not step costs or a unit's abilities. For "how far can *this*
unit actually get," that is a movement query, covered in
[Movement](reference-movement.md).

!!! warning "`GetNeighbors` on the grid, `GetLatticeNeighbors` on the topology"
    Two different questions, and only one of them is about connections:

    - **`UGridGraph::GetNeighbors`** reads the graph's connection list. *"What can
      I actually reach from here?"* — the authoritative answer, and almost
      certainly the one you want: pathing, reachability, movement.
    - **`UGridTopology::GetLatticeNeighbors`** returns whatever cell happens to
      sit at an adjacent *coordinate*, computed from lattice offsets without
      consulting a single connection. *"What does the lattice look like here?"* —
      generation, outline extraction, terrain conforming.

    They agree on a freshly generated board and **diverge the moment connectivity
    is edited at runtime**: sever a connection between two cells that both still
    exist, and the grid call reports them as unconnected while the lattice call
    still reports them as neighbours. Since editing connectivity at runtime is
    exactly what [GridCommands](../gridcommands/index.md) makes routine, be
    deliberate about which one you call.

!!! note "`FGridOffsetPattern` is the lightweight one"
    `FGridOffsetPattern` is just a baked list of coordinate offsets relative to a
    center — a cheap value type for a fixed AoE stamp or a precomputed template.
    For composable, topology-aware shapes (rectangles, disks, rings, boolean
    combinations) use the pattern system in [Generation](reference-generation.md).

!!! tip "Uniform-cost convenience pathfinding"
    The graph also offers `FindPath`, `ComputeReachability`, and
    `GetReachableNodesWithinCost` — simple cost-aware searches over the bare
    connections, useful for an abstract board with no per-unit rules. Real
    per-unit movement (capabilities, cost modifiers, occupancy blocking) is a
    richer, separate query documented in [Movement](reference-movement.md).

## Coordinates, world position, and levels

Square and hex grids carry coordinates; freeform grids do not. Ask before you
assume — the coordinate calls return invalid or zero results on a topology that has
none.

```cpp
// True for square/hex; false for freeform.
bool SupportsCoordinates() const;

// The cell at a coordinate (invalid handle if none, or if the topology is coordinate-less).
FGridNodeHandle GetNodeAt(FIntVector Coord) const;

// The coordinate of a cell. Returns false and leaves OutCoord untouched for freeform.
bool TryGetCoord(FGridNodeHandle Handle, FIntVector& OutCoord) const;
```

Position calls turn cells and coordinates into world space. Local position is
relative to the hosting actor; world position folds in that actor's transform.

```cpp
// World-space center of a cell (its topology position plus its own height offset).
FVector GetWorldPosition(FGridNodeHandle Handle) const;

// Cell center in the host actor's local space.
FVector GetLocalPosition(FGridNodeHandle Handle) const;

// World-space center of an arbitrary coordinate — no cell needs to exist there.
FVector CoordToWorld(FIntVector Coord) const;

// The coordinate nearest a world-space point.
FIntVector WorldToCoord(FVector WorldLocation) const;
```

**Height and levels.** Each cell carries a small height offset — the vertical nudge
that sits it on the real ground under it. This is separate from *levels*: a
multi-storey board stacks whole extra levels of cells above the ground, spaced by
the topology's level height, connected only by ramps and movement abilities and
never by a free vertical link. Discovering those levels is a conform-time job (see
[Generation](reference-generation.md)); the per-cell offset is directly settable.

```cpp
// The cell's small within-level vertical offset (ground height under it).
void  SetNodeHeightOffset(FGridNodeHandle Node, float HeightOffset);
float GetNodeHeightOffset(FGridNodeHandle Node) const;
```

!!! note "Freeform has no coordinates"
    On a freeform graph, `SupportsCoordinates()` is false, `GetNodeAt` /
    `CoordToWorld` don't apply, and `TryGetCoord` returns false. Its cells sit at
    positions you set directly (see [Topologies](#topologies)) — but neighbor,
    range, and reachability queries all still work, because those ask about
    connections, not coordinates.

## Cell and edge tags, and edge costs

Cells and connections can carry gameplay tags — the durable "what kind of place /
crossing is this" facts a board is made of (a `Terrain.Water` cell, a
`Crossing.Door` edge). Cell tags are undirected; **edge tags are directional**, so
pass `bBothDirections` when you mean both ways.

```cpp
// --- Cell tags ---
void                  AddNodeTag(FGridNodeHandle Node, FGameplayTag Tag);
void                  RemoveNodeTag(FGridNodeHandle Node, FGameplayTag Tag);
bool                  NodeHasTag(FGridNodeHandle Node, FGameplayTag Tag) const;
FGameplayTagContainer GetNodeTags(FGridNodeHandle Node) const;

// --- Connection tags (directional) ---
void                  AddEdgeTag(FGridNodeHandle A, FGridNodeHandle B, FGameplayTag Tag, bool bBothDirections = true);
void                  RemoveEdgeTag(FGridNodeHandle A, FGridNodeHandle B, FGameplayTag Tag, bool bBothDirections = true);
bool                  EdgeHasTag(FGridNodeHandle From, FGridNodeHandle To, FGameplayTag Tag) const;
FGameplayTagContainer GetEdgeTags(FGridNodeHandle From, FGridNodeHandle To) const;
```

A connection can also carry an explicit step cost that overrides the default. Costs
are whole scaled integers — one straight step is `10`, a diagonal `14` — so a replay
never drifts on a floating-point tie. The override feeds the movement query's cost
pipeline; how a moving unit reprices steps on top of it is covered in
[Movement](reference-movement.md).

```cpp
// Cost is a scaled int (one whole step == 10). Directional unless bBothDirections.
void SetEdgeCostOverride(FGridNodeHandle From, FGridNodeHandle To, int32 Cost, bool bBothDirections = true);
void ClearEdgeCostOverride(FGridNodeHandle From, FGridNodeHandle To, bool bBothDirections = true);
bool GetEdgeCostOverride(FGridNodeHandle From, FGridNodeHandle To, int32& OutCost) const;
```

## Topologies

A **topology** owns the shape math the graph itself doesn't: coordinates, which
cells are neighbors, and where each cell sits in the world. The graph delegates
every coordinate and position question to its `Topology`. Three ship, and you can
swap between them without changing anything else about the board.

`UGridTopology` is the abstract base every topology implements:

```cpp
class UGridTopology : public UObject
{
    // Coordinate support (false for freeform).
    bool            SupportsCoordinates() const;
    FGridNodeHandle CoordToHandle(FIntVector Coord) const;
    bool            TryGetCoord(FGridNodeHandle Handle, FIntVector& OutCoord) const;
    FIntVector      ApplyOffset(FIntVector Coord, FIntVector Offset) const;

    // Lattice neighbors and distance. GetLatticeNeighbors returns whatever cells sit
    // at adjacent COORDINATES — pure shape math, never a connection. For "what can be
    // reached from here", call UGridGraph::GetNeighbors instead.
    TArray<FGridNodeHandle>           GetLatticeNeighbors(FGridNodeHandle Handle) const;
    TArray<FGridPlanarNeighborOffset> GetPlanarNeighborOffsets() const;
    bool                              ClassifyPlanarOffset(FIntVector Offset, EGridPlanarNeighborKind& OutKind) const;
    int32                             GetDistance(FGridNodeHandle A, FGridNodeHandle B) const;

    // World placement (local space, relative to the host actor).
    FVector          GetLocalPosition(FGridNodeHandle Handle, float HeightOffset = 0.f) const;
    FVector          CoordToLocal(FIntVector Coord) const;
    FIntVector       LocalToCoord(FVector Local) const;
    TArray<FVector>  GetCellCorners(FGridNodeHandle Handle, float HeightOffset = 0.f) const;
};
```

`GetPlanarNeighborOffsets` is the single source of truth for a topology's in-plane
neighbor directions — each tagged **cardinal** (shares a cell edge) or **diagonal**
(shares only a corner). Lattice-neighbor generation, outline extraction, and terrain
conforming all read it, so a topology never repeats its own direction list.

```cpp
enum class EGridPlanarNeighborKind : uint8 { Cardinal, Diagonal };

struct FGridPlanarNeighborOffset
{
    FIntVector              Offset;   // planar (Z == 0) coordinate delta to the neighbor
    EGridPlanarNeighborKind Kind;     // Cardinal (edge-adjacent) or Diagonal (corner-adjacent)
};
```

`ClassifyPlanarOffset` is the reverse lookup, resolved against that same table:
hand it a coordinate delta and it tells you whether that step is cardinal or
diagonal *for this lattice*. It returns false — classifying nothing — for a delta
that is not one of the lattice's own directions: a purely vertical step, a
multi-cell or off-lattice hop (a placed link, a cliff jump, a teleport), or any
coordinate-less topology.

!!! note "Cardinal versus diagonal is a property of the lattice"
    It is tempting to classify a step by counting how many coordinate axes it
    changes. That is right for a square and wrong for a hex, whose six sides are
    *all* cardinal even though two of them move on both axes and are exactly as
    far apart as the other four. Anything that prices or reasons about step
    direction should ask the topology, which is what the shipped
    [diagonal cost modifier](reference-movement.md#cost-modifiers-what-movement-costs-what-is-forbidden)
    does.

### Rotation and forward direction

A topology also owns the answer to "which way is this unit facing?" and "what does
its shape look like turned 90°?" — the questions a multi-cell unit with an
orientation asks. Rotation is expressed in whole **steps**, never in degrees, so it
is exact and replays identically.

```cpp
// How many discrete turns map the lattice onto itself: square 4, hex 6, freeform 1.
int32      GetRotationStepCount() const;
float      GetRotationStepAngleDegrees() const;   // 360 / step count

// Turn an anchor-relative offset by whole steps. Negative steps are fine; Z passes
// through untouched. Freeform's single rotation maps every offset to itself.
FIntVector RotateOffset(FIntVector Offset, int32 Steps) const;

// The direction a unit at rotation step 0 faces, and at any step.
FIntVector GetForwardOffset() const;
FIntVector GetStepDirection(int32 Steps) const;
```

Rotating a set of offsets by *k* steps gives the same result as yaw-rotating the
resolved world positions by *k* step-angles — the coordinate math and the visible
transform agree by construction, so a rotated footprint never drifts from the model
the player sees.

!!! note "Hex facing points at a neighbour, not at a corner"
    Square step 0 faces `+X`. Hex step 0 faces `+Q` — the eastern neighbour — so
    each of the six hex directions points squarely at an adjacent cell. If your game
    wants units facing hex *corners* instead, offset the cosmetic facing by half a
    step and leave the logical rotation alone.

A freeform board has one rotation step and a degenerate forward direction, so
orientation is meaningless there — the calls stay safe, they just do nothing.

Two grid-level helpers turn those offsets into actual cells:

```cpp
// Resolve an anchor + rotated offsets into cells. Offsets that land off the board
// are dropped silently; an empty offset list resolves to the anchor alone.
bool UGridGraph::ResolveRotatedOffsets(FGridNodeHandle Anchor,
    const TArray<FIntVector>& Offsets, int32 Steps,
    TArray<FGridNodeHandle>& OutHandles) const;

// Is this set of cells one connected body? Missing connections split it; edges
// carrying any of SeveringEdgeTags sever it too (a closed drawbridge).
bool UGridGraph::AreCellsMutuallyConnected(const TArray<FGridNodeHandle>& Cells,
    FGameplayTagContainer SeveringEdgeTags,
    bool bTreatSameColumnLayersAsConnected = true) const;
```

`ResolveRotatedOffsets` **fails soft**: it reports which cells exist and never
judges whether the result is a legal place to stand. Overhanging a board edge and
being split by a wall are two different questions — the second is
`AreCellsMutuallyConnected`, which is what stops a two-cell unit straddling a
chasm. Both are shared primitives: spawn validation, UI previews, and the movement
gates all call them, so your own checks agree with the framework's.

For the entity-facing layer built on all of this — a unit's stored rotation, its
footprint shape, and the front/flank/rear arcs — see
[Facing & Arcs](../gridentity/reference-facing.md) and
[Footprints](../gridentity/reference-footprints.md) in GridEntity.

### Square

`USquareTopology` — Cartesian coordinates, four-way by default and eight-way when
diagonals are allowed. It can wire whole stacked levels together vertically for a
multi-storey board. Four rotation steps; step 0 faces `+X`.

```cpp
class USquareTopology : public UGridTopology
{
    bool  bAllowDiagonals        = false;  // false = 4 neighbors, true = 8
    bool  bAllowVerticalAdjacency = true;  // link cells across stacked levels
    float TileSize    = 100.f;             // cell-to-cell spacing
    float LayerHeight = 300.f;             // world height of one level
    FVector GridOrigin = FVector::ZeroVector;  // local offset from the host (zeroed on the main grid)
};
```

### Hex

`UHexTopology` — six neighbors on **axial coordinates** (Q maps to X, R to Y). Same
tile-size / level-height / origin knobs as square, minus diagonals (a hex has none).
Six rotation steps; step 0 faces `+Q`, the eastern neighbour.

```cpp
class UHexTopology : public UGridTopology
{
    bool  bAllowVerticalAdjacency = true;
    float TileSize    = 100.f;   // center-to-center distance between adjacent hexes
    float LayerHeight = 300.f;
    FVector GridOrigin = FVector::ZeroVector;

    // Hex distance between two axial coordinates (ignoring level).
    static int32 HexDistance(FIntVector A, FIntVector B);
};
```

### Freeform

`UFreeformTopology` — **no coordinate system at all**. Cells sit at positions you
set by hand, and neighbors are exactly the connections you wire with `AddEdge`.
`SupportsCoordinates()` is false, and seed patterns don't apply — build a freeform
graph with the [construction API](#building-and-editing-structure) plus
`SetNodePosition`. It has a single rotation step, so orientation and multi-cell
footprints don't apply to a freeform board.

```cpp
class UFreeformTopology : public UGridTopology
{
    // Each cell's position in the host actor's local space.
    TMap<FGridNodeHandle, FVector> NodePositions;

    // Place a cell.
    void SetNodePosition(FGridNodeHandle Handle, FVector Position);
};
```

## Hosting a grid in the world

`UGridGraph` is world-agnostic. `UGridGraphComponent` is the actor component that
owns one and bridges it to a level — it holds the generation settings, drives the
bake, and fires Blueprint events. The component is where a designer configures a
board.

```cpp
class UGridGraphComponent : public UActorComponent
{
    UGridGraph*      Grid;           // the board data this component hosts
    FInstancedStruct SeedPattern;    // shape that seeds generation (see Generation)
    FInstancedStruct ExcludePattern; // coords carved back out (declarative holes)

    bool                    bConformToTerrain = false;  // conform the bake to level geometry
    FTerrainConformSettings ConformSettings;            // (details: Generation)

    bool bIsMainGrid        = false; // register as the world's main grid at BeginPlay
    bool bLockToWorldOrigin = true;  // keep coord*spacing == world (main grid)

    // Rebuild the board from the patterns and (optionally) conform it. This is the
    // authoritative build — the CallInEditor "Generate Grid" button.
    void GenerateGrid();

    // Re-run conform over the current cells without regenerating (after a mesh tweak).
    void Reconform();

    // Nearest cell to a world point (brute-force scan).
    FGridNodeHandle GetNearestNode(FVector WorldPosition) const;

    // Blueprint-assignable events.
    FOnGridNodeEvent OnNodeSelected;
    FOnGridNodeEvent OnNodeHovered;
    // Fires after a (re)build; visualizers subscribe. The component also forwards a
    // runtime structural change into it, so a Blueprint visualizer bound here keeps up
    // with a board edited during play (rebuilding wholesale each time).
    FOnGridRebuilt   OnGridRebuilt;
};
```

`GenerateGrid` and `Reconform` are the **Generate Grid** and **Reconform** buttons
in the editor; the mechanics of what they bake — patterns, terrain conforming,
handle-stable vs. full rebuild — are documented in
[Generation](reference-generation.md).

### The main grid

Most games have one board. Flag a component `bIsMainGrid` and it registers as the
world's designated grid, reachable from anywhere without a reference — this is the
model to build against. `AMainGridActor` is a ready-made actor with that flag set,
locked to the world origin so its `coordinate × spacing` maps cleanly to world
space.

`UMainGridSubsystem` tracks the one registered main grid per world:

```cpp
class UMainGridSubsystem : public UWorldSubsystem
{
    static UMainGridSubsystem* Get(const UObject* WorldContext);

    UGridGraphComponent* GetMainGridComponent() const;
    UGridGraph*          GetMainGrid() const;
    bool                 HasMainGrid() const;
};
```

`UMainGridBlueprintLibrary` wraps that subsystem in flat static helpers so Blueprint
(and C++) can reach the world's board and do the common coordinate/world conversions
against it directly. Each resolves the main grid from a world-context object and
warns if none is registered.

```cpp
static UGridGraph*          GetMainGrid(const UObject* WorldContext);
static UGridGraphComponent* GetMainGridComponent(const UObject* WorldContext);
static bool                 HasMainGrid(const UObject* WorldContext);

static FGridNodeHandle GetNodeAt(const UObject* WorldContext, FIntVector Coord);
static FGridNodeHandle GetOrCreateNodeAt(const UObject* WorldContext, FIntVector Coord);
static FVector         CoordToWorld(const UObject* WorldContext, FIntVector Coord);
static FIntVector      WorldToCoord(const UObject* WorldContext, FVector WorldLocation);
static FGridNodeHandle WorldToNearestNode(const UObject* WorldContext, FVector WorldLocation);
```

!!! note "One main grid per world"
    A second component that tries to register as the main grid is rejected with a
    warning — there is exactly one. If you need several independent boards, host
    each on its own `UGridGraphComponent` and hold references to them directly;
    only the designated one answers the main-grid helpers.

## Subgrids

A **subgrid** is a region you drop into the level that folds its own shape into the
main grid at generation time. `USubgridComponent` contributes its pattern — anchored
at the nearest main-grid cell to its world position — so you can compose a board out
of placed pieces instead of one monolithic pattern. `ASubgridActor` is a ready-made
host: drop it in, pick a pattern, and the main grid picks the region up on its next
bake.

```cpp
class USubgridComponent : public USceneComponent
{
    // Shape contributed to the main grid, anchored at this component's world position.
    FInstancedStruct      Pattern;
    bool                  bEnabled = true;
    FGameplayTagContainer NodeTags;   // stamped onto every cell this subgrid contributes
};
```

A subgrid can also override how *its footprint* conforms to terrain — a small
multi-storey building on an otherwise flat map, for instance, without forcing the
whole board to search for stacked floors. Those overrides
(`bOverrideConformDiscovery`, `ConformPriority`, and the discovery settings) are a
conform concern, documented in [Generation](reference-generation.md). A subgrid
whose pattern is incompatible with the main grid's topology is skipped with a
warning.

## The transient occupancy index

`UGridGraph` keeps a live index of *world objects* standing on cells — actors and
props you want to look up spatially. It is multi-occupant, weak (a destroyed object
simply drops out), and returns occupants in insertion order.

```cpp
void            AddOccupant(FGridNodeHandle Node, UObject* Occupant);
void            RemoveOccupant(UObject* Occupant);
void            MoveOccupant(UObject* Occupant, FGridNodeHandle To);
TArray<UObject*> GetOccupants(FGridNodeHandle Node) const;
FGridNodeHandle  GetOccupantNode(UObject* Occupant) const;
```

!!! danger "Not for game state that must survive undo"
    This index is a transient spatial convenience for live world objects — it is
    **not** command-routed and does **not** undo, save, or replay. Never store here
    anything the game's rules depend on. The undo-correct record of which *entity*
    stands on which cell is the entity occupancy index in the **GridEntity** plugin,
    documented in its own section — that one rebuilds itself from command-routed
    placement and is correct across undo, redo, and reload.

## The Blueprint library

`UGridBlueprintLibrary` collects small static helpers for working with handles and
reading reachability results from Blueprint (and C++).

```cpp
// --- Handles ---
static FGridNodeHandle MakeGridNodeHandle(int32 Index);
static bool            IsValidHandle(const FGridNodeHandle& Handle);
static int32           GetHandleIndex(const FGridNodeHandle& Handle);
static bool            EqualEqual_GridNodeHandle(const FGridNodeHandle& A, const FGridNodeHandle& B);
static bool            NotEqual_GridNodeHandle(const FGridNodeHandle& A, const FGridNodeHandle& B);

// --- Reading a reachability result (FGridReachability) ---
static TArray<FGridNodeHandle> GetReachedNodes(const FGridReachability& Reach);
static bool                    IsReachable(const FGridReachability& Reach, FGridNodeHandle Node);
static bool                    GetCostToReach(const FGridReachability& Reach, FGridNodeHandle Node, int32& OutCost);
static TArray<FGridNodeHandle> ReconstructPath(const FGridReachability& Reach, FGridNodeHandle Target);
```

The `FGridReachability` readers turn a movement flood into the answers a UI needs —
the reachable set, whether a cell is in it, its cheapest cost, and the path back to
the start. Producing that flood (per-unit reachability, movement profiles, cost
modifiers) is covered in [Movement](reference-movement.md).
