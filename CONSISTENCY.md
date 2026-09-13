# 网页版 ↔ 小程序版 一致性对照报告

> 生成时间：2026-09-11（最后更新：2026-09-13）
> 网页版：`docs/`（本目录，GitHub Pages，**冻结于 v1.3.8**）　小程序版：`WeChatProjects/miniprogram-1`（**v1.4.8**）
> 目标：两边共用同一 GoEasy appkey 与房间协议，房间互通、一起开发一起更新。
>
> ⚠️ **2026-09-12 范围变更**：网页版停止更新（决策见 `SEAT_TOKEN_DESIGN.md` §零 R3/R4）。
> 此后新增能力（§11 座位令牌/换边、§12 观战者名册握手）**均为小程序端独有**，
> 但**房间互通性保持** —— 网页版忽略未知消息类型，且降级路径已逐条记录（§11.4、§12.4）。

## 一、结论

**协议级一致 ✓** —— 实时通信、房间、走子/悔棋/重开、升变、视角、表情全部对齐。
仅剩差异均为**平台必然性差异**（加载方式、原生 API），不属于"实现偏差"。
本轮已消除唯一实质分歧：离线同屏兜底（网页版原先只弹窗拒绝，现已补 `fallbackLocal` 与小程序对齐）。

**v1.4.1 状态**：小程序端已超出网页版基线两个版本（v1.4.0 座位令牌/换边、v1.4.1 观战者名册握手）。
两版均**不与网页版冲突**：网页版收不到新消息即忽略，遇到网页版在场时相关机制整体降级为 v1.3.8 行为。

**v1.4.2 状态**：入口按钮去掉「执白/执黑」（§13）。这是**唯一一处两端文案有意分叉** ——
网页版没有换边功能，它的颜色承诺仍然成立；小程序端有了换边，颜色就不再由入口决定。

**v1.4.3–v1.4.8 状态**：六轮均为**小程序端独有**的迭代 ——
v1.4.3 重进房间认得自己的座位（§14）、v1.4.4 手输同号等同恢复掉线对局（§15）、
v1.4.5 修 `bindtap` 传事件对象导致的恢复按钮失灵 + 存档防污染 + **删除「本地同屏对局」**（§16）。
v1.4.6 修「恢复按钮消失」+「认座成功却仍要重选身份」（**同一个根因：判据混用**，§17）——
把"存档可恢复"与"存档属于本房"两层判据拆开，并止住 joinRoom/createRoom 无条件删别房存档的行为。
v1.4.7 对局页控件重排：设置区（路径提示/声音/皮肤）上移到棋盘上方、底部按钮改两列、
新增退出房间（§18）—— **纯 UI 重排 + 为后续皮肤/声效预留入口**。
v1.4.8 把换边按钮从 v1.4.7 的"不可用即灰显"改为"**始终可点 + 提示具体原因**"（§18.3）——
用户实测后改主意：灰显说不出"为什么不能用"，而 `requestSwap` 的 7 条前置检查本来就能逐条说明。
两版均不触碰实时协议，因此对网页版零影响。
其中 v1.4.5 是唯一一处**能力削减**（不再支持同屏单机对战）—— 该模式与网页版本就无对应，
删除后小程序端**反而更贴近**网页版「必须进房才能落子」的既有语义。

---

## 二、已对齐项（两边行为完全相同）

| 维度 | 网页版 | 小程序版 |
|------|--------|----------|
| GoEasy Appkey | `BC-58fd21e3fbff443587a9b9f35137cb4e` | 同左 |
| GoEasy host | `hangzhou.goeasy.io` | 同左 |
| 房间号生成 | `floor(1000+rand*9000)` → 4位 | 同左 |
| channel | `= roomId` | 同左 |
| 消息协议 | move/sync/request_sync/request_undo/agree_undo/reject_undo/request_restart/agree_restart/reject_restart/emoji/room_check/room_info/spectator_joined/spectator_left | 同左 |
| 消息字段 | `{sender, target, fen, pgn, moveInfo, identity, value, color}` | 同左 |
| 收件人字段 `target` | 悔棋/重开全系列**必带**（=对方颜色）；非收件人忽略；老版本无 target 时按老逻辑兜底 | 同左 |
| 观战者权限 | 只读：收到 request_undo/agree_undo/reject_undo/request_restart/agree_restart/reject_restart 一律忽略；**发送端白名单**只允许 request_sync / spectator_joined / spectator_left / room_check / room_info / emoji | 同左 |
| request_sync 应答 | 只有对弈者应答，观战者不应答（避免用过期棋局覆盖对局） | 同左 |
| 观战者身份栏 | 上=黑方、下=白方，显示双方真实身份（按白方视角看棋盘） | 同左 |
| 悔棋请求超时 | 20s 未收到应答自动取消并复位按钮 | 同左 |
| 交叉悔棋 | 自己正在等应答时收到对方请求 → 直接回 reject，避免双方交叉撤销 | 同左 |
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
| 弹窗机制 | 页内：通知条 `notice` + 棋盘上的确认面板 `prompt`（**零** `alert`/`confirm`） | 同左（**零** `wx.showModal`/`wx.showToast`），见第六节 |
| 身份选择时机 | **进房后**再选（`needIdentityPick`），面板标"已被选"且禁选已占用 | 同左，见第八节 |
| 空房落子守卫 | 在线模式未收到对方颜色消息前不许落子（`opponentJoined`） | 同左，见第八节 |
| 取消选身份 | = 退出房间（广播 `spectator_left` + `unsubscribe`） | 同左 |
| 版本号展示 | 页面右下角灰色小字 `v1.3.8`（`APP_VERSION`，**已冻结**） | 同左（`version` 字段绑定），小程序端当前 **v1.4.8** |
| 入口按钮文案 | `新建房间 (执白)` / `加入房间 (执黑)` —— **网页版无换边，此文案仍准确** | 小程序端 **v1.4.2 起去掉颜色**：「新建房间」/「加入房间」，见 §13 |
| GoEasy 实例 | 模块级单例，`initGoEasy()` 只建一次（`_goeasySingleton`） | 同左；另有 `_goeasyConnected` 记录连接态（见 5.3） |
| 结局提示 | 观战者沿用「某某获胜」；对局者改为「你赢了 / 你输了」+ 配色类；**覆盖层半透明**（alpha 0.68，不糊棋盘） | 同左，见 5.3 |
| 轮次提示 | `active-turn` 带渐变流动 + 扫光；动画作用在既有 ●/○ 上（**不得**再加 `::before` 圆点） | 同左 |

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

## 五、观战者权限模型（v1.2.6 新增，两端同构）

### 5.1 问题

房间就是 GoEasy 的一个频道，**白方、黑方、观战者都收到全部消息** —— 协议层没有"私聊"。
于是出现了三类越权（都不是显示问题，是逻辑缺陷）：

| # | 缺陷 | 后果 |
|---|------|------|
| 1 | 悔棋/重开请求是广播的，观战者也会收到并弹出"是否同意" | 观战者能同意/拒绝别人的对局操作 |
| 2 | `agree_undo` / `agree_restart` 也是广播的，观战者收到后会执行 `executeUndo()` / `executeRestart()` | 观战者本地棋局被改写 |
| 3 | 观战者执行后会继续 `broadcast({type:'sync'})`，而它手上**不是权威棋局** | **观战者能用自己的棋局覆盖整局对局**（越权最严重的一条） |

另外 `request_sync` 的应答没有做身份限制，观战者也会应答，同样会用过期棋局污染对局。

### 5.2 规则（两端必须逐条实现）

1. **发送端白名单**：`isSpectator` 时只允许发
   `request_sync` / `spectator_joined` / `spectator_left` / `room_check` / `room_info` / `emoji`；
   其余（`move` / `sync` / `request_undo` / `agree_undo` / `reject_undo` / `request_restart` / `agree_restart` / `reject_restart`）
   **在 broadcast 出口直接拦截**。这是最关键的一道闸 —— 即使 UI 有疏漏也污染不了对局。
2. **接收端忽略**：`isSpectator` 时收到 `PLAYER_ONLY_RECV` 那 6 种消息**立即 return**（不弹窗、不执行）。
3. **收件人校验**：悔棋/重开全系列带 `target`（= 对方颜色）。
   接收方 `if (data.target && data.target !== myColor) return`。
   不带 `target` 视为老版本发的，按老逻辑处理 → **新旧版本可混用**。
4. **只有对弈者应答 `request_sync`**。
5. **动作层双保险**：`executeUndo()` / `executeRestart()` 开头 `if (isSpectator) return`。
6. **观战者身份栏**：观战者不是对局方，上下两栏显示白方/黑方**真实身份**
   （身份通过收到的 `move`/`sync`/`request_sync`/`room_info` 里的 `sender` + `identity` 记录）。
   观战者按白方视角看棋盘 → 上=黑方、下=白方。

### 5.3 顺带修掉的同类缺陷

| 缺陷 | 表现 | 修法 |
|------|------|------|
| 悔棋请求无超时 | 对方掉线时不回，请求方永远卡在「⏳ 等待同意」 | 20s 定时器自动取消并复位 |
| 双方同时请求悔棋 | 两边都在 pending，交叉 `agree` 后撤销步数错乱 | 自己 pending 时收到对方请求 → 直接回 `reject` |
| `load_pgn` / `load` 无保护 | 对端发来残缺 fen/pgn 时抛错，整条消息链中断 | try/catch，失败只跳过这一条 |
| 观战者收到走子会振动 | 只读方不该产生走子反馈 | 观战者只刷新棋盘 |
| 观战者看到自己被宣布为冠军 | 将杀提示用 `myIdentity`/`oppIdentity` 取胜者，而观战者**不是对局方**，这两个字段指向观战者自己 → 把观众名当冠军 | 观战者改用身份表 `_playerIdentities[winnerColor]`；对弈者逻辑不变 |
| 网页版 `#status` 元素已删但代码仍在写它 | `getElementById` 返回 null → `TypeError`。**`requestUndo`/`requestRestart` 在 broadcast 之前抛错 → 悔棋/重开请求根本发不出去**；`window.onload` 里同样抛错 → `checkSavedGame()` 执行不到 → 「恢复刚才断线的对局」按钮永不显示 | 移除全部 `#status` 访问 |
| **小程序加入房间永久卡在「正在检查房间…」（v1.3.4）** | GoEasy SDK 是**模块级单例**：`getInstance()` 内部走 `init()`，而 `init()` 在"已连接"状态下抛 `Initialization failed. Please disconnect and try again.`。小程序页面共享同一 JS 上下文，故**第二次进页面**必然抛错。旧代码 `onLoad` 每次 `this.goeasy = null` 后重建实例 → `onLoad` 在 `initGoEasy()` 处中断：`connectState` 永远 `idle` → 点「加入房间」落进 `pendingJoin` → 又 `initGoEasy()` → 又抛错 → `roomTip` 永久卡住。**建房看似正常，只因 `doCreateRoom()` 的身份面板不等连接就弹，属巧合掩盖**；点分享卡片（新开页面实例）则必然踩中 | 实例提到模块作用域只建一次（`_goeasySingleton`）+ `_goeasyConnected` 记录连接态；`connectRoom` 改为先 `unsubscribe` 再 `subscribe`，保证回调绑定当前实例且不重复登记 |
| **表情面板点开后上半部分被遮住（v1.3.4，网页端）** | `#game-controls` 用了 `overflow-x: auto`，绝对定位的 `#emoji-panel`（`position:absolute`）被这个滚动容器裁掉 | `#game-controls` 改 `overflow: visible` + `flex-wrap: wrap`；面板加 `clampEmojiPanel()` 视口避让（窄屏也不切图） |
| **将杀结局覆盖层完全糊住棋盘（v1.3.5）** | 结局层底色 `rgba(255,255,255,0.9)`、胜负主题色用**不透明 hex 渐变** → 终局后完全看不到最后一步棋的盘面，等于把"怎么被将死的"藏起来了 | 底色降到 alpha 0.68，胜负/平局主题色一律改 **rgba 半透明渐变**（0.70→0.76）；文字改用**白色光晕** `text-shadow` 保证在深浅格上都可读 |
| **轮次动画与既有 ●/○ 标记重复（v1.3.5）** | 玩家名文本里已内嵌回合标记（`render()` 写入 `oppDot`/`myDot`：`●` 该走 / `○` 待走），又额外加了 `.active-turn .player-name::before` 脉冲圆点 → 同一行出现**两个符号** | 删掉名字里的 ●/○（及 `oppDot`/`myDot`/`topDot`/`botDot` 变量），**保留** `::before` 脉冲圆点作为唯一轮次指示，幅度 `scale(1.55)` 保证可见 |
| **掉线方座位被静默接手（v1.3.6，用户实测）** | 只有观战者离开会广播，**对弈者掉线完全静默**；`doJoinRoom` 又只看 `_roomColors`（掉线方不会应答 `room_info`）→ 掉线方座位被判成空位；新人进房发 `request_sync` 就拿到整盘残局，直接顶替那位子 | 座位租约：5s 心跳 `ping` 续租 + 60s 宽限期锁定；`room_check` 应答补发 `seat_state`；空位判定改两路取或；非对局座位请求 `sync` 先弹接手确认面板，拒绝则 `reject_takeover` → 请求方转观战 |
| **网页端身份面板自定义输入行不随选项收起（v1.3.6，用户实测）** | 点过「自定义...」后再改点预设身份，输入框不消失 → `confirmRolePicker()` 仍走"自定义"分支 → 卡在「请输入自定义名称」，只能以自定义身份加入。小程序端无此问题 | 预设选项 onclick 显式 `display='none'`；`renderRolePicker()` 末尾加收敛兜底（`_rolePickIdx>=0` 即收起） |
| **观战者误触发接手确认面板 → 双方双双弹窗（v1.3.8，用户实测）** | 第三个进房观战者确认身份后广播 `request_sync`（取当前棋局渲染棋盘，属白名单合法消息），而接手守卫只判「请求方不是我对手」，`sender='spectator'` 恒 ≠ `oppColor()` → 守卫成立 → 两个对局者各弹一次「有人想接手这局棋」 | 接手守卫加前提：必须 `requesterIsPlayer`（请求方本身是白/黑座位）才进入确认流程；观战者的 `request_sync` 走正常回 sync；连带修 `request_sync` 的身份写入不再接受观战者 `identity` |
| **【P0】v1.3.6 座位租约把新人自己锁死 → 后进房直接变观战（用户实测）** | `joinRoom()` 先把 `myColor` 预设为 `'black'` → 新人广播 `room_check` 时 `sender='black'` → `room_check` 在回声过滤里被豁免，而 `touchSeat(data.sender)` 对它无条件生效 → 房主误以为黑座有人并通过 `seat_state` 回告 → 新人吃下后自己的黑座也被锁死 → `hasWhite && hasBlack` → 强制观战，**压根无法对局** | ① `touchSeat` 忽略**自己的座位**（自己在线由自己的心跳维护，不靠"收到自己的消息"续租）；② `room_check` 不参与续租（进房前探房消息，发送方还没决定座位，`sender` 只是脚手架值） |

