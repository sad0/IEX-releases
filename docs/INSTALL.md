# Installing IEX

IEX supports Baldur's Gate II: EE and EET, engine 2.7.3, on Windows x64 and Linux x64. The installer refuses other
games ("BG2EE and EET 2.7.3 only"), BGEE and SoD included.

## Before you start

- **Uninstall EEex first.** IEX replaces it; the IEX installer refuses to run while EEex is installed.
- **Install order.** Install IEX where EEex would go: before every mod that needs EEex. For EET, install IEX on the
  finished EET game, before the EEex mods.
- **Back up** your game folder, or use a copy, while IEX is a pre-release.

## What is in the archive

| Folder | What it is |
|---|---|
| `IEX/` | The IEX WeiDU mod. This folder stays in the game folder: IEX reads its Lua files and engine descriptions from it at every start. |
| `SpellSlots/` | Optional companion mod: wizard and priest spell slots 51-99 for players (see [FEATURES.md](FEATURES.md#spell-slots-51-99)). |

`IEX-<version>-windows-x64.zip` holds the Windows binaries, `IEX-<version>-linux-x64.zip` the Linux ones. The rest
is the same.

## IEX components

| # | Component | When to install it |
|---|---|---|
| 0 | IEX core (EEex-compatible engine extender) | Always. |
| 1 | EEex-compatible data tables and script IDS | Only to update an old IEX core install that was made before these tables existed. Component 0 already includes it. |
| 2 | Lua io, os and bit libraries for mods (EEex LuaJIT mode) | When a mod asks for EEex's LuaJIT mode, e.g. BuffBot and EEex Remote Console. |

Component 0 also writes `InfinityLoader.ini` in the game folder if there is none (IEX's own settings file, in the
format EEex mods read), and copies the IEX options screen into `override/`.

## Install manually with WeiDU

### Windows

1. Extract `IEX/` and `SpellSlots/` into the game folder, next to `Baldur.exe`.
2. Copy `weidu.exe` to the game folder as `setup-IEX.exe` and run it. Install component 0 (and 2 if needed).
3. Optional: copy `weidu.exe` as `setup-SpellSlots.exe` and install SpellSlots right after IEX, before spell mods.
4. Install your EEex mods.
5. Start the game with **`IEXLauncher.exe`** in the game folder. Starting `Baldur.exe` directly runs the game without
   IEX. Make a shortcut to `IEXLauncher.exe` if you like.

### Linux

1. Extract `IEX/` and `SpellSlots/` into the game folder, next to the `BaldursGateII` executable.
2. Run WeiDU on `IEX/IEX.tp2` (for example a copy of the Linux `weidu` binary named `setup-IEX`, run from the game
   folder). Install component 0 (and 2 if needed).
3. Optional: install `SpellSlots/SpellSlots.tp2` right after IEX.
4. Install your EEex mods.
5. Start the game with **`./iex-launch.sh`** from the game folder. The installer puts `iex-launch.sh` and
   `libiex.so` there; the script preloads `libiex.so` and starts `./BaldursGateII`.

### EEex mods that check for EEex at install time

IEX runs EEex mods' Lua unchanged, but some mods check for EEex in their installer (`MOD_IS_INSTALLED ~EEex.tp2~`,
`FILE_EXISTS ~override/M___EEex.lua~`, ...) and refuse to install without it. These checks must accept IEX too.
[MOD_COMPAT.md](MOD_COMPAT.md#install-time-checks) lists them and the change each needs. The Candlekeep mod manager
(coming) is meant to handle this for the mods in its catalog.

## Install with Candlekeep (coming)

The Candlekeep mod manager (coming) will list IEX in its catalog and install it before the EEex mods.

## Update

Recommended way:

1. Extract the new archive over the old `IEX/` and `SpellSlots/` folders (replace the files).
2. Reinstall IEX component 0 with WeiDU (run `setup-IEX` and choose to reinstall). This copies the new binaries to
   the game folder. WeiDU uninstalls and reinstalls the mods installed after IEX on its own; that can take a while on a
   big install.

If you also use SpellSlots, reinstall it the same way when its version changed.

## Uninstall

1. Run `setup-IEX` and uninstall IEX's components (and SpellSlots, if installed). WeiDU removes the files IEX installed
   (`IEX.dll`, `IEXLauncher.exe` or `libiex.so`, `iex-launch.sh`, the options screen, the data tables) and restores
   what it replaced.
2. Delete the `IEX/` and `SpellSlots/` folders.

Mods that need EEex stop working without IEX (or EEex).

## Troubleshooting

### Where the logs are

| File | Where | What |
|---|---|---|
| `IEX.log` | game folder | Everything IEX did at start (game build found, features installed, settings read) and every error from mods' Lua listeners. Capped at 64 MB; repeated messages are logged at 1, 10, 100... occurrences. |
| `IEX.log.prev` | game folder | The log of the previous run (useful after a crash and restart). |
| `IEX/missing_api.txt` | `IEX` folder in the game folder | Every EEex name a mod used that IEX does not provide, with the file and line of its first use. |
| `Baldur.lua` | the game's user folder: Windows `Documents\Baldur's Gate II - Enhanced Edition\`, Linux `~/.local/share/Baldur's Gate II - Enhanced Edition/` | Game and IEX settings (see [FEATURES.md](FEATURES.md#settings-reference)). |

The first line of `IEX.log` names the IEX version and platform.

### Common problems

| Symptom | Cause, fix |
|---|---|
| Mods behave as if no extender is installed. | The game was started with `Baldur.exe` / `BaldursGateII`. Start it with `IEXLauncher.exe` / `./iex-launch.sh`. |
| The installer stops with "BG2EE and EET 2.7.3 only". | The game is not BG2EE / EET (e.g. BGEE, SoD). IEX does not support it. |
| `IEX.log` says `No engine description for this game build in IEX/engine, IEX stays inactive`. | Your game executable is not a build IEX knows (not 2.7.3, or a 2.7.3 build from another source). Report it with your game version and store. |
| The installer stops with "Uninstall EEex first". | EEex is installed. Uninstall it, then install IEX. |
| An EEex mod's installer says EEex is missing. | See [install-time checks](MOD_COMPAT.md#install-time-checks). |
| A mod's feature does nothing. | Look for its name in `IEX/missing_api.txt` and for errors in `IEX.log`. Missing names do not stop the game: IEX gives the mod a stand-in that does nothing. |
| A mod (or your game) behaves differently since 0.4.0 in combat, XP, traps or HP rolls. | Probably an [engine fix](FEATURES.md#engine-fixes). Turn that fix off in the IEX options screen (Engine Fixes tab) or in `Baldur.lua`: `SetPrivateProfileString('IEX','Engine Fix <name>','0')` (all: `'IEX','Engine Fixes','0'`). Restart the game. |
| BuffBot or EEex Remote Console asks for LuaJIT. | Install IEX component 2. |
| Linux: `./iex-launch.sh: Permission denied`. | `chmod +x iex-launch.sh libiex.so` in the game folder (the installer normally does it). |

When you report a problem, send `IEX.log`, `IEX.log.prev`, `IEX/missing_api.txt` and `WeiDU.log`.
