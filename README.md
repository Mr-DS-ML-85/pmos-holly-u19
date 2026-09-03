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
| WiFi/BT | ❌ Not started | MT6628 WMT driver needed |
| Touchscreen | ❌ Not started | Goodix GT9xx I2C driver needed |
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

### Build & Flash

```bash
# Full build (initramfs + kernel + DTB + pack + flash)
bash build_boot.sh

# Or manual steps:
cd out/linux-7.2.2
make ARCH=arm CROSS_COMPILE=../../toolchain/arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf- -j$(nproc) zImage dtbs
cd ../..
BUSYBOX_INSTALL_DIR=/tmp/bb_install bash build_boot.sh
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
├── build_boot.sh              # Build initramfs + kernel + pack + flash
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
- `# CONFIG_USB_G_SERIAL is not set` — legacy g_serial disabled (was stealing the UDC)

## References

- [MediaTek MT6582 — postmarketOS wiki](https://wiki.postmarketos.org/wiki/MediaTek_MT6582)
- [Upstream MT6582 DTS (v6.19)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=mt6582-alcatel-yarisxl)
- [mtk-sd (MSDC) driver](https://elixir.bootlin.com/linux/latest/source/drivers/mmc/host/mtk-sd.c)

## License

Kernel source is licensed under GPL-2.0. See individual file headers for details.
