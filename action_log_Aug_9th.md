# Master-Thesis-Client — Unified Camera/Viewer Unity App

This app (desktop + Android) now handles **both** roles of the multi-camera streaming system at runtime:

- **Viewer**: connects to LiveKit, subscribes to camera tracks, switches between cameras via a dynamic grid.
- **Camera**: registers with the server, opens the device camera, publishes it into the room, and shows the operator their own live feed.

It was originally viewer-only; the camera-publisher logic was merged in from the sibling repo `Master-Thesis-Camera-Instance` because the two apps shared nearly all framework code and only diverged after the connect screen. `Master-Thesis-Camera-Instance` is superseded — this repo is now the source of truth for both roles. See `../CLAUDE.md` for how this fits into the wider workspace.

## Role selection flow

1. `RoleSelectScreen` (shown first, via `AppRoleManager.Start()`) lets the user pick Camera or Viewer.
2. `AppRoleManager.SelectRole(AppRole)` activates exactly one of two manager subtrees under `Managers` in the scene: `ViewerManagers` (`ClientManager` + `LiveKitManager` + `ViewerTokenClient`) or `CameraManagers` (`CameraClientManager` + `RegistrationClient` + `LiveKitCameraPublisher`). Both start `SetActive(false)` — only the selected one ever wakes up, so the other role's Singletons never exist for that session.
3. `Assets/Editor/SceneSetupEditor.cs` (`Tools > Setup Scene`) builds both subtrees plus `AppRoleManager` in one idempotent pass — re-running it never duplicates anything.

## Unified connect flow (`ConnectionScreen`)

One `ConnectionScreen` handles **both** roles' entire connect sequence, deliberately mirroring the Viewer flow for Camera too:

- `OnConnectClicked()` branches on `CameraClientManager.HasInstance` (a reliable proxy for "which role is active," since only one manager subtree is ever active) — either drives `RegistrationClient` + `CameraClientManager.StartRegistering()`, or `ViewerTokenClient.FetchToken()`.
- Subscribes directly to `RegistrationClient`/`LiveKitCameraPublisher` events (Camera) and `ViewerTokenClient`/`LiveKitManager` events (Viewer) to drive the same status text and the same retry button for both roles — no separate error UI per role.
- Stays visible through the **entire** connect sequence — registration *and* the LiveKit connect/publish handshake for Camera; token fetch *and* LiveKit connect for Viewer — only switching away once actually fully connected/streaming, never partway through on a transient failure.
- `CameraClientManager.RegisteringState` re-shows `ConnectionScreen` (guarded against a redundant `Show()` if it's already the current screen) rather than a separate status screen, so even a mid-stream disconnect-and-retry routes back through the same UI.

## Camera role specifics

- `ConnectionStatusScreen` is the **self-preview** screen, shown only once `CameraClientManager` reaches `StreamingState` (i.e. actually publishing) — renders the local `WebCamTexture` (exposed as `LiveKitCameraPublisher.Texture`) onto a `RawImage`, with rotation/mirror correction (`videoRotationAngle`/`videoVerticallyMirrored`) and an `AspectRatioFitter`. Exists so the camera operator sees their own feed, not just a status string — it would be a bad experience if only the viewer could see what a phone was streaming.
- Optional camera label: `ConnectionScreen` has a `CameraLabelField` (visible for Camera role only) that sets `LiveKitCameraPublisher.Label`. Right after the LiveKit room connect succeeds, `LocalParticipant.SetName(label)` sends it — this needs `canUpdateOwnMetadata: true` on the token grant (see `Master-Thesis-Server/registration-service/server.js`). An empty label is a full no-op (no `SetName` call at all); the viewer just falls back to the raw identity.
- `LiveKitCameraPublisher`'s camera source `Start()` must run **before** `PublishTrack` — reversing this order deadlocks WebRTC SDP negotiation. This predates the merge; don't regress it.

## Viewer role specifics

- `CameraSwitcher` builds camera-switch buttons fully dynamically from `LiveKitManager.VideoTracks` — there is no hardcoded `cam1..cam10` list. `OnConnected` replays `OnVideoTrackAvailable` for whatever's already in `VideoTracks` at connect time (tracks are populated before `OnConnected` fires), which also correctly creates buttons for cameras that were already streaming before the viewer joined.
- Button labels use the camera's LiveKit participant `Name` if the camera set one (via `ResolveLabel`), falling back to `identity.ToUpper()` otherwise. Labels update live via `LiveKitManager.OnParticipantNameChanged` (wraps the SDK's `Room.ParticipantNameChanged`).
- Camera capacity is **not** capped client-side. The only cap is the server's `SLOT_COUNT` env var (`Master-Thesis-Server/registration-service`), which was already configurable, not hardcoded — raising capacity is a config change, not a code change.

## Shared code patterns

- `PostRequestClient<T> : Singleton<T>` (`Assets/Scripts/Manager/PostRequestClient.cs`) is the shared POST-with-retry-coroutine base for `ViewerTokenClient` and `RegistrationClient`. Subclasses only supply `Endpoint` and `TryApplyResponse(json)` — the retry loop, attempt counting, and failure messaging live in the base once.
- `Patterns/*` (Singleton, StateMachine, PubSub, ObjectPooling, PersistAcrossScenes) and `UI/Base/*` (BaseScreen/BasePopup/BaseNotify/BaseOverlap) are framework code both roles share unchanged.

## Local server stack for testing

`Master-Thesis-Server`'s `.env`/`livekit.yaml` `LIVEKIT_NODE_IP` must match this machine's **current** LAN IP, or the LiveKit WebRTC connect fails outright (an unreachable advertised address) even though plain HTTP calls to the registration service still succeed. Re-run `configure.bat` (in `Master-Thesis-Server/`) and `docker compose up -d` after any network change (switching Wi-Fi, hotspot, VPN, etc.) before testing against the local stack.

## Known environment gotcha (automation only, not a code bug)

Driving this project's Unity Editor headless/unfocused via Unity MCP automation is unstable: Play Mode's frame loop can stall for many seconds, newly-`SetActive(true)`-enabled objects' `Start()` doesn't always fire on schedule, and the Editor occasionally auto-exits/re-enters Play Mode mid-session (which can also take the LiveKit native FFI layer down with it, surfacing as an "FFI panic"). Worked around during development by driving state via `execute_code` reflection rather than relying on real frame ticks. A normal, focused Editor session should behave normally — this is specific to automated/headless testing, not a symptom of a real bug.
