---
name: magpie-monitor
description: 通过 150 服务器 (18.188.150.249) 的 SSH MCP (ssh-amazon-150) 连接 Magpie，采集分析多账号配额用量、调用汇总、单个账号请求记录、单次请求明细及耗时占比分布、实时请求路由决策与首字时间拆解。Use when the user asks to 查看 magpie 监控、账号用量、5小时/7天配额、请求明细、耗时统计与占比、首字时间或实时请求状态.
---

# Magpie 监控与状态查询技能 (Magpie Monitor)

本技能用于通过 `ssh-amazon-150` MCP 远程连接服务器 `18.188.150.249`，结合 `magpie` 容器 CLI 及宿主机持久化目录 `/home/ubuntu/apps/magpie/config/magpie` 中的日志与指标文件，精准提取 Magpie 监控大屏中的各项数据。

> **注意：** 容器内无独立 Python 环境，结构化分析与耗时计算请直接在宿主机通过 `sudo python3` 读取 `/home/ubuntu/apps/magpie/config/magpie/` 下的文件，或通过 `sudo docker exec magpie magpie <subcommand>` 运行内置 CLI。

---

## 核心数据源速查

| 监控项 | 对应前端界面 | 推荐执行方式 | 核心输出 |
|---|---|---|---|
| **一、账号 5 小时 & 7 天配额** | 概览页顶部【配额卡片】 | `sudo docker exec magpie magpie quota` 或解析 `quotas.json` | 各账号 5h、7d 配额百分比与倒计时 |
| **二、当天模型调用汇总** | 概览页中间【用量统计】 | `sudo docker exec magpie magpie usage today` 或统计 `usage.jsonl` | 输入/输出/缓存 Tokens、费用、总次数、失败数、最后调用时间 |
| **三、单个账号的请求详细列表** | 请求页（表格列表） | `magpie usage --csv --account <email> today` 或解析 `usage.jsonl` | 请求时间、模型、Tokens、耗时、状态码、会话 ID |
| **四、单次请求明细与耗时分布** | 请求页详情展开 (图六) | 解析 `routing/<YYYY-MM-DD>.jsonl` 中的单条请求 | 网关内耗、发送耗时、供应商响应/等待、首字时间 (TTFT)、流式输出耗时及**各项占比** |
| **五、实时请求详细信息** | 路由页【正在做出的决定】(图五) | 读取 `routing/<YYYY-MM-DD>.jsonl` 最新一条记录 | 实时命中的账号、模型、路由裁决理由、首字时间、完整的各阶段耗时比例拆解 |

---

## 五大场景标准化查询指令

全部通过 `mcp__ssh-amazon-150__execute-command` 执行。

### 场景一：查询各账号 5 小时与 7 天配额用量

```bash
sudo docker exec magpie magpie quota
```

或通过 Python 提取精准数值：
```bash
sudo python3 -c '
import json
with open("/home/ubuntu/apps/magpie/config/magpie/quotas.json") as f:
    data = json.load(f)
for acc_id, acc_info in data.items():
    quota = acc_info.get("quota", {})
    user = quota.get("user")
    plan = quota.get("plan")
    print(f"账号: {user} ({plan})")
    for w in quota.get("windows", []):
        if w.get("name") in ["5 hours", "7 days"]:
            used = w.get("used", 0)
            resets = w.get("resetsAt", "")
            pool = w.get("pool", "")
            print(f"  - [{pool}] {w[\"name\"]}: 已用 {used:.1f}% | 重置时间: {resets}")
'
```

---

### 场景二：查询各账号模型请求调用汇总（默认当天）

```bash
sudo python3 -c '
import json, datetime

today_str = datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%d")
accounts = {}

with open("/home/ubuntu/apps/magpie/config/magpie/usage.jsonl") as f:
    for line in f:
        if not line.strip(): continue
        item = json.loads(line)
        t_str = item.get("t", "")
        if not t_str.startswith(today_str):
            continue
        acc = item.get("providerAccount", "unknown")
        if acc not in accounts:
            accounts[acc] = {
                "calls": 0, "errors": 0, "in_tokens": 0, "out_tokens": 0,
                "cache_read": 0, "cost": 0.0, "last_time": ""
            }
        a = accounts[acc]
        a["calls"] += 1
        if item.get("status", 200) >= 400:
            a["errors"] += 1
        a["in_tokens"] += item.get("in", 0)
        a["out_tokens"] += item.get("out", 0)
        a["cache_read"] += item.get("cache_read", 0)
        a["cost"] += item.get("cost", 0.0)
        a["last_time"] = t_str

print(f"=== {today_str} 当天各账号调用汇总 ===")
for acc, stat in accounts.items():
    print(f"账号: {acc}")
    print(f"  调用次数: {stat[\"calls\"]} (失败: {stat[\"errors\"]})")
    print(f"  输入 Tokens: {stat[\"in_tokens\"] / 1e6:.2f} M | 输出: {stat[\"out_tokens\"] / 1e3:.1f} k | 缓存读取: {stat[\"cache_read\"] / 1e6:.2f} M")
    print(f"  估算费用: ≈${stat[\"cost\"]:.3f}")
    print(f"  最后调用时间 (UTC): {stat[\"last_time\"]}")
'
```

