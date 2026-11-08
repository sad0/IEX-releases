# Lua API for modders

IEX gives mods the EEex Lua API under the same names, plus a few `IEX_*` functions of its own. This page covers:

1. [How EEex calls are handled](#how-eeex-calls-are-handled)
2. [Features and detection](#features-and-detection)
3. The `IEX_*` additions: [camera](#camera), [keys and actions](#keys-and-direct-actions),
   [frame rate](#frame-rate), [HD area tiles](#hd-area-tiles), [class spell lists](#class-spell-lists), [kits](#kits),
   [fog of war](#fog-of-war-viewers), [object ids](#object-ids), [Pathfinding Redux](#pathfinding-redux),
   [native hook points](#native-hook-points), [network bridge](#loopback-network-bridge), [utilities](#utilities)

Engine objects (sprites, areas, effects...) are userdata with the same field names as under EEex. Numbers are
integers unless said otherwise. All listeners run under `pcall`: an error is logged to `IEX.log` (repeated errors at
1, 10, 100... occurrences) and never stops the next listener.

Functions not listed here (IEX's other `IEX_*` globals) are building blocks of the EEex layer and may change between
versions: do not call them from mods.

## How EEex calls are handled

| EEex API | Under IEX |
|---|---|
| `EEex_*` functions, listeners, options, menus, resources, script actions / triggers / objects | Same names, same arguments and results, reimplemented by IEX (610 names, listed in `IEX/IEX_API.txt`). A mod's code runs unchanged on Windows and Linux. |
| Engine fields (`sprite.m_baseStats...`), engine methods (`area:CheckLOS(...)`) | Same names, resolved per game build from IEX's engine descriptions, so they work on Linux too. |
| `EEex_UDToPtr(ud)` / `EEex_PtrToUD(ptr, type)` | Provided. IEX's own equivalents: `IEX_Address(ud)` / `IEX_ToObject(address, typeName)`. |
| `EEex_Read*` / `EEex_Write*`, `EEex_Malloc` / `EEex_Free` | Provided. Hard-coded offsets are build-specific: a Windows offset is wrong on Linux. Use field names instead. On Linux, reading an unmapped address raises a Lua error instead of crashing; near-null addresses raise an error on both systems. |
| x64 assembly / JIT API (`EEex_HookBeforeCallWithLabels`, `EEex_JITAt`, `EEex_GenLuaCall`...) | Inert stand-ins: nothing is assembled or written. A request whose patch address IEX knows switches on the matching [native hook point](#native-hook-points) and binds the Lua callback the request names. Unknown requests are logged and recorded in `IEX/missing_api.txt`. `EEex_JITNear` stays undefined. |
| `EEex_Label(name)` | Only the labels IEX knows (a few used by mods to test the game version). |
| Opcodes 400-403, 409 | Provided. 404-408: not yet. |
| `EEex.ini` modules | Empty Container, Time Step, Timer, Effect Menu, Attack Info. `EEex_Modules.<name>` is true when IEX has the module and `EEex.ini` enables it. |
| Any other `EEex_*` name EEex defines | A stand-in that can be called, indexed and iterated and does nothing; recorded in `IEX.log` and `IEX/missing_api.txt`. Names EEex does not define either are `nil`, as under EEex. |

## Features and detection

```lua
if IEX_Features then
    -- running on IEX
    IEX_Log('platform: ' .. IEX_Features.platform)   -- 'win64' or 'linux64'
end
```

| Global | Meaning |
|---|---|
| `IEX_Features` | Table, exists only under IEX. Set before any `M_*.lua` runs. |
| `IEX_Features.platform` | `'win64'` / `'linux64'`. |
| `IEX_Features.nativeHooks.<point>` | `true` when that [native hook point](#native-hook-points) exists on this game build. |
| `IEX_Features.partyVision` | Unlimited fog-of-war viewers active. |
| `IEX_Features.objectIdRecycling` | Object id recycling active. |
| `IEX_Features.frameInterpolation` | Frame interpolation available. |
| `IEX_Features.pathfindingRedux` | Native Pathfinding Redux policy on. |
| `IEX_Features.kitLimit`, `IEX_KitLimit()` | [Unlimited kits](#kits) on (call sites patched). |
| `IEX_Features.engineFixes`, `IEX_EngineFixes()` | `{ name = installed }` for each [engine fix](FEATURES.md#engine-fixes). |
| `IEX_Features.stickHooks` | Number of StickLuaFunctions hooks IEX runs natively on this build. |
| `IEX_Log(text)` | Writes a line to `IEX.log`. |

Install-time detection: `MOD_IS_INSTALLED ~IEX/IEX.TP2~ ~0~`; version in `IEX/VERSION`; EEex names IEX defines in
`IEX/IEX_API.txt` (one per line).

## Camera

No EEex equivalent.

| Function | Behaviour |
|---|---|
| `IEX_Camera_AddRenderListener(fn)` | `fn(area)` every frame just before the area is drawn, on every screen that shows it; a camera moved there is drawn in the same frame. |
| `IEX_Camera_GetView(area)` -> `x, y, w, h` | Top-left world point and world size of the view. |
| `IEX_Camera_SetView(area, x, y)` / `IEX_Camera_CenterOn(area, x, y)` | Moves the view; the engine clamps it to the area. |
| `IEX_Camera_GetZoom(area)` -> `zoom, enabled` | 1 = none, smaller = closer (world pixels per screen pixel). |
| `IEX_Camera_SetZoom(area, zoom)` | Clamped to the limits, keeps the view centre; ignored when zoom is disabled. |
| `IEX_Camera_ZoomLimits([closest, farthest])` -> `closest, farthest` | Engine-wide zoom range (defaults 0.5 / 1.4; the game recomputes the closest one on area load). |

```lua
IEX_Camera_AddRenderListener(function(area)
    local sprite = EEex_Sprite_GetSelected()
    if sprite then IEX_Camera_CenterOn(area, sprite.m_pos.x, sprite.m_pos.y) end
end)
```

## Keys and direct actions

| Function | Behaviour |
|---|---|
| `IEX_ScancodeIsDown(scancode)` -> bool | Physical key held (SDL scancode, layout independent: 26/4/22/7 = WASD on QWERTY = ZQSD on AZERTY). |
| `IEX_ScancodeFromKey(keycode)` -> scancode | Physical key that types `keycode` in the current layout. |
| `EEex_Key_Hook_AfterEventsPoll(event)` | The event also has `event.key.repeat` and `event.key.keysym.scancode`. |
| `IEX_SpriteSetAction(sprite, actionId, x, y)` | Makes a point action (ACTION.IDS number: 23 MoveToPoint, 0 NoAction) the current action now, replacing the one in progress (queue kept). No voice, no destination marker. |

Sprite fields added: `m_nDirection` / `m_nNewDirection` (facing 0-15: 0 south, 4 west, 8 north, 12 east).

## Frame rate

The options screen and `Baldur.lua` settings ([FEATURES.md](FEATURES.md#smooth-frame-rate-and-frame-interpolation))
use these.

| Function | Behaviour |
|---|---|
| `IEX_SetFrameRate(n [, cap])` -> applied rate | Draw `n` frames per second while the logic keeps its own rate. 0 = engine default (one frame per update); -1 = refresh rate of the window's display (followed when the window moves), at most `cap`. |
| `IEX_GetFrameRate()` -> `requested, ticksPerSecond, inUse, modFloor, framesSkipped, swapInterval` | Current state. |
| `IEX_GetRenderRate()` -> `frames, ticks` | Frames drawn and logic updates run during the last second. |
| `IEX_SetFrameRateFloor(n)` | Draw at least `n` frames per second (0 = none). |
| `IEX_SetVSync(v)` | -1 engine setting, 0 off, 1 on. |
| `IEX_SetBusyWait(ms)` | Last `ms` (0-10) of each wait spun instead of slept. |
| `IEX_SetFrameInterpolation(on)` / `IEX_GetFrameInterpolation()` -> `setting, active, framesInterpolated, movedLastFrame, snaps, mismatches` | Frame interpolation. |
| `IEX_SetKeepGameSpeed(on)` / `IEX_GetKeepGameSpeed()` | Keep Game Speed: mods' writes to `EEex_CChitin.TIMER_UPDATES_PER_SECOND` become frame rate requests instead of speeding up the game. |
| `IEX_GameSpeedWriters` | List of Lua chunk names whose writes do change the game speed (default `{ 'BfBotFst' }`, BuffBot's fast mode). |
| `IEX_ApplyFrameSettings()` | Re-reads the `Baldur.lua` frame settings. |

## HD area tiles

See [HD_PACKS.md](HD_PACKS.md).

| Global | Meaning |
|---|---|
| `IEX_HdTiles_Status` | `'off'`, `'off (Baldur.lua)'`, `'on'`, or why the switch could not be installed. |
| `IEX_HdTiles_Stats()` | 14 values: HD tile draws, other tile draws (since start), tile sets seen, HD page filter writes, HD pages mipmapped, mip levels uploaded, HD seam shader active (bool), HD pages left without mips, worker build ms, render-thread attach ms, mip chains attached, mip chains dropped, pages with stored mips, longest render-thread time for one HD page (ms). |

```lua
local hd, plain = IEX_HdTiles_Stats()
Infinity_DisplayString('HD tile draws: ' .. hd .. ', 1x: ' .. plain)
```

## Class spell lists

The wizard and priest class lists (chargen, level-up, sorcerer / bard picks, dual-class, priests' automatic spells)
continue past the engine's slots (SPWI/SPPR 01-50, or 01-99 with SpellSlots) with IEX lists of any resrefs.
Entries keep the engine's checks (file present, level matches, `HIDESPL.2DA`, class / kit / school / alignment
flags). The first entry turns the native switch `spellLists` on.

| Function | Behaviour |
|---|---|
| `IEX_SpellList_Add(type, level, resref)` -> `true` / `false` (already listed) | `type` 1 priest (levels 1-7), 2 wizard (1-9). Turns the switch on. |
| `IEX_SpellList_Remove(type, level, resref)` -> `true` if removed | |
| `IEX_SpellList_Get(type, level)` -> `{ resref, ... }` | List order. |
| `IEX_SpellList_Count(type, level)` -> `n` | |
| `IEX_SpellList_Clear([type [, level]])` | |
| `IEX_SpellLists_Load2DA([resref])` -> rows added | `IEXSPLST.2DA` by default; IEX reads it at `GameStateInitialized`. |
| `IEX_SpellLists_Status` | `'off'`, `'on'`, or why the switch did not install. |

`IEXSPLST.2DA`: `2DA V1.0`, a default line, the header `TYPE LEVEL`, then one row per spell
(`<resref> <type> <level>`, type `1` / `PRIEST` or `2` / `WIZARD` / `MAGE`), in list order. The IEX installer
creates it empty. From WeiDU:

```
APPEND ~iexsplst.2da~ ~MYSPL01 2 3~
```

From Lua (an `M_*.lua` file):

```lua
if IEX_SpellList_Add then
    IEX_SpellList_Add(2, 3, 'MYSPL01')   -- wizard level 3
end
```

Spells keep their own resref; numbered access (`Spell()` by number, `ADD_SPELL`, IDS names) stays limited to 99 per
level.

## Kits

| Function / flag | Behaviour |
|---|---|
| `IEX_Features.kitLimit` | Engine call sites patched (2) when [unlimited kits](FEATURES.md#unlimited-kits) is on, 0 when off. |
| `IEX_KitLimit()` -> `{installed, sites, engineSlots, maxRow, tables, why}` | `engineSlots` 256; `maxRow` 16383 (256 when off); `tables` = CLAB tables IEX loaded for rows past 256 so far; `why` = reason when off. |

Nothing to call: kits on rows past 256 work like the others (`Kit()`, `AddKit`, `EEex_Sprite_GetStat(sprite, 152)`).

## Fog-of-war viewers

| Function / flag | Behaviour |
|---|---|
| `IEX_Features.partyVision` | `true` when active. Big-party mods can test it and skip their own visibility patches. `false` when another patch already changed the engine there (IEX then changes nothing and logs it), or with `IEX_NO_PARTY_VISION=1`. |
| `IEX_VisionInfo([map])` -> table or list | Counters of one visibility map, or of every map IEX knows: `viewers`, `tableSlots` (of the engine's 15), `overflow`, `detached`, `stamps`, `visibleTiles`, `captures`, `width`, `height`. |
| `IEX_VisionViewer(map, id)` -> `onMap, slot, stamps, tiles` | `slot` 0-14 = engine slot, -1 = IEX overflow viewer; `false` if `id` never was a viewer there. |
| `IEX_VisionTile(map, x, y)` -> `value, viewers` | Tile under area pixel (x, y): engine value (bit 15 explored, bits 0-14 visible) and the number of viewers that see it. |
| `IEX_VisionCheck()` -> `mismatches, maps` | Test: tiles whose visible bit disagrees with the viewer counts (0 expected). |

IEX uses only bit 0 of a tile's visible bits; bits 1-14 set by the engine or a mod are kept.

## Object ids

| Function / flag | Behaviour |
|---|---|
| `IEX_Features.objectIdRecycling` | `true` when active (single player; off with `IEX_OBJECT_IDS=0`). |
| `IEX_ObjectIdInfo()` -> table | `active`, `adds`, `reused`, `wraps`, `nextKey`, `maxIndex`, `nextObjectId`, `deletedHead`. |

## Pathfinding Redux

| Function / flag | Behaviour |
|---|---|
| `IEX_Features.pathfindingRedux` | `true` while the native allied pass-through policy is on. |
| `IEX_PathfindingRedux([{ enabled = b, combatOnly = b, reset = b }])` -> table | Switches for the session; returns `installed`, `enabled`, `combatOnly`, `combatBlocked`, `modInstalled` and timing statistics. |
| `IEX_PathfindingCheck(spritePtr)` -> `mismatches, cells, first` | Test: search-map cells of the sprite's area whose creature counts differ from the footprints (0 expected). |

When the policy is on, IEX sets `MRIP_DispatchLoaded = true` and `MRIP_CompatibilityStatus = { supported = true,
provider = 'IEX native' }`, so the mod's own loader steps aside.

## Native hook points

Engine moments that mods used to patch in Windows machine code, implemented once in C++ for Windows and Linux,
with one Lua interface. Hooks are installed on demand, after checking that the game's code is still unchanged
there.

| Function | Behaviour |
|---|---|
| `IEX_NativeHook_AddListener(point, fn)` -> `true` / `false, reason` | Installs the point if needed and adds a listener. |
| `IEX_NativeHook_RemoveListener(point, fn)` -> bool | |
| `IEX_NativeHook_Enable(point [, a, b])` -> `true` / `false, reason` | Installs a switch point and turns it on. |
| `IEX_NativeHook_Install(point [, a, b])` -> `true` / `false, reason` | Installs the engine side only. |
| `IEX_NativeHook_Installed(point)` -> bool, `IEX_NativeHook_Sites(point)` -> n | State, number of patched sites. |
| `IEX_NativeHook_Points()` -> `{ [name] = { category, kind, result, moment, available, installed, active, sites, why } }` | Every point. |

Listener results: **value** points get the current value as their last argument and may return a new number (or
`nil` to keep it; the next listener sees the new value). **anyTrue** points: every listener runs; one `true` = yes.
Other listeners return nothing.

| Point | Kind | Listener / arguments |
|---|---|---|
| `hitRoll` | value | `(attacker, target, roll 0..19)`: the d20 attack roll |
| `toHitTotal` | value | `(attacker, target, total)`: right-hand to-hit total |
| `critThreshold` | value | `(attacker, target, value)`: critical threshold |
| `critBlocked` | switch | a critical stopped by a helmet sets the critical flag to 2 instead of 0 |
| `attackResult` | listener | `(attacker, target, hit 0/1)` |
| `damageRoll` | value | `(attacker, target, damage)`: weapon dice + damage bonus |
| `critMultiplier` | value | `(attacker, target, multiplier)` |
| `attackIndex` | listener | `(attacker, target, index)`: attack frame reached |
| `attackCancel` | anyTrue | `(attacker, target)` -> true cancels the attack |
| `hideInSight` | anyTrue | `(spriteId)` -> true allows hiding while seen |
| `casterSchoolSpell` / `casterSchoolItem` | value | `(caster, effect, school)`: caster school for the save penalty |
| `targetSchool` | value | `(target, effect, school)`: target school for the save bonus |
| `actionButtonL` / `actionButtonR` | listener | `(index 0..12)`: action bar button pressed |
| `classDone` | listener | `(sprite)`: chargen class step completed |
| `profMaxUI` / `profMax` | value | `(proficiency, max)`: proficiency maximum shown / reachable in chargen |
| `keepAmmo` | anyTrue | `(sprite, item)` -> true keeps the charge / ammo count |
| `groundMoveClick` | listener | `(area, x, y)`: move order on the ground |
| `deathDrop` | anyTrue | `(sprite)` -> true keeps the inventory on the corpse |
| `respecGear` | switch | `(flagAddress)`: no starting gold / weapon while the byte is not 0 |
| `campMarker` | switch | `(flagAddress)`: no party marker on the world map while the byte is not 0 |
| `campSameArea` | listener | `()`: the current area clicked on the world map |
| `groundContainer` | anyTrue | `(container, sprite)` -> true lists a non-pile container in the ground bar |
| `smallFootprint` | switch | `(mode)`: bumpable creatures take 1 search square (1) or none (0) |
| `containerMarker` | switch | `(colorAddress, marker)`: permanent outline color of containers with `m_drawPoly == marker` |
| `lastingModes` | switch | modal states are not cancelled by actions, buttons, a failed stealth re-roll or moving |
| `findTrapsButton` | anyTrue | `(sprite)` -> true = handled (the engine's toggle is skipped) |
| `stealthReveals` | switch | talking, containers, items and friendly spells do not reveal a stealthed creature |
| `spellSlots99` | switch | class spell lists read slots 01-99 (the SpellSlots mod turns it on) |
| `spellLists` | switch | class spell lists continue with the IEX lists ([above](#class-spell-lists)) |
| `hdTiles` | switch | HD area tiles ([above](#hd-area-tiles)) |
| `partyVision`, `alliedPassThrough`, `emptyContainerOutline` | feature | installed at start, reported only |

```lua
-- Endless ammo for the party
IEX_NativeHook_AddListener('keepAmmo', function(sprite, item)
    return sprite.m_typeAI.m_EnemyAlly == 2
end)
```

## Loopback network bridge

For external tools (e.g. a separate renderer) talking to the game on the same machine.

| Function | Behaviour |
|---|---|
| `IEX_NetListen(port)` -> bool | Listens on 127.0.0.1:`port` only (never other interfaces). Once per session. |
| `IEX_NetPoll()` -> `connected, { lines }` | Accepts one client, sends queued output, returns the complete lines received. Call every frame. |
| `IEX_NetSend(text)` -> bool | Queues text for the client (add the newline yourself; non-blocking; the client is dropped past 8 MB of backlog). False when no client. |

## Utilities

| Function | Behaviour |
|---|---|
| `IEX_Log(text)` | A line in `IEX.log`. |
| `IEX_ReadINIFile(file, section, key)` -> string or nil | A value from an INI file in the game folder (relative path, no `..`). |
| `IEX_WriteINIFile(file, section, key, value)` -> bool | Writes it. |
| `IEX_Address(object)` -> integer, `IEX_ToObject(address, typeName)` -> object or nil, `IEX_TypeName(object)` -> string or nil | Engine object primitives (what `EEex_UDToPtr` / `EEex_PtrToUD` use). |
| `IEX_TicksPerSecond([n])` -> n | Logic updates per second (the game speed); sets it when `n` is given. |
