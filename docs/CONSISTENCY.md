# 网页版 ↔ 小程序版 一致性对照报告

> 生成时间：2026-09-11
> 网页版：`chess-web/`（本目录）　小程序版：`WeChatProjects/miniprogram-1`
> 目标：两边共用同一 GoEasy appkey 与房间协议，房间互通、一起开发一起更新。

## 一、结论

**协议级一致 ✓** —— 实时通信、房间、走子/悔棋/重开、升变、视角、表情全部对齐。
仅剩差异均为**平台必然性差异**（加载方式、原生 API），不属于"实现偏差"。
本轮已消除唯一实质分歧：离线同屏兜底（网页版原先只弹窗拒绝，现已补 `fallbackLocal` 与小程序对齐）。

---

## 二、已对齐项（两边行为完全相同）

| 维度 | 网页版 | 小程序版 |
|------|--------|----------|
| GoEasy Appkey | `BC-58fd21e3fbff443587a9b9f35137cb4e` | 同左 |
| GoEasy host | `hangzhou.goeasy.io` | 同左 |
| 房间号生成 | `floor(1000+rand*9000)` → 4位 | 同左 |
| channel | `= roomId` | 同左 |
| 消息协议 | move/sync/request_sync/request_undo/agree_undo/reject_undo/request_restart/agree_restart/reject_restart/emoji/room_check/room_info/spectator_joined/spectator_left | 同左 |
| 消息字段 | `{sender, fen, pgn, moveInfo, identity, value}` | 同左 |
| 回声过滤 | `data.sender === myColor` 丢弃 | 同左 |
| 走子同步 | `load_pgn(pgn)` 否则 `load(fen)` | 同左 |
| 悔棋步数 | 联网局=2 / 单机局=1；自己回合且 history<2 禁止 | 同左 |
| 重开 | 双方 `reset()` + `broadcast(sync)` | 同左 |
| 升变 | 弹窗选 Q/R/N/B，默认 `promotion:'q'` | 同左 |
| 视角翻转 | myColor==='black' 时翻转 ranks/files | 同左 |
| 高亮类 | white-sq/black-sq/highlight-select/hint/capture/in-check/last-move | 同左 |
| 身份与表情 | 米爸/米妈/小米米；15 个 emoji 一致 | 同左 |
| 断线恢复 | localStorage `chessSave` | wx.Storage `chessSave`（等价） |
| **离线同屏兜底** | **`fallbackLocal()` → roomId='本地'** | **同左（本轮新增对齐）** |

---

## 三、平台必然性差异（非 bug，不改）

| 差异点 | 网页版 | 小程序版 | 说明 |
|--------|--------|----------|------|
| chess.js 引入 | `<script>` UMD 全局 `Chess` (`libs/chess.min.js`) | `require('utils/chess.js')` | 同一 0.10.3 引擎 |
| GoEasy 引入 | `<script src="libs/goeasy.js">` | `require('utils/goeasy.js')` | 同一通用构建（含 wx 支持） |
| 棋子图来源 | 本地 `images/pieces/*.png` | 本地 `/images/pieces/*.png` | 同一套 12 张，避免 CDN/域名配置 |
| 走子反馈 | WebAudio `playTone` 音效 | `wx.vibrateShort` 振动 | 平台原生 API |
| 弹窗 | `alert` / `confirm` | `wx.showModal` / `wx.showToast` | 平台原生 UI |
| 连接失败提示 | `network-warning` 横幅 | `networkWarning` 横幅 | 文案一致 |

---

## 四、连接失败根因与修复（独立于代码）

临床症状：点"建房"报"连接失败"。勾选「不校验合法域名」后可正常建房 → **根因 100% 是域名白名单**：`wss://<GoEasy host>` 未命中小程序「socket 合法域名」→ `wx.connectSocket` 被微信网络层拒绝 → GoEasy `onFailed`。

