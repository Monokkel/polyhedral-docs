# Facing & Arcs

For developers who need to know which way a unit is pointing and who is standing
behind it. After this page you can rotate a unit undoably, ask whether an attacker is
in its front, flank, or rear arc, tune how wide those arcs are per project or per
unit, and drop a ready-made backstab check into any rule that takes a condition.

Signatures below are hand-written to show the shape of each call; they are
illustrative, not source excerpts.

## The model: facing is placement data

A unit's *logical* facing is a whole number of turns on its
[grid placement](reference.md#the-placement-struct-and-tags) — `RotationSteps`. One
step is one of the board's directions: 90&deg; on a square board, 60&deg; on a hex
board. Because it lives in the same placement facet as the unit's cell, **rotating is
a command**: it undoes, saves, and replays exactly like moving.

Three consequences worth internalising before the API:

- **Rules read steps, never degrees.** Every arc query below derives from
  `RotationSteps` through the board's own direction math, so two machines classify
  "behind" identically.
- **The cosmetic `Facing` rotator is free.** It is a presentation yaw composed on top
  of the logical step. No rule reads it, and it never changes which cells a unit
  covers — so a mini can lean, drift, or ease into a turn without touching game state.
- **A freeform board has no directions.** A [topology](../gridgraph/reference-graph.md#topologies)
  with no fixed step directions is *facing-inert*: every direction and arc query
  returns false rather than inventing an answer.

For a multi-cell unit, rotating also changes **which cells it covers** — see
[Footprints](reference-footprints.md).

## Reading and changing facing

`UPGxFacingLibrary` is a Blueprint function library holding the read side and the
one write convenience. The write calls that own the placement facet itself live on
`UPGxPlacementLibrary`.

| Call | Kind | What it answers or does |
|---|---|---|
| **Get Entity Facing Step** | Blueprint pure | The unit's facing as a step in `[0, step count)`. False when it is not on a cell, or the board cannot be resolved. |
| **Get Entity Facing Direction** | Blueprint pure | The coordinate direction it faces. False when unplaced, unresolvable, or facing-inert. |
| **Get Step Towards Cell** | Blueprint pure | The step that would point this unit at a cell — the pre-check before rotating. False on a degenerate bearing (the target sits on the unit's own pivot, or directly above/below it). |
| **Face Entity Towards Cell** | Blueprint callable | Rotate in place to face a cell. One command; skipped entirely when already facing that way. |
| **Rotate Entity In Place** | Blueprint callable | Set the logical step directly. One command; cell, board, and cosmetic facing are preserved. False if the unit carries no cell placement. |
| **Set Entity Facing** | Blueprint callable | Set the **cosmetic** rotator only. Never read by rules, never affects occupancy. |

```cpp
// Which way is it pointing?
int32 Step = 0;
if (UPGxFacingLibrary::GetEntityFacingStep(this, Unit, Step)) { /* ... */ }

// The direction on the board it faces (invalid on a freeform board).
FIntVector Dir;
UPGxFacingLibrary::GetEntityFacingDirection(this, Unit, Dir);

// "What step would point me at that cell?" — ask before you turn.
int32 Wanted = 0;
if (UPGxFacingLibrary::GetStepTowardsCell(this, Unit, TargetCell, Wanted))
{
    UPGxPlacementLibrary::RotateEntityInPlace(this, Unit, Wanted);
}

// Or do both in one call — the auto-face atom.
UPGxFacingLibrary::FaceEntityTowardsCell(this, Unit, TargetCell);
```

!!! note "*When* a unit faces is your game's rule"
    The framework supplies the turn; it never decides that attacking, moving, or being
    attacked should cause one. Call **Face Entity Towards Cell** at whatever moments
    your design wants a unit to turn — as a step in an ability, on a move commit, or
    from the player's own rotate control.

!!! warning "A multi-cell unit's rotation can be illegal"
    Rotating changes the set of cells a footprinted unit covers, so a legal facing at
    one anchor may hang off the board or straddle a wall. Call
    [**Can Entity Stand At**](reference-footprints.md#can-this-unit-stand-here) with the
    unit's current anchor and the *new* step first. The rotate calls do **not** check
    it for you.

## Arcs: front, flank, and rear

An **arc** is the answer to "where is that observer relative to the way I am facing?"
The framework ships three arc tags, and they are **classification outputs only** —
never stored on an entity, never something you set:

| Tag | Meaning |
|---|---|
| `Arc.Front` | The observer is within the defender's frontal arc. |
| `Arc.Flank` | Off to one side. |
| `Arc.Rear` | Behind — the backstab arc. |

Three queries classify, all Blueprint pure:

```cpp
// Classify one CELL against a defender's facing — for UI hints and cell-based rules.
FGameplayTag Arc;
UPGxFacingLibrary::GetRelativeArc(this, Defender, ObserverCell, Arc);

// Classify an ENTITY against a defender's facing.
UPGxFacingLibrary::GetRelativeArcBetween(this, Defender, Observer, Arc);

// The one-call test: "is this attacker behind me?"
const bool bBackstab =
    UPGxFacingLibrary::IsObserverInArc(this, Defender, Observer, PGxFacingTags::TAG_Arc_Rear);
```

Two behaviours matter when you build rules on these:

- **Both sides are footprint-aware.** An entity observer is represented by whichever
  of its covered cells is geometrically nearest the defender's pivot — a dragon
  attacking with its head is judged by its head, not by a distant anchor cell. Ties
  resolve to the earliest offset in authored order, so the answer is stable across
  undo and replay.
- **Matching is hierarchical.** A game that authors a finer arc such as `Arc.Rear.Deep`
  still satisfies a query for `Arc.Rear`, so a shipped rule keeps working when you
  subdivide the space.

An entity observer is classified **purely geometrically**: its cells do not have to
resolve to live cells on the board, so a unit standing on a cell that was just
removed can still be classified.

## Arc bands: how the space is divided

How wide is "front"? Games disagree — a hex tactics game may want a one-direction
front arc, a shield-wall game a 180&deg; one — so the mapping is **data, not code**.
It is an `FPGxArcBands` value:

```cpp
// The angular separation -> arc tag mapping, sampled at HALF-step resolution.
struct FPGxArcBands
{
    // Index 0 = dead ahead, last entry = dead behind.
    TArray<FGameplayTag> BandsByHalfStep;
};
```

The array is sampled every **half** step, so the between-step bearings (a square
board's diagonals, a hex board's vertex directions) get their own entry instead of
landing on a tie:

| Board | Entries are read at |
|---|---|
| Square (4 steps) | 0&deg;, 45&deg;, 90&deg;, 135&deg;, 180&deg; |
| Hex (6 steps) | 0&deg;, 30&deg;, 60&deg;, 90&deg;, 120&deg;, 150&deg;, 180&deg; |

Angular separation is **unsigned**, so arcs are always left/right symmetric — you
author one side and both sides follow. A shorter array simply extends its last entry
out to 180&deg;, so `[Arc.Front, Arc.Rear]` is a legal, very blunt split.

### Which bands a unit uses

The value used for a defender resolves in this order, first non-empty wins:

| Order | Source | Use it for |
|---|---|---|
| 1 | The unit's own `Data.Placement.ArcBands` tagged data | One unit whose arcs differ — a shield-wall soldier with a wide front. Authored on the template, overridable live. |
| 2 | The project setting for that board's step count | Your game's answer, set once. |
| 3 | The built-in split | The sensible default if you set nothing. |

The built-in split is: **front** under 60&deg;, **flank** under 120&deg;, **rear**
from 120&deg;. On a square board that samples to
`[Front, Front, Flank, Rear, Rear]`; on a hex board to
`[Front, Front, Flank, Flank, Rear, Rear, Rear]`.

### The project setting

Arc defaults live in **Project Settings &rarr; Plugins &rarr; Polyhedral Grid
Entity**, under **Facing**:

- **Default Arc Bands By Step Count** — a map keyed by the board's step count
  (**4** = square, **6** = hex), each entry an arc-band value. A missing or empty
  entry falls through to the built-in split.

Keying by step count rather than by board means one project setting covers a game
that uses a square overworld and a hex battle map.

=== "Blueprint"
    1. Open **Project Settings &rarr; Plugins &rarr; Polyhedral Grid Entity**.
    2. Under **Default Arc Bands By Step Count**, add an entry with key `4` (or `6`
       for hex).
    3. Fill **Bands By Half Step** with one tag per half-step — e.g. for a
       narrow-front square game: `Arc.Front`, `Arc.Flank`, `Arc.Flank`, `Arc.Rear`,
       `Arc.Rear`.
    4. To give one unit different arcs, add a **Tagged Data** entry on its template
       keyed `Data.Placement.ArcBands` holding its own arc-band value.

=== "C++"
    ```cpp
    // Give one unit a wide frontal arc, overriding the project default.
    // Square board -> five half-step entries: 0/45/90/135/180 degrees.
    FPGxArcBands ShieldWall;
    ShieldWall.BandsByHalfStep = {
        PGxFacingTags::TAG_Arc_Front.GetTag(),   //   0 deg
        PGxFacingTags::TAG_Arc_Front.GetTag(),   //  45 deg
        PGxFacingTags::TAG_Arc_Front.GetTag(),   //  90 deg
        PGxFacingTags::TAG_Arc_Flank.GetTag(),   // 135 deg
        PGxFacingTags::TAG_Arc_Rear.GetTag()     // 180 deg
    };

    // Stored like any other entity tagged data — command-routed, so it undoes.
    State->SetTaggedData(Unit, PGxFacingTags::TAG_Data_Placement_ArcBands,
        FInstancedStruct::Make(ShieldWall));
    ```

!!! tip "Author your own arcs freely"
    `BandsByHalfStep` takes any tag under `Arc`, not just the three shipped ones. A
    game that wants `Arc.Front.Narrow` for a critical-hit rule authors it, points the
    relevant half-step entries at it, and existing `Arc.Front` rules keep matching it
    because arc matching is hierarchical.

## The ready-made backstab check

`FPGxCondCalc_ObserverInArc` — shown in the editor as **Observer In Facing Arc** — is
a [condition calc](../evaluators/reference.md#conditions) that passes when one entity
sits in a required arc of another. It is the drop-in backstab, flanking, and
shield-arc primitive, and it fits everywhere a condition fits: a conditional branch
inside a damage magnitude, an ability's use condition, a granted modifier's active
condition.

```cpp
struct FPGxCondCalc_ObserverInArc   // DisplayName "Observer In Facing Arc"
{
    FGameplayTag DefenderSource;  // whose facing defines the arc  (default: the target)
    FGameplayTag ObserverSource;  // who is being classified       (default: the owner)
    FGameplayTag RequiredArc;     // the arc that must match       (default: Arc.Rear)
};
```

=== "Blueprint"
    1. In the damage magnitude, add a **Conditional** magnitude calc.
    2. Set its condition to **Observer In Facing Arc**. The defaults are already the
       backstab case: the **Defender Source** is the target, the **Observer Source** is
       the owner, and the **Required Arc** is `Arc.Rear`.
    3. Point the true branch at your bonus damage. Attacks from behind now hit harder,
       and nothing else in the ability needed to know about facing.

=== "C++"
    ```cpp
    // "+50% damage when striking from behind."
    FPGxCondCalc_ObserverInArc FromBehind;   // defaults are already the backstab case
    FromBehind.RequiredArc = PGxFacingTags::TAG_Arc_Rear;

    // Drop it into a conditional magnitude branch, an ability use condition,
    // or a granted modifier's active condition — anywhere a condition goes.
    ```

!!! tip "It re-evaluates when either unit turns"
    The condition reads both entities through the framework's tracked entity accessors,
    so its reads are recorded as dependencies. A granted modifier gated on it
    re-evaluates by itself when either unit's placement, footprint, or arc bands change
    — you never invalidate a "flanked" buff by hand. This is the usual
    [derived-state](../../concepts/derived-state.md) behaviour, applied to geometry.

Because the condition classifies through the *same* core the Blueprint queries use,
a rule and the UI hint that previews it can never disagree about what "behind" means.

## Tags

| Tag | Kind | Meaning |
|---|---|---|
| `Arc.Front` / `Arc.Flank` / `Arc.Rear` | Classification output | The three shipped arcs. Never stored on an entity — only ever returned by a query. |
| `Data.Placement.ArcBands` | Tagged-data key (`FPGxArcBands`) | One unit's arc-band override. Authored on a template, overridable live. |

!!! note "C++ only"
    The shared resolution and classification cores — reading a unit's effective arc
    bands, the built-in split, and the geometry that turns a bearing into an arc — are
    plain C++ statics with no Blueprint nodes. Blueprint reaches all of the same
    behaviour through the pure queries and the **Observer In Facing Arc** condition
    above; the cores exist so those two paths can never drift apart.

## See also

- [Footprints](reference-footprints.md) — why rotating a multi-cell unit needs a
  legality check, and the check to call.
- [Placement](reference.md#placement) — the placement facet facing lives in, and the
  calls that write it.
- [Evaluators reference](../evaluators/reference.md#conditions) — where condition
  calcs plug in.
