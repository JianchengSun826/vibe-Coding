# 自动剪辑视频 Agent — 实施方案

> 基于 Claude API + Whisper + FFmpeg 构建端到端的智能视频剪辑 Agent

**文档状态**：规划阶段
**适用场景**：短视频精华剪辑、播客高亮提取、教程视频自动化
**更新日期**：2026-03-06

---

## 目录

1. [项目概述](#1-项目概述)
2. [核心需求与约束](#2-核心需求与约束)
3. [工具链选型](#3-工具链选型)
4. [推荐架构方案](#4-推荐架构方案)
5. [费用分析](#5-费用分析)
6. [实施流程](#6-实施流程)
7. [预期结果与局限性](#7-预期结果与局限性)

---

## 1. 项目概述

本项目旨在构建一个**自动剪辑视频 Agent**，输入一段原始视频（会议录屏、播客、Vlog、课程视频等），Agent 自动完成：语音识别 → 内容理解 → 智能剪辑决策 → 渲染输出，生成符合目标平台规格的成品视频。

**核心价值**：
- 人工剪辑一个 60 分钟视频需要 3~8 小时，Agent 可在 5~15 分钟内完成
- 不依赖视频剪辑软件经验，用自然语言描述剪辑需求即可（"剪成 60 秒精华"、"删除沉默段落"）
- 可批量处理，适合播客系列、课程系列的规模化生产

---

## 2. 核心需求与约束

| 需求 | 说明 |
|------|------|
| **语音驱动** | 通过分析字幕文本决定保留哪些片段，而非仅靠视觉 |
| **时间精度** | 剪切点必须精确到帧（≤ 1/30 秒），避免语句截断 |
| **自然衔接** | 剪辑点前后音频须平滑，无明显断裂感 |
| **格式兼容** | 输出支持 MP4/MOV，兼容抖音、B 站、YouTube 等平台规格 |
| **成本可控** | 处理 1 小时视频的 API 成本不超过 ¥15 |
| **可干预** | 提供剪辑时间线 JSON，用户可手动调整后再渲染 |

---

## 3. 工具链选型

### 3.1 语音识别层（STT）

| 工具 | 精度 | 速度 | 价格 | 时间戳精度 |
|------|------|------|------|-----------|
| **Whisper large-v3（本地）** | ✓✓ 最高 | △ 慢 | 免费（需 GPU） | ✓✓ 词级别 |
| **Whisper API（OpenAI）** | ✓✓ 最高 | ✓✓ 快 | $0.006/分钟 | ✓ 段落级别 |
| **Deepgram Nova-2** | ✓✓ 高 | ✓✓ 极快 | $0.0036/分钟 | ✓✓ 词级别 |
| AssemblyAI | ✓ 高 | ✓ 快 | $0.012/分钟 | ✓✓ 词级别 |
| 阿里云语音识别 | ✓ 中文优秀 | ✓✓ 快 | ¥0.03/分钟 | ✓ |

**推荐**：
- **中文视频**：阿里云语音识别（中文识别精度领先，且国内延迟低）
- **英文/双语视频**：Deepgram Nova-2（价格最低且时间戳精度高，适合做剪辑点定位）
- **离线/隐私场景**：本地 Whisper large-v3

### 3.2 剪辑决策层（LLM）

LLM 的职责：分析字幕文本，决定保留/删除哪些片段，生成剪辑时间线。

| 模型 | 指令遵循 | 中文 | 成本 | 适合度 |
|------|---------|------|------|--------|
| **Claude Sonnet 4.6** | ✓✓✓ 最强 | ✓✓ | $3/$15 per M | ✓✓ 首选 |
| Claude Haiku 4.5 | ✓✓ 优秀 | ✓ | $1/$5 per M | ✓✓ 经济首选 |
| DeepSeek V3 | ✓✓ 优秀 | ✓✓ 优秀 | $0.27/$1.1 per M | ✓✓ 极低成本 |
| GPT-4o mini | ✓✓ | ✓ | $0.15/$0.60 per M | ✓ |

**推荐**：**Claude Haiku 4.5** 作为剪辑决策模型
- 指令遵循能力强，生成结构化 JSON 时间线极其稳定
- 成本远低于 Sonnet，对于文本分析任务已足够
- 若预算极低，可用 DeepSeek V3 代替（成本再降 5 倍）

### 3.3 视频分析层

| 工具 | 用途 | 成本 |
|------|------|------|
| **FFmpeg** | 视频切割、拼接、静音检测、帧提取 | 免费开源 |
| **PySceneDetect** | 场景切换点自动检测 | 免费开源 |
| **pydub** | 音频波形分析、静音段落检测 | 免费开源 |
| Google Video Intelligence | 视觉内容分析（人脸、场景标签） | $0.10/分钟（可选，非必须）|

### 3.4 渲染层

| 工具 | 适用场景 | 速度 | 成本 |
|------|----------|------|------|
| **FFmpeg（本地）** | 剪切拼接、字幕烧录、格式转换 | 极快 | 免费 |
| MoviePy | Python 脚本控制复杂特效 | 慢 | 免费 |
| Creatomate API | 云端渲染，无需本地 GPU | 中 | $0.05~0.15/分钟输出 |
| RunwayML Gen-4 | AI 补帧、风格化特效 | 慢 | $0.05/秒（高端可选）|

**推荐**：本地 FFmpeg 负责主要渲染，MoviePy 处理需要动态计算的效果。

---

## 4. 推荐架构方案

### 方案 A：轻量本地方案（个人 / 低频）

```
原始视频
    ↓
[音频提取]  FFmpeg → 分离音轨
    ↓
[语音识别]  Whisper API / Deepgram → 带时间戳字幕 (.srt)
    ↓
[静音检测]  pydub → 标记无声段落时间区间
    ↓
[场景检测]  PySceneDetect → 标记镜头切换点
    ↓
[剪辑决策]  Claude Haiku 4.5（分析字幕 + 接收剪辑指令）
            → 输出剪辑时间线 JSON
    ↓
[人工确认]  （可选）展示时间线，用户调整
    ↓
[执行渲染]  FFmpeg → 按时间线剪切 + 拼接 + 字幕烧录
    ↓
成品视频（MP4）
```

**适合**：个人 Vlogger、播客主、内容创作者
**成本**：处理 1 小时视频约 ¥5~10

---

### 方案 B：生产级方案（SaaS / 高频批量）

```
原始视频（批量上传）
    ↓
[任务队列]  Redis + Celery → 分发处理任务
    ↓
[并行处理]  多 Worker 同时处理多个视频
    │
    ├─ [STT]         Deepgram API → 字幕 + 词级时间戳
    ├─ [视觉分析]    PySceneDetect → 场景切换
    └─ [音频分析]    pydub → 静音 / 高能量段落
    ↓
[AI 决策]   Claude Haiku 4.5（并发调用）
            → 生成剪辑计划 + 字幕 + BGM 建议
    ↓
[渲染集群]  FFmpeg 多进程 / Creatomate API
    ↓
成品视频 → 存储到 S3 / OSS → 通知用户
```

**适合**：视频 SaaS 产品、媒体公司、MCN 机构
**成本**：处理 1 小时视频约 ¥3~8（批量折扣后）

---

### Agent 工具定义（核心实现）

```python
# Claude Agent 的工具集
tools = [
    {
        "name": "get_transcript",
        "description": "获取视频的带时间戳字幕，返回每句话的开始/结束时间",
        "input_schema": {
            "type": "object",
            "properties": {
                "video_path": {"type": "string"},
                "language": {"type": "string", "default": "zh"}
            }
        }
    },
    {
        "name": "detect_silence",
        "description": "检测视频中的静音段落，返回静音区间列表",
        "input_schema": {
            "type": "object",
            "properties": {
                "video_path": {"type": "string"},
                "silence_threshold_db": {"type": "number", "default": -40},
                "min_silence_duration": {"type": "number", "default": 1.5}
            }
        }
    },
    {
        "name": "cut_and_merge",
        "description": "按照时间线剪切视频并拼接",
        "input_schema": {
            "type": "object",
            "properties": {
                "video_path": {"type": "string"},
                "segments": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "start": {"type": "number"},
                            "end": {"type": "number"},
                            "label": {"type": "string"}
                        }
                    }
                },
                "output_path": {"type": "string"}
            }
        }
    },
    {
        "name": "add_subtitle",
        "description": "将字幕烧录到视频，支持样式自定义",
        "input_schema": {
            "type": "object",
            "properties": {
                "video_path": {"type": "string"},
                "srt_content": {"type": "string"},
                "font_size": {"type": "number", "default": 36},
                "position": {"type": "string", "default": "bottom"}
            }
        }
    },
    {
        "name": "add_bgm",
        "description": "添加背景音乐，自动调整音量平衡",
        "input_schema": {
            "type": "object",
            "properties": {
                "video_path": {"type": "string"},
                "bgm_path": {"type": "string"},
                "bgm_volume": {"type": "number", "default": 0.15}
            }
        }
    }
]
```

---

## 5. 费用分析

### 5.1 按视频时长计算（方案 A 本地渲染）

**基础假设**：处理 1 小时原始视频，输出 5 分钟精华剪辑

```
STT（Deepgram Nova-2）：
    60 分钟 × $0.0036/分钟         = $0.22 ≈ ¥1.6

LLM（Claude Haiku 4.5）：
    字幕 token 约 30K（中等密度对话）
    + 系统提示 2K + 输出 3K
    输入：35K × $1/M               = $0.035
    输出：3K × $5/M                = $0.015
    小计                           = $0.05 ≈ ¥0.36

FFmpeg 渲染（本地 CPU）：
    约 5~15 分钟 CPU 时间           = $0（电费忽略）
────────────────────────────────────────────
合计（1 小时视频）                   ≈ ¥2.0
```

### 5.2 不同规格对比

| 视频时长 | STT | LLM | 渲染（本地） | **合计** |
|---------|-----|-----|------------|---------|
| 10 分钟 | ¥0.26 | ¥0.06 | ¥0 | **≈ ¥0.35** |
| 30 分钟 | ¥0.78 | ¥0.18 | ¥0 | **≈ ¥1.0** |
| 1 小时 | ¥1.56 | ¥0.36 | ¥0 | **≈ ¥2.0** |
| 3 小时 | ¥4.68 | ¥1.08 | ¥0 | **≈ ¥6.0** |

### 5.3 云端渲染附加成本（Creatomate）

若使用云渲染（无本地 GPU 场景）：

```
输出 5 分钟视频 × $0.10/分钟 = $0.50 ≈ ¥3.6
```

即云渲染场景下，处理 1 小时视频总成本约 **¥5.6**。

### 5.4 月费估算（内容创作者场景）

**假设**：每月处理 20 个视频，每个平均 30 分钟

```
STT：20 × ¥0.78                  = ¥15.6
LLM：20 × ¥0.18                  = ¥3.6
渲染（本地）                       = ¥0
────────────────────────────────────────
月合计                             ≈ ¥20/月
```

> **结论**：个人创作者月费约 **¥20**（低于任何付费剪辑软件订阅费）。

---

## 6. 实施流程

### Phase 0：环境准备（第 1 天）

```bash
# 创建项目目录
mkdir video-agent && cd video-agent
python -m venv venv && source venv/bin/activate

# 安装核心依赖
pip install anthropic openai pydub scenedetect ffmpeg-python python-dotenv

# 检查 FFmpeg 安装
ffmpeg -version

# macOS 安装 FFmpeg（若未安装）
brew install ffmpeg

# 配置 API Keys
cat > .env << EOF
ANTHROPIC_API_KEY=your_key_here
DEEPGRAM_API_KEY=your_key_here   # 或 OPENAI_API_KEY
EOF
```

**检查项**：
- [ ] FFmpeg 版本 ≥ 5.0
- [ ] Python 环境正常
- [ ] API Key 配置完成

---

### Phase 1：语音识别模块（第 1~2 天）

**核心文件**：`src/transcriber.py`

```python
import anthropic
from deepgram import DeepgramClient

def transcribe_video(video_path: str, language: str = "zh") -> list[dict]:
    """
    返回格式：[{"start": 0.5, "end": 3.2, "text": "今天我们来讲..."}, ...]
    """
    # 1. 用 FFmpeg 提取音频
    audio_path = extract_audio(video_path)

    # 2. 调用 Deepgram（词级时间戳）
    client = DeepgramClient(api_key=DEEPGRAM_API_KEY)
    with open(audio_path, "rb") as f:
        response = client.listen.rest.v("1").transcribe_file(
            {"buffer": f, "mimetype": "audio/mp3"},
            {"model": "nova-2", "language": language,
             "punctuate": True, "utterances": True}
        )

    return parse_utterances(response)
```

**验证**：使用一段 5 分钟测试视频，检查时间戳精度是否在 ±0.3 秒以内。

---

### Phase 2：视频分析模块（第 2~3 天）

**核心文件**：`src/analyzer.py`

```python
from pydub import AudioSegment
from scenedetect import detect, ContentDetector

def detect_silence(video_path: str, threshold_db: float = -40.0) -> list[dict]:
    """检测静音段落，返回 [{start, end, duration}]"""
    audio = AudioSegment.from_file(video_path)
    silence_ranges = silence.detect_silence(
        audio, min_silence_len=1500, silence_thresh=threshold_db
    )
    return [{"start": s/1000, "end": e/1000} for s, e in silence_ranges]

def detect_scenes(video_path: str) -> list[float]:
    """返回场景切换时间点列表（秒）"""
    scenes = detect(video_path, ContentDetector(threshold=30.0))
    return [scene[0].get_seconds() for scene in scenes]
```

---

### Phase 3：Agent 剪辑决策（第 3~4 天）

**核心文件**：`src/agent.py`

```python
import anthropic
import json

client = anthropic.Anthropic()

SYSTEM_PROMPT = """你是专业视频剪辑师 Agent。根据视频字幕和分析数据，生成精准的剪辑时间线。

输出格式（严格遵守）：
{
  "segments": [
    {"start": 12.5, "end": 45.0, "reason": "核心论点"},
    {"start": 67.2, "end": 89.5, "reason": "精彩案例"}
  ],
  "total_duration": 54.8,
  "removed_reasons": ["00:00~00:12 开场寒暄", "00:45~01:07 重复内容"]
}

原则：
1. 删除：沉默超过 2 秒的段落、重复表达、无实质内容的开场/结尾
2. 保留：完整句子，不在句中截断
3. 剪切点尽量选在自然停顿处（句末、段落末）
"""

def generate_edit_plan(
    transcript: list[dict],
    silence_segments: list[dict],
    target_duration: int,
    user_instruction: str
) -> dict:
    """让 Claude 生成剪辑时间线"""

    context = f"""
目标时长：{target_duration} 秒
用户要求：{user_instruction}

字幕（含时间戳）：
{json.dumps(transcript, ensure_ascii=False, indent=2)}

静音段落：
{json.dumps(silence_segments, ensure_ascii=False, indent=2)}
"""

    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=2048,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": context}]
    )

    return json.loads(response.content[0].text)
```

---

### Phase 4：渲染执行模块（第 4~5 天）

**核心文件**：`src/renderer.py`

```python
import ffmpeg
import subprocess

def render_segments(video_path: str, segments: list[dict], output_path: str):
    """按时间线剪切并拼接视频"""

    # 1. 为每个片段生成临时文件
    temp_files = []
    for i, seg in enumerate(segments):
        temp_path = f"/tmp/seg_{i:03d}.mp4"
        (
            ffmpeg
            .input(video_path, ss=seg["start"], t=seg["end"]-seg["start"])
            .output(temp_path, c="copy", avoid_negative_ts="make_zero")
            .overwrite_output()
            .run(quiet=True)
        )
        temp_files.append(temp_path)

    # 2. 生成 concat 列表
    concat_list = "/tmp/concat_list.txt"
    with open(concat_list, "w") as f:
        for path in temp_files:
            f.write(f"file '{path}'\n")

    # 3. 拼接所有片段
    (
        ffmpeg
        .input(concat_list, format="concat", safe=0)
        .output(output_path, c="copy")
        .overwrite_output()
        .run(quiet=True)
    )

    # 4. 清理临时文件
    for path in temp_files:
        os.remove(path)

def add_subtitle(video_path: str, srt_path: str, output_path: str):
    """将 SRT 字幕烧录进视频"""
    (
        ffmpeg
        .input(video_path)
        .output(output_path,
                vf=f"subtitles={srt_path}:force_style='FontSize=36,Alignment=2'")
        .overwrite_output()
        .run(quiet=True)
    )
```

---

### Phase 5：主入口 & 端到端测试（第 5~7 天）

**核心文件**：`src/main.py`

```python
async def process_video(
    video_path: str,
    output_path: str,
    target_duration: int = 60,
    instruction: str = "保留最精彩、信息密度最高的部分",
    add_subs: bool = True
):
    print("1/5 语音识别中...")
    transcript = transcribe_video(video_path)

    print("2/5 视频分析中...")
    silences = detect_silence(video_path)
    scenes = detect_scenes(video_path)

    print("3/5 AI 生成剪辑方案...")
    edit_plan = generate_edit_plan(transcript, silences, target_duration, instruction)

    # 展示计划，供用户确认
    print(f"\n剪辑方案：保留 {len(edit_plan['segments'])} 个片段")
    print(f"预计时长：{edit_plan['total_duration']} 秒")
    for seg in edit_plan['segments']:
        print(f"  [{seg['start']:.1f}s ~ {seg['end']:.1f}s] {seg['reason']}")

    print("\n4/5 渲染中...")
    render_segments(video_path, edit_plan['segments'], output_path)

    if add_subs:
        print("5/5 烧录字幕...")
        srt_path = generate_srt(transcript, edit_plan['segments'])
        add_subtitle(output_path, srt_path, output_path.replace(".mp4", "_sub.mp4"))

    print(f"\n✓ 完成！输出：{output_path}")
    return edit_plan
```

**端到端测试用例**：

| 测试类型 | 输入 | 期望结果 |
|---------|------|---------|
| 基础剪辑 | 10 分钟对话视频 → "剪成 2 分钟" | 输出约 2 分钟，无截断句子 |
| 静音删除 | 含大量停顿的录音 | 停顿超过 2 秒的全部删除 |
| 重复内容 | 讲师重复解释同一内容 | 只保留首次说明 |
| 时间戳精度 | 检查剪切点是否在句末 | 不出现词语被截断 |
| 超长视频 | 3 小时录屏 → "30 分钟精华" | 成功处理，不超时 |

---

## 7. 预期结果与局限性

### 7.1 预期效果

| 功能 | 预期表现 |
|------|---------|
| 静音删除 | 准确率 95%+，基于波形分析，效果稳定 |
| 内容筛选 | 基于语义理解，比简单关键词匹配更准确 |
| 时间戳精度 | Deepgram 词级时间戳 ≤ ±0.1 秒误差 |
| 处理速度 | 1 小时视频约 5~10 分钟处理完成（本地渲染） |
| 字幕生成 | 自动同步剪辑后的时间线，无需手动校正 |

### 7.2 已知局限性

**纯文字驱动的天花板**：Agent 只分析语音内容，无法理解视觉信息（如"这里画面很精彩但没什么台词"的片段会被删掉）。需要结合视觉分析 API（额外成本）才能解决。

**音频连贯性**：剪辑后的音频在衔接处可能有轻微不自然感（背景音突变）。解决方案是在 FFmpeg 渲染时加入 3~5 帧音频淡入淡出（`afade`），或使用专业的音频处理工具。

**多语言混合**：中英文混合视频（如技术演讲）的 STT 识别精度会下降。建议分语言分段处理。

**长视频上下文**：超过 3 小时的视频，字幕 token 量可能超过模型单次 context window。解决方案：分段处理，每段 30~40 分钟，最后合并时间线。

**版权音乐**：自动添加 BGM 功能不包含版权音乐库，需用户自备无版权音乐文件。

### 7.3 未来扩展路径

```
现阶段（语音驱动剪辑）
    ↓
加入视觉分析
  → 集成 Google Video Intelligence 或 AWS Rekognition
  → 识别精彩表情、关键动作、屏幕内容变化
    ↓
加入 AI 特效
  → RunwayML：自动补帧、风格迁移、背景替换
  → 适合 Vlog、创意短视频场景
    ↓
Web 界面
  → 上传视频 → 查看剪辑预览 → 一键调整时间线 → 下载成品
  → 基于 Next.js + FastAPI 构建
    ↓
多平台规格自适应
  → 同一原视频自动输出：16:9（YouTube）、9:16（抖音）、1:1（Instagram）
  → 自动检测主要人物位置进行智能裁切
```

---

## 附录：技术栈汇总

| 组件 | 选型 | 用途 |
|------|------|------|
| LLM（决策） | Claude Haiku 4.5 | 剪辑方案生成、内容理解 |
| STT（中文） | 阿里云语音识别 | 中文字幕 + 时间戳 |
| STT（英文） | Deepgram Nova-2 | 英文字幕 + 词级时间戳 |
| 视频处理 | FFmpeg | 剪切、拼接、字幕烧录 |
| 静音检测 | pydub | 音频波形分析 |
| 场景检测 | PySceneDetect | 镜头切换点识别 |
| Python 封装 | ffmpeg-python | FFmpeg 的 Python API |
| 任务队列（生产） | Redis + Celery | 批量视频并发处理 |
| 云端渲染（可选） | Creatomate API | 无本地 GPU 的云渲染方案 |
| AI 特效（可选） | RunwayML | 高端特效、补帧 |

---

*本文档基于 2026 年 3 月技术现状撰写，API 价格和模型能力随市场变化，建议定期复核。*
