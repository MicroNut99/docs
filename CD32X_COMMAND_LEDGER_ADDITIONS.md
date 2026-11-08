# CD32X – Command Ledger additions

An addition to **KOBO-Sega 32XCD Boot Sequence and Sub-CPU Command Ledger.pdf**. It records the
commands added to the boot stack after that ledger was written, so every CD32X project uses one
registry. As the OpenTyrian addendum asks: **never reuse a number with a different meaning.**

Stack: the OpenLara / Hurrican CD32X boot path (Sub-CPU `main.c`, 68000 listener `hw_md.s`),
as used by SuperTux CD32X REV, and since 29 Oct 2026 packaged as MicroNut99/CD32X-BootCamp (`stack/`).

Last audit against the code: **31 Oct 2026** (BootCamp `a8c8afd`, OpenJazz REV `d486f58`, Hurrican `009912a`,
SuperTux REV `83032ed`, D32XR `91a7baf`, Rayman `ab5e08b`, Sonic `bc3c30b`): sections 2, 3, 4, 5, 7, 10 corrected,
sections 16-19 added. Where this file and a repository's `main.c` disagree, the code wins; fix this file and say so.

## 1. SH2 → Sub-CPU commands 62–66: the Genesis picture (SuperTux CD32X REV2)

**Status: boot-proven.** The REV2 disc with these commands booted and ran on real hardware
(Sega CD + 32X + 4 MB RAM cart, 15 Oct 2026). They are not yet exercised by a game.

**Why:** the 68000 listener always had VDP routines (Chilly's `set_vram`, `copy_vram`,
`set_palette`; Kobo's `set_vdp_reg`), but the SH2 could only reach CMD 52 (Genesis display
on/off). These commands let a game put pictures on the Genesis planes and scroll them, as Sonic
CD does, while the 32X draws on top.

**Rules (as every command):** the SH2 writes COMM2 / COMM8 first, then the command number in
COMM0; done = COMM0 cleared by the Sub-CPU. Nothing in the boot sequence sends them. Cart data
is read by the 68000 through the 32X bank window `0x900000` (RV 0, as CMD 46's read-back), so
the SH2 is never locked out of the cart. A copy that crosses a 1 MB bank of the window is done in
two parts.

| CMD | Name | COMM2 (16 bit) | COMM8 / COMM8L | Reply | 68000 routine |
|---|---|---|---|---|---|
| 62 | VRAM word | VRAM byte address (even) | COMM8 = the word | – | 5 `set_vram` |
| 63 | Cart → VRAM | VRAM byte address (even) | COMM8L = words << 22 \| cart offset (even, < 4 MB; 1–1023 words) | COMM12 = 1 ok / 0 refused | 7 `copy_vram` |
| 64 | Cart → CRAM | first colour (0–63) | COMM8L = count << 22 \| cart offset (count 1–64, first + count ≤ 64) | COMM12 = 1 ok / 0 refused | 12 `set_palette` |
| 65 | VDP register | `0x8000 \| reg << 8 \| value` (registers 0–23 only, others ignored) | – | – | 34 `set_vdp_reg` |
| 66 | VSRAM word | VSRAM byte address: 0 plane A, 2 plane B (full-screen V scroll; up to 0x4E in column mode) | COMM8 = the scroll value | – | 36 `set_vsram` (new) |
| 67 | Cart → VRAM columns (REV4) | VRAM byte address of the first column's top word (even) | COMM8L = (columns − 1) << 30 \| words a column << 22 \| cart offset (even; 1–4 columns of 1–255 words, one column after the other; all within one 1 MB bank of the cart window) | COMM12 = 1 ok / 0 refused | 37 `copy_vram_col` (new) |

Uses:
- tiles, plane maps and the horizontal scroll table: 63 (from the cart) or 62 (single words);
- palette: 64; plane size, scroll mode, plane addresses, display: 65; vertical scroll: 66.
- a plane map column (a level scrolling sideways: a new column every 8 pixels): 67 - column k
  goes to address + 2 k, its words 128 bytes apart (a 64-cell-wide map).

**67 status: added in SuperTux CD32X REV4 (15 Oct 2026)**; SuperTux REV5 and REV10b ran on the console
with it in the program (CD32X-BootCamp docs/07, 29 Oct 2026). **67 has a second meaning in OpenJazz
CD32X REV** (Genesis plane: load / show / off / cell, section 19); the two never share a Sub-CPU program,
but code moved between them must renumber.

**62 has a second meaning too (31 Oct 2026 audit).** On the EverDrive launcher builds of OpenLara
CD32X-REV (EDSAVE-1, console-proven 26 Oct 2026), OpenJazz REV and Hurrican, **62 = "save piece
in"**; on D32XR CD32X **62 = "Genesis background"** (HUR genbg, with listener routine 36 `set_scroll`) (a 4 KB piece of the EverDrive's bank 31 → the Sub-CPU's buffer, section 16). SuperTux, whose 62 is
`VRAM word`, put its copy of that command on **69** - which is OpenJazz REV's LZ unpack (section 19).
Both breaches of the "one number, one meaning" rule are recorded here rather than hidden; the numbers
that clash nowhere for a bank-31 copy are the BootCamp `sdcard/` kit's **74 / 75** (section 18) - and 74
itself now clashes with Rayman (section 17). Pick from 76 up for anything new.

## 2. Sub-CPU → 68000 listener commands 36 and 37 (SuperTux CD32X REV2 / REV4)

| MD_CMD | Name | Arguments | Notes |
|---|---|---|---|
| 36 | `set_vsram` | addr (VSRAM byte address, masked to 0x7E), word | REV2 |
| 37 | `copy_vram_col` | addr (VRAM byte address), src (68000 address), words | REV4: `copy_vram` with the VDP auto-increment at 128 (0x8F80), put back to 2 afterwards. `MD_CMD_END` is now **38** |

**Why a new routine:** VSRAM is not video RAM. `set_vram`'s address command always targets VRAM
(`0x4000….`); a VSRAM write needs the code `0x40000010`. Same shape as `set_vram`:
auto-increment 2, the 32-bit address command in one `move.l`, then the data word. It runs in the
command loop, never in an interrupt. It adds 48 bytes to the listener (`md_start` …
`md_init_end`, copied to `0xFF1000`).

The ledger PDF's 68000 list ends at 34 `SET_VDP_REG`, 35 `END`. The current list on the OpenLara /
Hurrican path: 34 `set_vdp_reg` (Kobo K19), **35 `start_d32xr`** (D32X M1), **36 `set_vsram`** (REV2),
**37 `copy_vram_col`** (REV4), `MD_CMD_END` 38.

**Each game's listener numbers its own routines above 35** (31 Oct 2026 audit). They are listed here so
that nobody mistakes one game's 38 for another's. The BootCamp's own `hw_md.s` stops at 35 (`MD_CMD_END, 36`:
routines 1-35) unless it is built with `GENPLANE=1`; then it has SuperTux's 36 / 37 and `MD_CMD_END` 38
(section 20):

| Listener | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | `MD_CMD_END` |
|---|---|---|---|---|---|---|---|---|---|---|---|
| BootCamp `stack/subcpu/hw_md.s` | - (`GENPLANE=1`: `set_vsram`) | - (`GENPLANE=1`: `copy_vram_col`) | - (`VSPLIT=1`: `set_vs_line`, section 21) | - (`FBVDP=1`: `fb_list`, section 24) | - (`TILEP=1`: `tile_copy`, section 25; 36-39 `cmd_nop` when their switches are off) | - | - | - | - | - | 36 (`GENPLANE=1`: 38; `VSPLIT=1`: 39; `FBVDP=1`: 40; `TILEP=1`: 41) |
| SuperTux REV | `set_vsram` | `copy_vram_col` | - | - | - | - | - | - | - | - | 38 |
| D32XR CD32X | `set_scroll` (HUR genbg: plane A/B h- and v-scroll in one call) | - | - | - | - | - | - | - | - | - | 37 |
| OpenJazz REV (`d486f58`) | `set_vsram` | `genbg_load` | `genbg_start` | `genbg_off` | `genbg_cell` | `rv_copy_fast` (R7: cart ↔ Word RAM, 48 bytes a `movem`, RV = 1) | `bmd_setup` | `bmd_queue` | `bmd_off` | `cmd_nop` (free) | 46 |
| Rayman (`tools/stack_genvdp.py`, `RAY_GENVDP`) | `set_vsram` | `copy_vram_col` | `set_scroll_vb` (section 17) | - | - | - | - | - | - | - | 39 |
| Dracula X CD32X (`cd32x/genplane.patch`, `DX_GENPLANE`, section 20) | `set_vsram` | `copy_vram_col` | - | - | - | - | - | - | - | - | 38 |
| Sonic CD 2011 (Kobo path) | - | - | - | - | - | - | - | - | - | - | 35 |

So MD 36 is `set_vsram` in three listeners and `set_scroll` in D32XR's; MD 38 is `genbg_start` in OpenJazz
REV and `set_scroll_vb` in Rayman; MD 37 is `copy_vram_col` in SuperTux and Rayman but `genbg_load` in
OpenJazz REV. The `CD32X_PAGED_CART_STACK_PROPOSAL.md` plan to
move OpenJazz to MD 38-41 was not carried out: OpenJazz REV kept 37-40.

## 3. The SH2 command numbers in use (this stack)

