# Mega EverDrive PRO: the SD card, the PSRAM, and preloading a CD32X game from the card

Written 31 Oct 2026 after a full reading of MicroNut99/CD32X-BootCamp (commit `a8c8afd`), the game
ports (OpenLara REV `a9f13c4`, OpenJazz REV `d486f58`, Hurrican `009912a`, SuperTux REV `83032ed`,
Sonic CD 2011 `bc3c30b`, D32XR `91a7baf`, Rayman `ab5e08b`), krikzz's public EverDrive sources
(krikzz/mega-ed-pub `7064f45`) and this folder. It records what is known about the EverDrive's SD card
path and its PSRAM, which of it is console-measured and which comes from krikzz's source, and a design
for loading a CD32X game's data from the SD card into the PSRAM instead of from the CD. **Nothing in
section 4 has run on a console.** Every rule in section 1 says where it was measured.

Sources named below: `[edtest]`, `[edw0]`, `[edsave]`, `[edsave3]`, `[edpage]` are the BootCamp's
console probes (docs/10, docs/12 there); `[krikzz]` is mega-ed-pub, with the file; `[sdcard/]` is the
BootCamp's `sdcard/` kit (ares only, 27 Oct 2026); `[OL-EDCD]` is OpenLara CD32X-REV EDCD, console-proven
26 Oct 2026.

## 1. What is known about the EverDrive PRO as a CD32X cart

### 1.1 The memory (console-measured unless marked)

| Fact | Measured | Where |
|---|---|---|
| 16 MB in 32 banks of 512 KB, any bank in any of eight 512 KB windows (`CTRL0-7` at `0xA130F0-FE`), for a ROM whose header says `SEGA SSF` and was started from the EverDrive menu | console, 24 Oct | `[edtest]`, all 27 writable banks 0 bad words |
| Banks 0-15 are PSRAM chip 0, 16-30 PSRAM chip 1, **bank 31 is the 512 KB battery RAM** | krikzz RTL | `[krikzz]` `fpga/mapper/map_ssf/map_ssf.sv:58-61`: `rom_addr[23]==0 → rom0`, `rom_addr[23:19]!=11111 → rom1`, else `bram` |
| **Banks 0-29 (PSRAM) are wiped every time the EverDrive menu runs.** Only bank 31 survives | console, 25 Oct | `[edsave]`, three runs |
| Bank 31 goes to the SD card as `MEGA/gamedata/<launcher name>/bram.srm`, **256 KB written**, only when the launcher's header carries the `RA` save-RAM entry (`EDCD_SAVE=1` in `edboot/make_edcd.py`) | console, 25 Oct | `[edsave]`; the entry is `RA F8 20 00200001 0023FFFF` |
| CTRL registers are write-only, 16-bit writes; **an even-byte write to CTRL1-7 is ignored**; a word write is taken | console, 26 Oct, and the spec | `[edpage]`; `[krikzz]` `extended-ssf.md` "accept 16-bit writes only" |
| `W` (CTRL0 bit 13) must be **off** except inside a cart write: with it on, a 68000 write anywhere else lands in the cart and an SH2 cart read at that moment returns 0 | console, 25 Oct | `[edw0]` |
| Window 0 (`0x000000-0x07FFFF`) cannot be written by the 68000 directly with the 32X on; the copy goes through window 1 with window 0's bank mapped there | console, 25 Oct | docs/12 MODE1-3 |
| An SH2 cart access while the 68000 holds RV = 1 is **answered at once with garbage**, not held | console, 26 Oct | `[edsave3]`, 965,207 bad of 978K |
| The `C` bit (CTRL0 bit 11, "#CART") changes nothing: the EverDrive stays the cartridge at `0x000000` | console, 25 Oct | `edboot/EDPROBE.BIN`. **krikzz's public RTL agrees**: `map_ssf.sv:68` is `assign cart = ssf_ctrl[0];` to a dangling wire, never to `mout.cart`, so the bit is decoded and goes nowhere |
| Writes to PSRAM and bank 31 from the 68000 are gated by `W` (`wr_on`); **the MCU's own writes are not** (they come in on the DMA bus, which `mem_ctrl` switches in front of the mapper) | krikzz RTL | `map_ssf.sv:32-45`, `lib_base/var.sv:19-23` |
| The register writes reach the cart with the 32X on **only with RV = 1** | console, 25 Oct | docs/12 ("writing the mapper with RV = 0 left the EverDrive write-protected") |

