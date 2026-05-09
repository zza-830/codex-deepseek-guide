# 2026 Windows 环境 Codex 桌面版接入 DeepSeek V4 完整教程

> 免降级、免破解，用本地代理实现 Codex 最新版 + DeepSeek 的完美协作

---

## 一、背景

OpenAI 的 [Codex](https://github.com/openai/codex) 是目前最强的 AI 编程助手之一，有 CLI 和桌面 App 两种形态。DeepSeek V4 则是性价比极高的国产大模型，代码能力出众且 API 价格低廉。

**但两者直接对接会失败**，原因在于 API 协议不兼容：

| | Codex (v0.81.0+) | DeepSeek API |
|---|---|---|
| 协议 | **Responses API** | **Chat Completions API** |
| 请求路径 | `/v1/responses` | `/v1/chat/completions` |
| 工具调用格式 | `tools` 字段内联 | `tool_calls` 独立消息 |

直接配 `base_url = "https://api.deepseek.com/v1"` 会报 400 错误。

**解决方案**：在本地部署一个轻量代理，实时翻译两种协议。

---

## 二、方案架构

```
┌──────────────────┐         Responses API         ┌─────────────────┐
│  Codex 桌面版     │ ──────────────────────────────▶│  codex-bridge    │
│  (v0.129.0)      │   Authorization: Bearer <key>  │  localhost:4000  │
└──────────────────┘                                └────────┬────────┘
                                                             │
                                                    Chat Completions API
                                                             │
                                                             ▼
                                                  ┌──────────────────┐
                                                  │  DeepSeek API    │
                                                  │  api.deepseek.com│
                                                  └──────────────────┘
```

我们使用开源项目 **[codex-bridge](https://github.com/wujfeng712-ui/codex-bridge)**（Node.js 单文件，零依赖）作为代理。

---

## 三、环境要求

- **Windows 10/11**
- **Node.js 18+**（[下载](https://nodejs.org/)）
- **Codex 桌面版**（[下载](https://github.com/openai/codex/releases) 最新版即可，无需降级）
- **DeepSeek API Key**（[获取](https://platform.deepseek.com/api_keys)）
- 基本的终端操作能力（PowerShell 或 Git Bash）

---

## 四、详细步骤

### 步骤 1：确认环境

打开终端（推荐 Git Bash），执行：

```bash
node --version    # 应输出 v18.0.0 或更高
codex --version   # 应输出 codex-cli 0.x.x
```

### 步骤 2：获取 codex-bridge 代理

```bash
# 克隆仓库
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

> 如果无法访问 GitHub，可以用镜像：`git clone https://gitcode.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge`

### 步骤 3：配置代理环境变量

编辑 `C:\Users\<你的用户名>\.codex\codex-bridge\.env`，填入以下内容：

```bash
# === DeepSeek 上游密钥 ===
DEEPSEEK_API_KEY=sk-你的DeepSeek密钥

# === 暴露的模型列表 ===
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner

# === 默认供应商 ===
DEFAULT_PROVIDER=deepseek

# === 日志级别 ===
LOG_LEVEL=info
```

> **安全提示**：密钥不要加引号，直接写 `sk-xxx`。此文件包含明文密钥，切勿提交到 Git 仓库。

### 步骤 4：配置 Codex

#### 4.1 编辑 `~\.codex\config.toml`

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

> `cli_auth_credentials_store = "file"` 是关键配置——它告诉 Codex 桌面版从文件读取密钥，而不是弹出浏览器 OAuth 登录界面。

#### 4.2 编辑 `~\.codex\auth.json`

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-你的DeepSeek密钥"
}
```

> 注意：即使走的是自定义 API，变量名也必须叫 `OPENAI_API_KEY`，这是 Codex 的硬编码要求。

#### 4.3 初始化登录状态

```bash
echo "sk-你的DeepSeek密钥" | codex login --with-api-key
```

执行后应该看到：`Successfully logged in`

验证：

```bash
codex login status
# 输出：Logged in using an API key - sk-***xxxx
```

### 步骤 5：启动代理

```bash
cd ~/.codex/codex-bridge
node --env-file=.env proxy.mjs
```

看到以下输出即表示成功：

```
[codex-bridge] Listening on http://localhost:4000
[codex-bridge] Default provider: deepseek
[codex-bridge] Deepseek: https://api.deepseek.com/v1 | models=deepseek-v4-pro, deepseek-v4-flash, deepseek-reasoner
```

> **保持此终端窗口打开**，关闭终端会导致代理停止。也可设为开机自启（见文末进阶技巧）。

### 步骤 6：启动 Codex 桌面版

从开始菜单或桌面快捷方式打开 Codex。此时应直接进入工作界面，不再弹出登录窗口。

### 步骤 7：切换模型

在 Codex 对话中输入：

```
/model deepseek-v4-pro     # 推理模型，适合复杂编码任务
/model deepseek-v4-flash    # 快速模型，适合简单编辑
```

---

## 五、验证

用 CLI 做个端到端测试：

```bash
codex exec "回复一个字：好"
```

预期输出类似：

```
OpenAI Codex v0.129.0 (research preview)
model: deepseek-v4-pro
provider: local_proxy
...
codex
好
```

如果看到 `好` 说明整条链路通了。

---

## 六、常见问题

### Q1：桌面版仍然弹登录界面，提示"请登录 ChatGPT"

这是 Codex 桌面版的已知 bug（[GitHub Issue #2450](https://github.com/openai/codex/issues/2450)）。推荐使用 **CC Switch** 绕过：

1. 下载 [CC Switch](https://github.com/farion1231/cc-switch/releases/latest) 的 `.msi` 安装包
2. 安装后打开，进入 Codex 选项卡 → 添加供应商
3. 填写：
   - API Key：你的 DeepSeek 密钥
   - Base URL：`http://127.0.0.1:4000/v1`
4. 点击启用，重启 Codex

### Q2：`wire_api = chat is no longer supported`

Codex v0.81.0 起已移除对 `wire_api = "chat"` 的支持。确认 `config.toml` 中写的是 `wire_api = "responses"`（代理会自动翻译为 Chat Completions）。

### Q3：端口 4000 被占用

```bash
# 查看占用端口的进程
netstat -ano | findstr ":4000"

# 结束进程（替换 PID）
taskkill /PID <PID> /F
```

或在 `.env` 中修改端口：`PROXY_PORT=4001`，同时更新 `config.toml` 中的 `base_url`。

### Q4：代理启动后提示连接超时

检查：
1. DeepSeek API Key 是否正确（`.env` 文件中）
2. 网络能否访问 `https://api.deepseek.com`
3. 防火墙是否放行了 Node.js

### Q5：关闭终端后代理就停了

参考下方"代理设为开机自启"。

---

## 七、进阶技巧

### 代理设为开机自启（Windows）

**方法一：任务计划程序**

```powershell
# 创建开机启动任务
$action = New-ScheduledTaskAction -Execute "node" `
  -Argument "--env-file=$env:USERPROFILE\.codex\codex-bridge\.env $env:USERPROFILE\.codex\codex-bridge\proxy.mjs" `
  -WorkingDirectory "$env:USERPROFILE\.codex\codex-bridge"

$trigger = New-ScheduledTaskTrigger -AtLogon

Register-ScheduledTask -TaskName "CodexBridge" -Action $action -Trigger $trigger `
  -Description "Codex → DeepSeek 协议代理" -RunLevel Highest
```

**方法二：创建快捷方式放入启动文件夹**

```
%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup
```

快捷方式目标：
```
node --env-file=.env proxy.mjs
```
起始位置：
```
C:\Users\<用户名>\.codex\codex-bridge
```

### 同时使用多个模型

在 `.env` 中修改模型列表即可：

```bash
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner
```

重启代理后，在 Codex 中 `/model` 切换。

---

## 八、总结

| 对比维度 | 本方案 | 降级 Codex 到 0.80.0 |
|---|---|---|
| Codex 版本 | 最新版 | 停留在旧版 |
| 新特性（Computer Use 等） | 可用 | 不可用 |
| 稳定性 | 代理维护中 | 旧版本不再更新 |
| 复杂度 | 中等 | 低（但功能受限） |

通过本地代理的方式，我们既保留了 Codex 最新版的所有特性，又享受到了 DeepSeek V4 的高性价比。代理本身只有 2000 行 Node.js 代码，零依赖，运行开销极小（内存 ~30MB）。

---

> **作者**：[你的名字]
> **日期**：2026-05-10
> **工具版本**：Codex v0.129.0 / codex-bridge v1.0.0 / DeepSeek V4

---

## 参考资料

- [codex-bridge GitHub](https://github.com/wujfeng712-ui/codex-bridge)
- [Codex 官方仓库](https://github.com/openai/codex)
- [DeepSeek API 文档](https://platform.deepseek.com/api-docs)
- [CC Switch](https://github.com/farion1231/cc-switch)
- [Codex Issue #2450 - 登录强制弹窗](https://github.com/openai/codex/issues/2450)
