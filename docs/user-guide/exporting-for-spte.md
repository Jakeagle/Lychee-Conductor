# Exporting for SPTE

Exporting a show produces a validated `.rshw` file that can be loaded directly into RR-Engine or SPTE.

The export is a two-stage process: generate a 4-channel WAV, then validate it and produce the `.rshw`.

---

## Stage 1 — Export 4ch WAV

1. Make sure your show is complete and saved.
2. Click **"Export 4ch WAV"** in the toolbar.
3. Pyodide loads (first run only — this may take a few seconds; a progress modal is shown).
4. The browser downloads a `.wav` file named after your show.

The file has four channels:

| Channel | Content                              |
| ------- | ------------------------------------ |
| 1       | Music left (or silence if no WAV)    |
| 2       | Music right (or silence if no WAV)   |
| 3       | TD BMC (Treble Data control signal)  |
| 4       | BD BMC (Bass Data control signal)    |

---

## Stage 2 — Validate and Export .rshw

1. Click **"4ch Tester"** in the toolbar to open the tester modal.
2. In the **Validate row** at the bottom, click **"Upload 4ch WAV"**.
3. Select the `.wav` file you just downloaded.
4. SViz (running in Pyodide) decodes the BMC signals and validates them. This takes a few seconds.
5. A badge appears:
   - **PASS** — the signal is hardware-compatible; the "Export .rshw" button is now enabled.
   - **FAIL** — there is a signal error; check the details shown in the modal.
6. Click **"Export .rshw"** to download the binary showtape file.

---

## Loading the .rshw in SPTE

1. Copy the `.rshw` file into your SPTE/RR-Engine showtapes folder.
2. Launch SPTE and open the showtape browser.
3. Your show will appear in the list — select it and press Play.

---

## Troubleshooting Export Issues

| Symptom                          | Likely cause                                    | Fix                                                       |
| -------------------------------- | ----------------------------------------------- | --------------------------------------------------------- |
| Pyodide takes very long to load  | First-run download of the WASM runtime          | Wait; subsequent exports are faster                       |
| Export button is greyed out      | Show is empty (no signal blocks)                | Add at least one signal block before exporting            |
| Validation badge shows FAIL      | BMC encoding error or bad signal timing         | File a bug; include the downloaded WAV as an attachment   |
| "Export .rshw" stays disabled    | Validation not yet run, or result is FAIL       | Upload a WAV and pass validation first                    |
| .rshw does not appear in SPTE    | File copied to wrong folder                     | Check SPTE documentation for the correct showtapes path   |
