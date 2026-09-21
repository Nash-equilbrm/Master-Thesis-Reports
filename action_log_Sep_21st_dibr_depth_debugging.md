# Action Log — Sep 21st (live DIBR grey-view debugging: resolution + control-channel fixes, depth root cause isolated)

Follow-up to `bug_report_Sep_21st_dibr_grey_synthetic_view.md`, which captured
the first live end-to-end run of the `Master-Thesis-Client` + `dibr-bridge` +
`OpenDIBR` pipeline showing a **flat grey synthetic view**. That report listed
two suspected root causes (empty stereo depth from a resolution mismatch, and
an OpenDIBR↔bridge resolution-contract mismatch). Today we fixed the
resolution issues, uncovered and fixed a second, independent blocker on the
Unity side, and isolated the *actual* remaining depth bug down to two specific
lines in the bridge's stereo code. The view is still grey, but the pipeline
now runs to completion on every switch and the remaining work is precisely
scoped and handed to the bridge repo.

---

## 1. Resolution / intrinsics mismatch (bridge) — FIXED

The grey view's primary cause per the bug report was that
`StereoDepthComputer` built its rectify maps at the **calibration resolution**
(aspect 2.222, e.g. 1200×540) while the rest of the pipeline runs at the fixed
OpenDIBR output resolution **1280×720** (aspect 1.778). Two coordinated fixes
landed in the bridge:

**`dibr-bridge/bridge/session.py`**
- Added `_resize(frame)` — a no-op when the frame already matches
  `self._width × self._height`, else `cv2.resize` to the output resolution.
