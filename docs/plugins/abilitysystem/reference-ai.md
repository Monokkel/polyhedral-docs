# Enemy AI: Considerations & Profiles

For developers teaching enemies what a *good* move looks like in their game — and
authoring that judgment as data on a row instead of as a scorer in C++. After
reading you'll know how one **consideration** measures a single axis of a candidate
action, how considerations are granted and bundled into an **AI profile**, the
formula that combines them into one score, and how to tune a misbehaving enemy from
the console.

This page is the authoring surface. The loop it feeds —
deliberate, telegraph, commit, all riding the throwaway-copy machinery — is on
[Previews, AI & What-Ifs](reference-previews.md#enemy-ai); read that first if the
what-if vocabulary here is new. Signatures below are hand-written to show the shape
of each API; they are illustrative, not source excerpts.

## The model in one screen

The AI enumerates **candidate actions** (an ability, and the targets it would take)
and gives each one a number. Everything on this page exists to produce that number
from data:

- A **consideration** is one axis of judgment: a feature to measure, a curve that
  shapes it, whether it *adds to* or *multiplies* the score, and how much it weighs.
  "Damage I would deal to enemies" is one consideration; "how exposed I would be
  afterwards" is another.
- A **consideration grant** is the unit in which considerations are gained and
  lost — either a reference to a shared **consideration set** asset or a single
  inline consideration, plus the entity that granted it.
- An **AI profile** is the bundle of grants an entity carries, stored as tagged data
  under `Data.AI.Profile`. A unit's archetype row carries its temperament; an
  *ability* can carry judgment about its own use; a buff or an item appends grants to
  whoever wears it.
- A **disposition** is an ordinary stat under `Stat.AI.*` that scales every
  consideration declaring it — the knob that makes one goblin reckless and another
  cautious *without* changing either one's considerations.

Two entities' profiles are in play for any one score: the **caller's** and the
**scored ability's**. Their grants are unioned, deduplicated, and sorted into a
canonical order before anything is evaluated.

!!! note "The framework ships no personalities"
    There is no built-in "Aggression", no default consideration set, no sample
    profile. Every axis and every consideration is yours — an "Aggression" the
    framework defined would be content wearing a framework's clothes. What ships is
    the vocabulary, the four scoring calculations below, and the aggregation.

## A consideration

`FPAbConsideration` is a plain struct, and every field is editable in a details
panel — on a row, on a data asset, or on an inline grant. There is no C++ in the
authoring path.

| Field | What it says |
|---|---|
| **Id** | A stable name. Two grants that share the same non-empty `Id` collapse to **one** consideration, so granting the same judgment from an archetype and from a buff does not double-count it. Leave it blank for a genuinely one-off axis — a blank id never deduplicates. |
| **Label** | The designer-facing name printed in the score breakdown. Never used as a key, so renaming it is always safe. |
| **Feature** | The thing being measured: an [Evaluators](../evaluators/reference.md) magnitude calculation, evaluated against the candidate's score context. Usually one of the [four shipped scoring calculations](#the-four-scoring-calculations). Leave it empty and the consideration is neutral — it contributes nothing and vetoes nothing. |
| **Response Curve** | An optional `UCurveFloat` that shapes the raw feature before it is weighed. Unset = the raw value passes through. It is deliberately its own field rather than a step inside the feature, so the breakdown can show *raw* and *curved* side by side. |
| **Bucket** | **Value** or **Modulator** — see below. Defaults to Value. |
| **Weight** | How much this axis matters, for a Value. Defaults to `1.0`, and may be negative — a negative weight is how you price a cost or a risk. Ignored for a Modulator. |
| **Disposition Axis** | An optional `Stat.AI.*` tag whose value *on the caller* scales this consideration. One axis per consideration. Leave it unset for judgment every unit shares. |

### Values add, modulators multiply

The bucket is the whole of a consideration's arithmetic, and a consideration does
not get to choose an operator beyond it:

- A **Value** is weighed and **summed** with every other value. It is the "how much
  is this worth?" bucket, and it may be negative.
- A **Modulator** is clamped to the range 0–1 and **multiplied** in. It is the "how
  much of that should survive?" bucket — a mask. Zero is a veto.

That restriction is deliberate: grants arrive from a row, an ability, a buff and an
item with no ordering between them, so no consideration may declare "I apply after
that one". Because a value only ever adds and a modulator only ever multiplies, the
order they arrive in cannot change the result. That is the property that makes
data-authored AI predictable — see [the score formula](#the-score-formula).

Each consideration evaluates in three moves:

```text
raw     = Feature evaluated against the candidate's score context
curved  = Response Curve applied to raw   (or raw, if there is no curve)

Value:      contribution = curved × Weight × (axis ÷ 100)
Modulator:  contribution = clamp(curved, 0, 1) ^ (axis ÷ 100)
```

!!! tip "Turning any feature into a modulator"
    Features produce raw magnitudes — hit points, path costs — not fractions, so a
    modulator almost always wraps its feature in the shipped **Normalize To Unit**
    calculation, which maps a range onto 0–1 (and can invert it, for "closer is
    better"). A modulator whose value escapes 0–1 is clamped and flagged in the
    breakdown rather than silently distorting the score.

## Granting considerations to an entity

A grant carries **either** a shared set **or** one inline consideration, plus the
source that granted it.

```cpp
// One granted atom. Set OR Inline — if Set is non-null it wins.
struct FPAbConsiderationGrant
{
    UPAbConsiderationSet* Set;         // a shared consideration-set asset
    FPAbConsideration     Inline;      // ...or a single one-off consideration
    FPGeEntityRef         GrantSource; // the buff / item / aura that granted it
};

// The value stored under Data.AI.Profile. A BUNDLE of grants, not one asset ref.
struct FPAbAiProfileFacet
{
    TArray<FPAbConsiderationGrant> Grants;
};
```

`GrantSource` is what makes a temporary personality change revocable: an "Enrage"
buff writes its grants stamped with its own reference and strips exactly those on
expiry, leaving the unit's innate judgment untouched. Leave it invalid for judgment
the entity owns permanently.

### The shared set asset

`UPAbConsiderationSet` is a data asset holding a list of considerations — your
game's reusable judgment vocabulary ("standard melee threat assessment"), granted as
a unit. This is the path most content should take: author the set once, reference it
from every archetype that should think that way.

=== "Editor"
    1. In the Content Browser, create a data asset and pick the **Consideration Set**
       class (`UPAbConsiderationSet`).
    2. Add entries to its **Considerations** list. Each entry collapses to its
       **Label**, so name them for the breakdown you'll be reading later.
    3. Reference the asset from a grant's **Set** field on any entity's profile.

!!! note "Sets do not nest"
    A consideration set holds considerations, never other sets. Nesting would
    reintroduce exactly the ordering question the two-bucket rule exists to remove.
    Compose by granting several sets to the same entity instead — the union is a
    concatenation, and same-id considerations collapse.

### Attaching a profile to a row

`Data.AI.Profile` is ordinary [tagged data](../taggeddata/index.md), so it attaches
the way every other authored facet does: declare an `FPAbAiProfileFacet` property on
your game's entity row struct and mark it with the tagged-data key. The row builder
folds it into the row's tagged data with no extra authoring machinery, and the plugin
registers the key's type into the tagged-data schema at startup, so a mistyped write
is refused rather than stored.

```cpp
// The game's own row struct. The meta tag is the whole wiring.
USTRUCT(BlueprintType)
struct FMyUnitRow : public FPGeGameEntityRow
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, meta=(TaggedData="Data.AI.Profile"))
    FPAbAiProfileFacet AiProfile;
};
```

Because a profile is just tagged data on an entity, template delegation applies: put
the profile on an archetype template and every unit spawned from it inherits the same
judgment for free.

!!! warning "A child row that declares an empty profile *clears* the parent's"
    Row inheritance replaces this key wholesale — it does not merge grant by grant.
    If a child row declares the profile property and leaves it empty, the child ends
    up with **no** grants, not the parent's. Either restate the parent's grants on the
    child or leave the property off the child row entirely.

### Attaching a profile to an ability

The key is symmetric — an *ability* entity carries a profile the same way a unit
does, and its grants join the caller's for the duration of scoring **that** ability.
This is how "the AI understands what this particular spell is for" gets authored:
the spell's own row grants the considerations that price its effect, and only
abilities that carry it pay the cost of evaluating it.

### Granting at runtime

A buff, an item, or an aura adds grants by reading the facet, appending, and writing
it back through the entity system's command-routed tagged-data setter — so the change
is undoable, saves, and replays like any other entity change, and undo restores the
exact bundle that was there before.

=== "Blueprint"
    1. Read the wearer's `Data.AI.Profile` value with the typed tagged-data getter
       node. An entity that has none yields an empty facet — that is a valid start.
    2. Append an element to its **Grants** array with ordinary array nodes: set
       **Set** (or fill **Inline**), and set **Grant Source** to the buff entity so
       expiry can find it again.
    3. Write the facet back with the typed tagged-data setter node. That write is the
       undoable change.
    4. On expiry, read, remove every grant whose **Grant Source** is the buff, and
       write back.

=== "C++"
    ```cpp
    // Read-modify-write: the buff appends its grant, stamped with its own ref.
    FPAbAiProfileFacet Profile = /* read Data.AI.Profile off the wearer */;

    FPAbConsiderationGrant Grant;
    Grant.Set         = BloodlustSet;   // a shared consideration-set asset
    Grant.GrantSource = EnrageBuff;     // so expiry can strip exactly this one
    Profile.AddGrant(Grant);            // C++ helper; Blueprint appends to Grants directly

    // ...write Profile back through the command-routed tagged-data setter.

    // On expiry: strip only what this buff wrote, then write back.
    Profile.RemoveGrantsBySource(EnrageBuff);
    ```

!!! note "C++ only"
    `AddGrant` and `RemoveGrantsBySource` are C++ helpers with no Blueprint nodes.
    Blueprint is not blocked — the **Grants** array is directly readable and writable,
    so the same two operations are ordinary array nodes.

## Disposition axes

A disposition is an ordinary **base stat** under `Stat.AI.*` on the caller. It does
not decide *which* considerations exist; it scales the ones that name it.

| Rule | Detail |
|---|---|
| **Namespace** | `Stat.AI.<Axis>` — for example `Stat.AI.Aggression`. `Stat.AI` itself is the root and is never read as an axis. A stat minted anywhere else is silently ignored: no scaling, no error. |
| **Scale** | Percent, where **100 is neutral**. `0` zeroes a value or disables a modulator; `200` doubles a value. A whole percent is exact, so an axis sitting at exactly 100 gives bit-for-bit the same score as declaring no axis at all. |
| **Authoring** | Write whole display units on the archetype row — `Stat.AI.Aggression = 100`, never `1.0`. |
| **It is a stat** | So the entire stat-modifier machinery is free: a rage buff is `+50 Aggression` with an expiry, a "Pacify" ability is a `−60 Aggression` modifier, and every one of those is command-routed and undoable. |
| **Resolution** | The caller's axes are read **once** per deliberation, from the state before any candidate is tried. |

Only the caller's axes are read — which axes exist is a property of the *unit*, not
of the action being weighed.

!!! warning "A missing axis reads as neutral, not zero"
    If a consideration names `Stat.AI.Caution` and the caller carries no such stat,
    the axis resolves to **100** (neutral), not 0 — a content omission must never
    silently zero a value or disable a mask. That is exactly why the
    [profile linter](#checking-a-profile-the-linter) has a check for it: the failure
    is invisible in play.

    One asymmetry: a **negative** axis is legal for a Value (it inverts what that axis
    is worth) but is **refused with a warning** for a Modulator, because a negative
    exponent would take the mask outside 0–1 and break the score's shape.

## How a candidate is scored

Scoring happens inside the throwaway-copy run for one candidate, after its steps
have run and before the copy is discarded. Every consideration's feature is evaluated
against a **score context** — a read surface describing that one candidate.

The score context is placed in the evaluation context under the tag
**`Eval.Source.Score`**, alongside two ordinary entity sources: `Eval.Source.Owner`
(the caller) and `Eval.Source.Ability` (the ability being weighed). That second half
matters — it means every entity calculation you already have (read a stat, test a
tag) composes into a consideration unchanged.

Three reads are available for anything the score context measures:

| Read | Answers |
|---|---|
| **Baseline** | the world as it stands if this candidate is *not* taken |
| **Resultant** | the world as the candidate's trial run left it |
| **Delta** | Resultant minus Baseline — *"what did this candidate change?"* |

**Delta is the judgment primitive.** "This move deals 12 damage" is a delta; "this
enemy has 34 health" is a baseline read that says nothing about whether acting was
worthwhile.

Beyond entities, the score context also carries a **board view**: the connectivity,
occupancy, and terrain of the grid *as the candidate left it*, so a consideration can
ask "how far would I then be from the player?" or "did this dig open a path?"

!!! warning "Prefer base stats over current stats for judgment"
    A score context reads a stat either as its **base** value or as its resolved
    **current** value. Base is exact on both sides. Current is exact on the resultant
    side but, on the baseline side, reads whatever the live world last materialized —
    a stat nothing has ever asked for reads zero there. Score against **base** unless
    you specifically need a modifier-inclusive number.

### The four scoring calculations

Four magnitude calculations ship for reading a score context. They appear in the
struct dropdown of any **Feature** field under these names:

| Picker name | What it measures |
|---|---|
| **Score Stat Delta** | How much one entity's stat changed because of this candidate. The subject is the **Target** (by index), the **Caller**, or the **Ability**; the read defaults to **Delta** and the stat kind to **Base**. This is the workhorse — "how much damage did I do to this unit?" |
| **Score Faction Aggregate** | The same question asked over a *set* of entities matched by tags, then summed (or counted). This is the calculation that makes *damage to enemies is good, damage to allies is bad* authorable without C++: one consideration with a positive weight over `Faction.Enemy`, one with a negative weight over `Faction.Ally`. Tag matching is hierarchical, so `Faction.Enemy` matches `Faction.Enemy.Goblin`. |
| **Score Target Distance** | Distance from the subject to a pinned target **on the resultant board**, in whole steps — so "did this move close the distance?" prices the board as the candidate would leave it. Either trivial path cost or the subject's own movement profile. Its **No Path Value** is returned verbatim when there is no path, no board, or no resolvable cell: set it high for a distance penalty, or 0 for a proximity bonus. |
| **Normalize To Unit** | Maps another calculation's output onto 0–1 between an authored **In Min** and **In Max**, optionally inverted. This is the transform that turns any of the three above into a **modulator**. It clamps by default; leave that on. |

**Score Faction Aggregate** matches its entity set on the **Baseline** side by
default, and that default is deliberate: a roster that shifts *because of* the
candidate being judged is almost never the set you meant to weigh.

!!! note "C++ only"
    The score read surface itself has no Blueprint nodes — a Blueprint graph cannot
    call "get the delta of this stat". Data-authored considerations reach it through
    the four calculations above; a game that needs a fifth writes a C++ magnitude
    calculation. A Blueprint-authored feature (through the Evaluators object-magnitude
    bridge) still works and can read the caller and the ability as ordinary entity
    sources, but it cannot reach the baseline/resultant/board reads.

    This is by design rather than by omission: every accessor on that surface returns
    an *answer* — a number, a tag container, a cell — and never a pointer to the world,
    a subsystem, or the game state. A consideration therefore cannot reach around the
    trial copy and read the live world, which is what keeps a score honest.

### The score formula

Values sum. Modulators multiply. The two combine like this:

```text
Score = max(ΣValues, 0) × ΠModulators + min(ΣValues, 0)
```

Two properties are worth internalising, because both will bite otherwise:

**A modulator can only ever make a candidate look worse — never better.** The naive
form (`ΣValues × ΠModulators`) inverts the ranking of bad options: a risk mask of
`0.3` applied to a candidate worth `−10` yields `−3`, which now outranks a safe
candidate worth `−5`. The AI would prefer the dangerous disaster, and prefer it *more*
the more dangerous it got. Clamping the multiply to the positive component removes
that class of bug entirely.

**The score does not depend on the order the grants arrived in.** Values sum in an
exact fixed-point kind (no floating-point re-association), and modulators fold in the
profile's canonical order. Grant the same considerations from a row, a buff and an
item in any sequence and the number is identical. That is the property that lets you
reason about a profile by reading it.

!!! tip "Author values first, add masks second"
    A profile of values alone already ranks correctly. Add a modulator when you want a
    condition to *suppress* an otherwise-good option ("don't step into the fire"),
    not when you want to price something — that is a negative-weight value.

## The default scorer, and overriding it

Everything above is the framework's **default** behaviour. Leaving the candidate
scorer unbound now means *score by the entity's own AI profile*:

1. If a game bound a scorer explicitly, it wins — always, for every entity.
2. Otherwise the profile scorer runs, per entity.
3. If that entity's effective profile is **empty** — no grants on the caller and none
   on the ability — it falls back to a content-neutral stand-in that rewards net stat
   *decrease* inflicted on units other than the caller.

Step 3 is what makes profiles opt-in: an enemy carrying no `Data.AI.Profile` behaves
exactly as it did before profiles existed.

```cpp
// The content-neutral fallback is public, so a game scorer can blend it rather
// than reimplement it: "the framework's reading of magnitude, plus MY faction
// knowledge". C++ only.
static float ContentNeutralScore(const TArray<FPGeEntityChange>& Coalesced,
                                 const FPGeEntityRef& Caller);

// Binding a scorer overrides profiles entirely, for every entity.
void SetCandidateScorer(FPAbCandidateScorer Scorer);
```

The full scorer contract, and when reaching for it is still the right call, is on
[Previews, AI & What-Ifs](reference-previews.md#the-scorer-contract).

## Tuning: the console loop

A profile that misbehaves is almost always one consideration out of scale, and the
score breakdown tells you which. **The tuning loop is a console loop** — there is no
editor window for it.

| Console | What it does |
|---|---|
| `PAb.AI.ScoreBreakdown` | `0` (default) captures nothing and costs nothing. `1` captures a per-consideration breakdown for every candidate the AI scores. `2` also logs each candidate to `LogPAb` as it is scored. Compiled out of Shipping builds. |
| `PAb.AI.DumpLastBreakdown` | Prints the last deliberation's per-candidate breakdown. Requires the capture to have been on **before** that deliberation ran. |

The workflow is: set `PAb.AI.ScoreBreakdown 1`, let the enemy take its turn, then run
`PAb.AI.DumpLastBreakdown`. Each candidate prints its final number first, then the
arithmetic that produced it:

```text
[AI score] 12.5000  ability <ref>  candidate 0 sample 0  targets 1  (profile)
    consideration                bucket        raw     curved      contrib  axis
    Hurt the enemy               value         ...        ...          ...  Stat.AI.Aggression @ 150%
    Risk of retaliation          mod           ...        ...          ...    [CLAMPED to [0,1]]
    => SumValues ...  x  ProductModulators ...  =  total ...
```

Read it in this order:

1. **The bucket column.** A row you meant as a mask sitting in the value bucket is the
   single most common authoring slip.
2. **`raw` versus `curved`.** A curve doing nothing means it is unset or its input is
   outside the range you drew it for.
3. **`[CLAMPED]`.** A modulator whose value fell outside 0–1 was clamped — its feature
   needs a **Normalize To Unit** wrapper, or a different range on the one it has.
4. **The axis column.** `@ 100%` means the axis is neutral: either that is intended,
   or the caller does not carry the stat.

Two lines you may see instead of a breakdown, and what each means:

- *"(no AI profile on this entity — nothing to tune here; grant `Data.AI.Profile`)"* —
  this enemy was scored by the content-neutral fallback.
- *"(breakdown not captured — set `PAb.AI.ScoreBreakdown 1` before deliberating)"* —
  the capture was off when the decision was made. The setting is sampled once per
  deliberation, so turning it on mid-turn does not retroactively fill it in.

The same records are readable in C++, which is what a test asserts against:

```cpp
// The captured records for the LAST deliberation — appended in deliberation order
// and deliberately kept after the decision is made. C++ only (no Blueprint nodes).
const TArray<FPAbCandidateScoreRecord>& GetLastDeliberationScores() const;
FString FormatLastDeliberationScores() const;
static FString FormatScoreRecord(const FPAbCandidateScoreRecord& Record);
```

```cpp
// One scored (candidate, sample). The struct is BlueprintReadOnly, so a debug widget
// can display one it is handed — but only C++ can fetch them.
struct FPAbCandidateScoreRecord
{
    FPGeEntityRef      Caller;
    FPGeEntityRef      Ability;
    int32              CandidateIndex;  // which candidate of this ability
    int32              SampleIndex;     // which repeat, when sampling more than once
    TArray<FPAbTarget> Targets;
    float              Score;
    bool               bFromProfile;    // false = scored by the content-neutral fallback
    FPAbScoreBreakdown Breakdown;       // filled only on the profile path
};
```

## Checking a profile: the linter

A **profile linter** checks a set of considerations for the three mistakes that are
invisible in play. It is advisory — it never blocks a save — and it reports the same
finding shape the [ability linter](reference-tooling.md#the-ability-linter) uses.

| Check | Severity | What it catches |
|---|---|---|
| `ZeroWeightValue` | Warning | A **Value** with a weight of 0. It contributes exactly 0 for every candidate, so the row is authored, evaluated, listed in the breakdown, and completely inert. Give it a weight, delete it, or make it a Modulator if the intent was to gate rather than to price. |
| `UnnormalizedValueFeature` | Info | A **Value** whose feature has nothing bounding its range — no clamp, no curve, no **Normalize To Unit**, and no response curve on the consideration itself. It is a *shape* observation, not an accusation: the linter cannot know what range a stat delta spans at run time, so it declines to certify rather than claiming a fault. |
| `DispositionAxisNotGranted` | Warning | A disposition axis that will always read neutral. Two cases: the tag is not under `Stat.AI.*` at all (so resolution never sees it), or it is a legitimate axis the **caller carries no stat for**. Grant the stat on the archetype row that grants the judgment, so every entity inheriting the profile also inherits the knob it is steered by. |

The linter expands what the scorer will actually see — grants resolved, set assets
flattened, duplicate ids collapsed — so a set granted twice is reported once.

!!! warning "The profile linter has no editor UI today"
    There is **no right-click action on a consideration set**, no asset validator, and
    no Message Log wiring, and the **Validate Ability** action on an ability program
    does *not* invoke it. It is a headless C++ entry point, so today it runs from an
    automated test or a tools commandlet you write:

    ```cpp
    // Four static entry points, all returning the same report shape.
    FPAbLintReport R1 = FPAbAiProfileLinter::LintConsiderationSet(MySet);
    FPAbLintReport R2 = FPAbAiProfileLinter::LintProfileFacet(Facet, TEXT("Goblin row"));

    // The form that also checks the caller actually carries its disposition stats —
    // this is the only one that can catch the second half of DispositionAxisNotGranted.
    FPAbLintReport R3 = FPAbAiProfileLinter::LintEntityProfile(GameState, Caller, Ability);
    ```

    Wiring it into an automation test that walks your consideration-set assets is the
    practical way to keep it running today.

## Limits worth knowing

- **The board view is resultant-only.** A consideration can ask about the board as the
  candidate left it, not about the board before it. Anything genuinely
  baseline-dependent about terrain has to be computed outside deliberation — the live
  board *is* the baseline there, and it is restored exactly between candidates.
- **Main grid only.** Board queries resolve against the world's main grid. A world with
  no grid is supported and silent: board queries answer empty rather than failing.
- **No line of sight.** There is no visibility query anywhere in the framework, so no
  consideration can ask for one.
- **How many candidates get weighed** is bounded by the `MaxCandidatesPerGate` project
  setting, and **how many times each is re-run** by `DefaultAiScoringSamples` — see
  [Configuration, Tags & Tooling](reference-tooling.md#project-settings). Raise the
  sample count for abilities whose outcome varies with a roll, so the AI scores the
  expected outcome rather than one sample.
- **Considerations are resolved before any candidate runs.** Both the caller's
  disposition axes and each ability's effective profile are assembled from the state
  *before* deliberation. An ability whose effect would grant judgment that flatters
  that same ability cannot bootstrap its own victory, and every candidate is scored by
  the same function.

---

**See also:** [Previews, AI & What-Ifs](reference-previews.md#enemy-ai) (the
deliberate–telegraph–commit loop and the C++ scorer) ·
[Evaluators reference](../evaluators/reference.md) (the magnitude calculations a
feature is built from) ·
[Configuration, Tags & Tooling](reference-tooling.md) (the tags, settings, and
linter reference) ·
[Guides](guides.md#give-an-enemy-a-personality) (the followable recipe).
