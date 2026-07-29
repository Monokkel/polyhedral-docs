# Polyhedral Core

For developers installing the framework or wiring a C++ module against it. After
this page you'll know which four systems ship inside the **PolyhedralCore** plugin,
where each one's documentation lives, and the one naming rule that trips up almost
every first build.

PolyhedralCore is a single plugin hosting four independent systems as four separate
**modules**. Nothing in the plugin depends on anything else in the framework — it is
the bottom of the stack, and everything else is built on top of it.

## The four modules

| Module | Section | What it gives you |
|---|---|---|
| `CommandSystem` | **[CommandSystem](../commandsystem/index.md)** | The undo/redo/replay command stack: every authoritative change goes through one door, so the whole game rewinds. |
| `QueueFramework` | **[QueueFramework](../queueframework/index.md)** | Ordered, channel-based processing of work items — some instant, some spanning real time. |
| `TagEventsRuntime` | **[TagEvents](../tagevents/index.md)** | Tag-keyed dispatch: call a handler on an object by gameplay tag, with no delegate wiring. |
| `NoiseBasedRandomSeed` | **[NoiseBasedRandomSeed](../noisebasedrandomseed/index.md)** | Deterministic, save- and replay-safe randomness keyed by a domain tag. |

Each keeps its own section in these docs, with its own overview, guides, and API
reference. This page only covers what they share: the folder they ship in.

Two of them also ship an editor-only companion module — `QueueFrameworkEditor` and
`TagEventsEditor` — which provide the Blueprint nodes those systems add to the
palette. You never reference those from game code; enabling the plugin is enough.

## Why one plugin

The four systems were separate plugins as an accident of the order they were
written in, not because the split meant anything. None of them depends on anything,
none of them knows the others exist, and every consumer in the framework already
referenced them one module at a time. Merging the folders changed the *installation*
story and nothing else: module names, `#include` paths, `.Build.cs` dependency
strings, type names, and prefixes are all identical to what they were as four
plugins.

So: one folder to copy, one checkbox to tick, four systems to use.

## The one rule: plugins and modules use different names

There are two dependency lists in an Unreal project and they do **not** take the
same names.

- Your **`.uplugin`** names *plugins*. The name is `PolyhedralCore` — once, no
  matter how many of the four systems you use.
- Your **`.Build.cs`** names *modules*. The names are `CommandSystem`,
  `QueueFramework`, `TagEventsRuntime`, and `NoiseBasedRandomSeed` — list only the
  ones you actually call.

```json
// In YourPlugin.uplugin (or your .uproject) — plugin names.
"Plugins": [
    { "Name": "PolyhedralCore", "Enabled": true }
]
```

```csharp
// In YourModule.Build.cs — module names.
PublicDependencyModuleNames.AddRange(new string[]
{
    "CommandSystem",         // from the PolyhedralCore plugin
    "NoiseBasedRandomSeed",  // also from PolyhedralCore
});
```

!!! warning "`PolyhedralCore` is not a module, and `CommandSystem` is not a plugin"
    Putting `PolyhedralCore` in a `.Build.cs` fails to resolve; putting
    `CommandSystem` in a `.uplugin` fails to find a plugin by that name. This is the
    most common first-build failure against the framework. The
    [installation page](../../getting-started/installation.md#depending-on-the-framework-from-c)
    shows the same pair side by side.

A Blueprint-only project can ignore all of this — tick **Polyhedral Core** in
**Edit → Plugins** and every node from all four systems is in the palette.

## Load order is already handled

The command stack module loads earlier than the rest of the plugin, so anything
that wants the stack during startup finds it already there. That's declared inside
the plugin; there is nothing for you to configure, and no ordering constraint on
where you put `PolyhedralCore` in your own dependency lists.

## Where to go next

- **[Installation](../../getting-started/installation.md)** — copying the folders,
  ticking the plugins, and the engine plugins that go with them.
- **[Commands & Undo](../../concepts/commands-and-undo.md)** — the model the command
  stack exists to serve, and the best place to start reading.
- **[Plugins](../index.md)** — the rest of the framework, all of which is built on
  this plugin.
