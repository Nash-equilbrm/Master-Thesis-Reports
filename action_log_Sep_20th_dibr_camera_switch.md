# Action Log — Sep 20th (live DIBR camera-switch: plan, three-repo build, packaging)

## Session summary

Designed and implemented a live synthetic-view camera-switch feature for the
Viewer: instead of a hard cut, clicking a different camera crossfades
instantly, then swaps to an OpenDIBR-rendered in-between viewpoint (lerped
from camera A's calibrated pose to camera B's) if it becomes ready in time,
settling on a plain camera feed either way. Spans three components across
two sessions working in parallel — this one (`Master-Thesis-Client` +
`Master-Thesis-Server` + a new `dibr-bridge` component) and a second Claude
session (`opendibr-c3`) patching `OpenDIBR` directly, coordinated via
cross-session messages throughout. Ends with all three components
feature-complete, mutually consistent, and packaged into a self-contained
Windows build — not yet exercised as a live end-to-end run (needs real
camera devices).

Full plan: `C:\Users\Admin\.claude\plans\tranquil-seeking-clarke.md`.

---

## Architecture decided

Considered and rejected a lighter-weight "Unity-native DIBR" approach
(extending an existing offline prototype, `DIBRSceneManager`/
`DepthMeshBuilder`) in favor of driving the real `OpenDIBR` C++/CUDA
renderer live, per explicit user request for that renderer's actual
occlusion-aware quality. Resulting pipeline, all local to the Viewer's own
machine (OpenDIBR needs a local NVIDIA GPU):

```
LiveKit tracks --> dibr-bridge (live stereo depth, OpenCV) --RTSP--> mediamtx --RTSP--> OpenDIBR (headless)
                                                                                              |
Unity Viewer <--shared memory (rendered frame)-- OpenDIBR <--UDP (pose) / TCP (control)-- Unity Viewer
```

`Master-Thesis-Server` is only ever an HTTP dependency the bridge calls
(`/viewer-token`, `/calibration-data/pair`) — same relationship the Unity
app itself has, never part of the pipeline's runtime.

---

## Cross-session coordination with `opendibr-c3`

Wrote a self-contained handoff spec
(`handoff_spec_Sep_20th_opendibr_live_control_and_export.md`) scoping four
OpenDIBR-side items (RTSP input, external pose channel, headless live frame
export, dynamic camera add/remove + active-pair control channel), then
exchanged several rounds of messages as implementation on both sides
surfaced real contract gaps — each one found by actually reading code or
running something, not assumed:

- Depth is pairing-dependent (stereo scale depends on the current partner)
  but OpenDIBR's `add_camera` needs a `Depth_range` up front — resolved by
  reordering so the bridge's `set_active_pair` (which computes it) always
  runs before OpenDIBR's `add_camera`.
- OpenDIBR's depth decode is inverse-depth (disparity-like), not linear —
  caught by `opendibr-c3` reading the actual shader
  (`src/vertex.fs`), not docs.
- OpenDIBR's H.264/HEVC decoder only recognizes 8/10/12-bit YUV420p — a
  16-bit grayscale depth stream (the original implementation) would have
  silently corrupted; switched to 8-bit YUV420p with proper inverse-depth
  encoding.
- `add_camera`'s `Rotation` field is 3 floats (Rodrigues/axis-angle, same
  convention as calibration's own `rvec`) — not a quaternion, which was the
  original (wrong) assumption. Item 2's live pose channel is still a
  quaternion — separate field, unaffected.
- OpenDIBR refuses to launch with zero cameras (a hard startup check,
  confirmed by `opendibr-c3` actually launching it, not from memory) —
  resolved by having the bridge run a permanent placeholder color+depth
  stream rather than asking for a code change to an already-verified system.

`opendibr-c3` also found and fixed a real concurrency bug unprompted while
verifying Item 4 (`Pool.h`'s decode-wait had no timeout; reusing a slot
right after `add_camera` could hang the whole render thread — fixed with a
bounded 2s timeout).

All four OpenDIBR items are done and verified end-to-end on that side
(real RTSP streams, real add/remove/set_active_pair cycles, confirmed the
exported frame actually changes between cameras, a 5-cycle add/remove
stress pass with no crash or memory growth).

---

## Client-side (`Master-Thesis-Client`)

New: `CalibrationPairClient.cs`, `DibrCapability.cs` (Windows+NVIDIA-GPU+
bundled-binary gate — this feature is a permanent Windows-desktop-only
limitation, Android always falls back), and `Assets/Scripts/Dibr/`
(`OpenDibrPoseChannel`, `JsonLineTcpClient` + `OpenDibrControlChannel` +
`DibrBridgeControlChannel`, `OpenDibrFrameReceiver` — plain
`MemoryMappedFile`/`EventWaitHandle`, no native plugin needed,
`OpenDibrPoseMath`, `OpenDibrProcessLauncher`, `OpenDibrStartupJson`,
`OpenDibrSessionManager`). Modified: `CameraSwitcher.cs` rewired for the
fallback-first UX (crossfade starts instantly on click; DIBR swaps in only
if it becomes ready before the crossfade finishes). Scene: added
`CalibrationPairClient` + `OpenDibrSessionManager` to `Main.unity` under
`ViewerManagers`.

