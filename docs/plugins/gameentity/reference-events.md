# API Reference: Change Events

For developers wiring UI, visuals, or cached lookups to game state. Every
authoritative mutation broadcasts a change event — during normal play **and**
during undo, redo, and load — and reactive code stays correct by reconciling to
what state now is rather than reasoning about why it changed. Read
[Derived State and Change Events](../../concepts/derived-state.md) for the model;
this page is the exact event surface.

Here you'll find the change record, every change type and cause, how to
subscribe from Blueprint and C++, and the mutation-to-event matrix. The stat and
modifier *API* has [its own page](reference-stats.md), and the entity, tag,
hierarchy, and tagged-data *API* that produces these events is documented under
[Entities & Templates](reference-entities.md). All public types carry the `PGe`
prefix, and the signatures below are hand-written illustrations, not source
excerpts.

## The change record

The catch-all event hands you one `FPGeEntityChange` per mutation — enough to
react without re-querying the subsystem.

```cpp
struct FPGeEntityChange
{
    FPGeEntityRef          EntityRef;      // the entity that changed
    EPGeEntityChangeType   ChangeType;     // what kind of change (see below)
    FGameplayTag           Tag;            // the affected tag, stat, or data key
    FEvalStatValue         OldValue;       // stat value before  — stat/modifier events only
    FEvalStatValue         NewValue;       // stat value after   — stat/modifier events only
    FPGeEntityRef          OtherEntityRef; // the child, on ChildAdded / ChildRemoved
    EPGeChangeCause        Cause;          // why it fired (see below)
    // FinalState — C++-only snapshot of a just-destroyed entity (see note)
};
```

- **`Tag`** carries whichever key the change is about: the gameplay tag for a
  tag change, the stat tag for a stat or modifier change, the `Data.*` key for a
  tagged-data change. It is unset for whole-entity events.
- **`OldValue` / `NewValue`** are meaningful only on stat and modifier events.
  They are an **exact fixed-point** value type — never a lossy float mirror — so
  a comparison on the payload is exact by construction. Convert to a float or
  text only at your own display edge (its `ToFloat` / `ToString`, or the
  Evaluators display helpers). On every non-stat event they are a natural zero.
- **`OtherEntityRef`** names the second party in a hierarchy change — the child
  for `ChildAdded` and `ChildRemoved`.

!!! note "Destroy carries a final snapshot (C++)"
    An `EntityDestroyed` change also carries `FinalState`, a snapshot of the
    entity's closing data captured before it left the game state, so death
    presentation can read what it was after it is already gone. The broadcast
    itself happens *after* removal — a listener that queries the subsystem sees
    the entity absent. `FinalState` is available to C++ listeners; it is not
    reflected to Blueprint.

## Change types

`EPGeEntityChangeType` names what happened. The catch-all carries every value;
the typed delegates cover the common ones.

| Value | Meaning |
|---|---|
| `EntityCreated` | An entity was created. |
| `EntityDestroyed` | An entity was destroyed (carries `FinalState`). |
| `TagAdded` | A gameplay tag was added. |
| `TagRemoved` | A gameplay tag was removed. |
| `StatChanged` | A **base** stat's value was set or changed. |
| `StatRemoved` | A **base** stat was removed entirely. |
| `TaggedDataChanged` | A tagged-data entry was set or overwritten. |
| `TaggedDataRemoved` | A tagged-data entry was removed. |
| `ChildAdded` | A child was attached in the hierarchy. |
| `ChildRemoved` | A child was detached. |
| `ModifierAdded` | A runtime stat modifier was added. |
| `ModifierRemoved` | A runtime stat modifier was removed. |
| `ModifierUpdated` | A live modifier's spec was replaced in place. |
| `CurrentStatChanged` | A **resolved current** stat's value changed (derived). |
| `CurrentStatRemoved` | A resolved current stat stopped being materialised. |

!!! tip "Base stat vs current stat"
    `StatChanged` / `StatRemoved` report edits to an entity's authored **base**
    stat. `CurrentStatChanged` / `CurrentStatRemoved` report the **derived**
    current value after modifiers fold in — the number you usually display.
    Adding a modifier does not change a base stat, so it fires `ModifierAdded`
    *and*, via recompute, `CurrentStatChanged`. See
    [Stats & Modifiers](reference-stats.md).

## The change cause

