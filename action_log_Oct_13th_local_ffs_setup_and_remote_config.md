# Action Log — Oct 12th–13th (switch to local testing, FFS as the stereo backend, remote tuning config, mobile build exclusion)

## Session summary

Coordination session run from the workspace root (`repo-ab`), driving the per-repo Claude sessions (`streaming-server`, `dibr-bridge`, `Master Thesis Client`) through read-only status checks and then scoped changes. Nothing in this log was run end to end with real cameras: the physical test (recalibrate, capture a pair, check epipolar error, click between cameras) is still pending and is the next step.

Direction set by the user during the session:
- The remote EC2 server is deactivated, so all testing moves to the **local** stack.
- **Fast-FoundationStereo (FFS) replaces SGBM** as the intended stereo matcher. SGBM stays in the code as the fallback.
- End goal: a Windows user with an NVIDIA GPU downloads the app and runs it with **no extra installation**.
- The **Camera role** needs only calibration + streaming: no FFS, no OpenDIBR, no bridge. Android and iOS builds must not pack those.
- Stereo tuning numbers (max_disp etc.) must be changeable **without a rebuild**, via a JSON file hosted on the internet.

Root `CLAUDE.md` (not under git) was fully refreshed from all reports and from the sessions' answers; it carries the long-form version of everything below.

---

## 1. State of the reports vs. reality (what the earlier CLAUDE.md got wrong)

Reading every report in this folder plus asking each repo session showed the status section was stale:
- Camera slot drift (`deviceId` sticky slots, generation counter) was fixed Sep 11 (`44988e0` server, `29c2731` client).
- Server calibration endpoints (`a73800b`) and per-device TTL (`f0f8e7d`) exist; board default is now 5×7 at 40 mm squares / 20 mm markers (`6974945`).
- EC2 network posture was verified Sep 12: correct, no TLS, accepted.
- Unity already passes `--room-code` (built in `OpenDibrSessionManager`, not in the launcher). Control-channel timeouts and the CameraSwitcher white-screen fixes are committed and pushed (`e970334`, `e58d5b0`, `371f981`).
- `dibr-bridge` work after Sep 21 had **no report**: an FFS branch with six commits (Sep 26–Oct 10), including `stereoRectify` alpha 0 → -1, which raised valid-depth coverage from ~1% to 8–9% (SGBM) and ~13% to 65–77% (FFS).
- The Sep 21 blocker stands: the synthetic view is grey because cam2 was physically flipped after calibration (~333 px median epipolar error; 180° software rotation only reaches ~91 px; target ~2 px). No fresh capture or `check_stereo_calib.py` result exists since.

## 2. Local server

`streaming-server` session brought the stack up on LAN IP 192.168.1.3. Docker Desktop was not running; once started, `start.bat` regenerated `.env`/`livekit.yaml` (old value 192.168.1.6 was stale). A livekit container that had auto-started on the old yaml needed `docker compose restart livekit`. Verified: `/calibration-config` returns the 5×7 / 40 / 20 mm board, `/viewer-token` returns `ws://192.168.1.3:7880`, ports 3000/7880/7881/7882 are listened on by Docker with inbound allow rules on the Public profile (not yet probed from a second device). Registration state (slots, `deviceSlots`, calibration) is in memory only, so any registration-container recreate wipes calibration and every camera must recalibrate.

The client default `AppConfig` server URL is still the dead EC2 address. Every device must override it to `http://192.168.1.3:3000`. New Editor menu `Tools > Dev Server > Use Local` (client `4c100ed`) writes the `dev_server_url` PlayerPrefs key.

## 3. FFS on this machine (RTX 5070, driver 591.86, sm_120)

