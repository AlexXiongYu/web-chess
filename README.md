# 熊府专用国际象棋 · 网页版工作区

本目录是 `chess.yiruifu.cn` 网页版的**可维护工作区**，与小程序版 `miniprogram-1` 共用同一套实时通信协议，两边房间互通、一起开发、一起更新。

## 目录结构

```
chess-web/
├── index.html          # 网页版主程序（已本地化依赖，可直接静态托管）
├── libs/
│   ├── chess.min.js    # chess.js@0.10.3 UMD 构建（全局 Chess）
│   └── goeasy.js       # GoEasy 通用构建（与小程序版同一份，含 wx 小程序支持）
├── images/pieces/      # 12 张棋子图（wP/wN/wB/wR/wQ/wK + b*，与小程序版同一套）
└── README.md
```

## 与小程序版共用的关键参数（改一处必须两边同步）

| 项目 | 值 |
|------|-----|
| GoEasy Appkey | `BC-58fd21e3fbff443587a9b9f35137cb4e`（Common/Universal，web 与小程序通用） |
| GoEasy host | `hangzhou.goeasy.io` |
| 房间号规则 | `Math.floor(1000 + Math.random()*9000)` → 4 位（1000–9999），作为 pubsub channel |
| 棋子图 | `w<类型>.png` / `b<类型>.png`，类型大写（P/N/B/R/Q/K） |

## 消息协议（JSON over GoEasy pubsub）

所有消息 `JSON.stringify(payload)` 后 `publish` 到 `channel = roomId`，每条消息带 `sender = myColor`（'white'/'black'），接收方丢弃 `sender === myColor` 的回声。

| type | 字段 | 说明 |
|------|------|------|
| `move` | fen, pgn, moveInfo, identity | 对方走子；收到后 load_pgn/load 并重渲染 |
| `sync` | fen, pgn, identity | 全量状态同步（订阅成功 / 悔棋 / 重开后广播） |
| `request_sync` | identity | 订阅成功后主动要一份最新状态 |
| `request_undo` | — | 请求悔棋 |
| `agree_undo` / `reject_undo` | — | 同意 / 拒绝悔棋 |
| `request_restart` | — | 请求重开 |
| `agree_restart` / `reject_restart` | — | 同意 / 拒绝重开 |
| `emoji` | value | 飘动表情 |

## 走子 / 悔棋 / 重开 规则（两边必须一致）

- **走子**：本地落子后 `broadcast(move)`；对方收到后若 `fen` 不同则 `load_pgn(pgn)`（否则 `load(fen)`）并重渲染。
- **悔棋**：当前轮到自己且 `history < 2` 禁止；网络局 `undo` 步数 = 2（回退到对方走之前），单机局 = 1；需对方 `agree_undo` 才执行。
- **重开**：双方 `reset()` 后 `broadcast(sync)`。
- **连接失败兜底**：GoEasy 连接失败 → 进入「同屏单机对战」（`roomId='本地'`，无广播，轮流转交由本地 `onSquareTap` 自然实现），与小程序版行为一致。

## 本地运行

直接用任意静态服务器打开 `index.html` 即可，例如：

```bash
cd chess-web
python3 -m http.server 8080
# 浏览器访问 http://localhost:8080
```

## 发布到 chess.yiruifu.cn

整目录上传即可（已无外部 CDN 依赖）。如仍走 CDN，可把 `libs/` 改回 `<script src="https://cdn.jsdelivr.net/npm/...">`。
