# LSF services (the "nano" DSP server)

ListenAI's LSF framework runs an Android-style service model across the two
cores: the DSP runs the servers, the M33 runs thin clients. Calls go through
urpc and shared memory. Everything below is from ListenAI's
`lsf-zephyr-sdk` (Apache-2.0); a mirror is at
[github.com/OpenRTX/lsf-zephyr-sdk](https://github.com/OpenRTX/lsf-zephyr-sdk).
*Confirmed* in source unless marked.

## Model

- A service is a group of **properties**. Each property has an ID, flags
  (`LSF_PROPERTY_FLAG_SYNC`, `_SET_ONLY`, `_GET_ONLY`) and a type
  (`LSF_PROPERTY_TYPE_PTR`, `_BUF`, `_UINT32`, ...).
- A client registers the table with
  `lsf_import_properties(service_id, defs, count)` and then uses
  `lsf_set(service_id, property_id, &variant)` and
  `lsf_get(service_id, property_id, &variant_ptr)`.
- Values are `LsfVariant`s: `type_uint` (`u.uiVal`), `type_membuf`
  (`u.memBuf.pData`, `.size`, copied), `type_pointer` (`u.pointer.pAddr`,
  `.size`, passed by address; the memory must be visible to the DSP, e.g.
  flash/XIP, PSRAM or shared SRAM).
- Services are enabled from the devicetree. Clients are defined with
  `LSF_SERVICE_DEFINE(name, init_fn)`.
- Binder IDs on the wire are `(group << 20) | (prop << 10) | iface`.

## Service IDs (`dts/lsf/lsf.dtsi`)

| Group | Node | Compatible |
|---|---|---|
| 0 | `lsf_audio` | `lsf,audio-service` (AudioFlinger: AudioTrack, AudioRecord, AudioPolicy, AudioSystem key=value parameters) |
| 1 | `lsf_thinker` | `lsf,thinker-service` |
| 2 | `lsf_wasm` | `lsf,wasm-service` |
| 3 | `lsf_vad` | `lsf,vad-service` |

Enable one with `&lsf_thinker { status = "okay"; };`.

## Thinker service (group 1), `src/services/service_thinker.c`

| ID | Property | Flags | Type | Client call |
|---|---|---|---|---|
| 0 | INPUT | SYNC, SET_ONLY | PTR | `int lsf_thinker_set_input(const void *ptr, uint32_t size)` |
| 1 | OUTPUT | SYNC, GET_ONLY | PTR | `int lsf_thinker_get_output(void **ptr, uint32_t *size)` |
| 2 | MODEL | SYNC, SET_ONLY | PTR | `int lsf_thinker_set_model(const void *ptr, uint32_t size)` |

- All synchronous. Typical use: set the model once (pointer into a flash
  partition), then per frame set input, get output.
- The branch `dnm/thinker-multi-output-sample` (2023-09-21) adds an output
  count property and returns all outputs.
- Input and output are handed to the **WASM app** running on the DSP (see
  [wasm-on-hifi4.md](wasm-on-hifi4.md)): the DSP image logs "WASM module not
  loaded" if a model is set with no app loaded (string in the image).
- SDK sample `samples/thinker_wasm`: ResNet18 CIFAR-100 classifier, model
  (2,932,400 bytes) in a `thinker_model` flash partition at 0x300000, WASM app
  partition at 0x200000, DSP firmware at 0x100000; inference every 200 ms,
  int8 outputs, argmax on the AP.

## WASM service (group 2), `src/services/service_wasm.c`

| ID | Property | Flags | Type |
|---|---|---|---|
| 0 | APP | SYNC, SET_ONLY | PTR |

- `int lsf_wasm_load_app(const void *ptr, uint32_t size)`. The blob format:

  | Offset | Field |
  |---|---|
  | 0 | `"LSHD"` magic |
  | 4 | u32 app size |
  | 8 | the module (`.aot` or bytecode) |

  The client checks the magic and size and passes `{blob + 8, app_size}` as
  a `type_pointer`.
- Load at boot: `CONFIG_LSF_WASM_SERVICE_LOAD_APP_ON_BOOT=y` and
  `chosen { lsf,dsp-wasm-app = &wasm_app; };` (a flash partition). The
  client then loads straight from XIP flash.

## VAD service (group 3), `src/services/service_vad.c`

| ID | Property | Type |
|---|---|---|
| 0 | CONFIG | BUF |
| 1 | ENABLE | UINT32 |

Config struct, filled from devicetree properties:

```c
struct lsf_vad_config {
	uint8_t channel;          /* input channel bitmap: 0x1 or 0x3 (default 0x3) */
	uint16_t amp_l;           /* lower amplitude threshold (default 1638) */
	uint16_t amp_h;           /* higher amplitude threshold (default 2949) */
	uint8_t minlen_trans_neg; /* default 3 */
	uint8_t minlen;           /* minimum voice length, in 10 ms (default 3) */
};
```

`int lsf_vad_enable(void)` sets ENABLE = 1. An amplitude-threshold detector
for wake-up (SDK sample `voice_wakeup`).

## Audio service (group 0)

Client: `AudioSystem`, `AudioTrack`, `AudioRecord` (Android-like). Known
`AudioSystem` key=value parameters: `samplingRate`, `channels`, `format`,
`frame_count`, `routing`, and the codec gains `ADC_PDM_GAIN_{A,D}_{LEFT,RIGHT}`,
`DAC_GAIN_{A,D}_*`. Streams are 16-bit; the SDK commit
2ac605527dcd (2026-09-15) adds run-time 16 kHz / 48 kHz selection and an
`audio_duplex_48k` sample.

The DSP side contains ALC, EQ and echo HAL functions (`adc_set_alc_params`,
`apc_eq_set_coef*`, `apc_echo_setup`, names from strings), but no
parameter keys reaching them were found (*unverified*).
