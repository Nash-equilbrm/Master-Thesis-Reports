# Action Log — Sep 20th (calibration data TTL / expiry)

## Session summary

Follow-up to `action_log_Sep_12th_calibration_pipeline.md`, whose three
calibration endpoints (`/calibration-config`, `/calibration-data`,
`/calibration-data/pair`) landed in `Master-Thesis-Server` as
`a73800b [feat]: add server-side ChArUco calibration endpoints`. That
implementation stored calibration keyed only by identity (`cam1`, `cam2`, ...)
in an in-memory `Map`, with no expiry. Fixed in
`f0f8e7d [fix]: expire stale calibration data per-device instead of
retaining it forever`.

---

## The bug

`calibrationData` was overwritten wholesale per `POST /calibration-data`,
but a stale entry was never cleared otherwise. Since camera identities are
slot-based (`camN`), not tied to a specific physical device, this meant:

- A slot freed by one camera and reassigned to a different physical camera
  (per the `44988e0` slot-drift fix) could inherit the old camera's
  intrinsics/extrinsics until it re-calibrated — silently wrong until then.
- The same physical camera reconnecting hours or days later would also serve
  its old calibration, even if the camera/board setup had physically moved
  in the meantime.

## The fix — `registration-service/server.js`

- Each calibration entry now records `calibratedAt: Date.now()` on write.
- New `CALIBRATION_TTL_HOURS` env var (default `4`), converted to
  `CALIBRATION_TTL_MS`.
- `clearExpiredCalibrations()` runs on a 15-minute `setInterval`, deleting
  any entry whose TTL has elapsed — but **only if its slot is currently
  free**. A still-connected device keeps its calibration indefinitely,
  regardless of age, since it's known to still be the same physical camera.
- `getFreshCalibration(identity)` performs the same free-slot + TTL check
  lazily, used by `GET /calibration-data/pair`, so a read landing between
  two sweeps can never return calibration older than the TTL.

Net effect: expiry is per-device and driven by disconnect + age, not a bulk
clear, and never punishes a camera that's still actively streaming.

---

## Files changed

- `Master-Thesis-Server/registration-service/server.js` (+33/-5)

---

## Next steps

Per `action_log_Sep_12th_ec2_tls_verification.md`'s outstanding list:
1. Run the Aug 18th room/session test plan (two clients, create/join,
   dynamic room names, webhook still frees slots).
2. Multi-camera test (2+ camera clients) since the Client/Camera Instance
   merge.
3. Standalone Windows viewer build.
4. Live-test the calibration TTL behavior end-to-end (disconnect a camera,
   wait past `CALIBRATION_TTL_HOURS`, confirm `/calibration-data/pair`
   404s and the sweep log line appears) — done by code inspection only so
   far, not exercised against a running stack.
