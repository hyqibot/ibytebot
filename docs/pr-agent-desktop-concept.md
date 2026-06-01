# PR Agent 图形化工具：独立项目方案

本文说明：能否把「开空分支 → 开 PR → 发 `/cursor` / `/claude` → review → Merge」做成 **Windows 图形化 exe**；以及 exe 与 GitHub Actions 的分工。

相关现有文档：

- [cursor-pr-bot.md](./cursor-pr-bot.md) — `/cursor` 手动流程
- [claude-code-kimi-bot.md](./claude-code-kimi-bot.md) — `/claude` 手动流程

---

## 先分清两层

| 层 | 现在怎么做 | 能否做成 exe |
|----|------------|--------------|
| **本地 / 网站操作** | 开分支、push、开 PR、写评论 | ✅ 可以图形化 |
| **云端 Agent** | GitHub Actions 跑 `/cursor` 或 `/claude` | ❌ 不能搬进 exe（必须在仓库 Actions 里） |

exe 的定位是 **「任务容器助手 / 遥控器」**，不是替代 Cursor Agent 或 Claude Code 本身。

```text
┌─────────────────┐
│  图形化 exe（新）  │  ← 开分支、开 PR、发 /cursor / /claude
└────────┬────────┘
         │ GitHub API / git
         ▼
┌─────────────────┐
│  GitHub PR      │  ← 任务容器（审批、看 diff）
└────────┬────────┘
         │ issue_comment 触发
         ▼
┌─────────────────┐
│  GitHub Actions │  ← Cursor / Claude Code 真正读代码、改文件、push
└─────────────────┘
```

---

## 图形化 exe 可一键完成的事

1. 选择本地仓库（如 `d:\bytebot`）
2. 填写任务名 → 自动：`git checkout main` → `pull` → 开空分支 → `push`
3. 调用 GitHub API 自动 **Create pull request**
4. 选择 Agent：**Cursor** / **Claude（Kimi）**
5. 填写任务说明 → 自动在 PR **Conversation** 发评论（`/cursor …` 或 `/claude …`）
6. 轮询 Actions 状态，展示 PR 链接、通过/失败、改动文件
7. 可选：一键 **Merge PR**、本地 `pull main`

用户侧：从「记命令 + 点网页」变为 **点按钮**。

---

## 技术实现（独立项目结构）

| 组件 | 用途 |
|------|------|
| **UI** | Tauri / Electron / WPF（打包为 Windows exe） |
| **Git** | 调用本机 `git` 或 libgit2：开分支、空提交、push |
| **GitHub** | `gh` CLI 或 REST API：开 PR、发评论、查 Actions、Merge |
| **本地配置** | 默认 remote、分支前缀、最近用过的仓库列表 |
| **仓库模板** | 内置 `.github/workflows/`、`.cursor/cli.json` 等，支持「初始化 Agent 仓库」 |

建议目录：

```text
pr-agent-desktop/
  src/              # UI + 流程编排
  templates/        # cursor / claude workflow 模板
  config.json       # 用户配置（不含模型 Key）
```

---

## 与现有两套 Agent 的关系

| 能力 | ibytebot（公开仓） | cc-haha（私有仓） |
|------|-------------------|-------------------|
| PR 评论 `/cursor` | ✅ `cursor-pr-comment.yml` | — |
| PR 评论 `/claude` | ✅ `claude-code-kimi.yml` | — |
| 手动 dispatch Cursor | — | ✅ `cursor-agent.yml` |

图形化 exe 可统一封装：

- **PR 模式**：空分支 → PR → 发 `/cursor` 或 `/claude`（与 [cursor-pr-bot.md](./cursor-pr-bot.md) 流程一致）
- **Dispatch 模式**：对 cc-haha 调 `gh workflow run Cursor Agent`（填 prompt，workflow 推 `cursor/agent-*` 分支）

---

## 最小 MVP（建议先做 4 个按钮）

| 按钮 | 行为 |
|------|------|
| **新建审查 PR** | 空分支 + push + 开 PR |
| **发送 Agent 任务** | 选 cursor/claude，填说明，发 PR 评论 |
| **查看状态** | 打开 Actions run / PR Files changed |
| **合并 PR** | Merge + 提示本地 pull |

后续可加：多仓库、任务历史、审批清单（先方案后执行的两段式模板）。

---

## 局限（设计时需接受）

1. **Agent 仍在 GitHub 云跑**，exe 不能离线改代码。
2. 本机需安装 **Git**；建议安装 **GitHub CLI（gh）** 或配置 **PAT**。
3. **Secrets**（`CURSOR_API_KEY`、`MOONSHOT_API_KEY`）仍在 GitHub 仓库 Settings 里配置，exe **不应**保存模型 Key（更安全）。
4. 规则不变：同仓库 PR（非 fork）、OWNER/协作者才能触发、Merge 前人工 review。
5. API 额度、Actions 分钟数仍消耗 GitHub / Cursor / Kimi 账户配额。

---

## 技术选型参考

| 方案 | 优点 | 缺点 |
|------|------|------|
| **Tauri** | 与 cc-haha 桌面栈一致，体积小 | 需 Rust 环境打包 |
| **WPF / WinUI** | 纯 Windows、原生 exe | 仅 Windows |
| **Electron** | 开发快、跨平台 | 体积大 |

本地需能执行（或内置调用）：

```powershell
git checkout main
git pull origin main
git checkout -b audit/任务名
git commit --allow-empty -m "chore: start agent PR"
git push -u origin audit/任务名
# PR / 评论 由 gh 或 API 完成
```

---

## 配置项（exe 内）

| 配置 | 说明 |
|------|------|
| 本地仓库路径 | 如 `d:\bytebot` |
| GitHub remote | 如 `ibytebot` → `hyqibot/ibytebot` |
| GitHub Token / gh 登录 | 开 PR、发评论、查 Actions |
| 默认分支 | 通常 `main` |
| 分支前缀 | 如 `audit/`、`feat/` |

**不要**在 exe 里存：`CURSOR_API_KEY`、`MOONSHOT_API_KEY`（留在 GitHub Secrets）。

---

## 总结

| 问题 | 答案 |
|------|------|
| 能否做成独立 exe 项目？ | **可以** |
| exe 做什么？ | 自动化开分支、开 PR、发 Agent 评论、看状态、Merge |
| exe 不做什么？ | 不跑 Cursor/Claude Agent，不替代 GitHub Actions |
| 与现有 md 流程关系？ | exe 是 [cursor-pr-bot.md](./cursor-pr-bot.md) 的图形化壳 |

若启动开发，建议仓库名如 `pr-agent-desktop`，首版只做 **ibytebot 的 PR + `/cursor`/`/claude`**，再扩展 cc-haha 的 `workflow_dispatch`。
