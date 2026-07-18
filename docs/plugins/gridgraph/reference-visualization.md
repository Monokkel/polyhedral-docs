# Visualization API Reference

For developers who want to draw tactical overlays on the board: a blue move-range
fill under a selected unit, a yellow arrow tracing the path to the hovered cell, a
red ring on every cell an ability would hit. This page covers the presenter family
that renders those overlays, the driver that feeds them, and the theme that keeps
them all consistent. For the model behind cells, reachability, and paths see
[Grids & Occupancy](../../concepts/grids-and-occupancy.md); for generating the board
itself see [Graph](reference-graph.md) and [Generation](reference-generation.md);
for where reachability and paths come from see [Movement](reference-movement.md).

The visualization types carry plain `Grid…` names — the GridGraph plugin does not
use a short prefix. Signatures below are hand-written to show the shape of each API;
they are illustrative, not source excerpts. This surface is heavily Blueprint-facing
— presenters and themes are authored on data assets — so the recipes lean Blueprint,
with C++ shown where it clarifies a call.

## How grid visualization works

Overlays are built from three pieces that snap together the same way every time:

- **A set of cells** — a reachable move-range, an ordered path, an ability's target
  cells. Nothing more than which cells to light up, and (for a path) in what order.
- **A presenter** — one reusable unit of visual logic that draws *one thing* from
  that set: a decal fill, an outlined boundary, a row of arrows, a target marker.
- **A theme** — shared color and animation knobs so every presenter in an overlay
  reads as one coordinated highlight.

