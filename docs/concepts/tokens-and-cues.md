# Presentation: Tokens and Cues

For developers putting the game on screen — minis, card faces, health bars, character sheets — without ever letting the screen corrupt the game. After reading, you'll know how every entity gets an on-screen stand-in that stays correct through undo and load with zero custom code, how animated playout is layered on top with cues and cue handlers, and how one entity can drive many surfaces at once.

This is the second capstone. It pays off two promises from earlier pages: that [visuals are a projection of state](entities-as-data.md), and that [the on-screen token](derived-state.md) stays right by reacting to change. Everything the framework has built — data, commands, derived state, grids, turns — reaches the screen here, and the whole page rests on one rule: presentation reads game state and is *driven* by it, but never changes it.

## The screen is a projection

An entity holds no pointer to an Actor, and an Actor holds no game state. The visual layer sits entirely outside authoritative state: it reads that state and reacts to it, and it produces no changes of its own. Because presentation writes nothing, an undo, a reload, or a replay with a completely different set of panels open all land on byte-identical game state — what was on screen at the time never mattered to the outcome.

That gives you a clean two-tier model, and the honest way to hold it in your head is:

- **Always correct, for free.** The framework keeps an on-screen stand-in for every entity and *snaps* it to the truth on every change — create, destroy, move, load, undo. With no presentation code written at all, the board is already right.
- **Animated playout, opt-in.** On top of the snapping you layer sequenced, animated presentation with **cues**. Snapping is automatic; cues are polish that somebody plays.

Keep that split straight and the rest of the page follows from it.

## Tokens: one stand-in per entity

A **Token** is whatever puts one entity on screen — an Actor for a unit's mini, a UMG widget for a card face, a plain object for a headless readout. It is an *interface* (`IPTkTokenInterface`), not a base class, so an `AActor`, a `ACharacter`, a `UUserWidget`, or a bare `UObject` can all be Tokens. A Token implements three lifecycle hooks: bind to an entity once (`InitToken`), snap its visuals to current state with no animation (`SyncToState`), and free itself (`ReleaseToken`).

Because entities are pointerless data, the entity-to-Token map can't live on the entity. It lives in the **token registry** — a world subsystem that subscribes to change events and reconciles the whole set for you: it spawns a Token when an entity is created, releases it when the entity is destroyed, and re-snaps every Token when state is restored on load or rewound by undo. You never write that bookkeeping.

Which Token class visualizes an entity is authored as data on the entity's template, under the tag `Data.Presentation.TokenClass`. The look sits right next to the unit definition. An entity that names no Token class — and matches no project-wide fallback — is simply invisible, which is exactly what you want for the rules-only entities (a status effect, a zone) that should never appear.

=== "Blueprint"
    1. In the unit's data-table row, add a **Tagged Data** entry keyed `Data.Presentation.TokenClass` whose value is your Token Blueprint (for example `BP_GoblinToken`, derived from the actor-token base).
    2. Create the entity as usual with **Create Entity From Row**. The registry spawns the Token automatically — there is no "spawn token" call.
    3. Wire an **Undo** button. Undoing the creation despawns the Token; redo brings it back. You wrote no presentation code to make that work.

=== "C++"
    ```cpp
    // Author which Token class visualizes this entity — usually in its data-table row,
    // or on the template's tagged data in code:
    FInstancedStruct TokenData = UPTkTokenSubsystem::MakeTokenClassData(BP_GoblinToken);
    State->SetTaggedData(GoblinTemplate, PTkTokenTags::TAG_Data_Presentation_TokenClass, TokenData);

    // From here on the registry does the work. Creating the entity spawns its Token;
    // destroying it (or undoing the creation) releases it. To reach a live Token:
    UPTkTokenSubsystem* Registry = UPTkTokenSubsystem::Get(this);
    TScriptInterface<IPTkTokenInterface> Token = Registry->GetToken(GoblinRef);
    ```

