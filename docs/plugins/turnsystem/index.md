# TurnSystem

For developers building a match out of rounds, turns, and phases — deciding who
acts next and driving control from one side to the next — who want to know
exactly what to construct, call, and subclass to get there. After this section
you'll know the whole consumer turn story: the core turn machinery that ships
with entities, and the optional scheduling-and-flow toolkit this plugin layers on
top of it.

The core turn machinery lives **inside the entity system**. A framework-minted
**turn tracker** entity holds the round, turn, and active side; one call
**advances** the turn as a single undo step; and each turn boundary is broadcast
as a gameplay event. Because all of that is part of the [GameEntity](../gameentity/index.md)
plugin, a game can run turns with **nothing else added** — no extra plugin, no
manager to keep in sync.

**TurnSystem** is the optional toolkit that sits on top of that core. It adds the
scheduling and flow pieces you reach for once "advance the turn" isn't enough on
its own: participation helpers, a **scheduler policy** that decides who acts next,
a **projected turn order** for an initiative UI, and a **flow driver** that routes
control between player and AI sides. Every piece is optional — take the toolkit
whole, take one part of it, or skip it and drive the core yourself.

!!! note "Read the concept first"
    If you haven't yet, start with
    [Turns and Scheduling](../../concepts/turns-and-scheduling.md) in Core
    Concepts. It explains why turn state lives on an entity, what happens (and in
    what order) when a turn advances, and how a scheduler policy and the projected
    order work. Everything below assumes that model. This section is where the
    line between the core machinery and this plugin's toolkit is drawn — the
    concept page deliberately hides it; the reference reveals it.

## Where it fits in the framework

The turn machinery is built on three lower plugins, and this toolkit adds a fourth
layer on top of them:

- **[GameEntity](../gameentity/index.md)** owns the turn *core* — the turn
  tracker entity, `InitializeTurnTracker`, `AdvanceTurn` / `AdvanceRound`, and the
  turn state struct. Turn state is entity state, so it undoes, saves, and replays
  exactly like the units on the board. This plugin is a reactive consumer of that
  core; it never replaces it.
- **[EventSystem](../eventsystem/index.md)** carries the turn boundaries. "The
  turn advanced" is broadcast as an ordinary gameplay event on the tracker's
  channel and re-broadcast to the units the advance names, so "at the start of my
  turn" is just an observer on that unit.
- **[CommandSystem](../commandsystem/index.md)** makes an advance one undo step.
  The whole advance — the events that fired, the effects that expired, the indices
  that moved — runs inside a single [transaction](../../concepts/commands-and-undo.md),
  so one **Undo** rewinds the entire turn.

This plugin adds the scheduling toolkit — the scheduler subsystem, the policies,
and the flow driver — over that stack.

## The framework supplies mechanism; the meaning stays yours

The turn machinery cycles opaque tags and hands you the seams. It never decides
what a "side" or a "phase" *means*, who wins, or when an advance is allowed —
those are your game's. The division of labor:

| The framework supplies | Your game supplies |
|---|---|
| The turn tracker, the advance, and the one-undo-step guarantee | What a side or a phase *means* (opaque tags you define) |
| The turn-boundary events and the fan-out to named units | Which units act, and who wins or loses |
| A scheduler policy seam + one shipped preset | Action budgets — how many moves a turn grants |
| The projected order, computed on demand | Whether an advance is allowed (your end-turn gate) |
| A flow driver that routes control by a data binding | The AI pump — the framework never runs it for you |

!!! tip "Gate before advancing; don't try to veto after"
    A turn event is a notification, not a proposal — marking one "interrupted" is a
    dev-warned no-op. A rule like "you can't end your turn while burning" belongs
    *before* the advance call, in your own end-turn gate. Decide whether the turn
    may advance; then advance it.

## The pieces at a glance

| Piece | Plugin | What it is |
|---|---|---|
| `FPGeTurnState` | GameEntity | The turn tracker's indices: round, turn-in-round, a monotonic serial, phase, active side. Ordinary entity data. |
| `InitializeTurnTracker` / `AdvanceTurn` / `AdvanceRound` | GameEntity | The turn core: mint the tracker, then advance it as one undo step. Usable with no other plugin. |
| `FPGeTurnFanOut` | GameEntity | The set of units an advance names, so the turn events reach the right entities' own channels. |
| `FPTsSchedulerPolicy` | TurnSystem | The open extension point: a pure rule from participants + turn state to who acts next. Subclass it for any ordering. |
| `FPTsSchedulerPolicy_Igougo` | TurnSystem | The one shipped preset — side-based (sometimes called I-go-you-go). Configure it with an ordered list of side tags. |
| `UPTsSchedulerSubsystem` | TurnSystem | Participation helpers (`JoinTurnOrder` / `LeaveTurnOrder`), the active policy, and the projected order. A GameInstance subsystem. |
| `UPTsTurnFlowDriver` | TurnSystem | The optional match conductor: start the match, advance, and route control between player and AI sides. A constructed object you hold, **not** a subsystem. |
| `PTsTags` | TurnSystem | The tagged-data keys and the two canonical controller tags (player / AI). |

## Where to go next

- **[Guides](guides.md)** — task recipes: starting a match and advancing turns,
  joining and leaving the turn order, showing an initiative strip, writing your own
  scheduler policy, and driving turns with no toolkit at all.
- **[API Reference](reference.md)** — the full public surface, grouped by area:
  the turn tracker and the advance, scheduler policies, the scheduler subsystem,
  the flow driver, and the tags.
- **[Turns and Scheduling](../../concepts/turns-and-scheduling.md)** — the model
  and the rationale.
- **[GameEntity](../gameentity/index.md)** — the entity system the turn core lives
  in, and where turn state gets its undo, save, and replay guarantees.
