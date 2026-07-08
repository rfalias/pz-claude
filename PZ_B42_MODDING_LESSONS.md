# Project Zomboid Build 42 — Modding Lessons Learned

Hard-won lessons from building and shipping PlayerShops (Workshop ID `3749824460`) and BattlePass, plus building DynamicSpawnPoints, on B42.
Covers things that aren't in the official docs, are wrong in the official docs, or will burn hours if you discover them the wrong way.

---

## Table of Contents

1. [B42 Breaking Changes vs B41](#b42-breaking-changes)
2. [Folder & File Layout](#folder-and-file-layout)
3. [Client/Server Split Architecture](#clientserver-split-architecture)
4. [Monkey-Patching Vanilla Classes — Hook Selection & Timing](#monkey-patching-vanilla-classes--hook-selection--timing)
5. [Testing & Debugging Environment](#testing--debugging-environment)
6. [Workshop Publishing Gotchas](#workshop-publishing-gotchas)
7. [Admin / Privilege Checking](#admin--privilege-checking)
8. [ISUI: ScrollingListBox, Draw Callbacks, Resize, HUD Integration](#isui-scrollinglistbox-draw-callbacks-resize-hud-integration)
9. [World Object & Container APIs](#world-object--container-apis)
10. [Custom Items (registries, equip slots, loot tables)](#custom-items)
11. [Picker & Display-Name UX Gotchas](#picker--display-name-ux-gotchas)
12. [HaloText & Server→Client Notification Pattern](#halotext--serverclient-notification-pattern)
13. [Safehouse API](#safehouse-api)
14. [Dedicated Server Setup](#dedicated-server-setup)
15. [Boot-Once & Cached Server State](#boot-once--cached-server-state)
16. [Miscellaneous Engine Behavior](#miscellaneous-engine-behavior)
17. [Verifying APIs Against Real Game Files (Not Just Docs)](#verifying-apis-against-real-game-files-not-just-docs)
18. [Sandbox Options — Format Gotchas](#sandbox-options--format-gotchas)
19. [Server-Authoritative Progression & Reward Systems](#server-authoritative-progression--reward-systems)
20. [Cross-Session Persistent Storage (Outside the Save)](#cross-session-persistent-storage-outside-the-save)
21. [Content Authoring & Verification](#content-authoring--verification)

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

## Monkey-Patching Vanilla Classes — Hook Selection & Timing

### Prefer a real `Events.OnX` listener over monkey-patching, when one already exists

If vanilla already fires a genuine event at the moment you care about (e.g. `Events.OnWeaponSwingHitPoint` for "a ranged weapon was fired"), add an independent listener instead of overwriting a vanilla function. It's less fragile across game updates and can't accidentally break vanilla's own logic if your wrapper has a bug.

```lua
Events.OnWeaponSwingHitPoint.Add(function(swingState)
    local weapon = swingState.weapon
    if weapon and weapon:isRanged() then
        -- count a shot fired
    end
end)
```

Only fall back to monkey-patching a function (`local original = SomeClass.someMethod; SomeClass.someMethod = function(...) ... end`) when no such event exists for the moment you need.

### A file-level `if isClient() then return end` guard makes monkey-patching that class unsafe at top level

Some vanilla class files (e.g. `Farming/SFarmingSystem.lua`, and its parent `Map/SGlobalObjectSystem.lua`) open with a guard like:

```lua
if isClient() then return end
```

This means the class simply **doesn't exist** in any context where `isClient()` evaluates true at `require` time — and on a freshly wiped dedicated server, `isClient()`/`isServer()` are not guaranteed to be correctly resolved yet during the very first mod-loading pass. A plain top-level `local original = SFarmingSystem.harvest` in your own mod file can crash the entire mod-load pass with `attempted index: harvest of non-table: null` — and this may not reproduce when testing against an existing save, only on a genuine fresh wipe.

**Before adding a top-level monkey-patch to any vanilla class**, grep that class's own file (and its parent, if it `:derive`s from one) for a file-level `isClient()`/`isServer()` guard on or near line 1. If present, don't capture/patch it at file scope — defer until the engine is fully booted:

```lua
Events.OnGameBoot.Add(function()
    if not SFarmingSystem then return end  -- defensive, still worth checking
    local original = SFarmingSystem.harvest
    SFarmingSystem.harvest = function(...)
        -- your logic
        return original(...)
    end
end)
```

Vanilla itself is aware of this exact race — `SGlobalObjectSystem.lua` has dead (commented-out) code wrapping its own init in `Events.OnGameBoot.Add(...)`, evidence this is a known, previously-worked-around timing issue in the base game.

### `getDuration() == -1` TimedActions don't follow the normal perform()/complete() split

The standard "`perform()` is client-only presentation, `complete()` is server-only logic" rule (see [B42 Breaking Changes](#b42-breaking-changes)) does **not** hold for every TimedAction. Some — `ISReloadWeaponAction`, `ISInsertMagazine` — are event-driven (`getDuration()` returns `-1`) and their `complete()` is a trivial `return true` stub. Their real, server-verified logic instead lives inside `animEvent(event, parameter)`, gated by an internal `if not isClient() then ... end` check on the specific branch that matters (e.g. the `'loadFinished'` case, which also grants vanilla's own Reloading XP — proof it genuinely runs server-side):

```lua
local original = ISReloadWeaponAction.animEvent
function ISReloadWeaponAction:animEvent(anEvent, param)
    if anEvent == 'loadFinished' and not isClient() then
        -- your server-verified logic here
    end
    return original(self, anEvent, param)
end
```

**How to apply:** before assuming a TimedAction follows the standard split, check what `getDuration()` returns. If it's `-1`, read the actual `animEvent`/`serverStart` bodies to find where the real `isClient()`/`isServer()` gate is — don't hook `perform()`/`complete()` blindly and assume silence means you did it right; it may simply never fire.

### One user-facing "feature" can map to several unrelated action classes

Don't assume the first class you find that looks related is the only path to the mechanic you're hooking. Confirmed multiple times:
- Weapon reload: direct-load weapons use `ISReloadWeaponAction`; magazine-fed weapons route through a completely separate class chain (`ISEjectMagazine` / `ISLoadBulletsInMagazine` / `ISInsertMagazine`) instead.
- "Fixing a vehicle": `ISFixVehiclePartAction` only covers in-place repair (not every part type even offers it in its context menu) — a full swap (most parts, always for windshields/batteries) goes through the entirely different `ISInstallVehiclePart`/`ISUninstallVehiclePart` pair.

**How to apply:** test the exact player action your feature is supposed to reward (not just whichever hook looked cleanest in the source) before considering the feature done. If a mechanic has both an "in-place fix" and a "replace the whole item" flow, you likely need to hook both. For actions that roll success/failure server-side (`ISInstallVehiclePart`, `ISUninstallVehiclePart`), check the actual state transition to confirm success rather than assuming `complete()` firing means it worked — e.g. `part:getInventoryItem() == item` is only true after a successful install; on failure the item goes back to the character's inventory instead.

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

### Local "Host" dual-process quirk: two live processes, two private copies of your server-side state

This is distinct from the "not single-process" note above (which is about *where to look for logs*) — this is about *data correctness*. The embedded client-hosting process AND the separate `zombie.network.GameServer` subprocess both load and execute `server/*.lua` independently, each with its own private memory. A naive in-memory counter or ModData increment done from a passive/frequent code path can be applied in one process while a read happens against the other, producing a torn or double-counted result — confirmed via a genuinely torn debug-log line from concurrent writes during local-host testing.

**How to apply:** for any hot, frequently-incrementing counter (a per-tick poll, a death counter, etc.) where losing an increment is a real problem, don't trust a bare ModData/in-memory value as the single source of truth across the two processes. Dedupe through a small resource both processes actually share on disk (a state file, checked/updated atomically) rather than assuming "it's one game session" means "one process." This is NOT worth doing for every ModData write — an idempotent, periodic full-snapshot overwrite (e.g. a 10-minute data export) can tolerate a stale-by-one-interval read and self-corrects next cycle; reserve the file-based dedup treatment for counters where a lost race silently and permanently drops data.

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

## ISUI: ScrollingListBox, Draw Callbacks, Resize, HUD Integration

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

### Prefer patching a vanilla panel to add a child over an independent `addToUIManager()` element

For anything meant to live permanently in vanilla's own persistent HUD chrome (e.g. a toolbar icon next to the Inventory/Crafting/Admin icons), adding your own top-level element via `addToUIManager()` gives you none of the lifecycle guarantees a real panel's own **child** gets for free. Two custom toolbar buttons built this way exhibited a whole cluster of bugs that all traced back to this one root cause:
- Independent position computed once at creation, anchored off another panel's layout-dependent getter (see below) — drifted out of sync with a sibling button computed at a slightly different moment.
- Disappeared or swapped icons on ANY big UI overlay (Escape menu, inventory, etc.) opening — `getPlayer()`-based "is this still the right player" identity checks are not reliable across all engine/UI states, and top-level `addToUIManager()` elements have no reliable tie to a specific character's lifecycle the way a panel's own child does.

**Fix that eliminated the entire bug class:** delete the standalone button classes and instead monkey-patch the real vanilla panel (e.g. `client/ISUI/ISEquippedItem.lua`, the Health/Inventory/Crafting/Safety/Admin icon column) to add your buttons as permanent **children** in its own `initialise()`, toggling visibility every frame in `prerender()` exactly the way vanilla's own admin-tool icon does (`self.chr:getRole():hasAdminTool()` → `setVisible(...)`), rather than adding/removing them from any manager. This rides on the panel's already-solid per-character lifecycle (`self.chr` resolved fresh by the panel itself, not cached by your mod) and its own consistent layout pass (siblings chained off each other's real bottom edge instead of independently re-deriving the same offset math).

### Never cache an `IsoPlayer` object across a possible death boundary

Death + respawn creates a brand-new `IsoPlayer` object. If a click handler or window captured the old one as its target, or a window bound `self.playerObj` at open time, every subsequent action goes out referencing the dead object — server commands sent with it get no response, since the server correctly has nothing to say about a dead character. Resolve `getPlayer()` fresh at the moment of use (click time, panel-open time), not once at construction. `Events.OnCreatePlayer` is the right place to recreate any toolbar button/UI element that needs to exist for the new character, and `Events.OnPlayerDeath` is a good place to force-close windows bound to the old one.

### Don't trust a vanilla UI element's layout-dependent getter as "settled" the instant an event fires

Anchoring your own element's position off another panel's `getBottom()`/`getHeight()` at the exact moment a creation event fires assumes that panel's own layout has already fully resolved. On a dedicated server specifically (not necessarily reproducible on local host), that isn't guaranteed — whichever of two competing buttons computed its position first could lock in a stale/partial height for the rest of the session, with no error. **Fix:** poll and self-correct (e.g. every ~2 seconds via a throttled `Events.OnPlayerUpdate`) instead of trusting a one-shot snapshot at creation. If two of your own elements need to agree on layout, anchor the second directly off the first's own live position/getter rather than having both independently re-derive the same offset math — that makes them mechanically incapable of disagreeing.

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

## Picker & Display-Name UX Gotchas

### Any admin-facing picker built from `getDisplayName()` alone is fragile once more than one mod is loaded

Many distinct items resolve to the exact same display string — vanilla vehicle parts are a good example: `Base.CarBattery1`/`2`/`3` (Standard/Heavy-Duty/Sports variants) all translate to plain "Car Battery" (confirmed via `shared/Translate/EN/ItemName.json`), and this gets worse across mods (several Authentic Z recipes are ALL literally named "Convert into Wearable"). A picker list showing only `getDisplayName()` makes duplicate-looking rows genuinely unpickable-by-label, even though the underlying fullType/recipe-name payload was always correct.

**Fix:** always show the real internal identifier alongside the friendly name in any admin-facing picker row, not just on hover/request — it's usually also the exact string the admin needs elsewhere (a CRAFT line, a recipe name):

```lua
local label = item:getDisplayName() .. "  (" .. item:getFullName() .. ")"
```

### Fixing only the picker's label is cosmetic if the stored value is independently re-resolved

A quest/tier's stored `displayName` is typically computed **again**, server-side, at creation time by calling `scriptItem:getDisplayName()` fresh — it does not inherit whatever string the picker showed. Adding a disambiguating suffix to the picker's row label alone does nothing for the persisted value. Trace where a display string is actually **persisted** (usually a shared helper function called at creation, not the picker itself) and fix the suffix there; then simplify the picker to just call that same shared helper instead of concatenating its own separate suffix (having both silently double-append or, worse, mask that the shared helper wasn't updated at all).

### A generic `DisplayCategory`/category tag is often the predicate you're looking for, even if it's not obvious at first

"No predicate found" during initial research doesn't mean one doesn't exist. Two confirmed examples:
- Every vehicle part/mechanic item in vanilla declares `DisplayCategory = VehicleMaintenance` in its script (~106 items) — `scriptItem:getDisplayCategory()` is a real, already-used Lua API (vanilla itself compares it against strings elsewhere), giving a clean filter predicate for "is this a vehicle part" with no per-item lookup table needed.
- Real cooking recipes (Stew, Pizza, Guacamole, ...) are ordinary `craftRecipe` entries tagged `category = Cooking`, not a separate system — `recipe:getCategory() == "Cooking"` is a clean filter, discovered only by reading a mature third-party mod's recipe files rather than grepping vanilla's own recipe *names* (which don't share a common keyword the way the category field does).

**How to apply:** before shipping an unfiltered "any item"/"any recipe" picker as the permanent answer, grep the relevant script files for a shared category/tag field — search for the *concept* (a script field these items share), not specific item/recipe *names*, and check whether a similar, mature third-party mod has already solved the identical filtering problem.

### When a value is confirmed unreachable via any live Lua getter, check whether it's actually static per-type data you can extract once

Some per-item classification (e.g. a vehicle part's Standard/Heavy-Duty/Sports tier) has no getter reachable from Lua at all — confirmed by exhaustively enumerating every `get`/`is`/`set` method compiled into the relevant Java class (`unzip -p projectzomboid.jar '*.class' | strings`, a full symbol list, not a keyword grep) and finding the underlying field (`vehicleType`) has no accessor, private-only. But the value itself was static data baked into vanilla's own script files (`VehicleType = N` on ~91 items in `media/scripts/generated/items/{drainable,normal}.txt`), not runtime state. Extracting it once and shipping it as a hardcoded lookup table in your own mod is a legitimate, robust fix in this situation — not a hack — since it can't drift out of sync with base-game files you haven't updated for (it would only break on a game update that also touches those specific items, which you'd be re-verifying against anyway).

**Caution on evidence quality:** a bare string found in a compiled class's constant pool (e.g. a translation key like `"IGUI_VehicleType_"`) is much weaker evidence than an actual method signature — it tells you a name/concept exists *somewhere*, not that it's callable the way you're assuming, or on the object you're assuming. Two different plausible-looking guesses built from string search alone (`scriptItem:getVehicleType()`, then constructing a real instance via `instanceItem(fullType)` and trying again) both tested as complete no-ops in-game. Don't treat a decompiled string as confirmed until tested live; a full method-list enumeration (not just a keyword hit) is strong enough to trust a *negative* result even without a real decompiler (`javap`/`javac` may not be present in the game's bundled JRE — check `--list-modules` for `jdk.compiler` before assuming you can use one).

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

## Boot-Once & Cached Server State

### Some server-authoritative systems compute their data exactly once at boot, and never again for that process's lifetime

Confirmed the hard way with MP spawn-point selection: the dedicated server's list of valid spawn regions (backed by `zombie.iso.SpawnPoints`, a singleton whose complete method list is `init()`/`initServer1()`/`initServer2()`/`initSinglePlayer()` — every method named `init*`, nothing named `reload`/`refresh`/`update` anywhere in the class, confirmed via decompiled-bytecode method enumeration) is genuinely final for the whole session once computed at boot. This is **not** a ModData-load-ordering race and **not** fixable by switching storage mechanism — a plain file (no ModData/load-order dependency at all) reloaded live mid-session still didn't get picked up; only data present on disk **before the server process started** became a valid, honored destination, confirmed by a live add during a running session showing up immediately in the client-side picker (rebuilt fresh from local Lua state — no such restriction there) while still failing the actual server-side spawn-validation check until a full restart.

**How to apply:** for anything that needs to be a validated destination/target passing through server-side validation logic (not just displayed in a client UI), don't promise true zero-restart live functionality — the data must exist on disk before that server process boots. Surface this plainly to the end user (a tooltip/description), don't bury it in code comments only. If you need the live, no-restart version of the feature anyway, the workaround that actually worked: don't fight the server's validation at all — let a live-added point be handled through a small custom **client-side correction** instead. Capture which "region" was actually selected at the exact moment of the relevant decision (a menu-driven fresh join and an in-place death/respawn typically use two entirely separate code paths — verify both if the feature needs to work on both), and if it's a custom point your mod added, apply a client-side `player:teleportTo(x, y, z)` (the same primitive vanilla's own debug teleport tool uses) inside `Events.OnCreatePlayer`, instead of trying to make it a real spawn-region entry.

### A table an engine getter hands you is not necessarily freshly constructed on every call

Once a mod-added entry becomes part of the server's own boot-time snapshot, the underlying table a vanilla "getter" function returns can start **persisting mutations across repeated calls** within the same client session, rather than being rebuilt fresh each time. A naive "does this already exist? if so, rename to disambiguate" check run against that same mutable table will find its own previous insertion still sitting there on the next call and misread it as a genuine new collision, re-inserting a disambiguated duplicate every single call thereafter.

**How to apply:** never assume an engine-provided "getter" returns a fresh container just because it looks like a plain accessor. If you need to track "have I already processed this," don't infer it from re-inspecting the shared table — track your own bookkeeping (a `managedNames` set covering only *this specific processing pass*) instead, and prefer a **reconcile-in-place** strategy (find-or-create each desired entry, only remove entries you previously added that are no longer desired) over strip-and-rebuild-from-scratch, since strip-and-rebuild can wipe legitimate data that arrived from elsewhere (see next point).

### Never use a field on the data itself as a "have I processed this" marker, if that data might be serialized/networked

A first attempt at the disambiguation bug above tagged every region a mod's wrapper created with a marker field (`region.myModGenerated = true`), stripped anything carrying it at the start of every call, then rebuilt. This looked correct but broke a fresh client's very first render: the SERVER embeds the identical enrichment at its own boot (since that data was already legitimate, on disk before that boot), so the marker field gets serialized and sent to clients as part of the region payload — meaning a freshly-connecting client's first-ever received copy of that region already arrives **pre-tagged from the network**, before the client's own logic has run at all. Stripping it at that point removed data with nothing yet available to replace it.

**How to apply:** never store a state-tracking marker as a field on data that might round-trip through serialization (network sync, save data) — anything embedded in the payload itself can leak in from the *other side* and be misread as your own prior work. Keep such bookkeeping in a variable/table entirely separate from the data being tracked.

### When deciding "is this a genuine collision," use the narrowest signal available — same-call-only, not cross-call state

The eventual correct fix for the disambiguation bug above: only treat two names as colliding if both were proposed *within the same processing pass* (e.g. two of an admin's own entries literally sharing a name in the current desired-state computation) — never by checking "does this already exist in the shared table" in any form, since a separate find-or-update-in-place step already handles "this name already exists, whatever its origin" correctly by refreshing it rather than duplicating. General shape: prefer the narrowest possible signal (only things computed fresh *this call*) over anything relying on cross-call bookkeeping, since cross-call state is never guaranteed to be populated yet on any given call — especially the very first one, which will always look like "not yet managed" no matter what you compare against.

### After deleting/renaming a local helper function, grep the whole file for the old name

`luac -p` (parse-only syntax check) does **not** catch a stray call to a deleted/renamed local helper — a reference to an undefined global/upvalue is syntactically valid Lua and only fails when actually invoked (`Object tried to call nil`). Passing `luac -p` is not sufficient evidence that every call site was updated after a refactor; grep the file for the old name too.

---

## Miscellaneous Engine Behavior

### Floating world labels — rebuild every 500ms

Floating labels attached to world objects (via `IsoCell.setLabel()` or similar) get wiped when the player opens and closes the Escape menu. If you want persistent floating labels, rebuild them on a timer (every 500ms works well) via `Events.OnTick` or similar, not just once on chunk load.

### `getText(key, ...)` has an undocumented substitution-argument limit

Confirmed via a live crash (`java.lang.RuntimeException: No implementation found for function: getText(...)` at `MultiLuaJavaInvoker.call`, in the **client's** `console.txt` — this is client-side UI text building, not a server-log error) when a call grew to 6-8 total arguments (key + params) as a feature accumulated more fields over time. The highest-arity `getText` call confirmed working elsewhere is 5 total args (4 substitution params).

**How to apply:** never let a `getText` call's substitution-argument count grow past ~4 as you add fields to a message over successive features. Once you're tempted to add a 5th, pre-format the variable part into one string yourself via Lua's `string.format` (note: `%d`/`%s`, not `getText`'s own `%1`/`%2` syntax) against a new translation key, and pass that single formatted string as one `getText` param instead:

```lua
local summary = string.format("%d feats, %d quests, %d tiers", featCount, questCount, tierCount)
getText("IGUI_MyMod_SummaryFormat", summary)
```

### A field added to a stored table doesn't need a migration if every read site defaults it

Adding a new optional field (e.g. a `difficulty` tag) to a data structure that already has live saved instances doesn't require a backfill/migration pass if every read site tolerates its absence via a default:

```lua
function BattlePass.getQuestDifficulty(quest)
    return quest.difficulty or "Medium"
end
```

Entries created before the field existed just permanently read as the default — no write-back pass needed anywhere, and no risk of a migration bug. Any external text-import format that also encodes the new field should make it an **optional trailing token** (defaulting the same way when omitted) so every previously-written import stays valid unchanged.

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

### There is no built-in JSON library, and no master list of "every known username"

PZ's Lua sandbox exposes no JSON encode/decode globals. If you need to export mod state as JSON for external tooling, you have to hand-roll an encoder — straightforward (array-vs-object detected via a gapless 1..n integer-key check, `%c` control-char escaping for strings) but **verify it independently before wiring it in**: a standalone `lua5.1` smoke test of just the encoder function, piped through a real JSON parser (e.g. Python's `json.loads`), catches encoding bugs far faster than testing through the full in-game export path.

Separately, if your mod tracks several independent per-player `ModData` stores (a token ledger, several lifetime-stat trackers, quest progress, etc.), there is usually **no single existing list of every player who has ever interacted with the mod** — an admin panel's balance list, for instance, is typically just `pairs(balances)`, which misses anyone whose only activity landed in a store that never touched the balance table. Build the full username set as the **union of keys across every store** you maintain, not just the most "obvious" one.

### Retiring an old design: check whether the underlying action is genuinely repeatable before reusing a baseline-subtraction pattern

A "quest" system built around a repeatable lifetime counter with baseline-subtraction (store a per-player baseline, count new progress as `current - baseline`, reset the baseline once the quest is claimed) breaks down for actions that are structurally a lifetime one-shot (e.g. "read this specific book for the first time" — the underlying tracker is capped at 1 forever per player+item, since the game itself only grants a first-time bonus once). A returning player whose tracker is already pinned at 1 can never show new progress again once a future season reuses the same target, permanently stuck at "0 of 1" even though they clearly did the thing.

**How to apply:** before modeling any new reward around the existing baseline-subtraction quest pattern, confirm the underlying action can actually happen more than once per player. If it can't, a flat always-on reward (grant once, the first time the condition becomes true, independent of any season/quest scoping) or a one-time Feat is the right shape instead — not a forced fit into the repeatable-quest machinery.

---

## Content Authoring & Verification

Lessons from authoring large amounts of game content (item/recipe references, numeric balance tables) by hand or via an AI research pass.

### Run a scripted existence check over final content IDs — don't trust a research pass's "verified" claim

When authoring dozens of item fullTypes / recipe names by hand (or having an agent research them), a wrong entry can slip through even after being explicitly marked "verified" by that research. A small Python/shell script that extracts every fullType/recipe name from your final content tables and greps each one against the real vanilla script files (`item <name>` / `craftRecipe <name>` in `media/scripts/`) catches this mechanically — confirmed catching a real bad ID (a plausible-looking but nonexistent item name) that an agent's manual research pass had wrongly signed off on. Silent failure mode if you skip this: most seed/loot loops skip a missing item without erroring, so a bad ID just silently drops that quest/tier reward at seed time rather than crashing anything you'd notice.

### Cross-check numeric thresholds against real game data, don't pick them from intuition

Balance numbers (weight thresholds, kill counts, escalating reward curves) that interact with your mod's own *other* default content need to be checked against the real underlying data, not guessed. Confirmed failure: a fish-weight achievement ladder's first tier (3 lbs) was picked without checking real per-species max weights, and turned out to sit *above* the actual maximum weight obtainable from the mod's own suggested starter fishing targets — making tier 1 uncompletable via the content the mod itself pointed players toward. Fix was to grep the real values (vanilla's `fishing_properties.lua` in this case) and recalibrate the whole ladder against the real roster.

### Before proposing a Craft Quest (or similar) for content in a companion mod, check that mod's own recipe file — a real item existing isn't enough

A companion mod having an item worth rewarding doesn't guarantee a clean, uniquely-matchable recipe exists for it. Confirmed case: one companion mod's clothing recipes were almost all named identically (`Convert into Wearable`, reused across dozens of unrelated items) — a Craft Quest keyed on that recipe name would fire for the wrong item entirely. Always grep the actual `recipe <Name>` lines in the companion mod's own scripts before proposing a craft-based reward for its content; a loot-only item (zero matching recipes) is a legitimate reason to leave it out of that reward type rather than force a match.

### When a new "kind" is added to an N-kind system, grep every call site that destructures all N kinds together

Summary formatters, admin inspector windows, and similar aggregating call sites are exactly where a newly-added kind (a 6th quest type, say) quietly gets forgotten — these are copy-paste-shaped by nature, and older instances of the identical gap (an earlier kind that was *also* never added to one of these call sites) tend to already be sitting there unnoticed until you go looking. When you add kind N+1 to a system, search for every place that already handles kinds 1..N together and update them all in the same pass, rather than trusting each site was already complete.

### Some state genuinely can't be read for an offline player — mark it, don't guess at it

If a feature reads live character state (e.g. `player:getZombieKills()`) rather than a persistent tracker, that data is simply unavailable for a player who isn't currently online — there's no way to fetch it. Rather than silently omitting it or guessing, add an explicit flag (e.g. `requiresOnline = true` on that field's definition) and surface it as `unavailable = true` in any admin-facing "inspect this player" view when the target isn't connected. Every persistent-tracker-backed field, by contrast, should read identically whether the target is online or not — if it doesn't, that's a bug in the read path, not a fundamental limitation.

---

## Reference

- Local docs: `~/Zomboid/mod_docs/` — `Migration Guide.pdf`, API docs, PZwiki HTML snapshots (path may vary; see [Verifying APIs Against Real Game Files](#verifying-apis-against-real-game-files-not-just-docs) if you don't have a local copy)
- Community guides: https://github.com/demiurgeQuantified/PZModdingGuides — general Lua/engine mechanics guides (modules pattern, script function signatures, load order, sandbox options, remote debugging). Does not cover animation, traits/perks/occupations, or item/recipe script syntax.
- Unofficial decompiled Java docs: https://github.com/demiurgeQuantified/ProjectZomboidJavaDocs — browsable via its GitHub Pages `member-search-index.js` (a flat list of every class/method; grep it directly rather than relying on the site's client-side search, which a non-interactive fetch can't execute). The strongest available ground truth for "does this native method actually exist / is this engine behavior a hard constraint" questions that can't be answered from Lua alone — used to definitively confirm the boot-once spawn-region constraint in [Boot-Once & Cached Server State](#boot-once--cached-server-state).
- Example mods this document was drawn from: PlayerShops (safehouse kiosk economy), BattlePass (seasonal token progression, quests, feats), and DynamicSpawnPoints (custom MP spawn regions) — all built iteratively while compiling these lessons
- PlayerShops Workshop item: https://steamcommunity.com/sharedfiles/filedetails/?id=3749824460
- BattlePass Workshop item: https://steamcommunity.com/sharedfiles/filedetails/?id=3756808742 (currently unlisted/private)
