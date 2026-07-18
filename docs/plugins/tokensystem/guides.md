# TokenSystem Guides

For developers with the plugin enabled who want to get something on screen. Each
section is a short, followable recipe. For the full type-by-type surface see the
[API Reference](reference.md); for the model behind it all see
[Tokens & Cues](../../concepts/tokens-and-cues.md).

## Give an entity a token

The most common task needs no cue handling and no spawn call. You author which
token class visualizes an entity as data on its template, and the registry does
the rest: it spawns the token when the entity is created, positions it, snaps it
to state, and releases it when the entity is destroyed — including on undo and
load.

=== "Blueprint"
    1. In the unit's data-table row, add a **Tagged Data** entry keyed
       `Data.Presentation.TokenClass` whose value is your token Blueprint — for
       example `BP_GoblinToken`, derived from the **PTk Token Actor** base for a
       3D mini, or the **PTk Token Widget** base for a card face.
    2. Create the entity as usual (**Create Entity From Row**). The registry
       spawns the token automatically — there is no "spawn token" node.
    3. Wire an **Undo** button. Undoing the creation releases the token; redo
       brings it back. You wrote no presentation code to make that work.

    An entity that names no token class — and matches no project-wide fallback —
    is simply invisible, which is exactly right for a rules-only entity like a
    status effect or a zone.

=== "C++"
    ```cpp
    // Author which token class visualizes this entity. Usually done in the
    // data-table row; in code, write the presentation facet onto the template:
    FInstancedStruct TokenData = UPTkTokenSubsystem::MakeTokenClassData(BP_GoblinToken);
    State->SetTaggedData(GoblinTemplate,
                         PTkTokenTags::TAG_Data_Presentation_TokenClass, TokenData);

    // From here the registry does the work. Creating the entity spawns its token;
    // destroying it (or undoing the creation) releases it. To reach a live token:
    UPTkTokenSubsystem* Registry = UPTkTokenSubsystem::Get(this);
    TScriptInterface<IPTkTokenInterface> Token = Registry->GetToken(GoblinRef);
    ```

!!! tip "The snap guarantee"
    With nothing but a token class assigned, the board is always correct — it
    snaps to the truth on every change, including undo and load. Everything below
    is *polish*, not correctness. If a token ever shows the wrong state, that's a
    bug in the snap, not something a cue was meant to paper over.

## Write a cue handler

A **cue handler** is a small, reusable unit of visual logic keyed by a cue tag. It
renders one cue against one token and, when its visual finishes, calls **Complete**
on the cue context so the next beat can start. Handlers register themselves
globally by their class defaults — the registry scans them at startup — so a thin
token gets sensible behavior for free.

