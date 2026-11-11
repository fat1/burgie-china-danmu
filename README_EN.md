English | [简体中文](README.md)

> ⚠️ **This document is written by WorkBuddy AI.**

# Burgie China Danmu Assistant

> This project is derived from the original author **HeyNau Games**' [BLR (Burgie's Livestream Reader) 1.02](https://steamcommunity.com/app/3314340/discussions/0/561408449130437193/) plugin.

> 📌 **Repository**: [GitHub](https://github.com/wwwqsid/burgie-china-danmu) | [Gitee](https://gitee.com/wwwqsid/burgie-china-danmu)

The original plugin already supported three platforms: Douyin, Bilibili, and Xiaohongshu. This project makes the following **4 modifications** to the original plugin (assisted by WorkBuddy AI):

1. **Added Kuaishou platform** — WebSocket interception + Protobuf parsing
2. **Added WeChat Channels platform** — HTTP response interception + WebSocket dual-channel
3. **Chinese localization** — UI adapted for Chinese users
4. **Added browser display** — Visual inspection of the interception process

## ✨ Supported Platforms

| Platform | Source | Connection Method | Login Required |
|----------|--------|------------------|----------------|
| Douyin (TikTok China) | Original Author | WebSocket interception | No |
| Bilibili | Original Author | WebSocket interception | No |
| Xiaohongshu (RED) | Original Author | WebSocket interception | No |
| Kuaishou | ✨ New | WebSocket interception + Protobuf parsing | No |
| WeChat Channels (视频号) | ✨ New | HTTP response interception + WebSocket | Yes (WeChat QR scan, streamer account) |

## 🎮 How It Works

The tool reads live chat messages from Chinese streaming platforms and forwards them to the game **Burgie's Cozy Kitchen** via a local IRC server. Viewers can order burgers, ring bells, change skins, and file complaints — all through chat commands.

## 🚀 Getting Started

### Option 0: Download Executable (Recommended ⭐)

**GitHub (single file download)**: Go to the [Releases page](https://github.com/wwwqsid/burgie-china-danmu/releases/tag/v1.02) and download `Burgie弹幕助手-v1.02.zip`. Extract and run `Burgie弹幕助手.exe` — no Python installation required.

**Gitee (split archive)**: Go to the [Gitee Releases page](https://gitee.com/wwwqsid/burgie-china-danmu/releases/tag/v1.02) and download all 4 parts (part1 ~ part4). Merge them with the following command, then extract:

```cmd
copy /b Burgie-danmu-v1.02.part1.zip+Burgie-danmu-v1.02.part2.zip+Burgie-danmu-v1.02.part3.zip+Burgie-danmu-v1.02.part4.zip Burgie-danmu-v1.02.zip
```

### Option 1: Run from Source

1. Install Python 3.10+
2. Install dependencies: `pip install playwright`
3. Install Chromium: `playwright install chromium`
4. Run `启动弹幕助手.bat`

### Option 2: Build as Executable

```bash
pip install pyinstaller
pyinstaller "Burgie弹幕助手.spec" --noconfirm
```

The built executable will be in `dist/Burgie弹幕助手/`.

## 📋 Chat Commands

| Command | Description |
|---------|-------------|
| `!Burgie [ingredients]` | Order a burger |
| `!Bell` | Ring the bell |
| `!Skin {gender} {animal}` | Change skin (Subscribers/VIP only) |
| `!Leave` | Leave the counter |
| `!Dirty` | Complain: food touched the floor |
| `!Fire` | Complain: pan is on fire |
| `!Noise` | Complain: too much bell ringing |

Full command list: [Official Site](https://heynaugames.com/burgie-commands)

## 🔧 Tech Stack

- **Python** + **Tkinter** — GUI
- **Playwright** — Browser automation / WebSocket interception
- **Protobuf** — Manual parsing (no third-party protobuf library)
- **IRC Server** — Local TCP relay to forward danmaku to the game

## Architecture

### Overview

This program is a **danmaku bridge**: it reads chat messages, gifts, likes and join events from live streaming platforms, converts them into **IRC protocol messages**, and pushes them to Burgie's Cozy Kitchen (typing `BLRMod` in-game makes the game connect to this program as an IRC server).

```
Platform web page --(WebSocket/HTTP)--> Playwright browser --intercepted frames--> Platform parser
                                                                                        |
                                                              Unified events CHAT/GIFT/LIKE/JOIN...
                                                                                        v
Burgie's Cozy Kitchen <--(TCP localhost:6667, PRIVMSG #live)-- IRCServer
```

All logic lives in `BLR_ks.py`:

| Component | Thread | Responsibility |
|-----------|--------|----------------|
| `App` (Tkinter GUI) | Main thread | Platform selection, room ID input, connect/disconnect, log display |
| `IRCServer` | Accept thread + one thread per client | Emulates an IRC server on local port 6667 and broadcasts events to the game |
| `run_douyin` / `run_bilibili` / `run_xhs` / `run_kuaishou` / `run_wechat_channels` | asyncio event loop in a dedicated thread | Launch the browser and intercept network data |
| `proto_fields` and helpers | - | Hand-written schema-less Protobuf decoder |
| Logging | - | Written to `blr_debug.log` (2 MB rotation); messages containing `[调试]` (debug) go to the file only, not the GUI |

### Startup and Connection Flow

1. **Locate a browser** (`find_chromium_exe`): prefers the bundled `chromium_data/chromium-*/chrome-win64/chrome.exe`, falling back to an installed Chrome / Edge.
2. **Start the IRC server**: listens on `0.0.0.0:6667` as soon as the program opens.
3. **Click "Connect"** (`App.start`):
   - Parse the room ID: Kuaishou accepts alphanumeric IDs or `live.kuaishou.com/u/xxx` links; WeChat Channels needs no room ID; other platforms extract a number of 5+ digits from the input.
   - A new `asyncio.Event` is created as the stop signal for every connection, then the platform coroutine runs in a new thread.
4. **Click "Disconnect"**: the stop signal is set across threads via `loop.call_soon_threadsafe(stop_ev.set)`; the coroutine closes the browser and the GUI returns to the disconnected state.

### Core Mechanism: Real-Browser Interception

Every platform uses the same approach: launch Chromium in headed mode (with automation flags hidden and a regular user agent), open the live room page, and listen to network events:

```python
page.on('websocket', on_ws)        # fired for every WebSocket the page opens
ws.on('framereceived', on_frame)   # every raw frame received
page.on('response', on_response)   # WeChat Channels only: HTTP responses
```

Signatures, encrypted parameters, heartbeats and authentication are all handled by the platform's own front-end code; this program only "listens in", so no connection protocol needs to be reverse-engineered. The trade-off is running a full browser for every connection.

### Per-Platform Parsing

| Platform | Page opened | WebSocket filter | Data format and parsing |
|----------|-------------|------------------|-------------------------|
| Douyin | `live.douyin.com/{id}` | URL contains `webcast` | Protobuf outer frame (payload may be gzipped), dispatched by `method`: `WebcastChatMessage` / `WebcastGiftMessage` / `WebcastLikeMessage` / `WebcastMemberMessage` |
| Xiaohongshu | `xiaohongshu.com/livestream/{id}` | URL contains `xiaohongshu` | JSON -> `data.items[].b` (Base64) -> inner JSON `customData`, dispatched by `type`: `text` / `gift_comment` / `audience_join_v2` / `praise` / `follow_emcee` |
| Bilibili | `live.bilibili.com/{id}` | URL contains `chat.bilibili.com` and `/sub` | 16-byte header (total length/header length/ver/op/seq), multiple packets per frame, only op=5 handled; ver=2 is zlib and ver=3 is brotli, decompressed and parsed recursively; dispatched by `cmd`: `DANMU_MSG` / `SEND_GIFT` / `SUPER_CHAT_MESSAGE` / `INTERACT_WORD(_V2)` / `GUARD_BUY` |
| Kuaishou | `live.kuaishou.com/u/{id}` | URL contains `kuaishou` or `live-ws` | SocketMessage Protobuf (field 1=payloadType, 2=compression, 3=payload). `310` is the feed push: field 5 comments, 9 gifts, 8 likes, 12 shares; `340` is the online viewer list: the first list is the baseline and newly appearing users are treated as joins; `300` is the enter-room ack |
| WeChat Channels | `channels.weixin.qq.com/platform/live/liveBuild` | URL contains `weixin` / `channels` / `finder`; HTTP JSON responses from these domains are also monitored | The streamer must scan a QR code to log in to the dashboard (waits up to 5 minutes). Heuristic parsing: tries both JSON and Protobuf, looks in lists such as `comments` / `msgs` / `list`, and extracts by candidate field names like `nickname` / `content` / `giftName` |

### Output: IRC Protocol

After the game connects and sends `NICK` / `USER`, the server replies with `001` and `376` and joins the `#live` channel. Events from all platforms are broadcast in one unified format:

```
:{nick}!live@BLRMod PRIVMSG #live :{type}|{user}|{content}|{extra}
```

- `type`: `CHAT` / `GIFT` / `LIKE` / `JOIN` / `SC` / `SHARE` / `follow` / `guard`
- `extra`: additional info such as gift count
- Spaces in nicknames are replaced with `_`; line breaks are removed

This program **does not parse game commands**; commands such as `!Burgie` and `!Bell` are recognized by the game itself from `CHAT` messages.

### Known Limitations

- **Kuaishou gifts** only expose a gift ID, output as `gift_{id}` rather than the gift name.
- **Kuaishou joins** are inferred from changes in the viewer list, so they are approximate and may miss users.
- **WeChat Channels** parsing relies on guessing field names and may miss or misreport events.

## 📄 License

This project is a modification of HeyNau Games' BLR 1.02, for educational and learning purposes only.

## 🤖 About This Project

This project only makes 4 modifications to the original BLR 1.02 plugin (adding Kuaishou platform, adding WeChat Channels platform, Chinese localization, and browser display), plus PyInstaller packaging configuration and this document, all completed with the assistance of **WorkBuddy AI**.

- Kuaishou platform danmaku parsing: WorkBuddy AI
- WeChat Channels platform danmaku parsing: WorkBuddy AI
- Chinese localization: WorkBuddy AI
- Browser display feature: WorkBuddy AI
- PyInstaller packaging config & exe generation: WorkBuddy AI
- This README.md was written by: WorkBuddy AI

To learn more about WorkBuddy, visit [codebuddy.cn](https://www.codebuddy.cn)

## 📬 Contact

- Email: wwwqsid@outlook.com
