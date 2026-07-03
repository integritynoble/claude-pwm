# Session handoff — 2026-07-02/03 — Feedback buttons + Projects feature (claude.ai parity complete)

> **Purpose of this file:** a self-contained, reproducible record of this
> session, written so another agent can reproduce the same result from scratch.
> It duplicates the narrative entries in `HISTORY.md` but adds the exact
> commits, commands, artifacts, and verification. Unlike earlier handoffs it
> does not inline the full diff (~1,000 lines): the implementation plan named
> below contains every line of code and is itself the reproduction script.

---

## 1. Goals (two work items in one session)

1. **Thumbs up/down feedback buttons** under assistant messages — the last
   deferred item from the 2026-06-28 parity audit.
2. **Projects** — the last major missing claude.ai feature: group chats under
   named projects with per-project custom instructions and text knowledge
   injected into every project chat.

After this session, every claude.ai parity item is closed. Remaining
deliberate divergences: the PWM token flow (token.comparegpt.io), the
Settings "PWM API Key" field, the Settings button in place of an account
avatar.

## 2. Where the code lives (important — not in this repo)

| Thing | Path |
|-------|------|
| This repo (`claude-pwm`) | `/home/spiritai/pwm/claude-pwm` — binary packaging + history log only. No UI here. |
| The actual web UI | `/home/spiritai/pwm/Claude_UI_PWM` — FastAPI app, branch `master` |
| Running container | `claude_ui_pwm-app-1`, bound `127.0.0.1:8103 -> 8000` |
| Prod / dev domains | `https://claude.comparegpt.io` / `https://claude.platformai.org` (both nginx → 8103) |
| Deploy | `docker compose up -d --build` in `Claude_UI_PWM/` (static baked into image; `?v=<hash>` cache-bust auto-advances) |

## 3. Pre-flight health check (do this first)

```bash
curl -s https://claude.comparegpt.io/healthz            # {"status":"ok",...}
curl -s https://claude.platformai.org/healthz           # {"status":"ok",...}
curl -s "https://claude.comparegpt.io/healthz?deep=1"   # upstream reachable
docker ps --format '{{.Names}}\t{{.Status}}' | grep claude_ui_pwm
```

## 4. Work item 1 — feedback buttons (commit `c753df3`)

Assistant message action row became Copy · 👍 · 👎 · Retry:

- `static/js/app.js`: `ICON_THUMB_UP`/`ICON_THUMB_DOWN`; `makeFeedbackBtn()`
  creates icon-only buttons with `data-verdict="up|down"`, tooltips
  "Good response"/"Bad response"; verdict stored on the message object as
  `feedback: 'up'|'down'|null` (click again clears; selecting one deselects
  the other) and persisted via `saveCurrentChat()`; `stripForStorage` keeps
  the `feedback` field so it also syncs to the server.
- `static/css/style.css`: `.msg-action-btn.icon-only` (tighter padding),
  `.msg-action-btn.selected` (accent color, filled icon).

Verified with headless Playwright (seeded conversation): render, toggle,
switch, clear, survive reload, zero page errors. Test-writing gotcha: a
Playwright `add_init_script` that seeds localStorage unconditionally re-seeds
on `page.reload()` and destroys the state you're asserting — make the seed
conditional (`if (!localStorage.getItem(...))`).

## 5. Work item 2 — Projects

### Artifacts (all in `Claude_UI_PWM`, committed)

| Artifact | Path |
|----------|------|
| Design spec | `docs/superpowers/specs/2026-07-02-projects-feature-design.md` |
| Implementation plan (contains ALL code, 8 tasks) | `docs/superpowers/plans/2026-07-02-projects-feature.md` |
| E2E gate | `scripts/e2e_projects.py` |
| SDD ledger + task briefs/reports | `.superpowers/sdd/` (git-ignored scratch) |

### To reproduce from scratch

Execute the plan task-by-task (it contains the exact code, anchors, TDD steps,
and gates for each task), on a `feature/projects` branch, then merge. Commits
as shipped:

