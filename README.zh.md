# 第三方 AI 中转站接入 Hermes Agent 完整指南

[![GitHub stars](https://img.shields.io/github/stars/roytj888/connect-any-llm-gateway-to-hermes?style=social)](https://github.com/roytj888/connect-any-llm-gateway-to-hermes/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/roytj888/connect-any-llm-gateway-to-hermes/pulls)
[![测试平台](https://img.shields.io/badge/已测试-Windows%2010-blue)](#限制与已知问题)

**[English README →](README.md)**

---

> ⭐ 如果这篇文章帮你省了时间，请给这个仓库点个 Star —— 让更多遇到同样问题的人能找到它。

---


> 你买了一个 AI 中转站的 API，装好了 Hermes Agent，把 URL 和密钥填进去。  
> 然后报错了。错误信息完全没用。  
>
> 这篇文章记录了每一个报错、每一个根本原因、每一个解决方法 —— 让你不用重走我踩过的坑。

---

## 这是什么

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是 NousResearch 开源的 AI Agent 框架。接入官方 provider（Anthropic、OpenAI 等）很顺畅，但接入**第三方 API 中转站**会碰到 3 个没有任何文档记录的坑：

| # | 报错现象 | 根本原因 |
|---|---------|---------|
| 1 | `401 ApiKey Validate fail` | Hermes 识别到 URL 含 `/anthropic` 就自动切换为 Anthropic SDK 认证，第三方中转站不认这种格式 |
| 2 | `empty stream with no finish_reason` | 大部分中转站不支持真正的 SSE 流式，`stream: true` 时返回完整 JSON，Hermes 解析失败 |
| 3 | 重启后配置恢复原样 | Hermes 桌面应用退出时会覆盖 `config.yaml`，在运行时编辑的内容会丢失 |

本仓库记录了全部三个问题的解法，并提供了一个可直接安装的 **Hermes Skill**，让 Hermes 自动完成配置。

---

## 方式一：安装 Hermes Skill（推荐）

安装 Skill，让 Hermes 自动帮你配置：

```bash
hermes skills install https://raw.githubusercontent.com/roytj888/connect-any-llm-gateway-to-hermes/main/llm-gateway-connect.skill.md
```

然后在 Hermes 里直接说：

```
帮我接入第三方中转站：
URL: https://你的中转站地址/路径
API Key: 你的密钥
模型: 模型名称
```

Hermes 会自动检测你的中转站类型、写入配置、必要时打补丁。

---

## 方式二：手动配置

### 第一步：找到中转站根路径

如果你的 URL 是：
```
https://gateway.example.com/v2/gws/abc123/anthropic
```
只用**根路径**，去掉 `/anthropic` 及之后的内容：
```
https://gateway.example.com/v2/gws/abc123
```

### 第二步：编辑 `config.yaml`

- Windows：`%LOCALAPPDATA%\hermes\config.yaml`
- macOS / Linux：`~/.hermes/config.yaml`

> ⚠️ **编辑前必须完全退出 Hermes 桌面应用。** 右键任务栏图标 → 退出，或在任务管理器里结束 `Hermes.exe`。应用运行时编辑，退出时会被覆盖。

```yaml
model:
  default: 你的模型名
  provider: custom:my-gateway
  base_url: https://gateway.example.com/v2/gws/abc123

providers:
  my-gateway:
    api: https://gateway.example.com/v2/gws/abc123
    transport: chat_completions      # 强制走 OpenAI 兼容协议
    api_key: 你的API密钥
    default_model: 你的模型名

display:
  streaming: false

streaming:
  enabled: false
```

### 第三步：检测你的中转站是否支持真正的流式输出

```bash
curl https://你的中转站根路径/v1/chat/completions \
  -H "Authorization: Bearer 你的密钥" \
  -H "content-type: application/json" \
  -d '{"model":"你的模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}],"stream":true}'
```

**真正的 SSE 流式（不需要打补丁）：**
```
data: {"choices":[{"delta":{"content":"你"}}]}
data: [DONE]
```

**假流式（需要打补丁）：**
```json
{"id":"...","choices":[{"message":{"content":"你好！"}}],"finish_reason":"stop"}
```

### 第四步：打补丁（仅假流式需要）

文件路径：
- Windows：`%LOCALAPPDATA%\hermes\hermes-agent\agent\chat_completion_helpers.py`
- macOS / Linux：`~/.hermes/hermes-agent/agent/chat_completion_helpers.py`

找到 `interruptible_streaming_api_call` 函数，在 docstring 后面加几行：

```python
def interruptible_streaming_api_call(agent, api_kwargs: dict, *, on_first_delta=None):
    """...(原有 docstring 不变)..."""

    # 修复：中转站不支持真正的 SSE，直接走非流式调用
    if (
        isinstance(getattr(agent, "provider", ""), str)
        and "my-gateway" in agent.provider    # ← 替换成你在 config 里的 provider 名
    ):
        return interruptible_api_call(agent, api_kwargs)

    # 函数其余部分不变...
```

### 第五步：重新启动 Hermes

完全重启（不是恢复最小化窗口）。启动后应该能看到你的中转站显示为当前 provider。

---

## 配置好之后怎么用

跟官方 provider 完全一样，没有任何区别：

```
# 在 Hermes 对话中切换模型
/model custom:my-gateway:你的模型名
```

你的中转站会出现在 `hermes model` 选择器里。

---

## 报错详解

### `HTTP 401: ApiKey Validate fail`

Hermes 检测到 URL 以 `/anthropic` 结尾，自动切换为 Anthropic 原生 SDK（用 `x-api-key` 头部）。第三方中转站不认这种认证方式，返回 401。

**解决：** 用根路径 + `transport: chat_completions`。

---

### `empty stream with no finish_reason (possible upstream error or malformed SSE response)`

中转站对 `stream: true` 返回完整 JSON，而不是真正的 SSE 分块。Hermes 遍历到零个 chunk，找不到 `finish_reason`，抛出错误。

**解决：** 打补丁，见第四步。

---

### 重启后配置恢复原样

Hermes 桌面应用退出时把运行时状态（包括模型配置）写入 `config.yaml`。在应用运行时的编辑会被覆盖。

**解决：** 编辑前完全退出应用。

---

## 限制与已知问题

| 项目 | 状态 |
|------|------|
| Windows 10 | ✅ 已测试，正常工作 |
| macOS | ⚠️ 未测试。配置文件路径不同（见上），理论上应该可用。**欢迎 PR。** |
| Linux | ⚠️ 未测试。路径同 macOS。**欢迎 PR。** |
| 支持真正 SSE 流式的中转站 | ✅ 跳过第四步，只需改配置 |
| 需要 `x-api-key` 认证的中转站 | ⚠️ 可能需要 `transport: anthropic_messages`，参考 [Hermes provider 文档](https://hermes-agent.nousresearch.com/docs/integrations/providers) |
| Hermes 版本 | 在 2026 年 6 月最新版本测试。未来版本的内部文件路径可能变化。 |

---

## 检查清单

- [ ] 使用根路径（不含 `/anthropic` 后缀）
- [ ] 明确设置了 `transport: chat_completions`
- [ ] 编辑配置前完全退出了 Hermes 桌面应用
- [ ] 用 curl 测试了流式输出类型
- [ ] 假流式的话打了补丁
- [ ] 完全重启了 Hermes

---

## 贡献

在 macOS 或 Linux 上测试成功了？欢迎 PR 更新限制表格。  
遇到了其他中转站的问题？欢迎提 Issue 或 PR。

---

## License

MIT