A unit on the board is a special case worth calling out: its Token derives its world position from the entity's placement data, so a [born-placed unit](grids-and-occupancy.md) lands on the correct cell with no origin-pop and a commanded move re-snaps it to the new cell — all through the registry, with no positioning code in the Token. The grid-aware Token base adds path-walking on top, so a played move cue animates the step instead of teleporting.

!!! tip "The snap guarantee"
    With zero presentation code beyond assigning a Token class, the board is always correct — it snaps to the truth on every change, including undo and load. Everything after this section is *polish*, not correctness. If a Token ever shows the wrong state, that is a bug in the snap, not something a cue was supposed to fix.

The grid-aware token that snaps a mini to its cell and walks it along a move is the [grid token](../plugins/gridentity/reference.md#the-grid-token) in the GridEntity reference.

For the full registry API — resolving a token, the automatic spawn and release, and playing a cue — see [The token registry](../plugins/tokensystem/reference.md#the-token-registry) in the TokenSystem reference.

## State commits instantly; the screen plays it out

Here is why cues exist at all. Game state commits the *whole* change before a single pixel moves. A three-hit combo takes a unit from 100 to 70 the instant the [command](commands-and-undo.md) runs; the intermediate values 90 and 80 never exist in game state. If your health bar tried to *read* state to animate the drop, there would be nothing to read but the final 70.

So live displays are **pushed**, never pulled. During playout, the only place the in-between values survive is the cue's payload — a cue hands the bar "90," then "80," while state has long since settled on 70. This is the presentation-side sharpening of the [derived-state subscription rule](derived-state.md): a reactive consumer reconciles to *what state now is*, but an animated surface is *fed* a sequence that state no longer holds.

!!! warning "Presentation surfaces don't subscribe"
    A game-facing Token or readout should never bind its own change-event subscriptions. The registry drives every surface — it snaps them on load and undo, and cues animate them during playout. A surface that subscribed on its own would snap to the committed value at the moment of the command and skip the animation entirely, and it would fight the registry during undo.

    The health-bar example on the [derived-state page](derived-state.md) — a widget that subscribes to `OnAnyEntityChanged` and reconciles — remains exactly the right *pattern* for tools and debug UI, where you want the raw truth with no choreography. For game-facing presentation, use the building blocks on this page instead: let the registry drive the snap, and let cues carry the animation.

## Cues: what happened, not how it looks

A **Cue** is a small, semantic message addressed to one entity's visuals. It carries three things: an intent tag in the `Cue.*` namespace that names *what happened* (`Cue.TakeDamage.Fire`, `Cue.Move`), an optional instigator (the attacker, the spell), and a typed payload with the facts a handler needs (the path to walk, the amount lost). It says *what*, never *how it looks* — the look is the handler's job.

Cues are post-commit and presentation-only: a cue plays after the state change it depicts is already committed and snapped. They are async by design — a handler reports completion when its visual finishes, so a sequence of cues plays out cleanly one after another instead of stepping on itself.

!!! note "Three broadcasts, three jobs"
    The framework has three separate "something happened" signals, and they never substitute for one another:

    - **Change events** reconcile derived state — read-only, they keep caches and displays in sync (the [derived-state page](derived-state.md)).
    - **Gameplay events** run game rules — reactions, interrupts, ordering (see [Events and Reactions](events-and-reactions.md)).
    - **Cues** drive visuals — semantic, post-commit, presentation-only.

    Cues never route through the event system, and a rule never listens for a cue. Mixing them is the fastest way to make undo and replay drift.

## Cue handlers: the look, shared through the registry

A **cue handler** renders one cue against one Token. It is a small, reusable, Blueprint-authorable unit of visual logic, keyed by cue tag with most-specific-wins fallback: a `Cue.TakeDamage.Fire` cue looks for a `Cue.TakeDamage.Fire` handler, then falls back to `Cue.TakeDamage`. Handlers register themselves globally by their class defaults — the registry scans them at startup — so a thin Token gets sensible default behavior for free.

Reuse is by the registry, not by inheritance. When one Token needs a signature look — the boss's custom flinch — you don't subclass it; you add a single row to that Token's override map pointing the tag at a shared handler class. No switch statements inside Tokens, no inheritance chains to reuse a montage.

=== "Blueprint"
    1. Create a Blueprint subclass of the **Cue Handler** base.
    2. In class defaults, set **Handled Cue** to `Cue.TakeDamage`.
    3. Implement the **Handle** event: play your reaction on the Token through a capability (next section), and call **Complete** on the cue context when the animation ends so the sequence can continue.
    4. To give one Token a special look, add a row to its **Cue Overrides** map: key `Cue.TakeDamage`, value your handler class. Every other Token keeps the global default.

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
            // Drive whatever the Token can do; the capability owns Complete().
            if (Token.GetObject()->Implements<UPTkMontageCapable>())
            {
                IPTkMontageCapable::Execute_PlayReaction(Token.GetObject(), Cue.Intent, Ctx);
                return;
            }
            Ctx->Complete(); // no capability — finish gracefully, sequence moves on
        }
    };
    ```

Ready to write one? Follow [Write a cue handler](../plugins/tokensystem/guides.md#write-a-cue-handler) in the TokenSystem guides.

## Capabilities: drive a token without knowing its class

A handler must not care whether it is animating a 3D mini or a 2D card. That is what **capabilities** are for: small opt-in interfaces a Token advertises — movable, montage-capable, a numeric stat display, previewable. A handler asks-and-skips. A knockback handler moves anything that is movable and silently no-ops on a Token that isn't. A damage handler plays a reaction on anything montage-capable and animates the number on anything that is a stat display. One handler serves the character mini and the card widget alike, because it talks to the capability, never the class.

The core Token interface stays deliberately tiny — three lifecycle hooks and nothing else — precisely so it can't grow into a fat base class everything must implement. Everything a handler drives lives in a capability a Token opts into.

=== "Blueprint"
    On your Token Blueprint, add the interfaces for the capabilities it supports — for example **Movable** (implement **Move Along Path**, calling **Complete** when the motion ends) or **Montage Capable** (implement **Play Reaction**). A handler queries these and skips the ones your Token doesn't have; you never register a capability, you just implement it.

=== "C++"
    ```cpp
    // A handler drives a capability without knowing the concrete Token class:
    if (Token.GetObject()->Implements<UPTkMovable>())
    {
        const FPTkMovePayload Move = Cue.Payload.Get<FPTkMovePayload>();
        IPTkMovable::Execute_MoveAlongPath(Token.GetObject(), Move.Path, Ctx);
    }
    else
    {
        Ctx->Complete(); // not movable — nothing to animate; complete cleanly
    }
    ```

For the full list of capabilities and how a handler drives them, see [Capabilities](../plugins/tokensystem/reference.md#capabilities) in the TokenSystem reference.

## One entity, many surfaces

An entity is rarely on screen in just one place. A unit might be a mini on the board, a row in a party frame, a portrait on an open character sheet, and a line in the combat log — all at once. The framework calls these **presentation channels**. The Token is the entity's **primary** channel; any number of **auxiliary** surfaces can register for the same entity and receive the identical cues.

The one rule that keeps this safe: *only the primary channel gates playout timing.* Auxiliaries are always fire-and-forget — their cues play, but nothing ever waits on them. That is deliberate and load-bearing. It means whether a panel happens to be open can never change how long a beat takes, and a replay runs the same length whether the character sheet was up or not.

The shipped building block for auxiliary surfaces is the **entity widget** — a small readout base bound to one entity and one value (a stat or a data entry). Bind it and it snaps that one value to the truth, receives cues to animate it, and can paint a preview overlay — you implement only the paint. A long-lived panel like a character sheet is not a special class; it is a plain container that holds several entity widgets and re-binds them to whichever unit is selected. Selection changes are just re-binds.

=== "Blueprint"
    1. Create a widget subclass of the **Entity Widget** base; set its **Facet Tag** to `Stat.Health`.
    2. Implement **On Stat Display Changed** to snap the bar's fill to the incoming value (this is the load/undo/bind path — no animation).
    3. Implement **On Preview Delta** to paint a predicted `From`→`To` band without disturbing the current fill.
    4. When selection changes, call **Set Bound Entity** with the new unit and `Stat.Health`. It deregisters the old binding, registers the new one with the registry, and snaps — one node handles the whole re-bind.

=== "C++"
    ```cpp
    // Bind a readout to one entity and one value. The base registers it with the
    // registry as an auxiliary channel, so the registry snaps it — the widget never
    // subscribes to change events itself. Re-callable: pass a new entity to re-bind.
    HealthBar->SetBoundEntity(SelectedUnit, TAG_Stat_Health);
    ```

!!! tip "Party frames and sheets are auxiliaries, not second Tokens"
    A HUD readout of an entity is an auxiliary channel, never a second Token. There is exactly one primary Token per entity — the one the registry spawns and positions — and it is the only surface that gates timing. Register extra surfaces as auxiliaries (an entity widget does this for you on bind); don't try to give one entity two Tokens.

For the channel API and the entity-widget building block, see [Presentation channels and the entity widget](../plugins/tokensystem/reference.md#presentation-channels-and-the-entity-widget) in the TokenSystem reference.

## Previews ride the same rails

Hovering a move or an attack, a game wants to paint the *predicted* outcome — the damage a target would take, the cell a unit would end on, a translucent **ghost** of something that would be summoned. Presentation supports this through the same handlers and capabilities as real playout: a previewable surface overlays the predicted change non-destructively (a health bar stays full and paints the red band it *would* lose) instead of snapping to it, and the overlay is torn down cleanly when the hover moves on. Real playout and predicted overlay are distinct by design — a preview never commits and never animates the live path.

What *computes* those predictions — running the hovered action against a temporary copy of the game state and discarding it afterward — is the ability system's work, documented in a later section. This page only establishes the presentation side: the surfaces, the ghosts, and the overlays exist and share the cue rails, waiting for a driver to feed them.

## Where cues come from today

The registry's snapping is fully automatic — you get a correct board with no cues at all. Cues are the layer you reach for when you want richer playout, and today you play them yourself, wherever you want the extra beat. After a commanded move, for instance, you can play a `Cue.Move` carrying the path so the mini walks the route instead of blinking to the destination.

=== "Blueprint"
    1. After the move command commits, build a **Cue** struct: set **Intent** to `Cue.Move` and pack the world-space path into its **Payload** (a **Move Payload**).
    2. Construct a **Cue Context**.
    3. Get the token subsystem and call **Play Cue** with the unit's reference, the cue, and the context. The registry resolves the handler and drives the walk.

=== "C++"
    ```cpp
    // The move command already committed and the Token already snapped to the new cell.
    // Play a Cue.Move so it *walks* there instead:
    FPTkMovePayload Move;
    Move.Path = WorldPathFromCells(Cells);

    FPTkCue Cue;
    Cue.Intent  = PTkTokenTags::TAG_Cue_Move;
    Cue.Payload = FInstancedStruct::Make(Move);

    UPTkCueContext* Ctx = NewObject<UPTkCueContext>(this);
    UPTkTokenSubsystem::Get(this)->PlayCue(UnitRef, Cue, Ctx);
    ```

The next step — turning committed state changes into cues *automatically*, choosing snap-versus-sequence, and fanning previews out on hover — is work the ability system does for you, documented in a later section. Until then the story is exactly the two tiers this page opened with: snapping is free and automatic, and cues are opt-in polish that you play. Nothing about hand-playing cues changes when the automatic layer arrives — the same cues, handlers, and capabilities are what it will drive.

Ready to play one yourself? Follow [Play a cue yourself](../plugins/tokensystem/guides.md#play-a-cue-yourself) in the TokenSystem guides.