- `add_camera` now creates the color/depth RTSP publishers at the **fixed
  output resolution** (not the camera's native LiveKit frame size), and logs
  `native → output` so mismatches are visible.
- `_pump_color`, `_pump_depth`, and `set_active_pair` resize every frame to
  the output resolution before publishing or feeding stereo.

**`dibr-bridge/bridge/stereo_depth.py`**
- `StereoDepthComputer.__init__` now takes `out_size`; when it differs from
  the calibration resolution, `_prepare` scales `Ka`/`Kb` (fx, cx by `sx`; fy,
  cy by `sy`) before `stereoRectify` / `initUndistortRectifyMap`, and builds
  all maps at `out_size`. This mirrors Unity's `ScaleIntrinsics`
  (`OpenDibrSessionManager.cs`) so the maps match the resized frames.

**Result:** depth is no longer uniformly empty. The `Active pair set` logs went
from `A=[0.10,10.00] B=[0.10,10.00]` (both fallback) to per-camera values,
confirming `compute_pair` now produces real disparity for at least one camera.
The bug report's root cause #2 (native-resolution RTSP publishing) is also
resolved — all streams now publish at 1280×720, matching what
`OpenDibrSessionManager` declares to OpenDIBR.

**Caveat noted for later:** calibration aspect (2.222) still differs from
output aspect (1.778), so frames are anamorphically squished into 1280×720 —
a quality/distortion concern (Unity logs a `>5% aspect mismatch` warning), not
a correctness blocker. Fixing it means matching OpenDIBR's output resolution to
the calibration aspect.

---

## 2. Control-channel timeouts + cache desync (Unity client) — FIXED

With depth partially working, a **second, independent blocker** surfaced:
`PreparePairAsync` failed on *every* switch, so nothing ever rendered:

```
[PreparePair] failed at 5030ms: No reply within 5000ms.
[PreparePair] failed: OpenDIBR rejected command: a camera named "cam1" is already active
```

**Root cause A — timeout too short.** OpenDIBR's `add_camera` connects to the
RTSP stream and warms up its decoder before replying (the log shows seconds of
`not enough frames to estimate rate` / `decoding for stream 0 failed` before
`ready (slot N)`). This routinely exceeded the client's control-channel
timeouts. It is *worse for cam1*, which is a **camera-role instance running on
a separate Windows laptop** — its frames travel laptop → LiveKit → bridge →
RTSP → OpenDIBR, so its warmup is the slowest.

**Root cause B — cache desync.** When `AddCameraAsync` timed out, it threw
*before* `_openDibrAddedOrder` recorded the camera — but OpenDIBR had already
processed the add. The next switch re-sent `add_camera` and OpenDIBR rejected
it with "already active", permanently wedging the pair.

**Fixes (`Master Thesis Client`, uncommitted):**
- `OpenDibrControlChannel.cs`: `AddCameraAsync` timeout `5000 → 20000ms`;
  `SetActivePairAsync` / `RemoveCameraAsync` `3000 → 8000ms`.
- `OpenDibrSessionManager.cs::EnsureOpenDibrCameraAsync`: wrap the
  `AddCameraAsync` call and treat an `"already active"` rejection as success,
  reconciling `_openDibrAddedOrder`, so a mid-sequence timeout can no longer
  wedge a pair.

**Result:** every switch now completes end-to-end. No more timeouts or
rejections; one run even succeeded at `OpenDIBR pair set at 3818ms — ready`,
which the old 3000/5000ms caps would have killed.

```
[PreparePair] OpenDIBR pair set at 1682ms — ready
[PreparePair] OpenDIBR pair set at 3818ms — ready
```

Verified compiling clean via Unity MCP (`read_console` → 0 errors).

---

## 3. Remaining depth bug isolated to two lines (bridge) — HANDED OFF

The view is still grey because the depth, while no longer empty, is still
wrong. The `Active pair set` logs show a hard, reproducible pattern:

```
(cam2, cam1)  A=[0.01,109.81]  B=[0.10,10.00]      ← cam1 empty
(cam1, cam2)  A=[0.10,10.00]   B=[0.01,23.08]      ← cam1 empty
(cam2, cam1)  A=[26.89,107.28] B=[0.10,10.00]      ← cam1 empty
```

Two independent bugs, both in `dibr-bridge/bridge/stereo_depth.py`:

**Bug 1 — right-camera disparity via swapped args returns nothing.** The
`[0.10,10.00]` fallback (`_range_with_margin` when `depth[depth>0].size==0`)
tracks **cam1 in every pair regardless of A/B order** — it is *identity*-bound,
not position-bound. Cause: `compute_pair` gets the right camera's disparity via
`self._matcher_b.compute(gray_b, gray_a)` (swapped image order). StereoSGBM with
`minDisparity=0` only yields valid **positive** disparities for the *left*
image, so feeding the physically-right camera first produces an all-zero map.
Since cam2 is the physical-left camera, cam2 always works and cam1 (right) is
always empty. Fix: use `cv2.ximgproc.createRightMatcher`, or flip-compute-flip.

**Bug 2 — `numDisparities=128` too small.** The magnitudes (`[26–107 m]` for a
marker ~1 m away) are ~100× too large. With `fx_rect ≈ 1200 px`, ~0.3 m
baseline, marker ~1 m away, the true disparity is `1200·0.3/1.0 ≈ 360 px` —
beyond the 128 px search window. SGBM latches onto low-disparity noise (3–14 px)
→ `depth = fx·baseline/disparity` blows up. Fix: raise `numDisparities` to 256+
(multiple of 16); expect the range to collapse toward ~0.5–3 m.

**Ruled out today:**
- *Calibration metric scale.* The printed board is 5×7 cells @ 40 mm (200×280
  mm). `CameraCalibrationData.cs` converts mm→m (`SquareLengthM => mm/1000`)
  and `CameraCalibrator` feeds metres to `estimatePoseCharucoBoard`, so tvec/
  baseline are in metres as the bridge assumes. Server config updated to
  40 mm. A units error would have shown a clean 100× — but that magnitude is
  explained by Bug 2 (disparity clamping), not scale.
- *Scene overlap.* Both cameras are calibrated and both view the ChArUco board,
  so correspondences genuinely exist.
- *The 1536×864 the viewer displays* is the on-screen RawImage panel size
  (1280×720 layout-scaled), not the texture; `FrameExporter` exports 1280×720
  into shared memory as intended.

**Deliverable:** a self-contained handoff (bug diagnosis + evidence + a full
`tests/test_stereo_depth.py` spec) was prepared for the bridge repo session.
The test fixture is a fronto-parallel textured plane at known depth Z viewed by
two x-translated cameras (views differ by a pure shift `d = fx·b/Z`), asserting
(1) both `depth_a` and `depth_b` are populated — catches Bug 1 — and (2) median
recovered depth ≈ Z within 15% — catches Bug 2. Work order: write tests first
(they currently fail), fix Bug 1, fix Bug 2, rerun the live viewer.

---

## Files touched today

**`Master Thesis Client` (uncommitted):**
- `Assets/Scripts/DIBR/OpenDibrControlChannel.cs` — control-channel timeout
  bumps (add 20 s; set-pair/remove 8 s).
- `Assets/Scripts/DIBR/OpenDibrSessionManager.cs` — idempotent
  `add_camera` ("already active" → success + cache reconcile).

**`dibr-bridge` (via the bridge session):**
- `bridge/session.py` — `_resize`, output-resolution publishers, resize before
  publish/compute.
- `bridge/stereo_depth.py` — `out_size` intrinsics scaling in `_prepare`.

---

## State at end of day

- Pipeline runs to completion on every camera switch (was failing before).
- Stereo depth produces real values for one camera per pair (was empty for
  both before).
- Synthetic view still grey — blocked only by the two isolated
  `stereo_depth.py` bugs above, now scoped and handed to the bridge session
  with unit tests.
- Focus moves entirely to the `dibr-bridge` repo until depth is correct.

---

## 4. Bridge-session fixes (same day, later) — DONE

All five issues below were fixed in the `dibr-bridge` repo in a single
follow-up pass, ending with a rebuild and copy to
`Master-Thesis-Client/Assets/StreamingAssets/DibrBridge/`.

### 4a. PyInstaller didn't bundle `livekit_ffi.dll`

Discovered at the very first exe launch attempt. `livekit.rtc._ffi_client`
loads its native DLL via `importlib.resources.files("livekit.rtc.resources")`
— a dynamic import PyInstaller cannot trace statically. The spec had empty
`hiddenimports`, `datas`, `binaries`.

Fix: added `collect_all('livekit')` to `dibr-bridge.spec`, which found 65
data files, 1 binary (`livekit_ffi.dll`), and 47 hidden imports. DLL now
lands at `_internal/livekit/rtc/resources/livekit_ffi.dll` — exactly where
`importlib.resources` looks for it. Smoke-tested: exe reaches an HTTP error
(correct) instead of an `ImportError`.

### 4b. Bridge joined the wrong LiveKit room

`BridgeSession.start()` called `fetch_viewer_token()` with no `room_code`,
so `/viewer-token` received an empty body and the server fell back to its
`ROOM_NAME` env default — not the actual session room (e.g. `FADW88`).

Fix: added `--room-code` (required) to `argparse` in `bridge/main.py`;
threaded through `_run()` → `BridgeSession.__init__(room_code)` →
`fetch_viewer_token(room_code=self._room_code or None)`. No changes needed
to `registration_client.py` — it already sent `roomCode` in the POST body
when provided. Unity's `OpenDibrProcessLauncher.cs` still needs to pass
`--room-code <AppConfig.RoomCode>` — tracked separately.

### 4c. Bug 1 — right-camera disparity (flip-compute-flip)

`self._matcher_b.compute(gray_b, gray_a)` fed the right image first.
`StereoSGBM` with `minDisparity=0` only yields valid positive disparities
when the left image is first — the swapped call always returned all-zero,
giving exactly one empty depth map per pair.

Fix: horizontal-flip trick (no `cv2.ximgproc` — only `opencv-python`
installed):
```python
disp_b_raw = cv2.flip(
    self._sgbm.compute(cv2.flip(gray_b, 1), cv2.flip(gray_a, 1)), 1
)
```
Mirrors the right image so it becomes the effective left input, runs SGBM,
flips result back. Consolidated the two identical SGBM instances into one
`self._sgbm`.

### 4d. Bug 2 — `numDisparities` too small for live geometry

With `fx_rect ≈ 1200`, baseline `≈ 0.3 m`, `Z ≈ 1 m`: true disparity
`≈ 360 px`. First bump to 256 (per the original spec) still didn't cover
it (256 < 360). Bumped to **384** (next multiple of 16 above 360, with
margin). Added INFO logging in `_prepare` (`fx_rect`, `baseline_m`, `size`,
`numDisparities`) and DEBUG per-frame disparity stats (`valid_px`, `min`,
`max`) to `compute_pair`.

### 4e. Unit tests — `tests/test_stereo_depth.py`

Five tests, all green in ~4s. Fixture: fronto-parallel textured plane at
known depth Z, two cameras separated by baseline b along x, zero distortion,
no rotation. Ground truth depth is exact so failures are unambiguous. No
LiveKit/RTSP needed.

| Test | What it pins |
|------|-------------|
| `test_both_cameras_have_valid_depth` | flip-fix: `(depth_b>0).mean()>0.5` |
| `test_metric_depth_accuracy` | both cameras within 15% of true Z |
| `test_wide_baseline_requires_large_num_disparities` | d=180 px: fails nD=128, passes nD=256+ |
| `test_real_geometry_z1m` | d=360 px: fails nD=256, passes nD=384 |
| `test_out_size_intrinsics_scaling` | intrinsics scaling; calib 640×480, out 960×720 |

**SGBM border constraint found during test development:** SGBM marks the
leftmost `numDisparities-1` columns of the left image invalid (the full
search window must fit in the right image). With `nD=384` and `W=640`:
only `(640-384)/640 = 40%` valid — below the 50% threshold even when
disparity is found correctly. Tests use `W=1280` (real camera resolution):
`(1280-384)/1280 = 70%` valid. The first three tests had to be updated from
`W=640` to `W=1280` after this was discovered.

### Files changed (bridge session)

- `dibr-bridge.spec` — `collect_all('livekit')`
- `bridge/main.py` — `--room-code` argument
- `bridge/session.py` — `room_code` threading; `_resize()`; `out_size` pass
- `bridge/stereo_depth.py` — flip-compute-flip; `numDisparities=384`;
  `out_size` + intrinsics scaling; logging; single `self._sgbm`
- `tests/__init__.py`, `tests/test_stereo_depth.py`, `conftest.py` — new

---

## Updated state at end of day

- All five bridge bugs fixed and exe rebuilt.
- 5 unit tests green, including the real-geometry Z=1m case.
- **Next live test**: check whether `Active pair set` logs now show
  real depth ranges (~0.5–3 m) instead of `[0.10,10.00]` fallback.
- **Unity still needs**: `--room-code` added to `OpenDibrProcessLauncher.cs`.
- **Quality note**: anamorphic squish (calibration aspect 2.222 vs output
  1.778) remains — non-blocking for getting depth flowing, but will affect
  synthetic view geometry. Fix later by aligning OpenDIBR output resolution
  to the calibration aspect.
