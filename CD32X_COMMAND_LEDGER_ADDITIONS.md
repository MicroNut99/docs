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
| 15 | 0x3FF000 | **Cart test scratch** (OpenLara CD32X-REV `CART=` builds; SuperTux CD32X REV `CART_MB=8/16` builds - its size probe writes 8-byte markers here on pages 1, 2 and 6 at every start): a 4 KB test pattern per page, written and read back at every start of a test disc. Never holds data worth keeping; a game must not store anything here |

Free: 11, 12, 14. (OpenLara CD32X used `0x3F0000` settings and `0x3F0400` saved game -
inside piece 0; it and OpenJazz share the cart only one at a time.)

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

## 7. SH2 → Sub-CPU command 68: cart page, window 1 (OpenLara CD32X-REV)

For the paged 8 / 16 MB SRAM cart (MicroNut99/16MB-PCB-CD32X): the SH2s can't reach the cart's
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
- **Status:** run in MicroNut99/ares_android (headless, the 16 MB board emulated): the probe finds
  4 MB and 16 MB, the first-start copy, the resume after a power-off, later starts with no copying,
  the 4 MB cart unchanged. On the console: the first two (broken) discs booted on the 4 MB cart
  after the stop; the fixed build is not yet run on the console, and a real page switch still
  needs the 8 / 16 MB board.

