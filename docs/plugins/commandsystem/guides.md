# CommandSystem Guides

For developers with the plugin enabled who want to get something done. Each
section is a short, followable recipe. For the full type-by-type surface see the
[API Reference](reference.md); for the model behind it all see
[Commands & Undo](../../concepts/commands-and-undo.md).

## Submit a change and wire Undo and Redo buttons

The most common task needs no custom command at all. A built-in command-driven
mutator (here, modifying a stat) is already on the undo stack the moment it
returns — so Undo and Redo are one call each.

=== "Blueprint"
    1. Get the game-state subsystem and call a command-driven mutator such as
       **Modify Stat** (for example `Stat.Health`, delta `-5`). The change is
       recorded automatically.
    2. On your **Undo** button, **Get Command Stack** and call **Undo**. Bind the
       button's *enabled* state to **Can Undo**.
    3. On your **Redo** button, call **Redo**, enabled by **Can Redo**.
    4. To keep the buttons in sync, bind **On Stack Changed** and refresh their
       enabled state (and any history label) when it fires.

    With no custom command written, the buttons already work.

=== "C++"
    ```cpp
    UPCsCommandStack* Stack = UPCsCommandStack::Get(this);

    // A built-in command-driven mutator: already undoable.
    State->ModifyStat(Hero, HealthTag, -5);

    if (Stack->CanUndo()) { Stack->Undo(); }   // health back to its previous value
    if (Stack->CanRedo()) { Stack->Redo(); }   // and forward again
    ```

!!! tip "Reads never touch the stack"
    Only mutating calls are recorded. Reading a stat, a tag, or an entity is a
    plain lookup — undo/redo costs you nothing on the read path.

## Group changes into one undo step with a transaction

A player's action is rarely one change. Moving a unit might clear its old cell,
set the new one, and spend a movement point. You want a single **Undo** to reverse
the *whole move*. Wrap the steps in a **transaction** — one undo step.

=== "Blueprint"
    1. Command stack &rarr; **Begin Transaction** (`"Move Unit"`).
    2. Run the individual command-driven steps — clear the old placement, set the
       new placement, spend movement.
    3. Command stack &rarr; **Commit Transaction**.

    A single **Undo** now reverses the entire move. If one of those steps is itself
    a helper that brackets its own transaction, it *joins* this one rather than
    creating a separate undo step.

=== "C++"
    ```cpp
    Stack->BeginTransaction(TEXT("Move Unit"));

    ClearPlacement(Unit);                       // several command-driven writes
    SetPlacement(Unit, TargetCell);
    State->ModifyStat(Unit, MovementTag, -Cost);

    Stack->CommitTransaction();                 // one undo step reverses all of it
    ```

!!! note "Transactions nest by joining"
    A nested **Begin Transaction** joins the already-open one instead of starting
    an independent step, and only the outermost **Commit** pushes the group onto
    the undo stack. So a helper that brackets its own transaction composes
    correctly inside a larger action. Every **Begin** must be balanced by exactly
    one **Commit** or **Abort**.

## Abort a transaction on failure

When a multi-step action can't complete, **abort** instead of committing. Abort
rolls back the changes made since the matching **Begin**, in reverse order,
leaving the world as if the action never started.

=== "Blueprint"
    1. Command stack &rarr; **Begin Transaction**.
    2. Run the steps. After each, check whether the action can still proceed.
    3. If a step can't proceed, call **Abort Transaction** and return. Otherwise
       call **Commit Transaction**.

=== "C++"
    ```cpp
    Stack->BeginTransaction(TEXT("Cast Spell"));

    if (!SpendMana(Caster, Cost))
    {
        Stack->AbortTransaction();   // undo whatever already ran; nothing sticks
        return;
    }
    ApplySpellEffect(Target);

    Stack->CommitTransaction();
    ```

!!! tip "Abort is safe even mid-effect"
    Abort undoes the children of the current nesting level for you. In the rare
    case a child refuses to reverse, the framework re-applies the ones it already
    rolled back and commits the group to the undo stack instead of discarding it —
    so an applied effect is never orphaned off the stack. You get undo-ability
    either way.

