# TokenSystem API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for the model see
[Tokens & Cues](../../concepts/tokens-and-cues.md).

All public types carry the `PTk` prefix. Signatures below are hand-written to show
the shape of each API; they are illustrative, not source excerpts.

## The token registry

`UPTkTokenSubsystem` is a `UWorldSubsystem` — one per running play world. It is the
entity-to-token map and the reconciliation hub. Because entities are pointerless
value structs, the mapping can't live on the entity; it lives here. The registry
subscribes to the [GameEntity change-event feed](../gameentity/reference-events.md)
and, with no code from you, spawns a token when an entity is created, releases it
when the entity is destroyed, and re-snaps every token when state is restored on
load or rewound by undo.

```cpp
// Resolve the registry through any world-context object. In Blueprint the context
// defaults to self.
static UPTkTokenSubsystem* Get(const UObject* WorldContextObject);

// The token visualizing this entity, or an invalid interface if the entity is
// headless (no token class resolved). A plain lookup — never touches the stack.
TScriptInterface<IPTkTokenInterface> GetToken(const FPGeEntityRef& Ref) const;
```

**Playing a cue.** `PlayCue` resolves the handler for the cue's intent and drives
it against the entity's token, fanning the same cue to every auxiliary channel
registered for that entity. It **requires** a cue context — construct one and pass
it (see [Playing a cue yourself](guides.md#play-a-cue-yourself)).

```cpp
// Resolve a handler for Cue.Intent and play it on Ref's token. If no token or no
// handler is found, Ctx->Complete() fires immediately (a graceful no-op). The
// handler and Ctx are kept alive until completion so async playback isn't GC'd.
void PlayCue(const FPGeEntityRef& Ref, const FPTkCue& Cue, UPTkCueContext* Ctx);
```

**Assigning a token class.** `MakeTokenClassData` builds the presentation facet you
store under `Data.Presentation.TokenClass` on an entity or its template; the
registry reads it back when it spawns the token. It hides the stored wire format,
so you assign a token without depending on TaggedData internals.

```cpp
// C++ only — there is no Blueprint node for this. In Blueprint you author the
// token class directly in the data-table row's tagged data instead.
static FInstancedStruct MakeTokenClassData(TSubclassOf<UObject> TokenClass);
```

!!! note "How a token class is resolved"
    When an entity is created the registry looks for its token class in order:
    the entity's own `Data.Presentation.TokenClass` facet (which delegates to its
    template), then the per-category fallbacks in project settings, then the
    default fallback. An entity that matches none is left **headless** — no token,
    which is exactly right for a rules-only entity. Fallbacks are configured on
    `UPTkTokenSettings` (see [Settings](#settings)).

### Auxiliary presentation channels

An entity may drive more than its primary token — a HUD party-frame row, an open
character-sheet readout. These extra surfaces register as **auxiliary channels**;
`PlayCue` fans each cue to them alongside the primary, always fire-and-forget.

```cpp
// C++ only — no Blueprint nodes. In practice you never call these directly: the
// entity widget's SetBoundEntity registers and deregisters itself for you.
void RegisterAuxiliaryChannel(const FPGeEntityRef& Ref,
                              const TScriptInterface<IPTkTokenInterface>& Channel);
void UnregisterAuxiliaryChannel(const FPGeEntityRef& Ref,
                                const TScriptInterface<IPTkTokenInterface>& Channel);
```

!!! warning "Only the primary channel gates timing"
    Auxiliary cues play, but nothing ever waits on them — their completion is
    never wired to a sequence's timing. That is deliberate: whether a panel
    happens to be open can never change how long a beat takes, and a replay runs
    the same length whether the character sheet was up or not. There is exactly
    one primary token per entity, and it is the only surface that gates playout.

!!! note "The previews-and-ghosts surface is driven elsewhere"
    The registry also exposes a preview-session surface (translucent ghosts and
    non-destructive overlays for hover what-ifs). Those hooks exist so the
    [previewable and ghostable capabilities](#capabilities) have something to
    drive them, but *what* computes and feeds a preview — running a hovered action
    against a throwaway copy of state — is the ability system's job, documented in
    a later section. This page establishes only the presentation side.

## The token interface and lifecycle

`IPTkTokenInterface` is what anything on screen implements. It is an *interface*,
not a base class, so an `AActor`, a `ACharacter`, a `UUserWidget`, or a bare
`UObject` can all be tokens — they share no common base below `UObject`. The
surface is deliberately tiny: three lifecycle hooks plus a cue-override accessor,
so it can never grow into a fat base class everything must implement. Everything a
handler drives lives in an opt-in [capability](#capabilities).

```cpp
// Bind to an entity; cache the Ref; one-time setup. Called once on spawn.
void InitToken(const FPGeEntityRef& Entity);

// Snap visuals to current authoritative state — NO animation. Runs on spawn,
// load, undo-rebuild, and every registry re-snap. Skipping every cue and calling
// SyncToState alone still yields a correct board — this is what makes a token a
// projection of state, never state.
void SyncToState();

// The entity was destroyed — free the token. A synchronous snap-remove: an actor
// is destroyed, a widget removed from its parent. There is no death cue.
void ReleaseToken();

// This token's own cue-handler overrides, consulted before the global registry.
// Defaults to empty; the convenience bases return their authored map.
TMap<FGameplayTag, TSubclassOf<UPTkCueHandler>> GetCueOverrides() const;
```

!!! note "Release is a snap-remove, not choreography"
    A token frees itself immediately when its entity is destroyed. There is no
    built-in death animation — if you want one, play a cue *before* you destroy
    the entity, while its token is still live. Undo of a destroy re-spawns the
    token and snaps it, symmetric with the create path.

### Convenience token bases

You rarely implement the interface by hand. Two `Blueprintable`, `Abstract` bases
pre-implement it and cache the bound entity, so you subclass one and override the
lifecycle events:

- **`APTkTokenActor`** — an `AActor` token for a 3D mini or board piece. The
  registry positions it (through the grid-aware resolver, when one is bound).
- **`UPTkTokenWidget`** — a `UUserWidget` token for a card face, portrait, or log
  line. Adding it to the viewport is the token's own `InitToken` / `SyncToState`
  job.

Both expose a `CueOverrides` map you author in class defaults: key a `Cue.*` tag
to a handler class to give *this* token a signature look without subclassing a
handler.

## Cues

A `FPTkCue` is a small semantic message addressed to one entity's visuals. It says
*what happened*, never *how it looks* — the look is a handler's job.

```cpp
struct FPTkCue
{
    // What happened, in the Cue.* namespace (e.g. Cue.TakeDamage.Fire, Cue.Move).
    // Resolved to a handler by most-specific-wins fallback.
    FGameplayTag     Intent;

    // Optional source of the cue — the attacker, the spell entity. May be invalid.
    FPGeEntityRef    Instigator;

    // Typed per-cue data the handler reads — a movement path, a damage amount.
    FInstancedStruct Payload;

    // True when this cue is a non-destructive preview overlay (a hover what-if)
    // rather than a real playout beat. A preview-aware handler paints a predicted
    // delta instead of animating. Left false for ordinary live cues.
    bool             bIsPreview = false;
};
```

`UPTkCueContext` is the async-completion handle a cue carries. A handler whose
visual takes time calls `Complete()` when its montage or timeline finishes, so a
sequence of cues plays cleanly one after another.

```cpp
class UPTkCueContext : public UObject
{
    // Blueprint-assignable. Fires once when the cue finishes (via Complete()).
    FPTkOnCueComplete OnComplete;

    // Signal that the cue's visual has finished. Double-fire guarded — the
    // continuation runs exactly once; later calls no-op.
    void Complete();

    // Whether Complete() has already fired.
    bool IsComplete() const;

    // The entity this cue targets. Stamped by PlayCue before dispatch.
    FPGeEntityRef TargetRef;
};
```

!!! warning "Cues are presentation-only"
    A cue plays *after* the state change it depicts is committed and snapped, and
    it never routes through the event system. Change events reconcile derived
    state; gameplay events run rules; cues drive visuals — three separate signals
    that never substitute for one another. A rule that listened for a cue, or a
    cue that tried to mutate state, is the fastest way to make undo and replay
    drift. See [Derived State & Events](../../concepts/derived-state.md).

## Cue handlers

`UPTkCueHandler` is a reusable, tag-keyed, `Blueprintable` unit of visual logic
that renders one cue against one token through the token's capabilities. It
produces no commands and touches no state.

```cpp
class UPTkCueHandler : public UObject
{
    // The cue tag this handler renders. Set in class defaults. Scanned off the
    // class default object at startup to build the global registry.
    FGameplayTag HandledCue;

    // Render the cue against the token via its capabilities. Async: call
    // Ctx->Complete() when the visual finishes. The default completes immediately.
    void Handle(const TScriptInterface<IPTkTokenInterface>& Token,
                const FPTkCue& Cue, UPTkCueContext* Ctx);
};
```

**Registration is by the registry, not inheritance.** At startup the registry
scans handler class defaults and maps each `HandledCue` tag to its class, so a
thin token gets sensible default behavior for free. To give one token a signature
look, you don't subclass — you add a row to that token's `CueOverrides` map
pointing the tag at a shared handler class.

**Resolution is most-specific-wins.** For a `Cue.TakeDamage.Fire` cue the registry
walks the tag from most specific to least:

1. the token's own `CueOverrides` map — `Cue.TakeDamage.Fire`, then `Cue.TakeDamage`;
2. the global registry — the same, most-specific first;
3. nothing found — the context is completed immediately (a graceful no-op).

So a token's override always beats the global default, and a specific tag always
beats its parent.

## Capabilities

A handler must not care whether it is animating a 3D mini or a 2D card.
**Capabilities** are small opt-in interfaces a token advertises; a handler
*asks and skips* — it drives the capability if the token implements it and no-ops
cleanly if it doesn't. Each is queried with `Implements<...>()` and called through
`Execute_...`. The five shipped capabilities:

| Capability | What a token that implements it can do |
|---|---|
| `IPTkMovable` | Travel along a path (`MoveAlongPath`, carrying an `FPTkMovePayload`); report its anchor. A walking mini and a sliding card implement it in their own terms. |
| `IPTkMontageCapable` | Play a tagged reaction — a montage, flipbook, or timeline — for a reaction tag (`PlayReaction`). |
| `IPTkStatDisplay` | Show a numeric value and advance it two ways: `SetValue` snaps it (bind / load / undo), `AnimateTo` tweens it `From`→`To` for a live change. |
| `IPTkPreviewable` | Paint a non-destructive preview overlay (`ShowPredicted` / `ClearPredicted`) — a health bar stays full and paints the band it *would* lose. |
| `IPTkGhostable` | Render itself as a translucent "ghost" (`MarkAsGhost`) — the look for something a hover what-if predicts but never commits. |

Each capability that animates owns its own completion: `MoveAlongPath`,
`PlayReaction`, and `AnimateTo` all call `Ctx->Complete()` when the motion or tween
settles, so a sequence waits for the visual to finish.

!!! note "Which capability wins for a value change"
    A montage flinch ignores the number; a stat display animates it. A damage
    handler driving a token that implements *both* prefers the stat display's
    `AnimateTo` so the number actually tweens. Previewable and ghostable are the
    preview half of the model — the surfaces exist and share the cue rails, and
    the driver that feeds them is documented in [Hover previews](../abilitysystem/reference-previews.md#hover-previews).

## Presentation channels and the entity widget

An entity is rarely on screen in just one place. The framework calls these surfaces
**presentation channels**: the token is the entity's **primary** channel, and any
number of **auxiliary** surfaces can register for the same entity and receive the
identical cues. Only the primary channel gates playout timing; auxiliaries are
always fire-and-forget.

The shipped building block for an auxiliary surface is `UPTkEntityWidget` — a
`Blueprintable`, `Abstract` readout bound to one entity and one value (a single
`Stat.*` or `Data.*` entry). It implements the token interface and the previewable
capability, self-registers as an auxiliary channel on bind, and surfaces three
paint hooks to Blueprint. You implement only the paint; the registry drives it.

```cpp
class UPTkEntityWidget : public UUserWidget, public IPTkTokenInterface, public IPTkPreviewable
{
    // Bind this readout to (Entity, InFacetTag): deregister the prior binding,
    // register the new one with the registry as an auxiliary channel, and snap.
    // Re-callable — a character-sheet panel calls it as selection changes.
    // Passing an invalid Entity clears the binding.
    void SetBoundEntity(const FPGeEntityRef& Entity, FGameplayTag InFacetTag);

    // Deregister and clear the binding (also runs on teardown).
    void ClearBoundEntity();

    // The entity this readout currently presents (invalid if unbound).
    const FPGeEntityRef& GetBoundEntity() const;

    // --- The paint hooks you implement (BlueprintNativeEvent) ---

    // Snap: the stat facet is now Value — NO animation (bind / load / undo).
    void OnStatDisplayChanged(FGameplayTag Tag, float Value);

    // Snap for a Data.* facet (name, portrait, description) — the typed value.
    void OnTaggedDataDisplayChanged(FGameplayTag Tag, const FInstancedStruct& Value);

    // Preview: the facet would change From->To under the hovered what-if. Paint a
    // non-destructive overlay; keep the baseline.
    void OnPreviewDelta(FGameplayTag Tag, float From, float To);
};
```

The widget's live animation for a real change arrives as a **cue** driving one of
its capabilities — not as a base paint event. The base gives you the snap
(`OnStatDisplayChanged` / `OnTaggedDataDisplayChanged`) and the preview overlay
(`OnPreviewDelta`); an animated beat rides the cue rails like any other.

!!! tip "A panel is a pattern, not a class"
    There is no dedicated panel or character-sheet base class. A long-lived panel — a party frame, a character
    sheet — is a plain container that holds several entity widgets and re-binds
    them to whichever unit is selected. Selection changes are just calls to
    `SetBoundEntity`. And a HUD readout is always an auxiliary channel, never a
    second token: don't try to give one entity two tokens.

## Projected actions

`UPTkProjectedAction` is a queued playout unit that fans a batch of cues across
several tokens at once and reports done when the last of them finishes. It runs on
a [QueueFramework](../queueframework/index.md) channel, so it sequences cleanly
behind and ahead of other queued work.

```cpp
class UPTkProjectedAction : public UQueuedAction
{
    // The grouped changes this action visualizes, and the tokens it drives.
    TArray<FPGeEntityChange> Changes;
    TArray<FPGeEntityRef>    AffectedEntities;

    // Begin a fan-out of N concurrent token visuals; when N report in, the action
    // proceeds. N <= 0 proceeds immediately.
    void BeginParallel(int32 N);

    // Report one fanned-out visual finished.
    void ReportDone();

    // Build a cue context wired to ReportDone, PlayCue it on the token, and return
    // it — one node per token inside the fan-out loop.
    UPTkCueContext* PlayCueOnToken(const FPGeEntityRef& Ref, const FPTkCue& Cue);
};
```

You author one directly only to exercise the sequenced-playout path by hand. The
automatic layer that builds and feeds these from committed changes — grouping a
batch, choosing snap-versus-sequence — is the ability system's work, documented in
a later section.

## The shipped cue handlers

Two example handlers ship registered and ready; they are the global defaults a thin
token inherits for free, and each is a worked model for writing your own.

| Handler | Handles | What it does |
|---|---|---|
| `UPTkDamageCueHandler` | `Cue.TakeDamage` | Plays a reaction on any token that is `IPTkMontageCapable`, passing the cue intent as the reaction tag. A token without the capability no-ops cleanly. A boss overrides it with a signature flinch via its `CueOverrides` map. |
| `UPTkMoveCueHandler` | `Cue.Move` | Walks any token that is `IPTkMovable` along the path in the cue's `FPTkMovePayload`. The same handler drives an actor token and a widget token through the shared capability. |

## Settings

`UPTkTokenSettings` (Project Settings → Plugins → Token System) provides the
fallback token classes used when an entity's template carries no
`Data.Presentation.TokenClass`:

- **`CategoryTokenClasses`** — a priority-ordered list mapping a category tag to a
  token class. The first entry whose category the entity carries wins; array order
  is designer priority.
- **`DefaultTokenClass`** — the last-resort class when neither the template nor a
  category match supplies one. Leave it empty to leave unmatched entities headless.
