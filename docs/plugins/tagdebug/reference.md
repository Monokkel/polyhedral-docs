# TagDebug API Reference

For developers who want the complete public surface of the plugin. For the model and
when to use it, see the [overview](index.md); the dispatch core it rides on is
[TagEvents](../tagevents/index.md).

Signatures below are hand-written to show the shape of each API; they are illustrative,
not source excerpts.

## Tag schema

A debug tag names one debug option. Its segments carry meaning:

```
Debug.[Type].[Category?].[Option]
```

- **Type** — the second segment, one of `Bool`, `Int`, `Float`, or `Vector`. It fixes
  the kind of value the option stores. If the second segment is none of those, the tag
  is a **typeless action** — a single-shot trigger with no stored value.
- **Category** — an optional middle segment, purely for grouping options in menus and
  pickers. It has no effect on the value.
- **Option** — the final segment, the name of the option itself.

| Tag | Kind | Meaning |
|---|---|---|
| `Debug.Bool.ShowHiddenObjects` | Bool | a boolean flag (no category) |
| `Debug.Float.GameFeel.MoveSpeed` | Float | a tuning number under category *GameFeel* |
| `Debug.Int.Combat.ForcedDamage` | Int | an integer under category *Combat* |
| `Debug.Vector.Camera.Offset` | Vector | a vector under category *Camera* |
| `Debug.Network.KickPlayer` | Action | a typeless single-shot action |

Each read/write function is filtered to its own `Debug.<Type>` namespace, so the tag
picker on a **Set Debug Float Value** node offers only `Debug.Float.*` tags — you can't
wire a Bool tag into a Float setter by mistake.

## Reading & checking values

`UPDbTagDebugFunctionLibrary` is a Blueprint function library — every entry is static
and callable from Blueprint and C++. The getters are pure (no exec pin); pass a world
context object (self, in Blueprint).

```cpp
// Return the value currently stored for a typed debug tag. If nothing has been set,
// returns the type's default: false / 0 / 0.0 / zero vector.
static bool    CheckDebugBoolValue  (const UObject* WorldContextObject, FGameplayTag Tag);
static int32   CheckDebugIntValue   (const UObject* WorldContextObject, FGameplayTag Tag);
static double  CheckDebugFloatValue (const UObject* WorldContextObject, FGameplayTag Tag);
static FVector CheckDebugVectorValue(const UObject* WorldContextObject, FGameplayTag Tag);

// True only if a value has been explicitly set for this tag — distinguishes
// "set to false / 0" from "never set".
static bool IsDebugValueSet(const UObject* WorldContextObject, FGameplayTag Tag);

// Parse a Debug.* tag into its type, category, and option name (see Types below).
static FPDbDebugTagInfo ParseDebugTag(FGameplayTag Tag);
```

=== "Blueprint"
    Drop **Check Debug Bool Value**, pick a `Debug.Bool.*` tag from the filtered picker,
    and branch on the boolean result. Because the node is pure it has no exec pins —
    read it wherever you'd read a variable. `ParseDebugTag` is the Blueprint way to
    inspect a tag's parts.

=== "C++"
    ```cpp
    // Gate a debug-only code path on a boolean tag.
    if (UPDbTagDebugFunctionLibrary::CheckDebugBoolValue(this, ShowHiddenTag))
    {
        RevealHiddenActors();
    }
    ```

!!! note "ParseDebugTag is the Blueprint entry point"
    The struct's own `FPDbDebugTagInfo::ParseTag` (and `ParseTypeFromString`) are plain
    C++ statics with no Blueprint exposure. From Blueprint, always go through
    `ParseDebugTag`.

## Writing values & firing actions

Setters are callable (they have exec pins). Each takes a `bPersistent` flag — covered in
the warning below — and dispatches its change to the debug component after storing it.

