# Action Log — Sep 11th (open-dibr integration planning)

## Session Summary

Architecture planning session: how to make open-dibr work with live camera feeds.
No code written — research and decision-making only. Full plan saved to
`~/.claude/plans/okay-now-that-you-cuddly-fern.md`.

---

## Research findings

### open-dibr input pipeline (from codebase exploration)
- **File-based only** — `FFmpegDemuxer` calls `avformat_open_input(filePath)`. No live
  stream abstraction exists. However, libavformat natively supports RTSP/pipe URLs in
  the same call — only 3–4 lines of C++ needed to enable it (not needed for PoC).
- **Two decoders per camera** — `decoders[2*i]` (color) + `decoders[2*i+1]` (depth).
  Depth is mandatory; no depth = crash. Decoded via NvDecoder on GPU, copied to OpenGL
  texture via CUDA-GL interop.
- **JSON config** requires per-camera: `Focal [fx,fy]`, `Principle_point [cx,cy]`,
  `Position`, `Rotation`, `Resolution`, `Depth_range`, `NameColor`, `NameDepth`.

### LiveKit Unity SDK video pipeline
- Frames arrive as I420 (YUV planar) via native Rust FFI, exposed through `VideoStream`
- Raw I420 plane pointers accessible via `VideoFrameBuffer.Info.Components[].DataPtr`
- `VideoStream` converts to RGBA `RenderTexture` for Unity display
- Could intercept at I420 level before conversion for lowest-overhead frame extraction

### Three gaps identified
1. **No calibration data** — need intrinsics + extrinsics per camera
2. **No depth** — cameras send RGB only; depth estimation required
3. **open-dibr reads files only** — needs RTSP support for real-time (small C++ change)

---

## Decisions made

### Calibration: ChArUco markers (Python + OpenCV)
- Tool: `opencv-contrib-python` (free, standard) — NOT OpenCV for Unity (paid asset)
- Print one ChArUco board (A1/A0), place in scene centre visible to all cameras
- Python script subscribes to all camera streams via `livekit-agents`, captures frames,
  runs `calibrateCameraCharuco()` (intrinsics) + `estimatePoseBoard()` (extrinsics)
- Output: `calibration.json` in open-dibr format

Two options considered:
- **ChArUco markers** (chosen) — sub-pixel accuracy, one session, reliable for DIBR
- **Markerless COLMAP** — no printed target, viable (cameras have significant overlap),
  but more complex pipeline and accuracy depends on scene texture. Keep as cross-check.

### Depth estimation: Depth Anything V2 (offline first)
- Offline PoC: run on recorded MP4s, output grayscale depth MP4s
- Real-time: Python process subscribes to LiveKit, runs Depth Anything V2 per frame,
  serves output as localhost RTSP — **no server-side changes, no Docker changes**

### No server involvement required
The depth estimation process runs locally on the same machine as open-dibr. It
subscribes to LiveKit as a participant (same mechanism as the Unity viewer). The
Docker stack, registration service, and EC2 deployment are untouched.

---

## Phased plan

| Phase | Task | Est. time |
|-------|------|-----------|
| 1 | Camera calibration Python script | 1–2 days |
| 2a | Record multi-camera session | 0.5 day |
| 2b | Depth Anything V2 setup + process recordings | 1–2 days |
| 2c | open-dibr offline test + tuning | 1–2 days |
| 3a | Real-time depth estimation process (Python) | ~1 week |
| 3b | open-dibr RTSP input (C++ patch) | 1–2 days |
| 3c | Integration + latency tuning | ~1 week |
| **Total to real-time** | | **~3–4 weeks** |

---

## Next steps (start of next session)

1. **Phase 1 — calibration script**: write Python script using `opencv-contrib-python`
   + `livekit-agents` to calibrate all cameras and output open-dibr JSON.
   - Print ChArUco board first (board parameters TBD — pick dictionary + grid size)
2. **Phase 2a** — record a multi-camera session while server is running
3. **Phase 2b** — install + test Depth Anything V2 on a single recording

---

## Later same day: implementation (Phase 1 Unity side + test scenes)

Reconsidered calibration toolchain: **OpenCV for Unity** (already imported
as a paid asset) replaces the `opencv-contrib-python` plan for the Unity-side
calibration work. The Python script approach is still valid for Phase 3a
(real-time depth estimation / LiveKit subscription), but for the calibration
pipeline the Unity asset means no separate Python environment is needed and
the operator can run it inside the same app they already use.

Also decided to validate the full DIBR pipeline (depth mesh rendering,
virtual camera interpolation) with a simple Unity test scene before committing
to the open-dibr C++ integration. This gives a visual sanity check on
calibration quality and depth scale before Phase 2c.

### Scripts written (all in `Master Thesis Client`)

**Calibration (`Assets/Scripts/Calibration/`):**
- `CameraCalibrationData.cs` — `IntrinsicsData`, `ExtrinsicsData`,
  `SceneCalibrationData` data classes; JSON serialization via `JsonUtility`
