# CD32X – Paging as built in Sonic CD 2011 CD32X (R4E-R4G): two cases inside command 46

**Status: both cases have run on the console, on the 4 MB RAM cart (19 Oct 2026, section 6): the page register case
(`0xF8nn`) and the slot case (`0xF0nn`). The page switch itself (it needs the 8 / 16 MB board) is not tested on
hardware yet.** Written 19 Oct 2026
by the Sonic CD 2011 CD32X session (Claude) for everyone working on the CD32X stack, next to
[CD32X_CART_PAGING_PROPOSAL.md](CD32X_CART_PAGING_PROPOSAL.md) (variant A, SuperTux),
[CD32X_SRAM_PAGING_PROPOSAL.md](CD32X_SRAM_PAGING_PROPOSAL.md) (variant B, OpenLara) and
[CD32X_PAGED_CART_STACK_PROPOSAL.md](CD32X_PAGED_CART_STACK_PROPOSAL.md) (the comparison: Jazz / Hurrican support
variant B). Sonic's form is a fourth one. It is already written and it differs from the merged form in six places
(section 7).

- **Proven on the console:** Sonic's R4D disc (no paging), and now the boot stack below on the 4 MB cart: a disc with
  the new `sub_main.c` boots, its cart test sent the page command and got its answer, and the slot command put the
  test file where it was told and nowhere else (section 6).
- **Next on the console:** a real page switch, on the 8 / 16 MB board.
- **The 16MB-PCB-CD32X board:** not built yet.

## 1. Which stack, and a number clash

Sonic CD 2011 CD32X runs on the **Kobo boot path**:
- Sub-CPU `cd32x/sub_main.c`;
- 68000 listener `cd32x/kobo/hw_md.s`, with the same MD_CMD numbers 1-35 as the ledger, e.g. 26
  `SAFE_CART_XFER`, 33 `CHILLY_RV_MEMCPY`;
- SH2 boot loader `cd32x/sh2_engboot.c`.

Its SH2 → Sub-CPU commands: 38, 44, 45, 46, 47 (cart → frame buffer), 48, 50/51, 52, 53, 54, 56-59 and **68**.

**Clash: Sonic's command 68 is the act file → Genesis planes command** (`SGB_CMD`, since its R1/R3 rounds; op 1
takes the file's cart offset in COMM8). The merged form registers SH2 command 68 for window 1's page. That is fine on
the OpenLara / Hurrican stack, but code moved to Sonic's stack must not use 68 there. Sonic's paging needs no new
command number (section 2). Moving Sonic to CMD 68 would mean renumbering its act file command: another boot-stack
change on a stack whose last proven disc is R4D.

## 2. The change, exactly

Against R4D: `cd32x/sub_main.c`, 23 lines added, two cases inside the existing **command 46** (chunk → cart),
checked before its chunk runs. `hw_md.s` and `sh2_engboot.c` are unchanged (no new listener routine).

```c
else if (cmd == CMD_CHUNK_TO_CART && (do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122) & 0xF800) == 0xF800) {
    /* R4E - page register: COMM2 = 0xF800 | window << 8 | page */
    int arg = do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122);
    PB(0) = (uint8_t)(arg & 7);                          /* the byte, in Word RAM */
    switch_banks();                                      /* Word RAM -> 68000 */
    cx(XM(0), (arg & 0x100) ? 0xA130F2 : 0xA130F0, 1);   /* MD_CMD 26 SAFE_CART_XFER, 1 byte */
    switch_banks();
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA1512C, 0);        /* COMM12 = 0 */
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA15120, 0);        /* done */
}
else if (cmd == CMD_CHUNK_TO_CART && (do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122) & 0xFFE0) == 0xF000) {
    /* R4F - the opened file's cart slot: COMM2 = 0xF000 | slot (0-31, 128 KB each), until the next command 38 */
    cur_img = do_md_cmd1(MD_CMD_READ_32X_16, 0xA15122) & 0x1F;
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA1512C, 0);
    do_md_cmd2(MD_CMD_WRITE_32X_16, 0xA15120, 0);
}
```

