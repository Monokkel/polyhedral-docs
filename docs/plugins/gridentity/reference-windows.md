# Movement & Structural Windows

For developers writing rules that must interject *while* something happens on the
board: a trap that springs as a unit enters a cell, overwatch that fires on a step, a
wall of force that refuses entry, or a collapse that drops the floor out from under
somebody. After this page you know which moment to bind, which one lies to you, and how
a board edit that displaces units is submitted so it stays one undo step.

These moments are [reaction windows](../../concepts/events-and-reactions.md#reaction-windows)
— the same interrupt-then-commit-then-react shape the rest of the framework uses. What
this page adds is *where they sit around movement and board edits*, and what facts they
carry.

## Two moments per step

A rule-relevant move does not move a unit from A to Z. It takes one step at a time,
and each step offers two moments to bind:

| Event tag | When it fires | Bind it for |
|---|---|---|
| `Event.Movement.StepIntent` | Before the step's placement write. Nothing has moved yet and no change is on the table. | Rules that must act on the *declaration* — a guard that shouts, a stop that truncates the move, an interface prompt. |
| `Event.Movement.StepCommit` | The placement write itself. Its interrupt phase can deny entry; its reaction phase runs after the unit has arrived. | Everything that must be true of a step that *actually happened* — traps, overwatch, entry denial. |

You bind them the way you bind any moment: a
[trigger](../abilitysystem/reference-triggers.md#authoring-triggers) whose event tag is
one of these two. The [sign of the trigger's order](../abilitysystem/reference-triggers.md#interrupt-vs-reaction-the-orders-sign)
picks the phase — negative runs before the step commits and may deny it, zero or
positive runs after the unit has arrived.

!!! warning "Step intent fires for steps that never happen"
    `Event.Movement.StepIntent` is a *declaration*. The step it announces may be
    truncated, vetoed at commit, or abandoned — so intent **leaks**: it fires for moves
    that do not occur.

    Anything that reveals hidden information, or that must only pay off on a real
    step, belongs on `Event.Movement.StepCommit`. A hidden trap bound to intent
    announces itself to a player who merely *considered* the cell; an attack of
    opportunity bound to intent swings at a unit that never left. Commit does not leak:
    it fires only for a step that actually committed.

    Intent is the right moment for exactly one thing — deciding, before anything moves,
    that this move should not continue.

!!! note "C++ only: nothing emits these from Blueprint"
    The two window emitters are C++-only, and the per-step pump that drives them is the
    framework's own stepping module. A game does not open these windows itself: it
    **binds triggers to the two event tags** and lets the ordinary move machinery raise
    them. If nobody is bound to a moment, no broadcast happens at all and the step runs
    exactly as it would with no window — a step you are not listening to costs nothing.

    One consequence worth knowing: the [per-unit movement queries](reference.md#computing-reachability-and-paths)
    reposition units without any of this. They are deliberately reaction-blind. The
    commit window is the only reaction-aware move path, and it is what an ability's move
    step uses.

## What a step window carries

Both moments carry the same read-only fact:

```cpp
// The step fact both movement windows carry. Blueprint-readable; nothing on it is writable.
struct FPGxStepWindowPayload
{
    FPGeEntityRef             Mover;          // who is stepping
    FGridNodeHandle           FromAnchor;     // the anchor cell it leaves
    FGridNodeHandle           ToAnchor;       // the anchor cell it steps to
    int32                     StepIndex;      // which step of the move this is
    TArray<FGridNodeHandle>   CoveredBefore;  // every cell it covers now
    TArray<FGridNodeHandle>   CoveredAfter;   // every cell it will cover
};
```

Three things to know about it:

- **It speaks in cells, not shapes.** A [2&times;2 golem](reference-footprints.md)
  taking a one-cell step still enters and vacates a whole footprint's worth of cells,
  and the covered lists say so. A single-cell mover simply has one-element lists — no
  special case in your rule.
- **It is a fact, never a lever.** Nothing on it is writable; you do not reshape a step
  by editing the payload. A step is stopped or denied through the calls below and
  through the interrupt phase.
- **The lists are in a fixed order**, identical in live play, after an undo, and in a
  replay — so a rule that iterates them behaves the same every time.

The payload also carries the generic window facts every reaction gets: the entity the
window concerns (here, the mover) and the instigator behind the move.

!!! note "C++ only"
    The convenience views for "which cells did it *enter*" and "which cells did it
    *vacate*" are C++ inline helpers with no Blueprint nodes. Blueprint content
    computes them from `Covered Before` and `Covered After` — for the common
    single-cell case, `To Anchor` and `From Anchor` already answer it.

**Build Step Window Payload** is Blueprint callable, if you need to construct the same
fact for your own code — it resolves the mover's covered cells at both anchors for you.

## Ending a move in flight

The lever an intent-phase rule reaches for is the **movement stop**: a fact written on
the mover meaning "the movement currently in flight ends here". The move re-reads it
before every step and truncates when it is set.

```cpp
// "The movement in flight ends." Blueprint callable; idempotent.
static void WriteMovementStop(const UObject* WorldContext, const FPGeEntityRef& Mover);

// Is it set? Blueprint pure.
static bool HasMovementStop(const UObject* WorldContext, const FPGeEntityRef& Mover);

// Clear it. Blueprint callable; true if it was set and is now cleared.
static bool ConsumeMovementStop(const UObject* WorldContext, const FPGeEntityRef& Mover);
```

It is stored as ordinary command-routed tagged data, so it rides the move's own
transaction: the truncation is part of the same undo step as the move, and undoing the
move restores the fact along with everything else. You do not clear it yourself in the
normal case — the move consumes it when it completes or truncates.

!!! warning "This stops one move, not a turn"
    The movement stop ends the movement *currently executing*. It is not how you deny a
    unit its movement for a turn — that is ordinary content: a turn-scoped modifier on
    the unit's movement stat, read by the move's own use condition. Writing the stop
    fact to mean "you may not move this turn" leaves a fact on the unit that the next
    move will consume and discard.

=== "Blueprint"
    1. Author a trigger whose **Event Tag** is `Event.Movement.StepIntent` and whose
       **Order** is negative — it runs before anything moves.
    2. In its program, read the payload's **Mover** and decide (a guard's zone of
       control, a spike field, a fear effect).
    3. Call **Write Movement Stop** with the mover. The move truncates at this step and
       the whole exchange is still one undo step.

=== "C++"
    ```cpp
    // Inside a rule reacting to Event.Movement.StepIntent:
    if (ShouldHaltThisMover(Payload.Mover))
    {
        // Rides the move's open transaction — one undo step for the move AND the halt.
        UPGxMovementWindowLibrary::WriteMovementStop(this, Payload.Mover);
    }
    ```

Denying a *single* step rather than ending the move is the other phase's job: an
interrupt on `Event.Movement.StepCommit` refuses entry, and the unit stays where it is.

## Editing the board under units

The board itself can change at runtime — a dug tunnel, a collapsed bridge, a
disintegrated floor. When the cells being removed might have units on them, the
edit goes through one door:

```cpp
// The sanctioned door for "dig / collapse / disintegrate these cells."
// Blueprint callable. True iff the removal actually committed.
static bool SubmitCellRemovalWithDisplacement(
    const UObject* WorldContext, FName GridId,
    const TArray<FGridNodeHandle>& CellsToRemove, const FPGeEntityRef& Instigator);
```

It returns **false with nothing on the undo stack** when the edit was vetoed, when the
board or command stack is unavailable, when the named board does not resolve, or when
none of the listed cells is actually a live cell. A false return is always "nothing
happened", never "something half happened".

What it does, in order:

1. **Opens one window** on the board's reserved structural channel — even when the edit
   displaces nobody, because an unoccupied dig is still something a rule may want to
   refuse.
2. **The interrupt phase may veto** the edit before any command is built, or *shape* it
   — folding relocations of the units about to be displaced into the same transaction.
3. **The commit** submits the cell removals as one nested transaction, so the whole edit
   is **one undo step on every path**, including the path where nobody was listening.
4. **The reaction phase** runs after the board has changed — rubble damage, an
   auto-eject, a kill.

Undoing that one step restores the cells and revalidates the placements that were
dangling.

### The structural channel

Edit-level veto power belongs to whoever subscribes the board's channel, not to
whichever displaced unit happens to sort first — so the window is keyed to a **channel**
rather than to an entity.

```cpp
// The reserved channel for a board's edit-level windows. Blueprint pure.
// The main board -> "Channel.Grid.Structure"; a named board suffixes its id.
static FName GetStructureChangeChannel(FName GridId);
```

Subscribe to `Event.Grid.StructureChange` on that channel with a **negative order** and
**not transient** to hold veto power over board edits. The global channel reaches it
too, for a listener that wants everything.

!!! note "C++ only"
    The structural window's payload — the removed cells and the units about to be
    displaced — and the per-unit displacement payload are **native structs with no
    Blueprint exposure**. A rule that needs to read them is C++. Blueprint can still
    submit the edit (**Submit Cell Removal With Displacement**), resolve the channel
    (**Get Structure Change Channel**), and react to a displacement by binding the
    per-unit event below.

### Two displacement signals, on purpose

When a removal leaves a unit's cell gone, two different things fire, for two different
audiences:

| Signal | Audience | When |
|---|---|---|
| `Event.Placement.Displaced` | **Rules.** Broadcast per displaced unit on that unit's own window channel — a trigger binds it the way it binds any moment. | The live forward path only. It rides the same undo step as the edit and **never re-fires on undo or redo**. |
| The occupancy subsystem's displacement delegate | **Derived state and presentation.** A C++ multicast. | Every apply **and** every undo/redo — because presentation must reconcile however the timeline moves. |

The split is what lets a "when I am displaced, take falling damage" rule fire exactly
once, while the mini that must be repositioned still hears about every rewind.

Undoing the edit makes the unit's placement resolve again by itself. **No counter-event
is fired** — the rules signal belongs to the forward path, and the reconciling
consumers are already listening to the delegate.

!!! note "C++ only"
    The occupancy displacement delegate is a plain C++ multicast, not a Blueprint
    assignable one. Blueprint reacts to displacement through the
    `Event.Placement.Displaced` trigger, or through the presentation cue below.

### When a unit's cell is gone: the unresolvable cue

A displaced unit keeps its placement — it now names a cell that no longer exists. Its
on-screen mini cannot be positioned from it, so the framework raises a presentation
cue rather than freezing silently:

| Cue | Payload | Meaning |
|---|---|---|
| `Cue.Placement.Unresolvable` | The unit and the placement that no longer resolves | "This unit's cell is gone. Decide what its mini does." |

**No handler ships for it.** With none registered the cue completes gracefully and the
token simply holds its last transform — correct, but inert. Register a
[cue handler](../tokensystem/reference.md#cue-handlers) for the tag, globally or per
token class, and decide what a falling unit looks like: sink it, play rubble, ragdoll
it, or hide it until your rules relocate it.

=== "Blueprint"
    1. Create a Blueprint subclass of the cue handler base and set its handled cue to
       `Cue.Placement.Unresolvable`.
    2. In its play step, read the payload's unit, animate the mini falling, and complete
       the cue.
    3. Separately, author a trigger on `Event.Placement.Displaced` for the *rules* half
       — the falling damage, the relocation, the death.

=== "C++"
    ```cpp
    // Submit the collapse. The window may veto it; a true return means it committed
    // as exactly one undo step.
    const bool bCollapsed = UPGxStructuralWindowLibrary::SubmitCellRemovalWithDisplacement(
        this, /*GridId=*/NAME_None, CollapsedCells, /*Instigator=*/Caster);
    ```

!!! tip "Displacement is not relocation"
    The framework tells you a unit was displaced; it never guesses where the unit should
    end up. Falling to the level below, being ejected to the nearest legal cell, or
    dying in the rubble are all your game's rules — written as reactions to
    `Event.Placement.Displaced`, or folded into the edit itself by an interrupt that
    relocates the unit inside the same transaction.

## Board edits during a preview

Board edits behave **identically** between turns, during a live ability, and inside a
what-if [preview run](../abilitysystem/reference-previews.md). A preview that
disintegrates a floor simulates the veto and the displacement reactions honestly, and
its board is restored exactly when the preview is discarded — so an AI weighing
"what if I collapse this bridge?" sees the same rules a real cast would.

!!! note "C++ only"
    The subsystem that brackets a speculative run's board edits has **no Blueprint
    surface at all** — it is framework wiring, driven by ability resolution. There is
    nothing to call and nothing to configure; you get the behaviour by submitting board
    edits through **Submit Cell Removal With Displacement**. One rule is visible from
    content: authoring edits to the board are refused while a preview is in flight.

## See also

- [Events, Ordering, and Reaction Windows](../../concepts/events-and-reactions.md) —
  the interrupt/reaction model these windows implement.
- [Triggers & Reactions](../abilitysystem/reference-triggers.md) — how a rule binds an
  event tag and picks its phase.
- [Footprints](reference-footprints.md) — why a step's covered lists are plural.
- [Occupancy](reference.md#occupancy) — the index that reconciles itself when the board
  changes underneath it.
