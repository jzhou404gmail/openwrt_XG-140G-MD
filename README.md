# OpenWrt for Nokia XG-040G-MD (AIROHA AN7581)

Custom OpenWrt firmware for the **Nokia XG-040G-MD** GPON/10G ONT, based on the AIROHA AN7581 SoC platform with extensive driver modifications, hardware acceleration, and a custom LuCI monitoring interface.

> **This is a personal project.** It is **not** intended for upstream inclusion into official OpenWrt.

---

## Device Overview

| Item | Description |
|------|-------------|
| Device | Nokia XG-040G-MD |
| SoC | AIROHA AN7581 (ARM64 Cortex-A53) |
| RAM | 512 MB DDR4 |
| Flash | FM25G02B SPI-NAND 256MB |
| WAN | 2.5G Ethernet (GDM4 + EN8811H PHY) |
| LAN | 3x 1G Ethernet (GDM1 via MT7530 switch) |
| USB | 2x USB 3.0 |
| LEDs | 12x GPIO LEDs (Power, WAN, WAN-Online, USB x2, LAN x4, etc.) |
| Bootloader | Custom U-Boot (FIP-based, LZMA compressed) |
| Kernel | Linux 6.18 |

### Port Mapping

| Physical Port | Interface | DSA Port | Role |
|---------------|-----------|----------|------|
| LAN1 (2.5G) | GDM4 | wan | WAN (2.5G) |
| LAN2 | MT7530 P2 | lan2 | LAN (br-lan) |
| LAN3 | MT7530 P3 | lan3 | LAN (br-lan) |
| LAN4 | MT7530 P4 | lan4 | LAN (br-lan) |

- GDM1 (eth0) is renamed to `dsa-mgr` via DTS `openwrt,netdev-name` to hide the internal switch management interface
- LAN MAC: device factory address (from NVMEM `factory` partition offset `0x3e`)
- WAN MAC: LAN MAC + 1

---

## Key Features

### Hardware Support

- **FM25G02B SPI-NAND flash** - Custom driver patch for FMSH SPI-NAND in U-Boot
- **2.5G WAN (GDM4)** - Enabled via DTS with EN8811H PHY (address 0xf, 2500base-x mode)
- **GDM1 Ethernet** - Internal switch management interface
- **NPU firmware** - AIROHA NPU with sysfs firmware version reporting
- **CPU PM domain** - Custom `airoha-cpufreq` driver fix for proper frequency/governor reporting
- **Thermal & Watchdog** - AIROHA thermal sensor and hardware watchdog support
- **GPIO LEDs & buttons** - 12 LEDs with internet-online trigger, reset button

### Hardware Acceleration (PPE/NPU)

- **PPE Flow Offload** - Hardware packet forwarding enabled by default via `99-enable-flowoffload` uci-defaults script
  - Sets `firewall.@defaults[0].flow_offloading=1` and `flow_offloading_hw=1`
  - PPE entries visible in LuCI and debugfs (`/sys/kernel/debug/airoha_ppe/`)
- **NPU Traffic Statistics** - Hardware flow statistics via `CONFIG_NET_AIROHA_FLOW_STATS=y`
- **Frame Engine Counters** - Real-time monitoring of GDM/CDM/PSE counters

### LuCI Application: `luci-app-airoha-npu`

Custom LuCI app providing a real-time monitoring dashboard for the AN7581 SoC:

| Feature | Description |
|---------|-------------|
| **NPU Status** | Firmware version, driver status, NPU core info |
| **CPU Frequency** | Current frequency, governor, max frequency, overclock support |
| **PPE Flow Offload** | Hardware flow table entries viewer with debugfs integration |
| **Frame Engine** | GDM1/GDM4 TX/RX counters (via sysfs), CDM1/CDM2 counters (via devmem) |
| **PSE Shared Buffer** | Port queue status with indirect register read (ports 0-9) |

#### Counter Implementation Details

- **GDM1/GDM4** - Read via `/sys/class/net/<dev>/statistics/` to avoid hardware MIB clear race condition with kernel driver
- **CDM1/CDM2** - Read via `devmem` at GDM_BASE+0x80 offsets (CDM1: `0x1fb50580`, CDM2: `0x1fb51580`)
- **PSE Queue Status** - Indirect read: write port/queue ID to `0x1fb50180`, read value from `0x1fb50184`
- **devmem** - Enabled in busybox (`CONFIG_BUSYBOX_DEFAULT_DEVMEM=y`) for direct register access

### Network Configuration

- DSA-based port naming (lan2, lan3, lan4, wan)
- MAC address assignment from NVMEM factory partition
- Hotplug script (`20-set-mac`) for MAC address propagation
- UCI defaults script (`99-set-mac`) for persistent MAC configuration
- Internet online LED trigger (`99-internet-led`) - lights up `green:wan-online` when WAN has internet connectivity

