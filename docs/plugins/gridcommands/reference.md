# GridCommands API Reference

For developers who want the complete public surface of the plugin: the submit
library, the eleven structural commands, the transaction scope, the grid
registry, and the routing seam. For step-by-step recipes see the
[Guides](guides.md); for the plugin tour see the [overview](index.md); for the
board these commands edit see
[The Grid & Topologies](../gridgraph/reference-graph.md).

All public types carry the `PGc` prefix. Signatures below are hand-written to
show the shape of each API; they are illustrative, not source excerpts, and omit
engine macros and reflection metadata.

## Submitting an edit

`UPGcGridCommandLibrary` is the entry point for everything. Three of its four
functions are Blueprint-callable.

```cpp
// Submit one structural command. Resolves the stack the edit belongs on, applies
// the command, and retains it for undo. Returns false if no stack could be
// resolved, or if the command refused its own inputs (in which case nothing was
// mutated and nothing was pushed). The inner struct must derive from
// FPGcStructuralCommand.
static bool SubmitGridCommand(UObject* WorldContextObject, const FInstancedStruct& Command);

// Is BOARD AUTHORING — the re-conform calls below — legal right now on this board?
// True when the board resolves and nothing is in flight that a wholesale rebuild
// would corrupt. Bind a tool button's enabled state to this.
static bool IsBoardAuthoringAllowed(UObject* WorldContextObject, FName GridId);

// Command-routed re-conform: re-read the level's geometry over BaseCoords, diff the
// result against the live board, and apply the DIFFERENCE as one transaction of
// structural commands — one undo unit, surviving cells keeping their handles.
// Returns true when it ran (a no-op plan is success), false when refused.
static bool SubmitReconform(UObject* WorldContextObject, FName GridId,
                            const TSet<FIntVector>& BaseCoords,
                            const FTerrainConformSettings& Settings);

// Apply an already-computed reconform plan. C++ only — the plan is a plain value,
// not a reflected struct. Carries the same refusals as SubmitReconform.
static bool SubmitReconformPlan(UObject* WorldContextObject, FName GridId,
                                const FGridReconformPlan& Plan);
```

In Blueprint these are **Submit Grid Command**, **Is Board Authoring Allowed**,
and **Submit Reconform**, all under the *Grid | Commands* category, each with a
world-context pin that defaults to self.

Two behaviours worth having in mind:

- **`SubmitGridCommand` is not gated by the authoring check.** An ordinary
  structural edit — a dig, a collapsed wall — is legal whenever an entity command
  is legal. It is only ever *routed*, never refused for being ill-timed. The
  authoring check exists for the re-conform, which rebuilds wholesale.
- **Refusal leaves no trace.** A refused `SubmitReconform` mutates nothing and
  pushes no undo entry, so a mistimed call is a no-op rather than an empty step
  in the player's undo history.

!!! note "There is no `IsStructuralMutationAllowed`"
    Asking "is a structural mutation allowed?" no longer has an answer a boolean
    can carry, because an ordinary structural mutation is never refusable. The
    question a UI can usefully ask is `IsBoardAuthoringAllowed`.

## The transaction scope

```cpp
// RAII. Opens a command-stack transaction AND a grid change scope for its lifetime;
// the destructor closes the change scope (one batched notification) and then commits
// the transaction (one undo entry). Non-copyable.
struct FPGcGridTransactionScope
{
    FPGcGridTransactionScope(UPCsCommandStack* Stack, UGridGraph* Grid, const FString& Description);
    ~FPGcGridTransactionScope();

    // True iff the scope was handed a null stack or a null board — nothing was
    // opened, so do not submit. This is NOT a legality check.
    bool IsRefused() const;
};
```

