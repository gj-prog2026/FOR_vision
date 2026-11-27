# ssl-vision Testing — Progress Update

## Bottom line

Official ssl-vision **does run and can play back real recorded footage**, but
**out-of-the-box detection output is basically empty** — camera calibration
and colour classification both need manual setup, and the shipped defaults
don't match our scenario at all. This actually supports the case for building
tagvision (automatic line-based calibration + ring markers) this year.

---

## Done

- [x] Set up WSL2 + Ubuntu; worked around `InstallPackagesUbuntu.sh` not
      recognising Ubuntu 26.04 (installed dependencies manually with `apt-get`)
- [x] Compiled successfully (`make` runs clean)
- [x] Downloaded the official test data (`make install_test_data`); fixed a
      missing `unzip` binary and an extra nested folder level (fixed with a
      symlink)
- [x] Got the GUI running, playing back official footage via `Read from
      files` mode (`Capture ~40fps`)

## Problems found

1. **Test data is seriously incomplete**
   - The official zip's `rc2022` set only actually has footage from **cam0
     and cam1** — no cam2/cam3 data at all
   - Each scenario folder only has **4 frames** (`00000–00003.png`), not a
     full video — very little to work with
   - In `rc2022`, cam0 and cam1 each only cover **half the field** (cam0 =
     far half, cam1 = near half, confirmed against the same pattern in the
     `go2024` data set) — this is a real multi-camera layout, not corrupted
     data
   - No extra/alternate test data could be found anywhere else in the repo;
     `make install_test_data` pulls everything that's officially available.
     If we want more footage, we'll have to record our own (see "Next steps")

2. **Camera calibration (extrinsics) never actually took effect**
   - `Camera Calibrator`'s `focal length / principal point / q0–q3 /
     tx,ty,tz` are all factory placeholder defaults (e.g. principal point
     390/290 while the real image is 2448×2048 — clearly never calibrated
     for this footage)
   - The red geometric reference overlay doesn't line up with the real white
     field lines at all
   - Root cause: official calibration is a **manual, multi-step GUI
     workflow** — drag 4 control points onto the field corners, then "Do
     initial calibration" → "Detect additional calibration points" (uses a
     Sobel edge operator internally) → "Do full calibration" (non-linear /
     Levenberg–Marquardt refinement). None of this is automatic.

3. **Colour classification thread count = 0**
   - `Color Threshold → number of threads` showed `0`, so this stage wasn't
     actually running
   - This starves `Blob Finding` / `Robot Detection` downstream — no
     detection boxes show up on screen at all
   - **Pitfall:** setting `number of threads` to `1` did **not** fix it. The
     thread count only controls *parallelism/speed*; the real missing piece
     is the colour lookup table (LUT) itself, which is empty because the
     `robocup-ssl-cam-0-lut-yuv.xml` file was never generated. With an empty
     LUT, every pixel gets classified as "no known colour" regardless of how
     many threads are running. Fixing this needs the `YUV Calibrator` panel —
     manually clicking on sample pixels (ball, field, lines, etc.) in the
     video to populate the LUT. There is no automatic way to do this.

4. **The calibration control points wouldn't drag** (new pitfall)
   - Clicking and dragging on the red corner markers in the video pane did
     nothing
   - Root cause: mouse clicks on the video pane are only routed to whichever
     plugin currently owns the right-hand control panel. We had "Image
     Capture" selected (showing the Record/Playback panel), not "Camera
     Calibrator" — so the click never reached the calibration code at all.
     **Fix:** click directly on the "Camera Calibrator" entry in the left
     parameter tree first (not just its expand arrow) to make it the active
     plugin, *then* the 4 control points become draggable in the video pane
     (must click within ~10px of a point's current position).
   - Workaround that avoids dragging entirely: expand `Camera Calibrator →
     Calibration Parameters → Control Point 0–3` and type the pixel
     coordinates into `image x` / `image y` directly — same effect as
     dragging, more precise, no 10px tolerance to fight with.

## Calibration cheat sheet (manual, 5 steps)

| Step | Button | What we do | What it does internally |
|---|---|---|---|
| 1 | — | Set each Control Point's `image x/y` to the pixel location of a real field corner (type the numbers directly, or drag after selecting "Camera Calibrator" as the active plugin) | — |
| 2 | `Do initial calibration` | Click | Runs OpenCV `calibrateCamera` on the 4 point pairs for a rough pose |
| 3 | `Detect additional calibration points` | Click | Auto-finds more points along the field lines using a 3×3 Sobel edge operator |
| 4 | `Do full calibration` | Click | Non-linear (Levenberg–Marquardt) refinement using all points — fits focal length, rotation, translation, principal point, distortion together |
| 5 | `Reset` | Click if it goes wrong | Clears and starts over |

## Next steps

- [ ] Re-attempt calibration using the corrected 5-step process above
      (typing control-point coordinates directly, not dragging)
- [ ] Manually populate the colour LUT via `YUV Calibrator` click-sampling,
      then re-check `Blob Finding` / `Robot Detection` output
- [ ] Try feeding our own recorded footage into ssl-vision (needs `ffmpeg`
      to split video into sequentially numbered frames first, matching the
      `00000.png` naming convention) to see how the official algorithm
      performs under our real field's lighting conditions — this requires
      robots to be wearing the official coloured-dot markers to trigger
      robot detection, which ours currently aren't, so we may need to
      temporarily fit a test set for a fair comparison
- [ ] Write up "the official system needs a lot of manual calibration to
      work at all" as supporting evidence for why we're building the
      automated tagvision pipeline
