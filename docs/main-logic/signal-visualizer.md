# signal-visualizer.js — Legacy SViz Bridge (v2)

> **This module is not loaded by the v3 application.** In v3, WAV validation is integrated directly into the **4ch Tester modal** inside `index.html`. `signal-visualizer.js` remains in the repo as a reference for the v2 drop-zone approach.

---

## v3 Approach: Validation in the Tester Modal

In v3, signal validation works as follows:

1. The user opens the **4ch Tester** modal (`#tester-overlay`).
2. In the **Validate for export** row, they upload the downloaded `_broadcast.wav`.
3. The SCME IIFE decodes the WAV, extracts TD and BD channels as `Float32Array`s, and converts them to `Int16Array`.
4. Pyodide loads (or reuses `window._svizPyodide`) and runs `SCME/SViz/visualizer_bridge.py`.
5. `verify_and_decode(td_samples, bd_samples, sample_rate)` returns a validation result.
6. The `#tester-val-badge` updates to **PASS** (green) or **FAIL** (red).
7. If PASS, `_tValidated = true` and the **Export .rshw** button is enabled.

---

## Python Function Called (unchanged from v2)

```python
result = verify_and_decode(
    td_samples,     # list[int]  — Int16 PCM, TD channel
    bd_samples,     # list[int]  — Int16 PCM, BD channel
    sample_rate,    # int        — e.g. 44100
)
# Returns a dict → Pyodide converts to JS object
```

### Return Object

```js
{
  ok: true | false,
  td: {
    locked: true,
    frames: [...],           // decoded frame data
    error_rate: 0.001,
    blank_integrity: 0.998,
  },
  bd: {
    locked: true,
    frames: [...],
    error_rate: 0.000,
    blank_integrity: 1.000,
  },
  summary: "TD: PASS  BD: PASS  — 2847 frames decoded"
}
```

---

## Validation Thresholds

The Python SViz code applies the same thresholds the real hardware uses:

| Metric              | Required | Meaning                                                     |
| ------------------- | -------- | ----------------------------------------------------------- |
| Sync lock           | Yes      | Decoder must establish frame lock within first 3 sync bytes |
| Blank-bit integrity | ≥ 98%    | Hardware-reserved bits must be `0` in 98%+ of frames        |
| Bit error rate      | ≤ 2%     | Max allowed fraction of bits outside PLL tolerance window   |

A WAV that fails these checks **will be rejected by physical Cyberstar hardware**.

---

## v2 Architecture (reference)

In v2, `signal-visualizer.js` provided a drop-zone on the main page. After dropping a 4ch WAV:

1. The Web Audio API decoded the WAV into an `AudioBuffer`.
2. Channels 2 and 3 (TD and BD) were extracted as `Int16Array`.
3. They were passed to Pyodide, which ran `verify_and_decode()`.
4. The JS side rendered the decoded data as an HTML5 Canvas signal chart (one column per frame, one row per channel).

This approach was replaced in v3 by the tester modal, which combines live preview, 4ch WAV upload, validation, and `.rshw` export in one place.

---

## Python Function Called

```python
result = verify_and_decode(
    td_samples,     # list[int]  — Int16 PCM, TD channel
    bd_samples,     # list[int]  — Int16 PCM, BD channel
    sample_rate,    # int        — e.g. 44100
)
# Returns a dict (Python) → Pyodide converts to JS object
```

### Return Object

```js
{
  ok: true | false,
  td: {
    locked: true,
    frames: [...],           // decoded frame data
    error_rate: 0.001,
    blank_integrity: 0.998,
  },
  bd: {
    locked: true,
    frames: [...],
    error_rate: 0.000,
    blank_integrity: 1.000,
  },
  summary: "TD: PASS  BD: PASS  — 2847 frames decoded"
}
```

---

## Validation Thresholds

The Python SViz code applies the same thresholds the real hardware uses:

| Metric              | Required | Meaning                                                     |
| ------------------- | -------- | ----------------------------------------------------------- |
| Sync lock           | Yes      | Decoder must establish frame lock within first 3 sync bytes |
| Blank-bit integrity | ≥ 98%    | Hardware-reserved bits must be `0` in 98%+ of frames        |
| Bit error rate      | ≤ 2%     | Max allowed fraction of bits outside PLL tolerance window   |

A WAV that fails these checks **will be rejected by physical Cyberstar hardware**. The old Web Audio API export pipeline failed these tests (see [research/sgm-validation-history.md](../research/sgm-validation-history.md)).

---

## Chart Rendering

The browser-side chart uses the HTML5 Canvas API. Each column of pixels is one frame; each row is one channel. A lit pixel means the actuator was on in that frame. This gives an instant visual impression of show density and pattern.

The chart is scrollable horizontally and zoomable. Hovering a pixel shows the character name, movement name, and frame number in a tooltip.