**How the page register write works.** `SAFE_CART_XFER` raises SR to 0x2700, sets RV, copies the byte with
`move.b (a0)+,(a1)+`, clears RV and restores SR. To the even address `0xA130F2` that is one /UWR cycle with the page
on D8-D10. This is the same bus cycle as variant A's `move.b d0,(a0)` and variant B's `CHILLY_RV_MEMCPY`. The only
difference is interrupts off, which settles point 4 of the comparison note: no 68000 interrupt handler can touch the
cart during the write.

**Why not a new command number.** Command 46's chunk runs send COMM2 = first chunk | more chunks << 8 (at most
`0x03FF`), so `0xF0xx` and `0xF8xx` never occur in them. Old games never send these values.

**Why the slot case.** Command 38 picks a file's cart slot from its file number (`cur_img`). Sonic must put files at
offset 0 and `0x080000` of a page seen through window 1 (slots 16 and 20). This plays the same role as
`CD32X_PAGED_CART_STACK_PROPOSAL.md`'s "CMD 38 bit 15 = destination given", at 128 KB granularity.

## 3. Sonic pages window 0 too, and why that is safe there

The other proposals keep window 0 at page 0, which is the right default. Sonic's act scenes must run in place: their
native scripts are linked to `0x02080000 + offset`, so an act's image has to be at window 0's `0x080000`. R4G keeps
pages 2-6 as **act pages**, each holding:
- **a byte-identical copy of the engine** at its `0x000000`;
- one act scene at its `0x080000`.

Window 0 is switched only under these conditions:

1. The target page's engine was compared byte for byte, uncached, with the running engine (through window 1) after
   it was loaded. **No compare, no switch.**
2. The SH2 that sends the command waits in SDRAM code (`sd_cmd`). The other SH2 is idle in SDRAM. The 68000
   listener runs from work RAM.
3. Both caches are purged after every switch and after every load (`0xFFFFFE92 = 0x11`).

Page 7 holds the act files in four 512 KB slots, read by Sonic's unchanged command 68 through the `0x900000` bank
window. Each slot lies inside one 1 MB bank.

## 4. The probe: read-only, no cart RAM written

At the title, the SH2 checks:
1. Window 1 to page 0. Compare window 1 with window 0 (uncached), 4 KB at the engine (`0x10000`) and 4 KB at the
   scene image's header (`0x080000`). Different: no mapper (4 MB). Window 1 back to page 1, two register writes in
   all.
2. Window 1 to page 4 and compare again. The same: 8 MB (alias); different: 16 MB. Back to page 1.

