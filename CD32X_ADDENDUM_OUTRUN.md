# CD32X – Addendum: booting OutRun (Cannonball CD32X)

An addendum to KOBO-CD32X_BOOT_PROCESS, the Command Ledger and the OpenLara, OpenJazz and
OpenTyrian addenda. It covers what was learned getting a new game, Cannonball (OutRun),
to boot on real hardware: Sega CD + 32X + 4 MB RAM cart.

## 1. The rule
**One proven boot path. Never change it. The game runs last.**

| Stage | Part | Source |
|---|---|---|
| 0 | Sega CD BIOS | console |
| 1 | Boot block (`US_BOOT.BIN`) | toolchain |
| 2–4 | Sub-CPU program + 68000 listener (`APP.BIN`, `hw_md.s`) | OpenLara CD32X `subcpu/` |
| 5–6 | CD loader (`CDLOADER.BIN`): copies the game into the cart, starts both SH2s | OpenLara CD32X `sh2/` |
| 7 | The game's start-up: `crt0.s` + `src-md/` (`m68k.bin`) | OpenLara's, byte for byte |
| 8 | The game (`main()`) | the new project |

The new game is only the file the loader copies (`OPENLARA.32X`, file 20) and the data
files it loads afterwards.

## 2. What went wrong
- **The game's `crt0.s` was edited**: the `m68k.bin` include was removed, the header text
  changed and the `.data` contents altered. Result on the console: **black screen after the
  SEGA logo**, before the splash. Nothing on screen gives a clue at that stage.
- The fix was to restore OpenLara's `crt0.s` and `src-md/` unchanged. The game's code
  follows after them.
- "I compared the loader, the Sub-CPU files and the boot sector, they are identical" was
  not enough. The game's own start-up is part of the boot stack too.

## 3. Do
- Copy **every** start-up file from the working OpenLara folder unchanged, the game's
  `crt0.s` and `src-md/` included. Keep OpenLara's header (`"OpenLara"` at 0x3C0) and identify
  the game by other text inside it.
- Change behaviour only with the **switches that already exist** (`KOBO_DIAG_TEXT`,
  `KOBO_BOOT_TEXT`, ...). Don't add new switches to the loader or the Sub-CPU.
- Load data under **file numbers and names the Sub-CPU program already knows**. OutRun's 2.4 MB
  pack goes on the disc as `TITLE.PKD`: CMD 38 file 200 → cart slot 4 (0x080000). The program
  keeps the name `OPENLARA.32X` (file 20 → cart 0).
- Make the build **compare** its `APP.BIN` and `CDLOADER.BIN` with the working OpenLara build
  (`cmp`). Identical = the boot stack is untouched.
- Post status codes in **COMM12** from `main()` (4B00 started, 4B01 loading, 4B02 pack in the
  cart, 4B03 engine ready, 4B10 running). The Sub-CPU shows them as `STATE:` when the green
  text is on, and 4B10 makes it stop redrawing its text.
- Bring a new game up in steps: logic only with a 32X text status screen first, then the
  picture, then speed. Each step proves the one before.
- Run the engine **on the PC first** with the same port layer and the same data pack.
  It finds game and data bugs, but never boot or timing bugs.
- Burn the `.cue` the build writes. A data-only disc (3 MB, one MODE1/2048 track) boots fine.
- Before testing, check that the folder holds the **latest build**: `git log` and the file
  dates. One "it still fails" was an older disc.

## 4. Don't
- **Don't edit the game's `crt0.s`, `src-md/` or `m68k.bin`.**
- Don't add a Sub-CPU file number when an existing one fits.
- Don't turn the Genesis text on over the game. The green and white text has priority over
  the 32X picture: it covered the game's own status screen. Use 32X-drawn status lines
  instead, and switch the Genesis text off (CMD 52, COMM2 = 0) once the game runs.
- Don't add diagnostics to the boot stack to find a boot problem. Turn on the existing ones,
  or swap one file at a time against the working disc.

## 5. Cart layout used by OutRun
| Cart offset | Contents | Loaded by |
|---|---|---|
| 0x000000 | the program (~140 KB) | CD loader, file 20 |
| 0x080000 | OUTRUN.PAK as TITLE.PKD (2.5 MB) | the game, CMD 38 file 200 + CMD 46 |
| 0x3F0000 | saves (later) | – |

Loading OUTRUN.PAK overwrites OpenLara's Caves area (0x080000). OpenLara notices and loads it
again on its next boot.

## 6. Quick check before burning
1. The build says the CD loader and the Sub-CPU program are identical to OpenLara's.
2. The disc folder holds `APP.BIN`, `CDLOADER.BIN`, `IMAGE.RAW`, `OPENLARA.32X`, `TITLE.PKD`.
3. Burn the `.cue`.

On the console: SEGA logo → boot splash → the game's loading screen → the game.
