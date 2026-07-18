# Installation

For a developer adding the framework to an Unreal Engine 5.8 project for the
first time. After this page you'll have the four Phase 1 plugins enabled, a
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

## The four Phase 1 plugins

The starting set is four plugins. Enable **all four** — they build on each
other, and the one you'll use most (GameEntity) depends on the other three.

| Plugin | Prefix | What it gives you |
|---|---|---|
| **TaggedData** | `PTa` | Typed structs keyed by gameplay tags, on any object |
| **CommandSystem** | `PCs` | The undo/redo/replay command stack |
| **Evaluators** | `Eval` | Data-driven number and condition calculations |
| **GameEntity** | `PGe` | Struct-based game entities, stats, templates, hierarchy |

!!! note "Enable the whole set, in any order"
    GameEntity depends on TaggedData, CommandSystem, and Evaluators, so enabling
    GameEntity alone will fail its dependency check. Enable the four together and
    the dependency order sorts itself out — you don't have to sequence them by
    hand.

## Add the plugins to your project

1. Copy the four plugin folders into your project's `Plugins/` directory
   (create it next to your `.uproject` if it doesn't exist yet):

    ```text
    YourProject/
      Plugins/
        TaggedData/
        CommandSystem/
        Evaluators/
        GameEntity/
      YourProject.uproject
    ```

2. Enable them. Either edit the `.uproject` directly, adding each to the
   `"Plugins"` array with `"Enabled": true`, or open the editor and tick each
   one in **Edit → Plugins**, then restart when prompted.

3. Regenerate project files (right-click the `.uproject` → **Generate Visual
   Studio project files**, or the equivalent for your IDE). This is only needed
   for a C++ project.

## Build the editor target

Build your project's **editor** target the way you normally do — from your IDE,
or by letting the editor compile the plugins on first launch. A Blueprint-only
project can skip straight to opening the editor; it will compile the plugin
modules for you.

!!! warning "Build with the editor closed"
    If you compile from your IDE while the editor is open with Live Coding,
    link steps can fail on locked binaries. Close the editor first, build, then
    reopen it.

## One-time setup: the tagged-data schema

TaggedData reads a **schema** — a data asset that maps each `Data.*` gameplay
tag to the struct type stored under it. The schema powers two things: optional
validation when data is set, and the typed Blueprint pins on the custom
get/set nodes (with a safe fallback to a generic struct container when a tag
has no schema entry).

At a high level:

1. Create a **Tagged Data Schema** data asset. Add one entry per tag you'll
   store, each mapping a `Data.*` tag to its struct type.
2. Open **Project Settings → Plugins → Tagged Data** and add your schema asset
   to the **Schema Assets** list. This is stored in your project config, so it
   travels with the project.
3. Leave schema enforcement on while you author (it catches type mistakes at the
   moment you set data); you can relax it later if you need to.

You can start with an empty schema and add entries as the tutorial introduces
them — the typed nodes fall back gracefully until a tag is listed.

<!-- pluginlink: taggeddata-reference -->

## Verify it worked

You're set up correctly when:

- The **editor target builds** without errors.
- In the editor, a Blueprint graph offers the framework's nodes — search for
  **Get Game State Subsystem** and **Get Command Stack**; both should appear.
- Adding a typed **Set Tagged Data** node and picking a tag that's in your
  schema gives you a real struct pin rather than a generic container.

If all three hold, the plugins are live and wired together.

## Next

Head to **[Build Your First Board](first-board.md)** to author entities, spawn
them, store tagged data, and make a command-driven change with a working Undo —
the whole core loop in one sitting. If you'd rather read the model first, start
with **[Entities](../concepts/entities-as-data.md)** in Core Concepts.
