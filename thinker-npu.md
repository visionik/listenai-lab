# Thinker and the NPU

[Thinker](https://github.com/LISTENAI/thinker) is ListenAI's int8 inference
engine (Apache-2.0; v3.0.12, 2026-06-29). It pairs with
[linger](https://github.com/LISTENAI/linger), a PyTorch quantisation-aware
training tool. Platforms: `venus` (CSK6), `arcs`, `venusA`.

## Where it runs on CSK6: the HiFi4

*Confirmed*: the `venus` chip libraries in the Thinker repo
(`executor/libs/venus/chip/`: `libthinker.a`, `libluna.a`,
`libnlangdriver.a`, `libhifi.a`, `libnnblas.a`, `libopi_cpy.a`) are Xtensa
ELF objects (e_machine 94). The venus backend uses Cadence NatureDSP
(`ops/venus/hifi/NatureDSP_*.h`) and Luna kernels (`ops/venus/luna/`), and
launches work on the NPU through "nlang" (`nlangInit`,
`nlangMallocSharemem`, `nlang_launch_async_start/wait`; the string "nlang
launch: time out while waiting for interrupt"). So: Thinker runs on the
HiFi4, and the NPU is a coprocessor of the HiFi4. The M33 reaches it only
through the LSF Thinker service and a WASM app
([lsf-services.md](lsf-services.md), [wasm-on-hifi4.md](wasm-on-hifi4.md)).

## Operators on venus

From `docs/support_quant_ops.md` (summarised):

| Kind | Ops and limits |
|---|---|
| Recurrent | `GRUInt` (batch 1; tensors in shared memory), `LSTMInt` (weight matrices in chunks of at most 32 KiB) |
| Convolution | `Conv1dInt` (4/8-bit weights, stride 1/2/4, kernel <= 5, pad <= 4), `Conv2dInt` (8-bit weights, int8 out), `ConvTranspose2dInt` |
| Linear | `LinearInt` (`transB=1`, rank <= 3), `BmmInt` |
| Activations | ReLU, PReLU, `iqSigmoid`, `iqTanh`, `SoftmaxInt`, `LogSoftmaxInt` |
| Other | BatchNorm, LayerNorm, pooling, elementwise `iq*` ops, reshape/concat/slice/etc. |

Int8 activations; int4/8/16 weights; no float compute except Quant/Dequant
at the model edges.

## Workflow

1. Train with linger (quantisation-aware), export ONNX.
2. `tpacker -g model_quant_venus.onnx -o model.bin` (graph optimisation,
   memory planning, serialisation).
3. On the DSP: `tInitialize`, `tModelInit(buf, size, dev_id)`,
   `tCreateExecutor`, `tSetInput`, `tForward`, `tGetOutput`, `tReleaseExecutor`,
   `tModelFini`, `tUninitialize` (`docs/thinker_api.md`).

Worked example with venus targets: ListenAI's
[Snore-Detection_lnn](https://github.com/LISTENAI/Snore-Detection_lnn).

## Fit for audio models (our assessment)

| Model type | Fit |
|---|---|
| RNNoise-class denoiser (GRU + dense, frame-based) | Good: small, all ops supported. |
| Frame-based neural vocoder (FARGAN-style) | Plausible; int8 quality is the risk. |
| Per-sample autoregressive vocoder (LPCNet) | Poor: per-launch overhead. |

Per-call latency through the LSF service is not yet measured.
