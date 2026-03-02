# Troubleshooting

Common issues and how to fix them.

---

## Editor / General

### The page is blank or shows an error on load

- Make sure you are opening `index.html` directly in a browser (Chrome or Edge).
- If you see a CORS error in the console, serve the folder with a local HTTP server:
  ```
  python -m http.server 8080
  ```
  Then open `http://localhost:8080`.

### My saved show is gone

- Shows are stored in `localStorage`. Clearing browser data will delete them.
- Back up shows by exporting a 4ch WAV before clearing browser storage.

---

## Playback

### Play button is disabled (greyed out)

- The show must have at least one signal block.
- If a WAV is loaded and the AudioContext was suspended (browser autoplay policy), click anywhere on the page first, then press Play.

### Audio plays but the timeline cursor does not move

- The timeline clock uses `performance.now()`, not the AudioContext clock. If they drift, refresh the page and try again.

### Live tester shows no LED activity

- In the 4ch Tester modal (Live mode), the stage only shows events from `showData`. If there are no signal blocks in the current show, nothing lights up.
- Make sure you have a show loaded with signal blocks.

---

## Export

### Pyodide takes a very long time to load

- The first load downloads the WASM runtime (~10 MB). This is normal. Subsequent exports reuse the cached runtime.
- Do not close the tab or navigate away while the progress modal is shown.

### "Export 4ch WAV" produces a file with no music (silent channels 1/2)

- This is expected if no WAV file was loaded. Channels 3 and 4 will still contain valid BMC data.
- Load a WAV via the toolbar before exporting if you want music in the output.

### Validation shows FAIL

- The BMC signal did not pass hardware compatibility checks.
- Check the detail output in the tester modal for which check failed.
- If the show looks correct in the editor, file a bug and attach the downloaded WAV.

### "Export .rshw" button is still disabled after upload

- The validation must return PASS. A FAIL result keeps the button locked.
- Re-export the 4ch WAV and try validation again.

---

## Browser Compatibility

| Browser         | Status                                                    |
| --------------- | --------------------------------------------------------- |
| Chrome 110+     | Fully supported                                           |
| Edge 110+       | Fully supported                                           |
| Firefox         | Mostly working; Pyodide may be slower                     |
| Safari          | AudioContext autoplay restrictions may cause issues       |
| Mobile browsers | Not supported (layout not optimised for touch)            |
