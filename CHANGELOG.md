# Changelog

All notable changes to avocado-bsp-mic-742 are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0]

### Added
- Initial release: board support for the Advantech MIC-742 (carrier extension on
  `jetson-agx-thor`, 2026 feed).
- Carrier flash-time BCTs shadowing the stock names: MB1 pinmux (61 pin blocks —
  CAN2/CAN3, eth0/eth3 MDIO, gen3/gen9 I2C, PWM2/3, DAP2/DAP6), MB1 boot GPIO
  defaults, and the MB1 prod BCT I2C1 drive-strength fix that lets MB1 read the
  SoM EEPROM through the carrier's 1.8↔3.3V level shifter (NVIDIA bug 5459342).
- `tegra264-mic-74x-carrier.dtbo` — the whole kernel-DT carrier delta as a
  four-fragment overlay instead of a forked 256KB DTB: discrete Infineon SLB9670
  TPM 2.0 on SPI2 CS0, PCIe C3 (ASMedia ASM106x SATA), DCE SHA carveout, and
  internal-speaker audio routing. Shared verbatim with the sibling MIC-74x
  extension; verified offline with `fdtoverlay` against Advantech's shipped DTB.
- SATA enablement via three coordinated settings: `/uphy/uphy0-config = 6` and
  `/pcie/pcie@3 status = "okay"` baked into the carrier BPMP DTB, plus the
  overlay's `pcie@a808440000` fragment. `UPHY_CONFIG` is cleared to match
  Advantech; the BPMP's `uphy0-config` is what assigns the C3 lanes.
  `uphy0-config` is baked rather than left to `ODMDATA` because avocado's
  `initrd-flash.sh` `sign_binaries()` does not export `ODMDATA` to the flash
  helper, so `--odmdata` is never passed and `tegraflash_update_bpmp_dtb()`
  never runs. Missing it stops the board before the kernel with
  `PCIe(3): is enabled in DT without enabling it in UPHY DT` ->
  `BPMP firmware is not ready` -> BL31 `ASSERT` at `plat_setup.c:726`.
  Found on hardware during first provision of a MIC-743.
- Carrier module set with explicit intermediates: TPM (`tpm-tis-spi` +
  `tpm-tis-core`), CAN (`mttcan` + `can-dev`/`can`/`can-raw`/`can-bcm`), I2C
  devices (`at24`, `lm90`, `ina238`, `ina3221` + `i2c-core`/`regmap-i2c`), audio
  (`rt5640` + `rl6231`), `spidev`, `onboard-usb-hub`, and M.2 WiFi/WWAN.
- CI via the shared `avocado-linux/actions` reusable workflows: PR build check
  (`test.yml`) and tag-driven package + publish (`release.yml`).

### Notes
- Uses the discrete SLB9670 TPM rather than the OP-TEE fTPM: the overlay replaces
  `tegra264-ftpm.dtbo` and `kernel-module-tpm-ftpm-tee` is intentionally omitted.
  Enabling both would race for `/dev/tpm0` vs `/dev/tpm1` and silently break PCR
  sealing.
- Advantech's `usb3-1` trim is deliberately not carried; see the README.
- The T4000 module flashvars are derived from NVIDIA's
  `p3834-0000-p4071-0000-nvme.conf`, not from Advantech (their SDK only ships the
  T5000 profile). Carrier half is hardware-verified; module half is not.
