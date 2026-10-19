# CD32X – Proposal: Sub-CPU commands for the 8 MB / 16 MB SRAM cart

**Status: proposal, not built, not tested.** The cart page register must first be proven on the
console (the test being worked out with the SonicCD2011 project). Nothing here goes into a
disc until that test passes. Numbers 68 and 69 are proposed; register them in
`CD32X_COMMAND_LEDGER_ADDITIONS.md` only after they've run on hardware.

Hardware: MicroNut99/16MB-PCB-CD32X rev 0.5 (README §2 "Memory organisation and the mapper").
First user: OpenLara CD32X v2.0 (`OpenLaraCD32X/docs/SRAM_8MB_16MB_DESIGN.md`, branch
`v2.0-sram-design`).

## 1. The problem
- The 8/16 MB cart shows 2 MB pages through two 2 MB windows. Window 1 (cart 2–4 MB) picks its
  page from a register at **`0xA130F2`** (even byte, page 0–7 in D8–D10, write-only). Window 0's
  register is at `0xA130F0`.
- The register is in the 68000's /TIME space. **The SH2s can't reach it**, and no existing SH2 → Sub-CPU
  command writes it (checked: OpenLara, OpenJazzCD32X, OpenJazzCD32X-REV).
- **Without a new command the cart works as a 4 MB cart** (reset state: pages 0 and 1). That's
  safe, but the extra memory goes unused.

## 2. What stays untouched
Byte for byte:
- the boot block;
- `sh2/` (CDLOADER.BIN);
- the game's `crt0.s` and `src-md/`;
- **the 68000 listener `hw_md.s`**: no new listener routine, no new MD_CMD number.

The change is limited to **new branches in the Sub-CPU command handler (`subcpu/main.c`)**:
- They're built only from listener commands that exist in every stack.
- **Nothing new is sent during boot.**
- CMD 38's boot branch (file 20 = the game) and CMD 46 are unchanged.
- APP.BIN changes, so `build.sh`'s "APP.BIN identical" check will report a difference; that's
  expected for a game that uses these commands.

Listener commands used (all existing):

| MD_CMD | Routine | Used for | Already used by |
|---|---|---|---|
| 33 | `chilly_rv_memcpy` | RV = 1, copy, RV = 0: one byte to `0xA130F2`, test markers into the cart | CMD 46, CMD 61 |
| 20 | `write_32x_16` | `0xA15104` bank select for the read-back | CMD 46, CMD 61 |
| 19 | `read_32x_16` | reading the cart back through `0x900000` | CMD 46 |
| 32 | `chilly_memcpy` | block read-back | CMD 46, CMD 61 |

The 32X manual asks for RV = 1 before a bank-register access. Listener 33 does exactly that, the
same way every cart load already does.

## 3. The commands

### CMD 68 – cart page (window 1)
| In | Reply |
|---|---|
| COMM2 = page 1–7 | COMM12 = the page, or 0 = refused |

```c
else if (cmd == 68) {
    int page = do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122) & 0xFFFF;
    int ok = page >= 1 && page <= cart_max_page;      /* never page 0: program + resident data */
    if (ok) {
        volatile uint8_t *wr = (volatile uint8_t *)0x0C0000;
        wr[0] = (uint8_t)page;
        switch_banks();                                /* Word RAM -> 68000 */
        do_md_cmd3(MD_CMD_CHILLY_RV_MEMCPY, 0xA130F2, 0x200000, 1);  /* even byte, /UWR */
        switch_banks();
    }
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA1512C, ok ? page : 0);
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA15120, 0);
}
```

- **`cart_max_page`:** 7 by default, 3 once CMD 69 has found 8 MB.
- **Window 0 is never switched.** The game runs from it.
- **The SH2 side:** before CMD 68, finish drawing and have the second SH2 idle. After it, both
  SH2s clear their caches (CacheClear; the slave via its cache command). With RV = 1 an SH2 cart
  access just waits, so the only risk is stale cached data.

### CMD 69 – cart size
| In | Reply |
|---|---|
| – | COMM12 = 4, 8 or 16 |

M = window 1 + 0x1F02F0: in the last 64 KB of the page, the game's own bytes (on page 1 that's
cart 0x3F02F0, inside the game's save-area piece). Writes go through listener 33; reads use
listener 20 to select the bank, then 19 through `0x900000`.

1. Page 1: write A at M.
2. Page 2: write B at M, then ~B at M + 2. The second write stops a floating bus from echoing B.
3. Page 1: read M. **B → 4 MB**: there's no mapper, so the page write did nothing.
4. Page 6: write C at M, ~C at M + 2, read M. **Not C → 8 MB** (16 MB image, no U2).
5. Page 2: read M. **C → 8 MB** (8 MB image: page 6 mirrors page 2).
6. **Otherwise 16 MB.**
7. Window 1 back to page 1.

### CMD 38 – offset for a game's own files (optional)
For the game's own level/data branches only, never the boot branch (file 20): COMM8L = offset in
window 1 where this file goes (0 = the slot start). It lets two files share one page, for example a
level and its cut-scene. The branch refuses a file whose offset + length passes 0x1F0000 (the
start of the save area / page tail).

## 4. Rules for games that use paging
1. **Saves stay on page 1** (cart 0x3F0000, the ledger's pieces). CMD 61 writes window 1 +
   0x3F0000, so window 1 must be on **page 1** for every save-area read *and* write. Switch, save,
   switch back.
2. **Pages 2–7 belong to the game that wrote them.** The 4 MB-era games never touch them, so data
   there survives playing other games. A game must still check what it finds: a tag with its own
   identity plus a checksum, as OpenLara's Caves record and Jazz's start-up check do.
3. **The last 64 KB of every page is reserved** (save area on page 1; tags and the size marker on
   2–7).
4. **Leave window 1 on page 1 before a reset or another game.** A console reset puts both
   windows back to pages 0/1 anyway.

## 5. The test first
The order:
1. A bring-up disc with CMD 68/69 and a test screen:
   - the size found;
   - each page's tail written and read back, through CMD 61 and the SH2's window 1;
   - on 8 MB, the mirror.
2. If the page register can't be written this way with the 32X on (PCB README §12: /TIME writes
   with ADEN = 1), the alternative to try is the same byte write before the 32X adapter is enabled.
   That would be a boot-time step, so it would need its own decision.
3. Only then OpenLara v2.0.
