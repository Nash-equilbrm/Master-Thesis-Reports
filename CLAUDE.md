# Repo — Master's Thesis Workspace

Master's thesis (HCMUS): a **multi-camera live streaming system** with a **live synthetic in-between view on camera switch**. Cameras (phones/laptops) publish over WebRTC via LiveKit; a Unity viewer switches between them, and on Windows+NVIDIA it renders a DIBR (depth-image-based rendering) transition between the two cameras using stereo depth + OpenDIBR.

The workspace root is **not** a git repo — this file is not version-controlled. Each subdirectory is its own repo; commit/push per repo.

> Last full audit: 2026-10-10 (see `Master-Thesis-Reports/action_log_Oct_10th_status_audit.md`). Last updated: 2026-10-17 (FFS-only bridge exe, build naming, two-machine setup; see `Master-Thesis-Reports/action_log_Oct_17th_ffs_exe_build_and_android_fixes.md`). A copy of this file is tracked in `Master-Thesis-Reports/CLAUDE.md` so other machines and cloud sessions get it; keep both in sync.

## Repos

| Dir | Role |
|---|---|
| `Master-Thesis-Client/` | Unity app (Android + Windows). One build, runtime role select: **Camera** (register → calibrate → publish) or **Viewer** (subscribe, switch cameras, drive the DIBR pipeline). |
| `Master-Thesis-Server/` | Docker Compose: LiveKit server + Node `registration-service` (slots, tokens, calibration storage). Deployed to EC2 via GitHub Actions. |
| `dibr-bridge/` | Python. Subscribes to LiveKit, computes live stereo depth for the active camera pair, republishes color+depth as RTSP for OpenDIBR. Runs on the viewer's machine; frozen to `dibr-bridge.exe`. |
| `OpenDIBR/` | Fork of IDLabMedia/open-dibr (C++/CUDA). Patched for RTSP input, external pose (UDP), headless shared-memory frame export, live camera add/remove (TCP). |
| `Fast-FoundationStereo/` | Fork of NVlabs Fast-FoundationStereo, unmodified (tested at `476f424`). Sibling dependency of dibr-bridge's experimental `ffs` depth backend. Weights are downloaded manually into `weights/`. |
| `client-sdk-unity/` | LiveKit Unity SDK, referenced by the client as `file:../../client-sdk-unity`. Has its own `CLAUDE.md`. After changing it, re-resolve packages in Unity. |
| `Master-Thesis-Reports/` | Dated action logs, bug reports, handoff specs. The project's history — read the latest ones for context. |
| `Master-Thesis-Camera-Instance/` | **Superseded** (merged into the client Aug 9). Reference only. |
| `DibrDepthTestCaptures/` | Not a repo. Output of the client's `DibrDepthTestCapture` tool: per switch, one raw frame from each camera + the pair's `calibration.json`. Input for dibr-bridge's offline scripts. |

Older reports use other names: `streaming-server` = `Master-Thesis-Server`, `Master Thesis Client` = `Master-Thesis-Client`, `open-dibr` = `OpenDIBR`, and paths under `C:\Workspace\HCMUS\Master\Repo` (another machine).

## Architecture

```
Streaming
  Client (Camera) --POST /register, /calibration-data--> registration-service
  Client (Camera) --WebRTC publish--> LiveKit --WebRTC subscribe--> Client (Viewer)
  Client (Viewer) --POST /viewer-token--> registration-service

DIBR transition (all local to the viewer machine)
  LiveKit tracks --> dibr-bridge (stereo depth) --RTSP--> mediamtx --RTSP--> OpenDIBR (headless)
  Viewer --TCP 40125--> dibr-bridge      (add_camera / remove_camera / set_active_pair)
  Viewer --TCP 40124--> OpenDIBR         (add_camera / remove_camera / set_active_pair)
  Viewer --UDP 40123--> OpenDIBR         (pose: 7×float32 pos xyz + quat xyzw, ~60 Hz)
  OpenDIBR --shared mem "OpenDIBR_FrameExport" + event "OpenDIBR_FrameReady"--> Viewer
```

