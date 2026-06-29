# Session handoff — 2026-06-29 — Token reminder (token.comparegpt.io)

> **Purpose of this file:** a self-contained, reproducible spec of the work done
> in this session, written so another agent (e.g. Codex) can reproduce the same
> result from scratch. It duplicates the narrative entry in `HISTORY.md` but adds
> the exact diff, commands, and verification so nothing has to be re-derived.

---

## 1. Goal

When a user opens Claude PWM **without a PWM token set**, show a reminder that
directs them to **https://token.comparegpt.io** to get a free token first, with a
shortcut into Settings to paste it. Hide it the moment a token exists.

Keep the rest of the UI at parity with Anthropic's claude.ai. This reminder is a
**deliberate, necessary divergence** (claude.ai has no token step) — same accepted
category as the existing Settings "PWM API Key" field. Style it to match Claude.

## 2. Where the code lives (important — not in this repo)

| Thing | Path |
|-------|------|
| This repo (`claude-pwm`) | `/home/spiritai/pwm/claude-pwm` — **binary packaging + history log only**. No UI here. |
| The actual web UI | `/home/spiritai/pwm/Claude_UI_PWM` — FastAPI app, branch **`master`** |
| Files to edit | `static/index.html`, `static/css/style.css`, `static/js/app.js` |
| Running container | `claude_ui_pwm-app-1`, bound `127.0.0.1:8103 -> 8000` |
| Prod domain | `https://claude.comparegpt.io` (nginx vhost → 8103) |
| Dev / staging domain | `https://claude.platformai.org` (nginx vhost → 8103) |
| Deploy mechanism | `docker compose up -d --build` in `Claude_UI_PWM/` (static is baked into the image; cache-bust `?v=<hash>` query string auto-advances) |

## 3. Pre-flight health check (do this first)

```bash
curl -s https://claude.comparegpt.io/healthz            # -> {"status":"ok",...}
curl -s https://claude.platformai.org/healthz           # -> {"status":"ok",...}
curl -s "https://claude.comparegpt.io/healthz?deep=1"   # -> upstream reachable, status 404 (expected)
docker ps --format '{{.Names}}\t{{.Status}}' | grep claude_ui_pwm
```

## 4. The exact change

Apply this diff in `/home/spiritai/pwm/Claude_UI_PWM` (branch `master`). This is
the full content of commit `d11dd80`.

```diff
diff --git a/static/css/style.css b/static/css/style.css
--- a/static/css/style.css
+++ b/static/css/style.css
@@ after the "#greeting { ... }" rule (around line 205) @@
 #greeting { font-family: var(--font-serif); font-size: 32px; font-weight: 400; color: var(--text); letter-spacing: -.01em; margin-top: 18px; }
 
+#token-reminder {
+  display: inline-flex; align-items: flex-start; gap: 9px;
+  margin: 20px auto 0; max-width: 520px; padding: 11px 15px;
+  background: var(--raised-bg); border: 1px solid var(--border-strong);
+  border-radius: 12px; font-size: 13.5px; line-height: 1.5;
+  color: var(--text-muted); text-align: left;
+}
+#token-reminder[hidden] { display: none; }
+#token-reminder svg { width: 17px; height: 17px; color: var(--accent); flex-shrink: 0; margin-top: 1px; }
+#token-reminder a { color: var(--accent); font-weight: 600; text-decoration: none; }
+#token-reminder a:hover { text-decoration: underline; }
+.inline-link {
+  background: none; border: none; padding: 0; font: inherit;
+  color: var(--accent); font-weight: 600; cursor: pointer; text-decoration: none;
+}
+.inline-link:hover { text-decoration: underline; }
+
 #suggestions {

diff --git a/static/index.html b/static/index.html
--- a/static/index.html
+++ b/static/index.html
@@ inside <div id="hero">, right after <h1 id="greeting">…</h1> @@
           <h1 id="greeting">How can I help you today?</h1>
+          <div id="token-reminder" hidden>
+            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><path d="M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0z"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/></svg>
+            <span>To start chatting, get your free PWM token at
+              <a href="https://token.comparegpt.io" target="_blank" rel="noopener">token.comparegpt.io</a>,
+              then add it under <button type="button" id="token-reminder-settings" class="inline-link">Settings</button>.</span>
+          </div>
           <div id="suggestions"></div>
@@ in the Settings > Profile section, the PWM API Key field hint @@
-              <p class="field-hint">Get your key at <strong>physicsworldmodel.org</strong> &rarr; Wallet. Responses are billed in PWM tokens.</p>
+              <p class="field-hint">Get your free PWM token at <strong><a href="https://token.comparegpt.io" target="_blank" rel="noopener">token.comparegpt.io</a></strong>. Responses are billed in PWM tokens.</p>

diff --git a/static/js/app.js b/static/js/app.js
--- a/static/js/app.js
+++ b/static/js/app.js
@@ in the DOM-element refs block (after `const errorBanner = …`) @@
 const errorBanner       = document.getElementById('error-banner');
+const tokenReminder     = document.getElementById('token-reminder');
 const sidebar           = document.getElementById('sidebar');
@@ in the saveSettingsBtn click handler, after setGreeting(); @@
   setGreeting();
+  updateTokenReminder();
   closeSettings();
@@ at the end of startNewChat(), after loadConversations(); @@
   loadConversations();
+  updateTokenReminder();
   if (window.innerWidth <= 760) sidebar.classList.add('collapsed');
@@ right after hideError() (the "Error banner" section) @@
 function hideError()    { errorBanner.classList.remove('show'); }
 
+// ── Token reminder ──────────────────────────────────────────────
+// Until the user has a PWM token, remind them where to get one
+// (token.comparegpt.io). Shown only in the empty landing state; hidden
+// the moment a key is present so returning users never see it.
+function updateTokenReminder() { if (tokenReminder) tokenReminder.hidden = !!getApiKey(); }
+document.getElementById('token-reminder-settings')?.addEventListener('click', () => openSettings());
```

