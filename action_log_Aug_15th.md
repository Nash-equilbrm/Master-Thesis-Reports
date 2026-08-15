# Master-Thesis-Client — Room/Session UI Race & JSON-Escaping Fixes

Code review of the recent commit range (`1fd60aa..HEAD`: DevCommandMenu, dotween, UI polish/AppManager, room/session system) surfaced five correctness bugs, all fixed today. Two were independent event-ordering races between screens and their owning state machines; the other three were narrower defects in JSON body construction and input validation. See `../Master-Thesis-Client/CLAUDE.md` and the merged-architecture notes in `action_log_Aug_9th.md` for how these pieces fit into the wider app.

## Root cause 1: dual subscribers racing the same event

Both a screen (`RoomScreen`, `ConnectionScreen`) and its owning state (`AppManager.RoomEntryState`, `ClientManager.IdleState`) were independently subscribing to the same upstream event (`RoomClient.OnRoomReady`, `LiveKitManager.OnConnected`) and reacting to it in parallel, with no guarantee about subscription order. Whichever handler fired first won the race.

- **`AppManager.RoomEntryState` vs `RoomScreen`** (`Assets/Scripts/Manager/States/AppManager.RoomEntryState.cs`, `Assets/Scripts/UI/Screen/RoomScreen.cs`): `AppManager` subscribed to `OnRoomReady` *before* `RoomScreen` did (in `Enter()`, ahead of `ShowScreen<RoomScreen>`'s own subscription inside `Show()`), so on room creation `AppManager` unconditionally jumped to `RoleSelectState` before `RoomScreen`'s own creator-gated "show code, wait for Continue" logic ever ran. The room creator was bounced straight to role-select and never saw (or got a chance to share) the generated room code.
  - **Fix**: `RoomScreen` now owns a new `event Action OnEntryComplete`, fired immediately for a joiner (room ready) and only on the Continue-button click for a creator. `AppManager.RoomEntryState` subscribes to that instead of to `RoomClient.OnRoomReady` directly — it now waits on the screen's own notion of "done" rather than racing it.
- **`ConnectionScreen` double-hide** (`Assets/Scripts/UI/Screen/ConnectionScreen.cs`, via `Assets/Scripts/UI/Base/BaseUIElement.cs`): the same shape of race let a screen's `Hide(callback)` get called twice in one frame (once via `UIManager.RemoveScreen`'s state-driven teardown with a `Destroy` callback, once via the screen's own event handler with no callback). `AnimateHide` unconditionally called `Kill(false)` on any in-flight tween, silently dropping the first call's `onComplete` — so the `Destroy` callback never ran and the GameObject leaked (inactive, un-destroyed, orphaned under `cScreen.transform`).
  - **Fix**: `BaseUIElement.Hide()` now no-ops (queuing the callback instead of restarting the tween) when called again while already hiding, so a second `Hide()` call can no longer discard the first one's completion callback.

## Root cause 2: `UIManager` removed screens from its dictionary before their hide-out animation finished

`RemoveScreen`/`RemovePopup`/`RemoveNotify`/`RemoveOverlap` all removed the dictionary entry synchronously, then started an async hide animation. A fast re-show of the same type (e.g. rapid connect/disconnect/reconnect toggling `ClientManager` between `StreamingState` and `IdleState` faster than the hide duration) would find no entry and instantiate a second live instance while the first was still animating out — leaving two `StreamScreen`s alive at once, both subscribed to `CameraStreamPlayer`.

- **Fix**: all four `Remove*` methods in `Assets/Scripts/Manager/UIManager.cs` now keep the dictionary entry until the hide-out animation's completion callback actually fires (guarded so a re-show that replaces the tracked instance in the meantime doesn't get double-removed/destroyed). A same-type `Show*<T>()` called mid-hide now reuses the still-animating instance instead of duplicating it.

## Root cause 3: unescaped JSON request bodies

`RegistrationClient.GetRequestBody()` and `ViewerTokenClient.GetRequestBody()` built their POST bodies via raw string interpolation of user-controlled `AppConfig.Username`/`RoomCode`, with no JSON escaping. A username or join-room code containing `"` or `\` produced invalid JSON and broke the request. `RoomClient.JoinRoom` also only checked room-code *length* (`>= 6`), not character content, so such input could reach the request body via the join path even though generated codes are restricted to an unambiguous charset.

- **Fix**: both clients now build their bodies via `JsonUtility.ToJson` on a small private `[Serializable]` request struct instead of string interpolation — consistent with how they already parse responses. `RoomClient.JoinRoom` now validates every character of the trimmed, uppercased code against the same charset used by `GenerateCode()`, and requires an exact 6-character match rather than merely `>= 6`.

## Verification

No live Unity Editor MCP connection was available this session (`Connection revoked` — needs re-approval under **Project Settings → AI → Unity MCP**) to confirm a clean recompile. All five fixes were instead verified by a careful manual read-through of the edited files. Recommend opening the project in Unity to confirm no compile errors before committing.

## Files touched

- `Assets/Scripts/UI/Base/BaseUIElement.cs`
- `Assets/Scripts/Manager/UIManager.cs`
- `Assets/Scripts/Manager/States/AppManager.RoomEntryState.cs`
- `Assets/Scripts/UI/Screen/RoomScreen.cs`
- `Assets/Scripts/Manager/RegistrationClient.cs`
- `Assets/Scripts/Manager/ViewerTokenClient.cs`
- `Assets/Scripts/Manager/RoomClient.cs`
