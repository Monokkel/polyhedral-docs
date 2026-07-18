# Evaluators: Numbers and Conditions as Data

For developers and designers who want damage formulas, requirements, and tuning
knobs to live in data tables and assets instead of scattered Blueprint branches.
After reading, you'll know the two evaluator families, how pipelines compose a
formula from steps, and how to write your own magnitude or condition.

## What an evaluator is

An evaluator answers one of two questions and hands you the answer as a piece of
data you can place in a data table, a data asset, or a details panel:

- **How much?** A *magnitude* produces a number — a damage amount, a cost, a bonus.
- **Is it true?** A *condition* produces a yes/no — a requirement, a gate, a filter.

Because the formula is data, a designer can retune it without touching code, and
you can reuse the same authored formula in many situations by feeding it different
inputs.

## Two families: structs and objects

Every evaluator comes in two forms, and both do the same job.

- **Struct calcs** are lightweight polymorphic structs (`FEvalMagnitudeCalc` for
  magnitudes, `FEvalConditionCalc` for conditions). They are cheap, work directly
  in data tables, and create no object instances. This is the common case.
- **Object calcs** are Blueprintable classes (`UEvalCondition` and its magnitude
  twin). They support instanced editing in a details panel and let a designer
  script logic entirely in Blueprint.

!!! tip "Which one to reach for"
    Reach for a struct calc first — it is the data-table-friendly path and covers
    most formulas. Use an object calc when the logic must be authored in Blueprint.
    Bridges exist in both directions, so a struct pipeline can call into a Blueprint
    magnitude and vice versa.

You never enumerate the built-in calc types by hand — the editor offers them in a
dropdown. Out of the box you get pieces like a constant, a curve lookup, a clamp,
a rounding step, and a read of another entity's stat.

## Pipelines: a formula built from steps

A magnitude *pipeline* (`FEvalMagnitudePipeline`) is an ordered list of steps.
Each step produces a value and folds it into a running total with one operation:

`+`  `−`  `×`  `÷`  `override`  `min`  `max`

The pipeline starts from a base value, then applies each step in order. A damage
formula might read: start at 0, add a base of 50, add the attacker's strength,
multiply by a critical factor, then take the max with 1 so it never drops below a
point of damage. Complex formulas come from composition, not from code — and a
step's value source can itself be another pipeline, so formulas nest.

## Conditions mirror magnitudes

A condition is checked the same way a magnitude is evaluated, against the same
inputs. Each condition carries an **invert** flag that flips its result, and
conditions compose into **all** (every child must pass) or **any** (at least one
must pass) groups. That is enough to express requirements like "the target is
alive AND (adjacent OR ranged)" as authored data.

## The evaluation context

Every evaluator reads from a shared bag of inputs called the **evaluation
context** (`UEvalContext`). It holds three things:

- **Sources** — objects keyed by a gameplay tag, so a formula can name "the owner"
  or "the target" abstractly and the caller supplies the concrete objects.
- **Context tags** — free-form tags describing this particular evaluation.
- An optional **collector** for debugging (below).

The caller fills the context; the evaluator only reads it. That separation is what
lets one authored formula serve every attacker and every target — the formula
refers to roles, and each call binds those roles to real objects.

!!! warning "Evaluators are pure reads"
    An evaluator computes a value; it never changes game state. This is deliberate:
    the same formula runs during normal play, during undo, and during a preview,
    and it must give the same answer every time. Anything that *changes* state goes
    through a command instead — see [commands-and-undo.md](commands-and-undo.md).

## Breakdowns for debugging and tooltips

Enable the context's **collector** (`UEvalCollector`) and each step records a
labeled contribution — its label, the value it produced, and the running total
after it. While authoring, that trace tells you exactly where a number came from.
At runtime, the same trace is the raw material for a player-facing tooltip.

## Write your own

=== "Blueprint"

    1. Create a Blueprint class deriving from **Condition** (`UEvalCondition`).
    2. Override the **Evaluate** event.
    3. Read what you need from the context (Get Source, Has Tag) and return
       true or false.

    The **Invert** flag on the node flips your result for you, so author the
    plain, positive form of the check.

=== "C++"

    ```cpp
    // A struct condition: passes when the evaluation carries the "Empowered" tag.
    USTRUCT()
    struct FCond_Empowered : public FEvalConditionCalc
    {
        GENERATED_BODY()

        virtual bool Evaluate(UEvalContext* Context) const override
        {
            return Context->HasTag(TAG_State_Empowered);   // your own check here
        }

        virtual FString GetDebugLabel() const override { return TEXT("Empowered"); }
    };
    ```

## Putting it together

Fill a context, evaluate a pipeline, and read the breakdown back.

=== "Blueprint"

    1. Make an **Eval Context** and set its sources (for example
       `Eval.Source.Owner` = the attacker, `Eval.Source.Target` = the defender).
    2. Call **Enable Collector** on the context.
    3. Call **Evaluate Pipeline**, passing your data-table pipeline and the context.
    4. Call **Format Simple** on the context's collector to show the labeled steps.

=== "C++"

    ```cpp
    UEvalContext* Ctx = UEvalContext::MakeContext(
        this, { { OwnerTag, Attacker }, { TargetTag, Defender } });
    Ctx->EnableCollector();

    const float Damage = DamagePipeline.Evaluate(Ctx);   // your authored pipeline

    UE_LOG(LogEval, Display, TEXT("%s"), *Ctx->Collector->FormatSimple());
    ```

## Where this shows up

The framework's flagship use of evaluators is stat modifiers: a modifier takes a
magnitude as its value source, so "+1 attack per level" is an authored pipeline
rather than hand-written code. See [stats-and-modifiers.md](stats-and-modifiers.md).

Evaluators commonly read entity data through their context sources — the typed
data described in [tagged-data.md](tagged-data.md). And the read-only rule above
is the other side of the write path in
[commands-and-undo.md](commands-and-undo.md).

<!-- pluginlink: evaluators-reference -->
For the full list of built-in magnitude and condition calcs, their parameters,
and the debug tooling, see the Evaluators plugin reference.
