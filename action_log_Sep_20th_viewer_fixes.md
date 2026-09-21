# Action Log — Sep 20th (BuildStreamingAssets fixes, Android exclusion, white-screen camera switch)

## Session summary

Three separate issues found and fixed in a single session: the
`BuildStreamingAssets.ps1` tool had two bugs that prevented it from running at
all, the StreamingAssets desktop binaries had no mechanism to be excluded from
Android builds, and the Viewer's camera-switch transition showed a solid white
screen after the first switch due to a rendering-order bug in `CameraSwitcher`.

---

## Fix 1 — `BuildStreamingAssets.ps1` encoding (parse errors on every run)

**Symptom.** Running `.\Tools\BuildStreamingAssets.ps1` from PowerShell threw
~10 parser errors: `Unexpected token`, `Array index expression is missing`,
`The string is missing the terminator: "`. The script was completely
un-runnable.

**Root cause.** The file was saved as UTF-8 without BOM. PowerShell 5 (the
Windows built-in shell) reads `.ps1` files as the system ANSI code page
(Windows-1252) when no BOM is present. The em-dash character `—` (UTF-8
`E2 80 94`) decodes as three bytes: `â` (`E2`), `€` (`80`), and `"` (`94`
= RIGHT DOUBLE QUOTATION MARK in Windows-1252). That curly-quote character
terminates the surrounding string literal mid-line, cascading into every parse
error shown.

**Fix.** Re-saved the file with a UTF-8 BOM using
`[System.IO.File]::WriteAllText(path, content, UTF8Encoding(true))`. No
content changed; PowerShell now reads it correctly.

---

## Fix 2 — `BuildStreamingAssets.ps1` winget output leaked into return value

**Symptom.** After the encoding fix the script ran, but the MediaMTX step
failed with:

```
Copy-Item : Cannot find path 'C:\...\Master Thesis Client\This application is
licensed to you by its owner.' because it does not exist.
```

The path was the current working directory concatenated with a line of winget
license text.

**Root cause.** In PowerShell, every output line written inside a function
body is part of that function's return value — `return $x` appends `$x` to
whatever was already emitted, it does not replace it. `winget install` writes
several lines to stdout (progress, license text, success message), so
`Find-WinGetBinary` returned an array:
`@("Found ...", "This application is licensed to you ...", "C:\...\mediamtx.exe")`.
`Copy-Item` then iterated that array and tried the first element — the license
text — as a file path, resolving it relative to the current directory.

**Fix.** Piped the `winget install` call to `Out-Null`
(`Tools/BuildStreamingAssets.ps1:44`), so its stdout is discarded and the
function returns only `$found.FullName`.

**Verified.** Re-ran the full script: FFmpeg skipped (already present),
MediaMTX installed and copied, DibrBridge venv created + PyInstaller freeze
succeeded, OpenDIBR skipped (sibling repo not yet cloned). Commit `e527861`.

---

## Fix 3 — Android builds must not include StreamingAssets desktop binaries

**Observation.** The four StreamingAssets subdirectories (`FFmpeg`, `MediaMTX`,
`DibrBridge`, `OpenDIBR`) total ~467 MB and are Windows-desktop/NVIDIA-only.
Camera clients (Android) never use any of them. Including them in an Android
APK wastes ~467 MB and would likely exceed APK size limits.

**Fix.** Added `Assets/Editor/AndroidStreamingAssetsFilter.cs` — an editor
script implementing `IPreprocessBuildWithReport` and
`IPostprocessBuildWithReport`. Before any Android build, the four folders are
moved from `Assets/StreamingAssets/` to
`<ProjectRoot>/Temp/AndroidExcludedStreamingAssets/` (inside Unity's already-
gitignored `Temp/` directory) and `AssetDatabase.Refresh()` is called so
Unity's packer sees them as absent. After the build (success or failure both),
they are moved back and the database is refreshed again. Non-Android builds
(Windows standalone) are unaffected — the callbacks check
`report.summary.platform` first. If Unity crashes mid-build the folders stay
in `Temp/`; they are restored at the start of the next Android build attempt.