So, for a CD32X game started from `EDCD_<GAME>.32X`: the game, its packs and its "resident" data are
copied from the CD into PSRAM at **every** launch (the menu wiped them), and the saves live in bank 31
through the Sub-CPU (`[OL-EDCD]`, `[sdcard/]`). "Is it already in the cart?" checks can never hit.

### 1.2 The SD card as the EverDrive uses it today (what a CD32X game can rely on)

- `bram.srm`: the OS reads bank 31 from the card when a launcher with an `RA` entry starts, and writes the
  bank back when the menu runs again. Layout of the 256 KB per launcher, as the kits use it: OpenLara
  `0x00000-0x2FFFF` (save area at 0, 16 slots at `0x10000`), Rayman `0x30000-0x3FFFF` (3 slots), the
  `sdcard/` demo `0x00000` (`SDV1` records). Each launcher has its own file, so the layout only matters
  within one game. (`[sdcard/] README_AI.md §4`.)
- The route of a byte, console-proven: master SH2 → CMD 58 per word → Sub-CPU 4 KB `bram_buf` → Word RAM
  → 68000 listener `m1_xfer` (MODE1-7: cart `0x800000 + piece × 4 KB` = bank 31 through window 1,
  RV = 1, `W` on inside the write only, SR = 0x2700) → bank 31 → the OS → the card. Both SH2s wait in
  SDRAM with interrupts masked meanwhile (EDSAVE-4).
- The commands that do it, per game (all the same code, OpenLara's EDSAVE-1, different numbers; see the
  ledger §16): OpenLara / OpenJazz REV / Hurrican **61 out, 62 in**; SuperTux **61 out, 69 in**; D32XR 61 for
  its loader's record and its own 68000 for the game's piece 11 (CTRL1 = 31, after the stack has ended);
  Sonic's launcher keeps its two backup-RAM files at bank offset `file × 512` inside 56 / 59;
  Rayman **72 out, 73 in** (pieces at `0x30000`); the BootCamp kit **74 out, 75 in** (pieces across the
  whole 256 KB, 0-63). Hurrican's launcher also reroutes **56 / 59** (backup RAM) to bank-31 piece 12.
- Nothing else on the card is reached by any CD32X program today: no game opens a file on the card, and
  the stack has no command for it.

### 1.3 The SD card as krikzz's hardware reaches it (`[krikzz]`, not yet used by any CD32X program)

The SD card and its FAT file system belong to the EverDrive's **ARM MCU**, not to the 68000. The 68000
talks to the MCU through a register block at **`0xA130D0`** (`edio/everdrive.h:64-79`, decoded in
`lib_base/base_io.sv:34-65` for every mapper, SSF included):

| Address | Register | What |
|---|---|---|
| `0xA130D1` | `FIFODATA` | byte in / out of the two 2 KB FIFOs (68000→MCU, MCU→68000) |
| `0xA130D2` | `FIFOSTAT` | bit 15 = MCU→68000 FIFO empty, bit 14 = 68000→MCU FIFO empty, bits 0-10 = bytes waiting |
| `0xA130D4` | `SYSSTAT` | read: `0x55A0 | strobe<<3 | fpga_busy<<2 | mcu_busy<<1 | !gmode`; write: bits 2:1 set the busy flags |
| `0xA130D6` | `TIMER` | free-running 1 ms counter |
| `0xA130D9` / `0xA130DB` | `MEMDATA` / `MEMADDR` | a serial byte window on the **whole** memory (32-bit address shifted in LSB first, bit 31 = write permit, auto-increment); chip from bits 29:23: 0 PSRAM0, 1 PSRAM1, 2 SRAM / battery RAM. Overrides the mapper's bus while active (`everdrive.sv:221-224`) |
| `0xA130DE` | `MBX` | a 16-bit mailbox shared with the MCU |