### 4.1 配置入口（官方路径）
`mp.weixin.qq.com` → 左侧「**开发**」→「**开发管理**」→ 顶部「**开发设置**」→ 页面**向下滚动** → 「**服务器域名**」区域 → 点「**修改**」。
- 该页共 8 栏：request / **socket** / uploadFile / downloadFile / udp / tcp / DNS预解析 / 预连接。
- **本项目只需填「socket 合法域名」一栏**，实测确认需填 **5 个节点域名**（用 `;` 分隔，缺一不可）：
  ```
  wss://1hangzhou.goeasy.io;wss://2hangzhou.goeasy.io;wss://3hangzhou.goeasy.io;wss://4hangzhou.goeasy.io;wss://5hangzhou.goeasy.io
  ```
  其余 7 栏**留空**。
- 理由：代码只用 `modules:['pubsub']`（WebSocket），无 `wx.request` / 上传 / 下载 / udp / tcp，故其余域名都不需要。
- 需要**管理员或开发者**权限扫码登录；入口不在「设置」菜单里，且要往下滑才可见。

### 4.2 个人主体能不能配？（结论：能）
- 微信官方仅对「**业务域名**」（web-view 用）限制个人主体：「个人主体小程序、小游戏不支持业务域名功能」「后台无相关设置入口」。
- 「**服务器域名**」（含 socket 合法域名）个人主体**可以配置**，与 web-view 无关。所以你的问题不是主体限制，是**入口/滚动位置**问题。

### 4.3 该填哪个域名（**已实测确认**）
**实测结论（2026-09-11，控制台日志证据）**：SDK 2.14.9 在小程序里对 `host: 'hangzhou.goeasy.io'` **自动加数字前缀做 5 节点负载均衡**，实连 `wss://1hangzhou.goeasy.io` ~ `wss://5hangzhou.goeasy.io`。

> 证据：`[WS] 实际连接地址 → wss://2hangzhou.goeasy.io/socket.io/?EIO=3&transport=websocket&b64=1`，并在报错中轮询 `1~5` 全部命中「不在 socket 合法域名列表中」。
> （SDK 内无硬编码域名，是运行时 `节点号 + host` 拼接，故 grep 不到 `2hangzhou` 字样。）

| 域名 | 是否需要 |
|------|----------|
| `wss://1hangzhou.goeasy.io` ~ `wss://5hangzhou.goeasy.io` | ✅ **5 个全加**（缺一不可） |
| `wss://hangzhou.goeasy.io` | ❌ 不需要（SDK 不直接连它） |
| `wss://wx-hangzhou.goeasy.io` | ❌ 不需要（1.x 旧版） |

**最权威的确认方法**（已验证有效）：取消勾选「不校验合法域名…」→ 重新编译 → 控制台打印 `[WS] 实际连接地址 → wss://...`，把它直接填进白名单。

**改完后台后必做**：开发者工具 →「**详情 → 域名信息**」→ 刷新（或重新打开项目），否则用的是旧缓存。

### 4.4 备案这个坑（务必实测确认）
- 官方硬规则：服务器域名**必须 ICP 备案**（`developers.weixin.qq.com/…/network.html`）。
- `goeasy.io` 是 `.io` 后缀，工信部未收录、**原则上无法备案**。
- 但 GoEasy 官方明确指引配置其域名，大量开发者实践成功 → 微信对这些第三方服务域名实际是放行的。**以你后台保存结果为准**：能保存即 OK；若报「该域名未备案 / 不可设置」，需走备选方案。

### 4.5 若 .io 配不上，备选方案
- A. **自有已备案域名反代**：若 `yiruifu.cn` 已备案且备案主体与你（小程序主体）一致，可用子域（如 `ws.yiruifu.cn`）反向代理到 GoEasy。
- B. **微信云托管 / 自建 WebSocket**：官方明确「使用微信云托管作为后端服务可**无需配置通讯域名**」（走 callContainer / connectContainer）。可用 CloudBase 云托管自建 WS，替换 GoEasy。
- C. **家庭自用临时**：体验版 + 手机开启「调试模式」可不校验域名（不发布上线，仅自用）。

