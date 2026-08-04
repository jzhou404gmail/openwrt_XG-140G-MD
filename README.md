# OpenWrt for Nokia XG-140G-MD (AIROHA AN7581)

This repository contains my **custom OpenWrt build** for the **Nokia XG-140G-MD** GPON/10G ONT,
based on the upstream OpenWrt source tree with extensive modifications for the AIROHA AN7581 platform.

> ⚠️ **This is a personal project.**
> It is **not** intended for upstream inclusion into official OpenWrt.

---

## 📌 Device Overview

| Item | Description               |
|-----|----------------------------|
| Device | Nokia XG-140G-MD        |
| SoC | AIROHA AN7581 (ARM64)      |
| Flash | FM25G02B SPI NOR         |
| Ethernet | GDM4 / GDM1           |
| NPU | AIROHA NPU (sysfs exposed) |
| Bootloader | Custom U-Boot       |

---

## 🚀 Features

- ✅ FM25G02B SPI NOR flash support
- ✅ GDM4 Ethernet enablement (switch & MDIO)
- ✅ GDM1 Ethernet support
- ✅ AIROHA NPU firmware version sysfs interface
- ✅ PM domain / cpufreq fixes for AN7581
- ✅ Enhanced U-Boot for FM25G02B (FMSH)
- ✅ Hotplug & uci-defaults tuning for XG-140G-MD
- ✅ LuCI & utility packages pre-integrated

