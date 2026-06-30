# Connect Any Third-Party LLM Gateway to Hermes Agent

[![GitHub stars](https://img.shields.io/github/stars/roytj888/connect-any-llm-gateway-to-hermes?style=social)](https://github.com/roytj888/connect-any-llm-gateway-to-hermes/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/roytj888/connect-any-llm-gateway-to-hermes/pulls)
[![Tested on](https://img.shields.io/badge/tested%20on-Windows%2010-blue)](#limitations)

**[中文版 README →](README.zh.md)**

---

> ⭐ If this saved you time, please star this repo — it helps others find it when they hit the same errors.

---


> You bought access to a third-party AI gateway. You installed Hermes Agent. You pasted in the URL and API key.  
> Then it broke. And the error messages told you nothing useful.  
>
> This guide documents every error, every root cause, and every fix — so you don't spend hours on what took me hours to debug.

---

## What This Is

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is an open-source AI agent by NousResearch. It works great with official providers (Anthropic, OpenAI, etc.) but connecting a **third-party API gateway** (relay / 中转站) hits 3 non-obvious problems that aren't documented anywhere:

| # | Symptom | Root Cause |
|---|---------|-----------|
| 1 | `401 ApiKey Validate fail` | Hermes auto-detects URLs ending in `/anthropic` and switches to Anthropic SDK auth — which third-party gateways don't support |
| 2 | `empty stream with no finish_reason` | Most gateways fake streaming: they return plain JSON for `stream: true` instead of real SSE chunks |
| 3 | Config reverts on restart | Hermes Desktop overwrites `config.yaml` on exit if the app is running when you edit it |

This repo documents all three, plus a **ready-to-install Hermes Skill** that automates the whole setup.

---

## Option A — Use the Hermes Skill (Recommended)

Install the skill and let Hermes configure itself:

```bash
hermes skills install https://raw.githubusercontent.com/roytj888/connect-any-llm-gateway-to-hermes/main/llm-gateway-connect.skill.md
```

Then in Hermes:

```
Help me connect my third-party gateway:
URL: https://your-gateway.com/v2/gws/abc123
API Key: your-api-key
Model: your-model-name
```

Hermes will run through the checklist, test your gateway, apply the right config, and patch the code if needed.

---

## Option B — Manual Setup

### Step 1: Find your gateway root URL

If your URL looks like:
```
https://gateway.example.com/v2/gws/abc123/anthropic
```
Use only the **root path** — drop `/anthropic` and anything after:
```
https://gateway.example.com/v2/gws/abc123
```

### Step 2: Edit `config.yaml`

- Windows: `%LOCALAPPDATA%\hermes\config.yaml`
- macOS / Linux: `~/.hermes/config.yaml`

> ⚠️ **Quit Hermes Desktop completely before editing.** Right-click tray → Exit, or kill `Hermes.exe` in Task Manager. If the app is running, it will overwrite your changes on exit.

```yaml
model:
  default: your-model-name
  provider: custom:my-gateway
  base_url: https://gateway.example.com/v2/gws/abc123

providers:
  my-gateway:
    api: https://gateway.example.com/v2/gws/abc123
    transport: chat_completions      # force OpenAI-compatible mode
    api_key: your-api-key
    default_model: your-model-name

display:
  streaming: false

streaming:
  enabled: false
```

### Step 3: Check if your gateway supports real streaming

```bash
curl https://your-gateway-root/v1/chat/completions \
  -H "Authorization: Bearer your-api-key" \
  -H "content-type: application/json" \
  -d '{"model":"your-model","max_tokens":20,"messages":[{"role":"user","content":"hi"}],"stream":true}'
```

**Real SSE (no patch needed):**
```
data: {"choices":[{"delta":{"content":"Hi"}}]}
data: [DONE]
```

**Fake streaming (patch required):**
```json
{"id":"...","choices":[{"message":{"content":"Hi!"}}],"finish_reason":"stop"}
```

### Step 4: Patch Hermes (only if fake streaming)

File:
- Windows: `%LOCALAPPDATA%\hermes\hermes-agent\agent\chat_completion_helpers.py`
- macOS / Linux: `~/.hermes/hermes-agent/agent/chat_completion_helpers.py`

Find `interruptible_streaming_api_call` and add right after the docstring:

```python
def interruptible_streaming_api_call(agent, api_kwargs: dict, *, on_first_delta=None):
    """...(existing docstring unchanged)..."""

    # Fix: gateway doesn't support real SSE — redirect to non-streaming path
    if (
        isinstance(getattr(agent, "provider", ""), str)
        and "my-gateway" in agent.provider    # ← your provider name from config
    ):
        return interruptible_api_call(agent, api_kwargs)

    # rest of function unchanged...
```

### Step 5: Cold-start Hermes

Launch Hermes fresh (not from a minimized window). You should see your gateway listed as the active provider.

---

## How to Use After Setup

Everything works the same as with an official provider:

```
# In Hermes chat
/model custom:my-gateway:your-model-name   # switch to your gateway mid-session
```

Your gateway appears in the `hermes model` picker. No difference in day-to-day use.

---

## Error Reference

### `HTTP 401: ApiKey Validate fail`

Hermes auto-detects URLs ending in `/anthropic` and switches to the Anthropic native SDK (using `x-api-key` header). Third-party gateways don't recognize this auth format.

**Fix:** Use root gateway URL + `transport: chat_completions`.

---

### `empty stream with no finish_reason (possible upstream error or malformed SSE response)`

Your gateway returns a complete JSON object for `stream: true` instead of real SSE. Hermes iterates over zero chunks, finds no `finish_reason`, and throws.

**Fix:** Patch `interruptible_streaming_api_call`. See Step 4.

---

### Config reverts on restart

Hermes Desktop saves its state (including model config) to `config.yaml` on exit. Edits made while the app is running get overwritten.

**Fix:** Quit Hermes fully before editing config.

---

## Limitations

| Item | Status |
|------|--------|
| Windows 10 | ✅ Tested and working |
| macOS | ⚠️ Not tested. Config path and file paths differ (see above). Should work in theory. **PRs welcome.** |
| Linux | ⚠️ Not tested. Same paths as macOS. **PRs welcome.** |
| Gateways with real SSE streaming | ✅ Skip Step 4, just do config |
| Gateways requiring `x-api-key` auth | ⚠️ May need `transport: anthropic_messages` — see [Hermes provider docs](https://hermes-agent.nousresearch.com/docs/integrations/providers) |
| Hermes version | Tested on latest as of June 2026. File paths may change in future versions. |

---

## Checklist

- [ ] Using root gateway URL (no `/anthropic` suffix)
- [ ] `transport: chat_completions` set explicitly
- [ ] Hermes Desktop fully quit before editing config
- [ ] Tested streaming with curl
- [ ] Applied patch if gateway uses fake streaming
- [ ] Cold-started Hermes

---

## Contributing

Tested on macOS or Linux? Please open a PR updating the Limitations table.  
Different gateway quirk not covered here? Issues and PRs welcome.

---

## License

MIT
