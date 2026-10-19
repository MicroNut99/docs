# CD32X – proposed boot-stack addition for the 16MB-PCB-CD32X (8 / 16 MB SRAM cart)

**Status: PROPOSAL - not implemented, not tested.** Written by the SuperTux CD32X REV session
(Claude) for everyone working on the CD32X stack to read and check. Nothing here may go into a
boot stack until it has been agreed and tested on real hardware. The 16MB-PCB-CD32X board
(MicroNut99/16MB-PCB-CD32X, rev 0.5) is itself not yet built or tested.

Applies to the OpenLara / Hurrican CD32X boot path (Sub-CPU `main.c`, 68000 listener `hw_md.s`)
as used by OpenJazz CD32X and SuperTux CD32X REV. Command numbers follow
[CD32X_COMMAND_LEDGER_ADDITIONS.md](CD32X_COMMAND_LEDGER_ADDITIONS.md): SH2 commands 62-67 and
68000 routines 36-37 are taken (SuperTux REV2-REV4); new numbers start at 68; OpenTyrian's
cartridge stack uses 60-63 for other things.



## Why a change is needed at all
The 16MB-PCB-CD32X shows more than 4 MB only when the **68000** writes a page number to the
board's page register (window 1: **0xA130F2**, even byte, page in D8-D10; README §2). The SH2s
cannot reach that address. Every existing command in `cd32x/subcpu/main.c` writes only fixed
32X registers (0xA151xx) or, for 46 / 61, inside the 4 MB cart window - none can write
0xA130F2. So one new command is the minimum. Everything else stays existing commands.

## The change (smallest form)

### A. SH2 → Sub-CPU command 68: "cart window 1 page"
- COMM2 = page, 1-7. Reply COMM12 = 1 done / 0 refused (page out of range). Done = COMM0 cleared.
- Window 0 is never paged (the SH2 program runs from it). Only window 1 (cart 0x200000-0x3FFFFF).
- Sub-CPU handler (same shape as REV2's 66):
```c
else if (cmd == 68) {
    int page = do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122) & 0xFFFF;
    int ok = page >= 1 && page <= 7;
    if (ok) do_md_cmd2(MD_CMD_SET_CART_PAGE, 0xA130F2, page);
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA1512C, ok);
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA15120, 0);
}
```

### B. 68000 listener routine 38: `set_cart_page(addr, page)`
```
| SuperTux: void set_cart_page(int addr, int page) - the 16MB-PCB-CD32X page register.
| Even-byte write (the board latches /UWR, D8-D10), with RV=1 as the board README asks and as
| chilly_rv_memcpy (CMD 46) already does. Runs in the command loop, never in an interrupt.
set_cart_page:
        movea.l 4(sp),a0
        move.l  8(sp),d0
        bset    #0,0xA15107             /* RV on */
        move.b  d0,(a0)                 /* even address: upper byte lane = D8-D15 */
        bclr    #0,0xA15107             /* RV off */
        moveq   #0,d0
        rts
```
`MD_CMD_END` becomes 39. About 24 bytes added to the listener.

### C. Bookkeeping
- Projects that check their boot path byte for byte (SuperTux REV: `tools/check_boot.py`) add
  both blocks as exact text, as for 62-67.
- Docs repo ledger: SH2 command 68, 68000 routine 38. (68 is free on this stack and outside
  OpenTyrian's 60-63; the ledger asks new numbers to be 68 and up.)

## What existing commands then do (no change to them)
- **46 (chunk → cart)** writes through window 1 to whatever page it shows: level packs go to
  pages 2-7 by setting the page first. Its read-back through 0x900000 sees the same page.
- **58 + 61 (save area)** write 0x3F0000 + n × 4 KB in window 1: page 1 for the saves - and on
  any other page, a 4 KB scratch piece (used by the probe below).
- **47 (compare)** checks a page's content against the disc, as Jazz's quick start does.
- **SH2 side** (game code, not boot stack): after each 68, purge the SH2 cache for the cart
  (`0x40000000` purge area or CCR purge, as CMD 61's `0xFFFFFE92 = 0x11` already does).

## Cart size without another command
With 68 + 58/61 the SH2 can probe: write marker A to page 1 piece 15, set page 2 and write B
there, set page 6 and write C; read each page back. Page 2 shows A = no paging (4 MB cart);
page 6 shows B (8 MB image mirrors pages 4-7) or junk / unwritable = 8 MB; all distinct =
16 MB. Restore page 1 last. (Piece 15 is free in the save-piece table; it is only scratch.)

## Risks to check
1. Board README §12: does the /TIME write reach the board with the 32X on (ADEN=1)? RV=1
   is the recommended way - untested.
2. The **4 MB RAM cart** (no paging): does it ignore an even-byte write to 0xA130F2? A plain
   SRAM cart should not decode /TIME, but it must be tested first on the 4 MB cart, before
   any game uses 68.
3. RV=1 locks the SH2s out of the cart for a few cycles - as in CMD 46 / 61 today: the SH2
   that sent the command only polls COMM0; the other SH2 must not touch the cart then (the
   game sends 68 only on the loading screen, as it sends 46).
4. The cache purge after a page change (SH2 code, testable on the PC model only as logic).

## Test disc (real hardware, before anything else)
Example (SuperTux): the normal REV10b disc plus *Options → Cart test* (game code only):
1. Probe the cart size (above) and show 4 / 8 / 16 MB.
2. For each page 1-7: write a 4 KB pattern to piece 15 (58/61), read back through the SH2,
   show OK / FAIL per page; then page 1 again and check the saves are untouched.
3. With the green diagnostics on, each 68 shows on screen as the other commands do.
Order: 4 MB cart first (must say 4 MB and change nothing), then the 8 MB board, then 16 MB.

## Questions for the reviewers
- Is RV=1 the right condition for the /TIME write, or should the 68000 also stop the 32X
  (ADEN) / use another sequence the board needs?
- Should command 68 also allow window 0 (some games might not run code from it)?
- Is piece 15 of the save area acceptable as the probe's scratch piece for every game?
