# Configuration, Tags & Tooling

For developers configuring the plugin, looking up its gameplay tags, or leaning on
its author-time safety tools. After reading you'll know every native tag the ability
system owns, the three project-settings knobs that bound its what-if and AI work, and
the linter, the Validate Ability action, and the two authoring nodes that catch
mistakes before they reach play.

This is appendix and reference material for the rest of the section — for the model
behind it, see [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md#authoring-and-safety-rails).

## Gameplay tags

Public types carry the `PAb` prefix; the plugin owns a small, fixed set of native
gameplay tags. Five of them are the **tagged-data keys** an ability entity carries as
[tagged data](../taggeddata/reference.md) — the presence of a key, and the value
struct stored under it, is what gives an entity its ability behavior. Any entity can
carry any of them: a sword, a status effect, and a spell all use the same keys.

| Tag | Value struct | What it carries |
|---|---|---|
| `Data.Ability.Program` | `FPAbProgramRef` | Points at the ability's [program of steps](reference-programs.md) — a program class or asset. Its presence is what gives the entity behavior when it activates or a trigger fires. |
| `Data.Ability.Activation` | `FPAbActivationSpec` | Marks the entity as something a player can activate manually, and carries a few activation options. |
| `Data.Ability.UseConditions` | `FPAbUseConditions` | The conditions under which the ability may be used — what the cheap per-frame availability check evaluates, and can name when it fails. |
| `Data.Ability.Triggers` | `FPAbTriggerList` | The [entity-carried triggers](reference-triggers.md) this entity listens on — its reactive entry, zero or more trigger specs. |
| `Data.Ability.Intent` | `FPAbAbilityIntent` | An optional presentation intent that tells the [automatic playout layer](reference-previews.md#the-automatic-playout-layer) how to group this activation's cues. When absent, cues default to the generic `Action` floor below. |

The remaining native tags are not data keys — they are shared markers the runtime
sets or reads:

| Tag | What it's for |
|---|---|
| `Eval.Source.Ability` | An evaluator-context source: lets a step's magnitude or condition read the stats of the *ability* entity itself, not just its caster. |
| `Eval.Source.Instigator` | An evaluator-context source made available when a trigger's condition is evaluated — the entity that instigated the event the trigger is answering. |
| `State.RemovedFromGame` | A marker a step applies the instant a rule removes an entity from play. Targeting and effect filters exclude it immediately, while the structural removal commits at a stable point in resolution. Ordinary death (a `dead` stat or tag) is separate content; this marks the distinct "a rule genuinely takes it off the board" path. |
| `Action` | The root of the presentation-intent namespace and the floor an activation's cues default to when it carries no `Data.Ability.Intent`. |

!!! note "Namespaces the plugin reads by convention"
    Beyond the tags it owns, the built-in steps read and write gameplay tags in
    several other namespaces **by convention**: a stat change names its stat under
    `Stat.*`; a by-magnitude effect can carry an `Element.*` tag (carried for your own
    rules to read, not consumed by the effect itself); a summon names a child slot
    under `Slot.*`; a declaration window broadcasts under `Event.*` (usually
    `Event.Window.*`); a marker step writes a `Marker.*` tag; and cues fan out under
    `Cue.*`. These are conventions, not constraints — the steps do not restrict their
    tag fields to a namespace, so a marker or tag field accepts *any* gameplay tag.
    Follow the conventions for a tidy, discoverable tag tree; the framework will not
    reject a tag that sits outside them.

## Project settings

The plugin exposes three settings under **Project Settings → Plugins → Ability
System**, held in `UPAbSettings` and saved to project config. All three are bounds on
how much work the system's what-if and AI machinery may do — they keep runs
determinate and repeatable, so they are **read-only from Blueprint by design**: a
project sets them once in config, and nothing changes them at runtime (a mid-game
write would desync a run from what a replay or an automated check assumed).

| Setting | Default | What it bounds |
|---|---|---|
| `DefaultSpeculativeStepCeiling` | `4096` | A safety ceiling on how much work a single what-if run may do before it is stopped. Previews, availability checks, and AI trials all run real steps against a temporary, throwaway copy of the game state; this bounds the total work one such run may perform. On overrun the run stops and the preview is reported inconclusive rather than producing a wrong answer or hanging. The default sits far above any realistic ability. |
| `MaxCandidatesPerGate` | `16` | The most target candidates the AI will weigh at one decision point while trying an ability on its turn. If a decision point offers more candidates than this, the surplus is dropped and the clip is logged (never silently). Raise it when your content needs the AI to consider a wider fan of targets. |
| `DefaultAiScoringSamples` | `1` | How many times the AI re-runs a candidate action to average its outcome. `1` is exact for abilities with no random rolls — every run coincides. Raise it for abilities whose outcome varies with rolls, so the AI scores the expected outcome rather than one sampled roll. (The AI always samples its own separate stream — never the roll your activation will actually get.) |

!!! note "The config you author against is stable"
    The data structs a project authors against — these settings, the entry value
    structs above, trigger specs, and step configuration — ship as a versioned
    contract: changes to them are additive-only, so authored content never silently
    breaks on a framework upgrade.

## The ability linter

The **ability linter** is a static validation pass over an ability program. Because a
program is inert data — a configured list of steps, not a running graph — the linter
reads it and flags structural mistakes *before* the program ever runs. It is advisory:
findings never block a save or a compile. Each finding carries a severity (info,
warning, or error), a human-readable message, a stable check id, and a pointer to the
offending step (its label and position in the program) so an editor surface can point
straight at it.

The stable check ids are the reference material:

| Check | What it flags |
|---|---|
| `EffectOnOverride` | A state-changing step set to *replace* the target list when its natural default is to leave it untouched — silently clobbering whatever the targeting steps found. |
| `DanglingLookback` | A step that reads back an earlier step's results, but the reference doesn't name a proven earlier sibling in the same program (missing, out of scope, or referring forward). |
| `CascadeNotFree` | The opening stretch of steps — everything up to the first decision point — could not be certified as making no state changes and opening no response window, so the quick interim preview can't be taken for it. Conservative: declining to certify is not proof of a real problem, only that the linter couldn't prove its absence. |
| `RedundantEvalOverride` | A value authored *both* as an evaluator override and as a non-default raw number, leaving it ambiguous which the step should use. |
| `DuplicateHandle` | Two steps in the same program share the same identifier, so a later step that reads one back can't tell them apart. |
| `CyclicSubProgram` | A nested sub-program references a program that (transitively) contains it — an endless nesting loop. |
| `BothProgramRefSet` | A program reference authored with both a program *class* and a program *asset* set. The class always wins at runtime; this is advisory only. |

!!! note "Two checks await Blueprint step bodies"
    Two further check ids — `BannedNode` (a step body calling something outside the
    one authoring rule) and `BespokeHalt` (a bespoke pause with no matching
    live-vs-preview branch) — scan Blueprint-authored *step bodies*. No Blueprint step
    bodies ship yet, so these two are currently inert. They are a forward provision for
    when that authoring path arrives; today's steps are covered by the structural
    checks above plus loud development-build checks at run time.

## Validate Ability

**Validate Ability** is a per-ability editor action. Right-click an ability program
asset in the Content Browser and choose it; results appear in the **Ability Validation**
Message Log. It runs the linter *plus* an automated consistency check:
it activates the ability against a synthesized solo caster — once for real and once as
a what-if run — and compares the net set of changes each produced.

The comparison reports one of a few results. Two are worth understanding precisely:

| Result | Meaning |
|---|---|
| **Passed** | The real run and the what-if run produced the same net changes — a clean bill of health. |
| **Diverged** | The real run and the what-if run **disagreed**. This is a genuine problem: the ability behaves differently under a preview than it does in play, which almost always means a step reached outside the one authoring rule. Fix it. |
| **Indeterminate** | The check was inconclusive — against a bare solo caster the ability produced no net change, so there was nothing to compare. This is **not** a failure. It means the ability needs a fixture — targets, enemies, a seeded stat — to exercise it. A false pass would be worse than an honest "inconclusive". |

An ability that cannot be activated at all is reported as such rather than as a
divergence — usually a sign it needs a real caller or fixture to run.
Only **Diverged** and linter findings of warning severity or higher count as
problems the report calls out.

## Authoring nodes

Two Blueprint nodes author steps. Both build a step instance from a chosen Ability
Module class, set its configuration from the node's pins, and then differ only in what
they do with it. Under the hood they expand into the `UPAbAbilityStatics` function
library — you rarely wire that library by hand.

!!! note "Finding these nodes in the palette"
    In the Blueprint node palette these two nodes currently appear as **Add ABM** and
    **Run ABM** (and **Add ABM: &lt;ClassName&gt;** / **Run ABM: &lt;ClassName&gt;** once you
    pick a step class) — the same two nodes described here.

- **Add Module** — used *inside* a program's build graph. It emits a step into the
  program's list; it does **not** run the step. Its Builder pin must be connected to
  the build graph's program builder (an unconnected Builder is a hard compile error,
  not a silent no-op). This is the graph-driven alternative to authoring the program's
  step list in class defaults — the two are mutually exclusive.

- **Run Module** — runs a single step *in place*, right where you place the node, as a
  reusable one-off ("spawn decals at these locations") without authoring a whole
  ability. It exposes a **Grouping** option — whether the step's undoable changes fold
  into a surrounding activation's single undo step or stand alone as their own undo
  unit — and an optional **Seed Targets** list to start the step's target list from.

!!! warning "Run Module runs synchronous steps only"
    The run-a-step-in-place node handles steps that finish in one go. Steps that pause
    — decision points and response windows — do not work standalone through it, and
    running an asynchronous step this way is not shipped in this version. Use a full
    ability program when a step needs to wait for a choice or open a window.

## Log categories

For troubleshooting, the plugin logs runtime activity under `LogPAb` and editor-tool
activity (the linter, Validate Ability) under `LogPAbEditor`.
