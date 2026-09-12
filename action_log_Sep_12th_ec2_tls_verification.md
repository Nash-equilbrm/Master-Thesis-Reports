# Action Log — Sep 12th (EC2 network posture verification)

## Session summary

Closed out the open question flagged in `action_log_Sep_10th.md`: whether the
EC2 production instance's LiveKit config is actually set up for real
public-internet WebRTC connectivity, or silently still running LAN-only
settings pointed at a public IP. No code changes — verification only, done
by SSHing into the instance and reading the live (gitignored) config files
directly, since none of this exists in git.

---

## What was checked

The user set up an Elastic IP (`13.228.8.204`) for the EC2 instance and
pointed camera/viewer clients at it. SSH'd in as `ec2-user` (key:
`thesis-livekit.pem` — required an `icacls` permission fix on Windows first,
OpenSSH refused the key with "Bad permissions" until inheritance was
stripped and read access restricted to the local user).

On the instance, `~/streaming-server/`:

- `livekit.yaml` → `rtc.node_ip: 13.228.8.204` (Elastic IP hardcoded).
  `use_external_ip` is `false`, but that's fine — `node_ip` achieves the same
  ICE-candidate-advertisement result on its own. `api_secret` under `keys:`
  is a real generated value (`thesis-prod`), not the dev placeholder.
- `.env` → `LIVEKIT_NODE_IP=13.228.8.204` — matches, so `registration-service`
  hands clients the correct server URL.
- `docker-compose.override.yml` → only overrides
  `LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET` to the `thesis-prod` credentials.
  No reverse proxy, no TLS/cert setup.
- `docker-compose.yml` on the instance matches the repo's checked-in
  version — two services only (`livekit`, `registration`), no
  nginx/Caddy/load-balancer container anywhere in the stack.
- EC2 security group inbound rules — confirmed TCP 7880/7881 and UDP 7882
  open to the internet.

---

## Conclusion

**WebRTC connectivity is correctly configured** — the scenario the Sep 10th
report worried about (silently LAN-only config just reachable at a public
IP) is not what's actually deployed. `node_ip` + security group rules are
both right.

**Confirmed gap: no TLS.** Signaling is plain `ws://`/`http://` on the
Elastic IP — LiveKit JWTs and `/register`/`/viewer-token` HTTP bodies cross
the public internet unencrypted. WebRTC media itself is separately encrypted
by the WebRTC standard (SRTP) regardless, so this only affects the
signaling/token exchange, not the video stream content.

**Decision: leave as-is.** Adding TLS would need a reverse proxy (e.g. Caddy
with auto Let's Encrypt) in front of both the LiveKit and registration
ports — reasonable infra for a real production service, not judged worth it
for a thesis demo's risk profile. Revisit only if this deployment is ever
exposed beyond thesis demo purposes.

---

## Files changed

None in any repo — read-only verification session. Findings recorded in
this log and in the auto-memory note `deployment_public_internet.md`
(`~/.claude/projects/.../memory/`), which previously said this was
unverified.

---

## Next steps

Per `action_log_Sep_10th.md`'s list, still outstanding:
1. Run the Aug 18th room/session test plan (two clients, create/join,
   dynamic room names, webhook still frees slots).
2. Multi-camera test (2+ camera clients) since the Client/Camera Instance
   merge.
3. Standalone Windows viewer build.
4. Implement the three calibration server endpoints spec'd in
   `action_log_Sep_12th_calibration_pipeline.md`
   (`/calibration-config`, `/calibration-data`, `/calibration-data/pair`).
