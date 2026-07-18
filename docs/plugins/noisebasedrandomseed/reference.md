# NoiseBasedRandomSeed API Reference

For developers who want the complete public surface of the plugin. For the model and
when to use it, see the [overview](index.md).

Signatures below are hand-written to show the shape of each API; they are
illustrative, not source excerpts. The `UNoiseSeedLibrary` forms shown here take a
hidden **World Context** that Blueprint fills automatically — it's omitted from the
signatures for clarity. The `UNoiseSeedSubsystem` forms are the same calls without
that convenience argument.

## Domains & tags

A **domain** is an RNG stream named by a `FGameplayTag` under `Domain.*`. The plugin
ships one native tag, `Domain.Default` (`TAG_Domain_Default`), which every node's
`DomainTag` pin defaults to; author your own — `Domain.LootTable`, `Domain.EnemyAI`,
`Domain.MapGen` — under the same namespace. Blueprint pins are filtered to the
`Domain.*` namespace and show only leaf tags.

Each domain's seed is **derived**, not chosen: it comes from the run's root seed
combined with the domain's tag. That's what keeps domains independent — one root seed
fans out into a distinct, stable stream per tag, so you seed once and every domain
follows deterministically.

## Seeding & lifecycle

Seeding lives on the subsystem only — there is no library wrapper for it. Get the
`UNoiseSeedSubsystem` (a Game Instance subsystem) and call:

```cpp
// Set the root seed for this run. Every domain's seed re-derives from this plus its
// tag. bResetDomains rewinds every cursor to 0 (usually what you want on a new game).
void SetNoiseSeed(int32 NewSeed, bool bResetDomains = true);

int32 GetNoiseSeed() const;             // pure

void ResetAllDomains();                 // rewind every domain's cursor to 0
void ResetDomain(FGameplayTag DomainTag);   // rewind one domain's cursor
```

Two read-only properties report the current seed state:

```cpp
int32 NoiseSeed;      // BlueprintReadOnly — the active root seed
bool  bHasNoiseSeed;  // BlueprintReadOnly — true once a seed has been set
```

!!! warning "Seed state is read-only by design"
    `NoiseSeed` and `bHasNoiseSeed` are exposed for reading only. A direct write would
    bypass the seeding invariants (the derived per-domain seeds would no longer match
    the root). Change the seed through `SetNoiseSeed`, and rewind streams through
    `ResetDomain` / `ResetAllDomains` — never by poking the property.

## Sequential random

Each call advances the domain's cursor and returns the next value in its stream.
These are **Callable** nodes (they carry an exec pin), not pure ones.

```cpp
// Library front door (UNoiseSeedLibrary):
float RandomFloatInDomain(FGameplayTag DomainTag);                              // 0..1
float RandomFloatInRangeInDomain(FGameplayTag DomainTag, float Min, float Max = 1.f);
int32 RandomIntInRangeInDomain(FGameplayTag DomainTag, int32 Min, int32 Max);
bool  RandomBoolInDomain(FGameplayTag DomainTag, float TrueChance);            // chance 0..1
int32 RandomIndexInDomain(FGameplayTag DomainTag, int32 ArrayLength);          // 0..ArrayLength-1
```

The subsystem exposes the identical set under `Get…` names —
`GetRandomFloatInDomain`, `GetRandomFloatInRangeInDomain`,
`GetRandomIntInRangeInDomain`, `GetRandomBoolInDomain` (its `TrueChance` defaults to
`0.5`), and `GetRandomIndexInDomain`.

!!! warning "Don't 'clean these up' into pure nodes"
    It's tempting to make a random getter pure so it reads like a value. Don't. A pure
    node re-evaluates per connected output pin and can be reordered by the compiler, so
    the cursor would advance an unpredictable number of times in an unpredictable
    order — the stream stops being reproducible. The Callable exec pin is load-bearing:
    it fixes when and how often the draw happens.

## Keyed random

Keyed draws are stateless: the value is a pure function of the domain and an explicit
integer **key**. The same key in the same domain returns the same number every time,
and nothing advances — so these are genuinely **Pure** nodes.

```cpp
// Library front door (UNoiseSeedLibrary):
float KeyedRandomFloatInDomain(FGameplayTag DomainTag, int32 Key);                       // 0..1
int32 KeyedRandomIntInRangeInDomain(FGameplayTag DomainTag, int32 Key, int32 Min, int32 Max);
bool  KeyedRandomBoolInDomain(FGameplayTag DomainTag, int32 Key, float TrueChance);      // chance 0..1
```

The subsystem mirrors these as `GetKeyedRandomFloatInDomain`, `GetKeyedRandomIntInRange`,
and `GetKeyedRandomBoolInDomain`.

