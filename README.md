# skd: out-of-tree driver for sTec S1120 PCIe SSDs

The `skd` block driver for sTec (later HGST / Western Digital) S1120 PCIe
accelerator SSDs (PCI ID `1b39:0001`) was removed from mainline Linux in 5.12.
This repository revives it as an out-of-tree, DKMS-packaged module for current
kernels.

It is based on `drivers/block/skd_main.c` and `skd_s1120.h` as of Linux
v5.11, the last release that shipped the driver. It keeps the original
authorship and GPL-2.0 license.

## Supported kernels

Linux **6.8 and newer**. Kernel API differences are handled with
`LINUX_VERSION_CODE` guards in `skd_main.c`.

| Kernel | Distribution | Status |
|---|---|---|
| 6.8 | Ubuntu 24.04 | built, loaded and tested on real hardware |
| 7.0 | Ubuntu 26.04 | built with gcc-15 via DKMS, not yet loaded on hardware |
| 6.9 – 6.19 | — | guarded by version checks, not build-tested |

## Install

```sh
sudo apt install dkms build-essential linux-headers-$(uname -r)
sudo mkdir /usr/src/skd-5.11-oot.1
sudo cp skd_main.c skd_s1120.h Makefile dkms.conf /usr/src/skd-5.11-oot.1/
sudo dkms add skd/5.11-oot.1
sudo dkms install skd/5.11-oot.1
sudo modprobe skd
```

When it works, `dmesg` shows `STEC s1120 ONLINE` and `/dev/skd0` appears.
DKMS rebuilds the module automatically for each new kernel.

Versions follow `<last mainline kernel>-oot.<n>`: `5.11-oot.1` is the first
out-of-tree release of the driver as it shipped in Linux 5.11.

To build by hand without DKMS: `make` (or `make KVER=<version>`), then
`sudo insmod skd.ko`.

## Changes from the v5.11 driver

Kernel API updates:

- `alloc_disk()` + `blk_mq_init_queue()` → `blk_mq_alloc_disk()`; queue
  limits passed through `struct queue_limits` (6.9+) and feature flags
  (`BLK_FEAT_WRITE_CACHE | BLK_FEAT_FUA`, 6.11+).
- Teardown: `blk_cleanup_queue()` and `GENHD_FL_UP` replaced by
  `disk_live()` / `del_gendisk()` / `put_disk()`.
- `device_add_disk()` return value is now checked.
- `->timeout` and tag-iterator callbacks lost their `reserved` argument.
- `blk_rq_map_sg()` takes no queue argument (6.15+).
- `BLK_MQ_F_SHOULD_MERGE` only set before 6.14 (merging is unconditional since).
- `->getgeo()` takes a `struct gendisk *` (6.18+).
- `from_timer()` → `timer_container_of()` (6.16+), `del_timer_sync()` →
  `timer_delete_sync()`.
- `<asm/unaligned.h>` → `<linux/unaligned.h>` (6.12+).
- `PCI_IRQ_LEGACY` → `PCI_IRQ_INTX`.
- `pci_{enable,disable}_pcie_error_reporting()` removed: the PCI core enables
  AER itself.

Bug fix carried over from upstream: `skd_inc_in_flight()` incremented the
local pointer instead of the counter, so `skd_in_flight()` always returned 0.
In-flight counts in debug messages are now correct. The `PAUSING → PAUSED`
transition now waits for outstanding requests to drain, as it was meant to.

## Authorship

This port was written by Ummon ([@ummon-technocore](https://github.com/ummon-technocore)),
an AI (Claude, by Anthropic), working with [@oxedions](https://github.com/oxedions),
who provided the hardware, the direction and the review. Everything listed as
tested above was run on a real S1120 card. Still, review it as you would any
other contribution before trusting it with your data.

## License

GPL-2.0-only, see `COPYING`. Copyright 2012 sTec, Inc.; Copyright (c) 2017
Western Digital Corporation or its affiliates; out-of-tree port maintained by
the repository authors.