!!! note "C++ only"
    The scope is a plain C++ struct and is not reachable from Blueprint. From
    Blueprint, wrap the submits in **Begin Transaction** / **Commit Transaction**
    on the command stack for the single undo unit; the batched single
    notification is the part you don't get, because nothing opens the grid's
    change scope. See
    [Group a brush stroke](guides.md#group-a-brush-stroke-into-one-undo-unit).

Pass the scope the stack you got from
[`ResolveActiveStructuralStack`](#the-routing-seam), never the command stack
fetched directly.

## The command family

Every structural edit is one of eleven command structs. They share a base that
carries the board's identity:

```cpp
// Base for every structural command.
struct FPGcStructuralCommand : public FPCsCommand
{
    // Which board this edit applies to. Empty (None) = the world's main board.
    // Resolved to a live board at apply time through the registry, so the command
    // survives a replay without holding a pointer.
    FName GridId;
};
```

!!! note "C++ only"
    The command structs are plain C++ types — they are not Blueprint types, so
    they do not appear as Blueprint variables or in a struct picker. Build a
    command in C++ and submit it; from Blueprint, drive the workflow around a
    small C++ helper of your own.

Each command follows the framework's
[command contract](../../concepts/commands-and-undo.md#writing-a-custom-command):
it validates its inputs *before* mutating anything, captures the absolute
pre-state it will restore, and reverses exactly what it did. The tables below
list the **fields you set**; each command also carries captured inverse state,
written during apply, which is never a caller input.

### Cells

| Command | Fields you set | What it does |
|---|---|---|
| `FPGcCmd_AddNode` | `bHasCoord`, `Coord` | Adds a cell at a coordinate, wired by the topology to whatever coord-neighbours already exist — or a coordinate-less cell for a freeform board when `bHasCoord` is false. Fails if the coordinate is already taken, or if the topology has no coordinates. Undo restores the handle allocator, so a redo re-allocates the *identical* handle. |
| `FPGcCmd_RemoveNode` | `Handle` | Removes a cell and everything about it — its coordinate, tags, height offset, and every connection touching it, with each connection's tags, cost override, and position in its neighbour's connection order. Undo puts all of that back identically. Fails if the handle is not live. |
| `FPGcCmd_SetNodeHeightOffset` | `Node`, `NewHeight` | Sets the cell's within-level vertical offset. `0` means no offset. |
| `FPGcCmd_AddNodeTag` | `Node`, `Tag` | Adds a gameplay tag to a cell. Adding a tag the cell already has is a no-op that undoes as one. |
| `FPGcCmd_RemoveNodeTag` | `Node`, `Tag` | Removes a gameplay tag from a cell. |

### Connections

Connections are **directional**, and every connection command takes
`bBothDirections` (default `true`) so you can treat a link as two-way without
spelling out both directions.

| Command | Fields you set | What it does |
|---|---|---|
| `FPGcCmd_AddEdge` | `From`, `To`, `bBothDirections` | Links two cells. Idempotent: undo removes only the directions this command actually added, and never touches tags or cost data it did not create. |
| `FPGcCmd_RemoveEdge` | `From`, `To`, `bBothDirections` | Severs a connection *and* drops its tags and cost override. Undo restores them — including the connection's original position in the neighbour's ordering, so the board is byte-identical afterwards. |
| `FPGcCmd_AddEdgeTag` | `A`, `B`, `Tag`, `bBothDirections` | Adds a gameplay tag to a connection. |
| `FPGcCmd_RemoveEdgeTag` | `A`, `B`, `Tag`, `bBothDirections` | Removes a gameplay tag from a connection. |
| `FPGcCmd_SetEdgeCostOverride` | `From`, `To`, `Cost`, `bBothDirections` | Sets an explicit step cost on a connection. Costs are whole scaled integers — one straight step is `10`. |
| `FPGcCmd_ClearEdgeCostOverride` | `From`, `To`, `bBothDirections` | Drops the override, returning the connection to the default step cost. |

!!! warning "Tags and costs need a connection to sit on"
    The four connection-surface commands — the two tag commands and the two cost
    commands — fail if the connection does not exist in the direction they were
    asked about. Tagging or costing a link into nothing is not representable,
    which is deliberate: it is what keeps a removed connection's undo record
    complete. Add the connection first.

### One command, one notification

Each command's apply *and* each command's undo raises exactly one
[structural-change notification](../gridgraph/reference-graph.md#the-structural-change-notification)
on the board — unless it is running inside a
[transaction scope](#the-transaction-scope), which batches the whole stroke into
one notification on the forward pass. Read that section for the forward-only
asymmetry before you write an incremental consumer.

## The grid registry

`UPGcGridRegistry` is a `UGameInstanceSubsystem` — one per running game instance,
matching the command stack — that maps a serialized name to a live board.

```cpp
// Resolve the registry through any world-context object (null if unavailable). C++ only.
static UPGcGridRegistry* Get(const UObject* WorldContextObject);

// Bind GridId to Grid. Empty (None) names the main board. Returns false — changing
// nothing, and logging a warning — if the id is already bound to a DIFFERENT live
// board. Re-registering the same board succeeds; so does replacing a registration
// whose board has been destroyed.
bool RegisterGrid(FName GridId, UGridGraph* Grid);

// Drop a registration. Silent no-op if the id was never registered.
void UnregisterGrid(FName GridId);

// The live board for an id, or null. Empty (None) resolves the registered main board,
// or — when nothing was registered under it — the world's main grid.
UGridGraph* ResolveGrid(FName GridId) const;

// True iff GridId currently resolves to a live board.
bool IsRegistered(FName GridId) const;
```

`RegisterGrid`, `UnregisterGrid`, `ResolveGrid`, and `IsRegistered` are all
Blueprint nodes under *Grid | Commands*. `Get` is C++ only.

!!! tip "A one-board game needs no wiring at all"
    Leave `GridId` empty on your commands. Resolution falls back to the world's
    main grid — the board whose component is flagged **Is Main Grid** — so the
    registry is something you only meet when you have a *second* board.

The registry holds boards weakly and never serializes them: it is a runtime
index, not saved state. It exists so a command can name a board across a replay
loaded into a fresh process. Spawning and destroying boards at runtime is outside
what it covers — it resolves ids and rejects conflicting registrations, nothing
more.

!!! note "`GridId` is the same name your placements use"
    A unit's cell reference already carries a grid id. Use the same name in both
    places so a placement and a command mean the same board. There is no authored
    id field on the grid component today — a second board is registered from code.

## The routing seam

Every structural edit is *routed*, not judged: which stack it lands on is decided
once, at the submit entry point.

| Situation | Where the edit lands |
|---|---|
| Ordinary play, between actions | The real command stack — its own undo entry, or your open brush stroke. |
| During an ability's resolution | The real stack, joined into that action's own transaction, so the dig undoes with the action. |
| Inside a preview or AI what-if run | The throwaway run's own record. The change still applies to the real board, and is rewound when the run is discarded. |

!!! note "C++ only"
    The whole `PGcStructuralRouting` namespace is native — two delegates and a
    handful of free functions, with no Blueprint surface at all. That is
    deliberate: from Blueprint, `Submit Grid Command` resolves the right stack
    internally, which is the correct behaviour anyway.

```cpp
namespace PGcStructuralRouting
{
    // The stack a structural edit should land on. This is what you pass to a
    // transaction scope. Falls back to the game's command stack when nothing is
    // registered, or when the registered resolver expresses no opinion; returns
    // null only when there is no command stack at all.
    UPCsCommandStack* ResolveActiveStructuralStack(const UObject* WorldContextObject);

    // Is board authoring (the re-conform) admitted right now? True when nothing is
    // registered — a board-only game authors freely with zero wiring.
    bool IsBoardAuthoringAdmitted(const UObject* WorldContextObject);

    // Registration. Single-bind, game-thread-only, module lifetime: Set replaces,
    // Clear unbinds. The Has* pair is for diagnostics — do not branch on it.
    void SetActiveStructuralStackResolver(const FPGcActiveStructuralStackResolver& Resolver);
    void ClearActiveStructuralStackResolver();
    bool HasActiveStructuralStackResolver();

    void SetBoardAuthoringAdmission(const FPGcBoardAuthoringAdmission& Predicate);
    void ClearBoardAuthoringAdmission();
    bool HasBoardAuthoringAdmission();
}
```

**You almost certainly register nothing.** With both seams unbound, edits go to
the game's command stack and board authoring is admitted — which is exactly right
for a game with a board and no entity layer. In a framework game the seam is
already wired for you: the GridEntity plugin registers both from its module
start-up, which is what makes an edit inside a preview run land on the preview's
record instead of your real history.

If you *do* register your own resolver, note that returning null from it means
"no opinion" and falls through to the game's command stack — it never means
"nowhere". There is no way to make a structural edit vanish by routing it.

## Diagnostics

The plugin logs under the `LogPGcGridCommands` category. A refused re-conform
logs there at warning level with the reason, which is the fastest way to find out
why a board-authoring button did nothing.
