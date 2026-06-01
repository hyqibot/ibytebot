# PR 评论 `/cursor` 机器人

在 Pull Request 里评论 `/cursor` + 任务说明，GitHub Actions 会：

1. checkout **当前 PR 分支**
2. 安装 [Cursor CLI](https://cursor.com/docs/cli/reference/configuration)
3. Cursor Agent 读取 PR diff、修改代码
4. **commit + push 回同一条 PR 分支**（不合并 main）

与 cc-haha 私有仓的手动 `Cursor Agent` workflow 独立；本仓库专用。

## 一次性配置（GitHub）

仓库：[hyqibot/ibytebot](https://github.com/hyqibot/ibytebot)

### 1. 添加 Secret

**Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
|------|--------|
| `CURSOR_API_KEY` | [Cursor Dashboard](https://cursor.com) 生成的 API Key |

`GITHUB_TOKEN` 由 Actions 自动提供，无需手动创建。

### 2. Actions 权限

**Settings → Actions → General → Workflow permissions**

选择 **Read and write permissions**（需要 push 到 PR 分支）。

### 3. 推送本仓库代码

若本地仍指向上游 `bytebot-ai/bytebot`，可添加你的远程：

```powershell
cd d:\bytebot
git remote add ibytebot https://github.com/hyqibot/ibytebot.git
git add .github/workflows/cursor-pr-comment.yml .cursor/cli.json docs/cursor-pr-bot.md
git commit -m "feat(ci): add PR comment /cursor Cursor Agent workflow"
git push ibytebot main
```

若 `ibytebot` 远程为空仓库，首次 push 用 `git push -u ibytebot main`。

## 使用方法

1. 开 PR（**同仓库分支**，非 fork）
2. 在 PR 评论里写，例如：

   ```
   /cursor 帮我把 packages/bytebot-ui 里这个按钮文案改成中文，并补一个简单的单元测试
   ```

3. 等待 Actions → **Cursor PR Comment** 跑完
4. PR 分支会出现新 commit，review 后合并

## 安全限制

| 规则 | 说明 |
|------|------|
| 触发者 | 仅 `OWNER` / `MEMBER` / `COLLABORATOR` |
| Fork PR | **不支持**（无法安全 push 到贡献者 fork） |
| Agent 权限 | 见 `.cursor/cli.json`；默认不可改 `.github/workflows`、`.env` |
| 额度 | 消耗仓库 Secret 里的 Cursor API 额度 |

## 相关文件

- `.github/workflows/cursor-pr-comment.yml` — workflow 定义
- `.cursor/cli.json` — Agent 在 CI 中的读写/Shell 权限

## 排查

```powershell
gh run list -R hyqibot/ibytebot -w "Cursor PR Comment" -L 5
```

常见失败：

- `Invalid project config ... cli.json` — 项目级 `cli.json` 只能有 `permissions`，不能有 `version`
- `Missing CURSOR_API_KEY` — 未配置 Secret
- `fork 跨仓库 PR` — 请让贡献者开同仓库分支 PR，或使用 maintainer 分支
