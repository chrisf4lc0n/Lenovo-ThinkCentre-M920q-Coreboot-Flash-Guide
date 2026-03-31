# Lenovo ThinkCentre M920q — Coreboot Flash Guide

Tested and verified guide for flashing coreboot on the Lenovo ThinkCentre M920 Tiny (M920q / M920x).

## Overview

| Feature | Value |
|---------|-------|
| CPU | Intel Core 8th/9th Gen (Coffee Lake / Coffee Lake Refresh) |
| Chipset | Intel Q370 (Cannon Lake PCH-H) |
| DRAM | 2× SO-DIMM DDR4-2400/2666, max 64 GB |
| ME version | v12 |
| Super I/O | NCT6686D-L (Nuvoton) |
| TPM | Infineon SLB 9670VQ2.0 (TPM 2.0) |
| Boot Guard | Not fused — direct external flash works |
| SoC coreboot directory | `soc/intel/cannonlake` |

### What Works

- USB 3.0 / 2.0 (front and rear)
- USB-C (charging and data)
- Gigabit Ethernet
- SATA and NVMe
- HDMI
- WiFi slot
- TPM 2.0
- Internal speaker
- COM1 serial (via daughter board)
- PCIe x8 riser slot (tested with BA7H70 Rev 1.2 riser + Intel X540-T2 10GbE)
- S3 suspend/resume
- PXE boot (via edk2 built-in UEFI network stack)

### Known Issues

