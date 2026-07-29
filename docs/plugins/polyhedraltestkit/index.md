# PolyhedralTestKit

For developers writing their own automation tests against the framework — asserting
that a stat change undoes, that an ability resolves, that a turn advances. After this
page you'll know how to stand up a headless game instance and world in three lines,
and why that scaffold is worth a plugin of its own.

The whole public surface is one type: **`FPTkTestGameInstance`**, a scope that boots a
standalone `UGameInstance` and `UWorld` in its constructor and tears them down in its
destructor. Every framework subsystem — game state, the command stack, the queue, the
event bus, the scheduler — comes up with it, so a test can drive the real systems
without a running game.

!!! note "C++ only"
    This is a C++ test-scaffolding type with no Blueprint surface, and it exists only
    in builds with automation tests compiled in. There are no nodes to look for in the
    palette.

## Three lines to a live framework

```cpp
#include "PTkTestGameInstance.h"

FPTkTestGameInstance Kit;
UPGeGameStateSubsystem* State = Kit.Subsystem<UPGeGameStateSubsystem>();
UPCsCommandStack*       Stack = Kit.Subsystem<UPCsCommandStack>();
```

That's the whole setup. `State` and `Stack` are the same subsystems your game talks
to at runtime, so a test creates entities, submits commands, and calls Undo exactly
the way gameplay code does. When `Kit` goes out of scope the world is destroyed.

## Why it exists

Standing this up by hand is deceptively hard, and two details are the reason:

- **A game-instance subsystem must be constructed with a real game instance as its
  outer.** Subsystems declare a `ClassWithin`, so creating one against the transient
  package doesn't work — you need an actual `UGameInstance`, initialized standalone,
  before you can resolve anything off it.
- **Teardown order matters.** Shutting the game instance down *before* destroying the
  world context and the world is the order that doesn't race the engine's own
  teardown. Get it backwards and you get intermittent crashes at the end of a test
  run, which is a miserable thing to debug.

That is the single hardest-to-rediscover piece of framework test setup, and this
plugin exists so nobody has to rediscover it. The framework's own suites all use it.

## The surface

| Member | What it does |
|---|---|
| `FPTkTestGameInstance()` | Boots a standalone game instance and world. Non-copyable. |
| `~FPTkTestGameInstance()` | Tears both down in the safe order. |
| `bool IsValid() const` | True once the scaffold stood up. Assert on this first. |
| `template <typename T> T* Subsystem() const` | Resolves a **game-instance** subsystem — game state, the command stack, the event bus. Returns `nullptr` rather than crashing. |
| `template <typename T> T* WorldSubsystem() const` | Resolves a **world** subsystem — the queue, for instance. |
| `GameInstance` / `World` | The game instance (held by a strong pointer, so it isn't collected mid-test) and the world, if you need them directly. |

A test that needs several subsystems usually wraps the scope in a small struct of its
own, caching the pointers it uses and adding an `IsValid()` that checks them:

```cpp
struct FMyTestScope
{
    FPTkTestGameInstance    Kit;
    UPGeGameStateSubsystem* State = nullptr;
    UPCsCommandStack*       Stack = nullptr;

    FMyTestScope()
    {
        State = Kit.Subsystem<UPGeGameStateSubsystem>();
        Stack = Kit.Subsystem<UPCsCommandStack>();
    }

    bool IsValid() const { return Kit.IsValid() && State && Stack; }
};
```

Deriving from `FPTkTestGameInstance` instead of embedding it works equally well, and
is the shape to reach for once your helper grows domain methods of its own.

## Wiring it up

Add the plugin to your `.uplugin` (or `.uproject`) and the module to the `.Build.cs`
of whichever module holds your tests:

```json
"Plugins": [
    { "Name": "PolyhedralTestKit", "Enabled": true }
]
```

```csharp
PrivateDependencyModuleNames.Add("PolyhedralTestKit");
```

The header's contents are compiled only when automation tests are, so guard your
usage the same way the framework's suites do:

```cpp
#if WITH_DEV_AUTOMATION_TESTS
// ... your tests and any helper scope built on FPTkTestGameInstance
#endif
```

!!! warning "This world never runs BeginPlay"
    The scope gives you initialized subsystems, not a play session. Actors spawned
    into its world are not initialized for play and never tick, so test the rules
    layer here — state, commands, events, turns — and leave anything that depends on
    the actor lifecycle to a real play-in-editor session.

## What it costs you

The plugin is declared as a **runtime** module, so it is present in packaged builds.
In practice it compiles to essentially nothing outside automation-test builds — the
one type it exports is entirely behind the automation-test guard — but the module
itself is real and it does ship.

It also **cannot be omitted**: seven of the framework's runtime plugins list it as a
dependency, so its folder has to be installed and the plugin enabled even in a
project that never writes a test.

## Where to go next

- **[Installation](../../getting-started/installation.md)** — copying folders and
  enabling plugins, including this one.
- **[CommandSystem](../commandsystem/index.md)** — the command stack a test drives to
  assert undo behaviour, and its conformance harness for custom commands.
- **[GameEntity](../gameentity/index.md)** — the game state subsystem most tests
  resolve first.
