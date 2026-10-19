# CD32X – Paged 8 / 16 MB cart: the three stack proposals, compared (Jazz / Hurrican view)

**Status: review note, untested.** Three sessions wrote stack proposals for the
[16MB-PCB-CD32X](https://github.com/MicroNut99/16MB-PCB-CD32X) on the same day:

| File | From | Listener change |
|---|---|---|
| [CD32X_SRAM_PAGING_PROPOSAL.md](CD32X_SRAM_PAGING_PROPOSAL.md) (variant B) | OpenLara CD32X v2.0 | **none**: CMD 68 built from listener 33 (`chilly_rv_memcpy`) |
| [CD32X_CART_PAGING_PROPOSAL.md](CD32X_CART_PAGING_PROPOSAL.md) (variant A) | SuperTux CD32X REV | new routine MD 38 `set_cart_page` |
| this file (first version) | OpenJazz CD32X REV / Hurrican CD32X | new routine MD 42 `cart_page` + a shared directory in save piece 14 |

The Jazz / Hurrican side **withdraws its own numbers** (MD 42, piece 14) and **supports
variant B** (OpenLara). This note says why, and lists what the merged form should still cover. The
game-side designs are:

- [OpenJazzCD32X-REV `docs/CART_8MB_16MB.md`](https://github.com/MicroNut99/OpenJazzCD32X-REV/blob/main/docs/CART_8MB_16MB.md)
- [HurricanCD32X `docs/CART_8MB_16MB.md`](https://github.com/MicroNut99/HurricanCD32X/blob/main/docs/CART_8MB_16MB.md)

## 1. Where all three agree

- **The register:** the page register is reachable only by the 68000. So one new SH2 → Sub-CPU
  command is needed: **CMD 68**, window 1's page, written to `0xA130F2` (even byte, /UWR).
- **Window 0 is never paged:** the game runs from it.
- **RV = 1 around the write.** That is the README's open point and the way CMD 46 / 61 already
  write the cart.
- **Saves stay at `0x3F0000` on page 1.** Window 1 must be on page 1 for every save-area read
  and write.
- **After a page change:** both SH2s purge their caches, and the other SH2 stays off the cart
  during the switch.
- **Test order:** the page register is tested first. On a 4 MB cart the game must say 4 MB and
  change nothing.

## 2. Why variant B

- **The 68000 listener stays byte for byte.** `chilly_rv_memcpy` (MD 33) already does RV = 1,
  then `move.b` byte by byte, then RV = 0. A 1-byte copy to `0xA130F2` is exactly the even-byte
  /UWR write the board needs (checked in `hw_md.s`: `move.b d1,(a1)+`).
- **No new MD number** means no clash with other projects' listener tables.
- **The size probe needs no new command:** the SH2 does it with 68 + 58 / 61 (variant A's
  idea, with B's inverse-word guard against a floating bus).

## 3. What the merged form should still cover (from the Jazz / Hurrican design and its review)

1. **The probe must not damage a save.** Its marker on page 1 lands in the save area. It must
   go in free **piece 15** (as variant A says), never in a piece a title owns. The first draft
   of the OpenLara proposal put it at cart `0x3F02F0`, inside piece 0, which is OpenJazz's
   `openjazz.cfg`. Pages 2-7 are fine: the page tail is reserved there.
2. **The page tail (last 64 KB of pages 2-7) as the owner record.** Jazz / Hurrican adopt this
   in place of their shared piece-14 directory. A title writes there:
   - its identity tag and disc build mark;
   - the list of packs in the page (id, offset, size, checksum);
   - a last-use stamp.

   Then another title can see whose page it is and evict the least recently used one, instead
   of overwriting blindly. The format should be fixed in the ledger with the commands.
3. **Exit without a reset.** Before CMD 60 (start D32XR) or any other hand-over that is not a
   reset, window 1 goes back to page 1, in the Sub-CPU, so no title can forget it.
4. **Interrupts during the write.** `chilly_rv_memcpy` runs with interrupts on. This is safe
   only if no 68000 interrupt handler touches the cart. Jazz REV's vblank handler (the Genesis
   plane) reads only work RAM and COMM registers. Any title whose interrupt handlers read the
   cart must mask them around CMD 68.
5. **CMD 38 needs a destination offset** (the one addition Jazz / Hurrican ask for). Today
   CMD 38 picks a fixed 128 KB slot from the file id, and CMD 46 writes there. Jazz and Hurrican
   place several packs of 0.1-1.0 MB one after another in a page, so they need:
   - COMM8L = a 4-byte-aligned offset in window 1, taken when a flag bit is set;
   - refused when offset + length passes 0x1F0000 (the page tail);
   - without the flag, nothing changes.

   The first draft of the OpenLara proposal had this as an option.

## 4. A numbering clash to fix (older than these proposals)

OpenJazzCD32X-REV R1-R4 use **CMD 67** for the Genesis plane and **68000 routines 37-40**
(`genbg_load` / `start` / `off` / `cell`). The ledger already gives CMD 67 = cart → VRAM columns
and MD 37 = `copy_vram_col` (SuperTux REV4).

Fix:
- Jazz's Genesis plane moves to **CMD 70** (68 is paging; 69 was proposed for the size probe, so it stays clear) and to
  the next free listener numbers: MD 38-41 (variant B adds none).
- The Jazz numbers are registered when it moves.
