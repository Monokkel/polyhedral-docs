# Ability System

For developers ready to turn entities, stats, events, turns, and the grid into
playable actions — spells, attacks, item uses, always-on passives, the counters
and interrupts that answer them, target previews on hover, and enemy AI that
plans its move before it makes it. After this section you'll know how an ability
is authored as data, how the framework resolves it one step at a time, and which
page owns each piece of the machinery.

This is the framework's capstone plugin: it composes every lower plugin into one
thing a designer authors as data and a player activates. The *why* — the model
behind programs, decision points, triggers, and what-if runs — lives on the
[Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md)
concept page. This section owns the mechanics: the exact nodes and classes, the
full step-and-program reference, trigger recipes, the targeting API, the preview
and AI hooks, and the tags and tooling.

!!! note "Read the concept first"
    If you haven't yet, start with
    [Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md)
    in Core Concepts. It is the final capstone of the concept spine and assumes
    the vocabulary of Events, Turns, and Tokens & Cues. Everything below builds
    on the model it establishes — *step*, *ability program*, *decision point*,
    *entity-carried trigger*, and *what-if run*.

## An ability is an entity

An ability is not a special class. It is an ordinary
[game entity](../gameentity/index.md) — template-backed, created from a
data-table row, and granted to a unit as a child in an ability slot, the same
hierarchy the entity system already gives a sword or a held card. Its cooldown
and damage are ordinary stats; granting and revoking it are ordinary undoable
changes that save and replay with the unit.

What *makes* an entity an ability is the data it carries under `Data.Ability.*`:
a reference to its program of steps, the conditions under which it may be used,
and any triggers it listens on. Those entries are not exclusive kinds — an
activatable ability, a trigger that fires in response, and an always-on passive
are just the entries an entity happens to carry, and *any* entity can carry
them. A poison status effect carries a trigger the same way a spell does.

Reach for the ability system when you want a unit's **actions** — the sequenced,
undoable, previewable things it does — rather than a single stat change (that is
[GameEntity](../gameentity/reference-stats.md)), a one-off event broadcast
([EventSystem](../eventsystem/index.md)), a grid query
([GridGraph](../gridgraph/index.md)), or a screen cue
([TokenSystem](../tokensystem/index.md)). An ability composes all of those into
one coherent action.

## The mental model in one screen

- An ability's behavior is an **ability program** — an ordered list of **steps**,
  authored as data. Each step does one small thing: find targets, ask for a
  choice, change game state, open a response window, show a marker. No event
  graph runs the ability; the program *is* its logic.
- The steps hand each other a running **target list**. Each step reads the list
  produced so far, contributes its own results, and declares how the two combine
  — replace it, merge into it, or leave it untouched (the default for a
  state-changing step, so it acts on exactly what the targeting steps found).
- The framework's **rules engine** — the resolution subsystem — runs the program
  one step at a time. It pauses at a **decision point** when a step needs a
  choice (pick a target, confirm a zone), publishing a request that a player's
  UI, a hovered target, or the AI can answer.
- It also pauses for **entity-carried triggers**: *interrupts* that reshape or
  veto a change before it commits, and *reactions* that answer with changes of
  their own after. Because a trigger is entity data, who is listening undoes,
  saves, and replays with the world.
- The very same steps can run against a **temporary, throwaway copy of the game
  state**, discarded afterward. That one mechanism powers target previews on
  hover, cheap availability checks, and enemy AI that tries each candidate move
  on a copy and scores the outcome.
- Whatever a real activation commits feeds **the automatic playout layer**,
  which turns the committed changes into the on-screen cues the
  [presentation layer](../tokensystem/index.md) documents.

Two invariants hold the whole thing together: **one activation is one undo
step** (everything it commits, including every reaction it provokes, folds into a
single transaction), and **one activation resolves at a time** (a new intent
issued mid-targeting simply replaces the old one; issued after anything has
committed, it is refused — committed state never silently unwinds).

## The 80% path

Most abilities never author a program by hand. The framework ships **ready-made
programs** — deal damage in an area, apply a status, buff self, summon, move to a
cell — that an ability references straight from its data-table row. A fireball
and an area-poison blast are the *same* area-damage program pointed at different
damage stats and element tags; authoring one is a data row plus a reference, not
a graph.

Custom programs remain fully supported. When you need bespoke logic, a custom
step is a small **Ability Module** subclass with a single body to implement, and
a custom target source is a small **Source Accessor** subclass. The ready-made
library is reuse, not a ceiling. Both paths are covered in
[reference-programs.md](reference-programs.md).

## The one authoring rule

