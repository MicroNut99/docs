# Sega 32X / Genesis reference docs (from sega-vr-disasm)

Copied from [matiaszanolli/sega-vr-disasm](https://github.com/matiaszanolli/sega-vr-disasm/tree/master/docs)
at commit 60819ef (Oct 18 2026): the hardware and programming references only, not that project's own
tooling notes. The repo's license is in `LICENSE-sega-vr-disasm.txt`; the transcribed Sega, Hitachi
and Motorola manuals remain their owners' documents.

Most useful for the 16 MB cart (16MB-PCB-CD32X):
- `32x-hardware-manual.md` - 3.1 memory maps (ADEN = 1 lists 0xA130F0 as "Bank Set Register"), 4.1 cart wait states (SH2 6-15 waits, 68000 0-5).
- `32x-sram-cartridge-manual.md`, `32x-eprom-cartridge-manual.md` - Sega's own 32X development carts: bank registers at 0xA130F1-FF (odd bytes).
- `32x-hardware-information.md` - SRAM development boards had to be modified for the 32X: their data buffers' DIR moved from /LWR, /UWR to the cart /OE (B16).
- `32x-technical-info.md` - RV bit rules (68000 interrupts off while RV = 1, BankSet example code), EPROM speed 120 ns or less.
