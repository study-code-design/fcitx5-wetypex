# AGENTS.md

给在本仓库工作的 AI 代理／自动化脚本用的索引。人类贡献流程以
[CONTRIBUTING.md](CONTRIBUTING.md) 为准，产品定位见 [README.md](README.md)，运行时组件与协议边界见
[docs/architecture.md](docs/architecture.md)，能力是否真的可用以
[docs/feature-status.md](docs/feature-status.md) 为准。

一句话：WeTypeX 是 Fcitx5 输入法插件，把**固定版本**的 macOS 微信输入法 2.2.3.657 输入核心
放进一个 Mach-O ABI 兼容宿主里运行，Linux 侧负责输入上下文、沙箱、音频、剪贴板与界面。

## 硬约束

1. **不引入、不生成、不提交**原版安装包、二进制、词典、模型、原图、账户数据、传输码、剪贴板内容或私人文本
   （包括日志、截图、测试夹具、commit message）。
2. **不伪造成功**：候选、登录、配对、同步、语音、AI、隔空传送都必须来自原版服务或真实状态；
   失败要如实返回错误，不允许静默返回成功。
3. **版本绑定**：结构体尺寸/字段偏移、固定地址、被解析的导入符号名都绑定 2.2.3.657。
   改 `src/backend/` 必须说明适用版本、兼容范围与失败边界。
4. **权限**：账户、语音、剪贴板状态用 `0700` 目录 + `0600` 文件；bwrap 沙箱只给对应状态目录写权限。
5. **语言**：用户可见文案与 `docs/` 用简体中文；代码注释与 commit message 用英文
   （commit 正文写「最终行为 + 适用环境 + 已执行的检查 + 遗留限制」）。
6. `scripts/*.sh.in` 是 CMake 模板，只能出现 `@CMAKE_INSTALL_FULL_BINDIR@` 这类占位符，不能写死路径。
   `nix/*.patch` 打到同一批文件，改完必须仍能 `git apply --check nix/runtime.patch`。

