# Movement & Reachability

For developers wiring up how units move across a board and where they can reach.
This page documents the GridGraph movement engine: the **profile** a unit's
movement is assembled from, the **reachability** query that runs over it, and the
two things that query reads — a snapshot of the mover and a read-only view of the
live board. For the model behind it, read
[Grids and Occupancy](../../concepts/grids-and-occupancy.md); for the board itself
see [Graph & topology](reference-graph.md), and for shape queries and generation
see [Generation](reference-generation.md).

Unlike most framework plugins, GridGraph's public types are **unprefixed** —
`FGridMovementProfile`, `IGridBoardView`, `UGridGraph`, and so on. Signatures below
are hand-written to show the shape of each API; they are illustrative, not source
excerpts.

!!! note "This page is the machinery; capability grants live in GridEntity"
    Nothing on this page knows what a game entity is — GridGraph reads only the
    generic mover snapshot and board view described here. The entity-facing helpers
    that **grant and revoke** a unit's movement capabilities and **run reachability
    for a game entity** — assembling the profile from the entity's data and building
    the board view for you — live in the **GridEntity** plugin and are documented in
    its own section. Reach for those from game code; reach for the types on this page
    when you want the underlying primitives, are driving a non-entity game, or are
    authoring a new movement source or cost modifier.

## The movement profile

A unit's movement is not a property of the map — it is a **movement profile**:
`FGridMovementProfile`, two composed lists that answer "how can this unit move?" and
"what does each move cost, and what is forbidden right now?" The lists are built from
data the same way an evaluator formula is built from steps rather than written as
code — each entry is a polymorphic instanced struct, so a profile is a value you can
assemble, serialize, and hand to a query.

```cpp
struct FGridMovementProfile
{
    // How the unit can move. A unit's candidate steps are the UNION of these.
    TArray<TInstancedStruct<FGridSuccessorSource>> SuccessorSources;

    // What movement costs / what is forbidden now. An ORDERED list, applied to every
    // proposed step. May be empty (then every step keeps its base cost).
    TArray<TInstancedStruct<FGridCostModifier>> CostModifiers;

    // The trivial profile: one ordinary walk, no cost modifiers.
    static FGridMovementProfile MakeTrivialWalk(int32 BaseCost = 10);
};
```

The profile is derived and transient — assembled on demand, never recorded on the
command stack. A non-entity game builds one by hand and hands it straight to a query;
in a game built on entities, GridEntity assembles it from the unit's granted movement
data. Gaining or losing a capability is adding or removing one entry.

### Movement sources — how a unit can move

A **movement source** proposes candidate steps out of a cell. Each source appends
zero or more steps; a unit's candidate steps are the **union** of all its sources.

```cpp
// One proposed step: a destination cell and the step's base cost, before modifiers.
struct FGridSuccessorEdge
{
    FGridNodeHandle To;
    int32 BaseCost = 10;   // scaled integer — one whole step
};

// Abstract base. Concrete sources are held as TInstancedStruct<FGridSuccessorSource>.
struct FGridSuccessorSource
{
    // Append candidate steps out of Node. Given the node being expanded, a snapshot
    // of the mover, a read-only view of the live board, and the graph (for its
    // connections and coordinate math). Append only — never clear OutSteps.
    virtual void GetSuccessors(
        FGridNodeHandle Node,
        const FGridTraversalContext& Mover,
        const IGridBoardView& Board,
        const UGridGraph& Graph,
        TArray<FGridSuccessorEdge>& OutSteps) const;

    virtual FString GetDebugLabel() const;
};
```

The shipped sources:

| Source | Proposes |
|---|---|
| `FGridSuccessorSource_AdjacencyWalk` | The ordinary walk: one step along each of the cell's connections. The default source, and what `MakeTrivialWalk` uses. |
| `FGridSuccessorSource_CliffJump` | A hop down (or up) a ledge to an existing cell on another floor, its vertical reach driven by a jump stat. |
| `FGridSuccessorSource_Jetpack` | Clears a horizontal **gap** to an existing cell, priced by the distance crossed (the intervening cells are never consulted). |
| `FGridSuccessorSource_Leap` | A multi-tile leap whose reach is an authored offset footprint, optionally clamped by a range stat. |
| `FGridSuccessorSource_Flight` | Reaches every existing cell within a range, ignoring walls and gaps — but lands only on cells that already exist (no air cells). |

The walk follows connections that already exist. The other four **synthesize** steps
from coordinates — a landing that the connections don't provide — which is why they
read a stat off the mover to decide how far they reach. `FGridSuccessorSource_CliffJump`
is representative:

