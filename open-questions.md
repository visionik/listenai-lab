# Open questions and next experiments

## Open
1. The exact `thinker/thinker.h` and the export names and signatures the
   Thinker service calls in the WASM app (in the login-gated wasm SDK).
2. Whether WASM apps get any native beyond Thinker and libc-builtin, in
   particular access to AudioFlinger streams.
3. Round-trip latency of `lsf_thinker_set_input` + `lsf_thinker_get_output`.
4. WAMR memory limits (app heap, stack) in the nano server.
5. Whether XtensaTools from ListenAI's CDN run without a Cadence licence.
6. Whether any AudioSystem keys reach the DSP-side ALC/EQ/echo functions.
7. Official NPU figures for the CSK6011B (the 128 GOPS figure is a family
   number from a community article).

## Next experiments (cheapest first)
1. Get a cloud.listenai.com account; fetch the wasm SDK and
   `thinker-csk-project`; answer 1, 2 and 4 from the headers.
2. Build a pass-through WASM app (copy input to output), AoT-compile with
   wamrc-xtensa, load it with `lsf_wasm_load_app`, time a set/get round trip
   with the DWT cycle counter on the M33 (answers 3).
3. Run a tiny GRU model through Thinker and measure frame time.
4. Try the XtensaTools container (answers 5).