### 5.4 回归验证

两端各有一份 Node 探针，**改协议后必须跑过**：

- 小程序：`WeChatProjects/.ci-secrets/probe-index-page.js`（152 项，覆盖观战者/收件人/超时/交叉/脏数据/身份栏/将杀胜者/页内提示与冻结/**进房后选身份**/**空房不许落子**/对弈者回归/**GoEasy单例复用**/**分享卡片自动进房**/**结局输赢配色**/**覆盖层半透明**/**轮次动画不重复**/**座位租约与残局接手守卫**/**P0：新人不得被自己锁死**）
- 网页版：`WeChatProjects/.ci-secrets/probe-web.js`（116 项，同一批场景 + **表情面板不裁切**/**结局配色**/**回合动画**/**覆盖层透明度**/**无重复圆点**/**身份面板自定义行收起**/**座位租约**/**P0：新人不得被自己锁死**）

做法：打桩 `wx`（或 `document`/`window`/`localStorage`）与 GoEasy，加载**真实** `pages/index/index.js`
或 `docs/index.html` 的内联脚本，然后手工喂 `onMessage` 消息，断言"有没有提示 / 有没有 publish / 棋局 fen 变没变"。

两份探针都支持从命令行传源码路径（`node probe.js <源码路径>`），
所以可对 `git show <旧commit>:<文件>` 导出的**修复前版本**跑一遍，
确认新断言在旧代码上**确实失败**（证明断言有效、不是恒真）—— 这一步是新增断言时的强制动作。
例：将杀胜者那条在修复前会打出 `🎉 将杀！👨🏻 米爸 获胜！`（把观战者那侧名字当冠军），修复后通过。

---

## 六、页内提示模型（v1.3.0 新增，两端同构）

### 6.1 为什么不再用系统弹窗

`wx.showModal` / `wx.showToast`（小程序）与 `confirm` / `alert`（网页）都是**系统级**弹层：
样式不可控、两端观感不一致，而且 `confirm`/`alert` 会**阻塞 JS 主线程**、弹窗期间页面完全冻结不可控。
改成页面内机制后，提示由我们自己的 WXSS/CSS 控制，且能精确控制"冻结什么、不冻结什么"。

### 6.2 两个机制

| 机制 | 替代 | 位置 | 是否冻结棋盘 |
|---|---|---|---|
| **notice** 通知条 | `showToast` / `alert` | 页面顶部居中，约 2.2s 自动消失 | **不冻结**（纯告知） |
| **prompt** 确认面板 | `showModal` / `confirm` | 覆盖在棋盘之上（`#board-container` 内绝对定位） | **冻结**（见 6.3） |

`prompt` 由 `mode` 驱动，两端 mode 表必须一致：

| mode | 触发 | 确认 | 取消 |
|---|---|---|---|
| `recv-undo` | 收到 `request_undo` | 回 `agree_undo`（撤销由**发起方**执行） | 回 `reject_undo` |
| `recv-restart` | 收到 `request_restart` | 回 `agree_restart` + 本地 `executeRestart()` | 回 `reject_restart` |
| `send-undo` | 点"悔棋" | 置 pending + 发 `request_undo` + 起超时 | 无动作 |
| `send-restart` | 点"重开" | 置 pending + 发 `request_restart` + 起超时 | 无动作 |
| `solo-restart` | 单机模式点"重开" | `executeRestart()` | 无动作 |

### 6.3 棋盘冻结（两道）

1. **覆盖层**：`prompt` 覆盖整个棋盘容器，点击落不到格子上。
2. **JS 守卫**（双保险，防覆盖层没铺满的极端情况）：`onSquareTap` / `handleSquareClick` 开头
   `if (promptShow) return`。
   另外 `requestUndo` / `requestRestart` 开头也 guard，避免"在面板上再叠一个面板"。

### 6.4 版本号

页面角落固定位置显示 `v<APP_VERSION>` 灰色小字，`pointer-events:none` 不拦点击。
`APP_VERSION` 在两端各有一个常量，**必须与头部注释同步**。

---

## 七、系统性扫描（v1.3.0）与挖出的缺陷

### 7.1 扫描矩阵（比 v1.2.6 更系统）

上一轮是按"观战者视角"顺藤摸瓜；这一轮改成**矩阵穷举**，四维交叉：

| 维度 | 取值 |
|---|---|
| 角色 | 白方 / 黑方 / 观战者 / 第三者试图加入 / 断线重连者 |
| 消息 | `move` `sync` `request_sync` `request_undo` `agree_undo` `reject_undo` `request_restart` `agree_restart` `reject_restart` `room_check` `room_info` `spectator_joined` `spectator_left` `emoji` |
| 时序 | 单方发起 / 双方同时 / 超时 / 迟到应答 / 断线 |
| 状态 | 对局中 / 已终局 / 升变未决 / **提示面板显示中** / 悔棋 pending / 重开 pending / 观战 |

重点补的是"**面板显示中**"这一个新状态，以及"迟到的应答"这一时序 —— 上一轮没覆盖。

### 7.2 挖出的缺陷（全部修复）

| # | 缺陷 | 触发场景 | 后果 | 修法 |
|---|---|---|---|---|
| 1 | 面板槽位被覆盖 | 收到悔棋请求（面板已弹出），对方又发来重开请求 | 面板被换掉，用户点"同意"实际应答了**另一个**请求 | 面板占用时新请求**按同类型直接拒绝**（busy 保护） |
| 2 | 交叉保护不跨类型 | 我方悔棋 pending 时，对方发来**重开**请求 | 老代码只拦 undo↔undo，这类请求会照常弹面板 → 交叉操作错乱 | busy 判定统一为"promptShow ‖ 任一 pending"，且**按收到的类型**回对应 reject |
| 3 | 迟到/重复 `agree_undo` | 悔棋请求已超时取消，对方 `agree` 才到；或 `agree` 重发 | 无条件 `executeUndo()` → **多撤一步** | `agree_undo`/`reject_undo` 仅在 `isUndoPending` 为真时处理 |
| 4 | 重开没有 pending 概念 | 陈旧 `agree_restart` 到达（如上一次请求的应答） | 无条件 `executeRestart()` → **把正在下的棋重置掉** | 新增 `isRestartPending` + 超时 + `agree_restart`/`reject_restart` 守卫（与悔棋对称） |
| 5 | 面板可永久冻结棋盘 | 收到请求后面板弹出，但本方一直没点（人离开了） | 棋盘**永久不可操作** | 面板 25s 自动关闭（`PROMPT_TIMEOUT`）；`send-*` 面板超时等同取消 |
| 6 | 离开页面不清状态 | 面板显示中退出 / 定时器仍在跑 | 卸载后 `setData` 告警；状态残留到下次进入 | `onUnload`（网页 `beforeunload`）清全部定时器与面板态 |
| 7 | 未决升变遇对端同步 | 升变面板显示中收到 `move`/`sync` | 棋局已换，升变面板残留 → 再点确认会按**旧格**走子 | 同步覆盖棋局时把 `pendingPromotionMove` 作废并关面板 |
| 8 | 文本注入 | 身份允许自定义，若含 `<`/`&` 等字符 | 网页用 `innerHTML` 会解析成标记 | 通知条/面板一律 `textContent`（小程序 `{{}}` 天然转义） |

### 7.3 本轮明确"知道但没改"的

- **加入房间的 3 秒房满判定窗口**：两人在 3s 窗口内同时加入，理论上可能都认为自己有空位。
  属既有设计（靠 GoEasy 频道 + 延时收集 `room_info`），要彻底解决需引入服务端仲裁，超出本次范围。
- **同一身份重名**：v1.3.1 已把身份选择移到进房之后，面板会标注"已被选"并**禁止选已被占用的身份**，
  同时在对方与你撞名时给出提示；但**极端竞态**（两人几乎同时确认同一个身份）仍可能各选一个同名 ——
  此时只提示、不强制改名（需要服务端仲裁才能真正杜绝）。

---

## 八、身份选择模型（v1.3.1 新增，两端同构）

### 8.1 为什么必须"进房后再选"

v1.2.5～v1.3.0 的流程是 **先选身份 → 再进房**（`createRoom()`/`joinRoom()` 直接弹身份面板，
确认后才 `connectRoom` + `subscribe`）。结果是身份面板里的"已被选"标记**永远是空的**：

- 占用列表 `_roomIdentities` 的唯一数据来源是房间里其他成员的消息
  （`room_info` / `request_sync` / `move` / `sync` / `spectator_joined` 里的 `identity` 字段）；
- 而那时**还没订阅频道**，一条房间消息都收不到 → 重复检验形同虚设。

所以 v1.3.1 把顺序反过来：**先订阅进房 → 收集房内真实身份 → 再弹身份面板**。

### 8.2 两条建房/加入路径

| 路径 | 进房动作 | 何时弹面板 | 面板里的占用列表 |
|---|---|---|---|
| **建房** `createRoom()` | 立刻分配房号 + `showGameUI` + `subscribe` | 订阅成功即弹（房里只有自己，列表为空） | 之后有人进房会通过 `room_info`/`request_sync` 实时补上 |
| **加入** `joinRoom()` | 立刻 `subscribe` + 发 `room_check` | **等 `ROOM_CHECK_MS`(3000ms) 房况检查结束**再弹 | 检查期内收到各成员的 `room_info`，已填好 |

加入路径为什么必须等 3s：房况检查同时决定"**有空位 → 当对弈者**"还是"**已满 → 转观战**"，
而占用列表也要靠这段时间的 `room_info` 才能填上；两者共用同一个等待窗口。

### 8.3 规则（两端逐条实现）

1. **进房即锁棋盘**：`needIdentityPick` 为真期间
   - `onSquareTap` / `handleSquareClick` 直接 `return`（不让落子）；
   - `requestUndo` / `requestRestart` 只出通知条"请先选择身份"，不弹面板。
2. **身份未定不广播身份**：`subscribe` 的 `onSuccess` 里
   `if (needIdentityPick) return` —— 否则会把脚手架默认名（`👨🏻 米爸`）当成本人身份发出去，
   污染对端的"已被占用"列表。**确认身份后**才补发 `request_sync`（观战者先补发 `spectator_joined`）。
3. **房满应答不带假身份**：响应别人的 `room_check` 时，若自己还没选身份，`room_info.identity` 填**空串**；
   房满判定只看 `color`，与 `identity` 无关。
4. **占用列表实时刷新**：`refreshRoleOccupied()` 在每次收到新身份时重算"已被选"标记；
   若**自己当前选中的那个身份刚被别人占了**，立刻清空选中，逼用户换一个。
5. **确认时二次校验**：`confirmRolePicker()` 落定前再查一次 `rolePickerOccupied`，
   命中则提示"该身份已被占用，请另选一个"并保持面板打开（防"面板打开期间对方刚好确认同名"）。
6. **默认避开已占用**：面板打开时默认选中第一个**未被占用**的预设身份；
   预设全被占则自动展开自定义输入。
7. **取消 = 退出房间**：`cancelRolePicker()` → `_leaveRoom()` / `leaveRoom()`
   （广播 `spectator_left`（若观战）→ `unsubscribe` 频道 → 复位全部房间状态 → 回开局面板）。
   面板遮罩**不再**绑 `cancelRolePicker`，避免误触退房。
8. **撞名双向提醒**：收到与 `myIdentity` 相同的身份（且来自对方颜色）→ 通知条提醒一次。
9. **空房不许落子**：收到**对方颜色**（`white`/`black` 且不是自己）的任何房间消息 → `opponentJoined = true`；
   在线模式且 `!opponentJoined` 时落子被拦并提示"等待对手加入…"。
   本地同屏模式（`currentRoom === ''`）不受此约束。观战者消息不解锁。

### 8.4 回归验证

小程序探针 I/J/K 三节（40 项）、网页探针 W23–W34（24 项）覆盖上述全部规则。
反证（对 v1.3.0 旧版跑）：小程序 **31 项 FAIL**、网页版 **19 项 FAIL** —— 证明断言有效、非恒真。

---

## 九、座位租约与在线维持（v1.3.6 新增，v1.3.7 修 P0，v1.3.8 修误报，两端同构）

### 9.1 问题（用户实测发现）

> 「我发现掉线后别人再进房可以直接接手掉线那方的残局。」

两个独立缺陷叠加：

1. **掉线是静默的**：原先只有观战者离开会广播 `spectator_left`。**对弈者掉线没有任何信号**，
   别人只能靠"他还说没说话"来猜。
