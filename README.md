# Custom Embedded Linux for the Orange Pi 3B (Yocto Project)

A step-by-step guide to building a custom Linux image for the **Orange Pi 3B** (Rockchip RK3566) using the **Yocto Project / OpenEmbedded**. It was developed as the final project of an Embedded Linux diploma, and is written so you can reproduce it and adapt it to your own needs.

**Result:** a bootable SD-card image with U-Boot, a custom kernel, an OpenSSH server, Wi-Fi and NetworkManager support, an on-board toolchain (gcc/g++, gdb, strace), a package manager, and basic editors.

---

## Table of contents

1. [What you will build](#1-what-you-will-build)
2. [Prerequisites](#2-prerequisites)
3. [Project layout](#3-project-layout)
4. [Get the sources](#4-get-the-sources)
5. [Create the build environment](#5-create-the-build-environment)
6. [Configure the layers (`bblayers.conf`)](#6-configure-the-layers-bblayersconf)
7. [Configure the build (`local.conf`)](#7-configure-the-build-localconf)
8. [Build the image](#8-build-the-image)
9. [Flash the SD card](#9-flash-the-sd-card)
10. [First boot and serial console](#10-first-boot-and-serial-console)
11. [Wi-Fi and networking](#11-wi-fi-and-networking)
12. [Customizing further](#12-customizing-further)
13. [Troubleshooting](#13-troubleshooting)
14. [Hardening before real use](#14-hardening-before-real-use)
15. [References](#15-references)

---

## 1. What you will build

| Component | Choice |
|---|---|
| Build system | Yocto Project (Poky reference distro) |
| Target board | Orange Pi 3B (RK3566) |
| `MACHINE` | `orangepi-3b` (defined in `meta-rockchip/conf/machine/orangepi-3b.conf`) |
| Bootloader | U-Boot 2024.01 |
| Kernel | Linux 6.12 (upstream tarball from kernel.org), recipe `linux-orangepi3b_6.12.bb` from `meta-rockchip` |
| Kernel image type | `Image` (arm64) |
| Init / userland | Poky defaults (BusyBox + SysV/systemd per your distro config) |
| Networking | NetworkManager + `nmcli`, `wpa-supplicant`, `iw`, Broadcom SDIO Wi-Fi firmware |
| Dev tools on target | `gcc`, `g++`, `tools-sdk`, `tools-debug`, `nano`, `vim-tiny`, `htop`, `less`, `file` |

> **Note:** this guide targets the Yocto **scarthgap** release (5.0 LTS). The custom layer declares `LAYERSERIES_COMPAT_meta-orangepi3b = "scarthgap"`, so every other layer must use the **same scarthgap branch**. Mixing branches is the most common cause of parse errors.

## 2. Prerequisites

**Host machine**

- 64-bit Linux (Ubuntu 22.04/24.04 or Debian 12 recommended). A VM or WSL2 works, but native Linux is faster.
- **16 GB RAM** minimum (more is better), **100+ GB free disk** (the first build is large: downloads, sstate, tmp).
- A multi-core CPU. The first build can take several hours.

**Install host packages (Ubuntu/Debian)**

```bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
  chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
  iputils-ping python3-git python3-jinja2 python3-subunit zstd liblz4-tool \
  file locales libacl1 libsdl2-dev bmap-tools
sudo locale-gen en_US.UTF-8
```

> `libsdl2-dev` is only needed because `local.conf` enables the SDL frontend for QEMU. `bmap-tools` is optional but gives faster, safer flashing.

**Hardware**

- Orange Pi 3B board and a good 5 V / 3 A+ power supply
- microSD card (16 GB or larger, A1/A2 class recommended)
- USB-to-TTL serial adapter (3.3 V logic!) for the debug UART
- Ethernet cable (useful for first login)

## 3. Project layout

Everything lives under `/yocto` in this guide (change the path if you prefer, but keep it consistent in the config files):

```
/yocto
├── poky/                       # Yocto reference distro (meta, meta-poky, meta-yocto-bsp)
├── meta-openembedded/          # meta-oe, meta-python, meta-networking
├── meta-arm/                   # meta-arm, meta-arm-toolchain
├── meta-rockchip/              # Rockchip SoC + board support (machine, kernel, U-Boot)
├── meta-orangepi3b/            # project layer (your own recipes, bbappends, images)
├── downloads/                  # DL_DIR  (source tarballs)
├── sstate-cache/               # SSTATE_DIR (shared state cache)
└── build/                      # build directory (conf/, tmp/)
    └── conf/
        ├── bblayers.conf
        └── local.conf
```

Keeping `downloads/` and `sstate-cache/` **outside** the build directory means you can delete `build/tmp` and rebuild quickly.

## 4. Get the sources

Use the `scarthgap` branch for every layer.

```bash
sudo mkdir -p /yocto && sudo chown $USER:$USER /yocto
cd /yocto

git clone -b scarthgap git://git.yoctoproject.org/poky
git clone -b scarthgap https://github.com/openembedded/meta-openembedded.git
git clone -b scarthgap https://git.yoctoproject.org/meta-arm
git clone -b scarthgap https://git.yoctoproject.org/meta-rockchip
git -C meta-rockchip checkout 9690180b7e9953361958c962b80d891eda8c1028   # the commit this guide was built with

# Your own project layer (this repository or a sibling repo)
git clone <your-meta-orangepi3b-repo-url> meta-orangepi3b
```

> **Check:** run `git branch` in each layer and verify they all target scarthgap. `meta-rockchip` was built from branch `scarthgap` at commit `9690180b7e9953361958c962b80d891eda8c1028` (3 June 2025). Pin to this commit for a reproducible build; newer commits may change the boot flow.

## 5. Create the build environment

```bash
cd /yocto
source poky/oe-init-build-env build
```

This creates `/yocto/build` with a `conf/` directory and drops you into it. **Run this command again in every new terminal** before using `bitbake`.

## 6. Configure the layers (`bblayers.conf`)

Edit `/yocto/build/conf/bblayers.conf`:

```bitbake
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /yocto/poky/meta \
  /yocto/poky/meta-poky \
  /yocto/poky/meta-yocto-bsp \
  "
BBLAYERS += " /yocto/meta-openembedded/meta-oe "
BBLAYERS += " /yocto/meta-openembedded/meta-python "
BBLAYERS += " /yocto/meta-openembedded/meta-networking "
BBLAYERS += " /yocto/meta-arm/meta-arm "
BBLAYERS += " /yocto/meta-arm/meta-arm-toolchain "
BBLAYERS += " /yocto/meta-rockchip "
BBLAYERS += " /yocto/meta-orangepi3b "
```

**Why each layer is here**

| Layer | Purpose |
|---|---|
| `meta`, `meta-poky`, `meta-yocto-bsp` | OpenEmbedded-Core and the Poky distro policy |
| `meta-oe`, `meta-python`, `meta-networking` | Extra packages; NetworkManager and many of its dependencies live in `meta-networking`/`meta-oe` |
| `meta-arm`, `meta-arm-toolchain` | Arm toolchain and firmware components (for example Trusted Firmware-A) |
| `meta-rockchip` | Rockchip SoC and board support: the `orangepi-3b` machine, the `linux-orangepi3b` kernel (6.12), U-Boot, boot flow and image layout |
| `meta-orangepi3b` | Project layer for your own recipes, `.bbappend` files and custom images (currently a skeleton) |

You can validate the layer list at any time:

```bash
bitbake-layers show-layers
```

### 6.1 The custom layer: `meta-orangepi3b`

This layer is where your own customizations live, so the upstream layers stay untouched and updatable. In this project the board support itself (machine, kernel, U-Boot) comes from `meta-rockchip`; `meta-orangepi3b` is the project layer, created with Yocto's helper and ready for your own recipes, `.bbappend` files and images:

```bash
cd /yocto
bitbake-layers create-layer meta-orangepi3b
bitbake-layers add-layer meta-orangepi3b      # adds it to bblayers.conf
```

This generates the skeleton below (the `COPYING.MIT` license file is included by the tool):

```
meta-orangepi3b/
├── COPYING.MIT
├── README
├── conf/
│   └── layer.conf
└── recipes-example/
    └── example/
        └── example_0.1.bb
```

**`conf/layer.conf`** as used in this project:

```bitbake
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-orangepi3b"
BBFILE_PATTERN_meta-orangepi3b = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-orangepi3b = "6"

LAYERDEPENDS_meta-orangepi3b = "core"
LAYERSERIES_COMPAT_meta-orangepi3b = "scarthgap"
```

What the important lines mean:

- `BBFILES`: BitBake picks up every `recipes-*/*/*.bb` and `.bbappend` in the layer. New recipes must follow this directory pattern.
- `BBFILE_PRIORITY = "6"`: when two layers provide the same recipe, the higher priority wins, so your layer can override others.
- `LAYERSERIES_COMPAT = "scarthgap"`: the layer refuses to load on other releases.
- `LAYERDEPENDS = "core"`: only depends on OpenEmbedded-Core. Once you add recipes or `.bbappend` files that depend on `meta-rockchip` (for example a kernel append), add it here (`LAYERDEPENDS_meta-orangepi3b = "core rockchip"`; `rockchip` is the collection name declared in `meta-rockchip/conf/layer.conf`), so a missing layer gives a clear error.

**Clean up the generated skeleton**

- `recipes-example/example/example_0.1.bb` is only the demo recipe that `create-layer` produces (it prints a banner during `do_build` and installs nothing). Delete it, or replace it with real recipes.
- Fill in the layer's `README` template (dependencies with URI and branch, maintainer, how to add the layer), since it is still the placeholder text.

**What comes from `meta-rockchip` (not from this layer)**

| Item | Location in `meta-rockchip` |
|---|---|
| Machine `orangepi-3b` | `conf/machine/orangepi-3b.conf` |
| Kernel recipe | `recipes-kernel/linux/linux-orangepi3b_6.12.bb` (source: `https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.tar.xz`) |
| U-Boot | recipe with `UBOOT_MACHINE = "orangepi-3b-rk3566_defconfig"` (see section 7) |

The bundled configuration is selected from `local.conf` with `MACHINE`, `PREFERRED_PROVIDER_virtual/kernel` and `PREFERRED_VERSION_u-boot`. This layer is where you add *your* changes on top, using these paths (they follow the `recipes-*` pattern in `BBFILES`):

```
meta-orangepi3b/
├── recipes-kernel/linux/linux-orangepi3b_%.bbappend   # kernel config fragments / patches
├── recipes-bsp/u-boot/u-boot_%.bbappend               # U-Boot patches
└── recipes-core/images/orangepi-image.bb              # your custom image (section 12)
```

> **Tip:** moving the package lists from `local.conf` into a custom image recipe in this layer is the cleanest way to make the layer the real, reproducible deliverable of the project.

## 7. Configure the build (`local.conf`)

Below is the *essential* part of my `/yocto/build/conf/local.conf` (the long comment block that Yocto generates is unchanged and omitted here). Add these lines at the end of the file.

```bitbake
#### Target machine ####
MACHINE = "orangepi-3b"
DISTRO ?= "poky"

#### Shared locations (survive deleting build/tmp) ####
DL_DIR     = "/yocto/downloads"
SSTATE_DIR = "/yocto/sstate-cache"

#### Bootloader and kernel ####
PREFERRED_VERSION_u-boot = "2024.01"
PREFERRED_PROVIDER_virtual/kernel = "linux-orangepi3b"
KERNEL_IMAGETYPE = "Image"
BB_SRCREV_POLICY = "cache"
IMAGE_INSTALL:append = " kernel-modules"

#### Image features ####
EXTRA_IMAGE_FEATURES += "ssh-server-openssh debug-tweaks"
EXTRA_IMAGE_FEATURES:append = " package-management"
EXTRA_IMAGE_FEATURES:append = " tools-sdk tools-debug"

#### Extra packages ####
IMAGE_INSTALL:append = " nano vim-tiny less file htop gcc g++"

#### Networking / Wi-Fi ####
DISTRO_FEATURES:append = " wifi"
CORE_IMAGE_EXTRA_INSTALL:append = " brcmfmac_sdio-firmware iw wpa-supplicant"
IMAGE_INSTALL:append = " networkmanager networkmanager-nmcli"

#### Optional: mirrors, if upstream hosts are slow or unreachable ####
PREMIRRORS:prepend = " \
    git://source.denx.de/u-boot/u-boot.git https://downloads.yoctoproject.org/mirror/sources/ \
    https://source.denx.de/u-boot/u-boot.git https://downloads.yoctoproject.org/mirror/sources/ \
"
```

### What each setting does

- **`MACHINE = "orangepi-3b"`**: selects the machine configuration shipped in `meta-rockchip`. Without it the build defaults to QEMU or fails.
- **`DL_DIR` / `SSTATE_DIR`**: shared caches. Back these up; they save hours on rebuilds.
- **`PREFERRED_VERSION_u-boot`**: pins U-Boot so a layer update does not silently change your bootloader.
- **`PREFERRED_PROVIDER_virtual/kernel`**: selects the `linux-orangepi3b` recipe (Linux 6.12) from `meta-rockchip` as the kernel provider.
- **`KERNEL_IMAGETYPE = "Image"`**: arm64 boards boot an uncompressed `Image`.
- **`kernel-modules`**: installs all kernel modules built by the kernel recipe (needed for Wi-Fi drivers, filesystems, etc.).
- **`ssh-server-openssh`**: installs OpenSSH so you can log in over the network.
- **`package-management`**: puts the package manager on the target so you can install packages at runtime from a feed.
- **`tools-sdk` / `tools-debug`**: compiler, make, pkg-config, gdb, strace on the board. Great for development, large for production.
- **`DISTRO_FEATURES:append = " wifi"`**: enables Wi-Fi support across the distro.
- **`brcmfmac_sdio-firmware`**: firmware for the Broadcom SDIO Wi-Fi chip. **Verify the exact recipe/package name** in your release with `bitbake-layers show-recipes "*brcm*"`. It may be named differently (for example under `linux-firmware-*`).
- **`PREMIRRORS`**: fallback download sources for when the upstream server is unreachable.

### Bootloader details (from `bitbake -e virtual/bootloader`)

| Variable | Value |
|---|---|
| `PV` | `2024.01` |
| `UBOOT_MACHINE` | `orangepi-3b-rk3566_defconfig` |
| Source | `github.com/Kwiboo/u-boot-rockchip.git`, branch `rk3xxx-2024.07` |
| `RK_CONSOLE_DEVICE` / `RK_CONSOLE_BAUD` | `ttyS2` / `1500000` |
| `RK_IMAGE_INCLUDES_UBOOT_ENV` | `no` |

The recipe also applies a series of upstream CVE security patches (CVE-2024-57254 through CVE-2024-57259, CVE-2024-42040, CVE-2025-24857 and CVE-2026-46728). You can inspect the full list yourself with:

```bash
bitbake -e virtual/bootloader | grep -E "^(SRC_URI|UBOOT_MACHINE|PV)="
```

### Important syntax notes

- Use `+=`/`:append` for list variables like `IMAGE_INSTALL`. A plain `=` replaces the whole list.
- **`:append` does not add a space for you**: always start the value with a space (`" nano"`, not `"nano"`).
- Each variable needs to appear only once with each package. Duplicates are harmless but make the file harder to read.
- Setting `MACHINE` twice is also harmless but redundant; keep a single line.

## 8. Build the image

Pick a target image. `core-image-minimal` is the smallest; `core-image-base` is a better starting point if you want networking tools and a bit more userland:

```bash
cd /yocto
source poky/oe-init-build-env build
bitbake core-image-base
```

For a first sanity check you can parse everything without building:

```bash
bitbake -p
```

The first build downloads and compiles the full toolchain, kernel, U-Boot and all packages. Expect **2 to 8 hours**, depending on hardware. Later builds reuse `sstate-cache` and are far faster.

### Where are the outputs?

```bash
ls /yocto/build/tmp/deploy/images/orangepi-3b/
```

You should find (names vary with your image and layer config):

- `core-image-base-orangepi-3b.rootfs.wic` (or `.wic.gz` / `.wic.bz2`), the **complete disk image**
- `Image`, the kernel
- `*.dtb`, device tree blobs
- `u-boot*.bin` / `idbloader.img` / `u-boot.itb`, bootloader pieces
- `*.rootfs.tar.*` / `*.ext4`, the root filesystem

## 9. Flash the SD card

> **Warning:** `dd` overwrites whatever device you point it at. Identify your SD card carefully with `lsblk` first.

```bash
lsblk                       # find your SD card, e.g. /dev/sdX (NOT your system disk!)
cd /yocto/build/tmp/deploy/images/orangepi-3b/
```

**Option A: `bmaptool` (recommended: faster and verifies the write)**

```bash
sudo bmaptool copy core-image-base-orangepi-3b.rootfs.wic.gz /dev/sdX
```

**Option B: `dd`**

```bash
# if the image is compressed:
zcat core-image-base-orangepi-3b.rootfs.wic.gz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
# if it is an uncompressed .wic:
sudo dd if=core-image-base-orangepi-3b.rootfs.wic of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

Eject the card, insert it into the Orange Pi 3B.

## 10. First boot and serial console

The serial console is the most reliable way to see boot problems.

**Wiring (3.3 V TTL adapter to the board's debug UART header):**

| Adapter | Board |
|---|---|
| GND | GND |
| RX | TX |
| TX | RX |

> Do **not** connect the adapter's 5 V/VCC pin.

The console is `ttyS2` at **1500000 baud**, 8N1 (set by `RK_CONSOLE_DEVICE` and `RK_CONSOLE_BAUD` in the Rockchip layer):

```bash
sudo picocom -b 1500000 /dev/ttyUSB0
# or
sudo screen /dev/ttyUSB0 1500000
```

Power on the board. You should see the boot ROM / U-Boot messages, then the kernel log, then the login prompt.

**Login:** because `debug-tweaks` is enabled, `root` has an **empty password**. Log in as `root`.

If you connected Ethernet, find the board's IP (router's DHCP list, or `ip a` on the serial console) and:

```bash
ssh root@<board-ip>
```

## 11. Wi-Fi and networking

With NetworkManager installed:

```bash
nmcli radio wifi on
nmcli device wifi list
nmcli device wifi connect "<SSID>" password "<PASSWORD>"
nmcli connection show
ip a
```

If no Wi-Fi interface appears:

```bash
dmesg | grep -i -E "brcm|wlan|firmware"
ls /lib/firmware/brcm/
iw dev
```

Typical causes: missing firmware file (wrong package name in `CORE_IMAGE_EXTRA_INSTALL`), the driver module missing (`kernel-modules`), or the firmware/NVRAM file name not matching what the kernel driver requests.

## 12. Customizing further

**Add a package to the image:** append it in `local.conf`, then rebuild:

```bitbake
IMAGE_INSTALL:append = " python3 python3-pip"
```

**Find the recipe for something:**

```bash
bitbake-layers show-recipes "*python3*"
```

**Create your own image recipe** (cleaner than piling everything into `local.conf`). In `meta-orangepi3b/recipes-core/images/orangepi-image.bb`:

```bitbake
SUMMARY = "Orange Pi 3B custom image"
require recipes-core/images/core-image-base.bb

IMAGE_INSTALL:append = " nano vim-tiny htop networkmanager networkmanager-nmcli"
```

Build it with `bitbake orangepi-image`.

**Modify the kernel configuration:**

```bash
bitbake -c menuconfig virtual/kernel
bitbake -c diffconfig virtual/kernel     # produces a config fragment (.cfg)
```

Put the generated `.cfg` fragment in `meta-orangepi3b/recipes-kernel/linux/files/` and add it from a `linux-orangepi3b_%.bbappend` (`FILESEXTRAPATHS:prepend := "${THISDIR}/files:"` plus `SRC_URI += "file://my.cfg"`) so the change is reproducible and `meta-rockchip` stays untouched.

**Change the device tree:** add a patch or a new `.dts` from a `linux-orangepi3b_%.bbappend` in your layer. Check which device tree the machine uses with `bitbake -e virtual/kernel | grep ^KERNEL_DEVICETREE=` (defined in `meta-rockchip`).

**Build an SDK** for cross-compiling applications on your PC:

```bash
bitbake core-image-base -c populate_sdk
```

## 13. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `Nothing PROVIDES 'linux-orangepi3b'` | Layer not in `bblayers.conf`, wrong recipe name, or recipe not matching `COMPATIBLE_MACHINE`. Run `bitbake-layers show-layers` and `bitbake-layers show-recipes "linux-*"`. |
| `Layer ... is not compatible with the core layer` | A layer's `LAYERSERIES_COMPAT` does not include your release. Make all branches match. |
| `Nothing PROVIDES 'brcmfmac_sdio-firmware'` | Wrong package name. Search with `bitbake-layers show-recipes "*firmware*"`. |
| Fetch errors / timeouts | Check your network; keep the `PREMIRRORS` entries; retry with `bitbake <recipe> -c cleansstate` then rebuild. |
| `Do not use Bitbake as root` | Build as a normal user, not with `sudo`. |
| Out of disk space | Build monitors disk; free space, then `rm -rf build/tmp` (keeps `sstate-cache` and `downloads`). |
| Board does not boot, no serial output | Re-flash; try another SD card; verify power supply; check UART wiring and baud rate (1500000). |
| Stops in U-Boot | Wrong DTB / kernel path in the boot config, or kernel `Image` missing from the boot partition. Inspect with `printenv` and `ls mmc 1:1` at the U-Boot prompt. |
| Kernel panic: cannot mount root | Root device (`root=`) mismatch, or the needed storage driver is not built in. |
| Rebuild everything weirdly | `bitbake -c cleansstate <recipe>` for one recipe; as a last resort remove `build/tmp`. |

Useful debugging commands:

```bash
bitbake -e <recipe> | grep ^VARIABLE=        # show the final value of a variable
bitbake <recipe> -c devshell                  # open a shell in the recipe's work dir
bitbake -g core-image-base                    # dependency graph
oe-pkgdata-util find-path /path/in/image      # which package owns a file
```

## 14. Hardening before real use

The configuration in this guide is a **development image**. Before deploying on a networked device:

- **Remove `debug-tweaks`.** It sets an empty root password and allows passwordless SSH root login. Instead, set a password (for example with the `extrausers` class and `EXTRA_USERS_PARAMS`) or use SSH keys only.
- **Remove `tools-sdk` / `tools-debug`**, `gcc`, `g++` from the final image. They greatly increase size and attack surface.
- Build a dedicated image recipe rather than loading `local.conf` with packages.
- Pin layer revisions (commit hashes) so builds are reproducible.
- Consider read-only rootfs and an update mechanism (RAUC, SWUpdate, Mender) for production.

## 15. References

- Yocto Project Documentation: <https://docs.yoctoproject.org/>
- Yocto Project Quick Build
- OpenEmbedded Layer Index: <https://layers.openembedded.org/>
- `meta-rockchip`: <https://git.yoctoproject.org/meta-rockchip>
- Orange Pi 3B official resources (schematics, user manual): <http://www.orangepi.org/>
- U-Boot documentation: <https://docs.u-boot.org/>

---

## Repository contents

```
.
├── README.md                # this guide
├── conf/
│   ├── bblayers.conf        # layer configuration used in this project
│   └── local.conf           # build configuration used in this project
└── meta-orangepi3b/         # project layer (or link to its own repo)
```

## License

The `meta-orangepi3b` layer and this guide are released under the **MIT License** (see `COPYING.MIT` in the layer; copy it to `LICENSE` at the repository root so GitHub detects it). Add your own copyright line at the top, as the MIT text requires a copyright notice. Upstream layers (Poky, OpenEmbedded, meta-arm, meta-rockchip) and the software they build keep their own licenses.

## Author

**Mohamed Yasser**: [github.com/Mohamedyasser-07](https://github.com/Mohamedyasser-07)