| CMD | Meaning |
|---|---|
| 38 | open file (length → COMM8L; 0 = not on the disc). The number also picks the cart slot for the next 46. **Paged builds of Hurrican and D32XR (`CART_PAGED`): bit 15 = the destination comes in COMM8L** (`cur_dest`, `& 0x3FFFFC`, used by 46 / 47 instead of the slot; D32XR wraps it in the 4 MB) |
| 44 | clear the Genesis planes. **Always acknowledged** (`main.c` 31 Oct 2026 reading: COMM0 is cleared in every build); the clears themselves (MD 4 / 8) are compiled out while `SLIDESHOW_DEBUG` is 1, which it is in this stack - so 44 does nothing but answer |
| 45 / 54 | play CD track (bit 15 = once, OpenLara OL-4; **Sonic's Kobo path has no bit 15: repeat only**) / stop music |
| 81 | CD music pause (COMM2 = 1) / resume (0) in place: BIOS MSCPAUSEON / MSCPAUSEOFF. **Built 4 Nov 2026 behind `CDPAUSE=1`** (section 31; Metal Slug CD32X's CD player). No reply |
| 46 | chunk → cart (bit 15 = no read-back); reply COMM12 = 0 ok / 1 bad. Switches: `FMVRING` bit 14 ring, reply \| 0x100 (section 12); `CARTW` word copy, MD 27 (section 13); `STREAM` read-ahead (section 10) |
| 47 | chunk compare with the cart (`CMD_CART_TO_FB` also has the value 47 in `main.c`; its branch is never reached) |
| 48 | pad (+ sound effect in the high byte, COMM2 = its volume); reply COMM8 pad, COMM10 ticks (pad 2 when built `PAD2=1`, section 23), in one 32-bit write. Answered before any other command |
| 50 / 51 | file chunk → frame buffer (off-screen 16 KB / splash 32 KB). **Both write COMM12 32 bits wide** (chunk, sectors, a data word): the one place the stack itself overwrites COMM14 |
| 52 | Genesis display on / off |
| 53 | load sound pack `SFXnn.BIN`; reply COMM12 = number of effects, 0 = failed |
| 56 / 59 | backup RAM load / save (BIOS `_BURAM`, `GAME_BRAM_NAME`, 64 blocks = half the console's 8 KB; Rayman's `stack_save.py` cuts its own to 5 blocks, Hurrican's HB-1 to 2); reply COMM12 1 ok. **Sonic R6 (`SCD_SAVE_MIN`): COMM2 = the file, 0 progress (`SONICCD32X_`, 2 blocks) / 1 time attack (`SONICCD32XT`, 5 blocks); its ED launcher (`SCD_MODE1`) keeps both in bank 31 at `0x800000 + file × 512`.** Hurrican ED: → bank-31 piece 12 |
| 57 / 58 | buffer word get / put (COMM2 = word 0-2047 of the 4 KB buffer; 58 fills it for 59, 61 and every bank-31 command of section 16) |
| 60 | D32X: start D32XR from the cart (never returns). D32XR CD32X sets window 1 to page 7 first (section 15) |
| 61 | save area: 4 KB piece → cart 0x3F0000 + (COMM2 & 15) × 4 KB, read back; reply COMM12 1 ok. `SLOTS=1`: bit 8 → OpenLara's slot area, reply 0x101 (section 11). **ED launcher builds:** the same piece → bank 31, reply `1 \| 0x200` (section 16) |
| **62–67** | **the Genesis picture (section 1)** - **the BootCamp stack itself with `GENPLANE=1` (since `252265a`, 31 Oct 2026, section 20)**, SuperTux REV, Rayman (`RAY_GENVDP`, 62-67 as SuperTux's), Dracula X CD32X. **62 = "save piece in" on OpenLara EDCD, OpenJazz REV, Hurrican** (section 16); **62 = "Genesis background" on D32XR** (COMM2 1 load + show from cart offset COMM8L (`GBG1` pack: tiles → VRAM `0x1000`, map → plane B `0xE000`, CRAM line 3), 2 scroll (COMM8 h, COMM12 v → MD 36), 0 off; reply COMM12 1) |
| **67** | OpenJazz REV: Genesis plane (section 19) - not SuperTux's columns |
| **68** | **cart page, window 1 (section 7)**: 8 / 16 MB SRAM cart, OpenLara CD32X-REV test builds; SuperTux CD32X REV `CART_MB=8/16` builds (REV11-16MB-T1); OpenJazz REV, Hurrican, D32XR, Rayman paged builds. `CART=ED`: pages 1-6 only, four word writes to CTRL4-7 (section 16) |
| **69** | OpenJazz REV: unpack a compressed pack page → page (section 19). **SuperTux REV ED builds: "save piece in"** (section 16) |
| **70** | OpenJazz REV: the disc's stamp → COMM8L (section 19) |
| **71** | OpenJazz REV: bonus level floor on the graphics chip (section 19) |
| **72 / 73** | Rayman: bank-31 save piece out / in at bank offset `0x30000` (section 17) |
| **74** | **two owners:** Rayman: plane A scroll at the vertical blank (section 17, MD 38); BootCamp `sdcard/`: `SDSAVE_CMD_OUT` (section 18) |
| **75** | BootCamp `sdcard/`: `SDSAVE_CMD_IN` (section 18) |
| **78** | **BootCamp stack, `VSPLIT=1` (needs `GENPLANE=1`): V scroll line split** - the planes' V scroll changed part-way down the screen by the Genesis H interrupt; double-buffered list (sections 21, 21a) |
| **79** | **BootCamp stack, `FBVDP=1` (needs `GENPLANE=1 VSPLIT=1`): the Genesis VDP from a list in the 32X frame buffer** - map words, converted tiles, H scroll tables a running game makes; part kept for a commit after the 32X swap (section 24) |
| **80** | Kobo path only (Sonic CD 2011): the stamp floor (section 6). Not on this stack, but keep it clear |
| **82** | **Jumping Flash CD32X, `JFGROUND=1`: the ground on the graphics chip** (section 32; listener routines 45-48) |
| **84** | **BootCamp `traytest/` (`TRAYTEST=1`, merged into a copy of the stack): CD status** - BIOS CDBSTAT's table, and (T4) the BIOS ROM's header and the version registers (section 22). Ares only (2 Nov 2026; T3 with no cart, and with the 4 / 16 MB carts and the EverDrive PRO in place) |
| **90** | **BootCamp `traytest/` (`TRAYTEST=1`): the tray** - COMM2 1 open (DRVOPEN), 0 close and read the disc (DRVINIT) (section 22). Ares only (2 Nov 2026; T3 as 84) |
| **99** | **BootCamp stack, `TILEP=1`: a tile page's cache** - tiles copied from one window-1 page into the slots of another by listener routine 40 `tile_copy` (Metal Slug CD32X, section 25) |
| **100** | **BootCamp stack, `ZCHUNK=1`: CMD 46 from an LZ4-packed file** - a chunk unpacked from its block (Metal Slug CD32X, section 27) |

**Free on this stack (3 Nov 2026: 100 went to `ZCHUNK`, section 27; 99 went to `TILEP`, section 25; 79 went to `FBVDP`, section 24; 84 and 90 to `traytest/` on 2 Nov, section 22):** 39-43, 49, 55, **76, 77, 81, 83, 85-89, 92-98, 102 and up** (91 went to `CDRING`, section 29; 101 to `FBCART`, section 30) (78 went to `VSPLIT` on 31 Oct,
section 21; 76 / 77 are the ones section 18 asks `sdcard/` to move to). Free on every stack (this one, Kobo,
OpenTyrian): **76, 77, 81, 83, 85-89, 92-98, 102 and up** (82: Jumping Flash, section 32, 7 Nov 2026). The old line "69 and up" was behind the code by eight numbers
(69-75: OpenJazz REV, SuperTux, Rayman, `sdcard/`); BootCamp docs/03 had it right (72-79, 81+) until
Rayman and `sdcard/` took 72-75 on 26-27 Oct.

## 4. Numbers that mean something else on the cartridge stack (OpenTyrian, `cart_md.s`)

The OpenTyrian addendum lists its own numbers: **60** CD find, **61** CD read, **62 / 63** SRAM
read / write (streamed). Those belong to the cartridge stack, not this one, but they clash with
60–63 here. Code moved between the two stacks must check this table. New commands on either stack
should take numbers free on both (76 and up, 31 Oct 2026).

**Every number with more than one meaning, all stacks and forks (31 Oct 2026):**

| CMD | OpenLara path (BootCamp) | SuperTux REV | OpenJazz REV | Rayman | Kobo path (Sonic) | OpenTyrian cart stack |
|---|---|---|---|---|---|---|
| 46 | chunk → cart, bit 15 no read-back | same | same | same | 4 forms: `0x00nn`, runs `0x01nn-0x03nn` (no read-back, reply COMM8 = the Sub-CPU's `load_sum` checksum), `0xF8nn` page (ED: four CTRL word writes, page 7 refused), `0xF0nn` slot | - |
| 47 | chunk compare | same | same | same | cart → frame buffer | - |
| 60 | start D32XR | - | - | - | - | CD find |
| 61 | save piece → cart (ED: → bank 31) | same | same | same | - | CD read |
| 62 | **VRAM word (`GENPLANE=1`, since `252265a`)** / **save piece in** (OpenLara EDCD, Hurrican) / **Genesis background** (D32XR, MD 36) | VRAM word | **save piece in** (ED) | VRAM word | - | SRAM read |
| 63-66 | Genesis picture (`GENPLANE=1`) | Genesis picture | - | Genesis picture | - | 63 SRAM write |
| 67 | cart → VRAM columns (`GENPLANE=1`) | cart → VRAM columns | **Genesis plane** load / show / off / cell | cart → VRAM columns | - | - |
| 68 | cart page, window 1 | same | same | same | act file → Genesis planes | - |
| 69 | - | **save piece in** (ED) | **unpack LZ** page → page | - | - | - |
| 70 | - | - | disc stamp | - | - | - |
| 71 | - | - | bonus floor | - | - | - |
| 72 / 73 | - | - | - | bank-31 piece out / in | - | - |
| 74 | `sdcard/` **SDSAVE out** | - | - | **scroll at vblank** | - | - |
| 75 | `sdcard/` SDSAVE in | - | - | - | - | - |
| 78 | **V scroll line split** (`VSPLIT=1`) | - | - | - | - | - |
| 79 | **Genesis VDP from the frame buffer** (`FBVDP=1`) | - | - | - | - | - |
| 80 | - | - | - | - | stamp floor | - |

A Sub-CPU program belongs to one game, so none of these clashes is a run-time bug; every one is a trap for
code moved between repositories (the Rayman port merged SuperTux's 62-67 *and* OpenLara's 61/62 save
pattern into one program - it only avoided the collision by renumbering the saves to 72/73).

## 5. The RAM cart save area: pieces in use (CMD 58 + 61)

Cart `0x3F0000`-`0x400000`, 16 pieces of 4 KB (CMD 61 takes COMM2 & 15). Read straight from the
cart by the SH2 at `0x223F0000 + piece × 4096` (cache-through). Battery-backed: survives power-off.
One piece, one owner - **never reuse a piece with a different meaning**, the same rule as the
command numbers.

| Piece | Cart | Owner |
|---|---|---|
| 0-4 | 0x3F0000-0x3F4FFF | OpenJazz CD32X: `openjazz.cfg`, `SAVE.0`-`SAVE.3` |
| 5-9 | 0x3F5000-0x3F9FFF | Holiday Hare '95 (OpenJazz CD32X) |
| 10 | 0x3FA000 | OpenJazz CD32X port settings (J19) |
| 11 | 0x3FB000 | **DOOM CD32X Fusion (D32XR CD32X)**: the game's saves (section 15) |
| 12 | 0x3FC000 | **Hurrican CD32X, EverDrive launcher builds only** (`CART_EDSAVE`, 26 Oct 2026): the high-score table that its CMD 56 / 59 used to put in the Sega CD's backup RAM, rerouted in the Sub-CPU program to **bank-31 piece 12** (`0x800000 + 12 × 4 KB` in the listener's map). The 4 MB cart's piece 12 is never written by Hurrican, but the number is Hurrican's on both: one piece, one owner. The BootCamp's `hello/` borrows it for its boot counter (magic `0x4243`): a demo, not an owner - a game copied from `hello/` must take another piece, and there is none free |
| 13 | 0x3FD000 | **SuperTux CD32X REV5**: options, high scores, 7 game slots ("STX2", 368 bytes) |
| 14 | 0x3FE000 | **OpenLara CD32X-REV** (SAVE-2): the Caves record `'OLCV'` at `0x3FE000` and the level slot record `'OLLV'` at `0x3FE010` (section 8). Rewritten after every level load from the CD |
| 15 | 0x3FF000 | **Cart test scratch** (OpenLara CD32X-REV `CART=` builds; SuperTux CD32X REV `CART_MB=8/16` builds - its size probe writes 8-byte markers here on pages 1, 2 and 6 at every start): a 4 KB test pattern per page, written and read back at every start of a test disc. Never holds data worth keeping; a game must not store anything here. D32XR CD32X (section 15) also writes its board probe markers and a `D32XR PRELOAD OK` hint here; losing it only makes the next boot recheck |

**Free: none** (31 Oct 2026; 12 went to Hurrican's ED builds on 26 Oct, see the row). A new game on
the shared 4 MB cart must share a piece by agreement with its owner, or save to the Sega CD's backup RAM
(56-59, a few hundred bytes), or - on the EverDrive - to its own launcher's bank 31, where the whole
256 KB is that launcher's alone (section 16). **On bank 31 the piece numbers are per launcher, not
shared:** OpenLara's bank-31 piece 0 and OpenJazz's bank-31 piece 0 are in different `bram.srm` files.
(OpenLara CD32X uses piece 0 and on: `0x3F0000` settings, `0x3F0400` saved game; it and
OpenJazz share the cart only one at a time. Its Caves and level slot records were at `0x3F0300` /
`0x3F0310` in piece 0 until SAVE-2 moved them to piece 14, so a level load no longer rewrites the piece
that holds the settings and the saved game. A build with SAVE-2 ignores the old records: it loads Caves
and the level from the CD once.)

**Outside this area (OpenLara CD32X-REV v1.6, SLOTS-1):** OpenLara's 16 save slots are at cart
`0x060000`–`0x07FFFF`, in OpenLara's own layout (between its game image and Caves), written by CMD 61
with COMM2 bit 8 (section 11). They are not pieces of the shared save area above.

## 6. The Kobo stack (Sonic CD 2011 CD32X): its numbers, and the paging cases

Sonic CD 2011 CD32X runs on the **Kobo boot path**, not on this ledger's stack:
- Sub-CPU `cd32x/sub_main.c`;
- 68000 listener `cd32x/kobo/hw_md.s`: routines 1-34 as the ledger PDF, and **`MD_CMD_END` = 35** (no `start_d32xr`
  there);
- SH2 boot loader `cd32x/sh2_engboot.c`.

Its SH2 commands are recorded here the way section 4 records OpenTyrian's, so code moved between the stacks can be
checked. Design and status: [CD32X_PAGING_SONICCD.md](CD32X_PAGING_SONICCD.md).

| CMD | Meaning on the Kobo stack (Sonic) | Same as this stack? |
|---|---|---|
| 38 | open file (length → COMM8). The file number picks the file and its 128 KB cart slot (`cur_img`): 16 `KOBOGFX.BIN` → slot 16, 21 `SONICCD.PAK` → 0, 22 `SCDDATA.PAK` → 24, `0x100`-`0x4FF` scene images → 4, `0x1100`-`0x14FF` act files → 28 | open: yes; the file numbers are Sonic's |
| 44 | clear the Genesis planes | yes |
| 45 / 54 | play CD track / stop | yes |
| **46** | chunk → cart, with three forms selected by COMM2: **a)** `0x00nn` one chunk, read back by the Sub-CPU (reply COMM12); **b)** `first \| more << 8` (`0x0100`-`0x03FF`) a run of up to 4 chunks in one CD read; **c)** two cases added for the 8 / 16 MB cart, see below | **no:** this stack's bit 15 means "no read-back"; on the Kobo stack `0x8000`+ is not a read-back flag |
| **47** | cart → frame buffer (`CMD_CART_TO_FB`: COMM2 = slot << 8 \| chunk) | **no:** here 47 is chunk compare |
| 48 | pad | yes |
| 50 / 51 | file chunk → frame buffer | yes |
| 52 | Genesis display on / off | yes |
| 53 | load a sound bank: COMM2 = bank (0 `KOBOSFX.BIN`, n `SFXn.BIN`) | load sound pack; the bank number is Sonic's |
| 56 / 59, 57 / 58 | backup RAM load / save, word get / put | yes |
| **68** | **act file → Genesis planes** (`SGB_CMD`: COMM2 = op, COMM8 = the file's cart offset) | **no:** on this stack 68 is window 1's cart page (section 7). The Kobo stack pages with command 46 `0xF8nn` instead |

Not on the Kobo stack: 60, 61 (no cart save area: Sonic saves to the backup RAM), 62-67.

**The two paging cases inside command 46** (`sub_main.c`, 23 lines against Sonic's R4D; no new SH2 or MD number):

| COMM2 | What it does | Reply | Status |
|---|---|---|---|
| `0xF800 \| window << 8 \| page` | the page byte into Word RAM, then MD 26 `SAFE_CART_XFER` (interrupts off, RV = 1) copies it to `0xA130F0` (window 0) or `0xA130F2` (window 1): one even-byte /UWR write, the page on D8-D10 | COMM12 = 0 | **run on the console** (Sega CD + 32X + 4 MB RAM cart, 19 Oct 2026): the disc booted, the command was answered twice, the 4 MB cart showed no effect. A real page switch is untested (it needs the 8 / 16 MB board) |
| `0xF000 \| slot` (0-31) | the file command 38 opened goes to 128 KB slot `slot` (`cur_img`) until the next command 38 | COMM12 = 0 | **run on the console** (the same console and 4 MB cart, 19 Oct 2026): the 256 KB test file went to slot 21 (cart `0x2A0000`) and read back right on the Sub-CPU and the SH2; its own slot (16) was not written |

**Sonic R4K (20 Oct 2026): faster loading.** These are two changes inside existing commands; **no new SH2 or MD
number**, and no change to what any command takes or returns. The SH2 side, `sh2_engboot.c` and the engine, is the same
as R4J's.

| Where | What changed | Used by | Why |
|---|---|---|---|
| SH2 **38** open (Sub-CPU `open_file`, `cd32x/kobo/sub_stubs.c`) | The first open after power-on reads the disc's whole root directory (8 sectors) into Sub-CPU RAM in **one** CD read (`dirc[]`, 48 KB, `.bss`). Every later open looks the name up there, with no disc access. A name not in the root returns not-found at once (`scd.c`: no `/ROMS` search, no root listing). If the root is longer than 24 sectors, or its first record is not `.`, there is no cache and every open goes the old way. | every command 38, and the Sub-CPU's own opens (the sound banks, command 53) | the old open read the root one sector at a time, each read a new seek: **about 1.7 s per open** (emulation trace) |
| MD **33** `CHILLY_RV_MEMCPY` (`cd32x/kobo/hw_md.s`) | when source, destination and length are all multiples of 4, it copies with `move.l` (two word writes to the 16-bit cart, the same bytes in the same places), about 9.5 clocks a byte instead of 42. Anything else (the 2-byte cart probe) takes the old byte loop. Interrupts stay on, RV = 1 as before. | command 46 forms a) and b): each chunk run goes Word RAM `0x200000` → cart in 16 KB pieces | the byte loop was **0.7 s of every 2.0 s** per 128 KB loaded (emulation trace) |

The other stacks have their own copies of these files: this changes Sonic's Kobo stack only. A stack that wants the
same speed-up can take `open_file` / `dirc_*` from Sonic's `sub_stubs.c` and the aligned path of `chilly_rv_memcpy`.

**Results:**
- **emulation** (the same cold boot of the R4J and R4K discs without music, a fresh 16 MB cart):

  | | R4J | R4K |
  |---|---|---|
  | `SCDDATA.PAK` open → `SONICCD.PAK` open | 5.5 s | 2.4 s |
  | power-on → the title running | 87.5 s | **67.9 s** |
  | the title demo's act starts | 134.6 s | **109.6 s** |

- **The owner:** reports that R4K works (20 Oct 2026).
- **Console:** not burned yet.

**Sonic R5C (21 Oct 2026): the special stages' floor on the Sega CD's graphics chip.**
- **One new SH2 → Sub-CPU number, 80**, in `cd32x/sub_main.c`. Only the Sub-CPU can drive the stamp / rotation chip,
  so the floor needs a Sub-CPU command.
- **No new 68000 listener routine:** `kobo/hw_md.s` is R4K's. Command 80 uses the existing MD routines 7
  (`copy_vram`), 19-22 (32X / VDP reads and writes) and 34 (`set_vdp_reg`).
- `sh2_engboot.c`, `kobo/sub_stubs.c` and `kobo/scd.c` are unchanged.

| CMD 80, COMM2 low byte | What it does | Reply |
|---|---|---|
| 0 | begin: COMM2 high byte = special stage 1-8. Opens `FLOOR<n>.BIN` (256 KB, `tools/build_floor.py`: stamps at 0, stamp map at 0x20000, header 'FLR1'), switches Word RAM to 2 Mbit mode (MODE written, then read back until set), reads the file into it, and sets the Genesis up: window plane rows 15-27 (lines 120-223), palette line 3, two name tables (0xA000 / 0xB000) over two image buffers (0x0020 / 0x4420) | COMM12 = 1 on, 0 no floor (no file, wrong length or header, more than 512 stamps) |
| 1 | COMM8 = the floor's X << 16 \| Z, 13.3 each | – |
| 3 | COMM8 = the floor's height (the engine's YPos), sent when it changes | – |
| 4 | COMM8 = sin << 16 \| cos of its angle. The Sub-CPU writes the trace table (one line per screen line, the engine's `Draw3DSkyLayer` arithmetic) and starts the chip. Then, between other commands: the image to the main CPU (RET), into the hidden buffer in 1024-word pieces (MD 7), and that buffer's name table shown (register 3) after the next vertical blank | COMM12 = 1 on, 0 the floor was ended |
| 5 | COMM2 high byte = colour k (1-15), COMM8 = the Genesis colours k and k + 1 (CRAM line 3, MD 22) | – |
| 6 | COMM2 high byte = colour k: the floor file's palette index for it | COMM8 |
| 7 | end: display off, Word RAM back to 1 Mbit mode (0x05, read back), the listener's set-up (MD 1), planes cleared (MD 4 / 8) | – |

- **While the floor is on, Word RAM belongs to it.** Any command other than 0, 80, 45, 48, 52, 54, 57 and 58 ends the
  floor first: the loads, the sound banks, the backup RAM file, the cart pages. The SH2 sees COMM12 = 0 on its next
  op 4 and draws the floor itself again.
- **No green diagnostic text** while the floor is on (`print_diag`). It would write Word RAM.
- **The Sub-CPU's own `switch_banks`** (the diagnostic print, `scd_open_file`'s checkpoint line) must not run while the
  floor is on. In 2 Mbit mode it would hand Word RAM to the main CPU, and the Sub-CPU's next Word RAM access hangs it
  (Mega-CD manual p.24). Only the command list above keeps them out: anyone adding a command to that list must check
  this.
- **Status:**
  - emulation: special stage 1 drawn by the chip;
  - sh2emu: answers command 80 as the Sub-CPU does, without drawing the chip's picture;
  - console: R5C itself was not burned; command 80 was **run on the console as part of R5G** (below).

**Sonic R5G (21 Oct 2026): command 80 extended – the title, the special stages' skies, the DMA copy.** R5G includes
R5D, R5E and R5F, which were never burned on their own.
- **No new SH2 → Sub-CPU number.** Everything is inside command 80, in `cd32x/sub_main.c`.
- **No new 68000 listener routine:** `kobo/hw_md.s` is still R4K's. The new work uses its existing routines 7
  (`copy_vram`), 19-22 (32X / VDP reads and writes) and 34 (`set_vdp_reg`), and its existing record handling
  (`sgb_poll` / `sgb_vblank`, flags `SGB_FLAG` - `SGB_BAD` at `0xFF8000`-`0xFF8006`), which command 68 already uses for
  the acts.
- `sh2_engboot.c` (`SH2-VR4I`), `kobo/sub_stubs.c` and `kobo/scd.c` are unchanged.

| CMD 80, COMM2 low byte | What changed in R5G (R5C's meaning otherwise) |
|---|---|
| 0 begin | COMM2 high byte = **1-9**. **9 = the title** (`FLOOR9.BIN`, 0x40000 + 0x4000 bytes): the clouds on the window plane rows 0-12 (lines 0-103), the mountains and water on plane B, the planet as 8 hardware sprites. **1-8** (`FLOOR<n>.BIN`, 0x40000 + 0x8000 bytes; special stage 1 + 16 × 7680 bytes for its swirl): the floor as before, plus the stage's sky on planes A and B, lines 0-119. Any other length = no floor (COMM12 = 0). The begin also turns the listener's record handling on (`SGB_FLAG` = 1, `SGB_LAST` / `SGB_READY` / `SGB_BAD` = 0, COMM6 = 0) |
| 1 | the title: COMM8 = the clouds' scroll value. The Sub-CPU adds 8 a frame between the SH2's pictures and starts the next picture as soon as the last copy ends. Special stages: as R5C |
| 4 | the copy into video RAM is now the **VDP's DMA from Word RAM** (see below), not the 68000 word by word. The buffer switch (register 3) happens only in the vertical blank or the screen's first lines (the VDP's line counter, read with MD 21), otherwise at the next vertical blank |
| 5 | **no longer used:** the colours come in the SH2's records |
| 6 | COMM2 high byte = the CRAM slot (the title: 1-47; a special stage's floor: 49-63, its sky 1-47) → COMM8 = its palette index in the file |
| **8 (new op)** | special stage 1 only: COMM8 = the swirl's picture (0-15) for this frame. The Sub-CPU keeps the 16 pictures in its own RAM, copies the wanted one into Word RAM (`0x14200`) while the chip draws, DMAs it into the hidden one of two plane B buffers (`0xA000` / `0xE000`) and switches register 4 in the vertical blank |
| 7 end | records off first (`SGB_FLAG` = 0), then as R5C, and the **Genesis display left off** (register 1 = `0x34`) |

**The SH2 sends one record a picture through the listener** (the same path as the acts' planes, `md/sgb.h`): each line's
plane scroll, the vertical scroll, the title's planet sprite heights, and every colour that changed (CRAM 0-63). The
68000 writes it to the VDP in the vertical blank, unchanged code.

**The DMA from Word RAM, for any stack that copies the chip's picture this way:**
- The Sub-CPU issues it through MD 22 (one long write to the VDP control port `0xC00004`): registers 15 / 23, 21 / 22,
  19 / 20 two at a time, then the destination with the DMA bit. The 68000 waits while it runs.
- **A DMA from Word RAM writes a wrong first word** (the word read before it). Each piece therefore starts one word
  early. The pieces go from the buffer's end to its start, so each wrong word lands on a word the next piece writes
  again. The last one lands in a spare cell before each buffer.
- **The 68000 must not write a VDP address and its data as two commands** while the records are on: the vertical-blank
  handler can come between them and set an address of its own. Only single long writes and register writes are used.
- About twice as fast as the word-by-word copy.

**Which commands end the floor** (they need Word RAM): any except 0, 45, 48, 52, 54, 56, 57, 58, 59 and 80. **New in
R5G: 56 and 59** (backup RAM load / save) no longer end it - they use the BIOS's own buffers, not Word RAM. Command 48
with a sound effect in the high byte (Sonic's K33) is answered by the pad fast path before this test, so it never ends
the floor either.

**COMM4 at the end.** While the records are on, COMM4 holds the last record id. The Sub-CPU's start-up check prints
`SH2 NEVER EVEN REACHED THIS FILE` for any COMM4 other than 0, `0x1234` and `0x0EEE` once the planes are off. The SH2
therefore clears COMM4 when it ends the floor. Another stack reusing the records outside an act must do the same.

**Results:**
- sh2emu: all 31 check runs `no problems found`. Special stage pictures a second (last 10 s): 20.0 / 28.4 / 28.2 /
  30.0 / 15.5 / 14.9 / 17.7 / 29.9 for stages 1-8 (R5C: 8.6-11.7); the title 18.1.
- emulation: the title, its fade to the menu, special stages 1, 2 and 6.
- **console: works** (Sega CD + 32X + 16 MB cart, 21 Oct 2026, the owner's report).

Details: the Sonic port's `docs/R5G_NOTES.md`.

**Sonic R5H (22 Oct 2026): command 80 op 4 answered first.** No new number, no new op, no listener change.
- **`cd32x/sub_main.c`, op 4:** the Sub-CPU now writes its answer (COMM12) and COMM0 = 0 as soon as it has read the
  angle, then writes the trace table and starts the chip. Before, the SH2 waited through the trace table: 100 lines of
  32-bit arithmetic on the Sub-CPU's 68000.
- **On the SH2 side,** op 1 (X, Z) is sent at the picture's first draw and op 4 is not waited for. Its answer is read
  before the SH2's next command, or at the picture's end.
- **For other stacks using the same pattern:**
  - A command answered early must not be followed by a new one until COMM0 is 0 again.
  - Its reply register must be read before any later command can overwrite it.
- **Data, not code:** `FLOOR2.BIN` - `FLOOR8.BIN` list plane B's name table for rows 0-27 (layer 1 under the floor),
  in the same block list op 0 already copies.
- `kobo/hw_md.s`, `kobo/sub_stubs.c`, `kobo/scd.c` and `sh2_engboot.c` are unchanged.
- **The 32X frame buffer rule this round ran into:** a **byte** write of 0 into the frame buffer (not the overwrite
  image) does not happen. A word or long write of 0 does. Keep tables in the frame buffer as 16-bit words.
- **Status:**
  - sh2emu: all 31 check runs `no problems found`;
  - emulation: special stages 1, 2, 6, 7 and the title;
  - **console: not burned yet.**

Details: the Sonic port's `docs/R5H_NOTES.md`.

## 7. SH2 → Sub-CPU command 68: cart page, window 1 (OpenLara CD32X-REV)

For the paged 8 / 16 MB SRAM cart (the 16MB-PCB-CD32X board): the SH2s can't reach the cart's
page register, so the Sub-CPU has the 68000 write it. Design: [CD32X_SRAM_PAGING_PROPOSAL.md](CD32X_SRAM_PAGING_PROPOSAL.md)
(variant B); code: MicroNut99/OpenLaraCD32X-REV, branch `cart-test-8mb-16mb` (`subcpu/main.c`,
`CART-1`).

| CMD | Name | COMM2 | Reply | How |
|---|---|---|---|---|
| 68 | cart page, window 1 | page 1–7 (page 0 refused: the program and resident data stay in window 0) | COMM12 = 1 done / 0 refused | page byte → Word RAM, `switch_banks`, then the existing 68000 routine **26 `safe_cart_xfer`** (interrupts off, RV = 1) copies it to **`0xA130F2`**: one even-byte /UWR write, the page on D8–D10 |

- **No new 68000 routine.** `hw_md.s` is unchanged, and so are the boot block, CDLOADER, `crt0.s`
  and `src-md/`. Nothing sends 68 during boot.
- **Compiled only in the test builds:** `CART=8` / `CART=16 bash build.sh` (`OL_CART_MB`, passed to
  `main.c` through `HWFLAGS`). Without `CART` the Sub-CPU has no command 68.
- **The SH2 side:** after 68, clear both SH2 caches. Settings and saves stay on page 1 (cart
  0x3F0000), so put window 1 back on page 1 before using the save area (CMD 58 / 61).
- **The same write on the Kobo stack** is command 46 `0xF8nn` (section 6). Code moved between the
  stacks must change the number.

**Status: run on the console** (Sega CD + 32X + **4 MB** RAM cart, 19 Oct 2026, the owner's report):
the OpenLara CD32X-REV `CART=16` test disc booted and its cart test showed exactly the expected
4 MB result: `FOUND 4MB`, pages 1–6 `SAME AS 7`, page 7 `OK` (every write landed in page 1, as the
4 MB cart has no page register), `SAVE AREA UNCHANGED`; the game played afterwards. So command 68
is answered and harmless on the 4 MB cart. **A real
page switch is untested on the 8 / 16 MB PCB board.** On the Mega EverDrive PRO the switch itself is
console-measured (`edpage`, 26 Oct 2026: pages 1-6 through CTRL4-7 word writes, and the listener's `m1_page`
lifted into the probe), but the games' first EverDrive builds switched no page at all, because this command's
one even byte to `0xA130F2` is ignored there - section 16 has the fix.

### 7a. SuperTux CD32X REV: the same command 68 (REV11-16MB-T1, 20 Oct 2026)

SuperTux CD32X REV uses command 68 exactly as above - same number, same COMM2 / COMM12, the same
68000 routine 26 `safe_cart_xfer` to `0xA130F2`, no new 68000 routine. Code: MicroNut99/SuperTuxCD32X_REV,
`cd32x/subcpu/main.c` (the block inside `#if STX_CART_PAGED`), `game/cart.c`, `game/yt_mars.c`.

- **Compiled only in `CART_MB=8` / `CART_MB=16 ./build.sh`:** `-DSTX_CART_PAGED=1` in the Makefile's
  **`HWFLAGS`** (main.c is compiled with `HWFLAGS`, not `CCFLAGS`). `tools/check_boot.py` accepts
  exactly that block (fingerprinted); the normal 4 MB build has no command 68.
- **Lesson for every stack: check the switch reached `main.c`.** SuperTux's first two test discs
  put the switch in `CCFLAGS`, so 68 was not in the Sub-CPU program. Nothing answered it: the SH2
  waited out its command timeout (about 35-50 s each) and the console sat on a blank screen for
  about 2 minutes after the loading bar, then went on as a 4 MB cart. `build.sh` now stops unless
  main.c's compile line carries `STX_CART_PAGED=1`. An unanswered command looks like a hang, not
  an error.
- **What the game does with it:** a size probe at every start (markers in save piece 15 of pages
  1, 2 and 6 → 4, 8 or 16 MB on the title); on a 16 MB cart all 70 level packs get their own
  place in pages 1-7 (0x280000-0x3EFFFF of each page) and are copied in from the disc at the
  first start (once; checked by their sums at later starts). Window 0 is never paged; saves stay
  on page 1 (window 1 is put back on page 1 before every CMD 58 / 61); both SH2 caches are purged
  after every page change.
- **Status:** run in emulation (headless, the 16 MB board emulated): the probe finds
  4 MB and 16 MB, the first-start copy, the resume after a power-off, later starts with no copying,
  the 4 MB cart unchanged. On the console: test disc 2 (without 68) booted on the 4 MB cart
  after the stop; the fixed build is not yet run on the console, and a real page switch still
  needs the 8 / 16 MB board.

## 8. OpenLara CD32X-REV RESUME-1: the level stays in the cart after power-off (no new command)

**What was done.** The cart is battery-backed, so the level in the level slot (cart `0x200000`, up to
`0x3EFFFF`) is still there at the next power-on, like the save. After every level loaded from the CD,
the SH2 now writes a record into the save area (piece 14 since SAVE-2, section 5; it was in piece 0 at
`0x3F0310`):

| Cart | Contents |
|---|---|
| `0x3FE010` | `'OLLV'` (written last, after a complete load) |
| `0x3FE014` | level number (`LevelID`) |
| `0x3FE018` | file length (bytes loaded by command 46) |
| `0x3FE01C` | checksum of the level as it is in the cart |
| `0x3FE020` | ~checksum |

The first level after power-on (Current Position from the passport, or a level picked there) checks
it: same level, same length as the file on this disc (command 38, lookup only), and the checksum of
what is in the cart now matches. If so, the level is **not loaded from the CD**: the game goes
straight to the level's sound effects (command 53, only for a level without its own sound bank since
SFX-4) and starts. Anything else (first start, a movie
or the cart test's load test used the slot, a load cut off by power-off, a different disc) loads
from the CD as before and writes a new record. **A + B + C** held = always from the CD. Same rules as
CAVES-1's Caves record at `0x3FE000` (was `0x3F0300`; the title level, cart `0x080000`).

**The fast load advantage.** Emulation, Lost Valley, Current Position after a power cycle: **about
28 s → about 3 s** (the checksum read of the level, ~1.9 MB at most, and the sound effects). Within
one session nothing changes: going back to the level already in the slot was already instant (OL-2c),
a new level still loads from the CD. Not tried on the console yet.

**Seen only during a preload of a new level (a CD load), never on a resume from the cart:**
- the music stops (command 54: no CD audio while data is read);
- the splash picture with the red progress bar;
- COMM12 = `0x4Fnn` (nn = level) while loading, then `0x4E14` (level ready); `0x4EEn` = load failed;
- the extra save-area write of the new record (commands 58 + 61, one 4 KB piece) after the load.

On a resume the passport stays on screen for those ~3 s, then the level starts; COMM12 goes straight
to `0x4E14` (no `0x4Fnn`).

**Boot stack: nothing added.** RESUME-1 uses only existing commands: SH2 38, 46, 53, 54, 58, 61.
No change to the Sub-CPU `main.c`, `hw_md.s`, the boot block, CDLOADER, `crt0.s` or `src-md/`.
The only boot stack addition for OpenLara CD32X-REV is command **68** (section 7), in the `CART=`
test builds only. Code: MicroNut99/OpenLaraCD32X-REV, branch `cart-test-8mb-16mb`,
`engine/openlara-opt2.patch` (`main.cpp`: `slotInCart`, `slotRecord`, `osLoadLevel`), `docs/OPT.md`.

## 9. OpenLara CD32X-REV SFX-1/2 and MUSIC-1: sound effects and the music switch (no new command)

**Boot stack: nothing added.** No change to the Sub-CPU `main.c`, `hw_md.s`, the boot block, CDLOADER,
`crt0.s` or `src-md/`. The only boot stack addition for OpenLara CD32X-REV is still command **68**
(section 7, `CART=` test builds only).

What changed is in the game (engine patch `engine/openlara-sfx.patch`, tools `tools/sfx_bank.py`,
`tools/make_sfx_packs.py`; MicroNut99/OpenLaraCD32X-REV, branch `cart-test-8mb-16mb`, `docs/SFX.md`):

| What | Commands used | Notes for other stacks |
|---|---|---|
| **SFX-1**, Sega CD PCM effects (the fallback) | **48** pad request + effect in the high byte, COMM2 = volume; now **up to 3 times per frame** (was once) | A second or third command 48 in the same frame is allowed: it plays the effect and reads the pad again (the game keeps the first pad value). The Sub-CPU driver plays 8 channels. |
| SFX-1 packs | **53** load sound pack (`SFX<level>.BIN`) | Pack format unchanged (`KSFX`, at most 31 effects, 64 KB wave RAM); only the packer's choice of samples changed. **SFX-4:** not sent for a level whose file has its own sound bank (SFX-2), so no file open + read and no music stop for it. |
| **SFX-2**, all effects through the **32X PWM** (the default) | none on the Sega CD | The samples are appended to each level file (`.PKD`) and loaded into the cart by the normal level load (**38** + **46**). The mixer runs on the **slave SH2** in its DMA 1 interrupt (`sec_dma1_handler`, the existing crt0.s vector, level 4); the slave now runs at interrupt level 3 instead of 13. Its 32X interrupts stay off (INTMSK 0). The movie player stops the mixer while a movie plays. |
| **MUSIC-1**, Options → Sound → Music | **45** play / **54** stop | The switch now stops the CD audio and keeps it off (cut-scene voices included). |

- **Level files are bigger** with SFX-2 (200–500 KB of samples each), so a new level loads about 30% longer
  from the CD. RESUME-1 (section 8) covers the whole file, samples included.
- **Status: run on the console** (Sega CD + 32X + RAM cart, 21 Oct 2026, the owner's report): the effects
  play through the PWM at the right volume.

## 10. OpenLara CD32X-REV STREAM-1: the drive reads ahead during commands 46 / 47 (Sub-CPU `main.c`, switched)

**Boot stack: one change, behind a switch.** The Sub-CPU `main.c` gets the drive read-ahead from
OpenJazz CD32X R7 / Hurrican CD32X P2b, inside `#if OL_STREAM`. `subcpu/Makefile` passes it in
**HWFLAGS** (`STREAM ?= 1`, so it is on). `make cd STREAM=0` or `STREAM=0 bash build.sh` builds
`main.c` exactly as before (the preprocessed source is identical). No new command number, and no
change to `hw_md.s`, `scd.c`, the boot block, CDLOADER, `crt0.s` or `src-md/`: the two BIOS stream
calls are inline assembly in `main.c`.

| What | Before | With STREAM-1 |
|---|---|---|
| Command **46** / **47** chunk (32 KB) | `scd_sub_read`: CDCSTOP + ROMREADN for that chunk (a seek every chunk) | `cd_chunk_read`: ROMREADN once to the end of the file; each sector's header (minute/second/frame) is checked, and a missed or wrong sector starts the read again at the sector wanted. After 64 failures it falls back to `scd_sub_read` |
| Between chunks | the drive is stopped | `cd_pump` takes the sectors that arrive into a 64 KB ring (`st_buf`, 32 sectors) while the 68000 copies to the cart, during the read-back compare and while the Sub-CPU waits for the next command |
| Everything else that uses the drive (`scd_sub_read`, `scd_open_file`, `scd_bios_call`: file opens, PCM packs, CD audio) | — | stops the stream first (macros), so the next chunk starts afresh |

- **BIOS calls used** (`jsr 0x5F22`): CDCSTOP `0x89`, ROMREADN `0x20`, CDCSTAT `0x8A`, CDCREAD `0x8B`
  (CDC destination set to the Sub-CPU: `0xFF8004` bits 8–10 = 3), CDCTRN `0x8C`, CDCACK `0x8D`.
- **What the SH2 sees: nothing new.** Same commands, same replies, same COMM registers. Only the
  time a level takes to load changes.
- **Memory:** the ring adds 64 KB to the Sub-CPU program's bss (program RAM ended at `0x1E7A0`, now
  about `0x2E7A0`; the stack is at the top, `0x80000`).
- **Other stacks:** the same code is in OpenJazz CD32X (R7) and Hurrican CD32X (P2b), both run on
  the console. How it works and the load times measured there: `CD32X_STREAM_READ.md`. Any game on this stack that loads with command 46 can take it the same way.
- **Status: run in emulation in OpenLara (v1.6 tests): the first level load 40 s → 17 s. Not yet on the
  console in OpenLara (on the console in OpenJazz and Hurrican).** Note (31 Oct 2026): `CD32X_STREAM_READ.md`
  still says "measured in ares, not yet on the console"; this section and OPENLARA_CD32X_RELEASE.md say
  OpenJazz R7 / Hurrican P2b ran it on the console. The two were not reconciled; docs/07 in the BootCamp
  lists OpenJazz R7-R9 as emulation-only. Treat STREAM as **emulation-proven** until a dated console run is
  written into one of these files. The BootCamp ships it off by default for that reason.

## 11. OpenLara CD32X-REV SLOTS-1: command 61 writes the save slot area (Sub-CPU `main.c`, switched)

**Boot stack: one change, behind a switch** (`#if OL_SLOTS`, `subcpu/Makefile` HWFLAGS `SLOTS ?= 1`;
`SLOTS=0` builds command 61 as before). No new command number.

| Command 61 | COMM2 | Writes | Reply COMM12 |
|---|---|---|---|
| as before | piece 0–15 (bit 8 clear) | cart `0x3F0000 + (COMM2 & 15) × 4 KB` | `1` ok / `0` failed |
| **slot area** | bit 8 set, piece 0–31 | cart **`0x060000 + (COMM2 & 31) × 4 KB`** (`0x060000`–`0x07FFFF`) | **`0x101`** ok / `0` failed |

- The data still comes through command 58 (the 4 KB buffer); the copy and the read-back check are
  the same as for the save area.
- **How the SH2 finds out:** at start OpenLara rewrites slot piece 31 with what it holds already. `0x101`
  = the slots are there; `1` = an older Sub-CPU (it took `31 & 15` and wrote save piece 15, the cart
  test scratch piece of section 5, which never holds anything worth keeping): single save as before.
- The area belongs to OpenLara's cart layout: its game image must end before `0x060000` (`build.sh`
  stops above 384 KB). Another game on this stack that wants slots should pick its own place in its
  own layout and say so here.
- Status: run in emulation (save, list, load; old Sub-CPU: single save). Not yet on the console.

## 12. OpenLara CD32X-REV FMV-2: command 46 ring mode (Sub-CPU `main.c`, switched)

**Boot stack: one change, behind a switch** (`#if OL_FMVRING`, `FMVRING ?= 1`; `FMVRING=0` builds
command 46 as before). No new command number.

| COMM2 bits | Meaning |
|---|---|
| 15 | no read-back (as before, OL-13) |
| **14** | **ring mode**: file chunk `COMM2 & 0x1FFF` goes to ring slot `chunk % 60` of the image slot (60 × 32 KB) |
| **13** | (ring mode) to slot **60** instead: the spare after the ring's end, a copy of slot 0, so a file chunk across the end stays in one piece |

- **Reply:** COMM12 = `bad | 0x100` in ring mode. An older Sub-CPU masks COMM2 with `0x7FFF`, writes
  nothing (offset past the file) and replies `0`: the SH2 then falls back to loading the movie whole.
- **While it copies in ring mode** the Sub-CPU writes **COMM12 = `0x8000`** (before the first 4 KB
  piece) and `0` (after the last), then the reply. The slave SH2 does not read the cart while bit 15 is
  set, so RV never holds it while its PWM block runs out. (Two extra 68000 commands 20 per chunk.)
- Only movies use ring mode; levels load as before.
- Status: run in emulation (CANYON, 20 MB, streamed whole at 15 pictures a second, the sound in real
  time). Not yet on the console.

## 13. OpenLara CD32X-REV CARTW-1: command 46 copies words (Sub-CPU `main.c`, switched)

**Boot stack: one change, behind a switch** (`#if OL_CARTW`, `CARTW ?= 1`; `CARTW=0` builds the
copy as before). No new command number, no listener change: it uses the existing **68000 command
27 `safe_cart_xfer_w`** (`hw_md.s`: interrupts masked, RV = 1, `move.w` loop, RV = 0) instead of
**33 `chilly_rv_memcpy`** (byte loop).

| | Command 33 (before) | Command 27 (CARTW-1) |
|---|---|---|
| Arguments | destination, source, length | **source, destination**, length |
| Loop | `move.b` + `subi.l` + `bgt`: about 42 clocks a byte | `move.w` + `subq.l` + `bgt`: about 15 clocks a byte |
| Piece | 16 KB (ring mode 4 KB) | 4 KB (about 8 ms with the 68000's interrupts masked: less than a frame, so the vertical blank waits but is never lost) |

- Lengths are multiples of 2048 and all addresses are even (Word RAM `0x200000`, cart `0x400000 + …`).
- The cart must take 16-bit writes: the 4 MB RAM cart and the 16MB-PCB-CD32X do (`/WE` with byte
  enables). Command 61 (the save area) still uses command 33.
- Status: run in emulation (levels and movies). Not yet on the console.


## 14. OpenLara CD32X-REV UB-1 and PICS-1: more files through command 38 (Sub-CPU `main.c`, switched)

**UB-1, boot stack: one change, behind a switch** (`#if OL_UB`, `subcpu/Makefile` HWFLAGS `UB ?= 1`;
`UB=0` builds the list as before). No new command number.

- `ol_level_files[]` gets four more names after `LEVEL10C.PKD`: `EGYPT.PKD`, `CAT.PKD`, `END.PKD`,
  `END2.PKD` (Unfinished Business, LevelID 21–24). So command 38 with COMM2 = **121–124** opens them
  into the level slot (and 221–224 into the Caves slot, never used), exactly like levels 100–120.
- Without the switch, 121–124 fall through to `IMAGE121.RAW` … (not on the disc): length 0. OpenLara
  asks for file 121 once at start and leaves Unfinished Business out of the passport when it is 0.

**PICS-1: no Sub-CPU change.** Command 38's existing fallback opens `IMAGEn.RAW` for any index not
in another range and command 46 writes it to cart **n × 128 KB**. OpenLara uses **17, 18, 19, 21, 22**
(area loading pictures) and **23–27** (credit pages): cart `0x220000`–`0x37FFFF`, inside its level
slot, loaded just before the level (which overwrites them) or after the END movie. Never 16 (a pack
name) or 20 (`OPENLARA.32X`). Each file is 72 192 bytes (3 chunks). Another game on this stack using
`IMAGEn.RAW` must keep n × 128 KB inside its own free cart space.

- Status: run in emulation. Not yet on the console.

## 15. DOOM CD32X Fusion (D32XR CD32X): the game from the CD into the 8 / 16 MB cart (no new command number)

D32XR (DOOM CD32X Fusion v3.0, viciious/d32xr) booted from the disc on this stack, Hurrican CD32X
P2a2's: Sub-CPU `main.c`, 68000 listener `hw_md.s`, SH2 loader `cdloader_sh2.c`. Code:
MicroNut99/d32xr_cd32x (`cd32x/`, built by `build.sh`; `README.md` "The boot stack").

**Boot stack: no new command number, no new 68000 routine.** Everything below is inside existing
commands, behind `D32XR_CD` (Sub-CPU, `GAME_DEFS="-DD32XR_CD=7"`) and `CDL_D32XR_PAGE` (SH2 loader,
`LOADER_DEFS`), with the paged-cart code of section 7 (`CART_DEFS=-DCART_PAGED`). Without them the
stack builds as before.

| Command | What D32XR CD32X adds | Where |
|---|---|---|
| SH2 **38** open | file **20** = `D32XR.32X` (the game's 4 MB ROM, cart 0); files **300 + n** = the preload files (`preload_names.h`, made by `tools/make_preload.py`: SOUNDS.WAD, DOOM.WAD, DOOM_II.WAD, RESURRECTION.WAD). With COMM2 bit 15 the destination comes in COMM8L, as section 7's paged builds | `main.c` |
| SH2 **46** chunk → cart | with a destination given (38 bit 15), the cart offset wraps in the 4 MB (`& 0x3FFFFF`): a preload file that goes on into the next page starts there at offset 0, so "where chunk 0 would go" can be below 0 | `main.c` |
| SH2 **68** cart page, window 1 | used as section 7 (page 1-7) for the board probe and the preload copy (pages 2-6) | – |
| SH2 **60** start D32XR | window 1 is set to page **7** first (`D32XR_CD`): the game's second 2 MB. The game runs from pages 0 + 7 | `main.c` |
| MD **35** `start_d32xr` | the backdrop is set to **black** (was the D32X M1 debug checkpoint red: a red flash around the screen as the game starts) | `hw_md.s` |
| SH2 **51** chunk → frame buffer | the loading screen (`IMAGE.RAW`, `tools/make_splash.py`): the picture, the 256-colour palette and two text lines after it ("FIRST TIME: COPYING DOOM INTO THE RAM CART", "THIS DISC NEEDS THE 8 MB OR 16 MB RAM CART"). Only the gold part of the progress bar is drawn, only while copying | `cdloader_sh2.c` |

**Every boot (SH2 loader, `CDL_D32XR_PAGE`):**
1. **Board probe**: markers in save piece 15 of pages 1, 2 and 6 (CMD 68, then 58 + 61), read back by the SH2 through window 1: 4, 8 or 16 MB. A 4 MB cart is refused (the game's 4 MB would overwrite the save area).
2. **Checksums**, read by the SH2 straight from the cart (cache-through), of the ROM in pages 0 + 7 and of every preload file in pages 2-6, against the ones the build computed (`preload.h`). A + B + C held = copy everything again.
3. **Copy only what differs** (CMD 38 + 46, the page set with 68 between chunks), with the text and the gold bar.
4. **Record**: `D32XR PRELOAD OK` in save piece 15 of page 1 (CMD 58 + 61). The game reads the WADs from the cart only when it is there.
5. CMD 60. **D32XR then restarts the Sega CD with its own Sub-CPU program** (mega CD mode 2 addresses: BIOS `0x000000`, Program RAM window `0x020000`, Word RAM `0x200000`), so this stack's Sub-CPU commands end here. The game's own Sub-CPU protocol (`d32xr/src-md/cd/crt.s`: `D` disc info, `P` play track, `S` stop, `H` read sectors, `Q` / `R` streamed PCM ...) is D32XR's, through `0xA1200E`, not this ledger's COMM0 numbers.

**The cart (16 MB board):**

| Page (2 MB) | Holds |
|---|---|
| 0 | the game, first 2 MB (window 0) |
| 1 | the shared save area (section 5) and other titles' data: untouched except pieces 11 and 15 below |
| 2-6 | the preload: 63 × 32 KB used per page (0x000000-0x1F7FFF of the window), the files back to back across pages |
| 7 | the game, second 2 MB (window 1 while the game runs) |

Pages 2-6 overlap other titles' paged data (SuperTux's level packs, section 7a). They share the
cart one at a time: the checksums at every boot find what another title overwrote and copy it again.

**Save area pieces (section 5):**

| Piece | Cart | Owner |
|---|---|---|
| **11** | 0x3FB000 (page 1) | **DOOM CD32X Fusion (D32XR CD32X)**: the game's saves (its save RAM, `d32xr/src-md/crt0.s` `read_sram` / `write_sram`, through RV = 1) |
| 15 | 0x3FF000 | the board probe's markers (pages 1, 2, 6), as SuperTux's; and the record `D32XR PRELOAD OK` on page 1. **The record is only a hint**: when another title's test writes piece 15, the next boot rechecks the sums and writes it again; nothing is lost |

So piece 11 is no longer free: free pieces are now **12** only.

- **Status:** run in emulation (ares with the 16 MB board emulated): first boot copy, later boots
  checking only, the copy after a damaged WAD, the game, its CD audio and its streamed PCM music.
  **Not yet run on the console.**

## 16. The Mega EverDrive PRO launcher builds (EDCD): saves in bank 31, pages 1-6 (26-27 Oct 2026)

Recorded 31 Oct 2026 from the code of every port; the background (the PSRAM wipe, the `W` rule, the RV
garbage, the even-byte rule) is in CD32X-BootCamp docs/10 and docs/12 and in `MegaEverDrivePro_SDCARD.md`
here. A game started from its `EDCD_<GAME>.32X` launcher runs the same stack built with `MODE1=1`; its
Sub-CPU program is a **copy** of the stack with the game's own switches (`CART_ED`, `CART_EDSAVE`,
`RAY_SAVE`, `SDSAVE` ...). The disc's own program never has the bank: every "in" command below answers
without the `0x200` bit there, and the game reads the cart as before.

**The route** (console-proven, OpenLara CD32X-REV EDCD, 26 Oct 2026): CMD 58 per word → the 4 KB
`bram_buf` → Word RAM `0x0C0000` → MD 33 `chilly_rv_memcpy` with a cart address of
**`0x800000 + offset`** → the listener's `m1_cart` (MODE1-7, `M1SAVE=1`): bank 31 through window 1,
RV = 1, SR = 0x2700, `W` on inside the write only → read back the same way → compared. Both SH2s park in
SDRAM with interrupts masked for the whole command (EDSAVE-4: the EverDrive answers an SH2 cart access
during RV = 1 with garbage). Only the bank's first 256 KB is reachable (`m1_cart` refuses `0x840000`
and up): the OS writes that much to `MEGA/gamedata/<launcher>/bram.srm`.

| Game (launcher build) | Out: buffer → bank 31 | In: bank 31 → buffer | Bank offset of piece n | Reply | Code |
|---|---|---|---|---|---|
| OpenLara CD32X-REV EDCD (EDSAVE-1) | **61** (COMM2 = piece 0-15; the save area's own numbers) | **62** (COMM2 = piece; `0x8000` = only ask) | `n × 4 KB`; slots at `0x10000` | 61: `1 \| 0x200`; 62: `1 \| 0x200` | `OL_EDSAVE` (branch after `a9f13c4`; cited by `sdcard/README_AI.md`) |
| OpenJazz CD32X REV | 61 | 62 | `n × 4 KB` | as OpenLara | `cd32x/subcpu/main.c` `CART_EDSAVE` (`ed_page == 1` only) |
| Hurrican CD32X | 61; **56 / 59 → piece 12** (high scores) | 62 | `n × 4 KB`; 12 for 56/59 | as OpenLara; 56/59: 1 ok | `cd32x/subcpu/main.c` `CART_EDSAVE` |
| D32XR CD32X | 61 (the loader's piece-15 record only) | **none**: D32XR's own 68000 reads bank 31 itself (`src-md/crt0.s`, CTRL1 = 31, `CD32X_EDSAVE 0x080000`) after CMD 60 ends the stack; its 62 is the Genesis background | `n × 4 KB`; piece 11 the game's | 61 as OpenLara | `cd32x/subcpu/main.c` `CART_EDSAVE` (61 only) |
| SuperTux CD32X REV | 61 | **69** (62 is its `VRAM word`) | `n × 4 KB` | as OpenLara | `cd32x/subcpu/main.c` `CART_EDSAVE`, `STX_CART_PAGED` |
| Rayman CD32X (section 17) | **72** (COMM2 & 15) | **73** | **`0x30000 + n × 4 KB`** | `0x201` / 0 | `tools/stack_save.py`, `RAY_SAVE` |
| CD32X-BootCamp `sdcard/` (section 18) | **74** (COMM2 = piece 0-63) | **75** (`0x8000` = only ask) | `n × 4 KB` across the whole 256 KB | `0x201` / 0 | `sdcard/subcpu/merge_sdsave.py`, `SDSAVE` |

- The ports' 61 keeps its save-area meaning on the 4 MB cart and goes to bank 31 only in the launcher's
  program while window 1 shows page 1 (`ed_page == 1`), so a game's saves are page-1 saves on the PCB cart
  and bank-31 saves on the EverDrive without a change to the engine.
- **Status.** OpenLara: console (26 Oct, save / load / relaunch / load). OpenJazz REV, Hurrican, SuperTux,
  D32XR, Rayman, `sdcard/`: ares only (the EverDrive board), code-reviewed for the parked waits. ares holds
  the SH2s during RV = 1 and keeps the PSRAM across runs, so an ares pass covers neither the garbage rule
  nor the wipe (BootCamp docs/11).

**Command 68 on the EverDrive (`CART_ED`, 26 Oct 2026, OpenJazz `77e167f`, Hurrican `8382713`, SuperTux
`7dd1613`, D32XR `fd86f3c`; Rayman through `M1PAGE=1` instead).** Pages 1-6 only (page 7 = banks 28-31,
the OS's and the battery's; the Sub-CPU refuses 7). The page byte the PCB cart wants is one even byte to
`0xA130F2`, and **the EverDrive ignores an even-byte write** (`edpage`, console), so an ED build asks the
listener for **four word writes to `0xA130F8-FE` (CTRL4-7 = banks 4p..4p+3) through MD 27
`safe_cart_xfer_w`** - MD 26 would write them as bytes. Rayman keeps the one-byte request and lets the
launcher's listener translate it (`m1_page`, MODE1-9, built with `M1PAGE=1`). Measured on the console:
`m1_page` and the mapper (`edpage`, 26 Oct); the games' own word-write path is not yet on the console, and
the CTRL4-7 writes made with `W` on (the `m1_cpy_w` route the games' request takes) are unmeasured
(Rayman `EVERDRIVE_FINDINGS.md` §6).

## 17. Rayman CD32X (R1, 26-27 Oct 2026): commands 72, 73, 74 and listener routine 38

Rayman CD32X (MicroNut99/Rayman_32X_RAM, `ab5e08b`) runs on the BootCamp's stack with two merges into a
copy: `tools/stack_save.py` (`RAY_SAVE`) and `tools/stack_genvdp.py` (`RAY_GENVDP`: SuperTux's 62-67 as
they are, plus one of its own). The BootCamp folder is never touched; with the blocks taken out every file
is the BootCamp's again.

| CMD | Name | COMM2 | COMM8 / COMM8L | Reply | 68000 routine |
|---|---|---|---|---|---|
| 72 | save piece out (bank 31) | piece 0-15 → bank offset `0x30000 + piece × 4 KB` | - | COMM12 `0x201` done / 0 | 33 (the listener's bank-31 route) |
| 73 | save piece in (bank 31) | piece 0-15 | - | `0x201` / 0 | 33 |
| **74** | **plane A scroll at the vertical blank** | the H-scroll table's VRAM address (even) | COMM8L = horizontal << 16 \| vertical | - (COMM0 cleared) | **38 `set_scroll_vb`** (new): the 68000 waits for the blank, then writes HSCROLL and VSRAM in one go; the vblank handler touches no VDP register, so nothing comes between |

- Why 74: the Genesis scroll written whenever the 68000 reached the command landed mid-picture; the level
  layer showed two scroll positions, torn at a line walking down the screen (71 of 150 frames in ares).
  With 74 posted together with the 32X frame flip: 0 of 180.
- `MD_CMD_END` is 39 in a `RAY_GENVDP` program (36 `set_vsram`, 37 `copy_vram_col`, 38 `set_scroll_vb`).
- Status: ares only (R1 is not for the console yet; the R0 disc's console failure was the EDSAVE-4 class,
  not a stack fault - `EVERDRIVE_REPLY.md`).
- **74 clashes with the BootCamp `sdcard/` kit's `SDSAVE_CMD_OUT`** (section 18), taken the next day. The
  kit's numbers are build-time variables; Rayman's 74 is in a built launcher. See section 18 for the
  resolution asked for.

## 18. CD32X-BootCamp `sdcard/` (27 Oct 2026): commands 74 / 75, and the number clash

`sdcard/subcpu/merge_sdsave.py` puts OpenLara's EDSAVE-1 pair into a copy of the stack behind `SDSAVE ?= 0`
with the piece numbered across the whole 256 KB of bank 31:

| CMD | Name | COMM2 | Reply | How |
|---|---|---|---|---|
| 74 (`SDSAVE_CMD_OUT`) | buffer → bank 31 | piece 0-63 | COMM12 `0x201` done, 0 failed or no bank | MD 33 to `0x800000 + piece × 4 KB`, read back, compared |
| 75 (`SDSAVE_CMD_IN`) | bank 31 → buffer | piece 0-63; `0x8000` = only ask | `0x201` / 0 | MD 33 |

The SH2 side is `sdcard/sh2/sdsave.c` (`sds_spin` in `.data.sdram`, SR \|= 0xF0, the slave parked with
COMM4 `0x7A60` / `0x7A61`, bounded waits that refuse the copy rather than run it unparked; records with
magic, length, FNV-1a, ~sum). Status: ares only (27 Oct 2026); the console-proven part is OpenLara's
underneath.

**The clash.** The kit's README says "74/75 are free in the stack (it uses 0-68)" - true of the stack,
but Rayman had taken 74 for its scroll command the day before (section 17), and the ledger's own free
line still said "69 and up". Rule: one number, one meaning. **Resolution asked for: the kit moves its
defaults to 76 / 77** (`SDSAVE_OUT=` / `SDSAVE_IN=` in the make, `-DSDSAVE_CMD_OUT=` / `-DSDSAVE_CMD_IN=`
on the SH2 side, together) before any game ships with it; until then a game that merges both Rayman's
scroll and the kit must pass the override. This ledger keeps 76 / 77 free (78 went to `VSPLIT`, section 21) so that the next taker sees the
move; whoever makes it updates sections 3, 18 and this line.

## 19. OpenJazz CD32X REV (R7-R11, to `d486f58`): commands 67, 69, 70, 71

These were only in CD32X-BootCamp docs/03's clash table until now. All in `cd32x/subcpu/main.c`; 67 and
71 use the REV listener (`cd32x/subcpu/hw_md.s`, routines 37-44, `MD_CMD_END` 46, section 2).

| CMD | Name | COMM2 | COMM8 / other | Reply | 68000 routines |
|---|---|---|---|---|---|
| 67 | Genesis plane (`docs/GENESIS_PLANE.md`) | 1 load the level's plane data, 2 show it, 3 off, 4 a map cell changed | COMM8L = cart offset / camera x << 16 \| y / - / cell << 16 \| tile | COMM12 = the 68000's result | 37 `genbg_load`, 38 `genbg_start`, 39 `genbg_off`, 40 `genbg_cell` |
| 69 | unpack a compressed pack page → page (`CART_PAGED` only) | source page \| dest page << 8 (1-7 each) | COMM8L = source cart offset (≥ `0x200000`, even), **COMM12 in** = dest offset in 32 KB units (`& 0x7F`, ≥ `0x200000`) | COMM8L = raw length (0 = refused), COMM12 = 0 ok / 1 | 41 `rv_copy_fast` and the page writes |
| 70 | the disc's stamp | - | - | COMM8L = stamp, 0 unknown (loads the root directory cache first) | - |
| 71 | bonus level floor (R9: the Sega CD graphics chip, Word RAM 2M) | op 1 load (COMM8L = cart offset), 2 frame (COMM8L = x << 16 \| y, COMM4 = cos, COMM12 = sin, op >> 8 = picture), 3 suspend | as listed | COMM12 = result | 42 `bmd_setup`, 43 `bmd_queue`, 44 `bmd_off` |

- 71 op 2 reads **COMM4 and COMM12 as inputs**, which breaks the "one writer per register" habit of
  docs/06 for that command: the SH2 writes both before COMM0 and must not expect COMM12 to hold its status
  through it. 69 reads COMM12 as an input the same way.
- Status: R1 ran on the console (20-22 fps); R7-R11 are emulation only (BootCamp docs/07, 29 Oct 2026).

## 20. SuperTux's 62-67 in the BootCamp stack (`GENPLANE`), run by a game in ares (31 Oct 2026)

**Update, 31 Oct 2026: merged into the BootCamp stack** at the owner's request (CD32X-BootCamp `252265a`).
- **Where:** `stack/subcpu/main.c`, `hw_md.s`, `Makefile`, behind `GENPLANE ?= 0`. Same numbers and meanings as
  below.
- **Switch off:** the Sub-CPU program is byte-identical to the stack before (APP.BIN compared), and
  `BOOT_FILES.md5` is unchanged.
- **The check:** `tools/check_boot.py` takes the `DX-GENPLANE` blocks out before the hash and checks them
  exactly against `stack/GENPLANE_BLOCKS.md5`; a changed byte inside a block fails.
- **Re-run in ares from the BootCamp tree:** the same results as below.
  - `hello` with `GENPLANE=1`: frames 1800 / 2100 identical to the baseline.
  - The Dracula X test ROM: `4B10`, 0 of 57,344 pixels off, the plane scrolls.
- **Status of 62-67 on the BootCamp stack:** emulation. Not yet on the console.

The first test, before the merge (a copy of the stack in DraculaX_32X):

MicroNut99/DraculaX_32X (`c8e8099`) takes SuperTux CD32X REV's Genesis picture commands onto the BootCamp
stack, unchanged: **no new number, no new meaning.** The commands and listener routines are exactly
sections 1 and 2:
- SH2 62-67;
- MD 36 `set_vsram`, MD 37 `copy_vram_col`, `MD_CMD_END` 38 (the same numbers as SuperTux and Rayman).

**Where:** `cd32x/genplane.patch` on a copy of the BootCamp (`cd32x/apply.sh`); the BootCamp folder itself is
never touched.

**Switch:** `GENPLANE ?= 0`.
- `main.c` gets `-DDX_GENPLANE` through **HWFLAGS**.
- `hw_md.s` gets `--defsym DX_GENPLANE=1`.
- `build.sh` stops unless `main.c`'s compile line has `-DDX_GENPLANE=1` and the listener has `copy_vram_col`
  (the lesson of section 7a).
- `tools/check_boot.py` strips exactly the `DX-GENPLANE begin / else / end` blocks. The rest must match the
  BootCamp's `BOOT_FILES.md5`.

| Check | Result (ares, the owner's ares_android `ares-headless`, 4 MB cart, US BIOS) |
|---|---|
| `GENPLANE=0` | the Sub-CPU program (`APP.BIN`) byte-identical to the BootCamp's build |
| BootCamp `hello/` with `GENPLANE=1` | boots and runs as without it: frames 1800 and 2100 identical pixel for pixel, `4B10` |
| the test ROM `dxtest/` (`GENPLANE=1`) | every command answered; every 63 / 64 / 67 replied 1; `4B10`. One PC Engine frame drawn the REV way (background on plane A, sprites on the 32X in front at PRI 1, through-bit colours): **0 of 57,344 pixels** differ from the frame computed on the PC |

What `dxtest` sent:

| Command | Times | Use |
|---|---|---|
| 65 | 12 | display off / on; planes A and B at `0xE000` (64 × 64); sprite table `0xD000`; H-scroll `0xD400`; full-screen scroll; window off; backdrop; H40 |
| 64 | 1 | all 64 colours |
| 63 | 2 | 77 tiles: 1023 words, then the rest |
| 67 | 16 | the map: 4 columns × 64 words each |
| 62 | 6 | the sprite table's first entry; H-scroll of A and B |
| 66 | 2 | V-scroll of A and B |
| 62 / 66 | 2 a frame | while the D-pad is held: the plane scrolls under the fixed 32X layer |

**Status:**
- **62-67 on the BootCamp stack:** emulation (ares, 31 Oct 2026).
- **67:** now exercised by a game in emulation, with a 64-word column, the most its 8-bit count allows on a
  64-row plane.
- **The console:** not yet.

**Toolchain:** the container's Sub-CPU build used a stand-in m68k-elf (Ubuntu binutils / GCC 13, GCC 12.1's
`lb1sf68.S`, a small libc; DraculaX_32X `tools/toolchain/`). The owner's sh-elf GCC 12.1 was used. A disc for
the console should be built with the owner's own toolchain.

Details: DraculaX_32X `docs/03_M1_TEST1.md`.

## 21. VSPLIT: command 78 and listener routine 38, the V scroll changed part-way down the screen (31 Oct 2026)

**Owner:** the BootCamp stack (CD32X-BootCamp `c643910`), at the owner's decision D4 for Dracula X CD32X.

**Why:** Dracula X's stage 8 changes its background's vertical offset up to 9 times down one picture (a strip
whose rows repeat, with a ripple that moves every few frames: DraculaX_32X docs/04). The Genesis has per-line
H scroll, but only one V scroll per plane, and two planes.

**Switch:** `VSPLIT ?= 0`, needs `GENPLANE=1`.
- `main.c` gets `-DDX_VSPLIT` through **HWFLAGS**; `hw_md.s` gets `--defsym DX_VSPLIT=1`.
- `build.sh` stops unless `VSPLIT=1` comes with `GENPLANE=1` and reached `main.c` and the listener (`hint_vs`).
- `tools/check_boot.py` strips the `DX-VSPLIT` blocks (outermost first, then `DX-GENPLANE`) and fingerprints
  them in `stack/GENPLANE_BLOCKS.md5`.

| CMD | Name | COMM2 | COMM8L | Reply | 68000 routine |
|---|---|---|---|---|---|
| 78 | V scroll line split | index 0-15; or `0x8000 \| count` = show the filled list, its first count entries (0 = off; D5: from the end of the picture on screen) | with an index: the entry, line << 20 \| V scroll A << 10 \| V scroll B | - (COMM0 cleared) | **38 `set_vs_line`** (new), and the H interrupt handler `hint_vs` |

- **The list:**
  - Entry 0 is the top of the picture: its line is ignored, and it is written to VSRAM at once.
  - The others must be in line order.
  - "Use" (`0x8000 | count`): the H interrupt is turned on every line (VDP register 10 = 0, register 0
    = `0x14`) and `$A12006` is set to `hint_vs`'s low word.
  - Count 0: register 0 = `0x04` (off).
- **`hint_vs`:**
  - Runs at the end of every line: it reads the V counter (`$C00008` high byte), and when the next line is an
    entry's line, it writes that entry's two VSRAM words.
  - After line 223 it writes entry 0 again for the next picture.
  - Its VSRAM write sets the VDP's address, so it must never run inside a routine that is streaming to the
    VDP. The dispatcher therefore runs every listener routine with **SR = `0x2400`** (level 4 masked) and
    goes back to `0x2000` after.
  - A split that falls due during a routine is done when the routine ends, late on that picture.
  - The vertical blank (level 6) is not masked and touches no VDP register.
- **The H-INT vector:** Mega-CD manual, "MAIN-CPU registers, `$A12006` H-INT vector (level 4)". On H-INT the
  main CPU jumps to the high word of the vector at `$70` with `$A12006` as the low word. The listener runs in
  work RAM at `$FF1000`, so the low word alone points the jump at `hint_vs`.
- **Cost:** 224 short interrupts a picture while the split is on, most of them only a compare.

**Status: emulation** (ares, 31 Oct 2026, 4 MB cart, CD32X-BootCamp `c643910` built with the stand-in m68k-elf of
DraculaX_32X `tools/toolchain/`):

| Test | Result |
|---|---|
| `VSPLIT=0` | byte-identical (`GENPLANE=0` = the shipped stack; `GENPLANE=1` = `252265a`) |
| `hello`, `GENPLANE=1 VSPLIT=1` | boots as the baseline, frames identical |
| Dracula X's stage 8 test frame | 9 entries (offset 0, then −1 … −8 every three lines from line 201) with the per-line H scroll table; **0 of 57,344 pixels** off the frame computed on the PC (itself 0 off Mednafen) |
| Dracula X's stage 3 frame on the same build | 0 off |

**Open on the console:** when the H interrupt arrives relative to the line, and what the V counter reads
then. If the console's split lands a line early or late, the off-by-one goes in `hint_vs`'s `addq.w #1,d0`.

### 21a. D5: the list double-buffered, the mask narrowed (CD32X-BootCamp `1ae4847`, 31 Oct 2026)

**Why:** a game sends the next picture's splits while the current picture is still on screen (the 32X draws the
next frame meanwhile).
- **The fault:** with one list, the entries sent then changed the lower bands of the picture on screen, and the
  commit wrote entry 0 at once. So the bands ran a frame ahead of the 32X frame, part-way down the screen: the
  hero shaking against the background (DraculaX_32X docs/07). §21's test drew one still picture and could not
  show it.
- **The owner:** "make the change and test it".

**The change (`hw_md.s`, inside the `DX-VSPLIT` blocks; command 78 and its COMM layout unchanged):**
- **Two lists of 16:** `vs_vars` = count, position, the list shown (byte offset 0 / 64), pending count (−1 none),
  then lists 0 and 1.
- **An entry (index 0-15)** goes into the list **not** shown.
- **A commit (`0x8000 | count`):**
  - while the splits run: it only sets the pending count;
  - the first commit (splits off until then): shows the filled list at once, as before;
  - count 0: off at once.
- **`hint_vs`, after line 223:** if a count is pending, it switches lists. That is the vblank at which the 32X
  shows a frame whose swap was asked for after the commit. Then it writes entry 0 as before.
- **The dispatcher masks level 4 only for routines that touch the VDP.** `get_pad` (3), `delay` (11) and the
  32X register reads and writes every command makes (19-22) run with SR = `0x2000`, so they never hold a split
  back.
  - Before: every routine was masked. A command in flight while the bottom bands were drawn made one split
    line land part-way through the line.
- **The game's side:** commit, then ask for the 32X swap, in the same frame. A commit that lands after the
  picture's line 223 waits for the next picture.

**Status: emulation** (ares, owner's ares_android headless, 4 MB cart):

| Test | Result |
|---|---|
| Dracula X stage 8 frame, the list changed **every frame while on screen** (DraculaX_32X `dxtest` `DXT_VSWAP`): layout 0 = the capture's 9 entries, layout 1 = the same lines + 8 lines of V scroll; a 32X marker says which layout each frame should show | **60 of 60 pictures exact** (0 pixels off the layout's own picture computed on the PC) and in step with the 32X frame. §21's stack: 0 of 12, 6,600-9,100 pixels off each. The double buffer alone: 50 of 60, with one late split row in the other 10 |
| `hello`, `GENPLANE=1 VSPLIT=1` | pictures identical to the baseline |
| Dracula X stage 8 and stage 3 test frames | 0 of 57,344 pixels off |
| default build (switches off) | the disc's files and boot area identical to the old stack's (only the ISO dates differ); `check_boot.py` unchanged |

**Open on the console:** the same as §21. Also whether the console's interrupt latency in the unmasked
routines matches ares.

### 21b. VSPLIT-MODE1: the H interrupt in a cartridge (EDCD) boot (CD32X-BootCamp `1eaff00`, PR #1, 7 Nov 2026)

**Found by:** Sapphire CD32X (its docs/05 §3), testing its HUD split on the EverDrive PRO boot in ares.
**The owner:** "If you can fix the bug then document the fix and move on."

**The fault:**
- `set_vs_line` hooks the H interrupt by writing `hint_vs`'s low word to `$A12006`, the Mega-CD gate array's H-INT
  redirect (§21). That only applies when the 68000 runs from the Sega CD's vectors, as in a CD boot.
- In a cartridge boot (`MODE1`, `EDCD.32X`) the 68000's vectors are the cart's. Level 4 goes through the cart's
  table to the RAM jump table at `$FFFD0C`, which the launcher fills with `jmp $FFFDAE`, an `rte`
  (`edboot/edcd_md.s` step 2). `$A12006` redirects nothing there.
- So command 78's split never ran on EDCD: the plane kept its top V scroll on every line.

**The fix (`stack/subcpu/hw_md.s`, `set_vs_line`, inside the `DX-VSPLIT` block, under `.if OL_MODE1`):**
- When the splits are turned on, `hint_vs`'s address is also written to the jump's target at `$FFFD0E`.
- The dispatcher runs `set_vs_line` with level 4 masked, so no H interrupt meets the target half written.
- A CD boot's listener (`OL_MODE1` = 0) assembles exactly as before. No new command, number or routine.
- `stack/GENPLANE_BLOCKS.md5` updated (`check_boot.py --update-genplane`).

**Status: emulation** (ares, `ARES_32X_TIMING=console`):

| Test | Result |
|---|---|
| default build (`hello`) | `APP.BIN` and `CDLOADER.BIN` byte-identical to `63d1798`'s |
| `GENPLANE=1 VSPLIT=1`, CD boot | `APP.BIN` byte-identical to `63d1798`'s |
| Sapphire's stage 3 test frame (one split at line 203), `EDCD=1 GENPLANE=1 VSPLIT=1`, EverDrive board | before: 3,233 of 71,680 pixels off (the HUD's lines); after: **0**, at two frames, `4B10` |

**Open on the console:** as §21 (where the H interrupt lands on the line), now for the EDCD boot too.
This affects every game using VSPLIT on EDCD: Dracula X's stage 8 bands included. Documented in CD32X-BootCamp
docs/03 (command 78) and docs/12.

## 22. PROPOSED (2 Nov 2026): commands from the Sega CD BIOS - all untested

**Status of everything in this section: proposed, untested as of 2 Nov 2026.**
- Nothing here is in any Sub-CPU program: not built, not run in an emulator, not run on a console.
- The numbers 81-98 are proposals, not taken, **except 84 and 90**: built 2 Nov 2026 in the BootCamp's
  `traytest/` (the tray test disc, `traytest/merge_tray.py`, `TRAYTEST ?= 0`), run in ares only, not on a console;
  section 3 lists them. The tray test T3 needs no cart: its SH2 program runs from SDRAM in CDLOADER's place
  (built with the stack's `cdl_crt0.s` and `cdloader.ld`), and the copy's start-up cart check is off
  (`TRAY_NOCART ?= 0`, the stack's own `KOBO_NOCART 1`). Not a command: no number. T5 (3 Nov 2026) is also a 32X
  cartridge ROM for the Mega EverDrive PRO (the EDCD launcher around a `MODE1=1` build): no disc needed; its start-up
  disc read (`InitCD`) is off (`TRAY_NODISC ?= 0`), since the tray may hold a music CD or nothing. Check section 3 before building any of the others, and renumber here if a number has gone.
- When one is built: behind a switch (`HWFLAGS`), its own section.

**Where it comes from:**
- **DarkMorford/scd-bios-disassembly** (github.com/DarkMorford/scd-bios-disassembly, last commit `41acd10`,
  29 Sep 2016): a commented disassembly of the US Model 2 BIOS **2.00W**. Its Sub-CPU part (`sub-68k/prog0_bios`)
  names the CD read, drive, subcode and backup RAM code. The music handlers are still mostly unnamed labels: what
  22a says of them was read from the code.
- **drojaazu/megadev** (`lib/sub/bios.def.h`, `lib/sub/bram.def.h`): the BIOS call numbers and their arguments.
- **The US Model 1 BIOS 1.10's Sub-CPU program**, read out of ares's program RAM after power-on (2 Nov 2026):
  used only for the table sizes in 22d and for WONDERREQ / WONDERCHK.
- The BIOS files and the disassembly's extracted pieces are Sega's: never copy them into a repository.

**The BIOS calls the stack makes today** (`stack/subcpu/main.c`, `scd.c`, `cdfs.s`):
- CD (`_CDBIOS`, `jsr 0x5F22`): MSCSTOP 0x02, DRVINIT 0x10, MSCPLAY1 0x12, MSCPLAYR 0x13, ROMSEEK 0x18,
  ROMREADN 0x20, CDBSTAT 0x81, CDBTOCREAD 0x83, FDRSET 0x85, CDCSTOP 0x89, CDCSTAT 0x8A, CDCREAD 0x8B,
  CDCTRN 0x8C, CDCACK 0x8D, CDCSETMODE 0x96.
- Backup RAM (`_BURAM`, `jsr 0x5F16`): BRMINIT 0, BRMREAD 3, BRMWRITE 4 (commands 56 / 59, one fixed file,
  `GAME_BRAM_NAME`).

Every proposal below uses calls the stack does not make.

### 22a. What the BIOS code shows (US 2.00W; the facts the proposals rely on)

- **Two kinds of CD call.** Calls 0x80 and up run at once and return their result. Calls under 0x80 are queued:
  the BIOS takes the call and returns, and its drive state machine carries it out over the next interrupts.
  CDBSTAT (0x81) shows when the drive has done it.
- **MSCPLAY (0x11)** plays on to the end of the disc, then pauses (state 0x25, then 0x03 at the lead-out).
- **MSCPLAY1 (0x12)** pauses when the next track starts (state 0x26).
- **MSCPLAYR (0x13)** repeats a track by starting it again with a **new seek each time round** (state 0x27 calls
  0x13 again): the loop is not seamless.
- **MSCSEEK (0x15) and MSCSEEKT (0x16)** seek and hold paused (state 0x28). MSCPAUSEOFF (0x04) then plays on to
  the end of the disc. **MSCSEEK1 (0x19)** is the same seek, but MSCPAUSEOFF then plays **only that track**: a
  flag picks state 0x26 instead of 0x25.
- **Spin-down.** A paused drive counts frames: after a music seek or MSCPAUSEON (state 0x28), and a data read
  paused by ROMPAUSEON or held after ROMSEEK (state 0x2B). When the count passes the spin-down delay, it goes to
  standby (0x0B).
  - The delay is CDBPAUSE (0x84), in 1/75 s. The BIOS sets 4500 (60 s) at start-up.
  - A resume from standby starts the kept play again with a new seek.
- **ROMPAUSEON (0x08) / ROMPAUSEOFF (0x09):** the data read pauses with its place kept (state 0x2B, status
  0x0505), then goes on from there (state 0x2A).
- **FDRSET (0x85):** the CD audio volume at once, 0-0x400 (bit 15 set: the master volume instead).
  **FDRCHG (0x86):** D1 = target << 16 | slope. The volume moves towards the target one slope step per BIOS
  interrupt.
- **DRVOPEN (0x0A):**
  1. Stops the drive if it is running.
  2. Sends the drive its "open door" command (CDD 0x0D).
  3. Reports "tray open" (status word 0x4000) while the drive says the door is open.
  4. Then runs DRVINIT itself: the new disc's table of contents is read again.
- **WONDERREQ (0x97) and WONDERCHK (0x98) are empty** in both US BIOSes read (2.00W in the disassembly, 1.10 in
  ares): WONDERREQ only tests D1 and returns; WONDERCHK always returns 0.

### 22b. Proposed commands

| CMD | Name | COMM2 | COMM8 / COMM8L | Reply | BIOS call | Why |
|---|---|---|---|---|---|---|
| 81 | CD music pause / resume (**built: `CDPAUSE=1`, section 31, 4 Nov 2026 - ares only**) | 1 pause, 0 resume | - | - | MSCPAUSEON 0x03 / MSCPAUSEOFF 0x04 | A pause menu keeps the song's place; today 54 stops it and 45 starts it from the top |
| 82 | CD music volume / fade | target 0-1024 (1024 = full) | COMM8 = slope; 0 = at once | - | FDRSET 0x85 / FDRCHG 0x86 | Fades at level ends, a music volume option, music lower under speech. **45 calls FDRSET 0x400 every time it starts a track**: send 82 after 45, or change 45 behind the same switch |
| 83 | Cue a track | track; bit 15 = when started, play this track only (MSCSEEK1) | - | - | MSCSEEK 0x15 / MSCSEEK1 0x19 | The drive waits at the start of the track; 81 (resume) starts it with no seek, so a short jingle starts on time |
| 84 | CD status and music position (**built: `traytest/`, ares only**) | 0 status, 1 position, 2 disc time | - | COMM2 0: COMM12 = the BIOS status word (0x0000 stopped, 0x0100 playing, 0x0300 scanning, 0x0500 paused, 0x1000 no disc, 0x2000 reading the disc, 0x4000 tray open; bit 15 = busy), COMM8L = the drive's status word << 16 \| first track << 8 \| last track. COMM2 1: COMM12 = current track (low byte), COMM8L = time in the track (BCD mm ss ff 00). COMM2 2: COMM12 = the drive lights, COMM8L = time on the disc. Added for the tray test T4 (2 Nov 2026), so one photo of a console that does not open is the whole record: COMM2 0x10-0x17: COMM8L = table bytes 4n-4n+3 (the whole table); and, with no CDBSTAT call, COMM2 3: COMM12 = $FF8000 (gate array: LEDs, chip version), COMM8L = the 68000's word at $A10000 (the Mega Drive's version, low byte); COMM2 4: COMM8L = the Sub-CPU BIOS's `_CDBIOS` jump target (0x5F24), COMM12 = its `_BURAM` jump target's low word (0x5F1A), which differ by BIOS version; COMM2 0x20-0x5F: COMM8L = the 68000's long at 0x100 + 4n (the BIOS ROM's header: name, date, version, region; ares 1.10: `BR 000003-1.10`); in a cartridge boot (`MODE1=1`, the tray test T5's `.32X` for the EverDrive PRO, 3 Nov 2026) at 0x400100 + 4n, where the Sega CD's ROM is then | CDBSTAT 0x81 (its table at 0x5E80 in the US BIOS 1.10 and 2.00W) | Carry the music on after a disc read (with 85), sync to the music, a CD player's time; after 90, see the tray open, close and the new disc read |
| 85 | Play from a time / seek to a time | 0 play, 1 seek and hold | COMM8L = disc time (BCD mm:ss:ff:00) | - | MSCPLAYT 0x14 / MSCSEEKT 0x16 | With 84, a song goes on where it stopped. MSCPLAYT plays on into the next tracks (the 0x25 play): the game stops it at the track's end |
| 86 | Disc track list | track 1-99; 0 = first and last track numbers | - | COMM8L = the track's table entry (start time and type: data or audio); for 0: first << 8 \| last | CDBTOCREAD 0x83 | Check the disc has the music tracks before using them; a CD player screen |
| 87 | Scan | 1 forward, 2 back, 0 off | - | - | MSCSCANFF 0x05 / MSCSCANFR 0x06 / MSCSCANOFF 0x07 | A CD player screen |
| 88 | Drive lights | 0-7 (ready, disc in, access, standby, error, modes 5-7: megadev's table); `0x8000` = back to the BIOS | - | - | LEDSET 0x95 | A sign on the console itself when the screen shows nothing (a stuck load, a console test) |
| 89 | Spin-down delay | ticks of 1/75 s (megadev: normal range 0x1194-0xFFFE; 0xFFFF never stops and can damage the drive) | - | - | CDBPAUSE 0x84 | Keep the disc spinning through a pause menu, so 81 resumes without a spin-up and a seek |
| 90 | The tray (**built: `traytest/`, ares only**) | 1 open, 0 close and read the disc | - | COMM12 = 1 asked, 0 refused | DRVOPEN 0x0A / DRVINIT 0x10 (parameters 01 FF: closes an open tray) | "Insert disc 2", a disc swap. Not every model: 22c |
| 91 | Backup RAM status | - | - | COMM8L = free blocks << 16 \| number of files; COMM12 = 0 no backup RAM, 1 unformatted, 2 other format, 3 ready | BRMINIT 0, BRMSTAT 1 | Say why a save failed (unformatted, full) instead of failing silently. SuperTux reported that backup RAM saving never worked on its console: this would tell why |
| 92 | Backup RAM directory | number of files to skip | - | the 4 KB buffer (read with 57): up to 255 entries of 16 bytes (name 11 + 0, start, blocks); COMM12 = entries | BRMDIR 7 | List every game's saves, as the Sega CD's own screen does |
| 93 | Backup RAM: read a file | part 0-1 | - | buffer bytes 0-4079 = that part of the file; COMM8L = blocks << 16 \| mode; COMM12 = 1 ok, 0 not found, 2 part past the end | BRMSERCH 2, BRMREAD 3 | Any file by name. The Sub-CPU reads the whole file (up to 8 KB) into its own work area and hands it out in 4080-byte parts |
| 94 | Backup RAM: write a file | part 0-1; bit 15 = last part: write the file now | - | COMM12 = 3 part kept, 1 written and verified, 0 failed (full, bad name) | BRMWRITE 4 (D1 = 0, Tech Bulletin #1), BRMVERIFY 8 | Any name, any size up to the free space (56 / 59 write one fixed name of 64 blocks) |
| 95 | Backup RAM: delete a file | - | - | COMM12 = 1 deleted, 0 failed | BRMDEL 5 | An "erase save" option; tidy up a full backup RAM from the game |
| 96 | Backup RAM: format | must be `0x464D` (`FM`); anything else is refused | - | COMM12 = 1 formatted, 0 failed or refused | BRMINIT 0, BRMFORMAT 6 | Make an unformatted backup RAM usable. **Erases every game's saves**: the game asks the player twice |
| 97 | WonderMega request | D1 for WONDERREQ | - | - | WONDERREQ 0x97 | Find out what a WonderMega's BIOS does with it (22b note) |
| 98 | WonderMega check | - | - | COMM8 = WONDERCHK's D0 | WONDERCHK 0x98 | 0 on every US BIOS read; non-zero would mean a BIOS that implements it |

**For 92-95:** the file name goes in the last 16 bytes of the 4 KB buffer (bytes 4080-4095, CMD 58): the name
(11 characters) and a 0, then for 94 the mode (0 normal, 0xFF protected) and the size in blocks (a word; 64 bytes a
block in normal mode, 32 protected). The data parts are bytes 0-4079. 92 takes a name pattern there (`SONIC*`,
`*` = all, megadev's BRMDIR note).

**Backup RAM, which one.** These calls reach the Sega CD's own 8 KB backup RAM (`0xFE0000`). The Sega CD's
backup RAM cartridge can't be used on a CD32X: the cartridge slot holds the 32X and the RAM cart. Today only the
Sega CD's own screen and Sonic CD manage this RAM; 91-96 would let any game on the stack do it.

**WonderMega (97 / 98).** On the US BIOSes both calls are empty, and megadev's notes say "no documentation;
research needed; presumably exclusive to the WonderMega". Nothing is known of what a WonderMega's (Victor / JVC)
BIOS does with them. 97 / 98 are the way to find out: read a WonderMega BIOS's two handlers first (in ares), then
on a WonderMega. Whether a 32X runs on a WonderMega with the CD32X stack is untested too.

**Inside an existing command (no number): pause the data read instead of losing it.** (**Built 3 Nov 2026:**
`ROMPAUSE=1`, section 26, ares only.) In STREAM (section 10), the
64 KB read-ahead ring (`st_buf`) fills while the 68000 copies to the cart. The sectors after that are lost
(`cd_pump`: "lost: start again when asked"), and the next chunk seeks again. ROMPAUSEON (0x08) when the ring is
full and ROMPAUSEOFF (0x09) when there is room keep the drive in place (22a). Behind STREAM or a switch of its own.

### 22c. Open the tray (DRVOPEN 0x0A): by model

| Model | Tray | DRVOPEN |
|---|---|---|
| Model 1 (front tray, motorised) | **No eject button** (the owner, 2 Nov 2026). The tray opens only by software: the Sega CD's own screen, or a game calling DRVOPEN | The way a game opens the tray, e.g. to ask for disc 2 |
| Model 2 (top lid) | The lid has its own button | The 2.00W (Model 2) BIOS still has DRVOPEN and sends the drive the same "open door" command; whether a Model 2 drive does anything with it is untested |
| CDX / Multi-Mega, WonderMega, X'Eye, LaserActive | - | Untested |

- After DRVOPEN the status word reads 0x4000 (tray open) until the tray is closed. Then the BIOS reads the new
  disc's table of contents itself (DRVINIT).
- Anything read from the old disc is then stale: a game opens its files again (38). A stack with a root directory
  cache (section 6, Sonic R4K) must clear it.

### 22d. The BIOS's 11 extra calls 0x22-0x2C (US 2.00W): what each is

They are listed in the same table as the queued calls 0x02-0x21, but **they are not functions**: they are the
states of the BIOS's own drive state machine. A queued call puts its number in the state slot; the BIOS moves to
these states itself. A game calling one would drop the state machine into a state without the step that normally
leads to it (no track, no time). **Not proposed as commands.** What each one does, from the 2.00W code:

| Code | What it does | The BIOS goes there from |
|---|---|---|
| 0x22 | After reading the table of contents: waits for the drive to report "paused", then holds paused at the first track (status 0x0500, drive light "disc in"; goes on as 0x28) | DRVINIT (0x10) without the play flag; power-on (0x2C) |
| 0x23 | After reading the table of contents: waits for the drive to report "playing", then plays on (status 0x0100, light "access"; goes on as 0x25) | DRVINIT with bit 7 of its first byte set (play the first track) |
| 0x24 | Stopped: status 0x0000. Watches the drive; if the tray opens, the disc is missing or the tray moves, goes to 0x01 | MSCSTOP (0x02); standby (0x0B) |
| 0x25 | Playing on: status 0x0100. Follows the current track number; at the end of the disc (lead-out) pauses (0x03) | MSCPLAY (0x11), MSCPLAYT (0x14), MSCPAUSEOFF after MSCSEEK / MSCSEEKT |
| 0x26 | Playing one track: when the next track starts, or at the lead-out, pauses (0x03) | MSCPLAY1 (0x12); MSCPAUSEOFF after MSCSEEK1 (0x19) |
| 0x27 | Repeating a track: when the next track starts, or at the lead-out, calls MSCPLAYR (0x13) again with the same track: a new seek every time round | MSCPLAYR (0x13) |
| 0x28 | Paused after a music seek: status 0x0500. Stops the CD data controller and counts frames; after the spin-down delay, standby (0x0B) | MSCSEEK (0x15), MSCSEEKT (0x16), MSCSEEK1 (0x19), MSCPAUSEON (0x03), 0x22 |
| 0x29 | Scanning: status 0x0300. When the drive plays, seeks or reaches the lead-out, pauses (0x03) | MSCSCANFF (0x05), MSCSCANFR (0x06) |
| 0x2A | Reading data: watches the CD data controller. On an error it starts the read again, up to 3 times; then pauses (0x0C) | ROMREAD (0x17), ROMREADN (0x20), ROMREADE (0x21), ROMPAUSEOFF (0x09) |
| 0x2B | Data read paused, or held after a data seek: status 0x0505. Counts frames; after the spin-down delay, standby (0x0B) | ROMSEEK (0x18), ROMPAUSEON (0x08) |
| 0x2C | Power-on: drive light "ready", the CD data controller stopped. If the tray is open or moving, goes to 0x01; otherwise reads the table of contents as DRVINIT does, then 0x22 | The BIOS's start-up (its first state) |

- **The Model 1 BIOS 1.10** (read in ares): its table has 0x00-0x2B, 44 entries (2.00W: 45). It has no 0x2C: the
  power-on step with the tray check is new in 2.00W. Its 0x22-0x2B were not compared one by one with 2.00W's.
- **Also undocumented in the same table (2.00W):**
  - 0x00: no disc (status 0x1000, light "ready");
  - 0x01: no disc, a bad disc or a drive error (light "error"), then DRVOPEN's steps: the tray is opened if it
    is not open already (0x24 and 0x2C go here when the tray opens, the disc is missing or the tray moves);
  - 0x0B: standby: the drive stopped after the spin-down delay, the last play kept for a resume;
  - 0x0C: pause after a data read failed three times;
  - 0x0D-0x0F and 0x1A-0x1F do nothing.

### 22e. Backup RAM: the two undocumented calls, and what format does

**`_BURAM` 9 and 10** (megadev: `BRM__UNKNOWN0` / `1`; 2.00W code):
- 9 copies the first 64 bytes of the internal backup RAM (`0xFE0000`, odd bytes) to the buffer at A0.
- 10 writes 64 bytes from A0 there, then checks the first 4.
- Raw bytes, no file system. Not proposed.

**BRMFORMAT (6)** (2.00W code). It does not clear the data area. It rewrites the backup RAM's format block, which
BRMINIT finds:
- the volume name `___________`;
- the number of files: 0;
- the free blocks: all;
- the marks `SEGA_CD_ROM` and `RAM_CARTRIDGE___`.

So every file of every game is gone from the directory: the space counts as free and is written over by the next
saves. Call BRMINIT first (megadev). Proposal 96 asks for a key and the game asks the player twice.


## 23. PAD2-1 (Metal Slug CD32X, 3 Nov 2026): pad 2 in command 48's reply (Sub-CPU `main.c`, switched)

**Boot stack: one change, behind a switch** (`#if OL_PAD2`, `PAD2 ?= 0`, block `DX-PAD2` in `main.c` and the
`Makefile`; `PAD2=0` builds `main.c` as before: `tools/check_boot.py` takes the block out before the hash and
fingerprints it in `GENPLANE_BLOCKS.md5`). **No new command number, no listener change**: the 68000's vblank
handler already reads both pads (`hw_md.s`: `get_pad` 0 into `0xA12018`, `get_pad` 1 into `0xA1201A`); the
Sub-CPU's pad fast path replies with the second one too.

| | Command 48's reply (before) | `PAD2=1` |
|---|---|---|
| COMM8 | pad 1 (`SEGA_CTRL_*`, `0xF000` = no pad) | the same |
| COMM10 | the 68000's vblank counter (low 16 bits) | **pad 2** (`GET_PAD(1)`, the same bits; `0xF000` = no pad) |

- A game built for `PAD2=1` must not use COMM10 as a clock; one that does (Kobo's 30 ms step) keeps `PAD2=0`.
- Metal Slug CD32X (MicroNut99/FBNeo_MetalSlug `tools/edcd/build_edcd.sh`): the launcher's (MODE1) and the
  disc's Sub-CPU programs built `PAD2=1`; the game (`m3main.c`, `M3_PAD2`) reads COMM10 as the Neo Geo's player 2.
- Status: built `PAD2=1` and run in ares (Metal Slug CD32X v8: a whole attract and every mission load with the
  game reading COMM10; no pad 2 pressed - the ares headless runner presses pad 1 only). Not yet on the console.

## 24. FBVDP: command 79 and listener routine 39, the Genesis VDP from a list in the 32X frame buffer (3 Nov 2026)

**Owner:** the BootCamp stack, at the owner's request for Dracula X CD32X ("Yes, add the FB→VDP command", 3 Nov 2026;
DraculaX_32X docs/60-61).

**Why:** commands 62-67 take their data from the cart (63, 64, 67) or one word a command (62, 65, 66). The SH2 cannot
write the cart (CD32X-BootCamp docs/06 rule 9), and one word a command is about **0.44 ms** (ares, console timing:
2,000 command-62 writes in 53 refreshes) - a frame carries about 35. A running game's background is made on the
console each frame: map words (28-64 a column scrolled in), tiles converted for their palette, the per-line H scroll
table (up to 224 words), colours. The 32X frame buffer is the one memory both the SH2 and the 68000 reach.

**Switch:** `FBVDP ?= 0`, needs `GENPLANE=1` and `VSPLIT=1` (its table entry sits after VSPLIT's routine 38).
- `main.c` gets `-DDX_FBVDP` through **HWFLAGS**; `hw_md.s` gets `--defsym DX_FBVDP=1`.
- `build.sh` stops unless `FBVDP=1` comes with `GENPLANE=1 VSPLIT=1` and reached `main.c` and the listener (`fb_list`).
- `tools/check_boot.py` strips the `DX-FBVDP` blocks (also the two nested in `DX-VSPLIT` blocks) and fingerprints
  them in `stack/GENPLANE_BLOCKS.md5`.

| CMD | Name | COMM2 | COMM8L | Reply | 68000 routine |
|---|---|---|---|---|---|
| 79 | the Genesis VDP from the frame buffer | 0: run the list at frame buffer offset COMM8L (even, below `0x20000`); 1: the commit - run the part the last lists kept, then forget it; 2 / 3: run the list as op 0, then show 32X frame buffer 0 / 1 and run the kept part at the vblank that shows it (added 3 Nov 2026, below) | the list's offset (ops 0, 2, 3) | COMM12 = 1 ok, 0 a bad list (an unknown header, or no room to keep: the rest is not run) | **39 `fb_list`** (new) |

- **The SH2's side:** it writes the list into the frame buffer it draws in (the one not shown: the 68000 sees the same
  one at `0x840000` + the offset), **gives the frame buffer to the 68000** (`FM` = 0: the byte `0x00` to
  `0x20004000`, then waits for bit 15 to read 0, as CDLOADER's `give_fb`), sends 79, and **takes it back** after
  (`0x80`, waits for bit 15). While `FM` = 0 the SH2 must not touch the frame buffer, the 32X VDP registers or its
  palette (32X manual: such an access is "interrupted by force"), its interrupt handlers included. Nothing in
  `fb_list` touches `FM`.
- **The list** (big-endian words, read with `move.w (a0)+`): runs, each a header and its words.

  | Header | Then | Does |
  |---|---|---|
  | `0x0000` | - | the end |
  | `0x1nnn` | n words | each to the VDP control port `0xC00004` (address commands as two words, register writes, the auto-increment) |
  | `0x2nnn` | n words | each to the data port `0xC00000` |
  | `0x3nnn` | n words | kept in the listener (`fb_keep`, 256 words in all) to run at the commit; they may hold only `0x1nnn` / `0x2nnn` runs |
  | `0x4nnn` | n words: n / 2 pairs (frame buffer word index, word) | kept (`fb_jrn`, 68000 RAM `0xFFC000`, 2,048 words) and **written into the frame buffer at the commit**, after its runs (added 3 Nov 2026, DraculaX_32X docs/65: the SH2 sees only the buffer not shown, so tables it keeps in both buffers go into the other one once the swap has been) |

  Nothing is assumed of the VDP's state: the list sets its own addresses and increment, and the increment is left as
  the list leaves it (62-67's routines set their own).
- **The commit (COMM2 = 1)** runs the kept runs, then writes the kept pairs (`0x4nnn`) into the frame buffer: the SH2
  gives the 68000 the frame buffer for it too (`FM` = 0) and takes it back after. It is for what must change at one vblank: the plane and H scroll table base registers of
  double-buffered maps and tables, V scroll, colours. The game sends its lists during the frame (into the Genesis
  areas not shown) and the commit right after its 32X swap is seen: one command, a few words, inside the vblank.
- **Ops 2 / 3: the swap and the kept runs at one vblank** (added 3 Nov 2026 for the same request, DraculaX_32X
  docs/66). The commit sent by the SH2 from its vblank interrupt started too late now and then: the Sub-CPU polls
  COMM0 through the 68000, and in ares 1 picture in 25 had its top 1-33 lines with the old H scroll. Ops 2 / 3 run the
  list as op 0, then the **68000 writes FBCTL's FS itself** (0 / 1: it holds the frame buffer, `FM` = 0; the 32X
  swaps at the next vblank, at once in a blank) and reads it back: not swapped yet - the kept runs are **armed** and
  `vblank_int` (the Genesis' level 6, line 224: the vblank the swap shows at) runs them first thing; swapped already
  (the list ended in a blank) - the kept runs, and the pairs, at once. Op 1 then writes the pairs (the SH2 sends it
  when it sees the swap). Armed runs still waiting when any command comes (`vblank_int` held off) are run first. The
  SH2 must not swap itself after an op 2 / 3, and must send nothing else that touches the VDP before the swap is
  shown (`vblank_int` interrupts the command loop).
  Since 3 Nov 2026 (DraculaX docs/68): a list that ends during a vertical blank **waits for the display** before
  writing FS (a swap at once in a blank came before the SH2's interrupt could write the sprites' colours for it), and
  **COMM10 = `0xFB00` | the buffer** once FS is written (COMM8L, the list's offset, has been read by then), so the
  SH2's vblank interrupt knows the swap is at this blank before the command's answer comes.
- **Level 4 masked** while it runs (VSPLIT's dispatcher: routine 39 is not in its unmasked list), so `hint_vs` never
  moves the VDP's address under it.
- **Speed (ares):** 10 lists of 4,000 data words (VRAM) in 15 refreshes: **about 6.3 µs a word** during the display,
  the Genesis VDP's own pace for VRAM writes then. A map column is about 0.2 ms, an H scroll table 1.4 ms.

**Status: emulation** (ares, the owner's ares_android headless, 4 MB cart, 3 Nov 2026):

| Test | Result |
|---|---|
| `FBVDP=0` | the listener's bytes and `main.c`'s code identical to the stack before (`GENPLANE` / `VSPLIT` on or off) |
| `hello`, `GENPLANE=1 VSPLIT=1` against `+ FBVDP=1` | frames 1800 and 2100 identical, `4B10` |
| DraculaX_32X `dxspeed` `FBVDP_TEST` | a list setting CRAM 0 red, the backdrop register and the display on: the Genesis backdrop red; then 10 lists of 4,000 words: every reply 1, 15 refreshes |

| DraculaX_32X `dxspeed` `BG=1` (3 Nov 2026, docs/65) | the running game's background on plane A: each picture's list (op 0, not waited for: the game's code runs meanwhile) and the commit in the vblank after the swap with the `0x4nnn` pairs; stage 4's 600 pictures in ares, screenshots equal to the PC's pictures |

| DraculaX_32X `dxspeed` `BG=1 SEC=2` (3 Nov 2026, docs/66) | ops 2 / 3 for each picture, op 1 for the pairs: stages 4, 2 and 5, 600 pictures each in ares, 303 screenshots (one every 7 frames) equal to the PC's pictures, top lines included; `hello` with `FBVDP=1` the same at frames 1800 and 2100 |

**Open on the console:** the 68000 reading the frame buffer at this pace while the SH2 is off it; the `FM` hand-over
between two commands of a frame.


## 25. TILEP (Metal Slug CD32X, 3 Nov 2026): command 99 and listener routine 40, a tile page's cache (switched)

**Owner:** Metal Slug CD32X (MicroNut99/FBNeo_MetalSlug), at the owner's request ("no level should be missing tiles",
3 Nov 2026). **Numbers:** command 99 - free on every stack (sections 3 and 4); 91-98 stay clear for section 22's
proposals. Listener routine **40** (`tile_copy`) - after FBVDP's 39; when GENPLANE, VSPLIT or FBVDP is off, its
numbers 36-39 are `cmd_nop` in the table, so 40 is 40 on every build.

**Why:** the SH2 sees one 2 MB page of the cart at a time (window 1, command 68). A game whose sprite tiles are
cut into pages by place draws a picture from the page shown; a picture that needs a tile of another page cannot draw
it. With this command each page keeps P slots: a tile another page has is copied in when a picture wants it.

**Boot stack: behind a switch** (`TILEP ?= 0`: `#if OL_TILEP` in `main.c`, `--defsym DX_TILEP=1` for `hw_md.s`;
blocks `DX-TILEP` in `main.c`, `hw_md.s` and the `Makefile`, taken out before the hash by `tools/check_boot.py`).
Needs the EverDrive launcher's listener (MODE1 + M1PAGE): built without them, routine 40 returns -1 and 99 replies
`0xFFFF`.

**TILEP-1 (first version, replaced the same day):** an op per tile through the listener's routine 26 and MODE1-9's
page register - nine copies and two page writes, each a listener call: ~10 ms a tile in ares (Metal Slug, 4,585
tiles in 230 s of play, the game down to ~30 % where it copied most). Not kept.

**TILEP-2 (this one):** one command for all a picture's tiles; the 68000 does the copies itself and the SH2 hands
it the tiles through the 32X COMM registers while the command runs.

| | |
|---|---|
| COMM2 (at the command) | p: the page window 1 shows (1-6) |
| COMM8L (at the command) | where the tile part (`MSPG`) starts in every page |
| then, tile k = 0, 1, ... | the 68000 puts `0x8000 \| k` (12 bits) in COMM12; the SH2 puts the leaf's offset in page p / 4 (COMM4 = bits 0-15, COMM6 bits 0-2 = bits 16-18), the page q that has the tile (COMM6 bits 8-10), the tile's index j in q (COMM8), the slot s (COMM10), and last `0x4000 \| k` in COMM2; `0x7FFF` in COMM2 = no more |
| Reply COMM12 | the tiles copied; `0xFFFF` refused (p not 1-6, the offset, no `MSPG` in page p, or no slots) |

- **The page (Metal Slug's `tile_pack.py --patch`):** the header's rows and flags offsets (bytes 28, 24), the first
  slot's index (36), the slots (40) and the slot table's offset (44); a tile is 128 bytes of rows at rows + index ×
  128 and 4 bytes of flags at flags + index × 4.
- **Routine 40, `tile_copy(p, off)`** (`hw_md.s`): page p's header read once. Each tile with RV = 1 and interrupts
  masked: window 1 = page q (CTRL4-7 = 4q..4q+3, MODE1-9's banks), q's rows and flags to `m1_buf`; window 1 = what
  `m1_wb` says (page p); the slot table's entry s (a leaf's offset, its value before) read; then **with W on and no
  other write by the 68000** (MODE1-6): that leaf gets its value back (the slot's last tile is "not here" again), the
  rows and flags into p's index first + s, the table's entry = this leaf, `0x80000000 | q << 24 | j`, and last the
  leaf = index << 16 | `0xFFFE`; W off, RV = 0. A tile whose page q has no `MSPG` is skipped. The COMM waits are
  bounded (~2 s: the routine then ends and replies what it did).
- **`main.c`:** reads COMM2 and COMM8L, calls routine 40 once, replies. No Word RAM.
- **The SH2's side:** both SH2s off the cart for the whole command (Metal Slug: `cd_tiles`, the slave parked in
  SDRAM, the master waiting in SDRAM with its interrupts masked, as command 68's `cd_rv`).
- **Status (3 Nov 2026): ares only, not on a console.** Metal Slug's EDCD build (`build/edcd_v12`, the launcher's
  and the disc's Sub-CPU programs built `TILEP=1`), 230 s of play in ares (console timing): 5,667 tiles copied in,
  **0 tiles missing**, the picture's build step 8.3 ms (TILEP-1: 18.3 ms), the game at 72 % of its speed (TILEP-1:
  66 %). The graphics checked in screenshots.
- **4 Nov 2026 (v22's stack):** routine 40's two 128-byte copies 16 bytes a step (`movem.l` of four registers,
  unrolled) instead of a long at a time: ares, Metal Slug's recorded Mission 1 (1,000 vblanks), 0.505 -> 0.463 ms a
  tile as the SH2 waits (most of the rest is the handshake and the listener's latency, not the copies).
  The second copy block had pushed a register on the 68000's stack with W on (MODE1-6: no other write while W is
  on); it uses d7 now, no push (4 Nov 2026).

## 26. ROMPAUSE-1 (Metal Slug CD32X, 3 Nov 2026): the read-ahead pauses the drive (Sub-CPU `main.c`, switched)

**Owner:** Metal Slug CD32X, at the owner's request (3 Nov 2026: section 22b's "pause the data read instead of losing
it", "implement it anyway"). **No number:** it changes STREAM's read-ahead (section 10) inside commands 46 / 47.

**Boot stack: behind a switch** (`ROMPAUSE ?= 0`, `#if OL_ROMPAUSE`, blocks `DX-ROMPAUSE` in `main.c` and the
`Makefile`, out of the hash by `tools/check_boot.py`). Meaningful with `STREAM=1` only.

- `cd_pump`: when the 64 KB ring (`st_buf`, 32 sectors) holds 24 or more sectors of a stream still running, the
  drive is paused with its place kept: BIOS **ROMPAUSEON (0x08)** (state 0x2B, section 22a). The ring keeps
  8 slots (`ST_PAUSE_ROOM`) for the sectors still on their way; the pump goes on taking them.
- `cd_chunk_read` (the next chunk asked for): **ROMPAUSEOFF (0x09)**, the read goes on from its place (state 0x2A).
- A new read (`scd_stream_start`: CDCSTOP + ROMREADN) and any other use of the drive (`scd_sub_read`,
  `scd_open_file`, `scd_bios_call`: the CD audio, the PCM packs) drop the pause with the stream (`st_kill`), so a
  paused data read never meets a music call. Every sector's header is checked as before: one that does not follow on
  starts the read again, as without the switch.
- **Status (3 Nov 2026): ares only, not on a console.** Metal Slug's EDCD (from the recorded game at the end of
  Mission 2: Mission 2's 4 pages, then Mission 3's, from the disc), counters shown in COMM14 by a diagnostic build
  (not kept): without the switch, 2 lost-sector restarts in Mission 2's load; with it, 4 pauses in that load and 7 by
  Mission 3's end, nothing lost, the game the same. Both loads 42 s either way (at ares's 1-second log): the time
  goes to the drive's speed (5.7 MB at 150 KB/s is 38 s) and a seek per file, not to lost sectors. With ZCHUNK
  (section 27) the drive outruns the unpacking and the ring fills: pauses then, resumed only when the ring is low
  (ZCHUNK's `cd_chunk_read` condition), so one seek serves many chunks.

## 27. ZCHUNK-1 (Metal Slug CD32X, 3 Nov 2026): command 100, CMD 46 from an LZ4-packed file (Sub-CPU `main.c`, switched)

**Owner:** Metal Slug CD32X (the owner: "There should not be much loading"). **Number:** command **100** (free on
every stack: section 3's list, "100 and up"). No listener routine.

**Why:** a mission load copies ~5.7 MB (4 pages of 32 KB chunks) from the disc; the drive gives 150 KB/s at most, so
~42 s however the reading is done (section 26). Metal Slug's tile pages pack ~2.1x with LZ4: the drive reads half.

**Boot stack: behind a switch** (`ZCHUNK ?= 0`, `#if OL_ZCHUNK`, blocks `DX-ZCHUNK` in `main.c` and the
`Makefile`, out of the hash by `tools/check_boot.py`; with `MODE1=1` the Makefile adds `-DOL_ZMODE1=1`). Needs
`STREAM=1`.

| CMD | COMM2 | COMM8L | Reply COMM12 |
|---|---|---|---|
| 100 | as 46: the chunk (bit 15 = no read-back) | the chunk's file length as 46 sees it (Metal Slug: 0x200000, the page) | 0 ok, 2 a bad block (as 46: non-zero = failed) |

- **The file** (Metal Slug's `tools/edcd/zpack.py`): sector 0 = `MSZ1`, the number of chunks (u32), then 12 bytes a
  chunk, big-endian: its first sector (u16), its sectors (u16), the packed length (u32; bit 31 = stored as it is),
  the chunk's length (u32). Each chunk's block starts a sector, in order, so the stream reads on from one to the next.
  The blocks: LZ4's block format (`lz4.block`, high compression), at most 34 KB.
- **The command:** the table (sector 0) read once a file (`zt`); the block read with `cd_chunk_read` into `zbuf`,
  unpacked into the Sub-CPU's Word RAM bank, where CMD 46 reads its chunk; then 46's copy to the cart. The
  unpacker is 68000 code (`lz4_unpack`, ~100 ms a chunk in ares: the C version took ~230 ms), pumping the read-ahead
  every 4 KB.
- **With the cartridge-boot listener (`OL_ZMODE1`):** the copy is one listener command of 32 KB (`m1_xfer` masks
  the 68000's interrupts 1 KB at a time), sent without waiting; meanwhile the Sub-CPU unpacks the **next** chunk
  into `zout` (PRG-RAM); the next command, if it asks for that chunk of that file, copies `zout` into Word RAM
  instead of reading and unpacking. And no read-back, window check or text for 100 (5 listener calls a chunk fewer).
- The read-ahead ring is 96 sectors (192 KB) instead of 32 when the switch is on (`ST_RING`). The program's end
  goes from ~0x38800 to ~0x60800 (PRG-RAM, the stack below 0x80000).
- **Status (3 Nov 2026): ares only, not on a console.** Metal Slug's EDCD from the recorded game at the end of
  Mission 2: Mission 2's 4 pages and Mission 3's, each load **30 s** (unpacked, the same build: 42 s); the game the
  same after. Per chunk (COMM14 in a diagnostic build, not kept): ~100-115 ms unpacking, the 68000's copy done
  within it, ~2 vblanks for the rest. The drive's own time is ~110 ms a chunk (8.2 sectors), so the unpacking
  is now the limit.

## 28. CDVOL-1 (Metal Slug CD32X, 3 Nov 2026): the CD audio's volume (Sub-CPU `main.c`, switched)

**Owner:** Metal Slug CD32X (the owner: "the audio is bad and needs some focus"). **No number:** a build value.

- `CDVOL ?= 0` (`#if OL_CDVOL`, blocks `DX-CDVOL` in `main.c` and the `Makefile`, out of the hash by
  `tools/check_boot.py`): with 1-0x3FF, command 45 sets the CD audio's volume with BIOS FDRSET (0x85) to that
  value at each track's start instead of full (0x400). Section 22b's 82 (a volume / fade command) is not built.
- **Why:** in ares the PCM effects came out ~10 dB under the CD music; in the arcade (FBNeo's renders) they are as
  loud as it. Metal Slug: `CDVOL=0x200` (-6 dB) with its effects soft-limited (+3-7 dB, its `make_sfx.py`): the
  attract's effects alone and the music in play then measure alike (ares, 50 ms windows, median ~1,200-1,300).
- **Status (3 Nov 2026): ares only.** On a console the balance may differ (the CD's analogue mix): the value is a
  build switch so it can be set again without code.

## 29. CDRING-1 (Dracula X CD32X, 3 Nov 2026): command 91, sectors of the open file into the cart (Sub-CPU `main.c`, switched)

**Owner:** the BootCamp stack, at the owner's request for Dracula X CD32X ("Yes: chunk ring, my size", 3 Nov 2026;
DraculaX_32X docs/69).

**Why:** the PC Engine's data track (20.8 MB) is read by the game while it runs, a few sectors at a time, anywhere in
the track. Command 46 fills a file's fixed cart slot 32 KB by chunk number; `FMVRING`'s ring is 60 chunks (1.9 MB),
more than the 4 MB cart has free beside the game.

**Switch:** `CDRING ?= 0` (`main.c` `OL_CDRING` through HWFLAGS). `build.sh` checks it reached `main.c`; a `*.TRK` file in
the game's `cd/` folder skips the slot-size check (command 91 never loads it whole). `tools/check_boot.py` strips the
`DX-CDRING` blocks into `stack/GENPLANE_BLOCKS.md5`.

| CMD | Name | COMM2 | COMM8L | Reply |
|---|---|---|---|---|
| 91 | sectors of the open file into the cart | the first sector in the file (CMD 38 opened it) | bits 0-23 the cart byte offset (2 KB aligned, the end below 0x3F0000); bits 24-27 the sectors - 1 (1-16) | COMM12 = 0 ok, 1 refused |

- The read and the copy are command 46's: `STREAM`'s read-ahead (`cd_chunk_read`) or `scd_sub_read`, then `CARTW`'s
  word copies (MD 27) or byte copies (MD 33), 4 KB at a time with `RV` = 1.
- Past the file's end the count is cut; nothing past the cart's save area is written.
- 91 was free (the ledger's list; 81-85 are left to section 22's proposals).

**Status: emulation** (ares, 3 Nov 2026): the DraculaX build `CDR=1 CDRING=1` on stage 4's window opens the track and
runs the game with its drive; every chunk the game asked for arrived (the window reads none on stage 8, its PC log
says). The game's speed with the drive is the DraculaX model's to improve (docs/69), not the command's.

## 30. FBCART-1 (Dracula X CD32X, 3 Nov 2026): command 101, the 32X frame buffer into the cart (Sub-CPU `main.c`, switched)

**Owner:** the BootCamp stack, for Dracula X CD32X ("you can edit and test the boot stack on your own", the owner, 3 Nov
2026; DraculaX_32X docs/72).

**Why:** the PC Engine's RAM (8 KB base, 64 KB CD RAM, 192 KB Super RAM) is more than the SH2 can keep writable in
SDRAM beside its code. Its banks live in the cart, where the SH2 reads them; a bank the game writes is copied into a
small SDRAM pool and, when its slot is needed again, written back to the cart. The SH2 cannot write the cart; the
68000 can, and it can read the 32X frame buffer. Measured over the game's recordings: 60-175 banks written back in five
minutes with a one-slot pool (DraculaX docs/72).

**Switch:** `FBCART ?= 0` (`main.c` `OL_FBCART` through HWFLAGS). `build.sh` checks it reached `main.c`.
`tools/check_boot.py` strips the `DX-FBCART` blocks into `stack/GENPLANE_BLOCKS.md5`.

| CMD | Name | COMM2 | COMM8L | Reply |
|---|---|---|---|---|
| 101 | the 32X frame buffer into the cart | the frame buffer byte offset >> 4 | bits 0-23 the cart byte offset (even, the end below 0x3F0000); bits 24-27 the length in 2 KB - 1 (2-32 KB) | COMM12 = 0 ok, 1 refused |

- The SH2 gives the 68000 the frame buffer (FM = 0) before the command and takes it back after, as for command 79.
- The copy is the listener's `safe_cart_xfer_w` (MD 27: words, `RV` = 1, interrupts off), from 0x840000 + the offset to
  0x400000 + the cart offset, 4 KB at a time.
- Refused (nothing written): an odd cart offset, an end past the cart's save area or past the frame buffer's 128 KB.
- 101 was free on every stack (the ledger's list; 92-98 stay with section 22's proposals).

**Status: emulation** (ares, 3 Nov 2026): see DraculaX docs/72.

## 31. CDPAUSE-1 (Metal Slug CD32X, 4 Nov 2026): command 81, the CD music paused in place (Sub-CPU `main.c`, switched)

**Owner:** Metal Slug CD32X (its credits screen's CD player: Start pauses the track and plays it on from the same
place, as Rayman CD32X's player does with the same command built in its own copy of the stack,
`tools/stack_music.py`). Section 22b's proposal 81, built as proposed; 82 (volume / fade) is not.

- `CDPAUSE ?= 0` (`#if OL_CDPAUSE`, blocks `DX-CDPAUSE` in `main.c` and the `Makefile`, out of the hash by
  `tools/check_boot.py`). With 1, command 81: COMM2 = 1 pauses the CD music where it is (BIOS MSCPAUSEON 0x03), 0
  plays on from there (MSCPAUSEOFF 0x04). Both are queued BIOS calls (section 22a): the drive is let finish what it
  is doing first (`bios_wait_ready`, as 54). No reply; COMM0 cleared when the call is made.
- A drive paused past the spin-down delay (60 s, CDBPAUSE) goes to standby; MSCPAUSEOFF then starts the kept play
  again with a seek (22a). 45 after a pause starts a track from its top as always; 54 stops.
- **Status (4 Nov 2026): ares only.** Not run on a console.


## 32. JF-G (Jumping Flash CD32X, 7 Nov 2026): command 82 and listener routines 45-48, the ground on the graphics chip (switched)

MicroNut99/JumpingFlash_CD32X (the BootCamp stack), Makefile `JFGROUND=1` (`build.sh` passes it to the game, the
Sub-CPU program and the listener). **`JFGROUND=0` (the default) builds `main.c` and `hw_md.s` as before** (all of
it under `#if` / `.if`). The pattern is OpenJazz REV's bonus floor (command 71, section 19); Jumping Flash's camera
turns, rises and looks up and down. (First numbered 72 / 36-39 in the working copy; 72 is Rayman's and 36-37
SuperTux's, so it was renumbered before any release.)

| CMD 82, COMM2 low 2 bits | What it does | Reply COMM12 |
|---|---|---|
| 1 LOAD | COMM8 = the ground data (`tools/jf_ground.py`: "JFG1", 16 colours, 4 stamps 32 x 32) as the 68000 reads the cart (0x400000 + offset). Word RAM to 2M; the 68000 copies the data in (MD 48) and sets the VDP up (MD 45) | 1; 0 = no data / a CD boot |
| 2 FRAME | COMM2 bits 2-11 yaw, 12-15 height / 1024; COMM12 = pitch (10 bits) \| (height / 16 & 63) << 10; COMM8 = camera x / 8 << 16 \| z / 8. The chip draws up to 80 rows of 320 for the lines below the horizon; the 68000 shows them at the 32X's next swap (COMM6 = 0x4000 \| a bit that changes each picture) | 1; 0xFFFF = not loaded (another command took Word RAM: LOAD again) |
| 3 OFF | back to 1M, the Genesis display off (MD 47) | 1 |

- **Sent without waiting:** the SH2 writes command 82 FRAME and goes on; every other command (the pad first)
  waits for COMM0 = 0 before it writes its arguments (`cdapi.c` `cd_idle`).
- **Which commands end the ground** (they may need Word RAM in 1M): any but 0, 48 (answered first) and 82.
- **Listener routines 45-48** (`jfground_md.s`, included by `hw_md.s` only when `JF_GROUND` = 1; 36-44
  `cmd_nop` in that build; `MD_CMD_END` = 49). Free on every listener in section 2's table (OpenJazz REV's 45 is
  `cmd_nop`): **45 `jg_setup`** (VDP, colours, the H interrupt, Word RAM to the Sub-CPU), **46 `jg_queue`** (a
  picture is ready in Word RAM), **47 `jg_off`**, **48 `jg_copy`** (dst, src, len: the cart read through the 32X's
  1 MB window at 0x900000 with **RV = 0** - with RV = 1 the SH2s read garbage from the cart and the master crashed
  in its vblank handler, ares 7 Nov 2026; a cartridge boot only, a CD boot returns 0).
- **The vblank** (`jg_vblank`, at the start of `vblank_int`): display off; at the SH2's swap the ready picture is
  shown (plane A at 0xC000 or 0xE000, its line table); a new picture is copied by DMA into the other one, as many
  columns as the display-off lines allow (more vblanks if needed), then Word RAM goes back to the Sub-CPU. **The H
  interrupt** every line turns the display on at the horizon and sets plane A's V scroll per line.
- **What it costs the SH2s:** the video DMA holds the Genesis bus, and the 32X reaches the cart over it: while it
  runs the SH2s cannot read the cart. 104 rows took the FPGA build from 30 to 24 pictures a second; 80 rows keep 30
  (ares, console timing).
- Status: **emulation (ares)**, not yet on a console.