A command is four bytes into the FIFO: `'+'`, `~'+'`, `cmd`, `~cmd`, then its arguments big-endian,
then a status read (`CMD_STATUS 0x10` → `0xA5xx`, `xx` = error). The file API (`everdrive.c`):

| MCU command | What |
|---|---|
| `CMD_F_FOPN 0xC9` mode, path | open a file on the card (`FA_READ 1`, `FA_WRITE 2`, `FA_OPEN_EXISTING 0`, ...) |
| `CMD_F_FPTR 0xCF` offset | seek |
| `CMD_F_FRD 0xCA` len | read into 68000 RAM through the FIFO, at most 4096 (recommended 512) a block |
| **`CMD_F_FRD_MEM 0xCB` addr, len** | **the MCU writes the file straight into cart memory** at an FCI address (`0x0000000` + offset = PSRAM, banks 0-15 then 16-30 in order; `0x1080000` = battery RAM), while the 68000 waits |
| `CMD_F_FWR 0xCC` / `CMD_F_FWR_MEM 0xCD` | the same the other way (write a file from 68000 RAM / from cart memory) |
| `CMD_F_FCLOSE 0xCE`, `CMD_F_FINFO 0xD0`, `CMD_F_FCRC 0xD1`, `CMD_F_AVB 0xD5` | close, size + date, CRC of a file, bytes left |
| `CMD_MEM_RD 0x19`, `CMD_MEM_WR 0x1A`, `CMD_MEM_SET 0x1B`, `CMD_MEM_CRC 0x1D` | the MCU reads / writes / fills / sums cart memory by FCI address |
| `CMD_DISK_INIT 0xC0`, `CMD_F_DIR_LD 0xC5`, `CMD_F_DIR_GET 0xC8` ... | card init, directory listing |