The server is only an HTTP dependency of the bridge (`/viewer-token`, `/calibration-data/pair`), never part of the DIBR runtime.

## Master-Thesis-Client

- **App flow:** `Init → RoomEntry → RoleSelect → AppRunning`. Room codes are 6 chars, generated client-side. LiveKit creates rooms on join. `AppRoleManager` activates exactly one of `CameraManagers` or `ViewerManagers`. `Tools > Setup Scene` rebuilds both trees idempotently.
- **Camera states:** `Init → Registering → Connecting → Calibrating → Streaming`. `ConnectionScreen` handles both roles' connect flow and retries.
  - `CalibrationTutorialScreen` is a live ChArUco tutorial: marker → intrinsics → extrinsics (stable pose across frames), then upload. It is skipped when `AppConfig.CalibrationAcknowledged` is set; the **Recalibrate** button clears it. `DevSkipCalibrationUpload` (dev menu) bypasses a failing upload.
  - Calibration frames are row-flipped before ArUco to match what `WebCameraSource` actually publishes. Calibrations made before commit `8994564` (Sep 21) were taken on the un-flipped image and are invalid. K3 and tangential distortion are fixed to 0.
  - The camera source must `Start()` **before** `PublishTrack`; reversing the order deadlocks SDP.
- **Viewer:** `CameraSwitcher` builds its buttons dynamically from `LiveKitManager.VideoTracks`, labelled with the participant name or else the identity.
  - On switch it calls `OpenDibrSessionManager.PreparePairAsync`. If DIBR becomes ready, it waits `_dibrStartupDelay`, fades the synthetic overlay in while streaming a lerped pose A→B, swaps the feed under it, then fades it out. Otherwise, or on timeout, it does an **instant cut**. The old two-player crossfade is gone.
- **DIBR session:** `DibrCapability` gates DIBR on Windows + NVIDIA + `StreamingAssets/OpenDIBR/RealtimeDIBR.exe` present. Android always cuts instantly.
  - `OpenDibrSessionManager` starts eagerly on Viewer role select and launches mediamtx → `dibr-bridge.exe --server-url --rtsp-base --width --height --room-code` → OpenDIBR. Output is fixed at 1280×720 and intrinsics are scaled to it.
  - Per pair, the bridge's `set_active_pair` runs **before** OpenDIBR's `add_camera`, because it computes the depth range.
  - OpenDIBR `add_camera` times out at 20 s, set-pair/remove at 8 s. An "already active" reply counts as success and the cache is reconciled.
- **Bundled binaries:** `Assets/StreamingAssets/{DibrBridge,FFmpeg,MediaMTX,OpenDIBR}/` are **gitignored** (`DibrBridge` is now ~5.8 GB because it bundles torch; the rest ~300 MB). Rebuild with `Tools/BuildStreamingAssets.ps1 [-Only <name>]`. The script must stay UTF-8 *with BOM* (PowerShell 5). `-Only DibrBridge` builds **only the Fast-FoundationStereo exe**: it needs `../dibr-bridge/.venv` with torch, `../Fast-FoundationStereo` (with `core/`, `LICENSE.txt` and `weights/20-30-48/` = `model_best_bp2_serialize.pth` + `cfg.yaml`) and fails loudly if any is missing. `Assets/Editor/AndroidStreamingAssetsFilter.cs` moves every StreamingAssets entry except `build_info` out of `Assets/` during Android/iOS builds and restores them afterwards, so the APK contains no bridge/OpenDIBR/FFmpeg/MediaMTX.
- **Android build:** needs the custom Gradle properties template `Assets/Plugins/Android/gradleTemplate.properties` (`org.gradle.jvmargs=-Xmx8192M`, enabled in Player Settings) or `:launcher:packageRelease` fails with OutOfMemoryError. ARM64 only, IL2CPP. The APK is still ~1 GB, mostly `Assets/OpenCVForUnity` (iOS/WebGL payloads); trimming it is open work.
- `Assets/OpenCVForUnity/` is a paid asset and is gitignored, so it must be imported locally. Its `Plugins/iOS/opencv2.framework` must be limited to iOS in its importer settings (otherwise Android logs "not supported on Android").
- **Offline DIBR tools:** `CalibrationScene` and `DIBRTestScene` (`DIBRSceneManager`, `DepthMeshBuilder`) are the early prototype. `DibrDepthTestCapture` is the capture tool that writes `DibrDepthTestCaptures/`.
- **Automation gotcha:** driving the Editor headless through Unity MCP is flaky (stalled frames, `Start()` not firing, FFI panics). That is an artifact of automation, not a bug.

