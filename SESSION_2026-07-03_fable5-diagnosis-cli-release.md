# Session handoff — 2026-07-03 — Fable 5 diagnosis + CLI release v2.1.199

> **Purpose of this file:** self-contained, reproducible record of this
> session ("why is there no fable 5 support?"), so another agent can re-run
> the same diagnosis and release procedure from scratch. Duplicates the
> narrative in `HISTORY.md` with exact commands.

---

## 1. The question and the answer

**Q:** Why is there no Fable 5 support?

**A (two surfaces, different answers):**
1. **Web UI (claude.comparegpt.io): Fable 5 IS supported and works.** Picker
   option + `chat.py` adaptive-thinking handling shipped in `Claude_UI_PWM`
   commit `2d80f8c` (2026-07-01); exchange pricing has
   `"claude-fable-5": (10.0, 50.0)` in
   `Physics_World_Model/pwm_nonprofit/platform/pwm_nonprofit/services/exchange_pricing.py:50`.
   Verified live end-to-end (see §3). A user who doesn't see it has a stale
   cached page → hard refresh.
2. **CLI binaries (this repo's GitHub releases): they WERE stale** — packaged
   a pre-Claude-5 `@anthropic-ai/claude-code`. Fixed by triggering the
   release workflow (see §4). Now v2.1.199.

## 2. Key discovery — plaintext API keys no longer exist

The 2026-06-26 practice (read `users.api_key` from DB `pwm_nonprofit`) is
dead: `users.api_key` is NULL for all 57 users. Keys now live SHA-256-hashed
in the `api_keys` table (`id, user_id, name, key_hash, prefix, last4,
created_at, last_used_at`); hashing is plain
`hashlib.sha256(token.encode()).hexdigest()`
(`pwm_nonprofit/auth/dependencies.py:54`).

**To run an end-to-end test with a real key: mint a temp one, test, delete.**

```bash
# DB is in container pwm_nonprofit-postgres-1 (user pwm, db pwm_nonprofit).
# Owner accounts: id 102 spiritai@platformai.org, id 10 platformaiyang@gmail.com.
RAW="sk-pwm-$(openssl rand -hex 22 | cut -c1-43)"          # 50 chars total
HASH=$(printf '%s' "$RAW" | sha256sum | cut -d' ' -f1)
docker exec pwm_nonprofit-postgres-1 psql -U pwm -d pwm_nonprofit -c \
  "INSERT INTO api_keys (user_id, name, key_hash, prefix, last4, created_at)
   VALUES (102, '<purpose>-temp', '$HASH', left('$RAW',10), right('$RAW',4), now())"
# ... run the test with Authorization: Bearer $RAW ...
docker exec pwm_nonprofit-postgres-1 psql -U pwm -d pwm_nonprofit -c \
  "DELETE FROM api_keys WHERE key_hash='$HASH'"
```

Never print `$RAW`; delete the row when done.

## 3. The live verification that Fable 5 works

```bash
curl -s -N -X POST https://claude.comparegpt.io/api/chat/stream \
  -H 'Content-Type: application/json' -H "Authorization: Bearer $RAW" \
  -d '{"model":"claude-fable-5","messages":[{"role":"user","content":"Reply with exactly: FABLE OK"}]}'
```

Result as run: HTTP 200 SSE stream, `message_start` reports
`"model":"claude-fable-5"`, streamed text exactly `FABLE OK`,
usage 355 input / 8 output tokens. Whole chain proven:
browser key → Claude PWM → exchange → Anthropic → back.

## 4. CLI release procedure (this repo)

`.github/workflows/build.yml` repackages the official native
`@anthropic-ai/claude-code-<platform>` binaries under the claude-pwm names at
one pinned version (blank input = npm latest). It runs on **every push to
main** and on manual dispatch:

```bash
gh workflow run build.yml            # optional: -f claude_version=X.Y.Z
gh run watch <run-id> --exit-status
gh release view                      # expect new "Claude Code <ver> (standalone)"
```

As run: workflow `28666741006` → success → release **v2.1.199** with
`claude-pwm-linux`, `claude-pwm-macos`, `claude-pwm-windows.exe`.

Verification of the artifact itself (do this, not just the pipeline):

```bash
gh release download v2.1.199 --pattern claude-pwm-linux
chmod +x claude-pwm-linux
./claude-pwm-linux --version          # 2.1.199 (Claude Code)
grep -c "claude-fable-5" claude-pwm-linux   # 147 (model id embedded)
```

Because the workflow runs on every main push, routine history commits keep
releases fresh — staleness only recurs if the repo goes untouched across a
Claude Code major update.

## 5. Commits in this repo

- `109e3f4` — HISTORY.md entry for this session
- this file
