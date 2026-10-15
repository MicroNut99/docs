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

Free: 39–43, 49, 55, 68 and up.

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

Free: 11, 12, 14, 15. (OpenLara CD32X used `0x3F0000` settings and `0x3F0400` saved game -
inside piece 0; it and OpenJazz share the cart only one at a time.)
