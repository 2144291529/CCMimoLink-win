# CCMimoLink Windows 使用指南

> 本文档解释 CCMimoLink 在 Windows 上**到底是怎么接管 Codex 流量的**，以及推荐的 A 方式使用流程。看完你应该能完全理解每一步背后系统里发生了什么。

---

## 1. 背景：Codex / cc-switch / 小米 MiMo 三者的关系

| 角色 | 它在哪 | 它的职责 |
|---|---|---|
| **Codex CLI** | 全局命令 | 启动时读 `~/.codex/config.toml`，按里面的 `base_url` 发请求 |
| **cc-switch** | `~/.cc-switch/cc-switch.db`（SQLite）+ GUI | 管理多个 provider（OpenAI / MiMo / DeepSeek 等）。"切换 provider"动作的本质是把数据库里某个 provider 的 TOML 片段**拷贝写入** `~/.codex/config.toml` |
| **小米 MiMo 上游** | `https://token-plan-cn.xiaomimimo.com/v1` | 真正提供模型推理。**只接受 Chat Completions 协议**，不认识 Codex 用的 Responses 协议 |
| **CCMimoLink**（本程序） | `ccmimolink.exe` | 在本地起一个 HTTP 代理（默认 `127.0.0.1:9876`），同时**改写**两边的配置文件让 Codex 流量绕道过来；代理本身负责 Codex Responses 协议 ↔ MiMo Chat Completions 协议的双向翻译 |

**关键认知**：Codex 自己**不读 cc-switch 数据库**。Codex 永远只看 `~/.codex/config.toml`。cc-switch 的所有"切换"都是在替 Codex 改这个文件。

---

## 2. CCMimoLink 干的事 = 两个改写 + 一个代理

### 改写 ①：cc-switch 数据库
把数据库里 MiMo provider 自带的 TOML 片段中的 `base_url` 从小米官方地址改成 `http://127.0.0.1:9876/v1`，同时更新 `provider_endpoints` 表对应行的 url。这样 cc-switch GUI 显示的、以及之后再次"切换 provider"写出去的，都是本地代理地址。

### 改写 ②：`~/.codex/config.toml`
- 找到 cc-switch 里 MiMo provider 声明的 `model_provider = "<name>"`（你这台机器是 `custom`），定位到 `[model_providers.<name>]` section
- 把 `base_url` 改成 `http://127.0.0.1:9876/v1`
- 在文件末尾追加 `[model_providers.<name>.http_headers]`，写入 `Authorization = "Bearer local-mimo-proxy"` 和 `X-Mimo-Api-Key = "<真 key>"`
- 改写之前自动备份成 `config.toml.bak.<时间戳>`

### 代理 ③：监听 `127.0.0.1:9876`
注册路由 `POST /v1/responses`、`GET /v1/models`、`GET /health` 等。每个请求进来时做协议翻译并转发给 MiMo 上游，再把上游的响应翻回 Codex 期望的格式。

---

## 3. A 方式：完整流程

### 前置条件

- 已在 cc-switch GUI 里添加 **Xiaomi MiMo Token Plan (China)** provider
- 已在该 provider 里填入 MiMo 的 API Key
- 已在 cc-switch 里把 Codex 的当前 provider **切到 MiMo**（settings.json 里 `currentProviderCodex` 是 MiMo 的 UUID）

### 步骤

```bash
cd D:\ai-work\CCMimoLink-main

# 1. 同步配置（改写 cc-switch 数据库 + Codex config.toml，不启动代理）
.\ccmimolink.exe --sync-only

# 2. 启动代理（独立终端窗口，进程要一直留着）
.\ccmimolink.exe
```

注意：`ccmimolink.exe` 不带 flag 时其实**也会先跑一遍同步逻辑再起代理**。所以你完全可以跳过第 1 步直接 `.\ccmimolink.exe`。先跑 `--sync-only` 只是为了让你能先看一眼改对没对，再决定要不要正式启动。

### 必做：重启两个进程

- **重启 cc-switch GUI** —— GUI 启动时把数据库读进内存，之后不再读。改完数据库不重启，GUI 显示的还是旧地址（功能不影响，但会让你 confused）
- **重启 Codex CLI** —— 同理，Codex 启动时把 `config.toml` 整个读进内存。不重启 Codex 看不到新的 base_url 和 http_headers

