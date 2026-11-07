# CSK6: the three engines

| Engine | Details | How you program it |
|---|---|---|
| Arm Cortex-M33 ("AP") | Up to 300 MHz. Single-precision FPU (`fpv5-sp-d16`), ARMv8-M DSP extension. *Confirmed* from a Zephyr build: `-mcpu=cortex-m33 -mfpu=fpv5-sp-d16 -mfloat-abi=hard`, `CONFIG_ARMV8_M_DSP=y`, `CONFIG_FPU=y`. | Zephyr (ListenAI's fork, 3.4), GCC. |
| Cadence HiFi4 DSP ("CP") | Xtensa with FLIX bundles and HiFi4 SIMD. Runs XOS. Core name `venus_hifi4`. | A firmware image loaded by the AP; see [dsp-images.md](dsp-images.md), [wasm-on-hifi4.md](wasm-on-hifi4.md), [hifi4-toolchain.md](hifi4-toolchain.md). |
| NPU | 128 GOPS (CSK6 family figure from a community article, no datasheet seen). Int8. | From the HiFi4, through Thinker / Luna; see [thinker-npu.md](thinker-npu.md). |

## DSP on the M33

- No NEON (Cortex-A only) and no Helium/MVE (Cortex-M55/M85 only). The SIMD
  available is the DSP extension: two 16-bit lanes per register,
  `SMLAD`/`SMLALD` dual multiply-accumulate, saturating arithmetic.
- **CMSIS-DSP** ships with Zephyr 3.4 in `modules/hal/cmsis/CMSIS/DSP`.
  Enable with:

  ```
  CONFIG_CMSIS_DSP=y
  CONFIG_CMSIS_DSP_FILTERING=y
  ```

  (`zephyr/modules/Kconfig.cmsis`, `Kconfig.cmsis_dsp`). It selects
  `REQUIRES_FULL_LIBC`. Useful functions: `arm_fir_decimate_q15`,
  `arm_fir_interpolate_q15`, `arm_fir_q15`, `arm_biquad_cascade_df2T_f32`.
- In Zephyr 3.4 the CMSIS core header is
  `<zephyr/arch/arm/aarch32/cortex_m/cmsis.h>` (not `cmsis_core.h`). The DWT
  cycle counter works for benchmarking (`CoreDebug->DEMCR`, `DWT->CYCCNT`).

## Memory seen from both cores

| Region | AP view | CP view | Notes |
|---|---|---|---|
| PSRAM | 0x30000000 | 0x60000000 | The DSP image is copied here by the AP before the CP is released. |
| Shared SRAM | 0x200C0000 (64 KB) | 0x200C0000 | IPC: LSF stream FIFOs, properties, allocator pool. The AP zeroes it before releasing the CP. |

Boot sequence and image layout: see
[c62-analysis/dsp-firmware-report.md](https://github.com/visionik/c62-analysis/blob/main/dsp-firmware-report.md).

## NPU control from the AP

The CSK6 HAL (`csk6_cm33/include/SysManager.h`) has only reset control for the
NPU; it is not otherwise driven from the M33:

```c
#define __HAL_SYS_NPU_RESET_KEY   0xCAFE000C
__HAL_SYS_NPU_IS_RESET()   /* IP_SYSCTRL->REG_SW_RESET_NPU */
__HAL_SYS_NPU_RESET()
__HAL_SYS_NPU_RELEASE()
```

`power_control.c` enables and releases the CP and NPU clocks together.
