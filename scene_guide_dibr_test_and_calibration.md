# Scene Guide — DIBRTestScene and CalibrationScene

## What these scenes are for

The project synthesizes novel viewpoints between two cameras using
Depth Image Based Rendering (DIBR). These two scenes support the offline
test phase of that work:

- **CalibrationScene** — a one-time operator step. Records how the cameras
  relate to each other in space (intrinsics + extrinsics). Output is a
  `calibration.json` file.

- **DIBRTestScene** — the actual DIBR test. Loads a color video and a depth
  video per camera, builds a 3D mesh from each frame's depth, and renders a
  synthesized viewpoint between the two cameras in real time.

Neither scene is intended for end-users. They are operator/dev tools.

---

## CalibrationScene

### What it does

Calibration answers two questions:

1. **Intrinsics** — for each camera individually: what are its focal length
   and principal point? (How does the lens map the 3D world onto the 2D
   image sensor?) These change with lens settings and stay fixed once set.

2. **Extrinsics** — for both cameras together: how are they positioned and
   oriented relative to each other? This is needed to project each camera's
   depth mesh into a shared 3D space.

Both are measured by showing a known target (a ChArUco board) to the cameras
and letting OpenCV for Unity solve the geometry from the detected corners.

### Scene hierarchy

```
CalibrationScene
├── Cam1Calibrator            VideoPlayer + CameraCalibrator
├── Cam2Calibrator            VideoPlayer + CameraCalibrator
├── SceneCalibratorManager    SceneCalibrator (coordinates the two)
├── EventSystem
└── Canvas
    ├── Cam1Panel             RawImage — cam1 video feed preview
    ├── Cam2Panel             RawImage — cam2 video feed preview
    ├── Cam1ControlPanel      Process Cam1 / Stop Cam1 / Calibrate Cam1 + Cam1StatusText
    ├── Cam2ControlPanel      Process Cam2 / Stop Cam2 / Calibrate Cam2 + Cam2StatusText
    └── GlobalControlPanel    Capture Extrinsics / Export JSON + GlobalStatusText + OutputPathText
```

### How to use it (step by step)

**Before entering the scene:**
- Print the ChArUco board (generate via `Tools → DIBR → Generate ChArUco Board`
  in the editor, or via the CharucoBoardPopup in the built app; default is a
  5×7 board with 30mm squares, DICT_5X5_250). Print at 100% scale, no
  fit-to-page, and measure one square with a ruler — the physical size needs
  to match `CameraCalibrator.squareLengthM`.
- Record two intrinsics videos: one per camera. Move the board around the
  camera's full field of view — tilt it, rotate it, move it to different
  corners. At least 30 frames where the board is cleanly detected are needed.
  30 seconds of footage at various angles is usually enough.
- Record an extrinsics video (or just keep the cameras running): a frame where
  both cameras see the board at the same time from the same board position.

**In the scene:**
1. Assign each `VideoPlayer` (on `Cam1Calibrator` and `Cam2Calibrator`) an
   intrinsics recording in the Inspector (`clip` field).
2. Press **Process Cam1** → the video plays and the `CameraCalibrator` samples
   one frame every 5 frames, running ChArUco corner detection on each. The
   status text shows the accumulated frame count.
3. When the video ends (or after enough frames are collected), press
   **Stop Cam1**, then **Calibrate Cam1**. The reprojection error is reported
   in `Cam1StatusText`. A value under 1.0 px is good; under 0.5 px is great.
4. Repeat steps 2–3 for Cam2.
5. Switch the VideoPlayer clips to the extrinsics video (a frame where both
   cameras see the board simultaneously). Seek to the clearest frame.
6. Press **Capture Extrinsics** — the scene pauses both videos and runs
   `estimatePoseCharucoBoard()` on the current frame from each calibrator.
   The board-relative pose (rvec, tvec) is saved internally for each camera.
7. Press **Export JSON** — writes `calibration.json` to
   `Application.persistentDataPath` and shows the full path in OutputPathText.
   Copy that file to wherever `DIBRTestScene` expects it (`calibrationJsonPath`
   field on `DIBRSceneManager`).

### Data in calibration.json

```json
{
  "cam1": {
    "cameraName": "cam",
    "intrinsics": {
      "fx": 800.0, "fy": 800.0, "cx": 640.0, "cy": 360.0,
      "distCoeffs": [0.01, -0.02, 0, 0, 0],
      "imageWidth": 1280, "imageHeight": 720,
      "reprojectionError": 0.42
    },
    "extrinsics": {
      "rvec": [0.01, -0.02, 0.003],
      "tvec": [-0.5, 0.0, 0.0]
    }
  },
  "cam2": { ... },
  "depthMin": 0.3,
  "depthMax": 5.0
}
```

`rvec`/`tvec` are the board-to-camera rotation and translation vectors in
OpenCV convention (Rodrigues rotation, metres). `depthMin`/`depthMax` are
set manually in the Inspector on `SceneCalibratorManager` based on your
physical scene — they tell `DepthMeshBuilder` how to convert the 0–255 depth
pixel range to real-world metres (0=far, 255=near for Depth Anything V2 output).

---

## DIBRTestScene

### What it does

Loads a color video and a depth video per camera, builds a 3D point-cloud mesh
from the depth on every frame, textures it with the color video, and renders the
result from an interpolated virtual camera position. Moving the slider moves the
virtual camera between the two real camera positions.

### Scene hierarchy