### Why it works
- The element starts `hidden` in HTML. `updateTokenReminder()` clears `hidden`
  only when `getApiKey()` is falsy (no token). `getApiKey()` reads
  `localStorage['claude_pwm_apikey']`.
- `startNewChat()` runs on page load (called from the init block at the bottom of
  `app.js`), so the reminder is evaluated on entry. It's also re-evaluated after
  Save Settings (so it vanishes the instant a key is saved).
- It lives inside `#hero`, which only renders in the empty/landing state
  (`#conversation.empty #hero { display:block }`), so it never covers an active chat.

## 5. Deploy

```bash
cd /home/spiritai/pwm/Claude_UI_PWM
docker compose up -d --build
```

## 6. Verification (all must pass)

```bash
# 1. JS parses
node --check static/js/app.js          # -> (no output) then echo OK

# 2. app healthy
curl -s http://127.0.0.1:8103/healthz   # -> {"status":"ok","service":"claude-ui-pwm"}

# 3. reminder present in served HTML on both domains
curl -s https://claude.comparegpt.io/  | grep -c token.comparegpt.io   # -> 2
curl -s https://claude.platformai.org/ | grep -c token.comparegpt.io   # -> 2

# 4. cache-bust advanced (hash will differ from f54c9bbe; just confirm it changed)
curl -s http://127.0.0.1:8103/ | grep -o 'style.css?v=[a-z0-9]*'
```

Headless render check (proves it shows with no key, hides with a key):

```python
# python3 with playwright installed (it is, system-wide)
import asyncio
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        b = await p.chromium.launch()
        pg = await b.new_page(viewport={"width":900,"height":760})
        await pg.goto("http://127.0.0.1:8103/", wait_until="networkidle")
        assert await pg.locator("#token-reminder").is_visible()      # no key -> visible
        await pg.evaluate("localStorage.setItem('claude_pwm_apikey','sk-pwm-test')")
        await pg.reload(wait_until="networkidle")
        assert not await pg.locator("#token-reminder").is_visible()  # key -> hidden
        await b.close()
        print("PASS")
asyncio.run(main())
```

Expected landing text (fresh, no key):
> To start chatting, get your free PWM token at **token.comparegpt.io**, then add it under **Settings**.

## 7. Commit & push

```bash
cd /home/spiritai/pwm/Claude_UI_PWM
git add static/index.html static/css/style.css static/js/app.js
git commit -m "feat: token reminder pointing users to token.comparegpt.io"
git push                                # remote: integritynoble/Claude_UI_PWM master
```

Then log the session in this repo (`/home/spiritai/pwm/claude-pwm`): append a dated
entry to `HISTORY.md` and `git push` (remote: `integritynoble/claude-pwm` main).

## 8. Result of this session (Claude run)

- `Claude_UI_PWM` master: **`d11dd80`** (`6d89ede..d11dd80`) — pushed.
- `claude-pwm` main: **`375ccd1`** (HISTORY.md entry) + this file — pushed.
- Live and verified on prod (`claude.comparegpt.io`) and dev (`claude.platformai.org`).
- Cache-bust hash this run: `style.css?v=f54c9bbe`.
