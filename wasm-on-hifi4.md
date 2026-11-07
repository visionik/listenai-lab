# Your own code on the HiFi4: WASM apps

The LSF "nano" DSP server embeds WAMR (WebAssembly Micro Runtime, built as
`wamr_hifi4`). It loads interpreted bytecode or ahead-of-time (AoT) compiled
modules, optionally in place from flash (XIP), and calls `main()` (strings in
the image).

## App model

From ListenAI's examples and community write-ups (*likely*: the SDK itself
is behind a login, see below):

1. The app runs on the HiFi4 and exports `set_input` and `get_output`.
2. On the M33, `lsf_thinker_set_input()` / `lsf_thinker_get_output()` call
   those exports through the Thinker service ([lsf-services.md](lsf-services.md)).
3. The app includes `"thinker/thinker.h"` (the same functions as native
   Thinker, [thinker-npu.md](thinker-npu.md)) and runs the model on the NPU.
   Newer SDK versions take the model from its own flash partition (address +
   size) instead of embedding it in the app.

So the Thinker service is a synchronous call channel from the M33 to your
own DSP code, carrying an input buffer and returning an output buffer. The
app can do any processing, with or without the NPU.

Beyond Thinker, the app gets WAMR's libc-builtin natives (printf, malloc and
friends; export `malloc`/`free` from the app). **No native API for direct
audio-stream access has been found**; data reaches the app through the
set/get calls.

## Tools

| Piece | Where | Access |
|---|---|---|
| wasi-sdk 17.0 | [WebAssembly/wasi-sdk releases](https://github.com/WebAssembly/wasi-sdk/releases) | public |
| `wamrc` for Xtensa (AoT compiler) | [LISTENAI/wasm-micro-runtime](https://github.com/LISTENAI/wasm-micro-runtime), release `wamrc-20221213.2`: `wamrc-xtensa-darwin-x64.tar.xz`, `wamrc-xtensa-linux-x64.tar.xz` | public |
| WAMR XOS port | same repo, branch `xos` (`core/shared/platform/xos`: memory, threads, mutex, time, dcache, print) | public |
| ListenAI wasm SDK (Thinker headers, CP resources, build scripts) | `https://cloud.listenai.com/listenai/wasm/wasm-sdk.git` (clone `--recursive`), env `WASM_THINKER_SDK` | **login required** |
| Examples | `cloud.listenai.com/listenai/wasm/thinker-csk-project`, `cloud.listenai.com/listenai/samples/camera_image_detect` | **login required** |

The fork's `build-wamrc-xtensa` branch builds wamrc against Espressif's
LLVM (`xtensa_release_13.0.0`, experimental Xtensa target). That LLVM knows
the base Xtensa ISA only, not HiFi4 SIMD or FLIX, so AoT code runs as a
plain scalar Xtensa core: fine for glue and control, no faster than the M33
for DSP loops. The speed comes from calling Thinker and its native kernels.

## Build and flash (from the examples)

```sh
export WASI_TOOLCHAIN_PATH=/path/to/wasi-sdk-17.0
export WASM_THINKER_SDK=/path/to/wasm-sdk
cd app_wasm
lisa zep exec python $WASM_THINKER_SDK/cmake/sdk.py build -p .   # older SDK
# or: lisa zep exec python $WASM_THINKER_SDK/tools/build.py       # newer SDK
```

| Flash offset | Contents |
|---|---|
| 0x000000 | Zephyr app (`zephyr.bin`) |
| 0x100000 | DSP firmware (`cp.bin` / `dsp_firmware.bin`) |
| 0x200000 | WASM app (`.aot`) |
| 0x300000 | Thinker model (`.bin`) |

The examples flash with `cskburn -s <port> -C 6 <offset> <file> -b 748800`.
For the app to load with the SDK's client, the partition must hold the
`LSHD` blob (magic, size, module); check whether the SDK's build output
already includes that header.

## Logs

The DSP side logs on its own UART, on GPIOA15 (RX) and GPIOA18 (TX) on the
CSK6 dev kit.
