# Action Log — Sep 21st (dibr-bridge: real root cause found for the grey
synthetic view, 3 stereo bugs fixed, PyInstaller build bug found and fixed)

Follow-up to `action_log_Sep_21st_dibr_depth_debugging.md`, which ended with
the pipeline running end-to-end but depth still empty/wrong, handed off to
the bridge session with a scoped bug diagnosis and unit tests. This session
picked that up, got a peer review of the same problem, verified it
independently, found the review's diagnosis was right but incomplete, fixed
three real bugs in `stereo_depth.py`, and separately discovered the frozen
exe has never actually worked at all due to an unrelated build-tooling bug.

---

## 1. Peer review + independent verification

A peer review (pasted into the session, not self-generated) diagnosed the
grey-view bug as primarily a **calibration/frame mismatch**, not a tuning
problem: using the ChArUco markers both cameras see as ground truth, it
found corners sitting a median ~363px off their epipolar lines under the
stored calibration — meaning SGBM was searching the wrong rows entirely, and
no parameter sweep could ever fix that. It also flagged 3 code bugs in
`stereo_depth.py` and recommended capturing calibration through the same
orientation path as live frames, plus switching extrinsics to a joint
`cv2.stereoCalibrate`.

Rather than trust this at face value, spent real effort verifying it:

- **Reproduced the review's exact number independently.** Recomputed the
  intrinsics-scaling bug (calib_a's resolution used for both cameras) by
  hand against the real capture data and got the review's exact reported
  figure (cam3's fx becoming 618 instead of ~940) when testing with the pair
  in the opposite call order from an earlier test — confirms the bug is
  real, not a one-off misreading.
- **Two Explore agents** confirmed: (a) `Master-Thesis-Client`'s calibration
  capture path (`CalibrationTutorialScreen.cs:107-111`) applies no
  rotation/flip correction to the pixel buffer fed to ArUco, while the
  live-publish path (`WebCameraSource.cs:105-112`) explicitly flips rows and
  tags rotation metadata — a real, independently-confirmed divergence; (b)
  the existing `tests/test_stereo_depth.py` fixtures structurally cannot
  catch any of the 3 code bugs (equal calibration resolutions, A always
  genuinely left, zero relative rotation, in every fixture).
- **Derived and numerically verified** the third bug's fix (rectified-frame
  vs. original-frame depth) from scratch — a synthetic 25°-rotation test
  case matched the derived correction formula to 1e-12 precision before any
  code was written.

Decision with the user: fix the orientation bug + recalibrate + measure
epipolar error first, before committing to the bigger joint-`stereoCalibrate`
rearchitecture (cam1/cam2 run on separate physical laptops, so a joint solve
needs real cross-device sync engineering, not a function swap). The
calibration-capture fix was handed off to the `Master-Thesis-Client` session
rather than edited directly, since another live session was already working
in that repo.

---

## 2. Three bugs fixed in `dibr-bridge/bridge/stereo_depth.py`

1. **Per-camera intrinsics scaling.** Both cameras were scaled to output
   resolution using only `calib_a`'s native calibration resolution — wrong
   whenever the two cameras were calibrated at different resolutions
   (confirmed real: 1600×720 vs 2432×1080). Now each camera scales by its
   own resolution.
2. **Hardcoded left/right camera assumption.** `compute_pair` always fed
   camera A first to StereoSGBM assuming A is physically left. Verified via
   `P2[0,3]`'s sign (from `cv2.stereoRectify`) that this is false for the
   real (cam2, cam3) pair — B is actually left. Now decided per-pair from
   the real geometry.
3. **Rectified-frame vs. original-frame depth.** The returned depth was Z
   along the *rectified* camera's optical axis, not the original camera's —
   these differ whenever the rectification rotation isn't identity
   (confirmed real: ~26° between the actual rig's two cameras). Fixed with a
   per-pixel correction factor (`R1[:,2] · K_rect⁻¹[u,v,1]`, applied before
   un-rectifying), derived independently and verified numerically before
   implementing.

Added 3 new regression tests, each specifically targeting one bug (unequal
per-camera calibration resolution; B genuinely left of A; non-identity
rectification rotation checked against closed-form ground truth, not a noisy
image-based test). All 8 tests (5 original + 3 new) pass.

## 3. New tool: `check_stereo_calib.py`

Detects shared ArUco markers between two camera captures and measures
epipolar error against the current calibration — independent of
`StereoDepthComputer`'s own code, no LiveKit/RTSP/OpenDIBR needed. Run
against the two real capture pairs from the previous session:

```
median = 333.5 px, max = 597.4 px   (cam3_cam2 pair)
median = 281.5 px, max = 599.1 px   (cam2_cam3 pair)
```

Independently reproduces the peer review's ~363px finding almost exactly —
strong confirmation the calibration genuinely doesn't describe these frames,
via a completely separate tool and marker detection than the review used.

---

## 4. Turns out cam2 was just physically flipped, not a software bug

Partway through, the user clarified directly: cam2 was **physically placed
upside-down by accident after being calibrated** — not (primarily) a
software orientation-handling bug. The numbers fit this well: a 180° flip
lines up almost exactly with the review's ~160°-actual-vs-~26°-stored
rotation discrepancy.

Re-tested with cam2's captured frame rotated 180° (undoing the flip) against
the **original, unmodified** calibration:

```
epipolar error: 333.5px -> 90.6px   (better, but nowhere near the ~2px target)
```

Confirms the flip wasn't a perfectly clean 180° roll (the camera likely also
shifted slightly on remount) — a software rotation can reduce but not fully
substitute for a real recalibration. Re-ran the full depth pipeline (with
the 3 code fixes in place) against the same rotated data:

- Both cameras' median depth estimates now agree closely with each other
  (~0.34–0.35m each, vs. wildly divergent numbers before the code fixes) —
  real evidence the left/right and Z-correction fixes are working.
- The depth visualization shows a large, single-colored coherent region for
  the first time (previously pure scattered rainbow noise) — real signal is
  present.
- Still only ~8-9% valid-pixel coverage with a noisy tail up to hundreds of
  metres — not yet usable, consistent with the still-large (~90px) residual
  calibration error.

**Conclusion, not yet decided:** whether classical SGBM is good enough once
calibration is genuinely fixed (not just image-rotated around a bad
calibration) is still open. Explicitly deferred the "switch to a learned
stereo matcher (RAFT-Stereo/CREStereo)" question until a real recalibrated
capture can be tested — right now we'd be judging the matcher against
calibration already known to be ~45x worse than acceptable.

---

## 5. Separate discovery: the frozen exe has never actually worked

While rebuilding to pick up today's fixes, found the PyInstaller build has
been silently broken since before this session started. `dibr-bridge.spec`
has a `collect_all('livekit')` fix (committed 2026-09-21, from an earlier
session) needed to bundle `livekit_ffi.dll` — without it, the frozen exe
raises `ImportError` on launch, before reaching any real code.

**Root cause:** `README.md`'s documented rebuild command passed a script
path (`entrypoint.py`) instead of the `.spec` file. PyInstaller silently
regenerates and overwrites a same-named `.spec` file with bare defaults
whenever it's invoked this way — deleting the `collect_all` fix on every
single rebuild, including the very first build after whoever wrote that fix
committed it. The dist build and the copy already sitting in
`Assets/StreamingAssets/DibrBridge/` (157MB, dated Sep 20 20:17) had **zero**
livekit files in them — confirmed by direct inspection, not inference.

**Fixed:** restored `dibr-bridge.spec` (found reverted, uncommitted, in the
working tree), corrected the README to
`python -m PyInstaller --noconfirm --clean dibr-bridge.spec`, rebuilt
(183MB now, +26MB of livekit data/binaries). Verified: `livekit_ffi.dll`
present, and a real launch against a bogus server reaches its actual HTTP
call (`/viewer-token`) and fails with a normal connection error, not an
`ImportError` — the first time this exe has verified as actually
functional. Copied into `Master-Thesis-Client/Assets/StreamingAssets/DibrBridge/`.

**Practical effect:** any live test attempted before today, on any build,
would have died immediately on launch regardless of what else was fixed.
That blocker is gone now.

---

## Files touched

**`dibr-bridge`** (commits `1ab7b0a`, `5ba076c`, pushed):
- `bridge/stereo_depth.py` — the 3 bug fixes above.
- `tests/test_stereo_depth.py` — 3 new regression tests.
- `check_stereo_calib.py` — new epipolar-error verification tool.
- `scratch_test_capture2.py` — added cam2-rotation option for testing.
- `dibr-bridge.spec` — restored (had been silently reverted).
- `README.md` — corrected rebuild instructions.
- `dist/dibr-bridge/`, `Master-Thesis-Client/Assets/StreamingAssets/DibrBridge/` — rebuilt exe (gitignored binaries, not committed, but copied).

**`Master-Thesis-Client`**: nothing edited directly this session — the
calibration-capture orientation fix (now downgraded from "urgent blocker" to
"worth doing eventually, not blocking" per the physical-flip explanation)
was handed off to the `master-thesis-client-d8` session via message, not
implemented here.

---

## State at end of day

- 3 real stereo-depth bugs fixed and tested in `dibr-bridge`, pushed.
- `check_stereo_calib.py` built and validated against real data — ready to
  gate the next live re-test.
- The exe that gets bundled into the Unity build was silently broken since
  before this session; now fixed, verified, and redeployed.
- Root cause of the grey view is now understood as **calibration/frame
  mismatch from a physical camera flip**, not primarily a code or technology
  problem — though the 3 code bugs were real and independently necessary
  fixes regardless.
- **Blocked on:** a fresh `DibrDepthTestCaptures/` pair from
  `master-thesis-client-d8`, captured after cam2 is either physically
  re-flipped to match its original calibration or recalibrated in its
  current orientation. Once that lands: run `check_stereo_calib.py` first
  (target ~2px median epipolar error); only if that passes, re-run the full
  depth pipeline and decide whether SGBM is sufficient or a learned stereo
  matcher is actually needed.
