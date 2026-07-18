# GridEntity API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Grids & Occupancy](../../concepts/grids-and-occupancy.md).

All public types carry the `PGx` prefix. Signatures below are hand-written to show
the shape of each API; they are illustrative, not source excerpts. Every write call
in this plugin is a thin wrapper over a [GameEntity command](../gameentity/reference-commands.md) —
GridEntity authors no commands of its own, so everything it changes undoes, saves,
and replays with the rest of game state.

## Placement

A unit's position is **placement**: tagged data on the entity, written through
commands like every other change to game state. `UPGxPlacementLibrary` is a Blueprint
function library — a set of static calls — that reads and writes it. Two facets exist,
and an entity carries at most one at a time:

- **`Data.Placement.Cell`** — an `FPGxGridPlacement` for a unit on a grid. The cell is
  authoritative; the world position is derived from it.
- **`Data.Placement.World`** — a free world transform for an off-grid piece.

When a grid argument is left null, these calls default to the world's main board —
the model to build against.

### Creating a placed unit

Both create calls seed the placement into the entity's *creation* command, so the
unit is born standing somewhere — there is never an instant where it exists but is
placed nowhere.

```cpp
// Born on a cell. Seeds Data.Placement.Cell into the create command — one atomic,
// undoable step. Grid defaults to the world's main board. Returns the new entity.
static FPGeEntityRef CreateEntityAtCell(
    const UObject* WorldContext, FName TemplateId,
    FGridNodeHandle Cell, FRotator Facing, UGridGraph* Grid = nullptr);

// Born off the grid, at a free world transform — the mirror of CreateEntityAtCell.
// Seeds Data.Placement.World instead. For dragged tokens, off-board reserves.
static FPGeEntityRef CreateEntityAtTransform(
    const UObject* WorldContext, FName TemplateId, const FTransform& Transform);
```

### Moving a placed unit

Each move call rewrites the placement to the new location. Moving onto a grid also
clears any stale off-grid facet in the *same* undo unit (and vice versa), so the
one-facet rule always holds and the occupancy index and token follow.

```cpp
// Reposition onto a cell: sets Data.Placement.Cell, preserving the unit's facing.
// A bare command when there is nothing to clear; either way it JOINS the caller's
// open transaction rather than forcing its own undo step. Grid defaults to the main board.
static void MoveEntityToCell(
    const UObject* WorldContext, const FPGeEntityRef& Ref,
    FGridNodeHandle Cell, UGridGraph* Grid = nullptr);

// Reposition to a free world transform — the off-grid mirror of MoveEntityToCell.
static void MoveEntityToTransform(
    const UObject* WorldContext, const FPGeEntityRef& Ref, const FTransform& Transform);
```

