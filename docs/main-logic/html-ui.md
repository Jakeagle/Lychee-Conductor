# HTML UI — index.html

As of v3, the entire application lives in a single HTML file:

| File         | Purpose                                                                                 |
| ------------ | --------------------------------------------------------------------------------------- |
| `index.html` | All-in-one show editor, live tester, and export tool — the only page in the application |

> **`editor.html` no longer exists.** It has been retired to `index.legacy.html` for reference only.
> The old `index.html` (simulator shell) is also gone — the editor replaced it as the main entry point.

---

## Overall Page Structure

```
┌────────────────────────────────────────────────────────────────────┐
│  #intro-overlay (video, dismissed on click / any key)             │
├────────────────────────────────────────────────────────────────────┤
│  #toolbar                                                          │
│    [New Show] [Save] [Saved Shows] [Load WAV]                      │
│    Show title + band badge (ROCK-AFIRE)                            │
│    [▶ Play] [⏸ Pause] [⏹ Stop] [📌 Snap] [📌 State] [time]      │
│    [− Zoom Label +]                                                │
│    [🎭 4ch Tester]  [🎧 Export 4ch WAV]                           │
├────────────────────────────────────────────────────────────────────┤
│  #empty-state  (shown when no show is loaded)                      │
├────────────────────────────────────────────────────────────────────┤
│  #editor-body  (shown when a show is loaded)                       │
│    #label-panel   (character / movement row labels, fixed left)    │
│    #timeline-wrap (ruler canvas + scrollable tracks)               │
│       #ruler-canvas                                                │
│       #tracks-container                                            │
│          #playhead  #drag-preview  #snap-line  #select-rect        │
│          .char-group + .movement-row rows                          │
│          .state-block elements                                     │
├────────────────────────────────────────────────────────────────────┤
│  #help-bar  (keyboard shortcut hints)                              │
├────────────────────────────────────────────────────────────────────┤
│  #status-bar                                                       │
└────────────────────────────────────────────────────────────────────┘
```

---

## Toolbar Groups

### Left — file operations

| Control               | Action                                                |
| --------------------- | ----------------------------------------------------- |
| **✚ New Show**        | Opens `#new-show-overlay`; creates a blank `showData` |
| **💾 Save**           | Saves `showData` to `localStorage` (also Ctrl+S)      |
| **📁 Saved Shows**    | Opens `#saved-shows-overlay`                          |
| **🎵 Load WAV** label | File picker; decodes WAV into `audioBuffer`           |

### Centre — title area

`#show-title` and `#band-badge`. Only Rock-Afire Explosion is supported in v3; Munch's Make Believe Band has been deprecated.

### Transport

| Control           | Action                                                         |
| ----------------- | -------------------------------------------------------------- |
| ▶ Play            | Start timeline preview via `requestAnimationFrame`             |
| ⏸ Pause           | Pause at current position                                      |
| ⏹ Stop            | Stop and rewind to frame 0                                     |
| 📌 Snap (toggle)  | Snap new blocks to the BPM beat grid                           |
| 📌 State (toggle) | State-block mode — drag a row after a block to add hold events |
| `#time-display`   | Shows `m:ss.sss / F<frame>`                                    |

### Zoom

`−` / `+` adjust pixels-per-frame. Current zoom shown in `#zoom-label`.

### Export

| Button                | Action                                                                            |
| --------------------- | --------------------------------------------------------------------------------- |
| **🎭 4ch Tester**     | Opens `#tester-overlay`                                                           |
| **🎧 Export 4ch WAV** | Runs Python SGM → downloads `<title>_broadcast.wav`; disabled until WAV is loaded |

---

## Piano Roll Editor

### Label Panel (`#label-panel`)

Fixed-width (200 px) left column. Scrolls vertically in sync with `#timeline-wrap`. One header per character group, one label per movement row.

### Timeline (`#timeline-wrap`)

Scrollable both horizontally and vertically. Contains:

- **`#ruler-canvas`** — time ruler; click anywhere to seek the playhead.
- **`#tracks-container`** — all rows and blocks:
  - **`#playhead`** — red vertical line tracking current time.
  - **`.state-block`** — coloured ON/OFF block; drag to create, resize by edges, Delete to remove.
  - **`#drag-preview`** — ghost rectangle while drawing a new block.
  - **`#snap-line`** — cyan vertical indicator when snapping is active.
  - **`#select-rect`** — marquee drag-selection rectangle.

### Keyboard shortcuts

| Key              | Action                               |
| ---------------- | ------------------------------------ |
| Space            | Play / Pause                         |
| S                | Toggle snap                          |
| T                | Toggle state-block mode              |
| Delete           | Delete selected blocks               |
| Ctrl+C / V / D   | Copy / Paste at playhead / Duplicate |
| Ctrl+Z / Y       | Undo / Redo                          |
| Ctrl+Drag        | Marquee select                       |
| Ctrl+Click block | Add to selection                     |
| Shift+Drag       | Bypass snap                          |

