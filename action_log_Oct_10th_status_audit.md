# Action Log — Oct 10th (status audit: loose ends vs. actual repo state)

## Summary

Audited every open item from the Aug 12 to Sep 21 reports against each repo's git history, index and source. Most items were resolved in later commits that no report mentions. There was also an unreported Sep 26 session on `dibr-bridge` (Fast-FoundationStereo backend). The root `CLAUDE.md` was rewritten to match.

Method: there was no shell on this machine this session. I read each repo's reflog (`.git/logs/HEAD`), parsed `.git/index` to compare staged mtimes with working-tree mtimes, and grepped the current source. All tracked files checked were clean, meaning committed.

## Resolved, but not in any report

| Loose end (source report) | Status | Evidence |
|---|---|---|
| Unity doesn't pass `--room-code` to the bridge (Sep 21 depth debugging) | **Fixed** | `OpenDibrSessionManager.cs:156-157` passes `--room-code "{AppConfig.RoomCode}"`. |
| Control-channel timeouts and the "already active" wedge uncommitted (Sep 21) | **Committed** | 20 s / 8 s timeouts in `OpenDibrControlChannel.cs`; "already active" reconciliation at `OpenDibrSessionManager.cs:374`. Pulled in at client `e970334` (Sep 21 16:34). |
| `CameraSwitcher` white-screen fix uncommitted (Sep 20 viewer fixes) | **Superseded** | The two-player crossfade was removed entirely. `RunSwitch` now uses a single stream player with a DIBR overlay, or an instant cut. Committed in `8994564`. |
| Calibration capture orientation differs from the publish path (Sep 21 bridge fixes, handed to the client session) | **Fixed** | `CalibrationTutorialScreen.cs:111-121` row-flips via `Graphics.Blit` to match `WebCameraSource`. `CameraCalibrator` fixes K3 and tangential distortion to 0. Commit `8994564` "[fix]: calibration orientation/distortion + live DIBR depth-test capture tool". |
| Extrinsics captured from a single noisy frame | **Fixed** | `681d67d`: extrinsics now need a stable pose across frames. |
| StreamingAssets binaries: commit or ignore? (Sep 20 camera switch) | **Gitignored** | `e41c2fa` plus `.gitignore` lines 121-130. Rebuild script committed. |
| `dibr-bridge` as its own repo? (Sep 20) | **Done** | Repo since `2aad6a4` (Sep 20), with a GitHub remote. |
| Calibration server endpoints (Sep 12 EC2 log's next steps) | **Done** | `a73800b`; TTL in `f0f8e7d`; board resized to 40 mm squares in `6974945`. |
| Multi-camera test since the merge (since Aug 18) | **Exercised de facto** | The Sep 20–21 live DIBR runs used cam1–cam3 across rooms (BD624Y, FADW88), with cam1 on a separate laptop. No dedicated test plan was run. |
| Room/session isolation (Aug 12 plan) | **Partly exercised** | Dynamic room codes worked live in the same runs. Webhook slot-freeing across concurrent rooms was not checked. |

## Unreported: Sep 26 `dibr-bridge` session (branch `feature/fast-foundationstereo-backend`)

Commits:
- `bc74417` adds Fast-FoundationStereo as an opt-in, swappable backend (`--stereo-backend ffs`). SGBM stays the default. Shared logic was pulled up into `BaseStereoDepthComputer`.
- `17f40be` fixes FFS NaN output on Turing GPUs (testing was on a GTX 1650 Max-Q) and adds a resolution override.
- `297d49c` fixes a hardcoded `alpha=0` rectify crop that **killed depth coverage for both backends**, SGBM included.
- `ce01916` adds two diagnostic scripts: uncalibrated rectification, and a public-benchmark FFS comparison.
- `071762a` fixes the depth visualization, which was clipping to one colour because of stale `depthMin`/`depthMax`. Real depth after the max-disparity fix was in the 0–0.55 m range.

State:
- `origin` only has up to `17f40be`. The last 3 commits are **local only**. `scratch_test_selfcalib_ffs.py` is untracked.
- The branch is **not merged** into `main`. The bundled `dibr-bridge.exe` comes from `main` (`5ba076c`), so it lacks the `297d49c` coverage fix.
- `Fast-FoundationStereo/` was cloned as a sibling repo (fork at `476f424`, unmodified).
- Output images are in `depth_out_ffs/`, `depth_out_selfcalib/` and `depth_out_benchmark/` (gitignored). No latency or quality conclusions were written down.

## Still open

1. Live synthetic-view correctness. Needs recalibration of the current rig with a post-`8994564` build, a fresh `DibrDepthTestCaptures/` pair (the latest is still from Sep 21), `check_stereo_calib.py` at about 2 px, and then SGBM vs FFS.
2. Push or merge the FFS branch. At minimum, bring `297d49c` to `main` and rebuild the exe.
3. Calibration aspect ≈2.22 vs. output 1.78 (anamorphic squish).
4. Never run: the live calibration-TTL test, and webhook slot-freeing in multi-room use.
5. Hardening: bridge LiveKit reconnect, bridge startup error handling, pruning `sessionGeneration`, ffmpeg encode tuning.

## Other findings

- The detailed root `CLAUDE.md` described in `action_log_Sep_10th.md` (Known Constraints, Current Status) is not on this machine. The workspace root isn't a git repo, so it never synced from the other machine. `Master-Thesis-Server/README.md` still points at `../CLAUDE.md` "Known Constraints". The rewrite covers this in its Server section.
- The root `CLAUDE.md` referred to a `Master-Thesis-Client/CLAUDE.md` that doesn't exist. Its content survives as `action_log_Aug_9th.md` and has been folded into the root file.
- `Master-Thesis-Client/Assets/StreamingAssets/build_info` is stamped "Build from NASH-EQUI17BRIU at 10/10/2026 10:13 PM". A build was made today; the target platform wasn't confirmed.

## Files changed

- `CLAUDE.md` (workspace root): full rewrite.
- `Master-Thesis-Reports/action_log_Oct_10th_status_audit.md`: this file (not committed).