=== "Blueprint"
    1. Create a Blueprint subclass of the **PTk Cue Handler** base.
    2. In class defaults, set **Handled Cue** to the tag you render — for example
       `Cue.TakeDamage`.
    3. Implement the **Handle** event. Drive your reaction on the token through a
       [capability](#drive-a-token-through-a-capability) (next section), and call
       **Complete** on the cue context when the animation ends.
    4. That's it — the handler is now the global default for `Cue.TakeDamage`. To
       give one token a *signature* look, don't subclass it: add a row to that
       token's **Cue Overrides** map keying `Cue.TakeDamage` to your handler
       class. Every other token keeps the global default.

=== "C++"
    ```cpp
    UCLASS()
    class UMyDamageCueHandler : public UPTkCueHandler
    {
        GENERATED_BODY()
    public:
        UMyDamageCueHandler() { HandledCue = PTkTokenTags::TAG_Cue_TakeDamage; }

        void Handle_Implementation(const TScriptInterface<IPTkTokenInterface>& Token,
                                   const FPTkCue& Cue, UPTkCueContext* Ctx) override
        {
            // Drive whatever the token can do; the capability owns Complete().
            if (Token.GetObject()->Implements<UPTkMontageCapable>())
            {
                IPTkMontageCapable::Execute_PlayReaction(Token.GetObject(), Cue.Intent, Ctx);
                return;
            }
            Ctx->Complete();   // no capability — finish gracefully; the sequence moves on
        }
    };
    ```

!!! warning "Always complete the context"
    Every path through **Handle** must eventually reach **Complete** — either your
    own visual calls it when it finishes, or a capability you delegate to owns it,
    or you call it yourself for the no-op case. A cue that never completes stalls
    anything waiting behind it. Handlers produce *no* commands and touch *no* game
    state; they only read the cue and drive the token.

## Drive a token through a capability

A handler must not care whether it's animating a 3D mini or a 2D card. That's what
**capabilities** are for: small opt-in interfaces a token advertises. A handler
*asks and skips* — it drives the capability if the token has it and no-ops cleanly
if it doesn't. So you implement a capability once on the token, and any handler
that speaks that capability drives it.

=== "Blueprint"
    On your token Blueprint, add the interface for each capability it supports and
    implement its events:

    - **Movable** — implement **Move Along Path**; move the token through the
      points and call **Complete** when the motion ends.
    - **Montage Capable** — implement **Play Reaction**; play the montage/flipbook
      for the reaction tag and call **Complete** when it finishes.
    - **Stat Display** — implement **Set Value** (the instant snap) and
      **Animate To** (the tween for a live change; call **Complete** when it
      settles).

    You never *register* a capability — a handler queries it and skips the ones
    your token doesn't implement.

=== "C++"
    ```cpp
    // A handler drives a capability without knowing the concrete token class:
    if (Token.GetObject()->Implements<UPTkMovable>())
    {
        const FPTkMovePayload Move = Cue.Payload.Get<FPTkMovePayload>();
        IPTkMovable::Execute_MoveAlongPath(Token.GetObject(), Move.Path, Ctx);
    }
    else
    {
        Ctx->Complete();   // not movable — nothing to animate; complete cleanly
    }
    ```

!!! tip "One handler, every surface"
    Because a handler talks to the capability and never the class, a single
    `Cue.Move` handler walks a character mini and slides a card widget
    identically — each implements **Movable** in its own terms. Keep the token
    interface tiny and put everything a handler drives in a capability.

## Play a cue yourself

The registry's snapping is automatic — you get a correct board with no cues at
all. Reach for a cue when you want richer playout. After a commanded move, for
instance, the token has already snapped to the destination cell; play a `Cue.Move`
carrying the path so it *walks* there instead of blinking.

=== "Blueprint"
    1. After the move command commits, build a **Cue**: set **Intent** to
       `Cue.Move` and pack the path into its **Payload** (a **Move Payload**).
    2. Construct a **Cue Context**.
    3. **Get** the token subsystem and call **Play Cue** with the unit's
       reference, the cue, and the context. The registry resolves the handler and
       drives the walk, fanning the same cue to any auxiliary surfaces.

=== "C++"
    ```cpp
    // The move command already committed and the token already snapped to the new
    // cell. Play a Cue.Move so it *walks* there instead:
    FPTkMovePayload Move;
    Move.Path = WorldPathFromCells(Cells);

    FPTkCue Cue;
    Cue.Intent  = PTkTokenTags::TAG_Cue_Move;
    Cue.Payload = FInstancedStruct::Make(Move);

    // PlayCue requires a context — construct one and pass it. Bind OnComplete if
    // you want to run something after the walk finishes.
    UPTkCueContext* Ctx = NewObject<UPTkCueContext>(this);
    UPTkTokenSubsystem::Get(this)->PlayCue(UnitRef, Cue, Ctx);
    ```

!!! note "A cue depicts a change that already happened"
    Cues are post-commit and presentation-only. Play a cue *after* the state
    change it depicts is committed and snapped — never route a cue through the
    event system, and never let a game rule listen for one. The in-between values
    an animation needs live only in the cue's payload; game state has already
    settled on the final value.

## Bind an entity widget to one value

An entity is rarely on screen in just one place — a unit might be a mini on the
board, a row in a party frame, and a portrait on a character sheet at once. The
extra surfaces are **auxiliary channels**, and the shipped building block for one
is the **entity widget**: a readout bound to one entity and one value that snaps
that value to the truth, receives cues to animate it, and can paint a preview
overlay. You implement only the paint.

=== "Blueprint"
    1. Create a widget subclass of the **PTk Entity Widget** base; set its
       **Facet Tag** to the value it shows — a stat like `Stat.Health` or a
       `Data.*` entry like a portrait.
    2. Implement **On Stat Display Changed** to snap the bar's fill or number to
       the incoming value (this is the bind / load / undo path — no animation).
       For a `Data.*` facet implement **On Tagged Data Display Changed** instead.
    3. Implement **On Preview Delta** to paint a predicted `From`→`To` band without
       disturbing the current fill.
    4. When selection changes, call **Set Bound Entity** with the new unit and
       facet tag. It deregisters the old binding, registers the new one with the
       registry, and snaps — one node handles the whole re-bind. Passing an
       invalid entity clears the binding.

=== "C++"
    ```cpp
    // Bind a readout to one entity and one value. The base registers it with the
    // registry as an auxiliary channel, so the registry snaps it — the widget
    // never subscribes to change events itself. Re-callable: pass a new entity to
    // re-bind (a character sheet does this as selection changes).
    HealthBar->SetBoundEntity(SelectedUnit, TAG_Stat_Health);
    ```

!!! tip "A character sheet is a pattern, not a class"
    A long-lived panel — a party frame, a character sheet — is not a special
    class. It's a plain container holding several entity widgets that it re-binds
    to whichever unit is selected. Selection changes are just re-binds. And a HUD
    readout of an entity is always an *auxiliary* channel, never a second token:
    there is exactly one primary token per entity, and it is the only surface that
    gates playout timing. Auxiliaries are fire-and-forget — whether a panel is
    open can never change how long a beat takes.