---

## Modals

### New Show (`#new-show-overlay`)

Centred modal with title field, duration (minutes + seconds), and Create / Cancel buttons. On Create: blank `showData` is built from `CHARACTER_MOVEMENTS` for all nine RAE characters.

### Saved Shows (`#saved-shows-overlay`)

Lists shows in `localStorage`. Each entry has Load and Delete buttons. Shows are serialised `showData` JSON blobs.

> **There is no `.cybershow.json` file export.** Shows persist only in `localStorage`. Always export a 4ch WAV or `.rshw` to preserve work externally.

---

## 4ch Tester Modal (`#tester-overlay`)

The tester serves two related but distinct purposes.

### Live preview mode

When the modal opens without a 4ch WAV loaded it runs in **live mode**:

- `_tBuildLiveStage()` reads `showData` and builds one `.tester-character` card per RAE character.
- Each card has a row per movement (`CHARACTER_MOVEMENTS[charName].movements`), each with a small LED dot.
- Pressing Play drives a `requestAnimationFrame` tick loop clocked by **`performance.now()`** (not `AudioContext.currentTime`, which browser auto-suspend can freeze).
- Events from `_tLiveEvents` (sorted `{timeSec, charName, movKey, on}` entries from `showData`) advance via `_tLiveEventIdx`; matching LEDs toggle `.active`.
- `audioBuffer` (song loaded via Load WAV) plays simultaneously through Web Audio. Play also works without a song — visualization runs in silence.
- `_tCtx.resume()` is called before audio start to unblock a browser-suspended AudioContext.

### 4ch WAV validation and `.rshw` export

The **Validate for export** row at the bottom:

| Control               | Purpose                                                      |
| --------------------- | ------------------------------------------------------------ |
| **📂 Upload 4ch WAV** | Loads exported `_broadcast.wav`; switches tester to 4ch mode |
| Validation badge      | **NOT VALIDATED** → **PASS** or **FAIL** after SViz runs     |
| **📼 Export .rshw**   | Only enabled when `_tValidated === true` (badge = PASS)      |

In 4ch mode, the audio graph plays `_tBuf` (the uploaded WAV) and the LED visualization is driven from the decoded `arena._4chTimeline` rather than `_tLiveEvents`.

### Transport controls

| Control             | Description                                                             |
| ------------------- | ----------------------------------------------------------------------- |
| Play / Pause / Stop | Standard; Stop rewinds to t=0 and calls `_tLiveSeekTo(0)`               |
| `#tester-progress`  | Scrub bar; `input` event calls `_tLiveSeekTo(newTime)` then `_tStart()` |
| `#tester-time`      | `mm:ss / mm:ss`                                                         |
| 🎵 Music slider     | `_tMusicVol`; GainNode on music channels                                |
| 📡 Signals slider   | `_tSigVol`; GainNode on signal channels (audible in 4ch mode only)      |

### Key tester state variables

| Variable         | Meaning                                                          |
| ---------------- | ---------------------------------------------------------------- | --- | ------------------------------------------------ |
| `_tBuf`          | `AudioBuffer` of 4ch WAV. `null` = live mode.                    |
| `_tRaw`          | Raw `Uint8Array` bytes of 4ch WAV used for `.rshw` assembly.     |
| `_tPlayPerf`     | `performance.now()/1000` at play-start minus offset (viz clock). |
| `_tCurTime`      | Current playhead in seconds.                                     |
| `_tLiveEvents`   | Sorted `{timeSec, charName, movKey, on}` from `showData`.        |
| `_tLiveEventIdx` | Pointer into `_tLiveEvents`; advances during tick.               |
| `_tLiveState`    | `"charName                                                       |     | movKey"`→`bool`snapshot; used by`\_tLiveSeekTo`. |
| `_tLiveDur`      | Show length in seconds (`frameToMs(totalFrames)/1000`).          |
| `_tValidated`    | `true` after SViz validation returns PASS.                       |

---

## Python Progress Modal (`#ed-py-modal`)

A full-screen spinner overlay while Pyodide runs (WAV export or validation). Controlled by the `_edModal` helper object inside the SCME IIFE. Hidden via the `hidden` HTML attribute.

---

## Styling

All CSS is inline in `index.html` — no external stylesheet. Key CSS custom properties:

| Property    | Value / Role                |
| ----------- | --------------------------- |
| `--bg`      | `#0d0d1a` — page background |
| `--panel`   | Panel background            |
| `--toolbar` | Toolbar background          |
| `--border`  | Border colour               |
| `--text`    | Primary text                |
| `--muted`   | Dimmed / secondary text     |
| `--ruler-h` | `44px` ruler height         |
| `--label-w` | `200px` label panel width   |
| `--row-h`   | `22px` movement row height  |