- `CameraCalibrator.cs` — MonoBehaviour wrapping a `VideoPlayer`;
  `StartProcessingVideo()` accumulates frames with ChArUco detected (every
  5th frame, minimum 30); `CalibrateIntrinsics()` runs
  `calibrateCameraCharuco()`; `EstimatePoseFromCurrentFrame()` runs
  `estimatePoseCharucoBoard()`. Events: `OnIntrinsicsCalibrated`,
  `OnExtrinsicsEstimated`
- `SceneCalibrator.cs` — coordinates two `CameraCalibrator` instances,
  exposes buttons for each step, writes `calibration.json` on export

**DIBR (`Assets/Scripts/DIBR/`):**
- `VideoFileSource.cs` — wraps two `VideoPlayer`s (color + depth);
  `Load()` / `Play()` / `Pause()` / `Stop()`; fires `OnReady` when both
  prepared; exposes `ColorTexture` and `DepthTexture` as `RenderTexture`
- `DepthMeshBuilder.cs` — reads depth `RenderTexture` each frame via
  `ReadPixels`, projects pixels to 3D using camera intrinsics (with OpenCV
  y-down → Unity y-up flip), builds and uploads a mesh; per-frame alpha
  for cross-fade blending; discontinuity culling at object edges;
  `IndexFormat.UInt32` for >65k-vertex meshes
- `DIBRSceneManager.cs` — loads calibration JSON, wires sources → meshes →
  virtual camera; moves virtual camera via Lerp/Slerp based on a UI slider;
  cross-fades mesh alpha (`cam1 = 1−t`, `cam2 = t`)

**Shader (`Assets/Shaders/DIBR/DepthMesh.shader`):**
- URP Unlit, transparent, two-sided (`Cull Off`), `ZWrite Off`
- `_MainTex` (color) × `_BaseColor.a` (cross-fade alpha from script)

**UI popups / editor tools:**
- `CharucoBoardPopup.cs` — runtime board generator in the dev gesture menu;
  saves to user-specified path; remembers settings in PlayerPrefs
- `GenerateCharucoBoardWindow.cs` — editor menu `Tools → DIBR → Generate ChArUco Board`
- `DevCommandPopup.cs` — added ChArUco board button

All committed in `Master Thesis Client` commit `be5fa5e`.

### Scenes built (Unity MCP)

**`Assets/Scenes/DIBRTestScene.unity`**
- `Cam1` [-1,0,0] → `Cam1Mesh` (DepthMeshBuilder + DepthMeshMaterial)
- `Cam2` [1,0,0] → `Cam2Mesh` (DepthMeshBuilder + DepthMeshMaterial)
- `VirtualCamera`, `VideoSources/Cam1Source` + `Cam2Source`,
  `DIBRManager` (all Inspector refs wired)
- Canvas: OutputDisplay, Cam1Preview, Cam2Preview, ControlPanel (Load,
  Play/Pause, ViewSlider, StatusText)

**`Assets/Scenes/CalibrationScene.unity`**
- `Cam1Calibrator` / `Cam2Calibrator` (VideoPlayer + CameraCalibrator,
  status texts wired)
- `SceneCalibratorManager` (SceneCalibrator, all 8 buttons wired)
- Canvas: per-camera feed panels + control panels + global panel
  (Capture Extrinsics, Export JSON, GlobalStatusText, OutputPathText)

Also committed: `Assets/Materials/DIBR/DepthMeshMaterial.mat`.

Committed `44e5005`, pushed to `origin/main`.

A separate scene guide describing how to use both scenes step by step is at
`Master-Thesis-Reports/scene_guide_dibr_test_and_calibration.md`.

### What's still needed before running the test

- [ ] Assign video clip paths (`colorClipPath`, `depthClipPath`) on
  `Cam1Source` and `Cam2Source` in the Inspector — needs real recorded files
- [ ] Set `calibrationJsonPath` on `DIBRManager`
- [ ] Physical: print ChArUco board, record intrinsics + extrinsics videos
- [ ] Run Depth Anything V2 on the color recordings to produce depth MP4s
- [ ] Set accurate `depthMin` / `depthMax` on `SceneCalibratorManager` once
  depth scale is measured against known distances

### What changed vs. the morning's plan

- Calibration toolchain: OpenCV for Unity (in-app) replaces Python +
  `opencv-contrib-python` for the calibration step. Python path remains
  for the real-time depth estimation process in Phase 3a.
- Added a Unity DIBR test scene (Phase 2c equivalent) as a sanity check
  before moving to the open-dibr C++ integration — not in the original plan
  but lower friction than setting up open-dibr for a first quality check.
- `.gitignore` updated to exclude `Assets/OpenCVForUnity/` (large paid asset,
  not committed). Committed `42c1ddb`.
