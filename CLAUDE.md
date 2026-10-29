# CLAUDE.md

Working notes for contributors, human or AI, who maintain this driver. It
covers what the code assumes, how it was ported and tested, and where the
known loose ends are. Read it before touching `skd_main.c`.

## What this is

`skd` is the block driver for sTec S1120 PCIe SSDs (PCI `1b39:0001`). It was
removed from mainline in Linux 5.12. This repository is the v5.11 driver
(`drivers/block/skd_main.c`, `skd_s1120.h`) ported to current kernels as an
out-of-tree DKMS module.

- `skd_main.c`: the whole driver, plus a short compatibility block after the
  includes.
- `skd_s1120.h`: hardware/FIT protocol definitions, unchanged from v5.11.
- `Makefile`: kbuild wrapper; `KVER=` picks the kernel, `KDIR=` the build tree.
- `dkms.conf`: `PACKAGE_VERSION` must match `MODULE_VERSION` in `skd_main.c`.

## Conventions

- **Kernel coding style.** Tabs, 80 columns, the original naming. The diff
  against v5.11 should stay small and readable, so a reviewer can compare
  with the upstream history.
- **Minimum kernel is 6.8**, enforced by `#error`. Don't add guards for older
  kernels: v5.11 itself covers those.
- **One guard per API change, keyed on the release that changed it.** Before
  adding a guard, check the boundary against the kernel tags (for example
  `git show v6.14:include/linux/blk-mq.h`), never from memory. Every boundary
  below was checked this way.
- **Versioning:** `<last mainline kernel>-oot.<n>`. Bump `n` in
  `skd_main.c` (`MODULE_VERSION`), `dkms.conf` and `README.md` together.

## Compatibility map

| Change | Boundary | Handling |
|---|---|---|
| `->timeout(rq, reserved)` → `->timeout(rq)` | 5.19 | unconditional |
| `busy_tag_iter_fn` lost `reserved` | 6.0 | unconditional |
| `alloc_disk`/`blk_cleanup_queue`/`GENHD_FL_UP` gone | ≤ 6.0 | `blk_mq_alloc_disk`, `disk_live`, `del_gendisk` + `put_disk` |
| `pci_{en,dis}able_pcie_error_reporting` gone | 6.5 | removed, the PCI core enables AER |
| `PCI_IRQ_LEGACY` → `PCI_IRQ_INTX` | `INTX` exists in 6.8 | unconditional |
| `del_timer_sync` → `timer_delete_sync` | exists in 6.8 | unconditional |
| `blk_mq_alloc_disk(set, lim, data)` + `queue_limits` | 6.9 | guarded; ≤ 6.8 uses `blk_queue_*` setters |
| `blk_queue_max_hw_sectors` and friends removed | 6.10 | covered by the 6.9 guard |
| `queue_limits.features` (`BLK_FEAT_*`), `blk_queue_write_cache`/`QUEUE_FLAG_NONROT` removed | 6.11 | guarded |
| `<asm/unaligned.h>` → `<linux/unaligned.h>` | 6.12 | guarded include |
| `BLK_MQ_F_SHOULD_MERGE` removed | 6.14 | guarded |
| `blk_rq_map_sg(q, rq, sg)` → `blk_rq_map_sg(rq, sg)` | 6.15 | guarded |
| `from_timer` → `timer_container_of` | 6.16 | compat macro for < 6.16 |
| `->getgeo(bdev, geo)` → `->getgeo(disk, geo)` | 6.18 | guarded |

Queue defaults since 6.11: blk-mq sets only `BLK_FEAT_IO_STAT | BLK_FEAT_NOWAIT`
(checked in `block/blk-mq.c` at v7.0). Non-rotational and no-entropy are
therefore the defaults, and the driver only requests
`BLK_FEAT_WRITE_CACHE | BLK_FEAT_FUA`. That matches the old
`blk_queue_write_cache(q, true, true)`.

Teardown order, which matters: `del_gendisk` (only if `disk_live`), then
`put_disk` (this frees the queue), then `blk_mq_free_tag_set`. The probe error
path reaches `skd_free_disk()` before the disk has been added, which is why
the `disk_live()` check is there.

## Behavior change from v5.11

Upstream `skd_inc_in_flight()` did `count++` on the pointer, so
`skd_in_flight()` always returned 0. That is fixed. The visible effect: in
`skd_isr_completion_posted()`, the `PAUSING → PAUSED` transition now waits for
outstanding requests to drain, as originally intended. If a suspend or pause
ever hangs, look here first.

## Building for a kernel you are not running

The host compiler has to accept the target kernel's flags. Ubuntu 24.04's
gcc-13 cannot build against 7.x headers, which were built with gcc-15
(`-fmin-function-alignment`). Use a container that matches the target
distribution instead:

```dockerfile
FROM ubuntu:26.04
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    --no-install-recommends make gcc-15 linux-headers-7.0.0-14-generic kmod dkms
```

```sh
docker run --rm -v $PWD:/src -w /src skd-build:7.0 \
    make KVER=7.0.0-14-generic CC=gcc-15
```

The "compiler differs from the one used to build the kernel" warning is
harmless when only the binary name differs (`gcc` versus
`x86_64-linux-gnu-gcc-13`, same version). It is not harmless when the major
versions differ.

A release should build with **no warnings** on every kernel in the test
matrix. Nothing is checked into CI yet.

## Test ladder

Run the steps in order and stop at the first failure. Each step only makes
sense once the previous one passes.

1. Build: `make` and the container build(s). No warnings.
2. Load: `insmod skd.ko`; dmesg shows `STEC s1120 ONLINE` within a few seconds.
3. `/dev/skd0` appears, and the partition table is read if one exists.
4. Queue attributes in `/sys/block/skd0/queue/`: `rotational=0`,
   `add_random=0`, `write_cache=write back`, `fua=1`,
   `max_hw_sectors_kb=1024`, `max_segments=256`, `optimal_io_size=8192`.
5. Read-only: checksum a region through buffered I/O, O_DIRECT 1M and
   O_DIRECT 4k; all three must match. Then
   `fio --readonly --rw=randread --iodepth=64`.
6. Unload/reload several times; dmesg must show no warnings, leaks or traces.
7. Destructive, needs a disk you can wipe: fio `randwrite` at QD64 with
   `--verify=crc32c --verify_fatal=1`, mixed `--bsrange=4k-1M` with several
   jobs, then a full-disk sequential write with verify.
8. Filesystem: mkfs, mount, write, `sync`, unmount, check (`xfs_repair -n`
   or `e2fsck -fn`).
9. Reboot: the module autoloads from its modalias and the device comes ONLINE.

## Status of the 5.11-oot.1 port

Test card: S1120, 1 TB (1953525168 × 512 B), PCIe Gen2 x4.

- **6.8 (Ubuntu 24.04): steps 1–9 passed.** After a reboot, the module
  autoloaded from its modalias at 3.3 s and the card was ONLINE at 3.8 s.
  XFS checksums still matched after the power cycle. Details:
  - randwrite 4k QD64, 16 GiB, crc32c verify: err=0, 20.6k write IOPS,
    82.8k IOPS during verify reads.
  - mixed 4k–1M, 4 jobs × 8 GiB, verify: err=0, 495 MB/s write.
  - full disk (932 GiB) sequential 1M QD32, verify: err=0, 881 MiB/s write,
    1276 MiB/s read. No driver messages in dmesg during the whole suite.
  - XFS: 16 GiB large files plus 20,000 small files; sha256 matched after a
    cache drop and after a remount; `xfs_repair -n` clean.
- **7.0 (Ubuntu 26.04): step 1 only** (gcc-15 container, plain make and DKMS
  3.2.2). Not loaded on hardware yet.
- 6.9 – 6.19: not built. They're covered only by guards whose boundaries were
  checked against the tags.

Unexplained observations, probably device-side but not proven:

- Read throughput varied a lot between runs on the same region: one buffered
  4 GiB read gave 30 MB/s, later runs 332–876 MB/s. O_DIRECT 1M reads ran at
  516 MB/s over 4 GiB and 1.3 GB/s over the first 512 MiB. Request sizes were
  correct (a full 1 MiB, no splitting) and the driver logged nothing.
- Rare 1.5–1.9 s completion stalls appear under 4k random I/O at QD64,
  on both reads and writes. Sequential 1M I/O never went above 42 ms. The
  stalls are under the 8 s request timeout, so `skd_timed_out()` never fired.
  This looks like device-side garbage collection, but that is not proven.

## Known loose ends

- **Queue depth:** the driver logs `Queue depth limit=64 dev=255`. The
  `skd_max_queue_depth` module parameter and its interaction with the
  device-reported limit have not been explored; 4k random-read IOPS may be
  capped by it.
- **Power management** still uses the legacy `pci_driver.suspend/.resume`
  hooks. They still exist in 7.0, but the modern way is `dev_pm_ops`.
  Suspend and resume are untested.
- **Secure Boot is untested.** The test host had Secure Boot disabled and
  loaded the module unsigned (the kernel logged the usual taint notice). On
  Ubuntu, DKMS signs with the MOK key, which the user has to enroll once.
- **No discard/TRIM support.** The v5.11 driver had none either.
- **Legacy INTx and MSI** fallbacks (`skd_isr_type`) are untested; the test
  card used MSI-X.
- `skd_pci_info()` only decodes link speeds of 2.5 and 5 GT/s, which is right
  for this Gen2 card.
