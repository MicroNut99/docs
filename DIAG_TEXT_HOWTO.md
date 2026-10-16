# Instructions: turn on the green and white boot diagnostics (CD32X)

For any game that uses the OpenLara CD32X boot stack (Sega CD + 32X + RAM cart).
Follow exactly. Change nothing else.

## Rules
- Do NOT edit, rewrite or "improve" the boot stack: boot block, `subcpu/`, `hw_md.s`, `sh2/`,
  the game's `crt0.s`, `src-md/`, `m68k.bin`.
- Do NOT add new switches, new diagnostics or new text. The switches below already exist.
- Change only the values listed below, from 0 to 1. Nothing else in those files.

## Turn them on
1. `subcpu/main.c`: set
   `#define KOBO_DIAG_TEXT 1` (green text: boot steps, and the game's COMM12 status as `STATE:`)
2. `subcpu/main.c`: set
   `#define KOBO_BOOT_TEXT 1` (white text: the 32X start-up messages)
3. Only if the text must stay on after the game starts: in the game's `main()` set its own
   switch to 1 (Cannonball: `port/32x/main_32x.cpp`, `#define CB_DIAG_TEXT 1`). The game
   sends CMD 52 with COMM2 = that value; 0 switches the Genesis text off.
   Leave it at 0 if the boot fails before the game starts: it changes nothing then.
4. Rebuild with the project's normal build (`bash build.sh`).
   Expect "APP.BIN differs from ../OpenLaraCD32X". That is correct while the diagnostics
   are on. CDLOADER.BIN must still be identical.
5. Run the `.cue` in Fusion (Sega CD + 32X) first. Then burn the `.cue`.

## Reading the screen
- White text = the 32X start-up. Green text = the Sub-CPU's boot steps and `STATE:`.
- `STATE:` codes the game posts in COMM12 (Cannonball): 4B00 main reached, 4B01 loading,
  4E8n loading chunk n, 4B02 data in the cart, 4B03 engine ready, 4B10 running,
  4BE1-4BE4 errors. The last code shown is where it stopped.
- Pad button C shows or hides the green text.
- The Genesis text sits on top of the 32X picture. Once the game is running it covers
  the game's own screen: then turn it off again.

## Turn them off again
Set all three back to 0 and rebuild. `build.sh` must again say APP.BIN and CDLOADER.BIN
are identical to OpenLara's.
