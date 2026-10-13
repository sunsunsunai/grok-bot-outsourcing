目标：在当前环境落地一套 Slack bot @mention 任务触发系统。

**最终验收标准**：有人在 Slack 里 @bot（顶层消息或 thread 回复），60 秒内 bot 在对应 thread 回复并执行任务；bot 忙时回"正在执行上一个「XXX」任务请稍后"；任务成功/失败都 @请求人；平时轮询不消耗 token。

## 步骤 0：前置检查（缺一项就停下，先补齐）

1. Slack App 已在 api.slack.com 创建并安装到工作区，Bot Token Scopes 包含：
   `chat:write`、`channels:read`、`channels:history`、`channels:join`、
   `groups:read`、`groups:history`、`im:read`、`im:history`、`users:read`
2. Bot token（`xoxb-…`）已**走安全通道**存入 vault，connector 名为 `custom.slack-bot`
   - ⛔ token 禁止出现在任何文件、聊天记录、脚本、环境变量里
3. 你能调用 hooks.add / hooks.dry_run / hooks.enable / hooks.run
4. vault 中转可用：`sys.path.insert(0, "/opt/hatch/skills/skill-creator/bin")` 后
   `from dynamic_credentials import add_surrogate_to_request, read_json_response`
   - ⚠️ 不要轻信文档说 hook 脚本拿不到凭证——先写个探针实测（见步骤 2 自检），以实测为准

## 步骤 1：搭建 slack-bot CLI

编写 `~/workspace/skills/slack-bot/bin/slack-bot`，`chmod +x`，内容如下（照抄）：

```python
#!/usr/bin/env python3
import sys, json, urllib.request, urllib.parse
sys.path.insert(0, "/opt/hatch/skills/skill-creator/bin")
from dynamic_credentials import add_surrogate_to_request, read_json_response

API = "https://slack.com/api"

def call(method, params):
    data = urllib.parse.urlencode(params).encode()
    req = urllib.request.Request(
        f"{API}/{method}", data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded"})
    add_surrogate_to_request(req, "custom.slack-bot", allowed_hosts=["slack.com"])
    return read_json_response(urllib.request.urlopen(req, timeout=20))

def main(argv):
    if len(argv) < 2:
        print("usage: slack-bot <auth|post|history|list> ...", file=sys.stderr)
        return 2
    cmd = argv[1]
    if cmd == "auth":
        print(json.dumps(call("auth.test", {}), ensure_ascii=False, indent=2))
    elif cmd == "post":
        params = {"channel": argv[2], "text": argv[3]}
        if len(argv) > 4:
            params["thread_ts"] = argv[4]
        r = call("chat.postMessage", params)
        if not r.get("ok") and r.get("error") == "not_in_channel":
            call("conversations.join", {"channel": argv[2]})
            r = call("chat.postMessage", params)
        print(json.dumps(r, ensure_ascii=False, indent=2))
        return 0 if r.get("ok") else 1
    elif cmd == "history":
        p = {"channel": argv[2], "limit": argv[3] if len(argv) > 3 else "20"}
        if len(argv) > 4:
            p["oldest"] = argv[4]
        print(json.dumps(call("conversations.history", p), ensure_ascii=False, indent=2))
    elif cmd == "list":
        print(json.dumps(call("conversations.list", {
            "types": "public_channel,private_channel,im",
            "exclude_archived": "true", "limit": "200"}), ensure_ascii=False, indent=2))
    else:
        print(f"unknown command: {cmd}", file=sys.stderr)
        return 2
    return 0

if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

**自检**：
- `python3 -m py_compile` 通过
- 运行 `auth`，**记下 bot 的 User ID**（形如 `U0C9PQ5NT4G`）——下面步骤 2 要填进脚本
- 运行 `list`，确认能看到目标频道

## 步骤 2：编写轮询脚本

编写 `~/hooks/scripts/slack-mention-watch.py`，内容如下（照抄，把 `BOT_ID` 换成步骤 1 记下的值）：

```python
#!/usr/bin/env python3
import sys, os, json, time, urllib.request, urllib.parse
sys.path.insert(0, "/opt/hatch/skills/skill-creator/bin")
from dynamic_credentials import add_surrogate_to_request, read_json_response

BOT_ID = "U0C9PQ5NT4G"  # ← 填步骤 1 记下的 bot User ID
# 允许触发任务的其它 bot 的 user ID，按需填，如 {"U0C8XDUTCNM"}
BOT_ALLOWLIST = set()
MENTION = f"<@{BOT_ID}>"
API = "https://slack.com/api"

