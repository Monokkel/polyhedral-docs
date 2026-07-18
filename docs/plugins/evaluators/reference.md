# Evaluators API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Evaluators](../../concepts/evaluators.md).

All public types carry the `Eval` prefix. Signatures below are hand-written to show
the shape of each API; they are illustrative, not source excerpts. Names in
**bold** are the editor display names you see in dropdowns.

## Magnitudes

A magnitude produces a number. The struct family is the common path; the object
family is for Blueprint-authored logic.

### Struct calc base

`FEvalMagnitudeCalc` is the base for struct-based magnitudes. Concrete subtypes
plug into any `TInstancedStruct<FEvalMagnitudeCalc>` property, giving the editor a
type dropdown. Derive from it and override `Evaluate` to make your own.

```cpp
// Produce a value. Input is contextual — in a pipeline it is the running total.
virtual float Evaluate(float Input, UEvalContext* Context) const;

// The exact fixed-point twin. The default runs Evaluate and quantizes the result;
// override it only when your calc has an exact integer form to preserve.
virtual FEvalStatValue EvaluateScaled(FEvalStatValue Input, UEvalContext* Context) const;

// Human-readable label for debug and collector output.
virtual FString GetDebugLabel() const;
```

### Built-in magnitude calcs

These ship ready to use; pick them from the dropdown on any magnitude step or calc
property.

