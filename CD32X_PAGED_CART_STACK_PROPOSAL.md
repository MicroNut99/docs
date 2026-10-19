# CD32X – Boot stack changes for the paged 8 / 16 MB cart (proposal)

**Status: proposal, untested.** The numbers below are *proposed*. They are not registered in
[CD32X_COMMAND_LEDGER_ADDITIONS.md](CD32X_COMMAND_LEDGER_ADDITIONS.md) and nobody should use
them until the page register test on the first
[16MB-PCB-CD32X](https://github.com/MicroNut99/16MB-PCB-CD32X) board passes. Then they go into
the ledger.

Stack: the OpenLara / Hurrican CD32X boot path (Sub-CPU `main.c`, 68000 listener `hw_md.s`),
as in the ledger.

Game-side designs that use this:
- [OpenJazzCD32X-REV `docs/CART_8MB_16MB.md`](https://github.com/MicroNut99/OpenJazzCD32X-REV/blob/main/docs/CART_8MB_16MB.md)
  (the reference)
- [HurricanCD32X `docs/CART_8MB_16MB.md`](https://github.com/MicroNut99/HurricanCD32X/blob/main/docs/CART_8MB_16MB.md)

## 1. The board, as software sees it

From the PCB README:

- 16 MB (or 8 MB with U1 only) in 2 MB **pages**, seen through two 2 MB **windows**: cart
  0x000000-0x1FFFFF is window 0, and 0x200000-0x3FFFFF is window 1.
- **Page registers:** `0xA130F0` holds window 0's page and `0xA130F2` holds window 1's page.
  They are the even byte, clocked by /UWR, in the /TIME area, and write-only.
- **Reset state:** W0 = 0, W1 = 1. The first 4 MB look exactly like the 4 MB cart.
- **8 MB build:** pages 4-7 mirror pages 0-3.
- **Who can switch pages:** only the 68000. The SH2s cannot reach /TIME, and the Sub-CPU
  cannot reach the cart. **So every page switch goes SH2 → Sub-CPU → 68000 listener.**

## 2. Why it needs no change to the boot sequence itself

- **Power-on and /RESET** set W0 = 0, W1 = 1. The boot, the disc identity check, the engine copy
  and the common pack load all see a plain 4 MB cart, unchanged.
- **The 68000 listener runs from work RAM** (0xFF1000), not the cart. A page switch can never
  pull its own code away.
- **Titles that never send the new command** run as today. They only ever see pages 0 and 1.
- **The shared save area** (`0x3F0000`, the ledger's 16 pieces) is page 1, offset 0x1F0000. It
  stays where every title expects it, as long as titles map W1 = 1 before they touch it (rule 3).

## 3. Proposed additions

### SH2 → Sub-CPU

| CMD | Name | COMM2 | COMM8L | Reply |
|---|---|---|---|---|
| **68** | cart page | page 0-7 for window 1; bit 15 = probe | – | COMM12 = page now in window 1; probe: pages found (2 = 4 MB, 4 = 8 MB, 8 = 16 MB) |
| **38** (extended) | open file | as today, plus bit 15 = "destination given" | cart destination offset (4-byte aligned, < 4 MB) | as today |

**CMD 38 / 46 today** put a file at a fixed 128 KB slot chosen by its file id (`cur_img`).
With bit 15 set, CMD 38 takes the destination from COMM8L, and CMD 46 writes chunk n at
destination + n × 32 KB. Without bit 15 nothing changes. This is needed because cached packs
go at arbitrary offsets inside a page.

### Sub-CPU → 68000 listener

| MD_CMD | Name | Arguments | What it does |
|---|---|---|---|
| **42** | `cart_page` | page, probe flag | interrupts off, RV = 1 (bit 0 of `0xA15107`), `move.b d0,0xA130F2`, RV back, interrupts back. Keeps the page in work RAM (the register cannot be read). With the probe flag: runs the probe (section 4) and leaves W1 = 1 |

**Why RV = 1:** the PCB README's open point notes that with the 32X adapter on, /LWR is not
pulsed for /TIME writes (Charles MacDonald). The registers use /UWR, but the safe way is the
one that already writes the cart on real hardware: RV = 1 around the write, as
`safe_cart_xfer` (Kobo) and `chilly_rv_memcpy` do. While RV = 1 the SH2s are locked out of the
cart:
- the master waits for CMD 68's reply in SDRAM code;
- the slave must be idle in its cached loop;
- both purge their cache afterwards.

Listeners whose `MD_CMD_END` is below 42 fill the gap with `cmd_done` entries.

### Changes to existing commands

| CMD | Change |
|---|---|
| **61** (save piece → cart) | 68000 side only: sets W1 = 1 around its write and read-back, then restores the page it found |
| **60** (start D32XR from the cart) | sets W1 = 1 before it hands over, so the next program starts with the reset mapping. The same goes for any other exit that is not a reset |

### Save area: piece 14 = the page directory (shared by every title)

Proposed: **piece 14** (`0x3FE000`, free in the ledger) holds the directory of which pack of which
title is in which page. Titles that cache packs use it, so a pack loaded once is found again
after power-off, and titles evict each other's packs by least-recent use instead of
overwriting blindly. Its format is in the Jazz design (section 4).

## 4. The probe (68000, once at boot)

The probe overwrites nothing for good: every word it uses is saved first and put back.

1. With W1 = p, read the probe word (cart `0x3FEFFE`, the last word of piece 14) of pages 0-7.
   Keep the 8 values.
2. Write `0xC0DE + p` into page p, for p = 0 ... 7 in order.
3. Read the 8 words back:

   | What comes back | Board |
   |---|---|
   | all 8 hold their own marker | **16 MB** |
   | pages 4-7 hold their own markers; pages 0-3 hold `0xC0DE + p + 4` | **8 MB** |
   | every page reads `0xC0DE + 7` (the register write never reached the cart) | **4 MB** |
   | anything else | treat as **4 MB**, warn |

4. Write the saved values back in reverse order. Leave W1 = 1.

## 5. Rules for titles that use paging

1. **W0 is never changed.** The engine runs from page 0.
2. **Only W1 changes.** After a switch, both SH2s purge their cache before they read the cart.
3. **The save area only with W1 = 1.** Every reader and writer of `0x3F0000`-`0x3FFFFF` maps
   page 1 first: SH2 reads at `0x223F0000` and CMD 61.
4. **Before leaving without a reset** (CMD 60, a return to the BIOS), set W1 = 1.
5. **Do not trust pages 0 and 1** after another title ran: a title that does not page may have
   overwritten them. Check a checksum. Pages 2-7 are written only by titles that keep the
   directory.

## 6. To test first (before any number here is registered)

1. **The page register write through the 32X:** with the Sega CD and 32X attached and
   ADEN = 1, write `0xA130F2` with RV = 1 and read a marker back from window 1. Also try:
   - the write before the adapter is enabled;
   - Sega's odd-byte form (`0xA130F3`).
2. **/CAS2 at cart pin B21** covers /LWR and /UWR on a cart write in Sega CD + 32X mode
   (scope).
3. **Writes into a mapped page** with CMD 46, read-back on.
4. **The SH2s** read the new page at `0x02200000` after the purge.
5. **Save piece write and read** with an extended page mapped before and after (W1 = 1
   around them).
6. **A 4 MB cart:** the probe reports 4 and nothing else changes.

## 7. A numbering clash to fix (not caused by this proposal)

**The clash:** OpenJazzCD32X-REV R1-R4 use **CMD 67** (Genesis plane) and **68000 routines
37-40** (`genbg_load` / `start` / `off` / `cell`). The ledger already gives CMD 67 = cart → VRAM
columns and MD 37 = `copy_vram_col` (SuperTux REV4).

**Proposed fix:**
- Jazz moves to **CMD 69** and **MD 38-41**, and registers them.
- `cart_page` then takes MD 42, and `MD_CMD_END` becomes 43.
