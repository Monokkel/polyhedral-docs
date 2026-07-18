# Commands: One Door to Game State

For developers who want the undo/redo and replay guarantees the framework advertises — and need to know what those guarantees ask of them. After reading, you'll know how the command stack works, how to group changes into one undo step, and how to write a custom command that undoes itself correctly.

## One door for every change

Every authoritative change to game state goes through a **command** submitted to a single **command stack**. That is the whole trick behind undo, redo, and replay: because there is exactly one door that state changes pass through, the framework can record what happened and reverse it on demand.

Reads are direct and free. You look up an entity, a tag, or a stat whenever you like — no command needed. Only *writes* go through the door.

Most of the time you never write a command yourself. The game-entity mutation functions — create an entity, add a tag, modify a stat, add a modifier — are already command-driven. Call one and you get undo, redo, and replay with nothing extra to do. You author a *custom* command only when you add a new kind of state that the framework does not already manage. See [Entities Are Data](entities-as-data.md) for what that state is.

!!! tip "The split in one line"
    Reading state is a plain function call. Changing state is a command. If you find yourself mutating game state without going through the stack, that change will not undo, save, or replay.

## Submit, undo, redo

The lifecycle is small:

- **Submit** applies the command immediately. If it succeeds, it goes onto the undo stack.
- **Undo** reverses the top entry and moves it to the redo stack.
- **Redo** re-applies it.

Redo re-runs the command against the *current, restored* state — not against a saved snapshot. That single fact is the reason commands must be repeatable (more on that below).

=== "Blueprint"
    1. Get the game-state subsystem and call **Modify Stat** (for example, `Health`, delta `-5`). This is already command-driven — the change is on the undo stack the moment it returns.
    2. On your **Undo** button, get the command stack and call **Undo**. Bind the button's *enabled* state to **Can Undo**.
    3. On your **Redo** button, call **Redo**, enabled by **Can Redo**.

    With no custom command written, the buttons already work.

=== "C++"
    ```cpp
    // State = the game-state subsystem, Stack = the command stack.
    State->ModifyStat(HeroRef, HealthTag, -5); // command-driven: already undoable
    Stack->Undo();                             // health back to its previous value
    Stack->Redo();                             // and forward again
    ```

## Transactions: one undo step for a whole move

A player's turn is rarely one change. Moving a unit might clear its old position, set a new one, and spend a movement point. You want a single **Undo** to reverse the *whole move*, not to peel it back one internal step at a time.

Wrap the steps in a **transaction** — one undo step:

=== "Blueprint"
    1. Command stack &rarr; **Begin Transaction** (`"Move Unit"`).
    2. Run the move's individual command-driven steps — clear the old placement, set the new placement, spend movement.
    3. Command stack &rarr; **Commit Transaction**.

    Now a single **Undo** reverses the entire move. If one of those steps is itself a helper that brackets its own transaction, it *joins* this one instead of creating a separate undo step.

=== "C++"
    ```cpp
    Stack->BeginTransaction(TEXT("Move Unit"));
    MoveUnitToCell(UnitRef, TargetCell);        // several command-driven writes
    State->ModifyStat(UnitRef, MovementTag, -Cost);
    Stack->CommitTransaction();                 // one undo step; one Undo reverses it all
    ```

Transactions nest by joining the already-open one, so a helper that brackets its own transaction composes correctly inside a larger move. Aborting rolls back only what was added since the matching begin.

## Writing a custom command

When you add state the framework does not manage, you write a command for it. A command is a small struct that knows how to apply a change and how to reverse it. Four rules make that reversal correct.

!!! warning "The four rules every custom command must follow"
    **1. Record, don't diff.** Inside `Apply`, capture the exact *previous values* you will restore — not the delta you applied. A change that gets clamped or capped cannot be reversed by subtracting a delta.

    **2. Check first, mutate second.** Validate everything and bail out *before* touching any state. This is the most dangerous rule to break: a command that changes one field, then fails and stops, leaves a change that nothing on any stack will ever undo.

    **3. Be repeatable.** No randomness, no clock, no iterating an unordered map in hash order, no reading derived caches. Same state in, same result out — because undo and redo both re-run your `Apply`.

    **4. Reverse everything.** `Undo` must reverse every change `Apply` made, including side effects — and must know whether each side effect actually happened, so it doesn't over-correct.

