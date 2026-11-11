# UniXNU panther

UniXNU device support for the **Google Pixel 7** (panther, Google Tensor G2
`gs201`). UniXNU boots iOS and macOS on non-Apple ARM64 hardware through UEFI;
this repository supplies the board facts that make the generic stack run on
this phone.

Status: **device record and SoC description; firmware pending.** gs201 has no
existing UniXNU, Mu-Silicium, or other-UEFI support — the SoC repository
(`UniXNU-gs201-common`), the firmware repository (`uefi-panther`), and this
device repository are all new ports. What exists today:

- Verified hardware description: factory-image device trees decoded and
  cross-checked against direct register observation on the phone (see
  `dts/panther.dts`, `UniXNU-gs201-common/dts/gs201.dtsi` — every value
  carries an evidence tag).
- A measured account of how Pixel's ABL actually boots (PE32+ entry, AVB
  partition-shaped images, no initrd/FDT handoff, trapped timers) with
  silicon-proven vehicles: `docs/GS201-BOOT-DEVIATIONS.md`.
- The preceding HakiOS port's tools remain usable assets: boot-image packer
  and PE loader vehicles (`jiracipika/hakios-gs201`), phone test loop and
  backup scripts (`jiracipika/hakios-pixel7`).

Follow [Port a device](https://unixnu.com/porting/) from input collection
through publication. The porting record (`porting.toml`) covers every
subsystem with its evidence and next measurement.

## Bring-up order (board-specific)

1. `uefi-panther`: Mu-Silicium platform for gs201 (SEC from the PE entry
   under ABL's MMU, timer untrap, DRAM from ABL) — the critical path.
2. Display: DECON0/DPP0 basic scanout (the menu and QR logs depend on it;
   see deviations §5 for the parked failure).
3. Storage: UFS (google,ufshc at 0x3c400000) → payload partition.
4. Kernel: UniXNU build for the iPadOS 17.7.11 profile, DarwinBoot handoff.
5. Input, GPU (Mali G710 CSF), network (Broadcom over PCIe) — after root.

## Device facts

| Item | Value |
| --- | --- |
| SoC | Google Tensor G2 (gs201), Exynos-class |
| CPUs | 2x Cortex-X1 + 2x Cortex-A78 + 4x Cortex-A55 |
| RAM | 4 banks: 0x80000000 + 0x880000000/0x900000000/0x980000000 (8 GiB unit) |
| Storage | UFS 3.1, `google,ufshc` |
| GPU | Mali G710 MC10, Valhall arch 10.8 CSF |
| Display | Samsung S6E3FC3-P10, 1080x2400, DECON0/DPP0/DSIM0 pipeline |
| Bootloader | Pixel ABL (rango), PE32+ entry, PSCI smc |
| OS profile | ipados-17.7.11-21H461 |
