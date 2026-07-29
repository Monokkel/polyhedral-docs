# API Reference: Commands & Replay

This page is for developers who want undo, redo, and replay without hand-writing
the machinery. It catalogues the ready-made, **command-driven** mutators
GameEntity ships — you call a subsystem method and the right command is submitted
for you — and it covers recording and replaying the authoritative command
timeline (the **replay log**).

New to the idea of a single command stack behind every change? Read the
[Commands & Undo concept page](../../concepts/commands-and-undo.md) first — it
explains the one-door rule, transactions (one undo step), and why commands must
undo themselves exactly. This page assumes that background and lists what
GameEntity provides on top of it.

## Overview — you rarely write commands

Every authoritative change an entity undergoes is a command submitted to the
command stack. But you almost never construct one of those commands yourself.
The mutation methods on the game-state subsystem
([`UPGeGameStateSubsystem`](reference-entities.md)) are already command-driven:
call **Modify Stat**, **Add Tag**, **Add Child**, and the method builds the
matching command, submits it, and returns. You get undo, redo, save, and replay
with nothing extra to write.

!!! tip "The rule of thumb"
    If a method *changes* an entity, it is command-driven — the change is on the
    undo stack the moment the call returns. If a method only *reads*, it is a
    plain function call and touches no stack.

