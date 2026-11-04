# HD area packs

How to make area tiles that IEX draws at 2x, 4x or 8x resolution (IEX 0.4.0 and later, BG2EE and EET 2.7.3, Windows
and Linux, OpenGL renderer).

## The idea

Only the pictures get bigger. Every area keeps its normal size: the ARE and WED files, search, height and light maps,
and every coordinate stay exactly as in the game. Each 64x64 world tile is simply drawn from 128x128 (2x),
256x256 (4x) or 512x512 (8x) texels of a bigger PVRZ page.

So an HD pack only replaces an area's **TIS** file and its **PVRZ** pages. It works with existing saves and with
mods that change the area's scripts, creatures or doors.

## File format

IEX uses the same convention as InfinityEngine-Enhancer (IEE): the tile size is written in the TIS header.

### TIS

- A normal PVRZ-based TIS V1 (12-byte tile entries: page, x, y), header size 0x18 as usual.
- Header offset **0x14 = tile size**:

  | Value at +0x14 | Scale |
  |---|---|
  | 64 | 1x (every vanilla TIS: drawn exactly as without IEX) |
  | 128 | 2x |
  | 256 | 4x |
  | 512 | 8x |

  Any other value is drawn at 1x (and noted in `IEX.log`).
- The tile entries' `x` and `y` are texel positions in the HD page: the vanilla positions times the scale.
- The tile count and the tile order do not change.
- The HD TIS must be a loose file in `override/`. A TIS read from a BIFF archive is never HD.
- Palette-based TIS (the old format, 0x1400 bytes per tile) are never HD.

### PVRZ pages

- Same names as the vanilla pages (the engine builds them from the TIS name, e.g. `A060200.PVRZ` for `AR0602.TIS`).
- Same format: PVR3, zlib-compressed as PVRZ, **DXT1 (BC1)** like vanilla. DXT5 (BC3) also loads.
- **Keep the BC1 punch-through alpha.** Vanilla area pages are BC1 with 1-bit alpha: texels with alpha 0 mark where
  the water or other overlay shows through, under the WED's overlay mask cells. HD pages must keep the same
  transparent texels at the new size (N times bigger):
  - scale the alpha with nearest filtering (no soft edges: a texel is either opaque or transparent);
  - scale the colour weighted by alpha, so transparent texels do not bleed into the colour;
  - encode blocks that contain transparent texels in BC1's 3-colour mode (the one with a transparent index).

  Otherwise the tiles under water draw as **black squares**. Some BC1 encoders always write opaque blocks: check
  yours.
- Each side is the vanilla page side times the scale: a 1024x1024 page becomes 2048 (2x), 4096 (4x) or 8192 (8x).
  8192 is the largest size IEX can address.
- **Mipmaps**: if the PVR3 data holds mip levels after level 0 (level 0, then each next level at half size, down to
  any level), IEX uses them as they are. IEX finds them from the data size, which must match the levels exactly.
  Without stored mips, IEX builds them itself on a background thread (BC1 and BC3 only) and draws level 0 meanwhile.
  Storing mips saves that work and gives the same result on every machine.

### What else to know

- **Doors**: door tiles are extra tiles of the same TIS, so they are HD with it (tested: open and closed doors).
- **Overlays** (water, lava...): each overlay has its own TIS and decides its own scale. An HD base with a 1x overlay
  works, and the reverse. The vanilla water overlays are palette-based TIS and stay 1x. Water overlays tested in
  game (EET, Windows); the base tiles under the water need the punch-through alpha above.
- **Night areas** have their own TIS (e.g. `AR0602N.TIS`) and pages. Make them HD the same way; each TIS is
  decided on its own. Tested in game on Windows, with a MapSmithEE-made area at 2x and 4x, and on EET day and night
  areas (below).
- **Minimap, area map (MOS), fog of war**: unchanged, not part of the pack.
- **Save thumbnails** use the HD pages, read back at thumbnail size.

## Tested

