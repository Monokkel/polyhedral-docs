# TagEvents

For developers who want to call a handler on an object *by gameplay tag* — without
wiring a delegate, an interface, or a hard reference. After this section you'll know
how to raise a tag event, how an object declares a handler for one, and when to reach
for this lightweight core instead of the full event system built on top of it.

A **tag event** is a direct call: you name a gameplay tag and an object, and the
framework finds and runs that object's handler for that tag — if it has one — passing
along a caller and an optional payload. There is no subscription list and no broadcast;
one object raises, one object handles. That directness is the whole point: it's the
smallest possible "dispatch by tag" primitive.

!!! note "This is a foundation, not the usual entry point"
    Most gameplay code should use the higher-level **EventSystem** — broadcast to many
    listeners on channels, with ordering and reaction windows. TagEvents is the
    reflection-dispatch core that sits *underneath* it. Reach for TagEvents directly
    only when you want a single tag-keyed callback on one object and none of the
    broadcast machinery.

## Where it fits in the framework

TagEvents is a standalone, zero-dependency plugin. It provides one mechanism —
resolve the handler a gameplay tag names on an object, validate its signature, and
call it. Two things build on that core:

- The **[EventSystem](../eventsystem/index.md)** plugin adds broadcast, channels,
  subscriber ordering, and reaction windows on top of it — that's what "at the start
  of my turn" and "when damaged, retaliate" ride on.

- Tag-driven debugging tools are a thin specialization of the same core.

Because it's zero-dependency, you can adopt TagEvents on its own in any Unreal
project — it doesn't pull in the entity system or anything else.

## How a tag event works

Two halves meet at a gameplay tag:

1. **A handler is declared** on an object — in Blueprint with the **Tag Event** node
   (you pick the tag it listens for), or in C++ as a reflected function. You never
   name the function yourself; the tag identifies it.
2. **The event is raised** with `RaiseTagEvent` (or `RaiseTagEventWithPayload`),
   naming the target object and the tag. The core looks up the handler the tag maps
   to, checks that its signature is what the caller expects, and invokes it with the
   caller and payload.

If the object has no handler for that tag, the raise is simply a no-op that returns
`false` — asking is safe.

!!! warning "Handlers are matched by tag, never by function name"
    The function a tag event calls is resolved from the tag, not from a name you
    choose. In Blueprint the **Tag Event** node writes the handler for you; in C++ the
    name is generated (see [`GetGeneratedFunctionNameForTag`](reference.md#raising-a-tag-event)).
    A hand-named C++ function will never be found — declare handlers the framework's
    way, covered in the [reference](reference.md#declaring-a-handler).

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UTagEventFunctionLibrary` | The static API: raise a tag event, ask whether an object handles one, resolve a tag's handler name. |
| `UTagEventPayloadObject` | The optional payload passed to a handler — a typed struct plus a tag bag, a numbers map, and a pause/resume pair for event chains. |
| **Tag Event** node | The Blueprint node that declares a handler bound to a tag, with a fixed `(Caller, Payload)` output. |

## Where to go next

- **[API Reference](reference.md)** — the full public surface: raising events, the
  payload object, and declaring a handler in Blueprint or C++.
- **[Events & Reactions](../../concepts/events-and-reactions.md)** — the concept page
  for the broadcast-and-react model the EventSystem builds on this core.