## dibr-bridge

- **Modules** (`bridge/`):
  - `registration_client.py`: calibration and token HTTP calls.
  - `livekit_source.py`: verified against livekit v1.1.19 source.
  - `stereo_depth.py`: rectify + SGBM via `BaseStereoDepthComputer`.
  - `stereo_depth_ffs.py`: FFS backend (feature branch only, see below).
  - `rtsp_publisher.py`: ffmpeg push.
  - `session.py`: color passthrough, pairwise depth, and a permanent placeholder stream, because OpenDIBR refuses to start with zero cameras.
  - `control_server.py`: TCP 40125.
- **Contract with OpenDIBR:**
  - Depth is **inverse-depth in 8-bit YUV420p** (Y plane). OpenDIBR's decoder can't take 16-bit gray.
  - `add_camera` `Rotation` is Rodrigues (3 floats), while the UDP pose uses a quaternion.
  - Every stream is resized to the fixed output resolution.
- **Stereo correctness fixes in place (Sep 21):**
  - Intrinsics are scaled per camera.
  - Left/right is decided from the sign of `P2[0,3]`.
  - Depth is corrected from the rectified frame back to each camera's own frame.
  - `numDisparities = 384` (true disparity is about 360 px at 1 m).
- **Tests:** `pytest`. 8 SGBM tests plus FFS tests on the feature branch.
- **Calibration check:** `check_stereo_calib.py <capture_dir>` measures median epipolar error from shared ArUco markers. Target ≈ 2 px. Run it before judging depth quality.
- **Build:** the client's `BuildStreamingAssets.ps1 -Only DibrBridge` runs `python -m PyInstaller dibr-bridge-ffs.spec --noconfirm --workpath build-ffs --distpath dist-ffs` (torch + FFS source + hidden imports, `upx=False`) and copies the result, the checkpoint and the FFS license into `StreamingAssets/DibrBridge/`. Never pass `entrypoint.py` to PyInstaller (it overwrites the spec and drops `collect_all('livekit')`). `dibr-bridge.spec` (slim, torch excluded, SGBM only) remains in the repo but is no longer used by the client build. A frozen exe resolves the FFS source from the bundled `ffs_src/` (`FFS_REPO_PATH` overrides) and the checkpoint from `<exe dir>/weights/*/`.
- **Branches:**
  - `main` (`5ba076c`) is SGBM only and stale.
  - `feature/fast-foundationstereo-backend` (pushed through `e38969b`) is the working branch: FFS backend (`--stereo-backend ffs|sgbm|auto`, env `DIBR_STEREO_BACKEND`, `DIBR_FFS_CHECKPOINT`, `FFS_REPO_PATH`), the `297d49c` rectify-crop fix, nearest-neighbour unrectify (`000abb6`), and the FFS packaging (`b886b63`). It has not been merged to `main`; run the FFS test on a GPU first.
  - `auto` picks FFS only when torch, CUDA and the checkpoint are all present, otherwise it silently falls back to SGBM (look for `Stereo backend FALLBACK` in the log). The decision (Oct 17) is that SGBM results are not worth testing; the fallback code is left in.
- **Torch:** the dev venv has `torch 2.7.1+cu128` (matches `requirements-ffs.txt`). cu124 does not run on RTX 50xx (sm_120); cu128 also covers the GTX 1650 (sm_75). xformers is not needed. Run `pip check` after changing it.
- **FFS memory on a 4 GB GPU (GTX 1650 Max-Q):** warm-up peak VRAM is 1.4 GB at 640x360, 2.9 GB at 960x540, 5.2 GB at 1280x720 (spills into system RAM), and fp16 gives NaN there (fp32 is the default). Use an infer resolution of 960x540 or lower on that card. The remote tuning config key is `infer`; checkpoint `20-30-48` has `max_disp` 416.

