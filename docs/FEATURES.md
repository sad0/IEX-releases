# IEX features

Everything IEX does that a player can see or set, with its setting. All features work on Windows and Linux
(BG2EE / EET 2.7.3) unless a line says otherwise.

Settings live in `Baldur.lua` (the game's settings file, see [INSTALL.md](INSTALL.md#where-the-logs-are)) as lines
like `SetPrivateProfileString('IEX','HD Area Tiles','0')`. Edit it while the game is closed. Many settings are also
in the [IEX options screen](#iex-options-screen). The full list is at the end: [settings reference](#settings-reference).

Status marks: **WIP** = in progress, not finished or not in this release.

## EEex compatibility

The main feature: mods written for EEex run on IEX unchanged, on Windows and Linux. What is covered and what is not:
[MOD_COMPAT.md](MOD_COMPAT.md). For modders: [LUA_API.md](LUA_API.md).

- EEex calls IEX does not have never stop the game with a Lua error: the mod gets a stand-in that does nothing, and
  the name goes to `IEX/missing_api.txt`.
- IEX has no x64 assembly / JIT API. Engine patches that mods write in machine code are replaced by IEX's own native
  hooks where IEX has one (BG Pathfinding Redux, StickLuaFunctions, SadoEE); on Linux this is the only way they can
  work at all.

## Smooth frame rate and frame interpolation

The game draws frames at the monitor's refresh rate (60 FPS on a 60 Hz monitor, 144 on a 144 Hz one) while the
game logic keeps its normal speed (30 updates per second by default).

- **Frame interpolation**: creatures and projectiles are drawn between their positions in the extra frames, so they
  glide instead of stepping 15 times per second. The drawn picture is one logic step behind (33-66 ms). Only drawing
  changes: AI, pathfinding and saves never see the in-between positions. Active only while the frame rate is above
  the logic rate.
- **Keep Game Speed**: some mods raise the game's update rate to get more frames (Tweaks Anthology "Dynamic FPS",
  60 outside combat), which also makes the whole game run faster. With Keep Game Speed on, they get the frames and
  the game keeps its speed. BuffBot's fast mode still speeds the game up, as intended. Off = as under EEex.
- **AI Speed** is the game speed: updates per second (default 30). 60 runs the game twice as fast. With Keep Game
  Speed on, this setting decides the speed, not the engine's saved 'Maximum Frame Rate'.

| Setting | Default | Options screen |
|---|---|---|
| `'EEex','Uncap FPS'` | 1: frames independent of logic updates. 0: one frame per update (the engine's own behaviour). | Uncap FPS tab |
| `'EEex','VSync Enabled'` | 1: wait for the monitor refresh. 0: as many frames as possible. | yes |
| `'EEex','Uncap FPS Limit Enabled'`, `'EEex','Uncap FPS Limit'` | 0, 144: frame rate cap | yes |
| `'EEex','Uncap FPS Busy Wait Threshold'` | 1: last ms of each wait spun (more precise pacing) | yes |
| `'EEex','Uncap FPS Fullscreen VRR'` | 0. Stored and shown for EEex compatibility; IEX does not use it yet. | yes |
| `'IEX','Frame Interpolation'` | 1 | yes |
| `'IEX','Keep Game Speed'` | 1 | yes |
| `'IEX','AI Speed'` | 30 (1-240) | yes |
| `'IEX','Frame Rate'` | 0 = use the settings above. Any other value overrides them: frames per second, -1 = monitor refresh rate. | yes |

The `EEex` keys are the same keys and defaults as EEex's "Uncap FPS" options, so an existing `Baldur.lua` keeps its
settings.

Known limits: a few effects that the engine counts per drawn frame run at the frame rate (touch-scroll inertia, the
area flash, menus that count frames). EEex's 'Lua GC Steps' and 'Remove Middle Mouse Scroll Multiplier' options are
not implemented.

## HD area tiles

Area graphics (the tiles of the background) drawn at 2x, 4x or 8x resolution, from HD packs installed in
`override/`. Game data stays the same: areas, walls, doors, search maps and positions are unchanged, so HD packs
work with saves and with other mods. Zoom in to see the extra detail.

- OpenGL only (the game's default renderer). With Direct3D 9 (`Baldur.lua` 'Graphics' 'Backend' 1, Windows) HD tiles
  are drawn at normal size and `IEX.log` says so once.
- Areas without an HD pack are drawn exactly as before.
- Smooth filtering at every zoom level (mipmaps), no seams between tiles, works with "Use Nearest Neighbour Scaling".

| Setting | Default |
|---|---|
| `'IEX','HD Area Tiles'` | 1 (on). 0 turns it off; read at game start. Not in the options screen. |

For pack authors (file format, VRAM, how to check it works): [HD_PACKS.md](HD_PACKS.md).

HD creature sprites: being studied, not in this release.

## Unlimited class spell lists

The wizard and priest spell lists players pick from (character creation, level-up, sorcerer and bard picks,
dual-class, the spells priests get automatically) normally stop at slot 50 of each level (`SPWI<level>01-50`,
`SPPR<level>01-50`). With IEX, mods can add any number of spells to these lists, with any resource name.

- Mods add rows to `IEXSPLST.2DA` (the IEX installer creates it empty) or call `IEX_SpellList_Add` from Lua
  ([LUA_API.md](LUA_API.md#class-spell-lists)). The feature (`spellLists`) turns itself on when the first spell is
  added; with an empty list nothing changes.
- Added spells go through the engine's usual checks (file exists, spell level matches, `HIDESPL.2DA`, class / kit /
  school / alignment exclusion flags). The specialist mage bonus slot sees them too.
- A lower `SPELLS.2DA` limit for priests still applies to the engine slots of the automatic priest spells.

Limits: scripts and WeiDU still number spells (`Spell()` by number, `ADD_SPELL`, IDS names): that stays at 99 per
level. Very large lists (tens of thousands) make the character creation spell screen slow. Mods that inject spells
into the chargen lists themselves (ClassSpellTool, SFO's extra spells) may show a spell twice if it is also in an IEX
list; not tested.

Tested on Windows (character creation for every caster class, level-up, dual-class) and Linux (character creation,
priest spells, dual-class).

## Spell slots 51-99

The `SpellSlots` mod, shipped in the IEX archive, makes wizard and priest spells in slots 51-99 (`SPWI<level>51-99`,
`SPPR<level>51-99`) available to players like slots 01-50.

- Install it right after IEX, before spell mods, ClassSpellTool mods and SFO mods.
- It sets the `MAGE` and `PRIEST` rows of `SPELLS.2DA` to 99, hides the engine's own internal spells in 51-99 in
  `HIDESPL.2DA`, and turns on the IEX switch `spellSlots99` at game start. It also works with EEex on Windows.
- `SpellSlots_Status` (Lua) tells what happened.

Limit: WeiDU's `ADD_SPELL` still fills slots 01-50 only.

Tested with IEX on Windows; Linux uses the same native switch.

## Unlimited kits

The engine keeps the class abilities table (CLAB) of only 256 kits: a single-class kit on `KITLIST.2DA` row 257 or
higher crashes the game or gets no or wrong abilities at character creation, level-up, `AddKit` and dual-class.
With IEX, single-class kits work up to row 16383 (`KIT.IDS` 0x7FFF). True multiclass kits never had the limit.

- On by default (`kitLimit`), Windows and Linux. No save format change.
- `Baldur.lua` `SetPrivateProfileString('IEX','Unlimited Kits','0')` turns it off (restart needed; also on the
  Engine Fixes tab of the IEX options screen). Environment variable `IEX_KIT_LIMIT=0` for tests.
- Installing kits past row 256: WeiDU's own `ADD_KIT` refuses them (it expects GemRB or ToBEx, and stops at 1280).
  Kit mods use `IEX/lib/iex_kits.tpa` and Argent77's `ADD_KIT_EX` instead
  ([MOD_COMPAT.md](MOD_COMPAT.md#kit-mods-past-256-kits)). Kit mods that use `ADD_KIT` are not changed and keep
  working on rows up to 256.
- A save with such a kit still loads without IEX, but that character's next level-up, `AddKit` or dual-class has
  the engine bug again: mods using rows past 256 should require IEX.

Tested on Windows and Linux: character creation with kits on rows 257 and 301, level-up, `AddKit` to another kit
and back, save and reload, level-up after reload, dual-class (same result as a kit on row 249); with the feature off,
the same kits crash (row 257, Windows) or get no abilities. Not tested yet: level drain, EET.

## Unlimited fog-of-war viewers

Every party member and every clairvoyance-type viewer clears fog of war and sees creatures, however many there are.
The unmodded engine keeps only 15 viewers per area, and viewers past the 8th clear no fog. This matters for
big-party mods (tested with 17 party members) and fixes areas that never unload with more than 15 members.

- On by default. `IEX_Features.partyVision` is true when it is active.
- Bits of the visibility map that the engine or a mod sets for other uses are left alone.
- Turn it off (for tests): environment variable `IEX_NO_PARTY_VISION=1`.

## Object ids that never run out

The engine gives every game object (creatures, projectiles, effects on the ground...) an id and never reuses them.
After many hours in one session (about 30,000 objects created since the last load) it runs out, prints "Too many
objects!" and the game crashes when the next creature enters an area. IEX reuses free ids, so this never happens.

- On by default, single player only (multiplayer sessions keep the engine's ids).
- Turn it off (for tests): environment variable `IEX_OBJECT_IDS=0`.

## Engine fixes

Fixes for engine bugs that the EE Fixpack "Code fixes" thread on Gibberlings3 lists as fixed by EEex, written by IEX
from the bug descriptions. All are **on by default**, on Windows and Linux. Each fix checks the game's code first; if
the code differs (other build, another patch) that fix is left out and `IEX.log` lists it as not installed.

| Name | Fix |
|---|---|
| `op182EquippedItem` | Opcode 182 (apply EFF if item equipped) applies only when the item really is equipped (it applied almost always, and never for the amulet slot). |
| `weaponSpeed` | The WSPECIAL.2DA SPEED column is applied (High / Grand Mastery speed bonus), and ranged attacks use the launcher's weapon speed instead of the ammo's (always 0). |
| `splprotSigned` | SPLPROT.2DA compares stats as signed numbers (negative resistances no longer count as huge values). |
| `trapLimit` | TRAPLIMT.2DA cap respected exactly (it allowed one trap more). |
| `op33SaveVsDeath` | Opcode 33 with param2 = 3 changes save vs. death, not save vs. wands. |
| `immunityMessages` | Opcode 206 shows any string given in param1; the opcode 232 / 256 "cannot cast multiple instances" message shows. |
| `hpTableModifier` | HP tables: the static modifier is added when a die roll is set (1d6+2 rolled 1d6). |
| `simulacrumLevelDrain` | Opcode 236 (Simulacrum): the level drain uses the average level of dual- and multi-class creatures. |
| `negativeXPFeedback` | `AddXPObject()` with a negative value shows "The Party Has Lost Experience", as `AddExperienceParty`. |
| `xpFloor` | XP never goes below 0 (it wrapped to the maximum). |
| `eventTriggerObjects` | Event triggers with standard objects (`AttackedBy(PartySlot1,DEFAULT)`, `Heard(...)`) follow whoever the object names now, instead of locking on to one creature. Current EEex still has this bug. |

Since 0.4.1, also fixes found by IEX's own engine analysis (not in that thread):

| Name | Fix |
|---|---|
| `op183EffectLeak` | Opcode 183 (apply EFF if an item type is equipped) no longer leaks memory on every effect pass where no such item is equipped. (Opcode 182 had the same leak; `op182EquippedItem` covers it.) |
| `partyXPSigned` | Party XP with a negative value (`AddExperienceParty(-1000)`, XPLIST `P_` values...) takes XP away. It gave each member about 4.3 billion / members XP when 2 or more were alive. A loss is shared rounded toward zero (-1000 over 6 members: -166 each). |
| `op248MissingEff` | Opcode 248 (melee hit effect) with a missing EFF file and parameter 2 bit 2 no longer crashes. |
| `partyLevelSum` | The average party level no longer wraps when the members' levels add up past 255 (ARE spawn points, the NPC level chart on area load, `AddXP2DA`). |
| `twoDARows` | 2DA files with an indented row or a line of only spaces between rows load correctly (all rows kept; lines of spaces ignored). The engine wrote past its row arrays (crash, often later). Files without such lines load exactly as before. |

Tested on Windows and Linux with each fix off and on: the opcode 182 / 183 leak (memory grew 35-135 MB a minute in
the test, flat with the fixes), `AddExperienceParty(-1000)` with 6 members, `AddXP2DA` with a party level sum of 300,
an indented 2DA list and an indented startup table (both crash without the fix), opcode 248 with a missing EFF
(crash without the fix). The spawn point and NPC level chart uses of the party level are patched but were not run in
a game.

Not done: the chargen kit list bug (a generalist school flagged unusable turns the "Mage" entry into a specialist).

Settings (defaults on; a change applies after a game restart):

| Setting | Effect |
|---|---|
| `'IEX','Engine Fixes'` | 0 turns every fix off. |
| `'IEX','Engine Fix <name>'` | 0 turns that one fix off, e.g. `SetPrivateProfileString('IEX','Engine Fix xpFloor','0')`. Names in the table above. |
| Options screen | **Engine Fixes** tab: a master switch and one toggle per fix. |
| `IEX_ENGINE_FIXES=0` / `1` (environment) | Sets the master switch, overriding `Baldur.lua`. |
| `IEX_ENGINE_FIXES_OFF=name,name` (environment) | Turns off the fixes named; when it is set, the per-fix `Baldur.lua` keys are ignored. |

`IEX.log` says `Engine fixes: on: ...; not installed: ...`.

Also in the Lua layer (always on):

- **UI text function alias**: UI mods made for 2.6 call `t()`, 2.7 renamed it `getUiString()`. IEX provides
  whichever is missing, so 2.6-era UI mods do not show empty labels on 2.7.
- **No crash from early messages**: a mod that displays a message before a game exists (start-up, main menu)
  crashed the engine. The text now goes to `IEX.log` instead.

## BG Pathfinding Redux, native

BG Pathfinding Redux (by Zach Hopkins) lets party members and other allies walk through one another; walls, doors,
neutrals and enemies still block. Its own runtime needs EEex's assembly API, so IEX runs its movement rule natively
instead (Windows and Linux). The mod's other optional features (attack spacing, gentle settle, visual route
preference, enemy cooperation, overlap escape, its F3-F6 keys) are not native yet.

| Setting | Default |
|---|---|
| `'IEX','Pathfinding Redux'` | On when the mod is installed. Options screen: Movement tab, "Allied Pass-Through". |
| `bg-redux-movement.ini` `[Movement] OutOfCombatOnly` | The mod's own setting: allies block one another during combat. Options screen: "Only Outside Combat". |
| Left Ctrl + Left Shift + F1 / F2 | As in the mod: allied passage on/off for the session / out-of-combat-only mode (saved). |

## EEex player modules

EEex's optional modules, switched on in `EEex.ini` (`[<Name> Module] Enable=1`, read at game start) or in the
options screen's Modules tab:

| Module | What it does |
|---|---|
| Empty Container | Empty containers get a gray highlight. |
| Time Step | A key (D) advances a paused game by one tick; held half a second, time flows. |
| Timer | Bars next to the party portraits: modal action (red), contingency (green), spell / item cooldown (cyan). |
| Effect Menu | Hold a key (Left Shift) over a creature to list the spells affecting it. |
| Attack Info | To-hit chances of the selected party against the creature under the attack cursor (Left Alt: the reverse). |

Not available: Scale, Hotkey and Invisibility modules (an enabled one is noted in `IEX.log`).

## Concentration checks from Lua

Mods that set `CONCENTR.2DA` to `EEex-LuaFunction=<function>` (Tactics Remix #60, Tweaks Anthology #2690) get their
function called when a caster is hit, as under EEex. The function gets `damageData.damageTaken` and
`damageData.targetSprite`; `effect` and `sourceSprite` are not passed yet.

## Skip intro movies

Skips the movies the game plays at start (logos, intro) and shows the main menu at once. Movies later in the game
still play.

| Setting | Default |
|---|---|
| `'IEX','Skip Intro Movies'` | 0. 1 skips them. Not in the options screen. Environment variable `IEX_SKIP_INTRO=1` does the same. |

## LuaJIT mode libraries

IEX keeps the game's own Lua 5.2. Mods that need EEex's LuaJIT mode (BuffBot, EEex Remote Console) get what they use
from it: the `io`, `os` and `bit` libraries, `loadstring`, `unpack`, `coroutine` and `io.popen`. On Linux,
`io.popen('dir /b "<folder>\<pattern>"')` (a Windows command some mods use to list files) returns the same listing.

| Setting | Default |
|---|---|
| `InfinityLoader.ini` `[General] LuaPatchMode` | `INTERNAL`. `EXTERNAL` or `REPLACE_INTERNAL_WITH_EXTERNAL` gives mods these libraries. IEX component 2 sets it. |
| `InfinityLoader.ini` `[General] LogFile` | empty. A file name in the game folder gets a copy of every Lua `print()` line. |

## IEX options screen

IEX has the options screen EEex mods register their tabs in (`EEex_Options_*`), with EEex's layout.

- Open it with the **IEX Options** button in the game's Options menu (with Infinity UI++: in its options column), or
  with the **`\`** key. Values apply at once; a few say "(applies after a restart)".
- IEX's own tabs: **Miscellaneous Keybinds** (the open key, a key that shows key codes), **Modules** (EEex.ini
  modules), **Uncap FPS** (frame rate and game speed, above), **Movement** (Pathfinding Redux, shown when it is
  available), **Engine Fixes** (master switch and one toggle per [engine fix](#engine-fixes), restart required).
  Mods add their own tabs.

## Settings reference

`Baldur.lua` keys (`SetPrivateProfileString('<section>','<key>','<value>')`):

| Section | Key | Default | Feature |
|---|---|---|---|
| IEX | Frame Rate | 0 | [Frame rate](#smooth-frame-rate-and-frame-interpolation) |
| IEX | Frame Interpolation | 1 | Frame rate |
| IEX | Keep Game Speed | 1 | Frame rate |
| IEX | AI Speed | 30 | Frame rate |
| IEX | HD Area Tiles | 1 | [HD area tiles](#hd-area-tiles) |
| IEX | Skip Intro Movies | 0 | [Skip intro movies](#skip-intro-movies) |
| IEX | Pathfinding Redux | on if the mod is installed | [Pathfinding Redux](#bg-pathfinding-redux-native) |
| IEX | Options Key | `\` | [Options screen](#iex-options-screen) |
| IEX | Keycode Output Key | `` ` `` | Options screen |
| IEX | Engine Fixes | 1 | [Engine fixes](#engine-fixes) |
| IEX | Engine Fix `<name>` | 1 | Engine fixes (one key per fix) |
| IEX | Unlimited Kits | 1 | [Unlimited kits](#unlimited-kits) |
| EEex | Uncap FPS | 1 | Frame rate |
| EEex | VSync Enabled | 1 | Frame rate |
| EEex | Uncap FPS Limit Enabled | 0 | Frame rate |
| EEex | Uncap FPS Limit | 144 | Frame rate |
| EEex | Uncap FPS Busy Wait Threshold | 1 | Frame rate |
| EEex | Uncap FPS Fullscreen VRR | 0 | stored only |

Other files: `EEex.ini` (module switches), `InfinityLoader.ini` (Lua libraries, log file), `bg-redux-movement.ini`
(Pathfinding Redux).

Environment variables (for testing; set them before starting the game):

| Variable | Effect |
|---|---|
| `IEX_SKIP_INTRO=1` | Skip the intro movies. |
| `IEX_NO_PARTY_VISION=1` | Engine fog of war (15 viewers). |
| `IEX_OBJECT_IDS=0` | Engine object ids. |
| `IEX_PATHFINDING=0` / `1` | Pathfinding Redux off / on for this run. |
| `IEX_NO_PATHFINDING=1` | Pathfinding Redux hooks not installed. |
| `IEX_ENGINE_FIXES=0` / `1` | [Engine fixes](#engine-fixes) master switch (overrides `Baldur.lua`). |
| `IEX_KIT_LIMIT=0` / `1` | [Unlimited kits](#unlimited-kits) off / on (overrides `Baldur.lua`). |
| `IEX_ENGINE_FIXES_OFF=name,name` | The engine fixes named off (the per-fix `Baldur.lua` keys are then ignored). |
