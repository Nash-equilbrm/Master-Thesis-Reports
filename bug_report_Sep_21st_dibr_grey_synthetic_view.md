# Bug Report — Sep 21st (live DIBR camera-switch renders a grey synthetic view)

## Symptom

On the Viewer, switching cameras plays the intended crossfade UX — but the
"synthetic view" that fades in over the transition is a **flat grey frame**,
not an OpenDIBR-rendered in-between viewpoint. The user sees: current feed →
grey fade-in → grey fade-out → next camera feed. No synthetic geometry is
ever visible.

This is the first live end-to-end run of the three-component pipeline
(`Master-Thesis-Client` + `dibr-bridge` + `OpenDIBR`) built on Sep 20th,
which was packaged but never exercised together (see
`action_log_Sep_20th_dibr_camera_switch.md`, "Next steps"). Two of the
predicted first-run risks are exactly what surfaced.

---

## The Unity client side is working correctly

The console shows the entire switch sequence completing as designed — the
overlay is shown and faded, the feed is swapped:

```
[PreparePair] cam1→cam2 start
[PreparePair] OpenDIBR pair set at 2450ms — ready
[CameraSwitcher] DIBR ready for cam1→cam2 at 2.48s — waiting 1.5s for pipeline
[CameraSwitcher] DIBR opaque — swapping feed to cam2
```

So the grey is not a UX/sequencing bug. `CameraSwitcher` is faithfully
displaying whatever `OpenDibrFrameReceiver.Texture` contains, and
`OpenDibrSessionManager.PreparePairAsync` is reporting the pair as ready.
The problem is upstream: **what OpenDIBR exports into shared memory is
blank**, because it has no valid depth geometry to reproject.

---

## Root cause #1 (primary): stereo depth is empty on every pair

This log line appears for **every** `set_active_pair`, in every direction,
without exception:

```
[Bridge stderr] ... Active pair set to (cam1, cam2); depth range A=[0.10,10.00] B=[0.10,10.00]
[Bridge stderr] ... Active pair set to (cam2, cam1); depth range A=[0.10,10.00] B=[0.10,10.00]
```

`[0.10, 10.00]` is **not a computed range** — it is the hardcoded fallback
in `dibr-bridge/bridge/session.py:201-209`, returned only when the depth map
has zero valid pixels:

```python
@staticmethod
def _range_with_margin(depth: np.ndarray) -> tuple[float, float]:
    valid = depth[depth > 0]
    if valid.size == 0:
        return 0.1, 10.0  # fallback — no valid disparity this frame
    ...
```

Real disparity data would produce arbitrary floats, never exactly
`(0.1, 10.0)`. So `StereoDepthComputer.compute_pair()`
(`dibr-bridge/bridge/stereo_depth.py:107`) is returning an **all-zero depth
map** every single time. OpenDIBR reprojects the color stream per-pixel using
this depth; with no valid depth there is no geometry to draw, so it exports a
blank/grey frame → grey overlay in Unity.

This matches `stereo_depth.py`'s own header comment: *"UNVERIFIED … this
whole module has not been run against real frames. Visually check
depth-vs-color alignment before trusting it."* It has now been run, and it
produces nothing.

### Likely causes inside `StereoDepthComputer`, to investigate in order

1. **Resolution mismatch between calibration and live frames.**
   `self._size` is taken from `calib_a.intrinsics.image_width/height`
   (`stereo_depth.py:45`), and the rectify maps
   (`initUndistortRectifyMap`, lines 71-72) are built at that size. But the
   live frames arriving from LiveKit are at the camera's native capture
   resolution — see the bridge log: `cam1 (1280x720)`, `cam2 (1200x540)`. If
   calibration was captured at a different resolution than the live feed,
   `cv2.remap` samples the frame with maps built for the wrong grid → garbage
   rectification → SGBM finds no matches → zero disparity.
2. **The two cameras have different resolutions** (1280×720 vs 1200×540)
   while a single `stereoRectify(..., self._size, ...)` is used for both
   (line 66). The A/B rectified images must share a size and epipolar
   geometry for SGBM to correlate them; feeding differently-sized frames
   breaks that.
3. **Disparity search range** (`numDisparities=128`, line 85) may not cover
   the actual baseline/disparity for this rig, though this is secondary to
   (1) and (2).

Note the depth map is computed from a **single snapshot** of each camera at
`set_active_pair` time (`session.py:158-160`), then re-pumped — so if that
first frame yields no disparity, the whole active pair renders grey.

---

## Root cause #2 (contributing): OpenDIBR ↔ bridge resolution contract mismatch

The bridge publishes each camera's RTSP stream at that camera's **native
resolution** (`session.py:110-127`, using `frame.shape`):