**The transfer ("DMA") handshake** (`ed_halt_app`, `everdrive.c:655-676`): the 68000 writes
`SYSSTAT = 0x02` (mcu_busy), writes one `0` byte to `FIFODATA`, then spins on `SYSSTAT` until the
strobe bit toggles between two reads and `mcu_busy` is clear. **That loop must run from 68000 work RAM:
the MCU owns the PSRAM, SRAM and battery RAM for the whole transfer** (`edio/main.c:226`: "m68k have no
acces to rom memory during io DMA"). krikzz's MegaColor player streams 46 KB every two frames from the
card into PSRAM this way, non-blocking, with a custom mapper that arbitrates the chip between the MCU and
the CPU (`edapp/mcv`); with the stock SSF mapper there is no arbitration, so a transfer is exclusive.

What the public source does **not** say: any throughput figure (the only numbers are the 512-byte FIFO
advice and MegaColor's ~1.4 MB/s design point, arithmetic from 46,328 bytes per 2 frames); anything about
the 32X (the C headers have `MAP_32X = 2` and `SYS_CTRL_32X = 0x20 "specific reset mode for 32x"`, the
FPGA defs have no mapper 2); the MCU firmware; the USB protocol (`edlink.exe` is a binary).

## 2. Why the SD card matters for CD32X

1. **The PSRAM is wiped at every menu run** (`[edsave]`), so an EverDrive game loads everything from the
   CD at every launch: the ROM (0.1-0.5 MB), the resident pack (1-2.5 MB), the first level. On the
   console the CD path moves 42 KB/s without read-ahead and 104-143 KB/s with it (`CD32X_STREAM_READ.md`),
   so a 2.5 MB resident pack is 20-60 s at every start, on top of the ROM. The battery-backed 4 MB cart
   never had this cost: its "already in the cart" checks hit.
2. **The CD drive is the game's music.** Every data read stops the CD audio (docs/06 rule 15). A game
   whose data came from the card could leave the disc to the music for the whole session.
3. **The card is where the EverDrive already keeps the game's saves.** A game that could read the card
   could also keep more than 256 KB there, and read it without the bank-31 copy chain.

## 3. The one path that needs nothing new: bank 31 (what exists)

The `sdcard/` kit and the per-game EDSAVE commands already give a game 256 KB on the card through bank
31 (§1.2). It is the only SD route that has run on a console (OpenLara EDCD, 26 Oct 2026: save, load,
relaunch, load). It costs 2,048 word round trips plus two 4 KB RV copies per piece, both SH2s parked.
It is for saves, not for data: 256 KB, written back only when the menu runs, and the OS wipes nothing
there, so a game must checksum what it reads (`[sdcard/]` records: magic, length, FNV-1a, ~sum).

## 4. Preloading the PSRAM from the SD card: the design

### 4.0 The simplest form: a launcher file that carries the data (31 Oct 2026)

The EverDrive menu copies a `.32X` from the card into the PSRAM, linearly from the start of the 16 MB,
before it starts it. So the cheapest preload needs no EDIO code at all: **make `EDCD_<GAME>.32X` big**,
with the game's packs behind its first 512 KB at the offsets the game's own CMD 38 / 46 would fill (file
offset *n* × 0x80000 = bank *n* = the launcher's linear window map). The menu wipes the PSRAM first and
loads the file after, so what the file holds is what the game finds, and the ports' content checks (§4.3)
hit without a change to the game or the stack. `edboot/pack_edcd.py` in the BootCamp builds such a file
(`--at offset file`, or `--d32xr plan.json D32XR.32X <disc files>` for DOOM CD32X Fusion's layout).

What stands in the way, in order:

1. **Bank 0 is the launcher**, and the game ROM also goes to cart 0, so the ROM's first 512 KB cannot be
   packed; the loader copies that from the CD as before (BootCamp's always copies the whole ROM; D32XR's
   whole-ROM checksum fails on bank 0 and copies all 4 MB - a 14 MB first boot becomes a 4 MB one; its
   `LOADER_CHECK` per-chunk sums could cut that to 512 KB with a loader change that is D32XR's, not the
   stack's). The `EDCD_W0=<bank>` switch (window 0 shown from another bank) is the other way out; it is
   in docs/12 and has not run on the console.
2. **Unmeasured: does the OS load a multi-megabyte `.32X` in full in its 32X mode?** For `SEGA SSF`
   Mega Drive ROMs that is what extended SSF exists for (up to 16 MB, `mask_off`), but krikzz's public
   source has no 32X mapper to read the rule from, and every `.32X` run so far was tiny. **`edpack/` in
   the BootCamp is the probe**: `edtest`'s 68000 side with a read-only SH2 program, the ROM padded to
   14 MB with a per-bank pattern, each bank read back by the 68000 and both SH2s with `W` off. The screen
   says `LOADED n OF 14 MB` and, past that, whether the PSRAM shows another bank's pattern (a mirror or a
   size mask) or the wipe. No disc, one run. Timing the menu's load with a watch gives the cost per launch.
3. **Stop at 14 MB** (banks 0-27): 31 is the battery RAM; krikzz's layout puts the OS's data buffer, save-
   state buffer and menu at the top of the PSRAM (`0xF00000` up).
4. **Every launch reloads the whole file** (nothing survives the menu), at the OS's own SD-to-PSRAM speed -
   unmeasured; the probe measures it.
5. **OpenLara is the first game to try, and the best case.** Its EverDrive build keeps the Caves record
   (`OLCV`, piece 14) and the level record (`OLLV`) in bank 31, which the EverDrive writes to `bram.srm`,
   so they survive the menu; the record holds length + checksum of the data in the cart. The first launch
   of a packed launcher loads Caves from the CD and writes the record; from the second launch on the
   packed Caves (`LEVEL1.PKD` at `0x080000`) passes the check and nothing loads - the title is up in the
   time the engine takes from the CD (<= 384 KB). The level slot (`LEVEL2.PKD` at `0x200000`, or any
   level named) hits whenever the `OLLV` record names that level. And the whole OpenLara pack ends under
   4 MB, so it works even if the probe says the OS stops at 4 MB. `pack_edcd.py --openlara <disc files>`.
   (The earlier draft of §4.3 said OpenLara would not hit: wrong - its records are in bank 31, not in
   the wiped PSRAM.)

If the probe says the whole file arrives, §4.1-4.5 below are the way to load *more* than fits in one file,
or to load at run time; for a first game they are not needed.

### 4.1 Where to do it: in the launcher, before the Sega CD starts

`EDCD_<GAME>.32X` is a 32X cartridge ROM that the EverDrive menu starts (docs/12). At that moment:

- the ROM is exactly what krikzz's `edio` sample is: a `SEGA SSF` program started from the menu, with the
  `0xA130D0` block live (`[krikzz]` `base_io.sv:34`: not gated by the mapper) and the card mounted;
- the 32X boot ROMs are loading CDLOADER into SDRAM (the cartridge's 32X header points at it) and then
  **wait for `M_OK`/`S_OK` to be cleared** by the 68000 (Evolution2 §manuals item 13), so both SH2s are
  parked by the hardware itself and touch nothing;
- the Sega CD is not yet running: no Sub-CPU, no Word RAM traffic, no listener yet;
- the launcher's own 68000 code can run from work RAM (EDCD already runs its mapper writes and the `w0_move`
  copy from `0xFF8000`);
- the menu has just wiped the PSRAM, so whatever the launcher writes now is what the game will find.

So the preload belongs in **`edboot/edcd_md.s`, between the `M_OK`/`S_OK` wait and the mapper set-up**:

```
EDCD (cartridge 68000, from work RAM):
  1. wait M_OK / S_OK                              (as now; the SH2s are held after this)
  2. RV = 1; CTRL0 = 0x8000 (P, W off), CTRL1-7 = 1-7  (as now)
  3. NEW: ed_init (FIFO flush, CMD_F_DIR_LD "/")
          for each file in the launcher's preload list:
              CMD_F_FOPN  path                       e.g. MEGA/gamedata/EDCD_<GAME>/cd/LEVEL00.BIN
              CMD_F_FRD_MEM fci_addr, length         fci_addr = the file's place in the 16 MB
              halt loop (SYSSTAT strobe / mcu_busy)  from work RAM, RV = 1 held
              CMD_F_FCLOSE
          optionally CMD_F_FCRC / CMD_MEM_CRC to verify, and write a 16-byte record per file
          (magic, length, sum) into a tag area the game reads (see 4.3)
  4. RV = 0; the BIOS RAM jump table, ED01..ED04, the Sub-CPU, the hand-over   (as now)
```

Nothing in the boot stack changes: `stack/` stays md5-identical, the disc is the same disc, the Sub-CPU
program is the same `MODE1=1` build. The launcher grows by the `edio` client (about 2 KB of 68000 code,
MIT) and a table of file names and destinations. A launcher without a card file, or a card without the
folder, falls through to step 4 and the game loads from the CD exactly as today.

### 4.2 Where the files go: the game's own cart layout, by FCI address

`CMD_F_FRD_MEM` takes an **FCI address**, which is the physical place in the 16 MB (`0x0000000` = bank 0
offset 0, bank *n* = `n × 0x80000`; banks 16-30 continue at `0x0800000`; `0x1080000` = bank 31), not a
68000 address and not a window: the mapper's `CTRL` registers do not matter to it (`[krikzz]` `pi_map.sv`).
So the launcher writes each file to the bank and offset where the game's own CMD 38 / 46 would have put
it:

| Game asks for | Cart address on a 4 MB cart | FCI address (EDCD's linear start: window n = bank n) |
|---|---|---|
| the ROM, file 20 | `0x000000` | `0x0000000` |
| the resident slot (BootCamp `GAME_RESIDENT_SLOT` 4, OpenJazz `IMAGE8.RAW` `0x100000`, Hurrican `0x100000`, SuperTux `0x120000`, Rayman `0x080000`) | slot × 128 KB | the same number |
| the level slot (`0x200000`, OpenJazz / Hurrican / SuperTux `0x280000`) | `0x200000` | the same, **or** page *p*'s copy at bank `4p` × `0x80000` + (offset − `0x200000`) for the paged games' pages 1-6 |

For a paged game (`CART=ED`: pages 1-6 = banks 4-27 in window 1), a whole world cache can be put in
place at once: page *p*'s `0x280000-0x3EFFFF` is FCI `p × 0x200000 + 0x080000 ...` (bank `4p + 1`
onward). Page 7 (banks 28-31) is never written: bank 31 is the battery RAM (`[edsave]`), and banks 28-30
hold the OS's own things (docs/10; krikzz: the data buffer at `0xF00000`, save state at `0xF40000`, menu
at `0xF80000` of the PSRAM, `everdrive.h:98-101`).

### 4.3 How the game finds out: the checks it already has

Every port already asks "is it in the cart?" before loading from the CD, and on the EverDrive those checks
simply never hit (docs/12). A preload makes them hit again, with **no game change**, when the check is
content-based:

| Game | Its check | Hits after a preload? |
|---|---|---|
| OpenLara REV | Caves record `OLCV` and level record `OLLV` in save piece 14 (length + checksum of the data in the cart), RESUME-1 | **yes, from the second launch**: on an EverDrive piece 14 is in bank 31 (EDSAVE-1), kept in `bram.srm`, so the record the first launch writes after its CD load is still there, and the packed data passes its checksum (§4.0 item 5) |
| OpenJazz REV, Hurrican | quick check of the ROM (CMD 47 three chunks) and of the common / resident pack (length + sum) | **yes** for the pack; the CDLOADER quick check is a chunk compare against the disc, so it hits too |
| SuperTux REV | area A kept; 16 MB: `SUMS.BIN` sums per pack at power-on | **yes** |
| Sonic CD 2011 | `scd_cart_sum` of the engine and the title against the pack's `load_sum` table (R4I) | yes for what the launcher put in; its act files on page 7 cannot be preloaded (page 7 is the OS's on an EverDrive) |
| D32XR | checksums of the ROM in pages 0 + 7 and of every WAD in pages 2-6, record in save piece 15 | the WADs yes; its page 7 half of the ROM cannot (same reason), so D32XR is out until its ED layout avoids page 7 |
| BootCamp hello | none: `cd_load` always loads | needs the record pattern (CAVES-1's) in `cdapi.c` first |

A content check is the only safe one: `[edsave]` says the PSRAM comes up wiped, and a wipe is not
zeros - so a magic word alone is not proof, and a record that holds length + checksum of the data is.

### 4.4 What must be measured before any of this is built (the console questions)

Everything above comes from krikzz's source and the 68000 side of the EverDrive. The 32X adds questions
the source cannot answer, in the order a probe should take them (a no-disc `.32X`, like `edtest`):

1. **Does the `0xA130D0` block answer the 68000 with the 32X on, and only with RV = 1?** `[edpage]` and
   docs/12 showed the `CTRL` writes need RV = 1 then; reads of `FIFOSTAT` / `SYSSTAT` have never been
   tried through the adapter. If reads of `0xA130xx` do not come back with RV = 1, the halt loop cannot
   see the strobe, and the preload can only be done **before** the 32X is turned on - which the launcher
   cannot do (the boot ROMs are already running when it starts). Probe: `SYSSTAT` read twice (the strobe
   must toggle), `TIMER` read 10 ms apart, with RV = 1 and with RV = 0.
2. **What do the SH2s see while the MCU owns the PSRAM?** The 68000 is kept off the cart by the halt
   loop in work RAM; the SH2s are held by `M_OK`/`S_OK` only while the launcher has not cleared them.
   For a preload *after* the hand-over (section 5) the SH2s would have to park in SDRAM as for an RV copy,
   and whether a stray SH2 fetch during MCU DMA stalls, returns garbage or disturbs the transfer is
   unknown. `[edsave3]`'s pattern (slave hammering a window from SDRAM code) is the probe.
3. **Does `CMD_F_FRD_MEM` land where the SSF bank arithmetic says?** Fill a file with a bank pattern,
   load it to FCI `n × 0x80000`, map bank *n* into window 1 and read it back with the 68000 (RV = 1) and
   both SH2s (cache-through), as `edtest` does.
4. **Throughput.** Time `CMD_F_FRD_MEM` for 1 MB with `TIMER`. If it is anywhere near MegaColor's
   1.4 MB/s, a 2.5 MB resident pack is under 2 s instead of 20-60 s from the CD.
5. **Does the card path survive the Sega CD coming up?** After step 3 of §4.1 the launcher resets the
   Sub-CPU and unpacks the BIOS; the MCU is idle by then (every transfer is finished before RV is dropped),
   so nothing should change - but the `RA` entry's bank-31 write-back by the OS at the next menu run must
   still happen with the preload code in the ROM. `[edsave]`'s markers check it.
6. **Does `W` matter for the MCU's writes?** The RTL says no (`mem_ctrl` puts the DMA bus in front of the
   mapper), so the launcher can leave `W` off throughout, which `[edw0]` requires. Confirm with `W` off.
7. **Firmware.** `ed_init` needs the card mounted and `CMD_F_*` present; `CMD_STATUS2` and
   `CMD_ROM_PATH` need firmware v24+ (`[krikzz]` `main.c:415`). The launcher must treat any non-zero
   status as "no preload" and go on to the CD, never stop.

### 4.5 What the user puts on the card

A folder per launcher, next to where the OS already keeps its save:
`MEGA/gamedata/EDCD_<GAME>/cd/` with the disc's data files under their ISO names (`GAME.32X`,
`LEVEL00.BIN`, `IMAGE8.RAW`, `OJ012.PAK` ...). The launcher's table names each file, its FCI
destination and its expected length; a missing or short file is skipped (the game then loads that one
from the CD as now). The disc stays the master copy: music is on it, the Sub-CPU program is on it, and a
card without the folder changes nothing.

## 5. The other designs, and why they come later

| Design | What it would give | What stands in its way |
|---|---|---|
| **A Sub-CPU command "file from the card → cart"** (a new CMD on the stack: the Sub-CPU hands the 68000 listener a path and a destination, the listener runs the EDIO client from work RAM with RV = 1) | loads at run time from the card: levels, movies, without stopping the music | the listener is the boot stack (frozen, md5-guarded); the halt loop holds RV = 1 for the whole transfer with the 68000's interrupts off (seconds, not the 5 ms of a MODE1 block: no vblank, no pad, no Sub-CPU level-2 tick meanwhile) and both SH2s parked in SDRAM (EDSAVE-4) for as long; question 2 of §4.4 is open. Numbers for it are free (76 and up: see the ledger §3); it belongs behind an `HWFLAGS` switch in a copy, like `sdcard/` |
| **The whole game from the card** (the launcher loads the Sub-CPU program and the ROM too; the disc only for CDDA) | a start in seconds | the Sub-CPU program is already in the launcher (`app_mode1.bin`); the ROM preload is §4; the disc is still needed for music and for the Sega CD BIOS to boot the Sub-CPU - so this is §4 plus the `IMAGE.RAW` splash from the card, not a new design |
| **`MEMADDR`/`MEMDATA` as a write path** for the listener (byte-serial, any address, no `W`, no window, no page) | would remove the window-0 and `W` rules from `m1_xfer` | one byte per two bus cycles at most; slower than the word copy, and untested through the 32X. A probe, not a plan |
| **Saves larger than 256 KB, or a save file per slot**, through `CMD_F_FWR` from the launcher or a command | bigger and faster saves than bank 31 | the same run-time questions as the first row; the bank-31 route works today |

## 6. What this changes in the documents

- The command ledger (`CD32X_COMMAND_LEDGER_ADDITIONS.md` §16) now lists every EverDrive save command per
  game and the numbers free for a card command.
- Nothing in `CD32X-BootCamp` changes until §4.4's probe has run. The probe is the next piece of work;
  `edtest/` is the template (its 68000 side already runs with interrupts off and RV = 1).
- When a console answers one of §4.4's questions, write it into §1.1's table with the date and the probe,
  as docs/11 and docs/12 do, and move the row out of §4.4.
