# TSBot 项目熟悉地图

## 1. 一句话认识项目

TSBot 是一个 TeamSpeak 音乐机器人：Vue Web 控制台负责交互，FastAPI 后端负责音乐源、队列、认证与配置，Rust `voice-service` 负责通过 FFmpeg 解码并向 TeamSpeak 发送音频。

```text
浏览器
  └─ HTTP /api
     └─ web（Vue 3）
        └─ backend（FastAPI）
           ├─ SQLite：队列、历史、管理员、会话、运行配置、加密凭据
           ├─ 网易云 API 服务 / QQ 音乐接口 / Bilibili 接口
           └─ gRPC（proto/voice.proto）
              └─ voice-service（Rust + FFmpeg + TeamSpeak）
```

关键结论：`backend/main.py` 是当前业务编排中心；音乐源适配、数据库模型、认证、运行配置和 gRPC 客户端已拆到独立模块，但大部分 HTTP 路由和播放状态仍集中在该文件。

## 2. 技术栈、构建与启动

| 部分 | 语言 / 框架 | 构建工具 | 入口 |
| --- | --- | --- | --- |
| Web | TypeScript、Vue 3、Vue Router、Tailwind CSS | npm、Vite、PostCSS | `web/src/main.ts` |
| 后端 | Python、FastAPI、SQLAlchemy、Pydantic、httpx | venv、pip、Uvicorn | `backend/main.py` 中的 `app` |
| 语音服务 | Rust、Tokio、Tonic、tsclientlib、audiopus | Cargo；`build.rs` 生成 gRPC 代码 | `voice-service/src/main.rs` |
| 接口契约 | Protocol Buffers / gRPC | Python `grpcio-tools`、Rust `tonic-build` | `proto/voice.proto` |
| 部署 | Shell、PowerShell、Docker Compose、Nginx | Make、Docker | 根目录启动脚本与 Dockerfile |

常用方式：

```bash
# 一次构建三个组件
make all

# 本地开发（分别运行）
./run-voicemake.sh
backend/.venv/bin/uvicorn backend.main:app --reload --host 127.0.0.1 --port 8009
npm --prefix web run dev

# 单机后台运行
./nohup-start.sh

# Docker
docker compose up -d --build
```

默认端口：Web 开发 `5173`、Web 生产 `8080`、后端 `8009`、Voice gRPC `50051`。

## 3. 入口、配置与测试

| 路径 | 职责 |
| --- | --- |
| `web/src/main.ts` | 初始化主题和品牌信息，挂载 Vue 应用与路由。 |
| `web/src/router.ts` | 页面路由及管理员登录、首次改密守卫。 |
| `web/src/api.ts` | 统一 HTTP 请求、Cookie、API Token、CSRF 和错误解析。 |
| `web/src/views/` | 搜索、歌单、喜欢、收藏、队列、历史、歌词、登录和设置页面。 |
| `web/src/components/MusicPlayer.vue` | 全局播放状态、音量、进度、音效、切歌、循环和随机控制。 |
| `backend/main.py` | FastAPI 路由、队列编排、音乐源聚合、播放状态、TeamSpeak 聊天命令。 |
| `backend/models.py` | SQLAlchemy 表：管理员、会话、设置、密钥、队列和历史。 |
| `backend/db.py` | 数据库引擎、建表和 Session 生命周期；默认使用 SQLite。 |
| `backend/runtime_config.py` | 配置定义、校验、加密持久化、应用配置及生成 Voice 共享配置。 |
| `backend/auth.py` | 管理员初始化、PBKDF2 密码、会话 Cookie、CSRF 和首次改密。 |
| `backend/netease.py` | 外部 NeteaseCloudMusicApi 的 HTTP 客户端。 |
| `backend/qqmusic.py` | QQ 音乐搜索、播放地址、歌词、歌单和登录适配。 |
| `backend/bilibili_auth.py` | Bilibili Cookie、扫码登录及 Playwright 字幕补抓。 |
| `backend/bilibili_cache.py` | Bilibili 音频缓存过期和容量淘汰。 |
| `backend/voice_client.py` | 后端到 Voice 服务的异步 gRPC 客户端和事件订阅。 |
| `voice-service/src/main.rs` | gRPC 服务、TeamSpeak 连接、FFmpeg 解码、音频发送及事件发布。 |
| `tsbot.env.example` | 启动前必须提供的后端、数据库、Voice 和 Web 环境变量模板。 |
| `web/vite.config.ts` | 开发/预览服务器及 `/api` 到后端的反向代理。 |
| `docker-compose.yml` | 三服务拓扑、端口、共享日志/数据/临时音频目录。 |
| `tests/` | Python `unittest` 回归测试，覆盖认证配置、缓存、聊天命令和播放完成逻辑。 |

最小检查：

```bash
backend/.venv/bin/python -m unittest discover -s tests
npm --prefix web run build
cargo test --manifest-path voice-service/Cargo.toml
cargo fmt --manifest-path voice-service/Cargo.toml --check
```

## 4. 核心模块依赖

```text
Web 页面 / 组件
  → web/src/api.ts
  → backend/main.py
      → auth.py + runtime_config.py + managed_assets.py
      → db.py + models.py
      → netease.py / qqmusic.py / Bilibili 适配逻辑
      → voice_client.py
          → proto/voice.proto
          → voice-service/src/main.rs
              → FFmpeg
              → TeamSpeak 服务器
```

数据边界：