```
DIBRTestScene
├── Cam1 (at [-1, 0, 0])
│   └── Cam1Mesh              MeshFilter + MeshRenderer + DepthMeshBuilder
├── Cam2 (at [1, 0, 0])
│   └── Cam2Mesh              MeshFilter + MeshRenderer + DepthMeshBuilder
├── VirtualCamera             Camera, renders to a 1280×720 RenderTexture at runtime
├── VideoSources
│   ├── Cam1Source            VideoFileSource (color + depth VideoPlayers)
│   └── Cam2Source            VideoFileSource
├── DIBRManager               DIBRSceneManager (wires everything together)
├── EventSystem
└── Canvas
    ├── OutputDisplay         RawImage — shows VirtualCamera's RenderTexture
    ├── Cam1Preview           RawImage — shows Cam1Source color texture
    ├── Cam2Preview           RawImage — shows Cam2Source color texture
    ├── OutputLabel
    ├── Cam1Label / Cam2Label
    └── ControlPanel
        ├── LoadButton
        ├── Play/PauseButton
        ├── SliderLabel
        ├── ViewSlider        0 = pure Cam1, 1 = pure Cam2, 0.5 = midpoint
        └── StatusText
```

### How to use it (step by step)

**Before entering the scene:**
- Have your color and depth MP4 files per camera. Depth videos are produced
  by running Depth Anything V2 on the color recordings (`python run_video.py
  --encoder vitl --grayscale ...`). Depth output is a grayscale video where
  bright = near, dark = far.
- Have `calibration.json` from CalibrationScene.

**In the scene:**
1. On `Cam1Source` (child of `VideoSources`), set `colorClipPath` and
   `depthClipPath` in the Inspector to your cam1 color and depth MP4 paths.
   Repeat for `Cam2Source`.
2. On `DIBRManager` (`DIBRSceneManager`), set `calibrationJsonPath` to the
   absolute path of `calibration.json`.
3. Adjust `Cam1` and `Cam2` GameObject transforms to match the real physical
   camera positions (roughly — the calibration data will fine-tune projections,
   but the transforms set the scene-space coordinate reference).
4. Enter Play mode. Press **Load** — both video sources prepare, calibration
   is loaded, the depth meshes get their intrinsics from the JSON.
5. Press **Play** — videos start playing, meshes rebuild every frame from the
   depth data, color textures update.
6. Move the **ViewSlider** — the virtual camera lerps position between
   `Cam1Transform` and `Cam2Transform`, and mesh opacity cross-fades so cam1's
   mesh fades out as you slide toward cam2.

### How DepthMeshBuilder works

`DepthMeshBuilder` reads the depth `RenderTexture` every frame via
`ReadPixels`, then iterates over every `downsampleFactor`-th pixel (default 4,
so 1280×720 → 320×180 = 57,600 vertices) and projects each pixel into 3D:

```
d     = Lerp(depthMax, depthMin, depthNorm)   // bright = near, dark = far
X     = (srcX - cx) / fx * d
Y     = -((srcY - cy) / fy * d)               // flip Y: OpenCV y-down → Unity y-up
Z     = d
```

Adjacent vertices that differ in depth by more than `depthDiscontinuityThreshold`
(default 0.5 m) don't get a triangle between them — this prevents "rubber sheet"
artefacts at object edges.

The mesh is assigned `DepthMeshMaterial` (URP unlit, transparent, two-sided),
which multiplies the color texture by a `_BaseColor.a` tint. The alpha is set
by `DIBRSceneManager` per frame to cross-fade the two meshes based on the slider.

### What "novel view synthesis" looks like in this setup

At slider = 0.5 (midpoint), the virtual camera sits exactly between Cam1 and
Cam2. Both meshes are rendered at 50% opacity into the same RenderTexture.
Surfaces seen by both cameras blend together; surfaces only seen by one camera
fade out toward the centre.

This is a simplified version of DIBR — it uses alpha blending rather than
proper occlusion-based hole-filling. The quality is sufficient to validate:
- that calibration data is correct (meshes align in 3D space at the midpoint)
- that depth estimation quality is acceptable (no severe geometric distortion)
- that the depth scale is right (`depthMin`/`depthMax` in calibration.json)

Full DIBR with hole-filling is what open-dibr does and is the Phase 2/3 target.

---

## Workflow end to end

```
[Print ChArUco board]
        ↓
[Record: intrinsics videos (one per cam) + extrinsics video (both cams + board)]
        ↓
[CalibrationScene → export calibration.json]
        ↓
[Run Depth Anything V2 on color recordings → depth MP4s]
        ↓
[DIBRTestScene → Load → Play → slide virtual camera → evaluate quality]
        ↓
[Feed calibration.json + color/depth MP4s into open-dibr (Phase 2c)]
```

---

## Notes

- **Depth scale**: Depth Anything V2 outputs relative depth. `depthMin` and
  `depthMax` in `calibration.json` need to be set so the 0–255 range maps to
  real metres. Calibrate by placing a known object (a flat board, a box) at
  measured distances (e.g. 0.5m, 2m, 4m) and reading what depth pixel value
  comes out. Then solve: `depthMin` is where pixel=0 maps to, `depthMax` is
  where pixel=255 maps to.

- **`downsampleFactor`**: increasing it makes the mesh coarser but faster.
  At 4 (default) the mesh has ~57k vertices and rebuilds comfortably at 30fps
  on a modern CPU. At 2, it's ~230k vertices and becomes compute-bound.

- **Reprojection error threshold**: `CameraCalibrator.CalibrateIntrinsics()`
  prints RPE in the status text. If it's above 1.0px, the intrinsics video
  likely had too few distinct board poses — record more angles and retry.
  Extrinsics estimation can fail silently if the board is too small in frame
  or partially occluded; both cameras need to see at least 4 ChArUco corners
  simultaneously.
