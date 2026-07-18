# QueueFramework API Reference

For developers who want the complete public surface of the plugin. For the model and
when to use it, see the [overview](index.md).

Signatures below are hand-written to show the shape of each API; they are illustrative,
not source excerpts.

## The queue subsystem

`UQueueSubsystem` is a world subsystem — one per play world.

```cpp
// Resolve the subsystem for the current play world (nullptr if there is none).
static UQueueSubsystem* Get();

// Queue Item at the END of Channel. If the channel is idle, it starts immediately.
void QueueItem(UQueuedItem* Item, FName Channel);

// Queue Item just AFTER the currently-running item on Channel, so it (and anything it
// nests) resolves before the rest of the queue — depth-first. Multiple nested calls
// from one handler keep their relative order.
void QueueItemNested(UQueuedItem* Item, FName Channel);

// Batch forms: queue several items with processing triggered only once, after all are
// in — use when every item must be present before any starts. (C++.)
void QueueItems(const TArray<UQueuedItem*>& Items, FName Channel);
void QueueItemsNested(const TArray<UQueuedItem*>& Items, FName Channel);

// Set a channel's per-tick budgets, creating the channel if needed.
void ConfigureChannel(FName Channel, FQueueChannelConfig Config);
```

A channel's budgets bound how much it does per frame:

```cpp
struct FQueueChannelConfig
{
    int32 MaxItemsPerTick = 0;   // cap items processed per tick; 0 = unlimited
    float TimeBudgetMs    = 0.f; // cap milliseconds spent per tick;  0 = unlimited
};
```

## Queued items

Derive from `UQueuedItem` (Blueprint or C++) and override one execution hook.

```cpp
class UQueuedItem : public UObject   // Blueprintable, EditInlineNew, DefaultToInstanced
{
    // Called once after construction, after spawn properties are set. Initialization.
    void Setup();                       // BlueprintImplementableEvent

    // Synchronous body: override for instant work. The queue proceeds automatically
    // when it returns.
    void RunOnQueue();                  // BlueprintImplementableEvent

    // Asynchronous body: override for work that spans time. The C++ default just runs
    // RunOnQueue() then Proceed(); if you override it, YOU must call Proceed when done.
    void RunOnQueueAsync();             // BlueprintNativeEvent

    // Signal that this item has finished, releasing the channel to the next item.
    // Automatic for sync items; manual for async ones.
    void Proceed();

    // A human-readable label for debugging.
    FString DescribeItem() const;       // BlueprintNativeEvent, pure

    // Fires when this item proceeds. Bind it to react to one item finishing.
    FOnQueuedItemProceed OnProceed;     // BlueprintAssignable (dynamic multicast)

    // The channel this item is queued on (set when queued).
    FName Channel;                      // read-only
};
```

!!! warning "Async items must call Proceed exactly once"
    If you override **Run On Queue Async** and never call **Proceed**, the channel
    stalls on that item forever — nothing behind it runs. Call it once, when your
    async work (an animation, a timer, a player choice) completes. Sync items that
    use **Run On Queue** get this for free.

An item's lifecycle state is available for inspection:

```cpp
enum class EPQfQueueItemState : uint8
{
    None,            // never queued, or cleared
    Pending,         // waiting in a channel
    Running,         // executing inside RunOnQueueAsync
    WaitingProceed,  // async body returned but hasn't called Proceed yet
    Completed,       // has proceeded
};
```

Three convenience bases match the well-known channels — subclass whichever fits, or
`UQueuedItem` directly for a custom channel:

```cpp
class UQueuedOrder  : public UQueuedItem {};   // for the "Order" channel
class UQueuedAction : public UQueuedItem {};   // for the "Action" channel
class UQueuedEvent  : public UQueuedItem {};   // for the "Event" channel

// The well-known channel names (a channel is just an FName; these are shared constants):
namespace QueueChannel
{
    const FName Order  = "Order";
    const FName Action = "Action";
    const FName Event  = "Event";
}
```

## Channel control

Pause, single-step, and clear are what make a queue debuggable — freeze a channel and
walk it one item at a time.

```cpp
void PauseChannel(FName Channel);    // stop dequeuing new items
void ResumeChannel(FName Channel);   // resume normal processing

// Run exactly one pending item, then pause again — step through a sequence by hand.
void StepChannel(FName Channel);

// Drop all PENDING items. Does not interrupt an item already running.
void ClearChannel(FName Channel);
```

## Query and debug

Every getter here is pure and Blueprint-readable.

```cpp
TArray<FName> GetChannelNames() const;              // channels that exist
int32         GetQueueLength(FName Channel) const;  // pending count
UQueuedItem*  GetCurrentItem(FName Channel) const;  // the running item, or null
TArray<UQueuedItem*> GetPendingItems(FName Channel) const;
bool          IsChannelBusy(FName Channel) const;   // running or has pending
FQueueChannelState   GetChannelState(FName Channel) const;

// Recent completions (oldest first), for a debug timeline. Retention is bounded by
// HistoryCapacityPerChannel (default 32).
TArray<FPQfCompletedItemRecord> GetRecentHistory(FName Channel) const;

// Log every channel's current state to the output log.
void DumpDebugInfo() const;
```

## The Queue Item node

In Blueprint you rarely construct and queue an item in two steps — the **Queue Item**
node does both. Like **Spawn Actor**, it exposes the item class's spawn-time properties
as input pins and its delegate outputs as event pins, then queues the finished item on
a channel.

=== "Blueprint"
    1. Add **Queue Item** and pick your `UQueuedItem` subclass as the **Class**.
    2. Fill the exposed spawn pins (whatever your item marked *Expose on Spawn*).
    3. Set the **Channel** (or use a channel-locked variant of the node).
    4. Optionally bind the item's **On Proceed** output to react when it finishes.

=== "C++"
    ```cpp
    // In C++ there is no node — construct, configure, and queue directly.
    UMyAttackAction* Action = NewObject<UMyAttackAction>(this);
    Action->Target = Victim;                       // your item's own properties
    UQueueSubsystem::Get()->QueueItem(Action, QueueChannel::Action);
    ```
