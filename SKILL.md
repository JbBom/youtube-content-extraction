---
name: youtube-content-extraction
description: Extract content from YouTube URLs by checking captions first, downloading permitted audio when captions are unavailable, transcribing locally with Whisper, validating the output, and producing timestamped text plus a concise summary. Use when the user asks to download YouTube audio, extract a transcript, transcribe a video without subtitles, or summarize a YouTube video from its URL.
---

# YouTube 内容提取

按“字幕优先、无字幕再转音频”的顺序处理 YouTube 视频。只处理用户有权访问和使用的内容；不绕过 DRM、付费限制、登录限制或验证码，不自动读取浏览器 Cookie。

## 工作流

### 1. 确认输入和输出范围

- 从用户提供的 URL 中保留完整视频地址；不要误把播放列表作为单个视频。
- 如果用户未指定输出目录，使用当前工作区；保留用户已有文件和原始音频，不删除中间文件，除非用户明确要求。
- 默认输出：`m4a` 音频、`txt` 纯文字、`srt` 带时间戳字幕；如用户还需要画面/PPT文字，明确说明仅凭音频无法提取视觉内容。

### 2. 只读检查工具

先检查工具是否存在：

```bash
command -v ffmpeg
python3 -m yt_dlp --version
python3 -m whisper --help
```

如果工具缺失，先报告缺失项和预计安装/下载量，等待用户明确授权后再安装。不要为了完成任务静默执行系统级安装。

### 3. 优先检查并获取字幕

先查看可用字幕：

```bash
python3 -m yt_dlp --list-subs "<YouTube URL>"
```

如果存在人工字幕或自动字幕，优先只下载字幕：

```bash
python3 -m yt_dlp --skip-download \
  --write-subs --write-auto-subs \
  --sub-langs "zh.*,en.*" \
  --sub-format "vtt/best" \
  "<YouTube URL>"
```

人工字幕优先于自动字幕；如果两者都生成，保留来源并在报告中标明。字幕不存在、下载失败或明显不完整时，转入音频转录。

### 4. 无字幕时下载音频

使用 `yt-dlp` 提取音频，并保留标题和视频 ID 以避免重名：

```bash
python3 -m yt_dlp -x --audio-format m4a --audio-quality 0 \
  -o "%(title)s [%(id)s].%(ext)s" \
  "<YouTube URL>"
```

如果出现“page needs to be reloaded”、SABR 或网页客户端格式缺失等错误，在不读取 Cookie 的前提下重试 Android 客户端：

```bash
python3 -m yt_dlp \
  --extractor-args "youtube:player_client=android" \
  -x --audio-format m4a --audio-quality 0 \
  -o "%(title)s [%(id)s].%(ext)s" \
  "<YouTube URL>"
```

如果备用客户端要求 PO Token、登录或 Cookie，不要自行获取；报告阻塞原因并请求用户决定。

### 5. 验证音频

下载后用 `ffprobe` 检查文件存在、格式和时长：

```bash
ffprobe -v error \
  -show_entries format=filename,duration,size,format_name \
  -of default=noprint_wrappers=1 \
  "<audio file>"
```

只有验证成功后才报告音频已完成。记录绝对路径、时长和大小。

### 6. 用 Whisper 本地转录

将转录结果放入独立目录，不覆盖音频：

```bash
python3 -m whisper "<audio file>" \
  --model turbo \
  --output_format all \
  --output_dir transcript
```

中文音频可以增加 `--language Chinese`；不确定语言时省略，让 Whisper 自动检测。`turbo` 首次运行可能下载约 1.5 GB 模型；CPU 环境会使用 FP32，耗时可能较长。若用户明确要求节省下载量，可改用 `small` 或 `base`，但要说明准确率可能下降。

### 7. 验证并交付

检查生成的 `txt`、`srt`、`vtt` 文件和非空行数：

```bash
ls -lh transcript
wc -l transcript/*
```

交付时区分：

- 已下载：给出音频的绝对路径和可点击链接。
- 已转录：给出 TXT 和 SRT/VTT 的绝对路径和可点击链接。
- 已摘要：根据当前转录生成简洁的主题、要点、操作步骤和风险。
- 未验收：如果视频含大量画面、代码或幻灯片，只完成语音转录时标明“视觉内容未提取”。

## 转录质量规则

- Whisper 结果是自动转录，不把它伪装成人工校对稿。
- 对 `CLAUDE.md`、Claude Code、命令、产品名、版本号等专有名词保留原始识别结果，并在不确定时标注 `[疑似]`；不要无证据静默纠正。
- 摘要可以纠正明显的标点和段落，但关键命令、数字和专有名词必须回看带时间戳文本或视频画面。
- 音频只覆盖语音；需要提取屏幕文字、PPT、代码或操作过程时，说明需要视频文件、关键帧截图和 OCR/视觉分析。

## 报告格式

用中文简洁报告：

1. 处理状态和实际产物链接。
2. 音频时长、文件大小和转录模型。
3. 3–10 条核心内容摘要。
4. 自动转录误差、未提取的视觉内容和其他未完成项。
