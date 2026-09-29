# openwrt-rootfs-expand

把 OpenWrt / ImmortalWrt 的 **ext4-combined** 镜像根分区离线扩展到整块盘——交互式、安全。

[English](README.md)

## 功能

OpenWrt / ImmortalWrt 的 `ext4-combined` 镜像自带一个约 300 MB 的根分区，写进 120 GB
固态后会有约 119 GB 闲置。本脚本把这个分区、以及分区里的 ext4 文件系统，扩展到占满整盘，
或扩展到任意你指定的大小。

- 自动探测磁盘与分区，不硬编码设备名，NVMe / SATA / MMC 通用。
- 动分区表前先备份。
- 每一步破坏性操作前都要求确认；`--dry-run` 只打印命令，不动盘。
- 拒绝操作已挂载的分区，也拒绝操作当前运行系统的根分区。
- 只扩不缩。缩容才是丢数据的操作。
- GPT 与 MBR 都支持。

## 环境要求

- 带 e2fsprogs 的 Live 救援系统，实测用 [SystemRescue](https://www.system-rescue.org/)，以 root 运行。
- 依赖工具：`lsblk`、`blkid`、`parted`、`partprobe`、`sgdisk`、`e2fsck`、`resize2fs`、`dumpe2fs`、`blockdev`（SystemRescue 已内置）。
- 目标盘必须**未挂载**。
- 根分区文件系统必须是 **ext2/ext3/ext4**。不适用于 squashfs 镜像。

## 快速上手

```bash
chmod +x expand-rootfs.sh

# 1) 先看会做什么，不做任何改动
./expand-rootfs.sh --dry-run

# 2) 全交互式：选盘 → 选分区 → 选大小 → 每步确认
./expand-rootfs.sh

# 3) 已经确认过环境，一键跑完
./expand-rootfs.sh -d /dev/nvme0n1 -p 2 -y

# 4) 只扩到 60 GiB，不吃满整盘
./expand-rootfs.sh --size 60G
```

### 参数

| 参数 | 作用 |
| --- | --- |
| `-d, --disk` | 指定磁盘，如 `/dev/nvme0n1`、`/dev/sda` |
| `-p, --part` | 指定分区号，如 `2` |
| `-s, --size` | 目标大小，见下节 |
| `-y, --yes` | 所有确认默认 yes，一键跑完 |
| `-n, --dry-run` | 只打印将要执行的命令，不动盘 |
| `--no-backup` | 跳过分区表备份（不建议） |
| `-l, --log` | 指定日志文件路径 |
| `-h, --help` | 查看帮助 |

### `--size` 怎么写

必须**带单位**。裸数字会被拒绝——parted 的裸数字含义取决于当前 unit，太容易误解。

| 写法 | 实际大小 | 说明 |
| --- | --- | --- |
| `100%` | 吃满整盘 | **默认值** |
| `60G` / `60GiB` | 60 GiB | 简写 `G` 按 1024 进制 |
| `60GB` | 55.9 GiB | 十进制，比 `60G` 小 4 GiB |
| `61440MiB` | 60 GiB | 等价写法 |
| `45%` | 盘可用空间的 45% | |

**三条硬性限制**（脚本会拦，不让继续）：

1. 不能比当前分区小——本脚本只扩不缩，缩容会丢数据。
2. 不能超过盘尾。注意 parted 按十进制 GB、`lsblk` 按 GiB 显示，别照抄数字。
3. 不能和后面的分区重叠。检测到会警告并给出不重叠的上限，例如 `p128` 那个小保留分区。

扩完剩下的空间是空闲未分配，**以后可以再扩**（改回 `100%` 重跑即可）。

## 实际执行了什么

```
Step 0  sfdisk -d / sgdisk -b            备份分区表
Step 1  sgdisk -e                        把 GPT 备份表挪到盘尾（MBR 盘自动跳过）
Step 2  parted resizepart + partprobe    扩展分区
Step 3  e2fsck -fy                       强制文件系统体检
Step 4  resize2fs                        撑满文件系统
Step 5  lsblk + dumpe2fs                 验证，打印扩容前后对比
Step 6  reboot                           然后拔 U 盘
```

**起始扇区全程不动**，这是整个操作里唯一真正危险的地方，脚本把它固定住。

Step 3 不是可选项：不先跑 `e2fsck`，`resize2fs` 会**静默失败**——不报错，大小也不变。
`e2fsck` 退出码 1 属正常（`FILE SYSTEM WAS MODIFIED`），只有退出码 4（有错未修）脚本才会中止。

## 怎么判断成功

看 Step 5 的对比，三项都对上才算成：

```
  partition size       : 300.0 MB -> 120.0 GB
  filesystem blocks    : 76800 -> 31449019 (block size 4096)
  filesystem size      : 300.0 MB -> 120.0 GB
[OK] Filesystem grew by 119.7 GB.
```

然后 `reboot`（屏幕开始刷日志时拔 U 盘，否则又会进救援系统），回到 OpenWrt 确认：

```bash
df -h /overlay     # 应接近硬盘实际大小
```

## 常见问题

| 现象 | 原因 | 怎么办 |
| --- | --- | --- |
| parted 报 `GPT PMBR size mismatch` | GPT 备份表没挪到盘尾 | Step 1 已处理；手动补 `sgdisk -e /dev/nvme0n1` |
| `resize2fs` 跑完大小没变 | 没先跑 `e2fsck`，被静默拒绝 | 严格按 `e2fsck -fy` → `resize2fs` 顺序 |
| `e2fsck` 打印 `FILE SYSTEM WAS MODIFIED` | 退出码 1 = 已修正小错误 | **不是故障**；只有 `exit 4` 脚本才会中止 |
| `lsblk` 里看不到硬盘 | 固态没识别 | 重插 / 重做启动盘 / 进 BIOS 确认没被关 |
| U 盘启动不了 | Secure Boot 没关 | BIOS 里关掉，启动菜单选带 **UEFI** 的 U 盘 |
| 想还原分区表 | — | `sfdisk /dev/nvme0n1 < partition-table-backup-xxx.sfdisk` |

## 每次升级都要重跑

`sysupgrade` 会重写整盘镜像，把根分区打回约 300 MB。升级后重跑一次本脚本即可。

## 两点提醒

- **备份文件要拷走**：`partition-table-backup-*.sfdisk` / `.gpt`，把两个都拷到 U 盘，这是你的回滚手段。
- **别用 Windows 记事本另存**：会把行尾变成 CRLF，Linux 下报 `bad interpreter`。仓库里 `.gitattributes` 已锁定 `eol=lf`。

## 实际运行示例

在 SystemRescue 里跑一遍的真实输出：

```shell
[root@sysrescue ~]# ./expand-rootfs.sh
Log file: ./expand-rootfs-20260927-040027.log

==> Detected disks:
  1) /dev/nvme0n1      120G     nvme    VMware Virtual NVMe Disk
Select the disk holding ImmortalWrt [1-1]: 1
[OK] Target disk: /dev/nvme0n1 ( 120G, model: VMware Virtual NVMe Disk)
Partition table type: gpt

==> Partitions on /dev/nvme0n1:
nvme0n1      120G
nvme0n1p1     32M vfat
nvme0n1p2    300M ext4
nvme0n1p128  239K
Which partition number holds rootfs? [1 2 128] (default 2): 2
[OK] Target partition: /dev/nvme0n1p2
Filesystem: ext4
[OK] Safety checks passed: partition is unmounted and is not the running root.

==> Current layout
  disk            : /dev/nvme0n1 (120.0 GB)
  partition       : /dev/nvme0n1p2  #2
  start sector    : 66048   <- must stay unchanged
  end sector      : 680447   (partition size 300.0 MB)
  filesystem size : 300.0 MB

==> How far should the partition grow?
  1) 100%  - use all remaining space (default; parted may warn about overlap)
  3) custom - type a size yourself, e.g. 60G / 40960M
Choose [1]: 1
[OK] Target: 100% (~120.0 GiB, end sector 251658206)

==> Plan
  disk        : /dev/nvme0n1
  partition   : /dev/nvme0n1p2 (#2), filesystem ext4
  grow to     : 100%
  steps       : backup partition table -> sgdisk -e -> parted resizepart
                -> partprobe -> e2fsck -fy -> resize2fs -> verify
  dry-run     : no
Proceed? (data on /dev/nvme0n1p2 will NOT be erased, but always keep a backup) [y/N] y

==> Step 0/6  Back up the partition table
[OK] Saved: ./partition-table-backup-nvme0n1-20260927-040038.sfdisk + ./partition-table-backup-nvme0n1-20260927-040038.gpt
Copy these files off the machine (to the USB stick) before rebooting.

==> Step 1/6  Move the GPT backup table to the end of the disk
Run: sgdisk -e /dev/nvme0n1 [y/N] y
$ sgdisk -e /dev/nvme0n1
The operation has completed successfully.
[OK] GPT backup table relocated.

==> Step 2/6  Extend partition #2 to 100%
Run: parted -s /dev/nvme0n1 resizepart 2 100% [y/N] y
$ parted -s /dev/nvme0n1 resizepart 2 100%
$ partprobe /dev/nvme0n1
$ lsblk /dev/nvme0n1
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0 120G  0 disk
├─nvme0n1p1   259:4    0  32M  0 part
├─nvme0n1p2   259:5    0 120G  0 part
└─nvme0n1p128 259:6    0 239K  0 part
[OK] Partition extended.

==> Step 3/6  Force a filesystem check (e2fsck -fy)
Run: e2fsck -fy /dev/nvme0n1p2 [y/N] y
$ e2fsck -fy /dev/nvme0n1p2
e2fsck 1.47.4 (6-Mar-2025)
Pass 1: Checking inodes, blocks, and sizes
...
rootfs: ***** FILE SYSTEM WAS MODIFIED *****
rootfs: 1557/19200 files (0.0% non-contiguous), 12005/76800 blocks
[OK] e2fsck repaired minor inconsistencies (exit 1). This is NORMAL, not a failure.

==> Step 4/6  Grow the filesystem (resize2fs)
Run: resize2fs /dev/nvme0n1p2 [y/N] y
$ resize2fs /dev/nvme0n1p2
resize2fs 1.47.4 (6-Mar-2025)
Resizing the filesystem on /dev/nvme0n1p2 to 31449019 (4k) blocks.
The filesystem on /dev/nvme0n1p2 is now 31449019 (4k) blocks long.
[OK] Filesystem grown.

==> Step 5/6  Verify
  partition end sector : 680447 -> 251658206
  partition size       : 300.0 MB -> 120.0 GB
  filesystem blocks    : 76800 -> 31449019 (block size 4096)
  filesystem size      : 300.0 MB -> 120.0 GB
[OK] Filesystem grew by 119.7 GB.

==> Step 6/6  Next
  1. Run: reboot
  2. Pull the USB stick while the shutdown logs scroll (otherwise you boot the rescue system again).
  3. Back in ImmortalWrt: df -h /overlay   -> should be close to the disk size.

Remember: every sysupgrade rewrites the whole image and resets the partition to its
original size. Just run this script again after upgrading.
[root@sysrescue ~]# reboot
```

## 许可

MIT