### 4.6 代码上传：两条通道（**2026-09-11 已跑通方案 B**）

| | 方案 A：开发者工具 CLI | 方案 B：miniprogram-ci（**已采用**） |
|---|---|---|
| 依赖 | 必须开着微信开发者工具窗口 | **完全无 GUI**，命令行/双击即传 |
| 前置 | 设置 → 安全设置 → 开启「服务端口」 | 后台生成「小程序代码上传密钥」 |
| 稳定性 | 工具关掉后 CLI 拉起会卡 `wait IDE port timeout`（已实测失败） | 稳定，40s 完成 |
| 版本归属 | 显示你自己的微信号 | 显示「**ci机器人1**」（见下） |

#### 4.6.1 方案 B 用法

**一键（推荐）**：双击 `WeChatProjects/upload_miniprogram.bat` → 按提示输入版本号与备注。

**带参**：`upload_miniprogram.bat 1.1.4 "清理开发期调试代码"`

**底层命令**（bat 内部做的事）：
```
cd C:\Users\52300\WeChatProjects\.ci-secrets
set NODE_PATH=C:\Users\52300\.workbuddy\binaries\node\workspace\node_modules
set APPDATA=C:\Users\52300\AppData\Roaming
<managed node> wx-upload.js 1.1.4 "备注"
```

#### 4.6.2 三个必须知道的坑（踩过）

1. **`APPDATA` 必须是有效值**。当前 shell 里 `APPDATA` 为空，会让 miniprogram-ci 的依赖 `npm-conf` 抛
   `TypeError: The "paths[0]" argument must be of type string`。bat / 脚本里显式 `set APPDATA=...` 解决。
2. **加载方式用 `NODE_PATH`**，不要在 `.ci-secrets` 里另装一份 node_modules；托管 node：
   `C:\Users\52300\.workbuddy\binaries\node\versions\24.14.0\node.exe`，包在 `...\node\workspace\node_modules`。
3. **bat 必须纯 ASCII**（沿用 `restart_autoreply.bat` 的三条铁律）：`chcp 65001` + UTF-8 中文会让 cmd 立即中止整个批处理；
   中文输出只允许由 `node.exe` 打印，不能写进 bat 文本。

#### 4.6.3 关于「开发者 = ci机器人1」

miniprogram-ci 上传的版本，后台「版本管理」里**开发者一栏显示的是 CI 机器人代号，不是你的微信号**。
上传请求带 `robot=1`（实测 URL 可见 `&robot=1&`），所以显示「ci机器人1」。
- 这是**来源标记，不是权限问题**，不影响体验版、不影响提交审核。
- 密钥本身就是在后台「小程序代码上传 → ci机器人1」下生成的，两边编号一致。
- 想改成别的编号：后台给另一个机器人（2~30）生成密钥，并在 `wx-upload.js` 的 `ci.upload()` 里加 `robot: 2`。
- 想显示回你自己的账号：用方案 A（开发者工具）上传即可。

#### 4.6.4 已知无害告警

```
try to get input sourcemap of .../utils/chess.js catch error TypeError ...
```
原因：`utils/chess.js`（压缩版）第 13 行残留 `//# sourceMappingURL=/sm/....map`，指向一个包里不存在的 map。
**不影响上传与运行**，可忽略；或删掉该注释行消除告警（属清理动作，先确认再删）。

---

## 五、同步开发约定（防漂移）

- 改 appkey / host / 房间号规则 / 任一消息 type 或字段 → **两边必须同步改**，并同步更新本文件与 README。
- 新增交互（如新消息类型、新按钮）→ 先定协议，再各自实现，附跨端自测（网页建房↔小程序加入）。
- 棋子图、emoji 列表、身份列表属"展示数据"，改任一边需同步另一边。
