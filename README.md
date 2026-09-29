# openwrt-rootfs-expand

Grow the root filesystem of an OpenWrt / ImmortalWrt **ext4-combined** image to the whole disk — offline, interactive, safe.

[中文说明](README.zh-CN.md)

## What it does

An OpenWrt / ImmortalWrt `ext4-combined` image ships with a ~300 MB rootfs
partition. Flash it to a 120 GB SSD and ~119 GB sits unused. This script grows
that partition, and the ext4 filesystem inside it, to fill the whole disk — or
to any size you choose.

- Auto-detects disks and partitions. No hardcoded device names; NVMe / SATA / MMC all work.
- Backs up the partition table before touching it.
- Confirms before every destructive step. `--dry-run` prints the commands and changes nothing.
- Refuses to touch a mounted partition, or the running system's root filesystem.
- Only grows, never shrinks. Shrinking is what destroys data.
- Works with GPT and MBR.

## Requirements

- A live rescue system with e2fsprogs — [SystemRescue](https://www.system-rescue.org/) is the tested one. Run as root.
- Tools: `lsblk`, `blkid`, `parted`, `partprobe`, `sgdisk`, `e2fsck`, `resize2fs`, `dumpe2fs`, `blockdev` (all included in SystemRescue).
- The target disk must be **unmounted**.
- The rootfs partition must be **ext2/ext3/ext4**. Not for squashfs images.

## Quick start

```bash
chmod +x expand-rootfs.sh

# 1) See what it would do. Changes nothing.
./expand-rootfs.sh --dry-run

# 2) Fully interactive: pick disk -> pick partition -> pick size -> confirm each step
./expand-rootfs.sh

# 3) Non-interactive, once you know the target
./expand-rootfs.sh -d /dev/nvme0n1 -p 2 -y

# 4) Grow to 60 GiB instead of taking the whole disk
./expand-rootfs.sh --size 60G
```

### Options

| Option | Effect |
| --- | --- |
| `-d, --disk` | Target disk, e.g. `/dev/nvme0n1`, `/dev/sda` |
| `-p, --part` | Partition number, e.g. `2` |
| `-s, --size` | Target size (see below) |
| `-y, --yes` | Answer yes to every confirmation |
| `-n, --dry-run` | Print the commands, change nothing |
| `--no-backup` | Skip the partition-table backup (not recommended) |
| `-l, --log` | Log file path |
| `-h, --help` | Help |

### `--size` syntax

A unit is mandatory. A bare number is rejected on purpose: parted interprets a
bare number according to the current unit, which is far too easy to misread.

| Value | Actual size | Notes |
| --- | --- | --- |
| `100%` | whole disk | default |
| `60G` / `60GiB` | 60 GiB | short form `G` is 1024-based |
| `60GB` | 55.9 GiB | decimal; 4 GiB smaller than `60G` |
| `61440MiB` | 60 GiB | equivalent form |
| `45%` | 45% of the free space | |

Three hard limits, enforced by the script: it will not go below the current
partition size, it will not exceed the end of the disk, and it warns — offering
a non-overlapping cap — if the new size would overlap a later partition such as
the small `p128` reserved one.

Space left over stays unallocated, so you can grow again later by re-running
with `100%`.

## What it actually runs

```
Step 0  sfdisk -d / sgdisk -b            back up the partition table
Step 1  sgdisk -e                        move the GPT backup table to the end of the disk (skipped on MBR)
Step 2  parted resizepart + partprobe    extend the partition
Step 3  e2fsck -fy                       force a filesystem check
Step 4  resize2fs                        grow the filesystem
Step 5  lsblk + dumpe2fs                 verify, with a before/after comparison
Step 6  reboot                           then pull the USB stick
```

The start sector never moves. That is the only genuinely dangerous part of the
operation, and the script keeps it fixed.

Step 3 is not optional: skip `e2fsck` and `resize2fs` fails **silently** — no
error, no size change. `e2fsck` exit code 1 is normal (`FILE SYSTEM WAS
MODIFIED`); the script only aborts on code 4 (errors left uncorrected).

Sample run:

```
==> Partitions on /dev/nvme0n1:
nvme0n1      120G
nvme0n1p1     32M vfat
nvme0n1p2    300M ext4
nvme0n1p128  239K
Which partition number holds rootfs? [1 2 128] (default 2): 2
[OK] Target partition: /dev/nvme0n1p2
Filesystem: ext4
[OK] Safety checks passed: partition is unmounted and is not the running root.
...
==> Step 5/6  Verify
  partition size       : 300.0 MB -> 120.0 GB
  filesystem blocks    : 76800 -> 31449019 (block size 4096)
  filesystem size      : 300.0 MB -> 120.0 GB
[OK] Filesystem grew by 119.7 GB.
```

## Verify

Check the before/after block in Step 5 — all three lines must have changed.

Then `reboot` (pull the USB stick while the shutdown logs scroll, otherwise you
boot the rescue system again) and confirm inside OpenWrt:

```bash
df -h /overlay     # should be close to the disk size
```

## FAQ

| Symptom | Cause | Fix |
| --- | --- | --- |
| parted: `GPT PMBR size mismatch` | GPT backup table still sits at the image's end | Handled in Step 1. Manually: `sgdisk -e /dev/nvme0n1` |
| `resize2fs` ran but the size did not change | `e2fsck` was skipped | Always run `e2fsck -fy` first |
| `e2fsck` prints `FILE SYSTEM WAS MODIFIED` | Exit code 1 = minor errors corrected | Normal, not a failure. Only exit 4 aborts |
| Disk missing from `lsblk` | NVMe/SSD not detected | Reseat it, recreate the USB, check BIOS |
| USB will not boot | Secure Boot is on | Disable Secure Boot, pick the **UEFI** entry |
| Restore the partition table | — | `sfdisk /dev/nvme0n1 < partition-table-backup-xxx.sfdisk` |

## Re-run after every upgrade

`sysupgrade` rewrites the whole image and resets the rootfs back to ~300 MB.
Run this script again after upgrading.

## Notes

- Copy `partition-table-backup-*.sfdisk` / `.gpt` off the machine (to the USB stick). They are your rollback.
- Do not save the script with Windows Notepad — it writes CRLF, and Linux then fails with `bad interpreter`. `.gitattributes` pins `eol=lf`.

## License

MIT
