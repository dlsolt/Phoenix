# Experiment Log

Record of investigations into removing the `code/core_mods/` dependency
(patched `usb_audio.cpp`/`usb_audio.h`) from the FT8 USB-audio RX path.
Written so future attempts don't re-derive the same conclusion from scratch.

---

## Session 2026-07-04: stock-API RX replacement — concluded not viable

**Branch:** `ft8-stock-core` (rebased onto `ft8-on-new-version` at `75b7da5`)

**Prior history referenced but not found:** an earlier attempt was said to be
documented as "Sessions 45-47" in this file, reportedly showing a comb
artifact caused by a non-stock `tx_event` combined with both I2S and an
IntervalTimer triggering audio updates. No record of those sessions was
found anywhere — not in this file (didn't exist until now), not in git
history on any branch, not in either of the two backup zips
(`20260213 teensy.avr.teensy41.zip`, `src/20260415 PhoenixSketch_FT8.zip`).
If that write-up turns up later, reconcile it against the findings below —
but the analysis here was derived independently from the actual installed
core source, not from that prior account.

### Goal

Feed FT8 RX audio (192 kHz DSP audio, resampled to ~44.1 kHz) to
`AudioOutputUSB` using only public Teensy Audio Library APIs — an
`AudioPlayQueue` → `AudioConnection` → `AudioOutputUSB` graph, filled from
an `IntervalTimer` — so the patched `usb_audio.cpp`/`.h` in `core_mods/`
(which adds `usb_audio_push_block()`) could be dropped in favor of a fully
stock Teensyduino 1.62 core.

### What was tried

The existing WIP commit (rebased cleanly, no conflicts) implements:
- `AudioPlayQueue Q_usbOut_L, Q_usbOut_R;` and `AudioOutputUSB usbOut;` in
  `MainBoard_AudioIO.cpp`, connected via `AudioConnection` through a pair of
  `AudioAmplifier` gain stages.
- `Ft8UsbBridge_DrainToUSB()` rewritten to use `Q_usbOut_L.getBuffer()` /
  `playBuffer()` instead of `usb_audio_push_block()`, called from an
  `IntervalTimer` at 2902 µs (≈344.5 Hz — one native 128-sample USB audio
  block period at 44.1 kHz).

This alone was **not implemented as final** — investigation below found a
structural blocker before it was written to disk as a real answer.

### The blocker: who drives `AudioStream::update_all()`

Traced directly in the installed library/core source
(`C:\Users\daves\AppData\Local\Arduino15\packages\teensy\hardware\avr\1.62.0\`):

- `AudioInputI2SQuad`'s constructor self-invokes `begin()`
  (`libraries/Audio/input_i2s_quad.h:37`), which calls
  `update_responsibility = update_setup()`
  (`libraries/Audio/input_i2s_quad.cpp:122`).
- `AudioOutputI2SQuad` does the same in its own constructor
  (`output_i2s_quad.h:37`, `output_i2s_quad.cpp:126`).
- In `MainBoard_AudioIO.cpp`, `i2s_quadIn` (line 123) is declared before
  `i2s_quadOut` (line 142) — same translation unit, so construction order
  is guaranteed. `i2s_quadIn` wins `update_setup()` at global static-init
  time, before `InitializeAudio()` ever runs.
- `AudioInputUSB`/`AudioOutputUSB` never compete for this responsibility —
  in both the stock `usb_audio.cpp` and the patched `core_mods` version,
  the line `update_responsibility = update_setup();` is commented out.

Net effect: **`i2s_quadIn`'s I2S RX DMA ISR is the sole driver of
`update_all()`**, firing once per completed 128-sample block at whatever
`SR[SampleRate].rate` is configured. At 192 kHz that's ~1500 Hz. This can't
be changed — the main DSP RX/TX pipeline (`Q_in_L`/`Q_in_R` etc.) requires
servicing at the true I2S rate to avoid overrunning the hardware ADC
stream.

Consequence: calling `NVIC_SET_PENDING(IRQ_SOFTWARE)` from our own
`IntervalTimer` is a no-op. `update_all()` is already firing continuously
at ~1500 Hz regardless of what our timer does; it doesn't give `usbOut` a
correct ~344 Hz cadence, because `update_all()` has exactly one global rate
shared by every `AudioStream` object — there is no per-object rate.

### Why "empty queue = harmless" doesn't hold all the way through

`AudioPlayQueue::update()` (`libraries/Audio/play_queue.cpp:219`) is
well-behaved when empty:

```cpp
void AudioPlayQueue::update(void)
{
    ...
    t = tail;
    if (t != head) {          // nothing queued -> no-op, no transmit
        ...
        transmit(block);
        release(block);
    }
}
```

But `AudioOutputUSB::update()` — confirmed identical in the **pristine,
pre-T41 stock file** `usb_audio.original.cpp` (predates any patch) — is
not:

```cpp
void AudioOutputUSB::update(void)
{
    audio_block_t *left, *right;
    left = receiveWritable(0);
    right = receiveWritable(1);
    if (usb_audio_transmit_setting == 0) { ... return; }  // only "skip" path
    if (left == NULL) {
        left = allocate();
        if (left == NULL) { if (right) release(right); return; }
        memset(left->data, 0, sizeof(left->data));         // fabricates silence
    }
    if (right == NULL) { /* same for right */ }
    __disable_irq();
    if (left_1st == NULL) {
        left_1st = left; right_1st = right; offset_1st = 0;
    } else if (left_2nd == NULL) {
        left_2nd = left; right_2nd = right;
    } else {
        // buffer overrun - PC is consuming too slowly
        audio_block_t *discard1 = left_1st;
        left_1st = left_2nd; left_2nd = left;
        ...
        release(discard1); release(discard2);
    }
    __enable_irq();
}
```

There is no branch that skips queuing when input is `NULL`. Every call
(except when USB isn't yet configured, or the block pool is exhausted)
pushes *something* — real or fabricated silence — into the fixed 2-deep
`left_1st`/`left_2nd` buffer.

Since real FT8 RX data only arrives at ~344 Hz (via `playBuffer()`) but
`update()` fires at ~1500 Hz (≈4.35× more often), roughly 77% of ticks find
`Q_usbOut_L`/`R`'s output empty and inject a manufactured silent block into
that 2-slot FIFO — overrunning it and discarding/diluting real audio
essentially at random relative to when the actual USB isochronous transfer
drains a slot. That's the same signature as a comb/dropout artifact.

### Conclusion

This is a structural conflict in the SDR's audio graph — the shared,
I2S-driven `update_all()` rate (~1500 Hz, required by the main DSP path)
versus USB audio's fixed native block rate (~344 Hz) — **not an artifact of
the core patch**. A fully stock core does not fix it. There is no public
API that lets code feed `AudioOutputUSB` at its correct rate outside the
shared `update_all()` cycle; that's almost certainly why the original
working implementation bypasses the `AudioConnection`/`update()` graph
entirely and writes straight into the isochronous transmit buffer via
`usb_audio_push_block()`.

**Decision:** do not pursue a public-API-only `AudioOutputUSB` feed further
under the current audio graph architecture. `ft8-on-new-version`
(tag `ft8-working-v2`) remains the working reference build, with
`code/core_mods/` treated as a documented, intentional dependency rather
than a bug to route around.

### Possible future angles (not investigated, listed for the record)

- A different sink than `AudioOutputUSB` (e.g., a raw USB endpoint managed
  outside the Audio library entirely) — bigger rework, unexplored.
- Toggling `AudioConnection::connect()`/`disconnect()` around `usbOut`'s
  inputs from the `IntervalTimer` so it's only `active` (and thus only
  gets `update()` called) in the brief window right after `playBuffer()`.
  Uses only public APIs but is untested, and repeatedly connecting/
  disconnecting from ISR context at high rate has its own correctness
  risks (linked-list mutation races, timing fragility) that would need
  careful validation before trusting it on air.
- Revisit whether `i2s_quadIn` truly must retain `update_responsibility`,
  e.g. by decoupling the DSP block-consumption logic from `update_all()`
  timing entirely — likely a much larger architectural change.
