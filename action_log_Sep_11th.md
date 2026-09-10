# Action Log — Sep 11th

## Session Summary

Camera slot-drift fix, both halves, implemented and live-tested end-to-end.
`RegistrationClient.cs` now sends `deviceId` in the `/register` request body
(client-side, done earlier this session — see below). `registration-service/
server.js` now implements device-aware slot reclaim. The original plan's
`sid`-based guard turned out to still have a bug — caught by an actual `lk`
CLI reconnect-race test, not just code review — and was replaced with a
generation-counter guard. Both the bug and the fix are reproduced with real
LiveKit participants and logged evidence below.

---

## What was done (client — `Master Thesis Client`)

`RegistrationClient.cs` `RegistrationRequest` struct: added `deviceId` field,
populated with `AppConfig.UserId`.

`AppConfig.UserId` is already a persistent UUID generated once on first launch
and stored in `PlayerPrefs` — exactly the device-stable identifier the slot-drift
fix needs. Using the same value for both `userId` and `deviceId` is intentional:
for this project one device = one user, and reusing the existing persistent UUID
avoids adding a second `PlayerPrefs` key with identical semantics. The explicit
`deviceId` field makes the server-side contract unambiguous without introducing
new state.

**File changed:**
- `Assets/Scripts/Manager/RegistrationClient.cs` — added `deviceId` to
  `RegistrationRequest` struct and to `GetRequestBody()`

---

## What the server session needs to do (`streaming-server`)

### Background — the slot-drift problem

