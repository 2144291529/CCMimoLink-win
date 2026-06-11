# Changelog

## v2.0.0 (2026-06-11)

基于上游 [SimonLeen22/CCMimoLink](https://github.com/SimonLeen22/CCMimoLink) v2.0 重建 Windows 版。

### 新增

- **Anthropic Messages 上游协议适配器** (`anthropic_adapter.go`)
  - 通过 `MIMO_UPSTREAM_PROTOCOL=anthropic` 可选使用 Anthropic Messages API 作为上游协议
  - 支持扩展思考（extended thinking）：thinking blocks 流式传输和回放
  - 正确的 OpenAI→Anthropic 消息转换，同角色合并（parallel tool_use 合并为一个 assistant turn）
  - 工具选择映射：auto/required/none/function 全部正确处理
  - 调试 dump：`MIMO_DEBUG_DUMP` 环境变量指定目录
  - 9 个新测试用例

- **describe_image 内建工具**
  - `mimo-v2.5-pro` 收到图片请求时，proxy 自动注入 `describe_image` 工具
  - pro 调用后，proxy 内部用 `mimo-v2.5` 读图，把文字描述回灌给 pro
  - 最多 2 轮 describe_image 循环

- **HTTP 韧性改进**
  - `ResponseHeaderTimeout`（60s）：上游永远不返回响应头时自动终止
  - `idleTimeoutBody`：60s 无读取自动关闭卡住的响应体，释放限流槽位
  - 并发 slot 泄漏修复：`handleAnthropicStream` 固定 defer `closeUpstream`

- **Codex auth.json 支持**
  - 新增 `CODEX_AUTH_PATH` 环境变量（默认 `~/.codex/auth.json`）
  - 启动同步时自动更新 auth.json 中的 OPENAI_API_KEY

- **子命令系统**
  - `./ccmimolink model set mimo-v2.5-pro` — 切换模型
  - `./ccmimolink model status` — 查看当前模型状态
  - `./ccmimolink model restart` — 重启服务
  - `./ccmimolink sync` — 手动同步配置

### 变更

- 默认模型从 `mimo-v2.5` 改为 `mimo-v2.5-pro`
- 默认并发从 1 提升到 4（`MIMO_PROXY_MAX_CONCURRENT`）
- 默认请求间隔从 1500ms 降低到 600ms（`MIMO_PROXY_MIN_INTERVAL_MS`）
- 新增 `MIMO_UPSTREAM_PROTOCOL` 环境变量
- 新增 `MIMO_DEBUG_DUMP` 环境变量

### Windows Fork 特有

- 保留 Manager UI（Web 仪表盘）
- 保留 `--auto-start` 和 `--no-open` 启动参数
- 保留 proxyControl（HTTP 启停代理）
- 使用纯 Go sqlite 驱动（`modernc.org/sqlite`），无需安装 sqlite3 CLI

### 修复

- 上游 v2.0 使用 sqlite3 CLI 导致 Windows 上无法运行，已改回纯 Go 驱动

---

## v0.1.0 (2026-05-31)

首个 Windows 版本发布。

### 功能

- 完整的 Responses → Chat Completions 协议适配
- cc switch 数据库自动同步
- Codex 配置自动改写和备份
- 本地 Manager UI（启停代理、状态查看）
- 纯 Go sqlite 驱动，Windows 免安装
- 跨平台 CI 构建（Linux/Windows/macOS）
