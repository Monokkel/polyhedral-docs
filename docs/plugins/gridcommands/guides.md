# GridCommands Guides

For developers with a board in the world who now need to change it while the game
is running, and keep undo, save, and replay honest. Each section is a short,
followable recipe. For the plugin tour start at the [overview](index.md); for the
full surface see the [API Reference](reference.md); for the model see
[Grids & Occupancy](../../concepts/grids-and-occupancy.md) and
[Commands & Undo](../../concepts/commands-and-undo.md).

Most of this page is C++, and honestly so: the command structs and the
transaction scope are C++ types. What Blueprint *does* get — submitting a
command, registering a board, the board-authoring check, the re-conform — is
called out in each recipe.

## The shape of every structural edit

Four steps, always in this order:

1. **Resolve the stack** the edit should land on, through the plugin's routing
   helper — never by grabbing the command stack directly.
2. **Resolve the board** you are editing, through the grid registry. An empty
   `GridId` means the world's main board.
3. **Open a transaction scope** if this edit is part of one player-visible action.
4. **Submit** one or more commands.

```cpp
#include "PGcGridCommandLibrary.h"   // the submit library + the transaction scope
#include "PGcStructuralCommands.h"   // the command family
#include "PGcStructuralRouting.h"    // ResolveActiveStructuralStack
#include "PGcGridRegistry.h"

// 1. The stack this edit belongs on. Resolve it here, not with the stack's own Get:
//    while a preview or an AI what-if run is in flight, edits belong somewhere else,
//    and this helper is the one place that knows where.
UPCsCommandStack* Stack = PGcStructuralRouting::ResolveActiveStructuralStack(this);

// 2. The board. NAME_None = the world's main board.
UGridGraph* Grid = UPGcGridRegistry::Get(this)->ResolveGrid(NAME_None);
```

!!! warning "Resolve the stack through the routing helper"
    Reaching for the command stack directly works right up until the moment
    something runs your edit inside a preview or an AI what-if run — and then the
    edit lands on the real history instead of the throwaway one, and the board
    keeps a change the player never made. `ResolveActiveStructuralStack` returns
    the real stack when there is nothing special going on, so using it costs you
    nothing and covers the case you would otherwise have to remember.

## Dig a cell in, and take it back

The smallest complete example: add a cell, connect it, and undo the lot.

=== "Blueprint"
    1. Build the command value in C++ and expose a small helper — the command
       structs are not Blueprint types.
    2. From Blueprint call your helper, or drive the workflow around it with
       **Resolve Grid**, **Is Registered**, and **Is Board Authoring Allowed**.
    3. **Undo** on the command stack reverses the edit exactly like any other
       command; the grid is a value, so nothing else needs telling.

=== "C++"
    ```cpp
    // Add a cell at a coordinate. The topology wires it to whatever coord-neighbours
    // are already there, so a cell dug beside a corridor joins the corridor.
    FPGcCmd_AddNode Add;
    Add.GridId    = NAME_None;      // the main board
    Add.bHasCoord = true;
    Add.Coord     = FIntVector(4, 0, 0);
    UPGcGridCommandLibrary::SubmitGridCommand(this, FInstancedStruct::Make(Add));

    // Remove a cell — everything about it (its connections, tags, height) goes with it,
    // and comes back byte-identically on undo.
    FPGcCmd_RemoveNode Remove;
    Remove.GridId = NAME_None;
    Remove.Handle = Grid->GetNodeAt(FIntVector(5, 0, 0));
    UPGcGridCommandLibrary::SubmitGridCommand(this, FInstancedStruct::Make(Remove));

    Stack->Undo();   // the cell at (5,0,0) is back, with its connections and tags
    Stack->Undo();   // the cell at (4,0,0) is gone again
    ```

Each `Submit Grid Command` outside a transaction is **its own undo entry** and
raises **its own** structural-change notification. That is what you want for a
gameplay rule — "the bridge collapses" is one thing that happened.

!!! note "A command validates itself, and fails without mutating"
    Adding a cell at an occupied coordinate, removing a handle that is not live,
    or tagging a connection that does not exist all fail cleanly: the call
    returns `false`, nothing is mutated, and nothing is pushed onto the stack.
    You never have to check first to stay safe — check first only when the *UI*
    needs to know.