## Write a custom command in C++

Write a command only when you own a *new kind of authoritative state* the
framework doesn't already manage. A command is a small struct that knows how to
apply a change and how to reverse it — the previous values live *inside* the
command, captured during `Apply`, so `Undo` has exactly what it needs.

!!! warning "The four rules every custom command must follow"
    **1. Record, don't diff.** In `Apply`, capture the exact *previous values* you
    will restore — not the delta you applied. A change that gets clamped or capped
    can't be reversed by subtracting a delta.

    **2. Check first, mutate second.** Validate everything and bail out *before*
    touching any state. A command that changes one field, then fails and stops,
    leaves a mutation nothing will ever undo.

    **3. Be repeatable.** No randomness, no clock, no iterating an unordered map in
    hash order, no reading derived caches. Same state in, same result out — because
    undo and redo both re-run your `Apply`.

    **4. Reverse everything.** `Undo` must reverse every change `Apply` made,
    including side effects — and must know whether each side effect *actually
    happened*, so it doesn't over-correct.

Derive a `USTRUCT` from `FPCsCommand`. Store your inputs, and — captured inside
`Apply` — the inverse state `Undo` will restore:

```cpp
USTRUCT()
struct FCmd_SetScore : public FPCsCommand
{
    GENERATED_BODY()

    UPROPERTY() int32 NewScore = 0;      // input

    UPROPERTY() int32 OldScore = 0;      // captured inverse state
    UPROPERTY() bool  bClearedBonus = false;

    bool Apply(FPCsCommandContext& Ctx) override
    {
        UScoreBoard* Board = UScoreBoard::Get(Ctx.WorldContext);
        if (!Board) { return false; }        // (2) check first, mutate second

        OldScore = Board->Score;             // (1) record the value, not a delta
        Board->Score = NewScore;

        // A side effect: crossing 100 clears a pending bonus. Record whether it fired.
        bClearedBonus = (NewScore >= 100 && Board->bBonusPending);
        if (bClearedBonus) { Board->bBonusPending = false; }

        return true;
    }

    bool Undo(FPCsCommandContext& Ctx) override
    {
        UScoreBoard* Board = UScoreBoard::Get(Ctx.WorldContext);
        if (!Board) { return false; }

        Board->Score = OldScore;             // (4) reverse everything Apply did...
        if (bClearedBonus) { Board->bBonusPending = true; }  // ...only if it happened
        return true;
    }

    FString Describe() const override { return TEXT("Set Score"); }
};
```

Submit it through the stack — never by calling `Apply` yourself:

```cpp
FCmd_SetScore Cmd;
Cmd.NewScore = 120;
Stack->Submit(FInstancedStruct::Make(Cmd));   // applied now; on the undo stack if it succeeds
```

!!! tip "Reach the world through the context"
    A command reaches its target subsystem through `Ctx.WorldContext`. Resolve the
    subsystem from it inside `Apply` and `Undo` rather than caching a pointer on
    the command — the same command runs again on redo, possibly after a reload.

Commands mutate *base* state only. Derived values — resolved current stats and
other caches — are never captured in a command; the framework recomputes them
from change events afterward. That boundary is exactly why rule 3 forbids reading
a derived cache inside `Apply`. See
[Derived State & Events](../../concepts/derived-state.md).

## Write a Blueprint command

You can author a command entirely in Blueprint. Subclass the Blueprint command
class and implement its Apply and Undo events — but follow one extra rule that has
no equivalent in C++.

!!! warning "Keep all captured state in Payload"
    Store **every** captured undo value in the command's `Payload` struct. A
    *fresh* command object is built for each Apply and each Undo call, so ordinary
    variables on the class do **not** survive the round-trip — anything outside
    `Payload` is silently lost and undo corrupts. Capture your old values into
    `Payload` inside **Apply**.

