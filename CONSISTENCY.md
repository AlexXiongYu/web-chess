# 网页版 ↔ 小程序版 一致性对照报告

> 生成时间：2026-09-11
> 网页版：`docs/`（本目录，GitHub Pages）　小程序版：`WeChatProjects/miniprogram-1`
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
| 版本号展示 | 页面右下角灰色小字 `v1.3.5`（`APP_VERSION`） | 同左（`version` 字段绑定） |
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
| **轮次动画与既有 ●/○ 标记重复（v1.3.5）** | 玩家名文本里已内嵌回合标记（`render()` 写入 `oppDot`/`myDot`：`●` 该走 / `○` 待走），Task 7 又额外加了 `.active-turn .player-name::before` 脉冲圆点 → 同一行出现**两个符号** | 删掉 `::before` 圆点，把呼吸动画改作用在**既有 ●/○ 本身**（`.active-turn .player-name { animation: turnDot }`），并把幅度从 `scale(1.55)` 收敛到 `1.12` 避免过冲 |

### 5.4 回归验证

两端各有一份 Node 探针，**改协议后必须跑过**：

- 小程序：`WeChatProjects/.ci-secrets/probe-index-page.js`（123 项，覆盖观战者/收件人/超时/交叉/脏数据/身份栏/将杀胜者/页内提示与冻结/**进房后选身份**/**空房不许落子**/对弈者回归/**GoEasy单例复用**/**分享卡片自动进房**/**结局输赢配色**/**覆盖层半透明**/**轮次动画不重复**）
- 网页版：`WeChatProjects/.ci-secrets/probe-web.js`（87 项，同一批场景 + **表情面板不裁切**/**结局配色**/**回合动画**/**覆盖层透明度**/**无重复圆点**）

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

## 九、同步开发约定（防漂移）

- 改 appkey / host / 房间号规则 / 任一消息 type 或字段 → **两边必须同步改**，并同步更新本文件与 README。
- 新增交互（如新消息类型、新按钮）→ 先定协议，再各自实现，附跨端自测（网页建房↔小程序加入）。
- 棋子图、emoji 列表、身份列表属"展示数据"，改任一边需同步另一边。
