# TokenSystem

For developers putting the game on screen — minis on a board, card faces, health
bars, character sheets — and want each one to stay correct through undo, save, and
replay without wiring the bookkeeping themselves. After this section you'll know
how every entity gets an on-screen stand-in for free, how to layer animated
playout on top with cues and handlers, and how one entity can drive many surfaces
at once.

TokenSystem is the framework's **presentation layer**. It keeps an on-screen
**token** for every entity and *snaps* it to the truth on every change — create,
destroy, move, load, undo — so the board is already right with no presentation
code written. On top of that automatic snapping you layer **cues** and **cue
handlers** for sequenced, animated playout. The whole plugin rests on one rule:
presentation reads game state and is driven by it, but never changes it.

!!! note "Read the concept first"
    If you haven't yet, start with
    [Tokens & Cues](../../concepts/tokens-and-cues.md) in Core Concepts. It owns
    the "why" — the screen as a projection of state, the two-tier "snap is free /
    cues are opt-in" model, and why a presentation surface must never write. This
    section owns the "how": the exact types, the lifecycle, and the tasks you'll
    perform.

## Where it fits in the framework

The token registry is a **world subsystem** — one per running play world. It
builds on two plugins below it:

- **[GameEntity](../gameentity/index.md) change events.** The registry subscribes
  to the entity change-event feed and reconciles the whole set of tokens for you:
  spawn on create, release on destroy, re-snap every token on load and undo. You
  never write that bookkeeping. See the
  [change-event reference](../gameentity/reference-events.md).
- **[QueueFramework](../queueframework/index.md).** Animated cues sequence on a
  queue channel, so a run of visuals plays out cleanly one after another instead
  of all firing on the same frame.

A token holds no game state, and an entity holds no pointer to a token. Because
presentation writes nothing, an undo, a reload, or a replay with a completely
different set of panels open all land on byte-identical game state.

!!! tip "Presentation surfaces don't subscribe"
    A game-facing token or readout should never bind its own change-event
    subscriptions — the registry drives every surface. A surface that subscribed
    on its own would snap to the committed value at the moment of the command and
    skip the animation, and it would fight the registry during undo. This is the
    presentation-side sharpening of the
    [derived-state subscription rule](../../concepts/derived-state.md): reconcile
    a *tool* to what state now is, but let the registry drive a *game-facing*
    surface.

## The honest two-tier model

Hold the plugin in your head as two tiers:

- **Always correct, for free.** Assign a token class to an entity's template and
  the registry does the rest — it spawns the token, positions it, and snaps it to
  the truth on every change including undo and load. With no cues written at all,
  the board is right.
- **Animated playout, opt-in.** On top of the snap you play **cues** — semantic
  "what happened" messages that handlers turn into montages, walks, and number
  tweens. Snapping is automatic; cues are polish that somebody plays.

Today you play cues yourself, wherever you want the extra beat. The next step —
turning committed state changes into cues *automatically*, choosing snap-versus-
sequence, and fanning previews out on hover — is work the ability system does for
you, documented in a later section. Nothing you build against cues now changes
when that layer arrives; it drives the same cues, handlers, and capabilities.

!!! note "The grid-aware token lives in GridEntity"
    A unit on a board needs its token positioned on the correct cell and walked
    along a path when it moves. That grid-aware token — the one seam where the
    grid and the entity systems meet — ships in the **GridEntity** plugin and is
    documented in its own section. TokenSystem itself stays grid-agnostic: it
    positions a token through a resolver that GridEntity supplies, so the registry
    works the same whether the game is gridded or off-grid.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPTkTokenSubsystem` | The **token registry** — a world subsystem. Spawns, releases, and snaps one token per entity; plays cues; owns the cue-handler registry. |
| `IPTkTokenInterface` | The tiny interface anything on screen implements — bind, snap, release. An actor, a widget, or a plain object can all be tokens. |
| `APTkTokenActor` / `UPTkTokenWidget` | Convenience bases that pre-implement the interface for a 3D actor or a UMG widget, so you override events instead of plumbing. |
| `FPTkCue` | A semantic "what happened" message addressed to one entity's visuals — an intent tag, an optional instigator, a typed payload. |
| `UPTkCueContext` | The async-completion handle a cue carries — a handler calls **Complete** when its visual finishes so the sequence moves on. |
| `UPTkCueHandler` | A reusable, tag-keyed unit of visual logic that renders one cue against a token through the token's capabilities. |
| Capabilities (`IPTkMovable`, `IPTkMontageCapable`, `IPTkStatDisplay`, `IPTkPreviewable`, `IPTkGhostable`) | Small opt-in interfaces a token advertises so a handler can drive it without knowing its concrete class. |
| `UPTkEntityWidget` | A readout bound to one entity and one value — the building block for HUD party frames and character sheets. |
| `UPTkTokenSettings` | Project settings: the per-category and default fallback token classes. |

## Where to go next

- **[Guides](guides.md)** — task recipes: giving an entity a token, writing a cue
  handler, driving a token through a capability, playing a cue yourself, and
  binding an entity widget to one value.
- **[API Reference](reference.md)** — the full public surface, grouped by area,
  with clean signatures and short examples.
- **[Tokens & Cues](../../concepts/tokens-and-cues.md)** — the model and the
  reasoning behind the two tiers.
- **[QueueFramework](../queueframework/index.md)** — the sequencing primitive that
  animated cues play out on.
- **[GameEntity change events](../gameentity/reference-events.md)** — the feed the
  registry reconciles every token against.