It writes no cart RAM, and nothing is saved and restored. **Known limitation:** an 8 MB board with the 16 MB GreenPAK
image has floating pages 4-7, which would read as "different", so R4G would take it for 16 MB. The PCB README
recommends the 8 MB image for a U1-only board. R4G assumes that image; with the other one it is untested. A write
probe (variant A/B's markers plus inverse) would settle it.

## 5. Living with other titles on the same cart

- **Pages 0 and 1:** rebuilt by Sonic's boot loader at every power-on (engine, data pack, title), as on the 4 MB cart.
- **Pages 2-7 are used freely**, with no owner record.
  - Sonic never trusts data it did not write: after a power-on it looks each page up in its own table (in its data
    pack on the disc) by a header fingerprint and checks it whole with the boot loader's checksum.
  - But it **will overwrite other titles' cached packs in pages 2-7**.
  - The comparison note proposes an owner record in each page's last 64 KB. Sonic's stack has no command that
    writes a few bytes at a chosen cart place (no CMD 58/61 save area: its saves go to the backup RAM). It can only
    write whole files into 128 KB slots, so writing that record would need one more change. Reading other titles'
    records, and not evicting their pages, would not.
- **The last 64 KB of every page are not touched** (variant B's rule 5): engine copy up to `0x062000`, images up to
  `0x193800`, act files up to `0x1EF000` of their page (whole 2 KB sectors).
- **Sonic does not use the cart save area** (`0x3F0000`). Its saves are in the Sega CD backup RAM.

## 6. The test: it answers the shared open questions too

**Result so far (19 Oct 2026, Sega CD + 32X + the 4 MB RAM cart).** A Sonic disc built before the slot test was added
(R4F or the first R4G build; both carry this `sub_main.c`) booted, and the Options Menu's `BIG CART` row ran the cart
test. The owner's photo of the screen shows:

- `CART TEST: NO PAGE MAPPER` / `(A 4 MB CART) - NOTHING WRITTEN`, and the row still `BIG CART   OFF`;
- the Options Menu drawn and working after the test.

What that proves:
- **The disc with the new `sub_main.c` boots** to the game and its menu.
- **The page register case ran end to end on the console, twice.** The SH2 sent command 46 `0xF8nn` (window 1 to page
  0, then back to page 1). The Sub-CPU took it, and the 68000's routine 26 (`SAFE_CART_XFER`: interrupts off, RV = 1)
  wrote `0xA130F2` and returned. The Sub-CPU answered both times: an unanswered command reads `CART TEST: NO ANSWER
  (SUB-CPU)`.
- **The 4 MB cart showed no effect from that write.** Window 1 kept showing its own memory, not page 0, so the test
  found no mapper. Nothing hung.

What it does not prove yet:
- that a byte somewhere in the 4 MB cart could not have changed (a cart that decoded `/TIME` as a RAM write would do
  that without showing here);
- the slot case `0xF0nn` (tried next - result below);
- a real page switch (the 8 / 16 MB board).

**Second result (19 Oct 2026, the same console and 4 MB cart): the slot case.** Sonic's build with the slot test (the
second R4G zip) booted and its cart test was run from the main menu. The owner's photo shows
`CART TEST: NO PAGE MAPPER (4 MB)` / `SLOT 21 LOAD OK (SUB-CPU AND SH2)`, and the row still `BIG CART   OFF`. Slot 21
is right for the main menu (its image ends at `0x298000`; from the title it would be slot 19, as sh2emu also showed).
What that proves:
- **The slot case ran on the console.** The SH2 opened the 256 KB test file (command 38, which puts it in slot 16 =
  cart `0x200000`), sent command 46 `0xF015` (slot 21), and loaded eight 32 KB chunks one by one. The Sub-CPU read
  every chunk back from cart `0x2A0000`-`0x2DFFFF` and found it right.
- **The SH2 read the same 256 KB through its cache at `0x022A0000`**, word for word against the pattern it computes
  itself.
- **Nothing went to the file's own slot:** cart `0x200000` (inside the menu's image) did not show the test data.
- **The page register case ran again** (two more page commands before the slot test), with the same result as
  before.

The owner's videos of the same session show the power-on: the boot text as R4D's (`BUILD: SONIC CD R4G GAME START` -
the round label stayed R4G -, `32X PAYLOAD: 5032 BYTES SH2-VR4...`, `SAFE CART: FOUND`, `32X GO!`), the loading picture
with its bar, then the title (`PRESS START`). The title is where the game sends its first two page commands (the
detection), so **the page command at every power-on did not stop the title on the 4 MB cart either.**

The game on that disc: the owner reports "First level plays good" (19 Oct). (After the cart test the menu's scene is
loaded whole at the next load, so no stale image is trusted.) **The 4 MB test burn is done:** the new boot stack boots
and plays, and both new cases ran. What is left needs the 8 / 16 MB board: a real page switch, then the game's own use
of the pages.

**The steps (all games can use them):**

Every variant does the same bus cycle (one even-byte write to `0xA130F2` with RV = 1), so one console test answers
for all of them.

1. **The 4 MB RAM cart first.** Burn Sonic's R4G disc.
   - At the title start it sends the page command twice (window 1 to page 0, back to page 1).
   - Options Menu → `BIG CART` (A) runs the cart test: two more page commands, then (no mapper found) the **slot
     command**. The 256 KB test file goes to the first free 128 KB slot after page 0's scene image, below the data
     pack at `0x300000` (from the title: slot 19, cart `0x260000`). Each 32 KB chunk is read back by the Sub-CPU, then
     all of them by the SH2. The file's own slot (16, `0x200000`) must not show them. It must read
     `CART TEST: NO PAGE MAPPER (4 MB) / SLOT 19 LOAD OK (SUB-CPU AND SH2)`. **Both new cases then ran on the
     console.** (Added 19 Oct, after this doc was first posted.)
   - Then the game must play exactly as R4D (loads, saves, music).
   - This answers **"does the 4 MB cart ignore an even-byte write to 0xA130F2"**. Sonic's own question: whether the
     4 MB cart (v01) uses /TIME at all. The 16 MB repo has only v01's edge connector, where B31 is a named net.
