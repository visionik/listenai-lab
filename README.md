# listenai-lab

Notes on programming the **ListenAI CSK6** family (tested on a CSK6011B),
beyond its Arm core: the HiFi4 DSP, the NPU, and the LSF services that let
the Arm side use them. Collected while working with the chip in the Retevis
C62 handheld radio (see [c62-analysis](https://github.com/visionik/c62-analysis)).

Everything here comes from ListenAI's public SDK sources and repositories,
community articles, build experiments, and strings in DSP images. **No vendor
binaries are included**: no DSP images, libraries, toolchains or models. Links
point to where they are published.

Confidence is marked where it matters: *confirmed* (read in source or
tested), *likely*, or *unverified*.

## Contents

| File | What it covers |
|---|---|
| [csk6-chip.md](csk6-chip.md) | The three engines (Cortex-M33, HiFi4, NPU), what the M33 can do for DSP (CMSIS-DSP, no NEON), shared memory, NPU reset control. |
| [lsf-services.md](lsf-services.md) | The LSF service framework in the "nano" DSP server: service IDs, property tables and the Arm-side client calls for the audio, Thinker, WASM and VAD services. |
| [thinker-npu.md](thinker-npu.md) | Thinker, ListenAI's NN inference engine: where it runs, supported operators on CSK6 ("venus"), the linger/tpacker workflow, C API, fit for audio models. |
| [wasm-on-hifi4.md](wasm-on-hifi4.md) | Running your own code on the HiFi4 as a WebAssembly app: the app model, the public Xtensa AoT compiler, build and flash layout, logs. |
| [hifi4-toolchain.md](hifi4-toolchain.md) | The Cadence XtensaTools build of the CSK6 HiFi4 core that ListenAI publishes, and alternatives. |
| [dsp-images.md](dsp-images.md) | The two DSP firmware families (stock product firmware and the LSF "nano" server) and what each offers. |
| [open-questions.md](open-questions.md) | What is still unknown, and the cheapest next experiments. |
| [sources.md](sources.md) | Every source used, with links. |

## Short version

- The Arm core is a Cortex-M33 with the DSP extension and a single-precision
  FPU. No NEON, no Helium. CMSIS-DSP is the library to use.
- The NPU is driven from the HiFi4, not from the Arm core: ListenAI's Thinker
  runtime for CSK6 is built for Xtensa and launches NPU work through shared
  memory.
- The LSF "nano" DSP server exposes a Thinker service and a WASM service to
  the Arm side. Your own WASM app runs on the HiFi4, exports `set_input` and
  `get_output`, and calls the Thinker API, so the Arm side can send a buffer,
  have a model run on the NPU, and read the result.
- WASM apps can be compiled ahead of time to Xtensa with public tools
  (ListenAI's `wamrc-xtensa` release and wasi-sdk 17), with no Cadence
  licence. That code uses the base Xtensa ISA, not HiFi4 SIMD.
- ListenAI's WASM SDK (headers, build scripts) and examples sit behind a
  cloud.listenai.com login.