## Master-Thesis-Server

- **Stack:** `livekit` + `registration` only. Ingress, Redis and FFmpeg/WHIP were retired Aug 18; re-add them only if IP-camera ingest is needed (backup branch `dev/aug_18th_before_removing_ingress`).
- **Local dev:** `start.bat` re-detects the LAN IP and regenerates `.env`/`livekit.yaml`. `LIVEKIT_NODE_IP` must match the current LAN IP, or WebRTC fails while HTTP still works. `ingress.yaml` on disk is a stale, ignored leftover.
- **Endpoints:**
  - `POST /register`: `{roomCode, userId, deviceId}` → `camN` + publish token.
  - `POST /viewer-token`: subscribe token.
  - `POST /unregister`.
  - `POST /webhook`.
  - `GET /calibration-config`.
  - `POST /calibration-data`.
  - `GET /calibration-data/pair?cam1=&cam2=` (404 if either side is missing or stale).
- **Slots:** capacity is `SLOT_COUNT` (default 10). A device gets its previous slot back when it reconnects (`deviceId`). Stale `participant_left` webhooks are ignored via a per-identity generation counter. Don't use LiveKit `sid` for this, because it doesn't exist until after connect.
- **Calibration:** stored in memory per identity. Entries expire after `CALIBRATION_TTL_HOURS` (default 4), but only while the slot is free.
- **ChArUco board:** default `DICT_5X5_250`, 5×7, 40 mm squares, 20 mm markers (printed 200×280 mm). Configurable via `CALIBRATION_*` env vars.
- **EC2:**
  - Elastic IP `13.228.8.204`, with `rtc.node_ip` set to it.
  - Security group: TCP 7880/7881, UDP 7882.
  - Production LiveKit keys live in `docker-compose.override.yml` on the instance.
  - `.github/workflows/deploy.yml` SSH-deploys on push and runs smoke tests.
  - **No TLS** (`ws://`/`http://`), a deliberate choice for a demo. Media is still SRTP-encrypted.
  - `*.pem` and `.sg_id.txt` are gitignored.

## OpenDIBR

- **Build fixes for this machine:** CUDA 13.3 changed `cuCtxCreate` to take 4 args; `NvOptimusEnablement` is exported so GL and CUDA share the RTX GPU; the CUresult `check()` overload is re-enabled.
- **Live-control patches** (`01d32f6`): RTSP input via JSON `NameColor`/`NameDepth`, UDP pose (falls back to keyboard/mouse after 250 ms without packets), headless FBO export to shared memory, and the TCP control channel (acks only after the first decoded frame). `Pool.h` decode-wait has a 2 s timeout.
- `bin\Release\RealtimeDIBR.exe examples\Fan\example_omaf.json` is the reference demo.

## Machines and setup

Work happens on two Windows machines with NVIDIA GPUs. Hard facts:

| Machine | GPU | Notes |
|---|---|---|
| Dev PC (`NASH-EQUI17BRIU`, Windows 10 Home, user `Admin`) | GTX 1650 Max-Q, 4 GB, driver 566.36 | Runs Unity MCP, the local server and the cloud-linked Claude. FFS only up to ~960x540. Not the machine for judging final depth quality. |
| Second PC | RTX 5070 (12 GB) | Where FFS at 1280x720 was measured (README, torch 2.7.1+cu128, checkpoint `23-36-37`). Use it as the viewer for the live test. |

