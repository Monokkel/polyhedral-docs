# QueueFramework

For developers who need gameplay work to happen *in order, one at a time* — a move
animation before the hit reaction before the death, some steps instant and some that
take real time — without hand-rolling a state machine for it. After this section
you'll know how to put work on a channel, how an item signals it's done, and how
nested work resolves depth-first.

A **queue** here is a named **channel** that runs **items** in order. Each item is a
small object with a body you write: a synchronous one completes instantly, an
asynchronous one runs for as long as it needs and calls **Proceed** when it's
finished. The channel never starts the next item until the current one proceeds — so
a sequence of animated steps plays out cleanly instead of all firing on the same
frame.

!!! note "A sequencing primitive, not a gameplay system"
    QueueFramework knows nothing about entities, commands, or rules — it just runs
    work items in order. It's the plumbing other systems sequence *on*: the event
    system uses it to defer a broadcast until the current work finishes, and the
    presentation layer uses it to play animated [cues](../../concepts/tokens-and-cues.md)
    one after another. You can also use it directly for any ordered async work of
    your own.

## The model

- **Channels** are named (`FName`). Items queued on the same channel run in order;
  different channels run independently. Three channel names ship as well-known
  constants — `Order`, `Action`, and `Event` — but a channel is just a name, so you
  can make your own.
- **Items** derive from `UQueuedItem` (Blueprint or C++). Override **Run On Queue**
  for instant work, or **Run On Queue Async** for work that spans time — and in the
  async case, call **Proceed** yourself when it's done.
- **Proceed gates the channel.** The next item waits until the current one proceeds.
  That single rule is what turns a pile of effects into an ordered playout.

## Depth-first nesting

While an item is running it can queue *more* items that should resolve **before** the
rest of the channel — the reactions a step sets off, resolved fully before the next
step. Queue those with the **nested** call and they're inserted right after the
current item rather than at the back of the line, giving depth-first resolution. It's
the same "resolve this fully before moving on" shape the
[reaction-window cascades](../../concepts/events-and-reactions.md) rely on.

## Where it fits in the framework

QueueFramework is a standalone **world subsystem** — one per play world, zero
dependencies on the rest of the framework. Because it's pure sequencing, systems that
need ordered async execution build on it rather than reinventing it:

- The **EventSystem** can queue a broadcast to run after the current work instead of
  immediately (the deferred-timing option on an event).
- The presentation layer sequences animated playout on it, so a multi-step visual
  finishes step-by-step in order.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UQueueSubsystem` | The world subsystem: queue items, configure and control channels, inspect state. |
| `UQueuedItem` | The base class for a unit of work — override sync or async execution and call Proceed. |
| `UQueuedAction` / `UQueuedOrder` / `UQueuedEvent` | Convenience bases matching the three well-known channels. |
| `QueueChannel::Order` / `Action` / `Event` | The well-known channel-name constants. |
| **Queue Item** node | The Blueprint node that constructs an item, sets its spawn properties, and queues it. |

## Where to go next

- **[API Reference](reference.md)** — the full public surface: the subsystem, the
  item lifecycle, channel control, and the queue-item node.
- **[Tokens & Cues](../../concepts/tokens-and-cues.md)** — the presentation layer that
  sequences animated playout on a queue.
