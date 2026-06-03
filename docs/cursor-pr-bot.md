# Cursor PR 机器人：完整使用指南

仓库：[hyqibot/ibytebot](https://github.com/hyqibot/ibytebot)

在 Pull Request 里评论 `/cursor` + 任务说明，GitHub Actions 会：

1. checkout **当前 PR 分支**
2. 安装 Cursor CLI，启动 Agent（可读全仓库、按权限改文件）
3. Agent 读 PR diff、改代码、**commit + push 回同一条 PR 分支**
4. 你在 PR 里 review，满意后 **Merge** 才进入 `main`

---


## 本地 Git remote 说明（bytebot vs cc-haha）

| 项目 | remote 名 | 指向 | 原因 |
|------|-----------|------|------|
| **bytebot**（`d:\bytebot`） | `origin` | [bytebot-ai/bytebot](https://github.com/bytebot-ai/bytebot) | clone 官方上游时的默认名 |
| | `ibytebot` | [hyqibot/ibytebot](https://github.com/hyqibot/ibytebot) | 你实际部署 PR Agent 的 fork，**push 用这个** |
| **cc-haha** | `origin` | hyqibot/claude-code-private | 只有一个仓，默认名即可 |

`git remote` 只是本地别名，不是 GitHub 强制命名。bytebot 因「上游 + 自己的 fork」有两个名字；cc-haha 只对接一个仓所以只有 `origin`。

PR Agent Desktop 设置里的 **GitHub Remote**（如 `ibytebot`）对应 `git push <remote>` 时用的名字，需与本地 `git remote -v` 一致。

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

在 **GitHub 网站**（浏览器）上发起：**「请把我的分支改动合进 `main`」**。  
这一步**不在**本地 `d:\bytebot` 里完成。

PR 页面包含：

| 区域 | 作用 |
|------|------|
| **Conversation** | 讨论、发 `/cursor` 指令 |
| **Files changed** | 查看 Agent / 你改了什么 |
| **Merge pull request** | 你最终批准，合入 `main` |

具体怎么开 PR，见下文 **标准流程 → ②**。


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
  ├─ ① 开空分支（audit/project-improvements）
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

### ① 开空分支并推送（本地 PowerShell）

```powershell
cd d:\bytebot
git checkout main
git pull ibytebot main
git checkout -b audit/project-improvements
git commit --allow-empty -m "chore: start project audit PR"
git push -u ibytebot audit/project-improvements
```

### ② 在 GitHub 网站开 PR（浏览器操作）

**方式一：点按钮（推荐，不用记 URL）**

1. 浏览器打开：https://github.com/hyqibot/ibytebot  
2. 顶部点 **Pull requests**  
3. 点绿色 **New pull request**  
4. **base** 选 `main`（合入目标）  
5. **compare** 选 `audit/project-improvements`（① 里 push 的分支名，按你实际改的）  
6. 点 **Create pull request** → 标题如 `audit: project improvement plan` → Description 可空 → 创建  

**方式二：用快捷链接（效果与方式一相同）**

把下面地址里的分支名换成 ① 里用的名字，粘贴到浏览器地址栏：

```text
https://github.com/hyqibot/ibytebot/compare/main...audit/project-improvements
```

链接拆解（全是 **GitHub 网站路径**，不是本地路径）：

| 片段 | 含义 |
|------|------|
| `github.com/hyqibot/ibytebot` | 打开你的仓库主页 |
| `/compare/` | 网站「对比分支、准备开 PR」功能（类似 `/pulls` 是 PR 列表页） |
| `main` | 合入目标分支（base） |
| `...` | GitHub 规定的分隔符 |
| `audit/project-improvements` | 你的源分支（compare / head） |

打开后应显示 `base: main` ← `compare: audit/project-improvements`，再点 **Create pull request**。

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

## 只审查、不改业务代码（如依赖/Docker 风险扫描）

适用：「列出风险，但不要改代码，等我审批」。

### 为什么有时 Files changed 为空、也看不到结论？

旧版 workflow 设计是 **改代码 → commit → push**，成功回复固定为「已提交改动」。  
若 Agent 遵守「不改代码、不 commit」，**结论只留在 Actions 日志里**，PR 里就像「没回复」。

**2026-06 起 workflow 已改进**：无新提交时，会把 Agent 终端输出贴到 PR 评论（见 `cursor-pr-comment.yml`）。  
**需把更新后的 workflow push 到目标仓 `main`**（ibytebot）后，新触发的 `/cursor` 才生效。

### 推荐写法（二选一）

**方式 A（推荐）**：允许写 audit 文档，不改业务代码

```text
/cursor 审查 packages/bytebotd 的依赖与 Docker 配置，列出风险。

要求：
1. 将结论写入 docs/audit-bytebotd-YYYY-MM-DD.md（P0/P1/P2 + 风险 + 建议）
2. 不要修改 packages/bytebotd 下的业务代码或 Dockerfile
3. commit 并 push 仅 audit 文档到当前 PR 分支
```
或更简短如：/cursor 审查 packages/bytebotd 的依赖与 Docker 配置，列出风险。
将结论写入 docs/audit-bytebotd-YYYY-MM-DD.md，不要改业务代码，commit 并 push 仅 audit 文档。

完成后看 **Files changed** 里的 audit 文档。

**方式 B**：完全禁止写任何文件

```text
/cursor 审查 packages/bytebotd 的依赖与 Docker 配置，列出风险。
不要改任何文件，不要 commit。在最终回复里用结构化列表给出全部风险点。
```

workflow 会把 Agent 最终输出贴到 PR 评论（需已部署新版 workflow）。

---

## 日常小改动（单轮 `/cursor`）

1. 开空分支 → push（同上 ① 的 `--allow-empty` 命令，分支名按任务改）
2. 开 PR  
3. 评论一条 `/cursor` 说明要改什么  
4. review → Merge  

示例：

```text
/cursor 把 packages/bytebot-ui 里登录页标题改成中文，并补一个最小单元测试。
```

---


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

以上是：在 Actions 脚本里加判断，只允许指定用户执行 /claude 指令，彻底限制陌生人调用 AI。
另外，可以：
权限开关：是否允许 Fork 仓库的 PR 运行 Actions
GitHub 默认设置：

◦ 公开仓库：允许来自 Fork 的 PR 触发本仓库 Actions（也就是路人提的 PR 也能用你的自动化）

◦ 私有仓库：默认禁止 Fork 触发 Actions

如果你想限制，关闭该权限：
仓库 → Settings → Actions → General
找到 Workflow permissions / Pull requests from forks

◦ 勾选 Do not allow GitHub Actions workflows from forked repositories
→ 效果：路人 Fork 后提的 PR，无法触发你的自动化，只有你仓库内部成员可用。

---

## 相关文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/cursor-pr-comment.yml` | `/cursor` / `/claude` / `/codex` 统一触发（PR Agent Comment） |
| `.github/workflows/claude-code-kimi.yml` | 手动 workflow_dispatch 调试 |
| `.github/workflows/cursor-agent.yml` | workflow_dispatch（audit/implement） |
| `.cursor/cli.json` | Agent 读写 / Shell 权限 |
| `README.md` | 仓库首页简要说明 |

---

## 排查

**`/cursor` 跑在 GitHub Actions 云服务器上**，workflow 里用的 `gh`、`git` 等由 **GitHub 自动提供**，你**不用**为了用机器人而安装任何东西。

**查看运行记录（推荐，浏览器）：**

1. 打开 https://github.com/hyqibot/ibytebot/actions  
2. 左侧选 **Cursor PR Comment**  
3. 点某次运行看哪一步失败  

PR 评论里的 ❌ 回复也会带 Actions run 链接，点进去即可。

**可选（仅当你在本地 PowerShell 想查时）：** 需自行安装 [GitHub CLI (`gh`)](https://cli.github.com/)，Windows 默认没有：

```powershell
gh run list -R hyqibot/ibytebot -w "Cursor PR Comment" -L 5
```

不装 `gh` 完全不影响 `/cursor` 正常使用。

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
| 让 Cursor 改代码 | 开空分支 → 开 PR → 评论 `/cursor` |
| 先出方案再改 | 同一 PR 发两次 `/cursor`（先 docs，后代码） |
| 批准方案 | 在 PR Conversation 写「批准 xxx」 |
| 批准上线 | Merge PR |
| 看改了什么 | PR → Files changed |
| 扩大可改范围 | 改 `.cursor/cli.json` → push `main` |



后只做「查看 + 建议」怎么 dispatch
在 prompt 里写死约束，例如：

只读评审，禁止修改任何文件。
输出：1) 按优先级排序的改善建议 2) 每项说明位置/理由/风险 3) 可选的小步实施顺序。
不要 git commit，不要改代码，不要跑会写文件的命令。

已为 Cursor Agent workflow 加上 audit / implement 双模式，并写了说明文档。

改动文件
.github/workflows/cursor-agent.yml
docs/reference/cursor-agent-workflow.md（新建）
docs/reference/public-repo-sync-bootstrap.md（Secret 表加链接）
怎么用
场景	选什么
看项目、提改善建议（类似 #5）
mode = audit
真要改代码
mode = implement（默认，行为与以前一致）
audit 模式行为
Agent 指令：禁止改任何文件，结论只写在 stdout
不跑 bun install（只 checkout + Cursor CLI）
若仍产生 diff → 步骤 Reject audit workspace mutations 失败，打印 diff，不推分支
报告位置：Actions → run → Run Cursor Agent (audit — read only) 日志
implement 模式额外修复
commit 时 排除 changed-files.txt，避免 CI 临时文件被提交
下次 dispatch 示例

mode: audit
prompt: 通读 desktop 与 server 核心模块，按优先级列出改善建议，不要改任何文件。