!!! note "Palette label quirk"
    The middle node's Blueprint display name currently reads *Keyed Random Float In
    Range* even though it returns an **integer** in the `Min…Max` range (the subsystem's
    matching call is the plainly-named `GetKeyedRandomIntInRange`). If you're hunting
    for a keyed integer draw, that "Float"-labelled node is the one.

## Array helpers

Three helpers on the library draw against an array using the domain's **sequential**
RNG — so, like the sequential draws above, they **advance the cursor** and are Callable
nodes. They're wildcard nodes: the item pins take on whatever element type you wire in,
so a single node works for an array of any type. Describe them by their pins, not by a
concrete element type.

| Node (display name) | Inputs | Outputs | Effect |
|---|---|---|---|
| **Shuffle Array (In Domain)** | the target array (by reference), `DomainTag` | — | Shuffles the wired array in place. |
| **Random Array Item (In Domain)** | the target array, `DomainTag` | `OutItem` (element type follows the array), `OutIndex` | Picks one element and reports its index. |
| **Random Array Items (In Domain)** | the target array, `NumItems`, `bAllowRepeats`, `DomainTag` | `OutItems` (element type follows the array), `OutIndices` | Picks `NumItems` elements — with replacement if `bAllowRepeats`, otherwise without — and reports their original indices. |

These live on the library only; there is no subsystem equivalent. Because they consume
the sequential stream, treat them exactly like a sequential draw for reproducibility —
route the exec pin, don't reorder them.

## Save / load

The subsystem can snapshot its entire state to a plain struct and restore it. Both
structs are `BlueprintType` and fully readable/editable, so the snapshot drops straight
into a `USaveGame`.

```cpp
void GetSaveData(FRngSaveData& OutData) const;   // snapshot the root seed + every domain
void ApplySaveData(const FRngSaveData& InData);  // restore that snapshot exactly

struct FRngSaveData
{
    int32 NoiseSeed;                  // the root seed
    TArray<FRngDomainState> Domains;  // one entry per active domain
};

struct FRngDomainState
{
    FGameplayTag DomainTag;
    int32 Seed;        // this domain's derived seed
    int32 NextIndex;   // the cursor — where the next sequential draw resumes
};
```

The live per-domain state is also readable directly, though more narrowly:

```cpp
// BlueprintReadOnly — the domains that currently exist and their runtime state.
TMap<FGameplayTag, FDomainRuntimeState> DomainStates;
```

!!! warning "`DomainStates` is a read-only map with opaque values"
    You can read `DomainStates` to see which domains exist, but two limits apply.
    First, it's **read-only** — mutate state through `SetNoiseSeed` / `ResetDomain` /
    `ApplySaveData`, never by writing the map. Second, `FDomainRuntimeState` is a
    `BlueprintType` struct whose `Seed` / `NextIndex` fields are **not** Blueprint-exposed,
    so you can't break out per-domain values from the map in Blueprint. When you need
    a domain's seed or cursor as data, read it from `GetSaveData`'s `FRngDomainState`
    entries instead.

## Library vs subsystem

Both surfaces speak the same domains-and-draws model; the library adds the World
Context convenience and the array helpers, and the subsystem adds the things a library
front door has no reason to wrap — seeding and save/load.

| You want to… | On the Library (BP front door) | On the Subsystem |
|---|---|---|
| Draw sequentially | `RandomFloatInDomain`, `RandomIntInRangeInDomain`, … | `GetRandomFloatInDomain`, `GetRandomIntInRangeInDomain`, … |
| Draw by key (pure) | `KeyedRandomFloatInDomain`, … | `GetKeyedRandomFloatInDomain`, … |
| Shuffle / pick from an array | `ShuffleArrayInDomain`, `RandomArrayItemInDomain`, `RandomArrayItemsInDomain` | — |
| Set the seed / reset a stream | — | `SetNoiseSeed`, `ResetDomain`, `ResetAllDomains` |
| Save or restore state | — | `GetSaveData`, `ApplySaveData` |

Rule of thumb: draw through the library; reach for the subsystem when you need to seed
the run or persist its state.

## Dev-only instrumentation

For plugin authors only: the subsystem exposes a native multicast delegate,
`OnCursorAdvancedDev`, that fires whenever any domain's sequential cursor advances. It's
a C++-only diagnostic hook (not a dynamic/Blueprint delegate) compiled in **only for
non-shipping builds** — it doesn't exist in Shipping and is unreachable from Blueprint
under any configuration. Downstream dev tooling can bind to it (for example the ability
system's resolution sentinel); the plugin itself neither knows nor cares who listens.
It is not a gameplay feature — general consumers can ignore it.
