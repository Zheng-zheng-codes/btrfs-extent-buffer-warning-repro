# Reproduction run log (updated as runs complete)

Program set: repro/program-set.db (483 programs, extracted from crash-store
execution logs + state-primed recipe). Each run boots a fresh QEMU VM with the
same kernel/image/cmdline as the fuzzing campaign (KASAN, panic_on_warn=1
unless noted) and runs `syz-execprog -procs=8 -threaded -repeat=0` without a
time limit until the kernel panics on the target warning.

| run | start (CST) | result | time to hit | kernel uptime at warning | notes |
|-----|-------------|--------|-------------|--------------------------|-------|
| run1 (initial campaign report) | 2026-11-04 16:05 | HIT | n/a (found during fuzzing) | n/a | first report, crash store e8b4cb6e |
| run2 (external, fresh VM, panic_on_warn=1) | 2026-11-05 01:11 | HIT | ~32 min | 1807.6 s | serial: logs/external-repro-full-serial.log |
| run3 (external, fresh VM, panic_on_warn=1) | 2026-11-05 10:01 | HIT | ~45 min | 2732.2 s | same warning site disk-io.c:3326; serial kept in validation/external_repro_20261105/btrfs-repro2/ |
| run4 (external, fresh VM, panic_on_warn=1) | 2026-11-05 10:01 | HIT | ~2h03m | 7445.1 s | same warning site disk-io.c:3326; serial: logs/serial-btrfs-repro3.log |
| run5 (fresh VM, panic_on_warn=0, diagnostics) | 2026-11-05 10:01 | 4 warnings captured | first at ~1.6 h | 5617.8 s | `refs`/`flags` samples below; run continues |

Diagnostics captured by run5 (`logs/refcap-refs-flags-diagnostic.log`), all
during btrfs unmount:

```text
BTRFS warning (device loop4 state I): unable to release extent buffer 6881280 owner 5 gen 9 refs 2 flags 0xd
BTRFS warning (device loop0 state I): unable to release extent buffer 6885376 owner 5 gen 9 refs 2 flags 0x1d
BTRFS warning (device loop8 state I): unable to release extent buffer 6885376 owner 5 gen 9 refs 2 flags 0x1d
BTRFS warning (device loop1 state I): unable to release extent buffer 6885376 owner 5 gen 9 refs 2 flags 0x5
```

Flag bits (fs/btrfs/extent_io.h): bit0 UPTODATE, bit1 DIRTY, bit2 TREE_REF,
bit3 STALE, bit4 WRITEBACK.

- In all four instances `refs == 2`, i.e. no reference holder beyond the
  baseline (one held by the invalidation loop, one for the buffer-tree
  ownership) — `refs > 2` was not the trigger at print time.
- In two of four instances `flags` include EXTENT_BUFFER_WRITEBACK, i.e.
  `extent_buffer_under_io()` was true and the extent buffer was still under
  writeback at `close_ctree()`.
- In the other two instances the printed flags show neither WRITEBACK nor
  DIRTY, indicating the state had already changed within the window between
  the refcount check (under `eb->refs_lock`) and the message formatting.

This suggests the warning fires on transient unfinished I/O (writeback) on a
btree extent buffer during unmount rather than on a leaked reference;
the VM kept running and later unmounts proceeded normally.
