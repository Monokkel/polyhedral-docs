# GridEntity

For developers who have units in a game state and a board in the world, and now
need the two to meet: a unit that *stands on a cell*, moves there undoably, and can
be asked "who else is here?" After this section you'll know how a unit's position is
stored, how the "who is on this cell" index is kept correct for free, how per-unit
movement is run against the live board, and how the on-screen mini follows along.

GridEntity is the one plugin where the **grid system** and the **entity system**
meet. On one side sits [GameEntity](../gameentity/index.md): value-struct entities
living in a central game state, mutated only through commands. On the other sits
[GridGraph](../gridgraph/index.md): a static board — cells, connections, and the
shape math over them. GridGraph knows nothing about your units; GameEntity knows
nothing about the board. GridEntity is the seam that binds them, and it is the only
place the two systems know about each other.

!!! note "Read the concept first"
    Start with [Grids & Occupancy](../../concepts/grids-and-occupancy.md) in Core
    Concepts. It explains the model this section implements — a static board versus
    a per-unit query, placement as entity data, and occupancy as a derived index —
    and everything below assumes it.

## What it adds over a raw board

A [GridGraph](../gridgraph/index.md) board is pure geometry. It can tell you which
cells connect, which cells lie within two steps, and — given a mover's profile — the
cost-aware reachable set. What it deliberately cannot tell you is anything about
*your game's units*, because it has never heard of them.

GridEntity supplies exactly that missing half:

- **A unit's position becomes entity data.** Instead of a transform on an Actor, a
  unit's cell is stored as tagged data on the entity, written through a command — so
  moving a unit undoes, saves, and replays with the rest of game state, with no extra
  machinery.
- **"Who is on this cell" gets an answer.** A derived index, rebuilt reactively from
  the same placement data, reports a cell's occupants — correct through undo and load,
  with nothing of its own to save.
- **Movement becomes a query about a *unit*.** You hand it an entity, not a raw
  profile; GridEntity reads the unit's granted capabilities, snapshots its stats, and
  runs the board search — feeding the live occupancy back in so a unit's reach
  respects who is standing where.
- **The mini follows on its own.** A grid-aware presentation token derives its world
  position from the unit's placement and walks the path on a move — no per-move
  positioning code.

## Where it fits in the framework

GridEntity sits **downstream** of three plugins and depends on all of them; the
dependency arrows point one way, into GridEntity, never out.

- **[GameEntity](../gameentity/index.md)** — the entities GridEntity places, and the
  command-routed mutators (set tagged data, create an entity) that all its writes
  ride on. GridEntity authors **no commands of its own**; every change it makes is a
  GameEntity command underneath, which is why it inherits undo, save, and replay for
  free.
- **[GridGraph](../gridgraph/index.md)** — the board its entities stand on, and the
  movement engine its per-unit reachability calls into. GridEntity supplies the
  entity-backed [read-only view of the live board](../gridgraph/reference-movement.md#the-movement-snapshot-and-board-view)
  that engine reads occupants through.
- **[TokenSystem](../tokensystem/index.md)** — the presentation layer its grid token
  specializes. The grid token is an ordinary [token actor](../tokensystem/reference.md#the-token-interface-and-lifecycle)
  taught to place itself from grid data and walk a move path.

Nothing drawable and nothing in the grid or entity cores depends *back* on
GridEntity — you can build a board with no entities, or run entities with no board,
and only pull this plugin in where the two must actually meet.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPGxPlacementLibrary` | Static functions to create a unit already standing on a cell, and to move it — each riding a GameEntity command, so each is undoable. |
| `Data.Placement.*` tags | The tagged-data keys a unit's position is stored under: `Data.Placement.Cell` on a grid, `Data.Placement.World` off it. |
| `UPGxOccupancySubsystem` | The derived cell&nbsp;&rarr;&nbsp;entities index. Ask it who is on a cell; it reconciles itself from change events and stays correct through undo. |
| `UPGxMovementLibrary` | Static functions to grant/revoke a unit's movement capabilities and to compute its reachability and paths against the live board. |
| `Data.Movement.*` tags | The tagged-data keys a unit's movement sources and cost modifiers are granted under. |
| `FPGxEntityBoardView` | The read-only view of the live board a movement search reads occupants through — backed by occupancy and entity tags. |
| `APGxGridTokenActor` | A grid-aware presentation token: it snaps to its entity's cell and walks the path on a move cue. |
| `UPGxSelectionVisualizationSubsystem` | An optional feeder that turns a selected unit into a reachable-tile highlight on a grid render surface. |

## Where to go next

- **[Guides](guides.md)** — task recipes: place a unit born on a cell, move it (and
  wrap the move when it also spends a movement point), query who is on a cell, grant
  and revoke a movement capability, and show a unit with a grid token.
- **[API Reference](reference.md)** — the full public surface grouped by area:
  placement, occupancy, movement, the board view, the grid token, and selection
  visualization.
- **[Grids & Occupancy](../../concepts/grids-and-occupancy.md)** — the model and the
  reasoning behind placement, per-unit movement, and the occupancy index.
- **[Derived State & Events](../../concepts/derived-state.md)** — the reactive
  pattern the occupancy index (and the selection highlight) are built on.
- **[Commands & Undo](../../concepts/commands-and-undo.md)** — why placing and moving
  a unit undoes with everything else.
