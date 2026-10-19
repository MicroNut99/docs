# CD32X – Paging proposal, variant B: command 68 with no listener change

**Status: proposal, not built, not tested.** A response to
[CD32X_CART_PAGING_PROPOSAL.md](CD32X_CART_PAGING_PROPOSAL.md) (variant A, from the SuperTux
REV session), written from the OpenLara CD32X v2.0 design (`OpenLaraCD32X/docs/SRAM_8MB_16MB_DESIGN.md`,
branch `v2.0-sram-design`). Both variants agree on the problem, on SH2 command 68, and on testing
first on real hardware (the test being worked out with the SonicCD2011 project). The difference
is how much of the stack changes.

## 1. The difference in one line
Variant A adds a 68000 listener routine (`set_cart_page`, MD_CMD 38). **Variant B leaves
`hw_md.s` byte for byte** and writes the page register with the listener's existing
`chilly_rv_memcpy` (MD_CMD 33), the routine every cart load and save already uses.

Why it matters:
- **The rule:** the boot stack stays unedited. With B the 68000 listener is identical in every
  game; only `subcpu/main.c`'s command handler gets one branch.
- **MD_CMD 38 is already taken:** OpenJazzCD32X-REV's `hw_md.s` uses listener numbers 37–40 for
  its Genesis plane (`genbg_load`, `genbg_start`, `genbg_off`, `genbg_cell`), and SuperTux uses 37
  for `copy_vram_col`. A new listener number would add to that clash. B needs none.

## 2. Command 68 – window 1 page (variant B)
Same interface as variant A: COMM2 = page 1–7, reply COMM12 = 1 done / 0 refused.

```c
else if (cmd == 68) {
    int page = do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122) & 0xFFFF;
    int ok = page >= 1 && page <= 7;                   /* never window 0 / page 0 */
    if (ok) {
        volatile uint8_t *wr = (volatile uint8_t *)0x0C0000;
        wr[0] = (uint8_t)page;                         /* the byte, in Word RAM */
        switch_banks();                                /* Word RAM -> 68000 (0x200000) */
        do_md_cmd3(MD_CMD_CHILLY_RV_MEMCPY, 0xA130F2, 0x200000, 1);
        switch_banks();
    }
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA1512C, ok);
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA15120, 0);
}
```

- **What the 68000 does:** `chilly_rv_memcpy(dst, src, len)` sets RV, copies `len` bytes with
  `move.b d1,(a1)+`, clears RV. With dst = 0xA130F2 (even) and len = 1 that's one even-byte write:
  /UWR with the page on D8–D10. That's exactly what the board latches, the same bus cycle as
  variant A's `move.b d0,(a0)`.
- **Same Word RAM handover** as CMD 46 and CMD 61 (`switch_banks()` around the 68000's copy).
- **Not used during boot.** CMD 38, CMD 46 and the boot branch of CMD 38 (file 20) are unchanged.
  APP.BIN changes by this branch only.

## 3. Things variant A gets right and B takes over
- **Size probe from the SH2 with 68 + 58/61**, no second command. One addition: after writing a
  marker, write its inverse at the next word. On an 8 MB board with the 16 MB GreenPAK image,
  pages 4–7 read a floating data bus, which can still hold the last value written (bus
  capacitance). Writing the inverse second means a floating bus can't return the marker.
- **Existing commands do the rest:**
  - 46 loads into whatever page window 1 shows;
  - 58/61 write that page's tail at window 1 + 0x3F0000;
  - 47 compares.
- **Test order:** the 4 MB cart first (it must report 4 MB and change nothing), then the 8 MB
  board, then the 16 MB board.

## 4. Rules for games (both variants)
1. **Window 0 never changes:** the program runs from it.
2. **Saves stay on page 1** (cart 0x3F0000). Every save-area read *and* write needs window 1 on
   page 1: switch, save, switch back. That includes reads, because the SH2 reads saves at
   0x223F0000 directly.
3. **Before 68:** finish drawing and have the second SH2 idle. **After 68:** clear both SH2s'
   caches.
4. **Pages 2–7 aren't touched by the 4 MB-era games**, so data kept there survives playing them.
   A game still checks it before trusting it: its own identity tag plus a checksum, as OpenLara's
   Caves record and Jazz's start-up check do.
5. **The last 64 KB of every page is reserved:** the saves on page 1, tags and markers on 2–7.

## 5. Open, for the test
- **Does the /TIME write reach the board with the 32X on (ADEN = 1, RV = 1)?** (PCB README §12)
  Both variants do the same bus cycle, so one test answers both.
- **Does the 4 MB cart ignore the write to 0xA130F2?**
- **The ledger:** once tested, register SH2 command 68. With variant B there's no listener number
  to register.
