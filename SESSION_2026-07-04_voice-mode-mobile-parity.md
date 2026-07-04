# Session handoff — 2026-07-04 — Voice conversation mode, mobile parity, slow-response fix

> **Purpose:** self-contained, reproducible record of this session ("focus
> voice conversation; solve the mobile match", plus the mid-session prod
> "responds so slow" report), so another agent can re-run every check and
> reproduce the features.

## 1. Repo / deploy map (unchanged)

| Thing | Path |
|-------|------|
| This repo (`claude-pwm`) | history log + CLI packaging only |
| Web UI | `/home/spiritai/pwm/Claude_UI_PWM`, branch `master` |
| Container / domains | `claude_ui_pwm-app-1` on 8103; claude.comparegpt.io (prod), claude.platformai.org (dev); deploy = `docker compose up -d --build`, then **wait for `/healthz` before gates** |

## 2. Commits this session (`Claude_UI_PWM` master, pushed `7198be8..563385e`)

```
34dacac fix: end stream loop on message_stop so held-open connections don't wedge isStreaming
9a38477 docs: voice conversation mode + mobile composer parity design spec
563385e feat: voice conversation mode + mobile composer parity (claude.ai parity)
```

Pre-existing on master from another session (docs only, NOT implemented):
`7d24e8a` account sign-in design spec, `4b21822` its implementation plan.

## 3. "Responds so slow" triage (mid-session user report)

1. Static + infra were fast: edge TTFB 0.16 s, local 3 ms, load normal.
2. Timed end-to-end chat with a temp key (mint/delete procedure:
   `SESSION_2026-07-03_fable5-diagnosis-cli-release.md` §2): local 1.4 s,
   edge ~2 s, TCP closed promptly *in those runs*.
3. The real user-facing culprit: an **uncommitted `message_stop` fix in the
   working tree** — when the exchange holds the SSE open after `message_stop`
   (intermittent), `isStreaming` stayed true for tens of seconds and
   follow-up sends were silently dropped. Deployed the fix on its own first
   (stash WIP → build → verify `v=c6e3ea03` on edge → stash pop).
4. Gotchas for future diagnostics:
   - Cloudflare 403s `Python-urllib` UAs on POST `/api/chat/stream`; use curl.
   - `pkill -f <pattern>` can match your own shell's command string — scope
     it (`fuser -k 8199/tcp` is safer).

## 4. Voice conversation mode (the session focus)

Spec: `Claude_UI_PWM/docs/superpowers/specs/2026-07-04-voice-mode-mobile-parity-design.md`.
Design: client-only Web Speech loop; server STT/TTS rejected (PWM exchange
proxies only Anthropic `/v1/messages`).

Mechanics (all in `static/js/app.js`, section "Voice conversation mode"):
- `#voice-mode-btn` → `startVoiceMode()` → full-screen `#voice-overlay`
  (markup in `index.html`, CSS section "Voice conversation overlay").
- States listening/thinking/speaking as overlay classes; orb animates
  (pulse/spin/talk); `Muted` dims the orb.
- Listening: dedicated `SpeechRecognition` (`continuous`, `interimResults`),
  live transcript; final results accumulate in `voiceFinal`; every final
  (re)arms a **1400 ms** silence timer → `voiceSendTurn()`.
- Sending: transcript → `userInputEl.value` → existing `sendMessage()` (so
  storage/sync/rendering unchanged); recognition **stopped during
  thinking/speaking** (no echo cancellation → the mic would transcribe TTS).
- Speaking: `speechText()` strips markdown/code/citations; `speechChunks()`
  splits ≤220 chars at sentence bounds; last utterance's `onend` resumes
  listening. Tap orb = interrupt (cancels TTS, `onend` handlers nulled first).
- Chrome ends recognition spontaneously → `onend` restarts while state is
  `listening` and not muted. `not-allowed` → in-overlay error, auto-exit.
- `buildSystemPrompt()` appends `<voice-mode>` brevity block only while
  `voiceModeActive`.

## 5. Mobile match

Audit method (reusable): Playwright 390×844, walk surfaces, and per surface
evaluate `document.documentElement.scrollWidth` + element rects beyond the
viewport (ignore descendants of the zero-width artifact panel — false
positives). Pre-fix: scrollWidth 453 (landing), 489 (style menu open),
chips overlapping mic/send. Post-fix: 390 everywhere.

Fix, all inside the existing `@media (max-width: 760px)` block in
`style.css`: hide `#composer-model` and the chip label `<span>`s
(`#think-toggle`, `#search-toggle`, `#style-btn` → icon-only, padding 7px,
svg 15px), gaps 6/8px, `#composer` padding 8/12/14, `#style-menu`
`max-width: calc(100vw - 56px)`.

## 6. Verification as shipped

```bash
cd /home/spiritai/pwm/Claude_UI_PWM
node --check static/js/app.js                      # OK
python3 -m pytest tests/ -q                        # 27 passed
# dev-loop trick: uvicorn main:app --port 8199 serves the working tree
python3 scripts/e2e_voice_mode.py  --base http://127.0.0.1:8103   # VOICE MODE E2E PASS
python3 scripts/e2e_projects.py    --base http://127.0.0.1:8103   # PROJECTS E2E PASS
python3 scripts/e2e_web_search.py  --base http://127.0.0.1:8103   # WEB SEARCH E2E PASS
curl -s https://claude.comparegpt.io/healthz                      # 200 both domains
# prod + dev serve app.js?v=aaf6484f, HTML contains voice-overlay
```

The voice e2e stubs `SpeechRecognition`/`speechSynthesis` in
`add_init_script` and intercepts `/api/chat/stream` — no audio hardware
needed; it asserts the `<voice-mode>` system block, the spoken reply text,
return-to-listening, and mute/end behavior.

## 7. Open follow-ups
- Voice quality is the browser's built-in TTS (below claude.ai's neural
  voices); a server-side TTS would need a new upstream + billing decision.
- No barge-in while Claude speaks (mic is off during TTS by design); tap the
  orb to interrupt.
- Account sign-in spec/plan on master remain unimplemented (other session).
- Prior accepted follow-ups from 2026-07-03 web search unchanged.
