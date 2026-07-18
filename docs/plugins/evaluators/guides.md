# Evaluators Guides

For developers and designers who have the plugin enabled and want to get things
done. Each section is a short, followable recipe; pick the one that matches your
task. For the full type-by-type surface see the [API Reference](reference.md), and
for the model behind it all see [Evaluators](../../concepts/evaluators.md).

## Build a magnitude pipeline from steps

A pipeline starts from a base value and folds each step into a running total with
one operation. This is the everyday way to author a formula.

The example below reads: start at 0, add a base of 50, add the attacker's
strength, multiply by a critical factor, then take the max with 1 so damage never
drops below a point.

=== "Blueprint"
    1. Add an `FEvalMagnitudePipeline` property to your data-table row, data asset,
       or Blueprint (it is a `BlueprintType` struct).
    2. Set **Base Value** to `0`.
    3. Add a **Step**. Set **Op** to `+` and pick **Constant** in the step's
       **Magnitude** dropdown; enter `50`. Give it a **Label** ("Base Damage") so
       it reads well in a breakdown.
    4. Add a step with **Op** `+` whose magnitude reads the attacker's strength
       (a custom calc, or a **Curve** driven by an input).
    5. Add a step with **Op** `×` and a **Constant** of `2` ("Critical").
    6. Add a step with **Op** `Max` and a **Constant** of `1` ("Minimum").
    7. Evaluate it with **Evaluate Pipeline** (see
       [Fill an evaluation context and evaluate](#fill-an-evaluation-context-and-evaluate)).

=== "C++"
    ```cpp
    FEvalMagnitudePipeline Pipeline;
    Pipeline.BaseValue = 0.f;

    auto AddStep = [&](EMagnitudeOp Op, float Constant, const TCHAR* Label)
    {
        FEvalMagnitudeStep Step;
        Step.Op = Op;
        Step.Label = Label;
        Step.Magnitude.InitializeAs<FEvalMagCalc_Constant>();
        Step.Magnitude.GetMutable<FEvalMagCalc_Constant>().Value =
            FEvalStatValue::FromDecimal(Constant);
        Pipeline.Steps.Add(Step);
    };

    AddStep(EMagnitudeOp::Add,      50.f, TEXT("Base Damage"));
    AddStep(EMagnitudeOp::Multiply,  2.f, TEXT("Critical"));
    AddStep(EMagnitudeOp::Max,       1.f, TEXT("Minimum"));

    const float Damage = Pipeline.Evaluate(Context);   // Context filled below
    ```

!!! tip "Nest a pipeline inside a step"
    A step's value source is any struct calc, and one of the built-in calcs is a
    **Pipeline**. Drop a *Pipeline* calc into a step to nest a sub-formula; set its
    **Use Input As Base** to feed the parent's running total in as the sub-pipeline's
    starting value.

## Branch a formula on a condition

Use a **Conditional** magnitude calc when a step should compute one way if a check
passes and another way if it fails — for example, double damage against a marked
target, otherwise flat damage.

=== "Blueprint"
    1. In a step's **Magnitude** dropdown, pick **Conditional**.
    2. Set its **Condition** to any condition calc (for example **Context Tag**
       requiring a "Marked" tag).
    3. Set **If True** and **If False** to their own magnitude calcs (each can be a
       constant, a curve, or another nested pipeline).

=== "C++"
    ```cpp
    FEvalMagCalc_Conditional Branch;
    Branch.Condition.InitializeAs<FEvalCondCalc_ContextTag>();
    Branch.Condition.GetMutable<FEvalCondCalc_ContextTag>()
        .RequiredTags.AddTag(MarkedTag);

    Branch.IfTrue.InitializeAs<FEvalMagCalc_PercentBonus>();
    Branch.IfTrue.GetMutable<FEvalMagCalc_PercentBonus>().Percent = 100; // ×2
    // IfFalse left unset → passes the input through unchanged.
    ```

## Write a custom magnitude calc (C++ struct)

When a built-in calc can't express your formula, derive a struct from
`FEvalMagnitudeCalc` and override `Evaluate`. Your calc then appears in every
magnitude dropdown automatically.

```cpp
// A magnitude that returns the owner's current health, read from a source object.
USTRUCT(BlueprintType, DisplayName="Owner Health")
struct FMyMagCalc_OwnerHealth : public FEvalMagnitudeCalc
{
    GENERATED_BODY()

    virtual float Evaluate(float Input, UEvalContext* Context) const override
    {
        if (const AMyPawn* Owner = Context->GetSourceAs<AMyPawn>(EvalTags::TAG_Source_Owner))
        {
            return Owner->GetHealth();
        }
        return Input;   // a well-behaved calc passes the input through on a miss
    }

    virtual FString GetDebugLabel() const override { return TEXT("Owner Health"); }
};
```

!!! warning "Keep calcs stateless and read-only"
    A calc must compute purely from its properties and the context, and must not
    write to any member during `Evaluate` or touch game state. The framework
    reuses calc instances across evaluations, and the pure-reads rule is what keeps
    undo and replay correct. If your value must feed a stat exactly, also override
    `EvaluateScaled` (see [the reference](reference.md#magnitudes)); otherwise the
    default quantizes your float result at the boundary for you.

## Write a custom condition (Blueprint class)

For a check you'd rather author visually, make a Blueprint class and override one
event. This is the object/Blueprintable style.

=== "Blueprint"
    1. Create a Blueprint class deriving from **Condition** (`UEvalCondition`).
    2. Override the **Evaluate** event.
    3. Read what you need from the context — **Get Source** to fetch the owner or
       target, **Has Tag** to test a context tag — and return true or false.
    4. Author the plain, positive form of the check. The **Invert** flag on the
       instance flips the result for you.

=== "C++"
    ```cpp
    // The same check as a struct condition: passes when the target is adjacent.
    USTRUCT(BlueprintType, DisplayName="Target Adjacent")
    struct FMyCondCalc_TargetAdjacent : public FEvalConditionCalc
    {
        GENERATED_BODY()

        virtual bool Evaluate(UEvalContext* Context) const override
        {
            const AActor* Owner  = Context->GetSourceAs<AActor>(EvalTags::TAG_Source_Owner);
            const AActor* Target = Context->GetSourceAs<AActor>(EvalTags::TAG_Source_Target);
            return Owner && Target && AreAdjacent(Owner, Target);
        }

        virtual FString GetDebugLabel() const override { return TEXT("Target Adjacent"); }
    };
    ```

!!! note "Invert is applied for you"
    Both base types expose `bInvert`. You call `Check()` (or `CheckWithOwner()`),
    which runs your `Evaluate` and then applies the flag — so always author the
    un-inverted logic and leave the flip to the instance.

## Compose conditions into all/any groups

Requirements usually combine several checks. Group them instead of nesting
branches.

=== "Blueprint"
    - **Struct path:** pick a **Group** condition calc, add child conditions to it,
      and set **Combine** to `AND` (all must pass) or `OR` (any must pass). `XOR`
      and `NAND` are available too.
    - **Object path:** use a **Group** condition object the same way, or pass an
      array of conditions to **Check All Conditions** (AND) / **Check Any
      Condition** (OR).

=== "C++"
    ```cpp
    // (alive) AND (adjacent OR ranged)
    FEvalCondCalc_Group Inner;                       // adjacent OR ranged
    Inner.Combine = EEvalCombineOp::Or;
    Inner.Conditions.Add(MakeAdjacentCond());
    Inner.Conditions.Add(MakeRangedCond());

    FEvalCondCalc_Group Root;
    Root.Combine = EEvalCombineOp::And;
    Root.Conditions.Add(MakeAliveCond());
    Root.Conditions.Emplace(TInstancedStruct<FEvalConditionCalc>::Make(Inner));

    const bool bCanAct = Root.Check(Context);
    ```

!!! note "Empty groups and short-circuiting"
    An `AND` group with no children passes (vacuous truth); `OR`, `XOR`, and `NAND`
    with no children fail. Groups short-circuit on the first determining child, so
    keep child checks free of side effects — which the pure-reads rule already
    requires.

## Fill an evaluation context and evaluate

Every evaluator reads a `UEvalContext`: the **sources** (objects keyed by tag), the
**context tags**, and an optional collector. The caller fills it; the evaluator
only reads it. That is what lets one authored formula serve every attacker and
target.

You have two options. Build a context once and reuse it across several
evaluations, or use the **inline-context** library functions that build a
throwaway context from raw sources and tags for a single call.

=== "Blueprint"
    **Reuse a context:**

    1. **Construct Object** of class **Eval Context**.
    2. **Set Source** for each role — `Eval.Source.Owner` = the attacker,
       `Eval.Source.Target` = the defender.
    3. Optionally **Add Tag** for context flags this evaluation should carry.
    4. Call **Evaluate Pipeline** (or **Check Condition Entry**) with your data and
       the context.

    **One-shot:** call **Evaluate Pipeline (Inline Context)** and hand it a
    `Sources` map and `Tags` container directly — it builds the context for you.

=== "C++"
    ```cpp
    // Build once, reuse.
    UEvalContext* Ctx = UEvalContext::MakeContext(
        this,
        { { EvalTags::TAG_Source_Owner,  Attacker },
          { EvalTags::TAG_Source_Target, Defender } });

    const float Damage = DamagePipeline.Evaluate(Ctx);
    const bool  bOk    = RequirementGroup.Check(Ctx);
    ```

!!! tip "Standard source tags"
    The plugin ships the common role tags: `Eval.Source.Owner` (the instigator),
    `Eval.Source.Target`, and `Eval.Source.Context` (the context object itself).
    Author your formulas against these so any caller can bind them.

## Read a breakdown for a tooltip or debug

Turn on the context's **collector** and each pipeline step records a labeled
contribution — its label, the value it produced, and the running total after it.
The same trace serves an authoring debug log and a player-facing tooltip.

=== "Blueprint"
    1. Call **Enable Collector** on the context before evaluating.
    2. Evaluate the pipeline as usual.
    3. Read the collector's **Steps** to drive your own UI, or call **Format
       Simple** for a ready-made human-readable string.

=== "C++"
    ```cpp
    Ctx->EnableCollector();
    const float Damage = DamagePipeline.Evaluate(Ctx);

    // Ready-made string:
    UE_LOG(LogTemp, Display, TEXT("%s"), *Ctx->Collector->FormatSimple());

    // Or walk the steps for a custom tooltip:
    for (const FEvalCollectorStep& Step : Ctx->Collector->Steps)
    {
        // Step.Label, Step.Value, Step.Running
    }
    ```

## Author an evaluator in a data table or details panel

Struct calcs are the reason evaluators sit comfortably in authored data — they
need no object instances, so they serialize into a data-table cell or a nested
property and edit inline in the details panel.

- Put an `FEvalMagnitudePipeline` (or a single `FEvalMagnitudeEntry`) column on a
  data-table row, and designers build the formula per row with the dropdown-driven
  step editor.
- Nest calcs to any depth — a step's magnitude can be a **Pipeline**, a
  **Conditional**, an **Additive Multiplier** whose bonuses are themselves calcs,
  and so on. The details panel expands each inline; a step's label doubles as its
  collapsed title.
- To share one complex formula across many places, store it once as a
  **Magnitude Recipe** data asset and reference it (a *From Recipe* magnitude), so
  a retune in one asset updates every user.

!!! tip "Object calcs edit inline too"
    `UEvalMagnitude` / `UEvalCondition` subclasses are `EditInlineNew`, so an
    object pipeline or a Blueprint calc also edits directly in the panel where it's
    referenced — no separate asset needed unless you want to share it.

## Work with exact stat values

When a formula feeds an entity stat, it runs on the exact fixed-point path so the
result is reproducible across undo and replay. You mostly meet `FEvalStatValue` at
the edges — authoring a **Constant** calc, or converting a result for display.

=== "Blueprint"
    - Author decimals normally; the details customization stores them exactly.
    - To turn a stat value into UI text use **Stat Value → Display Text** (rounds to
      a whole number, "13") or **Stat Value → String** (the exact form, "12.5").
    - To build one in a graph use **Make Stat Value (From Decimal)** or
      **(From Whole)**.

=== "C++"
    ```cpp
    FEvalStatValue V = FEvalStatValue::FromDecimal(12.5);

    const double  Display = V.ToDisplay();       // 12.5, exact
    const int64   Whole   = V.ToWholeRounded();  // 13, half away from zero
    const FString Text    = V.ToDisplayText();   // "13" for a HUD
    ```

!!! note "Convert only at the display edge"
    Keep values in `FEvalStatValue` while you compute, and convert to `float`,
    `double`, or text only when you present them. Converting mid-formula reintroduces
    the drift the exact type exists to avoid. The
    [Stats & Modifiers](../../concepts/stats-and-modifiers.md) concept page explains
    why.