The **driver component** wires them together at runtime. `UGridVisualizationDriverComponent`
is a scene component you drop on your grid actor. You hand it a set of cells; it runs
the presenters listed on its assigned [profile](#themes-and-profiles) over those cells
and spawns the render components. It is deliberately **input-source-agnostic**: it
draws whatever set you pass and never reaches back for a selection or hover source, so
a game controller, an ability system, or a plain reachability flood all drive it the
same way. A game with no entity system at all can drive it from a hand-built set.

### Three independent channels

The driver keeps three separate channels so overlays coexist without fighting each
other — a target telegraph can sit on top of a reachable-area fill while a path arrow
runs through both:

- **Area** — a region (a move-range, an aura). Changes on *selection*.
- **Path** — an ordered route to a destination. Changes on *hover*.
- **Target** — an ability / area-of-effect target set. Driven on its own channel.

Each channel has a Show / Clear pair and reads-back queries. Showing a channel again
refreshes it in place; clearing one leaves the others untouched.

```cpp
// Configure once (usually in the editor): which grid, which profile of presenters.
UGridGraphComponent* TargetGrid;              // auto-resolves to the grid on this actor if unset
UGridVisualizationProfile* Profile;           // the presenters + theme this driver runs

// --- Area channel: a region highlight, changes on selection ------------------
void ShowArea(const TSet<FGridNodeHandle>& AreaNodes);
void ShowAreaFromReachability(const FGridReachability& Reach); // convenience: reached cells
void ClearArea();
bool HasArea() const;
const TSet<FGridNodeHandle>& GetAreaNodes() const;

// --- Path channel: an ordered route, changes on hover ------------------------
void ShowPath(const TArray<FGridNodeHandle>& PathNodes);   // explicit path, source-first
void SetHoveredCell(FGridNodeHandle Hovered);              // retrace to a cell — no recompute
void ClearPath();
bool HasPath() const;
bool HasReachability() const;                              // true once a reachability is stored
const TArray<FGridNodeHandle>& GetPathNodes() const;
FGridNodeHandle GetHoveredCell() const;

// --- Target channel: an ability / AoE target set -----------------------------
void ShowTargets(const TSet<FGridNodeHandle>& TargetNodes);
void ClearTargets();
bool HasTargets() const;
const TSet<FGridNodeHandle>& GetTargetNodes() const;

UGridGraph* GetGrid() const;                               // resolved grid, or null
```

The typical selection-plus-hover flow uses two of the channels together. Show the
area from a unit's reachability once on selection; then, as the cursor moves, call
`SetHoveredCell` to redraw the path.

!!! tip "Hover is cheap — the driver remembers the reachability"
    `ShowAreaFromReachability` stores the reachability result. Every later
    `SetHoveredCell` retraces the route to that cell from the stored result — it does
    **not** recompute reachability. Hovering across a whole move-range is free. (A
    plain `ShowArea` with a bare cell set stores no reachability, so hover-to-path
    won't work off it; use `ShowAreaFromReachability` when you want hover paths.)
    `ReconstructPath` and reachability are covered in [Movement](reference-movement.md).

=== "Blueprint"
    1. On the grid actor, add a **Grid Visualization Driver** component and set its
       **Profile** to your visualization profile asset.
    2. On select, call **Compute Entity Reachability** for the unit, then
       **Show Area From Reachability** — the move-range lights up.
    3. On cursor move, call **Set Hovered Cell** with the cell under the cursor — the
       path redraws to it with no recompute.
    4. On deselect, call **Clear Area** (and **Clear Path**).

=== "C++"
    ```cpp
    // On select: compute the unit's reach and light the area channel.
    FGridReachability Reach =
        UPGxMovementLibrary::ComputeEntityReachability(this, Unit, Grid, /*MaxCost=*/50);
    Driver->ShowAreaFromReachability(Reach);

    // On hover: retrace the path to the pointed cell (no reachability recompute).
    Driver->SetHoveredCell(HoveredCell);

    // On deselect: the whole highlight goes away.
    Driver->ClearArea();
    Driver->ClearPath();
    ```

The **target** channel is the counterpart the ability system drives to telegraph what
a spell would hit; that wiring is documented in [Painting targets on the grid](../abilitysystem/reference-targeting.md#painting-targets-on-the-grid). Because `ShowTargets`
takes the same plain cell set as `ShowArea`, you can also drive it directly for any
"these cells are targeted" overlay.

## Presenters

A presenter is one reusable, inline-editable unit of visual logic that draws a single
thing from the context handed to it. Every presenter derives from
`UGridVisualizationPresenter` and implements three events. They are `EditInlineNew`
and `Blueprintable`, so you configure them inline on a profile and — because the base
is abstract and instanced — every shipped subclass shows up in the profile's
add-dropdown automatically. Writing a new way to display something is a new subclass
and nothing else; there is no display-method enum to extend anywhere.

```cpp
// Abstract base. Build/Update/Teardown are BlueprintNativeEvents you can override
// in C++ or Blueprint. The driver calls these — you never call them yourself.
class UGridVisualizationPresenter : public UObject   // Abstract, EditInlineNew, Blueprintable
{
    // Draw from scratch against the context, honoring the shared theme.
    void Build(const FGridVisualizationContext& Context, const FGridVisualTheme& Theme);

    // Refresh for a changed context. Default is teardown + rebuild.
    void Update(const FGridVisualizationContext& Context, const FGridVisualTheme& Theme);

    // Drop all rendered state. Called before a rebuild and when the channel clears.
    void Teardown();
};
```

The context is the transient, per-refresh payload a presenter reads. It is never saved
or command-captured — it lives only for the duration of one rebuild.

```cpp
struct FGridVisualizationContext
{
    UGridGraph*      Grid;             // geometry the presenter reads (cell centres, heights)
    USceneComponent* AttachComponent;  // what the spawned render components parent under
    TSet<FGridNodeHandle>    AreaNodes; // the cells for an area / target presenter
    TArray<FGridNodeHandle>  PathNodes; // the ordered route for a path presenter
};
```

Reach for the presenter that matches what you want to draw:

| Want to draw | Presenter | Channel |
|---|---|---|
| A filled move-range on a **sparse** set | **Area – Decal Per Tile** (`UGridAreaDecalPresenter`) | Area |
| A filled range on a **dense** map | **Area – Instanced Mesh Per Tile** (`UGridAreaInstancedMeshPresenter`) | Area |
| Just the **border** of a region | **Area – Outline (Boundary Loops)** (`UGridOutlinePresenter`) | Area |
| Arrows tracing a route | **Path – Arrow Per Tile** (`UGridPathArrowPresenter`) | Path |
| A smooth swept band along a route | **Path – Ribbon (Procedural Mesh)** (`UGridPathRibbonPresenter`) | Path |
| A marker on each targeted cell | **Target – Decal Marker** (`UGridTargetMarkerPresenter`) | Target |

### Area fills — decal vs. instanced mesh

**`UGridAreaDecalPresenter`** paints one decal component per cell in the set. It is
cheap and crisp for the small sets a move-range or an area-of-effect produces, and it
drapes over uneven ground because decals project. Reach for it first for a blue
move-range fill. Key config: `DecalMaterial`, `DecalSize`, `DecalRotation` (default
projects straight down), `SortOrder`, and `HeightBias` (lifts the decal off the floor
to avoid z-fighting).

**`UGridAreaInstancedMeshPresenter`** renders the whole set as instances of one mesh
on a single instanced-static-mesh component — one draw call for the entire region. Use
it when the set is large enough that per-cell decal components would cost too much
(big reachable ranges, dense tactical maps). Key config: `TileMesh`, an optional
themed `OverrideMaterial`, `InstanceScale`, and `HeightBias`.

Both draw the same region; the choice is a performance trade between per-cell decal
components and one batched mesh. They swap freely on a profile.

### Outlines

**`UGridOutlinePresenter`** draws only the *boundary* of the region — one closed
ribbon per edge loop — instead of filling it. It is multi-level correct: stacked
platforms on different levels and disjoint regions each get their own loop, so a
region spread across levels outlines cleanly. Stack it with a fill on the same Area
channel for a filled region with a bright edge, or use it alone for a lighter-weight
highlight. Key config: `OutlineMaterial` (use a two-sided material so the edge reads
from below) and `NormalOffset` (lifts the ribbon off its surface). The ribbon `Width`
comes from the shared theme so it stays consistent with the path ribbon.

### Path arrows and ribbons

**`UGridPathArrowPresenter`** places one oriented arrow per cell along the ordered
path, each pointing toward the destination — the classic tactics "walk this way"
overlay, and what you want for a yellow path arrow. It is tile-based: it reads the raw
ordered path, no curve smoothing. Key config: `ArrowMesh` (authored pointing along
+X), an optional themed `OverrideMaterial`, `InstanceScale`, `HeightBias`, and
`bSkipSourceCell` (drop the first arrow on the cell the unit already stands on, so the
arrows read as "to here").

**`UGridPathRibbonPresenter`** sweeps a flat band along the path's smoothed centerline
as one procedural-mesh strip — a continuous ribbon rather than discrete arrows. Reach
for it when you want a flowing route with an animated stripe. Key config:
`RibbonMaterial`, `NormalOffset`, and `bConformFrameToRampSurface` (off by default;
turn it on for ramp-heavy maps so the strip hugs a sloped surface instead of clipping
— on flat ground it makes no difference). Its `Width` and stripe come from the theme.

Arrow and ribbon are sibling path presenters — swapping one for the other on a
profile's path list is a dropdown change, no code.

!!! note "The stripe animation is deterministic and lives in the material"
    Every material-bearing presenter pushes the theme's color, scroll speed, stripe
    period, and width onto a dynamic material instance and then never ticks. The
    scrolling stripe is driven by the material's own engine time against an
    arc-length coordinate, so it runs at a constant world-space size and speed
    regardless of a path's length or curvature, and it is frame-rate independent and
    replay-safe. No per-frame offset is ever accumulated in code. See
    [Themes & profiles](#themes-and-profiles) for the material contract this relies on.

### Target markers

**`UGridTargetMarkerPresenter`** is the shipped presenter for the Target channel: one
decal marker per target cell, tuned to read *on top of* a reachable-area fill — a red
ring on each cell an ability would strike. It is a decal presenter under the hood, so
it shares the same config (`DecalMaterial`, `DecalSize`, `HeightBias`, and the rest)
and the same theming. Drop it in a profile's target list; the ability system feeds the
Target channel through the driver's `ShowTargets`, as documented in [Painting targets on the grid](../abilitysystem/reference-targeting.md#painting-targets-on-the-grid).

## Themes and profiles

A **profile** is the designer-facing asset that bundles the presenters for a driver
plus the theme they share. `UGridVisualizationProfile` is a data asset with one list
per channel and one theme:

```cpp
class UGridVisualizationProfile : public UDataAsset
{
    // Each list is instanced over the presenter base, so the editor's add-dropdown
    // offers every shipped presenter with no extra wiring. Stack as many as you like.
    TArray<UGridVisualizationPresenter*> AreaVisualizers;   // e.g. a fill + an outline
    TArray<UGridVisualizationPresenter*> PathVisualizers;   // e.g. footstep decals + an arrow
    TArray<UGridVisualizationPresenter*> TargetVisualizers; // e.g. a target marker

    FGridVisualTheme Theme;   // shared knobs every presenter above honors
};
```

The lists are the point: put a decal fill *and* an outline in one area list and both
render for that channel. Each channel is optional — an overlay can fill area-only,
path-only, both, or neither.

The **theme** is the small set of knobs every presenter honors where it makes sense,
so a designer can swap a decal for an outline for a ribbon and keep color and animation
consistent:

```cpp
struct FGridVisualTheme
{
    FLinearColor Color;        // overlay tint
    float ScrollSpeed;         // stripe scroll speed, world units / second (0 = static)
    float StripePeriod;        // world distance one stripe repeat spans
    float Width;               // world-space width of a strip / outline ribbon
};
```

Authoring is once-and-reused: set the theme on the profile, and every material-bearing
presenter on that profile builds a dynamic material instance carrying those values.
That means a themed material has to expose four named parameters for the theme to reach
it. The names are fixed by `FGridVisualThemeParams`, and `UGridVisualThemeLibrary`
provides the helper the presenters route through:

```cpp
// The parameter-name contract a themed base material must expose.
struct FGridVisualThemeParams
{
    static const FName Color;         // vector param  <- Theme.Color
    static const FName ScrollSpeed;   // scalar param  <- Theme.ScrollSpeed
    static const FName StripePeriod;  // scalar param  <- Theme.StripePeriod
    static const FName Width;         // scalar param  <- Theme.Width
};

// Build a themed dynamic material instance from a base material. Presenters call
// this; you rarely call it directly. Returns null if Base is null.
static UMaterialInstanceDynamic* CreateThemedMID(
    UMaterialInterface* Base, UObject* Outer, const FGridVisualTheme& Theme);
```

!!! warning "A themed material must expose the four named parameters"
    The theme reaches a material only through the parameter names above. A material
    that omits one simply ignores that value — an unknown-parameter set is a silent
    no-op — so a partially-authored material degrades gracefully rather than erroring,
    but its color or stripe won't respond to the theme until the parameter exists.
    Author your base material against these names, with a time-driven stripe pan for
    the animated look. A presenter left with no material renders with the engine
    default rather than failing.

## Previewing

**`UGridVisualizationPreviewComponent`** lets you see an overlay in the editor from a
hand-authored cell set, with no gameplay running — the way to build and check a profile
before anything drives it live. Drop it on a grid actor, assign a profile, type the
cells you want lit as coordinates, and press the rebuild button.

```cpp
class UGridVisualizationPreviewComponent : public USceneComponent
{
    UGridGraphComponent*        TargetGrid;       // auto-resolves on this actor if unset
    UGridVisualizationProfile*  Profile;          // the profile to preview
    TArray<FIntVector>          PreviewAreaCoords; // cells to light, as coordinates

    void RebuildPreview();   // CallInEditor — teardown + rebuild from the current coords
    void ClearPreview();     // drop everything this preview spawned
};
```

It runs the profile's **area** channel only, over the coordinates you list (coords not
present in the grid are silently skipped). It is the same input-source-agnostic idea as
the runtime driver — render whatever cells you are handed — just fed by a designer list
instead of live reachability, which makes it a clean harness for iterating on a
profile's look.

There is also an ability-system-driven preview that shows a predicted overlay while
hovering an action; that driver is documented in [Hover previews](../abilitysystem/reference-previews.md#hover-previews).

## Whole-grid visualizers

Separate from the presenter-driven overlays above is a simpler family that renders
*every cell of a static board* — useful for a skill tree, an interactive zone map, or
just seeing the whole grid. These rebuild automatically whenever the grid is
regenerated, rather than being driven by a changing cell set.

`UGridVisualizerComponent` is the base. Attach it to a grid actor; it auto-finds the
grid component and rebuilds itself when the grid changes.

```cpp
class UGridVisualizerComponent : public USceneComponent   // Abstract, Blueprintable
{
    UGridGraphComponent* TargetGrid;   // auto-resolves on this actor if unset
    float HeightBias;                  // lifts every tile off the floor

    void RebuildVisualization();       // override in a subclass
    void ClearVisualization();
    void ForceRebuild();               // CallInEditor button
    void DebugDrawConform();           // CallInEditor: debug lines for nodes + adjacency
};
```

Two concrete subclasses ship, mirroring the two area-fill presenters:

- **`UDecalGridVisualizer`** — one decal per tile. Cheap for small boards (skill trees,
  interactive zones); not recommended past a few hundred tiles. Config: `DecalMaterial`,
  `DecalSize`, `DecalRotation`, `SortOrder`.
- **`UInstancedMeshGridVisualizer`** — every tile as an instance of `TileMesh` on one
  instanced-static-mesh component. The right choice for dense boards (large tactical
  maps, terrain). Config: `TileMesh`, `OverrideMaterial`, `InstanceScale`.

!!! note "Whole-grid vs. driver overlays"
    Use a whole-grid visualizer to render the board itself — the static stage every
    cell of which is always shown. Use the [driver](#how-grid-visualization-works) and
    its presenters for the dynamic, per-moment overlays: a move-range, a path, a target
    telegraph that appear and change as play proceeds. `DebugDrawConform` on the base
    visualizer is also handy while building a level — it draws a marker per node at its
    conformed height and a line per connection, so a terrain-conformed board's holes,
    heights, and adjacency are inspectable in-editor.

For step-by-step recipes see the [Guides](guides.md); for the overall plugin tour see
the [overview](index.md).
