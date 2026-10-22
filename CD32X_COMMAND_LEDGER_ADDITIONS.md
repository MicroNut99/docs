# CD32X – Command Ledger additions

An addition to **KOBO-Sega 32XCD Boot Sequence and Sub-CPU Command Ledger.pdf**. It records the
commands added to the boot stack after that ledger was written, so every CD32X project uses one
registry. As the OpenTyrian addendum asks: **never reuse a number with a different meaning.**

Stack: the OpenLara / Hurrican CD32X boot path (Sub-CPU `main.c`, 68000 listener `hw_md.s`),
as used by SuperTux CD32X REV.

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

**67 status: added in SuperTux CD32X REV4 (15 Oct 2026), boot test pending** (Fusion, then the
console - as 62–66 were).

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

The ledger PDF's 68000 list ends at 34 `SET_VDP_REG`, 35 `END`. The current list:
34 `set_vdp_reg` (Kobo K19), **35 `start_d32xr`** (D32X M1), **36 `set_vsram`** (REV2),
**37 `copy_vram_col`** (REV4), `MD_CMD_END` 38.

## 3. The SH2 command numbers in use (this stack)

| CMD | Meaning |
|---|---|
| 38 | open file (length → COMM8) |
| 44 | clear the Genesis planes (only acknowledged while `SLIDESHOW_DEBUG` is 1, as in this stack) |
| 45 / 54 | play CD track (bit 15 = once) / stop music |
| 46 | chunk → cart (bit 15 = no read-back); reply COMM12 |
| 47 | chunk compare with the cart (`CMD_CART_TO_FB` also has the value 47 in `main.c`; its branch is never reached) |
| 48 | pad (+ sound effect in the high byte); reply COMM8 pad, COMM10 ticks |
| 50 / 51 | file chunk → frame buffer (off-screen / splash) |
| 52 | Genesis display on / off |
| 53 | load sound pack |
| 56 / 59 | backup RAM load / save |
| 57 / 58 | backup RAM word get / put |
| 60 | D32X: start D32XR from the cart (never returns) |
| 61 | save area: 4 KB piece → cart 0x3F0000 + COMM2 × 4 KB; reply COMM12 |
| **62–67** | **the Genesis picture (section 1)** |
| **68** | **cart page, window 1 (section 7)**: 8 / 16 MB SRAM cart, OpenLara CD32X-REV test builds; SuperTux CD32X REV `CART_MB=8/16` builds (REV11-16MB-T1) |

Free: 39–43, 49, 55, 69 and up.

## 4. Numbers that mean something else on the cartridge stack (OpenTyrian, `cart_md.s`)

The OpenTyrian addendum lists its own numbers: **60** CD find, **61** CD read, **62 / 63** SRAM
read / write (streamed). Those belong to the cartridge stack, not this one, but they clash with
60–63 here. Code moved between the two stacks must check this table. New commands on either stack
should take numbers free on both (68 and up).

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
| 13 | 0x3FD000 | **SuperTux CD32X REV5**: options, high scores, 7 game slots ("STX2", 368 bytes) |
| 14 | 0x3FE000 | **OpenLara CD32X-REV** (SAVE-2): the Caves record `'OLCV'` at `0x3FE000` and the level slot record `'OLLV'` at `0x3FE010` (section 8). Rewritten after every level load from the CD |
| 15 | 0x3FF000 | **Cart test scratch** (OpenLara CD32X-REV `CART=` builds; SuperTux CD32X REV `CART_MB=8/16` builds - its size probe writes 8-byte markers here on pages 1, 2 and 6 at every start): a 4 KB test pattern per page, written and read back at every start of a test disc. Never holds data worth keeping; a game must not store anything here |

Free: 11, 12. (OpenLara CD32X uses piece 0 and on: `0x3F0000` settings, `0x3F0400` saved game; it and
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
page switch is untested**: it needs the 8 / 16 MB board.

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
  console in OpenLara (on the console in OpenJazz and Hurrican).**

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
