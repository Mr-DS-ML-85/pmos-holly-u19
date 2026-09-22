# Huawei Honor Holly — postmarketOS Port

🚧 Active development has moved to my Gitea repository.

👉 **[Continue development on Gitea](https://git.furylogic.com/Mr-DS-ML-85/pmos-holly-u19)**

Mainline Linux kernel port for the Huawei Honor Holly (Hol-U19) / Honor 3C Lite,
a MediaTek MT6582 quad-core Cortex-A7 smartphone.

## Device Specs

| Component | Details |
|-----------|---------|
| SoC | MediaTek MT6582 (4× Cortex-A7 @ 1.3 GHz) |
| GPU | Mali-400 MP2 |
| RAM | 1 GB (960 MB + 18 MB banks) |
| Display | 720×1280 IPS LCD, 16bpp RGB565 framebuffer, MIPI-DSI |
| Touch | Goodix GT9157 (I2C0 @ 0x5D, INT=GPIO5, RST=GPIO8, AVDD/VDDIO=MT6323 VGP1 @ 2.8 V) |
| Storage | 4 GB eMMC + microSD |
| WiFi/BT | MediaTek MT6628 combo chip |
| Modem | MediaTek MT6260 (WCDMA/GSM) |
| Kernel | Mainline Linux 7.2.2 (custom) |

## Project Status

| Feature | Status | Notes |
|---------|--------|-------|
| Boot | ✅ Works | MTK boot image format, WDT disabled |
| pmOS rootfs | ✅ Boots | patched rootfs (sshd + fixed `/boot` fstab) physically on `mmcblk0p5` |
| SSH | ✅ Works | OpenSSH on `172.16.42.1:22`, login `user`/`12345678` |
| Serial console | ✅ Works | UART0 @ 921600 baud + **USB serial gadget** (CDC ACM, `/dev/ttyACM0`) |
| Decompressor display | ✅ Works | Green RGB565 VGA text via direct write |
| Kernel framebuffer | ✅ Works | simplefb + fbcon, correct RGB565 colors, no overlap |
| USB MUSB gadget | ✅ Works | IRQ `GIC_SPI 32` + VBUS comparator; phone enumerates & console reachable over USB |
| **BusyBox shell** | ✅ Works | Static ARM BusyBox 1.37.0, `holly#` prompt, **concurrent** USB serial + phone LCD consoles |
| eMMC | ✅ Works | `mmcblk0` enumerated (p1-p7 + boot0/boot1); `/init` probes each partition for a bootable root |
| Plasma desktop | ✅ Boots | full Plasma session (slow first paint — 4× A7) |
| GPU (lima) | ⚠️ Probes, unstable | Mali-400 MP2 initializes; job-timeout + IRQ-disabled on first real load |
| USB networking + internet | ✅ Works | gadget 172.16.42.1, **persistent host-side NAT** (udev rule) |
| Clock | ✅ NTP (no HW RTC) | date correct once online; `rtc-mt6397` blocked on PMIC IRQ |
| Remote control | ✅ Works | KRDP RDP server (port 3390) as touchscreen bypass — needs an H.264-capable client |
| WiFi/BT | ❌ Not started | MT6627N/MT6628 WMT driver needed |
| Touchscreen | ❌ **HW dead** | GT9xx IC register file/CPU dead (bus-echo signature); driver complete & waiting for healthy panel |
| Cellular | ❌ Not started | CCCI + ModemManager |
| Audio | ❌ Not started | ALSA + modem speech |
| Camera | ❌ Not started | OV5648 + ISP pipeline |

## What Works

- **Kernel boots** successfully from eMMC (stock LK bootloader handles early init)
- **Decompressor framebuffer** — green VGA 8×16 text rendered at boot (RGB565)
- **Kernel console** — correct white-on-black text via simplefb + fbcon (RGB565)
- **Serial console** — full kernel log output on UART0 (921600, 8N1)
- **USB serial gadget console** — phone reaches a **BusyBox** `holly#` shell over
  `/dev/ttyACM0` on the host (device has a dead battery / no usable screen, so this
  is the primary interactive interface)
- **Phone LCD shell** — a **second, concurrent** BusyBox shell on `/dev/tty0`
  (framebuffer console) so the display also shows the prompt
- **Initramfs shell** — static ARM BusyBox 1.37.0 (403 applets: `ls`, `grep`, `ps`,
  `mount`, `md5sum`, `zcat`, `dd`, `hexdump`, ...); init (PID 1) supervises and
  respawns either shell if it exits
- **Fastboot flash** — `fastboot flash boot` works
- **eMMC** — `mmcblk0` (p1-p7 + boot0/boot1) enumerates via MSDC0; `/init` probes each
  partition read-only and only pivots into a root that has a real `/init` (none on the
  stock Android partitions, so it drops to the shell)
- **pmOS rootfs on `mmcblk0p5`** — **physically flashed via an on-device `nc` stream**
  (md5 `a2c27b9a...` verified on-device), boots pmOS with **SSH** (OpenSSH_10.3) on
  `172.16.42.1:22`; the `/boot` fstab boot error is gone
- **Telnet recovery shell** — `pmos-shellet.img` (patched pmOS initramfs) always drops
  to a **root shell** on `172.16.42.1:23` + USB serial, for on-device recovery
- **MT6323 PMIC** — pwrap → `mt6397` MFD → `mt6323-regulator` + `reboot-mode`
- **mtkclient BROM flash** — fallback flash method

## PMOS over SSH (current working access)

After flashing the patched rootfs to `mmcblk0p5`, the phone boots pmOS with SSH enabled:

```bash
# SSH login (hostname huawei-holly, uid 10000)
sshpass -p 12345678 ssh -o StrictHostKeyChecking=no user@172.16.42.1
```

- **User:** `user` (uid 10000, groups wheel/audio/video/netdev/plugdev)
- **Password:** `12345678` (the shadow hash decodes to `12345678`, **not** the default `147147`)
- `/etc/fstab` is patched — only the root `UUID=a1a1f616-... / ext4 ...` line remains (no bogus
  `/boot` entry), so the boot no longer shows an fstab mount error.
- Need to re-flash or re-stage a rootfs? Use the **telnet recovery shell** (below) and stream
  on-device with `nc` — this is the only method that physically lands on eMMC.

## Telnet Recovery Shell (`pmos-shellet.img`)

Mass-storage `/dev/sda` writes and large `fastboot flash system` do **not** persist on this
device's eMMC; the reliable path is an **on-device shell writing to `/dev/mmcblk0p5`**.

`pmos-shellet.img` is a patched pmOS initramfs that **always drops to a root shell** (never
boots pmOS): USB-serial getty on `ttyGS0` + `telnetd` on `172.16.42.1:23`.

```bash
fastboot flash boot pmos-shellet.img && fastboot reboot
# on the host, connect to the telnet root shell:
nc 172.16.42.1 23
```

Inside the shell, write a rootfs to the physical eMMC (proven method):
```sh
# phone side: start a detached listener
setsid sh -c 'nc -l -p 9999 > /dev/mmcblk0p5' </dev/null >/dev/null 2>&1 &
```
```bash
# host side: stream the rootfs (697 MB over USB net, ~3.5 MB/s, ~197 s)
nc 172.16.42.1 9999 < out/artifacts/huawei-holly-root.img
```
```sh
# phone side: verify on-device (reads physical device)
#   dd if=/dev/mmcblk0p5 bs=1 skip=$((0x438)) count=2 | xxd   # expect: 53 ef
#   blkid /dev/mmcblk0p5    # LABEL="pmOS_root" UUID=a1a1f616-... TYPE="ext4"
#   dd if=/dev/mmcblk0p5 bs=128K count=5320 | md5sum          # expect a2c27b9a896e4cc2dc68c9ec341d27e7
```

### Do NOT re-try these (failed; see `engineering.md` §37)
- **`fastboot flash system` of a large (> 16 MiB) rootfs** — MTK LK `download_max` overflow +
  `FILL` chunk bug; **chunked/partial writes also fail**. Fastboot here only loads small images
  (≤ 16 MiB boot.img). `fastboot erase system` **hangs**. (Only a 4 KB `flash system` is useful —
  to *clear* p5 so the recovery shell drops in.)
- **USB Mass-Storage gadget (`/dev/sda`)** — writes read back correctly on the host but **never
  physically reach eMMC** (flushed only to a cached view). Do not trust `/dev/sda` read-back.
- **USB internet/ECM and USB HID gadgets** — ECM/net ping works but gave no reliable shell;
  HID carries no block/general I/O. Neither can land a rootfs.
- **Custom dev initramfs** (`build_boot.sh`/`init.c`) — pivots into pmOS whenever p5 has a valid
  root (no shell), provides no telnet, and is unstable. Use the patched **pmOS** initramfs instead.

## SSH access to the UNPATCHED pmOS image
If p5 already has a pmOS rootfs but SSH isn't on, you don't need the whole §36 flow. Boot the
`pmos-shellet.img` **telnet shell** (§37.6 of engineering.md) and on the rootfs make two edits,
then reboot normally:
1. `ln -s /etc/init.d/sshd /etc/runlevels/default/sshd` (enable sshd at boot).
2. Fix `/etc/fstab` — remove the bogus `/boot` line (UUID `ce17d9d1-...`), keep
   `UUID=a1a1f616-... / ext4 defaults 0 0`.
Login is `user` with the pmOS password (this image decodes the shadow hash to **`12345678`**).

## Quick Start

### Prerequisites

- ARM cross-compiler (`arm-none-linux-gnueabihf-gcc`)
- Fastboot or mtkclient for flashing
- USB-to-serial adapter for UART console (optional but recommended)

> **⚠️ `build_boot.sh` and `build_pmos.sh` are DEPRECATED.** Both pack the
> **custom dev initramfs** (`tools/initramfs/init.c`) instead of the real
> postmarketOS one. Use **`build_pmos.py`** below — it compiles the kernel with
> pmbootstrap and packs the **real pmOS initramfs** in MTK boot format.

### Build & Flash (current — `build_pmos.py`)

`build_pmos.py` is the one build entry point. It:
1. Builds the kernel from the **local, tuned tree** (`out/linux-7.2.2`) via
   `pmbootstrap build --src` — no kernel.org download, the tree is compiled
   as-is inside the pmbootstrap chroot.
2. Takes the **real postmarketOS initramfs** (from the rootfs chroot,
   `~/.local/var/pmbootstrap/chroot_rootfs_huawei-holly/boot/initramfs` — the
   same one pmOS installs). Trimming is **OPT-IN**: set `PMOS_TRIM_INITRAMFS=1`
   to drop non-boot files (btrfs/xfs/parted/X11…) so kernel+initramfs fit the
   16 MiB boot partition. Without it the untrimmed initramfs is packed as-is
   and the 16 MiB check fails loudly instead of silently dropping files.
3. Packs the **MTK boot image** (verified `ANDROID!` + `KERNEL`/`ROOTFS`
   headers per `engineering.md` §31) and optionally flashes it.

The old custom initramfs (`tools/initramfs/`, busybox shell) is **never** used
by this script.

```bash
python3 build_pmos.py                 # build kernel + pack (no flash)
python3 build_pmos.py --flash         # build + pack + fastboot flash boot
python3 build_pmos.py --no-build      # skip kernel rebuild (reuse newest apk)
PMOS_TRIM_INITRAMFS=1 python3 build_pmos.py   # OPT-IN trim to fit 16 MiB boot
python3 build_pmos.py --out my.img    # custom output path
```

Prerequisites: the repo venv (`pm-venv`, editable pmbootstrap install — refresh
with `uv pip install -e pmbootstrap-src/pmbootstrap`) and a configured
pmbootstrap work dir. Overrides: `PMB`, `APORTS`, `SRC_TREE`, `PMOS_ROOT`,
`KERNEL_APK`, `INITRAMFS`.

### Legacy build (deprecated — custom initramfs)

```bash
# Old flow — packs the CUSTOM dev initramfs; kept only for diagnostics:
bash build_boot.sh                    # or: BUSYBOX_INSTALL_DIR=/tmp/bb_install bash build_boot.sh
```

### Flash Methods

```bash
# Fastboot (preferred)
adb reboot bootloader
fastboot flash boot out/artifacts/boot-holly-pmos.img
fastboot reboot

# mtkclient BROM (fallback, no adb needed)
python mtkclient/mtk.py wo 0x1d80000 \
    $(stat -c%s out/artifacts/boot-holly-pmos.img) \
    out/artifacts/boot-holly-pmos.img
```

### Serial Console

There are two ways to get a console:

**USB serial gadget (primary, no extra hardware)** — the phone enumerates as a CDC
ACM gadget (`1d6b:0104`) and the initramfs redirects its console/shell onto
`/dev/ttyGS0`. On the host, load the module and connect:

```bash
sudo modprobe cdc_acm            # once
screen /dev/ttyACM0 115200       # or minicom -D /dev/ttyACM0
```

**Physical UART** — connect a USB-to-serial adapter to UART0 pins:

| Pin | Function |
|-----|----------|
| TX | UART0 TX (connect to adapter RX) |
| RX | UART0 RX (connect to adapter TX) |
| GND | Ground |

```bash
screen /dev/ttyUSB0 921600
```

## Initramfs Shell

When no rootfs is found, the initramfs drops to a **BusyBox** shell. It runs in two
places concurrently:
- **USB serial gadget** — `/dev/ttyACM0` on the host (`screen /dev/ttyACM0 115200`)
- **Phone LCD** — `/dev/tty0` framebuffer console

The prompt is `holly#`. `init` (PID 1) is a fork-supervisor: it spawns both shells and
respawns one if it exits, so a dying shell can never panic the kernel. BusyBox is
statically linked (no dynamic deps) and provides all standard applets via symlinks:

| Command | Description |
|---------|-------------|
| `help` | List all 403 BusyBox applets |
| `ls /dev /proc /sys` | Explore devices, proc, and sysfs |
| `mount` / `cat /proc/partitions` | Block devices — `mmcblk0` (p1-p7) now enumerates |
| `cat /proc/iomem` | Physical memory map |
| `cat /proc/interrupts` | IRQ lines (USB0 = IRQ 64, was wrong pre-fix) |
| `dmesg` / `dmesg \| grep mtk-sd` | Kernel message search |
| `md5sum`, `zcat`, `unzip`, `dd`, `hexdump`, `ps`, `grep` | Standard tools |
| `reboot` / `poweroff` | Device control |
| `exit` | Kill current shell — init respawns it |

### Building BusyBox

```bash
# Cross-compile static ARM BusyBox from Ubuntu source (busybox.net avoided)
apt-get source busybox                      # needs deb-src entries
cd busybox-1.37.0
make ARCH=arm CROSS_COMPILE=/usr/bin/arm-linux-gnueabihf- defconfig
sed -i 's/^# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make -j$(nproc)                             # static, stripped, no NEEDED entries
make CONFIG_PREFIX=/tmp/bb_install install  # 403 applet symlinks, /bin/sh -> busybox

# Then build + flash the boot image with BusyBox:
BUSYBOX_INSTALL_DIR=/tmp/bb_install bash build_boot.sh
```

## Hardware Map

```
SoC Peripherals (from stock /proc/iomem):

0x80000000 - 0xBBFFFFFF  System RAM (960 MB)
0xBD800000 - 0xBE9FFFFF  System RAM (18 MB)
0xBEb00000 - 0xBFFFFFFF  Framebuffer (64 MB reserved; OVL scans RGB565)

Display Pipeline (INFRA window):
0xF4000000               MMSYS config
0xF4007000               OVL (overlay engine)
0xF4008000               RDMA (read DMA)
0xF400C000               DSI (MIPI)
0xF400E000               MM Mutex

Storage:
0x11230000               MSDC0 (eMMC)
0x11240000               MSDC1 (SD card)

Connectivity:
MT6628                   WiFi + BT + GPS combo chip
```

## Repository Structure

```
├── build_pmos.py              # **Current build tool**: pmbootstrap kernel + real pmOS
│                             #   initramfs, MTK-format pack + flash
├── build_boot.sh              # DEPRECATED — custom dev initramfs packer (diagnostics only)
├── build_pmos.sh              # DEPRECATED — same custom initramfs, pmbootstrap artifacts
├── README.md                  # This file
├── out/
│   ├── linux-7.2.2/           # Mainline kernel tree
│   │   ├── arch/arm/boot/dts/mediatek/
│   │   │   └── mt6582-huawei-holly.dts    # Board device tree
│   │   ├── drivers/video/fbdev/
│   │   │   └── early_fbcon.c              # Framebuffer console driver
│   │   ├── init/main.c                    # Kernel init + WDT disable
│   │   └── .config                        # Kernel configuration
│   └── artifacts/
│       └── boot-holly-pmos.img            # Final boot image
├── tools/
│   ├── fbpan.c                 # FBIOPAN_DISPLAY utility
│   └── initramfs/
│       ├── init.c              # Initramfs init (BusyBox shell supervisor, USB+LCD consoles)
│       └── extlib/             # libz.a + zlib headers for static init link
├── bb_install/                 # BusyBox applet tree dropped into the ramdisk (gitignored)
├── stock-dumps/                # Stock ROM dumps
│   ├── iomem.txt               # Physical memory map
│   ├── partitions.txt          # eMMC partition layout
│   ├── fb_vs.txt               # Framebuffer virtual size
│   ├── fb_stride.txt           # Framebuffer stride
│   └── fb_bpp.txt              # Framebuffer bits per pixel
├── RE/
│   └── system_hals/            # Reverse-engineered HAL binaries
│       ├── lib_hw/
│       │   ├── gralloc.mt6582.so         # ION framebuffer allocator
│       │   └── hwcomposer.mt6582.so      # OVL display composer
│       └── bin/
│           ├── lcdc_screen_cap            # Screen capture tool
│           └── touch                      # Touch input daemon
├── archive/
│   └── scripts/                # Archived scripts (BROM flash, dump, etc.)
│       ├── brom_dump.sh        # BROM auto-detector + stock dump
│       ├── flash_brom.sh       # BROM flash retry loop
│       ├── flash_holly.sh      # Autonomous BROM flash runner
│       ├── flash_retry.py      # Emergency flash via mtkclient
│       ├── dump_stock.sh       # Stock ROM dumper + scatter gen
│       ├── handover.md         # Previous handoff document
│       └── mt6582_display_port_plan.md  # DRM display port plan
└── sources/
    └── linux-huawei-h30t00/    # Vendor kernel 3.4.67 reference
```

## Known Issues

### Touchscreen (FINAL: hardware fault — IC dead, driver complete)
Stock runs the Goodix "GT910/guitar" GT9xx driver (vendor kernel 3.4) with a
**driver-side config upload**: on probe it powers the touch rail, resets the chip,
verifies the HW-info magic (reg `0x4220` == `0x00900600`), reads the PID/version
(reg `0x8140`), pushes a validated 186-byte config bank to `0x8047` (recomputed
checksum + Config_Fresh), and only then enables the IRQ. Mainline `goodix.c`
contains none of that boot sequence (no HW-info gate, no driver-side config for
this chip) — it is a port of a *different* codebase and does not bring this panel
up.

The vendor sequence was ported to 7.x APIs into a dedicated driver instead of
copying the 3.4 source over:
- **`out/linux-7.2.2/drivers/input/touchscreen/goodix_gt9xx.c`**
  (`CONFIG_TOUCHSCREEN_GOODIX_GT9XX=y`, Kconfig/Makefile wired) — vendor
  GT9xx probe/power/config/interrupt protocol, rewritten on top of modern APIs:
  `.probe(struct i2c_client *)` + void `.remove` (post-6.3 i2c), devres
  everywhere, `gpiod` descriptors, standard regulator consumers,
  `dev_pm_ops` suspend/resume, threaded IRQ, `sysfs_emit`, and
  **I2C read chunking** (the MT6582 controller caps the read phase of a combined
  transaction at 31 bytes — `max_comb_2nd_msg_len` — so the vendor's 34-byte
  5-finger read would fail on mainline; this driver never reads more than 16
  bytes per transaction, writes stay whole at ≤188 bytes which fits the 255-byte
  write limit).
- **DTS** (`mt6582-huawei-holly.dts`): node renamed to `goodix,gt9157`, IRQ
  switched to `IRQ_TYPE_EDGE_FALLING` (vendor `GTP_INT_TRIGGER=1`), supplies
  still `mt6323_vgp1_reg` at its stock **2.8 V** (kk product header:
  `TPD_POWER_SOURCE_CUSTOM = MT6323_POWER_LDO_VGP1` with the driver's
  `hwPowerOn(..., VOL_2800, "TP")`; no DT voltage override needed — the
  mt6323 regulator default is 2.8 V).
- **Update-mode poke revive** (in the driver): if hw-info reads zeros
  (unbooted register bank), the driver performs the on-device-proven
  revive — address-select reset + pure write-write poke
  (`0x4180 = 0x0C`, `0x4010 = 0x00`, no readback) + hardware reset —
  then re-judges; only a chip still misreporting reaches the flash
  burn. The vendor's readback-confirmed hold is retried but its
  failure is non-fatal on this board (held-state reads are
  untrustworthy; RE notes §1.6/§1.7).
- Board config bank = stock `huawei82_cwet_kk CTP_CFG_GROUP1` (186 B,
  720×1280, 5 contacts), embedded in the driver and patchable via the DT
  property `goodix,cfg-group1` (186-byte array) without a rebuild.

What was **deliberately not carried over** from the vendor 3.4 source: MTK tpd
framework + `hwPowerOn`/`hwPowerDown` + `mt_set_gpio_*` + `mt_eint_*`, Android
`early_suspend`, the DMA I2C path (mainline `i2c-mt65xx` handles the transfer
sizes natively), the procfs debug node, the proximity/hwmsen and virtual-key
hooks (disabled in stock anyway), and the GT9xxF flashless firmware updater
(this panel is a flash-carrying GT9xx).

Old pre-driver state (for reference): the mainline `goodix.c` probe reported
`ID , version: 0000` and `Invalid config (719, 1279, 0), using defaults` — the
chip ACKed at 0x5D but never answered any register bank until the write-write
update-mode poke was proven to revive it (RE notes §1.6/§1.7).

**Final on-device verdict (2026-09-16/21, engineering.md §38):** even with the complete
vendor protocol running (power, reset, poke revive, ISP flash-burn of stock GT9157
firmware, 186-byte board config), the chip keeps returning `ID 0000` and every burn
fails (`hold ss51+dsp FAILED after 201 tries`). A raw I2C-dev forensic scan produced a
**conclusive address-echo signature**: reads of `0x4010` → `0x10`, `0x5094` → `0x94`,
and a shifted read of `0x400f` → `0x0f 00 00 00` show the I2C shift register replaying
the received address byte with no register-file pointer behavior. The chip's **serial
engine is alive but its register file and CPU are dead** — a hardware fault inside the
GT9xx on the display/digitizer flex. No software can fix it. Remedies: a shop
flex-connector reseat, or glass/panel replacement (the GT9xx lives on the display PCB,
not in the MT6582). **When a healthy panel is fitted, the driver needs zero changes** —
it shows `ID 9157` on the first read.

The dead-touch workaround: the display itself works fine — Plasma renders the wallpaper,
clock and notification popups smoothly on the panel — but **touch input is dead** (the IC
fault above), so the phone is controlled from a PC over **RDP** (KRDP,
port 3390, `user`/`12345678`). KRDP only streams H.264, so the client must have an
H.264 decoder — Flatpak FreeRDP, Thincast, Windows `mstsc` work; Ubuntu-repo
`xfreerdp3`/Remmina/GNOME Connections are built `WITH_GFX_H264=OFF` and fail at the
GFX caps exchange. Details in engineering.md §38.2. Host-side USB NAT is persistent
via udev (engineering.md §39), and internet time sync covers the absent hardware RTC.

### Display (solved): RGB565 scanning
The panel's OVL scans the framebuffer as **16-bit RGB565**, even though stock
mtkfb exposed 32bpp ARGB8888 to userspace. Writing 32-bit ARGB caused every
color to be misread (green bytes `0x00 FF 00 00` decoded as RGB565 word
`0xFF00` = red+green = yellow) and stretched each pixel across two display
pixels, producing overlapping/doubled text.

Fixed by emitting true 16-bit RGB565 pixels:
- `misc.c` decompressor now writes RGB565 (green `0x07E0`, black `0x0000`),
  1440 bytes/row (720 u16 per line)
- DTS simplefb changed to `format = "r5g6b5"` with `stride = <1440>` so
  fbcon renders 16-bit RGB565 as well
- initramfs `init.c` configures the OVL for RGB565 (`CLRFMT=0`) with pitch
  1440, and all solid-color diagnostics write RGB565
- `CONFIG_EARLY_FBCON` remains **disabled**; boot uses the default
  simplefb + fbcon path

### No eMMC Detection → SOLVED
The block root cause: `mt6397_probe` bailed on `platform_get_irq_optional` returning
`-ENXIO` (the MT6582 pwrap IRQ isn't wired in mainline), so the MT6323 PMIC MFD never
created the `ldo_vmc`/`ldo_vemc3v3` regulators that MSDC0's `vmmc`/`vqmmc` phandles
need. Fix: treat non-positive IRQ as "continue without PMIC interrupt support".
With regulators up, `mmcblk0` enumerates (`p1..p7` + `boot0/boot1`) and `/init` mounts
the ext4 rootfs from `/dev/mmcblk0p5`. Full write-up: `pmic_crack.md` + `emmc_fix.md`.

### No Virtual Keyboard (partially solved)
The initramfs has no on-screen keyboard. Input is via the **USB serial console**
(`screen /dev/ttyACM0 115200`) or UART serial console, and the **phone LCD shell**
shows the display output on `/dev/tty0`.

### Boot-time service errors (nftables, zram, bluetooth) — SOLVED
When booting the newest `pmos.img` (a `build_pmos.py --out` run, newer zImage than
`out/artifacts/` — same DTB + initramfs, only the zImage differs between builds), the
stock pmOS `init` scripts emit three non-fatal errors. Root causes and on-device fixes:

1. **`nftables: Could not process rule: No such file or directory`** at
   `/etc/nftables.nft:22` (`tcp dport 113 reject with icmpx type port-unreachable`).
   The kernel lacks the nf_tables `reject` expression: `CONFIG_NFT_REJECT` /
   `CONFIG_NFT_REJECT_INET` are not set (only `NF_TABLES_INET`, `NFT_CT`, `NFT_META`).
   Both `icmpx` and plain `icmp` reject fail against the running kernel.
   On-device fix: `sed -i 's/reject with icmpx type port-unreachable/drop/' /etc/nftables.nft`
   → `nft -c -f` now validates and the service starts. Proper fix for later builds:
   enable `CONFIG_NFT_REJECT=y` + `CONFIG_NFT_REJECT_INET=y` in
   `aports/device/testing/linux-huawei-holly/linux-huawei-holly.config`.

2. **`postmarketos-zram-swap failed to start`** — two independent causes:
   - `modprobe zram` returns 1 because `CONFIG_ZRAM=y` (built-in, no `.ko`), and
     `/lib/modules/.../modules.builtin` never lists it, so the script's `set -e`
     dies right there even though `/dev/zram0` already exists.
   - `zramctl -a zstd` then fails with `Invalid argument`: the running kernel's zram
     only compiled the **lzo / lzo-rle** backends (`/sys/block/zram0/comp_algorithm`
     → `[lzo-rle] lzo`) even though the crypto API exposes `zstd`.
   On-device fix: append `deviceinfo_zram_swap_algo="lzo-rle"` to `/etc/deviceinfo`
   (the clean override point — keep `zramstart` untouched). Result:
   `ZRAM swap device /dev/zram1 activated using lzo-rle and size 1458 MB`, `free -m`
   shows `Swap: 1458`. Proper fix for later builds: enable
   `CONFIG_ZRAM_BACKEND_ZSTD=y` + `CONFIG_ZRAM_DEF_COMP_ZSTD=y`.

3. **`skb_under_panic` BUG (net/core/skbuff.c:214) killing bluetoothd** — a kernel
   oops inside `net/bluetooth` (modules `bluetooth ecdh_generic ecc kpp`) during
   bluetoothd's HCI mgmt socket setup: `data` pointer before `head` (`head:c48c5188
   data:c48c517a`). BT on this SoC is on-die CONSYS with **no mainline driver**
   (`CONFIG_BT_MTKSDIO`/`CONFIG_BT_MTKUART` unset, see `hal_hw_specs/bt_oss_spec.md`),
   so the module has nothing valid to talk to. The BUG only kills bluetoothd — boot
   continues (tinydm ok). On-device fix: `rc-update del bluetooth default` (and
   `rc-service bluetooth stop`). Keep so long as `CONFIG_BT` is a module with no
   transport driver.

4. **`udevd: specified group 'i2c' unknown`** — cosmetic; the `60-autosuspend.rules`
   `SUBSYSTEM=="i2c-dev" GROUP="i2c"` rule references a group pmOS doesn't define.
   On-device fix: `echo 'i2c:x:82:' >> /etc/group`.

## Flash Commands

| Method | Requirements | Notes |
|--------|-------------|-------|
| Fastboot | USB, unlocked bootloader | Preferred method |
| mtkclient BROM | USB, mtkclient | Works without bootloader unlock |
| UART dump | Serial adapter | For reading kernel logs |

## Kernel Configuration Highlights

Key config options enabled:
- `CONFIG_SYSFB_SIMPLEFB=y` — Simple-framebuffer support
- `CONFIG_FB_SIMPLE=y` — Simple framebuffer driver
- `CONFIG_FRAMEBUFFER_CONSOLE=y` — fbcon for on-screen console
- `CONFIG_SERIAL_MTK_UART=y` — MediaTek UART driver
- `CONFIG_USB_MUSB_MEDIATEK=y` — MT6582 MUSB controller
- `CONFIG_USB_CONFIGFS=y` + `CONFIG_USB_F_ACM=y` — configfs gadget w/ ACM (USB console)
- `CONFIG_TOUCHSCREEN_GOODIX_GT9XX=y` — vendor-protocol GT9xx touchscreen (Holly panel;
  mainline `CONFIG_TOUCHSCREEN_GOODIX` stays enabled and matches nothing on this board)
- `# CONFIG_USB_G_SERIAL is not set` — legacy g_serial disabled (was stealing the UDC)

Recommended additions (fix boot errors, not yet in config):
- `CONFIG_NFT_REJECT=y` + `CONFIG_NFT_REJECT_INET=y` — lets the stock
  `/etc/nftables.nft` load (no more `reject with icmpx` error)
- `CONFIG_ZRAM_BACKEND_ZSTD=y` + `CONFIG_ZRAM_DEF_COMP_ZSTD=y` — zram swap with
  the default `zstd` algorithm; until applied, set
  `deviceinfo_zram_swap_algo="lzo-rle"` in `/etc/deviceinfo`

## References

- [MediaTek MT6582 — postmarketOS wiki](https://wiki.postmarketos.org/wiki/MediaTek_MT6582)
- [Upstream MT6582 DTS (v6.19)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=mt6582-alcatel-yarisxl)
- [mtk-sd (MSDC) driver](https://elixir.bootlin.com/linux/latest/source/drivers/mmc/host/mtk-sd.c)

## License

Kernel source is licensed under GPL-2.0. See individual file headers for details.