## Group a brush stroke into one undo unit

A tool drag is one action to the player. Wrap the submits in a transaction scope
and they become one undo entry — and, on the forward pass, one notification.

=== "Blueprint"
    Call **Begin Transaction** on the command stack, submit your edits, then
    **Commit Transaction**. That gives you the single undo unit.

    It does **not** give you the single structural-change notification: each
    command still raises its own, because only the C++ scope opens the grid's
    change scope. For a Blueprint-driven tool that is usually fine — the shipped
    grid visualizers rebuild from scratch and are happy to do it a few extra
    times.

=== "C++"
    ```cpp
    {
        FPGcGridTransactionScope Stroke(Stack, Grid, TEXT("Dig tunnel"));
        if (Stroke.IsRefused())
        {
            return;   // it was handed a null stack or a null board: nothing was opened
        }

        for (const FIntVector& Coord : BrushCells)
        {
            FPGcCmd_AddNode Add;
            Add.GridId    = NAME_None;
            Add.bHasCoord = true;
            Add.Coord     = Coord;
            UPGcGridCommandLibrary::SubmitGridCommand(this, FInstancedStruct::Make(Add));
        }
    }   // the scope closes: one batched notification, then one undo entry

    Stack->Undo();   // the whole stroke rewinds
    ```

`IsRefused()` is true for exactly one reason — the scope was handed a null stack
or a null board, so there was nothing to open. It is not a legality check; see
[Offer a board-authoring button](#offer-a-board-authoring-button-safely) for the
question a UI actually needs to ask.

!!! tip "A stroke inside an ability joins that ability's step"
    If your edit runs during an ability's resolution, the submit already lands
    inside that action's own transaction, so the dig and the rest of the action
    undo as one unit with no extra work from you. Nested transactions join the
    open one rather than starting a new one — see
    [Transactions](../commandsystem/reference.md#transactions).

## Register a board under a name

A command cannot hold a pointer to a board — a pointer means nothing to a replay
loaded in a fresh process. It carries a `GridId` name instead and resolves it to
a live board at the moment it applies.

**The zero-wiring case is the common one.** Leave `GridId` empty (`None`) and it
resolves to the world's main board, falling back to the main-grid registration
that a grid component flagged **Is Main Grid** already performs. A game with one
board registers nothing.

=== "Blueprint"
    1. For a second board, call **Register Grid** on the grid registry with the
       name you want and the board's **Grid** object — typically once, at
       start-up or when the board is spawned.
    2. **Resolve Grid** turns a name back into a board; **Is Registered** asks
       whether a name currently resolves.
    3. **Unregister Grid** drops a registration. It is a silent no-op if the name
       was never registered.

=== "C++"
    ```cpp
    UPGcGridRegistry* Registry = UPGcGridRegistry::Get(this);

    // A second board — the tactical map is the main board, this one is the ship interior.
    Registry->RegisterGrid(TEXT("ShipInterior"), ShipGridComponent->Grid);

    // Commands then name it.
    FPGcCmd_AddNodeTag Tag;
    Tag.GridId = TEXT("ShipInterior");
    Tag.Node   = SomeCell;
    Tag.Tag    = TAG_Terrain_Scorched;
    UPGcGridCommandLibrary::SubmitGridCommand(this, FInstancedStruct::Make(Tag));
    ```

!!! warning "One name, one board"
    Registering a name that is already bound to a **different** live board is
    rejected: the call returns `false`, changes nothing, and logs a warning.
    Re-registering the *same* board under the same name succeeds, and so does
    replacing a registration whose board has since been destroyed — the registry
    holds boards weakly. Use the same id in the placement data your units carry,
    so a cell reference and a command mean the same board.

## Offer a board-authoring button safely

Ordinary structural commands are never refused — they are only ever *routed*, so
a dig is legal whenever it is legal for an entity command. There is one operation
that is refusable, because it rebuilds the board wholesale: the
**re-conform**, which re-reads your level's geometry and applies the difference.

Ask before you offer it:

=== "Blueprint"
    1. Bind the button's enabled state to **Is Board Authoring Allowed**, passing
       the board's id (leave the name empty for the main board).
    2. On click, call **Submit Reconform** with the id, the base coordinate set
       the board is generated from, and your **Conform Settings**.
    3. A refused call returns `false`, mutates nothing, and leaves no undo entry —
       so a mistimed click is harmless, it just does nothing.

=== "C++"
    ```cpp
    if (!UPGcGridCommandLibrary::IsBoardAuthoringAllowed(this, NAME_None))
    {
        return;   // grey the button out instead
    }

    // Re-read the level and apply the DIFFERENCE as one undoable transaction:
    // surviving cells keep their handles, only added/removed/changed cells get commands.
    UPGcGridCommandLibrary::SubmitReconform(this, NAME_None, BaseCoords, ConformSettings);
    ```

A re-conform that finds nothing to change still reports success — "the board is
already correct" is not a failure. What makes the call refuse is the board being
busy in a way a wholesale rebuild would corrupt: an ability resolution or a
preview run in flight, or a re-conform already running.

!!! note "This is the runtime door; the editor button is a different one"
    **Generate Grid** and the editor's **Reconform** button are bake-time
    authoring and are not command-routed. The runtime, undoable path is this one.
    See [Re-conforming at runtime](../gridgraph/reference-generation.md#re-conforming-at-runtime).

## React to a structural change

Anything that mirrors the board — a custom visualizer, a fog-of-war mask, a
navigation cache — wants to know when the board changed and what changed. The
board publishes a **structural delta** describing exactly that, on a C++ delegate.

```cpp
// Subscribe once, e.g. when your subsystem binds to the board.
Grid->OnStructureChanged.AddRaw(this, &FMyMirror::HandleStructureChanged);

void FMyMirror::HandleStructureChanged(const FGridStructuralDelta& Delta)
{
    for (const FGridStructuralNodeChange& Added   : Delta.AddedNodes)   { /* ... */ }
    for (const FGridStructuralNodeChange& Removed : Delta.RemovedNodes) { /* ... */ }
    // Delta.AddedEdges / RemovedEdges / SurfaceChangedNodes / SurfaceChangedEdges
}
```

Two properties of that notification decide how you should write the handler, and
both surprise people:

- **It fires on undo too**, describing the change that just happened — undoing an
  add reports a *removed* cell. A handler that reconciles rather than
  "applies forward" is correct in both directions with no extra code.
- **The batching is forward-only.** A stroke of N commands raises **one**
  notification when it is submitted, but **N** when it is undone and **N** when it
  is redone.

!!! warning "Don't count notifications"
    Because an N-command stroke notifies once forward and N times backward, a
    handler that does per-notification bookkeeping — incrementing a version
    number, appending to a change log, kicking off one animation per
    notification — will not balance across an undo. Write handlers that are
    **idempotent reconcilers**: given the delta, make your mirror agree with the
    board, and be safe to run again. That is how the shipped consumers are
    written, and it is why the asymmetry is harmless to them.

!!! note "C++ only"
    `OnStructureChanged` is a plain C++ multicast delegate — it is not
    Blueprint-assignable, because the delta is a rich value struct and every
    consumer of it is native. Blueprint-side, the grid component forwards a
    structural change into its existing **On Grid Rebuilt** event, so a Blueprint
    visualizer that already rebuilds on that event keeps working. See
    [The structural-change notification](../gridgraph/reference-graph.md#the-structural-change-notification).

## Remove cells that units are standing on

GridCommands has never heard of a unit, and removing a cell someone occupies is
perfectly legal — the framework does not adjudicate where a displaced unit goes,
because that is your game's rule, not a framework decision.

What you get for free is **detection**, and it is total: cell handles are never
recycled, so a placement pointing at a removed cell can never quietly come to
mean a different cell. The occupancy index drops such a placement and reports the
units it affected; undoing the removal makes those placements valid again with no
compensating writes of your own.

If you want the displacement wrapped into the edit itself — a chance for a rule
to veto the removal before it happens, and a per-unit signal afterwards that your
abilities can react to — use GridEntity's cell-removal door
(`SubmitCellRemovalWithDisplacement`) instead of submitting a remove-cell command
yourself. It behaves identically in live play and in a preview run. See the
[GridEntity reference](../gridentity/reference.md) and
[When the board itself changes](../../concepts/grids-and-occupancy.md#when-the-board-itself-changes).
