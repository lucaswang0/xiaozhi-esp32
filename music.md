# 音乐播放功能移植与修复说明

将 `xiaozhi-esp32-music` 项目的音乐播放功能移植到本项目，并修复实机运行问题。

## 一、功能概述

- MCP 工具 `self.music.play_song`：AI 下发歌名/艺术家，设备请求音乐 API 获取下载地址
- HTTP 流式下载 + libhelix MP3 解码，边下边播
- 屏幕显示歌名/歌词，FFT 频谱动画（`LcdDisplay::UpdateSpectrum`）
- `BoxAudioCodec` 支持运行时切换采样率（22.05k/44.1k 等）
- 播放中被打断/切状态时自动停播（`Application::AddAudioData` + 状态监听钩子）

## 二、主要改动文件

| 文件 | 内容 |
|------|------|
| `main/boards/common/music.h` | 音乐 API 协议（搜索/获取 URL/歌词） |
| `main/boards/common/esp32_music.h/.cc` | 核心播放器：下载线程、MP3 解码线程、播放控制 |
| `main/audio/codecs/audio_codec.h/.cc` | 基类新增 `SetOutputSampleRate()` 虚接口 |
| `main/audio/codecs/box_audio_codec.*` | 覆写采样率切换（I2S 重配置） |
| `main/display/display.h` | 基类新增 4 个歌词/频谱空实现虚方法 |
| `main/display/lcd_display.cc` | FFT 频谱渲染（dl_fft 1024 点） |
| `main/audio/audio_service.h` | 新增 `UpdateOutputTimestamp()` |
| `main/application.cc/.h` | `AddAudioData()` 向音乐模块喂音频、状态监听停播钩子 |
| `main/mcp_server.cc` | 注册 `self.music.play_song` / `self.music.stop` 工具 |
| `main/CMakeLists.txt` | 注册 esp32_music.cc 源文件 |
| `main/idf_component.yml` | 新增 `chmorgan/esp-libhelix-mp3` 依赖 |

## 三、编译问题修复（IDF 6.0.3）

1. **sdkconfig.defaults 不可见符号**：删除 `NEWLIB_NANO_FORMAT`（IDF6 改名且本项目用 picolibc）、`LV_USE_CALENDAR_HEADER_*`（父符号被禁用）。
2. **组件管理器 Kconfig 检查**：在 `main/Kconfig.projbuild` 末尾添加 `ESP_VIDEO_USE_CUSTOMIZED_ESP_H264_VERSION`（bool, default n）。IDF6 组件管理器解析 registry 上 esp_video 2.5.0 的 `$CONFIG{}` 引用时要求该符号在 sdkconfig.json 中有定义，否则 exit 10 → 两轮重试后 FATAL。
3. **mbedTLS 4.x 移除 `mbedtls/sha256.h`**：`esp32_music.cc` 歌名哈希改用 PSA crypto API `psa_hash_compute(PSA_ALG_SHA_256, ...)`（IDF 已自动初始化 PSA）。

## 四、运行时崩溃修复（实机"播放时自动重启"）

1. **FFT 缓冲越界写（堆损坏）**：原实现首次播放时按当帧样本数 malloc 一次，换歌后帧长变化（如 1152→2304 样本）导致 `memcpy` 越界。改为固定 `FFT_BUFFER_SAMPLES=1152`，构造时分配、析构释放，写入截断补零。
2. **播放线程栈溢出**：栈 8KB 太小（`pcm_buffer[2304]` 占 4.6KB 在栈上），提升到 16KB。
3. **StopStreaming 中 detach 线程**：原逻辑用 `joinable()` 判线程结束（恒为 true），实际每次超时后 detach，旧线程与新播放线程并发使用同一 MP3 解码器导致堆损坏。改为无条件 `join()`。

## 五、自动连播（AI 续播方案）

- 播放线程自然播完（非打断）时，通过 MCP 发送通知：

```json
{"jsonrpc":"2.0","method":"notifications/music_playback_finished",
 "params":{"song_name":"...","artist":"..."}}
```

- **后端需处理该通知**并主动发起下一首播放；后端不处理则设备不会自动续播。
- 备选方案（未实现）：MCP 工具接收歌单数组、设备端自主连播，不依赖后端。

## 六、构建验证

- `python scripts/build.py espressif/esp32-s3-korvo-2-v3.0 --name esp32s3-korvo2-v3`：通过
- `python scripts/build.py bread-compact-wifi-lcd`：通过（当前 build/ 目录为该板状态）
- 宿主测试：62/67 通过，5 个 ERROR 均为 Windows 临时目录清理 `PermissionError`（环境问题，与改动无关）

## 七、待硬件验证

- 烧录后连续切歌 3~5 首验证不再重启
- 播放中打断/播报是否正常停播
- 歌名/歌词显示、频谱动画、采样率切换音质
- 连播通知链路（需后端配合）
