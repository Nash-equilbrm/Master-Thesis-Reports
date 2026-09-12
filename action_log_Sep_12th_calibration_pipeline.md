# Action Log — Sep 12th (live calibration tutorial + server calibration pipeline spec)

## Session summary

Client-only session (`Master-Thesis-Client`). Built a live, in-app, guided
ChArUco calibration tutorial for the Camera role, replacing the earlier idea
of a bare "please calibrate" notice popup. Board identity (dictionary/grid/
size) and per-camera calibration results are meant to be server-owned, but
**no server code was touched this session** — this doc is the handoff spec
for that work.

---

## Why

DIBR needs metric stereo depth between camera pairs (see `action_log_Sep_11th_opendibr.md`'s
stereo-via-OpenCV-for-Unity TODO), which needs every camera calibrated against
the *same* physical ChArUco board. Two problems with the pre-existing setup:

1. `CharucoBoardPopup.cs` (free-text board-size fields, `PlayerPrefs`-persisted
   per device) and `CameraCalibrator.cs` (separate hardcoded Inspector fields)
   could silently drift onto different board parameters across cameras, with
   nothing keeping them in sync.
2. Calibration only existed as an offline dev tool (`CalibrationScene`,
   pre-recorded video files) — no in-app path for a camera operator to
   calibrate their own device live before streaming.

Resolution, arrived at over the planning session:
- Board parameters become server-provided (a handful of numbers — the image
  is generated deterministically client-side from them, so nothing but a few
  ints/floats needs to cross the wire).
- Each camera device runs a live, guided tutorial (marker → intrinsics →
  extrinsics, auto-advancing) using its own camera feed, then uploads its own
  result to the server, keyed by its `/register` slot identity.
- Extrinsics capture is single-camera by nature — `estimatePoseCharucoBoard()`
  only ever computes the board's pose relative to *that* camera — so no
  camera needs to see another camera's feed to finish its own calibration.
- The server is what later hands two cameras' stored calibration to a Viewer
  requesting synthetic-view generation between those two slots. That
  consumption (stereo depth on the Viewer side) is separate, later work.

---

## Required server endpoints (not yet implemented)

### `GET /calibration-config`

Returns the ChArUco board identity every camera must render/calibrate
against. Suggested to live in `registration-service/server.js` near the
existing `/viewer-token` handler, config sourced from env vars the same way
`SLOT_COUNT` etc. already are (top-of-file `const X = process.env.X || default`).

```json
{
  "dictionaryId": 10,
  "squaresX": 5,
  "squaresY": 7,
  "squareLengthMm": 30.0,
  "markerLengthMm": 15.0
}
```

`dictionaryId: 10` = OpenCV's `Aruco.DICT_5X5_250`.

