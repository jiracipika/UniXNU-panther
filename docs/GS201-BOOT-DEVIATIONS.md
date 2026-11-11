# gs201 boot deviations from the generic UniXNU path

Everything in this document was measured on the panther (Pixel 7, gs201,
bootloader rango BP4A.251205.006) during the preceding HakiOS port, 2026-11-01
through 2026-11-09, and re-verified where noted. It overrides the generic
assumptions in UniXNU's porting docs for this board. Working vehicles and
scripts from that port live in the sibling repositories `jiracipika/hakios-gs201`
and `jiracipika/hakios-pixel7` (device loader, boot-image packer, phone test
loop, decoded DTBs).

## 1. The ABL enters the kernel field through its PE32+ entry point

The generic UniXNU boot image packs BootShim as a gzip stream with an ARM64
image header. Pixel's ABL rejects none of that loudly — it simply never jumps:

- The decompressed kernel field must be a **PE32+ image** (MZ code0 pattern,
  PE\0\0 at file offset 0x40, machine 0xAA64). ABL reads `AddressOfEntryPoint`
  and enters there. A classic arm64 Image header or an MZ without a valid PE
  header never executes (proven across 10+ runs; 104 s watchdog floor).
- Handoff state at entry: **x0 = x1 = 0** (no ImageHandle, no SystemTable, no
  FDT pointer), **MMU on** (ABL's own page tables — identity mappings exist
  only where ABL mapped; probing unknown regions needs fault-recovering
  handlers), **EL2**. Architectural timer reads (CNTPCT/CNTFRQ) are trapped or
  frozen until the boot code opens CNTHCTL.
- Disabling the MMU at the PE entry is fatal (fetch VA≠PA once off). Any
  MMU-off jump happens only at the final handoff, from code that stays mapped.

## 2. The boot image must be the full 64 MiB AVB-partition shape

`fastboot boot` on this ABL accepts **only** images shaped like the boot
partition itself: v4 header (header_size 1584) + lz4-legacy kernel stream +
stock AVB0 vbmeta at the page-aligned offset + AVBf footer in the last 64
bytes. Bare kernels with plain headers are rejected (no jump, 104 s floor).
The proven vehicle: patch a copy of the real stock boot image in place
(header + kernel region), keeping stock vbmeta/footer offsets — the
Magisk-patch shape. Bare images without vbmeta/footer are also accepted, but
the stock-shaped container is the proven default. Packer: `mkboot-gs201.sh`
and `mkpe-image.sh` in `jiracipika/hakios-gs201`.

The stock kernel stream decompresses to exactly 0x22e4a00 bytes (12 KiB below
the PE header's SizeOfImage 0x22e7a00 — vehicles extend with zeros; proven
fine). Image size edits are unconstrained: the "modification-volume gate"
reported in early notes was a measurement artifact and does not exist.

## 3. No initrd, no FDT handoff — payload discovery must be re-designed

The generic flow expects ABL to place the FAT payload in RAM and record
`/chosen/linux,initrd-start` in the firmware FDT. On this ABL, measured:

- **x0 = x1 = 0** at PE entry: no EFI handle, no DTB pointer.
- A self-relative + absolute scan over the full low DRAM bank and the high
  banks found **no ANDROID!/lz4/GSPE magic** in any region the ramdisk field
  could occupy: the ABL does not deliver the ramdisk field findably for
  `fastboot boot`.

Consequences for UniXNU integration:

1. The firmware FDT cannot come from ABL. BootShim-PE must carry or locate the
   DTB itself (packed in the boot image behind the PE blob, or embedded in the
   FD region) and install it for SEC.
2. `UniXNURamDiskDxe`'s initrd handoff (HOB from the preserved FDT) has no
   input at first boot. The payload volume must come from block storage
   (UFS partition — gs201 UFS HCI at 0x3c400000 per the factory DTB) per the
   dual-system procedure, or from a small test payload embedded in the boot
   image blob (the HakiOS GSPE-in-container technique, proven for ~16 MiB;
   the 64 MiB container caps this route — full payloads do not fit).
