# Installation

For a developer adding the framework to an Unreal Engine 5.8 project for the
first time. After this page you'll have the plugins installed and enabled, a
tagged-data schema wired up, and a project that compiles — ready for the
[first-board tutorial](first-board.md).

## Prerequisites

- **Unreal Engine 5.8.**
- A project you can build. A C++ project (or the ability to generate one) lets
  you follow the C++ examples; a Blueprint-only project works too, since every
  system exposes Blueprint nodes.
- The standard C++ toolchain for your platform if you plan to compile the editor
  yourself (Visual Studio on Windows, Xcode on macOS). Blueprint-only users can
  let the editor compile plugins on launch.

## Copy the plugin folders

Copy the plugin folders into your project's `Plugins/` directory (create it next
to your `.uproject` if it doesn't exist yet):

```text
YourProject/
  Plugins/
    PolyhedralCore/
    TaggedData/
    Evaluators/
    GameEntity/
    ...
  YourProject.uproject
```

!!! tip "Copy them all, enable what you need"
    The simplest correct approach is to copy **every** plugin folder and then tick
    only the ones your game uses. Unreal resolves each plugin's dependencies for
    you, so enabling `AbilitySystem` quietly brings up everything beneath it. If
    you copy only a subset, use the table below to make sure you have each
    plugin's dependencies on disk — a missing folder is a startup error, not a
    silent degradation.

## The plugins

### Start here: Polyhedral Core

**PolyhedralCore** is a single plugin that hosts four foundational systems as
separate modules:

| Module | What it gives you |
|---|---|
| `CommandSystem` (`PCs`) | The undo/redo/replay command stack |
| `QueueFramework` | Ordered, channel-based processing of async work items |
| `TagEventsRuntime` | Tag-keyed dispatch: call a handler on an object by gameplay tag |
| `NoiseBasedRandomSeed` | Deterministic, replay-safe keyed and sequential RNG |

They live in one plugin because none of them depends on anything — they are the
bottom of the stack, and shipping them as four separate plugins was an accident
of the order they were written in. Each still has its own module, its own
`#include` paths, and its own section in these docs; only the *packaging* is
shared.

