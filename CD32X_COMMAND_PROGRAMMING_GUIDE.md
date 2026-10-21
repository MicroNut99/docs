# CD32X Command Programming Guide

How to use the command ledger when you write code that talks to the Sega CD from the 32X.

**The registry is two documents, read together:**
- **KOBO-Sega 32XCD Boot Sequence and Sub-CPU Command Ledger.pdf**: the original commands and 68000 routines.
- **[CD32X_COMMAND_LEDGER_ADDITIONS.md](CD32X_COMMAND_LEDGER_ADDITIONS.md)**: everything added since. Where the two
  differ, the additions are newer and win.

The per-game addenda (OpenLara, OpenJazz 2, OpenTyrian, OutRun) explain how a game uses the commands. They are not
the registry.

## 1. Find your stack first

The same number can mean different things on different stacks. Know which one your disc runs on.

| Stack | Sub-CPU | 68000 listener | Games | Ledger |
|---|---|---|---|---|
| **OpenLara / Hurrican path** | `main.c` | `hw_md.s` (MD 1–37, `MD_CMD_END` 38) | OpenLara, Hurrican, SuperTux | sections 1–3, 5, 7, 8 |
| **Kobo path** | `cd32x/sub_main.c` | `cd32x/kobo/hw_md.s` (MD 1–34, `MD_CMD_END` 35) | Sonic CD 2011 | section 6 |
| **Cartridge stack** | – | `cart_md.s` | OpenTyrian | section 4 |

**The clashes to check when moving code between stacks:**

| CMD | OpenLara path | Kobo path | Cartridge stack |
|---|---|---|---|
| 46 | chunk → cart; bit 15 = no read-back | chunk → cart, 3 forms; `0xF8nn` = cart page, `0xF0nn` = slot | – |
| 47 | chunk compare | cart → frame buffer | – |
| 60–63 | D32XR start, save piece → cart, Genesis picture | – (60, 61 absent) | CD find, CD read, SRAM read / write |
| 68 | cart page, window 1 | act file → Genesis planes | – |

## 2. The handshake (SH2 side)

Only the **master SH2** sends commands. The slave never does. COMM4 is kept for master ↔ slave messages.

| Register (SH2 address, cache-through) | Use |
|---|---|
| COMM0 `0x20004020` | the command number; the Sub-CPU clears it to 0 when done |
| COMM2 `0x20004022` | first argument (16 bit) |
| COMM8 `0x20004028` (32 bit: COMM8L) | second argument, or the reply (lengths, pad bits) |
| COMM12 `0x2000402C` | reply flag of some commands; otherwise the game's status code (shown in the green text) |

