# YouTube Content Extraction

一个用于 Codex 的 YouTube 内容提取 Skill：优先获取字幕；没有字幕时下载音频，再使用本地 Whisper 转录，并输出带时间戳的文字和简要摘要。

适用于：

- 从 YouTube 视频提取字幕或文字稿
- 下载可访问视频的音频
- 处理没有字幕的视频
- 将视频内容整理成中文摘要、章节和操作清单

## 功能流程

```text
YouTube URL
    ↓
检查人工字幕/自动字幕
    ├─ 有字幕 → 下载 VTT/SRT
    └─ 无字幕 → 下载 M4A 音频 → Whisper 本地转录
                                      ↓
                              TXT/SRT/VTT + 摘要
```

## 安装依赖

macOS：

```bash
brew install ffmpeg
python3 -m pip install -U yt-dlp openai-whisper
```

检查依赖：

```bash
command -v ffmpeg
python3 -m yt_dlp --version
python3 -m whisper --help
```

## 使用方法

将下面的 URL 替换为目标视频地址。

### 1. 检查字幕

```bash
python3 -m yt_dlp --list-subs "<YouTube URL>"
```

如果存在字幕，优先下载字幕，不必下载视频或音频：

```bash
python3 -m yt_dlp --skip-download \
  --write-subs --write-auto-subs \
  --sub-langs "zh.*,en.*" \
  --sub-format "vtt/best" \
  "<YouTube URL>"
```

### 2. 没有字幕时下载音频

```bash
python3 -m yt_dlp -x \
  --audio-format m4a \
  --audio-quality 0 \
  -o "%(title)s [%(id)s].%(ext)s" \
  "<YouTube URL>"
```

如果 YouTube 返回网页重新加载、SABR 或格式缺失错误，可以尝试：

```bash
python3 -m yt_dlp \
  --extractor-args "youtube:player_client=android" \
  -x --audio-format m4a --audio-quality 0 \
  -o "%(title)s [%(id)s].%(ext)s" \
  "<YouTube URL>"
```

### 3. 验证音频

```bash
ffprobe -v error \
  -show_entries format=filename,duration,size,format_name \
  -of default=noprint_wrappers=1 \
  "<audio file>"
```

### 4. 使用 Whisper 转录

```bash
python3 -m whisper "<audio file>" \
  --model turbo \
  --output_format all \
  --output_dir transcript
```

中文音频可以指定语言：

```bash
python3 -m whisper "<audio file>" \
  --model turbo \
  --language Chinese \
  --output_format all \
  --output_dir transcript
```

首次使用 `turbo` 模型可能需要下载约 1.5 GB 模型文件；CPU 环境可能需要较长处理时间。需要节省下载量时，可以改用 `small` 或 `base`，但准确率可能下降。

## 输出文件

通常会得到：

- `*.m4a`：提取的音频
- `*.txt`：纯文字稿
- `*.srt`：带时间戳的字幕
- `*.vtt`：网页字幕格式
- `*.tsv` / `*.json`：可供进一步处理的结构化转录数据

## 在 Codex 中使用

将本仓库作为本地 Skill 安装到 Codex 技能目录后，可以直接说：

```text
提取这个 YouTube 视频内容：<YouTube URL>
```

Skill 会按字幕优先、无字幕再转音频的流程执行，并报告实际生成的文件、时长、模型和摘要。

核心规则详见 [`SKILL.md`](SKILL.md)。

## 注意事项

- 只处理用户有权访问和使用的视频内容。
- 不绕过 DRM、付费限制、登录限制、验证码或其他访问控制。
- 不自动读取浏览器 Cookie，不自行获取 PO Token。
- Whisper 是自动语音识别，专有名词、命令、数字和版本号可能存在识别错误，应结合时间戳或画面复核。
- 音频转录不能替代画面分析；如果视频包含 PPT、代码或屏幕操作，还需要视频关键帧和 OCR/视觉分析。