Suggested env vars (defaults match the client's built-in fallback, so nothing
breaks if these aren't set): `CALIBRATION_DICTIONARY_ID`, `CALIBRATION_SQUARES_X`,
`CALIBRATION_SQUARES_Y`, `CALIBRATION_SQUARE_LENGTH_MM`, `CALIBRATION_MARKER_LENGTH_MM`.

### `POST /calibration-data`

Called once by each camera device after it finishes its own live tutorial.
Store per-slot (keyed by `identity`, the same string `/register` already
hands out, e.g. `"cam1"`), overwriting any previous entry for that identity.

```json
// request body
{
  "identity": "cam1",
  "cameraName": "cam1",
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
}
```

No response body needed beyond a 2xx — the client only checks HTTP result,
not response content.

### `GET /calibration-data/pair?cam1=<identity>&cam2=<identity>`

For a later Viewer-side feature (not built yet): given two camera slot
identities, return both cameras' stored calibration so the viewer can run
stereo depth between them.

```json
{
  "cam1": { "intrinsics": {...}, "extrinsics": {...} },
  "cam2": { "intrinsics": {...}, "extrinsics": {...} }
}
```

404/empty response if either identity has no stored calibration yet — the
client isn't built to consume this yet, so exact error shape is flexible.

**Note:** `depthMin`/`depthMax` (present in the old offline `SceneCalibrationData`/
`calibration.json`) are **not** part of this new payload — they were a Depth-
Anything-V2 relic; stereo depth computes metric depth directly from
disparity + baseline (`depth = fx × baseline / disparity`), so no manual
depth-range calibration step is needed for this path.

---

## Client-side implementation (this session, all in `Master-Thesis-Client`)

**New:**
- `Assets/Scripts/Calibration/CameraCalibrationData.cs` — added `BoardConfig`
  (dictionaryId, squaresX, squaresY, squareLengthMm, markerLengthMm).
- `Assets/Scripts/Calibration/CharucoBoardGenerator.cs` — static PNG generator
  from a `BoardConfig`, shared by the dev popup and the live tutorial so both
  render pixel-identical boards.
- `Assets/Scripts/Manager/CalibrationConfigClient.cs` — GET-fetches
  `/calibration-config`; falls back to a hardcoded default (matching the
  numbers above) and a `PlayerPrefs`-cached last-known-good config if the
  server is unreachable, so client work isn't blocked on this endpoint
  existing yet.
- `Assets/Scripts/Manager/CalibrationUploadClient.cs` — POSTs to
  `/calibration-data` once a device finishes its tutorial.
- `Assets/Scripts/UI/Screen/CalibrationTutorialScreen.cs` — the guided flow:
  Step 0 (view/download the marker), Step 1 (intrinsics — auto-advances once
  `AccumulatedFrames >= minFramesForCalibration`), Step 2 (extrinsics —
  auto-completes on the first successful pose estimate), then uploads and
  fires `OnCalibrationComplete`. Reads live frames directly from
  `LiveKitCameraPublisher.Instance.Texture` (the same `WebCamTexture` already
  open for the self-preview — not a second camera handle).
- `Assets/Scripts/Manager/States/CameraClientManager.CalibratingState.cs` —
  new state between `Connecting` (camera open + room joined) and `Streaming`
  (publishing). Skips straight to publish if `AppConfig.CalibrationAcknowledged`
  is already true; otherwise shows the tutorial screen and waits for it.

**Modified:**
- `Assets/Scripts/Calibration/CameraCalibrator.cs` — board building moved from
  `Awake()` (hardcoded fields) into a `Configure(BoardConfig)` method; frame
  processing decoupled from `VideoPlayer` so both the offline `CalibrationScene`
  path and the new live path share one OpenCV pipeline (`ProcessLiveFrame`/
  `EstimatePoseFromTexture` added alongside the existing `VideoPlayer`-driven
  entry points).
- `Assets/Scripts/Calibration/SceneCalibrator.cs` — now fetches `BoardConfig`
  via `CalibrationConfigClient` at `Start()` and calls `Configure(...)` on both
  calibrators before enabling capture buttons, instead of relying on
  independently-hardcoded defaults.
- `Assets/Scripts/UI/Popup/CharucoBoardPopup.cs` — board-size fields are no
  longer editable; displays the fetched config read-only and generates via
  `CharucoBoardGenerator`.
- `Assets/Scripts/AppConfig.cs` — added `CalibrationAcknowledged` (persisted
  flag, set after a successful upload).
- `Assets/Scripts/Manager/LiveKitCameraPublisher.cs` — `StartStreamingRoutine()`
  now stops after camera-open + room-join and fires `OnReadyToPublish`; actual
  publish only happens when something calls the new `PublishNow()` — lets the
  state machine gate publish on calibration finishing.
- `Assets/Scripts/UI/Screen/ConnectionScreen.cs` — added a "Recalibrate"
  button (camera branch only) that clears `CalibrationAcknowledged` so the
  next connect re-runs the tutorial.

**Prefab/scene wiring (done this session):**
- `CalibrationTutorialScreen.prefab` built under `Assets/Resources/Prefabs/UI/Screen/`
  (Marker/Live panels, live preview `RawImage`+`AspectRatioFitter`, instruction/
  progress text, Download/Continue buttons, a child `Calibrator` GameObject
  with `CameraCalibrator` and no `VideoPlayer` assigned).
- `CharucoBoardPopup.prefab` created from scratch (it never existed before this
  session, despite the script referencing it since Sep 11th) — read-only
  `_boardInfoText`, save-dir field, Generate/Close buttons.
- `ConnectionScreen.prefab` — added `RecalibrateButton`.
- `DevCommandPopup.prefab` — added the missing `CharucoBoardButton` (script had
  referenced it since Sep 11th but the prefab button never existed) and a new
  `ToggleCalibrationBypassButton` (see bypass note below).
- Scene wiring: `CalibrationConfigClient` added to `CalibrationScene.unity`
  (root-level, needed by `SceneCalibrator`); `CalibrationConfigClient` +
  `CalibrationUploadClient` added to `Main.unity` under the `CameraManagers`
  container (sibling of `RegistrationClient`/`LiveKitCameraPublisher`, same
  inactive-until-camera-role-selected lifecycle).
- All of the above verified compiling clean and with every serialized field
  wired (checked via `SerializedObject` field dump, no nulls).

**Dev bypass added:** `AppConfig.DevSkipCalibrationUpload` (toggled via the new
`DevCommandPopup` button) lets `CalibrationTutorialScreen` complete and unblock
publishing even if the `/calibration-data` POST fails — needed because, by
design (approved plan: don't silently proceed to stream without the server
having this camera's calibration), the tutorial otherwise retries the upload
forever on failure and **no camera can ever reach `StreamingState` until
`/calibration-data` exists server-side**. Flip the bypass on for local
testing before that endpoint lands; leave it off otherwise.

**Still needed:** actual end-to-end run — fresh camera connect → camera opens,
room joins, no publish yet → tutorial runs against a real/printed board →
upload (will fail until the server endpoint exists; use the bypass above to
get past it) → publish proceeds. Not run yet this session (needs the physical
board + a camera device).