2. **空位判定只看一个信号**：`doJoinRoom` 用 `_roomColors[color]` 判空位，而它只在
   "对方应答了 `room_check`"时才被填充。掉线方**不会应答** → 他的座位被判成空位。
3. **残局可被静默继承**：新人随后发 `request_sync`，在位的那一方直接回整盘 pgn（含全部历史），
   新人就顶上了那个座位。

### 9.2 规则（两端必须逐条实现）

| # | 规则 | 网页版 | 小程序版 |
|---|------|--------|----------|
| 1 | 心跳间隔 `HEARTBEAT_MS = 5000`，宽限期 `SEAT_GRACE_MS = 60000` | 同 | 同 |
| 2 | 对弈者每 5s 广播 `{type:'ping', seat:mySeat()}` 续租 | `startHeartbeat()` | `_startHeartbeat()` |
| 3 | 任何来自某座位的消息都顺带续租 → `touchSeat(data.sender)`，但**必须忽略自己的座位**（v1.3.7） | ✓ | ✓ |
| 3b | **`room_check` 不参与续租**（进房前探房消息，发送方尚未决定座位，`sender` 是脚手架值）（v1.3.7） | ✓ | ✓ |
| 4 | 收到 `ping` → 用 `data.seat` 再续一次，然后 `return`（不参与对局逻辑） | ✓ | ✓ |
| 5 | `seatLocked(c)` = 最后活跃在 60s 内；`seatExpired(c)` = 超出 60s | ✓ | ✓ |
| 6 | 响应 `room_check` 时**补发** `seat_state{locked,expired}` 播报占用 | ✓ | ✓ |
| 7 | 收到 `seat_state` → 把 locked 记成"刚活跃"、expired 记成"已过期" | ✓ | ✓ |
| 8 | `doJoinRoom` 空位判定改为两路取或：`_roomColors[c] \|\| seatLocked(c)` | ✓ | ✓ |
| 9 | 非对局座位请求 `sync` 且本地有残局 → 弹 `recv-takeover` 确认面板，**不直接发残局** | ✓ | ✓ |
| 9b | **接手守卫须先验请求方本身是白/黑座位**：观战者的 `request_sync` 走正常回 sync，不得弹面板（v1.3.8） | ✓ | ✓ |
| 9c | `request_sync` 的身份写入只接受白/黑座位发来的 `identity`，观战者的不得写入 `oppIdentity`（v1.3.8） | ✓ | ✓ |
| 10 | 同意 → 广播 `sync`；拒绝 → 广播 `reject_takeover{target}` | ✓ | ✓ |
| 11 | 收到 `reject_takeover` → 请求方转为观战 | ✓ | ✓ |
| 12 | `SPECTATOR_ALLOWED_SEND` 追加 `'ping'`（观战者也要能续租） | ✓ | ✓ |
| 13 | 心跳生命周期：`visibilitychange` / `onShow`+`onHide`；`beforeunload`/`onUnload`/`leaveRoom` 停止 | ✓ | ✓ |
| 14 | **换房间清空租约**（`resetSeatLease()` / `_resetSeatLease()`），防跨房间泄漏 | ✓ | ✓ |

第 14 条是写断言时被探针挖出来的：`createRoom()/joinRoom()` 原先只清了 `_roomColors`，
上一房间的 `_seatLastSeen` 会残留，把新房间的空位误判成"有人"（网页探针 W29g/W29h 因此失败）。

### 9.3 为什么不用 GoEasy 原生 presence

GoEasy SDK 确实有 `subscribePresence` / `hereNow` 和 `join`/`back`/`leave`/`timeout` 事件，
但 `validateSubscribePresence` 要求 `connect()` 时显式传 `id`，且受套餐等级限制（免费套餐是否支持不确定）。
改用客户端心跳租约：**可测试**（探针能直接驱动）、**两端对称**、**不依赖套餐**。

### 9.4 网页端身份面板自定义输入行 bug（v1.3.6 顺带修复）

用户反馈：网页端点过「自定义...」后，再改点预设身份，输入框不消失 → 只能以自定义身份加入；小程序端正常。

根因：`role-picker-custom-row` 在 `role-picker-list` **之外**，`renderRolePicker()` 只重建列表、
不碰它。预设选项的 `onclick` 未显式收起它 → `confirmRolePicker()` 里
`customRow.style.display !== 'none'` 成立 → 走"自定义"分支 → `customIdentity` 为空 → 卡在
「请输入自定义名称」。小程序端 `onRolePickerSelect` 有 `setData({showCustomInput:false})`，所以无此问题。

修复两道：① 预设选项 onclick 显式 `display='none'`；② `renderRolePicker()` 末尾加**收敛兜底**
（只要 `_rolePickIdx >= 0` 就收起），防将来新增调用路径再踩坑。

### 9.5 v1.3.7 P0 复盘：座位租约把新人自己锁死

**用户实测**：

> 「现在无法对局了，后进房选择身份直接观战了。」

v1.3.6 刚上线的座位租约引入了**比原 bug 更严重**的回归 —— 原来只是"掉线后残局可能被接手"，
现在变成"**正常对局根本开不起来**"。完整故障链：

1. `joinRoom()` 会把 `myColor` **预设**为 `'black'`（等房况检查结束后才按实际情况修正）
2. 新人广播 `room_check`，`broadcast()` 注入 `sender = myColor = 'black'`
3. `room_check` 在回声过滤里被**豁免**（`data.type !== 'room_check'`），
   而 v1.3.6 新加的 `touchSeat(data.sender)` 对它**无条件生效**
4. 房主收到这条 `room_check` → `touchSeat('black')` → 误以为黑座有人
   → 随后应答的 `seat_state` 把 `black` 报成 `locked`
5. 新人吃下 `seat_state` → 自己的黑座也被锁死 → `hasWhite && hasBlack` → **强制观战**

**教训：脚手架值泄漏成了协议信号。** `myColor` 在进房检查期间只是占位值，
而 `broadcast()` 把它当真实座位号注入了每一条消息 —— 只要这条消息豁免了回声过滤，
任何"收到消息就续租"的逻辑都会把自己洗成"占座"。

**修复两处**（缺一不可）：

| 修复 | 位置 | 理由 |
|------|------|------|
| `touchSeat` 忽略**自己的座位** | 两端 `touchSeat` / `_touchSeat` 首行 | 自己在线由自己的心跳维护，不靠"收到自己的消息"续租 |
| `room_check` 不参与续租 | 两端 `onMessage` 的续租调用处 | 它是"进房前探房"消息，发送方还没决定座位，`sender` 只是脚手架值 |

**验收**：空房 + 新人进房 → 黑座必须仍为空 → 新人作为对弈者加入。

### 9.6 回归验证

| 探针 | 新增段 | 结果 |
|------|--------|------|
| 网页 `probe-web.js` | R1–R6（身份面板）+ N0–N10（座位租约）+ P1–P3b（**P0 复现**） | **116/116** |
| 小程序 `probe-index-page.js` | M1–M8e（座位租约）+ P1–P3b（**P0 复现**） | **153/153** |

反证（把两处修复退回）：
- web 修复 → **R1/R2/R4/R6 四项 FAIL**
- P0 修复（两端）→ **P1/P1b/P2/P2b/P3 五项 FAIL**，且复现出 `seat_state{locked:["black"]}`
  —— 精确还原用户现象，证明断言确实守住该 bug、非恒真。
- v1.3.8 修复（两端）→ **N6/N6b（web）、M6b/M6b2（MP）四项 FAIL**，
  复现出 `mode=recv-takeover shown=true` —— 精确还原"第三人观战导致双方弹窗"。

### 9.7 v1.3.8：观战者误触发接手确认面板

**用户实测**：

> 「1.3.7实测可以对局了，但是第三人加入观战后，对局双方都会收到一个有人想接手的提示。」

**根因**：观战者确认身份后**也会**广播 `request_sync` —— 这是它的合法用途
（取当前棋局用于渲染棋盘），`request_sync` 本就在 `SPECTATOR_ALLOWED_SEND` 白名单里。
而 v1.3.6 的残局接手守卫只判两件事：`iAmPlayer`（我是对弈者）+ `!reqIsMyOpponent`
（请求方不是我对手）。观战者的 `sender='spectator'` 恒 ≠ `oppColor()`：

| 条件 | 值 | 结论 |
|------|-----|------|
| `iAmPlayer` | `true`（我是对弈者） | ✅ |
| `hasHistory` | `true`（已走 N 步） | ✅ |
| `!reqIsMyOpponent` | `true`（`'spectator'` ≠ `'black'`） | ✅ |
| → 守卫成立 | | ❌ **误弹面板** |

两个对局者各收一条观战者的 `request_sync`，各自弹一次 —— 即用户看到的"双方都收到提示"。

> **关键认识**：v1.3.6 的守卫用"**不是我的对手**"来近似"**是来顶替掉线方的第三人**"，
> 这个近似在引入观战者后失效了 —— 观战者同样满足"不是我的对手"。
> **否定式判据（≠）永远要先确认正域（是什么），再加排除项。**

**修复**（接收端加前提，两端对称）：

```js
var requesterIsPlayer = requesterSeat === 'white' || requesterSeat === 'black';
if (iAmPlayer && hasHistory && requesterIsPlayer && !reqIsMyOpponent && needIdentityPick !== true) {
```

连带修：`request_sync` 的身份写入也不再接受观战者的 `identity`
（`if (!isSpectator && data.identity && (data.sender === 'white' || data.sender === 'black'))`），
否则观战者名字会被误写进对局者的对手身份栏。

**⚠️ 遗留议题：接手确认面板在修完 v1.3.8 后已无正常触发路径。**
协议上只有两个对弈座位，`requesterIsPlayer && !reqIsMyOpponent` 意味着请求方只能是
`oppColor()` —— 而那正好是"对手本人"、走 N9 老路径。换言之：
**`recv-takeover` 面板当前是死代码**。它的原始意图（"新人顶替掉线方"）在协议层面
**根本区分不出来**：`oppColor()` 座位的请求既可能是"对手重连回来了"，也可能是
"新人抢占了空位"，二者 `sender` 完全相同。
真正的解决方案是**座位身份令牌**（座位租约里带上会话令牌，重连者证明自己是原主），
属于 v1.4.0 范畴。当前保留面板代码但不清除，等该议题定案后一并处理。

> ✅ **v1.4.0 已解决**（2026-09-12）：上述判断经穷举证明坐实
> （`.ci-secrets/verify-guard-constant.js`，该条件**恒假**），
> 并已改用**座位令牌比对**替换，面板恢复有效触发。
> 完整机制、边界与验证见 **§十一**。

### 9.8 回归验证补充

`N6b` / `M6b2` 是本次新增的**反向断言**：修复后观战者的 `request_sync` 不仅要"不弹面板"，
还**必须**正常回一份 `sync` —— 观战者的棋盘就是靠它渲染的。
只测"不弹面板"会漏掉"顺手把观战者的棋盘来源一起掐了"这种过度修复。

---

## 十一、座位令牌与换边（v1.4.0 新增，**仅小程序端**）

> **范围**：本节只描述**小程序端**。按 2026-09-12 决策，网页版**冻结于 v1.3.8 不再更新**
> （详见 `SEAT_TOKEN_DESIGN.md` §零 R3/R4 与 §6.1 下线检查单）。
> 因此 §十 的"两边必须同步改"约定在本节**不适用**，但**兼容性约束反之更强**（见 11.4）。

### 11.1 解决什么：9.7 的"遗留议题"

§9.7 末尾明确指出 `recv-takeover` 面板已是死代码，并判定
**v1.3.6 的守卫 `requesterIsPlayer && !reqIsMyOpponent` 是恒假的**。
本次把该结论坐实并修复。

**穷举证明**（可执行证据：`.ci-secrets/verify-guard-constant.js`，6 passed / 0 failed）：

| 请求方座位 | 我的座位 | `reqIsMyOpponent` | `!reqIsMyOpponent` | 面板 |
|---|---|---|---|---|
| 我的座位 | 白/黑 | false | true | 触发（**此场景不存在**，座位被我自己占着） |
| **空出的对手位** | 白/黑 | **true** | **false** | **不触发** |

「抢空位」场景中请求方坐的**必然**是空出来的那个对手位 → 恒为第 2 行
→ **面板恒不触发** → 宽限期（60s）过后新人进房会**静默继承残局**。

**行为证据**：`.ci-secrets/repro-silent-takeover.js`（黑座）、
`.ci-secrets/repro-white-seat.js`（白座，更隐蔽：新人抢白座时对方连面板分支都进不去）。
**修复方式**：用「座位令牌比对」替换该死条件。

### 11.2 机制

每个对弈者持有一枚**座位令牌**（`genSeatToken()` 生成，随机+时间戳）：

| 环节 | 行为 |
|---|---|
| 生成 | `confirmRolePicker()` 首次确认身份时生成；已存储有则不重新生成 |
| 存储 | **独立 key** `chessSeatTokens`，**绝不**随 `chessSave` 清除 |
| 广播 | `broadcast()` 随**每一条**消息注入 `seatToken`（与 `clientId` 并列） |
| 记录 | 接收端 `_recordSeatToken()`：首次见到即采信（**TOFU**） |
| 校验 | `request_sync` 到达时比对：一致 = 原主；不一致 = 换了人；**无令牌 = 放行** |

**三态**：`unknown` / `verified` / `foreign`，记录在 `_peerTokenState`。

**剪枝**：只保留最近 3 个房间的记录（`SEAT_TOKEN_ROOM_KEEP`）。
**清除**：仅**显式退房**（`_leaveRoom()`）时清除本房间记录。

### 11.3 为什么 `move`/`sync` 也必须带令牌（不可简化）

凭证获取渠道**不对称**，这是 R2 阶段暴露的关键点：