One rule governs every step's body, and the whole undo / save / preview
guarantee rests on it: **a step touches the game only through the entity
system's command-routed calls and the context it is handed.** Anything else —
spawning an actor directly, mutating the grid behind the system's back, drawing
a random number from a sequential source — is invisible to undo and to what-if
runs, and will make a preview disagree with the real thing. The framework backs
the rule with a static linter and a per-ability *Validate Ability* action rather
than walls; see [reference-tooling.md](reference-tooling.md).

## Grant and activate an ability

=== "Blueprint"
    1. Author the ability as a data-table row, exactly like any other entity —
       give it a **Tagged Data** entry keyed `Data.Ability.Program` (a reference
       to a ready-made or custom program) and a `Data.Ability.Activation` entry
       (its presence is what makes the entity manually activatable).
    2. Grant it to the caster as a child in an ability slot with the ordinary
       entity-hierarchy call. Because it is entity state, the grant undoes and
       saves with the unit.
    3. On the "cast" input, get the resolution subsystem and call **Try Activate
       Ability** — the first pin is the Caller (the unit acting), the second is
       the Ability (the granted entity). The order matters; both pins are the
       same type.
    4. Read the outcome with **Get Last Order Resolution**: *committed*,
       *resolved-but-changed-nothing*, or *aborted*. Surface the middle case as
       "no valid target" instead of a dead-feeling click.

=== "C++"
    ```cpp
    // The ability is an ordinary entity: grant it as a child in an ability slot
    // through the entity system's command-routed hierarchy call — an undoable
    // change. (The grant call itself lives in the GameEntity section.)

    UPAbResolutionSubsystem* Rules = UPAbResolutionSubsystem::Get(this);

    // Caller first, Ability second — both are FPGeEntityRef, so mind the order.
    // Returns immediately: true if the action was accepted (its use conditions
    // passed and it began resolving), false if refused.
    if (Rules->TryActivateAbility(Caster, GrantedAbility))
    {
        // Once it has resolved, tell a real commit from a no-op — the
        // "I clicked and nothing happened" diagnostic.
        if (Rules->GetLastOrderResolution() == EPAbOrderResolution::ResolvedEmpty)
        {
            // e.g. nothing was in range — tell the player why.
        }
    }
    ```

## The five authoring entries

Everything an ability carries lives under five tagged-data entries. Each is an
ordinary tagged-data value on the entity, so it inherits from the template,
saves, and replays like any other data — and an entity carries only the ones it
needs.

| Tagged-data entry | What it does |
|---|---|
| `Data.Ability.Program` | References the program of steps the entity runs — when activated, or when one of its triggers fires. Points to a ready-made or custom program. |
| `Data.Ability.Activation` | Its presence marks the entity as something a player can manually activate. Optional fields tune the UI category and completion policy. |
| `Data.Ability.UseConditions` | The conditions checked cheaply, per frame, to decide whether the ability is usable right now — the greyed-button test, which can name the failing condition. |
| `Data.Ability.Triggers` | The entity-carried triggers: which events the entity reacts to, in which phase (interrupt or reaction), and the program each one runs. |
| `Data.Ability.Intent` | An optional presentation-intent tag that colours how the committed changes play out on screen. Omit it for the generic default. |

The full native-tag list and the project-settings knobs live in
[reference-tooling.md](reference-tooling.md).

!!! note "The config you author against is stable"
    The structs behind these entries ship as a versioned contract — changes to
    them are additive-only, so authored content never silently breaks on a
    framework upgrade.

## Where to go next

- **[Guides](guides.md)** — task recipes: author your first ability from a
  ready-made program, add a target-selection decision point, author a reactive
  trigger, wire a custom targeting UI, and validate an ability.
- **[Programs & Steps](reference-programs.md)** — the
  ready-made program library (the 80% path), the step base class and
  building-block steps, sub-programs and iteration, and the context handed to
  every step.
- **[Targeting & Running Abilities](reference-targeting.md)** — how a step finds
  what it acts on, the built-in target-selection decision point, and the
  resolution and targeting-session API a UI drives, including painting
  candidates on the grid.
- **[Triggers & Reactions](reference-triggers.md)** — authoring entity-carried
  triggers: interrupts and reactions, listening channels, order-sign phase
  selection, and declaration windows.
- **[Previews, AI & What-Ifs](reference-previews.md)** — hover previews, the
  availability checks, the automatic playout layer, and the enemy-AI hooks.
- **[Configuration, Tags & Tooling](reference-tooling.md)** — the full
  `Data.Ability.*` and native-tag reference, project settings, the ability
  linter, and the *Validate Ability* action.
- **[Abilities and step-by-step resolution](../../concepts/abilities-and-resolution.md)**
  — the model and the rationale behind all of the above.