之后正常用 Codex，请求自动走本地代理 → MiMo 上游。

### 看日志

```bash
type ccmimolink.log
# 或 PowerShell 实时跟随
Get-Content ccmimolink.log -Wait
```

### 健康检查

代理活着的时候浏览器打开 `http://127.0.0.1:9876/health` 应该看到 `ok`。

---

## 4. 一次 Codex 请求的完整路径

你在 Codex CLI 里输 "帮我写个 hello world"：

```
[1] Codex 读 ~/.codex/config.toml
    base_url = http://127.0.0.1:9876/v1
    构造请求：
      POST http://127.0.0.1:9876/v1/responses
      Authorization: Bearer local-mimo-proxy   （来自 http_headers）
      X-Mimo-Api-Key: tp-cp9io...gk            （来自 http_headers）
      Body: { model, instructions, input, stream, tools, ... }   ← Responses 协议

[2] CCMimoLink.exe（监听 9876）的 handleResponses 接收：
      a. 解析 input → 转成 Chat Completions 风格的 messages 数组
      b. 把 instructions 注入为 system message
      c. tools 规范化，过滤 MiMo 不支持的 built-in tool
      d. 选模型：检测到图片自动用 mimo-v2.5；否则用 MIMO_MODEL（默认 mimo-v2.5）
      e. 从入站 X-Mimo-Api-Key 头拿到真 key
      f. 限流闸口：限制并发 + 强制最小请求间隔（默认 1.5s）

[3] CCMimoLink 转发到上游：
      POST https://token-plan-cn.xiaomimimo.com/v1/chat/completions
      Authorization: Bearer tp-cp9io...gk      ← 这里换成了真 key
      Body: { model, messages, stream, tools, ... }   ← Chat Completions 协议

[4] MiMo 上游返回 SSE 流（chat.completion.chunk 格式）
    CCMimoLink 边读边把每个 chunk 翻译成 Responses 风格事件：
      response.created
      response.output_item.added
      response.output_text.delta    ← Codex 用这个 stream 显示
      response.output_text.done
      response.completed

[5] Codex 收到 Responses 风格事件流 → 在终端实时打印答案
```

**两边都被骗了**：Codex 以为它在和一个 OpenAI Responses 兼容服务说话；MiMo 以为它在服务一个普通的 Chat Completions 客户端。CCMimoLink 在中间做了协议双向翻译。

---

## 5. 配置对比表（A 方式跑完后）

| 文件 / 位置 | A 方式之前 | A 方式之后 |
|---|---|---|
| `~/.cc-switch/cc-switch.db` 里 MiMo provider 的 base_url | `https://token-plan-cn.xiaomimimo.com/v1` | `http://127.0.0.1:9876/v1` |
| `~/.codex/config.toml` 的 `[model_providers.custom].base_url` | `http://127.0.0.1:15721/v1`（被你那个 15721 工具占着） | `http://127.0.0.1:9876/v1` |
| `~/.codex/config.toml` 的 `[model_providers.custom.http_headers]` | 不存在 | 新增 Authorization + X-Mimo-Api-Key |
| `127.0.0.1:9876` 上是否有进程监听 | 无 | 有（`ccmimolink.exe`） |
| Codex 请求实际去哪儿 | 那个 15721 工具 | `ccmimolink.exe` → 协议翻译 → 小米 MiMo 上游 |
| 关掉 `ccmimolink.exe` 进程后 | 无影响 | Codex 报连接拒绝（9876 没人接） |

---

## 6. 重要：本机的"撞 section"问题

你 `~/.codex/config.toml` 里的 `[model_providers.custom]` 现在被另一个工具（监听 `127.0.0.1:15721`）占用，而 cc-switch 里 MiMo provider 也声明 `model_provider = "custom"` —— **两个工具用同一个 section 名**。

**结论**：CCMimoLink 和那个 15721 工具**不能同时使用**。要切就得改 base_url。

### 切回 15721 工具的两种方式

**方式 1：用 cc-switch GUI 切走 MiMo**（推荐）
1. cc-switch GUI 里把 Codex 的 provider 切到一个**非 MiMo** 的 provider（比如 OpenAI Official、xff、DD2API 中你给那个 15721 工具用的那个）
2. 重启 Codex CLI