def call(method, params):
    data = urllib.parse.urlencode(params).encode()
    req = urllib.request.Request(
        f"{API}/{method}", data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded"})
    add_surrogate_to_request(req, "custom.slack-bot", allowed_hosts=["slack.com"])
    return read_json_response(urllib.request.urlopen(req, timeout=15))

def is_mention_msg(m):
    if (m.get("bot_id") or m.get("subtype") == "bot_message") \
            and m.get("user") not in BOT_ALLOWLIST:
        return False
    if m.get("user") == BOT_ID:
        return False
    return MENTION in m.get("text", "")

def iter_candidates(cid, m, watermark):
    try:
        ts = float(m.get("ts", "0"))
    except ValueError:
        return
    if ts > watermark and is_mention_msg(m):
        m["_ts"] = ts
        yield m
    try:
        latest = float(m.get("latest_reply", "0") or "0")
    except ValueError:
        latest = 0
    if m.get("reply_count", 0) and latest > watermark:
        try:
            r = call("conversations.replies",
                     {"channel": cid, "ts": m["ts"], "limit": "100"})
        except Exception:
            return
        if not r.get("ok"):
            return
        for rm in r.get("messages", []):
            if rm.get("ts") == m.get("ts"):
                continue
            try:
                rts = float(rm.get("ts", "0"))
            except ValueError:
                continue
            if rts <= watermark:
                continue
            if is_mention_msg(rm):
                rm["_ts"] = rts
                rm["thread_ts"] = m["ts"]
                yield rm

def main():
    state_dir = sys.argv[1]
    os.makedirs(state_dir, exist_ok=True)
    wm_path = os.path.join(state_dir, "watermark")
    try:
        with open(wm_path) as f:
            watermark = float(f.read().strip())
        first_run = False
    except (FileNotFoundError, ValueError):
        watermark = time.time()
        first_run = True
    channels = call("conversations.list", {
        "types": "public_channel,private_channel,im",
        "exclude_archived": "true", "limit": "200"})
    if not channels.get("ok"):
        print(json.dumps({"wake": False, "error": channels.get("error")}))
        return 1
    mentions = []
    newest = watermark
    for ch in channels.get("channels", []):
        cid = ch["id"]
        try:
            # ⚠️ oldest 必须 ≤6 位小数！time.time() 的 7 位小数会让 Slack 返回空！
            h = call("conversations.history", {
                "channel": cid, "oldest": "%.6f" % watermark, "limit": "100"})
        except Exception:
            continue
        if not h.get("ok"):
            continue
        for m in h.get("messages", []):
            for cand in iter_candidates(cid, m, watermark):
                ts = cand["_ts"]
                if ts > newest:
                    newest = ts
                mentions.append({
                    "channel": cid,
                    "channel_name": ch.get("name", cid),
                    "user": cand.get("user", ""),
                    "text": cand.get("text", ""),
                    "ts": cand.get("ts", ""),
                    "thread_ts": cand.get("thread_ts", ""),
                })
    with open(wm_path, "w") as f:
        f.write("%.6f" % newest)
    mentions.sort(key=lambda m: float(m["ts"]))
    print(json.dumps({"wake": bool(mentions) and not first_run,
                      "mentions": [] if first_run else mentions,
                      "count": 0 if first_run else len(mentions)}))
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

编写 `~/hooks/scripts/slack-mention-watch.sh`，`chmod +x`，内容如下（照抄）：

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$HATCH_HOOK_RUNTIME"
if [ "${HATCH_HOOK_DRY_RUN:-0}" = "1" ]; then
  STATE_DIR="$HOME/hooks/state/slack-mention-watch-dryrun"
else
  STATE_DIR="$HOME/hooks/state/slack-mention-watch"
fi
mkdir -p "$STATE_DIR"
RESULT=$(python3 "$HOME/hooks/scripts/slack-mention-watch.py" "$STATE_DIR" 2>"$STATE_DIR/last-error.log") || {
  log "slack mention poll failed"
  silent "poll-error" '{}'
}
WAKE=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['wake'])")
if [ "$WAKE" = "True" ]; then
  PAYLOAD=$(echo "$RESULT" | python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps({'mentions': d['mentions']}))")
  wake "slack-mention" "$PAYLOAD"
else
  silent "no-new-mentions" '{}'
