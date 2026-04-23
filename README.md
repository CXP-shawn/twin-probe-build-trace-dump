# twin-probe-build-trace-dump

某次 probe agent build run 的**完整事件时间线 trace dump**，按分批切片推送到此仓库。

## 目标 run 元信息

| 字段 | 值 |
|---|---|
| agent_id | `019db95d-7d6b-79d1-9908-5276e6cd29d2` |
| run_id | `019db95d-7da4-7960-aa12-a6875a77cd6c` |
| 首条 event_index | `157912706` |
| 末条 event_index | `157921613`（本 dump 覆盖范围内） |
| 服务器报告 total_events | **1510**（动态增长中，因本 dump 过程本身也会触发新事件） |
| 本 dump 实际 dump events | **500** |
| batch_size | 50 |
| batch_count | 10 |

## API 分页能力实测（诚实说明）

Twin 公开 REST API `GET /v1/agents/{agent_id}/runs/{run_id}/events` **硬上限为 500 条 / 请求**。

实测分页参数均被服务器忽略：
- `after_index=<N>` — 实测忽略，仍返回最早 500 条
- `from_index=<N>` — 实测忽略，仍返回最早 500 条
- `limit>500` — 截断到 500
- 响应头无 `Link` / `X-Next-Cursor` 类分页指示

**后果**：即使 `total_count=1510`，通过公开 REST API 实际可访问的**只有最早 500 条**。本仓库的 dump 忠实反映这一上限，并不声称覆盖全部 1510 条。

## 事件顺序重组

按 `batch_0001.json` → `batch_0010.json` 顺序读取每批的 `events` 数组并串接即可得到完整时间线：

```bash
for f in batches/batch_*.json; do
  jq '.events[]' "$f"
done > all_events.jsonl
```

## 每条 event 的来源

每条 event 对象**原样**保留自 Twin 公开 REST API 响应：

```
GET https://build.twin.so/v1/agents/019db95d-7d6b-79d1-9908-5276e6cd29d2/runs/019db95d-7da4-7960-aa12-a6875a77cd6c/events
Header: x-api-key: <redacted>
响应体 .events[*] 元素 → 本仓库每个 event 对象
```

顶层 shape：

```json
{
  "event_index": 157912706,
  "recorded_at": "...",
  "event": {
    "agent_id": {"id": "..."},
    "run_id": "...",
    "user_id": "...",
    "event": { "event": { /* UserTick / PolicyDecided / ToolCallResolved / ... */ } }
  }
}
```

## 脱敏策略（逐字说明）

深度遍历每个 event JSON，**仅当**字段 key 名（case-insensitive）**精确等于**以下 6 个值之一时，该字段 value 替换为字符串 `***REDACTED***`：

- `authorization`
- `x-api-key`
- `api_key`
- `bearer`
- `secret`
- `password`

**其它所有字段一律保留原值**，包括：
- `agent_id` / `run_id` / `user_id` / `user_email`
- API 响应 body 原文、错误消息原文、时间戳
- tool 调用的 input / output 子对象
- dynamic tool 定义的 `path_schema` / `body_schema` / `url`

这是为了让分析者能原样看到"这次 build run 做了什么"的每个细节。

## 文件布局

```
twin-probe-build-trace-dump/
├── README.md                ← 本文件
├── MANIFEST.json            ← 批次索引 + 元信息
└── batches/
    ├── batch_0001.json      ← 批次 1 · 50 条 · event_index 157912706..15791...
    ├── batch_0002.json
    ├── ...
    └── batch_0010.json      ← 批次 10 · 末批
```

每个 batch JSON 的 schema：

```json
{
  "run_id": "...",
  "agent_id": "...",
  "batch_index": 1,
  "batch_count": 10,
  "event_index_start": 157912706,
  "event_index_end": ...,
  "count": 50,
  "events": [ ... ]
}
```

## 免责

这是**单次 run 的 trace 快照**，不代表 Twin 平台最新行为；且因公开 REST API 分页限制，仅含最早 500 条事件。
