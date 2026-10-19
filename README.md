# hermes-signals — HERMES → Muse 唤醒信号通道

这是一个**公开仓库**，用途只有一个：让 HERMES 把「任务完成」的信号推给 Muse，Muse 侧的 hook 监听到就唤醒 Muse。

- **HERMES** = 执行方（Mac 上的 Hermes Agent，干活的）
- **Muse** = 指挥方（发任务卡、定需求、审证据）

## 协议

### 触发时机

HERMES 每完成一个任务，**除了**发 TG 消息之外，**还必须**向本仓库推送一个信号文件。

> TG 消息是给人看的，信号文件是给机器看的。两者都要发，不能只发其中一个。

### 文件位置与命名

```
tasks/<任务编号>-done.json
```

- 任务编号用任务卡里的编号，例如 `002`、`003`
- 状态后缀：`done` 表示完成；`blocked` 表示受阻未完成（此时 `blockers` 字段必须非空）
- 示例文件：`tasks/002-done.json`、`tasks/003-blocked.json`

Muse 侧的 hook 通过「发现新增的 `tasks/*.json`」来唤醒，**不需要解析文件内容就能触发**；文件内容是给 Muse 醒来后读证据用的。

### 格式

严格 JSON，UTF-8 编码，字段如下：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `task_id` | string | ✅ | 任务编号，如 `"002"`。示例文件用 `"example"` |
| `title` | string | ✅ | 任务标题，如 `"个人主页建站"` |
| `status` | string | ✅ | `done` / `blocked` |
| `completed_at` | string | ✅ | ISO 8601 带时区，如 `"2026-10-19T23:05:08+08:00"` |
| `evidence` | object | ✅ | 见下 |
| `blockers` | array | ✅ | 无阻塞写 `[]`；受阻时写明缺什么 |
| `notes` | string | ❌ | 补充说明，可选 |

`evidence` 内部结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `folder` | string | 本次产出所在文件夹绝对路径；无产出写 `""` |
| `links` | array | 外部链接（GitHub 仓库 / commit / Release / 文档 URL） |
| `screenshots` | array | 截图文件绝对路径 |
| `logs` | array | 可选。日志或报告文件的绝对路径 |

### 完整示例

见 [`tasks/example-done.json`](tasks/example-done.json)。

```json
{
  "task_id": "002",
  "title": "个人主页建站",
  "status": "done",
  "completed_at": "2026-10-19T23:05:08+08:00",
  "evidence": {
    "folder": "/Users/daiba/homepage-2026-10-19/",
    "links": ["https://github.com/selectcpt-beep/md-reader"],
    "screenshots": ["/Users/daiba/homepage-2026-10-19/evidence/desktop-01-hero.png"]
  },
  "blockers": [],
  "notes": "静态站已完成，四视口验收通过。"
}
```

## 硬性规则

1. **绝对禁止在信号文件里写密钥、密码、Token、Cookie、私钥、`.env` 内容。** 只写路径、链接和说明。凭据永远不进 Git。
2. **不推 tag。** 信号只走 commit + push。
3. **不改动已有信号文件。** 每个任务写自己的新文件，历史信号保持原样，作为时间线。
4. **一个任务一个文件。** 不要把多个任务合并进一个 JSON。
5. **JSON 必须能被 `python3 -m json.tool` 解析。** HERMES 推送前自检一次。
6. **`completed_at` 用带时区的 ISO 8601**，不要写成本地化字符串。

## 目录结构

```
hermes-signals/
├── README.md              ← 本文件，协议说明
└── tasks/
    ├── example-done.json  ← 格式示例
    └── <任务编号>-done.json
```

## 消费方须知（Muse 侧 hook）

- 建议用 GitHub API 轮询 `tasks/` 目录，或 `git pull` 后看新增文件
- 判重：以**文件名**为准（同一文件名不重复唤醒）
- 唤醒后读该 JSON，拿到 `evidence` 里的路径和链接去核验
- 看到 `status: "blocked"` 时，读取 `blockers` 数组，它写的是「需要什么」

## License

MIT — 见 [`LICENSE`](LICENSE)。
