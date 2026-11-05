# Btrfs extent-buffer cleanup WARNING on Linux 7.4-rc1 (reproduction evidence)

WARNING in `invalidate_and_check_btree_folios()` (fs/btrfs/disk-io.c:3326)
during Btrfs unmount (`close_ctree`), observed while fuzzing with syzkaller on
Linux 7.4-rc1 with `CONFIG_BTRFS_DEBUG=y` and `panic_on_warn=1`.

This repository contains the kernel configuration, the kernel/syzkaller logs,
and the program set used to reproduce the warning.

## Tested kernel (version / commit / configuration)

| item | value |
|---|---|
| kernel release | `7.4.0-rc1-kernsight-kasan+ #3` |
| base source | v7.4-rc1 (`6628e7d4c`), Btrfs code unmodified |
| local delta | one io_uring SQPOLL fix on top: `00043c58565e2d0d804f50c2e127f6966cd594af` (touches only `io_uring/sqpoll.c`, not fs/btrfs) |
| architecture | x86_64 |
| environment | QEMU Standard PC (i440FX + PIIX), KVM, 4 vCPU / 6 GiB RAM per VM |
| notable config | `CONFIG_KASAN=y`, `CONFIG_BTRFS_DEBUG=y`, `CONFIG_BTRFS_ASSERT=y` |
| cmdline | `panic_on_warn=1 oops=panic hung_task_panic=0 softlockup_panic=1 sysctl.net.core.netdev_unregister_timeout_secs=140` (full cmdline in `repro/environment.txt`) |
| fuzzer | syzkaller (upstream 853b104bb + local integration) |

Full configuration: `config/kernel-7.4-rc1-kasan.config` (~295 KB, 4758
`CONFIG_` entries; check with
`grep -E "CONFIG_KASAN|CONFIG_BTRFS" config/kernel-7.4-rc1-kasan.config`).

## The warning

```text
------------[ cut here ]------------
WARNING: fs/btrfs/disk-io.c:3326 at invalidate_and_check_btree_folios+0x5ce/0xbe0

Call Trace:
 invalidate_and_check_btree_folios
 close_ctree+0xac9/0x16a0
 btrfs_put_super
 generic_shutdown_super+0x169/0x3a0
 kill_anon_super
 btrfs_kill_super+0x41/0x60
 deactivate_locked_super
 deactivate_super
 cleanup_mnt
 task_work_run
...
Kernel panic - not syncing: kernel: panic_on_warn set ...
```

Context: `invalidate_and_check_btree_folios()` runs at the end of
`close_ctree()` after `invalidate_inode_pages2()`; the warning fires when an
extent buffer still cannot be invalidated (`refs > 2` or
`extent_buffer_under_io(eb)`). Some reports carried the SOFTLOCKUP taint
(`L`); whether an earlier soft lockup contributed is **not established**.

## How to reproduce

1. Build the kernel with `config/kernel-7.4-rc1-kasan.config`
   (`CONFIG_KASAN=y` and `CONFIG_BTRFS_DEBUG=y` are required for this exact
   warning; `CONFIG_BTRFS_DEBUG=y` gates the `WARN_ON_ONCE`).
2. Boot a VM with the same QEMU shape and cmdline as the campaign
   (`repro/environment.txt` has the full cmdline; disk image is a standard
   Debian/trixie rootfs, `vm/trixie.img`, ~6 GB).
3. Copy `syz-execprog` and `syz-executor` (built from the same syzkaller
   source) and `repro/program-set.db` into the guest, then run inside the
   guest:

   ```bash
   cd /tmp/repro
   syz-execprog -procs=8 -threaded -repeat=0 \
       -executor ./syz-executor ./program-set.db
   ```

   `-repeat=0` loops the program set forever; no time limit.
4. Watch the serial console. When the warning fires,
   `panic_on_warn=1` turns it into a kernel panic and the VM stops.

`repro/program-set.db` contains **483 deduplicated syzkaller programs**
extracted from the crash-store execution logs (last 20 programs of each
crash log) plus the state-primed replay recipe. It is the exact set that
reproduced the warning in the runs below; no single-program minimized
reproducer exists yet (automated minimization did not converge).

## Reproduction results

| run | start (CST) | result | time to warning | kernel uptime at warning |
|-----|-------------|--------|-----------------|--------------------------|
| campaign reports | 2026-11-04 16:05 | 19 occurrences | n/a (found during fuzzing) | varies |
| external run 1 | 2026-11-05 01:11 | HIT | ~32 min | 1807.6 s |
| external run 2 | 2026-11-05 10:01 | HIT | ~45 min | 2732.2 s |
| external run 3 (repeat) | 2026-11-05 10:01 | in progress | | |
| diagnostics run (`panic_on_warn=0`, to capture `refs`/`flags`) | 2026-11-05 10:01 | in progress | | |

All runs used the same kernel/image/cmdline and the same program set.
The two reproducing runs panicked at the identical warning site
(fs/btrfs/disk-io.c:3326).

## Files

```
config/kernel-7.4-rc1-kasan.config    kernel .config of the tested build
logs/kernel-warning.log               warning + call trace + panic (excerpt)
logs/external-repro-full-serial.log   full serial console of external run 1
logs/syzkaller-execution-log0.txt     syzkaller execution log at report time
logs/syzkaller-reports/report-<n>.txt all 20 syzkaller crash reports
logs/syzkaller-logs/log-<n>.txt       all 20 syzkaller execution logs
logs/title-stat.json                  crash title counters
logs/repro-attempts.json              automated (state-primed) repro attempts
repro/program-set.db                  verified program set (483 programs)
repro/environment.txt                 kernel/image/QEMU/cmdline details
REPRO-RESULTS.md                      reproduction run log
REPORT-DRAFT.txt                      upstream bug-report email draft
```

## Open questions (not asserted)

- Whether this is a new extent-buffer lifetime problem, another manifestation
  of a known issue, or a secondary effect of an earlier soft lockup.
- Whether the `refs`/`flags` of the stuck extent buffer indicate a reference
  leak or unfinished I/O (the `panic_on_warn=0` diagnostics run is meant to
  capture the `unable to release extent buffer ... refs ... flags ...` line
  that follows the WARN).
