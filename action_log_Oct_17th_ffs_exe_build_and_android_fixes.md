# Action Log — Oct 17th (FFS-only bridge exe, build script, Android build fixes, 4 GB GPU findings)

## Summary

Bridge packaging moved to a single FFS exe; the client build script now builds only that. Android APK builds. All code changes are committed and pushed (bridge `b886b63`, `e38969b`; client and this log in the final push). Nothing here was run with real cameras; full FFS inference inside the frozen exe is not yet verified.

## 1. dibr-bridge

Pushed earlier (`000abb6`, `feature/fast-foundationstereo-backend`):
- Unrectify remap uses nearest-neighbour (`INTER_NEAREST`), with a regression test.
- torch / torchvision / timm excluded from the slim spec.

Pushed later today (`b886b63`, `e38969b`):
- New `dibr-bridge-ffs.spec`: torch + FFS source bundled, UPX off.
- `stereo_depth_ffs._ffs_repo_path` resolves the bundled `ffs_src` when frozen.
- `tests/test_ffs_repo_path.py` (5 tests).
- `.gitignore`: `build-ffs/` and `dist-ffs/`.

## 2. Client

- `Tools/BuildStreamingAssets.ps1 -Only DibrBridge` builds only the FFS exe: spec + `20-30-48` checkpoint + `cfg.yaml` + FFS license; fails loudly if any is missing. The SGBM-only exe path is removed from the script (SGBM code stays in the bridge as the `auto` fallback).
- Android: custom Gradle properties template (`-Xmx8192M`) fixes the `:launcher:packageRelease` OutOfMemoryError; OpenCVForUnity `opencv2.framework` limited to iOS.
- Android APK built OK (~1039 MB, 0 errors): `Builds/_9.2/MT_Client_v9.2_20261017_1635.apk`.
- Windows player build failed (6 errors, not investigated); testing continues in the Unity editor.

- Committed and pushed at the end of the session (explicit paths only): `Tools/BuildStreamingAssets.ps1`, `ProjectSettings/ProjectSettings.asset` (Gradle template flag), `Assets/Plugins/Android/` (`gradleTemplate.properties` + metas). Unity's own rewrites after the reimport (`build_info`, URP global settings, `packages-lock.json`, Performance Test JSON, ...) were deliberately left uncommitted. The `opencv2.framework.meta` change is local only (`Assets/OpenCVForUnity/` is gitignored).

## 3. Findings

GTX 1650 Max-Q (4 GB), FFS warm-up peak VRAM:

| Infer resolution | Peak |
|---|---|
| 640x360 | 1421 MiB |
| 960x540 | 2942 MiB |
| 1280x720 | 5249 MiB (spills past 4 GB) |

- fp16 produces NaN on this GPU.
- Dev venv upgraded torch 2.6.0+cu124 -> 2.7.1+cu128 (arch list includes sm_75 and sm_120), so one exe serves the 1650 and the RTX 5070 machine. `pip check` clean.
- FFS exe frozen OK: 5.8 GB, backend `ffs` on cuda, no import errors.
- Local server is up on 192.168.2.107.
- Two machines: GTX 1650 Max-Q (dev PC) and RTX 5070 (viewer / FFS at 1280x720); see `CLAUDE.md` > Machines and setup.
- Failed Windows player build left a 2.7 GB partial folder (deleted).

## 4. Decisions

- Build folder naming: `Builds/_<MM>.<NNN>.<yyyyMMdd-HHmm>` (details in the working-flow doc, section 10).
- FFS is the depth direction; no SGBM tests.

## 5. Open items

1. Physical recalibration + live test on both machines.
2. Verify FFS inference inside the frozen exe.
3. Lower infer resolution for the 1650 (960x540 fits).
4. No hard-coded IP: client `DefaultServerUrl` is the dead EC2 address; `DevServerUrlMenu` hard-codes 192.168.1.3.
5. Merge `feature/fast-foundationstereo-backend` to `main` after a GPU test.
6. Windows player build errors.
7. Trim the ~1 GB APK (OpenCVForUnity iOS/WebGL payloads).
