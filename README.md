# bsp-mic-742

Board support for the **Advantech MIC-742** — the NVIDIA Jetson AGX Thor **T4000**
(P3834-0000 CVM) variant of Advantech's MIC-74x industrial edge AI system.

Layers on top of the `jetson-agx-thor` target. 2026 feed only.

The **MIC-743** is the same carrier with a T5000 (P3834-0008) module and has its own
extension, [`bsp-mic-743`](https://github.com/avocado-linux/bsp-mic-743).

> ## ⚠️ The T4000 module configuration is not hardware-verified
>
> The **carrier** half of this extension is verified — it is byte-identical to the
> MIC-743's, and the MIC-743 was introspected on real hardware.
>
> The **module** half is not. Advantech's shipped SDK is labelled MIC-742 but its flash
> SOP only ever runs the T5000 profile (`jetson-agx-thor-devkit` →
> `p3834-0008-p4071-0000-nvme.conf`), so it contains no T4000 flash configuration to
> copy. The values in `carrier.env` are derived from NVIDIA's own
> `p3834-0000-p4071-0000-nvme.conf` in the same R39.2.0 BSP, which is exactly how L4T
> selects a T4000.
>
> Supporting evidence that a T4000 variant is real: Advantech shipped a
> `tegra264-p4071-0000+p3834-0000-nv.dtb` carrying the **same** carrier patch as the
> T5000 DTB, which they would not have built otherwise.
>
> **Flash a T4000 and confirm before treating this as proven.**

## Why two extensions rather than one

The carrier is genuinely module-agnostic: the pinmux, GPIO and prod BCTs are all named
`p3834-xxxx-p4071-0000` (SOM-SKU independent), and the kernel-DT overlay is byte-identical
between the two module DTBs.

What is *not* agnostic is the module selection — twelve flashvars:

| Flashvar | MIC-743 / T5000 | MIC-742 / T4000 |
|---|---|---|
| `DTB_FILE` / `DTBFILE` | `…+p3834-0008-nv.dtb` | `…+p3834-0000-nv.dtb` |
| `BPFDTB_FILE` | `…-3834-0008-4071-xxxx…` | `…-3834-0000-4071-xxxx…` |
| `BPF_FILE` | `bpmp_t264-TA1090SA-A1_prod.bin` | `bpmp_t264-TE1070M-A1_prod.bin` |
| `BCTFILE` / `EMC_BCT` | `…p3834-0008-sdram-bct-l4t.dts` | `…p3834-0000-…` |
| `WB0SDRAM_BCT` | `…p3834-0008-…warmboot…` | `…p3834-0000-…` |
| `BPMP_MEM_CONFIG` | `…p3834-0008-sdram-dfs.dts` | `…p3834-0000-…` |
| `PMIC_CONFIG` | `…pmic-p3834-0008-…` | `…pmic-p3834-0000-…` |
| `MISC_CONFIG` | `…misc-p3834-0008-…` | `…misc-p3834-0000-…` |
| `CHIP_SKU` | `00:00:00:A0` | `00:00:00:E2` |
| `RAMCODE` | `12` | `0` |
| `CHECK_BOARDSKU` | `0008` | `0000` |

Flashing a T4000 with T5000 SDRAM training and PMIC config is not a soft failure. Splitting
the extensions also lets each pin `CHECK_BOARDSKU`, so a mismatched module fails fast at
the tegraflash board-ID check instead of being mis-flashed — the same reasoning as the
Orin-era `bsp-mic-733-ao5a1` / `bsp-mic-733-ao6a1` split.

Every stock file referenced above already ships in `tegraflash-bsp` for this MACHINE, so
the only module-specific artifact carried here is the BPMP DTB.

## What this extension carries

See [`bsp-mic-743`'s README](https://github.com/avocado-linux/bsp-mic-743) for the full
carrier description — the flash-time BCTs, the four-fragment kernel-DT overlay, and the
offline `fdtoverlay` verification all apply identically here.

The one file that differs is `tegra264-bpmp-3834-0000-4071-xxxx-adv.dtb`: the stock T4000
BPMP DTB with the same two carrier properties the MIC-743's carries. Regenerate it with:

```sh
dtc -I dtb -O dts <stock>/tegra264-bpmp-3834-0000-4071-xxxx.dtb > bpmp.dts
# /pcie/pcie@3  { status = "okay"; }     enable the ASMedia SATA controller
# /uphy         { uphy0-config = <0x06>; }  UPHY lane map that includes PCIe C3
dtc -I dts -O dtb -o tegra264-bpmp-3834-0000-4071-xxxx-adv.dtb bpmp.dts
```

Both are module-SKU independent. `uphy0-config` **must** be baked rather than left to
`ODMDATA`: see the MIC-743 README — `sign_binaries()` in avocado's `initrd-flash.sh` does
not export `ODMDATA` to the flash helper, so `--odmdata` is never passed and the BPMP DTB
is never patched. Without it the board dies at `PCIe(3): is enabled in DT without enabling
it in UPHY DT` → `BPMP firmware is not ready` → BL31 `ASSERT`.

## TPM: discrete, not fTPM

The MIC-74x carries an **Infineon SLB9670 TPM 2.0** on SPI2 CS0, so this extension
replaces the MACHINE's `OVERLAY_DTB_FILE` (`tegra264-ftpm.dtbo`) rather than appending to
it, and ships `kernel-module-tpm-tis-spi` but not `kernel-module-tpm-ftpm-tee`. Shipping
both would race for `/dev/tpm0` vs `/dev/tpm1` and silently break PCR sealing. See the
MIC-743 README for the full rationale.

## Using this extension

`bsp-mic-742` is an [Avocado](https://avocadolinux.org) extension — a reusable fragment of
build- and runtime-configuration that you compose into your own Avocado project. To use it,
declare it as a package-sourced extension in your `avocado.yaml` and add it to a runtime:

```yaml
extensions:
  avocado-bsp-mic-742:
    source:
      type: package
      version: "*"        # or pin an exact version

runtimes:
  my-runtime:
    extensions:
      - avocado-bsp-mic-742
```

Then install and build:

```sh
avocado install   # fetches + installs the SDK, extensions and runtime deps from your config
avocado build     # builds the SDK compile steps, extensions and runtime images
```

`avocado install` pulls the extension from your target's package feed and merges its
config into your project; `avocado build` then produces the runtime.