**方式 2：手动还原 config.toml**
```bash
# 看看最近的备份（CCMimoLink 每次 sync 都生成一份）
dir %USERPROFILE%\.codex\config.toml.bak.*

# 选一份覆盖回去
copy %USERPROFILE%\.codex\config.toml.bak.<时间戳> %USERPROFILE%\.codex\config.toml
```

### 想要彻底分开（不撞）？

让 cc-switch 里 MiMo provider 用一个独立的 section 名。打开 cc-switch GUI，编辑 MiMo provider 的 TOML 片段：

```toml
# 改之前
model_provider = "custom"
[model_providers.custom]
name = "xiaomi_mimo_token_plan"
...

# 改之后
model_provider = "mimo"
[model_providers.mimo]
name = "xiaomi_mimo_token_plan"
...
```

之后再跑 CCMimoLink，它会自动检测到 `model_provider = "mimo"`，去改 `[model_providers.mimo]` 这个独立 section，和 `[model_providers.custom]` 互不干扰。

---

## 7. 切换模型 / 端口 / 限流

### 切换 mimo-v2.5 / mimo-v2.5-pro

```bash
# 默认 mimo-v2.5
.\ccmimolink.exe

# 切到 pro
.\ccmimolink.exe --v2.5-pro

# 或用环境变量
set MIMO_MODEL=mimo-v2.5-pro
.\ccmimolink.exe
```

请求里带图片时不管你设啥都自动回落 `mimo-v2.5`（pro 不支持图）。

### 常用环境变量（PowerShell 写法）

```powershell
$env:MIMO_PROXY_PORT = "19876"               # 改本地端口（默认 9876）
$env:MIMO_PROXY_LOG = "D:\logs\ccmimo.log"   # 改日志位置
$env:MIMO_PROXY_MAX_CONCURRENT = "2"         # 上游最大并发（默认 1）
$env:MIMO_PROXY_MIN_INTERVAL_MS = "1000"     # 上游最小请求间隔毫秒（默认 1500）
$env:MIMO_PROXY_429_BACKOFF_MS = "30000"     # 429 退避毫秒（默认 30000）
$env:MIMO_PROXY_SKIP_CC_SWITCH_SYNC = "true" # 跳过 sync（仅启动代理）
.\ccmimolink.exe
```

改了端口的话，需要重新跑 `--sync-only` 让 cc-switch 数据库和 Codex config.toml 也跟着改。

---

## 8. 排错

| 现象 | 原因 / 处理 |
|---|---|
| 启动报 `cc switch is not installed or incomplete` | 检查 `%USERPROFILE%\.cc-switch\cc-switch.db` 和 `%USERPROFILE%\.codex\config.toml` 是否都存在 |
| `Xiaomi MiMo provider not found` | cc-switch GUI 里没添加 MiMo provider，先去添加 |
| `Xiaomi MiMo API key is empty` | cc-switch 里 MiMo provider 没填 key |
| Codex 请求 connection refused | `ccmimolink.exe` 进程没在跑（必须保留这个终端窗口） |
| Codex 还是连旧地址 | 没重启 Codex CLI（或者你跑的旧版还没退出） |
| cc-switch GUI 还显示旧 base_url | 没重启 cc-switch GUI |
| 端口 9876 被占用 | `set MIMO_PROXY_PORT=19876` 然后重新 `--sync-only` 再启动 |
| 想完全卸载、还原 | 用最早那份 `config.toml.bak.<时间戳>` 覆盖回 `config.toml`，cc-switch GUI 里把 MiMo provider 的 base_url 手动改回 `https://token-plan-cn.xiaomimimo.com/v1` |

---

## 9. 总结

CCMimoLink = **本地代理 + 配置改写器**。

它做的事可以概括成一句话：
> 把 cc-switch 和 Codex 的配置都改到指向本地的 9876 端口，然后自己在 9876 上接住 Codex 的 Responses 协议请求，翻译成 MiMo 能听懂的 Chat Completions 协议转出去。

只要 cc-switch 当前激活的是 MiMo provider，跑一次 `.\ccmimolink.exe`（或先 `--sync-only` 再正式启动），重启 cc-switch GUI 和 Codex CLI，就能在 Codex 里用上小米 MiMo。
