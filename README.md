# 桌面版 Codex（ChatGPT.app）配置第三方供应商，绕过官方 Codex 额度

> 适用：macOS 上 ChatGPT.app 内置的**桌面版 Codex**（不是 Codex CLI）
> 目标：通过第三方供应商（如兰博官方、西瓜云等）使用 Codex，避免消耗官方 Codex 额度

---

## 背景与痛点

很多同学用 CC Switch 之类的工具，把 Codex 的模型指向第三方供应商，结果一打开 Codex 还是报：

- `The 'gpt-5.4' model is not supported when using Codex with a ChatGPT account.`
- `401 Unauthorized: Incorrect API key ... url: https://api.openai.com/v1/responses`

也就是说请求**仍然打到了官方 OpenAI**，第三方额度根本没用上，官方额度照样扣。

本项目记录了完整根因和一套可靠的手动配置方案。

---

## 根因（踩了四个坑才查清）

1. **CC Switch 接管会把 `auth.json` 强写成 `chatgpt` 模式。**
   桌面版 Codex 在 `chatgpt` 模式下会**硬编码连官方 OAuth、完全无视 `base_url`**。
   （Codex CLI 在 chatgpt 模式下会用 `base_url` 走代理，但桌面版不会——这是死结。）

2. **桌面版 Codex 不读 `config.toml` 顶层的 `base_url`。**
   它走的是「model provider」路由机制，真正生效的字段是 **`openai_base_url`**（给内置 openai provider）或 `[model_providers.<name>]` 自定义块。

3. **内置 `openai` provider 不能用 `[model_providers.openai]` 覆盖。**
   会报 `reserved built-in provider IDs`，导致整个 config 加载失败 → 回退官方 → 又 401。

4. **多数第三方不支持 responses 的 WebSocket。**
   Codex 先试 WS（404）再回退 HTTPS，每次请求约多 **12 秒**，但功能正常。

---

## 解决方案：手动配置（绕过 CC Switch 接管）

既然 CC Switch 对桌面版 Codex 的接管有死结，最稳的是**手动写配置，让 Codex 以 `apikey` 模式直连第三方**，同时让 CC Switch 别再碰 Codex。

### 1. 准备第三方供应商信息

你需要：

- `base_url`：例如兰博 `https://api.lanbuff.top/v1`
- `api_key`：第三方给你的密钥

### 2. 修改 `~/.codex/config.toml`

关键是在文件里加 `openai_base_url`，把内置 openai provider 指向你的第三方：

```toml
model = "gpt-5.4"
base_url = "https://api.lanbuff.top/v1"        # 桌面版会忽略这行，保留无害
openai_base_url = "https://api.lanbuff.top/v1" # ← 真正生效：让 openai provider 走第三方
```

> ⚠️ 注意：不要把 `openai_base_url` 写成带 `/responses` 后缀。
> Codex 会自动补路径，例如 `https://api.lanbuff.top/v1/responses` 会被拼成
> `.../v1/responses/responses` 导致 404。

### 3. 修改 `~/.codex/auth.json`

改成 `apikey` 模式，填入第三方 key：

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "你的第三方供应商 API Key"
}
```

### 4. （可选）如果用自定义 provider 块 + 环境变量

如果你想用 `[model_providers.xxx]` 显式指定，且用 `env_key` 从环境变量读 key：
**GUI 应用读不到 shell 的 env**，需要注入：

```bash
launchctl setenv CODEX_THIRDPARTY_API_KEY "你的key"
# 同时把下面这行加进 ~/.zshrc（从 auth.json 读取，单一来源）：
# export CODEX_THIRDPARTY_API_KEY="$(python3 -c "import json;print(json.load(open('/Users/hope/.codex/auth.json'))['OPENAI_API_KEY'])")"
```

> 但注意：用 `<provider>/<model>` 显式模型名时，若该模型也在 OpenAI 目录里（如 gpt-5.4），
> Codex 会把它解析回内置 openai provider，自定义块可能不生效。
> 所以**最可靠的还是第 2 步的 `openai_base_url` 路线**。

### 5. 让 CC Switch 别再接管 Codex

编辑 `~/.cc-switch/settings.json`，把 `visibleApps.codex` 设为 `false`，
并在 CC Switch 数据库里关掉 codex 代理：

```bash
sqlite3 ~/.cc-switch/cc-switch.db "UPDATE proxy_config SET proxy_enabled=0, enabled=0, live_takeover_active=0 WHERE app_type='codex';"
```

否则 CC Switch 每次启动都会把 `auth.json` 改回 `chatgpt` 模式，前功尽弃。

### 6. 重启生效

完全退出 ChatGPT.app（`Cmd+Q`），再重新打开，**开一个新对话**测试。

---

## 验证

### 方法 A：用内嵌 codex 二进制直接发请求

看日志里 `url:` 是第三方还是官方：

```bash
B="/Applications/ChatGPT.app/Contents/Resources/codex"
cd ~
"$B" exec --skip-git-repo-check --model gpt-5.4 "ping" < /dev/null
```

- 看到 `url: https://api.lanbuff.top/v1/responses` → ✅ 路由成功
- 看到 `url: https://api.openai.com/v1/responses` → ❌ 仍在走官方，检查上面步骤

（注意：日志里会出现 `failed to connect to websocket ... 404` 然后
`Falling back from WebSockets to HTTPS transport`，这是正常的 WS 回退，不是错误。）

### 方法 B：直接 curl 验证 key 有效

```bash
curl -s -X POST https://api.lanbuff.top/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 你的key" \
  -d '{"model":"gpt-5.4","input":"ping","stream":false}'
```

返回合法 JSON（含 `"object":"response"`）即说明 key 与地址都正确。

---

## 常见问题

**Q: 报 `429 Too Many Requests` / `503 无可用渠道`？**
A: 这是**第三方供应商侧**的限流或分组无渠道，与配置无关。等几分钟重试，
或换一个供应商（改 `openai_base_url` + `auth.json` 的 key 即可）。

**Q: 另一个供应商报 `Insufficient balance`？**
A: 余额不足，需要充值。

**Q: 每次请求要等 12 秒？**
A: 因为第三方不支持 responses 的 WebSocket，Codex 先试 WS（404）再回退 HTTPS。
功能正常，只是慢。目前无法干净关闭（内置 openai provider 不可覆盖）。
想零延迟，改用 **Codex CLI**（终端跑 `codex`），它对 CC Switch 接管正常，且不试 WS。

**Q: CC Switch 还能用吗？**
A: 能。Claude / Claude Desktop 仍由 CC Switch 正常管理。
只是桌面版 Codex 被移出了它的接管范围（见第 5 步）。
想恢复 CC Switch 对 Codex 的一键切换，改用 Codex CLI 最稳。

---

## 配置模板

见仓库 `examples/` 目录：

- [`examples/config.toml`](examples/config.toml) — `~/.codex/config.toml` 模板
- [`examples/auth.json`](examples/auth.json) — `~/.codex/auth.json` 模板

---

## 许可证

本仓库当前未包含 LICENSE 文件。如需在你的项目中使用或二次分发，请先与作者联系确认。
建议自行添加合适的开源协议（如 MIT）后再分发。
