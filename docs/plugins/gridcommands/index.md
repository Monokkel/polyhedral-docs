# GridCommands

For developers whose board changes *during play* — a tunnel dug, a bridge dropped,
a wall blown open, a city built tile by tile — and who need those edits to undo,
save, and replay exactly like everything else in the game. After this section
you'll know how a structural edit becomes a command, how to group a whole brush
stroke into a single undo step, and what the rest of the framework does when a
cell disappears out from under a unit.

## The problem it solves

A [GridGraph](../gridgraph/index.md) board is built once, at bake time, and its
raw construction calls — add a cell, remove a cell, link two cells, tag a
connection — mutate the graph directly. That is exactly right while you are
authoring a board in the editor. At runtime it is a trap: a direct edit lands on
no command stack, so it does not undo, does not replay, and nothing downstream is
told it happened. Dig a tunnel that way, walk a unit through it, then press undo,
and the unit walks back through a tunnel that never rewound.

GridCommands makes structural change **first-class state**. Every edit —

- add or remove a cell,
- add or remove a connection,
- add or remove a cell tag,
- add or remove a connection tag,
- set a cell's height offset,
- set or clear a connection's cost override

— is a command submitted to the *same* command stack as your entity commands, in
one total order. Undoing the action that dug the tunnel and moved the unit
rewinds tunnel and unit together, in the right order, with nothing to reconcile
by hand.

Reach for this plugin when the board is something the player (or your level
tooling) changes:

- runtime excavation and destructible terrain,
- city-builders and tile placement,
- in-game level editors and board-authoring tools that need a real undo history,
- any rule that permanently opens or closes a route.

Don't reach for it to answer "may *this* unit pass here?" — that is a movement
query, computed per unit and never stored on the board (see
[Movement & reachability](../gridgraph/reference-movement.md)) — nor to move a
unit, which is [placement](../gridentity/reference.md#placement) on the entity,
not a graph edit.

## One brush stroke, one undo unit

A single edit submitted on its own is its own undo entry, which is usually what a
gameplay rule wants ("the bridge collapses"). A *tool* wants something else: the
player drags across nine cells and expects one Ctrl+Z to take all nine back.

That is what the transaction scope is for. It opens a
[command-stack transaction](../commandsystem/reference.md#transactions) and a grid
change scope together for its lifetime, so every command submitted inside it
becomes:

- **one undo unit** — one press of undo reverses the whole stroke, in reverse
  order, and one redo re-applies it; and
- **one structural-change notification** on the forward pass, instead of nine, so
  a visualizer rebuilds once per stroke rather than once per cell.

!!! note "C++ only"
    The transaction scope (`FPGcGridTransactionScope`) is a plain C++ RAII type.
    Blueprint can still group a stroke into one undo unit with **Begin
    Transaction** / **Commit Transaction** on the command stack — it just doesn't
    get the batched single notification, because nothing opened the grid's change
    scope. See [Group a brush stroke](guides.md#group-a-brush-stroke-into-one-undo-unit).

## Where it fits in the framework

GridCommands sits between two plugins and depends on exactly those two:

- **[GridGraph](../gridgraph/index.md)** — the board being edited. GridGraph
  itself has no idea commands exist (it is zero-dependency by charter); it
  publishes the internal apply surface and the
  [structural-change notification](../gridgraph/reference-graph.md#the-structural-change-notification)
  that this plugin drives.
- **[CommandSystem](../commandsystem/index.md)** — the stack every edit is
  submitted to, and the transaction machinery a brush stroke groups with.

It is deliberately **entity-free**: it never mentions a unit, a placement, or an
occupancy index. That keeps it usable in a game with no entity layer at all — an
in-game level editor with a board and an undo button needs nothing else.

When your board *does* carry units, the consequences of removing a cell someone
is standing on are handled one layer up, by
[GridEntity](../gridentity/index.md): occupancy drops a placement that no longer
resolves, reports the affected units, and undo puts both back. See
[When the board itself changes](../../concepts/grids-and-occupancy.md#when-the-board-itself-changes)
for that whole story.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPGcGridCommandLibrary` | The submit surface: **Submit Grid Command**, the board-authoring check, and the command-routed re-conform. |
| The command family (`FPGcCmd_*`) | Eleven C++ command structs, one per kind of structural edit, each capturing its own exact inverse. |
| `FPGcGridTransactionScope` | The C++-only brush-stroke scope: one undo unit and one forward notification for N commands. |
| `UPGcGridRegistry` | Resolves a serialized `GridId` name to a live board at apply time, so a command survives a replay without holding a pointer. |
| The routing seam | C++-only wiring that decides *which* stack an edit lands on. Registering nothing is a supported, sensible default. |

## What is reachable from Blueprint

| Surface | Blueprint |
|---|---|
| **Submit Grid Command** | Yes |
| **Register Grid** / **Unregister Grid** / **Resolve Grid** / **Is Registered** | Yes |
| **Is Board Authoring Allowed** | Yes |
| **Submit Reconform** | Yes |
| The eleven command structs | **No** — plain C++ structs, built in C++ and submitted |
| `FPGcGridTransactionScope` | **No** — C++ RAII type |
| `SubmitReconformPlan` | **No** — takes a plain C++ plan value |
| The routing seam | **No** — C++ delegates, module lifetime |

## Where to go next

- **[Guides](guides.md)** — the recipes: dig a cell in and out at runtime, group a
  brush stroke, register a board under a name, offer a board-authoring button
  safely, and react to a structural change.
- **[API Reference](reference.md)** — the full public surface: the registry, the
  command family field by field, the submit library, the transaction scope, and
  the routing seam.
- **[The Grid & Topologies](../gridgraph/reference-graph.md)** — the board these
  commands edit, the handle-stability contract they rest on, and the
  structural-change notification they raise.
- **[Commands & Undo](../../concepts/commands-and-undo.md)** — the model: one
  door to state, inverse capture, and transactions.
- **[Grids & Occupancy](../../concepts/grids-and-occupancy.md)** — what a mutable
  board means for units standing on it.
