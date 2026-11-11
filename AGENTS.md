# Device agent guide

This repository owns the board DTS, peripheral wiring, component selections,
firmware inputs, partition layout, and packaging for the Google Pixel 7
(panther, Tensor G2 gs201).

## Read first

Read this README and the following UniXNU documentation before changing the port:

| Task | Required reading |
| --- | --- |
| Every port | [Porting](https://unixnu.com/porting/), [device repository](https://unixnu.com/porting/device-repository), [component catalog](https://unixnu.com/reference/components), UniXNU `components.toml` |
| Hardware | [Device tree](https://unixnu.com/porting/device-tree), [matching](https://unixnu.com/drivers/device-tree-matching), `UniXNU-gs201-common` README/AGENTS/soc.toml/DTSI |
| gs201 specifics | **`docs/GS201-BOOT-DEVIATIONS.md` in this repository — read before any firmware or boot-image work** |
| Component | Its site page, README, catalog build command, dependencies and source |
| Kernel | [Source and patches](https://unixnu.com/xnu/source-and-patches), [patch reference](https://unixnu.com/xnu/patch-reference), selected OS profile |
| Images | [Prelink](https://unixnu.com/concepts/prelink), [payload](https://unixnu.com/concepts/payload), [UEFI](https://unixnu.com/porting/uefi-firmware) |
| Verification | [Building](https://unixnu.com/building), [bring-up](https://unixnu.com/porting/bring-up) |

## gs201 context

gs201 has no existing UniXNU or Mu-Silicium support: the SoC repository
(`UniXNU-gs201-common`), the firmware repository (`uefi-panther`, not yet
created), and this device repository are all new. The board's bootloader
(Pixel ABL) deviates from the generic Qualcomm boot flow in four ways
(PE32+ entry, AVB-partition-shaped boot images, no initrd/FDT handoff,
trapped timers). The deviations document carries the measured facts and
points at the proven vehicles in `jiracipika/hakios-gs201` and
`jiracipika/hakios-pixel7`.

Hardware values carry evidence tags: `DTB` (factory device tree, decoded),
`HW` (observed on panther silicon), `TBD` (placeholder). Do not promote a
`TBD` value without the observation that verifies it.

## Work

Search the catalog and inspect existing component interfaces before adding code.
Match hardware scope, OS ABI, and board dependencies. Shared SoC resources and
operations belong in `UniXNU-gs201-common`. Reusable drivers and libraries
belong in their component repositories. Use canonical component IDs and preserve
runtime identifiers recorded in the catalog.

Replace every `PENDING_`/`NOT-` value, set the board name and firmware paths, and select
the complete dependency set. Verify memory ranges, CPU affinities, interrupts,
firmware hashes, and partition geometry from board evidence. Set `example = false`
after completing the inputs. Pin the SoC commit and inspect resolved profiles.

Build with the packaging script from the device repository; every value it needs
comes from board.toml ([package], [os_overrides], the sections the steps read):

```sh
scripts/package.sh --check --os <profile>              # deps, preflight, patches; stops before building
scripts/package.sh --os <profile> --uefi ../uefi-panther
```

The platform seed (`UNIXNU_UNIQUE_CHIP_ID`) and the firmware checkout are not in
any repository. `.github/workflows/package.yml` runs the same script on a macOS
runner: run it in a fork, set the seed as the fork's secret, download the images,
then delete the run.

For a Mu-Silicium initrd payload, include `UniXNURamDiskDxe`, DiskIoDxe and
FAT support in the firmware DSC and FDF — **on this board the initrd handoff
does not exist**; see the deviations document for the payload-delivery plan
before building the firmware.

Record commands, revisions, output paths, results, and unresolved inputs.
Report hardware boot, storage, display, input, and GPU results separately.
Finish both new ports and migrations with the same
[repository checklist](https://unixnu.com/porting/device-repository#completion).
Follow [publication](https://unixnu.com/porting/device-repository#publish), add
the `unixnu` GitHub topic, and update the device revision in UniXNU's
`devices.toml`. Keep this README concise.

## Porting record

Read the device's `porting.toml` and the
[record guide](https://unixnu.com/porting/records) before implementation.
For storage, also read the [UFS guide](https://unixnu.com/porting/ufs).
Update each affected area's findings or evidence with its code change.
The documentation site renders this same record. Follow its schema and
completion levels for both maintainer and agent work.

## Local builds and boot diagnosis

Copy `local.toml.example` to `local.toml` (ignored) for local overrides. On this
board, boot diagnosis runs through the HakiOS phone test loop and timing bands
(`docs/GS201-BOOT-DEVIATIONS.md` §4) until the firmware's own log channel works;
`fastboot boot` sessions never touch flash. Record the last confirmed stage and
relevant logs.