```cpp
// Store a value for a typed debug tag, then dispatch the change to the debug component.
// bPersistent = false (default) => session-only; true => also written to config.
static void SetDebugBoolValue  (const UObject* WCO, FGameplayTag Tag, bool Value,    bool bPersistent = false);
static void SetDebugIntValue   (const UObject* WCO, FGameplayTag Tag, int32 Value,   bool bPersistent = false);
static void SetDebugFloatValue (const UObject* WCO, FGameplayTag Tag, double Value,  bool bPersistent = false);
static void SetDebugVectorValue(const UObject* WCO, FGameplayTag Tag, FVector Value, bool bPersistent = false);

// Fire a typeless Debug.* action: no stored value, just a one-shot dispatch.
static void CallDebugAction(const UObject* WCO, FGameplayTag Tag);
```

=== "Blueprint"
    Drop **Set Debug Float Value**, pick a `Debug.Float.*` tag, and wire the value. Leave
    **Persistent** unchecked for a session tweak; check it to write the value through to
    the config file. For a typeless tag, use **Call Debug Action** instead — it takes no
    value.

=== "C++"
    ```cpp
    // Session-only nudge to a tuning number.
    UPDbTagDebugFunctionLibrary::SetDebugFloatValue(this, MoveSpeedTag, 2.0);

    // Persist a flag so it survives the next launch.
    UPDbTagDebugFunctionLibrary::SetDebugBoolValue(this, ShowHiddenTag, true, /*bPersistent=*/true);

    // Fire a one-shot action.
    UPDbTagDebugFunctionLibrary::CallDebugAction(this, KickPlayerTag);
    ```

!!! warning "`bPersistent` defaults to `false` — a plain Set is session-only"
    Every setter's `bPersistent` parameter defaults to `false`. A `Set` without it stores
    the value in memory for the current session only; it is **lost on the next launch**
    and never touches the config file. Pass `true` (check the pin in Blueprint) to write
    the value to `Saved/Config/TagDebugValues.ini`. It's easy to assume all debug sets
    persist — they don't.

## Handling events in Blueprint

Setting a value or firing an action doesn't only store data — it dispatches an event to
a debug component on the Game State, so Blueprints can react the instant a knob moves.
Two pieces make that work:

