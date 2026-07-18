# TagDebug

For developers and designers who want to flip a debug flag, tweak a tuning number, or
fire a one-shot debug action at runtime — each keyed by a gameplay tag, with no bespoke
UI to build and no delegates to wire. After this section you'll know how a debug tag is
shaped, how gameplay code reads one, and how a Blueprint reacts the moment one changes.

A **debug tag** is a named knob. You author a gameplay tag like
`Debug.Float.GameFeel.MoveSpeed` or `Debug.Bool.ShowHiddenObjects`, and TagDebug gives
you three ways to drive it — a Blueprint/C++ setter, a console command, or a hand-edited
config file — plus two ways to consume it: read its current value to branch on, or bind
a live Blueprint event that fires whenever it's set. Types in this plugin carry the `PDb`
prefix.

!!! note "A specialization of TagEvents, not a general event system"
    TagDebug is a thin, debug-flavoured specialization of the **[TagEvents](../tagevents/index.md)**
    dispatch core — that's what turns "a debug value was set" into "run this Blueprint
    handler." For gameplay events between systems, reach for the
    **[EventSystem](../eventsystem/index.md)** instead; TagDebug is for developer-facing
    knobs and switches, not game rules.

## Where it fits in the framework

TagDebug is a small, self-contained plugin that depends only on TagEvents. It gives a
project a uniform, tag-keyed surface for the throwaway toggles every game accumulates —
"show collision", "force this loot roll", "slow-mo the camera" — so they live in one
namespace, survive between runs when you want them to, and never need a custom widget.

- **Readers** branch on a value: any Blueprint or C++ calls a `Check…`/`Get…` function
  to ask "is this flag on?" or "what's this tuning number?".
- **Handlers** react to a change: a Blueprint subclass of the debug component wires a
  **Debug Tag Event** node so setting a tag runs live logic — teleport the player, dump
  state, spawn a marker.

## How it works

Everything meets at a `Debug.*` gameplay tag:

1. **You author the tag.** Its second segment names the value type —
   `Debug.Bool.*`, `Debug.Int.*`, `Debug.Float.*`, `Debug.Vector.*`. A tag with no type
   segment (for example `Debug.Network.KickPlayer`) is a **single-shot action** that
   carries no stored value.
2. **You drive it.** Call a setter from Blueprint or C++, type the tag as a console
   command, or edit the persisted config file by hand. A setter takes a `bPersistent`
   flag: leave it off and the value is session-only; turn it on and it's written to
   `Saved/Config/TagDebugValues.ini` and survives a restart.
3. **Consumers see it.** Readers get the new value on their next `Check…` call. Handlers
   fire immediately: the subsystem lazily spawns your debug component on the Game State
   and dispatches the change to the matching **Debug Tag Event** node, handing it the
   new value on a type-matched output pin.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPDbTagDebugFunctionLibrary` | The static front door — read a debug value, set one, or fire an action, all by tag, from Blueprint or C++. |
| `UPDbTagDebugSubsystem` | The game-instance subsystem the library wraps: value storage, config persistence, and event dispatch. |
| `UPDbDebugComponent` | A Blueprint-subclassable component, spawned lazily on the Game State, that hosts your debug event handlers. It has no callable API of its own. |
| **Debug Tag Event** node | The editor-only Blueprint node that declares a handler bound to a `Debug.*` tag, with a value output pin typed to match the tag. |
| `UPDbTagDebugSettings` | The Project Settings entry that names which `UPDbDebugComponent` subclass to spawn. |

## Where to go next

- **[API Reference](reference.md)** — the tag schema, the read/write functions, the
  Debug Tag Event node and its pin table, the subsystem, settings, and types.
- **[TagEvents](../tagevents/index.md)** — the reflection-dispatch core the Debug Tag
  Event node is built on.