Setting up the 5070 machine (nothing below is in git except the repos):
1. Clone the repos side by side (`Master-Thesis-Client`, `Master-Thesis-Server`, `dibr-bridge` on `feature/fast-foundationstereo-backend`, `Master-Thesis-Reports`, the `Fast-FoundationStereo` fork `Nash-equilbrm/Fast-FoundationStereo`, `OpenDIBR`, `client-sdk-unity`). Import `Assets/OpenCVForUnity` manually (paid, gitignored).
2. Create `dibr-bridge/.venv` (Python 3.10), `pip install -r requirements.txt` then `pip uninstall opencv-python -y` and `pip install -r requirements-ffs.txt` (cu128 torch).
3. Download the FFS checkpoint manually into `Fast-FoundationStereo/weights/<name>/` (`model_best_bp2_serialize.pth` + `cfg.yaml`; not in git, NVIDIA non-commercial license). The Dev PC has `20-30-48`.
4. Build the OpenDIBR, FFmpeg and MediaMTX StreamingAssets with `Tools/BuildStreamingAssets.ps1`, then `-Only DibrBridge -Force` for the FFS exe (about 6 minutes, 5.8 GB).
5. Server: `Master-Thesis-Server\start.bat` (Docker Desktop must be running); it detects the LAN IP. All devices must be on that subnet.

## Builds

- Output folder per build: `D:\WorkSpace\Master\Thesis\Builds\_<MM>.<NNN>.<yyyyMMdd-HHmm>\` (MM = month, NNN = 3-digit index per month starting at 001, time = local start time). Android file: `MT_Client_<version>.apk`; Windows player: `MT_Client_<version>_Windows\`. Never overwrite an earlier build. Older builds live in `Builds\_9.2\`.
- Windows player build was attempted once on Oct 17 and failed (6 errors, not investigated). Testing is done in the Unity editor with the platform set to Windows.
- The orchestrator workflow (cloud Claude + one agent per repo + a Windows "Thesis shell" for commands) is documented in the Claude Project "Thesis", doc `thesis-working-flow.md`.

## Current status (2026-10-17)

**Working:**
- Room/role/connect flow, multi-camera streaming, calibration tutorial/upload/storage, and a DIBR pipeline that starts and completes every switch.
- FFS packaged: exe frozen OK (5.8 GB, backend `ffs` on cuda, no import errors) and the model loads inside it. Local server up. Android APK builds.

**Not yet verified:** FFS inference running inside the frozen exe; the synthetic view being visually correct live.

**Open, in priority order:**
1. **Live test with FFS on the 5070 machine.** Recalibrate both cameras first (cam2 was flipped after calibration; calibrations before `8994564` are invalid), capture a pair with `DibrDepthTestCapture`, run `check_stereo_calib.py` (aim for ~2 px), then judge the synthetic view. Check the bridge log says `Stereo backend: ffs` and not `FALLBACK`, and that depth ranges are real, not `[0.10, 10.00]`.
2. **No hard-coded IP:** the client's `AppConfig.DefaultServerUrl` is the dead EC2 address (`http://13.228.8.204:3000`) and the `DevServerUrlMenu` editor item hard-codes `192.168.1.3:3000`. Proposed: empty default plus a first-run prompt; server derives `livekit_url` from the request host. Not decided or done.
3. Lower the FFS infer resolution for the 1650 (960x540) and record FFS latency against the ~66 ms/tick budget (still unrecorded).
4. Windows player build errors; trim the ~1 GB APK (OpenCVForUnity payloads); merge the bridge FFS branch to `main` after a GPU test.
5. Aspect mismatch (calibration ≈2.22 vs output 1.78); bridge hardening (depth range from percentiles over several frames, RTSP publisher blocks the event loop, pair switch cancels depth before fetching calibration, raw traceback when the server is unreachable, no track reconnect); server (calibration keyed by slot not device, unvalidated uploads, unpinned `livekit-server:latest`, webhook slot-freeing never live-tested).

**Future work, not scheduled:** Camera role on Meta Quest (`Master-Thesis-Reports/mixed-reality-migration-plan.md`), IP-camera ingest via LiveKit Ingress.

## Conventions

- Reports go in `Master-Thesis-Reports/` as `action_log_<Mon>_<Nth>[_topic].md`, with bug reports and handoff specs alongside.
- Commit message style: `[feat]: …`, `[fix]: …`, `[chore]: …`, `[report]: …`.
- `Master-Thesis-Client/report_2026-05-11_*.md` are stale, pre-merge snapshots. Prefer the code and this file.