2. **The 8 MB board, then the 16 MB board.** The same row runs the cart test:
   - the read-only probe;
   - then a 32 KB pattern into each of pages 2-7 that is not an alias of page 0, through window 1, read back by the
     Sub-CPU and by the SH2.

   It must read `CART 16 MB: PAGES 2-7 OK / PAGE SWITCH OK, GAME PAGES UNTOUCHED` (8 MB: `PAGES 2-3`). This answers
   **"does the /TIME write reach the board with the 32X on (ADEN = 1, RV = 1)"**.
3. **Then the game's own use:** an act start with its preload, time warps without the disc, a power-off and on with
   the act still in the cart.

**In ares with a 16 MB SRAM cart (the owner's ares build, 19 Oct 2026).** Sonic's R4H build: `BIG CART   16 MB` -
the read-only probe found the 16 MB cart (window 1 switched to page 0 and to page 4, read by the SH2). The preload
itself is being checked next.

## 6b. Sonic R4I: the warm start (a boot loader change, no new command) - sh2emu only, not yet in ares or on the console

The battery keeps the cart, so Sonic's SH2 boot loader (`cd32x/sh2_engboot.c`, label `SH2-VR4I`) no longer re-reads
what the cart already holds:
1. It reads the data pack (`SCDDATA.PAK`, about 130 KB) first.
2. The pack's preload table holds the disc's checksums of the engine (file 21, kind 3) and the title (0x100). They
   are the Sub-CPU's `load_sum`, over the file padded to 2 KB.
3. The loader runs its own `scd_cart_sum` over the engine's and the title's places in the cart. If a checksum
   matches, that file is not read from the CD.
4. Any mismatch, or no table, means a cold start exactly as before.

It uses only commands 38 and 46. The engine then does the same for the menu (its own pages 6/7) and the act pages. In
sh2emu, a second power-on with the cart kept read only the loading picture and the data pack from the disc; the title,
the menu and Palmtree Panic 1 came from the cart. Any game with a battery-backed cart can do this in its own loader:
it needs a table of checksums that is read from the disc every time.

## 7. Where Sonic's form differs from the merged form (variant B)

| | Merged form (variant B) | Sonic R4E-R4G |
|---|---|---|
| Page command | new SH2 CMD 68, window 1 only, page 1-7 | inside the existing CMD 46: COMM2 `0xF8nn`, window 0 or 1, page 0-7 |
| 68000 routine | MD 33 `CHILLY_RV_MEMCPY` (RV; interrupts on) | MD 26 `SAFE_CART_XFER` (RV; interrupts off) |
| Placing a file | asked for: CMD 38 + a destination offset | built: CMD 46 COMM2 `0xF0nn`, a 128 KB slot until the next CMD 38 |
| Window 0 | never paged | paged, to pages holding a verified copy of the engine (section 3) |
| Size probe | marker + inverse in save piece 15 via 58 / 61 | read-only compare, no cart RAM written (section 4) |
| Page owner record | in each page's last 64 KB | none yet; the last 64 KB are left alone |

All of them do the same bus cycle for the page register, so the console test (section 6) answers the open questions
for every variant.

## 8. For the ledger

No new SH2 or MD numbers on the Kobo stack. Status of the two cases:
- `0xF8nn` (page register): **run on the console** (4 MB cart, 19 Oct 2026). The page switch itself is still untested.
- `0xF0nn` (file slot): **run on the console** (4 MB cart, 19 Oct 2026): the file went to the slot it was given and
  read back right on the Sub-CPU and the SH2.

If the ledger lists the Kobo stack, the entry would be: **command 46, COMM2 `0xF8nn` page register, `0xF0nn` file
slot**, plus a note that **68 = act file → Genesis planes** there.

Source: MicroNut99/sonic2011cd32x (local commits, not pushed): `cd32x/sub_main.c`, `cd32x/engine/plat_mars.cpp`
("THE PRELOAD"), `docs/R4G_NOTES.md`, `docs/R4G_16MB_REPORT.md`.
