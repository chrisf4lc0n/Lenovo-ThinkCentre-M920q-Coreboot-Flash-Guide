#
# Lenovo ThinkCentre M920q coreboot defconfig
#
# Payload: edk2 (MrChromebox) — PXE, UEFI Shell, TPM 2.0, Secure Boot, NVRAM
# ME: HAP bit set in descriptor (no me_cleaner)
#
# CPU: T-series (35W) ONLY on M920q — 65W CPUs fail (VRM limitation)
#      For 65W CPUs use M920x (tested) or P330 Tiny (same board, untested)
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

# iPXE network boot — adds "Network Boot" to edk2 boot menu
CONFIG_EDK2_ENABLE_IPXE=y
CONFIG_EDK2_IPXE_OPTION_NAME="Network Boot"

# CONFIG_RUN_FSP_GOP is not set

CONFIG_CONSOLE_SERIAL=y
CONFIG_TTYS0_BAUD=115200
CONFIG_DEFAULT_CONSOLE_LOGLEVEL=7