---

### 场景三：获取单个账号的请求详细列表

```bash
sudo python3 -c '
import json

target = "y15570087076@gmail.com"  # 可替换为目标账号
records = []

with open("/home/ubuntu/apps/magpie/config/magpie/usage.jsonl") as f:
    for line in f:
        if not line.strip(): continue
        item = json.loads(line)
        if item.get("providerAccount") == target:
            records.append(item)

print(f"=== {target} 最近 10 次请求记录 ===")
for r in records[-10:]:
    t = r.get("t", "")[:19].replace("T", " ")
    model = r.get("model", "")
    inp = r.get("in", 0)
    out = r.get("out", 0)
    ttft = r.get("ttft_ms", 0) / 1000
    ms = r.get("ms", 0) / 1000
    st = r.get("status", 200)
    resp_id = r.get("response_id", "")
    print(f"[{t}] HTTP {st} | 模型: {model} | 输入: {inp} tok, 输出: {out} tok | 首字: {ttft:.1f}s, 耗时: {ms:.1f}s | ID: {resp_id}")
'
```

---

### 场景四：单次请求明细与耗时分解（时间花在哪里与占比）

针对指定的请求记录，提取精准耗时与百分比占用：

```bash
sudo python3 -c '
import json, glob

files = sorted(glob.glob("/home/ubuntu/apps/magpie/config/magpie/routing/*.jsonl"))
if not files: exit(0)

with open(files[-1]) as f:
    lines = [l.strip() for l in f if l.strip()]
rec = json.loads(lines[-1])

total_ms = rec.get("ms", 1)
ttft_ms = rec.get("ttft", 0)
tries = rec.get("tries", [{}])
t0 = tries[0] if tries else {}
net = t0.get("net", {})

magpie_own = net.get("own", rec.get("own", 0))
send_ms = net.get("send", rec.get("sent", 0))
wait_ms = net.get("wait", 0)
upstream_total = t0.get("ms", 0)
stream_ms = max(0, total_ms - ttft_ms)

print("=== 请求明细与时间花在哪里 ===")
print("请求 ID: %s | 状态: HTTP %s" % (rec.get("id"), rec.get("status")))
print("总耗时 (Total): %d ms" % total_ms)
print("首字时间 (TTFT): %d ms (占总耗时 %.1f%%)" % (ttft_ms, ttft_ms / total_ms * 100))
print("耗时阶段分布与占比:")
print("  - magpie 网关自身耗时: %.1f ms (%.1f%%)" % (magpie_own, magpie_own / total_ms * 100))
print("  - 发送请求耗时: %.1f ms (%.1f%%)" % (send_ms, send_ms / total_ms * 100))
print("  - 供应商等待首字响应: %.1f ms (%.1f%%)" % (wait_ms, wait_ms / total_ms * 100))
print("  - 供应商响应总耗时: %.1f ms (%.1f%%)" % (upstream_total, upstream_total / total_ms * 100))
print("  - 流式输出耗时: %.1f ms (%.1f%%)" % (stream_ms, stream_ms / total_ms * 100))
'
```

---

### 场景五：获取实时请求详细信息与时间分布（对应图五 & 图六）

无上下文窗口干扰，直接聚焦于实时路由、首字时间、各阶段时间占用比例：

```bash
sudo python3 -c '
import json, glob

files = sorted(glob.glob("/home/ubuntu/apps/magpie/config/magpie/routing/*.jsonl"))
if not files:
    print("未发现路由记录")
    exit(0)

with open(files[-1]) as f:
    lines = [l.strip() for l in f if l.strip()]
if not lines:
    print("今日暂无请求")
    exit(0)

rec = json.loads(lines[-1])
total_ms = rec.get("ms", 1)
ttft_ms = rec.get("ttft", 0)
tries = rec.get("tries", [{}])
t0 = tries[0] if tries else {}
net = t0.get("net", {})

magpie_own = net.get("own", rec.get("own", 0))
send_ms = net.get("send", rec.get("sent", 0))
wait_ms = net.get("wait", 0)
upstream_total = t0.get("ms", 0)
stream_ms = max(0, total_ms - ttft_ms)

order = rec.get("order", [])
selected_acc = order[0].get("who") if order else "unknown"
plan = order[0].get("plan") if order else "unknown"

usage = rec.get("usage", [{}])[0]
inp = usage.get("in", 0)
out = usage.get("out", 0)
cache_read = usage.get("cache_read", 0)
reasoning = usage.get("reasoning", 0)

print("=== 实时请求详细信息 ===")
print("时间 (UTC): %s | ID: %s" % (rec.get("time"), rec.get("id")))
print("发起 Agent: %s | 会话: %s" % (rec.get("agent"), rec.get("session")))
print("请求模型: %s (实际服务: %s)" % (rec.get("model"), rec.get("served")))
print("响应账号: %s (%s)" % (selected_acc, plan))
print("路由裁决原因: %s" % rec.get("affinity", {}).get("why", "smart"))
print("Tokens: 输入 %d, 输出 %d (思考 %d), 缓存命中 %d" % (inp, out, reasoning, cache_read))

print("\n=== 首字时间与各阶段耗时比例 (时间花在哪里) ===")
print("总耗时: %d ms (%.2f 秒)" % (total_ms, total_ms / 1000.0))
print("首字时间 (TTFT): %d ms (%.2f 秒) -> 占总耗时 %.1f%%" % (ttft_ms, ttft_ms / 1000.0, ttft_ms / total_ms * 100))
print("耗时阶段拆解:")
print("  1. magpie 网关处理 : %6.1f ms | 占比 %5.1f%%" % (magpie_own, magpie_own / total_ms * 100))
print("  2. 发送请求阶段     : %6.1f ms | 占比 %5.1f%%" % (send_ms, send_ms / total_ms * 100))
print("  3. 供应商等待首字   : %6.1f ms | 占比 %5.1f%%" % (wait_ms, wait_ms / total_ms * 100))
print("  4. 供应商总响应时间 : %6.1f ms | 占比 %5.1f%%" % (upstream_total, upstream_total / total_ms * 100))
print("  5. 流式数据输出耗时 : %6.1f ms | 占比 %5.1f%%" % (stream_ms, stream_ms / total_ms * 100))
'
```

