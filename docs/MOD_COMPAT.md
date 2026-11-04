# Mod compatibility

IEX aims to run every EEex mod unchanged. This page says how far it gets: how much of the EEex API is covered, how
gaps are reported, which mods were tested, and what is known not to work.

## API coverage

A static scan of 49 public EEex mods (every EEex mod found on GitHub, Gibberlings3, Spellhold and in mod lists, plus
SadoEE), run on 2026-11-03 against the 0.4.0 code, counts what these mods use and what IEX provides:

| What mods use | Distinct items | Provided by IEX | Uses in mods covered |
|---|---|---|---|
| EEex functions, tables and constants | 369 | 293 (+2 partial) | 846 of 947 |
| Listeners (`EEex_*_Add*Listener`) | 28 | 28 | 88 of 88 |
| Engine object fields | 291 | 239 | 601 of 662 |
| EEex opcodes 400-409 | 8 | 4 (400, 401, 402, 403; 409 through its listener) | 26 of 37 |
| Script IDS entries (actions, triggers, objects) and data tables | 79 | 77 | 101 of 103 |

IEX defines 610 EEex names in all (listed in `IEX/IEX_API.txt`). The scan is static: a name that exists can still
behave differently, which is why the mods below were also tested in game.

## How missing calls are reported

- An EEex name that IEX does not provide never causes a Lua error. Reading it gives a stand-in that can be called,
  indexed and iterated, and does nothing. The mod keeps loading; only the feature that needs the name does not work.
- Names that EEex itself does not define read as `nil`, as under EEex, so a mod's `if EEex_Something then` test
  gives the same answer.
- Each missing name is written once to `IEX.log` and to `IEX/missing_api.txt` (name, then file and line of the first
  use). Please include that file in bug reports.
- Engine patches that mods request through EEex's assembly hook API are matched to IEX's native hooks. A request
  without a native equivalent does nothing; it is logged once (`not supported natively: <mod> <address>`) and
  recorded in `IEX/missing_api.txt`.

## Tested mods

