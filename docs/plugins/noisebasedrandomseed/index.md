# NoiseBasedRandomSeed

For developers who need **reproducible** randomness — loot rolls, procedural
layouts, AI choices — that survives a save/reload or a replay and comes out
identical every time. After this section you'll know how randomness is keyed by a
**domain**, the difference between a **sequential** draw and a **keyed** one (and
why one is a Callable node and the other is Pure), and how to snapshot and restore
the whole RNG state.

A **domain** is a named RNG stream identified by a gameplay tag — `Domain.LootTable`,
`Domain.EnemyAI`, `Domain.MapGen`, or the built-in `Domain.Default`. Each domain
gets its own seed (derived from one root seed plus the tag) and its own cursor, so
draws in one domain never disturb another: shuffling loot doesn't shift what the AI
rolls next. Under the hood it's SquirrelNoise5 — fast, seekable, and deterministic.

!!! note "A determinism primitive, not a gameplay system"
    NoiseBasedRandomSeed knows nothing about entities, commands, or turns — it just
    hands out numbers that are reproducible from a seed. It's the RNG other systems
    draw from when a result has to replay identically. If you need "same seed, same
    outcome" across [undo, save, and replay](../../concepts/commands-and-undo.md),
    draw from here instead of `FMath::Rand`.

## Two ways to draw

Every domain supports two draw styles, and the difference between them is the single
most important thing to understand:

- **Sequential** draws advance the domain's cursor. Each call returns the *next*
  number in that domain's stream — call it three times and you get three different
  results, in a fixed order. This is your everyday "give me a random number" call.
- **Keyed** draws are a stateless lookup: you pass an explicit integer **key** and
  get the number that key maps to, the same value every time, forever. Nothing
  advances. Reach for a keyed draw when a result must be tied to a stable identity
  (a tile coordinate, an entity id, a card slot) rather than to call order.

!!! warning "Sequential draws are Callable nodes, never Pure — on purpose"
    Sequential functions are exposed as **BlueprintCallable** (they have an exec
    pin), *not* as pure nodes. A pure node re-evaluates once **per connected output
    pin**, and the compiler is free to reorder it — so a pure "next random" would
    fire an unpredictable number of times in an unpredictable order and silently
    corrupt the stream. Routing the exec pin pins down exactly when, and how many
    times, the cursor advances. Keyed draws mutate nothing, so they *are* pure — wire
    them like any other getter.

## Save, load, replay

Because a domain's state is just a root seed plus a per-domain cursor, the whole RNG
can be snapshotted to a plain struct (`FRngSaveData`) and restored later. Embed that
struct in your `USaveGame`, and a loaded game keeps drawing exactly where it left
off; feed the same seed and the same call sequence and you reproduce a run
number-for-number. That's the property replays and deterministic undo lean on.

## The front door and the engine

There are two ways in, and they are the *same* API seen from two places:

- **`UNoiseSeedLibrary`** is the Blueprint front door — static nodes that take a
  **World Context** and resolve the subsystem for you. Reach for it from Blueprint,
  and from anywhere you just want to draw a number. It also hosts the array helpers
  (shuffle, pick-one, pick-N).
- **`UNoiseSeedSubsystem`** is the engine the library calls into — a Game Instance
  subsystem that owns the root seed and the per-domain state. Grab it directly when
  you need the parts the library doesn't wrap: **seeding** (`SetNoiseSeed`,
  `ResetDomain`) and **save/load** (`GetSaveData`, `ApplySaveData`).

Don't treat them as two systems to learn — learn the domains-and-draws model once;
the library is just the convenient way to call it.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UNoiseSeedLibrary` | The Blueprint front door: sequential draws, keyed draws, and the array helpers, each taking a domain tag. |
| `UNoiseSeedSubsystem` | The Game Instance subsystem behind it — owns the root seed and per-domain state; adds seeding and save/load. |
| `FRngSaveData` | The serializable snapshot (root seed + every domain's cursor) you embed in a `USaveGame`. |
| `FRngDomainState` | One domain's saved state — its tag, seed, and next index. |
| `Domain.Default` | The built-in domain tag every node defaults to; author your own domains under `Domain.*`. |

## Where to go next

- **[API Reference](reference.md)** — the full public surface: seeding, sequential
  and keyed draws, the array helpers, and the save/load structs.
- **[Commands & Undo](../../concepts/commands-and-undo.md)** — the undo/replay model
  this RNG's determinism feeds.