```
5994100 feat: conversations carry optional projectId (Projects groundwork)
ebf9e8a feat: /api/projects CRUD with knowledge; delete unlinks chats
63aaa27 test: isolate suite DB via conftest (env var was set too late per-module)
450cab2 feat: projects data layer, server sync, and system-prompt injection
1735879 feat: Projects views — sidebar entry, hash routing, grid, create/edit/delete
16353c3 feat: project detail view — chats, instructions, text knowledge with caps
2e242f8 fix: pin upload target project; don't misattribute files after mid-upload navigation
bd4170b feat: project breadcrumb in chat + add-to-project picker on sidebar chats
bbddddc test: end-to-end Playwright script for Projects
d4dd39f fix: persist chat on stream error only when it grew (regenerate/edit safe) + e2e regression step
47d9a08 fix: byte-accurate knowledge caps, merge-by-updatedAt project sync, shortcut/picker/render hardening
f625c71 Merge feature/projects: claude.ai-parity Projects feature
```

### Bugs the review loops caught (do not re-introduce these)

1. **Test-DB isolation was broken suite-wide** (pre-existing): per-module
   `os.environ["CONV_DB_PATH"]` assignments never worked because
   `test_chat.py` imports `main` first, binding `DB_PATH` to the real
   `data/conversations.db`. Fix: `tests/conftest.py` sets the env before any
   test module imports (`63aaa27`).
2. **Upload race:** the multi-file knowledge upload loop re-read the mutable
   global `viewProjectId` after each `await f.text()` — navigating to another
   project mid-upload attached later files to the wrong project. Fix: pin
   `targetProjectId` at handler entry (`2e242f8`).
3. **Error-save data loss:** persisting the chat on stream error (needed so a
   failed first send survives) would let an errored regenerate/edit overwrite
   saved history with the truncated `currentMessages`. Fix:
   `saveCurrentChatIfGrown()` — save on error only when the conversation is
   new or grew (`d4dd39f`); e2e step 7 is the regression gate.
4. **UTF-16 vs UTF-8 caps (Critical, reproduced live):** client knowledge
   caps counted `content.length` (UTF-16 code units) while the server caps
   the body at 4 MB UTF-8 bytes. CJK-heavy docs passed client checks, 413'd
   silently on sync (fire-and-forget catch), and the then-overwrite-style
   sync wiped them from localStorage on next load. Fix: `utf8Bytes()`
   (TextEncoder) for all cap checks + `syncProjectsFromServer` merges by
   `updatedAt` (union by id, newer wins, local-only kept) (`47d9a08`).

### Accepted follow-ups (open by design)

- Maxed 2 MB knowledge exceeds upstream token limits → upstream error banner;
  consider lower cap or RAG later.
- `#/project/<id>` deep link on a fresh device falls back to the grid until
  the first sync lands.
- Offline project delete can resurrect on next sync (same semantics as
  conversations).
- `tests/conftest.py` hard-codes `/tmp/test_claude_ui_pwm.db` (collides only
  under parallel CI).

## 6. Verification (all PASS as shipped)

```bash
cd /home/spiritai/pwm/Claude_UI_PWM
python3 -m pytest tests/ -q                       # 25 passed
node --check static/js/app.js                     # exit 0
docker compose up -d --build                      # deploy
python3 scripts/e2e_projects.py --base http://127.0.0.1:8103   # PROJECTS E2E PASS (live build)
curl -s http://127.0.0.1:8103/ | grep -o 'app.js?v=[a-f0-9]*'  # v=f3e83403
curl -s https://claude.comparegpt.io/ | grep -c projects-btn    # 1
```

Deploy note: `docker compose up -d --build` failed once with a container-name
conflict against a leftover container in "Created" state; the immediate retry
of the same compose command succeeded (compose recovered by itself — check
`docker ps -a` before force-removing anything).

## 7. Session log commits in this repo

- `a2c8ebd` — feedback buttons + retroactive 2026-07-01 model-alignment entry
- `aed515a` — Projects feature HISTORY entry
- this file