| Game | System | What |
|---|---|---|
| BG2EE 2.7.3 | Windows, Linux (WSL2) | AR0602 at 2x / 4x: zoom, doors open and closed, mipmaps, scrolling |
| EET 2.7.3 | Windows | BG1-side area BG2600 (Candlekeep) and BG2-side AR0700, day and night, at 2x / 4x: doors, water overlays, zoom |
| BG2EE, EET | BG2EE: Windows, Linux; EET: Windows | Vanilla equivalence: tile pixels identical to the game with `hdTiles` off or with no HD files |

EET on Linux is not tested.

## Check that it works

Start the game with IEX, enter the area, then look in `IEX.log` (game folder):

| Line | Meaning |
|---|---|
| `[hdTiles] switch on` | HD tiles are enabled. |
| `HD tiles: N call sites redirected (...)` | IEX hooked the tile drawing on this game build. |
| `HD tiles: seam program HD-aware (...)` | The tile shader handles HD tiles (seamless, clamped filtering). |
| `HD tiles: AR0602 tile size 128 -> 2x` | Your TIS was recognised (once per tile set). |
| `HD tiles: AR0602 tile size 100 -> 1x (not 128/256/512)` | The header value is not a supported size. |
| `HD tiles: Direct3D 9 backend (...) drawn at 1x` | The game uses Direct3D 9; switch to OpenGL. |
| `[hdTiles] off (Baldur.lua IEX / HD Area Tiles = 0)` | The player turned HD tiles off. |

No `tile size` line for your area means the TIS was not read from `override/`, or its header still says 64.

From the Lua console, `IEX_HdTiles_Status` is `'on'`, and `IEX_HdTiles_Stats()` returns counters, the first two
being HD tile draws and other tile draws since start ([LUA_API.md](LUA_API.md#hd-area-tiles)).

To see the detail, zoom in. The engine's default zoom-in limit is 0.5 (2x magnification).

## Performance and VRAM

- Memory grows with the square of the scale: 2x = 4 times the page memory, 4x = 16 times, plus a third for mipmaps.
  Example, AR0602 (29 pages, BC1): about 80 MB at 2x, about 320 MB at 4x.
- **Recommended: 2x.** It matches the engine's default zoom-in limit (2x magnification), so the extra detail is all
  visible, at a quarter of the 4x memory. Use 4x or 8x only if you know players zoom further (mods can lower the zoom
  limit) and have the VRAM.
- Loading time: the engine unpacks each PVRZ page before uploading it. A 4096x4096 BC1 page took 15-38 ms in tests,
  several pages when a new part of the area scrolls into view. That cost is the data size, not IEX.
- Built mipmaps: about 100-300 ms per 4096 page on one background core, without stalling the game. Stored mipmaps
  skip this.
- Scrolling a 4x test area on Windows (RTX 5080), the longest single step IEX itself took on the render thread was
  1.5 ms (quiet machine) to 34 ms (busy machine) with built mips, and 7 ms with stored mips.

## Known limits

- OpenGL only. Direct3D 9 draws HD tiles at 1x.
- Pages larger than 8192 texels per side do not work.
- With "Use Nearest Neighbour Scaling" on, HD pages are mipmapped with nearest-mip filtering, and the edge clamp that
  prevents bleeding between neighbouring tiles of a page is not applied (nearest sampling of level 0 does not cross
  tile edges; coarse levels may by one texel).
- The lowest mip levels mix neighbouring tiles a little at the edges; the clamp limits it to what is sampled.
- Shader mods: IEX extends the game's tile shader (`fpSEAM`). A shader mod's version works if it keeps the game's
  variable names and `void main()`; otherwise IEX leaves that shader as it is and draws HD tiles with the game's
  plain tile shader (no seam smoothing on HD tiles).
- Only area tiles. HD creature sprites are being studied.
- Packs made for IEE use the same TIS convention but have not been tested with IEX yet.
- Linux was tested under WSL2 only; there, creating the texture for built mipmaps can take much longer (a property
  of that graphics layer).
