# PolyhedralTooltips

For developers who want rich, hover-driven tooltips — keyword popups inside a
block of prose, and imperative tooltips anchored to a widget, the cursor, or a
unit on the board — without hand-wiring a popup widget, a hover timer, and a
lookup table every time. After this section you'll know the two ways to raise a
tooltip, the three ways to author its content, and how one small display widget
renders whatever a lookup returns.

PolyhedralTooltips is the framework's **tooltip and display-data plumbing**. It
turns a piece of markup or a single function call into a positioned, hover-aware
popup: you name a piece of content, a **resolver** looks it up, and the plugin
spawns your **tooltip widget** to render it — handling the hover delay, the
dismiss grace period, dedup, and nested-popup cascades for you. Content lives in
data (a DataTable or per-entry assets) or on a live object, so writers add a
tooltip without touching code.

## Two ways to raise a tooltip

Everything funnels through one engine subsystem, but you reach it from two very
different places:

- **Inline, inside rich text.** Add a **decorator** to a `URichTextBlock` and
  write markup like `<tt_da>Fireball</>` in your text. On hover over the keyword,
  the plugin resolves `Fireball` through the resolver registered for the `tt_da`
  tag and shows its popup. This is how a rules paragraph gets live, hoverable
  keywords with no per-keyword code.
- **Imperatively, from code.** Call
  [`ShowTooltip` / `ShowTooltipFromTag`](reference.md#function-library) with an
  **anchor** — the cursor, a screen point or rect, a world point, or an actor —
  from a widget's `OnHovered`, a button, or any gameplay event. This is how a
  hovered inventory slot or a unit on the board raises its own tooltip.

Both paths do the same three things: pick a piece of content, resolve it to a
widget-plus-payload, and hand it to the subsystem to display.

## Where it fits in the framework

PolyhedralTooltips is a **standalone UI plugin** — it depends on nothing else in
the framework, only on Unreal's UMG `RichTextBlock` and (for the DataAsset path)
the Asset Manager. You can drop it into any project.

The registry is a `UPTtTooltipSubsystem`, an **`EngineSubsystem`** — deliberately
*not* a game-instance subsystem — so the resolver registry and popup machinery
are live at **editor time**, which is what lets a `RichTextBlock` preview its
tooltip keywords in the UMG designer without entering Play. Nothing here needs
PIE running.

!!! note "Backing a tooltip with a game entity"
    Tooltips describe *anything*, but a common case is "show this unit's or
    ability's live numbers." That's the job of the
    [object interface](reference.md#the-object-interface): implement
    `IPTtTooltipObjectInterface` on the object that owns the data and let the
    object resolver fetch it on hover. The plugin never reaches into the entity
    system itself — you hand it a payload.

## The three ways to author content

A resolver is the bridge from a markup tag to a payload. Three are shipped, and
you pick per content category — you can register all three side by side under
their default tags:

| Author content as | Resolver | Default tag | Reach for it when |
|---|---|---|---|
| Rows in one shared **DataTable** (`FPTtTooltipRow`) | `UPTtTooltipResolver_DataTable` | `tt` | You have many simple, uniform tooltips — a glossary of keywords with a title and body. |
| Per-entry **DataAssets** (`UPTtTooltipDataAsset`) | `UPTtTooltipResolver_DataAsset` | `tt_da` | Tooltips are heterogeneous — spells, items, and mechanics each want their own fields; subclass the asset per category. |
| A **live object** implementing `IPTtTooltipObjectInterface` | `UPTtTooltipResolver_Object` | `tt_ob` | The content is dynamic — a unit's current HP, a stacking buff — and must be read off a live object at hover time. |

Whatever the resolver returns, one designer-authored `UPTtTooltipWidget` subclass
renders it. That widget is the single piece of display code you always write.

## The pieces at a glance

| Piece | What it is |
|---|---|
| `UPTtTooltipSubsystem` | The **registry** — an engine subsystem owning the resolver map and the live popup stack (hover, dedup, nesting, lifetime). Live in-editor. |
| `UPTtTooltipFunctionLibrary` | The **Blueprint front door** — show/hide, keep-alive, and the anchor / source-key factories. The one class most game code touches. |
| `UPTtTooltipDecorator` | The **rich-text integration** — add it to a `URichTextBlock`'s decorator list to make `<tt>` / `<tt_*>` keywords hoverable. |
| `UPTtTooltipResolver` (+ `_DataTable`, `_DataAsset`, `_Object`) | The **lookup** — maps a markup tag + key to a payload. Three built-ins; subclass for your own. |
| `IPTtTooltipObjectInterface` | The interface a **live object** implements to describe its own tooltip. |
| `UPTtTooltipWidget` | The **display** — a UMG widget base you subclass to paint the payload. |
| `FPTtTooltipRow` / `UPTtTooltipDataAsset` | The two **content containers** — a DataTable row and a per-entry primary asset. |
| `UPTtTooltipSettings` | Project settings: the active resolvers, the default table and widget, hover/dismiss timing, and the tag-conflict policy. |

## Where to go next

- **[Guides](guides.md)** — task recipes: authoring an inline tooltip end to end,
  showing a tooltip from code with an anchor, and backing one with a live object.
- **[API Reference](reference.md)** — the full public surface, grouped by area,
  with hand-written signatures and short examples.
- **[Plugins overview](../index.md)** — the rest of the framework's plugins.
