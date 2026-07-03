# Session handoff — 2026-07-03 — Parity audit #3, Styles picker, Web search

> **Purpose of this file:** self-contained, reproducible record of this
> session ("ensure Claude PWM has the same features as Anthropic's Claude",
> then "implement web search"), so another agent can re-run the audit and
> reproduce both features. Duplicates the `HISTORY.md` narratives with exact
> commands, commits, and artifacts. The implementation plans named below
> contain every line of code and are the reproduction scripts.

---

## 1. Repo / deploy map (unchanged)

| Thing | Path |
|-------|------|
| This repo (`claude-pwm`) | history log + CLI packaging only |
| Web UI | `/home/spiritai/pwm/Claude_UI_PWM`, branch `master` |
| Exchange (read-only this session) | `/home/spiritai/pwm/Physics_World_Model/pwm_nonprofit/platform/pwm_nonprofit/routers/exchange_proxy.py` |
| Container / domains / deploy | `claude_ui_pwm-app-1` on 8103; claude.comparegpt.io (prod), claude.platformai.org (dev); `docker compose up -d --build` |

## 2. Parity audit #3 (do this to re-audit)

```bash
curl -s https://claude.comparegpt.io/healthz              # 200
curl -s "https://claude.comparegpt.io/healthz?deep=1"     # upstream reachable
H=$(curl -s https://claude.comparegpt.io/)
echo "$H" | grep -c "projects-btn\|model-picker\|voice-btn\|think-toggle\|token-reminder"  # feature markers
```

Result: all shipped surfaces live. Gaps vs claude.ai identified: **Styles**
(built), **web search** (built), and heavyweight subsystems (code
execution/analysis, file creation, Research, Memory, Connectors — deferred,
each a deliberate multi-session build).

## 3. Work item 1 — Styles picker (commit `fd0adc3`)

Spec: `Claude_UI_PWM/docs/superpowers/specs/2026-07-03-styles-feature-design.md`
(implemented inline, small enough to skip the subagent pipeline).

- Composer "Style" chip beside Think → upward popover: Normal / Concise /
  Explanatory / Formal presets + custom styles + "Create & edit styles…"
  modal (name ≤ 40, prompt ≤ 1500; creating auto-selects; deleting the
  active style resets to Normal).
- Injection: `buildSystemPrompt()` adds `<response-style-preset>` after user
  custom instructions, before language/project context.
- Storage: `claude_pwm_style` (active id) + `claude_pwm_styles` (list),
  device-local. Zero backend changes.
- Verified: Playwright (menu contents, captured `/api/chat/stream` `system`
  contains preset text and custom style text, reload persistence,
  delete-resets) — PASS against the live production build.

## 4. Work item 2 — Web search (merge `7198be8`)

Spec: `docs/superpowers/specs/2026-07-03-web-search-design.md`
Plan (all code, 4 tasks): `docs/superpowers/plans/2026-07-03-web-search.md`
Executed subagent-driven on `feature/web-search`.

### The load-bearing discovery
`exchange_proxy.py::_proxy_and_settle` forwards Anthropic request bodies
**as-is** (only strips the `[1m]` model tag and prepends the Claude-Code
system block for subscription OAuth creds) and streams raw SSE back. So
Anthropic's server-side `web_search_20250305` tool works through the
exchange with **zero exchange changes**. Feasibility was proven live BEFORE
design approval:

```bash
# temp key procedure: see SESSION_2026-07-03_fable5-diagnosis-cli-release.md §2
curl -s -N -X POST https://physicsworldmodel.org/api/v1/exchange/anthropic/v1/messages \
  -H 'Content-Type: application/json' -H "Authorization: Bearer $RAW" \
  -H 'anthropic-version: 2023-06-01' \
  -d '{"model":"claude-sonnet-5","max_tokens":1024,"stream":true,
       "tools":[{"type":"web_search_20250305","name":"web_search","max_uses":2}],
       "messages":[{"role":"user","content":"What is the current UTC date? Search."}]}'
# → server_tool_use, input_json_delta, web_search_tool_result (web_search_result items),
#   web_search_result_location citations — all pass through.
```

### Branch commits
```
90c2af3 feat: web_search flag adds Anthropic server-side search tool
3afd534 feat: composer web-search toggle sends web_search flag
ae224ba feat: render web-search chips, citations, and Sources section
e0a85c7 fix: skip non-http(s) source URLs in Sources section
6d46002 test: e2e gate for web search UI
7198be8 (merge to master)
```

### Review catches worth remembering
- `javascript:` URI in a search-result URL would have become a clickable
  source link — http(s)-only scheme guard added (`e0a85c7`).
- Dark-mode `.active` background override initially missed the new toggle.
- Final review verified two cross-task seams per-task reviews can't see:
  saved `searches`/`sources` round-trip through server sync verbatim, and
  `sanitizeForApi` strips them from upstream requests (no Anthropic 400s on
  multi-turn after a search reply).

### Verification as shipped
```bash
cd /home/spiritai/pwm/Claude_UI_PWM
python3 -m pytest tests/ -q                         # 27 passed
python3 scripts/e2e_web_search.py --base http://127.0.0.1:8103   # WEB SEARCH E2E PASS
python3 scripts/e2e_projects.py  --base http://127.0.0.1:8103   # PROJECTS E2E PASS
# live search through prod with temp key → 7 web_search_result + 2 citations
```
Cache-bust `app.js?v=6034a6d5`.

**Deploy gotcha:** run gates only after `/healthz` responds — the first
post-deploy check hit a still-starting container (empty local healthz, edge
502, one flaky e2e capture); a retry loop on `/healthz` fixed it.

## 5. Accepted follow-ups (open)
- Share page + Markdown export omit Sources (citation `[n]` markers dangle).
- Zero-result search shows "Search unavailable" (indistinguishable from
  error in the current handler).
- Sources numbering counts uncited results.
- Remaining claude.ai subsystem gaps: code execution/analysis tool, file
  creation, Research mode, Memory, Connectors/MCP.

## 6. Commits in this repo
- `cbaaa31` — HISTORY: audit #3 + Styles
- `160e64f` — HISTORY: web search
- this file
