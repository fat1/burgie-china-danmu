[English](README_EN.md) | 简体中文

> ⚠️ **本文件由 WorkBuddy AI 撰写。**

# Burgie 直播弹幕助手

> 本项目源自原作者 **HeyNau Games** 的 [BLR (Burgie's Livestream Reader) 1.02](https://steamcommunity.com/app/3314340/discussions/0/561408449130437193/) 插件。

> 📌 **仓库地址**：[GitHub](https://github.com/wwwqsid/burgie-china-danmu) | [Gitee](https://gitee.com/wwwqsid/burgie-china-danmu)

原插件已支持抖音、B站、小红书三大平台。本项目在原插件基础上做了以下 **4 项修改**（由 WorkBuddy AI 协助完成）：

1. **新增快手平台** — WebSocket 拦截 + Protobuf 解析
2. **新增微信视频号平台** — HTTP 响应拦截 + WebSocket 双通道
3. **界面改为中文** — 适配中国用户使用习惯
4. **新增浏览器显示** — 可直观查看拦截过程

## ✨ 支持平台

| 平台 | 来源 | 连接方式 | 需要登录 |
|------|------|---------|---------|
| 抖音 Douyin | 原作者 | WebSocket 拦截 | 否 |
| B站 哔哩哔哩 | 原作者 | WebSocket 拦截 | 否 |
| 小红书 XHS | 原作者 | WebSocket 拦截 | 否 |
| 快手 Kuaishou | ✨ 新增 | WebSocket 拦截 + Protobuf 解析 | 否 |
| 视频号 WeChat | ✨ 新增 | HTTP 响应拦截 + WebSocket | 需微信扫码（主播账号） |

## 🎮 配合游戏

[Burgie's Cozy Kitchen](https://heynaugames.com/burgie-commands) 直播做饭游戏，弹幕指令包括点餐、投诉、换皮肤等。

## 🚀 使用方法

### 方式零：直接下载 exe（推荐 ⭐）

**GitHub（单文件完整下载）**：前往 [Releases 页面](https://github.com/wwwqsid/burgie-china-danmu/releases/tag/v1.02) 下载 `Burgie弹幕助手-v1.02.zip`，解压后双击 `Burgie弹幕助手.exe` 即可运行，无需安装 Python。

**Gitee（分卷下载）**：前往 [Gitee Releases 页面](https://gitee.com/wwwqsid/burgie-china-danmu/releases/tag/v1.02) 下载全部 4 个分卷文件（part1 ~ part4），使用以下命令合并后再解压：

```cmd
copy /b Burgie-danmu-v1.02.part1.zip+Burgie-danmu-v1.02.part2.zip+Burgie-danmu-v1.02.part3.zip+Burgie-danmu-v1.02.part4.zip Burgie-danmu-v1.02.zip
```

### 方式一：从源码运行

1. 安装 Python 3.10+
2. 安装依赖：`pip install playwright`
3. 安装 Chromium：`playwright install chromium`
4. 双击 `启动弹幕助手.bat`

### 方式二：自行打包为 exe

```bash
pip install pyinstaller
pyinstaller "Burgie弹幕助手.spec" --noconfirm
```

打包结果在 `dist/Burgie弹幕助手/` 目录下。

## 📋 弹幕指令

| 指令 | 用途 |
|------|------|
| `!Burgie [食材]` | 点一个汉堡 |
| `!Bell` | 按铃铛 |
| `!Skin {性别} {动物}` | 更换皮肤（订阅者/VIP） |
| `!Leave` | 离开柜台 |
| `!Dirty` | 投诉食物碰地 |
| `!Fire` | 投诉起火 |
| `!Noise` | 投诉按铃铛 |

完整指令列表见 [官网](https://heynaugames.com/burgie-commands)。

## 🔧 技术栈

- **Python** + **Tkinter**（GUI）
- **Playwright**（浏览器自动化 / WebSocket 拦截）
- **Protobuf** 手动解析（无第三方 protobuf 库依赖）
- **IRC Server**（本地 TCP 转发弹幕到游戏）

## 运行原理

### 总体架构

本程序是一个**弹幕桥接器**：从直播平台读取弹幕、礼物、点赞、进场事件，统一转换为 **IRC 协议消息**，推送给游戏 Burgie's Cozy Kitchen（游戏内输入 `BLRMod` 后，会把本程序当作 IRC 服务器连接）。

```
直播平台网页 --(WebSocket/HTTP)--> Playwright 浏览器 --拦截数据帧--> 平台解析器
                                                                      |
                                                    统一事件 CHAT/GIFT/LIKE/JOIN...
                                                                      v
游戏 Burgie's Cozy Kitchen <--(TCP localhost:6667, PRIVMSG #live)-- IRCServer
```

全部逻辑位于 `BLR_ks.py`：

| 组件 | 运行线程 | 职责 |
|------|---------|------|
| `App`（Tkinter GUI） | 主线程 | 选择平台、输入房间号、连接/断开、显示日志 |
| `IRCServer` | accept 线程 + 每客户端一个线程 | 在本地 6667 端口模拟 IRC 服务器，向游戏广播事件 |
| `run_douyin` / `run_bilibili` / `run_xhs` / `run_kuaishou` / `run_wechat_channels` | 独立线程内的 asyncio 事件循环 | 启动浏览器并拦截网络数据 |
| `proto_fields` 等工具函数 | - | 手写的无 schema Protobuf 解码器 |
| 日志 | - | 写入 `blr_debug.log`（2MB 轮转）；含 `[调试]` 的消息只写文件，不显示在界面 |

### 启动与连接流程

1. **查找浏览器**（`find_chromium_exe`）：优先使用自带的 `chromium_data/chromium-*/chrome-win64/chrome.exe`，找不到时回退到系统安装的 Chrome / Edge。
2. **启动 IRC 服务器**：程序打开即监听 `0.0.0.0:6667`。
3. **点击"连接"**（`App.start`）：
   - 解析房间号：快手支持字母数字混合 ID 或 `live.kuaishou.com/u/xxx` 链接；视频号无需房间号；其他平台从输入中提取 5 位以上数字。
   - 每次连接新建一个 `asyncio.Event` 作为停止信号，然后在新线程中运行对应平台的协程。
4. **点击"断开"**：通过 `loop.call_soon_threadsafe(stop_ev.set)` 跨线程发出停止信号，协程检测到后关闭浏览器，界面恢复为未连接状态。

### 核心机制：真实浏览器拦截

所有平台都使用同一种方式：以有界面模式启动 Chromium（隐藏自动化特征并设置常规 UA），打开直播间页面，然后监听网络事件：

```python
page.on('websocket', on_ws)        # 页面每建立一个 WebSocket 时触发
ws.on('framereceived', on_frame)   # 收到的每一帧原始数据
page.on('response', on_response)   # 仅视频号：监听 HTTP 响应
```

签名、加密参数、心跳和鉴权都由平台自己的前端代码完成，本程序只"旁听"数据，无需逆向连接协议。代价是每次连接都要运行一个完整的浏览器。

### 各平台解析方式

| 平台 | 打开的页面 | WebSocket 筛选 | 数据格式与解析 |
|------|-----------|---------------|---------------|
| 抖音 | `live.douyin.com/{id}` | URL 含 `webcast` | Protobuf 外层帧（payload 可能 gzip 压缩），按 `method` 字段分派：`WebcastChatMessage` / `WebcastGiftMessage` / `WebcastLikeMessage` / `WebcastMemberMessage` |
| 小红书 | `xiaohongshu.com/livestream/{id}` | URL 含 `xiaohongshu` | JSON -> `data.items[].b`（Base64）-> 内层 JSON `customData`，按 `type` 分派：`text` / `gift_comment` / `audience_join_v2` / `praise` / `follow_emcee` |
| B站 | `live.bilibili.com/{id}` | URL 含 `chat.bilibili.com` 且含 `/sub` | 16 字节包头（总长/头长/ver/op/seq），一帧可含多个包，只处理 op=5；ver=2 用 zlib、ver=3 用 brotli 解压后递归拆包；按 `cmd` 分派：`DANMU_MSG` / `SEND_GIFT` / `SUPER_CHAT_MESSAGE` / `INTERACT_WORD(_V2)` / `GUARD_BUY` |
| 快手 | `live.kuaishou.com/u/{id}` | URL 含 `kuaishou` 或 `live-ws` | SocketMessage Protobuf（field 1=payloadType，2=压缩方式，3=payload）。`310` 为聚合推送：field 5 弹幕、9 礼物、8 点赞、12 分享；`340` 为在线观众列表，首个列表作为基线，之后新出现的用户视为进场；`300` 为进房确认 |
| 视频号 | `channels.weixin.qq.com/platform/live/liveBuild` | URL 含 `weixin` / `channels` / `finder`，同时监听这些域名的 HTTP JSON 响应 | 需主播扫码登录后台（最多等待 5 分钟）。采用启发式解析：同时尝试 JSON 和 Protobuf，在 `comments` / `msgs` / `list` 等列表中按 `nickname` / `content` / `giftName` 等候选字段名提取 |

### 输出：IRC 协议

游戏连接后发送 `NICK` / `USER`，服务器回复 `001`、`376` 并加入 `#live` 频道。所有平台的事件都按统一格式广播：

```
:{昵称}!live@BLRMod PRIVMSG #live :{类型}|{用户}|{内容}|{附加}
```

- `类型`：`CHAT` / `GIFT` / `LIKE` / `JOIN` / `SC` / `SHARE` / `follow` / `guard`
- `附加`：礼物数量等额外信息
- 昵称中的空格替换为 `_`，换行符会被去除

本程序**不解析游戏指令**；`!Burgie`、`!Bell` 等指令由游戏收到 `CHAT` 消息后自行识别。

### 已知限制

- **快手礼物**只能取到礼物 ID，输出为 `gift_{id}`，不是礼物名称。
- **快手进场**通过观众列表的变化推算，属于近似结果，可能漏掉部分用户。
- **视频号**解析依赖字段名猜测，可能漏报或误报。

## 📄 许可

本项目基于 HeyNau Games 的 BLR 1.02 修改，仅供学习交流使用。

## 🤖 关于本项目

本项目仅在原作者 BLR 1.02 基础上做了 4 项修改（新增快手平台、新增视频号平台、界面中文化、新增浏览器显示），以及 PyInstaller 打包配置和本文档，均由 **WorkBuddy AI** 协助完成。

- 快手平台弹幕解析：WorkBuddy AI
- 视频号平台弹幕解析：WorkBuddy AI
- 界面中文化：WorkBuddy AI
- 浏览器显示功能：WorkBuddy AI
- PyInstaller 打包配置与 exe 生成：WorkBuddy AI
- 本 README.md 文件撰写：WorkBuddy AI

如需了解 WorkBuddy，请访问 [codebuddy.cn](https://www.codebuddy.cn)

## 📬 联系方式

- Email: wwwqsid@outlook.com