| 要保护的座位 | 接收方需持有 | 该来源可靠吗 |
|---|---|---|
| 黑座 | 黑方的 `request_sync` | ✅ 白方（房主）此时已在房间 |
| **白座** | 白方的 `request_sync` | ❌ 白方**建房时**就发了，黑方那时**还没订阅**（pubsub 不重放历史） |

→ 黑方对白座的令牌**只能**从 `sync`（白方应答黑方时必发）或 `move`（白方执先，第一步必发）建立。

**结论：`sync` / `move` 必须带令牌。** 若砍掉，白座将永远停在 TOFU 分支，
把新人的令牌误采信为白方的 → **白座保护形同虚设，且是静默失效**。
（反证探针：S4 / S4b / S4c。）

### 11.4 ⚠️ 与网页版共存的降级（重要，须如实记录）

网页版**不认识令牌、也不会补上**。按「缺令牌 = 放行」的降级原则：

| 房间构成 | 座位保护 |
|---|---|
| MP v1.4.0 ↔ MP v1.4.0 | ✅ 完整校验 |
| MP v1.4.0 ↔ **网页版 v1.3.8** | ❌ **无保护**（等同 v1.3.8 现状，**无回归**） |
| MP v1.4.0 ↔ MP v1.3.8 | ❌ 同上 |

**该场景无法用代码改善**：两端 `_clientId` 生成代码**逐字符相同**，
且**共用同一 GoEasy appkey 与频道命名空间** → MP **无法识别**对端端别，
也**无法只封禁网页版**（撤销 appkey 会连小程序一起断）。
→ 唯一解是**线下下线网页版**（检查单见 `SEAT_TOKEN_DESIGN.md` §6.1）。

**另一个必须铭记的坑**：令牌不符时**绝不覆盖**已记录令牌
（`move`/`sync` 分支只标记状态，不调 `_setPeerToken`）。
否则新人只要先走一步棋就能把令牌刷成自己的，随后的 `request_sync` 反而比对成功 —— 守卫被绕过。
（反证探针：S5。）令牌的正式转移**只走**「房主同意接手」这一条路径。

**白座保护强度天然弱于黑座**：黑方换设备/清缓存/重装 → 无令牌 → 白座走 TOFU。
这是"无服务端"的固有代价，需接受并记录，不要写成"完全对称"。
小程序端的高频操作（**切后台再回前台**）需保证令牌不丢（探针 S16 思路 + W 段已覆盖换边路径）。

### 11.5 换边（本次新增功能；R5 已修订语义）

**语义（R5 定稿，按用户原话）**：**自己下第一步之前**可以申请换边 ——
判据是"**申请方自己**走过几步 = 0"，**不是**"双方都零步"。
> 「白方走了第一步，黑方依然可以申请换边，换完后应该同时重开。」

`history` 是半回合序列（白走第 1/3/5…、黑走第 2/4/6…），于是

| 申请方 | "自己 0 步"的等价条件 |
|---|---|
| 白 | `history.length === 0` |
| 黑 | `history.length ≤ 1` ← **含"白方已走 1 步"这一格** |

这正好把用户举的例子包进来：白方 1.e4（n=1）后，黑方自己 0 步 → 可申请。

**换边成功后同时重开**（R5 新增，不可省）。原因：
换边后新白方 = 原黑方，而原黑方在那个局面里本是**后手**；让他执白先行，
盘上却压着旧白方走过的那一步 —— 白先行的不变量被破坏，双方走子数必然失衡
（新白方 0 步 / 新黑方 1 步）。`game.reset()` 清盘后永远是"新白方在空盘上先行"，
于是**任何**换边 + 重开都自洽，守卫也得以收敛到极简。
（0 步时重开是恒等操作，非 0 步时是必要修复 —— 统一重开，逻辑只有一条路径。）

⚠️ **执行顺序不能反**：必须先 `game.reset()` 再改 `myColor`/身份。
`reset()` 不改 `myColor`，而 `myColor` 决定 `buildBoard()` 的棋盘方向；
若先换色再清盘，中间那一瞬是"新颜色 + 旧盘面"，落在两次 `setData` 之间的
渲染/同步会拿到错配数据。

| 项 | 规则 |
|---|---|
| 消息 | `request_swap` / `agree_swap` / `reject_swap`（均带 `target`） |
| 入口可见 | `canSwap` = **我自己 0 步** + 联机 + 对手已进房 + 非终局，由 `updateStatusText()` 下发 |
| 请求方守卫 | `_canSwapNow()`：`_seatMoveCount(myColor) === 0` |
| 接收方守卫 | 按**申请方座位**验算同一条规则（`_seatMoveCount(requesterSeat) === 0`），并额外要求"申请方在重开后的空盘上能落子"（否则整局卡死） |
| 终局 | 不提供换边（重开请走「重开」，语义更清楚） |
| 确认复核 | 面板停留期间盘面可能被对端改动 → 发出前/执行前各复核一次（`_canSwapNow()` / `_swapPromptSteps` 快照） |
| 观战者 | 三类消息纳入 `PLAYER_ONLY_RECV`，观战者不参与 |
| 面板 | `send-swap` / `recv-swap`，复用页内提示机制，文案明确写"会同时重开" |
| 超时 | 复用 `UNDO_REQ_TIMEOUT`（20s），超时自动取消 |
| busy 保护 | `isSwapPending` 计入 `_isBusy()`，防与其他请求叠加 |

**⚠️ R5 修正的原始 bug（务必记住）**：最初的接收侧判据写成"**接收方**自己 0 步"，
于是白方走 1.e4、黑方申请时，白方手里 `n=1` → **误拒**黑方的合法请求 ——
恰好把用户要的那个场景堵死。正确做法是按 `requesterSeat` 算。
（反证见 §11.6 注入 A：改回按接收方算 → W4/W4b/W4c 立刻转 FAIL。）

**★ 关键实现点（易错）**：`executeSwap()` 中令牌**必须随座位（人）转移**：

```
换边前： 我的令牌 T_me @ 旧座位  、  对方令牌 T_opp @ 新座位
换边后： 我的令牌 T_me @ 新座位  、  对方令牌 T_opp @ 旧座位
```

令牌标识的是**人**，座位标识的是**位置**；换边 = 两人交换位置。
**若不交换令牌，双方比对会全部失配** → 下次 `request_sync` 被误判成"换了人"并弹面板。
两侧各自独立计算，因规则相同故结论天然一致（探针 W7/W9/W9c 覆盖）。

**广播必须带新身份名**：`executeSwap()` 末尾 `broadcast(sync, identity: this.myIdentity)`
不能省略 `identity` —— 否则双方身份栏会互相显示成换边前的旧名字（探针 W10c）。

**棋盘方向无需额外处理**：`buildBoard()` 依 `this.myColor` 决定行列顺序，换边后自动翻转
（探针 W8：白方视角首格 `a8` → 换边后 `h1`）。

**座位租约无需搬运**：换边时两个座位**始终都有主**，"谁在线"没变，
双方心跳会立即给各自新座位续租，故 `_seatLastSeen` 保持原样。

### 11.6 回归验证

MP 探针 `probe-index-page.js` 新增 **S 段**（令牌，S1–S11）与 **W 段**（换边，W1–W15）：

- 全量 **238 项 / 通过 238 / 失败 0**（v1.4.0 初版 209 项 → R5 修订后 238 项，**零回归**）
- 另有端到端 `verify-seat-takeover.js` **10/10**、死条件取证 `verify-guard-constant.js` **6/6**

**反证一（令牌，已做）**：把守卫改回 v1.3.8 的 `!reqIsMyOpponent` 后重跑，
**S2/S3/S3b/S4b/S6/S6b 共 6 项转 FAIL**，其中 `S3b` 显示 `sync=1`
—— 即**残局被静默发出**，漏洞重现。证明断言非恒真。

**反证二（换边 R5，新增）**：`verify-swap-r5.js` 把 R5 要修的两个 bug 分别**注回**代码，
再用同一套 W 段探针跑，验证断言确有鉴别力：

| 注入 | 内容 | 转 FAIL 的断言 |
|---|---|---|
| A | 接收侧判据改回"按接收方步数算"（R5 原始 bug） | W4 / W4b / W4c / W15h（4 项） |
| B | `executeSwap()` 删掉 `game.reset()`（换边不重开） | W7f / W7f-b / W7f-c / W9c-c / W11g / W15e / W15f（7 项） |
| C | `_canSwapNow()` 改回"双方都零步"（最初被纠正的口径） | W1c / W3c / W11f / W11g（4 项） |

结果 **15 项 / 通过 15 / 失败 0**；基线（真实代码）在同脚本内先跑一次，确认为 **238/238**。

> 附带发现（记录性）：注入 C 时 `W11e`（弹面板）仍通过 —— 因为 `requestSwap()`
> 自带独立步数守卫，而 `_canSwapNow()` 的复核发生在**确认时**（`_resolvePrompt`
> → `send-swap` 分支）。这是刻意的**两层防线**，即使入口判据被改坏也不会把非法请求发出去。

---

## 十二、观战者名册的快照握手（v1.4.1 新增，**仅小程序端**）

### 12.1 问题（用户实测）

> 「现在退出房间再进房后就看不到观战者了。」

**根因：GoEasy pubsub 不回放历史消息**，而观战者名册此前**只靠一次性事件传播**：

| 环节 | 行为 |
|---|---|
| 观战者进房 | 广播 `spectator_joined` |
| 其他端 | `onMessage` 收到后 `spectators.push(...)` 本地累积 |
| 玩家 `_leaveRoom()` | `this.spectators = []` **清空**（退房语义正确，无法不清） |
| 玩家再进房 | 重新 `subscribe` —— 但此前所有 `spectator_joined` **早已发生完毕**，pubsub 不补发 |

于是名册**永久为空**，且**只能等下一个新人来观战才会重新有内容**。
这不是清理逻辑的 bug，是**传输模型的必然**：事件（event）≠ 状态（state）。

### 12.2 既有解法可复用：座位早已这么做

座位信息**从不依赖一次性事件**，而是 `room_check → room_info/seat_state` 的**握手快照**
（v1.3.6 引入）。观战者名册缺的正是这同一件东西。

> 判据：**任何"需要被后来者知晓"的状态，都必须能被主动拉取，不能只靠广播事件。**

### 12.3 机制（两端协议，当前仅小程序实现）

```
新人进房 → confirmRolePicker() 确认身份
        → 广播 { type: 'spectator_query' }
知情端   → onMessage 收到 query，若 (isSpectator || hasList)
        → 立即回包 { type: 'spectator_list', names, isQueryReply: true }
        （观战者自己也计入自己的快照）
新人    → onMessage 收到 list → _mergeSpectatorNames(names)
```

四个必须遵守的细节：

| 细节 | 理由 |
|---|---|
| `spectator_query` / `spectator_list` 都要进 `SPECTATOR_ALLOWED_SEND` 白名单 | 否则**观战者发不出 query**，问题只解决一半（对弈者能拉，观战者不能） |
| 回包带 `isQueryReply: true` | 区分"应答"与"普通广播"，作为**回环守卫**：应答不再触发新的 query |
| 合并用**并集**（`_mergeSpectatorNames`）而非覆盖 | 应答方可能自己也是刚进房的、快照不全。覆盖会让**晚到的旧快照删掉本地已知观战者** |
| 退房后（`currentRoom` 为空）**不应答** | 已 `unsubscribe`，广播会打到不存在或已换频道的连接上（探针 X8） |
| 观战者合并时跳过自己 | 自己的名字由本地维护，避免重名反复重排 |

### 12.4 已知残留（记录性，非本次修复范围）

- **离线者不在快照里**：真正离线的人无人代答，其名字只在**当时在线者**的快照中。
  与座位租约的 60s 宽限不同，名册**不做超时清理** —— 观战者离开靠 `spectator_left`，
  掉线者会残留，属**既有行为**，本次不引入新语义。
- **旧版本客户端不会应答**：v1.4.0 及更早不认识 `spectator_query`，收到即忽略。
  房间里有旧版本时，新端仍能拿到**新端**的快照（新旧混用不劣化，只是信息不全）。
- **网页版（冻结于 v1.3.8）完全不参与**：它既不回 `spectator_query` 也不懂 `spectator_list`，
  但**会照常发 `spectator_joined`** —— 所以网页版玩家在场时，那部分仍走老路径。

### 12.5 回归验证

- `probe-index-page.js` 新增 **X 段**（X1–X9g），全量 **267 项 / 通过 267 / 失败 0**
- `verify-spectator-roster.js` **21 项 / 通过 21 / 失败 0**（含基线对照）

**反向验证（5 组注入，逐项注回并确认断言转 FAIL）**：

| 注入 | 内容 | 触发的断言 |
|---|---|---|
| A | 删掉 `confirmRolePicker()` 里的 `_querySpectators()` 调用 | I4b / I4e / X1 / X1-1 / X9e |
| B | 快照不包含观战者自己 | X5 / X6 |
| C | 合并改回**覆盖**（非并集） | X9c 相关项 |
| D | 去掉 `isQueryReply` 标记 | X9f 相关项 |
| E | `spectator_query` 移出白名单 | X7 / X9b |

> 附带发现（记录性）：注入 A 的锚点最初写错（注释缩进差 4 空格），
> 导致注入静默失效、断言全绿 —— **这正是反向验证要防的假阳性**：
> 注入不生效时"看起来通过"和"修复正确"无法区分，故每次注入后
> `mutate()` 必须确认文件**确实被改写**。

---

## 十三、入口按钮不再承诺颜色（v1.4.2，**小程序端独有分叉**）

### 13.1 问题（用户提出）

> 「既然现在已经有换边功能了，那么建房和加房的按钮上就不要写执黑执白了。」

用户是对的。原文案：

