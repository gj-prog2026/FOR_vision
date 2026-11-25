# Vision System Rebuild: Possible Plan

## Context

- **Decision:** the team is rebuilding the vision system **from scratch** instead of extending TagVision.
- **What it must do:** stand in for SSL-Vision on our 4 × 2 m field. It sends `SSL_WrapperPacket` protobufs over UDP multicast `224.5.23.2:10006`, so the existing AI and tracker code runs unchanged.
- **Team goals:**
  - Setup is **fast**, with **little manual calibration**.
  - Measuring **camera intrinsics is easy**.
  - It **performs in real time**. TagVision never reached 2 × 1080p60 at under 50 ms.
- **TagVision's role:** we use it as a **reference implementation** (ring codec, camera model, protobuf output), not as a base to build on.

---

## 1. Problems in TagVision we must avoid

| Problem in TagVision | What we do instead |
|---|---|
| Written Python-first, so it fights the GIL, runs at 16–45 fps, drops frames and has ~60 ms full-frame passes | Set a **latency budget** and decide the **hot-path language** (C++/pybind, Rust, or optimised NumPy/OpenCV) **on day 1** |
| Accuracy tested almost only on synthetic data (`tests/fixtures/real/` is empty) | Record **real footage from our field** in week 1 and test against it |
| Compression artefacts cause ball false positives; the ball is found by hue only | Lock camera settings and reject compression fringes. Consider checking shape and size as well as colour |
| Only 30 marker IDs, no error correction, and the 170 mm marker doesn't fit a 180 mm robot | Settle the **marker system and size** before writing the detector |
| Line tracer takes 200–340 ms per solve | Start with **simple 4-corner / `solvePnP` calibration**. Automatic line-based pose comes later |
| Large (~31k lines, ~7.5k for Android alone), written by an agent, hard to own | Keep the core small. Leave phones, the full renderer and the auto line tracer until the core works |
| Linux-only paths (V4L2, `/dev/v4l/by-id`) | Decide which platforms we support up front |

---

## 2. Decisions to make as a team (Week 0)

- [ ] **Latency and FPS budget.** Example: 2 cameras, 1080p60, < 50 ms end to end. Each frame then has 16.7 ms in total.
- [ ] **Hot-path language** (detection and projection).
- [ ] **Marker system:** AprilTag, ArUco, or a custom ring like TagVision's. Fix the ID count, error correction and **printed size (140–150 mm)**.
- [ ] **Camera hardware:** model, resolution, global or rolling shutter, MJPEG or raw, lens field of view, mounting height.
- [ ] **Camera settings policy:** auto-exposure, auto white balance and autofocus **off**. Short exposure, set in multiples of 10 ms because of 50 Hz mains flicker.
- [ ] **Calibration workflow:**
  - Intrinsics: ChArUco board, measured once per camera and lens, stored per device.
  - Extrinsics: click 4 field corners → `solvePnP`, with reprojection error shown to the user.
- [ ] **Per-robot height planes** (marker height on each robot) and ball height.
- [ ] **Ground-truth format** for recorded footage.

---

## 3. Shared contracts (the `core` package)

Agree these first and put them in a small shared package **with stubs**, so everyone can build against fake data from day one:

- [ ] `Frame`: `image`, `t_capture`, `frame_number`, `meta`
- [ ] `MarkerDetection`: pixel centre, ID, heading reference point, confidence
- [ ] `BallDetection`: pixel centre, radius, confidence
- [ ] `CameraModel` API: `project()`, `pixel_to_plane(height)`, `undistort_points()`
- [ ] Config schema (cameras, intrinsics file, pose, HSV ranges, robot heights)
- [ ] Output: reuse the SSL protobufs as they are
- [ ] Shared folder of **real recorded footage** plus ground truth

---

## 4. Who owns what (4 people)

| Role | Owns | First deliverable (unblocks others) |
|---|---|---|
| **A: Platform and I/O** | Camera sources, record/replay, per-camera process, config, protobuf + multicast, timing stats | File/replay source and a "send fake detections" sender |
| **B: Detection** | Marker detector, ball detector, ROI tracking, marker printing | Detector running on recorded frames, plus an accuracy script |
| **C: Geometry and calibration** | Camera model, ChArUco intrinsics, pose from field points, height-plane projection, heading | `CameraModel` + 4-corner calibration |
| **D: Test harness and GUI** | Ground-truth tools, benchmarks/CI, live viewer with overlays, HSV tuning, calibration wizard | Ground-truth scoring tool + minimal live viewer |

- [ ] Assign people to roles.

---

## 5. Phases

### Weeks 1–2: walking skeleton
- [ ] Replayed video → stub detector → fixed pose → real packets
- [ ] Packets visible in **`ssl-vision-client`**
- [ ] Integrate as soon as it runs end to end, **not at the end**

### Weeks 3–6: real components
- [ ] Swap in the real detector, calibration and live camera source
- [ ] Run **accuracy and FPS benchmarks on every merge** (detection rate, false positives, median and p95 error, latency)
- [ ] Test on real footage from our field

### Week 7 onwards: robustness
- [ ] Multiple cameras and checks that cameras agree with each other
- [ ] Coping with lighting changes
- [ ] GUI polish (calibration wizard, HSV tuning)
- [ ] Optional: automatic line-based pose refinement, synthetic renderer, phone cameras

---

## 7. Questions to bring to the team

1. What latency are the AI and tracker teams actually able to tolerate?
2. Which cameras do we already have, and do they support manual exposure and uncompressed output?
3. Can we change the robot top plates to fit a 140–150 mm marker?
4. Will we ever use more than one camera at the same time on this field?
5. Who will own long-term maintenance once the rebuild is done?
