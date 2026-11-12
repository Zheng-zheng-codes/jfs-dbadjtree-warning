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

`dbAdjTree()` (fs/jfs/jfs_dmap.c) adjusts the dmap control tree for a leaf
index; the `WARN_ON_ONCE(lp >= size || lp < 0)` fired because the computed
leaf index (`lp = leafno + tp->dmt_leafidx`) fell outside the tree — the
crafted JFS image's block-allocation state drove an out-of-range leaf
index into the tree adjustment. The commit runs on the `jfsCommit`
kernel thread. The guest runs with `panic_on_warn=1`, which escalates
this warning to a kernel panic.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_JFS_FS=y, booted with
`oops=panic` and `panic_on_warn=1`. See `environment.txt` for the exact
QEMU command line and `kernel.config` for the configuration.

The crash depends on JFS block-allocation state built up by the preceding
programs, so the reproducer is an ordered program set (75 programs):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

Verified by the fuzzing pipeline's reproducer machinery on a fresh VM
(reproducer pipeline outcome: confirmed=true, status=verified). A C
reproducer has not been included yet; it will be added once syzkaller's
C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (75-program ordered set + triggering testcase)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The out-of-range leaf index comes from the block-allocation state built
  on the crafted image (`syz_mount_image$jfs` + allocation-heavy
  operations); bounding the leaf index computation against the tree size
  is the natural first check.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