Undo, redo, and transactions themselves are not on this subsystem — they live on
the command stack. To reverse a change, grab the stack and call **Undo**/**Redo**;
to make several changes into one undo step, wrap them in a transaction. Both are
documented in the [CommandSystem plugin](../commandsystem/index.md)
([reference](../commandsystem/reference.md)).

## The built-in command catalogue

Each method below submits one command (or, for a few, brackets a small group in a
single transaction). Call the method — you never build the command struct. The
**Command** column names the entry that lands on the undo stack, which is what
you see if you inspect stack history.

Every method resolves the entity through an [`FPGeEntityRef`](reference-entities.md);
tags are `Data.*` / `Slot.* ` / stat gameplay tags as appropriate. Stat and
modifier semantics (whole-unit base stats, resolved current stats) are detailed
in the [stats reference](reference-stats.md); the change events these commands
broadcast are in the [events reference](reference-events.md).

### Entity lifecycle

| Subsystem method | What it does | Command |
|---|---|---|
| `CreateEntity()` | Create an empty entity, return its ref | `FPGeCmd_CreateEntity` |
| `CreateEntityFromData(EntityData)` | Create by copying entity data | `FPGeCmd_CreateEntity` |
| `CreateEntityFromRow(RowHandle, bRegisterAsTemplate)` | Create from a data-table row | `FPGeCmd_CreateEntity` |
| `CreateEntityFromRegistry(RegistryId, bRegisterAsTemplate)` | Create from a Data Registry item | `FPGeCmd_CreateEntity` |
| `CreateEntityFromTemplate(TemplateId)` | Create linked to a registered template | `FPGeCmd_CreateEntity` |
| `CreateEntityFromTemplateWithData(TemplateId, InitialTaggedData)` | Create from a template, pre-seeded with tagged-data overrides | `FPGeCmd_CreateEntity` |
| `DestroyEntity(EntityRef)` | Destroy an entity; detaches from its parent and orphans its children | `FPGeCmd_DestroyEntity` |

### Tags

| Subsystem method | What it does | Command |
|---|---|---|
| `AddTag(EntityRef, Tag)` | Add a gameplay tag | `FPGeCmd_AddTag` |
| `RemoveTag(EntityRef, Tag)` | Remove a gameplay tag | `FPGeCmd_RemoveTag` |

!!! warning "`RemoveTag` refuses a tag that mirrors stored data"
    A tagged-data entry is readable only while its mirror tag is present, so
    stripping that tag with `RemoveTag` would leave a value stored and unreachable.
    The call **returns `false` and logs** instead, naming the route you wanted:
    `RemoveTaggedData` to clear the entity's own value, or
    `ShadowTemplateTaggedData` to hide a value its template supplies. Ordinary tags
    are unaffected — this only bites when the tag is a mirror.

### Base stats

| Subsystem method | What it does | Command |
|---|---|---|
| `SetStat(EntityRef, StatTag, Value)` | Set a base stat (creates it if absent) | `FPGeCmd_SetStat` |
| `ModifyStat(EntityRef, StatTag, Delta)` | Add a delta to a base stat, returns the new value | `FPGeCmd_ModifyStat` |
| `ModifyStats(EntityRef, Deltas)` | Apply several deltas as one batch (each is its own undoable command) | `FPGeCmd_ModifyStat` (one per entry) |
| `RemoveStat(EntityRef, StatTag)` | Remove a base stat entirely | `FPGeCmd_RemoveStat` |

### Stat modifiers

| Subsystem method | What it does | Command |
|---|---|---|
| `AddStatModifier(EntityRef, Spec, SourceRef)` | Add a runtime modifier, returns a handle | `FPGeCmd_AddStatModifier` |
| `AddFlatModifier(EntityRef, Stat, Amount, SourceRef)` | Sugar: add a flat `+N` / `-N` modifier | `FPGeCmd_AddStatModifier` |
| `AddPercentModifier(EntityRef, Stat, Percent, SourceRef)` | Sugar: add a `±N%` modifier | `FPGeCmd_AddStatModifier` |
| `RemoveStatModifier(Handle)` | Remove one modifier by handle | `FPGeCmd_RemoveStatModifier` |
| `UpdateStatModifier(Handle, NewSpec)` | Swap a live modifier's spec in place | `FPGeCmd_UpdateStatModifier` |
| `RemoveStatModifiersBySource(EntityRef, SourceRef)` | Remove all modifiers from one source entity, returns the count | `FPGeCmd_RemoveStatModifiersBySource` |
| `RemoveStatModifiersBySourceTag(EntityRef, SourceTag)` | Remove all modifiers matching a source tag | `FPGeCmd_RemoveStatModifiersBySourceTag` |
| `RemoveStatModifiersByScope(EntityRef, ScopeRef)` | Remove all modifiers stamped to an expiry scope | `FPGeCmd_RemoveStatModifiersByScope` |

### Tagged data

| Subsystem method | What it does | Command |
|---|---|---|
| `SetTaggedData(EntityRef, Tag, Value)` | Set a typed struct value keyed by tag | `FPGeCmd_SetTaggedData` |
| `RemoveTaggedData(EntityRef, Tag)` | Clear this entity's own value, falling back to its template's | `FPGeCmd_RemoveTaggedData` |
| `ShadowTemplateTaggedData(EntityRef, Tag)` | Make the key **absent for this entity**, even when its template supplies a value | `FPGeCmd_RemoveTaggedData` |

Note the difference between the last two. `RemoveTaggedData` clears an override, so
a template-provided value becomes visible again; `ShadowTemplateTaggedData` makes
the key absent outright for that one entity, without touching the shared template.
Both are ordinary undoable commands. See the
[entities & tagged-data reference](reference-entities.md) for what tagged data is
and how reads delegate to a template.

### Hierarchy (parent / child)

| Subsystem method | What it does | Command |
|---|---|---|
| `AddChild(ParentRef, ChildRef, SlotTag)` | Attach a child, optionally into a `Slot.*` | `FPGeCmd_AddChild` |
| `RemoveChild(ParentRef, ChildRef)` | Detach a child, clearing its slot tags | `FPGeCmd_RemoveChild` |
| `SetChildSlot(ChildRef, NewSlotTag)` | Move a child to a different slot | `FPGeCmd_SetChildSlot` |

### Expiry-scope bookkeeping

The framework also ships two commands that maintain a scope's close ledger
(`FPGeCmd_AppendScopeLedgerEntry`, `FPGeCmd_RemoveScopeLedgerEntry`). You do not
call these directly — the expiry / end-condition helpers (stamp a modifier
"until the scope closes", "until unequipped", and so on) submit them for you, and
`DestroyEntity` replays a closing entity's ledger inside its own transaction. The
end-condition authoring surface is covered in the
[stats reference](reference-stats.md).

### Why undo is exact

You do not need to reason about how these commands reverse themselves — that is
the framework's job — but it is worth knowing the guarantee. Every built-in
command records, when it applies, the exact prior state it will need to restore —
not the change it made — so the reversal is always exact, even when a value was
clamped on the way in.

- Setting or modifying a stat records the previous value *and* whether the stat
  existed at all, so undo restores the value or removes the stat again.
- Removing a modifier records its full entry *and its position in the list*, so
  undo re-inserts it in the same place (order can matter for how modifiers stack).
- Detaching or destroying records the parent link, slot tags, and array position,
  so undo re-links everything exactly as it was.

Because the inverse is captured at apply time, undo and redo restore state
precisely — including after the change was clamped, joined a transaction, or
cascaded into reactions.

## Writing a custom command

You only write a command when you add a **new kind of state** the framework does
not already manage — a turn counter, a resource pool, a bespoke tracker. Any
change to an *entity* is already covered by the catalogue above.

A custom command is a small struct that knows how to apply a change and how to
reverse it, submitted through the command stack. The four correctness rules
(record don't diff, check before you mutate, stay repeatable, reverse
everything), the Blueprint and C++ authoring shapes, and the conformance test
harness are all documented in the CommandSystem plugin:

- Concept and the four rules — [Commands & Undo](../../concepts/commands-and-undo.md#writing-a-custom-command).
- Full API and the correctness harness — [CommandSystem reference](../commandsystem/reference.md).

## Recording a replay

The **replay log** is the persisted, authoritative command timeline. Recording it
writes each committed command to a `.pgereplay` file under `Saved/Replays/`,
alongside a snapshot of the starting state and the random seed — everything needed
to reconstruct the session from the beginning.

Recording is driven by the replay subsystem, `UPGeReplaySubsystem`
(`GameEntity | Replay`).

| Member | Purpose |
|---|---|
| `StartRecording(Label)` | Begin a new recording. Returns `false` if already recording or called mid-change. |
| `StopRecording()` | Finish the recording, write a closing checkpoint, and close the file. |
| `IsRecording()` | Whether a recording is in progress. |
| `RequestCheckpoint(Label)` | Write a labelled integrity checkpoint at this point (no-op if not recording). |
| `GetCurrentFilePath()` | The file currently being written (empty when idle). |

!!! note "Start and stop only at a settled point"
    Recording may only start or stop at a **settled, idle boundary** — no open
    transaction is mid-flight. Both calls return `false` (with a log warning) if
    the boundary is illegal, so check the return value rather than assuming
    success.

!!! tip "A recording that has captured nothing yet will re-anchor itself"
    A recording embeds a baseline describing the state it starts from. If a
    [state slice](#extending-the-state-hash-and-the-save-file) registers *after* the
    recording began — the ordinary case, since a world-scoped system comes up later
    than a game-instance one — that baseline would describe a different composition
    from every checkpoint after it, which reads back as a divergence on the very
    first frame.

    So the recorder watches for the composition changing. While the recording is
    still empty (nothing but its anchor on disk), it silently rewrites the baseline
    on the new composition. Once real frames exist it refuses and warns instead —
    re-anchoring there would rewrite history. In practice this means you can let
    auto-recording start early and register slices at world start-up without
    sequencing the two by hand; what you must not do is register a new slice
    *mid-recording* and expect the log to stay verifiable.

Only the *authoritative* timeline is recorded. When the ability system (documented in
[Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#what-if-runs-previews-and-enemy-ai))
simulates an action in advance — to preview it or let AI weigh
it — that look-ahead never touches real state and is **invisible** to the
recording, so a replay always reflects what actually happened.

=== "Blueprint"
    1. Get the **Replay** subsystem.
    2. At a clean point (the start of a match, say), call **Start Recording** with
       a label. Branch on the returned bool.
    3. Play normally — every command commits into the file automatically.
    4. Call **Stop Recording** to close the file. **Get Current File Path** tells
       you where it landed.

=== "C++"
    ```cpp
    if (UPGeReplaySubsystem* Replay = UPGeReplaySubsystem::Get(this))
    {
        Replay->StartRecording(TEXT("Match 1"));
        // ... play the session; each command is appended as it commits ...
        Replay->RequestCheckpoint(TEXT("End of round 1")); // optional marker
        Replay->StopRecording();
    }
    ```

## Playing back a replay

Playback re-applies a recorded `.pgereplay` file against a fresh, idle game state
and verifies it reproduces the original run. It is the "single-steppable repro" of
a session: a bug you recorded can be replayed frame by frame. This surface is C++.

Load a file to get a replay handle:

```cpp
UPGeReplaySubsystem* Replay = UPGeReplaySubsystem::Get(this);
TSharedRef<FPGeReplayPlayer> Player = Replay->LoadReplay(FilePath);

if (!Player->GetReport().bLoaded)
{
    // Refused: the report's RefusalReason says why (e.g. a newer file version).
    return;
}
```

`LoadReplay` restores the embedded starting snapshot and seed, then hands back an
`FPGeReplayPlayer` you drive:

| Member | Purpose |
|---|---|
| `StepOne()` | Process the next frame and advance. Returns `false` at the end or on a stop. |
| `RunToCheckpoint()` | Step until the next integrity checkpoint has been verified. |
| `RunToEnd()` | Run to the end of the timeline (or the first divergence). |
| `GetReport()` | The verification report (see below). |
| `IsAtEnd()` / `GetCursor()` | Cursor position within the timeline. |

The `FPGeReplayReport` records what happened: `bLoaded` / `bRefused`, how many
command frames applied, per-checkpoint pass/fail, and whether playback
**diverged** (a frame that failed to re-apply, or a checkpoint whose state hash no
longer matched). `Passed()` is the single verdict — loaded, not refused, no
divergence, every checkpoint matched:

```cpp
Player->RunToEnd();

const FPGeReplayReport& Report = Player->GetReport();
if (Report.Passed())
{
    // The timeline replayed identically to the original run.
}
else if (Report.bDiverged)
{
    // Report.FirstDivergenceFrame / DivergenceDetail pinpoint where it drifted.
}
```

By default playback stops at the first divergence (the minimal repro). Pass
`EPGeReplayDivergencePolicy::ContinueAndReport` to `LoadReplay` to press on and
collect every mismatch instead.

!!! note "Playback needs a clean fixture"
    A replay loads only into an idle game state (no open transaction) and refuses
    while the same subsystem is recording. A refused load changes nothing and
    returns an inert handle whose report carries the reason — always check
    `GetReport().bLoaded` before stepping.

## Replay settings and checkpoints

Replay options live in **Project Settings → Plugins → Game Entity Replay**
(`UPGeReplaySettings`).

| Setting | Meaning |
|---|---|
| **Auto Record In Development** | Start recording automatically when the subsystem initializes. On in development builds, off in shipping. A game can always record explicitly with `StartRecording` regardless. |
| **Checkpoint Every Entry** | Write an integrity checkpoint after *every* command, not just at natural boundaries. Costly — a diagnostic aid for narrowing down exactly where a replay diverges. Off by default. |

**Checkpoints** are the log's integrity marks. Each records the **state hash** at
that moment — entity state plus every registered state slice; on playback the hash
is recomputed and compared, and a mismatch is a divergence. The framework writes them automatically at meaningful boundaries — the
start and end of a recording, turn advances, and scope closes — and you can add
your own labelled marks with `RequestCheckpoint(Label)` around anything you want
to be able to verify.

## Extending the state hash and the save file

Everything above assumes the state worth protecting is *entity* state. Often it
isn't all of it. If you build a system with its own authoritative substrate — a
board's cell graph, a deck's shuffle state, a fog-of-war grid — that state needs to
join the same guarantees: it must fold into the **state hash** so a replay
checkpoint catches it drifting, and it must ride along in the **save file** so a
load restores it.

You do that by registering a named **state slice** with the game-state subsystem: a
deterministic encoder that appends your bytes, and an optional decoder that applies
them back. From then on your state is part of the framework's one digest and one
save, under one version stamp — you do not maintain a parallel save path.

!!! note "C++ only"
    This is a C++ extension surface — there are no Blueprint nodes for it. It is
    aimed at a plugin or module author adding a new kind of state, not at content
    authoring.

```cpp
// What you supply. The encoder appends your slice's deterministic byte stream; the
// decoder applies a saved stream back onto your live substrate.
using FPGeCanonSliceEncoder = TFunction<void(TArray<uint8>& OutBytes)>;
using FPGeCanonSliceDecoder = TFunction<bool(const TArray<uint8>& Bytes, uint32 SliceVersion)>;

// Register under a name, with your own version number for your own byte format.
// Returns false if that name is already registered. A null decoder gives you a
// hash-only slice: it folds into the hash and is written to saves, but a loaded
// blob for it is skipped with a warning.
bool RegisterCanonSliceEncoder(FName SliceName, uint32 SliceVersion,
                               FPGeCanonSliceEncoder Encoder,
                               FPGeCanonSliceDecoder Decoder = nullptr);

void UnregisterCanonSliceEncoder(FName SliceName);   // silent no-op if absent
bool IsCanonSliceRegistered(FName SliceName) const;
int32 GetCanonSliceCount() const;
```

Registering is a few lines at the point your substrate comes up:

```cpp
UPGeGameStateSubsystem* State = UPGeGameStateSubsystem::Get(this);

State->RegisterCanonSliceEncoder(
    TEXT("Deck"), /*SliceVersion=*/1,
    [this](TArray<uint8>& OutBytes)          { EncodeDeckState(OutBytes); },
    [this](const TArray<uint8>& Bytes, uint32 Version)
    {
        return DecodeDeckState(Bytes, Version);   // false = nothing was touched
    });
