---
name: llm-gateway-connect
description: Use when connecting a third-party OpenAI/Anthropic-compatible API gateway (relay/中转站) to Hermes Agent. Diagnoses auth errors, fake-streaming failures, and config-revert issues, then applies the correct config and code patch automatically.
version: 1.0.0
author: roytj888
license: MIT
platforms: [windows, macos, linux]
metadata:
  hermes:
    tags: [gateway, third-party, relay, streaming, custom-provider, anthropic, openai, 中转站]
    related_skills: [hermes-agent, hermes-custom-providers]
---

# Connect Any Third-Party LLM Gateway to Hermes Agent

## Overview

Third-party API gateways (relays / 中转站) hit 3 non-obvious problems when connecting to Hermes Agent:

1. **401 auth error** — Hermes auto-detects URLs ending in `/anthropic` and switches to Anthropic SDK auth (`x-api-key`), which most gateways don't support
2. **Empty stream / no finish_reason** — Most gateways fake streaming: they return complete JSON for `stream: true` instead of real SSE chunks
3. **Config reverts on restart** — Hermes Desktop overwrites `config.yaml` on exit if edited while running

This skill diagnoses which problems apply and fixes them automatically.

## When to Use

- User says "help me connect my third-party gateway / relay / 中转站"
- User provides a gateway URL + API key + model name
- User reports `401 ApiKey Validate fail`
- User reports `empty stream with no finish_reason`
- User reports config reverting after Hermes restart

Don't use for:
- Official providers (Anthropic, OpenAI, OpenRouter) — use `hermes model` instead
- Local models (Ollama, llama.cpp) — different setup path

## Steps

### 1. Collect gateway info

Ask for (or extract from user message):
- Gateway URL (full URL including any path)
- API key
- Model name

### 2. Determine root URL

Strip `/anthropic`, `/v1`, or any trailing path segment that would trigger Hermes auto-detection:

```python
# If URL ends with /anthropic or /anthropic/v1 → strip it
# Example:
# https://gateway.com/v2/gws/abc123/anthropic  →  https://gateway.com/v2/gws/abc123
# https://gateway.com/api/v1  →  use as-is (no /anthropic suffix)
```

### 3. Test streaming support with curl

Run this to determine if the gateway supports real SSE:

```bash
curl https://<ROOT_URL>/v1/chat/completions \
  -H "Authorization: Bearer <API_KEY>" \
  -H "content-type: application/json" \
  -d '{"model":"<MODEL>","max_tokens":10,"messages":[{"role":"user","content":"hi"}],"stream":true}'
```

- **Real SSE**: response starts with `data: {` → no code patch needed
- **Fake streaming**: response is a single JSON object `{"id":...,"choices":[...]}` → patch required

### 4. Fully quit Hermes Desktop

Before writing config:
- Windows: right-click tray icon → Exit, or `taskkill /IM Hermes.exe /F`
- macOS: quit from menu bar or `killall Hermes`
- Verify: no `Hermes.exe` / `Hermes` process in task manager

### 5. Write config.yaml

Config path:
- Windows: `%LOCALAPPDATA%\hermes\config.yaml`  
- macOS/Linux: `~/.hermes/config.yaml`

Use a provider name derived from the gateway hostname (e.g. `edgecloud-gateway`, `my-relay`):

```yaml
model:
  default: <MODEL_NAME>
  provider: custom:<PROVIDER_NAME>
  base_url: <ROOT_URL>

providers:
  <PROVIDER_NAME>:
    api: <ROOT_URL>
    transport: chat_completions
    api_key: <API_KEY>
    default_model: <MODEL_NAME>

display:
  streaming: false

streaming:
  enabled: false
```

### 6. Patch code (only if fake streaming detected in Step 3)

File:
- Windows: `%LOCALAPPDATA%\hermes\hermes-agent\agent\chat_completion_helpers.py`
- macOS/Linux: `~/.hermes/hermes-agent/agent/chat_completion_helpers.py`

Find `interruptible_streaming_api_call`. Add immediately after the closing `"""` of its docstring:

```python
    # Fix: gateway does not support real SSE streaming
    if (
        isinstance(getattr(agent, "provider", ""), str)
        and "<PROVIDER_NAME>" in agent.provider
    ):
        return interruptible_api_call(agent, api_kwargs)
```

Also clear the Python cache file:
- Windows: delete `%LOCALAPPDATA%\hermes\hermes-agent\agent\__pycache__\chat_completion_helpers.cpython-*.pyc`
- macOS/Linux: delete `~/.hermes/hermes-agent/agent/__pycache__/chat_completion_helpers.cpython-*.pyc`

### 7. Cold-start Hermes

Launch Hermes fresh. Verify by running a test message and confirming the active model shows the new provider.

## Verification

After setup, confirm:
- [ ] Active model shows `custom:<PROVIDER_NAME>` as provider
- [ ] A simple message gets a response (no 401, no stream error)
- [ ] Config still correct after restarting Hermes

## Common Pitfalls

1. **Editing config while Hermes Desktop is running** — the app overwrites `config.yaml` on exit. Always quit fully first.

2. **Leaving `/anthropic` in the base_url** — Hermes auto-detects this and switches to Anthropic SDK auth, causing 401.

3. **Forgetting to clear Python cache** — old `.pyc` file may be used instead of the patched `.py` file.

4. **Only closing the window, not quitting** — minimizing or closing the window doesn't quit the app. Check the system tray.

5. **Gateway requires `x-api-key` instead of `Bearer`** — some gateways use Anthropic-style auth. If `Bearer` fails, try `transport: anthropic_messages` in config.

6. **Model name mismatch** — use the exact model name the gateway expects (e.g. `anthropic.claude-sonnet-4-6` not `claude-sonnet-4-6`). Test with curl first.