3. First light needs no payload at all: the UEFI menu, AdvancedLogger + QR
   export, and block device enumeration run before any payload discovery.

## 4. Proven ABL-context mechanics to reuse in BootShim-PE

From the HakiOS loader (v21 GSC container verified end-to-end: blob found,
kernel copied, stub registered, jump executed):

- Fault-recovering exception vectors (record ESR/ELR/FAR, advance ELR, skip)
  make unknown-mapping probes safe; every store the loader makes rides an
  ABL-mapped identity region.
- Working reset from the fastboot context, in observed-reliability order:
  SMC 0x82000504 (fast deterministic reset; with a recovery preset it lands
  in recovery); cluster watchdog enable-time-violation recipe at
  0x200c4000/0x200c5000 (unlock with key 0xc0ffee at base+0x04, then WTCON=0,
  WTMINCNT=0x100, WTCNT=0x80, WTCON=0x102021 — instant windowed violation);
  PSCI SYSTEM_RESET silently no-ops here.
- The 104 s comeback = ABL's own watchdog reaping a parked/hung loader;
  ~28-31 s comeback = our code executed and reset (ABL short watchdog +
  Android boot). These timing bands were the primary no-console observable
  across dozens of runs; do not re-derive them.
- `bootreason` (Android) after a run: `watchdog,apc,early`.

## 5. Display is the critical bring-up path (no serial on this host)

No Suzy-Q UART cable is available; observability = timing bands, pstore rings
(root via TWRP recovery adb), and ABL bl_old_log at 0xfd880000. The UEFI
menu and QR log export need a working framebuffer, so display comes first:

- ABL state: DECON0 (0x1c240000) TRIG_CON holds 0x3070 — HW trigger masked,
  screen frozen on the splash. The splash framebuffer is 0xfac00000, XRGB8888,
  1080x2400, stride 4320; DPP0 RDMA (0x1c0b0040) points at it.
- Two attempts to force DECON to fetch new frames (TRIG_CON force-write
  0x3061; TRIG_CON RMW clear-mask + DPP0 SHD_REG_UP_REQ 0x8010003f) **did not
  make the phone fetch** — user-visible frozen logo both times. Parked there.
  linux-omarchy's `gs-decon` is the known-good display code family (panel
  takeover + DCS brightness PASSED on panther, D3 flips pending) but it is
  cheetah-profiled (1440x3120) and refuses panther's 1080x2400 — panther
  S6E3FC3-P10 modes must be added.
- DSIM (0x1c2c0000) raises an **external abort** when touched while
  clock-gated — twice reproduced, MMU state irrelevant. Display code must
  gate DSIM access behind the DSIM clock bring-up path, never poke it cold.

## 6. Board facts snapshot (verify before reuse; full record in porting.toml)

- Serial 36071FDH20009D, slot b, Evolution X 11.10 ROM (spoofs stock identity;
  OS-level props untrustworthy), bootloader unlocked (no wipe needed for
  fastboot boot sessions), TWRP on the active slot, unrooted Android.
- RAM banks: 0x80000000, 0x880000000, 0x900000000, 0x980000000 (sizes
  unverified).
- Factory DTBs: `jiracipika/hakios-gs201/dtb/panther-lga-base{,-2}.dtb`
  (vendor_kernel_boot, "google,lga"). Live nodes (mali, decon/dpp/dsim) come
  from dtbo overlays + ABL fixups.
- CPU topology: 2x Cortex-X1 (cpu6-7), 2x Cortex-A78 (cpu4-5), 4x Cortex-A55
  (cpu0-3); PSCI smc; GICv3 GICD 0x5880000, GICR 0x5900000.
- Phone test loop with auto-recover: `phone-test-loop.sh` in
  `jiracipika/hakios-pixel7` (serial-strict adb; exit codes: 0 heartbeat,
  2 no-jump recovered, 3 parked).