A command captures its own inverse: the previous values live *inside* the command, written during `Apply`, so `Undo` has exactly what it needs.

=== "Blueprint"
    Subclass the Blueprint command class and implement its **Apply** and **Undo** events. Then follow the one bridge rule:

    !!! warning "Keep all captured state in Payload"
        Store **every** captured undo value in the command's `Payload` struct. Ordinary variables on the class do **not** survive the undo/redo round-trip — a fresh command object is built for each call, so anything outside `Payload` is lost and undo corrupts silently. Capture your old values into `Payload` inside **Apply**.

=== "C++"
    A custom command is a small `USTRUCT` deriving from `FPCsCommand`. It stores its inputs and, captured inside `Apply`, the previous value `Undo` will restore.

    ```cpp
    USTRUCT()
    struct FCmd_SetInitiative : public FPCsCommand
    {
        GENERATED_BODY()
        UPROPERTY() int32 NewValue = 0;  // input
        UPROPERTY() int32 OldValue = 0;  // captured inverse state

        bool Apply(FPCsCommandContext& Ctx) override
        {
            UInitiativeTracker* T = UInitiativeTracker::Get(Ctx.WorldContext);
            if (!T) { return false; }    // check first, mutate second
            OldValue = T->Value;         // record the absolute value, not a delta
            T->Value = NewValue;
            return true;
        }
    ```

    `Undo` restores exactly what `Apply` captured and reverses any side effects it caused:

    ```cpp
        bool Undo(FPCsCommandContext& Ctx) override
        {
            UInitiativeTracker* T = UInitiativeTracker::Get(Ctx.WorldContext);
            if (!T) { return false; }
            T->Value = OldValue;         // reverse everything Apply did
            return true;
        }
    };
    ```

    Submit it through the stack, never by calling `Apply` yourself:

    ```cpp
    FCmd_SetInitiative Cmd;
    Cmd.NewValue = 12;
    Stack->Submit(FInstancedStruct::Make(Cmd));
    ```

## Listeners never submit

Reactive code that watches state change — a health bar, a cache, an on-screen token — can never submit a command in response. The stack rejects it loudly in development builds, and the rule is permanent, not a temporary gap.

!!! warning "Change-event listeners are read-only"
    A listener runs *inside* command execution — including during undo and redo. If it could submit a command, undo would re-drive your game rules while it was trying to reverse them. So listeners observe; they never write.

Reactions that *mutate* state — "when this unit is damaged, it retaliates" — are game rules, not derived state. They belong to [reaction windows](events-and-reactions.md), the event system's reaction mechanism. For everything that only *reflects* state, see [Derived State and Change Events](derived-state.md).

## Why the strictness pays off

Undo, redo, and replay all work the same way: they re-run commands against reconstructed state. One command that captures a delta instead of a value, or half-applies before failing, or reads a random number, corrupts all three — and nothing flags it at the moment it happens. Follow the four rules and you get an undo/redo/replay system you never have to debug.

Commands mutate *base* state only. Derived values — like resolved current stats — are never captured in a command; the framework recomputes them automatically afterward (see [Derived State and Change Events](derived-state.md) and [Stats and Modifiers](stats-and-modifiers.md)). That boundary is exactly why rule 3 forbids reading derived caches inside `Apply`.

The framework ships an [automated test harness](../plugins/commandsystem/reference.md#conformance-harness) that exercises a command against these rules — applying, undoing, and redoing it and checking the state comes back byte-for-byte — so a command that honors the contract can be proven to honor it.

For the full command-stack API — transactions, the execution phase, stack events, and the correctness harness — see the [CommandSystem plugin reference](../plugins/commandsystem/reference.md).

GameEntity ships ready-made commands for every kind of state change — modifying stats, adding tags, creating entities, and more — so you rarely write one yourself; see the [built-in command catalogue](../plugins/gameentity/reference-commands.md#the-built-in-command-catalogue).

*See it in action: the [first board tutorial](../getting-started/first-board.md) makes a stat change and wires up a working Undo button.*