## 构建与检查

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release      # 默认优先 Qt6；-DWETYPE_QT_VERSION=5 走 Qt5
cmake --build build --parallel
# 脚本自检（syncd/update/sync 是 Python 载荷，不能过 sh -n）
for s in scripts/fcitx5-wetypex-{backend,account,setup,ai,voice}.sh.in; do sh -n "$s"; done
python3 -m py_compile scripts/setup_runtime.py scripts/voice_stream.py tools/*.py \
  build/fcitx5-wetypex-syncd build/fcitx5-wetypex-update build/fcitx5-wetypex-sync
```

构建依赖要点：CMake ≥ 3.16、Fcitx5（≥5.1 用 C++20，否则 C++17）、LibIME、json-c、fontconfig、
wayland-client、Qt5.12+/Qt6、**clang++**（生成原版 libc++ ABI 用的 `locale.so`）、Boost 头文件
（LibIME 的 `segmentgraph.h` 需要，包名在 Arch 上是 `boost`）。libkqueue 由 `FetchContent` 在构建时
从 GitHub 拉取（离线构建需预先提供），产物安装为 `libkqueue.so` 供宿主使用。

只改文档/脚本时不必全量构建；改 C++ 至少编译受影响的 target。运行时自检见文末「调试速查」。

如果环境缺 Boost 头文件（`libime/core/segmentgraph.h` 依赖）导致 `wetype.cpp` 无法编译，仍可先用已安装的
头文件对不依赖 LibIME 的 TU 做单文件语法检查（需补上 CMake 里的宏，否则会停在未定义的 `WETYPE_*`）：

```bash
g++ -fsyntax-only -std=c++17 -fPIC \
  -DWETYPE_ACCOUNT_TOOL='"/usr/bin/fcitx5-wetypex-account"' \
  -DWETYPE_BACKEND='"/usr/bin/fcitx5-wetypex-backend"' \
  -DWETYPE_TRANSFER='"/usr/bin/fcitx5-wetypex-transfer"' \
  -DWETYPE_VERSION='"dev"' \
  $(pkg-config --cflags Qt6Widgets Qt6DBus Qt6Network Qt6Svg Fcitx5Utils) \
  -I/usr/include/Fcitx5Qt6 -I/usr/include/Fcitx5Qt6/Fcitx5QtWidgetsAddons \
  src/settings/main.cpp
# 纯头文件（不需要 LibIME）用 C++20 单独 include 也可以查
g++ -fsyntax-only -std=c++20 -fPIC -I/usr/include/Fcitx5/Config \
  -I/usr/include/Fcitx5/Utils $(pkg-config --cflags Fcitx5Config) -x c++ - \
  <<< '#include "src/plugin/config.hpp"'
```

## 文件索引

### 顶层与构建

| 路径 | 内容 | 什么时候改 |
| --- | --- | --- |
| `CMakeLists.txt` | 所有 target、安装布局、`WETYPE_*` 宏（可执行文件路径、数据文件路径）、`configure_file` 生成 `scripts/*.sh.in`、可选的 Qt5/Qt6 双套 config-ui、BundledCurl/OpenSSL | 新增 target、新增编译宏、新增安装文件 |
| `cmake/BundledCurl.cmake`、`cmake/BundledOpenSSL.cmake` | 旧发行版（Ubuntu 20.04）静态链接 curl/OpenSSL 的辅助逻辑 | 只在兼容旧 glibc/OpenSSL 时动 |
| `data/` | `wetypex-addon.conf.in`（Fcitx5 addon 描述）、`wetypex-inputmethod.conf`、`wetypex-symbols.json`（符号面板数据）、`wechat-emoticons.json`（表情别名表）、`themes/wetypex-{dark,light}`（Classic UI 主题）、`icons/`、`hyprland.lua` | 改候选主题、符号表、表情别名、默认设置入口 |
| `docs/` | 见下方表格 | 行为变化必须同步 |
| `nix/` | `flake` 输入 + `package.nix` + 三个 patch：`runtime.patch`（Nix 运行时路径/关闭自动更新）、`feature-options.patch`（编译开关）、`preserve-desktop-theme.patch`（不覆盖桌面主题） | 改了被 patch 命中的文件（`wetype.cpp`、`config.hpp`、`settings/main.cpp`、`scripts/*.sh.in`）后必须重跑 `git apply --check` |
| `packaging/` | `PKGBUILD.in`（Arch）、`nfpm.yaml`（deb/rpm 元数据）、`docker/*.Dockerfile`（各基线镜像）、`stage.sh`/`build-*.sh`/`container-stage.sh`、`aur/` | 改依赖、版本、安装步骤 |
| `tools/prepare_macos_runtime.py` | 用标准库解析并重定位固定版本 Mach-O、生成固定地址映像与 `manifest.txt` | 改运行时提取/重定位逻辑 |
| `tools/package_release.py` | 生成源码 release 压缩包与固定的 Arch 构建配方 | 改发布产物 |
| `.github/workflows/*` | `build.yml`（构建矩阵）、`release.yml`（发布） | 改 CI |

### `src/plugin/` — Fcitx5 侧（用户输入的直接来源）

| 路径 | 内容 | 什么时候改 |
| --- | --- | --- |
| `wetype.cpp`（≈3.4k 行，本仓库主体） | `WeType : InputMethodEngine`：`keyEvent()`/`receive()` 驱动后端；`State`（每个 InputContext 的会话、epoch、preedit、候选、符号自动转换、V 模式、语音状态）；`Word`/`VMenuWord` 候选；`start()`/`flush()`/`send()` 与后端的私有管道；`reloadSettings()`/`saveNativeConfig()`（`wetypex.json`）；`applyAppearance()`（写 classicui.conf）；`applyDeviceFunctions()`（功能位）；`recordClipboard()`；语音段（`voiceControl`/`startVoice`/`stopVoice`/`cancelVoice`/`receiveVoice`/`primeVoice`）；V 模式菜单与工具栏（`startVModeWindow`/`receiveVModeAction`/`pollVModeWindow`）；`panel()`（预编辑/候选/辅助文本渲染）；光标与成对符号、数字标点纠正的延迟处理 | 几乎所有输入行为改动 |
| `config.hpp` | `FCITX_CONFIGURATION` 组：`InputConfig`（输入方式/五笔/双拼/模糊音/表情）、`VoiceConfig`、`PhraseClipboardConfig`、`AppearanceConfig`、`ShortcutConfig`（Fcitx `Key`/`KeyList`）、`DevicesConfig`、`UpdateConfig`、`WeTypeConfig`；`VoicePunctuationMode` 等枚举 | 新增/改设置项（Fcitx5 配置页与设置窗口共用） |
| `wechat_emoji.hpp` | 微信表情字体可用性检测（fontconfig）、候选原图/别名替换入口 | 改微信表情候选 |

### `src/backend/` — 原版兼容宿主（Darwin ABI）

| 路径 | 内容 | 什么时候改 |
| --- | --- | --- |
| `macos_host.cpp`（≈3.3k 行） | 固定地址加载器、导入符号解析与 clang/libc++/kqueue/CoreFoundation 轻量 shim、`argv[2]` 模式分发（见「宿主模式」）、`module-map`/`module-run`/`validate`/`network-info` 等诊断入口 | 新增原版 API 覆盖、改加载/沙箱/模式 |
| `service.hpp`（≈790 行） | 输入服务主循环：`ServiceTransport`（管道收发）、`ServiceSession`、候选推送/快照 `service_snapshot()`、云端候选 `service_listener()`、选词 `service_select()`、热词 `hotword_collect()` | 改输入数据流、快照字段、候选项 |
| `control.hpp` | `ServiceTransport`：一行一个 JSON 的收发、peer 校验、非阻塞写与 fd 管理 | 改管道传输细节 |
| `business_services.hpp`（≈1.0k 行） | 业务服务：`network_login_service()`、身份持久化、protobuf 编解码（`protobuf_*`、`protobuf_read_varint`）、设备组/剪贴板/隔空传送请求（`WETYPE_GROUP_INFO`、`WETYPE_SEND_CLIPBOARD*`、`WETYPE_SET_FUNCTIONS` 等）、JSON 报告输出 | 改账户/同步/传输协议字段 |
| `native_voice.hpp`（≈160 行） | `voice_engine_*` C ABI 调用封套：`start`/`pcm`（hex→int16）/`stop`/`finish`/`cancel`/`reset`、`init_vad` 与事件回调（`partial`/`final`/`state`/`error`/`vad`/`polishing`） | 改语音引擎交互 |
| `wcwss_bridge.cpp` | 把原版 WebSocket/TLS 调用桥接到 Linux curl（由 `CMakeLists.txt` 编译成 `wcwss_bridge.so`） | 改网络桥接 |
| `darwin_locale.cpp` | 用 clang++ 编译的 libc++ locale facet shim（`locale.so`） | 改区域设置兼容 |

### 其它可执行程序

| 路径 | 内容 | 什么时候改 |
| --- | --- | --- |
| `src/settings/main.cpp`（≈1.5k 行） | 独立设置窗口：`page()` 分页（输入/语音输入/常用语和剪贴板/外观/快捷键/跨设备/手机版/关于）、`PairingOverlay`（六位匹配码轮询）、麦克风枚举（`pw-dump`/`pactl`）、`writeSetting()` 直接写 `wetypex.json` | 改设置界面 |
| `src/settings/original_windows.hpp` | 通用控件：`card()`/`separator()`/`row()`/`iconRow()`/`radioRow()`/`shortcutRow()`/`page()` 等 | 改控件样式（会影响所有页面） |
| `src/ai/main.cpp` | `fcitx5-wetypex-ai-viewer`：用 surrounding text 提问、渲染返回的 HTML/Markdown、滚动与清理 | 改 AI 窗口 |
| `src/transfer/main.cpp`（≈1.2k 行） | `TransferWindow`：传送码/二维码、Flurry LAN 与 WXP2P 会话、进度、落盘 | 改隔空传送 |
| `src/vmode.cpp` | V 工具栏窗口：layer-shell（不可用时回退 Fcitx 菜单）、Hyprland 定位、按钮动作回传 | 改 V 模式工具栏 |
| `src/clipboard/main.cpp` | `fcitx5-wetypex-clipboard-set`：stdin→剪贴板、`--read-text`/`--read-image`/`--image`/`--normalize-image`/`--notify-ready`（长期持有选择）/`--wayland-clipboard-protocols`（只查 registry） | 改剪贴板读写 |
| `src/configui/main.cpp` + `src/configui/wetypexconfig.json` | Fcitx5 配置页里的入口页：只给一段说明和一个「打开 WeTypeX 完整设置」按钮，实际选项仍由 Fcitx5 配置页保存 | 改配置页入口文案 |

### `src/common/`

| 路径 | 内容 |
| --- | --- |
| `json.hpp` | `wire::` 极薄 json-c 封装：`object()`/`parse()`/`dump()`/`put()`/`get()`/`str()`/`number()`/`boolean()`，插件与宿主共用 |
| `qt_helpers.hpp` | Qt 侧通用：`initializeHighDpi()`、`stopProcess()`/`onProcessDone()`、`jsonInteger()`、`settingBool()`、`mousePosition()`/`mouseGlobalPosition()`、`localRequest()`（本地 socket 请求） |
| `wechat_emoticons.hpp` | `wetype::WeChatEmoticons`：加载 `wechat-emoticons.json`（`version`/`font_family`/`entries[].commit,codepoint,aliases`）、彩色字体族名 `WeTypeX WeChat Emoticons v1`、文字别名与码位的映射 |

### `scripts/` — 安装到 `bindir`/`datadir` 的运行时脚本

| 路径 | 内容 |
| --- | --- |
| `fcitx5-wetypex-backend.sh.in` | 在 bwrap 中启动持久输入服务（`wetypex-engine-host ... serve`），绑定运行时/状态/账户目录 |
| `fcitx5-wetypex-account.sh.in`（≈300 行） | 账户工具：子命令 → `WETYPE_*` 环境变量 → 沙箱内 `business-service`，把结果规整成 JSON（见「账户工具子命令」） |
| `fcitx5-wetypex-syncd.sh.in`（≈460 行） | 剪贴板同步守护进程：`group-info` 轮询、本地基线、`wl-copy`/`wl-paste` 与 Qt 静默回退、唯一写入端与版本游标 |
| `fcitx5-wetypex-sync.sh.in` | 手动/`--refresh-state` 拉取设备组状态 |
| `fcitx5-wetypex-voice.sh.in` | `exec python3 .../voice_stream.py "$@"` 的薄包装 |
| `voice_stream.py`（≈830 行） | 语音守护进程：Unix socket 控制（`start/stop/finish/cancel/status/transcribe/shutdown`）、PipeWire/PulseAudio 采集、PCM 分帧、原版宿主会话、`voice-inbox.json` 投递、`copy_to_clipboard()` |
| `fcitx5-wetypex-ai.sh.in` | 调原版 AI 服务并 `exec fcitx5-wetypex-ai-viewer --ask` |
| `fcitx5-wetypex-setup.sh.in` | 转调 `setup_runtime.py` |
| `setup_runtime.py`（≈360 行） | 校验官方压缩包、提取核心/词典/VAD、生成运行时与图标（`--archive`/`--icons-only`/`--check`） |
| `wechat_emoticons.py` | 用原版原图生成本地彩色字体 |
| `fcitx5-wetypex-update.sh.in`（≈250 行） | 按发行版下载并校验 GitHub Release，调用授权安装 |

### `docs/`

| 文件 | 内容 |
| --- | --- |
| `architecture.md` | 组件职责、输入路径、ABI 边界、网络/同步/语音/AI/传送、沙箱与权限 —— 改架构先看这里 |
| `configuration.md` | `wetypex.json` 各字段、语音项、快捷键默认值 |
| `feature-status.md` | 每项能力的真实状态（可用/预览/不提供），对外承诺的唯一依据 |
| `voice-input.md` | 语音操作、原版 VAD 资源、设备与诊断、会话与资源管理 |
| `troubleshooting.md` | 语音无结果、终端吞字（含 VS Code 的 XWayland 方案）、失焦、首帧面板、Electron 定位、V 工具栏、AI 窗口等 |
| `input-handling.md`、`date-time-input.md`、`wechat-emoticons.md`、`hyprland.md`、`distribution.md` | 按键与模式、日期时间候选、微信表情、Hyprland 兼容、打包与分发 |

## 跨文件契约（改一处要成对改）

### 1. 配置：`${XDG_CONFIG_HOME:-~/.config}/fcitx5/wetypex.json`

四个读写方必须一致：

- 插件 `wetype.cpp`：`reloadSettings()` 读、`saveNativeConfig()` 写（同时刷新 `config_` 里的 Fcitx Option）。
- 设置窗口 `src/settings/main.cpp`：`writeSetting()` 直接写 JSON。
- Fcitx5 配置页：走 `setConfig()` → `saveNativeConfig()`。
- 运行时脚本：`voice_stream.py`（`voice_microphone`/`voice_punctuation`/`voice_smart_polish`/`voice_copy_to_clipboard`）、`fcitx5-wetypex-sync.sh.in`（`standalone`）、`syncd`（功能位）。

新增设置项的最小改动集：`config.hpp` 加 Option → `reloadSettings()` 读 → `saveNativeConfig()` 写 →
设置窗口加控件（`row(...)` + `toggle(...)`/`QComboBox`）→ 文档（`configuration.md`）。
消费该键的脚本/守护进程要按缺省值兜底，旧用户没有这个键。

### 2. 插件 ↔ 宿主的私有管道（一行一个 JSON）

- 插件 → 宿主：`open`、`key`、`raw`、`select`、`move`、`backspace`、`delete`、`punctuation`、
  `hotword_list`、`poll`、`vmode`、`reset`、`close`。
- 宿主 → 插件：`event: ready`、更新快照（`session`/`epoch`/`seq`/`revision`/`update_id`/
  `commits`/`preedit`/`display_preedit`/`display_cursor`/`cursor_stops`/`cursor_commit*`/候选数组）、
  `hotwords`/`words`、`candidate_updates`（能力协商：支持推送就不再轮询云候选）。
- 语义：每个 `InputContext` 有独立会话 + epoch；旧 `seq`/`epoch`/重复 `update_id` 一律丢弃；
  提交只排出一次；密码上下文（`CapabilityFlag::Password`）直接绕过后端。
- 改动快照字段要同时改 `service.hpp` 的 `service_snapshot()` 与插件 `receive()`，并考虑旧插件/旧核心
  的兼容（`candidate_updates` 就是为此存在）。

### 3. 状态目录（`WETYPE_STATE_DIR`，默认 `~/.local/share/fcitx5-wetypex/state`）

| 路径 | 写入方 | 用途 |
| --- | --- | --- |
| `voice-inbox.json` | `voice_stream.py` | 语音状态投递（`token`/`session`/`epoch`/`state`/`text`/`level`/`timings`），插件按 `version` 单调读取 |
| `voice/control.json` | `voice_stream.py` | 控制 socket 地址与 pid，插件据此优先走 socket、失败才起进程 |
| `voice/{capture,backend,daemon}.log` | 语音相关进程 | 采集 stderr、原版宿主日志、守护进程日志 |
| `sync-state.json` | `syncd` | 最近一次 `group-info` 报告（含 `func_switch`） |
| `sync-runtime.json` | `syncd` | 本地基线摘要、远端版本游标、`clipboard_enabled` |
| `clipboard-history.json` | 插件 | V 模式/常用语剪贴板历史（≤20 条） |
| `vmode-action.json`、`vmode.log` | `vmode.cpp` | 工具栏动作回传与定位日志 |
| `vmode.lock`、`syncd.lock`、`backend.lock`、`voice/daemon.lock` | 各自进程 | `flock` 单实例守卫 |
| `user/`、`userDict/` | 原版核心 | 核心自己的用户数据与词库 |

同级目录（都在 `~/.local/share/fcitx5-wetypex/` 下）：`account/`（账户身份与业务缓存）、
`runtime/`（已准备的原版运行时 + `manifest.txt`）、`ui/`（从官方包提取的图标与资源）、
`voice/vad/`（VAD 配置与模型）。

原子写：临时文件 + `rename` + `chmod 0600`。插件读 `voice-inbox.json` 时先比 `token`、再比 `version`，
避免上一个会话的结果串到本次（中间结果与最终结果都会经过这个文件）。

### 4. 宿主模式（`wetypex-engine-host <runtime-dir> <mode>`）

`serve`（输入，`WETYPE_NETWORK_LIVE` 决定联网）、`voice-service`、`business-service`、`llm-service`、
`version`、`validate`、`network-info`、`module-map`、`module-run`、`wxp2p-probe`、`flurry-server`、
`flurry-wxp2p-server`、`flurry-credentials`、`flurry-loopback`。新增模式同步改
`scripts/fcitx5-wetypex-*.sh.in` 的 bwrap 参数与 `docs/architecture.md`。

### 5. 账户工具子命令（`fcitx5-wetypex-account`）

`pairing-code`、`pairing-status <6位码>`、`confirm-bind <bind_id>`、`pending-file-invite`、
`group-info [version]`、`set-functions <group> <mask>`、`send-clipboard <group> <text>`、
`send-image <group> <png>`、`p2p-code`/`p2p-init`/`p2p-peer`/`p2p-update`、`voice-file <f>`（转调
`fcitx5-wetypex-voice transcribe`）、`network-info`、`sync-now`。输出统一为 JSON：
`{"ok": bool, ..., "error": transport_error, "service_error": ...}`。

### 6. 语音 CLI 与结果链路

`fcitx5-wetypex-voice {start|stop|finish|cancel|status|serve|transcribe|shutdown}`，`transcribe` 支持
`--realtime --verbose`。链路：插件 `startVoice()` → socket/进程 → `voice_stream.py` 采集 16 kHz 单声道
s16 → 宿主 `voice-service`（原版 Opus/VAD/识别）→ `partial`/`final` 事件 → `voice-inbox.json` →
插件 `receiveVoice()`（校验 token/session/epoch/焦点）→ 预编辑 → `commitString()`，并按
`voice_copy_to_clipboard` 写系统剪贴板（`wl-copy` 优先，回退 `fcitx5-wetypex-clipboard-set`）。
文件转写（`session == 0`）不写剪贴板。

### 7. 剪贴板写入者

系统剪贴板只有两个写入方：`syncd`（设备组同步，且必须 `func_switch & 1`）和语音结果复制。
其它代码只读（插件 `recordClipboard()` 通过 Fcitx clipboard addon 读）。新增写入方要先想清楚
版本游标与基线，否则会把旧内容回传。

### 8. 设备组功能位

`0x1` 跨设备剪贴板、`0x2` 常用语同步、`0x4` 个人词库同步。插件 `applyDeviceFunctions()` 用本地
`device_*_sync` 选项覆盖低 3 位、保留其余位后调用 `set-functions`；`syncd` 只认服务端返回的
`func_switch`。

## 常见改动配方

**加一个设置项**：`config.hpp` → `reloadSettings()` → `saveNativeConfig()` → `settings/main.cpp`
的对应 `page()` 段（用 `row()`/`toggle()`/`QComboBox`）→ `docs/configuration.md`。若由脚本消费，
在脚本里用缺省值兜底；若涉及 Fcitx `Key`，同时看 `configui/`。

**加语音行为**：`wetype.cpp` 语音段（状态机 + `panel()` 呈现）→ `voice_stream.py`（采集/事件/投递）
→ `native_voice.hpp`（若需新引擎调用）→ `docs/voice-input.md` 与 `docs/configuration.md`。

**加按键/候选行为**：`keyEvent()` 决定是否消费 → `send(ic, "key", ...)` → 宿主 `service.hpp` 处理 →
`receive()` 消费快照（`commits`/`preedit`/候选）。注意 `seq`/`epoch`/`revision` 的丢弃规则，以及
Wayland-v2 客户端需要延后的光标移动（`forwardCursor()`/`resolveCursorMove()`）。

**加界面**：设置窗口在 `settings/main.cpp` 的 `page(...)` 段，控件优先复用
`original_windows.hpp` 的 `row()`/`card()`/`separator()`；所有字符串为简体中文。

**动打包**：`packaging/PKGBUILD.in`、`packaging/nfpm.yaml`、`nix/package.nix` 与 `CMakeLists.txt`
的 `install()` 保持一致；动被 patch 的文件后跑 `git apply --check nix/*.patch`。

## 调试速查

```bash
# 某个窗口到底连的哪条输入法通道（先让它获得焦点）
dbus-send --session --print-reply=literal --dest=org.fcitx.Fcitx5 /controller \
  org.fcitx.Fcitx.Controller1.DebugInfo | grep -E "program:|Group"
# 语音
fcitx5-wetypex-voice status        # state/capture_backend/vad/timings/native_ready
fcitx5-wetypex-voice shutdown      # 升级脚本后结束旧控制服务
# 同步
fcitx5-wetypex-account group-info  # func_switch、clipboard_version
# 重新加载插件
fcitx5-remote -r                   # 或 fcitx5 -r 重启输入法
```

日志位置：`state/voice/{daemon,backend,capture}.log`、`state/vmode.log`、`state/last-ai.log`、
`account/last-network.log`（最近一次账户/业务请求的宿主 stderr）。分享日志前必须脱敏 UIN、
设备码、传输码与识别文本。