```cpp
// A synthesized source: a hop down (or up) a ledge. Grantable as "boots of leaping".
struct FGridSuccessorSource_CliffJump : public FGridSuccessorSource
{
    // Which stat on the mover supplies the jump height (max floor levels it may drop
    // or rise). Leave UNSET to read the GridGraph default (Movement.Stat.JumpHeight);
    // set it to map the capability onto your game's own stat tag.
    FGameplayTag JumpHeightStat;

    // Height used when that stat is absent from the snapshot. 0 = a source whose stat
    // is missing proposes nothing (a safe default: no accidental free leaping).
    float DefaultJumpHeight = 0.f;

    int32 MaxHorizontalRange  = 1;      // how many cells out a landing may sit
    bool  bAllowDiagonalLeaps = false;  // include diagonal columns in the footprint
    FGridOffsetPattern Footprint;       // optional explicit footprint (overrides the range)

    int32 BaseCost          = 10;   // flat cost of a jump edge
    int32 HeightCostPerLevel = 10;  // added per level of drop/rise (effort grows with height)
};
```

```cpp
// A cliff-jump capability whose reach comes from the mover's own jump stat.
FGridSuccessorSource_CliffJump Jump;
Jump.DefaultJumpHeight = 2;   // may drop or rise up to two floor levels

FGridMovementProfile Profile = FGridMovementProfile::MakeTrivialWalk();
Profile.SuccessorSources.Add(TInstancedStruct<FGridSuccessorSource>::Make(Jump));
```