| 按钮 | 文案 | 实际情况 |
|---|---|---|
| 新建房间 | `新建房间（执白）` | `createRoom()` 预设 `myColor='white'` —— **脚手架值** |
| 加入房间 | `加入房间（执黑）` | `joinRoom()` 预设 `myColor='black'` —— **脚手架值** |

两个 `myColor` 赋值**都不是最终座位**：

- `createRoom` 的 `'white'` 只是进房前的初始值；
- `joinRoom` 的 `'black'` 明确注释为「先按黑方进房，房满检查结束后再按房间情况修正」——
  真实座位由 `doJoinRoom()` 在 `ROOM_CHECK_MS` 后按房况判定（`hasWhite → black`、
  `hasBlack → white`、全空 → `black`）；
- 而且 **v1.4.0 起双方进房后都能申请换边**（`request_swap`）。

→ 所以「执白/执黑」是**错误承诺**：新人照着「执黑」进来，完全可能执白。

### 13.2 修法

只改文案，**不动任何逻辑**：

```
新建房间（执白）  →  新建房间
加入房间（执黑）  →  加入房间
```

`myColor` 的脚手架赋值**保留**（协议需要它带 `sender`），但在 WXML 里加了防回退注释说明缘由。

### 13.3 为什么网页版**不改**（有意分叉，非遗漏）

| 判据 | 网页版 |
|---|---|
| 是否有换边功能 | ❌ **没有**（`request_swap` 出现 0 次） |
| 建房/加房的颜色是否可变 | ❌ 不可变 |
| 结论 | 其「执白/执黑」**仍然准确**，属正确文档 |

且网页版已冻结于 v1.3.8（§十）。**这是两端唯一一处文案有意分叉**，已记入 §二 表格。

> 换个说法：文案要不要写颜色，取决于**颜色在进房后还能不能变**。
> 小程序能变 → 不能承诺；网页版不变 → 承诺成立。

### 13.4 回归验证

`probe-index-page.js` 新增 **Y 段**（Y1–Y11，全量 **278/278**）：

| 断言 | 内容 |
|---|---|
| Y1 / Y2 | 能从 WXML 定位到两个按钮（防止正则失配导致后续断言空跑） |
| Y3 / Y4 | 建房按钮不含"执白" / 加房按钮不含"执黑" |
| Y5 | 两按钮均不含任何颜色词（执白/执黑/白方/黑方/白棋/黑棋） |
| Y6 / Y7 | 动作词「新建房间」「加入房间」仍在（防止改过头把按钮改空） |
| Y8 | `joinRoom` 的 `myColor='black'` 仍标注为脚手架（保留逻辑、只去文案） |
| Y9 | `doJoinRoom` 确实会覆盖 `myColor`（真实座位由房况判定） |
| Y10 ★ | **剥掉 WXML 注释后**全文无颜色词（比 Y3–Y5 更严，覆盖整个页面） |
| Y11 | 版本号 ≥ 1.4.2 |

**反向验证** `verify-entry-label.js`（**19/19**，2 组注入）：

| 注入 | 内容 | 转 FAIL 的断言 |
|---|---|---|
| A | 建房按钮改回「新建房间（执白）」 | Y3 / Y5 / Y10（3 项） |
| B | 加房按钮改回「加入房间（执黑）」 | Y4 / Y5 / Y10（3 项） |

注入时 Y1/Y6（定位、动作词）**仍 PASS** —— 证明断言各司其职、不误伤。

> **本次新增探针基建**：探针此前只能通过 `argv[2]` 替换 `index.js`，
> WXML/WXSS 是**写死路径**读的，导致"纯文案断言"**无法反向验证**。
> 现新增环境变量 `PROBE_PROJ_OVERRIDE`（指向一个只含注入版静态文件的假项目目录），
> 使静态资源也可注入；`utils/chess.js` 的重定向**故意不受它影响**
> —— 只换被测的那一个文件，引擎始终用真身，结论才聚焦。

---

## 十四、重进房间要认得自己的座位（v1.4.3，**小程序端独有**）

### 14.1 问题（用户实测）

> 「如果建好房后获取到房间号，然后点重新进入小程序回到首页，再用刚才的房间号进房，居然会变成黑方。」

复现路径：

```
建房（无人在场）→ 我是白方，拿到房间号 XXXX
→ 重进小程序 → 回首页
→ 输入 XXXX 点「加入房间」
→ 期望：仍是白方（这房本来就是我建的）
   实际：变黑方
```

### 14.2 根因

`doJoinRoom()` 的座位判定**只看房内有没有人，从不问「我是谁」**：

```js
if (hasWhite && hasBlack)      → 观战
else if (hasWhite)             → black
else if (hasBlack)             → white
else                           → black   // ← 空房一律黑方
```

而「我原是这个房的白方」这条信息 **本来就在本地** —— 座位令牌
（独立存储 key `chessSeatTokens`，只有**显式退房**才清除）。
问题在于 `joinRoom()` **从不读它**，只有 `resumeGame()` 会调 `_loadSeatTokens()`。

这又是一次 **§12 同一类病的变体**：信息存在，但**没有读取路径**。

### 14.3 为什么必须用座位令牌，不能用 `chessSave`

| 存储 | 能否用于认座 | 原因 |
|---|---|---|
| `chessSave` | ❌ **不行** | `joinRoom()` 会 `removeStorageSync('chessSave')` —— 它的语义是「上一次未结束的对局」，进新房就该清。实测确认：`joinRoom` 之后 `chessSave` 必为空 |
| `chessSeatTokens` | ✅ 可以 | 独立 key，**只有显式退房**才清；语义是「这座位是我占的」凭据，跨重进存活 |

> 这就是为什么 v1.4.0 把令牌放**独立 key**是对的 —— 当时是为了令牌比对，
> 现在它成了「认座」的唯一可用依据。

### 14.4 机制

三处配合，缺一不可：

| # | 改动 | 作用 |
|---|---|---|
| ① | 新增 `_myTokenSeatIn(roomId)` | 读令牌，回答「我曾坐哪个座位」 |
| ② | `joinRoom()` 进房前 `_loadSeatTokens()`，脚手架 `myColor` 改为 `原座 \|\| 'black'` | 让 3 秒检查期内的**心跳就续租在正确座位** —— 否则房主重进后前 3 秒会以黑座身份续租自己的白座，把座位搅乱 |
| ③ | `doJoinRoom()` 新增 `oldSeatFree` 优先 | 原座还空着 → 坐回去 |

③ 的完整判定表（**每种情况都必须明确**）：

| 房内状态 | 我有令牌 | 结果 | 理由 |
|---|---|---|---|
| 两座都有人 | 任意 | 观战 | 房满判定优先，认座不得覆盖 |
| 原座空着 | ✅ | **坐回原座** | ★ 用户报的场景：房主重进保住白方 |
| 原座被占、另一座空 | ✅ | 坐另一座 | 房内现况优先，**不硬抢** |
| 空房（两座都空） | ❌ | 黑方 | 真·新人加入，**旧行为不变** |

> ⚠️ `myColor` 的脚手架值**不再是写死的 `'black'`**（v1.4.2 的 Y8 断言因此更新）——
> 改为 `this._myTokenSeatIn(roomId) || 'black'`。
> 但它**仍是脚手架**：真实座位一律由 `doJoinRoom()` 在 3 秒后判定。

### 14.5 附带收益

分享卡片自动进房（`_tryAutoJoin()` → `joinRoom()`）走**同一路径**，
所以「从分享卡片重进同一个房间」也一并保住了原座。

### 14.6 回归验证

`probe-index-page.js` 新增 **Z 段**（Z1–Z9，共 16 项断言），全量 **294/294**：

| 断言 | 内容 |
|---|---|
| Z1 ★ | 空房 + 白座令牌 → 仍执白（**用户报的场景**） |
| Z2 | 空房 + 黑座令牌 → 仍执黑（对称，防「修成只会给白」） |
| Z3 ★ | 真·新人（无令牌）进空房 → 默认黑方（旧行为不变） |
| Z4 ★ | 原座已被占 → 不硬抢，退到另一空座 |
| Z5 ★ | 房满 → 仍转观战（认座不得覆盖房满判定） |
| Z6 / Z6b | 显式退房 → 令牌被清（不会记住已放弃的房） |
| Z7 / Z7b / Z7c | `joinRoom` 调 `_loadSeatTokens`；取值来自 `_myTokenSeatIn`；**不来自 `chessSave`** |
| Z8 / Z8b / Z8c / Z8d / Z8e | `doJoinRoom` 有 `myOldSeat` / `oldSeatFree` 判据；原座空时取 `myOldSeat`；无令牌仍回退黑方；**未用 `chessSave`** |
| Z9 | 版本 ≥ 1.4.3 |

**独立复现脚本** `.ci-secrets/repro-rejoin-seat.js`（13 项）：
修复前第 ⑨ 项 FAIL（`myColor=black`），并逐项证明
「信息就在本地令牌里，只是没人读它」「`chessSave` 被 `joinRoom` 清掉，故不可用」。

**反向验证** `.ci-secrets/verify-rejoin-seat.js`（**27/27**，4 组注入）：

| 注入 | 内容 | 转 FAIL |
|---|---|---|
| A | 去掉「原座优先」整段（= 恢复用户报的 bug） | Z1 / Z8c |
| B | `joinRoom` 不恢复令牌（认座入口断掉） | Z7 |
| C | 改用 `chessSave` 认座（依据选错） | Z7b |
| D | `oldSeatFree` 恒真（会硬抢已被占的座位） | Z4 |

> 注入 A 时 Z8b 仍 PASS、注入 C 时 Z8 仍 PASS —— 这是**正确粒度**：
> Z8b 只查变量名是否存在，Z8 只查 `doJoinRoom` 是否用令牌；
> 它们不负责守「值取得对不对」（那是 Z8c / Z7b 的职责）。
> 断言各司其职、不越界，才便于定位问题。

> ⚠️ **写 Z 段时踩的坑（记录性）**：用固定长度切片取 `doJoinRoom` 函数体
> （`slice(start, start + 3400)`）会**切进下一个函数** `resumeGame()`，
> 那里有 `getStorageSync('chessSave')` → Z8e 误报「doJoinRoom 用了 chessSave」。
> 修法：切片终点改为**下一个函数名**（`src.indexOf('fallbackLocal() {')`）。
> **教训：从大文件里"切一个函数"必须用函数边界，不能用字节长度。**

---

## 十五、手输房间号 == 存档房间号 → 等同「恢复刚才断线的对局」（v1.4.4，**小程序端独有**）

### 15.1 问题（用户需求）

> 「可以做成进的房间号跟恢复刚才掉线的对局按钮里记录的房间号一致就跟点恢复掉线按钮效果一致吗」

即：首页输入的房间号若与「恢复刚才断线的对局」按钮记录的**那一局**相同，
应当**直接走恢复逻辑**（还原棋谱、保住身份与座位），而不是当成「新加入一个房间」重新开局。

### 15.2 修复

`joinRoom()` 开头新增恢复捷径（**早于**一切「新加入」副作用）：

```js
const saved = this._readResumableSave()
if (saved && saved.roomId === roomId) {
  this.resumeGame(saved)
  return                       // ★ 随即返回，不再落入"选身份 → 清存档"流程
}
```

`_readResumableSave()` 的三条判据（**缺一不可**）：

| # | 判据 | 理由 |
|---|---|---|
| ① | 有存档且能解析 | 解析失败一律返回 `null`（**不抛异常**，进房流程不能被坏存档炸掉） |
| ② | `pgn` 非空 | ★ **最关键**：房间号只有 4 位、会被重复使用。只看「号相同」就恢复，新开的空房会吃进上一个房的旧棋谱 —— 有走子才算「未结束的对局」 |
| ③ | `color` 是 `white`\|`black` | 观战者没有「自己的对局」可恢复 |

> ⚠️ **不检查「是不是原建房方」**：对手手输同一房间号回来，同样该恢复残局。
> 与 §14 的认座是**两个独立问题**（14 解决「我坐哪」，15 解决「盘中内容从哪来」）。

### 15.3 顺带修掉的两个既有缺陷

这次改动牵出两个**早已存在**的问题，一并修复：

| # | 缺陷 | 后果 | 修法 |
|---|---|---|---|
| ① | `saveGameToLocal()` 只在「选身份后」被调用过一次，**对局中再没更新** | 存档永远停在**开局空盘** → 用户按「恢复刚才断线的对局」回来发现棋局没恢复（感觉像被重置）。**这是该按钮此前形同虚设的根因** | `handleMoveAftermath()` 每步落盘 |
| ② | `resumeGame()` 里 `'断线重连成功'` 的 `roomTip` **设在 `showGameUI()` 之前** | 后者内部会重写 `roomTip` → 这条提示**从未显示过** | 调换顺序，文案改为 `'已恢复未结束的对局'` |

① 的关键在于**选对了落盘点**：

> `handleMoveAftermath(move, shouldBroadcast)` 是**双方落子的唯一汇合点** ——
> 自己走子（`onSquareTap` / `confirmPromotion` → `handleMoveAftermath(move, true)`）
> 与收到对方 `move`（`onMessage` → `handleMoveAftermath(data.moveInfo, false)`）
> 都会走到这。故在此保存可**覆盖全部走子来源**，不需要在多个入口分别补。

落盘放在 `broadcast()` **之后**：先广播再落盘，存档不会停在「半更新」状态。

### 15.4 恢复路径的补强

`resumeGame(d)` 新增两项：

| # | 改动 | 理由 |
|---|---|---|
| ① | 新增可选参数 `d`（调用方传入已解析的存档） | 避免读两次，且防「两次读之间存档被改」的时序问题。省略时仍自行读，**旧调用方式不变** |
| ② | 恢复后立刻 `ping` 续租 + `_startHeartbeat()` | 宣告「我在这个座位上」。否则对手侧仍以为我掉线，60s 宽限期过后可能把座位放出去。`ping` 同时带上令牌 → 对手知道我是**原主重连**，不会弹接手面板 |