!!! warning "Move is bare — wrap it if it also spends a point"
    `MoveEntityToCell` repositions the unit and does nothing else. It does **not**
    spend a movement point — that is your game's decision. When a player move both
    repositions the unit and spends a point, wrap the two in a transaction so one
    Undo reverses the whole move. Because the move joins an open transaction, you get
    exactly one undo entry. See
    [Move units on the board](guides.md#move-units-on-the-board).

### Reading placement

```cpp
// The unit's current grid placement, off its Data.Placement.Cell tag. False if none.
static bool GetEntityCell(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FPGxGridPlacement& OutPlacement);

// The unit's resolved world transform, from whichever placement facet it carries.
// (For a cell placement, this resolves the cell against the grid.) False if unplaced.
static bool GetEntityTransform(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FTransform& OutTransform);

// A world-space path from a sequence of cells — each resolved against the grid.
// Feeds a Cue.Move payload so a token walks the route. Grid defaults to the main board.
static TArray<FVector> BuildPathFromCells(
    const UObject* WorldContext, const TArray<FGridNodeHandle>& Cells, UGridGraph* Grid = nullptr);
```

### The placement struct and tags

```cpp
// The grid-placement facet, stored under Data.Placement.Cell. The cell is the truth;
// the world position is derived from it against the grid, never stored as truth.
struct FPGxGridPlacement
{
    FName           GridId;   // empty = the world's main board
    FGridNodeHandle Cell;     // the authoritative cell
    FRotator        Facing;   // facing on the cell
};
```

The two tagged-data keys live in the `PGxPlacementTags` namespace:

| Tag | Value type | Meaning |
|---|---|---|
| `Data.Placement.Cell` | `FPGxGridPlacement` | A unit standing on a grid cell (the cell is authoritative). |
| `Data.Placement.World` | `FPTaTransform` | An off-grid piece at a free world transform. |

Because placement is ordinary [entity tagged data](../gameentity/reference-entities.md),
you never persist or capture it yourself — moving a unit is a command, so its position
undoes, saves, and replays like everything else.

## Occupancy

`UPGxOccupancySubsystem` is a `UWorldSubsystem` — one per world — holding the
**cell&nbsp;&rarr;&nbsp;entities index**: which entities sit on which cell. It answers
"who is on this cell?" and its reverse, "which cell is this unit on?".

The index is **derived state**, never command-captured. Its single source of truth is
each entity's command-routed `Data.Placement.Cell`, so it is rebuilt from change
events exactly like every other reactive consumer ([derived state](../../concepts/derived-state.md)
at board scale): it reconciles incrementally as units are created, moved, and
destroyed, and rebuilds wholesale when state is restored on load. It has nothing of
its own to save, and it is correct across undo and redo for free — it simply agrees
with whatever the entities now say.

```cpp
// Resolve the subsystem for the current world (context defaults to self in Blueprint).
static UPGxOccupancySubsystem* Get(const UObject* WorldContextObject);

// Every entity on Cell of Grid (null = the world's main board). Multi-occupant.
TArray<FPGeEntityRef> GetEntitiesOnCell(FGridNodeHandle Cell, UGridGraph* Grid = nullptr) const;

// The grid placement a given entity currently carries, if any. False if unplaced.
bool GetCellOfEntity(const FPGeEntityRef& Ref, FPGxGridPlacement& Out) const;
```

!!! note "The order guarantee"
    A cell's occupants come back sorted by **ascending entity id** — i.e. creation
    order. That order is stable across undo and redo (it keys on identity, not on when
    a unit stepped onto the cell) and is **identical between a live index and one
    rebuilt from a restored save**. So a query gives the same list in live play, after
    an undo, and in a replay — you can rely on it directly, with no defensive sort of
    your own.

!!! tip "Occupancy reports; your game adjudicates"
    The index only ever *reports* who is on a cell. It never decides whether a move is
    legal. Capacity, blocking, and stacking are your game's rules, written on top of
    what it reports — as queries over occupancy ("is this cell full?") and, to close
    the loop with movement, as [cost modifiers](#movement) that forbid a step ("can't
    enter a cell an enemy holds").

## Movement

`UPGxMovementLibrary` is a Blueprint function library that turns a *live entity* into
a movement query. GridGraph's engine runs on an abstract mover profile and a
[read-only board view](../gridgraph/reference-movement.md#the-movement-snapshot-and-board-view);
this library reads a unit's granted capabilities, snapshots its stats, builds the
board view for you, and drives that engine — so from an entity you get its reachable
set and its paths, respecting the live board.

Like placement, it authors no commands of its own: granting and revoking a capability
ride the existing [set/remove tagged-data commands](../gameentity/reference-commands.md#tagged-data),
so capability changes are undo/redo/replay-correct.

### Movement capabilities are granted tagged data

A unit's movement is assembled from two kinds of granted capability, each stored as
tagged data under a `Data.Movement.*` key — one grant per key:

- **Movement sources** (`Data.Movement.Source.*`) — the ways a unit can step. The
  ordinary walk follows the grid's connections; a cliff-jump invents a hop down a
  ledge; there are leaping and flying sources too. A unit's candidate steps are the
  union of its sources.
- **Cost modifiers** (`Data.Movement.Modifier.*`) — what a step costs, or what is
  forbidden. Each reprices or blocks the steps a source proposed. The engine sorts
  them into a fixed order at the start of every query, so grant order never changes
  the result.

Each grant records the entity that granted it (an item, a buff), which is what lets
you revoke exactly one item's contribution.

### Grant and revoke

```cpp
// Grant a movement source under a Data.Movement.Source.* key (one grant per key;
// overwrites any existing grant on that key). Source must be a movement-source struct.
// GrantSource records the item/buff/ability that granted it, for revoke-by-source.
// Rides one set-tagged-data command — undoable.
static void GrantMovementSource(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGameplayTag GrantTag,
    const FInstancedStruct& Source, FPGeEntityRef GrantSource);

// Grant a cost modifier under a Data.Movement.Modifier.* key (one grant per key).
// Modifier must be a cost-modifier struct. Rides one command — undoable.
static void GrantMovementModifier(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGameplayTag GrantTag,
    const FInstancedStruct& Modifier, FPGeEntityRef GrantSource);

// Grant the shipped default relational rule: forbid stepping onto a cell held by a
// unit of a faction OTHER than this one's. FactionRoot is the tag namespace whose
// children name factions. A convenience over GrantMovementModifier with the framework's
// faction-block rule; a game wanting a different relationship grants its own modifier.
static void GrantDefaultFactionBlock(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGameplayTag FactionRoot,
    FPGeEntityRef GrantSource, int32 Priority = 0);

// Revoke a single grant by its tag key. Rides one remove-tagged-data command.
// False if the unit carried no grant on that key.
static bool RevokeMovementGrant(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGameplayTag GrantTag);

// Revoke exactly the grants contributed by GrantSource, leaving every other grant
// intact (mirrors "remove modifiers by source" on stats). The removals run inside one
// transaction — one undo unit. Returns the number of grants removed.
static int32 RevokeMovementGrantsBySource(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FPGeEntityRef GrantSource);
```

!!! tip "Revoke-by-source is the unequip primitive"
    Because every grant carries the entity that granted it, unequipping an item that
    granted several capabilities is a single `RevokeMovementGrantsBySource(Item)` —
    one undo unit that removes exactly what that item added and nothing else.

### Computing reachability and paths

The ergonomic queries do everything for you: assemble the unit's profile, snapshot its
stats, build the board view over live occupancy, and call the engine. Costs are
**scaled whole integers** — one straight step is `10`, a diagonal `14`, so a "move 5"
unit searches on a `MaxCost` of `50`. A negative `MaxCost` means unbounded.

```cpp
// Flood the reachable set from the unit's CURRENT placement cell. The everyday entry —
// call it on selection. Empty if the unit is unplaced on this grid.
static FGridReachability ComputeEntityReachability(
    const UObject* WorldContext, const FPGeEntityRef& Ref, UGridGraph* Grid, int32 MaxCost = -1);

// Flood the reachable set from an explicit start cell (rather than the unit's own).
static FGridReachability ComputeEntityReachabilityFromCell(
    const UObject* WorldContext, const FPGeEntityRef& Ref, UGridGraph* Grid,
    FGridNodeHandle StartCell, int32 MaxCost = -1);

// The unit's cheapest path between two cells. Empty if none exists; OutPathCost is the
// total cost (scaled integer).
static TArray<FGridNodeHandle> FindEntityMovementPath(
    const UObject* WorldContext, const FPGeEntityRef& Ref, UGridGraph* Grid,
    FGridNodeHandle StartCell, FGridNodeHandle EndCell, int32& OutPathCost);
```

Read an `FGridReachability` — the reachable cells, whether a cell is reachable, the
cost to reach it, and the path back to the source — through the
[GridGraph result helpers](../gridgraph/reference-movement.md#reachability). The result
is a pure function of the unit's snapshot and the live board, so it is identical on
every machine and through undo, save, and replay; there is no cache to invalidate when
a capability changes — you just recompute.

### Assembling the profile and snapshot directly

For advanced use, the two building blocks the ergonomic queries compose are public.
Both are **derived and transient** — built on demand from current state, never
command-captured.

```cpp
// Assemble the unit's movement profile from its current Data.Movement.* grants:
// union the sources, collect the cost modifiers.
static FGridMovementProfile AssembleMovementProfile(
    const UObject* WorldContext, const FPGeEntityRef& Ref);

// Snapshot the unit into an immutable mover context: a copy of its gameplay tags and
// its whole current-stats map, plus a reference back to the entity for entity-aware
// modifiers. Read downstream by keyed lookup only.
static FGridTraversalContext SnapshotTraversalContext(
    const UObject* WorldContext, const FPGeEntityRef& Ref);
```

`FGridMovementProfile` and `FGridTraversalContext` are GridGraph types, documented in
[Movement & Reachability](../gridgraph/reference-movement.md#the-movement-profile).

### The grant structs and tags

```cpp
// A movement-source grant, stored under a Data.Movement.Source.* key.
struct FPGxMovementSourceGrant
{
    TInstancedStruct<FGridSuccessorSource> Source;       // how the unit can move
    FPGeEntityRef                          GrantSource;  // who granted it (may be invalid)
};

// A cost-modifier grant, stored under a Data.Movement.Modifier.* key.
struct FPGxMovementModifierGrant
{
    TInstancedStruct<FGridCostModifier> Modifier;     // what movement costs / is forbidden
    FPGeEntityRef                       GrantSource;  // who granted it (may be invalid)
};
```

The grant keys live in the `PGxMovementTags` namespace. `Data.Movement` is the parent
of every grant key (used to enumerate a unit's grants). The shipped leaves:

| Tag | Kind | Meaning |
|---|---|---|
| `Data.Movement.Source.AdjacencyWalk` | Source | The trivial walk along the grid's connections. |
| `Data.Movement.Source.CliffJump` | Source | Hop down (or up) a ledge, its reach driven by a jump stat. |
| `Data.Movement.Source.Jetpack` / `.Leap` / `.Flight` | Source | The conventional keys for the other shipped sources. |
| `Data.Movement.Modifier.UniformCost` | Modifier | Flatten every step to one uniform cost. |
| `Data.Movement.Modifier.OrthoDiagonal` | Modifier | The straight-`10` / diagonal-`14` cost rule. |
| `Data.Movement.Modifier.TagBased` | Modifier | Reprice steps by the tags on a cell. |
| `Data.Movement.Modifier.FactionBlock` | Modifier | The default "can't enter another faction's cell" gate (granted by `GrantDefaultFactionBlock`). |
| `Data.Movement.Modifier.EdgeAttunementGate` / `.ConditionalEdge` | Modifier | The shipped gates for special edges (teleporters) and conditional links (a drawbridge). |

!!! note "Grants route by payload type, not by a fixed list"
    A game adds its own source and modifier leaves freely. Profile assembly routes a
    grant by the *type* of its payload struct — any movement-source subtype is unioned
    into the sources, any cost-modifier subtype is collected into the modifiers — so a
    new capability needs no new grant code, just a new `Data.Movement.*` leaf and a
    payload struct. The full catalogue of shipped source and modifier structs is in
    the [GridGraph movement reference](../gridgraph/reference-movement.md#the-movement-profile).

## The board view

`FPGxEntityBoardView` is GridEntity's implementation of GridGraph's
[`IGridBoardView`](../gridgraph/reference-movement.md#the-movement-snapshot-and-board-view) —
the **read-only view of the live board** a movement search reads through. GridGraph's
engine asks a board view only "what is true of this cell / this connection right now",
and this is the implementation that answers from the entity and grid systems:

- **cell and connection tags** come straight from the static grid;
- **occupant facts** — whether a cell is occupied, and the merged tags of its
  occupants — come from the [occupancy index](#occupancy) combined with each occupant's
  live gameplay tags.

Because it reads whatever game state is currently active, a search run through it
automatically respects the current occupancy — a unit's reach stops at a cell an enemy
holds — with nothing extra to wire, and it stays correct through undo. It is a plain,
non-`UObject` value object, valid only for the synchronous duration of one search, so
it is **C++-only** (a Blueprint search reaches it indirectly through
[`ComputeEntityReachability`](#computing-reachability-and-paths), which builds one for
you).

```cpp
// Construct over the live systems; pass to UGridGraph::ComputeMovementReachability.
// Occupancy and game state may be null (a board with no entities) — then a cell simply
// never has occupants, and cell/edge tag reads still work off the grid.
FPGxEntityBoardView(
    const UGridGraph* Grid,
    const UPGxOccupancySubsystem* Occupancy,
    const UPGeGameStateSubsystem* GameState);
```

Occupant facts are exposed as a merged, any-match tag container — a cost modifier gates
with "does any occupant have this tag" rather than iterating occupants — which keeps a
search order-independent and therefore replay-stable.

## The grid token

`APGxGridTokenActor` is a **grid-aware presentation token**: a
[token actor](../tokensystem/reference.md#the-token-interface-and-lifecycle) taught to
stand on its entity's cell and walk a move. It is `Blueprintable` and `Abstract` —
you derive `BP_GridToken` from it, give it a look, and assign it as the unit's token
class (under `Data.Presentation.TokenClass`, like any token).

It contributes two things over a plain token actor:

- **It stands on the right cell with no positioning code.** Its world position is
  derived from the unit's placement, through the resolver the token registry uses when
  it spawns and re-snaps a token — so a born-placed unit lands on its cell with no
  origin-pop, and a commanded move re-snaps it to the new cell. This is automatic and
  requires no cue.
- **It walks a move path.** It implements the [movable capability](../tokensystem/reference.md#capabilities),
  so a played `Cue.Move` carrying a path (built with
  [`BuildPathFromCells`](#reading-placement)) walks it along the route instead of
  teleporting. The reverse change event from an undo re-snaps it through the same
  resolver.

```cpp
// Grid-aware token base. Derive BP_GridToken from this, give it a mesh, assign it as
// the entity's token class. The registry spawns and positions it automatically.
class APGxGridTokenActor : public APTkTokenActor, public IPTkMovable
{
    // Travel speed (cm/s) when walking a Cue.Move path.
    float MoveSpeed = 600.f;

    // The movable capability: walk the world-space path, completing the cue at the end.
    // Override MoveAlongPath as a Blueprint event (calling Parent) to customize the walk.
    void MoveAlongPath(const TArray<FVector>& Worldspace, UPTkCueContext* Ctx);
};
```

!!! tip "Snapping is free; the walk is opt-in"
    With only the token class assigned, the mini is always on the correct cell and
    re-snaps on every move, undo, and load — you write no positioning code. Playing a
    `Cue.Move` is the opt-in animation that makes a move *walk* rather than snap. See
    [Show a unit with a grid token](guides.md#show-a-unit-with-a-grid-token) and the
    presentation model in [Tokens & Cues](../../concepts/tokens-and-cues.md).

## Selection visualization

`UPGxSelectionVisualizationSubsystem` is a `UWorldSubsystem` and an **optional
convenience**: it is the one place a selected unit meets a grid render surface. It maps
a selected entity to its movement reachability and feeds the reachable tiles (and hover
paths) to a GridGraph
[visualization driver](../gridgraph/reference-visualization.md). The driver itself
knows nothing about entities — so nothing drawable depends on GridEntity, and a
non-entity game can drive the surface directly instead.

It is a reactive consumer: reachability is derived, never stored, so the highlight is
rebuilt whenever the selected unit moves, its capabilities or stats change, or any unit
steps onto or off a cell (occupancy shifted). Undo of a move updates the highlight for
free.

```cpp
// Resolve for the current world.
static UPGxSelectionVisualizationSubsystem* Get(const UObject* WorldContextObject);

// Bind the render surface this feeder drives (adopts the driver's grid). Refreshes now.
void SetDriver(UGridVisualizationDriverComponent* Driver);

// Override the grid reachability floods on (null falls back to the driver's grid).
void SetGrid(UGridGraph* Grid);

// The movement-cost budget for the flood (scaled integer; < 0 = unbounded).
void SetMaxMoveCost(int32 MaxMoveCost);

// Select a unit and show its reachable tiles (floods from its placement cell).
// An invalid ref clears the highlight — the same as ClearSelection.
void SetSelectedEntity(FPGeEntityRef Ref);

// Deselect: clear the highlight and forget the selection.
void ClearSelection();

// Feed a hovered cell to the path channel. The driver reconstructs the path over the
// flood the area was shown from — no reachability recompute. Invalid/unreachable clears it.
void SetHoveredCell(FGridNodeHandle Hovered);

// The currently selected unit (invalid when nothing is selected).
FPGeEntityRef GetSelectedEntity() const;

// One-call conveniences: bind the driver and select / hover in a single call.
void SelectEntityOnDriver(FPGeEntityRef Ref, UGridVisualizationDriverComponent* Driver);
void HoverCellOnDriver(FGridNodeHandle Hovered, UGridVisualizationDriverComponent* Driver);
```

A game's input layer calls `SetDriver` once, then `SetSelectedEntity` / `ClearSelection`
on selection change and `SetHoveredCell` as the cursor moves. For the render surface
itself — how tiles and paths are drawn — see the
[GridGraph visualization reference](../gridgraph/reference-visualization.md).
