---
name: ffmpeg-whispercpp-subtitles
description: Speeds video to 1.5x with ffmpeg-full, extracts audio, transcribes Chinese with whisper.cpp to SRT, then has the Cursor Agent correct the full SRT using the same LLM as the chat (homophones, terms, fluency), prompts the user to review/edit the file, and only after explicit user confirmation runs hard subtitles with ffmpeg VideoToolbox. Uses $(brew --prefix ffmpeg-full)/bin/ffmpeg. Offline macOS workflow.
---

# ffmpeg + whisper.cpp：变速、转写、Agent 纠错、人工确认、烧录字幕（macOS）

## 适用场景

- 把本地视频 **1.5 倍速**（画面 + 人声同步）。
- **提取音频** 供 whisper.cpp 识别。
- 生成 **SRT** 后，由 **当前 Cursor Agent 所用的大语言模型**（与对话中模型一致，**不是**再跑 whisper 权重）通读整份字幕做 **上下文纠错**。
- **提示用户**在本地打开字幕文件 **核对并可手工修改**；**用户确认后**再 **硬字幕** 烧进 MP4。
- 变速与烧录用 **ffmpeg-full**；识别用 **whisper-cli**（whisper.cpp）；**字幕文本润色**由 Agent 在对话内完成，**不把字幕内容上传到 whisper 或第三方 ASR**（仅 Agent 处理文本，是否经云端取决于用户 Cursor 设置）。

## Agent 约定（必读）

### ffmpeg

- 必须使用带 **`subtitles` 滤镜**（libass）的 **`ffmpeg-full`**：`"$(brew --prefix ffmpeg-full)/bin/ffmpeg"`。默认 `brew install ffmpeg` 往往无 libass。
- 安装：`brew install ffmpeg-full`。路径备选：`/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg`（Apple Silicon）、`/usr/local/opt/ffmpeg-full/bin/ffmpeg`（Intel）。

### Whisper 之后、烧录之前（本版核心）

1. **步骤顺序（不可跳过）**  
   `1.5x 成片` → `抽 WAV` → **`whisper-cli` 生成 `BASE.srt`** → **Agent 全文纠错并写回字幕文件** → **向用户说明路径并请其打开核对** → **等待用户明确确认** → **最后一步烧录**。

2. **纠错由谁做**  
   使用 **执行本 skill 的 Cursor Agent 当前对话所用的大语言模型**（即与用户聊天的同一模型），**通读整份 `BASE.srt`**，结合全文语境做修改。**不要**另起一个「纠错模型」文件名去指 whisper 的 `ggml-*.bin`；whisper 只负责从音频出初稿。

3. **纠错范围（在不大改说话人原意的前提下）**  
   - **同音/近音错别字**（结合前后句消歧）。  
   - **术语、专名、产品名、常见固定搭配**（若视频主题可知，与全文一致）。  
   - **明显不通顺的用词、缺字多字、重复赘字**，使单条及相邻条读起来自然。  
   **不要**为「华丽」而大幅改写口语风格；**不要**编造听不清的内容，宁可保留不确定表述或加「[听不清]」类标记（若用户规则允许）。

4. **SRT 格式与时间轴**  
   - **必须保留** SRT 标准结构：序号、时间轴行、`-->`、空行分隔。  
   - **默认不修改**每条的起止时间码；仅改 **对白文本行**。若发现某条文本明显属于相邻时间段，**先不要擅自挪时间轴**，可在回复里用文字提醒用户；除非用户明确要求改轴。  
   - 纠错后把结果 **写回** 用户将用于烧录的那份文件（一般为 **`BASE.srt`**）。可选：先把 whisper 原始输出复制为 **`BASE.raw.srt`** 再覆盖 `BASE.srt`，便于对比（agent 视情况决定，不必强行）。

