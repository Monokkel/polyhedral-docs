# GridGraph

For developers building a board — square, hex, or freeform; flat or multi-storey —
who need "where is everything?" and "where can this unit go?" answered correctly
through undo, save, and replay. After this section you'll know how a board is built,
the queries that read it, how a unit's reach is computed, and how to draw the result
on screen.

A board is a **static stage**: cells joined by connections, with a **topology** that
owns the shape math — coordinates, which cells are neighbors, and where each cell sits
in the world. You build it once — proposing cells from a pure-shape pattern, then
optionally conforming them to your level's geometry — and from then on the map holds
only durable facts. Everything per-unit and per-moment — "which cells can *this* unit
reach, and for how much" — is a **query**, computed fresh each time you ask, never
baked into the map. That one rule is the keystone: a connection means "these cells are
linked," never "this unit may pass," so nothing about a specific unit's movement is
ever stored on the board. It is what lets the same board hash, save, and rewind
identically, and it is why a replay never drifts.

!!! note "Read the concept first"
    This page is the plugin's overview; it assumes you know the model. If you don't
    yet, start with
    [Grids & Occupancy](../../concepts/grids-and-occupancy.md) in Core Concepts — it
    owns the "why": the static-stage idea, the difference between a shape query and a
    movement query, and how movement is assembled from data. This section owns the
    "how": the exact calls and the tasks you'll actually perform.

## Where it fits in the framework

GridGraph is **standalone and zero-dependency**. It needs none of the other plugins,
and a game with no game-entity layer at all can build a board, run a reachability
flood, and draw the result. GridGraph owns the grid data, the shape math, generation,
the movement engine, and the on-screen presenters — the whole spatial layer.

What it deliberately does *not* own is any tie to your game's units. Placing an entity
on a cell, asking "who is standing here?", and granting a unit the boots that let it
leap — those bind the grid to game entities and live in the **GridEntity** plugin,
documented in its own section. GridGraph never inspects entity state: the movement
query takes a plain snapshot of a unit's tags and stats and reads the board through a
read-only view of the live board, so it stays a pure function you can replay. Because
the grid itself is a self-contained value that saves and reloads verbatim, and a
unit's position is ordinary entity data (again, GridEntity), the whole picture undoes,
saves, and replays as one.

One board is the world's **main board** by default — the board these systems build and
query against when no grid is named. Flag a grid as the main grid and it registers for
static access from anywhere.

## The two questions a board answers

"Cells within 2" and "cells this unit can reach" sound alike and are answered by
completely different machinery. Reaching for the wrong one is the classic grid-tactics
bug:

- A **pattern** answers *geometric proximity* — every cell within 2, by coordinate
  math. Walls, closed doors, and teleporters cannot bend it. That is exactly what you
  want for a blast radius, an aura, or a targeting ring.
- **Reachability** answers *movement range* — the cells a unit can actually get to — by
  a cost-aware search over the grid's connections, the cost of each step, and the
  unit's own movement rules. That is what you want for a move preview.

At a wall the two sets diverge on purpose: a blast still covers the cells behind the
wall; the reachable set stops at it. The recipe
[Show a shape versus a range](guides.md#show-a-shape-versus-a-range) walks both — use a
pattern for a blast and reachability for movement, never the reverse.

## The pieces at a glance

| Piece | What it is | Reference |
|---|---|---|
| **Topology** | The shape math for a layout — square (four- or eight-way), hex (six-way), or freeform (arbitrary positions, no coordinates). Owns coordinates, neighbors, and world positions. | [Graph & topologies](reference-graph.md) |
| **Grid** | The board data: cells, the connections between them, per-cell heights, and gameplay tags on cells and connections. Pure data — untouched as units move. | [Graph & topologies](reference-graph.md) |
| **Grid component** | The actor component that hosts a grid in the world and runs the single **Generate Grid** bake from a seed pattern (and optional conform). | [Generation & patterns](reference-generation.md) |
| **Patterns** | Pure-shape footprints — a rectangle, a filled disk, a ring — with parameters only. Compose with union, subtract, intersect, and exclusive-or to carve any shape. | [Generation & patterns](reference-generation.md) |
| **Reachability & movement** | The cost-aware flood that answers "where can this unit go," and the movement profile (movement sources + cost modifiers) it searches under. Costs are whole scaled integers. | [Movement & reachability](reference-movement.md) |
| **Presenters** | A visualization driver that draws a supplied node set — a move range or a path — over the board, using a data-asset profile of stackable presenters. | [Visualization](reference-visualization.md) |

## Where to go next

- **[Guides](guides.md)** — task recipes: building a board, querying it,
  computing a unit's reachable set, telling a shape from a range, and drawing a range
  or path on screen.
- **[Graph & topologies](reference-graph.md)** — the grid data structure and the
  square, hex, and freeform topologies.
- **[Generation & patterns](reference-generation.md)** — the Generate Grid workflow,
  conforming a board to your level, and the pattern library.
- **[Movement & reachability](reference-movement.md)** — the reachability and
  pathfinding surface, and how a per-unit movement profile is assembled.
- **[Visualization](reference-visualization.md)** — the visualization driver,
  profiles, and presenters.
- **[Grids & Occupancy](../../concepts/grids-and-occupancy.md)** — the model and the
  rationale behind the static-stage rule.