Done by the `dibr-bridge` session on branch `feature/fast-foundationstereo-backend`:
- `.venv` (Python 3.13.7): torch 2.7.1+cu128 / torchvision 0.22.1 (the pinned 2.6.0+cu124 has no Blackwell kernels), `opencv-contrib-python` in place of `opencv-python`, xformers dropped (FFS core never imports it).
- Fast-FoundationStereo cloned as a sibling at `476f424`; checkpoint `weights/23-36-37/model_best_bp2_serialize.pth` (71 MB) placed and verified to load; 15 → later 59 tests pass.
- Code: model cached across pair switches, warmed up at startup; compute moved to `asyncio.to_thread` (previously blocked the event loop and stalled colour pumping even with SGBM); backend `auto|sgbm|ffs` with default `auto` (FFS if checkpoint + torch + CUDA work, otherwise a loud fallback to SGBM); checkpoint auto-discovery.
- Measured at 1280×720, 23-36-37, 4 iterations, fp32 (offline benchmark on the upstream demo pair, not live): ~280 ms per inference, ~590 ms per `compute_pair`, ~3 s warm-up, **~7.66 GB peak allocated VRAM** on a 12 GB card shared with OpenDIBR and Unity. fp16 is ~15% faster with disparity MAE 0.008 px but saved only ~0.1 GB. Depth refresh is only ~2 Hz vs the 15 Hz target.
- Upstream's own table shows ~650 MB peak at 640×480 for all three research checkpoints, so other variants differ in speed, not memory (inference, not verified here).
- Disparity scale: this rig (~0.3 m baseline, fx ~1200 px, subject ~1 m) has ~360 px disparities at full resolution; FFS was run with max_disp 192. The checkpoint supports up to 416, so `max_disp` 384 is possible; halving the inference resolution (640×360) halves the shifts and cuts memory. Neither was measured on real cameras. The **memory sweep was not run** (see open items).

## 4. Remote tuning config (no rebuild)

Bridge commits `d1f1496`, `f156c23`, `327d638`, `1651180`:
- JSON `{"version":1,"stereo":{backend, max_disp, valid_iters, infer_width, infer_height, fp16}}`; strict validation (ranges, multiples, enums), unknown keys dropped, paths/commands/URLs can never be set. `max_disp` must be ≤ the checkpoint's trained value (416; 448 is rejected).
- Precedence: CLI > env > config file (local path > URL > last-good cache in `%LOCALAPPDATA%\dibr-bridge\config_cache.json`) > built-in defaults. Overrides: `--stereo-override KEY=VALUE` / `DIBR_STEREO_OVERRIDES`.
- Fetch in a worker thread, 2 s timeout, ETag/304, 64 KB cap, https only (http for localhost). Applies on the next `set_active_pair`, with build-before-swap rollback to the last working settings if the new settings fail (e.g. OOM).
- Depth upsampling uses nearest-neighbour (metric Z with 0 = invalid); inference size is in multiples of 8.
- Hosted as a public gist: `https://gist.githubusercontent.com/Nash-equilbrm/e5d57617062f0dd1222c8bea30ed7163/raw/dibr-config.json`, now the **built-in default URL** (`DEFAULT_CONFIG_URL` in `bridge/remote_config.py`). Opt-out: `--config-url off` or `DIBR_CONFIG_URL=off`. Loading was verified through the bridge's own `ConfigProvider` (fetch, validate, cache write, 304, offline fallback), no GPU and no full bridge launch.
- Trade-offs recorded in the bridge README: anyone controlling the gist account can change behaviour of every installed copy (values only, within ranges); every bridge start and pair switch contacts GitHub; for reproducible thesis numbers pin a gist revision or use `--config-path`.
- Unity (client `d4ba975`): optional empty-by-default `OpenDibrSessionManager` fields `_bridgeConfigUrl`, `_bridgeConfigPath`, `_bridgeStereoOverrides`; the launcher no longer passes `--stereo-backend` (bridge `auto` decides) and still passes `--ffs-checkpoint` when a `.pth` exists under `StreamingAssets/DibrBridge/weights`.

## 5. Client changes today

Pushed to `origin/main`: `2de20c9` (dev launcher override: `_bridgeArgPrefix`, `_bridgeWorkingDirOverride`, so the bridge can run from `dibr-bridge\.venv` as `python.exe -m bridge.main`), `4c100ed` (Dev Server menu), `570babb`, `d4ba975`. `8994564` (pulled earlier) contains the calibration flip fix (`Graphics.Blit` with a (1,-1) scale in the capture path, constrained K3/tangential terms) and `DibrDepthTestCapture` (writes `<camA>.png`, `<camB>.png`, `calibration.json` to `DibrDepthTestCaptures/<camA>_<camB>_<timestamp>/` on every switch). **Calibrations made before `8994564` are stale and must be redone.**

Timing mismatch found: `CameraSwitcher` waits up to 8 s (`_dibrPrepareWaitMax`) for a pair, then does an instant cut; the bridge timeout is 15 s. With the model cached and warmed (~0.85 s per switch expected after warm-up, ≤ ~5 s if a switch arrives before warm-up ends) this should fit, but it was not run live.