5. **用户确认闸门（强制）**  
   - Agent 完成写文件后，必须用清晰中文 **提示用户**：字幕绝对路径、建议用编辑器打开检查、可继续手工改。  
   - **在用户明确回复可以烧录**（例如「确认」「可以烧了」「烧录吧」等）**之前，禁止执行**「步骤 5：硬字幕烧录」中的 `ffmpeg` 命令。  
   - 若用户说「跳过纠错直接烧」：仅当用户**明确放弃** Agent 纠错与审阅闸门时，才可从 whisper 直出进入烧录；否则默认仍应先纠错并提示审阅。

6. **用户已手工改字幕后再烧录**  
   若用户表示「我自己改好了」，Agent **只执行烧录步骤**，使用用户指定的 **`BASE.srt` 路径**（默认仍为当前工程的 `BASE.srt`）。

## ffmpeg 路径（固定写法）

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
# 若 brew 不可用，可改为例如：
# FFMPEG="/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg"
```

## 前置条件

1. **whisper-cli** 在 `PATH` 或绝对路径；**whisper** 模型为本地 `ggml-*.bin`。
2. **`$FFMPEG`** 为 **ffmpeg-full**，支持 `h264_videotoolbox` 与 `subtitles` 滤镜。
3. 自检 `subtitles`：

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -filters 2>/dev/null | grep subtitles
```

## 变量约定（全文替换）

| 变量 | 含义 |
|------|------|
| `FFMPEG` | `$(brew --prefix ffmpeg-full)/bin/ffmpeg` |
| `INPUT.mp4` | 原始视频 |
| `SPED.mp4` | 1.5 倍速后的视频（中间文件） |
| `AUDIO.wav` | 提取的音频（给 whisper） |
| `MODEL.bin` | 本地 ggml 模型路径（仅 whisper-cli 使用） |
| `BASE` | 输出基名（不要扩展名），whisper 生成 `BASE.srt`；Agent 纠错也写此文件 |
| `OUT.mp4` | 最终硬字幕视频 |

## 流程概览

1. **1.5 倍速**：`setpts=PTS/1.5` + `atempo=1.5`，VideoToolbox 编码。
2. **抽音频**：16 kHz 单声道 WAV。
3. **识别**：`whisper-cli` → **`BASE.srt`（初稿）**。
4. **Agent 纠错**：用 **当前对话 LLM** 通读 **`BASE.srt` 全文**，按上文「纠错范围」修改并 **写回文件**。
5. **人工确认**：**提示用户**打开 **`BASE.srt`** 核对与手工修改；**等待用户明确确认**后再进行下一步。
6. **烧录**：`subtitles=filename='...BASE.srt'` + `h264_videotoolbox`。

---

## 步骤 1：生成 1.5 倍速视频（音画同步）

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "INPUT.mp4" -filter_complex "[0:v]setpts=PTS/1.5[v];[0:a]atempo=1.5[a]" -map "[v]" -map "[a]" -c:v h264_videotoolbox -b:v 6000k -c:a aac -b:a 192k "SPED.mp4"
```

- `atempo` 在 **0.5～2.0** 内；1.5 可用一条。

---

## 步骤 2：从变速后的视频提取音频（WAV）

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "SPED.mp4" -ar 16000 -ac 1 -c:a pcm_s16le "AUDIO.wav"
```

---

## 步骤 3：whisper.cpp 生成 SRT（汉语）

```bash
whisper-cli -m "MODEL.bin" -f "AUDIO.wav" -l zh -osrt -of "BASE"
```

生成 **`BASE.srt`**。可执行名可能是 `main` / `whisper-cpp`，需替换命令名。

**时间轴**：字幕时间对齐 **SPED.mp4**；不要用原速 INPUT 的识别结果去烧 SPED。

---

## 步骤 4：Cursor Agent 全文纠错（当前对话模型）

**本步无 shell 一键命令**，由 Agent 在 Cursor 内完成：

