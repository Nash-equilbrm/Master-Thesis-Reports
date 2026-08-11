# Future work: running the Camera role on a Mixed Reality headset

> Status: **discussion / future work — not implemented, not scheduled.** This is a feasibility ranking to inform a future decision, not an active plan.

## Question

Can the Camera role of this unified app (currently: registers with the server, opens a device camera via `WebCamTexture`, publishes into a LiveKit room) run on a Mixed Reality headset instead of a phone? Three candidate platforms were evaluated: **Meta Quest 3/3S**, **Microsoft HoloLens 2**, and **Apple Vision Pro** — ranked by implementation ease, stability, and long-term support from LiveKit.

Research basis: (1) reading `client-sdk-unity`'s own `README.md`/`AGENTS.md` and its `Runtime/Plugins/` native-binary layout directly — this is the actual LiveKit Unity SDK checked out locally in this workspace (`client-sdk-unity/`), so its real support matrix is ground truth, not guesswork — and (2) current (August 2026) web research on each platform's camera-access policy and hardware lifecycle, since this is exactly the kind of fast-moving, SDK-version-dependent question where stale assumptions would be actively misleading.

## Ranking: Meta Quest > HoloLens 2 > Apple Vision Pro

### 1. Meta Quest 3/3S — clear best choice

- **LiveKit SDK support**: `client-sdk-unity/README.md`'s own "Platform Support" section explicitly lists `Android` as officially supported (alongside Windows/macOS/Linux/iOS; only WebGL is marked unsupported). Quest is an Android/ARM64 device, and the SDK already ships the exact binaries needed: `Runtime/Plugins/ffi-android-arm64/liblivekit_ffi.so`. The LiveKit connect/publish machinery this app already uses (`LiveKitCameraPublisher`, `RegistrationClient`, the whole registration→connect→publish flow) should work on Quest largely unmodified — Quest is not an unusual runtime from LiveKit's perspective, it's a normal Android app.
- **Camera access**: Meta's **Passthrough Camera API** is a stable, GA feature (publicly released Horizon OS v76, Camera2-based access since v74) with an official Unity integration and a full sample repo (`oculus-samples/Unity-PassthroughCameraApiSamples`). The `PassthroughCameraAccess` component's `GetTexture()` gives real, raw camera frames with intrinsics/extrinsics/timestamps — genuinely usable for streaming, not a toy/preview-only API.
- **What implementation would actually involve**: `LiveKitCameraPublisher.OpenCamera()` currently opens a `WebCamTexture`; Quest needs an alternate capture path using `PassthroughCameraAccess.GetTexture()` instead. The SDK's `Runtime/Scripts/Video/` folder has more than one `RtcVideoSource` implementation (`WebCameraSource.cs`, `TextureVideoSource.cs`, `CameraVideoSource.cs`) — `TextureVideoSource` in particular looks like it may accept a generic `Texture`, not specifically a `WebCamTexture`, which would make wiring Quest's camera texture into the existing publish pipeline (`LocalVideoTrack.CreateVideoTrack(...)`) a much smaller lift than it first sounds. Worth reading that file first if this is picked up.
- **Long-term outlook**: Meta is actively shipping new Quest hardware and actively investing in this exact API surface; LiveKit officially supports Android. Both halves of the dependency chain are alive and moving forward.

### 2. HoloLens 2 — technically possible, but a declining platform

- **LiveKit SDK support**: Not officially supported. A Windows-ARM64 binary exists (`Runtime/Plugins/ffi-windows-arm64/livekit_ffi.dll`), but its `.meta` only enables it for **Standalone** (desktop Win/Win64) builds — there is no "Windows Store Apps" (WSA/UWP) platform flag set, and zero `UNITY_WSA` conditional code anywhere in the SDK. Getting the binary *included* in a UWP build is a small, mechanical fix (edit the `.meta` platform flags); whether the Rust-based FFI core actually *functions* inside UWP's sandboxed app-container model (restricted raw socket/threading APIs) is untested, unofficial territory with no existing precedent in this codebase — real integration risk, not just a checkbox.
- **Camera access**: HoloLens 2 needs its own capture path too (Unity's `UnityEngine.Windows.WebCam` APIs, not `WebCamTexture`) — another from-scratch integration, same shape of work as Quest's.
- **Long-term outlook — the dealbreaker**: Microsoft discontinued HoloLens hardware production in October 2024 and confirmed a full exit from HoloLens hardware development in February 2025. HoloLens 2 now only receives security/critical fixes through **December 31, 2027**, with no new features. Even if the LiveKit integration were made to work, this directly fails the "long-term support" bar this ranking was asked to weigh — it's a sunsetting platform with a fixed expiration date, not a moving-forward one.

### 3. Apple Vision Pro — not currently practical

- **LiveKit SDK support**: None at any layer. No `visionOS/` plugin folder, no PolySpatial package reference, no `UNITY_VISIONOS` conditional code anywhere in `client-sdk-unity`. This would need LiveKit to ship official visionOS support (no announced roadmap/ETA — an external dependency entirely outside this project's control) or personally porting/building LiveKit's Rust FFI core for visionOS, which is a major undertaking far outside thesis scope.
- **Camera access — a separate, harder blocker**: Apple does not allow general third-party apps to access Vision Pro's raw camera/passthrough feed at all. Raw camera access is currently limited to (a) specially-licensed enterprise apps, explicitly restricted to "non-public, business-setting-only" use, requiring a license grant from Apple, or (b) a narrow accessibility API (visual-interpretation assistance, Be My Eyes-style) for Apple-approved apps. Neither realistically fits a thesis project. visionOS 26 (2026) reportedly opened camera access progressively (left camera, then stereo pair) but this still appears scoped to approved use cases, not general availability.
- **Net assessment**: this is a double blocker — no SDK-level path *and* an OS policy wall — making it the least tractable option by a wide margin, independent of engineering effort.

## Recommendation

Pursue **Meta Quest** first, if/when this becomes active work. It's the only option with both a genuinely supported LiveKit path and an official, stable, actively-maintained camera API — the implementation is additive (a new Quest-specific capture source alongside the existing `WebCamTexture` one, likely gated by `#if PLATFORM_ANDROID` plus a Quest/OpenXR check, or a build-target-specific `LiveKitCameraPublisher` variant) rather than a fight against unsupported territory. HoloLens 2 is a real fallback only if Quest turns out to be blocked for some unrelated reason, and even then its EOL timeline should be weighed again before committing effort. Vision Pro isn't worth pursuing until Apple's third-party camera policy changes materially.

## If/when this becomes active work

1. Confirm `TextureVideoSource.cs`'s exact API (does it accept an arbitrary `Texture`/`RenderTexture`, or is it still `WebCamTexture`-shaped under the hood?) before committing to the "small lift" framing above.
2. Prototype: get `PassthroughCameraAccess.GetTexture()` rendering to a `RawImage` in a bare Quest test scene first (proves camera access + permissions work end-to-end) before wiring it into `LiveKitCameraPublisher`.
3. Confirm Quest build settings (Android target, OpenXR provider, min API level, the camera permission, and any Meta Store review requirements) don't conflict with this project's existing Android player settings.
4. End-to-end target: a Quest headset registers via the existing `/register` flow, opens the passthrough camera, publishes to LiveKit, and a desktop Viewer instance subscribes and displays it — reusing the existing registration/connect/retry infrastructure entirely; only the capture source changes.
