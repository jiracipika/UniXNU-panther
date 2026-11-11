# UniXNU panther bring-up plan

Ordered from the porting steps with the gs201-specific deviations
(docs/GS201-BOOT-DEVIATIONS.md) folded in. Each step lists its exit
checkpoint. Update the porting record area when a step completes.

## Stage 0 — host toolchain (Linux, no sudo)

1. `scripts/linux/setup-toolchain.py` from the UniXNU checkout (the CLI
   notes the kernel step uses it; macOS hdiutil bits are not needed until
   payload seed work).
2. Exit check: `./unixnu build kernel -D ../UniXNU-panther` starts and
   gets past config/toolchain (kernel source is already fetched).

## Stage 1 — QEMU verification of the generic chain (no hardware)

UniXNU has a host QEMU workflow (docs/reference/qemu.md) and an
ArmVirtPlatform component. Run the iPadOS 17.7.11 chain in QEMU before any
phone session: it proves the resolved patch series, prelink and kernel
build on our toolchain, and gives the reference boot log shape.
Exit: XNU boots to the documented QEMU checkpoint.

## Stage 2 — uefi-panther firmware platform (the critical path)

Fork Mu-Silicium as `jiracipika/uefi-panther`:

1. Samsung/gs201 platform skeleton under Silicon/Samsung +
   Platforms/Google/pantherPkg (paths already reserved in board.toml).
2. BootShim-PE: repackage BootShim as a PE32+ image entered by the Pixel
   ABL (port the proven mkpe-image.sh vehicle; run under ABL's MMU, no
   MMU-off preambles, open CNTHCTL first, install the FDT ourselves).
3. Exit check: SILICIUM_UEFI.fd builds; packaged through the AVB
   container it reaches the phone and the first UEFI code runs
   (timing-band + plog observable until a better channel exists).

## Stage 3 — display (menu + QR logs depend on it)

DECON0/DPP0 scanout on panther (S6E3FC3-P10 1080x2400 modes; port from
linux-omarchy gs-decon which is cheetah-profiled). The dynamic-fetch
problem is parked and unresolved anywhere — budget for it; until it is
solved the QR log exporter has no display either. Exit: a frame we paint
is user-visible.

## Stage 4 — UniXNU boot manager on hardware

UniXNUPkg + DarwinBoot + UniXNURamDiskDxe/DiskIoDxe/FAT in the DSC/FDF,
AdvancedLogDxe + Tools menu from day one. Exit: menu appears (or
AdvancedLogger output read via QR/plog), firmware DTB installed.

## Stage 5 — storage → payload

UFS bring-up (google,ufshc 0x3c400000; PixOS gs101 shim + Scion Exynos
driver as sources; gs201 ufsinit + PHY tables needed), then the
dual-system partition procedure for the payload volume (no initrd
handoff exists on this ABL). Exit: FAT payload volume readable in UEFI.

## Stage 6 — kernel + userland

DarwinBoot handoff with the XNULDRST-slot pattern the HakiOS loader
proved (the return-stub contract is directly reusable for crash
reporting), then iPadOS 17.7.11 userland from userdata.