### 15.5 回归验证

`probe-index-page.js` 新增 **AA 段**（AA1–AA10，共 17 项断言），全量 **311/311**：

| 断言 | 内容 |
|---|---|
| AA1 ★ / AA1b / AA1c | 手输同号 → 棋谱还原；不走选身份；不排 3s 房况检查定时器 |
| AA2 ★ / AA2b | **号不同 → 绝不恢复**（新加入，棋盘为空）；旧存档按既有语义清除 |
| AA3 ★ | 号相同但存档**无走子** → 不恢复（防旧存档污染新局，守判据②） |
| AA4 ★ | 观战存档 → 不恢复（守判据③） |
| AA5 ★ | 存档损坏 → 不抛异常，降级为「新加入」 |
| AA6 ★ | 黑方手输同号 → 同样恢复（**不限定原建房方**） |
| AA7 / AA7b / AA7c | 比对在 `joinRoom` 内；**早于** `removeStorageSync('chessSave')`（否则自我清除）；恢复分支随即 `return` |
| AA8 / AA8b | 判据含「有走子」；判据含颜色限制 |
| AA9 / AA9b | `handleMoveAftermath` 每步落盘；落盘在广播**之后** |
| AA10 | 版本 = 1.4.4 |

**反向验证** `.ci-secrets/verify-resume-by-room.js`（**50/50**，5 组注入）：

| 注入 | 内容 | 转 FAIL | 仍 PASS（正确粒度） |
|---|---|---|---|
| A | 去掉恢复捷径整段（= 恢复用户报的行为） | AA1 / AA1b / AA1c / AA6 / AA7 / AA7b / AA7c | AA2 / AA3 / AA4 / AA5 / AA9 |
| B | 判据去掉「有走子」 | AA3 / AA8 | AA1 / AA4 / AA8b / AA9 |
| C | 判据去掉颜色限制 | AA4 / AA8b | AA1 / AA3 / AA8 |
| D | 比对挪到清存档**之后** | AA1 / AA6 / AA7b | AA2 / AA3 / AA5 / **AA7 / AA7c** |
| E | 去掉每步落盘 | AA9 / AA9b | AA1 / AA8 |

> 注入 D 是最能说明**断言粒度**的一组：AA7（两项都在 `joinRoom` 内）与 AA7c
> （恢复分支仍带 `return`）在挪位后**都继续 PASS**，只有专守「顺序」的 AA7b 能识破。
> 若没有 AA7b，这个 bug（恢复永远不生效）将无人把守。

> ⚠️⚠️ **写反向验证时踩的坑（重要，已固化为脚本内的强制检查）**：
> `String.prototype.replace` **只替换第一处**，锚点不唯一时会**静默打到别的函数**里。
> 本脚本首版连续踩了两次：
> - 锚点 `wx.removeStorageSync('chessSave')` 全文件有 **3 处** → 注入 D 打进了 `_leaveRoom()`
> - 改用 `game.reset() + 清存档` 两行组合后仍有 **2 处**（`createRoom` 与 `joinRoom` 各一）→ 再次打偏
>
> 两次的表象都一样：**「注入确实写入文件」通过**（文件里确实出现了注入串），
> 但注入打在了错误位置 → 断言表现与预期不符。
> 修法：注入器 `injectWith()` 新增 `anchor` 参数，**强制校验锚点全文件唯一**
> （`SRC.split(anchor).length - 1 !== 1` 即判错并中止该次注入）；
> 注入 D 的锚点改为 `joinRoom` 独有的三行（含 `this.currentRoom = roomId`）。
> **教训：反向验证脚本自身也需要"判据"——"文件被改过"不等于"改对了地方"。**

---

## 十六、bindtap 传事件对象（第二次）+ 存档防污染 + 删除「本地同屏对局」（v1.4.5，**小程序端独有**）

### 16.1 用户反馈

v1.4.4 的两处修复**都没生效**，实测两条现象：

1. 点「🔄 恢复刚才断线的对局」进去后，**房间号显示成「本地」**
2. 手输与存档一致的房间号进房，**仍走"检查房间 → 选身份"**（即 §15 的恢复捷径没触发）

### 16.2 根因（一个 bug 引发两条现象）

v1.4.4 给 `resumeGame(d)` **加了参数**，却忘了 WXML 里一直是 `bindtap="resumeGame"` ——
**小程序会把点击事件对象作为第一个实参传进来**（不是 `undefined`）。而守卫写的是 truthy 判定：

```js
resumeGame(d) {
  if (!d) {            // ← 事件对象是 truthy → 整段被跳过
    const savedStr = wx.getStorageSync('chessSave')
    if (!savedStr) return
    d = JSON.parse(savedStr)
  }
  this.currentRoom = d.roomId || ''   // ← 事件对象没有 roomId → ''
  ...
  this.showGameUI(this.currentRoom || '本地')   // ← 显示「本地」
}
```

→ 现象①：`showGameUI('')` 把空串当房间号渲染，且沿用了「本地」兜底字面量，棋盘为空。

**存档污染**（现象②的真正原因）：在"本地"空盘上随手走一步，
`handleMoveAftermath → saveGameToLocal()` 就以 `currentRoom = ''` 落盘，
**把真实 roomId 覆盖掉** → §15 的匹配条件 `saved.roomId === roomId` **恒不成立**
→ 恢复捷径永不触发。

> ⚠️ 这是本项目**第二次**踩「bindtap 会传事件对象」。
> 第一次是 `joinRoom`（当时靠 `typeof d === 'string' || 'number'` 判定挡住）。同一条坑
> 已写在 skill 里，但 v1.4.4 给 `resumeGame` 加参数时没有回查 —— 教训已补进 skill。

### 16.3 修复（4 条）

| # | 位置 | 改动 | 守什么 |
|---|---|---|---|
| ① | `resumeGame(d)` | 守卫改为 `if (!d \|\| typeof d !== 'object' \|\| Array.isArray(d) \|\| !d.roomId)` | 用「有没有 `roomId`」判定是不是存档 —— 事件对象没有 `roomId`，一眼可分；顺带挡住数组 |
| ② | `saveGameToLocal()` | `if (!this.currentRoom) return` | **防污染**：`roomId` 是恢复的唯一钥匙，落一个没钥匙的存档只会覆盖掉有价值的旧存档 |
| ③ | `checkSavedGame()` | 改用 `_readResumableSave()` **同一判据** | 按钮**可见性 = 可恢复性**（原先只看"存档存不存在"，会出现"按钮出来了、点了没反应"） |
| ④ | `onSquareTap()` | 守卫重构：**先无条件要求房间**，再判网络/对手 | 原先 `if (isNetworkReady && currentRoom) { … }` 在无房间时**整个守卫被跳过**，棋子照样能走 |

第 ④ 条是**顺带挖出的结构性漏洞**（K3 用例暴露的）：
`currentRoom` 为空时整块守卫被跳过 —— 那是给「本地同屏对局」留的逃生口。
功能删除后，这个口子成了"看着像在对局、其实没有房间"的幽灵棋局入口。
**顺序不可颠倒**：没有房间时，"有没有连上 / 对手在不在"都无从谈起。

### 16.4 删除「本地同屏对局」（用户要求）

删除范围（8 处）：`fallbackLocal()` 定义、`showGameUI('本地')` 调用兜底、
`showGameUI` 内 `roomTip` 的 `'本地'` 三元、分享钩子 `onShareAppMessage` /
`onShareTimeline` 的 `'本地'` 判断、连接失败（2 处）与实例创建失败的退化调用、
WXML 的同屏文案。

删它的三个理由：

1. 语义上没有"断线重连"可言，却调 `saveGameToLocal()` 往 `chessSave` 落了一个
   **roomId 为空**的存档 —— 正是本次污染的温床之一；
2. 它把**真实网络故障伪装成"能玩"**（用户以为在对战，其实双方从未连上）；
3. 连接失败时的正确行为是**明确失败并提示**，不是静默降级到另一种玩法。

`fallbackLocal()` 原址保留一行**占位注释**记录删除决策，防止以后有人"顺手加回来"。

### 16.5 回归验证

- 探针 `probe-index-page.js`：**338 项 / 通过 338 / 失败 0**（AB 段新增 26 项）
- 常驻回归 `verify-resume-fix-r6.js`：**29/29**（A–G 七段，自包含 wx/GoEasy 桩）
- 反向验证 `verify-resume-fix-r6-reverse.js`：**64/64**（10 组注入，见下表）

| 注入 | 内容 | 转 FAIL | 仍 PASS（正确粒度） |
|---|---|---|---|
| A | 入参守卫退回 `if (!d)`（= 重演本次故障） | AB1 / AB1b / **AB1c** / AB3 / AB6-1 / AB6-2 | AB2 / AB2b / AB4 / AB5 / AB7 |
| B | 去掉"空 roomId 不落盘"防线 | AB4 / AB4b | AB5 / AB1 / AB7 |
| C | `checkSavedGame` 退回只查"存在" | AB7b | AB7 / AB7c / AB1 |
| D | 落子守卫退回 `if (isNetworkReady && currentRoom)` | **K3** | AB1 / AB4 / AB7 |
| E | 把 `fallbackLocal()` 定义加回来 | AB8 | AB1 / AB4 / AB7 |
| H | 恢复 `showGameUI(currentRoom \|\| '本地')` 兜底 | AB8b | AB1 / AB7 / AB8 |
| I | 恢复 `roomTip` 的 `'本地'` 三元 | AB8c | AB1 / AB7 / AB8 |
| G | 失败分支退化回 `fallbackLocal()` | AB8d | AB1 / AB7 / AB8 |
| F | WXML 文案改回"仅支持同屏单机对战" | AB8e | AB8（源码断言不受静态文案影响） |

> **E / H / I / G 是四个独立的删除点**，必须各自注入验证 —— 只还原文案或只还原调用，
> 打不动"函数定义不存在"那条断言，反之亦然。**一个"功能已删除"的事实，
> 需要多条针对不同删除面的断言共同把守**，否则"删干净了"只是自我感觉。

### 16.6 ⚠️ 本轮反向验证抓出的 3 个**假鉴别力**（重要）

反向验证的价值就在于此 —— 三处断言**基线全绿、注入却打不动**，
即"看着在保护代码，其实没有"：

| 断言 | 病症 | 修法 |
|---|---|---|
| **K3** | 用例没设 `isNetworkReady = true`，而它默认为 `false`（`onLoad` 置）→ 落子被**网络检查**拦住，不是被**房间守卫**拦住 → 注入 D 下仍 PASS | 显式 `p.isNetworkReady = true`，让唯一能拦住落子的只剩房间守卫 |
| **AB1c** | 判据是 `roomId !== '本地'`，而缺陷态 `roomId` 是**空串**，`'' !== '本地'` 为真 → 恒 PASS | 改判 `roomId === room`（真实房间号）且非空 |
| **注入 D（脚本侧）** | 首版写成 `if (!isNetworkReady \|\| !currentRoom) return` —— **这本身就是一条无房间守卫**，注入等于没注入 | 注回 v1.4.4 的**真实形态**：`if (isNetworkReady && currentRoom) { [整段守卫] }` |

> **教训（已补进 skill）**：
> ① 反向验证不只校验"断言有没有鉴别力"，**也校验"注入是不是真把能力注回去了"** ——
> 两条都得对，缺任一条结论都不可采信。
> ② 写"退回旧写法"的注入时，必须**逐字对照 git 历史里的旧代码**，
> 不能凭印象写一个"语义等价"的版本 —— 语义等价的版本往往恰好**仍然是修复后的行为**。
> ③ 反向验证脚本首版若有注入写错，**失败项会指向"断言不行"的假象**：
> 报错信息说的是 AB1c / K3 / AB8c 转 FAIL 失败，实际原因却在注入侧。
> 排查顺序应是**先怀疑注入，再怀疑断言**。

---

## 十七、恢复按钮消失 + 认座成功却仍要重选身份（v1.4.6，**小程序端独有**）

> 上一版 §16 刚修完「bindtap 传事件对象」，v1.4.5 上传后用户实测又报两条现象。
> 本轮两条现象**共用一个根因** —— 判据（criterion）混用。

### 17.1 用户实测（2026-09-13 09:1x）

| # | 现象 |
|---|---|
| ① | 首页「🔄 恢复刚才断线的对局」按钮**不见了** |
| ② | 重进之前房间号，提示「欢迎回来，你仍执白」，**但仍要重新选身份** |

### 17.2 根因：`_readResumableSave()` 的判据「pgn 非空」太严

`chessSave` 在**选完身份**时就落盘（`confirmRolePicker()` → `saveGameToLocal()`），
**那一刻 `pgn` 还是空串**（一子未动）。而 v1.4.4/v1.4.5 的可恢复判据里有一条
`if (!d.pgn) return null` —— 于是"刚建好房、一子未动就退出"的存档被直接判死。

由此分出**两套数据源**，各自判断同一件事（"这还是我的那一局吗"）：

| 问题 | 数据源 | 判据 | 结果 |
|---|---|---|---|
| 认出座位（"欢迎回来，你仍执白"） | `chessSeatTokens`（座位令牌） | 令牌在 → 认座 | ✅ 命中 |
| 进房「等同恢复」 | `chessSave` | roomId 同 **且 pgn 非空** | ❌ 拒绝 |
| 首页按钮可见性 | `chessSave` | `!!_readResumableSave()`（含 pgn） | ❌ 按钮消失 |

→ **"认得你、但不认这局"**：认座成功（症状②的前半句），
却因为空盘走不进恢复分支，落到普通进房流程 → 弹身份选择（症状②的后半句）；
按钮也因同一判据消失（症状①）。

