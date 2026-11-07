# DSP firmware families

## LSF "nano" server (`cp_6012_nano`)

Shipped by ListenAI as `resources/dsp_firmware.bin` in `lsf-zephyr-sdk`;
versions v1.0.1 (2023-08) to v1.2.0-beta.1 (2024-01), and the image at
commit 2ac605527dcd (2026-09-15, sha256 `9579b617f048c646465118f6d0b9e1b765b4d7d811ff9c8649c5754fe45a8903`) examined here.

| Item | Contents |
|---|---|
| RTOS | XOS |
| IPC | urpc (`rpc_sync_serv`, `rpc_async_serv`), LSF properties and binders, ICStream FIFOs in shared SRAM |
| Audio | AudioFlinger: MixerThread, DirectOutputThread, RecordThread, AudioMixer with a linear resampler, AudioPolicyService, StreamManager. No audio algorithms in the stream path. |
| Services | ServiceManager, AudioFlinger, ThinkerService, WasmService (WAMR `wamr_hifi4`, bytecode and AoT, XIP), VadService |
| NN | Thinker (`c_api/thinker_api.c`) and Luna (`LUNA_VER_*`) |

Source path strings point to `.../csk6/cp_6012_nano/services/...` and
`.../modules/wamr_hifi4/wamr/core/iwasm/...`.

## Product firmware (the Retevis C62)

The C62's stock DSP firmware is a different, product-specific build
("venus cp application", product TALKIE 1.2.0.1016) with iFlytek/ListenAI CAE
noise suppression, AGC and VAD, an `anker_unet` neural denoiser on Thinker,
EQ, pitch shift, resampler and white-noise detection, controlled over a
private urpc command set. Details:
[c62-analysis/dsp-firmware-report.md](https://github.com/visionik/c62-analysis/blob/main/dsp-firmware-report.md).
Not redistributable, and its control protocol is product-specific.

## What this means

- For your own processing, use the nano server's WASM and Thinker services
  rather than a product image.
- An LSF server build that includes CAE-style NS/AGC in the audio path would
  be ideal; none is public. Ask ListenAI.
