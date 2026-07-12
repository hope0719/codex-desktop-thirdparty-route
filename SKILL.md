---
name: codex-desktop-thirdparty-route
description: 修复桌面版 Codex（ChatGPT.app 内）的路由问题。覆盖两种场景：① 直连官方 OpenAI 扣额度（401）；② 通过 CC Switch 代理时出现 429 Too Many Requests。适用：macOS 桌面版 Codex + CC Switch + 兰博/yunwu 等第三方端点。
---

# 桌面版 Codex 第三方供应商路由修复

## 适用场景
用户用 **ChatGPT.app 里的桌面版 Codex**，想通过 CC Switch 或自定义配置走第三方 OpenAI 兼容端点，
但遇到以下问题之一：
- 请求仍打到 `api.openai.com` 扣官方额度（401）
- 通过 CC Switch 代理时持续返回 429 Too Many Requests

## 核心认知（先读，避免走弯路）
1. **CC Switch 的 live takeover 会把 Codex 的 `auth.json` 强制写成 `auth_mode=chatgpt`**。
   桌面版 Codex 在 chatgpt 模式下**硬编码连官方 OAuth、完全无视 base_url**（Codex CLI 不会，但桌面版会）。
   → 所以 CC Switch 对桌面版 Codex 的接管是死结，应放弃让它接管 Codex。
2. **桌面版 Codex 不读 config.toml 顶层 `base_url`**，它走「model provider」路由。
   真正让内置 `openai` provider 改向的字段是 **`openai_base_url`**。
3. **内置 provider（openai / chatgpt）不能用 `[model_providers.openai]` 覆盖**，
   会报 `reserved built-in provider IDs` 导致整个 config 加载失败 → 回退官方 401。
4. 很多第三方端点**不支持 responses WebSocket**，Codex 先试 WS（404/405）再回退 HTTPS，
   每次新对话多 ~5-12s 延迟。
   **解决方案**：在自定义 model provider 中设置 `supports_websockets = false`，跳过 WebSocket 直走 HTTPS。
5. **GUI 应用不继承 shell 的 env**，自定义 provider 的 key 要用 `launchctl setenv` 注入。
6. **CC Switch 代理模式下 `auth.json` 的 key 是 `PROXY_MANAGED` 占位符**，不是真实 API Key。
   如果 `openai_base_url` 指向第三方直连（如 lanbuff），而 key 是 `PROXY_MANAGED`，第三方会返回 401，
   Codex 重试到极限后变成 **429 Too Many Requests**。
7. **Codex v0.144+ 已移除 `wire_api = "chat"`**，仅支持 `wire_api = "responses"`。
   但可以通过 `supports_websockets = false` 跳过 WebSocket 重试。

## 两种路由模式

### 模式 A：CC Switch 代理模式（推荐）
当 CC Switch 的本地代理（默认端口 15721）已配置好第三方供应商时，让 Codex 走代理：
- `openai_base_url` = `http://127.0.0.1:15721/v1`（CC Switch 本地代理）
- `auth.json` 的 key 保持 `PROXY_MANAGED`（CC Switch 自动替换为真实 key）
- CC Switch 代理负责转发到实际端点（如 yunwu.ai / lanbuff）

### 模式 B：直连第三方模式
不用 CC Switch 代理，Codex 直连第三方端点：
- `openai_base_url` = `https://api.lanbuff.top/v1`（直连）
- `auth.json` 必须包含**真实的第三方 API Key**（不能用 `PROXY_MANAGED`）

## 诊断步骤
1. 看当前认证模式和 key：
   ```bash
   grep -E '"auth_mode"|OPENAI_API_KEY' ~/.codex/auth.json
   ```
   - 若为 `chatgpt` → 桌面版会直连官方，必须改成 `apikey`
   - 若 key 为 `PROXY_MANAGED` → 走 CC Switch 代理模式（模式 A）
   - 若 key 为 `sk-xxx` → 可走直连模式（模式 B）

2. 看 config 真正生效的 endpoint：
   ```bash
   grep -nE 'base_url|openai_base_url|model =' ~/.codex/config.toml
   ```

3. 检查 CC Switch 代理是否在运行：
   ```bash
   lsof -i :15721 -P 2>&1 | head -5
   ```
   有 `cc-switch` 进程 LISTEN = 代理在跑。

4. 检查 CC Switch 数据库中 codex 的供应商配置：
   ```bash
   sqlite3 ~/.cc-switch/cc-switch.db "
   SELECT id, name, is_current, provider_type FROM providers WHERE app_type='codex' AND is_current=1;
   "
   ```
   查看实际配置（base_url、key、model）：
   ```bash
   sqlite3 ~/.cc-switch/cc-switch.db "
   SELECT settings_config FROM providers WHERE app_type='codex' AND is_current=1;
   "
   ```

5. 用 curl 直接测试代理是否通：
   ```bash
   # 测试 chat/completions
   curl -s -w "\n%{http_code}" http://127.0.0.1:15721/v1/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer PROXY_MANAGED" \
     -d '{"model":"gpt-5.4","messages":[{"role":"user","content":"hi"}],"stream":false,"max_tokens":5}'

   # 测试 responses API（Codex 用这个）
   curl -s -w "\n%{http_code}" http://127.0.0.1:15721/v1/responses \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer PROXY_MANAGED" \
     -d '{"model":"gpt-5.4","input":"hi","max_output_tokens":5}'
   ```
   返回 200 = 代理正常。