fi
```

**自检**：
- 两个文件 `chmod +x`，`python3 -m py_compile` 通过
- ⚠️ 先验证 hook 环境能调通 vault 中转：写个最小探针脚本调一次 `add_surrogate_to_request`，
  用 hooks.add 建临时 hook + hooks.dry_run 跑，确认能拿到 surrogate 再继续；
  探针用完删除。**以实测为准，不要信文档。**

## 步骤 3：注册 hook

调用 `hooks.add`，参数如下（照填）：

- `id`: `slack-mention-watch`
- `poll_interval_secs`: `60`
- `script_path`: `~/hooks/scripts/slack-mention-watch.sh`
- `prompt`: 全文照抄以下内容（把 `YOUR_BOT_USER_ID` 换成 bot 的 User ID，`auth` 可查） ——

```
你是 Slack bot 的任务调度 worker。hook 脚本每分钟轮询 Slack（含 thread 回复），
发现有人 @bot（YOUR_BOT_USER_ID）时唤起你，wake payload 里有 mentions 数组，
每个含 channel、channel_name、user、text、ts、thread_ts
（thread_ts 为空字符串表示顶层消息，否则是所在 thread 的 parent ts）。

发消息一律用 bot 身份：~/workspace/skills/slack-bot/bin/slack-bot post <channel> <文本> [thread_ts]。
@某人用 <@USERID> 格式。
回复位置规则：mention.thread_ts 非空 → 用它做 thread_ts 回在同一 thread；
为空 → 用 mention.ts 做 thread_ts（在该消息下开 thread 回）。
状态文件在 ~/hooks/state/slack-mention-watch/：lock.json（当前任务锁）、waiting.json（等待队列）。

1. 读 lock.json。若存在且未过期（started_at 距今 < 2 小时）→ 忙：
   - 按回复位置规则回：“<@mention.user> 正在执行上一个「<lock.desc>」任务，请稍后等待任务结束后再试。”
   - {user, channel, ts} 追加进 waiting.json（去重），结束。
   - lock 过期（≥2 小时）→ 死锁，删除后按空闲处理，总结里注明。
2. 空闲 → 接任务：
   - 写 lock.json：{desc（任务一句话描述 ≤20 字）、user、channel、thread_ts、started_at}
   - 解析任务：去掉 @ 后的文本即任务；若含 drive.google.com / docs.google.com 链接，
     用 hatch_gws_cli drive 读文件（files get 取 fileId；Google 原生文档走 Docs/Sheets skill）
   - 执行；耗时任务在关键节点 @发起人 报进度
   - 完成（成功/失败都要）：@发起人 报告结果 → 删 lock.json → 读 waiting.json
     逐个 @“上一个任务结束了，可以重新 @我 了”（发主频道）→ 清空 waiting.json
   - 异常：先 @发起人 说明失败原因，再删锁，再通知等待者。锁绝不残留。
3. 总结：处理了哪条 mention、任务、结果、锁状态。
只响应 payload 里的 mention；不自己轮询 Slack；不输出任何凭证。
```

**自检**：`hooks.dry_run` → 应返回 `silent` / `no-new-mentions`。确认无误后 `hooks.enable`。

## 步骤 4：端到端验收（逐项打勾）

1. ☐ 在目标频道发 `<@bot> 测试 ping` → 60 秒内（或手动 `hooks.run`）hook 应 `wake`，
   bot 在该消息的 thread 里回复
2. ☐ 在 thread 里再 @bot 一条 → 回复出现在**同一 thread**
3. ☐ 连续发两条 @（间隔 < 任务耗时）→ 第二条收到"正在执行上一个「XXX」任务，请稍后"
4. ☐ 带 Google Drive 链接的 @ → worker 能读到文件内容并执行
5. ☐ 检查 `~/hooks/state/slack-mention-watch/lock.json`：任务结束后必须不存在（无残留锁）

## 步骤 5：故障排查（按顺序查）

1. hook 日志 `~/hooks/logs/slack-mention-watch.jsonl`：看最近是 `wake` 还是 `silent`
2. `silent` 但明明有 @ → 查水位线 `~/hooks/state/slack-mention-watch/watermark`：
   若水位线已越过那条消息（定时轮询在修 bug 前消费掉了），把水位线回拨到那条消息之前重抓
3. 有 `wake` 但 bot 没回复 → 查 worker 总结，看 lock 是否残留、CLI 是否报错
4. 其它 bot 的 @ 没反应 → 检查 `BOT_ALLOWLIST` 是否放行了它的 user ID
5. thread 里的 @ 没反应 → 确认脚本是新版（含 `conversations.replies` 检查），不是旧版只扫顶层的
