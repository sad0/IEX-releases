# Changelog

All versions are pre-releases. Games: BG2EE / EET 2.7.3, Windows x64 and Linux x64.

## 0.4.0 (2026-11-04)

New:

- **Engine fixes** (on by default, Windows + Linux): opcode 182 equipped-item test, WSPECIAL.2DA SPEED column and
  launcher weapon speed, signed SPLPROT.2DA compares, TRAPLIMT.2DA off by one, opcode 33 param2 3, opcode
  206 / 232 / 256 messages, HP table modifier with dice, Simulacrum level drain on dual-classes, AddXPObject loss
  message, XP floor at 0, event triggers that lock on to one creature. `Baldur.lua`
  `'IEX','Engine Fixes'` / `'IEX','Engine Fix <name>'` (0 = off, restart needed), an **Engine Fixes** tab in the
  IEX options screen, and environment variables `IEX_ENGINE_FIXES` / `IEX_ENGINE_FIXES_OFF`.

- **HD area tiles** (`hdTiles`, on by default): PVRZ-based TIS files whose header tile size is 128 / 256 / 512 are
  drawn at 2x / 4x / 8x into the same 64x64 world tiles (the InfinityEngine-Enhancer convention). Game data stays at
  1x. Mipmaps without frame hitches (stored mips used as they are, missing ones built on a background thread and
  swapped in), seamless HD-aware tile shader, clamped sampling
  (no bleeding between tiles), door tiles included. OpenGL only; Direct3D 9 draws them at 1x. `Baldur.lua`
  `'IEX','HD Area Tiles'` 0 turns it off. See [docs/HD_PACKS.md](docs/HD_PACKS.md).
- **Unlimited class spell lists** (`spellLists`): mods add wizard and priest spells of any resref to the class lists
  (chargen, level-up, sorcerer / bard picks, dual-class, priests' automatic spells) with `IEXSPLST.2DA` or
  `IEX_SpellList_*`. The specialist bonus slot sees them; a lower `SPELLS.2DA` priest limit is kept.
- **Spell slots 51-99** (`spellSlots99`) and the **SpellSlots** companion mod.
- **Object ids never run out**: free object slots and ids are reused, which fixes the "Too many objects!" crash after
  long sessions (single player).
- **Concentration checks from Lua**: `CONCENTR.2DA` `EEex-LuaFunction=<fn>` calls the function (Tactics Remix #60,
  Tweaks Anthology #2690).
- The installer refuses games other than BG2EE / EET (BGEE and SoD included).
- **LuaJIT mode**: `loadstring`, `unpack`, the `coroutine` library and `io.popen`; on Linux, `io.popen` of a
  Windows `dir /b` listing works.

Compatibility fixes (from comparing the popular EEex mods side by side under EEex and IEX):

- Resource headers: `getAbility` / `getAbilityForLevel` on spell and item headers; mods can store their own fields on
  engine objects; item ability `type` read as 16 bits; `CGameSprite.m_nCopyParent`. BuffBot's own test suite now
  passes as under EEex; Infinity UI++'s timed effects and radar work.
- Creature file script fields (`m_scriptOverRide`, `m_scriptClass`, `m_scriptRace`, `m_scriptGeneral`,
  `m_scriptDefault`).

Fixes:

- Fog of war: IEX clears only its own visible bit, so visibility bits set by the engine or a mod are kept.

Package (compared with 0.3.1): `IEX/` adds `IEX_HdTiles.lua` and `IEX_SpellLists.lua`; the `SpellSlots/` mod is now
in the zip, next to `IEX/`.

## 0.3.1 (2026-10-30)

- Keep Game Speed: the game speed comes from `'IEX','AI Speed'` (the options screen's AI Speed), not from the tick
  rate the engine saved in `Baldur.lua`, which a mod could have raised (the game ran twice as fast after such a save).
- Options screen: with Infinity UI++, the IEX Options button is in its options column; the ESC menu stays under the
  options screen; mod tab labels shown.

## 0.3.0 (2026-10-30)

- **Native hook points** (Windows + Linux) replace engine patches that mods write in x64 machine code, with a router
  that turns EEex hook requests into native hooks. IEX's first assembly / JIT API was removed: IEX has none.
- **StickLuaFunctions**: 18 of its 24 engine hooks run natively.
- **BG Pathfinding Redux**: allied pass-through movement runs natively.
- **SadoEE**: endless ammo, attack-move, keep gear, respec, camp, chests in the ground bar, valuable containers, small
  footprint and lasting modes run natively on Windows and Linux; Party Stash and camp merchant engine calls on
  Linux; passive traps and the attack-move target work as under EEex.
- **Options screen** (`EEex_Options_*`) with EEex's layout, `\` key and button in the Options menu; `EEex.ini` and
  `InfinityLoader.ini` read.
- **EEex modules**: Empty Container, Time Step, Timer, Effect Menu, Attack Info.
- **Frame rate**: EEex's 'Uncap FPS' settings (same `Baldur.lua` keys), **frame interpolation**, **Keep Game Speed**.
- **Skip Intro Movies** (`'IEX','Skip Intro Movies'`).
- Per-creature mod data saved in games (`EEex_Sprite_AddMarshalHandlers`, EEex's save format), list row render
  listeners, opcodes 400 and 409, engine arrays as under EEex, many listener and value fixes found by an automated
  EEex-vs-IEX comparison.

## 0.2.2 (2026-10-29)

- **Unlimited fog-of-war viewers**: every party member and clairvoyance viewer clears fog and sees creatures (big
  parties); areas unload correctly with more than 15 members.
- `EEex_Read*` / `EEex_Write*` on near-null addresses raise a Lua error instead of crashing.

## 0.2.1 (2026-10-29)

- Bubb's Spell Menu Extended: spell button lists, spell list listeners, menu templates.
- The previous run's log is kept as `IEX.log.prev`.

## 0.2.0 (2026-10-29)

- Instant script actions; `EEex_LuaObject` / `EEex_Target` resolved when actions are queued.
- EEex raw memory API, UI menu API (find, items, templates, item render listeners).
- Tweaks Anthology gaps: script strings, object searches, weapon-hit and quick-list listeners.
- Engine methods callable on IEX objects.

## 0.1.0 (2026-10-28)

First preview.

- Runs inside BG2EE / EET 2.7.3 on Windows (launcher) and natively on Linux (preload).
- EEex-compatible Lua layer: listeners, game objects, sprites, scripting, resources, action bar, opcodes 401-403,
  `EEex_LuaAction` / `EEex_LuaTrigger`, the data tables and IDS entries EEex mods install against.
- Missing EEex names never raise errors: they are recorded in `IEX/missing_api.txt`.
- IEX additions: render frame rate above the logic rate, camera API, scancode keys, direct actions, loopback network
  bridge.
- WeiDU installer; bounded logging (`IEX.log` capped at 64 MB).
