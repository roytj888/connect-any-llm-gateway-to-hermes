# 第三方 AI 中转站接入 Hermes Agent 完整指南

> 真实踩坑记录：3 个报错、3 个解决方案 —— 教你把任何第三方 OpenAI/Anthropic 兼容中转站接入 [Hermes Agent](https://github.com/NousResearch/hermes-agent)。

## 适合哪些人看

如果你在用**第三方 API 中转站**（而不是直接调用 OpenAI 或 Anthropic 官方），想把它接入 Hermes Agent，这篇文章就是为你写的。

常见场景：
- 代理了 Anthropic Claude 的中转站
- 汇聚多个模型的统一中转接口
- 任何不支持 SSE 流式输出的 OpenAI 兼容接口

---

## 背景

我想用一个 Anthropic 兼容的中转站接入 Hermes Agent，中转站的接口是：

```
https://<中转站地址>/v2/gws/<token>/anthropic
```

接下来踩了 3 个坑，折腾了好几个小时才搞定。

---

## 报错一：`HTTP 401: ApiKey Validate fail`

### 现象

```
API call failed: HTTP 401: ApiKey Validate fail
```

### 根本原因

Hermes 在 `agent/anthropic_adapter.py` 里有 URL 自动识别逻辑：只要 URL 路径以 `/anthropic` 结尾，就会自动走 **Anthropic 原生 SDK**，用 `x-api-key` 头部认证：

```python
def _endpoint_speaks_anthropic_messages(base_url: str) -> bool:
    path = urlparse(normalized).path.rstrip("/")
    if path.endswith("/anthropic") or path.endswith("/anthropic/v1"):
        return True  # 所有以 /anthropic 结尾的 URL 都会触发！
```

中转站不认识 Anthropic 原生的 `x-api-key` 认证方式，所以返回 401。

### 解决方法

在 `config.yaml` 里**用中转站的根路径**（不要带 `/anthropic`），并明确指定 `transport: chat_completions`：

```yaml
providers:
  my-gateway:
    api: https://<中转站地址>/v2/gws/<token>    # ← 根路径，不带 /anthropic
    transport: chat_completions                   # ← 强制走 OpenAI 兼容模式
    api_key: <你的API密钥>
    default_model: <模型名>

model:
  default: <模型名>
  provider: custom:my-gateway
  base_url: https://<中转站地址>/v2/gws/<token>
```

---

## 报错二：`Provider returned an empty stream with no finish_reason`

### 现象

```
API call failed after 3 retries: Provider returned an empty stream 
with no finish_reason (possible upstream error or malformed SSE response).
```

### 根本原因

很多第三方中转站**不支持真正的 SSE 流式输出**。当 Hermes 发送 `stream: true` 时，中转站直接忽略，返回完整的 JSON —— 而不是 Hermes 期望的 `data: {...}` 分块格式。

用 curl 可以验证：

```bash
# 发送 stream: true，看看中转站返回什么
curl https://<中转站地址>/v2/gws/<token>/v1/chat/completions \
  -H "Authorization: Bearer <key>" \
  -H "content-type: application/json" \
  -d '{"model":"<模型名>","max_tokens":10,"messages":[{"role":"user","content":"hi"}],"stream":true}'

# 如果返回的是: {"id":"...","choices":[{"message":{"content":"你好！"}}],...}
# 而不是:       data: {"choices":[{"delta":{"content":"你"}}]}
# → 说明你的中转站不支持真正的流式输出
```

### 解决方法

修改 Hermes 安装目录里的 `agent/chat_completion_helpers.py`，在 `interruptible_streaming_api_call` 函数的 docstring 后面加几行代码，让你的 provider 直接走非流式路径：

```python
def interruptible_streaming_api_call(agent, api_kwargs: dict, *, on_first_delta=None):
    """...(原有 docstring 不变)..."""

    # 针对不支持 SSE 流式的中转站，直接走非流式调用
    # 把 "my-gateway" 替换成你的 provider 名称
    if (
        isinstance(getattr(agent, "provider", ""), str)
        and "my-gateway" in agent.provider
    ):
        return interruptible_api_call(agent, api_kwargs)

    # ... 函数其余部分不变
```

文件路径（Windows）：`%LOCALAPPDATA%\hermes\hermes-agent\agent\chat_completion_helpers.py`

---

## 报错三：重启后配置恢复原样

### 现象

修改了 `config.yaml`，重启 Hermes 后又恢复到旧的 provider 了。

### 根本原因

Hermes 桌面应用退出时会写入配置。如果你在**应用运行时**编辑 `config.yaml`，应用退出时会把你的修改**覆盖掉**。

### 解决方法

1. **完全退出 Hermes 桌面应用** —— 右键任务栏图标 → 退出，或者在任务管理器里结束 `Hermes.exe` 进程
2. 编辑 `config.yaml`
3. **重新启动 Hermes**（不要恢复最小化的窗口）

---

## 最终可用配置

**`config.yaml`**（Windows 路径：`%LOCALAPPDATA%\hermes\config.yaml`）：

```yaml
model:
  default: <你的模型名>
  provider: custom:my-gateway
  base_url: https://<中转站地址>/<路径>

providers:
  my-gateway:
    api: https://<中转站地址>/<路径>
    transport: chat_completions
    api_key: <你的API密钥>
    default_model: <你的模型名>

display:
  streaming: false

streaming:
  enabled: false
```

**`agent/chat_completion_helpers.py`** —— 在 `interruptible_streaming_api_call` 函数开头加：

```python
if (
    isinstance(getattr(agent, "provider", ""), str)
    and "my-gateway" in agent.provider   # ← 替换成你的 provider 名称
):
    return interruptible_api_call(agent, api_kwargs)
```

---

## 三个坑汇总

| # | 报错 | 根本原因 | 解决方法 |
|---|------|---------|---------|
| 1 | `401 auth fail` | URL 含 `/anthropic` → Hermes 误用 Anthropic SDK 认证 | 用根路径 + `transport: chat_completions` |
| 2 | `empty stream / no finish_reason` | 中转站对 `stream:true` 返回完整 JSON，不是真正的 SSE | 在 `interruptible_streaming_api_call` 里加非流式重定向 |
| 3 | 配置重启后恢复 | 桌面应用退出时覆盖配置 | 完全退出应用再改配置 |

---

## 第三方中转站接入检查清单

- [ ] 用 curl 测试：`stream: true` 返回的是真正的 `data:` SSE 分块，还是完整 JSON？
- [ ] 确认认证方式：接受 `x-api-key` 还是 `Authorization: Bearer`？
- [ ] 用根路径，不要用 `/anthropic` 或 `/v1` 结尾的子路径
- [ ] 在 `providers:` 配置里明确写 `transport: chat_completions`
- [ ] 修改配置前完全退出 Hermes 桌面应用
- [ ] 如果不支持流式：在 `interruptible_streaming_api_call` 里加重定向代码

---

## 环境

- [Hermes Agent](https://github.com/NousResearch/hermes-agent)（NousResearch）
- Windows 10
- 第三方 Anthropic 兼容中转站
- 模型：通过中转站使用 Claude Sonnet 4.6
