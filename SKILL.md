---
name: bilibili-video-extractor
description: >
  提取B站视频完整文字内容（音频→Whisper语音转文字），并生成结构化总结。
  支持 BV 号、b23.tv 短链接、完整URL。适用于需要获取B站视频逐字稿、内容总结、
  知识提取等场景。关键词：B站、bilibili、视频总结、视频提取、语音转文字、whisper、
  BV号、b23.tv。
---

# B站视频内容提取器

提取B站视频完整音频文字内容，并自动生成结构化总结。

## 前置依赖

使用前确认以下工具已安装：

```bash
# 检查
which agent-browser && which ffmpeg && python3 -c "import whisper" && echo "✅ 依赖齐全"
```

如缺少：
- `agent-browser`: `npm install -g agent-browser && agent-browser install`
- `ffmpeg`: 通常已预装
- `whisper`: 推荐使用虚拟环境安装，避免污染系统 Python
  ```bash
  # 方式一：虚拟环境（推荐）
  python3 -m venv ~/.venvs/bili && \
    ~/.venvs/bili/bin/pip install openai-whisper && \
    echo 'export PATH="$HOME/.venvs/bili/bin:$PATH"' >> ~/.bashrc

  # 方式二：用户级安装（不污染系统）
  pip3 install --user openai-whisper
  ```

## 工作流程

提取B站视频内容分为 **6个步骤**：

### 步骤 1：解析输入，获取视频元数据

从用户输入中解析 BV 号（支持 b23.tv 短链接、完整URL、纯BV号），然后通过 B站 API 获取标题、UP主、时长等。

```bash
# 用 curl 获取元数据
curl -s "https://api.bilibili.com/x/web-interface/view?bvid=<BV号>" \
  -H "Referer: https://www.bilibili.com" \
  -H "User-Agent: Mozilla/5.0"
```

### 步骤 2：浏览器打开页面，提取音频流 URL

B站视频需要浏览器环境才能获取带签名的播放地址。用 `agent-browser` 打开页面，从 `window.__playinfo__` 提取 DASH 音频流 URL。

```bash
agent-browser close --all 2>/dev/null
agent-browser open "https://www.bilibili.com/video/<BV号>"
# 等待加载后
agent-browser eval "(function(){const a=window.__playinfo__?.data?.dash?.audio?.[0];return JSON.stringify({url:a?.baseUrl})})()"
```

**关键点**：音频 URL 有时效性，必须在获取后立即下载。

### 步骤 3：下载音频流

用 curl 带 Referer/Origin/UA 头下载 m4s 音频：

```bash
curl -s -L -o "<output>.m4s" \
  -H "Referer: https://www.bilibili.com" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" \
  -H "Origin: https://www.bilibili.com" \
  "<audio_url>"
```

### 步骤 4：音频格式转换

ffmpeg 转 16kHz 单声道 WAV：

```bash
ffmpeg -y -i "<input>.m4s" -ar 16000 -ac 1 "<output>.wav"
```

### 步骤 5：Whisper 语音转文字

```bash
whisper "<input>.wav" --model tiny --language Chinese --output_format txt
```

- 默认用 `tiny` 模型（速度快，中文尚可），可设 `BILI_WHISPER_MODEL` 环境变量改为 `base`/`small` 提高精度
- 转录文本会保存为 `.txt` 文件

### 步骤 6：获取评论 + 关闭浏览器

```bash
curl -s "https://api.bilibili.com/x/v2/reply?type=1&oid=<aid>&pn=1&ps=10&sort=2" \
  -H "Referer: https://www.bilibili.com" -H "User-Agent: Mozilla/5.0"

agent-browser close --all
```

## 一键执行

也可以用封装好的脚本一键完成：

```bash
python3 <skill_dir>/scripts/extract_bilibili_video.py "<BV号或URL>"
```

脚本输出：
- `{OUTPUT_DIR}/{bvid}_meta.json` — 视频元数据
- `{OUTPUT_DIR}/{bvid}_transcript.txt` — Whisper 转录全文
- 最后打印 `---RESULT_JSON---` 标记的结构化结果

## 转录后：生成结构化总结

拿到转录文本后，按以下模板生成总结：

```markdown
# 视频总结：[标题]

## 📋 基本信息
| 项目 | 内容 |
|------|------|
| 标题 | ... |
| UP主 | ... |
| 时长 | ... |
| 播放/收藏 | ... |

## 🎯 核心观点
[1-3句话概括视频最核心的论点]

## 📝 内容详解
[按逻辑分段，使用表格整理关键概念、对比、方剂等]

## 🔑 金句摘录
[3-5句视频中的关键原话]

## 📊 评论精选
[热门评论及点赞数]
```

### 总结要点

1. **提取核心论点**：不要逐字翻译，提炼 UP 主的核心观点
2. **用表格呈现**：对比、分类、方剂等信息优先用表格
3. **保留关键术语**：中医方剂名、专业术语保持原文
4. **添加评论补充**：评论区常有观众做的笔记总结，可作为补充参考
5. **附免责声明**：医疗类内容必须加免责声明

## 注意事项

- **音频 URL 时效性**：从浏览器获取后尽快下载（通常几分钟内有效）
- **Whisper 模型选择**：tiny 模型约 72MB，转写 8 分钟视频约需 2 分钟（CPU）；base 模型约 142MB，精度更高
- **环境无音频设备**：沙箱环境可能没有声卡，这不影响下载音频流
- **B站反爬**：`yt-dlp` 直接下载 B站视频会被 412 拦截，必须通过浏览器获取签名 URL
- **字幕备选**：如果视频有官方字幕（`subtitles` 字段非空），优先使用字幕而非语音转文字
