# Sample Rate & Frame Timing Bug Report

**Date:** March 2, 2026  
**Commits:** `b16dadd` (audio rate fix), `162d5b0` (rshw frame timing fix)  
**Affected files:** `index.html`, `SCME/SGM/rshw_builder.py`  
**Status:** Both resolved.

---

## Summary

Two independent timing bugs were found and fixed during export testing. Both shared a
common theme: integer-vs-float precision in hardware timing constants. Neither was caught
by validation because the signal tracks themselves were bit-perfect — the errors only
surfaced in the final assembled output.

---

## Bug 1 — Music plays slow in the 4ch WAV export

### Symptom

The exported 4-channel WAV played back at roughly 91.9% speed. Music was audibly flat
and dragged behind the animation. Signal tracks (channels 3–4) were unaffected.

### Root Cause

`new AudioContext()` without a `sampleRate` option inherits the OS audio device rate.
On Windows, this is almost universally **48,000 Hz**.

When `decodeAudioData` runs, the browser silently resamples the source file into the
context's rate. So `audioBuffer.sampleRate` was **48,000**, even for a source file
encoded at 44,100 Hz. The BMC encoder writes at 44,100 Hz. The assembled WAV header
declared 44,100 Hz. But the music samples inside it were spaced as if 48,000 Hz:

```
Header rate:  44,100 Hz
Music data:   48,000 samples/second worth of content
Playback:     reader pulls 44,100 samples/second → 44100/48000 ≈ 91.9% speed
```

### What Was Tried First (Wrong)

An `OfflineAudioContext` resampling step was added at export time (`a6511c1`). This
correctly detected the mismatch at `audioBuffer.sampleRate !== result.sample_rate`, but
the resample path went: **source file → 48,000 Hz (via AudioContext) → 44,100 Hz (via
OfflineAudioContext)** — a two-hop conversion that still produced slightly imprecise
results and added complexity.

### Fix

Create the `AudioContext` with an explicit rate at construction time (`b16dadd`):

```js
// Before (implicitly adopts OS rate — 48,000 Hz on Windows):
audioCtx = new AudioContext();

// After (always 44,100 Hz — matches BMC encoder):
audioCtx = new AudioContext({ sampleRate: 44100 });
```

`decodeAudioData` then resamples any source file directly to 44,100 Hz in one step.
`audioBuffer.sampleRate` is always 44,100. The export copies samples directly with no
further resampling needed. In-app playback is not affected — the browser's audio
output stage resamples from 44,100 Hz to the device rate (48,000 Hz) internally.

### Impact

- Any user on Windows (48,000 Hz default) exporting a 4ch WAV would have received a
  slow/flat music track.
- Signal channels 3–4 were unaffected — they come from Python's BMC renderer, which
  always works at 44,100 Hz.

---

## Bug 2 — Animations drift behind music when converting to .rshw

### Symptom

After converting the 4ch WAV to `.rshw`, animations that repeated every few seconds
appeared correct at the start but progressively lagged behind the music. By the 1-minute
mark, they were noticeably behind; by the 2-minute mark, the drift was unmistakable.
The source 4ch WAV played correctly.

### Root Cause

`_build_signal_data()` in `rshw_builder.py` computed BMC frame duration from the baud
rate constant:

```python
td_frame_s = td_frame_bits / baud_rate   # 94 / 4800 = 0.019583 s
bd_frame_s = bd_frame_bits / baud_rate   # 96 / 4800 = 0.020000 s
```

The baud rate of 4,800 implies `44100 / 4800 = 9.1875` samples per bit. But the BMC
encoder uses the **integer constant** `SAMPLES_PER_BIT = 9`. One full TD frame is
therefore `94 × 9 = 846` samples, not `94 × 9.1875 = 863.625` samples.

The actual frame periods are:

| Track | Encoder output           | Baud-rate formula      | Error      |
| ----- | ------------------------ | ---------------------- | ---------- |
| TD    | `846/44100 = 0.019183 s` | `94/4800 = 0.019583 s` | **+2.08%** |
| BD    | `864/44100 = 0.019591 s` | `96/4800 = 0.020000 s` | **+2.09%** |

`_build_signal_data` used the baud-rate formula to compute which decoded BMC frame index
to read for each 60-fps RSHW frame:

```python
td_idx = int(t / td_frame_s)   # t = rshw_frame_num / 60
```

Because `td_frame_s` was 2.08% _too large_, the index advanced _too slowly_. At
time `t`, the function read a BMC frame that was ~2.08% too early in the decoded frame
list — meaning it was reading _older_ state than the actual content at that timestamp.

This lag accumulated linearly:

- At 30 s: ~1.3 BMC frames behind
- At 60 s: ~2.6 BMC frames behind
- At 120 s: ~5.2 BMC frames behind

For animations that fire and release within a handful of BMC frames, this was sufficient
to delay entire activation events by one or more RSHW frames per minute — visible as
progressive fade behind the music.

### Why Signal Validation Didn't Catch This

The 4ch Tester validates the BMC _signal encoding_ (bit integrity, run-length
tolerances, blank bit preservation). It does not check that the RSHW frame-mapping index
is derived from the correct sample count. The WAV itself was perfect; the bug only
manifested during the BMC-frame → RSHW-frame conversion.

### Fix

Replace the baud-rate formula with an integer sample-count computation (`162d5b0`):

```python
# Before:
td_frame_s = td_frame_bits / baud_rate   # assumes 9.1875 samp/bit

# After:
td_frame_s = (td_frame_bits * samples_per_bit) / sample_rate  # 846/44100
bd_frame_s = (bd_frame_bits * samples_per_bit) / sample_rate  # 864/44100
```

`samples_per_bit = 9` is the integer the encoder actually uses; `sample_rate = 44100`
is read from the WAV header and passed through. This matches the actual frame boundaries
in the PCM data exactly.

### Lesson

The hardware baud rate of 4,800 bps is a _nominal_ rate. The encoder must work in
integer samples, so `SPB = floor(44100 / 4800) = 9` (not 9.1875). Any formula deriving
timing from `bits / baud_rate` will be wrong by `frac(SR/baud_rate)` fraction of a
sample per bit — 0.1875 samples/bit in this case. Over 94 bits that is 17.6 samples per
frame, or about 0.4 ms. Small per frame, but it integrates linearly across thousands of
frames.

**Rule:** Always compute frame durations from `frame_bits × SPB / SR`, never from
`frame_bits / baud_rate`.

---

## Related Constants (hardware-confirmed)

| Constant          | Value  | Source                       |
| ----------------- | ------ | ---------------------------- |
| `SAMPLE_RATE`     | 44,100 | KWS WAV header, constants.py |
| `SAMPLES_PER_BIT` | 9      | Integer, `floor(44100/4800)` |
| `TD_FRAME_BITS`   | 94     | RAE_Bit_Chart_2.pdf          |
| `BD_FRAME_BITS`   | 96     | RAE_Bit_Chart_2.pdf          |
| TD frame samples  | 846    | `94 × 9`, KWS-confirmed      |
| BD frame samples  | 864    | `96 × 9`, KWS-confirmed      |
| `RSHW_FPS`        | 60     | UI_ShowtapeManager.cs source |