`EPGeChangeCause` says *why* an event fired. Use it for presentation decisions
only — never as a gameplay filter.

| Value | Fires from |
|---|---|
| `Command` | Live forward mutation (a normal submit or transaction commit). |
| `Undo` | An undo, an aborted transaction, or a mid-apply rollback. |
| `Redo` | A redo re-running a command. |
| `Restore` | A wholesale restore (load or state replacement). |

The classic use is feel: animate on `Command`, and snap on `Undo`, `Redo`, or
`Restore` because state is jumping rather than progressing.

!!! warning "Cause is presentation metadata, not a rule filter"
    Game rules never run from change events, so the cause is never a signal to
    apply a rule. It exists so *presentation* can tell a heal from an undo of
    damage. Reacting to state with rules is the event system's job, not a
    change-event listener's — see [Listeners are read-only](#listeners-are-read-only).

## Subscribing to change events

The subsystem exposes one catch-all delegate, one restore delegate, and a set of
typed conveniences. All are Blueprint-assignable multicast delegates.

```cpp
// The catch-all — fires once for EVERY live mutation, with the full record.
FPGeOnAnyEntityChanged  OnAnyEntityChanged;   // (const FPGeEntityChange& Change)

// Fires once when the whole state is replaced wholesale (load / reset).
FPGeOnStateRestored     OnStateRestored;      // ()

// Typed conveniences for narrow listeners.
FPGeOnEntityCreated     OnEntityCreated;      // (const FPGeEntityRef& EntityRef)
FPGeOnEntityDestroyed   OnEntityDestroyed;    // (const FPGeEntityRef& EntityRef)
FPGeOnTagChanged        OnTagChanged;         // (EntityRef, FGameplayTag Tag, bool bAdded)
FPGeOnStatChanged       OnStatChanged;        // (EntityRef, FGameplayTag StatTag, FEvalStatValue NewValue)
FPGeOnCurrentStatChanged OnCurrentStatChanged;// (EntityRef, StatTag, FEvalStatValue OldValue, NewValue)
FPGeOnChildAdded        OnChildAdded;         // (const FPGeEntityRef& ParentRef, const FPGeEntityRef& ChildRef)
FPGeOnChildRemoved      OnChildRemoved;       // (const FPGeEntityRef& ParentRef, const FPGeEntityRef& ChildRef)
```

=== "Blueprint"
    Get the game-state subsystem, then use the red **Bind Event to On Any Entity
    Changed** (and **On State Restored**) nodes, or the **Assign** shortcut, to
    wire a custom event. In the handler, branch on **Change Type** and read the
    fields you need.

=== "C++"
    ```cpp
    void UHealthBar::BindTo(UPGeGameStateSubsystem* State)
    {
        // Dynamic multicast delegates — the handlers must be UFUNCTION()s.
        State->OnAnyEntityChanged.AddDynamic(this, &UHealthBar::OnChanged);
        State->OnStateRestored.AddDynamic(this, &UHealthBar::Rebuild);
    }

    void UHealthBar::OnChanged(const FPGeEntityChange& Change)
    {
        if (Change.EntityRef != Watched) { return; }
        if (Change.ChangeType == EPGeEntityChangeType::CurrentStatChanged)
        {
            const bool bSnap = Change.Cause != EPGeChangeCause::Command;
            SetHealth(Change.NewValue.ToFloat(), bSnap); // convert only at display
        }
    }
    ```

!!! warning "Subscribe to the catch-all plus restore — always both"
    A consumer that maintains derived state binds **`OnAnyEntityChanged`** (to
    update incrementally on every mutation) **and** **`OnStateRestored`** (to
    rebuild wholesale on load). Deriving state from only a subset of the typed
    events is the classic way to desync silently — you miss the one mutation you
    forgot to subscribe to, and nothing tells you. `OnStateRestored` alone drifts
    everywhere except load boundaries; the catch-all alone misses the wholesale
    rebuild. Bind both. The full pattern is in
    [Derived State and Change Events](../../concepts/derived-state.md).

### Listeners are read-only

A change-event listener may **never** change authoritative state — not directly,
and not by submitting a command. This is permanent, and the command stack
enforces it in development builds.

!!! warning "Reactions to events are read-only, forever"
    Change events fire during undo and redo as well as forward play. A listener
    that mutated state would re-drive your rules while the game was trying to
    undo them. Change events are for **derived state and presentation only**.
    State-mutating reactions — "when damaged, retaliate" — are game rules and
    belong to the event system's reaction mechanism, not here.

