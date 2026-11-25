# Vision System: Simple Task Split for 4 People

_Each person builds **one small Python file** in parts, like the Mini-TagVision project. Each part ends with a **"Done when"** check so you know when to move on._

---

## Step 0: Use these default choices (no big decisions yet)

These defaults are deliberately simple. Change them later, once things work and you've measured them.

| Topic | Default for now |
|---|---|
| Language | **Python + OpenCV + NumPy** only. Measure speed before optimising anything |
| Robot markers | **OpenCV ArUco markers**, dictionary `DICT_4X4_50` (50 IDs, built-in error checking, corners give heading) |
| Ball | Orange ball found by **HSV colour + blob size/roundness** |
| Camera | **One** USB webcam, **1280×720 at 30 fps** to start |
| Camera settings | Auto-exposure / auto white balance / autofocus **off** if the camera allows it |
| Calibration | Lens: **printed chessboard** (OpenCV tutorial). Camera position: **click 4 field corners → `solvePnP`** |
| Output | Plain **JSON over UDP** first, then **SSL protobuf** once it works |
| Footage | Everyone uses the **same recorded video** in a shared folder |

### The one thing everyone agrees on: the detection format

Every file passes data in this shape:

```python
detections = {
    "frame_number": 123,
    "t_capture": 1732500000.123,   # time the picture was taken (seconds)
    "robots": [ {"id": 3, "u": 812, "v": 430, "heading_deg": 30.0} ],   # pixels
    "balls":  [ {"u": 640, "v": 360, "confidence": 0.9} ],             # pixels
}
# after geometry, u/v are replaced by x/y in mm on the field
```

---

## Person A: Camera and Output (`camera_io.py`)

_Mostly programming, very little maths._

- **Part 1: Show the camera.** Open the webcam with `cv2.VideoCapture`, show frames and print FPS.
  _Done when:_ a live window shows the FPS.
- **Part 2: Record and replay.** Save the camera to a video file, then play that file back through the **same** code.
  _Done when:_ the team has a shared recording of robots and ball on the field.
- **Part 3: Lock camera settings.** Turn off auto-exposure and auto white balance, set the exposure manually, and print the settings back.
  _Done when:_ the brightness stays steady when you wave your hand in front of the camera.
- **Part 4: Send fake data.** Send the `detections` dict as JSON over UDP multicast. Write a tiny receiver that prints it.
  _Done when:_ the receiver prints the packets on a second terminal or PC.
- **Part 5: Real SSL packets.** Replace the JSON with the SSL protobuf (`SSL_WrapperPacket`) sent to `224.5.23.2:10006`.
  _Done when:_ fake robots appear in **`ssl-vision-client`**.
- **Part 6: Timing.** Record `t_capture` for every frame and print how long each stage takes.
  _Done when:_ you can say "a frame takes X ms in total".

---

## Person B: Detection (`detect.py`)

_Image processing: thresholds, blobs, markers._

- **Part 1: Ball in a photo.** Convert to HSV, threshold orange, find contours and draw a circle on the ball.
  _Done when:_ it works on 3 different photos.
- **Part 2: Ball in video.** Run it on the shared recording. Filter blobs by size and roundness to remove false ones.
  _Done when:_ the circle stays on the ball for most of the video.
- **Part 3: Markers.** Print a few ArUco markers, detect them with `cv2.aruco` and draw their IDs.
  _Done when:_ the IDs show correctly on screen.
- **Part 4: Heading.** Use the order of the marker corners to work out which way the robot faces and draw an arrow.
  _Done when:_ the arrow turns as you rotate the marker.
- **Part 5: Output the format.** Wrap it all in `detect(frame) -> detections`, using the agreed format.
  _Done when:_ Person D can call your function.
- **Part 6: Count mistakes.** On the recording, count missed balls and false balls.
  _Done when:_ you have a number such as "found in 95% of frames, 2 false detections".

---

## Person C: Geometry and Calibration (`geometry.py`)

_The most maths. This is where Mini-TagVision Parts 1, 4 and 5 apply directly._

- **Part 1: Camera model.** `K`, `R`, `t` and `project(world_point) -> pixel`. Draw the field lines on a camera image.
  _Done when:_ the drawn lines look like a field.
- **Part 2: Lens calibration.** Follow the OpenCV chessboard calibration tutorial for the team's webcam and save `K` and the distortion values to a file.
  _Done when:_ the reprojection error is under 1 px.
- **Part 3: Camera position.** Click the 4 field corners in an image, then run `solvePnP` with their real positions in mm.
  _Done when:_ the projected field lines sit on the real white lines.
- **Part 4: Pixel to field.** `pixel_to_field(u, v, height_mm) -> (x, y)`: turn the pixel into a ray and intersect it with the plane at that height.
  _Done when:_ an object placed at a measured spot comes out within a few cm.
- **Part 5: Save and load.** Write the whole calibration to one file that the main program loads.
  _Done when:_ restarting the program doesn't need re-clicking.
- **Part 6: Make it easy.** Build a small click-the-corners tool that anyone on the team can use.
  _Done when:_ someone else can calibrate without your help.

---

## Person D: Viewer, Testing and Glue (`main.py` + `viewer.py`)

_Joins everyone's work together and checks it's right._

- **Part 1: Viewer.** Draw the `detections` dict (circles, IDs, arrows) on top of the video. Start with **fake** detections.
  _Done when:_ fake robots appear on the recording.
- **Part 2: Top-down view.** Draw a 2D picture of the field and plot the robots and ball in x/y mm.
  _Done when:_ fake positions show in the right places.
- **Part 3: Ground truth.** Write a click tool: for about 20 frames of the recording, click where the ball and robots really are and save the clicks to a file.
  _Done when:_ the file exists.
- **Part 4: Score.** Compare detections with the ground truth and print the detection rate and average error in pixels.
  _Done when:_ Person B can run it on their detector.
- **Part 5: Join it all.** In `main.py`: camera (A) → `detect` (B) → `pixel_to_field` (C) → send (A), with the viewer on top.
  _Done when:_ **robots from the real video appear in `ssl-vision-client`** (the team milestone).
- **Part 6: Simple settings file.** Move the HSV values, robot heights and camera index into a `config.json`.
  _Done when:_ nobody needs to edit code to change a setting.

---

## Team milestones

1. **End of week 1:** everyone finishes Parts 1–2. The shared recording exists.
2. **End of week 2:** Person D's Part 5 works on the **recorded video**, end to end into `ssl-vision-client`.
3. **After that:** switch to the **live camera**, measure FPS and latency, and only then decide what needs to be faster.

> **Tip:** meet briefly every few days and run `main.py` together, even if half of it is still fake. Joining the parts early avoids a painful integration at the end.
