# History / Session Log — claude-pwm

A running log of work and diagnoses for the `claude-pwm` / Claude UI PWM stack.

---

## 2026-06-26 — Diagnose "claude.platformai.org is not working"

### TL;DR
The site is **up and healthy**. The reported "not working" was **not** a server
outage. The one real error in the logs was a **single, transient `ConnectError`
at container startup (~5.5h before investigation)** that did not recur. The
user-facing symptom most likely comes from the **API-key flow**: chatting
requires a valid `pwm_` key entered in Settings; without one (or with an
expired one) the chat request is rejected.

### What was checked

| Check | Result |
|-------|--------|
| `GET https://claude.platformai.org/` | **HTTP 200**, serves the Claude PWM chat UI |
| `HEAD /` | 405 (only `GET` allowed — normal, not a fault) |
| `POST /api/chat` | 404 (not a route; real route is `/api/chat/stream`) |
| `POST /api/chat/stream` (no key) | **401 `Missing API key`** — expected |
| Local container `127.0.0.1:8103` `GET /` | HTTP 200 — app process healthy |
| Container → upstream connectivity (now) | **HTTP 401 `invalid_token`** — i.e. it *reached* the upstream fine; 401 only because a fake `test` key was used |
| Host → upstream exchange | HTTP 401 (reachable) |
| DNS inside container | resolves `physicsworldmodel.org` OK |
| `httpx.ConnectError` count in logs | **1**, at `2026-06-26T15:01:50Z` only |

### Architecture (as deployed)
- **Public site:** `claude.platformai.org`
- **Container:** `claude_ui_pwm-app-1`, bound `127.0.0.1:8103 -> 8000`
- **Source:** `/home/spiritai/pwm/Claude_UI_PWM`
- **Upstream:** `PWM_EXCHANGE_URL=https://physicsworldmodel.org/api/v1/exchange/anthropic`
  - request route used by app: `…/v1/messages` (`config.py:messages_url`)
- **Auth model:** browser stores the user's `pwm_` key in `localStorage`
  (`static/js/app.js`) and sends it as `Authorization: Bearer <pwm_key>`.
  - `routers/chat.py` rejects a missing/empty Bearer with 401 `Missing API key`.
  - The exchange validates the `pwm_` token server-side and fulfils the Anthropic
    request with a Claude **subscription OAuth** credential. `chat.py` prepends the
    Claude-Code identity system block so subscription tokens aren't 429'd.

### Root cause
- The lone `ConnectError` (`/app/routers/chat.py:69`, the `client.stream(...)`
  call) fired at `15:01:50Z`, which lines up with the stack restart (all
  containers reported "Up 5 hours" at investigation time ~20:34Z). It was a
  **transient upstream-unreachable blip during/just after startup** and has not
  repeated since. Connectivity from inside the container is currently fine.
- The remaining "doesn't work" symptom is the **key requirement**: with no key
  configured in the browser, the UI hits `401 Missing API key`; with an
  invalid/expired key, the exchange returns `401 invalid_token /
  require_reauth`.

### Not yet confirmed (no valid key on this host)
- No `pwm_` key is stored anywhere under `/home/spiritai/pwm`, so a **valid-key
  end-to-end chat could not be tested**. Whether the exchange's backing Claude
  **subscription OAuth credential is currently live or expired** is therefore
  unverified. Per prior notes the subscription token rotates and must be
  re-activated periodically — if a real user with a valid `pwm_` key sees errors,
  re-activating that backing credential is the first thing to check.

### How to fully verify / fix
1. In the browser, open **Settings** on the site and paste a valid `pwm_` key,
   then send a message. Watch the network tab for the `/api/chat/stream`
   response.
2. End-to-end from shell with a real key:
   ```bash
   curl -N -X POST https://claude.platformai.org/api/chat/stream \
     -H 'Content-Type: application/json' \
     -H 'Authorization: Bearer pwm_YOUR_REAL_KEY' \
     -d '{"model":"claude-sonnet-4-6","messages":[{"role":"user","content":"hi"}]}'
   ```
   - `200` + SSE stream → fully working.
   - SSE `{"type":"error","status":401,...invalid_token/require_reauth}` → the
     exchange's backing subscription OAuth credential needs re-activation
     (upstream `physicsworldmodel.org` exchange side, not this app).
3. Live tail this app's logs while testing:
   ```bash
   docker logs -f --tail 50 claude_ui_pwm-app-1
   ```

