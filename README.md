<p align="center">
  <img src="https://img.shields.io/badge/Codex-v0.129%2B-blue?logo=openai" alt="Codex">
  <img src="https://img.shields.io/badge/DeepSeek-V4-4B8BF5?logo=deepseek" alt="DeepSeek V4">
  <img src="https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white" alt="Node.js">
  <img src="https://img.shields.io/badge/Platform-Windows%2010%2F11-0078D6?logo=windows" alt="Platform">
  <img src="https://img.shields.io/badge/License-MIT-green" alt="License">
</p>

<h1 align="center">Codex + DeepSeek V4 接入指南</h1>

<p align="center">
  <strong>不降级、不破解，用本地代理让 Codex 最新桌面版跑在 DeepSeek 上</strong>
</p>

<p align="center">
  <a href="#-快速开始">快速开始</a> ·
  <a href="#-架构">架构</a> ·
  <a href="#-常见问题">FAQ</a> ·
  <a href="#-进阶">进阶</a>
</p>

---

## 📖 目录

- [背景](#-背景)
- [架构](#-架构)
- [快速开始](#-快速开始)
- [验证](#-验证)
- [常见问题](#-常见问题)
- [进阶](#-进阶)
- [参考资料](#-参考资料)
- [License](#-license)

---

## 💡 背景

[OpenAI Codex](https://github.com/openai/codex) 使用 **Responses API**，而 [DeepSeek](https://platform.deepseek.com) 只提供 **Chat Completions API**。两者协议不兼容，直接对接会返回 400。

| 维度 | Codex ≥ v0.81.0 | DeepSeek API |
|------|:---------------:|:------------:|
| 协议 | Responses API | Chat Completions API |
| 端点 | `/v1/responses` | `/v1/chat/completions` |
| 工具调用 | `tools` 内联 | `tool_calls` 独立消息 |

本指南使用 [codex-bridge](https://github.com/wujfeng712-ui/codex-bridge) 在本地做协议翻译，无需降级 Codex 版本。

---

## 🧱 架构

```
Codex Desktop (v0.129.0+)
       │
       │  Responses API
       ▼
codex-bridge (localhost:4000)
       │
       │  Chat Completions API
       ▼
DeepSeek API (api.deepseek.com)
```

- **codex-bridge**：Node.js 单文件，零依赖，~2000 行
- 支持流式 SSE、思考模式 (`reasoning_content`)、工具调用回合
- 内存占用 ~30MB

---

## 🚀 快速开始

### 前置条件

- Windows 10/11
- **Node.js ≥ 18** — [`nodejs.org`](https://nodejs.org/)
- **Codex 桌面版** — [`releases`](https://github.com/openai/codex/releases)
- **DeepSeek API Key** — [`platform.deepseek.com`](https://platform.deepseek.com/api_keys)
- 终端（Git Bash 推荐）

### 1. Clone & Config

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

编辑 `~/.codex/codex-bridge/.env`：

```ini
DEEPSEEK_API_KEY=sk-your-key-here
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner
DEFAULT_PROVIDER=deepseek
LOG_LEVEL=info
```

### 2. Codex 配置

**`~/.codex/config.toml`**

```toml
cli_auth_credentials_store = "file"
model = "deepseek-v4-pro"
model_provider = "local_proxy"

[model_providers.local_proxy]
name = "local_proxy"
base_url = "http://127.0.0.1:4000/v1"
wire_api = "responses"
requires_openai_auth = true
```

**`~/.codex/auth.json`**

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-your-key-here"
}
```

### 3. 登录

```bash
echo "sk-your-key-here" | codex login --with-api-key
```

验证：`codex login status` 应显示 `Logged in using an API key`。

### 4. 启动代理

```bash
cd ~/.codex/codex-bridge
node --env-file=.env proxy.mjs
```

输出：

```
[codex-bridge] Listening on http://localhost:4000
[codex-bridge] Deepseek: https://api.deepseek.com/v1
```

### 5. 启动 Codex

打开 Codex 桌面版，切换模型：

```
/model deepseek-v4-pro     # 推理模型
/model deepseek-v4-flash   # 快速模型
```

---

## ✅ 验证

```bash
codex exec "回复一个字：好"
```

看到模型为 `deepseek-v4-pro`、提供方为 `local_proxy`，且正常输出即表示成功。

---

## ❓ 常见问题

<details>
<summary><strong>桌面版仍然弹登录窗口</strong></summary>

这是 Codex [Issue #2450](https://github.com/openai/codex/issues/2450) 的已知问题。用 [CC Switch](https://github.com/farion1231/cc-switch/releases) 可绕过：

1. 下载 `.msi` 安装
2. Codex 选项卡 → 添加供应商
3. API Key 填 DeepSeek 密钥，Base URL 填 `http://127.0.0.1:4000/v1`
4. 启用后重启 Codex
</details>

<details>
<summary><code>wire_api = chat is no longer supported</code></summary>

Codex v0.81.0 起只支持 `wire_api = "responses"`。确认 `config.toml` 中为此值，由代理做 Chat Completions 翻译。
</details>

<details>
<summary>端口 4000 被占用</summary>

```bash
netstat -ano | findstr ":4000"
taskkill /PID <PID> /F
```

或在 `.env` 中改 `PROXY_PORT`，同步更新 `config.toml` 的 `base_url`。
</details>

<details>
<summary>关闭终端后代理停了</summary>

见 [进阶 → 开机自启](#-进阶)。
</details>

---

## 🔧 进阶

### 代理开机自启

**方式一：任务计划程序**

```powershell
$action = New-ScheduledTaskAction -Execute "node" `
  -Argument "--env-file=$env:USERPROFILE\.codex\codex-bridge\.env $env:USERPROFILE\.codex\codex-bridge\proxy.mjs" `
  -WorkingDirectory "$env:USERPROFILE\.codex\codex-bridge"
$trigger = New-ScheduledTaskTrigger -AtLogon
Register-ScheduledTask -TaskName "CodexBridge" -Action $action -Trigger $trigger `
  -Description "Codex → DeepSeek" -RunLevel Highest
```

**方式二：启动文件夹**

在 `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup` 创建快捷方式：

- 目标：`node --env-file=.env proxy.mjs`
- 起始位置：`C:\Users\<用户名>\.codex\codex-bridge`

### 多模型

编辑 `.env` 的 `DEEPSEEK_MODELS`，逗号分隔，重启代理后在 Codex 内 `/model` 切换。

---

## 📚 参考资料

- [codex-bridge](https://github.com/wujfeng712-ui/codex-bridge) — 协议翻译代理
- [Codex](https://github.com/openai/codex) — OpenAI 官方仓库
- [DeepSeek API](https://platform.deepseek.com/api-docs) — API 文档
- [CC Switch](https://github.com/farion1231/cc-switch) — 多供应商管理工具
- [Issue #2450](https://github.com/openai/codex/issues/2450) — 登录强制弹窗

---

## 📄 License

MIT © 2026