`Jetpack`, `Leap`, and `Flight` follow the same pattern — a capability stat read from
the snapshot, a footprint or range, and per-step cost terms — and each has its own
default stat key (see [Movement tags](#movement-tags)).

### Cost modifiers — what movement costs, what is forbidden

A **cost modifier** is applied to **every** proposed step, whichever source produced
it. It sees the running cost and returns a new cost, and it can **block** the step
outright. The modifiers run as an ordered list — each sees the cost the one before it
left — so an added surcharge and a multiplier compose predictably.

```cpp
// The outcome of one modifier over one step.
struct FGridCostModifierResult
{
    int32 Cost = 0;
    bool  bBlocked = false;   // blocking is TERMINAL — a veto, not a negotiation
};

// Abstract base. Concrete modifiers are held as TInstancedStruct<FGridCostModifier>.
struct FGridCostModifier
{
    // Ordering key: modifiers run by ascending Priority, ties broken deterministically
    // by type — never by insertion order.
    int32 Priority = 0;

    // Reprice or forbid one step. Sees the running cost; returns { new cost, blocked }.
    virtual FGridCostModifierResult ModifyCost(
        const FGridSuccessorEdge& Step,
        FGridNodeHandle From,
        int32 RunningCost,
        const FGridTraversalContext& Mover,
        const IGridBoardView& Board,
        const UGridGraph& Graph) const;

    virtual FString GetDebugLabel() const;
};
```

The shipped modifiers:

| Modifier | Effect |
|---|---|
| `FGridCostModifier_UniformCost` | Adds a flat surcharge to every step — a movement tax on top of the source's base cost. Never blocks. |
| `FGridCostModifier_OrthoDiagonal` | Multiplies a step by an orthogonal or diagonal factor from its coordinates. Defaults are the drift-free `10 / 14` pair, so a straight step stays `10` and a diagonal becomes `14`. Never blocks. |
| `FGridCostModifier_TagBased` | Adds a surcharge per matching tag on the destination cell or the traversed connection, and **blocks** the step if either carries a blocking tag. Reads tags live through the board view. |
| `FGridCostModifier_FactionBlock` | The default relational gate: blocks a step into a cell held by an occupant of a **different faction**. Occupant-aware but not entity-aware — it reads occupant tags through the board view. Its one knob is which tag namespace the factions live under; replace it with your own modifier for a different rule. |
| `FGridCostModifier_EdgeAttunementGate` | Blocks a special/teleporter connection unless the mover's snapshot carries the matching attunement tag. Put it in the default profile; capable units opt in by carrying the tag. |
| `FGridCostModifier_ConditionalEdge` | Blocks a conditional connection (a drawbridge) unless the **live board** reports it open. The connection is static; opening and closing it is a board-state change the board view reports, never a graph edit. |

```cpp
// A profile that walks, pays √2 for diagonals, and can't enter an enemy-held cell.
FGridMovementProfile Profile = FGridMovementProfile::MakeTrivialWalk();

FGridCostModifier_OrthoDiagonal Diag;   // 10 straight, 14 diagonal (the defaults)
Profile.CostModifiers.Add(TInstancedStruct<FGridCostModifier>::Make(Diag));

FGridCostModifier_FactionBlock Block;
Block.FactionRoot = TAG_Faction;        // children of this tag are the factions
Profile.CostModifiers.Add(TInstancedStruct<FGridCostModifier>::Make(Block));
```

!!! note "Blocking is terminal; cost stays positive"
    Once any modifier blocks a step, that step is dropped and the rest of the list
    does not run. The engine also keeps the running cost positive after every
    modifier, so a mis-authored reprice can never drive a step's cost to zero or
    below and break the search.

## Reachability

**Reachability** is a cost-aware flood: from a start cell, over a movement profile,
which cells can this unit reach and for how much. It is a **pure function** of
`(graph, mover snapshot, board view)` — the same inputs return the same answer on
every machine and through undo, save, and replay. Its result is `FGridReachability`,
which doubles as a flow field back toward the start.

```cpp
struct FGridReachability
{
    FGridNodeHandle Source;
    // Also holds, keyed by cell handle: the cheapest cost to reach each cell, and each
    // reached cell's predecessor on the way back to Source. Because those maps are keyed
    // by handle they are not Blueprint pins — read them through the helpers below.
};
```

Costs are **scaled whole integers**, never floats — that is part of why a replay
cannot drift. One straight step is `10`, a diagonal `14`. A "move 5" unit searches on
a budget of `50`. A `MaxCost` below zero means unbounded (flood the whole reachable
region).

Run the query on the grid. The full per-unit form takes a profile, a mover snapshot,
and a board view, and is **C++-only** (the board view is a plain, non-UObject
interface):

```cpp
// Flood every cell reachable within MaxCost, under a profile and a mover snapshot,
// reading the live board through a read-only view. MaxCost < 0 = unbounded.
FGridReachability UGridGraph::ComputeMovementReachability(
    FGridNodeHandle Start,
    const FGridMovementProfile& Profile,
    const FGridTraversalContext& Mover,
    const IGridBoardView& Board,
    int32 MaxCost = -1) const;

// The single cheapest path between two cells under the same inputs; empty if
// unreachable. OutPathCost is the path's total accumulated cost.
TArray<FGridNodeHandle> UGridGraph::FindMovementPath(
    FGridNodeHandle Start,
    FGridNodeHandle End,
    const FGridMovementProfile& Profile,
    const FGridTraversalContext& Mover,
    const IGridBoardView& Board,
    int32& OutPathCost) const;
```

Read the result — from C++ or Blueprint — through the library helpers:

```cpp
// Every reached cell (sorted by index, so the list is stable).
TArray<FGridNodeHandle> UGridBlueprintLibrary::GetReachedNodes(const FGridReachability& Reach);

// Is this cell in the reachable set?
bool UGridBlueprintLibrary::IsReachable(const FGridReachability& Reach, FGridNodeHandle Node);

// Cheapest cost to reach a cell (scaled integer); false if it wasn't reached.
bool UGridBlueprintLibrary::GetCostToReach(const FGridReachability& Reach, FGridNodeHandle Node, int32& OutCost);

// Rebuild the cheapest path Source -> Target from the flow field; includes both
// endpoints, ordered Source-first. Empty if Target wasn't reached.
TArray<FGridNodeHandle> UGridBlueprintLibrary::ReconstructPath(const FGridReachability& Reach, FGridNodeHandle Target);
```

For the common case — a plain grid with no per-unit capabilities — the grid exposes
one-call **Blueprint-callable** wrappers that assemble the trivial walk profile, an
empty snapshot, and an empty board view for you:

```cpp
// Trivial-profile convenience (BlueprintCallable). Uniform one-step (10) cost walk along connections.
TArray<FGridNodeHandle> UGridGraph::FindPath(FGridNodeHandle Start, FGridNodeHandle End, int32& OutPathCost) const;
FGridReachability       UGridGraph::ComputeReachability(FGridNodeHandle Start, int32 MaxCost = -1) const;
TArray<FGridNodeHandle> UGridGraph::GetReachableNodesWithinCost(FGridNodeHandle Start, int32 MaxCost) const;
```

!!! tip "Running a capability query from Blueprint"
    The custom-profile engine calls above are C++-only because the board view is a
    non-UObject interface. From Blueprint you reach per-unit movement — a unit's
    granted sources and modifiers, against the live board — through GridEntity's
    entity-facing reachability helper, which assembles the profile and board view and
    calls the engine for you. It is documented in the GridEntity plugin's section.

!!! warning "Reachability is a movement query, not a shape query"
    Reachability respects connections, per-step cost, and the unit's own abilities, and
    it stops at a wall. That is the opposite of a **pattern** — a coordinate shape ("every
    cell within 2") that walls and doors cannot bend, which is what you want for a blast
    or an aura. Patterns are a shape query, documented in [Generation](reference-generation.md).
    Use a pattern for a blast and reachability for movement — never the reverse.

## The movement snapshot and board view

A search reads two different things, and keeps them stable for its one synchronous
run: the **mover** and the **world**.

The mover is read as a **snapshot of the unit's tags and stats** —
`FGridTraversalContext` — taken once at the start of the search and never re-read.
A unit whose stats change "mid-move" is simply a *new* query with a *new* snapshot,
not a mutation of a live one; this is what makes a search a pure value-function.

```cpp
struct FGridTraversalContext
{
    FGameplayTagContainer     Tags;    // a copy of the mover's gameplay tags
    TMap<FGameplayTag, float> Stats;   // a copy of the mover's current stats

    // Optional opaque payload for entity-aware rules that plain scalars can't express
    // (e.g. a struct wrapping an entity reference). Empty for non-entity games; the
    // generic sources and modifiers ignore it.
    FInstancedStruct Extension;

    float GetStat(FGameplayTag Tag, float Default = 0.f) const;   // keyed read
    bool  HasTag(FGameplayTag Tag) const;                          // keyed test
};
```

The world is read through `IGridBoardView` — a **read-only view of the live board**.
It answers only "what is true of this cell / this connection right now": the live
tags, and what occupies a cell. It is a plain C++ interface, not a UObject, so a test
can implement it as a stack struct and the query never pulls object lifetime into a
read-only facade. A game built on entities supplies a view backed by the live board
state, so movement automatically respects the current occupancy — including when a
preview or AI what-if runs the same search against a temporary copy of the game state.

```cpp
class IGridBoardView
{
    // Live tags on a cell.
    virtual FGameplayTagContainer GetCellTags(FGridNodeHandle Cell) const = 0;

    // Live tags on the directed connection From -> To.
    virtual FGameplayTagContainer GetEdgeTags(FGridNodeHandle From, FGridNodeHandle To) const = 0;

    // The merged tags of every occupant on a cell — an any-match container, so a
    // modifier gates with HasAny / HasTag rather than iterating occupants.
    virtual FGameplayTagContainer GetOccupantTags(FGridNodeHandle Cell) const = 0;

    // Does any occupant sit on this cell?
    virtual bool HasOccupant(FGridNodeHandle Cell) const = 0;
};

// The empty view: no tags, no occupants. The default for a game with no board state,
// and the stub the engine tests run against.
struct FGridEmptyBoardView : public IGridBoardView { /* every query returns empty */ };
```

!!! note "Read by key, never by iteration"
    Both the snapshot and the board view are read by keyed lookup — `GetStat`,
    `HasTag`, `HasAny` — and never iterated to drive control flow. Map and
    tag-container iteration order is not stable across runs or platforms, so iterating
    to decide a step would silently diverge a replay. The engine also sorts the cost
    modifiers once, at the start of each search, so the order is identical however a
    profile was assembled.

## Movement tags

`GridMovementTags` is a small namespace of native gameplay tags: the **default stat
keys** the shipped synthesized sources read from the mover snapshot when their own
stat-key field is left unset. A source overrides which stat it reads by setting its
own key field, so a game can map a capability onto its own stat vocabulary and never
touch these.

```cpp
namespace GridMovementTags
{
    TAG_Movement_Stat_JumpHeight;    // FGridSuccessorSource_CliffJump — vertical reach, in floor levels
    TAG_Movement_Stat_JetpackRange;  // FGridSuccessorSource_Jetpack   — vertical range of a gap-crossing hop
    TAG_Movement_Stat_LeapRange;     // FGridSuccessorSource_Leap       — optional horizontal clamp on the footprint
    TAG_Movement_Stat_FlightRange;   // FGridSuccessorSource_Flight     — horizontal radius
}
```

!!! note "These are stat keys, not grant keys"
    These tags name the stats a source *reads*. The `Data.Movement.Source.*` and
    `Data.Movement.Modifier.*` tags a game uses to *grant* a capability to an entity
    belong to the GridEntity plugin, not here — see its section for the grant and
    revoke surface.

---

Related: [Grids and Occupancy](../../concepts/grids-and-occupancy.md) ·
[Graph & topology](reference-graph.md) · [Generation](reference-generation.md) ·
[Visualization](reference-visualization.md) · [Guides](guides.md) ·
[GridGraph overview](index.md)
