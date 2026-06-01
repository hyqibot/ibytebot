# Cursor PR 机器人：完整使用指南

仓库：[hyqibot/ibytebot](https://github.com/hyqibot/ibytebot)

在 Pull Request 里评论 `/cursor` + 任务说明，GitHub Actions 会：

1. checkout **当前 PR 分支**
2. 安装 Cursor CLI，启动 Agent（可读全仓库、按权限改文件）
3. Agent 读 PR diff、改代码、**commit + push 回同一条 PR 分支**
4. 你在 PR 里 review，满意后 **Merge** 才进入 `main`

与 cc-haha 私有仓的手动 `Cursor Agent` workflow **相互独立**。

---

## 名词解释

### 分支（branch）

Git 里的一条「工作线」。  
`main` 是主分支（稳定版）；你在旁边开新分支做任务，避免直接弄乱 `main`。

类比：`main` = 已出版的书；其他分支 = 正在写的草稿。

### 开功能分支（feature branch）

从 `main` 拉出一条新分支，专门做一件事（新功能、修 bug、项目审查等）。

```powershell
git checkout main
git pull ibytebot main
git checkout -b feat/某功能名    # 这就是「开功能分支」
```

分支名随意，常见前缀：`feat/`、`fix/`、`audit/`、`chore/`。

### 开 PR（Open Pull Request，打开合并请求）

在 GitHub 网页上发起：**「请把我的分支改动合进 `main`」**。

- 地址入口：https://github.com/hyqibot/ibytebot/compare  
- 选 `base: main` ← `compare: 你的分支` → **Create pull request**

PR 页面包含：

| 区域 | 作用 |
|------|------|
| **Conversation** | 讨论、发 `/cursor` 指令 |
| **Files changed** | 查看 Agent / 你改了什么 |
| **Merge pull request** | 你最终批准，合入 `main` |

### 「审查用 PR」（任务容器）

**不是新东西**，就是一条**用途为「审查 / 方案 / 多轮 Cursor 任务」的功能分支 + PR**。

- 功能分支 = 草稿纸  
- PR = 把草稿提交审核的「文件夹 + 评论区」  
- 叫「任务容器」是因为：所有 `/cursor` 的改动都堆在这条 PR 上，**不 Merge 就不会进 `main`**

和功能开发的唯一区别是**分支名和目的**（例如 `audit/project-improvements`），**操作完全一样**。

---

## 一次性配置（GitHub）

### 1. Secret

**Settings → Secrets and variables → Actions**

| Name | Value |
|------|--------|
| `CURSOR_API_KEY` | [Cursor Dashboard](https://cursor.com) 生成的 API Key |

### 2. Actions 权限

**Settings → Actions → General → Workflow permissions**

选择 **Read and write permissions**。

### 3. Agent 文件权限（`.cursor/cli.json`）

当前策略：

| 类型 | 规则 |
|------|------|
| 可读 | 整个仓库 `Read(**/*)` |
| 可写 | 整个仓库 `Write(**/*)` |
| 禁止写 | `.env`、`*secret*`、`credentials*`、`.key` 等敏感路径 |
| 禁止 Shell | `gh`（由 workflow 负责 PR 评论，不由 Agent 调） |

修改后需 **push 到 `main`** 才对新触发的 `/cursor` 生效。

---

## 标准流程：审查 → 方案 → 批准 → 执行 → 合入

适用场景：让 Cursor **先看全项目、出改善方案，你批准后再改代码**。

### 流程图

```text
main
  │
  ├─ ① 开功能分支（如 audit/project-improvements）
  │
  ├─ ② 在 GitHub 开 PR（任务容器）
  │
  ├─ ③ PR 评论 /cursor → 只写 audit-plan.md（出方案）
  │       ↓
  ├─ ④ 你看 Files changed，在评论里写「批准 P1-2、P1-5」
  │       ↓
  ├─ ⑤ 再评论 /cursor → 按批准项改代码
  │       ↓
  ├─ ⑥ Files changed 复审代码
  │       ↓
  └─ ⑦ Merge PR → 进入 main
```

### ① 开功能分支并推送

**必须在本地执行一次**（或 GitHub 网页创建分支），因为 PR 需要「一条已存在的远程分支」。

最简方式（**你本地还没有任何改动**时，用「空提交」把分支推上去即可）：

```powershell
cd d:\bytebot
git checkout main
git pull ibytebot main
git checkout -b audit/project-improvements
git commit --allow-empty -m "chore: start project audit PR"
git push -u ibytebot audit/project-improvements
```

**另一种常见情况**：开分支前，**你已经在本地改了一些文件**（例如手动改了 README、修了某个 bug）。这时：

- **仍然必须开功能分支**（不能直接在 `main` 上 push 再开 PR）
- 只是**不需要** `--allow-empty`，改成提交你真实改动的文件：

```powershell
cd d:\bytebot
git checkout main
git pull ibytebot main
git checkout -b audit/project-improvements
# 此处假设你已经用编辑器改好了某些文件
git add .
git commit -m "chore: start audit with my local edits"
git push -u ibytebot audit/project-improvements
```

总结：**无论代码是你自己改的，还是之后交给 `/cursor` 改，流程都是「先开分支 → push → 开 PR」**。上面两种写法差别仅在于——分支上第一个 commit 是「空的占位」还是「你已有的本地改动」。

### ② 在 GitHub 开 PR

浏览器打开：

https://github.com/hyqibot/ibytebot/compare/main...audit/project-improvements

→ **Create pull request** → 标题如 `audit: project improvement plan` → Description 可空 → 创建。

### ③ 第一次 `/cursor`：出方案（先不大改业务代码）

在 PR **Conversation** 评论：

```text
/cursor 请全面审查本仓库，找出可改进点。

要求：
1. 阅读 packages/、docs/、docker/、helm/、.github/ 等目录。
2. 将结论写入 docs/audit-plan-YYYY-MM-DD.md，包含：
   - 问题清单（P0/P1/P2）
   - 每项：现状、风险、建议改法、影响范围、预估工作量
   - 建议执行顺序
3. 本步骤仅新增/修改 audit-plan 文档，不要改业务代码。
4. commit 并 push 到当前 PR 分支。
```

等 Actions **Cursor PR Comment** 完成，看到 ✅ 回复。

### ④ 你批准方案

打开 **Files changed** 看 `docs/audit-plan-*.md`，然后在 **Conversation** 写：

```text
批准执行：P1-2、P1-5、P2-1
暂不执行：P0-3、P2-4
```

### ⑤ 第二次 `/cursor`：按批准项执行

```text
/cursor 请按 docs/audit-plan-YYYY-MM-DD.md 执行我已批准的项。

要求：
1. 逐项修改代码，改动范围不限（全仓库均可）。
2. 每完成一项，在 audit-plan 中标记 done 并注明改动文件。
3. 不要执行未批准项；不要写 .env 或密钥文件。
4. commit 并 push 到当前 PR 分支。
```

可多次重复 ④⑤，直到满意。

### ⑥ ⑦ 复审并 Merge

**Files changed** 确认无误 → **Merge pull request** → 本地：

```powershell
git checkout main
git pull ibytebot main
```

---

## 日常小改动（单轮 `/cursor`）

1. 开功能分支 → push  
2. 开 PR  
3. 评论一条 `/cursor` 说明要改什么  
4. review → Merge  

示例：

```text
/cursor 把 packages/bytebot-ui 里登录页标题改成中文，并补一个最小单元测试。
```

---

## 关于 `/tmp/pr.diff`

**只存在于 GitHub Actions 的 Linux runner**，是 workflow 临时生成的 PR 差异文件，**你电脑上没有**。

- Agent 在 CI 里读它  
- 你在本地/网页看同样内容：**PR → Files changed**，或 `git diff main...你的分支`

评论里写「读 /tmp/pr.diff」是写给 **CI 里的 Agent** 的，不是让你在本机找这个路径。

---

## 安全与限制

| 规则 | 说明 |
|------|------|
| 谁可触发 | 仅 OWNER / MEMBER / COLLABORATOR（见下方说明） |
| Fork PR | **不支持**（无法 push 到别人 fork） |
| 敏感文件 | Agent 不能写 `.env`、密钥等（`.cursor/cli.json` deny） |
| 合入 main | 只有你在 PR 点 **Merge** 才会合入 |
| API 额度 | 消耗 `CURSOR_API_KEY` 对应 Cursor 额度 |
| 大仓库 | 建议分模块多轮审查；workflow 内 PR diff 超过约 500 行会截断 |

### 「谁可触发 `/cursor`」要在 GitHub 里单独设置吗？

**不用。** 这是 workflow 里写死的条件，GitHub 会根据**发评论的人**自动判断身份：

| 身份 | 通常指谁 | 能否触发 |
|------|----------|----------|
| **OWNER** | 仓库拥有者（你） | ✅ |
| **MEMBER** | 组织仓库的成员 | ✅ |
| **COLLABORATOR** | 被你邀请的协作者 | ✅ |
| **NONE** / 路人 | 未授权的普通 GitHub 用户 | ❌ |

代码位置：`.github/workflows/cursor-pr-comment.yml` 里的 `author_association` 判断。  
若要让其他人也能触发，在 GitHub **Settings → Collaborators** 邀请为协作者即可，**不用改 workflow**。

---

## 相关文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/cursor-pr-comment.yml` | `/cursor` 触发与 CI 流程 |
| `.cursor/cli.json` | Agent 读写 / Shell 权限 |
| `README.md` | 仓库首页简要说明 |

---

## 排查

```powershell
# 需安装 GitHub CLI (gh)
gh run list -R hyqibot/ibytebot -w "Cursor PR Comment" -L 5
```

| 报错 | 处理 |
|------|------|
| `not a git repository` | 已修复：需 `actions/checkout` |
| `Workspace Trust Required` | agent 需加 `--trust` |
| `Missing CURSOR_API_KEY` | 配置 Secret |
| fork 跨仓库 PR | 改用同仓库分支 PR |
| 权限拒绝写某文件 | 查 `.cursor/cli.json` 的 deny 规则 |

---

## 快速对照表

| 你想做的事 | 怎么做 |
|------------|--------|
| 让 Cursor 改代码 | 开功能分支 → 开 PR → 评论 `/cursor` |
| 先出方案再改 | 同一 PR 发两次 `/cursor`（先 docs，后代码） |
| 批准方案 | 在 PR Conversation 写「批准 xxx」 |
| 批准上线 | Merge PR |
| 看改了什么 | PR → Files changed |
| 扩大可改范围 | 改 `.cursor/cli.json` → push `main` |