---

### 场景六：最近 N 次请求趋势（固定表格格式，必用）

**强制格式要求（不得改动）：**

| 列 | 格式 | 说明 |
|---|---|---|
| 时间 | `yyyy-mm-dd hh:mm:ss`，**北京时间 (UTC+8)** | 由日志中的 UTC 时间 `+8` 小时换算 |
| 总耗时 | **秒**，保留 2 位小数 | `ms / 1000` |
| 首字 | **秒**，保留 2 位小数 | `ttft / 1000` |
| 供应商等待首字 | **秒**，保留 2 位小数 | `tries[0].net.wait / 1000` |
| 供应商总响应 | **秒**，保留 2 位小数 | `tries[0].ms / 1000` |

**必须包含的列（顺序固定）：**
时间 | 状态 | 总耗时(s) | 首字(s) | TTFT占比 | 供应商等待首字(s) | 供应商总响应(s) | 网关处理(ms) | 流式输出(ms) | 账号 | 缓存读

```bash
sudo python3 -c '
import json, glob, datetime

N = 6  # 次数可调
files = sorted(glob.glob("/home/ubuntu/apps/magpie/config/magpie/routing/*.jsonl"))
with open(files[-1]) as f:
    lines = [l.strip() for l in f if l.strip()]
recs = [json.loads(l) for l in lines[-N:]]

def bj(iso):
    # UTC ISO -> 北京时间 yyyy-mm-dd hh:mm:ss
    try:
        dt = datetime.datetime.strptime(iso[:19], "%Y-%m-%dT%H:%M:%S")
        return (dt + datetime.timedelta(hours=8)).strftime("%Y-%m-%d %H:%M:%S")
    except Exception:
        return iso[:19].replace("T", " ")

print("| 时间 (北京时间) | 状态 | 总耗时(s) | 首字(s) | TTFT占比 | 供应商等待首字(s) | 供应商总响应(s) | 网关处理(ms) | 流式输出(ms) | 账号 | 缓存读 |")
print("|---|---|---|---|---|---|---|---|---|---|---|")
for r in recs:
    st = r.get("status")
    ms = r.get("ms", 0)
    ttft = r.get("ttft", 0)
    pct = ttft / ms * 100 if ms else 0
    tries = r.get("tries", [{}])
    t0 = tries[0] if tries else {}
    net = t0.get("net", {})
    own = net.get("own", r.get("own", 0))
    wait = net.get("wait", 0)
    up = t0.get("ms", 0)
    stream = max(0, ms - ttft)
    o = r.get("order", [{}])
    who = o[0].get("who", "?") if o else "?"
    cr = r.get("usage", [{}])[0].get("cache_read", 0)
    print("| %s | %s | %.2f | %.2f | %.1f%% | %.2f | %.2f | %.1f | %.0f | %s | %d |" % (
        bj(r.get("time", "")), st, ms / 1000.0, ttft / 1000.0, pct, wait / 1000.0, up / 1000.0, own, stream, who, cr))
'
```

---

## 呈现规范（强制）

1. **所有时间一律北京时间 (UTC+8)**，格式 `yyyy-mm-dd hh:mm:ss`。日志原始时间为 UTC，必须 `+8` 小时后再输出，并在表头标注「北京时间」。
2. **总耗时、首字、供应商等待首字、供应商总响应一律用秒**，保留 2 位小数。
3. 每次趋势表都必须带上 **供应商等待首字(s)** 与 **供应商总响应(s)** 两列。
4. 耗时分解必须给出**绝对秒数**以及**相对总耗时的百分比（%）**。
5. 纯净展示路由决策、账号命准与性能瓶颈，无需输出上下文色块分布。