复现证据：`.ci-secrets/diag-v146.js` **场景 I** 逐字复现两条现象 ——
`I1 showResume=false`（症状①）、`I2 needIdentityPick=true / noticeText="欢迎回来，你仍执白"`（症状②）。

### 17.3 修复：判据**分层**（松 / 严各司其职）

核心是承认「pgn 非空」的本意**只是消歧**（4 位房间号会重复，
同一个号可能是**另一局**新棋），它不该被当成"这局值不值得恢复"的总闸门。

```js
// 松：这个存档"可恢复"吗？（只管能不能恢复，不管是不是这一局）
_readResumableSave() {
  if (!d.roomId) return null
  if (d.color !== 'white' && d.color !== 'black') return null   // 观战无从恢复
  return d          // ← v1.4.6 移除了 `if (!d.pgn) return null`
}

// 严：这个存档"是这一局"吗？（消歧，只用在"房间号相同"的歧义场景）
_isSaveForThisRoom(roomId) {
  const d = this._readResumableSave()
  if (!d || String(d.roomId) !== String(roomId)) return false
  const myOldSeat = this._myTokenSeatIn(roomId)
  if (myOldSeat && myOldSeat === d.color) return true   // ★ 令牌佐证：我在此房坐的就是这个色
  return !!d.pgn                                        // 无佐证 → 退回严格判据
}
```

| 改动 | 位置 | 说明 |
|---|---|---|
| 移除 pgn 判据 | `_readResumableSave()` | 空房也"可恢复" → 按钮恢复显示（症状①） |
| 新增消歧函数 | `_isSaveForThisRoom(roomId)` | 令牌佐证优先，pgn 只作兜底 |
| 恢复分支改判据 | `joinRoom()` | 命中即 `resumeGame(_readResumableSave())` 并 `return`（症状②） |
| 清档改为条件式 | `joinRoom()` / `createRoom()` | 见 17.4 |
| 跨房覆盖护栏 | `saveGameToLocal()` | 见 17.4 |
| 退房后同步按钮 | `_leaveRoom()` | 清档后补 `this.checkSavedGame()` |

### 17.4 顺带修掉的一个更隐蔽缺陷：跨房存档被**无条件删除**

`joinRoom()` / `createRoom()` 里原本各有一句**无条件** `wx.removeStorageSync('chessSave')`。
后果：用户手输**另一个房间号**（哪怕只输错一位）→ **上一局残局当场被删**，
而那个房间的座位令牌还在 → 下次回去"欢迎回来"，却一子不剩。
（`.ci-secrets/diag-v146.js` **场景 J** 实测：进 9009 后原 7101 存档被清成空盘。）

改法：
- `joinRoom()` 只在**存档属于本房**时清（`String(oldSave.roomId) === String(roomId)`）；
- `createRoom()` 不清（新房间的存档属于别房，那是别人的数据）；
- `saveGameToLocal()` 加护栏：**本房空盘不得覆盖别房的真残局**。

### 17.5 用户决策（2026-09-13，两条）

| 问题 | 决策 | 依据 |
|---|---|---|
| 「刚开好、还没走子的空房」算不算可恢复的对局？ | **算** → 按钮照常显示 | 那是用户**自己的**局；恢复出来是空盘，无害 |
| 手输「另一个房间号」时，当前房间的存档怎么处理？ | **保留，不删** | 输入错误不该毁掉一局没下完的棋 |

### 17.6 验证

- 主探针 `probe-index-page.js`：**346/346**（新增 AC1/AC2/AC3/AC4）
- 常驻回归 `verify-resume-fix-r6.js`：**35/35**（新增 H1–H5 段）
- 反向验证 `verify-v146-reverse.js`：**39/39**（6 组注入）
- 反向验证 `verify-resume-fix-r6-reverse.js`：**65/65**（语义变更后重指鉴别点）
- 其余 7 个回归脚本全绿（resume-by-room 52 / rejoin-seat 27 / entry-label 19 /
  repro-rejoin-seat 13 / spectator-roster 21 / swap-r5 15 / seat-takeover 10 / guard-constant 6）

| 注入 | 内容 | 转 FAIL | 仍 PASS（正确粒度） |
|---|---|---|---|
| A | 把 `pgn 非空` 判据塞回 `_readResumableSave` | AB7b | AB7 / AB7d / AA1 |
| B | 消歧判据去掉「令牌佐证」 | AA3b | AA1 / AA3 / AA4 |
| C | 消歧判据恒真（只看房间号相同） | AA3 | AA1 / AA3b / AA4 |
| D | `joinRoom` 退回"无条件清存档" | AA2b | AA1 / AA3b / AB7 |
| E | `saveGameToLocal` 去掉跨房覆盖护栏 | AC1 | AC2 / AB7 |
| F | `_leaveRoom` 清档后不重算按钮状态 | AC3 | AC4 / AA1 |

### 17.7 ⚠️ 本轮踩的坑：**断言 ID 跨脚本撞车**（已写进 probe AC 段注释）

反向验证脚本 `verify-v146-reverse.js` 首版把注入 E/F 的鉴别点写成 `H4` / `H5`，
**恒 PASS**，一度被误读成"注入锚点失配"。

真实原因：`H4` / `H5` 是 **r6 脚本**里的断言（本房空盘不得覆盖别房残局 / 护栏不误伤），
而 `verify-v146-reverse.js` 跑的是 **probe**（`probe-index-page.js`）——
probe 里的 `H4` / `H5` 早已被另一段占用（**身份面板显示时棋盘冻结 / 面板占用时不叠加**）。
注入打得再准，也只是在动面板断言，跟跨房护栏毫无关系。

**教训**：
1. **跨脚本引用断言 ID 前，必须确认该 ID 在"目标探针"里的归属** ——
   `grep` 一次就能避免整轮误判；同名不同义是本项目最容易踩的坑。
2. 探针内新段一律用**新前缀**（本轮用 `AC`），不再复用已占用的字母段。
3. 定位"注入打不动断言"时，除"注入写没写对"之外，还要加一层前置检查：
   **这条断言到底在不在本次运行的探针输出里**（`aline()` 返回 `(缺失)` 就是信号，
   但若恰好撞上同名 ID，就会伪装成 PASS —— 更隐蔽）。

---

## 十八、对局页控件重排：设置区上移 / 两列按钮 / 换边按钮交互 / 退出房间（v1.4.7–v1.4.8，**小程序端独有**）

> 本轮是**纯 UI 重排 + 为后续功能预留入口**，不改动任何实时协议。
> 网页版 `docs/index.html` 仍冻结 v1.3.8，**不受影响**（本页控件无对应协议字段）。

### 18.1 用户需求（2026-09-13）

| # | 需求 | 落地 |
|---|---|---|
| ① | 路径提示开关放到上面去，要显示「路径提示」四个字 | 上移到棋盘上方，文案由「👁️ 提示」改为**完整四字** |
| ② | 上面增加换肤下拉框、声音开关（为皮肤与声效做准备） | 同处新增的「对局设置」区 |
| ③ | 下面的按钮分两列显示 | 底部控制区 flex+wrap → **grid 两列** |
| ④ | 换边按钮不可用后变灰，而不是消失 | v1.4.7 按此实现；**v1.4.8 用户改为"可点 + 提示原因"**（见 §18.3 的三段演进） |
| ⑤ | 增加退出房间按钮 | 新增按钮，走**页内确认面板** |

### 18.2 为什么把「设置」从「操作」里拆出去

原来的底部控制区一行塞了：路径提示开关、😀 表情、悔棋、重开、换边 ——
**两类语义混在一起**：
- **设置**（路径提示/声音/皮肤）：看棋过程中随时想调，属于"环境"
- **操作**（悔棋/重开/换边/退出）：改变对局状态，属于"动作"

混在一行的后果是双向的：想调设置要在一堆操作按钮里找；而操作按钮旁放个开关，
误触概率也高。现在**设置归棋盘上方一行，操作归棋盘下方两列**，语义分开。

### 18.3 ⚠️ 换边按钮的三段演进：消失 → 灰显 → **可点 + 提示原因**（v1.4.8 定稿）

这一处交互改了三版，值得完整记录，因为**每一版都解决了上一版的真问题**，
而最终结论是"信息量最大"的那个：

| 版本 | 写法 | 问题 |
|---|---|---|
| ≤ v1.4.6 | `wx:if="{{!isSpectator && canSwap}}"` | `canSwap` 为假时按钮**直接消失** → 用户不知道有这个功能，也分不清"没了"和"用不了" |
| v1.4.7 | `disabled="{{!canSwap}}"` + `.btn-off` 灰显 | 保留了"功能存在"的信息，但**说不出"为什么不能用"** → 用户只能猜：步数？对方没进房？网络？ |
| **v1.4.8（定稿）** | **始终可点**，不满足条件时**提示具体原因** | —— 与「悔棋」同一模式 |

最终写法（就是朴素的一个按钮）：

```xml
<button wx:if="{{!isSpectator}}" class="btn btn-info" bindtap="requestSwap">{{swapText}}</button>
```

**为什么最终选"可点 + 提示"**：`requestSwap()` 的前置检查**本来就有 7 条**，
每条都带具体文案 —— 这个结构从 v1.4.0 起就在，只是此前被 `wx:if` / `disabled`
挡在外面、用户永远看不到：

| 不满足的条件 | 点下去得到的提示 |
|---|---|
| 观战者 | 观战中不能换边 |
| 还没选身份 | 请先选择身份 |
| 未联机 / 无房间 | 换边需要联机对手 |
| 本局已终局 | 本局已结束，请直接「重开」 |
| **我自己已走棋** | **你已经走过棋，不能换边** |
| 对方还没进房 | 对方还没进房 |
| 已在申请中 | 已发送换边请求 |

> 对比一下信息量：灰按钮说"不能用"，提示说"**你已经走过棋，不能换边**"。
> 后者才是用户真正需要的那句话 —— 而且这正是本项目「悔棋」一直以来的做法
> （`requestUndo` 不满足条件就提示"已是开局，无棋可悔"、"你还没走棋"）。
> **与其发明新交互，不如对齐项目里已经验证过的那一种。**

### 18.3.1 ⚠️ 实现约束：「灰显」与「给提示」**互斥**，只能二选一

这不是风格问题，是**机制冲突**：

> **小程序里 `disabled` 的 `button` 不触发 `bindtap`。**
> 一旦加上 `disabled`，点击事件被整个吞掉 → 提示逻辑**永远不会执行**。

所以在 v1.4.7 的写法下，"点一下看看为什么不能换边"在物理上做不到 ——
灰显不是"提示的补充"，而是**提示的替代品**。这也是本轮反转的硬约束：
要提示，就**必须**把 `disabled` 拿掉（同时也要拿掉 `.btn-off` 灰样式，
否则会出现"看起来能点、其实是灰的"这种更糟的误导）。

探针为此专门设了两条断言，**分别**守两种失效模式：

| 断言 | 守什么 | 失效后果 |
|---|---|---|
| **AD7** | 按钮 tag 里**不得有 `disabled`** | 点击被吞 → 提示永不执行 |
| **AD7b** | `wx:if` 表达式**只能是 `{{!isSpectator}}`** | 混入 `canSwap` → 按钮又被隐藏 |

> 反向验证里这两条不能合并：注入"退回 `wx:if && canSwap`"只打掉 AD7b；
> 注入"加回 `disabled`"只打掉 AD7。一次注入只暴露一种失效模式 ——
> 若把两条断言写成一个复合判据，就**分辨不出是哪种退化**了。

**保留下来的部分**：`wx:if="{{!isSpectator}}"` ——
**观战者本就没有换边这个能力**，给他一个按钮（无论灰不灰）都会误导
（"我要怎样才能用？"）。判据仍是：
**「能力不存在」→ 隐藏；「能力存在但条件不满足」→ 可点 + 说明原因。**

### 18.4 退出房间：破坏性操作必须"过面板 + 说实话"

`_leaveRoom()` 会做三件事：清本房存档（`chessSave`）、清座位令牌、复位 UI 回首页。
其中**清存档**是 v1.4.6 明确定下的语义 —— **显式退房 = 主动放弃本局**（§17.4）。

所以确认面板的文案必须**如实说明**，不能让用户以为"退出去还能回来接着下"：

```
退出后将离开当前房间，本局未下完的进度不会保留（存档会被清除）。确定退出？
```

实现上：
- 复用既有页内确认面板（`_showPrompt({ mode: 'exit-room' })`），
  **全程不用 `wx.showModal`** —— 这是全项目铁律（页内面板不阻塞、可冻结棋盘、探针全程监控）；
- 面板占用时直接 return，不叠加（与 `D4 面板新请求不覆盖面板` 同规则）；
- 取消路径**什么都不做**（留在房里），由探针 AD13 守住。

### 18.5 偏好存储**分层**：`chessPrefs` 与 `chessSave` 分开

新增**独立 key** `chessPrefs`：`{ hints, sound, skin }`。

| key | 生命周期 | 变更时机 |
|---|---|---|
| `chessSave` | **单局** | 每步落盘、进房/退房/换房都在变 |
| `chessSeatTokens` | 跨房间（3 个房间内） | 显式退房才清 |
| **`chessPrefs`** | **跨房间、跨对局**（永久） | 只在用户改设置时写 |

三个 key 各有各的生命周期，**不能合并** —— v1.4.6 的教训正是"共享存储互踩"
（`chessSave` 被无条件清除，连带毁掉恢复能力）。偏好若混进 `chessSave`，
用户退个房就把自己的开关重置了。

读取（`_readPrefs()`）的三条防线，任何异常都退回默认值、**绝不抛异常**：

```js
if (!raw) return def                                    // 没存过
if (!d || typeof d !== 'object' || Array.isArray(d)) return def   // 脏结构
const skin = (typeof d.skin === 'number' && d.skin >= 0 && d.skin < SKIN_OPTIONS.length)
  ? d.skin : def.skin                                   // 越界索引
```

