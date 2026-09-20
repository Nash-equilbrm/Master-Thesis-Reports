# Handoff Spec — OpenDIBR: External Pose Control + Live Frame Export

## Purpose (read this first, self-contained)

`OpenDIBR` (this repo) is being wired up to a separate live multi-camera Unity
viewer app (`Master-Thesis-Client`, a different repo) that wants to drive this
renderer's virtual camera in real time and display its rendered output inside
its own UI, instead of a person using keyboard/mouse and looking at this app's
own window.

This spec covers ONLY the four changes needed inside **this** repo. It does
NOT cover: the Unity-side code, or a new Python bridge process that will feed
this app live RTSP camera+depth streams (both separate work, tracked
elsewhere). You do not need that context to do this work — just the four
items below, each independently testable.

**Process lifetime model (important context for all four items)**: this app
will be launched ONCE per Unity viewer session and stay running idle
(no camera streams loaded) until the first camera switch — NOT relaunched per
switch. Individual camera streams are added/removed at runtime via Item 4's
control channel as the user switches between cameras during the session — the
whole point of Item 4 is to make a "switch to a previously-used camera" cost
near-zero (no process restart, no window/GL re-init) and a "switch to a
never-yet-used-this-session camera" cost only the setup of that one new
stream, not the whole app.

Three previous investigation sessions already read this codebase in full to
locate the exact integration points (cited below with file:line). Do not
re-derive these from scratch — verify and build on them.

## Item 1 — Verify/enable RTSP input

**Finding**: `FFmpegDemuxer::FFmpegDemuxer(const char* szFilePath, ...)`
(`src/FFmpegDemuxer.h:61`) calls `avformat_open_input(&fmtc, szFilePath, NULL,
NULL)` directly — no scheme validation, no local-path assumption.
`szFilePath` comes from `inputCameras[i].pathColor`/`pathDepth`
(`src/Application.h:599-600`), sourced from the JSON config's
`NameColor`/`NameDepth` fields. Since FFmpeg's `avformat_open_input` natively
dispatches on URL scheme, an `rtsp://...` string in those JSON fields should
already reach FFmpeg unmodified.

