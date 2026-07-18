# EventSystem

For developers whose game rules need to *respond* — armor that softens a hit, a
passive that fires when poison lands, a trap that triggers when a unit steps on
it — and who want to know exactly what to broadcast, subscribe to, and call to
get those responses. After this section you'll know the full broadcast/subscribe
API, how ordering is made deterministic, and how a reaction window lets a rule
reshape or answer a change before and after it commits.

EventSystem is the rules-layer counterpart to change events. A **gameplay event**
is a declared moment — "this happened" — that other game rules may act on, sent
on a **channel** and carrying a **payload**. Rules subscribe to events; when one
is broadcast, its subscribers run in a fixed order, and some of them can shape or
veto what happens next. That mental model — the "why", the split between a change
event and a gameplay event, the reasoning behind reaction windows — lives on the
[Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md)
concept page. This section owns the mechanics: the exact nodes and functions, the
channel and ordering rules, the payload, and the windowed-change entry points.

!!! note "Read the concept first"
    If you haven't yet, start with
    [Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md)
    in Core Concepts. It explains why a change event (read-only, for your display)
    and a gameplay event (for rules) must never blur, and what a reaction window
    is. Everything below assumes that model.

## Where it fits in the framework

The event subsystem is a **GameInstance subsystem** — one per running game. It
builds on two lower-level plugins:

- The **[TagEvents](../tagevents/index.md)** dispatch core does the actual
  call-out to a handler: given an event tag and a target object, it finds and
  invokes the handler function for that tag by reflection. EventSystem adds
  channels, ordering, subscription bookkeeping, and reaction windows on top.
- The **[QueueFramework](../queueframework/index.md)** queue is what lets a
  reaction that runs *after* a change commits be scheduled and drained in order,
  rather than firing re-entrantly in the middle of the code that raised the
  event.

The **windowed change entry points** — Apply Stat Change and Apply Tag Change —
ship with the **[GameEntity](../gameentity/reference-stats.md)** plugin, because
they change entity state. But they open their reaction windows on *this* plugin's
machinery: a windowed change is just a broadcast on the target entity's own
channel that brackets the real command around its subscribers. This section
documents those entry points (see the [reference](reference.md#windowed-changes-entity-system)
and the [how-to guide](guides.md#react-to-a-change-with-a-reaction-window)); the
resolved stat and tag state they change is documented with GameEntity.

## Two kinds of listener

The single most important distinction in this plugin is *which kind of listener
you are creating*, because the two can do different things:

- An **observer** is told what happened *after* every rule has run, and can never
  cancel or alter anything. Every Blueprint subscribe node, and the default C++
  subscribe, creates one. This is exactly right for UI, sound, and logging.
- A **rules listener** can reshape or veto what happens. It is *authored onto an
  entity as data*, so who is listening undoes, saves, and replays with the world.
  You don't register one at runtime the way you hook up a widget — the framework
  derives the live subscription from entity state. The full authoring surface for
  entity-carried rules listeners is documented in
  [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#triggers-entity-carried-rules). <!-- pluginlink: abilitysystem-reference-triggers -->

!!! warning "Observers can never affect the outcome"
    An ordinary subscription — Blueprint or C++ — cannot block or reshape a
    change, no matter what order you give it. Observers are dispatched after every
    rule and are locked out of the pre-commit phase by construction. If your
    listener only needs to *know* something happened, that's perfect. If it needs
    to change the outcome, it must be a rules listener. There is one hand-written
    C++ path to a rules listener today, shown in the
    [guides](guides.md#react-to-a-change-with-a-reaction-window); the durable,
    entity-carried surface is documented in
    [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#triggers-entity-carried-rules). <!-- pluginlink: abilitysystem-reference-triggers -->

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPEsEventSubsystem` | The event subsystem — a GameInstance subsystem. Subscribe, query, broadcast bookkeeping, the loop-cap and depth hooks. |
| **Broadcast Event** node / `BeginBroadcastEvent` + `FinishBroadcastEvent` | Send an event on a channel with a payload. Blueprint uses the node; C++ uses the two-step Begin/Finish pair. |
| **Subscribe to Event** node / `UPEsEventSubsystem::Subscribe` | Start listening to an event on a channel at a chosen order. |
| `EPEsBroadcastTiming` | Whether the broadcast resolves depth-first now (`Immediate`) or is queued to run later (`Deferred`). |
| `IPEsEventSubscriberInterface` | Implement it to give a listener a stable id that survives save and replay. |
| The payload (`UPEsEventPayload`) | Carries the event's facts: a typed struct, a bag of tags, and a map of named numbers rules can read and reshape. |
| `EventOrder.*` presets | Named order values (Project Settings), so designers pick "interrupt early" or "react late" by name instead of a raw integer. |
| Apply Stat Change / Apply Tag Change | The windowed-change entry points (on GameEntity). Open a reaction window around a stat or tag change. |

## Where to go next

- **[Guides](guides.md)** — task recipes: broadcasting an event, observing one,
  reacting to a change with a reaction window (the worked armor example), ordering
  reactions with presets, and breaking a runaway loop.
- **[API Reference](reference.md)** — the full public surface, grouped by area:
  subscribing, broadcasting, ordering and channels, the payload, interrupts, the
  loop and depth safety hooks, and the windowed-change entry points.
- **[Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md)**
  — the model and the rationale.
- **[TagEvents](../tagevents/index.md)** and **[QueueFramework](../queueframework/index.md)**
  — the dispatch core and the queue this plugin is built on.
- **[GameEntity stats reference](../gameentity/reference-stats.md)** — the stat and
  tag changes that Apply Stat Change / Apply Tag Change wrap in a window.