- **Front audio jacks do not work** (upstream coreboot bug)
- **DIMM2 slot does not work** — [coreboot bug #592](https://ticket.coreboot.org/issues/592). Only DIMM1 is usable. Use a single larger SO-DIMM if more RAM is needed
- **ME Communication Controller (PCI 16.0) remains visible** with HAP bit set — this is normal hardware enumeration, the ME firmware is disabled

---

## Flash Chips

The M920q has **two** SOIC-8 SPI flash chips, both 3.3V, soldered (not socketed):

| Chip | Board Label | Model | Size | flashprog `-c` string | Pull-up Notes |
|------|-------------|-------|------|-----------------------|---------------|
| BIOS1 | U31 | W25Q128JV | 16 MiB | `W25Q128.V` | /WP and /HOLD pulled high by board — no action needed |
| BIOS2 | U32 | W25Q64JV | 8 MiB | `W25Q64JV-.Q` | **/WP (pin 3) and /HOLD (pin 7) must be held HIGH manually** |

Total ROM size: **24 MiB** (16 + 8).

---

## Prerequisites

### Hardware

- Raspberry Pi (3/4/5) with SPI enabled and [flashprog](https://flashprog.org/) installed, or any other SPI programmer providing **3.3V logic levels**
- SOIC-8 test clip (Pomona 5250 or equivalent)
- Dupont jumper wires
- IC hooks (for holding /WP and /HOLD high on BIOS2)

> **Warning:** Do not use a CH341A programmer unless you have confirmed it uses 3.3V logic on data lines. Many clones output 5V on MOSI/MISO/CLK which can damage the SPI flash and the PCH.

### Raspberry Pi SPI Pinout → SOIC-8

| RPi GPIO | SPI Function | SOIC-8 Pin |
|----------|-------------|------------|
| GPIO 8 (CE0, physical pin 24) | CS | 1 (dot on chip) |
| GPIO 9 (MISO, physical pin 21) | MISO (DO) | 2 |
| GPIO 11 (SCLK, physical pin 23) | CLK | 6 |
| GPIO 10 (MOSI, physical pin 19) | MOSI (DI) | 5 |
| 3V3 (physical pin 1 or 17) | VCC | 8 |
| GND (physical pin 6, 9, etc.) | GND | 4 |

> **Important:** Disconnect the M920q power adapter before attaching the programmer. Do not supply mains power while flashing.

---

## Step 1: Read Original Firmware

Triple-read each chip. All three SHA256 checksums **must** match before proceeding. If they don't, reseat the clip and read again.

### BIOS1 (16 MiB)

No extra pull-ups required.

```bash
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q128.V" -r m920q_bios1_dump1.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q128.V" -r m920q_bios1_dump2.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q128.V" -r m920q_bios1_dump3.bin

sha256sum m920q_bios1_dump*.bin
```

### BIOS2 (8 MiB)

Move clip to BIOS2. **Hold /WP (pin 3) and /HOLD (pin 7) HIGH** (tie to 3.3V with IC hooks).

```bash
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q64JV-.Q" -r m920q_bios2_dump1.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q64JV-.Q" -r m920q_bios2_dump2.bin
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q64JV-.Q" -r m920q_bios2_dump3.bin

sha256sum m920q_bios2_dump*.bin
```

> **Troubleshooting:** If BIOS2 is not detected, check that /WP and /HOLD are held HIGH. BIOS1 has on-board pull-ups for these pins; BIOS2 does not. You may also try lowering `spispeed` to `512` if detection is unreliable.

### Combine and Back Up

```bash
cat m920q_bios1_dump1.bin m920q_bios2_dump1.bin > m920q_original_24mb.bin

ls -la m920q_original_24mb.bin
# Must be exactly 25165824 bytes (24 MiB)
```

**Back up all dump files to a safe off-machine location.** These are your recovery lifeline.

---

## Step 2: Extract Regions and Set HAP Bit

Perform these steps on your build machine.

### Build ifdtool

```bash
git clone https://review.coreboot.org/coreboot
cd coreboot
git submodule update --init --checkout

cd util/ifdtool
make
cd ../..
```

### Extract Regions

```bash
util/ifdtool/ifdtool -p cnl -x /path/to/m920q_original_24mb.bin
```

The `-p cnl` flag specifies Cannon Lake platform. This produces:

| Output File | Contents |
|-------------|----------|
| `flashregion_0_flashdescriptor.bin` | Intel Flash Descriptor |
| `flashregion_1_bios.bin` | BIOS region (coreboot replaces this — not needed) |
| `flashregion_2_intel_me.bin` | Intel CSME firmware |
| `flashregion_3_gbe.bin` | Gigabit Ethernet NVM (contains MAC address) |

### Set HAP Bit on Descriptor

The HAP (High Assurance Platform) bit tells the ME to enter a graceful disabled state after early initialisation.

> **Do NOT use me_cleaner on this platform.** The M920q uses ME v12. Stripping ME modules with me_cleaner puts ME v12 into an error state and can prevent boot. The coreboot project [recommends using the HAP bit](https://doc.coreboot.org/getting_started/faq.html) rather than me_cleaner.

```bash
util/ifdtool/ifdtool -p cnl -M 1 flashregion_0_flashdescriptor.bin
mv flashregion_0_flashdescriptor.bin flashregion_0_flashdescriptor.bin.orig
mv flashregion_0_flashdescriptor.bin.new flashregion_0_flashdescriptor.bin
```

### Place Blobs

```bash
mkdir -p 3rdparty/blobs/mainboard/lenovo/m720q_m920q

cp flashregion_0_flashdescriptor.bin \
   3rdparty/blobs/mainboard/lenovo/m720q_m920q/descriptor.bin

cp flashregion_2_intel_me.bin \
   3rdparty/blobs/mainboard/lenovo/m720q_m920q/me.bin

cp flashregion_3_gbe.bin \
   3rdparty/blobs/mainboard/lenovo/m720q_m920q/gbe.bin
```

The ME binary must be the **original unmodified** version extracted from your dump. The VBT (`data.vbt`) is already bundled in the coreboot source tree at `src/mainboard/lenovo/m720q_m920q/data.vbt`. The FSP binaries are fetched automatically via git submodules (`3rdparty/fsp/CoffeeLakeFspBinPkg/`).

---

## Step 3: Power Limits (Optional — for non-T CPUs)

If installing a 65W or 95W CPU (e.g., i7-8700, i9-9900) into the M920q, the 35W cooler cannot handle the full TDP. Coreboot can enforce power limits via the devicetree.

Edit `src/mainboard/lenovo/m720q_m920q/devicetree.cb`. Change:

```c
register "power_limits_config" = "{
    .tdp_pl2_override = 65,
}"
```

To:

```c
register "power_limits_config" = "{
    .tdp_pl1_override = 35,
    .tdp_pl2_override = 45,
}"
```

| Setting | Value | Purpose |
|---------|-------|---------|
| `tdp_pl1_override` | 35 | Sustained power limit in watts (M920q cooler design envelope) |
| `tdp_pl2_override` | 45 | Short burst turbo limit in watts |

> **Important:** Both `tdp_pl1_override` and `tdp_pl2_override` must be explicitly set. If only one is specified, the other defaults to 0 (unlimited). These values are written to MSR `0x610` (`MSR_PKG_POWER_LIMIT`) during boot.

If using a native 35W T-series CPU, you can leave the stock devicetree values.

---

## Step 4: Configure and Build

### Install Build Dependencies

```bash
# Debian / Ubuntu
sudo apt install git build-essential gnat flex bison libncurses-dev wget \
  zlib1g-dev libelf-dev python3 iasl acpica-tools nasm curl libssl-dev pkgconf m4
```

### Fetch Submodules

```bash
cd /path/to/coreboot
git submodule update --init --checkout
```

### Build Cross-Compiler Toolchain (once)

```bash
make crossgcc-i386 CPUS=$(nproc)
```

### Using the Provided defconfig

A minimal defconfig is provided in this repository. It configures:

- edk2 payload (MrChromebox fork) with PXE, UEFI Shell, TPM 2.0, Secure Boot
- SMMStore v2 for persistent UEFI NVRAM
- libgfxinit for native graphics initialisation
- Serial console on COM1 at 115200 baud

```bash
make distclean
cp m920q_defconfig .config
make olddefconfig
make -j$(nproc)
```

### Verify Output

```bash
ls -la build/coreboot.rom
# Must be 25165824 bytes (24 MiB)

./build/cbfstool build/coreboot.rom print
# Should list: bootblock, romstage, ramstage, dsdt.aml, payload (edk2),
# cpu_microcode_blob.bin, vbt.bin, fspm.bin, fsps.bin, etc.
```

---

## Step 5: Split and Flash

### Split the ROM

The 24 MiB ROM must be split to match the two physical flash chips:

```bash
dd if=build/coreboot.rom of=build/coreboot_bios1.rom bs=1M count=16
dd if=build/coreboot.rom of=build/coreboot_bios2.rom bs=8M skip=2

# Sanity check — recombine and compare
cat build/coreboot_bios1.rom build/coreboot_bios2.rom > build/coreboot_recombined.rom
sha256sum build/coreboot.rom build/coreboot_recombined.rom
# Must match
```

### Flash BIOS1 (16 MiB)

No extra pull-ups needed.

```bash
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q128.V" -w coreboot_bios1.rom

# Verify
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q128.V" -r verify_bios1.bin
sha256sum coreboot_bios1.rom verify_bios1.bin
```

### Flash BIOS2 (8 MiB)

Move clip to BIOS2. **Hold /WP (pin 3) and /HOLD (pin 7) HIGH.**

```bash
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q64JV-.Q" -w coreboot_bios2.rom

# Verify
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q64JV-.Q" -r verify_bios2.bin
sha256sum coreboot_bios2.rom verify_bios2.bin
```

---

## Step 6: First Boot

1. **Install RAM in DIMM1 only** — DIMM2 does not work under coreboot ([bug #592](https://ticket.coreboot.org/issues/592))
2. Connect a display via HDMI (try DisplayPort if no output on HDMI)
3. Power on and **wait up to 2 minutes** — the first boot performs memory training with no display output. Subsequent boots are fast (MRC cache)
4. The edk2 splash screen (coreboot hare) should appear. Press **Esc** to enter the boot menu

### If It Doesn't Boot

- Verify RAM is in **DIMM1 only**
- Try **both display outputs** (HDMI and DisplayPort)
- Wait the full 2 minutes — memory training can be slow
- If still no output, flash your original backup dumps to restore stock firmware (see Recovery below) and review your build configuration

---

## Step 7: Post-Boot Verification

### Power Limits

The sysfs powercap interface (`/sys/class/powercap/intel-rapl:0/constraint_*`) **does not reliably report PL2** on this platform. Use MSR `0x610` directly:

```bash
sudo apt install msr-tools
sudo modprobe msr
sudo rdmsr 0x610
```

This returns a 64-bit hex value:

| Bits | Field | Expected (35W/45W config) |
|------|-------|---------------------------|
| 14:0 | PL1 power limit | `0x118` (280 × 0.125W = 35W) |
| 15 | PL1 enable | `1` |
| 46:32 | PL2 power limit | `0x168` (360 × 0.125W = 45W) |
| 47 | PL2 enable | `1` |

Verified output: `0001816800dd8118` — both PL1 (35W) and PL2 (45W) active and enabled.

### ME Status

```bash
sudo dmesg | grep -i mei
```

With HAP bit set, you should see only `mei_hdcp` binding for HDCP content protection via the i915 graphics driver:

```
mei_hdcp 0000:00:16.0-b638ab7e-94e2-4ea2-a552-d1c54b627f04: bound 0000:00:02.0 (ops i915 hdcp_ops [i915])
```

There should be **no** `mei_me` driver initialising and no AMT-related messages. The HECI controller at `00:16.0` will still appear in `lspci` as `Communication controller: Intel Corporation Cannon Lake PCH HECI Controller` — this is normal hardware enumeration. HAP prevents the ME firmware from actively running while the PCH hardware remains visible on the PCI bus.

### Stress Test

```bash
sudo apt install stress-ng linux-tools-common
sudo turbostat --show Core,MHz,Busy%,PkgWatt,PkgTmp --interval 2 \
  stress-ng --cpu $(nproc) --timeout 120
```

`PkgWatt` should spike briefly toward 45W (PL2) then settle at ≤35W (PL1) within ~30 seconds.

---

## Recovery

If the machine doesn't boot, flash your original backup dumps externally:

```bash
# BIOS1 (no pull-ups needed)
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q128.V" -w m920q_bios1_dump1.bin

# BIOS2 (hold /WP and /HOLD HIGH)
sudo flashprog -p linux_spi:dev=/dev/spidev0.0,spispeed=1000 \
  -c "W25Q64JV-.Q" -w m920q_bios2_dump1.bin
```

---

## Internal Flashing (Subsequent Updates)

Once running coreboot, the flash can be updated internally without an external programmer:

```bash
sudo flashrom -p internal -N -w coreboot.rom --ifd -i bios
```

This writes only the BIOS region, leaving the descriptor, ME, and GbE regions untouched. No need to split the ROM for internal flashing.

---

## defconfig

```
#
# Lenovo ThinkCentre M920q coreboot defconfig
#
# Payload: edk2 (MrChromebox) — PXE, UEFI Shell, TPM 2.0, Secure Boot, NVRAM
# ME: HAP bit set in descriptor (no me_cleaner)
# Power: PL1=35W, PL2=45W (edit devicetree.cb — see Step 3)
#
# Known issues:
#   - DIMM2 slot broken (bug #592) — use DIMM1 only
#   - Front audio jacks don't work
#   - BIOS2 chip needs /WP + /HOLD held HIGH when flashing externally
#

CONFIG_VENDOR_LENOVO=y
CONFIG_BOARD_LENOVO_M920Q=y

CONFIG_IFD_BIN_PATH="3rdparty/blobs/mainboard/lenovo/m720q_m920q/descriptor.bin"
CONFIG_ME_BIN_PATH="3rdparty/blobs/mainboard/lenovo/m720q_m920q/me.bin"
CONFIG_GBE_BIN_PATH="3rdparty/blobs/mainboard/lenovo/m720q_m920q/gbe.bin"

# CONFIG_USE_ME_CLEANER is not set

CONFIG_PAYLOAD_EDK2=y
CONFIG_EDK2_REPO_MRCHROMEBOX=y
CONFIG_EDK2_UEFIPAYLOAD=y

CONFIG_SMMSTORE=y
CONFIG_SMMSTORE_V2=y

# CONFIG_RUN_FSP_GOP is not set

CONFIG_CONSOLE_SERIAL=y
CONFIG_TTYS0_BAUD=115200
CONFIG_DEFAULT_CONSOLE_LOGLEVEL=7
```

---

## Quick Reference

| Item | Details |
|------|---------|
| ROM size | 24 MiB (two chips: 16 + 8) |
| BIOS1 split | `dd if=coreboot.rom of=bios1.rom bs=1M count=16` |
| BIOS2 split | `dd if=coreboot.rom of=bios2.rom bs=8M skip=2` |
| BIOS2 quirk | /WP + /HOLD must be held HIGH externally |
| HAP bit | `ifdtool -p cnl -M 1 descriptor.bin` |
| Platform flag | `-p cnl` for all ifdtool commands |
| Power limit verify | `sudo rdmsr 0x610` (not sysfs) |
| DIMM | Slot 1 only (bug #592) |
| CPU whitelist | None — any LGA 1151 Coffee Lake / CFL-R CPU works |
| Boot Guard | Not fused — no deguard needed |
| ME disable | HAP bit only — do NOT use me_cleaner (ME v12) |

## References

- [Coreboot M920q documentation](https://doc.coreboot.org/mainboard/lenovo/m920q.html)
- [Gerrit #80609 — original M920q port](https://review.coreboot.org/c/coreboot/+/80609) by Maciej Pijanowski (3mdeb)
- [Bug #592 — DIMM2 not working](https://ticket.coreboot.org/issues/592)
- [Coreboot FAQ — ME and HAP bit](https://doc.coreboot.org/getting_started/faq.html)
- [flashprog](https://flashprog.org/)
- [MrChromebox edk2 features](https://docs.mrchromebox.tech/docs/faq.html)

## License

This guide is released under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
