# Connecting a Third-Party AI Gateway to Hermes Agent

> A real debugging story: 3 errors, 3 fixes — how to connect any third-party OpenAI/Anthropic-compatible API gateway to [Hermes Agent](https://github.com/NousResearch/hermes-agent).

## Who This Is For

If you're using a **third-party API gateway / relay** (中转站) instead of calling OpenAI or Anthropic directly, and you want to use it with Hermes Agent — this guide is for you.

Common examples:
- A gateway that proxies Anthropic Claude behind a custom URL
- A gateway that routes to multiple providers under one endpoint
- Any OpenAI-compatible relay that doesn't support SSE streaming

---

## Background

I wanted to use an Anthropic-compatible gateway with Hermes Agent. The gateway exposes its endpoint at:

```
https://<gateway-host>/v2/gws/<token>/anthropic
```

What followed was a 3-error debugging journey that took hours to resolve.

---

## Error 1 — `HTTP 401: ApiKey Validate fail`

### Symptom

```
API call failed: HTTP 401: ApiKey Validate fail
```

### Root Cause

Hermes has URL-based auto-detection in `agent/anthropic_adapter.py`. Any URL path ending in `/anthropic` triggers the **Anthropic native SDK** with `x-api-key` header auth — even for third-party gateways:

```python
def _endpoint_speaks_anthropic_messages(base_url: str) -> bool:
    path = urlparse(normalized).path.rstrip("/")
    if path.endswith("/anthropic") or path.endswith("/anthropic/v1"):
        return True  # triggers for ANY url ending in /anthropic!
```

Your gateway doesn't understand Anthropic's native `x-api-key` format, so it returns 401.

### Fix

Use the **gateway root URL** (without `/anthropic`) and force `transport: chat_completions` in `config.yaml`:

```yaml
providers:
  my-gateway:
    api: https://<gateway-host>/v2/gws/<token>    # ← root URL, no /anthropic
    transport: chat_completions                     # ← force OpenAI-compatible mode
    api_key: <your-api-key>
    default_model: <model-name>

model:
  default: <model-name>
  provider: custom:my-gateway
  base_url: https://<gateway-host>/v2/gws/<token>
```

---

## Error 2 — `Provider returned an empty stream with no finish_reason`

### Symptom

```
API call failed after 3 retries: Provider returned an empty stream 
with no finish_reason (possible upstream error or malformed SSE response).
```

### Root Cause

Many third-party gateways **do not support real SSE streaming**. When Hermes sends `stream: true`, the gateway ignores it and returns a complete JSON response — not the `data: {...}` chunked SSE format Hermes expects.

Verify this yourself with curl:

```bash
# Send stream: true — gateway returns plain JSON, not SSE chunks
curl https://<gateway-host>/v2/gws/<token>/v1/chat/completions \
  -H "Authorization: Bearer <key>" \
  -H "content-type: application/json" \
  -d '{"model":"<model>","max_tokens":10,"messages":[{"role":"user","content":"hi"}],"stream":true}'

# If you get back: {"id":"...","choices":[{"message":{"content":"Hi!"}}],...}
# and NOT:         data: {"choices":[{"delta":{"content":"Hi"}}]}
# → your gateway doesn't support real streaming
```

### Fix

Patch `agent/chat_completion_helpers.py` in your Hermes installation to redirect requests for your provider to the non-streaming path.

Find the `interruptible_streaming_api_call` function and add these lines right after the docstring:

```python
def interruptible_streaming_api_call(agent, api_kwargs: dict, *, on_first_delta=None):
    """...(existing docstring)..."""

    # Fix for gateways that don't support real SSE streaming.
    # Replace "my-gateway" with your provider name.
    if (
        isinstance(getattr(agent, "provider", ""), str)
        and "my-gateway" in agent.provider
    ):
        return interruptible_api_call(agent, api_kwargs)

    # ... rest of function unchanged
```

The file is at: `%LOCALAPPDATA%\hermes\hermes-agent\agent\chat_completion_helpers.py` (Windows)

---

## Error 3 — Config Reverts on Restart

### Symptom

You edit `config.yaml`, restart Hermes, but it goes back to the old provider.

### Root Cause

The Hermes Desktop app writes config on exit. If you edit `config.yaml` while the app is running, the app **overwrites your changes** when it saves state.

### Fix

1. **Fully quit Hermes Desktop** — right-click tray icon → Exit, or kill `Hermes.exe` in Task Manager
2. Edit `config.yaml`
3. **Cold-start Hermes** (launch fresh, don't resume a minimized window)

---

## Final Working Configuration

**`config.yaml`** (Windows: `%LOCALAPPDATA%\hermes\config.yaml`):

```yaml
model:
  default: <your-model-name>
  provider: custom:my-gateway
  base_url: https://<gateway-host>/<gateway-path>

providers:
  my-gateway:
    api: https://<gateway-host>/<gateway-path>
    transport: chat_completions
    api_key: <your-api-key>
    default_model: <your-model-name>

display:
  streaming: false

streaming:
  enabled: false
```

**`agent/chat_completion_helpers.py`** — add at the top of `interruptible_streaming_api_call`:

```python
if (
    isinstance(getattr(agent, "provider", ""), str)
    and "my-gateway" in agent.provider   # ← your provider name
):
    return interruptible_api_call(agent, api_kwargs)
```

---

## Summary Table

| # | Error | Root Cause | Fix |
|---|-------|-----------|-----|
| 1 | `401 auth fail` | URL ends in `/anthropic` → Hermes uses Anthropic SDK with wrong auth | Use root gateway URL + `transport: chat_completions` |
| 2 | `empty stream / no finish_reason` | Gateway returns plain JSON for `stream:true`, not real SSE | Patch `interruptible_streaming_api_call` to use non-streaming path |
| 3 | Config reverts on restart | Desktop app overwrites config on exit | Fully quit desktop before editing config |

---

## Quick Checklist for Any Third-Party Gateway

- [ ] Test with curl: does `stream: true` return real `data:` SSE chunks, or complete JSON?
- [ ] Test auth: does it accept `x-api-key` or `Authorization: Bearer`?
- [ ] Use the root gateway URL, not a sub-path ending in `/anthropic` or `/v1`
- [ ] Set `transport: chat_completions` explicitly in `providers:` config
- [ ] Fully quit Hermes Desktop before editing config
- [ ] If the gateway doesn't stream: patch `interruptible_streaming_api_call`

---

## Environment

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch)
- Windows 10
- Third-party Anthropic-compatible gateway
- Model: Claude Sonnet 4.6 via gateway