```

Four rules to build against:

- **Encoding must be deterministic.** The same substrate must produce the same
  bytes every time, on every machine — that is the whole point of folding into a
  hash. All registered slices are folded in sorted registration-name order, so the
  order you register in never affects the digest.
- **A decoder parses fully before it mutates anything.** Returning `false` must
  mean *nothing was touched* — an atomic fail. Finish by broadcasting whatever
  wholesale-resync signal your own consumers rebuild from.
- **Decoders run before the entity-state swap.** So by the time change-event
  consumers see the state-restored notification, your substrate is already the
  restored one and they can rebuild against it.
- **Version your own bytes.** `SliceVersion` is yours, independent of the
  framework's stamps. A save carrying a *newer* version of your slice than the
  runtime registered refuses the whole load rather than guessing.

!!! warning "Registration is per game instance, and it is not permanent"
    Slices are registered on the **game-state subsystem instance**, not
    process-wide. Two game instances in one process (a Play-In-Editor multiplayer
    session, say) each carry their own registrations, and neither can stomp the
    other's. The practical consequence: re-register when your substrate comes up in
    a world, and do not assume a registration made in one session is visible in
    another.

    Because the set of slices can change during a session, anything holding a
    digest taken earlier is describing a different composition afterwards. The
    subsystem publishes a C++ notification when the set changes so a holder can
    react; the replay recorder is the framework's own consumer of it (see below).
    A listener on it observes only — it must never mutate state or submit a
    command.

Two hashing functions exist, and on an instance carrying slices they answer
differently. The instance's own `HashState()` folds registered slices;
`HashGameState(Snapshot)` is a pure function of an entity-state snapshot and never
does. `HashStateUncounted()` is the composed digest again, for a development-time
comparison that must not perturb the framework's own accounting. If you are
comparing two numbers and they disagree, check you are not comparing across that
line.

### What a save reports about slices

A load never silently drops a slice. `FPGeSaveLoadReport` gains
`UnappliedSlices` — the names of every slice that did not round-trip, in **both**
directions:

- a blob in the save with no registered slice or decoder to apply it (a save from a
  build that had a system this one doesn't), and
- a registered slice the save carried no blob for (an older save written before
  that slice existed).

Each is skipped with a warning and **the load proceeds**, leaving that slice's live
state untouched — so a game can inspect the list and decide whether to warn,
repair, or carry on. A blob that *is* routable but refuses — a newer slice version,
or a decoder that fails to parse — refuses the whole load instead, with `bRefused`
and a reason, matching how the framework treats every other version mismatch.

## Save and replay versioning

Both save files and replay logs carry **version stamps** in their headers — a
stamp for the state encoding and one for the file container format. On load, the
version is checked before anything is read:

- A file written by an **older** compatible build is migrated forward.
- A file written by a **newer** build than the runtime is **refused**, not
  guessed at — the load reports the refusal instead of silently mis-reading bytes.

That is all a consumer needs to know: an incompatible file is detected and
declined cleanly. The on-disk byte layout is an internal detail and is not part of
the public API. A refused replay load surfaces as `bLoaded == false` with a reason
on the report, exactly like the clean-fixture guard above.

Both stamps advanced when the state-slice seam landed: the state encoding now folds
registered slices into its digest, and a save now carries their byte streams. Both
migrations are free in the forward direction — a save written before the change
loads normally, with its (absent) slices reported in `UnappliedSlices` rather than
refused. With no slice registered at all, nothing about either format changes.