Session start moved from "lazy, on first camera switch" to "eager, on
Viewer role selection" per user request, to reduce first-switch cold-start
latency — `OpenDibrSessionManager.Awake()` fires it directly, no new event
wiring needed since that GameObject already lives under `AppRoleManager`'s
`ViewerManagers` container, which only activates when `AppRole.Viewer` is
chosen.

Real bugs caught and fixed during implementation, not just written once and
trusted: a `CalibrationPairClient` cache keyed by sorted pair order would
have silently swapped which camera's data was which on a reverse-order
lookup (fixed before it shipped); `OpenDibrSessionManager` was launching the
bridge with no CLI arguments at all, even though `--server-url` is required
— would have crashed on every single launch.

---

## Bridge (`dibr-bridge` — new component)

Python (`livekit`, `opencv-python`, `numpy`, `requests`), async, structured
as: `registration_client.py` (calibration/token HTTP), `livekit_source.py`
(shared room connection, per-camera frame access), `stereo_depth.py`
(`cv2.stereoRectify` + `StereoSGBM`, with an un-rectification step to keep
depth aligned to each camera's own raw pixel grid rather than the rectified
one — a correctness subtlety worked out during implementation, not in the
original plan), `rtsp_publisher.py` (ffmpeg subprocess push), `session.py`
(per-camera color passthrough + pairwise depth pipeline + the permanent
placeholder stream), `control_server.py` (TCP control channel, port 40125).

**Location corrected mid-session**: originally placed inside
`Master-Thesis-Server` on the reasoning that it matched that repo's existing
FFmpeg webcam-to-WHIP bridge — user caught that this bridge, unlike that
one, never runs on or is deployed with the server at all (it's local-only,
tied to wherever OpenDIBR's GPU is). Moved to `dibr-bridge/`, a workspace-root
sibling directory alongside `Master-Thesis-Client`/`Master-Thesis-Server`/
`OpenDIBR`/`client-sdk-unity`.

**Verified against the real installed SDK**, not just docs: read
`livekit`'s actual installed source directly (v1.1.19) to confirm
`VideoStream` yields a `VideoFrameEvent` dataclass with a `.frame` field —
confirmed the existing code was already correct, removed an unneeded
defensive fallback that had been written under uncertainty.

**Packaged for full portability** — the original goal was a Windows build
needing zero manual installs beyond the NVIDIA driver itself:
- Frozen into a standalone `dibr-bridge.exe` via PyInstaller (`--onedir`,
  through a new `entrypoint.py` — `bridge/main.py` can't be the PyInstaller
  target directly, its relative imports break without `bridge` loaded as a
  real package). Smoke-tested: native deps (livekit/cv2/numpy) load
  correctly, and a real launch against an unreachable server reaches an
  actual HTTP call before failing.
- `ffmpeg` (Gyan.FFmpeg full_build) and `mediamtx` (bluenviron.mediamtx)
  installed via `winget`, then bundled — `OpenDibrProcessLauncher` now
  launches mediamtx as a third child process (before the bridge, which
  needs it already listening) and injects the bundled FFmpeg directory into
  the bridge's own `PATH`.
- OpenDIBR's existing Debug build copied in too.
- Everything lands in `Master-Thesis-Client/Assets/StreamingAssets/`:
  `DibrBridge/` (~158MB), `FFmpeg/` (~212MB), `MediaMTX/` (~54MB),
  `OpenDIBR/` (~43MB) — ~467MB total, acceptable for a thesis build with no
  app-store constraints.

---

## Files changed / added

**`Master-Thesis-Client`**: `Assets/Scripts/Manager/CalibrationPairClient.cs`,
`Assets/Scripts/Manager/DibrCapability.cs`, `Assets/Scripts/Dibr/*.cs` (9
new files), `Assets/Scripts/Stream/CameraSwitcher.cs` (rewritten),
`Assets/Scenes/Main.unity`, `Assets/StreamingAssets/{DibrBridge,FFmpeg,
MediaMTX,OpenDIBR}/` (bundled binaries, gitignore status not yet decided —
these are large binary payloads, likely want the same gitignored-but-bundled
treatment as `Assets/OpenCVForUnity/`).

**`dibr-bridge`** (new, workspace-root sibling repo/directory, not yet a git
repo itself): full Python package, `README.md`, `requirements.txt`,
`entrypoint.py`.

**`Master-Thesis-Reports`**: this file;
`handoff_spec_Sep_20th_opendibr_live_control_and_export.md` (written earlier
this session, updated across the coordination rounds above).

---

## Next steps

1. Run the actual end-to-end test: registration-service + LiveKit already on
   EC2, 2+ real camera devices registered/calibrated/streaming, 1 Viewer —
   click between cameras and see what breaks. None of the three components
   have run together yet, only individually verified.
2. Likely first-run risks, in rough likelihood order: the fixed startup-
   timing heuristics in `OpenDibrSessionManager` (3s for the bridge's
   placeholder stream, 2s for OpenDIBR's own startup check) being too
   short; something in the generated startup JSON not matching exactly;
   the pose/coordinate math not looking visually correct on the first try.
3. Decide whether `Assets/StreamingAssets/{DibrBridge,FFmpeg,MediaMTX,
   OpenDIBR}/` should be committed to `Master-Thesis-Client` or gitignored
   like `Assets/OpenCVForUnity/` (recommended, given their size and that
   they're rebuildable/re-copyable from their own sources).
4. Decide whether `dibr-bridge/` should become its own git repo (matching
   `OpenDIBR`/`client-sdk-unity`) — not done yet, left as a plain directory.
