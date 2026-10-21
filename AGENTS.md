# 同步说明（给未来的 AI 会话看）

本仓库是 `magpie-monitor` 技能的**唯一源码入口**。

## 目录约定

| 角色 | 路径 |
|---|---|
| **源码（编辑这里）** | `F:\code\magpie-monitor-skill\SKILL.md` |
| **生效（DSH 读取这里）** | `C:\Users\yuanmingjie\.agents\skills\magpie-monitor\SKILL.md` |
| **远程仓库** | https://github.com/cainiaoNO1/magpie-monitor-skill |

## 修改流程（必须按此顺序）

用户提出修改 skill 时，执行：

1. **编辑源码**：改 `F:\code\magpie-monitor-skill\SKILL.md`
2. **提交推送**：
   ```powershell
   cd F:\code\magpie-monitor-skill
   git add .
   git commit -m "update: <改动说明>"
   git push
   ```
3. **复制到生效目录**：
   ```powershell
   Copy-Item F:\code\magpie-monitor-skill\SKILL.md `
     C:\Users\yuanmingjie\.agents\skills\magpie-monitor\SKILL.md -Force
   ```
4. **校验一致性**（可选）：
   ```powershell
   (Get-FileHash F:\code\magpie-monitor-skill\SKILL.md).Hash -eq `
   (Get-FileHash C:\Users\yuanmingjie\.agents\skills\magpie-monitor\SKILL.md).Hash
   ```

## 禁止

- ❌ 直接编辑 `~/.agents/skills/magpie-monitor/SKILL.md`（会被下次同步覆盖）
- ❌ 只在本地改而不推送
- ❌ 在 `.git/config` 中保留带 Token 的 remote URL

## Git 身份

- 账号：`cainiaoNO1`
- remote：`https://github.com/cainiaoNO1/magpie-monitor-skill.git`（**无 Token**）
