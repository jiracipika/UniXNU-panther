# BootShim-PE design (gs201 ABL contract)

Status: **design, not yet implemented** — the first phone-free implementable
chunk of Stage 2. Prerequisite reading: `GS201-BOOT-DEVIATIONS.md` (the
measured ABL contract), `mkpe-image.sh` in `jiracipika/hakios-gs201` (the
proven vehicle packer), and `BootShim.S` in uefi-panther (the generic shim's
SEC entry contract).

## The contract (HW-verified, HakiOS campaign 2026-11)

The ABL decompresses the boot image's kernel field and enters its PE32+
entry point with:

- x0 = x1 = 0 — no ImageHandle, no SystemTable, **no FDT pointer, no initrd**.
- MMU on (ABL's own page tables; identity mappings exist only where ABL
  mapped). Disabling the MMU before the final handoff is fatal (fetch
  VA≠PA once off — measured, not survivable).
- EL2; CNTPCT/CNTFRQ trapped until CNTHCTL is opened.
- Exception vectors: ABL owns VBAR_EL2. Installing our own fault-recovering
  vectors (record ESR/ELR/FAR, advance ELR past the faulting instruction)
  makes unknown-mapping probes safe; restore ABL's VBAR_EL2 before the
  final jump.

## Payload windows (TBD against ABL's live mappings at first light)

| Region | Address | Evidence |
| --- | --- | --- |
| UEFI FD | 0x90000000, 4 MiB | `Resources/Configs/panther.toml uefi_fd` |
| DXE heap | 0x90400000 + 0x08000000 | MemoryMapLib "DXE Heap" |
| UEFI stack | 0x9F000000 + 0x40000 | MemoryMapLib "UEFI Stack" |
| FDT window | 0x9F400000, 4 MiB | TBD — must not overlap ABL state |
| ABL state top-of-bank | 0xFD3FF000–0xFDA00000 | DTB (ramoops, bl_log) + HW (bl_old_log 0xfd880000) — never write |
| Splash framebuffer | 0xfac00000, 1080x2400 XRGB8888 stride 4320 | HW; DECON0+DPP0 still scan it out |

The vehicle (decompressed kernel field) is ABL-mapped by construction —
the decompressor wrote it. Every store the shim makes must ride an
ABL-mapped region until the shim owns the mapping.

## Shim algorithm

1. Install fault-recovering vectors at VBAR_EL2 (save ABL's value first).
2. Verify EL2 (`mrs CurrentEL` == 0xC); otherwise park forever.
3. Open CNTHCTL for timer access (EL1PCEN/EL1PCTEN at EL2).
4. Get the FD to 0x90000000 (its link address) and branch there for Mu SEC.
   Two routes, pick by measurement:
   - **Direct copy** if ABL maps a writable window covering 0x90000000:
     copy FD (+ FDT placed after it) and branch — the generic BootShim
     algorithm, under an MMU-on context.
   - **Page-table clone + extend**: ABL's tables may not cover 0x90000000.
     Read ABL's TTBR0_EL2/TCR_EL2/MAIR_EL2, replicate its L0/L1 into
     shim-owned tables built in the vehicle's zeros area (guaranteed
     mapped+writable), add block descriptors for the windows the shim
     needs (FD window WB, FDT window, stack), switch TTBR0_EL2 with the
     PC+stack regions identically mapped in old and new tables, `tlbi alle2`.
     Then copy FD+FDT, restore ABL's VBAR_EL2, `br 0x90000000`.
5. Mu SEC (ArmPlatform-style PrePi) rebuilds the MMU from
   `GetMemoryMap()` (our MemoryMapLib) and re-enables it — the shim's MMU
   state only has to keep PC+stack mapped until the `br` lands. OPEN
   QUESTION for the first hardware session: confirm SiliciumPkg/Sec's entry
   expectations (it is documented to run after the generic BootShim's
   MMU-off copy; the ABL MMU-on entry is the deviation this design
   reconciles).

## Vehicle layout inside the boot image

Reuse the proven container recipe (mkboot-gs201.sh): v4 boot header,
lz4-legacy-compressed kernel field, stock AVB0 vbmeta at the page-aligned
offset, AVBf footer. The kernel field decompresses to the vehicle:

- `+0x0000`–`+0x10000`: PE32+ headers + shim code (machine 0xAA64,
  AddressOfEntryPoint → shim), hand-crafted by the packer (no lld-link
  needed for a raw-blob PE: DOS header + NT headers + one section).
- `+FD_OFFSET` (≥ 0x200000, page aligned): the FD (SILICIUM_UEFI.fd).
- `+FDT_OFFSET`: the board DTB (dtc build of panther.dts + gs201.dtsi).
- zeros to PE SizeOfImage (0x22e7a00, matching the stock kernel field).

## Stage ring (no-console observability)

The shim appends `UXNUSHIM:stage=<n> esr=<v> far=<v>` markers to the
ramoops console record at 0xfd3ff000 (HW-verified to survive warm reboot;
read back via TWRP adb root). Probe the region with the fault-recovering
vectors before the first write; if ABL does not map it, fall back to the
timing bands only (28–31 s comeback = shim executed and reset; 104 s =
never jumped).

## Phone-free verification (implementable now)

1. Build the PE vehicle with a placeholder FD: python packer craft + parse
   (machine 0xAA64, entry RVA, section table, SizeOfImage ≥ FD+FDT size).
2. lz4-legacy compress + splice into the AVB container per mkboot-gs201.sh,
   then verify: ABL-valid shape, decompressed size, PE header parse, vbmeta
   offset, footer.
3. Assemble BootShim-PE.S with the host GNU as (aarch64); validate
   instruction encoding by disassembly (objdump) before any hardware run.

## First-light procedure (next hardware session)

1. Build FD + board DTB; pack the vehicle; `fastboot boot` via
   phone-test-loop.sh (exit codes 0 heartbeat / 2 no-jump recovered /
   3 parked; timing bands 28–31 s vs 104 s).
2. Read the stage ring via TWRP adb root; fix the first failed stage.
3. Display-first: SimpleFbDxe publishes the GOP on the splash framebuffer
   (PCDs already set); the panel fetch problem (DECON0 TRIG_CON 0x3070,
   frozen) is Stage 3 work (gs-decon port with panther S6E3FC3-P10 modes).
   Until then the observables are the stage ring + timing bands; QR log
   export (AdvancedLogger + AdvancedLogDxe) comes with Stage 4.