### Mobile builds
`AndroidStreamingAssetsFilter.cs` (`570babb`) is now deny-by-default for **Android and iOS**: everything under `StreamingAssets` except `build_info` is moved to `Library/MobileExcludedStreamingAssets` for the build and restored afterwards (post-build, on the next pre-build, and via an editor-update check, since post-build does not run when a build fails). `.meta` files move with their folders. `DibrDepthTestCapture` returns early on mobile; `DibrCapability` is already false on mobile. Effect: ~480 MB now, ~5.5 GB once the torch bundle exists, are kept out of mobile builds. **Not verified with a real build.** Windows builds still include the whole DIBR/FFS payload even for camera-only users; a camera-only Windows variant looks easy (scripting define or build profile + the same filter) but is not implemented.

## 6. Packaging and licensing for the zero-install goal (investigation only, nothing built)

- torch in onedir is ~4.6–5.4 GB of DLLs; the DibrBridge folder would be ~5 GB, total StreamingAssets ~5.3 GB, estimated 2.5–3 GB zipped (ratio is an estimate). Onefile is not viable (it would extract ~5 GB on every launch). Needs `collect_all('torch')`/`timm`, FFS `core/` and `Utils.py` bundled with a frozen-aware `FFS_REPO_PATH`, and the checkpoint under `DibrBridge/weights/`.
- Smaller alternatives to prototype later (untested): ONNX export via the FFS repo's `scripts/make_single_onnx.py` (fixed resolution, so 1280×736 padding) → onnxruntime-directml (~20 MB, no CUDA DLLs), or TensorRT / TensorRT-for-RTX.
- **Licence (a reading of the cloned repo's `LICENSE.txt`, `model_card.md` and `readme.md`, not legal advice):** the FFS code and the Google Drive "research checkpoints" are under the NVIDIA Source Code License (non-commercial / research only; `LICENSE.txt` must ship with any redistribution). The NVIDIA Open Model Agreement covers only the separate commercial C-Fast-FoundationStereo on Hugging Face. Bundling the research checkpoint in a non-commercial thesis app looks permitted; commercial distribution would need C-FFS and its terms, which were not read. The bridge README and comments that wrongly said the Drive weights are under the Open Model Agreement were corrected (`0585b97`).

## 7. Open items

1. **Physical test (user):** fix both cameras in place, recalibrate with the 40 mm board using the new client, capture a fresh pair via `DibrDepthTestCapture`, run `dibr-bridge/check_stereo_calib.py` (target ~2 px median), then click between cameras in the viewer. For a venv launch set `_bridgeExePathOverride` = `dibr-bridge\.venv\Scripts\python.exe`, `_bridgeArgPrefix` = `-m bridge.main`, `_bridgeWorkingDirOverride` = the `dibr-bridge` folder. Every device needs the server URL `http://192.168.1.3:3000`.
2. **VRAM sweep not run.** The `dibr-bridge` session's permission check blocked the benchmark runs (even a one-line typo fix, `reset_peak_stats` → `reset_peak_memory_stats`, in `C:\Users\Admin\AppData\Local\Temp\ffs_bench\ffs_vram.py`). Planned configs: 1280×720 at max_disp 192 / 384 / 128 / 96, 2 vs 4 iterations, 640×360, and `empty_cache` / `expandable_segments`. Its provisional, **unmeasured** pick for a 12 GB card: infer 640×360, fp16, 4 iterations, max_disp 192 (equivalent to 384 px at full resolution).
3. Real mobile build to confirm the StreamingAssets exclusion and restore, and to confirm the DIBR C# code compiles under IL2CPP.
4. Windows camera-only variant decision; FFS packaging (torch onedir vs ONNX/DirectML) not started; standalone Windows viewer build never done.
5. Anamorphic squish (calibration aspect 2.222 vs OpenDIBR output 1.778) still open.
6. Room/session test plan (Aug 12/18) and the calibration-TTL live test were never run; `deploy.yml` runs against the deactivated EC2 host would fail, so pushes to `streaming-server` `main` that touch `registration-service/**` should be avoided until EC2 is back.
7. The FFS branch is **not merged** into `dibr-bridge` `main` (`5ba076c`); the frozen `dibr-bridge.exe` bundled in StreamingAssets is still the Sep 21 build and cannot run FFS.

## 8. Commits (all pushed unless noted)

- `Master Thesis Client` `main`: `2de20c9`, `4c100ed`, `570babb`, `d4ba975`. The `DevServerUrlMenu.cs.meta` file was still untracked at the time of writing and was handed to the client session to commit.
- `dibr-bridge` `feature/fast-foundationstereo-backend`: `40aec72`, `5baf19f`, `0585b97`, `d1f1496`, `f156c23`, `327d638`, `1651180`.
- `streaming-server`, `open-dibr`, `client-sdk-unity`, `Fast-FoundationStereo`: no changes.