!!! note "One folder to copy, four systems to use"
    You enable **PolyhedralCore**. In C++ you still depend on the individual
    modules — see [Depending on the framework from C++](#depending-on-the-framework-from-c)
    below.

### Everything else

| Plugin | Prefix | What it gives you | Needs |
|---|---|---|---|
| **PolyhedralCore** | `PCs` and others | Command stack, queues, tag dispatch, deterministic RNG | — |
| **TaggedData** | `PTa` | The tagged-data schema, typed Blueprint pins, primitive wrappers | — |
| **Evaluators** | `Eval` | Data-driven number and condition calculations | — |
| **GridGraph** | — | Square / hex / freeform boards, generation, movement, visualization | — |
| **PolyhedralTooltips** | `PTt` | Inline keyword tooltips and display-data conventions | — |
| **PolyhedralTestKit** | `PTk` | A headless world for your own automation tests | — |
| **TagDebug** | `PDb` | A live, tag-driven debug console | PolyhedralCore |
| **EventSystem** | `PEs` | Gameplay events, ordering, and reaction windows | PolyhedralCore, TaggedData |
| **GridCommands** | `PGc` | Structural edits to the board that ride the command stack | GridGraph, PolyhedralCore |
| **GameEntity** | `PGe` | Struct-based entities, templates, stats and modifiers, save/replay | TaggedData, PolyhedralCore, Evaluators, EventSystem |
| **TokenSystem** | `PTk` | The presentation layer: tokens, cues, projected actions | GameEntity, PolyhedralCore, TaggedData |
| **TurnSystem** | `PTs` | Scheduler policies, projected turn order, turn-flow driver | GameEntity, EventSystem, PolyhedralCore |
| **GridEntity** | `PGx` | Where the grid and entity systems meet: placement, occupancy, facing | GameEntity, GridGraph, TokenSystem, GridCommands, and their dependencies |
| **AbilitySystem** | `PAb` | The capstone rules engine: ability programs, previews, enemy AI | GameEntity, GridEntity, TokenSystem, and their dependencies |

The **Needs** column lists framework plugins only; each also brings its own
dependencies transitively. `PolyhedralTestKit` is additionally required by
GameEntity, EventSystem, TokenSystem, TurnSystem, GridEntity, GridCommands, and
AbilitySystem, so keep its folder installed even if you never write a test.

!!! warning "PolyhedralTestKit is a runtime plugin today"
    It is test scaffolding, but it is declared as a runtime module, so it is
    present in packaged builds. It compiles to essentially nothing outside
    automation-test builds, and you cannot omit it — seven runtime plugins list it
    as a dependency.

### Engine plugins to tick

Three of these ship with the engine and must be **enabled**, not copied. Unreal
will usually prompt you, but it is quicker to tick them up front:

| Engine plugin | Needed by |
|---|---|
| **Data Registry** | GameEntity |
| **Procedural Mesh Component** | GridGraph (the path-ribbon presenter) |
| **Gameplay Tags Editor** | TaggedData, Evaluators, GameEntity (tag pickers) |

`Data Validation` is also used by GridGraph's editor tooling.

## Enable them

Either edit the `.uproject` directly, adding each plugin to the `"Plugins"` array
with `"Enabled": true`, or open the editor and tick each one in
**Edit → Plugins**, then restart when prompted.

You only need to list the plugins you actually use. Enabling `GameEntity` pulls
`TaggedData`, `Evaluators`, `EventSystem`, `PolyhedralCore`, and
`PolyhedralTestKit` in behind it — they don't need their own entries.

Then regenerate project files (right-click the `.uproject` → **Generate Visual
Studio project files**, or the equivalent for your IDE). This is only needed for
a C++ project.

## Build the editor target

Build your project's **editor** target the way you normally do — from your IDE,
or by letting the editor compile the plugins on first launch. A Blueprint-only
project can skip straight to opening the editor; it will compile the plugin
modules for you.

!!! warning "Build with the editor closed"
    If you compile from your IDE while the editor is open with Live Coding,
    link steps can fail on locked binaries. Close the editor first, build, then
    reopen it.

## Depending on the framework from C++

There are two different lists, and they use different names:

- Your **`.uplugin`** names *plugins* — so the command stack is `PolyhedralCore`.
- Your **`.Build.cs`** names *modules* — so the command stack is `CommandSystem`.

```csharp
// In YourModule.Build.cs — module names, not plugin names.
PublicDependencyModuleNames.AddRange(new string[]
{
    "CommandSystem",        // from the PolyhedralCore plugin
    "NoiseBasedRandomSeed", // also from PolyhedralCore
    "GameEntity",
    "TaggedData",
});
```

```json
// In YourPlugin.uplugin — plugin names.
"Plugins": [
    { "Name": "PolyhedralCore", "Enabled": true },
    { "Name": "GameEntity",     "Enabled": true }
]
```

Getting this backwards is the most common first-build failure: `PolyhedralCore`
is not a module, and `CommandSystem` is not a plugin.

## One-time setup: the tagged-data schema

TaggedData reads a **schema** — a data asset that maps each `Data.*` gameplay
tag to the struct type stored under it. The schema powers two things: a type
check when data is written, and the typed Blueprint pins on the get/set nodes
(with a safe fallback to a generic struct container when a tag has no entry).

At a high level:

1. Create a **Tagged Data Schema** data asset. Add one entry per tag you'll
   store, each mapping a `Data.*` tag to its struct type.
2. Open **Project Settings → Plugins → Tagged Data** and add your schema asset
   to the **Schema Assets** list. This is stored in your project config, so it
   travels with the project.
3. Leave schema enforcement on while you author (it catches type mistakes at the
   moment you write data); you can relax it later if you need to.

You can start with an empty schema and add entries as the tutorial introduces
them — the typed nodes fall back gracefully until a tag is listed.

Full schema reference: [TaggedData API Reference](../plugins/taggeddata/reference.md#schema).

## Verify it worked

You're set up correctly when:

- The **editor target builds** without errors.
- In the editor, a Blueprint graph offers the framework's nodes — search for
  **Get Game State Subsystem** and **Get Command Stack**; both should appear.
- Adding a typed entity **Set Tagged Data** node and picking a tag that's in your
  schema gives you a real struct pin rather than a generic container.

If all three hold, the plugins are live and wired together.

## Next

Head to **[Build Your First Board](first-board.md)** to author entities, spawn
them, store tagged data, and make a command-driven change with a working Undo —
the whole core loop in one sitting. If you'd rather read the model first, start
with **[Entities](../concepts/entities-as-data.md)** in Core Concepts.
