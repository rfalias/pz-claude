# Project Zomboid Build 42 — Modding Lessons Learned

Hard-won lessons from building and shipping PlayerShops (Workshop ID `3749824460`) on B42.
Covers things that aren't in the official docs, are wrong in the official docs, or will burn hours if you discover them the wrong way.

---

## Table of Contents

1. [B42 Breaking Changes vs B41](#b42-breaking-changes)
2. [Folder & File Layout](#folder-and-file-layout)
3. [Client/Server Split Architecture](#clientserver-split-architecture)
4. [Testing & Debugging Environment](#testing--debugging-environment)
5. [Workshop Publishing Gotchas](#workshop-publishing-gotchas)
6. [Admin / Privilege Checking](#admin--privilege-checking)
7. [ISUI: ScrollingListBox, Draw Callbacks, Resize](#isui-scrollinglistbox-draw-callbacks-resize)
8. [World Object & Container APIs](#world-object--container-apis)
9. [Custom Items (registries, equip slots, loot tables)](#custom-items)
10. [HaloText & Server→Client Notification Pattern](#halotext--serverclient-notification-pattern)
11. [Safehouse API](#safehouse-api)
12. [Dedicated Server Setup](#dedicated-server-setup)
13. [Miscellaneous Engine Behavior](#miscellaneous-engine-behavior)
14. [Verifying APIs Against Real Game Files (Not Just Docs)](#verifying-apis-against-real-game-files-not-just-docs)
15. [Sandbox Options — Format Gotchas](#sandbox-options--format-gotchas)
16. [Server-Authoritative Progression & Reward Systems](#server-authoritative-progression--reward-systems)
17. [Cross-Session Persistent Storage (Outside the Save)](#cross-session-persistent-storage-outside-the-save)

---

## B42 Breaking Changes

These aren't always in the Migration Guide and some things in the guide are wrong.

### registries.lua — required for custom IDs

Since 42.13, any custom `CharacterTrait`, `CharacterProfession`, `ItemTag`, `ItemType`, `BodyLocation`, etc. must be **registered before any script loads**, in a file named exactly `registries.lua` placed in `media/` (not `media/lua/`):

```lua
-- media/registries.lua
CharacterTrait.register("mymod:mytraitname")
ItemBodyLocation.register("mymod:wallet")
```

IDs must be fully namespaced: `modid:name`. Unnamespaced IDs silently fail or conflict.

### New definition block syntax

Traits and professions no longer use bare `trait { }` blocks. Use:
- `character_trait_definition { }` — references the registered namespaced ID
- `character_profession_definition { }` — same

### Migration Guide PDF is wrong about `DisplayName`

The official `Migration Guide.pdf` claims `DisplayName` was removed from item scripts. **It wasn't.** Real, working, published Workshop mods still use `DisplayName = ...` directly and it works fine as of B42.19. Cross-check against real Workshop mods (`~/.local/share/Steam/steamapps/workshop/content/108600/<id>/`) when the guide contradicts what you observe.

### TimedActions require a perform/complete split for MP

- `new()` args must be saved as named fields on `self`
- `getDuration()` is required
- `perform()` is **client-only** (sounds, animations only)
- `complete()` is **server-only** — ALL item/object creation, deletion, modification happens here

Any inventory mutation that happens inside `perform()` will be rolled back or invisible to other players.

### `getContainer()` is on `IsoObject`, not `IsoThumpable`

B42 pre-placed furniture from the "Moveables" catalog (restaurant counters, kitchen counters, Spiffo's furniture, etc.) is NOT `IsoThumpable` but DOES have a real `getContainer()` with live loot. If you gate on `instanceof(object, "IsoThumpable")` you silently exclude all of them.

Correct pattern for "does this have a real container?":
```lua
if not instanceof(object, "IsoDeadBody") and object:getContainer() ~= nil then
    -- safe to treat as a storage container
end
```

Never use `instanceof(object, "IsoThumpable")` as a proxy for "has a container" in B42.

---

## Folder and File Layout

B42 adds versioned subfolders. Layout:

```
mods/
  MyMod/
    mod.info
    media/
      registries.lua          ← custom type registrations (runs first)
      scripts/
        items.txt             ← item/recipe definitions
    42/                       ← B42-specific content
      media/
        lua/
          client/
          server/
          shared/
        sandbox-options.txt
```

You can have both `media/` (shared across versions) and `42/` (B42-specific). The `42/` files take precedence.

---

## Client/Server Split Architecture

### Local "Host" is NOT single-process

The in-client "Host" button spawns a **genuinely separate dedicated-server subprocess** (`zombie.network.GameServer`), even on one machine. Confirmed via `ps aux`.

- Server-side errors and `print()` go to: `Logs/<timestamp>_DebugLog-server.txt`
- Client errors go to: `console.txt`
- If you're debugging server Lua and nothing shows in `console.txt`, you're looking in the wrong log.

### Server-side command flow

Client → Server:
```lua
sendClientCommand(playerObj, "MyMod", "CommandName", { x=1, y=2, z=0 })
```

Server handler:
```lua
MyModServer.OnClientCommand = function(module, command, player, args)
    if module ~= "MyMod" then return end
    local handler = MyModServer.handlers[command]
    if handler then handler(player, args) end
end
Events.OnClientCommand.Add(MyModServer.OnClientCommand)
```

Server → Client:
```lua
sendServerCommand(player, "MyMod", "ResponseName", { data = result })
```

Client handler:
```lua
local function onServerCommand(module, command, args)
    if module ~= "MyMod" then return end
    if command == "ResponseName" then ... end
end
Events.OnServerCommand.Add(onServerCommand)
```

### ISInventoryTransferAction.isValid() — client-side loot gate

Vanilla uses this exact pattern for safehouse loot protection. Fires once per drag/drop transfer attempt. Returning `false` abandons the action client-side before it ever reaches the server:

```lua
local original_isValid = ISInventoryTransferAction.isValid

function ISInventoryTransferAction:isValid()
    if someConditionThatShouldBlock(self.srcContainer, self.character) then
        HaloTextHelper.addBadText(self.character, getText("IGUI_MyMod_NoAccess"))
        return false
    end
    return original_isValid(self)
end
```

This does NOT fire for server-authoritative transfers (the Buy command pattern), only for player-initiated drag/drop.

---

## Testing & Debugging Environment

### The Workshop staging-shadow bug — will burn hours

**Symptom:** You edit a file in `mods/MyMod/`, restart the game fully, but in-game behavior doesn't change.

**Cause:** Once a mod has had a Workshop item created for it, PZ's local "Host" can silently load the mod from the Workshop *staging folder* (`~/Zomboid/Workshop/MyMod/Contents/mods/MyMod/`) instead of your real dev folder (`~/Zomboid/mods/MyMod/`). No error, no warning, no log line says this is happening.

**Fix:** After every batch of edits, re-sync:
```bash
rm -rf ~/Zomboid/Workshop/MyMod/Contents/mods/MyMod/
cp -rL ~/Zomboid/mods/MyMod/ ~/Zomboid/Workshop/MyMod/Contents/mods/MyMod/
```

Note: `cp -rL` (follow symlinks) is required — a symlinked directory uploads/loads as empty.

**Safest practice:** Make re-syncing the staging folder part of your "test loop", not just your "publish loop".

### WorldDictionary poisoning on reused saves

**Symptom:** A custom item silently fails to register — no error, just absent from `/additem` and spawn lists — even though syntax is correct and confirmed working in a fresh test mod.

**Cause:** Multiple edit-test cycles on the same save (renaming the item, changing its module, changing fields) or a prior entity-script crash can leave a poisoned `WorldDictionary` entry in the save's DB.

**Fix:** Delete the save (`Saves/<type>/<name>/`) and the matching `db/<servername>.db`. Use a brand-new item name for testing if you suspect poisoning.

**Quick registration test:** `/additem "youruser" "Module.ItemName"` — gives an explicit "doesn't exist" error if the item isn't registered, unlike spawn-list UIs that just show nothing.

**Note:** Always rule out the Workshop staging-shadow bug (above) first — it's cheaper to check and was the actual culprit in at least one incident where dictionary poisoning was initially blamed.

### Fresh saves for every test session

The `servertest` default server accumulates state. Keep a "clean" test world name separate from long-running test sessions, especially when iterating on registration-sensitive things (custom items, entity scripts).

---

## Workshop Publishing Gotchas

1. **No symlinks.** The staging folder needs real files. Use `cp -rL`, not `ln -s`. Symlinked contents upload as 0 bytes.

2. **"Private" blocks anonymous dedicated servers.** If your server connects without a Steam identity (non-Steam mode, see below), it can't download "Private" Workshop items. Use **"Unlisted"** visibility for anything a non-Steam-authenticated server needs.

3. **Editing the staging folder ≠ updating the Workshop item.** You must resubmit via "Create/Edit Workshop Item" (in-game) or the Steam Workshop website. Editing files under `Workshop/` locally doesn't push anything.

4. **Re-sync staging after every edit session**, not just before publishing — see the staging-shadow bug above.

---

## Admin / Privilege Checking

### `isAdmin()` does not exist in B42 Lua — on client OR server

`player:isAdmin()` throws "Object tried to call nil" on **both** the client-side and server-side `IsoPlayer` Lua object in B42. Do not use it anywhere.

### Use `getAccessLevel():lower()` everywhere — it returns lowercase

`getAccessLevel()` works on both client and server and returns `"admin"` (all lowercase) on dedicated servers. An exact `== "Admin"` check silently fails. Always `.lower()` the result:

```lua
local level = (player:getAccessLevel() or ""):lower()
local privileged = level == "admin" or level == "moderator"
```

This is the correct pattern for both client-side context menus and server-side handlers. Confirmed working via dedicated server debug console: `getPlayer():getAccessLevel()` → `"admin"`.

### Keep privilege checks consistent

If you have a server-side `isPrivileged()` helper, use it in every handler that has a permission gate. Don't inline `getAccessLevel() == "Admin"` in individual handlers — it will diverge from the helper and cause subtle bugs where some actions work for admins and others don't.

---

## ISUI: ScrollingListBox, Draw Callbacks, Resize

### ISScrollingListBox — scrollbar state detection

`getIsVisible()` on a scrollbar always returns `true` even when no scroll is needed. The correct check for "is the scrollbar actually active?":

```lua
local scrollActive = self.vscroll and (self.vscroll:getHeight() < self:getScrollHeight())
```

When the scrollbar IS active, it clips the draw area via a stencil at `vscroll.x + 3`. This affects selection highlight width and right-edge text positioning:

```lua
function MyListBox:drawListItem(y, item, alt)
    local scrollActive = self.vscroll and (self.vscroll:getHeight() < self:getScrollHeight())
    if self.selected == item.index then
        local selW = scrollActive and (self.vscroll.x + 3) or self.width
        self:drawSelection(0, y, selW, item.height - 1)
    end
    local rEdge = scrollActive and (self.vscroll.x - 2) or (self:getWidth() - 4)
    -- draw text using rEdge as right boundary
end
```

If you use `self.width` for selection width when a scrollbar is present, you get a visible gap on the right edge of highlighted items.

### Mid-frame scrollbar state flicker

When a list is near its scroll threshold (content is just barely overflowing), `vscroll:getHeight() < self:getScrollHeight()` can return different values between the prerender and render phases due to Java state updates mid-frame. This causes visible flicker of the selection highlight.

Fix: cache the scroll state at the end of `prerender()` and use the cached value in draw callbacks:

```lua
function MyParentUI:prerender()
    -- ... layout/update logic ...
    local sv = self.myList.vscroll
    self.myList._scrollActive = sv and (sv:getHeight() < self.myList:getScrollHeight())
end

function MyListBox:drawListItem(y, item, alt)
    local scrollActive = self._scrollActive  -- cached once per frame, never stale
    ...
end
```

### ISCollapsableWindow resize wiring

`setResizable(true)` (default) enables both `resizeWidget` (corner drag) and `resizeWidget2` (bottom-edge drag). Wire both:

```lua
function MyWindow:createChildren()
    ISCollapsableWindow.createChildren(self)
    self.resizeWidget.resizeFunction = MyWindow.onResize
    self.resizeWidget2.resizeFunction = MyWindow.onResize
end

function MyWindow:onResize(w, h)
    self:layoutChildren()
end
```

Forgetting `resizeWidget2` leaves the bottom-edge drag broken with no error.

Set `self.minimumWidth` and `self.minimumHeight` on the window — `ISResizeWidget` clamps to them automatically.

### Tracking size changes in prerender

For windows that need to re-layout when size changes (e.g. from external resize events):

```lua
function MyWindow:prerender()
    if self._lastW ~= self.width or self._lastH ~= self.height then
        self._lastW, self._lastH = self.width, self.height
        self:layoutChildren()
    end
    ISCollapsableWindow.prerender(self)
end
```

---

## World Object & Container APIs

### Resolving objects by coordinate + index

Object list indices are unstable across chunk reloads. Store `{x, y, z}` as the canonical key. When you need the object, search by coords and then verify ownership/type:

```lua
local function resolveKioskAt(x, y, z, username)
    local square = getCell():getGridSquare(x, y, z)
    if not square then return nil, false end  -- chunk not loaded
    local objects = square:getObjects()
    for i = 0, objects:size() - 1 do
        local obj = objects:get(i)
        if isKiosk(obj) and getOwner(obj) == username then
            return obj, true
        end
    end
    return nil, true  -- loaded but not found = genuinely gone
end
```

The `(nil, false)` vs `(nil, true)` distinction matters for registry cleanup: don't prune a registry entry when the chunk simply wasn't loaded.

### Counting items across an entire inventory (including containers)

```lua
local count = player:getInventory():getItemCount("Base.Bandage", true)
```

The second boolean argument means "recurse into containers" (bags/backpacks worn or carried by the character) — confirmed from vanilla usage in `shared/TimedActions/ISBarricadeAction.lua` (`getItemCount("Base.MetalBar", true)`) and `client/ISUI/ISInventoryPaneContextMenu.lua`. Omit it (or pass `false`) to count only the top-level container. Useful for "collect N of item X" mechanics — safe to poll server-side, since B42 made all inventory item creation/deletion server-authoritative (per the official "API for Inventory Items" doc), so the server's view of a player's item count can't be spoofed by a modified client.

### modData persistence

`object:getModData()` returns a Lua table attached to that IsoObject. Write to it directly, then call `object:transmitModData()` to sync to all clients. Same pattern for players: `player:getModData()` + `player:transmitModData()`.

Player modData is available on the CLIENT side (synced from server on login and whenever `transmitModData()` is called). Useful for lightweight registry data you need client-side without a round-trip.

---

## Custom Items

### Custom equip slots require visual assets — not pure Lua

`ItemBodyLocation.register("mymod:slot")` + `BodyLocation = mymod:slot` in item script + `CanBeEquipped = mymod:slot` all work for the data/capacity side. But in multiplayer, `character:setWornItem()` broadcasts a `SyncClothingPacket` that calls `item:getVisual():getTint()` — if the item has no `ClothingItem` visual reference, this NPEs and the sync packet is never sent. The item IS set server-side (visible after full reconnect) but the live UI never refreshes for any client.

**Conclusion:** Live-updating custom equip slots in MP requires 3D model + texture assets (out of scope for pure script/Lua modding). Don't promise it without artist assets.

### `base:bagsfillexception` on wallets

`Wallet_Female` and `Wallet_Male` carry `base:bagsfillexception`, exempting them from the lazy-fill-on-open system. If you spawn a wallet programmatically, it will be empty unless you explicitly roll and add money:

```lua
local wallet = InventoryItemFactory.CreateItem("Base.Wallet_Male")
local inv = wallet:getInventory()  -- or however you access sub-inventory
-- explicitly add money items to inv
```

### Entity script sprite uniqueness

The B42 entity-script system (`entity { component SpriteConfig {...} }`) requires every sprite to be claimed by exactly one entity. Reusing an existing vanilla sprite crashes world load with `SpriteConfigManager.parseEntityScript: Sprite 'X' is duplicate` and can poison the save's dictionary. This does NOT apply to plain inventory `item {}` blocks.

Don't define new entity scripts reusing vanilla sprite names. For placeable objects, either find a genuinely unclaimed sprite or use a "convert existing container" pattern (no entity definition needed).

### Loot table scaling

```lua
Events.OnPostDistributionMerge.Add(function()
    -- safe to mutate Distributions and ProceduralDistributions here
    -- all mod overrides are already merged at this point
    local seen = {}
    -- use `seen` table when recursing to avoid double-scaling aliased sub-tables
end)
```

Zombie corpse inventory injection:
```lua
Events.OnFillContainer.Add(function(roomName, containerType, container)
    if containerType == "inventoryfemale" or containerType == "inventorymale" then
        -- this is a zombie corpse inventory
    end
end)
```

---

## HaloText & Server→Client Notification Pattern

```lua
-- Green text
HaloTextHelper.addGoodText(player, getText("IGUI_MyMod_Key"))

-- Red text
HaloTextHelper.addBadText(player, getText("IGUI_MyMod_Key"))

-- With line break separator
HaloTextHelper.addBadText(player, getText("IGUI_MyMod_Key"), "[br/]")
```

These are **client-side only**. For server→client notification:

```lua
-- Server side
sendServerCommand(player, "MyMod", "ShowBadText", { messageKey = "IGUI_MyMod_Key" })

-- Client side (in OnServerCommand handler)
if command == "ShowBadText" then
    HaloTextHelper.addBadText(getPlayer(), getText(args.messageKey))
end
```

---

## Safehouse API

### Events.OnSafeHouseRemoved (server-side)

Fires when a safehouse claim is released. Receives the `SafeHouse` object:

```lua
Events.OnSafeHouseRemoved.Add(function(safeHouse)
    local x1, y1 = safeHouse:getX(), safeHouse:getY()
    local x2 = x1 + safeHouse:getW() - 1
    local y2 = y1 + safeHouse:getH() - 1
    for tx = x1, x2 do
        for ty = y1, y2 do
            local square = getCell():getGridSquare(tx, ty, 0)
            if square then
                -- inspect square:getObjects() for cleanup
            end
        end
    end
end)
```

### Safehouse blocks ALL right-click interactions for non-members

PZ's safehouse system blocks context menu interaction in safehouse territory for non-members before your `OnFillWorldObjectContextMenu` handler even runs. If you need non-members to interact with objects in safehouse territory (e.g. browsing a player shop), **you cannot work around this with context menu code alone** — you'd need to remove the safehouse requirement entirely.

`isProtected()` / `SafeHouse` ownership: if you don't need a safehouse for your feature, don't use one. Protection can be implemented via modData ownership checks instead.

---

## Dedicated Server Setup

### Non-Steam client for testing alongside Steam client

The standalone dedicated server (`start-server.sh`) hardcodes `-Dzomboid.steam=0`. A regular Steam client connecting to it can fail with vague errors ("server failed to respond", stuck at `GettingServerInfo`).

Connect with a non-Steam client via a custom exe config:
1. Copy the default `projectzomboid64.json` pzexeconfig to a new file
2. Set `"-Dzomboid.steam=0"` in the JVM args
3. Launch: `projectzomboid.sh -pzexeconfig myconfig.json -cachedir=/path/to/isolated/cache`

Two Steam clients under the same account get "user already connected" — the non-Steam config sidesteps this since it has no SteamID identity.

### RCON

- Requires `RCONPassword` set in the server ini (empty = unusable)
- Uses the Source RCON protocol
- Some servers send an extra empty `SERVERDATA_RESPONSE_VALUE` packet before the real `SERVERDATA_AUTH_RESPONSE` — a hand-rolled client must read packets in a loop, not just read one response after AUTH

### Server logs

All server-side `print()` and errors go to `Logs/<timestamp>_DebugLog-server.txt`. The client's `console.txt` has nothing about server-side Lua execution.

---

## Miscellaneous Engine Behavior

### Floating world labels — rebuild every 500ms

Floating labels attached to world objects (via `IsoCell.setLabel()` or similar) get wiped when the player opens and closes the Escape menu. If you want persistent floating labels, rebuild them on a timer (every 500ms works well) via `Events.OnTick` or similar, not just once on chunk load.

### Sandbox options

Defined in `media/sandbox-options.txt`:

```
VERSION = 1,
option PlayerShops.MaxKiosksPerPlayer
{
    type = integer,
    min = 0,
    max = 100,
    default = 5,
    ...
}
```

Accessed server-side as `SandboxVars.PlayerShops.MaxKiosksPerPlayer`. Available client-side too, but treat client-side sandbox access as read-only.

For gotchas specific to list-style/string options (comma-splitting, `type = string`), see [Sandbox Options — Format Gotchas](#sandbox-options--format-gotchas).

Translation strings for sandbox options go in `media/lua/shared/Translate/EN/Sandbox.json`, NOT `IG_UI.json`:

```json
{
    "Sandbox_PlayerShops_MaxKiosksPerPlayer": "Max Kiosks Per Player",
    "Sandbox_PlayerShops_MaxKiosksPerPlayer_tooltip": "..."
}
```

### B42 pre-placed furniture object indexing

Object indices in `square:getObjects()` are NOT stable. They can change when a chunk is unloaded and reloaded (objects can shift positions in the list). Never persist an index as a long-term key. Use `{x, y, z}` coords + a search loop instead.

### `instanceof()` checks

`instanceof(object, "ClassName")` is the correct B42 pattern. Don't use Lua `type()` for game object type checks — everything is userdata.

---

## Verifying APIs Against Real Game Files (Not Just Docs)

The official Migration Guide and community wikis are sometimes wrong or incomplete (see the `DisplayName` example earlier in this doc). When you need ground truth on an exact function name, argument order, or whether something is client-only/server-only, grep the actual shipped Lua source directly — don't guess from documentation alone.

### Finding the game install and Zomboid user data directory (cross-platform)

**Do not hardcode a path you found on one machine into a mod, script, or doc as if it's universal** — either derive it (see below) or just ask the user for their install directory. This matters if you (or an AI assistant helping you) are operating on an unfamiliar machine, someone else's setup, or writing something meant for others to use, like this file.

**Zomboid user data directory** (saves, mods, Workshop cache, and the `Lua/` folder used for cross-session persistent files — see [Cross-Session Persistent Storage](#cross-session-persistent-storage-outside-the-save) below):
- Windows: `%USERPROFILE%\Zomboid\` (typically `C:\Users\<you>\Zomboid\`)
- Linux: `~/Zomboid/`
- macOS: `~/Zomboid/`
- A dedicated server started with `-cachedir=<path>` uses that path instead of the default — check the server's launch script/service unit if data isn't where you expect it.

**Game install directory** (vanilla Lua source under `media/lua/{client,server,shared}/`, plus the compiled `projectzomboid.jar` for anything not exposed in Lua):
- Steam default on Windows: `<SteamLibrary>\steamapps\common\ProjectZomboid\`
- Steam default on Linux: `<SteamLibrary>/steamapps/common/ProjectZomboid/projectzomboid/` (note the extra nested `projectzomboid/` folder)
- If it's not at the default library location, check `steamapps/libraryfolders.vdf` inside any Steam install for additional library paths — many users split games across drives.
- Non-Steam storefronts: locations vary; ask rather than assume.

Once you have the install path, grep directly for the API you're unsure about, e.g.:
```bash
grep -rn "getItemCount(" "<install>/media/lua"
```
This is far more reliable than the Migration Guide PDF or third-party wikis for exact signatures.

### When you can't decompile the Java side

The compiled game logic lives in `projectzomboid.jar` (`zombie/...` packages). Full decompilation needs an external tool that isn't bundled with the game or necessarily present on a dev machine (`javap`, a Java decompiler). If you need to know whether a Lua global is client-only or also usable server-side and can't decompile:
- Grep vanilla `media/lua/server/` for the same function — if vanilla itself never calls it server-side, that's inconclusive, not proof it's blocked.
- Fall back on established, widely-used community modding conventions, then confirm with a real in-game test (e.g. write a file from a server-side handler, then check the file appears where expected) rather than committing to code you haven't verified at all.

---

## Sandbox Options — Format Gotchas

### `sandbox-options.txt` splits on every top-level comma — quotes do NOT help

The option file's parser does a naive split on every comma inside an option block, with no quote-awareness. A `default` value that itself contains a comma gets silently truncated at the first one, **even if you wrap it in quotes**:

```
# BROKEN — silently becomes just `"5` (truncated at the first comma, quote kept literally)
option MyMod.SomeList
{
    type = string,
    default = "5,10,20,40,80",
    ...
}
```

**Fix:** never put a comma in a `default` value. Pick a different delimiter for admin-configurable lists — semicolon works well and doesn't collide with the file's own parsing — and don't quote it either (quotes aren't stripped; they just become part of the string):

```
option MyMod.SomeList
{
    type = string,
    default = 5;10;20;40;80,
    ...
}
```

Parse it in Lua with `string:gmatch("[^;]+")`. Document the delimiter choice in the option's tooltip (`_tooltip` key in `Sandbox.json`) so admins editing the value don't reach for commas out of habit.

### `type = string` is a supported, undocumented-feeling option type

Confirmed via vanilla `client/OptionScreens/SandboxOptions.lua`: `type = string` (same as `type = entry`) renders a free-text `ISTextEntryBox` in the Sandbox Options UI. Read/write it at runtime as a plain Lua string via `SandboxVars.<Page>.<OptionName>`. Useful for exactly this kind of delimited-list configuration — no need to fall back to a fixed number of separate integer options when a single flexible list will do.

---

## Server-Authoritative Progression & Reward Systems

Patterns for building any feature that grants something (currency, items, unlocks) based on a player's in-game actions or stats, safely in multiplayer.

### Trust only server-verifiable state, not client-fired "I did X" events

Many vanilla events that sound like good hooks (`OnZombieDead`, `OnAddItemCraftResult`, `OnPlayerDeath`) are, in the actual shipped game, only ever wired up in **client-side** Lua. Firing your own reward logic off one of these means trusting a client's self-report — a modified client can fire it without the action ever happening.

Before trusting an event, check where vanilla itself uses it:
```bash
grep -rln "OnZombieDead" "<install>/media/lua/client" "<install>/media/lua/server" "<install>/media/lua/shared"
```
If every hit is under `client/`, treat it as untrustworthy for anything that grants value.

The safe alternative: **poll a getter on the character object itself**, server-side, on a timer. Getters like `getZombieKills()`, `getHoursSurvived()`, `getPerkLevel()` return the server's own authoritative state — the same object the server already trusts for everything else. `Events.EveryOneMinute` is a good, vanilla-proven cadence for this (used by vanilla's own `SCampfireSystem.lua`):

```lua
local function checkPlayer(player)
    local current = player:getZombieKills()
    -- compare against a stored baseline, award if it crossed a threshold
end

Events.EveryOneMinute.Add(function()
    local players = getOnlinePlayers()
    for i = 0, players:size() - 1 do
        checkPlayer(players:get(i))
    end
end)
```

### Anti-farming: baselines must never decrease

If you track "reward every time stat X crosses another threshold" by storing a baseline/last-seen value, **that baseline must never be allowed to decrease** — even though character death creates a brand-new character object whose stats reset to zero. If the baseline re-synced downward on death, a player could deliberately die right after earning a reward and cheaply re-earn the same threshold on the fresh character.

```lua
-- baseline: last known value (nil if never seen). Never decreases.
function pollMilestone(baseline, currentValue, bucketSize, tokensPerBucket)
    if baseline == nil then return 0, currentValue end
    if currentValue <= baseline then return 0, baseline end
    local buckets = math.floor((currentValue - baseline) / bucketSize)
    if buckets <= 0 then return 0, baseline end
    return buckets * tokensPerBucket, baseline + buckets * bucketSize
end
```

This slightly under-counts true lifetime effort across a death (there's no real cross-character lifetime counter the engine exposes), but that's a safe failure mode — it costs the player nothing exploitable.

For a true one-time "quest" (not a repeating milestone) — e.g. "collect N of item X" — you don't need baseline logic at all: just a permanent `completed[username][questId] = true` flag. Nothing to farm since it can only ever be granted once.

### Escalating thresholds without an endless config list

For admin-configurable milestone lists that should keep paying out indefinitely (e.g. every N zombie kills), let admins specify only the first few stages and extrapolate the rest by repeating the gap between the last two:

```lua
function getMilestoneThreshold(milestones, stageNumber)
    local n = #milestones
    if n == 0 then return nil end
    if stageNumber <= n then return milestones[stageNumber] end
    local lastGap = n >= 2 and (milestones[n] - milestones[n - 1]) or milestones[n]
    if lastGap <= 0 then lastGap = milestones[n] end
    return milestones[n] + (stageNumber - n) * lastGap
end
```

`milestones = {5, 10, 20, 40, 80}` keeps paying out every 40 past stage 5, without the admin ever having to extend the list.

### Track "lifetime earned" separately from spendable balance

If your reward currency can be spent (on redeemable rewards, etc.), keep a second counter that only ever increases — incremented alongside every grant, never decremented by spending. Spendable balance alone unfairly punishes players who used their currency during the period you're measuring (e.g. a season). This is also the natural value to snapshot for any "carry some value into the next reset" feature — see below.

### Enumerating "real" skills/perks, excluding category headers

`PerkFactory.PerkList` contains both real, levelable perks and category headers (Combat, Crafting, Passive, ...). Vanilla's own Character Info panel (`ISCharacterInfo.lua`) filters headers out by checking `getParent()`:

```lua
for i = 0, PerkFactory.PerkList:size() - 1 do
    local perk = PerkFactory.PerkList:get(i)
    if perk:getParent() ~= Perks.None then
        -- perk is a real, trainable skill
    end
end
```

Also use `perk:getName()` directly as the display string — that's what vanilla itself draws, with no `getText()` wrapping. Routing it through `getText("IGUI_perks_" .. name)` breaks for any perk whose name doesn't happen to match an existing translation key, silently showing the raw key text instead.

---

## Cross-Session Persistent Storage (Outside the Save)

`ModData.getOrCreate(key)` — the usual place to persist mod state — is serialized **inside the world save**. Anything stored there is gone the moment that save is deleted or regenerated (a "server reset" for a new season/wipe).

To survive a save wipe, write to a plain file instead, using the built-in Lua file I/O globals:

```lua
local writer = getFileWriter("MyMod_data.txt", true, false)  -- create if missing, overwrite (not append)
writer:write("someuser\t42\n")
writer:close()

local reader = getFileReader("MyMod_data.txt", true)  -- true = don't create if missing (nil if absent)
if reader then
    local line = reader:readLine()
    while line do
        -- process line
        line = reader:readLine()
    end
    reader:close()
end
```

These resolve relative to `<Zomboid user data dir>/Lua/` — a sibling of `Saves/`, not inside it. Deleting/regenerating a save doesn't touch this folder; only deleting the whole Zomboid user data directory would. Confirmed usable from **server-side** handlers (not just the client-side UI code — layout/keybind persistence, etc. — that vanilla itself uses it for) via direct in-game testing.

There's no delete primitive exposed to Lua. To "clear" a file (e.g. after consuming a one-shot snapshot so it can't be re-imported), overwrite it with an empty write rather than trying to remove it.

This is the mechanism behind a "carry some value into the next season" feature: snapshot the value to a file right before wiping the old save, then read it back during the new save's very first `Events.OnInitGlobalModData` call (which only fires for a genuinely fresh save) — applying whatever conversion rate is *currently* configured, computed live, rather than baking a pre-converted amount into the snapshot.

---

## Reference

- Local docs: `~/Zomboid/mod_docs/` — `Migration Guide.pdf`, API docs, PZwiki HTML snapshots (path may vary; see [Verifying APIs Against Real Game Files](#verifying-apis-against-real-game-files-not-just-docs) if you don't have a local copy)
- Community guides: https://github.com/demiurgeQuantified/PZModdingGuides
- Example mods this document was drawn from: PlayerShops (safehouse kiosk economy) and BattlePass (seasonal token progression) — both built iteratively while compiling these lessons
- PlayerShops Workshop item: https://steamcommunity.com/sharedfiles/filedetails/?id=3749824460
