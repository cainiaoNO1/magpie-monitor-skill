# 同步说明（给未来的 AI 会话看）

本仓库是 `magpie-monitor` 技能的**唯一源码入口**。

## 目录约定

Windows 与 macOS 两套路径都有效，**按当前系统选择对应的一列**。

| 角色 | Windows | macOS |
|---|---|---|
| **源码（编辑这里）** | `F:\code\magpie-monitor-skill\SKILL.md` | `/Users/ymj/Desktop/code/dsh/magpie-monitor-skill/SKILL.md` |
| **生效（DSH 读取这里）** | `C:\Users\yuanmingjie\.agents\skills\magpie-monitor\SKILL.md` | `/Users/ymj/.agents/skills/magpie-monitor/SKILL.md` |
| **远程仓库** | https://github.com/cainiaoNO1/magpie-monitor-skill | 同左 |

## 修改流程（必须按此顺序）

用户提出修改 skill 时，执行：**1. 编辑源码 → 2. 提交推送 → 3. 复制到生效目录 → 4.（可选）校验一致性**。

### Windows

```powershell
# 1. 编辑源码：F:\code\magpie-monitor-skill\SKILL.md

# 2. 提交推送
cd F:\code\magpie-monitor-skill
git add .
git commit -m "update: <改动说明>"
git push

# 3. 复制到生效目录
Copy-Item F:\code\magpie-monitor-skill\SKILL.md `
  C:\Users\yuanmingjie\.agents\skills\magpie-monitor\SKILL.md -Force

# 4. 校验一致性（可选）
(Get-FileHash F:\code\magpie-monitor-skill\SKILL.md).Hash -eq `
(Get-FileHash C:\Users\yuanmingjie\.agents\skills\magpie-monitor\SKILL.md).Hash
```

### macOS

```bash
# 1. 编辑源码：/Users/ymj/Desktop/code/dsh/magpie-monitor-skill/SKILL.md

# 2. 提交推送
cd /Users/ymj/Desktop/code/dsh/magpie-monitor-skill
git add .
git commit -m "update: <改动说明>"
git push

# 3. 复制到生效目录
cp SKILL.md ~/.agents/skills/magpie-monitor/SKILL.md

# 4. 校验一致性（可选）
shasum -a 256 SKILL.md ~/.agents/skills/magpie-monitor/SKILL.md
```

## 禁止

- ❌ 直接编辑 `~/.agents/skills/magpie-monitor/SKILL.md`（会被下次同步覆盖）
- ❌ 只在本地改而不推送
- ❌ 在 `.git/config` 中保留带 Token 的 remote URL

## Git 身份

- 账号：`cainiaoNO1`
- remote：`https://github.com/cainiaoNO1/magpie-monitor-skill.git`（**无 Token**）

## 凭据

- **macOS**：Token 存于钥匙串（osxkeychain），`git push` 免交互。若需重新写入：
  ```bash
  printf 'protocol=https\nhost=github.com\nusername=cainiaoNO1\npassword=<TOKEN>\n\n' | git credential approve
  ```
- **Windows**：由 Git Credential Manager 保管。
- 两端共同要求：Token **只进凭据管理器**，绝不写进 remote URL 或 `.git/config`。