```
[Bridge stderr] ... Added camera 'cam1' (1280x720) -> rtsp://.../cam1_color / .../cam1_depth
[Bridge stderr] ... Added camera 'cam2' (1200x540) -> rtsp://.../cam2_color / .../cam2_depth
```

But `OpenDibrSessionManager.EnsureOpenDibrCameraAsync`
(`Master Thesis Client/Assets/Scripts/DIBR/OpenDibrSessionManager.cs:362-370`)
always declares `resolution: new[] { _outputWidth, _outputHeight }` =
**1280×720** to OpenDIBR's `add_camera`, for *both* cameras. OpenDIBR fixes
its decode/render resolution at process launch (documented in the same file,
lines 46-57), so cam2's 1200×540 stream does not match the declared 1280×720.

Consistent with this, OpenDIBR's stderr repeatedly reports:

```
[OpenDIBR stderr] [rtsp @ ...] Stream #0: not enough frames to estimate rate; consider increasing probesize
[OpenDIBR stderr] [rtsp @ ...] decoding for stream 0 failed
```

Some of these are the normal "just-connected RTSP stream" warning, but a
persistent decode failure on the mis-sized cam2 stream is expected here.
Even if depth (#1) is fixed, cam2 will keep failing to decode until the
resolution contract is made consistent — either the bridge scales/pads every
frame to the fixed 1280×720 pipeline resolution before publishing, or the
Unity side sends each camera's true resolution to OpenDIBR (only viable if
OpenDIBR actually supports per-camera resolutions; the current comments say
it does not).

The intrinsics side of this is *already* partly handled — `ScaleIntrinsics`
(`OpenDibrSessionManager.cs:314-330`) scales fx/fy/cx/cy from the calibration
resolution to 1280×720 — but the actual **pixel streams** are not scaled to
match, which is the gap.

---

## Evidence trail (console, this session)

- Session bootstrap is clean: mediamtx → bridge → placeholder stream →
  OpenDIBR launch, no errors. RTX 5070 detected, `FrameExporter` active on
  shared memory `OpenDIBR_FrameExport` / event `OpenDIBR_FrameReady`.
- Calibration fetch succeeds (`found=True`) for every pair — so this is
  **not** a missing-calibration problem.
- `add_camera "cam1": ready (slot 1)` / `add_camera "cam2": ready (slot 2)`
  — OpenDIBR accepts both cameras despite the resolution mismatch (it does
  not reject, contrary to the earlier assumption in `OpenDibrSessionManager.cs`
  comments — worth noting).
- Every `Active pair set` logs the `[0.10,10.00]` fallback range (root
  cause #1).
- Repeated `decoding for stream 0 failed` (root cause #2).

---

## Recommended fix path

1. **Instrument `StereoDepthComputer.compute_pair` first** (the code is
   explicitly unverified). Log the input frame shapes vs `self._size`, the
   raw disparity valid-pixel count and min/max, and dump one rectified pair +
   one depth map to PNG. This confirms *why* disparity is empty before
   changing rectification logic.
2. **Fix the resolution handling** — the most likely single cause. Ensure the
   frames fed to `compute_pair`, the calibration `image_width/height` driving
   `self._size`, and the rectify maps all agree; resize live frames to the
   calibration resolution (or vice-versa) before rectifying.
3. **Fix the OpenDIBR resolution contract** (#2): make the bridge scale/pad
   every published color+depth frame to the fixed 1280×720 pipeline
   resolution in `session.py` / `rtsp_publisher.py`, so what OpenDIBR decodes
   matches what `OpenDibrSessionManager` declares.
4. Re-run and confirm: `Active pair set` shows a **real** (non-`0.10/10.00`)
   depth range, `decoding for stream 0 failed` stops recurring for cam2, and
   the overlay shows actual reprojected content instead of grey.

---

## Files implicated

- `dibr-bridge/bridge/stereo_depth.py` — empty-depth root cause (#1),
  resolution/rectification handling.
- `dibr-bridge/bridge/session.py` — fallback range at `:201-209`; per-camera
  native-resolution publishing at `:104-128`.
- `dibr-bridge/bridge/rtsp_publisher.py` — where per-frame scaling to the
  fixed pipeline resolution would go (#2).
- `Master Thesis Client/Assets/Scripts/DIBR/OpenDibrSessionManager.cs` —
  `:362-370` declares 1280×720 to OpenDIBR; `:314-330` `ScaleIntrinsics`
  (intrinsics scaled, pixels not).
- `Master Thesis Client/Assets/Scripts/Stream/CameraSwitcher.cs` — confirmed
  working; renders whatever the receiver provides.
```