**Task**: confirm this actually works end to end against a real RTSP source
(e.g. serve a test pattern via `ffmpeg -re -f lavfi -i testsrc -c:v h264 -f
rtsp rtsp://localhost:8554/test`, using any local RTSP server such as
`mediamtx`, and point a minimal input JSON's `NameColor` at it). If it doesn't
connect reliably, pass an `AVDictionary` `options` argument to
`avformat_open_input` with `rtsp_transport=tcp` and reasonable timeout options
— this is expected to be a small change if needed at all, not a redesign.

**Definition of done**: a modified/test input JSON pointing at `rtsp://`
URLs (for both color and depth) successfully decodes and renders frames,
verified visually.

## Item 2 — External pose-input channel

**Finding**: virtual camera control is pure SDL2 keyboard/mouse polling in
`PCApplication::HandleUserInput()` (`src/PCApplication.h:33-207`). WASD/Q/Z
(`:172-190`) accumulate `movement`; left-mouse-drag accumulates `rotation`
(`:113-128`); these feed `accumMovement`/`accumRotation` into
`pcOutputCamera.model`/`.view` (`:192-204`). There is no existing IPC, network,
or pose-replay mechanism anywhere in `src/` (confirmed by grep — the closest
thing, an offline batch pose-path renderer via `-p/--output_json` in
`Application::MainLoop`, `src/Application.h:353-367`, is file-based, one-shot,
and non-interactive — not usable here).

**Task**: add a local IPC listener that can set `pcOutputCamera.model`/`.view`
directly (same target the existing mouse/keyboard code writes to), polled once
per frame inside/alongside `HandleUserInput()`.

**Interface contract** (build to this exactly, so the eventual Unity-side
sender matches without needing to renegotiate):
- Transport: a localhost UDP socket, fixed port **40123**.
- Packet: 7 little-endian `float32`s = position (x, y, z) + rotation as a
  quaternion (x, y, z, w). Confirm and document which units/handedness/axis
  convention `InputCamera`/`OutputCamera`'s `Position`/`Rotation` fields
  already use internally (check `src/ioHelper.h` and wherever `Position`/
  `Rotation` are consumed into `pcOutputCamera.model`/`.view`) and convert
  incoming packets into that same convention — do not invent a new one.
- Rate: expect ~60 packets/sec while a transition is being driven externally.
- **Fallback behavior (important for your own testing)**: if no packet has
  arrived in the last 250ms, control reverts to normal keyboard/mouse input
  (i.e. don't permanently disable manual control — only override per-frame
  when a fresh external pose is actually present). This lets you test/verify
  this feature by hand (fly around normally) whenever nothing external is
  sending.

**Definition of done**: a throwaway test script (Python or a shell one-liner
using `nc -u`/similar) that sends packets moving the camera in a small circle
visibly moves the rendered view with no keyboard/mouse input, and stopping the
script for >250ms hands control back to keyboard/mouse.

## Item 3 — Live frame-output export, headless (no window)

**Finding**: the only existing pixel-readback path is
`Application::SaveCompanionWindowToYUV` (`src/Application.h:760-772`) — does
`glReadPixels` (`:763`) into a CPU buffer, writes a `.png` (via `saveImage`,
stb_image_write) or raw `.yuv` to `options.outputPath`. Only reachable from the
offline `-p output_json` batch-render branch (`Application.h:364,390`) — never
called from the normal interactive loop. Rendering itself is normal windowed
SDL2/OpenGL (`SDL_CreateWindow(..., SDL_WINDOW_OPENGL | SDL_WINDOW_SHOWN)`,
`src/Application.h:167,177`; GL context via `SDL_GL_CreateContext`, `:184`) —
no headless mode, no shared-texture/DXGI/interop export exists anywhere in
`src/` today (confirmed by grep for `headless|offscreen|shared|interop`).

**Task, part A — generalize the export**: turn this into a per-frame live
export, called from the normal interactive render loop (not just the offline
batch branch), writing into shared memory instead of a file.

**Task, part B — make it headless**: the end product never wants an OpenDIBR
window visible on screen at all (a separate Unity app is the only UI the user
ever sees). Replace the window-backed default framebuffer with an off-screen
render target (an FBO sized to the output resolution) so no visible OS window
is required for rendering, or at minimum change `SDL_WINDOW_SHOWN` to
`SDL_WINDOW_HIDDEN` at `src/Application.h:167,177` if a full off-screen-FBO
conversion turns out to be a bigger lift than expected for a first pass. Feel
free to keep a debug-only flag that re-enables the visible window for your own
manual testing while building this — just make sure the default/production
path never shows one.

**Interface contract**:
- Transport: Windows named shared memory (`CreateFileMapping`/
  `MapViewOfFile`), name **`OpenDIBR_FrameExport`**.
- Layout: a small header — `uint32 width`, `uint32 height`, `uint32 format`
  (0 = RGBA8), `uint64 frameCounter` — immediately followed by the raw pixel
  buffer (`width * height * 4` bytes for RGBA8).
- Signaling: a named Windows Event, **`OpenDIBR_FrameReady`**, set after each
  write.
- Tearing: single-buffer is acceptable for a first cut — increment
  `frameCounter` before and after the pixel copy; a reader that sees the
  counter change between its own before/after read knows the frame it read
  may be torn and should retry next frame. (Double-buffering is a reasonable
  follow-up if this proves visually necessary — not required for a first
  pass.)
- Performance: the existing `glReadPixels` call is synchronous and can stall
  the GL pipeline if called every frame. If frame rate visibly suffers, switch
  to a PBO (pixel buffer object) based asynchronous readback (read into a PBO
  this frame, map+copy the PBO from N frames ago) — flag this as a follow-up
  if the naive synchronous version is good enough to unblock the rest of the
  pipeline first.

**Definition of done**: while running against any valid existing input (an
existing sample dataset, e.g. `examples/Fan/`, is fine for this test — no
dependency on Items 1/2 to validate this item in isolation), a throwaway
reader process/script maps `OpenDIBR_FrameExport`, waits on
`OpenDIBR_FrameReady`, and dumps a few frames to PNG — confirm they look
correct (compare against the debug-visible-window flag's on-screen output, or
against `SaveCompanionWindowToYUV`'s original file-dump output, from the same
input/pose) with no OS window visible during the run by default.

## Item 4 — Dynamic camera stream add/remove + active-pair selection

**Finding**: `readInputJson` (`src/ioHelper.h:411-429`) is called exactly once,
before the `Application` is constructed, and its result (`inputCameras`) is
copied into a member field at construction (`src/Application.h:146-152`) —
there is no code path anywhere in `src/` that adds, removes, or re-parses a
camera after that point. Per-camera decoders are indexed from this fixed list
(two decoders per camera — color + depth — per earlier research into this
codebase).

**Why this is needed**: per the process lifetime model above, this app runs
for an entire Unity viewer session, potentially switching between many
different camera pairs over that time — but the camera roster on the Unity
side is dynamic (devices connect/disconnect at will, up to ~10 slots). Only
2 cameras' streams are ever actively rendered/blended at once, so this app
should never need to load every possible camera up front — it needs to load
streams for cameras on demand, the first time each one is actually selected,
and should be able to drop ones that haven't been used in a while.

**Task**: add a second local control channel (separate from Item 2's
high-frequency pose stream — this one is low-frequency, structured commands)
that can, while the app is already running:
- **`add_camera`**: open a new camera's color+depth streams (same
  `NameColor`/`NameDepth`-equivalent RTSP URLs Item 1 already knows how to
  open) plus its intrinsics/extrinsics (same shape as the existing JSON
  camera schema — `Focal`/`Principle_point`/`Position`/`Rotation`), without
  touching any other already-open camera's decoders/render state.
