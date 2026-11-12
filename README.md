# WARNING in dbAdjTree (fs/jfs/jfs_dmap.c:2962, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

## Summary

The JFS block-map (dmap) tree adjustment path trips a range sanity check:

```
------------[ cut here ]------------
lp >= size || lp < 0
WARNING: fs/jfs/jfs_dmap.c:2962 at dbAdjTree+0x315/0x410 fs/jfs/jfs_dmap.c:2962, CPU#3: jfsCommit/130
RIP: 0010:dbAdjTree+0x315/0x410 fs/jfs/jfs_dmap.c:2962
```

`dbAdjTree()` (fs/jfs/jfs_dmap.c) adjusts the dmap tree for a leaf index.
The `WARN_ON_ONCE(lp >= size || lp < 0)` check was triggered because the
computed index (`lp = leafno + le32_to_cpu(tp->dmt_leafidx)`) was outside
the valid tree bounds. This may be related to inconsistent JFS block-map
metadata. The warning occurred in the `jfsCommit` kernel thread. The
guest runs with `panic_on_warn=1`, which escalates this warning to a
kernel panic.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_JFS_FS=y, booted with
`oops=panic` and `panic_on_warn=1`. See `environment.txt` for the exact
QEMU command line and `kernel.config` for the configuration.

The crash depends on JFS state built up by preceding programs. The
reproducer is a multi-program syzkaller execution log (76 programs,
including `syz_mount_image$jfs` calls and allocation-heavy operations):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

The programs are replayed repeatedly with 8 parallel workers (`-repeat=0`);
the execution order is not guaranteed to be strictly serial. The precise
triggering operation and required filesystem state have not yet been
isolated.

Verified by the fuzzing pipeline's reproducer machinery on a fresh VM
(reproducer pipeline outcome: confirmed=true, status=verified). A C
reproducer has not been included yet; it will be added once syzkaller's
C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (76-program multi-program execution log)
- `reproducer/repro.prog` — candidate testcase extracted from the reproduction pipeline; not confirmed as the trigger
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The out-of-range leaf index (`lp = leafno + le32_to_cpu(tp->dmt_leafidx)`)
  comes from the block-allocation state built on the crafted image. The
  `WARN_ON_ONCE` at fs/jfs/jfs_dmap.c:2962 is the existing check; the
  underlying cause of the inconsistent metadata that produces the
  out-of-range index has not been isolated.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
