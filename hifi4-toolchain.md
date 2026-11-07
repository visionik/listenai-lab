# HiFi4 toolchain

Native HiFi4 code (with SIMD and FLIX) needs Cadence XtensaTools and the core
configuration for this chip.

ListenAI's public [docker-env-hifi4](https://github.com/LISTENAI/docker-env-hifi4)
Dockerfile installs both from ListenAI's CDN:

```
https://cdn.iflyos.cn/public/lisa-binary/xt-venus/XtensaTools_RI_2021_7_linux.tar.zst   (567,811,887 bytes)
https://cdn.iflyos.cn/public/lisa-binary/xt-venus/venus_hifi4_linux-2021.7.tar.zst      (21,564,411 bytes)
```

Both answered HTTP 200 on 2026-11-07 (last modified 2022-11-14). The
Dockerfile patches `venus_hifi4/config/venus_hifi4-params` paths and sets:

```
PATH=/opt/hifi4/RI-2021.7-linux/XtensaTools/bin:$PATH
XTENSA_CORE=venus_hifi4
XTENSA_SYSTEM=/opt/hifi4/RI-2021.7-linux/venus_hifi4/config
```

*Unverified*: whether `xt-clang`/`xcc` run without a Cadence licence.

Even with a compiler, writing a native DSP image means your own audio,
IPC and service code: no source for the LSF DSP server is public. For most
uses the WASM route ([wasm-on-hifi4.md](wasm-on-hifi4.md)) is simpler.

The DSP images seen so far were built with Cadence RI-2020.x
([dsp-images.md](dsp-images.md)).
