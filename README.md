# IEX (Infinity Extender)

IEX is an engine extender for Baldur's Gate II: Enhanced Edition and EET. It replaces EEex: mods written for EEex
run on IEX without changes to their Lua code. IEX runs on **Windows and Linux**.

IEX is a clean-room project: it contains no code from EEex or InfinityLoader, and its authors do not read their
sources. Only the API that mods use is reproduced, from how the mods call it and from public documentation.

This repository holds the release archives and the documentation. The source code is at
https://github.com/sad0/IEX (GPL-3.0).

## Status

**Pre-release.** IEX works with the EEex mods listed in [docs/MOD_COMPAT.md](docs/MOD_COMPAT.md), but it is new.
Test it on a copy of your game or with a save you can lose, and report problems.

## Supported games and platforms

| | Supported |
|---|---|
| Games | Baldur's Gate II: EE and EET, engine **2.7.3** |
| Systems | Windows x64, Linux x64 |
| Renderer | OpenGL (the default). HD area tiles need OpenGL; with Direct3D 9 they are drawn at normal size. |

Not supported: BGEE / SoD on their own (the installer refuses them: BG2EE and EET 2.7.3 only), IWDEE, PSTEE,
engine 2.6 and older. On a game build IEX does not know,
it stays inactive and says so in `IEX.log`.

Linux has been tested under WSL2 (Windows Subsystem for Linux). Native Linux desktops have not been tested yet.

## What's new in 0.4.1

- **Mods that look for EEex at install now find IEX**, with no edit to the mod in most cases.
- **Unlimited single-class kits**: KITLIST rows 257 to 16383 get their class abilities. See
  [docs/FEATURES.md](docs/FEATURES.md#unlimited-kits).
- **Five more engine fixes** (opcodes 183 and 248, negative party XP, party level sum, indented 2DA rows), and a memory
  leak fixed in the 0.4.0 opcode 182 fix. On by default. See [docs/FEATURES.md](docs/FEATURES.md#engine-fixes).

Full list: [CHANGELOG.md](CHANGELOG.md).

## Quick install

1. Uninstall EEex if it is installed (IEX replaces it).
2. Download `IEX-<version>-windows-x64.zip` or `IEX-<version>-linux-x64.zip` from the Releases page.
3. Extract the `IEX` and `SpellSlots` folders into the game folder (next to `Baldur.exe` or `BaldursGateII`).
4. Install `IEX/IEX.tp2` with WeiDU, component 0. Install IEX **before** the mods that need EEex.
5. Start the game with `IEXLauncher.exe` (Windows) or `./iex-launch.sh` (Linux), both in the game folder.

The Candlekeep mod manager (coming) will also install IEX.

Details, uninstall, update and troubleshooting: [docs/INSTALL.md](docs/INSTALL.md).

## Documentation

| Page | For |
|---|---|
| [docs/INSTALL.md](docs/INSTALL.md) | Install, uninstall, update, troubleshooting |
| [docs/FEATURES.md](docs/FEATURES.md) | Every feature and option, with its setting |
| [docs/MOD_COMPAT.md](docs/MOD_COMPAT.md) | Which EEex mods work, what is missing |
| [docs/HD_PACKS.md](docs/HD_PACKS.md) | Authors of HD area packs |
| [docs/LUA_API.md](docs/LUA_API.md) | Modders: IEX's own Lua functions, how EEex calls are handled |
| [CHANGELOG.md](CHANGELOG.md) | Changes per version |

## Reporting problems

Please send `IEX.log` and `IEX/missing_api.txt` from your game folder (see
[docs/INSTALL.md](docs/INSTALL.md#troubleshooting)), your mod list (`WeiDU.log`) and what you did, to either:

- the [GitHub issues](../../issues) of this repository;
- the **Sado Mods** Discord server, for help and bug reports: https://discord.gg/3XzeQv7HeS