=== "Blueprint"
    1. Create a Blueprint class deriving from **PCs Blueprint Command**.
    2. Author a small struct to hold this command's inputs *and* its captured
       inverse state — this is what goes in `Payload`.
    3. Override **BP Apply**: read the current value from the world, write your
       captured "before" value into `Payload`, then make the change. Return
       **true** on success, **false** if you validated and bailed before touching
       anything.
    4. Override **BP Undo**: read `Payload` and restore exactly what you captured.
    5. Override **BP Describe** to return a label for the undo history.
    6. To run it: make an `FPCsCmd_Blueprint`, set its **Command Class** to your
       Blueprint and fill its **Payload**, wrap it in an instanced struct, and
       **Submit** it to the stack.

=== "C++"
    ```cpp
    // Submitting a Blueprint-authored command from C++: point the bridge at the
    // Blueprint class and hand it the payload. The bridge instantiates the BP
    // command and carries Payload across Apply/Undo/Redo for you.
    FPCsCmd_Blueprint Bridge;
    Bridge.CommandClass = MyBlueprintCommandClass;              // TSubclassOf<UPCsBlueprintCommand>
    Bridge.Payload      = FInstancedStruct::Make(FMyCmdPayload{ /* inputs */ });

    Stack->Submit(FInstancedStruct::Make(Bridge));
    ```

!!! note "The framework warns you if you slip"
    In development builds, the first time a Blueprint command with state stored
    *outside* `Payload` runs, the framework logs a warning naming the offending
    properties. Treat that warning as a bug: move that state into `Payload`.

## Verify a command against the correctness rules

The framework ships a **conformance harness** that turns the command contract into
test failures. You register a set of cases for your own commands; the harness
applies, undoes, and redoes each one and checks the state comes back
byte-for-byte. A command that honors the rules can be *proven* to honor them.

Per case the harness asserts:

- the state after Undo equals the state before Apply (the inverse is exact);
- the state after Redo equals the state after the first Apply (Apply is
  deterministic, capture is absolute);
- a command that returns `false` mutated **nothing** (atomic fail — the most
  dangerous rule to break).

Register a case set at module startup. You provide a fixture (a seeded world and a
canonical hash of its authoritative state) and a list of command factories:

```cpp
// In your module's StartupModule():
FPCsConformanceRegistry::Get().Register([]
{
    FPCsConformanceCaseSet Set;
    Set.Name = TEXT("MyGame");

    Set.BuildFixture = [](FPCsConformanceFixture& Fixture) -> bool
    {
        UScoreBoard* Board = /* stand up a game instance + subsystems, seed state */;
        Fixture.Stack     = UPCsCommandStack::Get(Board);
        Fixture.HashState = [Board] { return Board->HashAuthoritativeState(); };
        Fixture.TearDown  = [] { /* destroy the world you built */ };
        return Fixture.IsUsable();
    };

    // A normal case: the command must apply, undo, and redo cleanly.
    Set.Cases.Add({ TEXT("set score"),
        [](const FPCsConformanceFixture&) { return FInstancedStruct::Make(FCmd_SetScore{ 120 }); } });

    // A deliberate atomic-fail probe: Apply must return false and change nothing.
    FPCsConformanceCase Reject;
    Reject.Name = TEXT("rejects invalid target");
    Reject.MakeCommand = [](const FPCsConformanceFixture&) { return FInstancedStruct::Make(FCmd_SetScore{ 120 }); };
    Reject.bExpectApplyFailure = true;
    Set.Cases.Add(Reject);

    return Set;
});
```

Run the harness as an automation test (editor closed, headless) — one command:

```text
UnrealEditor-Cmd.exe "<YourProject>.uproject" -ExecCmds="Automation RunTests PolyhedralFramework.CommandSystem.ConformanceBench; Quit" -unattended -nopause -nosplash -nullrhi
```

Any broken invariant is reported as a named test failure identifying the case and
which rule it violated.

!!! tip "Blueprint commands are checked too"
    When a case submits a Blueprint command, the harness also verifies the
    Blueprint class keeps all its state in `Payload` and fails if it doesn't — the
    same check behind the development-build warning above. The state hash is
    yours to define, so it hashes exactly your authoritative state and ignores
    transient or presentation fields.