6. 用内嵌 codex 二进制直接验证路由（最关键）：
   ```bash
   B="/Applications/ChatGPT.app/Contents/Resources/codex"
   cd /tmp
   "$B" exec --skip-git-repo-check --model gpt-5.4 "say hello" <<< "" 2>&1 | tail -20
   ```
   - 看到 `url: ws://127.0.0.1:15721/...` 或 `Hello` 输出 = 成功
   - 看到 `url: wss://api.lanbuff.top/...` + 429 = key 不匹配（PROXY_MANAGED 发给了第三方）
   - 看到 `api.openai.com` = 路由未生效

## 修复步骤

### 场景 1：429 Too Many Requests（CC Switch 代理模式下）

**根因**：`openai_base_url` 指向第三方直连（如 lanbuff），但 `auth.json` 的 key 是 `PROXY_MANAGED`，
第三方不认这个占位符 → 401 → Codex 重试到极限 → 429。

**修复**：把 `openai_base_url` 改成 CC Switch 本地代理地址：
```toml
# ~/.codex/config.toml
model = "gpt-5.4"
base_url = "http://127.0.0.1:15721/v1"           # CLI 用
openai_base_url = "http://127.0.0.1:15721/v1"     # ← 桌面版用，改这里！
```

### 场景 1b：每次新对话 WebSocket 重试 5 次（延迟 ~5-12s）

**根因**：Codex v0.144+ 只支持 `wire_api = "responses"`，响应 API 默认先尝试 WebSocket 连接。
CC Switch 代理不支持 WS，返回 405 → Codex 重试 5 次后再回退 HTTPS，每轮对话多等 5-12 秒。

**修复**：在 config.toml 中创建自定义 model provider，设置 `supports_websockets = false` 跳过 WS：

```toml
# ~/.codex/config.toml
model = "gpt-5.4"
base_url = "http://127.0.0.1:15721/v1"
openai_base_url = "http://127.0.0.1:15721/v1"
model_provider = "ccswitch"

[model_providers.ccswitch]
name = "ccswitch"
base_url = "http://127.0.0.1:15721/v1"
wire_api = "responses"
supports_websockets = false        # ← 跳过 WebSocket，直走 HTTPS！
request_max_retries = 0            # 禁用请求重试
stream_max_retries = 0             # 禁用流重试
requires_openai_auth = false
```

这个修复对 **Codex CLI** 有效（`provider: ccswitch`，零 WS 延迟）。
桌面版 Codex 仍走内置 `openai` provider，需 `openai_base_url` 路由。
桌面版 WebSocket 延迟暂无法通过此方式消除（内置 provider 不支持 `supports_websockets`）。

CC Switch 代理会自动把 `PROXY_MANAGED` 替换为数据库中配置的真实 API Key，转发到实际端点。

### 场景 2：直连官方 OpenAI（401）

**根因**：`auth_mode` 为 `chatgpt`，桌面版硬编码连官方 OAuth，无视 base_url。

**修复**：
1. `~/.codex/auth.json` 改为 apikey 模式 + 第三方 key：
   ```json
   {"auth_mode":"apikey","OPENAI_API_KEY":"sk-xxxxx"}
   ```
2. `~/.codex/config.toml`：
   ```toml
   model = "gpt-5.4"
   base_url = "https://api.lanbuff.top/v1"
   openai_base_url = "https://api.lanbuff.top/v1"
   ```
   ⚠️ 不要写 `[model_providers.openai]`（保留 ID 报错）。
3. 注入第三方 key 给 GUI：
   ```bash
   KEY=$(python3 -c "import json;print(json.load(open('/Users/hope/.codex/auth.json'))['OPENAI_API_KEY'])")
   launchctl setenv LANBUFF_API_KEY "$KEY"
   ```
4. 让 CC Switch 别再接管 Codex：
   - `~/.cc-switch/settings.json` 设 `visibleApps.codex=false`
   - CC Switch 数据库 `proxy_config` 表 `live_takeover_active=0`（app_type='codex'）
5. **完全退出重开 ChatGPT.app**（Cmd+Q），开新对话测试。

## 验证
- **成功路由**：Codex 返回正常 content，或第三方侧报错（429「分组上游负载已饱和」/ 503「无可用渠道」），
  这些**是供应商侧问题，与配置无关**。
- **仍 api.openai.com 401**：说明 openai_base_url 未被读或 CC Switch 又把 auth.json 改回 chatgpt，回到诊断步骤。
- **429 持续**：检查 `openai_base_url` 是否和 `auth.json` 的 key 模式匹配——代理地址配 PROXY_MANAGED，直连地址配真实 key。

## 注意
- 桌面版每次请求约 5-12s WebSocket 重试延迟（第三方/代理多无 WS）。**Codex CLI 可通过自定义 provider 的 `supports_websockets = false` 跳过 WS 延迟**。
  桌面版内置 openai provider 不可覆盖此行为。
- `wire_api = "chat"` 已在 Codex v0.144+ 中彻底移除，请勿使用。
- 旧会话线程可能保留旧模型/认证选择，务必开新对话验证。
- CC Switch 代理模式下，实际端点和 key 在 CC Switch 数据库的 `providers` 表 `settings_config` 字段中管理，
  不需要手动改 `auth.json`。
