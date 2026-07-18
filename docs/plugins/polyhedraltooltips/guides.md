# PolyhedralTooltips Guides

For developers with the plugin enabled who want to get a tooltip on screen. Each
section is a short, followable recipe. For the full type-by-type surface see the
[API Reference](reference.md); for how the pieces fit together see the
[overview](index.md).

## Author an inline tooltip

The end-to-end path for a hoverable keyword inside a block of prose: enable a
resolver, add the decorator, write a row of content, and give the plugin a widget
to render it. This recipe uses the built-in **DataTable** resolver (tag `tt`) and
its `FPTtTooltipRow`, the simplest content source.

### 1. Enable a resolver in settings

Open **Project Settings → Plugins → Polyhedral Tooltips** and add the resolver
you want to the **Active Resolvers** list. Add **Tooltip Resolver (Data Table)**
for this recipe. The subsystem instantiates every entry at startup, and each
resolver declares the markup tag it claims (`tt` for the DataTable resolver).

!!! tip "The tag is a markup name, not a gameplay tag"
    A resolver's tag is a plain `FName` — `tt`, `tt_da`, `tt_ob`, or your own —
    **not** an `FGameplayTag`. Only tags in the decorator's namespace (`tt` or
    `tt_*`) are reachable from rich text; a resolver registered under any other
    name still works through a direct
    [`ResolveTooltip`](reference.md#the-subsystem) call, but the decorator will
    never route a keyword to it. See
    [the namespace rule](reference.md#resolvers).

### 2. Add the decorator to your RichTextBlock

Select the `URichTextBlock` in the UMG designer and add **PTt Tooltip Decorator**
to its **Decorator Classes** list. That single line is what makes `<tt>` and
`<tt_*>` runs in the block's text hoverable — the decorator claims any such run
for which a resolver is registered and leaves every other run (style tags, inline
images) untouched.

### 3. Author a DataTable row

Create a `UDataTable` whose row structure is **PTt Tooltip Row** (`FPTtTooltipRow`),
then point **Default Tooltip Table** in settings at it. Add a row — the **row name
is the lookup key**:

| Field | Example |
|---|---|
| *Row Name* | `Fireball` |
| **Title** | `Fireball` |
| **Body** | `Deals 3 fire damage to a target and its neighbours.` |
| **Widget Class Override** | *(leave empty to use the settings default)* |
| **Style Name** | *(optional — the text style for the keyword in the block)* |

Now write the markup in your block's text. The run's inner text is both the
lookup key and the visible keyword:

```text
Cast <tt>Fireball</> to clear the back row.
```

On hover over "Fireball", the DataTable resolver looks up the `Fireball` row and
hands its title and body to your tooltip widget.

### 4. Author the tooltip widget

The plugin spawns a widget to *display* the payload, and you must supply one.
Create a UMG Widget Blueprint whose parent class is **PTt Tooltip Widget**
(`UPTtTooltipWidget`), lay out a title text and a body text, and set it as
**Default Tooltip Widget** in settings (or per-row via *Widget Class Override*).

The one required step: implement **Receive Tooltip Payload**.

=== "Blueprint"
    1. In the widget graph, override the **Receive Tooltip Payload** event.
    2. It gives you a **Payload Object** and a **Payload Struct**. The DataTable
       resolver packs the row into the struct, so **Get** the `FPTtTooltipRow`
       out of **Payload Struct** and bind its **Title** / **Body** onto your text
       blocks.
    3. Compile. On hover the plugin creates the widget, fills the payload, and
       calls this event once.

=== "C++"
    ```cpp
    // A tooltip widget for FPTtTooltipRow payloads. ReceiveTooltipPayload is a
    // BlueprintImplementableEvent with NO native body — implement it in the BP
    // subclass (above) or, in a C++ widget, do the binding in InitFromPayload's
    // caller path. Sketch of reading the struct payload:
    void UMyTooltipWidget::ApplyRow(const FInstancedStruct& PayloadStruct)
    {
        if (const FPTtTooltipRow* Row = PayloadStruct.GetPtr<FPTtTooltipRow>())
        {
            TitleText->SetText(Row->Title);
            BodyText->SetText(Row->Body);
        }
    }
    ```

!!! warning "Every tooltip widget must implement Receive Tooltip Payload"
    `ReceiveTooltipPayload` is a `BlueprintImplementableEvent` with **no** native
    default. If your widget subclass doesn't implement it, the resolved payload
    arrives and goes nowhere — the popup shows up blank. This is the one hook you
    can never skip. (Contrast the resolver methods, which are
    `BlueprintNativeEvent`s that ship a working native fallback.)

### 5. Verify on hover

Because the subsystem is an engine subsystem, you can verify without entering
Play: the `RichTextBlock` previews its tooltip keywords in the designer. Hover
the keyword, watch for the popup after the **Hover Delay** (0.5 s by default),
and move off it to see it dismiss after the **Dismiss Grace** window. If a
keyword renders as plain text, its tag has no registered resolver — check
**Active Resolvers** and, if `bWarnOnUnresolvedTags` is on, the log
(`LogPTtTooltips`).

## Show a tooltip from code

For a tooltip that isn't inside prose — a hovered inventory slot, a portrait, a
unit on the board — call the function library directly with an **anchor** that
says where to place the popup. You can either resolve through a registered tag
(reusing your DataTable/DataAsset content) or pass a payload you built yourself.

=== "Blueprint"
    1. On the source widget's **On Hovered**, build a **Tooltip Source**: set its
       **Anchor** from an anchor factory — **Make Anchor / Cursor** is the
       simplest; **Make Anchor / Actor** or **/ Screen Rect** pin it to a thing.
    2. *(Optional)* Give the source a **Key** from **Make Key / Object** (pass
       the slot widget) so repeated hovers **refresh the same popup** instead of
       stacking new ones.
    3. Call **Show Tooltip From Tag** with the source, a **Tag Name** (`tt` /
       `tt_da`), and a **Key** (the row or asset name). Store the returned
       **Tooltip Handle**.
    4. On **On Unhovered**, call **Hide Tooltip** with that handle — a polite
       hide that lets the grace timer dismiss it (so the cursor can travel into a
       nested tooltip first).

=== "C++"
    ```cpp
    // Raise a tooltip anchored to the cursor, resolving through the "tt" table.
    FPTtTooltipSource Source;
    Source.Anchor = UPTtTooltipFunctionLibrary::MakeAnchor_Cursor();
    Source.Key    = UPTtTooltipFunctionLibrary::MakeKey_Object(HoveredSlot); // dedup
    Source.OwningPlayerController = GetOwningPlayer();

    FPTtTooltipHandle Handle = UPTtTooltipFunctionLibrary::ShowTooltipFromTag(
        Source, /*TagName=*/ TEXT("tt"), /*Key=*/ TEXT("Fireball"));

    // Later, on un-hover — polite hide, grace timer dismisses it:
    UPTtTooltipFunctionLibrary::HideTooltip(Handle);
    ```

!!! tip "Pass a pre-built payload when there's no registered tag"
    If the content isn't in a table or asset, skip resolution: fill an
    [`FPTtResolveResult`](reference.md#core-concepts) (a widget class plus a
    payload object or struct) and call **Show Tooltip** with it directly.
    `ShowTooltipFromTag` is just *resolve-then-show* for content that already
    lives behind a registered tag.

!!! note "Auto-dismiss vs. sticky"
    Both show calls take an optional **Max Lifetime Seconds**. Leave it at `0`
    (the default) and the popup lives until you `HideTooltip` / dismiss it —
    right for hover tooltips. Pass a positive value for a self-expiring toast, and
    call **Keep Tooltip Alive** to reset its deadline.

## Back a tooltip with a live object

When the content is dynamic — a unit's current HP, a buff's remaining stacks —
read it off a live object at hover time instead of baking it into a table. Two
steps: implement the interface on the object, and subclass the abstract object
resolver to map a key to that object.

### 1. Implement the object interface

On the class that owns the data (a unit actor, an ability object, a UMG item
widget), add **`IPTtTooltipObjectInterface`** and implement **Get Tooltip
Payload**.

=== "Blueprint"
    1. Add the **PTt Tooltip Object Interface** to the actor/object's
       **Implemented Interfaces**.
    2. Implement **Get Tooltip Payload**: fill the **Out Result** (its widget
       class and a payload object/struct — often just `Self` as the payload
       object so the widget can read live fields), and **return true** to show the
       tooltip or **false** to suppress it this frame.

=== "C++"
    ```cpp
    bool AMyUnit::GetTooltipPayload_Implementation(FPTtResolveResult& OutResult)
    {
        if (bIsDead) return false;             // suppress
        OutResult.WidgetClass   = UnitTooltipWidgetClass;
        OutResult.PayloadObject = this;        // widget reads live HP off us
        return true;
    }
    ```

### 2. Subclass the object resolver

`UPTtTooltipResolver_Object` is **abstract** — it knows how to query the interface
but not how to turn a key into an object. Subclass it and override **Resolve
Object** to do that mapping (a registry lookup, a name→actor search, etc.). It
keeps the default `tt_ob` tag.

=== "Blueprint"
    1. Create a Blueprint subclass of **Tooltip Resolver (Object)**.
    2. Override **Resolve Object**: given a **Key** and the **Context**, return
       the live `UObject` that key names (e.g. find the unit with that id). The
       resolver then calls **Get Tooltip Payload** on whatever you return.
    3. Add your subclass to **Active Resolvers** in settings.

=== "C++"
    ```cpp
    UObject* UMyUnitResolver::ResolveObject_Implementation(
        FName Key, const FPTtResolveContext& Context)
    {
        // Map the markup key to a live object however your project addresses them.
        return UMyUnitRegistry::Get(Context.OwningPlayerController)->FindUnit(Key);
    }
    ```

Now `<tt_ob>Goblin_07</>` in rich text — or a `ShowTooltipFromTag(Source,
"tt_ob", "Goblin_07")` from code — resolves to the live unit and renders its
current state.

!!! warning "Resolve Object and Resolve Tooltip override the `_Implementation`"
    `ResolveObject` and `ResolveTooltip` are `BlueprintNativeEvent`s. In C++ you
    override the `_Implementation`-suffixed method, not the bare virtual — and a
    native fallback already exists, so a Blueprint override is optional for the
    two built-in resolvers but **required** for the abstract object resolver.
