# Export Pipeline

This document describes how the Cyberstar Simulator produces `.rshw` files from a show authored in the piano roll editor.

---

## Pipeline Overview

```
showData (piano roll)
    |
    v
Stage 1: Export 4ch WAV
    Pyodide + SGM (export_bridge.py)
    BMC encode -> 4-channel WAV
    |
    v
Stage 2: 4ch Tester validation
    Upload WAV -> Pyodide + SViz (visualizer_bridge.py)
    PASS / FAIL badge
    |
    | (PASS only)
    v
Stage 3: .rshw export
    Pyodide + SGM (rshw_builder.py)
    .rshw binary (NRBF format)
    |
    v
RR-Engine / SPTE
```

---

## Stage 1 — 4ch WAV Generation

**Triggered by:** "Export 4ch WAV" toolbar button.

### Inputs
- `showData` — the current show's character movement timeline.
- `audioBuffer` (optional) — decoded song WAV; null if no song was loaded.

### Process
1. `export_bridge.generate_4ch_wav(showData, audioSamples)` is called via Pyodide.
2. The show's `{frame, on}` events are converted to a 96-bit-wide BMC packet stream.
3. Audio samples (if present) are interleaved into channels 1 and 2.
4. BMC-encoded channel 3 (TD) and channel 4 (BD) are filled with data.
5. A standard 4-channel 44100 Hz 16-bit PCM WAV is assembled and returned.

### Output
A `.wav` file downloaded to the browser's Downloads folder.

---

## Stage 2 — Signal Validation

**Triggered by:** "Upload 4ch WAV" in the 4ch Tester modal.

### Inputs
- The 4-channel WAV from Stage 1.

### Process
1. SViz (`visualizer_bridge.py`) decodes channels 3 and 4 from the WAV.
2. BMC frames are extracted and checked against hardware timing requirements.
3. Validation checks include:
   - Clock period stability (within tolerance of 1/FPS/96 samples)
   - No missing or extra bits per frame
   - No illegal state transitions
4. A PASS or FAIL result is returned with per-check detail.

### Output
- A PASS/FAIL badge displayed in the tester modal.
- If PASS: the "Export .rshw" button is enabled (`_tValidated = true`).

---

## Stage 3 — .rshw Export

**Triggered by:** "Export .rshw" (only enabled after PASS validation).

### Inputs
- `showData` — same source of truth used in Stage 1.

### Process
1. `rshw_builder.build_rshw(showData)` is called via Pyodide.
2. The NRBF binary format is assembled:
   - Header: magic bytes, version, character manifest
   - Track data: per-character frame arrays as BitArrays
   - Footer: length checksum
3. The binary blob is packaged as a `.rshw` file.

### Output
A `.rshw` file downloaded to the browser's Downloads folder, loadable by RR-Engine/SPTE.

---

## What Was Removed in v3

| v2 Feature        | v3 Status                                                   |
| ----------------- | ----------------------------------------------------------- |
| `.cso` export   | Removed. No CSO output in v3.                               |
| `cso-exporter.js` | Legacy module; not loaded by `index.html`.              |
| SAM integration   | Removed from UI. SAM still exists in `SCME/SAM/` standalone.|
| JSON export       | Removed.                                                    |
| MMBB support      | Removed. Only Rock-Afire Explosion characters.              |