1. Write the arguments: COMM2, COMM8.
2. If the command replies in COMM12, clear COMM12 first.
3. Write the command number to COMM0.
4. Wait until COMM0 reads 0. Pause between reads (a short loop that doesn't touch the bus), and give up after a
   timeout.
5. If COMM0 is still not 0, there was no answer: report it. Otherwise read the reply the ledger names for that
   command.

```c
static void cdPause(void) { for (volatile int k = 0; k < 64; k++) {} }   /* never hammer a COMM register */

static int cdCmd(uint16_t cmd)                       /* 1 = answered, 0 = no answer */
{
    MARS_SYS_COMM0 = cmd;
    for (int n = 0; n < 2000000 && MARS_SYS_COMM0 != 0; n++)
        cdPause();
    return MARS_SYS_COMM0 == 0;
}

/* example: open file 100 (OpenLara: a level), length comes back in COMM8L */
MARS_SYS_COMM2 = 100;
if (cdCmd(38)) len = *(volatile uint32_t *)0x20004028;
```

**Reply values are per command; don't assume them.** Command 61 replies COMM12 = 1 for OK. On OpenLara's use of
command 46, COMM12 = 0 means done. Command 38 replies in COMM8L. Read the command's row every time.

## 3. Looking up a command

| You want | Ledger |
|---|---|
| Which SH2 numbers exist on the OpenLara path, and which are free | section 3 |
| Genesis planes, tiles, palette, scroll (62–67) | section 1 |
| The 68000 routines a command uses (MD numbers) | section 2, and the PDF |
| Cart save area (58 + 61) | section 5 |
| Paged 8 / 16 MB cart (68) | section 7, 7a |
| Sonic / Kobo numbers, its paging and stamp commands (46 forms, 80) | section 6 |
| A game's use of existing commands (no new number) | e.g. section 8 |

Every row gives the same facts: CMD, name, COMM2, COMM8, reply, the 68000 routine, and a **status**. Trust a
command only as far as its status says:

| Status | Meaning |
|---|---|
| emulation / sh2emu | runs in an emulator only |
| boot-proven | a disc with it booted on the console; not yet used by a game |
| run on the console | used and checked on real hardware |

## 4. Rules that apply to every command

- **Never reuse a number with a different meaning**, on any stack. The same goes for save pieces.
- **Nothing in the boot sequence sends a new command.** New commands are sent by the game, after boot.
- **Cart writes go through the 68000 with RV = 1.** While RV = 1, SH2 cart reads are simply held, so no parking
  code is needed. After any command that changes cart contents or pages (46, 61, 68), **clear both SH2 caches**
  (the master sends the slave its cache command).
- **An unanswered command looks like a hang, not an error.** SuperTux's first test discs waited 35–50 s per
  command, because the Sub-CPU was built without the command. Use a timeout, and show a code in COMM12.
- **A command behind a build switch must reach `main.c`.** `main.c` is compiled with **`HWFLAGS`**, not `CCFLAGS`.
  Make the build stop if the switch isn't on `main.c`'s compile line.
- **Kobo Word RAM:** while Sonic's stamp floor (80) is on, Word RAM belongs to it. Only the commands listed in
  section 6 may run.

## 5. The save area (commands 58 + 61)

- **Where:** cart `0x3F0000`–`0x3FFFFF`, 16 pieces of 4 KB, battery-backed.
- **Reading:** read it directly from the SH2 at `0x223F0000 + piece × 4096` (cache-through).
- **Writing:** fill the Sub-CPU's buffer word by word with **58** (COMM2 = word index, COMM8 = word), then send
  **61** with COMM2 = piece. The 68000 copies it into the cart and checks it; COMM12 = 1 means OK.
- **Ownership:** one piece, one owner. Before taking a piece, check section 5's table.
  - Free as of 21 Oct 2026: **11, 12, 14**.
  - Piece 15 is test scratch: never keep data there.
- **Write order:** write the marker or magic word last, so a power-off mid-write leaves an invalid record rather
  than a wrong one.

## 6. The paged 8 / 16 MB cart

- **Page switch:**
  - OpenLara path: **68**, COMM2 = page 1–7, window 1 only.
  - Kobo path: **46** with COMM2 = `0xF800 | window << 8 | page`.
  - Both use the existing 68000 routine 26 `safe_cart_xfer`.
- **Window 0 is never paged.** It holds the program and resident data.
- **Saves stay on page 1.** Put window 1 back on page 1 before any 58 / 61, or any read of `0x3F0000`.
- **Clear both SH2 caches** after every page change.
- **Hardware status:** a real page switch on the 8 / 16 MB board is untested. Only the 4 MB answer, no paging, is
  confirmed. Run a cart test disc first.

## 7. Adding a new command

1. **Check that you need one.**
   - Section 8 is an example that needs no new number: it builds a new feature out of 38, 46, 53, 54, 58 and 61.
   - Prefer reusing an existing 68000 routine. A new `hw_md.s` routine changes the listener, which is copied at
     boot.
2. **Pick a number free on every stack.**
   - Check sections 3, 4 and 6.
   - As of 21 Oct 2026, unused everywhere: **69–79, 81 and up** (39–43, 49 and 55 are free on the OpenLara path
     only).
3. **Put the command behind a build switch** in the Sub-CPU (`HWFLAGS`). Leave the normal build byte-identical.
4. **Add a row to the additions ledger before the disc goes out.** Give:
   - CMD, name, COMM2, COMM8, reply, the 68000 routine, and which stack;
   - why it was needed;
   - its status;
   - a link to the code.
   - Also update section 3's "Free:" line.
5. **Raise the status as it is proven:** emulation → boot-proven → run on the console, each with a date.

## 8. Before a burn: checklist

- [ ] Every number the game sends is in the ledger for **this** stack, with the meaning the code expects.
- [ ] The Sub-CPU build has every switched command (`HWFLAGS`). Check `main.c`'s compile line.
- [ ] Every command wait has a timeout, and a failure shows a COMM12 code.
- [ ] Caches are cleared after cart writes and page changes.
- [ ] Save writes stay inside your own pieces, and window 1 is on page 1 when they run.
- [ ] Boot block, CDLOADER, `crt0.s` and the listener are unchanged, unless the ledger records the change.