!!! note "Look-ahead stays invisible"
    The ability system (documented in a later section) can simulate an action in
    advance without touching real state. During such a simulation the live
    delegates above stay quiet by design, so reactive presentation never reacts to
    a move that has not really happened — your listeners need no special case.

## Wholesale restore

Loading a save or replacing the state does not fire a stream of individual
mutation events for the whole board. Instead it fires **`OnStateRestored`** once,
and the individual differences it does broadcast carry the `Restore` cause.

```cpp
// Replace the entire game state. Clears the command stack, then diffs the old
// and new states and broadcasts the differences (cause = Restore) plus
// OnStateRestored. Loading a save does the same.
void ReplaceState(const FPGeGameState& NewState);
```

A derived-state consumer treats `OnStateRestored` as its cue to rebuild from
scratch — read the current state and reconstruct its structure from nothing —
rather than trying to follow each diff.

## The mutation-to-event matrix

Which subsystem call fires which event. Every row also fires
**`OnAnyEntityChanged`** with the matching change type, and every row fires the
same way during **undo, redo, and load** — an undo of a heal is, to a listener,
just another stat change.

| Mutation (see the linked API) | Change type | Typed delegate |
|---|---|---|
| `CreateEntity*` | `EntityCreated` | `OnEntityCreated` |
| `DestroyEntity` | `EntityDestroyed` | `OnEntityDestroyed` |
| `AddTag` | `TagAdded` | `OnTagChanged` (`bAdded = true`) |
| `RemoveTag` | `TagRemoved` | `OnTagChanged` (`bAdded = false`) |
| `SetStat`, `ModifyStat`, `ModifyStats` | `StatChanged` | `OnStatChanged` |
| `RemoveStat` | `StatRemoved` | *(catch-all only)* |
| `SetTaggedData` | `TaggedDataChanged` | *(catch-all; also `OnTagChanged` if the mirror tag is new)* |
| `RemoveTaggedData` | `TaggedDataRemoved` | *(catch-all; also `OnTagChanged` if the mirror tag drops)* |
| `AddChild` | `ChildAdded` | `OnChildAdded` (plus tag events for the slot tag) |
| `RemoveChild` | `ChildRemoved` | `OnChildRemoved` (plus tag events for the slot tag) |
| `SetChildSlot` | `TagRemoved` / `TagAdded` | `OnTagChanged` (old and new slot tag) |
| `AddStatModifier`, `AddFlatModifier`, `AddPercentModifier` | `ModifierAdded` | *(catch-all; recompute also fires `OnCurrentStatChanged`)* |
| `RemoveStatModifier`, `RemoveStatModifiersBy…` | `ModifierRemoved` | *(catch-all; recompute also fires `OnCurrentStatChanged`)* |
| `UpdateStatModifier` | `ModifierUpdated` | *(catch-all; recompute also fires `OnCurrentStatChanged`)* |
| *(derived recompute)* | `CurrentStatChanged` / `CurrentStatRemoved` | `OnCurrentStatChanged` |

Notes:

- **Every stat and modifier change drives a derived recompute**, which fires its
  own `CurrentStatChanged` event separately from the base-stat or modifier event
  that triggered it. That recompute is the resolved value you usually bind a
  health bar to. The stat and modifier API is on the
  [Stats & Modifiers page](reference-stats.md).
- **Tagged data mirrors a tag**, so setting or removing a tagged-data entry can
  also add or drop the mirrored tag, firing a tag event alongside the tagged-data
  one. That mirror behaviour is documented under
  [Tagged data on an entity](reference-entities.md#tagged-data-on-an-entity).
- A **no-op mutation broadcasts nothing** — adding a tag the entity already has,
  or removing a stat it doesn't have, fires no event.

## Related pages

- [Derived State and Change Events](../../concepts/derived-state.md) — the model
  and the copy-this consumer pattern.
- [Entities & Templates reference](reference-entities.md) — the entity, tag,
  hierarchy, and tagged-data API behind these events.
- [Stats & Modifiers reference](reference-stats.md) — the stat and modifier API,
  and how current-stat events are produced.
- [CommandSystem](../commandsystem/index.md) — the stack these events fire from,
  and its own execution-phase signal.
