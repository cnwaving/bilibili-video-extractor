# 🎬 Bilibili Video Extractor

> 提取B站视频完整文字内容（音频 → Whisper 语音转文字），并自动生成结构化总结。

支持 **BV 号**、**b23.tv 短链接**、**完整 URL** 三种输入方式，适用于视频逐字稿获取、内容总结、知识提取等场景。

---

## ✨ 功能特性

| 特性 | 说明 |
|------|------|
| 🎥 多格式输入 | 支持 `BV1xx411x7xx`、`https://b23.tv/xxx`、完整 URL |
| 🔊 音频提取 | 通过浏览器环境获取带签名的 DASH 音频流 URL |
| 📝 语音转文字 | 基于 OpenAI Whisper，支持 `tiny` / `base` / `small` 多种模型 |
| 📋 结构化总结 | 自动生成包含核心观点、内容详解、金句摘录、评论精选的 Markdown 总结 |
| 💬 评论抓取 | 获取热门评论作为内容补充 |
| 📜 字幕优先 | 若视频有官方字幕，优先使用字幕而非语音转文字 |

---

## 📦 环境依赖

| 依赖 | 用途 | 安装方式 |
|------|------|----------|
| [agent-browser](https://www.npmjs.com/package/agent-browser) | 浏览器自动化，获取音频流签名 URL | `npm install -g agent-browser && agent-browser install` |
| [ffmpeg](https://ffmpeg.org/) | 音频格式转换（m4s → wav） | 通常已预装，或通过系统包管理器安装 |
| [openai-whisper](https://github.com/openai/whisper) | 语音转文字 | 见下方安装说明 |
| curl | 下载音频流、调用 API | 系统自带 |

### 安装 Whisper（推荐虚拟环境）

```bash
# 方式一：虚拟环境（推荐，避免污染系统 Python）
python3 -m venv ~/.venvs/bili && \
  ~/.venvs/bili/bin/pip install openai-whisper && \
  echo 'export PATH="$HOME/.venvs/bili/bin:$PATH"' >> ~/.bashrc

# 方式二：用户级安装
pip3 install --user openai-whisper
```

### 验证依赖

```bash
which agent-browser && which ffmpeg && python3 -c "import whisper" && echo "✅ 依赖齐全"
```

---

## 🚀 快速开始

### 一键执行（推荐）

```bash
python3 <skill_dir>/scripts/extract_bilibili_video.py "<BV号或URL>"
```

**输出文件：**

| 文件 | 说明 |
|------|------|
| `{OUTPUT_DIR}/{bvid}_meta.json` | 视频元数据（标题、UP主、时长等） |
| `{OUTPUT_DIR}/{bvid}_transcript.txt` | Whisper 转录全文 |

### 分步执行

#### 1️⃣ 获取视频元数据

```bash
curl -s "https://api.bilibili.com/x/web-interface/view?bvid=<BV号>" \
  -H "Referer: https://www.bilibili.com" \
  -H "User-Agent: Mozilla/5.0"
```

#### 2️⃣ 浏览器提取音频流 URL

```bash
agent-browser close --all 2>/dev/null
agent-browser open "https://www.bilibili.com/video/<BV号>"
agent-browser eval "(function(){const a=window.__playinfo__?.data?.dash?.audio?.[0];return JSON.stringify({url:a?.baseUrl})})()"
```

> ⚠️ 音频 URL 有时效性，获取后必须立即下载。

#### 3️⃣ 下载音频流

```bash
curl -s -L -o "<output>.m4s" \
  -H "Referer: https://www.bilibili.com" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" \
  -H "Origin: https://www.bilibili.com" \
  "<audio_url>"
```

#### 4️⃣ 音频格式转换

```bash
ffmpeg -y -i "<input>.m4s" -ar 16000 -ac 1 "<output>.wav"
```

#### 5️⃣ Whisper 语音转文字

```bash
whisper "<input>.wav" --model tiny --language Chinese --output_format txt
```

可通过环境变量 `BILI_WHISPER_MODEL` 切换模型：

```bash
BILI_WHISPER_MODEL=base whisper "<input>.wav" --language Chinese --output_format txt
```

#### 6️⃣ 获取评论 + 清理

```bash
curl -s "https://api.bilibili.com/x/v2/reply?type=1&oid=<aid>&pn=1&ps=10&sort=2" \
  -H "Referer: https://www.bilibili.com" -H "User-Agent: Mozilla/5.0"

agent-browser close --all
```

---

## 📊 总结输出模板

拿到转录文本后，将生成如下结构化 Markdown 总结：

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
[按逻辑分段，使用表格整理关键概念、对比等]

## 🔑 金句摘录
[3-5句视频中的关键原话]

## 📊 评论精选
[热门评论及点赞数]
```

### 总结生成规则

1. **提取核心论点** — 不逐字翻译，提炼 UP 主的核心观点
2. **表格优先** — 对比、分类、方剂等信息优先用表格呈现
3. **保留术语** — 专业术语、方剂名等保持原文
4. **评论补充** — 评论区常有观众笔记，可作为参考
5. **免责声明** — 医疗、金融等内容必须附免责声明

---

## ⚠️ 注意事项

| 事项 | 说明 |
|------|------|
| 音频 URL 时效性 | 从浏览器获取后需尽快下载（通常几分钟内有效） |
| Whisper 模型选择 | `tiny`（72MB，快） / `base`（142MB，精度更高） / `small`（精度最高，最慢） |
| 无音频设备 | 沙箱环境无声卡不影响下载音频流 |
| B站反爬 | `yt-dlp` 直接下载会被 412 拦截，必须通过浏览器获取签名 URL |
| 字幕优先 | 若视频有官方字幕（`subtitles` 字段非空），优先使用字幕 |

---

## 📁 项目结构

```
bilibili-video-extractor/
├── SKILL.md                    # WorkBuddy Skill 定义文件
├── README.md                   # 本文件
└── scripts/                    # （可选）一键执行脚本
    └── extract_bilibili_video.py
```

---

## 📜 许可证

MIT License — 可自由使用、修改、分发。

---

## 🤝 贡献

欢迎通过 Issue 或 Pull Request 提交改进建议。

---

**⚠️ 免责声明：** 本工具仅用于学习研究和个人使用，请遵守 B站用户协议及相关法律法规，不要用于任何侵犯他人权益的场景。视频内容的版权归原作者所有。