Full install (WeiDU, every component listed) and a 10-minute scripted play session (walking, fights with spells,
rest, area transition, save) on fresh BG2EE 2.7.3 games, Windows and Linux, with IEX 0.3.0 (2026-10-30), re-checked
with the 0.3.1 package. Features of the most popular mods were then compared side by side, EEex against IEX, on the
same save (2026-10-30). Mod tp2 files had their EEex install checks adapted ([below](#install-time-checks)).

| Mod | Components | Install (Win / Linux) | Result |
|---|---|---|---|
| Tweaks Anthology v18 | 48 EEex components | 48/48, 48/48 | Works. Same feat locals, innates and stats as under EEex (stat 78 differs, see below). One Lua error per fight from TA's own code (`M_GTSPRT.lua:342`), being investigated. |
| Bardic Wonders | 19 | 19/19, 19/19 | Works. #2009 asks a question at install (answer it). |
| Shadow Magic | 0 1 5 | 3/3, 3/3 | Works. (#5 needs one of 1-4.) |
| The Artisan's Kitpack 4.81 | 1000 2000 7006 10001 10002 | 5/5, 5/5 | Works. Action bar the same as under EEex. Proficiency markers on freshly spawned creatures differ (see below). |
| Infinity UI++ v1.31 | 0 | 1/1, 1/1 | Works: items, timed effects, radar, portraits, scripts the same as under EEex. The IEX Options button is in its options column. |
| Bubb's Spell Menu Extended v5.2 | 0 | 1/1, 1/1 | Works in every mode, same lists as under EEex. |
| Tactics Remix 8.2 | 60 (Concentration) | 1/1, 1/1 | Works: its concentration function is called as under EEex. |
| Skills and Abilities | 11 20 50 60 120 160 | 6/6, 6/6 | Installs and loads. |
| StickLuaFunctions v0.11.0 | 0 | 1/1, 1/1 | 18 of its 24 engine hooks run natively. Not yet: the 6 backstab hooks, and its melee listener (`EEex_Opcode_Hook_OnAfterSwingCheckedOp248`). |
| BG Pathfinding Redux | 0 | 1/1, 1/1 | Allied pass-through runs natively ([FEATURES.md](FEATURES.md#bg-pathfinding-redux-native)); the mod's other options not yet. |
| BuffBot 1.12.1 | 1, 0 | 2/2, 2/2 | Works: its own test suite gives 812 pass / 0 fail / 12 warnings, as under EEex; buffs cast the same. Needs IEX component 2. Its window border (`EEex.DrawSlicedRect`) is not drawn. |
| EEex Remote Console | 0 | 1/1, 1/1 | Installs and runs. Needs IEX component 2. |
| SadoEE v1.0.2 | 25 BG2EE components | 25/25, 25/25 | Works; its engine patches run as IEX native hooks on Windows and Linux. |

Totals: 116 of 116 components installed on each system, no crash, no error popup, both systems giving the same
`WeiDU.log`. An automated differential test (about 4,400 checks run on EEex and on IEX with the same save, 0.3.1)
found no check that raises an error only under IEX.

Not yet tested with 0.4.0: the full install above was not repeated; the 0.4.0 changes (HD tiles, spell lists,
object ids) are off or invisible unless a mod or pack uses them; the engine fixes were tested on their own (each
bug shown with the fixes off and gone with them on, Windows and Linux).

## Install-time checks

Some mods check for EEex in their installer and refuse to install without it. These checks must also accept IEX.
The Candlekeep mod manager (coming) is meant to adapt them; if you install by hand, edit the mod's `.tp2` / `.tpa` / `.tph` files:

| The mod checks | Make it accept |
|---|---|
| `MOD_IS_INSTALLED ~EEex.tp2~ ~0~` (any spelling, or by label `B3-EEex-Main`) | `(<the check> OR MOD_IS_INSTALLED ~IEX/IEX.TP2~ ~0~)` |
| `REQUIRE_COMPONENT ~EEex.tp2~ ~0~` | `REQUIRE_PREDICATE (MOD_IS_INSTALLED ~EEex.tp2~ ~0~ OR MOD_IS_INSTALLED ~IEX/IEX.TP2~ ~0~)` with the same message |
| `FILE_EXISTS ~override/M___EEex.lua~`, `~M__EEex.lua~`, `~EEex.dll~`, `~lua51.dll~`, `~LuaProvider.dll~`, an `EEex_<name>.lua` file (not `EEex_Modules.lua`) | `(<the check> OR FILE_EXISTS ~IEX/IEX_Main.lua~)` |
| `FILE_CONTAINS` on an EEex Lua file for a function name | `(<the check> OR FILE_CONTAINS_EVALUATED (~IEX/IEX_API.txt~ ~^<name>$~))` (true only if IEX has that function) |

Left as they are on purpose (the mod's own fallback or refusal is right under IEX): checks for `InfinityLoader.db`,
`InfinityLoader.exe`, `EEex.exe`, `EEex.db`, EEex's pattern files, `EEex_Modules.lua`, and EEex component numbers
other than 0 (they mean different things in different EEex versions).

Detection for mod authors: IEX appears in `WeiDU.log` as `~IEX/IEX.TP2~ #0 #0`; `IEX/VERSION` holds the version;
at run time `IEX_Features` exists ([LUA_API.md](LUA_API.md#features-and-detection)).

## Known incompatibilities and differences

- **x64 assembly / JIT API** (`EEex_JITNear`, `EEex_Hook*WithLabels`, `EEex_GenLuaCall`...): not provided, and not
  planned. Mods that patch the engine this way get native IEX replacements only where one exists (Pathfinding Redux,
  StickLuaFunctions, SadoEE). Other such patches do nothing, e.g. the engine patches of the chriz-bg mods.
- **Opcodes 404-408**: not provided (used by High Power, Olvyn Spells, Improved Heart of Fury, some Tweaks Anthology
  components).
- **EEex modules** Scale, Hotkey and Invisibility: not provided (Infinity UI++ reads `B3Scale`).
- **Early x86 EEex API** (`EEex_ReadDword`, `EEex_ApplyEffectToActor`...: Improved Heart of Fury, EET Tweaks #2080):
  not provided; these mods do not work on current EEex either.
- **Concentration callback**: `damageData.effect` and `damageData.sourceSprite` are `nil` (EEex passes them). No
  tested mod reads them.
- **Stat 78 (PHYSICALSPEED)**: `EEex_Sprite_GetStat(sprite, 78)` is 0 under IEX where EEex gives 3-4 on some
  high-level warriors. No tested mod reads it.
- **Artisan's Kitpack proficiency markers**: freshly spawned creatures get only their class's set of proficiency
  marker effects, where EEex gives all 20. Re-applying them gives the same result on both. Under investigation.
- **Engine fixes**: IEX makes the fixes EEex makes ([FEATURES.md](FEATURES.md#engine-fixes)) except the chargen kit
  list one (not done yet).
- **Engine 2.6 and BGEE / SoD alone** are not supported, so EEex mods for those games cannot use IEX.
- **Multiplayer** has not been tested.
