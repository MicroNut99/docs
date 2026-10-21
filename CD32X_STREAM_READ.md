# CD32X boot stack: the drive reads on between chunks (STREAM)

20 Oct 2026, in OpenJazz CD32X R7 and Hurrican CD32X P2b (`cd32x/subcpu/main.c`, `scd.c`).
Measured in ares, not yet on the console.

## The problem

CMD 46 / 47 / 50 / 51 read each 32 KB chunk with `scd_sub_read` → `ReadSectorsSUB`: CDCSTOP,
then ROMREADN for 16 sectors. Every chunk stops the drive and sends it back to its sector, so
each chunk costs a seek. That gave about 41 KB/s, in ares and as measured on the console
(42 KB/s).

## The change

- **`scd_stream_start(lba, count)`:** CDCSTOP, then one ROMREADN to the end of the file (the file
  opened by CMD 38).
- **`scd_stream_sector(buf, hdr, spins)`:** the next sector and its header (CDCSTAT / CDCREAD /
  CDCTRN / CDCACK).
- **`cd_stream_read`:**
  - checks each header's MSF;
  - skips an earlier sector (the drive's run-up);
  - starts the drive again at the wanted sector when one was missed;
  - falls back to the old read after 64 failures.
- **`cd_pump` and the read-ahead:**
  - while the 68000 copies to the cart or reads back, and in the idle loop, the Sub-CPU takes
    the sectors that arrive into `st_buf`, a 64 KB ring in PRG-RAM;
  - every `do_md_cmdN` after its definition becomes `send_md_cmdN` + pump + `wait_md_cmd`.
- **Anything else that uses the drive stops the stream first:** `scd_sub_read`,
  `scd_open_file` and `scd_bios_call` are macros that clear it.
- **The read-back compare** runs four bytes at a time (the byte loop took about 100 ms a chunk).

No change to the 68000 listener or to the command numbers.

## Results (ares)

| | before | after |
|---|---|---|
| common pack (1.5 MB, with read-back) | 37 s, 41 KB/s | 15 s, ~104 KB/s |
| Hurrican stage 1 (no read-back) | 14.6 s | 5.3 s, ~143 KB/s |