- **`remove_camera`**: close a specific camera's decoders/streams and free its
  render resources, identified by name, without affecting others.
- **`set_active_pair`**: given two camera names (both must already be open via
  `add_camera`), mark them as the current blend pair — i.e. which two cameras'
  depth meshes the render loop actually uses when the pose (Item 2) moves
  between them. With more than 2 cameras potentially open/warm at once (per
  the "keep a few recently-used ones warm" policy on the Unity/bridge side),
  the render loop can no longer infer the active pair implicitly — this command
  makes it explicit.

**Interface contract**:
- Transport: a localhost TCP socket, fixed port **40124**, separate from Item
  2's UDP pose port. Simple newline-delimited JSON messages in each direction
  (one JSON object per line) — e.g.
  `{"cmd":"add_camera","name":"cam3","colorUrl":"rtsp://...","depthUrl":"rtsp://...","focal":[...],"principlePoint":[...],"position":[...],"rotation":[...]}`,
  `{"cmd":"remove_camera","name":"cam3"}`,
  `{"cmd":"set_active_pair","camA":"cam3","camB":"cam5"}`.
- Reply: send back `{"ok":true}` or `{"ok":false,"error":"..."}` per command,
  so the Unity/bridge side knows when a newly-added camera's stream is
  actually decoding and ready to be selected as part of an active pair (don't
  ack until the stream has produced at least one decoded frame, so the caller
  doesn't race a `set_active_pair` against a still-connecting stream).
- Exact JSON field names are a suggestion, not a hard requirement — pick
  whatever's most natural given how `readInputJson` already shapes
  `InputCamera`, just document whatever you land on clearly since the Unity
  side will need to match it.

**Definition of done**: a throwaway test script opens the TCP control
channel, sends `add_camera` for one dataset camera, waits for `{"ok":true}`,
sends `add_camera` for a second, then `set_active_pair` between them, then
drives Item 2's pose channel between their two positions — confirm the
blended render output (via Item 3's export) changes correctly, with no
process restart at any point. Then send `remove_camera` for one and confirm
its resources are freed (no crash, no leak growth over repeated add/remove
cycles — worth a basic repeated-cycle stress check given this is meant to run
for a whole viewer session).

## Explicit non-goals for this handoff

- No Unity-side code.
- No Python bridge/RTSP-serving process — that's what will eventually feed
  Item 1's RTSP input and Item 4's `add_camera` URLs; not this repo's concern.
- No full JSON-file hot-reload (Item 4 is a live control-channel mechanism,
  not a file-watcher — the original startup JSON can stay one-shot-parsed for
  whatever cameras are known at launch, if any; everything added afterward
  goes through the Item 4 channel instead).

## Background reading (optional, for context only)

`Master-Thesis-Reports/action_log_Sep_9th.md` — how the Fan dataset demo was
last gotten running on this machine (CUDA 13.3 fixes, multi-GPU Optimus fix —
still relevant if you hit the same build/run issues).
`Master-Thesis-Reports/action_log_Sep_11th_opendibr.md` — earlier integration
research this spec builds on.
