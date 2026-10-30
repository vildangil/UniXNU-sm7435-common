# UniXNU SM7435 (Parrot) common support

This is the SoC counterpart for [UniXNU-Dizi-Ruan](https://github.com/vildangil/UniXNU-Dizi-Ruan). It started as a fork of UniXNU-sm8550-common, but **SM8550-specific CPU power, Adreno 740/GMU, SMMU, and UFS settings must not be reused blindly**.

## Implemented groundwork

| File | Scope | Status |
| --- | --- | --- |
| `soc.toml` | Qualcomm SM7435 identity; only generic `ArmCpu` and `ArmDTPlatform` selected | Defined |
| `dts/sm7435.dtsi` | Eight MPIDRs 0x000..0x700; 4 E + 4 P; PSCI SMC; GICv3 MMIO/stride; architectural timer | Source-matched; XNU untested |
| `configs/sm7435.config` | Generic ARM64 CPU, DT and timer capabilities | Source configuration only |
| `patches/xnu/sm7435-smp-secondary.patch` | Eight-CPU capacity and unconditional Aff1-aware reset-vector comparison | Source-checked, hardware SMP untested |
| `patches/xnu/series` | SM7435 SMP patch only | Experimental, must not claim multi-core boot |

The original fork's `dts/sm8550.dtsi`, `configs/sm8550.config`, `kits/Sm8550Platform/`, and `patches/xnu/sheng-platform-power.patch` are **legacy reference files only**. They remain for code comparison. None is selected by `soc.toml`, `configs/sm7435.config`, or `patches/xnu/series`. Do not compile them as SM7435 drivers.

## Evidence and differences from SM8550

- Qualcomm Parrot device tree: [`qcom/parrot.dtsi`](https://github.com/LineageOS/android_kernel_xiaomi_sm7435-devicetrees/blob/849a8a76b0959f7e7c12eb67081591248b2f8e7b/qcom/parrot.dtsi) at commit `849a8a76b0959f7e7c12eb67081591248b2f8e7b`.
- Dizi bootloader FDT-derived addresses: [`data/dizi-fdt-observed.json`](https://github.com/vildangil/UniXNU-Dizi-Ruan/blob/main/data/dizi-fdt-observed.json).

| Resource | SM7435 / Parrot | SM8550 / sheng |
| --- | --- | --- |
| CPU count | 8, split 4 E + 4 P | 8, split 3 E + 5 P in reference DTS |
| MPIDR | 0x000..0x700 (Aff1) | 0x000..0x700 (Aff1) |
| GICD base | 0x17200000 | 0x17100000 |
| GICR base | 0x17260000, size 0x100000 | 0x17180000, size 0x200000 |
| GICR stride | 0x20000 | 0x40000 |
| PSCI | `arm,psci-1.0`, SMC | `arm,psci-1.0`, SMC |
| Timer | 19.2 MHz; virtual timer PPI 11 / INTID 27 | 19.2 MHz |

## 2026-10-31 Aff1 reset-vector correction

The pinned SoC patch now preserves **all 16 low affinity bits** of
`MPIDR_EL1` in `osfmk/arm64/start.s` **regardless of
`HAS_CLUSTER`**. In Qualcomm Parrot, all eight CPU IDs are
`0x000, 0x100, ..., 0x700`; masking only `0xFF` would
incorrectly make all eight look like CPU0 during the XNU
`ResetHandlerData` / `CPU_PHYS_ID` match.

The previous patch used `#if HAS_CLUSTER || defined(BCM2837)`.
There was no confirmation that either macro was defined in this
SM7435 build, so the fallback retained the old broken `#0xFF`
mask. The new SoC-only patch is unconditional:
`and x0, x15, #0xFFFF`. It preserves the **physical**
MPIDR affinity ID for comparison against the CPU's recorded
physical ID; it is *not* meant to turn `0x100` into the logical
CPU index `1`. That distinction remains the job of the
kernel/ADT topology mapper.

The patch was checked with an independent `git apply --check`
fixture and all eight MPIDR values were evaluated for uniqueness.
It is a **source-level repair, not a hardware SMP confirmation**.
Do not claim secondaries are online or set a multicore default
until the platform's PSCI CPU_ON, per-core exception entry,
GIC redistributor and timer delivery are checked on real dizi.
The board's safe default stays `cpus=1`; `cpus=2` is opt-in.

## Realistic XNU multi-core plan

The upstream [`ArmCpu` driver](https://github.com/UniXNU/ArmCpu/blob/main/ArmCpu.cpp) already has paths for GICv3 redistributors, architectural timer, PSCI `CPU_ON`, SGIs/IPIs and secondary `IOCPU` registration. Its early SMP code still uses `sheng-smp` boot arguments and contains device-specific assumptions; this **is not yet a validated SM7435 multi-core driver**.

1. Build the real patched XNU and `ArmCpu` for the exact compatible OS ABI. The SoC patch must apply cleanly; do not assume it does.
2. Implement an **on-device verified** memory map, XNU ADT and DarwinBoot handoff; boot XNU initially with `cpus=1`.
3. Validate GICD/GICR ID readbacks, PSCI version, per-core MPIDR and timer on the device.
4. Enable secondaries **one at a time**, recording PSCI return values, redistributor ownership, timer/IPI delivery and boot progress. Do not force SMMU bypass or replace Qualcomm power/clock registers with SM8550 values.
5. Only then consider general `8 CPU` support. GPU, UFS, touch, display and userland are separate problems.

A successful `dtc`/patch/config check is not proof of a bootable kernel. Builds are manual, and this repository must not trigger a kernel or firmware workflow after `push`.