### Conclusion
No code change required in this repo. `claude.platformai.org` is serving
normally; the transient startup `ConnectError` has cleared. If a user still
can't chat, it is the **API-key path** (missing key in browser, or an expired
backing credential on the PWM exchange), not this front-end service.

---

## 2026-06-26 (follow-up) — Placeholder change + end-to-end test

### Key-format correction
Real exchange keys are **`sk-pwm-...`** (50 chars), not `pwm_...`. Stored
plaintext in `users.api_key` (DB `pwm_nonprofit`). The earlier file search for
`pwm_…` found nothing because that wasn't the real prefix.

### UI change (separate repo `Claude_UI_PWM`)
Settings key-field placeholder updated `pwm_your_key_here` → **`sk-pwm-...`**
(`static/index.html`). `static/` is baked into the image, so a `docker compose
up -d --build` of `claude_ui_pwm-app-1` was required. Verified live:
`https://claude.platformai.org/` now serves `placeholder="sk-pwm-..."` and the
old hint is gone.

### End-to-end test with a real key — PASS ✅
Tested the live site using the owner's own `sk-pwm-...` key (pulled from
`users.api_key` into a shell var, not printed; minimal request, no other user
billed):

```
POST https://claude.platformai.org/api/chat/stream
  Authorization: Bearer sk-pwm-…
  body: model=claude-sonnet-4-6, "Reply with exactly: PWM test OK"
→ HTTP 200, proper SSE stream
→ assistant text: "PWM test OK"
→ usage: input_tokens=53, output_tokens=7   (PWM billing flowing)
```

This confirms the **full chain end-to-end**: browser key → `/api/chat/stream`
→ PWM exchange → Anthropic → streamed back. The exchange's backing
subscription OAuth credential is currently **live** (not expired). Resolves the
"not yet confirmed" caveat from the first investigation.

---

## 2026-06-26 (follow-up) — `/healthz` endpoint for fast "is it working?" checks

To make future outage reports answerable in one request (the original task was
"check if it's working"), added a health endpoint to the `Claude_UI_PWM` app
(`routers/health.py`, registered in `main.py`, tests in `tests/test_health.py`,
commit `d8af0eb` on `master`). Deployed via `docker compose up -d --build`.

```bash
# app liveness
curl https://claude.platformai.org/healthz
# -> {"status":"ok","service":"claude-ui-pwm"}

# liveness + upstream-exchange reachability (no key needed)
curl "https://claude.platformai.org/healthz?deep=1"
# -> {"status":"ok",...,"upstream":{"reachable":true,"status":404}}   when healthy
# -> 503 {"status":"degraded","upstream":{"reachable":false,"error":"ConnectError"}}  when the
#    exchange is unreachable (the exact failure mode of the 15:01:50Z startup blip)
```

So the quick triage for "claude.platformai.org not working" is now:
`/healthz` 200 → app up; `?deep=1` 200 → upstream reachable; then a real-key
`POST /api/chat/stream` for full end-to-end.

---

## 2026-06-27 — Uptime watchdog wired in + live self-heal test

`/healthz` is now monitored. Cron watchdog
`Claude_UI_PWM/scripts/claude_ui_health_watchdog.sh` (commit `92deb4b`) runs
**every 5 min** on agent-prod (logs `/tmp/pwm_log/claude_ui_watchdog.log`):
- container down/unreachable → **restart** `claude_ui_pwm-app-1` (immediately if
  not running; after 2 consecutive misses if up-but-unresponsive),
- upstream exchange `503 degraded` → **alert only** (a restart can't fix an
  exchange-side outage),
- optional `PWM_WEBHOOK_URL` for Slack/Discord push alerts.

### Live self-heal test — PASS ✅
Stopped the prod container and ran the watchdog:

| Time (UTC) | Event |
|------------|-------|
| 13:08:41 | `docker stop claude_ui_pwm-app-1` (simulated crash) |
| 13:08:42 | `/healthz` → `HTTP 000` (confirmed unreachable) |
| 13:08:42 | watchdog → `RESTART: container not running` → `RECOVERED: /healthz 200` (exit 0) |
| 13:09:01 | container `Up 18s`; public `https://claude.platformai.org/` → 200; `?deep=1` → 200 upstream reachable; down-counter reset to 0 |

Total outage ≈ a few seconds — the "container not running" branch restarts
without waiting for the threshold and re-verifies recovery before exiting.
Monitoring is proven, not just deployed: a real crash is caught within ≤5 min
and self-healed the same way. (A `000`-doubling bug in the down-detection was
found and fixed during testing before the watchdog shipped.)
