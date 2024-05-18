# AbstractInvLib
This is a library for computercraft that provides layers of abstraction over the generic inventory API.

AIL acts as a drop in replacement for the built in inventory API. It can be dropped into an existing program if, for example, you wish to use multiple inventories as one.

But that's not the only thing AIL does. It caches the entire content of all inventories passed in so that you may perform transfers by item name/nbt, and transfer thousands of items in just a few ticks.

## What is it (not) for?
AIL is intended for static inventories, like a computer managed storage area. It caches the contents of the inventory and has no way of knowing *if* the inventory changes. If a transfer it attempts does not work, because of an item being in the way or being removed from the storage, it will error. This can be bypassed by the `allowBadTransfers` option, but this will result in a recache anytime the state is detected to be bad.


## Quick Start
Download `abstractInvLib.lua` and place it on the computer. Then to use:

```lua
local ail = require"abstractInvLib"
local inv = ail({"minecraft:chest_0", "minecraft:chest_1"})

-- then you can use it like a normal inventory
inv.pushItems("minecraft:chest_2", 1)
-- or transfer items by name, will use multiple slots if necessary
inv.pushItems("minecraft:chest_2", "minecraft:cobblestone", 1000)
-- transfers can be performed between multiple AIL wrapped inventories
local inv2 = ail({"minecraft:chest_2"})
inv.pushItems(inv2, "minecraft:stone", 300)
```

## Additional Usage
AIL also exposes some functions to take advantage of its internal cache. For example you may get the count of a specific item by calling `getCount(name,nbt?)`.

For the most updated information on these functions see the functions exposed through the `api` table. Any functions prefaced with `_` are for internal use.


## Multithreading
This of course isn't real multithreading, but due to the way peripheral callls work in CC, they may be done concurrently with other code. For a high I/O application like a storage system you may want to offload all inventory operations to a dedicated thread. AIL has this functionality built in.

Taking advantage of this will also help prevent computer lockups. Calling an AIL inventory operation on multiple threads similtaneously MAY result in more than 256 events being queued, and some being dropped. This results in your code locking up for seemingly no reason.

Example:
```lua
local ail = require"abstractInvLib"
local inv = ail({"minecraft:chest_0", "minecraft:chest_1"})

parallel.waitForAny(inv.run, function()
    -- these calls behave identically, but the inventory operations are enqueued
    -- and executed on the inv.run thread. This thread waits for an event
    -- signalling that it has finished this transfer.
    inv.pushItems("minecraft:chest_2", "minecraft:cobblestone", 1000)
    
    -- if you want to take advantage of this asynchronous transfer yourself
    -- you may queue the transfer instead
    local id = inv.queuePush("minecraft:chest_2", "minecraft:cobblestone", 1000)

    -- then you can await for it using
    inv.await(id)
    -- or
    local _, ailid, tid, result = os.pullEvent("ail_task_complete")
    if ailid == inv.uid and tid == id then
        -- these conditions state that this task was complete
        -- each AIL wrapped inventory is assigned a unique ID in .uid
    end
end)
```