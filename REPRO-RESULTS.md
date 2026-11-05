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
| run4 (external, fresh VM, panic_on_warn=1) | 2026-11-05 10:01 | in progress | | | repeat run for the report |
| run5 (fresh VM, panic_on_warn=0, diagnostics) | 2026-11-05 10:01 | in progress | | | captures `unable to release extent buffer ... refs ... flags ...` lines |
