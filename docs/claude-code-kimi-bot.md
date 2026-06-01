# PR 评论 `/claude`：Claude Code + Kimi

## 原理（一句话）

**Claude Code** 是 Agent 框架（读代码、跑命令、改文件、git 提交）；**Kimi** 通过 Moonshot **Anthropic 兼容接口**当底层模型，不调用 Claude 官方 API。

```text
GitHub PR 评论 /claude → Actions 跑 claude CLI → 请求发到 api.moonshot.cn/anthropic → Kimi 模型
```

与 `/cursor`（Cursor Agent CLI）**并行独立**，可二选一或同时保留。

---

## 一次性配置

仓库：[hyqibot/ibytebot](https://github.com/hyqibot/ibytebot)

### 1. Secret

**Settings → Secrets and variables → Actions**

| Name | Value |
|------|--------|
| `MOONSHOT_API_KEY` | [Kimi 开放平台](https://platform.moonshot.cn/console/api-keys) 创建的 API Key |

### 2. Actions 权限

**Settings → Actions → General → Workflow permissions** → **Read and write permissions**

### 3. 相关文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/claude-code-kimi.yml` | workflow |
| `docs/cursor-pr-bot.md` | `/cursor` 用法（另一套 Agent） |

---

## 使用方法

与 `/cursor` 相同：**开空分支 → 开 PR → 在 PR Conversation 评论**。

示例：

```text
/claude 请审查本 PR，修复明显的 bug，并 commit push 到当前分支。
```

多轮任务（先方案后执行）流程见 [cursor-pr-bot.md](./cursor-pr-bot.md) 中的标准流程，把 `/cursor` 换成 `/claude` 即可。

---

## 环境变量说明（workflow 内已配置）

| 变量 | 值 | 说明 |
|------|-----|------|
| `ANTHROPIC_BASE_URL` | `https://api.moonshot.cn/anthropic` | 国内 Moonshot Anthropic 兼容端点 |
| `ANTHROPIC_AUTH_TOKEN` | `MOONSHOT_API_KEY` | Kimi 官方文档要求用此变量名 |
| `ANTHROPIC_MODEL` | `kimi-k2.5` | 主模型 |

参考：[Kimi 在 Claude Code 中使用](https://platform.moonshot.cn/docs/guide/agent-support)

---

## 与 `/cursor` 的区别

| | `/claude` | `/cursor` |
|--|-----------|-----------|
| 框架 | Claude Code（`@anthropic-ai/claude-code`） | Cursor Agent CLI |
| 模型 | Kimi（Moonshot） | Cursor 云端模型 |
| Secret | `MOONSHOT_API_KEY` | `CURSOR_API_KEY` |
| 触发词 | `/claude` | `/cursor` |

---

## 排查

失败时打开 https://github.com/hyqibot/ibytebot/actions ，点 **Claude Code (Kimi)** 查看日志。

| 报错 | 处理 |
|------|------|
| `Missing MOONSHOT_API_KEY` | 配置 Secret |
| `Not logged in` / 鉴权失败 | 确认 Key 有效；workflow 已固定 `claude-code@2.1.63`（新版对第三方端点更严） |
| 进程卡住无输出 | 多为工具权限等待；已用 `--permission-mode acceptEdits` + `--allowedTools` |
| fork 跨仓库 PR | 改用同仓库分支 PR |