- 默认 SQLite 保存 `QueueItem`、`HistoryItem`、`AdminCredential`、`AdminSession`、`AppSetting` 和 `Secret`。
- 敏感运行配置和音乐平台 Cookie 在写入数据库前加密；加密密钥来自 `TSBOT_COOKIE_KEY`。
- 管理图片默认放在 SQLite 文件同目录的 `uploads/`；Bilibili 音频缓存位于 `tmp/bilibili_audio/`。
- Voice 配置通过 `TSBOT_VOICE_CONFIG_FILE` 指向的 JSON 文件共享；文件变化时 Rust 服务自行重启。
- 播放音量和音效状态由 Voice 服务持久化到 `TSBOT_VOICE_STATE_FILE`。

## 5. 五条关键业务流程

### 5.1 跨平台搜索并点播

**入口**：`web/src/views/SearchView.vue`

**链路**：

```text
搜索页
  → 网易云 `/search` / QQ `/qqmusic/search/songs` / B站 `/bilibili/search/videos`
  → backend/main.py
  → NeteaseClient / QQMusicClient / Bilibili HTTP 适配
  → 返回统一供页面展示的搜索结果
  → POST `/queue/{netease|qqmusic|bilibili}`
  → 写入 QueueItem；play_now=true 时继续调用 VoiceClient.play
```

**存储 / 外部服务**：SQLite 队列表；NeteaseCloudMusicApi、QQ 音乐接口、Bilibili 接口；Bilibili 播放时可能生成本地音频缓存。

**返回结果**：搜索列表，或 `{ok, id, trial}` 点播结果；立即播放时 TeamSpeak 开始播音。

### 5.2 队列播放、控制与自动续播

**入口**：`web/src/components/MusicPlayer.vue`、`web/src/components/PlaylistView.vue`、`web/src/views/QueueView.vue`

**链路**：

```text
GET `/queue` / POST `/queue/{id}/play` / `/voice/*`
  → backend/main.py 队列与播放状态机
  → 根据 track_id 前缀解析最新播放资源
  → backend/voice_client.py
  → gRPC VoiceService
  → Rust 调用 FFmpeg 解码并向 TeamSpeak 发送 Opus 音频
  → Voice 通过 SubscribeEvents 回报完成/错误
  → 后端删除已完成项并自动播放下一项
```

**存储 / 外部服务**：SQLite `queue_items`、`history_items`；音乐平台播放地址；Voice 服务与 TeamSpeak。

**返回结果**：队列/播放器 JSON 状态；播放、暂停、进度、音量、音效、随机和循环状态；每次实际播放写入历史。

### 5.3 歌单、喜欢列表与歌词

**入口**：`PlaylistsView.vue`、`PlaylistDetailView.vue`、`LikesView.vue`、`LyricsView.vue`、`LyricsDisplay.vue`

**链路**：

```text
页面请求 `/netease/*` 或 `/lyrics/{queue_item_id}`
  → backend/main.py
  → NeteaseClient / QQMusicClient / Bilibili 字幕适配
  → 获取歌单、喜欢歌曲或时间轴歌词
  → 用户选择歌曲后复用 `/queue/*` 点播流程
```

**存储 / 外部服务**：管理员保存的音乐平台 Cookie；网易云、QQ 音乐、Bilibili 字幕接口；Bilibili 无公开字幕时可使用 Playwright 补抓。

**返回结果**：歌单和歌曲列表，或标准化为 `{time, text}` 的歌词时间轴。

### 5.4 管理员登录、授权与运行配置

**入口**：`LoginView.vue`、`ChangePasswordView.vue`、`SettingsView.vue`、`CookieView.vue`

**链路**：

```text
POST `/auth/login`
  → backend/auth.py 校验密码
  → 创建 HttpOnly 会话 Cookie 和 CSRF Token
  → GET/PUT `/admin/settings` 或音乐平台授权接口
  → runtime_config.py 校验并写入 AppSetting/Secret
  → 应用后端配置；Voice 配置写入共享 JSON
  → Voice 检测配置变化并重启
```

**存储 / 外部服务**：SQLite 管理员、会话、设置和加密 Cookie；Voice 配置 JSON；网易云、QQ 音乐、Bilibili 扫码/账号接口。

**返回结果**：认证状态、配置字段、待应用状态、Voice 重启标记和配置版本。

### 5.5 TeamSpeak 聊天指令点歌

**入口**：TeamSpeak 频道或私聊中的中英文命令，如 `搜索`、`歌单`、`选择`、`播放`、`暂停`、`下一首`。

**链路**：

```text
TeamSpeak ChatEvent
  → voice-service SubscribeEvents
  → backend/voice_client.py
  → backend/main.py::_chat_command_worker
  → _handle_chat_command
  → 搜索、入队或播放控制共用现有后端函数
  → VoiceClient.send_notice
  → TeamSpeak 聊天回复
```

**存储 / 外部服务**：SQLite 队列和历史；网易云服务；歌单搜索结果按用户保存在后端内存中，5 分钟过期。

**返回结果**：TeamSpeak 文本提示、队列变化和音频播放。

## 6. 待确认

- `web/src/components/PlaylistView.vue` 调用 `POST /queue/reorder`，但扫描后未找到对应后端路由；队列拖拽排序是否可用需要实际验证。
- `voice-service/CMakeLists.txt` 和 `voice-service/src/main.cpp` 仍保留 C++ 实现，但 Make、Docker 和常用脚本默认构建 Rust 版本；C++ 路径是否继续维护待确认。
- 文档声明 Python 3.8+，Docker 使用 Python 3.11，当前没有发现覆盖多个 Python 版本的 CI；实际最低兼容版本待确认。
- GitHub 工作流主要构建镜像和发布包，未发现自动执行现有 Python/Rust 测试的步骤；合并门禁策略待确认。