> 越界那条不是洁癖：`skinOptions[99]` 会得到 `undefined`，
> 渲染到 picker 上就是一个空白的皮肤名。偏好数据是**用户可见**的，
> 必须在入口处校验，不能指望渲染层兜。

### 18.6 声音开关**已接管反馈**（不是纯装饰）

`WebAudio` 在当前实现里已被移除，反馈由 `wx.vibrateShort` 承担
（`playFeedback` / `playError` 两个函数）。「声音」开关直接接管这两个**出口**：

```js
playFeedback(move) {
  if (this.soundEnabled === false) return
  ...
}
```

两个设计点：
- **守卫放出口，不放调用点**：调用点分散在走子/吃子/将军/悔棋/错误提示等十几处，
  逐个加必漏一处；收敛到 2 个出口，"关掉声音就一声不出"才成立。
- **判据用 `=== false` 而不是 `!this.soundEnabled`**：只有用户**显式关**才静默。
  未初始化（`undefined`）时保持原有行为 —— 否则会引入一类新的静默 bug
  （"为什么没震动？"排查半天，结果是开关变量还没赋值）。

皮肤下拉框则**只落盘选择**（`chessPrefs.skin`），棋子图仍是默认资源 ——
本轮只把接口、持久化、越界防护先就位，接图时只改渲染层。

### 18.7 验证

- 主探针 `probe-index-page.js`：**371/371**（AD 段 27 项）
- 反向验证 `verify-v147-reverse.js`：**63/63**（10 组注入）
- 其余 11 个回归脚本全绿

| 注入 | 内容 | 转 FAIL | 仍 PASS（正确粒度） |
|---|---|---|---|
| A | `playFeedback` 去掉声音守卫 | AD6 | AD5 / AD14 |
| B | `playError` 去掉声音守卫 | AD6 | AD5 / AD14 |
| C | 退出确认后不执行 `_leaveRoom` | AD12 | AD11 / AD13 / AD10 |
| D | `_savePrefs` 变空实现 | AD14 / AD14b | AD15 / AD6 |
| E | `_readPrefs` 去掉越界校验 | AD15b | AD15 / AD14b |
| F | 退出时**额外**调一次 `wx.showModal` | AD11 | AD12 / AD13 |
| G | 换边按钮退回 `wx:if="{{!isSpectator && canSwap}}"` | **AD7b** | AD7 / AD8 / AD10 / AD1 |
| **I** | **给换边按钮加回 `disabled`** | **AD7** | AD7b / AD8 / AD8b |
| **J** | **把一条前置检查改成静默 `return`**（删掉 `_notice` 但保留 return） | **AD8 / AD8c** | AD8b / AD7 |
| H | 路径提示退回底部旧写法 | AD2 / AD2b / AD3 | AD1 / AD5 |

> 三组注入的设计值得记一笔：
> - **A/B 分开注入**：`playFeedback` 与 `playError` 是两条**独立出口**，
>   只给一条加守卫时 AD6 必须能抓出来（否则"关掉声音"名不副实）。
> - **F 不是"把面板换成 modal"，而是在面板之外额外调一次 modal**：
>   这样 `AD12`（确认后确实退房）不受影响，失败项**精准**落在"用了系统弹窗"这一条上。
>   若直接把面板换成 modal，AD12 也会跟着 FAIL —— 那是**测试路径**坏了，
>   不是行为退化，会让失败信号失真。
> - **G 与 I 是换边按钮的两种**不同失效模式**，一次注入只触发一种：
>   G（`wx:if` 混入 `canSwap`）只打掉 AD7b；I（加回 `disabled`）只打掉 AD7。
>   这正好证明 AD7/AD7b 是**各司其职**、没有重叠 ——
>   若把它们写成一个复合判据，就分辨不出是哪种退化（见 §18.3.1）。

### 18.8 附：本轮反向脚本踩的小坑（锚点选到了"高频词"）

注入 H（把「路径提示」退回底部）首版锚点写成 `'路径提示'`，脚本立刻报
`锚点出现 4 次`。原因：**WXML 注释里反复出现该词**（本轮新增的注释就写了 3 次）。

> **教训**：注释里解释某功能时，会反复提到该功能的名字 ——
> 于是**用"名字"当锚点必然撞车**。锚点要选**代码结构**（如
> `<text class="setting-label">路径提示</text>` 整行），而不是**自然语言关键词**。
> 这与 §17.7 的"断言 ID 跨脚本撞车"同源：都是**标识符不够独特**的问题，
> 靠的是同一件事 —— 选锚点/ID 前先确认它唯一。

---

## 十、同步开发约定（防漂移）

> ⚠️ **2026-09-12 起范围变更**：网页版**冻结于 v1.3.8 不再更新**（决策见 `SEAT_TOKEN_DESIGN.md` §零 R3/R4）。
> 下面前两条对 v1.3.8 之前的历史内容仍然有效；
> **v1.4.0 起的新增协议字段（`seatToken` 等）不再要求网页版同步** —— 网页版忽略未知字段。
> 但**兼容性约束反而更强**：房间内一旦有网页版玩家，新机制整体降级（详见 §11.4）。

- 改 appkey / host / 房间号规则 / 任一消息 type 或字段 → **两边必须同步改**，并同步更新本文件与 README。
- 新增交互（如新消息类型、新按钮）→ 先定协议，再各自实现，附跨端自测（网页建房↔小程序加入）。
- 棋子图、emoji 列表、身份列表属"展示数据"，改任一边需同步另一边。
- ⚠️ **文案不得承诺"进房后还会变的量"**（v1.4.2，见 §13）：
  入口按钮写「执白/执黑」是错误承诺 —— 真实座位进房后才由房况判定，且双方都能换边。
  **判据**：一个量若在用户看到文案之后还可能变化，文案就不能对它下结论。
  （网页版不受此约束的**唯一理由**是它没有换边功能、颜色真的不可变，属有意的两端分叉。）
- ⚠️ **新增任何"共享状态"时必须配一条拉取路径**（v1.4.1 血的教训，见 §12）：
  GoEasy pubsub **不回放历史**，只广播事件的字段对**后来者永久不可见**。
  座位用 `room_check→room_info/seat_state`、名册用 `spectator_query→spectator_list`。
  凡新状态，先问一句：**"我刚进房，怎么知道它？"** 答不上来就是下一个 v1.4.1。
- ⚠️ **"有信息"不等于"有读取路径"**（v1.4.3 的教训，见 §14.2）：
  座位令牌一直存在本地、也能跨重进存活，但 `joinRoom()` 从不读它 →
  房主重进自己的空房被判成黑方。**新增状态之后，还要检查每个"该用到它"的分支是否真的读了它。**
- ⚠️ **短标识符判等必须叠加实质条件**（v1.4.4 的教训，见 §15.2）：
  房间号仅 4 位、会重复使用。凡"仅凭一个短 ID 相同就认定是同一实体"的判据
  （恢复对局、认座、找回存档…），必须再叠一条**只有真实发生过才会成立**的条件
  （这里是"存档里有走子"），否则旧数据会污染新场景。
- ⚠️ **落盘要选"唯一汇合点"，不要撒在多个入口**（v1.4.4，见 §15.3）：
  走子后的保存放在 `handleMoveAftermath()` —— 自己走子与收到对方 move 都经过它。
  撒在 `onSquareTap` / `confirmPromotion` / `onMessage` 三处迟早漏一处；
  而"只在选身份时存一次"这种写法会让存档永久停在开局（该按钮形同虚设的根因）。
- ⚠️ **反向验证脚本自身也要有判据**（v1.4.4，见 §15.5）：
  `String.replace` 只替换第一处。锚点不唯一时会**静默打到别的函数**，
  而"注入确实写入文件"这条检查**照样通过**。注入器必须强制校验**锚点全文件唯一**。
  同理：**"文件被改过"不等于"改对了地方"。**
- ⚠️⚠️ **`bindtap` 会把点击事件对象作为第一个实参传进处理函数**（v1.4.5，见 §16.2）：
  凡处理函数带**可选入参**（`fn(d)`），且 WXML 用 `bindtap="fn"` 绑定时，
  **不能用 truthy 判定**（`if (!d)`）—— 事件对象是 truthy，会一路穿过守卫。
  要按**语义特征**判定（这里是 `d.roomId` 是否存在）。
  **本项目已因此翻车两次**（`joinRoom`、`resumeGame`）。
  新增入参时的**必做回查**：该函数在 WXML 里是怎么绑的？带不带 `data-*`？
- ⚠️ **删功能要删干净，且每一条删除面各配一条断言**（v1.4.5，见 §16.5）：
  "某个函数已经没人调用了" ≠ "这个功能被删除了"。同一功能往往有**多个删除面**
  （函数定义 / 调用点 / 兜底字面量 / 分支文案 / 静态模板文案 / 失败时的降级路径）。
  只守其中一面的断言，在注入其他面时**照样全绿**。
  v1.4.5 的 E/H/I/G 四组注入就是为这四个独立删除面分别设的。
- ⚠️⚠️ **反向验证要同时校验"注入写对了"**（v1.4.5，见 §16.6）：
  反向验证有**两个**前提，缺任一条结论都不可采信 ——
  ① 断言有鉴别力；② **注入确实把旧能力注回去了**。
  写"退回旧写法"的注入时，**必须逐字对照 git 历史里的旧代码**：
  凭印象写一个"语义等价"的版本，往往**恰好仍然是修复后的行为**
  （v1.4.5 首版注入 D 就犯了这个错，`if (!net || !room) return` 本身就是一条守卫）。
  **脚本首版若报"某断言转 FAIL 失败"，排查顺序是先怀疑注入、再怀疑断言。**
- ⚠️ **断言可能因"别的守卫先拦住了"而假绿**（v1.4.5，见 §16.6 的 K3）：
  一个用例若同时满足多个守卫的拦截条件，它就**测不出**目标守卫在不在。
  `K3` 没设 `isNetworkReady=true`，于是落子被网络检查拦住 —— 基线绿、
  注入房间守卫后**还是绿**。写"不得发生 X"的用例时，要把**其它**拦截条件都置为"可通过"，
  只留下被测的那一个。
- ⚠️⚠️ **"判据混用"是同一现象成套报错的常见根因**（v1.4.6，见 §17）：
  同一个概念（"这还是我的那一局吗"）被两个函数、两套数据源各自判断，
  一旦两者判据宽严不一，就会**撕裂**成两条看似无关的现象：
  按钮消失（可见性用了严判据）+ 认座成功却仍要选身份（认座用令牌、恢复用严判据）。
  修法不是"把严判据放松"，而是**按调用意图分层**：
  "能否恢复"（松）与"是否属于本房"（严，仅用于消歧）拆成两个函数。
- ⚠️⚠️ **跨脚本引用断言 ID 前，必须确认该 ID 在目标探针里的归属**（v1.4.6，见 §17.7）：
  反向验证脚本跑的是 A 探针，鉴别点却写成只在 B 脚本里存在的 ID；
  若该 ID 恰好**在 A 探针里被别的断言占用**，就会恒 PASS，
  并**伪装成"注入锚点失配"**（本可 grep 一次就排除）。
  新型号段一律起**新前缀**，不复用已占用的字母段。
- ⚠️⚠️ **"不可用"与"不存在"要区分对待；而且"不可用"必须说清原因**（v1.4.7→v1.4.8，见 §18.3）：
  能力**不存在** → 隐藏（观战者不该看到换边按钮，那会误导他去找"点亮方法"）；
  能力**存在但条件不满足** → **可点 + 提示具体原因**（不要说"不能"，要说"为什么不能"）。
  演进：v1.4.7 先做成"灰显"，用户实测后改主意 —— 灰按钮只表达"不能用"、
  说不出原因，用户只能猜是步数、对手未进房还是网络；而"可点 + 提示"的信息量大得多，
  且与本项目「悔棋」的既有交互一致。
  > **准则：与其发明新交互，不如对齐项目里已验证过的那一种。**
  > ⚠️ **硬约束**：小程序里 `disabled` 的 `button` **不触发 `bindtap`** ——
  > 所以"灰显"与"给提示"**在实现上互斥**，只能二选一。要提示就必须拿掉 `disabled`
  > （同时拿掉灰样式，否则会变成"看着能点、其实是灰的"这种更糟的误导）。
  > ⚠️ 反向验证要**分别注入**这两种失效模式（v1.4.7 的 G 打 AD7b、I 打 AD7）——
  > 写成一个复合判据就分辨不出究竟是哪种退化。
- ⚠️ **断言要"剥注释再判"，否则会被自己的注释满足**（v1.4.7 / v1.4.8 各踩一次）：
  本项目惯例是"删除能力时在原址**留占位注释**说明决策"，而那行注释里
  **必然出现被删掉的选择器 / 关键字**。于是：
  - v1.4.7：`/\.btn-off/.test(wxss)` 被"说明 .btn-off 已移除"的注释命中；
  - 同理 WXML 侧（Y5）要 `replace(/<!--[\s\S]*?-->/g, '')` 后再判。
  > **准则：凡是断言"某东西**不存在**"，都要先剥掉注释与字符串字面量。**
  > 否则这条断言在"删干净了"和"只剩注释"两种状态下**结论相同** —— 等于没守。
- ⚠️ **UI 锚点不要用"自然语言关键词"，要用代码结构**（v1.4.7，见 §18.8）：
  注释里解释某功能时会反复提到它的名字 → 拿名字当锚点**必然撞车**
  （v1.4.7 注入 H 的锚点 `'路径提示'` 撞了 4 次，全是注释）。
  锚点选整行结构（`<text class="setting-label">路径提示</text>`）才唯一。
  > 与上一条同源：**选标识符前先确认它唯一** —— 无论是断言 ID 还是注入锚点。
