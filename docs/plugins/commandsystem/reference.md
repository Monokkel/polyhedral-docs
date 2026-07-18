# CommandSystem API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Commands & Undo](../../concepts/commands-and-undo.md).

All public types carry the `PCs` prefix. Signatures below are hand-written to show
the shape of each API; they are illustrative, not source excerpts.

## The command stack

`UPCsCommandStack` is a `UGameInstanceSubsystem` — there is exactly one stack per
running game instance, and every authoritative mutation is submitted to it.

```cpp
// Resolve the stack through any world-context object. In Blueprint the context
// defaults to self.
static UPCsCommandStack* Get(const UObject* WorldContextObject);
```

It has one authored setting:

- **`MaxUndoDepth`** — the maximum number of undo entries retained (default `100`).
  When the undo stack grows past it, the oldest entry is dropped. A transaction
  counts as one entry.

## Submit, undo, and redo

Submitting applies a command immediately; if its `Apply` returns `true` the
command is retained (on the undo stack, or as a child of the open transaction).
Undo reverses the top entry and moves it to the redo stack; Redo re-applies it.

```cpp
// Apply Command now. Returns true if Apply succeeded and the command was retained.
// The inner struct must be (or derive from) FPCsCommand. UserData is optional
// per-run data handed to the command through the context.
bool Submit(const FInstancedStruct& Command, UObject* UserData = nullptr);

// Reverse the top undo entry; move it to the redo stack. Returns true if an entry
// was undone. A no-op (and false) if the undo stack is empty.
bool Undo(UObject* UserData = nullptr);

// Re-apply the top redo entry; move it back to the undo stack. Returns true if an
// entry was redone.
bool Redo(UObject* UserData = nullptr);

// True if there is anything to undo / redo. Bind these to button enabled state.
bool CanUndo() const;
bool CanRedo() const;

// Drop both stacks entirely. History is gone; the world is left as-is.
void Clear();

// How many entries each stack currently holds.
int32 GetUndoDepth() const;
int32 GetRedoDepth() const;

// The label (from the command's Describe) of the entry that the next Undo / Redo
// would act on — for a history label or tooltip. Empty when the stack is empty.
FString PeekUndoDescription() const;
FString PeekRedoDescription() const;
```

