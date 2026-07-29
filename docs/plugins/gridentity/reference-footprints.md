# Footprints: Units on More Than One Cell

For developers whose game has a 2&times;2 golem, a 1&times;3 siege engine, or a dragon
that turns. After this page you can author a multi-cell shape on a template, rotate a
unit that has one, ask whether it may stand somewhere, and understand which of the two
movement gates you get for free and which you must grant.

Signatures below are hand-written to show the shape of each call; they are
illustrative, not source excerpts.

## The model: an anchor, a shape, and a rotation

A multi-cell unit is still a unit **at one cell**. Its
[placement](reference.md#placement) names an **anchor** cell, exactly as a single-cell
unit's does, and its shape is a separate, optional piece of tagged data: a list of
**anchor-relative offsets**.

- The anchor stays the single authoritative position fact. "Where is this unit?"
  always has one answer, and moving it is one write.
- The shape is *unrotated* offsets. The unit's
  [rotation step](reference-facing.md#the-model-facing-is-placement-data) turns them,
  so one authored shape covers every orientation.
- **Resolved cells** are what the offsets come out as on a particular board at a
  particular anchor and rotation. Everything downstream — occupancy, the movement
  gates, arc classification — speaks in resolved cells.

A unit with no footprint (the common case) occupies exactly its anchor cell, and every
call on this page behaves sensibly for it.

```cpp
// The optional multi-cell shape, stored under Data.Placement.Footprint.
struct FPGxFootprint
{
    // Anchor-relative, UNROTATED offsets. Empty = the anchor cell only.
    TArray<FIntVector> Offsets;

    // Where the on-screen mini pivots: Centroid (default) or Anchor.
    EPGxFootprintPivot Pivot;
};
```

!!! warning "The anchor cell is not implicit"
    `(0,0,0)` is **not** added for you. A shape authored as
    `[(1,0,0), (0,1,0), (1,1,0)]` leaves the anchor cell *unoccupied* — which is a
    legal doughnut, and almost never what you meant. A 2&times;2 unit's offsets are
    `[(0,0,0), (1,0,0), (0,1,0), (1,1,0)]`.

### Authoring one

A footprint is ordinary [entity tagged data](../gameentity/reference-entities.md) under
`Data.Placement.Footprint`, so it is authored on the **template** and inherited by every
instance created from it — no per-instance setup.

=== "Blueprint"
    1. In the unit's data-table row, add a **Tagged Data** entry keyed
       `Data.Placement.Footprint`.
    2. Fill **Offsets** with one entry per cell the unit covers, *including*
       `(0,0,0)` for the anchor itself.
    3. Leave **Pivot** on **Centroid** unless the mesh is authored anchor-relative.
    4. Create the unit with **Create Entity At Cell** as usual. It is multi-cell from
       birth; the occupancy index already lists it on every cell it covers.

=== "C++"
    ```cpp
    // A 2x2 golem, authored on its template (usually in the data-table row instead).
    FPGxFootprint Golem;
    Golem.Offsets = { FIntVector(0,0,0), FIntVector(1,0,0),
                      FIntVector(0,1,0), FIntVector(1,1,0) };
    Golem.Pivot   = EPGxFootprintPivot::Centroid;
    ```

| Pivot | Use it when |
|---|---|
| **Centroid** (default) | The mesh sits at the true centre of the shape — a 2&times;2 golem straddling four cells. |
| **Anchor** | The mesh is authored relative to the anchor cell itself. |

The pivot is presentation only: it decides where the mini stands and turns, never
which cells the unit occupies. It is also where arc classification measures a
defender's bearing from, so visuals and rules agree about where a unit "is".

### Changing the shape at runtime

A live override sits on top of the template shape — a giant squeezing into a
one-cell form, a construct that unfolds.

```cpp
// Write a live shape override. One command — undoable, and occupancy and the token follow.
static void SetEntityFootprint(
    const UObject* WorldContext, const FPGeEntityRef& Ref, const FPGxFootprint& Footprint);

// Drop the override so reads fall back to the template shape. False if there was none.
static bool ClearEntityFootprintOverride(const UObject* WorldContext, const FPGeEntityRef& Ref);

// Read the ACTIVE shape: the live override if there is one, else the template's.
// False when the unit is single-cell.
static bool GetEntityFootprint(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FPGxFootprint& OutFootprint);
```

**Set Entity Footprint** and **Clear Entity Footprint Override** are Blueprint
callable; **Get Entity Footprint** is Blueprint pure.

## Rotating a footprinted unit

Rotation is a placement write like any other, and it changes the covered cells.

```cpp
// Set the logical rotation step. One command; cell, board, and cosmetic facing survive.
static bool RotateEntityInPlace(
    const UObject* WorldContext, const FPGeEntityRef& Ref, int32 RotationSteps);

// Move AND rotate in ONE write — one command, one undo step.
static void MoveEntityToCellOriented(
    const UObject* WorldContext, const FPGeEntityRef& Ref,
    FGridNodeHandle Cell, int32 RotationSteps, UGridGraph* Grid = nullptr);

// Born placed AND oriented: cell, rotation step, and cosmetic facing all seeded into
// the creation command — there is never an unoriented instant.
static FPGeEntityRef CreateEntityAtCellOriented(
    const UObject* WorldContext, FName TemplateId, FGridNodeHandle Cell,
    int32 RotationSteps, FRotator Facing, UGridGraph* Grid = nullptr);
```

All three are Blueprint callable. Prefer **Move Entity To Cell Oriented** over a move
followed by a rotate: it is one command instead of two, so it is one undo step and the
unit never passes through an intermediate orientation the rules could observe.

## Asking where a shape lands

Two queries answer "what would this cover?" without changing anything.

```cpp
// Resolve the unit's ACTIVE shape at a hypothetical anchor and rotation.
// Returns whether EVERY cell resolved; OutCells holds the ones that did.
static bool ResolveEntityFootprintAt(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGridNodeHandle AnchorCell,
    int32 RotationSteps, TArray<FGridNodeHandle>& OutCells, UGridGraph* Grid = nullptr);

// Every entity overlapping that hypothetical footprint, EXCLUDING Ref itself.
// Deterministic order (ascending entity id). For rules gating, AI scoring, UI previews.
static TArray<FPGeEntityRef> GetFootprintOccupants(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGridNodeHandle AnchorCell,
    int32 RotationSteps, UGridGraph* Grid = nullptr);
```

Both are Blueprint callable. For "which cells is this unit on *right now*", use the
[occupancy index](reference.md#occupancy) instead — **Get Cells Of Entity** and
**Is Entity On Cell** answer that without recomputing anything.

## The two movement gates

A footprinted unit needs two different things to be true of a destination anchor, and
the framework treats them as two separate rules — each terminal on its own concern.

| Gate | How you get it | What it decides |
|---|---|---|
| **Fit** | **Automatic.** Injected into the query for you whenever the mover has a shape. | Every cell of the shape lands on a real cell of the board. A 2&times;2 golem cannot path to an anchor where part of it would hang off the edge. |
| **Connectivity (cohesion)** | **Opt-in.** Granted like any other movement capability. | The covered cells hang together — no straddling a wall. |

### Fit is automatic

You never grant the fit gate and you never author it. The
[per-unit movement queries](reference.md#computing-reachability-and-paths) —
**Compute Entity Reachability** and **Find Entity Movement Path** — inject it at query
time when the mover has a non-empty active shape, using the unit's current offsets and
rotation. Nothing is stored, and it reads only the *static* board, so it costs nothing
in board state.

Fit is purely geometric. Whether the destination is *occupied* is a separate question
— a rule you write as a [cost modifier](reference.md#movement-capabilities-are-granted-tagged-data)
over occupancy, exactly as for single-cell units.

### Connectivity is granted

Deliberately disconnected silhouettes are legal, so cohesion is a policy you opt into
rather than a law:

```cpp
// Grant the wall-cohesion gate. Rides one set-tagged-data command — undoable.
// SeveringEdgeTags: connections whose LIVE tags match any of these also count as severed
//   (a raised drawbridge, a closed portcullis). Empty = structural walls only.
// bTreatSameColumnLayersAsConnected: cells differing only in height count as one column,
//   so a two-storey giant needs no authored vertical connections. Turn it OFF for a game
//   whose levels genuinely are wired together with vertical connections.
static void GrantFootprintConnected(
    const UObject* WorldContext, const FPGeEntityRef& Ref,
    FGameplayTagContainer SeveringEdgeTags, FPGeEntityRef GrantSource,
    int32 Priority = 0, bool bTreatSameColumnLayersAsConnected = true);
```

It is Blueprint callable, and a convenience over **Grant Movement Modifier** on the
`Data.Movement.Modifier.FootprintConnected` key — so it is revoked like any other
grant, including by source. Grant it to every unit in a game with thin walls, and a
1&times;2 horse can no longer stand half in the corridor and half in the room.

!!! note "You grant policy, not geometry"
    The grant carries only the two policy fields above. The mover's own offsets and
    rotation are stamped into the gate at query time, so authoring them by hand is
    pointless — whatever you set is overwritten with the unit's real shape. A
    single-cell unit is stamped empty and the gate does nothing.

Connectivity is judged on the **resolved** cells only. Cells that fall off the board
are the fit gate's business, not this one's — which is why the two compose cleanly.

!!! warning "Assemble a profile by hand and you lose both gates"
    The fit injection and the connectivity stamping happen inside GridEntity's
    per-unit movement queries. Code that builds a movement profile itself and calls the
    board's search engine directly gets **neither** gate, and a multi-cell unit will
    happily path to anchors it cannot occupy. The per-unit queries are the supported
    path.

## Can this unit stand here?

The one-call legality check the rules layer should use — for a spawn, a teleport, a
push, or a rotation:

```cpp
// Fit AND connectivity, in one Blueprint-pure call. The rules-layer "may stand here".
static bool CanEntityStandAt(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGridNodeHandle AnchorCell,
    int32 RotationSteps, FGameplayTagContainer SeveringEdgeTags,
    bool bTreatSameColumnLayersAsConnected = true, UGridGraph* Grid = nullptr);

// Its two halves, if you want them separately — both Blueprint pure.
static bool DoesFootprintFit(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGridNodeHandle AnchorCell,
    int32 RotationSteps, UGridGraph* Grid = nullptr);

static bool IsFootprintConnectedAt(
    const UObject* WorldContext, const FPGeEntityRef& Ref, FGridNodeHandle AnchorCell,
    int32 RotationSteps, FGameplayTagContainer SeveringEdgeTags,
    bool bTreatSameColumnLayersAsConnected = true, UGridGraph* Grid = nullptr);
```

The severing-tag and vertical-column arguments mean the same thing here as in the
granted gate, and read connection tags through the live board — so a check and the
movement search that follows it agree by construction. Pass the same values you
granted.

!!! warning "This is a convention, not a guard — call it yourself"
    **The placement calls do not enforce it.** `Rotate Entity In Place`,
    `Move Entity To Cell Oriented`, `Move Entity To Cell` and `Set Entity Footprint`
    write what you tell them to write, and a game that skips the check *can* park a
    2&times;2 golem on an anchor where half of it hangs off the board. That is
    deliberate — a teleport-into-a-crevice mechanic, a shape change under duress, and a
    scripted set-up all need to write placement without asking permission. But it means
    the check is **your** call, at every door a unit can arrive through: spawn,
    teleport, push, pull, swap, and rotate.

    Movement is the exception — a pathfind cannot route a unit somewhere it does not
    fit, because the gates above run inside the search.

=== "Blueprint"
    1. Before rotating, call **Can Entity Stand At** with the unit, its **current**
       anchor cell, and the **new** rotation step.
    2. Branch on the result: on true, call **Rotate Entity In Place**; on false, refuse
       the rotation (grey out the control, play a refusal cue).
    3. Use the same check before a teleport or a push, passing the destination anchor
       and the unit's current step.

=== "C++"
    ```cpp
    // Rotating a multi-cell unit: check, then write.
    FGridNodeHandle Anchor;
    FPGxGridPlacement Placement;
    if (UPGxPlacementLibrary::GetEntityCell(this, Golem, Placement))
    {
        const int32 NewStep = Placement.RotationSteps + 1;

        if (UPGxPlacementLibrary::CanEntityStandAt(
                this, Golem, Placement.Cell, NewStep, SeveringTags))
        {
            UPGxPlacementLibrary::RotateEntityInPlace(this, Golem, NewStep);
        }
    }
    ```

## Occupancy of a multi-cell unit

The [occupancy index](reference.md#occupancy) lists a footprinted unit on **every** cell
it covers, so "who is on this cell?" answers correctly for a golem's far corner. The
reverse question gained a plural answer to match — **Get Cells Of Entity** returns the
whole covered set, and **Is Entity On Cell** answers directly.

**Get Cell Of Entity** is unchanged and still returns the **anchor** placement: the
anchor is the position fact, and footprints do not blur it.

!!! note "C++ only"
    The geometric helpers that compute a footprint's pivot and its local cell-centre
    points are plain C++ statics with no Blueprint nodes. They exist so the token
    transform resolver and arc classification measure from the same point; Blueprint
    reaches everything it needs through the calls on this page.

## See also

- [Facing & Arcs](reference-facing.md) — the rotation step footprints turn on, and what
  "behind a 2&times;2 unit" means.
- [Placement](reference.md#placement) and [Occupancy](reference.md#occupancy) — the
  facet a footprint rides on, and the index that reports it.
- [Movement & Reachability](../gridgraph/reference-movement.md#the-movement-profile) —
  the profile and cost-modifier model both gates are built on.