When a camera app crashes/force-quits, LiveKit waits ~15–20 s for the WebRTC/ICE
timeout before firing the `participant_left` webhook. If the app restarts within
that window and calls `/register` again, its old slot still looks occupied in the
`slots` map (webhook hasn't arrived yet), so the server assigns the next free slot
instead — `cam2` instead of `cam1`. The viewer then shows both slots as active,
the old one frozen, until the webhook eventually frees it.

### Required changes to `registration-service/server.js`

#### 1. Add a `deviceSlots` map

```js
// deviceId → last assigned slot. Never cleared — persists across reconnects.
const deviceSlots = new Map();
```

#### 2. Replace `assignSlot()` with a device-aware version

```js
function assignSlot(deviceId) {
  if (deviceId) {
    const preferred = deviceSlots.get(deviceId);
    if (preferred) {
      const occupant = slots.get(preferred);
      if (!occupant) {
        // Slot freed normally — hand it back.
        return preferred;
      }
      if (occupant.deviceId === deviceId) {
        // Same device reconnecting before webhook fired — force-reclaim.
        // The new /register call is proof the old session is dead.
        console.log(`[registration] force-reclaiming ${preferred} from stale session (reconnect race)`);
        return preferred;
      }
      // Preferred slot held by a different device — fall through.
    }
  }
  // First free slot fallback.
  for (const [identity, occupant] of slots) {
    if (!occupant) return identity;
  }
  return null;
}
```

#### 3. Update `/register` to pass `deviceId` and record the assignment

```js
app.post('/register', async (req, res) => {
  const { roomCode, userId, deviceId } = req.body ?? {};
  const room = roomCode?.trim() || ROOM_NAME;

  const identity = assignSlot(deviceId);
  if (!identity) {
    res.status(503).json({ error: 'no slots available' });
    return;
  }

  slots.set(identity, { registeredAt: Date.now(), deviceId });
  if (deviceId) deviceSlots.set(deviceId, identity);

  // ... rest of the handler unchanged (mintToken, res.json, etc.)
```

#### 4. `freeSlotByIdentity` — no changes needed

Keep `deviceSlots` intact when a slot is freed. The whole point is that the
preference survives across disconnect/reconnect cycles.

### Correctness notes

- **Force-reclaim safety:** the old WebRTC session times out naturally on the
  LiveKit side. The webhook will still arrive eventually and call
  `freeSlotByIdentity` — but by then `slots.get(preferred).deviceId` will already
  be the new session's deviceId (same device), so `freeSlotByIdentity` will only
  fire if the *new* session has also since disconnected. No double-free.
- **No starvation:** if a device never comes back, its slot is freed by the webhook
  as normal and becomes available for any other device.
- **Unknown deviceId (older clients):** `assignSlot(undefined)` falls straight
  through to first-free-slot — backwards-compatible with any client that doesn't
  send `deviceId`.

### After making the changes

Rebuild and redeploy:
```
docker compose up -d --build
```

Then verify with the reconnect race manually:
1. Start a camera client → confirm it gets `cam1`
2. Force-kill the app (don't use the back button)
3. Restart it immediately (within ~5 s, before the webhook fires)
4. Confirm server log shows `force-reclaiming cam1 from stale session (reconnect race)`
5. Confirm viewer sees `cam1` go briefly offline then come back — no `cam2` ghost slot

---

## Deviation 1 (code review, before implementing): the plan's "no double-free" note didn't hold up

Reviewed the plan above against the actual race it's meant to fix, before
writing any code:

1. Session A (`cam1`) crashes; LiveKit hasn't fired `participant_left` yet.
2. Session B re-registers within the window, force-reclaims `cam1`, connects to
   LiveKit with the same identity `cam1`. LiveKit's duplicate-identity handling
   evicts session A — which now fires `participant_left` for identity `cam1`,
   but *after* session B already took the slot.
3. `freeSlotByIdentity` as originally planned only checks `slots.has(identity)`
   — it can't tell "session A's stale leave event" from "session B's leave
   event," since both share the same identity string. It would free `cam1`
   while session B is still actively streaming on it, and a third camera could
   then be handed the same slot — an active collision, worse than the original
   drift bug (which only orphaned a slot, never double-assigned a live one).

**First fix attempt:** track the LiveKit participant `sid` (unique per
connection) alongside each slot occupant, via a new `participant_joined`
webhook handler, and only free on `participant_left` if the event's `sid`
matches what's on record.

---

## Deviation 2 (caught by a live test, not code review): the `sid`-guard itself had a gap

Rebuilt (`docker compose up -d --build`) and ran the reconnect-race test for
real, using `lk room join --publish-demo` as a stand-in for the Unity camera
app (see prior turn for why: this needs genuine LiveKit `participant_joined`/
`participant_left` webhook events with real signed payloads, which can't be
hand-crafted). Sequence: `/register` → `lk room join --identity cam1`
(session A) → `kill -9` the `lk` process → immediately `/register` again with
the same `deviceId` (session B) → `lk room join --identity cam1` again before
session A's ICE timeout fires.

**Result: the `sid`-guard did not work.** Log evidence:

```
[registration] force-reclaiming cam1 from stale session (reconnect race)
[registration] assigned cam1 → room "studio"
...
[registration] freed cam1                          ← wrong: session B is live
```

Confirmed the damage was real, not cosmetic — a third, unrelated `/register`
call for a throwaway `deviceId` was immediately handed `cam1` while session B
was still actively streaming on it:
```
curl .../register -d '{"deviceId":"UNRELATED"}'  →  {"identity":"cam1", ...}
```

**Root cause:** `/register` replaces the slot's occupant object wholesale
(`slots.set(identity, { registeredAt, deviceId })`), which has no `sid` yet —
`sid` is only known once the new session actually finishes its LiveKit
handshake, which can take over a second after `/register` already returned.
Session A's stale `participant_left` webhook landed in exactly that gap:
`occupant.sid` was `undefined`, the guard's mismatch check never triggered,
and it freed the slot. Then session B's real `participant_joined` webhook
arrived to find `occupant` already `null` and silently no-opped — the slot
stayed incorrectly marked free indefinitely, with session B still live on it.
A `sid` only exists *after* a session connects, but the race lives in the gap
*before* that — comparing on `sid` couldn't possibly close it.

**Actual fix:** a monotonic generation counter per identity, independent of
the replaceable occupant object.
- `slotGeneration: Map<identity, number>` — bumped on every successful
  `/register` for that identity (`nextGeneration()`), regardless of whether
  it's a fresh assignment or a force-reclaim. Exists from the moment
  `/register` succeeds — no connect-completion gap.
- `sessionGeneration: Map<sid, generation>` — populated on
  `participant_joined` (recording which generation was current when that
  specific connection came up), consumed on `participant_left`.
- `freeSlotByIdentity(identity, generation)` only frees if
  `occupant.generation === generation`; a mismatch is logged and ignored.
  `/unregister` still calls it with no `generation` arg → always frees
  unconditionally (unchanged, deliberate client action).

**Re-ran the identical test against the fix.** Full log sequence, confirming
correct behavior at every step:
```
[registration] participant_joined cam1 sid=PA_6nuFNwCgoPiU generation=1      (session A joins)
[registration] force-reclaiming cam1 from stale session (reconnect race)     (re-register)
[registration] assigned cam1 → room "studio"                                 (generation bumped to 2)
[registration] ignoring stale participant_left for cam1 (generation 1 != current 2)   ← A's stale leave correctly ignored
[registration] participant_joined cam1 sid=PA_xKjYocuiLoKd generation=2      (session B joins)
```
Then confirmed a fresh `/register` for an unrelated device correctly got
`cam2`, not `cam1` — no collision. Then killed session B for real (no
re-register racing it this time) and confirmed normal cleanup still works:
`[registration] freed cam1` fires with no stale-guard warning, since
generation 2 leaving matches generation 2 current.

**Files changed:**
- `registration-service/server.js` — `deviceSlots` map, device-aware
  `assignSlot(deviceId)`, `slotGeneration`/`sessionGeneration` maps,
  generation-guarded `freeSlotByIdentity(identity, generation)`,
  `participant_joined` webhook handler, `/register` now reads/records
  `deviceId` and stamps each occupant with a generation
- Verified with `node --check server.js` **and** live-tested against a real
  LiveKit server + real `lk` CLI participants (not just syntax-checked)

---

## Committed

- `streaming-server` `44988e0` — `[fix]: prevent camera slot drift and
  double-assignment on reconnect` (the generation-counter version above, not
  the intermediate `sid`-guard)
- `Master Thesis Client` `29c2731` — `[feat]: send deviceId in /register for
  camera slot-drift fix` (`RegistrationClient.cs` only — the repo had several
  unrelated Unity-generated project-file diffs sitting in the working tree,
  left uncommitted/untouched)
- Both local only, not pushed to origin as of this report.

---

## Next steps

1. Multi-camera test (2+ camera clients simultaneously) — not yet run.
2. Standalone Windows viewer build.
3. Verify EC2 instance `livekit.yaml` — confirm `use_external_ip`/STUN/TURN/TLS
   posture (still unverified as of Sep 10th).
4. Optional cleanup: `sessionGeneration` currently grows by one entry per
   LiveKit session ever seen and is never pruned. Fine at this project's
   scale (in-memory, resets on container restart, ≤10 cam slots), but worth a
   note if this service is ever pointed at a longer-lived deployment.
