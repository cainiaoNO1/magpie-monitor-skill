# magpie-monitor

Magpie 网关监控查询技能（DSH Agent Skill）。

## 用途

通过 SSH MCP (`ssh-amazon-150`) 连接 150 服务器上的 Magpie 容器，采集与分析：

- 多账号 5 小时 / 7 天配额用量
- 当天各账号模型调用汇总（Tokens、费用、次数、失败数、最后调用时间）
- 单个账号的请求详细列表
- 单次请求明细与各阶段耗时占比
- 实时请求路由决策、首字时间拆解

## 文件

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 技能正文（指令、查询脚本、呈现规范） |

## 安装到 DSH

把 `SKILL.md` 复制到 DSH 用户技能目录：

```powershell
Copy-Item .\SKILL.md "C:\Users\<你的用户名>\.agents\skills\magpie-monitor\SKILL.md" -Force
```

技能目录结构：

```
~/.agents/skills/magpie-monitor/SKILL.md
```

## 工作流约定

**源码目录**：`F:\code\magpie-monitor-skill`（唯一编辑入口）

修改流程：

1. 编辑 `F:\code\magpie-monitor-skill\SKILL.md`
2. `git add . && git commit -m "..." && git push`
3. 复制到 `~/.agents/skills/magpie-monitor/SKILL.md` 使其生效

## 关键约定

- 时间一律**北京时间**，格式 `yyyy-mm-dd hh:mm:ss`
- 总耗时、首字、供应商等待首字、供应商总响应一律用**秒**（保留 2 位小数）
- 趋势表必须包含 `供应商等待首字(s)` 与 `供应商总响应(s)` 两列
