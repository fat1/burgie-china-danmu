# Burgie 直播弹幕助手

> 本项目源自原作者 **HeyNau Games** 的 [BLR（Burgie's Livestream Reader）1.02](https://steamcommunity.com/app/3314340/discussions/0/561408449130437193/) 插件。

> **仓库地址**：[GitHub](https://github.com/wwwqsid/burgie-china-danmu) | [Gitee](https://gitee.com/wwwqsid/burgie-china-danmu)

原插件支持抖音、B站、小红书三个平台，并通过本地 IRC 服务器把弹幕转发给游戏 Burgie's Cozy Kitchen。本项目在此基础上做了以下修改：

1. **新增快手平台**：拦截 WebSocket 并解析 Protobuf 数据
2. **新增微信视频号平台**：同时拦截 HTTP 响应和 WebSocket
3. **界面改为中文**
4. **显示浏览器窗口**：可以直观看到拦截过程
5. **移除 IRC 转发**：弹幕不再发给游戏，直接显示在程序窗口中，并按类型着色
6. **修复抖音解析**：按抖音当前的数据格式重写，新增关注事件

## 支持平台

| 平台 | 来源 | 连接方式 | 是否需要登录 |
|------|------|---------|-------------|
| 抖音 | 原作者 | 拦截 WebSocket | 否 |
| B站 | 原作者 | 拦截 WebSocket | 否 |
| 小红书 | 原作者 | 拦截 WebSocket | 否 |
| 快手 | 新增 | 拦截 WebSocket，解析 Protobuf | 否 |
| 视频号 | 新增 | 拦截 HTTP 响应和 WebSocket | 需要主播用微信扫码 |

## 功能说明

读取直播间的弹幕、礼物、点赞、进场、关注等事件，直接显示在程序窗口中。

## 使用方法

### 方式零：直接下载 exe

**GitHub（完整压缩包）**：在 [Releases 页面](https://github.com/wwwqsid/burgie-china-danmu/releases/tag/v1.02) 下载 `Burgie弹幕助手-v1.02.zip`，解压后双击 `Burgie弹幕助手.exe` 即可运行，无需安装 Python。

**Gitee（分卷下载）**：在 [Gitee Releases 页面](https://gitee.com/wwwqsid/burgie-china-danmu/releases/tag/v1.02) 下载全部 4 个分卷（part1 至 part4），用以下命令合并后再解压：

```cmd
copy /b Burgie-danmu-v1.02.part1.zip+Burgie-danmu-v1.02.part2.zip+Burgie-danmu-v1.02.part3.zip+Burgie-danmu-v1.02.part4.zip Burgie-danmu-v1.02.zip
```

### 方式一：从源码运行

1. 安装 Python 3.10 或更高版本
2. 安装依赖：`pip install playwright brotli`（brotli 用于解压 B站弹幕）
3. 准备浏览器：已安装 Chrome 或 Edge 时会自动使用；否则执行 `playwright install chromium`
4. 双击 `启动弹幕助手.bat`

### 方式二：自行打包为 exe

```bash
pip install pyinstaller
pyinstaller "Burgie弹幕助手.spec" --noconfirm
```

打包结果在 `dist/Burgie弹幕助手/` 目录下。

## 技术栈

- **Python** + **Tkinter**：图形界面
- **Playwright**：浏览器自动化，拦截 WebSocket 和 HTTP 响应
- **手写 Protobuf 解码**：不依赖第三方 protobuf 库

## 运行原理

### 总体架构

本程序是一个**直播弹幕读取器**：从直播平台读取弹幕、礼物、点赞、进场等事件，解析后直接显示在程序窗口中。

```
直播平台网页 --(WebSocket/HTTP)--> Playwright 浏览器 --拦截数据帧--> 平台解析器
                                                                      |
                                                     弹幕/礼物/点赞/进场等事件
                                                                      v
                                                      Tkinter 窗口日志（按类型着色）
```

全部逻辑都在 `BLR_ks.py` 中：

| 组件 | 运行线程 | 职责 |
|------|---------|------|
| `App`（图形界面） | 主线程 | 选择平台、输入房间号、连接和断开、显示日志 |
| `run_douyin` / `run_bilibili` / `run_xhs` / `run_kuaishou` / `run_wechat_channels` | 独立线程中的 asyncio 事件循环 | 启动浏览器并拦截网络数据 |
| `proto_fields` 等工具函数 | 无 | 不需要协议定义文件的 Protobuf 解码器 |
| 日志 | 无 | 写入 `blr_debug.log`（超过 2MB 自动轮换）；含 `[调试]` 的消息只写入文件，不显示在界面 |

### 启动与连接流程

1. **查找浏览器**（`find_chromium_exe`）：优先使用程序自带的 `chromium_data/chromium-*/chrome-win64/chrome.exe`，找不到时改用系统安装的 Chrome 或 Edge。
2. **点击“连接”**（`App.start`）：
   - 解析房间号：快手支持字母数字混合的房间号，也支持 `live.kuaishou.com/u/xxx` 链接；视频号不需要房间号；其他平台从输入中提取 5 位以上的数字。
   - 每次连接都新建一个 `asyncio.Event` 作为停止信号，然后在新线程中运行对应平台的协程。
3. **点击“断开”**：通过 `loop.call_soon_threadsafe(stop_ev.set)` 跨线程发出停止信号，协程收到后关闭浏览器，界面恢复为“未连接”。直接关闭弹出的浏览器窗口也会结束连接。

### 核心机制：用真实浏览器拦截数据

所有平台都用同一种方式：以有界面模式启动浏览器（隐藏自动化特征，使用常规的浏览器标识），打开直播间页面，然后监听网络事件：

```python
page.on('websocket', on_ws)        # 页面每建立一个 WebSocket 时触发
ws.on('framereceived', on_frame)   # 收到的每一帧原始数据
page.on('response', on_response)   # 仅视频号使用：监听 HTTP 响应
```

签名、加密参数、心跳和鉴权都由平台自己的网页代码完成，本程序只“旁听”数据，不需要逆向平台的连接协议。代价是每次连接都要运行一个完整的浏览器。

### 各平台解析方式

| 平台 | 打开的页面 | 如何识别弹幕连接 | 数据格式与解析 |
|------|-----------|-----------------|---------------|
| 抖音 | `live.douyin.com/{房间号}` | 地址包含 `webcast` | 外层帧为 Protobuf：字段 7 是帧类型（`msg` 为消息，`hb` 为心跳），字段 8 是 gzip 压缩的内容。解压后，字段 1 是打包在一起的多条消息；每条消息的字段 1 是消息类型，字段 2 是消息正文，用户昵称在用户信息的字段 3。按消息类型分派：`WebcastChatMessage`（弹幕）、`WebcastGiftMessage`（礼物）、`WebcastLikeMessage`（点赞）、`WebcastMemberMessage`（进场）、`WebcastSocialMessage`（字段 4 为 1 时表示关注） |
| 小红书 | `xiaohongshu.com/livestream/{房间号}` | 地址包含 `xiaohongshu` | JSON 中的 `data.items[].b` 先做 Base64 解码，得到内层 JSON 的 `customData`，再按 `type` 分派：`text`（弹幕）、`gift_comment`（礼物）、`audience_join_v2`（进场）、`praise`（点赞）、`follow_emcee`（关注） |
| B站 | `live.bilibili.com/{房间号}` | 地址包含 `chat.bilibili.com` 和 `/sub` | 每个包有 16 字节包头（总长度、包头长度、协议版本、操作码、序号），一帧可能包含多个包，只处理操作码为 5 的消息包。协议版本 2 用 zlib 压缩、版本 3 用 brotli 压缩，解压后再递归拆包。按 `cmd` 分派：`DANMU_MSG`（弹幕）、`SEND_GIFT`（礼物）、`SUPER_CHAT_MESSAGE`（醒目留言）、`INTERACT_WORD` 和 `INTERACT_WORD_V2`（进场）、`GUARD_BUY`（上舰） |
| 快手 | `live.kuaishou.com/u/{房间号}` | 地址包含 `kuaishou` 或 `live-ws` | 外层为 Protobuf：字段 1 是消息类型编号，字段 2 是压缩方式，字段 3 是内容。类型 `310` 是聚合推送，其中字段 5 为弹幕、9 为礼物、8 为点赞、12 为分享；类型 `340` 是在线观众列表，收到的第一份列表作为基准，之后新出现的观众视为进场；类型 `300` 表示已进入直播间 |
| 视频号 | `channels.weixin.qq.com/platform/live/liveBuild` | 地址包含 `weixin`、`channels` 或 `finder`，同时监听这些域名的 HTTP 响应 | 需要主播扫码登录后台。按候选字段名猜测解析，详见下一节 |

### 视频号抓取说明

视频号和其他平台差别较大，这里单独说明（对应函数 `run_wechat_channels`）：

1. **打开主播后台并扫码登录**：视频号网页端没有供观众观看的直播间页面，只能打开主播后台 `channels.weixin.qq.com/platform/live/liveBuild`。所以不需要输入房间号，但必须由**主播本人**用微信扫码登录，程序最多等待 5 分钟。登录后进入直播管理页面并开播，后台页面才会开始获取弹幕。
2. **同时监听两条数据通道**：
   ```python
   page.on('websocket', on_ws)        # 地址包含 weixin / channels / finder 的 WebSocket
   page.on('response', on_response)   # channels / live / finder.weixin.qq.com 域名下的 JSON 类 HTTP 响应
   ```
   后台页面主要通过定时请求 HTTP 接口获取新弹幕，程序读取这些响应；WebSocket 作为备用通道。
3. **按候选字段名猜测解析**：
   - 先去掉 `{errCode, errMsg, data}` 这类外层包装；
   - 在名字类似 `comments`、`msgs`、`list` 的列表中逐条查找：
     - 昵称：`nickname`、`nickName`、`userName` 等，也会到 `user` 等子对象中查找；
     - 内容：`content`、`text`、`msg` 等，有内容就视为弹幕；
     - 礼物：`giftName`、`giftId` 等；
     - 点赞：出现 `like`、`praise` 等键；
     - 进场：有昵称、没有内容，并且出现 `enter`、`join` 等键；
   - 如果是二进制（Protobuf）数据，就尝试几个常见的字段编号来提取昵称和内容。

**局限**：这套解析从未用真实数据核对过。真实的字段名不在候选列表中时会漏掉弹幕；后台其他接口恰好带有 `name`、`content` 这类字段时，可能被误当成弹幕。要确认能否正常使用，需要在主播开播时抓取真实的 HTTP 响应和 WebSocket 数据，再按真实格式改写解析。

### 输出：界面显示

解析出的事件通过 `App.log(msg, tag)` 显示在窗口的日志区，按类型着色：

| 类型 | 颜色 | 示例 |
|------|------|------|
| 弹幕 | 白色 | `[B站] 昵称: 内容` |
| 礼物、B站上舰 | 金色 | `[抖音] [礼物] 昵称 送出 礼物 x数量` |
| 点赞 | 粉色 | `[快手] [点赞] 昵称` |
| 进场、关注、分享 | 绿色 | `[小红书] [进入] 昵称 进入直播间` |
| B站醒目留言 | 橙色 | `[B站] [SC 30元] 昵称: 内容` |
| 系统消息 | 灰色 | 连接状态、错误提示 |

调试信息（含 `[调试]`）只写入 `blr_debug.log`，不显示在界面。

### 已知限制

- **快手礼物**只能拿到礼物编号，显示为 `gift_{编号}`，没有礼物名称。
- **快手进场**是根据观众列表的变化推算的，只是近似结果，可能漏掉部分观众。
- **抖音礼物**的字段位置尚未用真实数据核对。
- **视频号**靠字段名猜测解析，可能漏报或误报。

## 许可

本项目基于 HeyNau Games 的 BLR 1.02 修改，仅供学习交流使用。

## 关于本项目

前 4 项修改（新增快手平台、新增视频号平台、界面中文化、显示浏览器窗口）以及 PyInstaller 打包配置，由 **WorkBuddy AI** 协助完成。如需了解 WorkBuddy，请访问 [codebuddy.cn](https://www.codebuddy.cn)。

## 联系方式

- 邮箱：wwwqsid@outlook.com
