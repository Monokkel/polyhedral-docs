# Evaluators

For developers and designers who want damage formulas, costs, requirements, and
tuning knobs to live in data — authored in a data table, a data asset, or a
details panel — instead of hard-coded in Blueprint branches or C++. After this
section you'll know the full public surface and how to build, evaluate, and debug
your own magnitudes and conditions.

An evaluator answers one of two questions and hands you the answer as a piece of
data: a **magnitude** produces a number (a damage amount, a cost, a bonus), and a
**condition** produces a true/false (a requirement, a gate, a filter). Because the
formula is data, a designer retunes it without touching code, and the same
authored formula serves every situation by being fed different inputs.

!!! note "Read the concept first"
    This page and its siblings own the API and the mechanics. The mental model —
    what an evaluator is, why formulas are data, and how the two families relate —
    lives on the [Evaluators](../../concepts/evaluators.md) concept page. Start
    there if the ideas below are new; everything here assumes it.

## Where it fits in the framework

Evaluators is a standalone calculation library. It depends on nothing else in the
framework, and you can adopt it entirely on its own — it computes numbers and
booleans and never reaches into other plugins.

Other systems build *on* it. The framework's flagship use is stat modifiers: a
modifier on an entity takes a magnitude as its value source, so "+1 attack per
level" or "×1.5 while empowered" is an authored pipeline rather than hand-written
code. (That entity/stat layer has its own section; for the consumer's view of how
a modifier consumes a magnitude, see the
[Stats & Modifiers](../../concepts/stats-and-modifiers.md) concept page.)

Evaluators commonly read game data through their **context sources** — the typed
[tagged data](../../concepts/tagged-data.md) on the objects a formula is pointed
at. And every evaluator is a **pure read**: it computes a value and never changes
state, which is the read-only half of the write-through-commands split in
[Commands & Undo](../../concepts/commands-and-undo.md).

This plugin is also the home of the framework's exact **stat value** number type
(`FEvalStatValue`) — see [The exact stat value type](#the-exact-stat-value-type)
below.

## The two families

Every evaluator belongs to one of two families that share the same inputs and the
same debugging tools.

- **Magnitudes** produce a number. The workhorse form is a **pipeline**: an
  ordered list of steps, each folding a value into a running total with one
  operation — `+`, `−`, `×`, `÷`, `override`, `min`, or `max`. A step's value
  source can itself be another pipeline, so formulas nest.
- **Conditions** produce a true/false. Each condition carries an **invert** flag
  that flips its result, and conditions compose into **all** (every child passes)
  and **any** (at least one passes) groups — enough to express "alive AND
  (adjacent OR ranged)" as authored data.

The two families interoperate: a magnitude step can branch on a condition
(a *Conditional* calc), and a condition can compare two magnitudes
(a *Compare Magnitudes* calc).

## Two authoring styles

Both families come in two forms that do the same job. You pick per situation, and
bridges exist in both directions.

- **Struct calcs** — lightweight polymorphic structs (`FEvalMagnitudeCalc` for
  magnitudes, `FEvalConditionCalc` for conditions). They are cheap, create no
  object instances, and work directly in data tables and nested details panels.
  This is the common path; reach for it first.
- **Object / Blueprintable calcs** — classes deriving from `UEvalMagnitude` or
  `UEvalCondition`. They support instanced editing in a details panel and let a
  designer script the logic entirely in Blueprint by overriding one event.

!!! tip "Which to reach for"
    Use a struct calc unless you need Blueprint-authored logic; then use an object
    calc. You can mix them freely — an *Object Magnitude* struct calc calls into a
    Blueprint magnitude, and a *Struct Calc* object wraps a struct calc back the
    other way.

You never type the built-in calc names by hand — the editor offers them in a
dropdown when you pick a struct-calc property. Out of the box you get a constant,
a clamp, a rounding step, a curve lookup, a percent bonus, an additive multiplier,
a nested pipeline, a conditional branch, a magnitude comparison, a tag check, and
an all/any group. The full list, with parameters, is in the
[reference](reference.md).

## The pieces at a glance

| Piece | What it is |
|---|---|
| `FEvalMagnitudeCalc` | Base struct for magnitude calcs. Subtypes plug into a `TInstancedStruct` dropdown. |
| `FEvalMagnitudePipeline` | A base value plus ordered steps folded by operations — the workhorse magnitude form. |
| `UEvalMagnitude` | Base Blueprintable magnitude class, for logic authored in Blueprint. |
| `UEvalMagnitudeLibrary` | Static Blueprint/C++ API that evaluates a magnitude, pipeline, or entry. |
| `FEvalConditionCalc` | Base struct for condition calcs, with the shared invert flag. |
| `UEvalCondition` | Base Blueprintable condition class. |
| `UEvalConditionLibrary` | Static API that checks a condition, or a whole all/any set. |
| `UEvalContext` | The shared bag of inputs: sources keyed by tag, context tags, and an optional collector. |
| `UEvalContextAccessor` | A pluggable way for an evaluator to name which source object it reads. |
| `UEvalCollector` | Records a labeled breakdown of a pipeline for tooltips and debugging. |
| `FEvalStatValue` | The exact fixed-point number type used across the stat path. |

## The exact stat value type

`FEvalStatValue` is an exact fixed-point number: an integer that carries a fixed
×1000 scale under the hood, so arithmetic is exact and reproducible — the same
inputs give the same result during play, undo, and replay, with none of the drift
a raw `float` accumulates. You author decimals ("12.5") and read display text
("13") through the conversion helpers; the scale never meets you. The magnitude
core computes in this type when a formula feeds a stat.

You rarely construct one directly — the details panel and the Blueprint make/break
helpers handle it. The [Stats & Modifiers](../../concepts/stats-and-modifiers.md)
concept page owns *why* the framework uses exact math; the conversion API is in the
[reference](reference.md#stat-value).

## Evaluators never change state

!!! warning "Pure reads only"
    An evaluator computes a value; it never mutates game state. That is what lets
    one authored formula run identically during normal play, during a preview, and
    during undo. Anything that *changes* state goes through a command instead — see
    [Commands & Undo](../../concepts/commands-and-undo.md).

## Where to go next

- **[Guides](guides.md)** — task recipes: build a pipeline, write a custom calc,
  compose conditions, fill a context and evaluate, and read a breakdown.
- **[API Reference](reference.md)** — the full public surface, grouped by area,
  with clean signatures and short examples.
- **[Evaluators concept](../../concepts/evaluators.md)** — the mental model this
  section builds on.
- Related foundation plugins: [TaggedData](../taggeddata/index.md) (the entity data
  evaluators read through context) and [CommandSystem](../commandsystem/index.md)
  (the write path evaluators deliberately stay out of).