---

## Fix 4 — White screen after first camera switch (`CameraSwitcher.cs`)

**Symptom.** Room BD624Y, 2 live cameras (cam1, cam2), 1 viewer. Start on
cam1, switch to cam2 → solid white screen.

**Root cause.** `RunSwitch` uses two `CameraStreamPlayer`/`RawImage` layers —
`_streamPlayer` (current camera) and `_transitionPlayer` (incoming camera,
fades in on top) — and swaps them at the end of each transition. The
`TransitionDisplay` GameObject is created as the last sibling
(`SetAsLastSibling`) so it initially renders above `VideoDisplay`. After the
first switch's swap, `_streamPlayer` = TransitionDisplay (top) and
`_transitionPlayer` = VideoDisplay (bottom). This inverts the intended render
order: for the second switch, the code correctly fades `_transitionPlayer`
(VideoDisplay) from 0→1, but VideoDisplay is now *below* the fully-opaque
TransitionDisplay, so the crossfade is invisible. After the crossfade loop,
`_streamPlayer.Unsubscribe()` clears TransitionDisplay's texture; with alpha
still at 1 and texture null, a RawImage renders its color — white by default.
A second bug: after each swap `_transitionImage` was never zeroed, so the
retired player's white blank image would persist into the next switch regardless.

**Fix.** Two changes in `CameraSwitcher.RunSwitch`
(`Assets/Scripts/Stream/CameraSwitcher.cs`):

1. At the top of `RunSwitch`, after `EnsureTransitionLayers()`:
   ```csharp
   _transitionPlayer.transform.SetAsLastSibling();
   _dibrImage.transform.SetAsLastSibling();
   ```
   This re-establishes the correct rendering order (incoming on top of
   outgoing, DIBR overlay on top of both) before every switch, regardless of
   which player was active last time.

2. At the end of `RunSwitch`, after the player swap and `_transitionImage`
   reassignment:
   ```csharp
   SetAlpha(_transitionImage, 0f);
   ```
   Hides the retired display immediately so it cannot bleed through on the
   next switch.

---

## Why the synthetic view doesn't appear yet

Switching between cameras uses the plain crossfade fallback, not the
OpenDIBR synthetic view, for two independent reasons:

1. **`RealtimeDIBR.exe` is not bundled.** `DibrCapability.IsAvailable`
   (`Assets/Scripts/Manager/DibrCapability.cs:64`) checks for
   `StreamingAssets/OpenDIBR/RealtimeDIBR.exe`. The file doesn't exist because
   the `BuildStreamingAssets.ps1` OpenDIBR step was skipped — the sibling repo
   `C:\Workspace\HCMUS\Master\Repo\OpenDIBR` was not found. With
   `IsAvailable = false`, `RunSwitch` never calls `PreparePairAsync` and
   always takes the crossfade path. Fix: clone + build OpenDIBR into that path
   and re-run `.\Tools\BuildStreamingAssets.ps1 -Only OpenDIBR`.

2. **Calibration data for the cam pair is required.** Even with the binary
   present, `PreparePairAsync` fetches extrinsics for the specific (camA, camB)
   pair from the server via `CalibrationPairClient`. If this pair has never
   been calibrated and uploaded, `PrepareResult.NotCalibrated()` is returned
   and the code permanently falls back to crossfade for that pair for the rest
   of the session. Fix: run the in-app ChArUco calibration with both cameras
   seeing the board simultaneously and upload the result.

---

## Files changed

- `Tools/BuildStreamingAssets.ps1` — UTF-8 BOM + `winget install | Out-Null`
- `Assets/Editor/AndroidStreamingAssetsFilter.cs` — new; excludes desktop
  StreamingAssets from Android builds
- `Assets/Scripts/Stream/CameraSwitcher.cs` — sibling-order fix +
  `SetAlpha(_transitionImage, 0f)` after swap

Commit `e527861` covers the script fixes and the Android filter. The
CameraSwitcher fix was applied in the same session but not yet committed as of
this log.