1. 用 Read 工具（或等价方式）读取 **`BASE.srt` 完整内容**。
2. 按本节「Agent 约定」中的纠错范围与 SRT 约束，生成 **修正后的完整 SRT 文本**。
3. 用 Write/StrReplace 将结果 **写回 `BASE.srt`**（或先备份 `BASE.raw.srt` 再写 `BASE.srt`）。
4. 在聊天中 **简短说明**做了哪几类修改（不必逐条罗列），并再次给出 **`BASE.srt` 的绝对路径**。

---

## 步骤 4.5：提示用户审阅（强制文案）

Agent **必须**向用户发送包含以下要素的说明（可合并为一段自然中文）：

- 请用户用本地编辑器打开：**`BASE.srt` 的绝对路径**。  
- 说明：可检查 Agent 修改是否准确，并 **随意手工改字**；烧录将严格以该文件为准。  
- 请用户 **确认无误后回复** 再进行烧录（例如「确认烧录」）。

**在用户发出确认前，不得执行步骤 5。**

---

## 步骤 5：硬字幕烧录到变速视频（仅用户确认后）

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "SPED.mp4" -vf "subtitles=filename='ABS/PATH/BASE.srt'" -c:v h264_videotoolbox -b:v 6000k -c:a copy "OUT.mp4"
```

将 `ABS/PATH/BASE.srt` 换为 **最终定稿的 SRT 绝对路径**（含空格则保持 `filename='...'` 单引号包裹）。

常见错误：`subtitles=` 后未用 `filename='...'`；二进制不是 ffmpeg-full。

---

## 可选：软字幕（不转码视频，仅封装）

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "SPED.mp4" -i "BASE.srt" -map 0:v:0 -map 0:a:0 -map 1:0 -c:v copy -c:a copy -c:s mov_text -metadata:s:s:0 language=chi "OUT_soft.mp4"
```

同样应在用户确认字幕定稿后再封装。

---

## 与 PATH / 系统自带 ffmpeg 的关系

- **`/opt/homebrew/bin/ffmpeg`** 可能是精简版，**硬烧录请始终用 ffmpeg-full 的绝对路径**。

## 禁止事项（避免再踩坑）

- **禁止**在用户确认 **`BASE.srt`** 之前执行烧录或软字幕封装（除非用户明确声明放弃审阅）。  
- **禁止**用原速 INPUT 的识别结果去烧 **1.5 倍速** SPED。  
- **禁止**在 Agent 纠错时破坏 SRT 结构或擅自大范围改时间轴（默认只改文本）。  
- **不要**用 `subtitles=/path/file.srt` 省略 `filename=`。  
- **不要**假设 `brew install ffmpeg` 即含 libass；硬字幕用 **ffmpeg-full**。

## 一键串联示例（仅自动化段；不含 Agent 纠错与用户确认）

**whisper 完成后必须停下**：插入步骤 4～4.5，用户确认后再手动或用 Agent 执行最后一行。

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
INPUT="/path/in.mp4" SPED="/path/sped.mp4" WAV="/path/sped.wav" MODEL="/path/ggml-small.bin" BASE="/path/sped" OUT="/path/out_hardsub.mp4" && \
"$FFMPEG" -y -i "$INPUT" -filter_complex "[0:v]setpts=PTS/1.5[v];[0:a]atempo=1.5[a]" -map "[v]" -map "[a]" -c:v h264_videotoolbox -b:v 6000k -c:a aac -b:a 192k "$SPED" && \
"$FFMPEG" -y -i "$SPED" -ar 16000 -ac 1 -c:a pcm_s16le "$WAV" && \
whisper-cli -m "$MODEL" -f "$WAV" -l zh -osrt -of "$BASE"
# 此处停止：Agent 纠错 + 用户确认 BASE.srt 后，再执行：
# "$FFMPEG" -y -i "$SPED" -vf "subtitles=filename='${BASE}.srt'" -c:v h264_videotoolbox -b:v 6000k -c:a copy "$OUT"
```

`BASE` 不要带 `.srt`；烧录行中 `${BASE}.srt` 须为定稿文件的绝对路径。