### Utility Packages

| Package | Description |
|---------|-------------|
| `luci-app-airoha-npu` | NPU/PPE/CPU monitoring dashboard |
| `luci-app-nfs` | NFS server LuCI frontend |
| `luci-app-vlmcsd` | KMS activation server LuCI frontend |
| `luci-app-vsftpd` | FTP server LuCI frontend |
| `nfs-kernel-server` | NFS v3/v4 server |
| `vlmcsd` | KMS emulation server |
| `vsftpd` | Secure FTP server with UCI auth support |
| `autocore` | CPU core info for LuCI status page |
| `automount` | Auto-mount USB storage (ext4/fat/ntfs/exfat) |
| `autosamba` | Auto-configure Samba shares |
| `cpufreq` | CPU frequency scaling utility |

---

## Build Instructions

### Prerequisites

```bash
# Ubuntu/Debian
sudo apt install build-essential clang flex bison gawk gperf \
    git gettext libncurses-dev libssl-dev rsync unzip wget \
    python3 python3-pyelftools
```

### Clone & Build

```bash
git clone git@github.com:jzhou404gmail/openwrt_XG-140G-MD.git
cd openwrt_XG-140G-MD

# Update feeds
./scripts/feeds update -a
./scripts/feeds install -a

# Configure (Nokia XG-040G-MD UBI variant is the default target)
make menuconfig
# Target: Airoha ARM  ->  AN7581 / AN7566 / AN7551
# Device: Nokia XG-040G-MD (UBI)

# Build
make -j$(nproc) V=s
```

### Build Outputs

Firmware images are generated in `bin/targets/airoha/an7581/`:

| File | Description |
|------|-------------|
| `*-ubi-squashfs-sysupgrade.itb` | UBI sysupgrade image (primary) |
| `*-ubi-initramfs-recovery.itb` | Initramfs recovery image |
| `*-ubi-preloader.bin` | BL2 preloader |
| `*-ubi-bl31-uboot.fip` | BL31 + U-Boot FIP |

### U-Boot Only Build

```bash
make package/boot/uboot-airoha/compile V=s
```

---

## Flash Instructions

### Initial Flash (from stock firmware)

1. Extract `factory-kernel.bin` and `factory-rootfs.bin` from the build output
2. Use the stock firmware's web UI or TFTP to flash kernel and rootfs separately
3. After first boot, use the sysupgrade image for future updates

### Sysupgrade (from OpenWrt)

```bash
# Via LuCI: System -> Backup/Flash Firmware
# Via CLI:
sysupgrade -n /tmp/openwrt-airoha-an7581-nokia_xg-040g-md-ubi-squashfs-sysupgrade.itb
```

### Recovery

If the device fails to boot, use the initramfs recovery image via TFTP or U-Boot TFTP boot.

---

## U-Boot Patches

Custom U-Boot patches for Nokia XG-040G-MD (in `package/boot/uboot-airoha/patches/`):

| Patch | Description |
|-------|-------------|
| `210-add-fm25g02b-to-fmsh.patch` | Add FM25G02B SPI-NAND support to FMSH driver |
| `401-add-nokia-xg-040g-md.patch` | Add Nokia XG-040G-MD board support |
| `402-enable-fmsh-in-xg040g-defconfig.patch` | Enable FMSH in defconfig |
| `403-enable-gdm1-ethernet-for-xg040g.patch` | Enable GDM1 Ethernet |
| `404-add-pcs-switch-gdm4-to-en7581-dtsi.patch` | Add PCS, switch, GDM4 to AN7581 DTSI |
| `405-enable-switch-mdio-gdm4-in-xg040g.patch` | Enable switch MDIO and GDM4 |
| `406-add-phy_airoha-parent-config.patch` | Add PHY parent configuration |
| `407-add-bootph-and-status-to-dtsi-nodes.patch` | Add bootph and status to DTSI nodes |
| `408-convert-dtsi-nodes-to-references.patch` | Convert DTSI nodes to references |

---

## Kernel Configuration

Key kernel config options for AN7581 (`target/linux/airoha/an7581/config-6.18`):

```
# SoC support
CONFIG_ARCH_AIROHA=y
CONFIG_AIROHA_CPU_PM_DOMAIN=y
CONFIG_AIROHA_SCU_SSR=y
CONFIG_AIROHA_THERMAL=y
CONFIG_AIROHA_WATCHDOG=y

# Ethernet
CONFIG_NET_AIROHA=y
CONFIG_NET_AIROHA_NPU=y
CONFIG_NET_AIROHA_FLOW_STATS=y

# PPE Flow Offload
CONFIG_NETFILTER_INGRESS=y
CONFIG_NF_FLOW_TABLE=y
CONFIG_NF_FLOW_TABLE_INET=y
CONFIG_NF_FLOW_TABLE_PROCFS=y
CONFIG_NFT_FLOW_OFFLOAD=y
CONFIG_NETFILTER_XT_TARGET_FLOWOFFLOAD=y
CONFIG_NET_CLS_ACT=y
CONFIG_NET_CLS_FLOWER=y

# PHY
CONFIG_PHY_AIROHA_PCIE=y
CONFIG_PHY_AIROHA_USB=y
```