- **`UPDbDebugComponent`** — an actor component with **no callable API of its own**. Its
  entire purpose is to be *subclassed in Blueprint*; your subclass's event graph is where
  handlers live. The subsystem spawns it lazily on the Game State the first time any
  debug value changes, choosing the subclass named in [Settings](#settings).
- **Debug Tag Event node** — an editor-only Blueprint node you drop into that subclass's
  graph. Bind it to a `Debug.*` tag and it becomes an event entry that fires whenever
  that tag is set or called, exposing the new value on a type-matched output pin.

The node's value output pin is typed from the tag:

| Tag pattern | Output pin |
|---|---|
| `Debug.Bool.*` | `bool` |
| `Debug.Int.*` | `int32` |
| `Debug.Float.*` | `double` (Blueprint **Float**) |
| `Debug.Vector.*` | `FVector` |
| `Debug.*` (typeless) | no value pin — exec only |

To wire a handler:

1. Create a Blueprint subclass of `UPDbDebugComponent`.
2. In its event graph, add a **Debug Tag Event** node and set its **Debug Tag**.
3. Wire the exec output; read the typed value pin (if the tag has one) to drive your
   debug behavior.
4. Point **Project Settings → Plugins → Tag Debug → Debug Component Class** at your
   subclass so the subsystem spawns it.

!!! note "The Debug Tag Event node is a graph node, not a runtime class"
    It lives in the editor-only module and exists only inside a Blueprint graph — there
    is no C++ type to include or call. Its behavior is entirely the pin table above. The
    node itself is a specialization of the shared TagEvents dispatch node.

## Subsystem reference

`UPDbTagDebugSubsystem` is the game-instance subsystem the function library wraps. Its
Blueprint surface mirrors the library (minus the world-context argument); call it
directly when you already hold the subsystem, or use the library for the shorter path.

```cpp
// Getters (pure). Current value = session override > persistent > type default.
bool    GetDebugBoolValue  (FGameplayTag Tag) const;
int32   GetDebugIntValue   (FGameplayTag Tag) const;
double  GetDebugFloatValue (FGameplayTag Tag) const;
FVector GetDebugVectorValue(FGameplayTag Tag) const;

// Setters (callable). Store + dispatch; bPersistent writes through to config.
void SetDebugBoolValue  (FGameplayTag Tag, bool Value,    bool bPersistent = false);
void SetDebugIntValue   (FGameplayTag Tag, int32 Value,   bool bPersistent = false);
void SetDebugFloatValue (FGameplayTag Tag, double Value,  bool bPersistent = false);
void SetDebugVectorValue(FGameplayTag Tag, FVector Value, bool bPersistent = false);

void CallDebugAction(FGameplayTag Tag);
bool IsDebugValueSet(FGameplayTag Tag) const;

// Find, or lazily spawn, the debug component on the current Game State.
UPDbDebugComponent* FindOrCreateDebugComponent();
```

### Persistence tiers

Values live in two layers, and a getter reads them in precedence order:

- **Persistent** — written to `Saved/Config/TagDebugValues.ini`, survives restarts. This
  is where a `Set` with `bPersistent = true` lands. You can hand-edit this file between
  sessions to change values without touching code.
- **Session-only** — held in memory, lost when the session ends. This is where a plain
  `Set` lands.

A session value **overlays** the persistent one: a session `Set` changes what the getter
returns without rewriting the config file, so `session override > persistent > default`.

### Console command

Every debug tag doubles as a console command — type the tag path, optionally followed by
a value:

```
Debug.Bool.ShowHiddenObjects true
Debug.Float.GameFeel.MoveSpeed 2.0
Debug.Network.KickPlayer
```

The format is `Debug.Tag.Path [Value]`: a value drives the matching `Set`; a typeless
tag with no value fires its action. Either way it dispatches the same event a Blueprint
`Set` would. The console glue that parses this text is internal C++ — there is no
Blueprint node for it; author values through the setters above.

## Settings

`UPDbTagDebugSettings` — **Project Settings → Plugins → Tag Debug**.

| Property | Type | What it does |
|---|---|---|
| **Debug Component Class** | `TSoftClassPtr<UPDbDebugComponent>` | The Blueprint subclass spawned on the Game State to host your Debug Tag Event handlers. If unset, the base C++ component is used — which handles nothing. |

This is a config-only setting: it's edited in Project Settings (or the `TagDebug` config
section) and exposes no Blueprint getter. The subsystem resolves it inside
`FindOrCreateDebugComponent`.

## Types

### `EPDbDebugValueType`

The value kind a tag names, derived from its second segment.

```cpp
enum class EPDbDebugValueType : uint8
{
    None,    // displayed as "Action" — a typeless single-shot tag with no value
    Bool,
    Int,
    Float,
    Vector,
};
```

### `FPDbDebugTagInfo`

The parsed form of a `Debug.*` tag, returned by `ParseDebugTag`.

```cpp
struct FPDbDebugTagInfo   // BlueprintType; all fields BlueprintReadOnly
{
    EPDbDebugValueType ValueType;   // Bool / Int / Float / Vector, or None for an action
    FString            Category;    // the optional middle segment, or empty
    FString            OptionName;  // the final segment
    FGameplayTag       Tag;         // the original tag

    bool IsValid() const;           // true when Tag is a valid gameplay tag
};
```

!!! note "Parsing from Blueprint vs C++"
    Build one of these with `UPDbTagDebugFunctionLibrary::ParseDebugTag` from Blueprint.
    The struct's static `ParseTag` / `ParseTypeFromString` helpers are C++-only and not
    Blueprint-reachable.
