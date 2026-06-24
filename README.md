# claude-pwm

Standalone Claude Code binaries — no Node.js required.

Packages the official `@anthropic-ai/claude-code` CLI with Node.js bundled in,
so any machine can run it with a single download.

Claude Code natively supports `ANTHROPIC_BASE_URL`, so no source patches are needed —
this repo just automates the bundling.

## Install

### macOS
```bash
curl -L https://github.com/integritynoble/claude-pwm/releases/latest/download/claude-pwm-macos \
  -o /usr/local/bin/claude-pwm && chmod +x /usr/local/bin/claude-pwm
```

### Linux
```bash
curl -L https://github.com/integritynoble/claude-pwm/releases/latest/download/claude-pwm-linux \
  -o /usr/local/bin/claude-pwm && chmod +x /usr/local/bin/claude-pwm
```

### Windows
Download `claude-pwm-windows.exe` from [Releases](https://github.com/integritynoble/claude-pwm/releases/latest).

## Use with PWM exchange

```bash
export ANTHROPIC_BASE_URL=https://physicsworldmodel.org/api/v1/exchange/anthropic
export ANTHROPIC_AUTH_TOKEN=pwm_your_key_here
claude-pwm
```