---

## Device Tree Structure

```
an7581-nokia_xg-040g-md-ubi.dts        # UBI variant (top-level)
an7581-nokia_xg-040g-md.dts            # Stock partition variant
an7581-nokia_xg-040g-md-common.dtsi    # Common board config (LEDs, keys, regulators, GDM)
an758x-nokia_xg-040g-common.dtsi       # Common SoC config (eth, npu, spi_nand, GDM1/4)
an758x-nokia_xg-040g-ubi-parts.dtsi    # UBI partition layout
an758x-nokia_xg-040g-stock-parts.dtsi  # Stock partition layout
```

---

## Verification Commands

After flashing, verify functionality:

```bash
# Check NPU status
ubus call luci.airoha_npu getStatus

# Check Frame Engine counters (should show non-zero with traffic)
ubus call luci.airoha_npu getFrameEngine

# Check PPE flow offload entries (should show entries with traffic)
ubus call luci.airoha_npu getPpeEntries

# Verify devmem works
devmem 0x1fb50000

# Verify PPE flow offload is enabled
uci get firewall.@defaults[0].flow_offloading
uci get firewall.@defaults[0].flow_offloading_hw

# Check network interfaces
ip link show
```

---

## Project Structure

```
target/linux/airoha/
├── an7581/
│   ├── base-files/etc/
│   │   ├── board.d/02_network          # Interface configuration
│   │   ├── hotplug.d/net/20-set-mac    # MAC address propagation
│   │   ├── hotplug.d/iface/99-internet-led  # WAN online LED
│   │   ├── init.d/internet-led         # Internet LED daemon
│   │   └── uci-defaults/
│   │       ├── 99-set-mac              # Persistent MAC setup
│   │       └── 99-enable-flowoffload   # Auto-enable PPE offload
│   ├── config-6.18                     # Kernel configuration
│   └── target.mk                       # Target definition
├── dts/
│   └── an7581-nokia_xg-040g-md*.dts*   # Device tree files
└── image/an7581.mk                     # Image build rules

package/
├── boot/uboot-airoha/patches/          # U-Boot patches (210, 401-408)
├── luci-app-airoha-npu/                # Custom NPU monitoring LuCI app
├── luci/applications/                  # LuCI apps (NFS, vlmcsd, vsftpd)
├── network/services/                   # Network services (NFS, vlmcsd, vsftpd)
└── utils/                              # Utilities (autocore, automount, cpufreq)

package/utils/busybox/Config-defaults.in  # Busybox config (devmem enabled)
```

---

## Technical Notes

### GDM Counter Race Condition

The kernel driver `airoha_update_hw_stats()` in `airoha_eth.c` reads and clears GDM MIB hardware counters on each update. Reading the same registers via `devmem` from userspace causes a race condition where counters may appear as zero. The LuCI RPC backend resolves this by reading GDM1/GDM4 counters from sysfs (`/sys/class/net/<dev>/statistics/`) which returns the driver's accumulated software counters instead of hardware registers.

### PSE Queue Indirect Read

PSE Port Queue status cannot be read directly from a fixed address. The read requires two steps:
1. Write port ID and queue ID to `REG_FE_PSE_QUEUE_CFG_WR` (`0x1fb50180`)
2. Read the result from `REG_FE_PSE_QUEUE_CFG_VAL` (`0x1fb50184`)

Port mapping: 0=CDM1, 1=GDM1, 2=GDM2, 3=GDM3, 4=PPE1, 5=CDM2, 6=CDM3, 7=CDM4, 8=PPE2, 9=GDM4

### CDM Counter Addresses

CDM counters are located at `GDM_BASE + 0x80` (not `CDM_BASE + 0x80`):
- CDM1 TX_OK: `0x1fb50580` (GDM1_BASE `0x0500` + `0x080`)
- CDM1 RXCPU_OK: `0x1fb50590`
- CDM2 TX_OK: `0x1fb51580` (GDM2_BASE `0x1500` + `0x080`)

---

## License

OpenWrt source code is licensed under GPL-2.0-or-later.
Custom LuCI app (`luci-app-airoha-npu`) is licensed under Apache-2.0.

---

## Acknowledgments

- [OpenWrt Project](https://openwrt.org/) - The upstream OpenWrt source
- [AIROHA/ MediaTek](https://www.airoha.com/) - AN7581 SoC and NPU firmware
- Upstream kernel contributors for the `airoha_eth` driver
