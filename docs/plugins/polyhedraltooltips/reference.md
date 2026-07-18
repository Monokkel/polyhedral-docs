# PolyhedralTooltips API Reference

For developers who want the complete public surface of the plugin. Each section
covers one area with clean signatures and short usage notes; for step-by-step
recipes see the [Guides](guides.md), and for how the pieces fit together see the
[overview](index.md).

All public types carry the `PTt` prefix. Signatures below are hand-written to show
the shape of each API; they are illustrative, not source excerpts.

## Core Concepts

Five value types carry a tooltip request through the plugin. Learn these and the
rest of the API reads straightforwardly.

**`FPTtTooltipAnchor` — where the popup goes.** An anchor names a screen region.
Its `Type` picks how that region is computed; you rarely fill the fields by hand,
you build one with an [anchor factory](#function-library).

```cpp
enum class EPTtAnchorType : uint8
{
    Cursor,       // cursor position at show-time
    ScreenPoint,  // a viewport-local point (DPI-scaled)
    ScreenRect,   // a viewport-local rect; popup placed below-right with edge-flip
    WorldPoint,   // a 3D world position projected to screen
    Actor,        // an actor's location projected to screen at show-time
    SlateWidget,  // INTERNAL: the rich-text keyword path — no Blueprint factory
};
```

!!! note "Position is fixed at show-time"
    An anchor is evaluated once when the popup appears; there is no continuous
    tracking in this version, so a world-point or actor tooltip does not follow a
    moving target. The `SlateWidget` type exists for the decorator's own keyword
    anchoring and has **no** `MakeAnchor_*` factory — consumer code uses only
    Cursor / ScreenPoint / ScreenRect / WorldPoint / Actor.

**`FPTtTooltipSourceKey` — dedup identity.** An optional caller-defined key. Two
`ShowTooltip` calls with matching keys *refresh the same popup* instead of
stacking a second one; a fully empty key disables dedup (every call spawns a new
popup). Build one with a [key factory](#function-library).

```cpp
struct FPTtTooltipSourceKey
{
    TWeakObjectPtr<UObject> ContextObject; // e.g. the hovered widget
    FName                   SubId;          // optional sub-identity within it
    bool IsEmpty() const;
    bool Matches(const FPTtTooltipSourceKey& Other) const;
};
```

**`FPTtTooltipSource` — the whole request.** Everything the subsystem needs to
place and dedup a popup.

```cpp
struct FPTtTooltipSource
{
    FPTtTooltipAnchor              Anchor;                 // where to place it
    FPTtTooltipSourceKey           Key;                    // dedup identity (optional)
    FPTtTooltipHandle              ParentHandle;           // parent, for nested popups
    TWeakObjectPtr<APlayerController> OwningPlayerController;
};
```

**`FPTtResolveResult` — the resolved payload.** What a resolver produces and what
`ShowTooltip` consumes: which widget to spawn and what data to hand it.

```cpp
struct FPTtResolveResult
{
    TSubclassOf<UPTtTooltipWidget> WidgetClass;   // null -> settings DefaultTooltipWidget
    TObjectPtr<UObject>            PayloadObject;  // a live object payload (may be null)
    FInstancedStruct               PayloadStruct;  // a static struct payload
    FName                          StyleName;      // keyword text style; None = parent default
};
```

**`FPTtResolveContext` — where the lookup happens.** Passed *into* a resolver so
it can disambiguate (per-player tooltips, nesting).

```cpp
struct FPTtResolveContext
{
    TObjectPtr<URichTextBlock>     OwningRichTextBlock;  // null for imperative calls
    TObjectPtr<APlayerController>  OwningPlayerController;
    FString                        RawRunText;           // the markup run's inner text
    int32                          NestingDepth = 0;     // 0 = top-level; use to prevent cycles
};
```

**`FPTtTooltipHandle` — the popup you spawned.** An opaque id returned by
`ShowTooltip`, used to hide, keep-alive, or nest.

```cpp
struct FPTtTooltipHandle { int32 PopupId = -1; bool IsValid() const; };
```

## Function Library

`UPTtTooltipFunctionLibrary` is the Blueprint front door and the class most game
code touches. Every function is static.

### Show / hide

```cpp
// Show a popup from an already-resolved result. Returns the new (or, on a dedup
// match, refreshed) popup's handle. MaxLifetimeSeconds <= 0 => no auto-dismiss.
static FPTtTooltipHandle ShowTooltip(const FPTtTooltipSource& Source,
                                     const FPTtResolveResult& Result,
                                     float MaxLifetimeSeconds = 0.f);

// Resolve TagName+Key through the registry, then show. Convenience wrapper around
// Resolve + ShowTooltip for content that lives behind a registered resolver.
static FPTtTooltipHandle ShowTooltipFromTag(const FPTtTooltipSource& Source,
                                            FName TagName, FName Key,
                                            float MaxLifetimeSeconds = 0.f);

// Polite hide: marks the source un-hovered so the existing grace timer dismisses
// it shortly after (lets the cursor cross into a nested tooltip first).
static void HideTooltip(FPTtTooltipHandle Handle);

// Reset the lifetime deadline on a popup shown with MaxLifetimeSeconds > 0.
// No-op on a popup that has no lifetime timer.
static void KeepTooltipAlive(FPTtTooltipHandle Handle);
```

### Anchor factories

Build an `FPTtTooltipAnchor` for a source. One per consumer anchor type:

```cpp
static FPTtTooltipAnchor MakeAnchor_Cursor();
static FPTtTooltipAnchor MakeAnchor_ScreenPoint(FVector2D Point);
static FPTtTooltipAnchor MakeAnchor_ScreenRect(FVector2D Min, FVector2D Max);
static FPTtTooltipAnchor MakeAnchor_WorldPoint(FVector World);
static FPTtTooltipAnchor MakeAnchor_Actor(AActor* Actor);
```

### Source-key factories

Build an `FPTtTooltipSourceKey` for dedup. Give repeated hovers of one source the
same key so they refresh a single popup:

```cpp
static FPTtTooltipSourceKey MakeKey_Object(UObject* Object);          // key on a whole object
static FPTtTooltipSourceKey MakeKey_Sub(UObject* Context, FName SubId); // an object + sub-id
static FPTtTooltipSourceKey MakeKey_Named(FName GlobalId);            // a global named key
```

### Resolver registration

Convenience wrappers over the subsystem's registry, for wiring resolvers at
runtime instead of (or on top of) the settings list:

```cpp
static UPTtTooltipSubsystem* GetTooltipSubsystem();
static void RegisterResolver(UPTtTooltipResolver* Resolver);
static void UnregisterResolver(UPTtTooltipResolver* Resolver);

// Construct a resolver of ResolverClass, assign it TagName, and register it.
static UPTtTooltipResolver* CreateAndRegisterResolver(
    TSubclassOf<UPTtTooltipResolver> ResolverClass, FName TagName);
```

## Resolvers

`UPTtTooltipResolver` is the abstract base that maps a markup tag to a payload.
Subclass it (in C++ or Blueprint) and override one method.

```cpp
// UCLASS(Abstract, Blueprintable)
class UPTtTooltipResolver : public UObject
{
    // The markup tag this resolver claims (e.g. "tt", "tt_da"). Set in class
    // defaults. EditDefaultsOnly / BlueprintReadOnly.
    FName TagName;

    // Look up a tooltip for Key. Return false to fall through (the keyword renders
    // as plain text). BlueprintNativeEvent — override ResolveTooltip_Implementation
    // in C++. Keep it cheap: it runs at decorate time for every keyword in a block;
    // defer heavy work (async streaming) into the tooltip widget.
    bool ResolveTooltip(FName Key, const FPTtResolveContext& Context,
                        FPTtResolveResult& OutResult);
};
```

!!! warning "The tag is an `FName`, and only `tt` / `tt_*` reach the decorator"
    A resolver's `TagName` is a plain markup name, **not** an `FGameplayTag` —
    there is no schema or type enforcement. But rich-text runs only ever route to
    tags in the decorator's namespace: `tt` or `tt_*`
    (`UPTtTooltipSubsystem::IsDecoratorReachableTagName` is the check). A resolver
    registered under any other name registers fine and works through a direct
    `ResolveTooltip` / `ShowTooltipFromTag` call, but the decorator will **never**
    claim its keywords — `RegisterResolver` only *warns*, it doesn't block. Keep
    inline-tooltip resolvers on the `tt` prefix.

The three built-in resolvers:

| Resolver | Default tag | Concrete? | How it resolves |
|---|---|---|---|
| `UPTtTooltipResolver_DataTable` | `tt` | **Yes** — drop into `ActiveResolvers` | Looks `Key` up as a row name in the settings' `DefaultTooltipTable`, packs the `FPTtTooltipRow` into `PayloadStruct`. |
| `UPTtTooltipResolver_DataAsset` | `tt_da` | **Yes** — drop into `ActiveResolvers` | Resolves `Key` to a primary asset of the settings' `TooltipPrimaryAssetType` through the Asset Manager. |
| `UPTtTooltipResolver_Object` | `tt_ob` | **No — `Abstract`** | Maps `Key` to a live `UObject` via `ResolveObject`, then queries `IPTtTooltipObjectInterface` on it. |

```cpp
// The object resolver's one extra hook. BlueprintNativeEvent — override
// ResolveObject_Implementation. Must be subclassed (the class is Abstract) to map
// a key to a live object; the base cannot know your project's addressing.
UObject* UPTtTooltipResolver_Object::ResolveObject(
    FName Key, const FPTtResolveContext& Context);
```

!!! note "Abstract vs. concrete"
    `_DataTable` and `_DataAsset` are ready to use — add them to `ActiveResolvers`
    and point settings at a table / asset type. `_Object` is `Abstract`: it does
    nothing until you subclass it and supply `ResolveObject`. Adding the abstract
    class itself to `ActiveResolvers` gives you a resolver that resolves nothing.

## The Object Interface

`IPTtTooltipObjectInterface` is what a live object implements so the object
resolver can ask it to describe its own tooltip.

```cpp
// UINTERFACE(BlueprintType). Implement on any UObject/AActor.
class IPTtTooltipObjectInterface
{
    // Fill OutResult (widget class + payload) for this object's tooltip. Return
    // true to show it, false to suppress. BlueprintNativeEvent + BlueprintCallable
    // — override GetTooltipPayload_Implementation in C++; a native fallback exists.
    bool GetTooltipPayload(FPTtResolveResult& OutResult);
};
```

A common pattern is to return `this` as `OutResult.PayloadObject` so the tooltip
widget can read the object's live fields directly. See
[Back a tooltip with a live object](guides.md#back-a-tooltip-with-a-live-object).

## Tooltip Widgets

`UPTtTooltipWidget` is the UMG base every displayed tooltip subclasses. It is
`Abstract` and `Blueprintable` — you author the layout in a Widget Blueprint and
implement one event.

```cpp
// UCLASS(Abstract, Blueprintable) : public UUserWidget
class UPTtTooltipWidget : public UUserWidget
{
    // Called once after the widget is created and its payload is known. This is a
    // BlueprintImplementableEvent with NO native default — you MUST implement it,
    // or the payload goes nowhere. Bind your UMG controls to the payload here.
    void ReceiveTooltipPayload(UObject* PayloadObject, const FInstancedStruct& PayloadStruct);

    // C++ entry that fires the event above. Public so the keyword path can drive it.
    void InitFromPayload(UObject* PayloadObject, const FInstancedStruct& PayloadStruct);

    // This widget's own hosting-popup handle. Use as a source ParentHandle when you
    // spawn a nested tooltip from inside this widget. BlueprintPure.
    FPTtTooltipHandle GetTooltipHandle() const;
};
```

!!! warning "Implement `ReceiveTooltipPayload` on every tooltip widget"
    Unlike the resolver methods, `ReceiveTooltipPayload` has no working native
    fallback — a subclass that omits it shows an empty popup. The handle is set
    for you: the subsystem calls a private `SetTooltipHandle` immediately after
    `CreateWidget` (it is deliberately **not** Blueprint-callable), so you only
    ever *read* the handle via `GetTooltipHandle`, never set it.

## Rich-Text Integration

`UPTtTooltipDecorator` is the bridge between authored markup and the resolver
registry. It is a `URichTextBlockDecorator`; you don't call it, you add it to a
block.

- **Wiring:** add `UPTtTooltipDecorator` to the `URichTextBlock`'s
  **Decorator Classes** list in the UMG designer.
- **Markup:** write a run tagged with a resolver's tag; the run's inner text is
  the lookup **key** and the visible keyword:

```text
Deal <tt>Fireball</> damage, or read the <tt_da>KeywordsAsset</> glossary.
```

The decorator only claims runs whose tag is both **registered** and in the
reachable `tt` / `tt_*` namespace; every other run (a style tag, an inline image)
passes through untouched. `StyleName` on the resolve result controls how the
keyword text itself renders in the block's style set.

## The Subsystem

`UPTtTooltipSubsystem` is a `UEngineSubsystem` — one per editor/runtime process,
**not** per game instance — so its resolver registry and popup stack are live at
editor time for rich-text preview. It owns the resolver map and the live popups
(hover state, dedup, nesting, lifetimes).

```cpp
static UPTtTooltipSubsystem* Get();

// --- Registry ---
bool          ResolveTooltip(FName TagName, FName Key,
                             const FPTtResolveContext& Context,
                             FPTtResolveResult& OutResult) const;
bool          IsTagRegistered(FName TagName) const;
static bool   IsDecoratorReachableTagName(FName TagName);  // true for "tt" / "tt_*"
TArray<FName> GetRegisteredTagNames() const;
void          RegisterResolver(UPTtTooltipResolver* Resolver);
void          UnregisterResolver(UPTtTooltipResolver* Resolver);
void          ReloadResolversFromSettings();               // after editing settings at runtime

// --- Popup stack ---
void HideTooltip(FPTtTooltipHandle Handle);      // polite; grace timer dismisses
void KeepTooltipAlive(FPTtTooltipHandle Handle); // reset lifetime deadline
void DismissPopup(int32 PopupId);                // immediate; cascades to descendants
void DismissAllPopups();                         // immediate; clears the whole stack
```

!!! warning "Show a popup through the function library, not the subsystem"
    The subsystem's `ShowPopup` (and its `IsPopupAlive`, `SetSourceHovered`,
    `NotifyPopupHoverState` companions) are **plain C++ methods, not
    `UFUNCTION`s** — they are the internal machinery the keyword path and the
    library call into, and are not Blueprint-callable. To show a tooltip from
    Blueprint or game code, always go through
    `UPTtTooltipFunctionLibrary::ShowTooltip` / `ShowTooltipFromTag`, which wrap
    `ShowPopup` for you.

!!! note "Hide vs. dismiss"
    `HideTooltip` is *polite*: it marks the source un-hovered and lets the
    `DismissGraceSeconds` timer remove the popup, so the cursor can travel into a
    nested tooltip without it vanishing. `DismissPopup` / `DismissAllPopups` are
    *immediate* — no grace — and `DismissPopup` cascades to any nested popups
    parented under it.

## Settings

`UPTtTooltipSettings` (Project Settings → Plugins → Polyhedral Tooltips) is a
config-backed developer-settings page.

| Setting | Default | What it does |
|---|---|---|
| `ActiveResolvers` | *(empty)* | Resolver classes instantiated at startup. Each declares the markup tag it claims. |
| `DefaultTooltipTable` | *(none)* | The DataTable the `tt` resolver reads. The asset picker is filtered to tables whose row struct is `FPTtTooltipRow`. |
| `TooltipPrimaryAssetType` | `PTtTooltipDataAsset` | The primary-asset type the `tt_da` resolver scans. Must match your data asset's `AssetType`. |
| `DefaultTooltipWidget` | *(none)* | Fallback tooltip widget when a resolve result names no `WidgetClass`. |
| `TagConflictPolicy` | `ErrorAtStartup` | What happens when two resolvers claim the same tag (see below). |
| `HoverDelaySeconds` | `0.5` | Delay after hover begins before a popup appears. |
| `DismissGraceSeconds` | `0.15` | Grace after losing hover before dismissal — the window to cross into a nested tooltip. |
| `bWarnOnUnresolvedTags` | `true` | Log a warning when a tooltip tag has no registered resolver. |

```cpp
enum class EPTtTagConflictPolicy : uint8
{
    ErrorAtStartup, // log an error and ignore later resolvers for a claimed tag
    FirstWins,      // first registration for a tag wins; later ones ignored silently
    LastWins,       // last registration wins; earlier ones dropped silently
};
```

!!! warning "The DataAsset path needs an Asset Manager entry too"
    For `tt_da` tooltips to resolve, the data asset's `AssetType` and settings'
    `TooltipPrimaryAssetType` must match — **and** that primary asset type must
    also be registered for scanning in **Project Settings → Asset Manager →
    Primary Asset Types to Scan**. That last step lives entirely outside this
    plugin's settings and is the usual reason a `tt_da` lookup finds nothing.
    Likewise, `DefaultTooltipTable`'s row-struct filter is editor-only — a table
    with the wrong row struct isn't caught until resolve time.

## Content Authoring Types

Two shipped containers for static content. Both feed the same tooltip widget via
the resolve result.

**`FPTtTooltipRow`** — a row struct (`FTableRowBase`) for the DataTable resolver.
Author one project-wide table of simple, uniform tooltips; the row name is the
lookup key.

```cpp
struct FPTtTooltipRow : public FTableRowBase
{
    FText                          Title;
    FText                          Body;                // multi-line
    TSoftClassPtr<UPTtTooltipWidget> WidgetClassOverride; // null => settings default
    FName                          StyleName;           // keyword run style
};
```

**`UPTtTooltipDataAsset`** — a `UPrimaryDataAsset` for the DataAsset resolver.
Subclass per content category (spells, items, mechanics) to add your own fields
without touching the resolver.

```cpp
// UCLASS(Blueprintable) : public UPrimaryDataAsset
class UPTtTooltipDataAsset : public UPrimaryDataAsset
{
    FText                          Title;
    FText                          Body;                // multi-line
    TSoftClassPtr<UPTtTooltipWidget> WidgetClassOverride;
    FName                          StyleName;
    FName                          AssetType;           // must match TooltipPrimaryAssetType
};
```