| Display name | Type | What it does |
|---|---|---|
| **Constant** | `FEvalMagCalc_Constant` | Returns a fixed value (`FEvalStatValue Value`), ignoring input. |
| **Clamp** | `FEvalMagCalc_Clamp` | Clamps the input between an optional `Floor` and `Ceiling` (each gated by a `bHas…` flag). |
| **Round** | `FEvalMagCalc_Round` | Rounds the input by `Method`: `Floor`, `Ceiling`, `Round`, or `Truncate`. |
| **Curve** | `FEvalMagCalc_Curve` | Evaluates a `UCurveFloat` at the input value. |
| **Percent Bonus** | `FEvalMagCalc_PercentBonus` | A whole-percent multiplier: `Percent` 50 → ×1.5, −30 → ×0.7, 0 → ×1. Use as a `×` step. |
| **Additive Multiplier** | `FEvalMagCalc_AdditiveMultiplier` | Sums a set of `Bonuses` (each a calc) and returns `1 + sum`. Use as a `×` step. |
| **Pipeline** | `FEvalMagCalc_Pipeline` | A nested pipeline (`BaseValue` + `Steps`); set `bUseInputAsBase` to start from the parent's running total. |
| **Conditional** | `FEvalMagCalc_Conditional` | Evaluates `IfTrue` or `IfFalse` based on a `Condition`. See [Cross-type calcs](#cross-type-calcs). |

```cpp
// Rounding methods for Round.
enum class EEvalRoundMethod : uint8 { Floor, Ceiling, Round, Truncate };
```

!!! note "Constant stores an exact value"
    `FEvalMagCalc_Constant::Value` is an [`FEvalStatValue`](#stat-value), so an
    authored `12.5` is held exactly. You still type the decimal — the scale is
    hidden by the details customization.

### Object magnitude base

`UEvalMagnitude` is the abstract, Blueprintable base for magnitudes authored as
objects. Subclass it (in C++ or Blueprint) and override `Evaluate`.

```cpp
// Designer note; not shown to players.
FString DevComment;

// Produce a value. In a pipeline, InputValue is the running total.
UFUNCTION(BlueprintNativeEvent)
float Evaluate(float InputValue, UEvalContext* Context);

// Label for debug output.
UFUNCTION(BlueprintNativeEvent, BlueprintPure)
FString GetDebugLabel() const;
```

Object magnitudes are `EditInlineNew`, so they edit directly in the details panel
where referenced. Built-in object families:

| Display name | Type | What it does |
|---|---|---|
| **Pipeline** | `UEvalMagnitude_Pipeline` | A pipeline of instanced object magnitude steps (`FEvalMagnitudeObjectStep`). |
| **Struct Pipeline** | `UEvalMagnitude_StructPipeline` | Wraps a struct `FEvalMagnitudePipeline` inside an object tree. |
| **Struct Calc** | `UEvalMagnitude_FromCalc` | Wraps a `TInstancedStruct<FEvalMagnitudeCalc>` inside an object tree. |
| **Entry** | `UEvalMagnitude_Entry` | Wraps an `FEvalMagnitudeEntry` (constant-or-calc choice). |
| **From Recipe** | `UEvalMagnitude_FromRecipe` | Delegates to a shared [Magnitude Recipe](#recipes) asset. |
| **Object Magnitude** | `FEvalMagCalc_Object` | *(struct calc)* Spawns a `UEvalMagnitude` from a `TSubclassOf` and delegates — the bridge from a struct pipeline into a Blueprint magnitude. |

### Entry points

Two small structs express "a value that may or may not need a calc", handy in
authored data.

```cpp
// A constant, optionally replaced by an instanced calc.
struct FEvalMagnitudeEntry
{
    float Constant = 0.f;                    // Input is added to this
    TObjectPtr<UEvalMagnitude> Calculation;  // if null, GetValue returns Constant + Input

    float GetValue(UEvalContext* Context, float Input = 0.f) const;
    bool  HasCalculation() const;
};

// Always an instanced calc (no constant fallback).
struct FEvalMagnitudeRef
{
    TObjectPtr<UEvalMagnitude> Magnitude;

    float GetValue(float InputValue, UEvalContext* Context) const;
    bool  IsValid() const;
};
```

## Pipelines

A pipeline is a base value plus ordered steps, each folding a value into the
running total with one operation.

```cpp
// How a step combines with the running total.
enum class EMagnitudeOp : uint8
{
    Add,       // running += value        (+)
    Subtract,  // running -= value        (−)
    Multiply,  // running *= value        (×)
    Divide,    // running /= value        (÷, skipped if value ≈ 0)
    Override,  // running  = value        (=, discards everything before)
    Min,       // running  = min(running, value)
    Max,       // running  = max(running, value)
};

// One step: an operation + a struct-calc value source + a label.
struct FEvalMagnitudeStep
{
    EMagnitudeOp Op = EMagnitudeOp::Add;
    TInstancedStruct<FEvalMagnitudeCalc> Magnitude;  // pick a calc type from the dropdown
    FString Label;                                   // shown in breakdowns and as the collapsed title
};

// A complete pipeline as a struct — usable in data tables and details panels.
struct FEvalMagnitudePipeline
{
    float BaseValue = 0.f;
    TArray<FEvalMagnitudeStep> Steps;

    // Evaluate and return the final value. Input is added to BaseValue.
    float Evaluate(UEvalContext* Context, float Input = 0.f) const;

    // The exact fixed-point path — computes in scaled integer space, keeping full
    // precision across steps. Used when a formula feeds a stat.
    FEvalStatValue EvaluateScaled(UEvalContext* Context, FEvalStatValue Input = {}) const;
};
```

!!! tip "Float path vs exact path"
    Most callers use `Evaluate` (and its Blueprint wrapper, **Evaluate Pipeline**).
    Whole numbers and whole percents round-trip exactly through it. When the result
    must feed a stat with no drift, callers on the stat path use `EvaluateScaled`
    instead — see [Stat value](#stat-value).

### Object pipeline steps

The object-family pipeline (`UEvalMagnitude_Pipeline`) uses instanced steps instead
of struct calcs:

```cpp
struct FEvalMagnitudeObjectStep
{
    EMagnitudeOp Op = EMagnitudeOp::Add;
    TObjectPtr<UEvalMagnitude> Magnitude;  // an instanced object magnitude
    FString Label;
};
```

## Recipes

`UEvalMagnitudeRecipe` is a data asset that stores one magnitude for reuse. Place
any `UEvalMagnitude` (including a pipeline) inside and reference it from many
places, so a single retune updates every user.

```cpp
class UEvalMagnitudeRecipe : public UPrimaryDataAsset
{
    TObjectPtr<UEvalMagnitude> RootCalculation;   // any magnitude, including a Pipeline
    bool bDuplicateBeforeEvaluate = false;        // set if the graph holds mutable scratch

    UFUNCTION(BlueprintCallable)
    float Evaluate(float InputValue, UEvalContext* Context) const;
};
```

Reference a recipe from inside another magnitude graph with the **From Recipe**
magnitude (`UEvalMagnitude_FromRecipe`).

## Magnitude library

`UEvalMagnitudeLibrary` is the static Blueprint/C++ API for evaluating magnitudes.
Each core function has a plain form that takes a `UEvalContext*` you already built,
and an **inline-context** form that builds a throwaway context from raw sources and
tags for a single call.

```cpp
// ── With an existing context ──────────────────────────────────────────
// Evaluate an entry; Input is added to its Constant.
static float GetMagnitudeEntryValue(const FEvalMagnitudeEntry& Entry,
    UEvalContext* Context, float Input = 0.f);

// Evaluate a pipeline; Input is added to BaseValue.
static float EvaluatePipeline(const FEvalMagnitudePipeline& Pipeline,
    UEvalContext* Context, float Input = 0.f);

// Spawn a magnitude from a class, evaluate it, return the result.
static float EvaluateMagnitudeFromClass(UObject* WorldContextObject,
    float InputValue, TSubclassOf<UEvalMagnitude> MagnitudeClass, UEvalContext* Context);

// ── Inline context (built from Sources + Tags for you) ────────────────
static float GetMagnitudeEntryValueWithContext(UObject* WorldContextObject,
    const FEvalMagnitudeEntry& Entry,
    const TMap<FGameplayTag, UObject*>& Sources,
    const FGameplayTagContainer& Tags, float Input = 0.f);

static float EvaluatePipelineWithContext(UObject* WorldContextObject,
    const FEvalMagnitudePipeline& Pipeline,
    const TMap<FGameplayTag, UObject*>& Sources,
    const FGameplayTagContainer& Tags, float Input = 0.f);

static float EvaluateMagnitudeWithContext(UObject* WorldContextObject,
    UEvalMagnitude* Magnitude, float InputValue,
    const TMap<FGameplayTag, UObject*>& Sources,
    const FGameplayTagContainer& Tags);
```

## Conditions

A condition produces a true/false. The struct and object families mirror the
magnitude ones, and both share an `bInvert` flag applied by their public entry
point.

### Struct calc base

`FEvalConditionCalc` is the base for struct-based conditions. Override `Evaluate`;
callers use `Check`, which applies `bInvert`.

```cpp
bool bInvert = false;               // flips the result after evaluation

// Public entry point — runs Evaluate, then applies bInvert.
bool Check(UEvalContext* Context) const;

// Exact fixed-point entry point — for conditions that compare magnitudes on the
// stat path; applies bInvert over EvaluateScaled.
bool CheckScaled(UEvalContext* Context) const;

// Override to implement the raw logic (author the un-inverted form).
virtual bool Evaluate(UEvalContext* Context) const;
virtual bool EvaluateScaled(UEvalContext* Context) const;  // default forwards to Evaluate
virtual FString GetDebugLabel() const;
```

### Built-in condition calcs

| Display name | Type | What it does |
|---|---|---|
| **Constant** | `FEvalCondCalc_Constant` | Always returns a fixed `bValue`. |
| **Context Tag** | `FEvalCondCalc_ContextTag` | Tests the context's own tags: passes with any of `PossibleTags`, all of `RequiredTags`, none of `ProhibitedTags` (each skipped if empty). |
| **Compare Magnitudes** | `FEvalCondCalc_Compare` | Compares two magnitude calcs, `Left (Op) Right`, with a `Tolerance` for equality. See [Cross-type calcs](#cross-type-calcs). |
| **Group** | `FEvalCondCalc_Group` | Combines child condition calcs with `Combine` (`AND`/`OR`/`XOR`/`NAND`). |
| **Object Condition** | `FEvalCondCalc_Object` | Spawns a `UEvalCondition` from a `TSubclassOf` and delegates — the bridge into a Blueprint condition. |

```cpp
// Comparison operators for Compare Magnitudes.
enum class EEvalCalcCompareOp : uint8 { Greater, GreaterEqual, Equal, NotEqual, LessEqual, Less };

// Combine operators for groups.
enum class EEvalCombineOp : uint8 { And, Or, Xor, Nand };
```

### Object condition base

`UEvalCondition` is the abstract, Blueprintable base. Override `Evaluate`; callers
use `Check` or `CheckWithOwner`.

```cpp
bool bInvert = false;
FString DevComment;

// Public entry points — run Evaluate, then apply bInvert.
UFUNCTION(BlueprintCallable) bool Check(UEvalContext* Context);
UFUNCTION(BlueprintCallable) bool CheckWithOwner(UObject* Owner);  // Owner bound to Source.Owner

// Override to implement the raw logic.
UFUNCTION(BlueprintNativeEvent) bool Evaluate(UEvalContext* Context);

UFUNCTION(BlueprintNativeEvent, BlueprintPure) FString GetDebugLabel() const;
```

Built-in object condition families:

| Display name | Type | What it does |
|---|---|---|
| **Group** | `UEvalCondition_Group` | Combines instanced child conditions with `Combine`. |
| **Struct Calc** | `UEvalCondition_FromCalc` | Wraps a `TInstancedStruct<FEvalConditionCalc>` inside an object tree. |
| **Entry** | `UEvalCondition_Entry` | Wraps an `FEvalConditionEntry` (default-bool-or-calc choice). |
| **Check Tags** | `UEvalCondition_Tag` | Tests gameplay tags on an object resolved via a [context accessor](#evaluation-context) — the object must implement `IGameplayTagAssetInterface`. |

!!! note "Group semantics"
    `AND` with no children passes (vacuous truth); `OR`, `XOR`, and `NAND` with no
    children fail. Evaluation short-circuits on the first determining child. This
    matches between the struct `FEvalCondCalc_Group` and object `UEvalCondition_Group`.

!!! note "Context tags vs an object's tags"
    Checking the *context's* tags is the struct calc **Context Tag**
    (`FEvalCondCalc_ContextTag`). Checking tags on a *source object* is the object
    condition **Check Tags** (`UEvalCondition_Tag`), which needs an instanced
    accessor a struct can't hold.

### Entry points

```cpp
// A default bool, optionally replaced by an instanced condition.
struct FEvalConditionEntry
{
    bool bDefault = true;                   // used when Condition is null
    TObjectPtr<UEvalCondition> Condition;   // if set, evaluated instead of bDefault

    bool Check(UEvalContext* Context) const;
    bool HasCondition() const;
};

// Always an instanced condition (no bool fallback).
struct FEvalConditionRef
{
    TObjectPtr<UEvalCondition> Condition;
    bool Check(UEvalContext* Context) const;
    bool IsValid() const;
};
```

## Condition library

`UEvalConditionLibrary` mirrors the magnitude library: a plain form taking an
existing context, and an inline-context form.

```cpp
// ── With an existing context ──────────────────────────────────────────
static bool CheckConditionEntry(const FEvalConditionEntry& Entry, UEvalContext* Context);

// AND over an array (empty → true).
static bool CheckAllConditions(const TArray<UEvalCondition*>& Conditions, UEvalContext* Context);

// OR over an array (empty → false).
static bool CheckAnyCondition(const TArray<UEvalCondition*>& Conditions, UEvalContext* Context);

// ── Inline context ────────────────────────────────────────────────────
static bool CheckConditionEntryWithContext(UObject* WorldContextObject,
    const FEvalConditionEntry& Entry,
    const TMap<FGameplayTag, UObject*>& Sources, const FGameplayTagContainer& Tags);

static bool CheckConditionWithContext(UObject* WorldContextObject,
    UEvalCondition* Condition,
    const TMap<FGameplayTag, UObject*>& Sources, const FGameplayTagContainer& Tags);

static bool CheckAllConditionsWithContext(UObject* WorldContextObject,
    const TArray<UEvalCondition*>& Conditions,
    const TMap<FGameplayTag, UObject*>& Sources, const FGameplayTagContainer& Tags);
```

## Cross-type calcs

Two calcs bridge the families, so magnitudes and conditions can reference each
other.

- **Conditional** (`FEvalMagCalc_Conditional`) — a magnitude that evaluates `IfTrue`
  or `IfFalse` based on a `Condition`. Each part is itself a `TInstancedStruct`
  (the condition is a `FEvalConditionCalc`, the branches are `FEvalMagnitudeCalc`),
  so branches nest arbitrarily. An unset branch passes the input through unchanged.
- **Compare Magnitudes** (`FEvalCondCalc_Compare`) — a condition that evaluates two
  magnitude calcs and compares them, `Left (Op) Right`, with a `Tolerance` used for
  equality checks.

## Evaluation context

`UEvalContext` is the shared bag of inputs that flows through every evaluation.

```cpp
// ── Sources: objects keyed by gameplay tag ────────────────────────────
TMap<FGameplayTag, TObjectPtr<UObject>> Sources;

UFUNCTION(BlueprintCallable, BlueprintPure) UObject* GetSource(FGameplayTag Tag) const;
UFUNCTION(BlueprintCallable)                void     SetSource(FGameplayTag Tag, UObject* Object);
template<typename T> T* GetSourceAs(FGameplayTag Tag) const;   // C++ typed helper

// ── Context tags ──────────────────────────────────────────────────────
FGameplayTagContainer Tags;
UFUNCTION(BlueprintCallable, BlueprintPure) bool HasTag(FGameplayTag Tag) const;
UFUNCTION(BlueprintCallable)                void AddTag(FGameplayTag Tag);
UFUNCTION(BlueprintCallable)                void RemoveTag(FGameplayTag Tag);

// ── Collector (see below) ─────────────────────────────────────────────
TObjectPtr<UEvalCollector> Collector;
UFUNCTION(BlueprintCallable)                UEvalCollector* EnableCollector();
UFUNCTION(BlueprintCallable, BlueprintPure) bool IsCollecting() const;

// ── Recursion safety ──────────────────────────────────────────────────
int32 MaxEvalDepth = 64;   // nested object evaluators bail past this, logging an error

// ── C++ factory ───────────────────────────────────────────────────────
static UEvalContext* MakeContext(UObject* Outer,
    const TMap<FGameplayTag, UObject*>& InSources,
    const FGameplayTagContainer& InTags = {});
```

In Blueprint, build a context with **Construct Object** of class **Eval Context**
and fill `Sources`/`Tags`, or let an inline-context library function build one for a
single call.

!!! note "MaxEvalDepth"
    Object-family evaluators can recurse into children, and a designer could wire a
    cycle. `MaxEvalDepth` caps recursion so a bad graph bails with a logged error
    instead of overflowing the stack. The default is generous; you rarely change it.

### Context accessors

An accessor lets an evaluator name *which* source object it reads, so the choice is
authored data rather than hard-coded.

```cpp
// Abstract, Blueprintable base. Override ResolveObject.
class UEvalContextAccessor : public UObject
{
    UFUNCTION(BlueprintCallable, BlueprintPure)
    UObject* GetObject(UEvalContext* Context, bool& bSuccess);   // applies your resolution

    UFUNCTION(BlueprintNativeEvent)
    UObject* ResolveObject(UEvalContext* Context, bool& bSuccess);

    template<typename T> T* GetObjectAs(UEvalContext* Context);  // C++ typed helper
};

// The common accessor: look a source up by tag.
class UEvalSourceAccessor : public UEvalContextAccessor
{
    FGameplayTag SourceTag;   // e.g. Eval.Source.Owner
};
```

## Collector / breakdown

Enable the collector on a context and each step records a labeled contribution. The
trace drives debug logs and player-facing tooltips alike.

```cpp
// One recorded step.
struct FEvalCollectorStep
{
    EEvalStepKind Kind = EEvalStepKind::Step;   // Base, Step, or Final
    FString Label;
    uint8   Op = 0;                             // the step's EMagnitudeOp
    float   Value = 0.f;                        // the value this step produced
    float   Running = 0.f;                      // the running total after it
    TObjectPtr<UObject> Source;                 // an optional attributed source
};

class UEvalCollector : public UObject
{
    TArray<FEvalCollectorStep> Steps;

    UFUNCTION(BlueprintCallable)                void    Reset();
    UFUNCTION(BlueprintCallable, BlueprintPure) FString FormatSimple() const;  // ready-made string
};

// Step kinds.
enum class EEvalStepKind : uint8 { Base, Step, Final };
```

Read `Steps` to build your own UI, or call `FormatSimple` for a quick
human-readable dump.

## Stat value

`FEvalStatValue` is an exact fixed-point number: an `int64` carrying a fixed ×1000
scale, so arithmetic is exact and reproducible across undo and replay. Author
decimals and read display text through the conversions — the raw scaled integer
stays hidden. Text serialization is a decimal string ("12.5"), so data tables,
CSV, and diffs never show the scaled form.

```cpp
// ── Construct ─────────────────────────────────────────────────────────
static FEvalStatValue FromWhole(int64 Whole);     // 12 → 12.0
static FEvalStatValue FromDecimal(double Value);  // 12.5 → 12.5 (rounds at 3 decimals)
static FEvalStatValue FromScaled(int64 Scaled);   // advanced: wrap an already-scaled value

// ── Read out ──────────────────────────────────────────────────────────
double  ToDisplay() const;        // 12.5, exact for ≤3 decimals
float   ToFloat() const;          // float-boundary / presentation form
int64   ToWholeRounded() const;   // 13  (half away from zero: 2.5 → 3, −2.5 → −3)
int64   FloorToWhole() const;     // toward −∞  (explicit authored rounding)
int64   CeilToWhole() const;      // toward +∞
int64   TruncToWhole() const;     // toward zero
FString ToString() const;         // exact canonical string: "12.5", "-0.25"
FString ToDisplayText() const;    // rounded whole-unit text for a HUD: "13"

bool IsZero() const;

// ── Arithmetic (the rounding rule is built in) ────────────────────────
FEvalStatValue operator+(const FEvalStatValue&) const;
FEvalStatValue operator-(const FEvalStatValue&) const;
FEvalStatValue Mul(const FEvalStatValue&) const;   // widen-multiply, round back
FEvalStatValue Div(const FEvalStatValue&) const;   // ÷0 yields 0; op-level guards decide skip
// plus == != < <= > >=
```

!!! tip "Author and display, don't hand-scale"
    Construct with `FromDecimal`/`FromWhole`, read with `ToDisplay`/`ToDisplayText`,
    and let arithmetic stay in this type. A raw whole number entered without the
    scale is a bug the type can flag — `IsSuspiciouslyUnscaled()` is a lint helper
    for validators, not a hard rule.

### Stat value library

`UEvalStatValueLibrary` is the Blueprint make/break surface — Blueprint never
touches the raw scaled integer.

```cpp
static FEvalStatValue MakeStatValueFromDecimal(double Value);   // Make Stat Value (From Decimal)
static FEvalStatValue MakeStatValueFromWhole(int64 Whole);      // Make Stat Value (From Whole)

static double  StatValueToDisplay(const FEvalStatValue& Value);        // → Display
static int64   StatValueToWholeRounded(const FEvalStatValue& Value);   // → Whole (Rounded)
static FString StatValueToString(const FEvalStatValue& Value);         // → String (exact)
static FString StatValueToDisplayText(const FEvalStatValue& Value);    // → Display Text (rounded)

static bool StatValueIsSuspiciouslyUnscaled(const FEvalStatValue& Value);   // lint warning
```

## Source tags

The plugin declares the standard role and debug tags used to key context sources.
Author formulas against these so any caller can bind the roles.

| Tag | Role |
|---|---|
| `Eval.Source.Owner` | The primary owner / instigator (e.g. the attacking actor). |
| `Eval.Source.Target` | A secondary target (e.g. the defender). |
| `Eval.Source.Context` | The evaluation context object itself, for calcs that inspect it. |
| `Eval.Debug` | Debug flag — enables verbose logging during evaluation. |
| `Eval.PrintToScreen` | Debug flag — prints evaluation info on screen. |

In C++ these are `EvalTags::TAG_Source_Owner`, `TAG_Source_Target`,
`TAG_Source_Context`, `TAG_Eval_Debug`, and `TAG_Eval_PrintToScreen`.

## Authoring in the editor

The details-panel experience is polished: struct calcs edit inline through a
type dropdown, object calcs (`EditInlineNew`) edit in place where referenced, a
step's label doubles as its collapsed title, and `FEvalStatValue` fields present as
plain decimals. No API calls are needed to author an evaluator — drop an
`FEvalMagnitudePipeline` or a magnitude/condition property onto a data-table row,
data asset, or Blueprint and build the formula visually. See
[Author an evaluator in a data table or details panel](guides.md#author-an-evaluator-in-a-data-table-or-details-panel).
