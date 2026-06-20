# `/claude` 测试指南（Claude Code + Kimi）

测试分支：`test/claude-kimi`  
仓库：https://github.com/hyqibot/ibytebot

---

## 前置条件

- [ ] 已配置 Secret：`MOONSHOT_API_KEY`
- [ ] Actions 权限：**Read and write permissions**
- [ ] `main` 上 workflow 已安装 `@anthropic-ai/claude-code@latest`

---

## 第 1 步：开 PR

分支 `test/claude-kimi` 已推送。浏览器打开：

1. https://github.com/hyqibot/ibytebot  
2. **Pull requests** → **New pull request**  
3. **base** `main` ← **compare** `test/claude-kimi`  
4. **Create pull request**（标题可写 `test: claude code kimi bot`）

---

## 第 2 步：发测试评论

在 PR **Conversation** 粘贴发送：

```text
/claude 请阅读 /tmp/pr.diff 和 docs/claude-kimi-test-target.md。把「测试状态」改成「已由 Claude Code（Kimi）自动更新」，并在文末加一行「验证日期：2026-06-01」。改完后 commit 并 push 到当前 PR 分支。
```

---

## 第 3 步：看结果

| 检查项 | 预期 |
|--------|------|
| 评论旁 | 👀 reaction |
| Actions | **Claude Code (Kimi)** 绿色通过 |
| PR 评论 | ✅ Claude Code（Kimi）已在 `test/claude-kimi` 提交改动 |
| Files changed | `docs/claude-kimi-test-target.md` 被更新 |
| Actions 日志 Install 步骤 | 打印 `claude --version`（最新版） |

失败时：https://github.com/hyqibot/ibytebot/actions → 点对应 run 看 **Run Claude Code with Kimi** 步骤。

---

## 第 4 步：合并或关闭

- 测试通过 → **Merge pull request** → 可选 Delete branch  
- 仅试验 → **Close pull request**

本地同步：

```powershell
cd d:\bytebot
git checkout main
git pull ibytebot main
```
