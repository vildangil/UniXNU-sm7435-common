# Agent instructions: UniXNU SM7435 / Parrot

This fork is **Qualcomm SM7435** SoC common support, not SM8550 support.
The device repository is vildangil/UniXNU-Dizi-Ruan; do not modify its
known-working Mu-Silicium EDK2, PMIC buttons or image layout as part of CPU
driver bring-up.

Always verify hardware details against:
- LineageOS/android_kernel_xiaomi_sm7435-devicetrees at qcom/parrot.dtsi
  commit 849a8a76b0959f7e7c12eb67081591248b2f8e7b
- UniXNU-Dizi-Ruan/data/dizi-fdt-observed.json (dizi; not ruan).
- UniXNU/ArmCpu and UniXNU device-tree/SoC support documentation.

Do NOT activate or copy `kits/Sm8550Platform`, sheng power hooks,
Adreno740/AdrenoMetal, SM8550 SMMU, UFS, GPU or clock addresses.
Those source files are historic and deliberately unreferenced.

`patches/xnu/series` is ordered and contains only SM7435-specific
changes. Native secondary-CPU startup is experimental and must be tested
only after DarwinBoot actually enters the XNU kernel. Keep cpus=1 as the
device bring-up default, and never claim SMP was hardware verified from
a build-only result.

Do not enable any GitHub Actions `push` or `pull_request` build trigger.
Report separate levels of evidence: reference source, compilation,
UEFI behavior, kernel handoff, and actual multi-core execution.

Read upstream:
https://unixnu.com/porting/soc-support/
https://unixnu.com/porting/device-tree/
https://unixnu.com/porting/drivers/