!!! warning "Submit, don't call Apply"
    Always route a command through `Submit`. Calling a command's `Apply` yourself
    changes the world without recording anything on the stack, so that change will
    never undo, save, or replay. The same applies to reactive listeners: they may
    never call `Submit`, `Undo`, or `Redo` — see
    [Stack events](#stack-events) and
    [Derived State & Events](../../concepts/derived-state.md).

## Transactions

A transaction groups several submitted commands into **one undo step**. Wrap a
multi-step action so a single Undo reverses the whole thing.

```cpp
// Open a transaction. Subsequent Submits are collected into it rather than pushed
// individually. The description labels the resulting undo entry.
void BeginTransaction(const FString& Description);

// Balance one BeginTransaction. The outermost Commit pushes the grouped command
// onto the undo stack (unless it collected nothing); inner commits just unwind a level.
void CommitTransaction();

// Roll back the changes made since the matching BeginTransaction, in reverse
// order, and unwind that level.
void AbortTransaction();

// True while a transaction is open.
bool IsTransactionOpen() const;
```

Semantics worth knowing:

- **Nesting joins, it doesn't stack.** A nested `BeginTransaction` joins the
  already-open transaction (one flattened undo unit) rather than starting an
  independent one. Only the outermost `CommitTransaction` commits to the undo
  stack. Every `BeginTransaction` must be balanced by exactly one
  `CommitTransaction` or `AbortTransaction`.
- **Abort keeps applied effects undoable.** `AbortTransaction` reverses the
  children added at the current level. If a child refuses to reverse, the ones
  already rolled back are re-applied and the group is committed to the undo stack
  instead of being discarded — so a partially-applied effect is never orphaned off
  the stack.
- **Submits inside a transaction don't settle.** History-navigation events fire at
  the transaction boundary, not per child (see [Stack events](#stack-events)).

## Execution phase

While the stack is working, it exposes *what* it's doing. Reactive consumers use
this to make presentation choices — for example, snap a token to its new cell on
an undo instead of animating.

```cpp
// What the stack is currently executing. Idle when nothing is running.
EPCsExecutionPhase GetExecutionPhase() const;

enum class EPCsExecutionPhase : uint8
{
    Idle,       // nothing executing
    Command,    // a live forward Submit
    Undo,       // history moving backward
    Redo,       // history moving forward again
};
```

!!! note "Rollbacks report Undo"
    Anything that moves state *backward* reports `Undo`, even mid-forward — a
    transaction child failing during Apply, or an `AbortTransaction`. Consumers
    that watch state move backward can treat all of these the same way. The phase
    stays valid through the operation's settle notification, so a listener that
    flushes at settle time still sees the operation's phase. Use the phase for
    presentation only, never as a gameplay filter.

## Stack events

The stack broadcasts as it changes. One event is Blueprint-facing; the rest are
C++-only observer taps for tooling such as a replay recorder or telemetry.

```cpp
// Blueprint-assignable. Fires whenever the stack's contents change — after a
// Submit, Undo, Redo, Commit, Abort, or Clear. Bind it to refresh Undo/Redo
// button state and any history label. Carries no parameters.
FPCsOnStackChanged OnStackChanged;   // dynamic multicast, no args
```

C++-only delegates (bind from native code):

- **`OnSettled`** `void()` — fires after a top-level command boundary finishes:
  a Submit outside a transaction, a Commit, an Abort, an Undo, a Redo, or a Clear.
  Submits *inside* an open transaction do not settle. Broadcast after execution
  ends, so a listener here may itself submit.
- **`OnEntryCommitted`** `void(const FInstancedStruct&)` — fires exactly once per
  new forward undo entry committed (a non-transactional Submit or the outermost
  Commit). Never fires for Undo/Redo/Clear or from inside an open transaction. The
  payload is the committed command. Ideal for a replay log.
- **`OnUndone`** `void()` / **`OnRedone`** `void()` — fire once per Undo / Redo
  call that actually moved an entry between stacks (not for an empty-stack no-op).
- **`OnCleared`** `void()` — fires once per Clear.

!!! warning "Observers are read-only"
    Every delegate here is a read-only observer tap. Never `Submit`, `Undo`,
    `Redo`, or open/close a transaction from one of these callbacks — the stack
    rejects re-entrant operations loudly in development builds, because a listener
    runs inside command execution (including during undo and redo). Reactions that
    *mutate* game state are game rules, not observers, and belong to the event
    system's reaction mechanism. See
    [Derived State & Events](../../concepts/derived-state.md).

## Command types

These are the building blocks a custom C++ command is made of.

**`FPCsCommand`** is the virtual `USTRUCT` base for every command. A custom command
derives from it and overrides `Apply` and `Undo`; both take a context and return
`true` on success. A command captures its own inverse state — the previous values
it will restore — as members written during `Apply`.

```cpp
struct FPCsCommand   // USTRUCT base; derive your command from this
{
    // Apply the change. Return false — WITHOUT mutating anything — to reject it.
    // Called for both the first Submit and every Redo, so it must be deterministic
    // given current world state.
    virtual bool Apply(FPCsCommandContext& Ctx);

    // Reverse exactly what Apply did, using the inverse state Apply captured.
    virtual bool Undo(FPCsCommandContext& Ctx);

    // A short label for the undo history.
    virtual FString Describe() const;
};
```

**`FPCsCommandContext`** is handed to `Apply` and `Undo`. It's how a command reaches
its target subsystem.

```cpp
struct FPCsCommandContext
{
    UObject* WorldContext = nullptr;  // resolve your target subsystem from this
    UObject* UserData     = nullptr;  // optional opaque per-run data (from Submit/Undo/Redo)
};
```

**`FPCsCompositeCommand`** is the grouping a transaction produces: a list of child
commands that apply in order and undo in reverse, as one step. You rarely construct
one directly — `BeginTransaction` / `CommitTransaction` build it for you.

```cpp
struct FPCsCompositeCommand : public FPCsCommand
{
    FString Description;                    // the transaction's label
    TArray<FInstancedStruct> Children;      // each is an FPCsCommand
};
```

## Blueprint commands

To author a command in Blueprint, subclass `UPCsBlueprintCommand` and override its
events. The bridge struct `FPCsCmd_Blueprint` is what actually runs it on the
stack.

```cpp
// Subclass this in Blueprint; override the three events.
class UPCsBlueprintCommand : public UObject   // Abstract, Blueprintable
{
    // Apply the change; return false (mutating nothing) to reject it.
    bool BP_Apply();

    // Reverse it using state read back from Payload.
    bool BP_Undo();

    // A short label for the undo history.
    FString BP_Describe();

    // Inputs AND captured inverse state. Write your "before" values here in
    // BP_Apply; the bridge carries this across the Apply/Undo/Redo cycle.
    FInstancedStruct Payload;

    // The world context for this run (resolve your subsystem from it).
    UObject* GetWorldContext() const;
};
```

!!! warning "All inverse state lives in Payload"
    A fresh command object is instantiated for every `BP_Apply` and every
    `BP_Undo` call, so state stored in an ordinary member variable does **not**
    survive between them and will silently corrupt undo. Everything `BP_Undo`
    needs must be written into `Payload` during `BP_Apply`. See
    [Write a Blueprint command](guides.md#write-a-blueprint-command).

```cpp
// The bridge: what you actually Submit to run a Blueprint command.
struct FPCsCmd_Blueprint : public FPCsCommand
{
    TSubclassOf<UPCsBlueprintCommand> CommandClass;  // your Blueprint class
    FInstancedStruct                  Payload;       // inputs + captured state
};
```

Submit it like any other command:

```cpp
FPCsCmd_Blueprint Bridge;
Bridge.CommandClass = MyBlueprintCommandClass;
Bridge.Payload      = FInstancedStruct::Make(FMyCmdPayload{ /* inputs */ });
Stack->Submit(FInstancedStruct::Make(Bridge));
```

## Conformance harness

A public test surface that turns the command contract into test failures. You
register a set of cases for your own commands; a framework automation test builds
every registered set and, per case, checks that Undo exactly reverses Apply, that
Redo reproduces Apply, and that a rejected command (`Apply` returning `false`)
mutated nothing. Full walkthrough:
[Verify a command against the correctness rules](guides.md#verify-a-command-against-the-correctness-rules).

```cpp
// Process-wide registry. Register a case-set factory from your module's startup.
class FPCsConformanceRegistry
{
    static FPCsConformanceRegistry& Get();
    void Register(TFunction<FPCsConformanceCaseSet()> CaseSetFactory);
    TArray<FPCsConformanceCaseSet> BuildAll() const;   // used by the test runner
};
```

The three structures you fill in:

```cpp
// A seeded world for one run, plus how to hash its authoritative state.
struct FPCsConformanceFixture
{
    TStrongObjectPtr<UObject> ContextObject;  // keeps the world's root alive
    UPCsCommandStack*         Stack;          // where cases are submitted
    TFunction<uint64()>       HashState;      // canonical hash of YOUR state
    TFunction<void()>         TearDown;        // optional cleanup after the set

    bool IsUsable() const;                    // Stack && HashState set
};

// One case: a factory producing a configured command against the current fixture.
struct FPCsConformanceCase
{
    FString                                          Name;
    TFunction<FInstancedStruct(const FPCsConformanceFixture&)> MakeCommand;
    bool bExpectApplyFailure = false;         // true for atomic-fail probes
};

// A client's fixture builder plus its cases.
struct FPCsConformanceCaseSet
{
    FString                       Name;
    TFunction<bool(FPCsConformanceFixture&)> BuildFixture;  // false if unhostable
    TArray<FPCsConformanceCase>   Cases;
};
```

The hash is *yours* to define because only the state owner knows what its
authoritative state is (a generic hash would include pointers and transient
fields). Cases run sequentially against one fixture and leave their applied state
in place, so a later case sees the world an earlier one produced.

For Blueprint commands there's also a reflection check you can call directly:

```cpp
namespace PCsConformance
{
    // Returns the names of any properties a UPCsBlueprintCommand subclass declares
    // outside the sanctioned surface (Payload) — each a potential silent-undo bug.
    // Empty means the class is conformant.
    TArray<FString> FindNonPayloadProperties(const UClass* CommandClass);
}
```

The harness runs this automatically for any Blueprint command in a case set, and
the framework logs the same finding as a warning the first time a nonconformant
Blueprint command runs in a development build.
