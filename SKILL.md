---
name: ffmpeg-whispercpp-subtitles
description: >-
  中文：使用 ffmpeg-full 将视频加速到 1.5x，提取音频并用 whisper.cpp 生成中文 SRT，再由当前会话
  Agent（如 Cursor/OpenClaw/Codex）同款 LLM 全文纠错，提示用户审阅后仅在明确确认时执行硬字幕烧录。English: Speeds
  video to 1.5x with ffmpeg-full, extracts audio, transcribes Chinese with whisper.cpp to SRT, then has
  the same LLM as the current session agent (for example Cursor/OpenClaw/Codex) correct the full SRT; prompts user review and only burns
  hard subtitles after explicit confirmation.
---

# ffmpeg + whisper.cpp：变速、转写、Agent 纠错、人工确认、烧录字幕（macOS）

[中文](#ffmpeg--whispercpp变速转写agent-纠错人工确认烧录字幕macos) | [English](#english-version)

## 平台兼容性

- 本 skill 不是 Cursor 专属，可用于支持文件读写与 shell 执行的 Agent 环境（例如 Cursor、OpenClaw、Codex）。
- 文中提到的 “Agent” 均指当前会话所使用的 AI 代理；纠错模型始终是当前会话同款 LLM，而非 whisper 模型权重。

## 适用场景

- 把本地视频 **1.5 倍速**（画面 + 人声同步）。
- **提取音频** 供 whisper.cpp 识别。
- 生成 **SRT** 后，由 **当前会话 Agent 所用的大语言模型**（与对话中模型一致，**不是**再跑 whisper 权重）通读整份字幕做 **上下文纠错**。
- **提示用户**在本地打开字幕文件 **核对并可手工修改**；**用户确认后**再 **硬字幕** 烧进 MP4。
- 变速与烧录用 **ffmpeg-full**；识别用 **whisper-cli**（whisper.cpp）；**字幕文本润色**由 Agent 在对话内完成，**不把字幕内容上传到 whisper 或第三方 ASR**（仅 Agent 处理文本，是否经云端取决于所用平台设置）。

## Agent 约定（必读）

### ffmpeg

- 必须使用带 **`subtitles` 滤镜**（libass）的 **`ffmpeg-full`**：`"$(brew --prefix ffmpeg-full)/bin/ffmpeg"`。默认 `brew install ffmpeg` 往往无 libass。
- 安装：`brew install ffmpeg-full`。路径备选：`/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg`（Apple Silicon）、`/usr/local/opt/ffmpeg-full/bin/ffmpeg`（Intel）。

### Whisper 之后、烧录之前（本版核心）

1. **步骤顺序（不可跳过）**  
   `1.5x 成片` → `抽 WAV` → **`whisper-cli` 生成 `BASE.srt`** → **Agent 全文纠错并写回字幕文件** → **向用户说明路径并请其打开核对** → **等待用户明确确认** → **最后一步烧录**。

2. **纠错由谁做**  
   使用 **执行本 skill 的当前会话 Agent 所用的大语言模型**（即与用户聊天的同一模型），**通读整份 `BASE.srt`**，结合全文语境做修改。**不要**另起一个「纠错模型」文件名去指 whisper 的 `ggml-*.bin`；whisper 只负责从音频出初稿。

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

## 步骤 4：Agent 全文纠错（当前会话模型）

**本步无 shell 一键命令**，由 Agent 在当前会话内完成：

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

---

## English Version

# ffmpeg + whisper.cpp: Speed-up, Transcription, Agent Correction, Human Review, and Hard Subtitles (macOS)

[中文](#ffmpeg--whispercpp变速转写agent-纠错人工确认烧录字幕macos) | [English](#english-version)

## Platform Compatibility

- This skill is not Cursor-only. It can run in any agent environment that supports file read/write and shell execution (for example Cursor, OpenClaw, Codex).
- "Agent" in this document means the AI agent used in the current session. The correction model is always the same LLM used in the current session, not whisper model weights.

## Use Cases

- Speed up a local video to **1.5x** while keeping video and voice synchronized.
- **Extract audio** for whisper.cpp transcription.
- After generating **SRT**, use the **same LLM as the current session agent** (not whisper weights) to read the whole subtitle file and perform **context-aware correction**.
- **Ask the user** to open the subtitle file locally for review and optional manual edits; burn **hard subtitles** into MP4 **only after explicit user confirmation**.
- Use **ffmpeg-full** for speed-up and burning; use **whisper-cli** (whisper.cpp) for ASR. Subtitle polishing is done by the Agent in chat, and subtitle text is **not** sent to whisper or third-party ASR (cloud usage depends on the platform configuration).

## Agent Contract (Required)

### ffmpeg

- You must use **ffmpeg-full** with **`subtitles` filter** (libass): `"$(brew --prefix ffmpeg-full)/bin/ffmpeg"`. `brew install ffmpeg` often lacks libass.
- Install with `brew install ffmpeg-full`. Alternative paths: `/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg` (Apple Silicon), `/usr/local/opt/ffmpeg-full/bin/ffmpeg` (Intel).

### After Whisper, Before Burn (Core of this skill)

1. **Required order (do not skip)**
   `1.5x video` -> `extract WAV` -> **`whisper-cli` outputs `BASE.srt`** -> **Agent corrects full SRT and writes back** -> **tell user the file path and ask them to review** -> **wait for explicit confirmation** -> **final burn step**.

2. **Who performs correction**
   Use the **same LLM model that powers the current session agent**. Read the **entire `BASE.srt`** and correct with full-context understanding. Do **not** treat whisper `ggml-*.bin` as a separate "correction model"; whisper only generates draft transcription from audio.

3. **Correction scope (without changing speaker intent)**
   - **Homophone / near-homophone errors** using context disambiguation.
   - **Terms, proper nouns, product names, fixed collocations** consistent with topic and full text.
   - **Obvious fluency issues** (missing/extra/repeated words) so each cue reads naturally.
   Do **not** heavily rewrite spoken style for polish. Do **not** fabricate unclear content; keep uncertainty or use markers like `[inaudible]` if user rules allow.

4. **SRT format and timeline**
   - You must preserve standard SRT structure: index, timestamp line, `-->`, and blank-line separators.
   - By default, do **not** change cue timestamps; only edit subtitle text lines. If text likely belongs to adjacent timing, do not shift timeline unless explicitly requested; mention the issue to the user.
   - Write corrected output back to the file used for burn (usually **`BASE.srt`**). Optional: save raw whisper output as **`BASE.raw.srt`** first, then overwrite `BASE.srt`.

5. **User confirmation gate (mandatory)**
   - After writing the corrected file, the Agent must clearly tell the user: absolute subtitle path, recommendation to open and review locally, and that manual edits are welcome.
   - **Do not run Step 5 (hard burn)** until user explicitly confirms (for example: "confirmed", "go burn", "proceed").
   - If user asks to "skip correction and burn directly", only do so when the user **explicitly waives** correction/review; otherwise follow correction + review by default.

6. **Burn after user manual edits**
   If user says they already edited subtitles manually, the Agent should **only run burn step**, using the user-specified **`BASE.srt` path** (default: current project `BASE.srt`).

## ffmpeg Path (fixed form)

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
# If brew is unavailable, use for example:
# FFMPEG="/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg"
```

## Prerequisites

1. `whisper-cli` is in `PATH` or available by absolute path; whisper model is local `ggml-*.bin`.
2. **`$FFMPEG`** is **ffmpeg-full** with both `h264_videotoolbox` and `subtitles` filter support.
3. Verify `subtitles` filter:

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -filters 2>/dev/null | grep subtitles
```

## Variable Conventions (replace globally)

| Variable | Meaning |
|------|------|
| `FFMPEG` | `$(brew --prefix ffmpeg-full)/bin/ffmpeg` |
| `INPUT.mp4` | Original source video |
| `SPED.mp4` | 1.5x speed-up video (intermediate) |
| `AUDIO.wav` | Extracted audio for whisper |
| `MODEL.bin` | Local ggml model path (for whisper-cli only) |
| `BASE` | Output basename (no extension). whisper generates `BASE.srt`; Agent correction writes to same file |
| `OUT.mp4` | Final hard-subtitled video |

## Workflow Overview

1. **1.5x speed-up**: `setpts=PTS/1.5` + `atempo=1.5`, encoded with VideoToolbox.
2. **Extract audio**: 16 kHz mono WAV.
3. **Transcribe**: `whisper-cli` -> **`BASE.srt` (draft)**.
4. **Agent correction**: current chat LLM reads full **`BASE.srt`**, applies correction rules, writes file back.
5. **Human review**: ask user to open and edit **`BASE.srt`**; wait for explicit confirmation.
6. **Burn**: `subtitles=filename='...BASE.srt'` + `h264_videotoolbox`.

---

## Step 1: Generate 1.5x Speed-up Video (A/V sync)

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "INPUT.mp4" -filter_complex "[0:v]setpts=PTS/1.5[v];[0:a]atempo=1.5[a]" -map "[v]" -map "[a]" -c:v h264_videotoolbox -b:v 6000k -c:a aac -b:a 192k "SPED.mp4"
```

- `atempo` supports **0.5 to 2.0**; 1.5 works in a single filter.

---

## Step 2: Extract Audio from Sped-up Video (WAV)

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "SPED.mp4" -ar 16000 -ac 1 -c:a pcm_s16le "AUDIO.wav"
```

---

## Step 3: whisper.cpp Generates SRT (Chinese)

```bash
whisper-cli -m "MODEL.bin" -f "AUDIO.wav" -l zh -osrt -of "BASE"
```

This generates **`BASE.srt`**. The executable might be `main` or `whisper-cpp`; replace command name if needed.

**Timeline note**: subtitle timing aligns with **`SPED.mp4`**. Do not burn transcription from original-speed input onto sped-up video.

---

## Step 4: Agent Full-file Correction (current session model)

**No one-line shell command for this step**; the Agent performs it in the current session:

1. Read full content of **`BASE.srt`**.
2. Produce corrected full SRT following "Agent Contract" correction scope and SRT constraints.
3. Write corrected full content back to **`BASE.srt`** (or back up to `BASE.raw.srt` first, then overwrite `BASE.srt`).
4. Briefly explain correction categories in chat, and include absolute path of **`BASE.srt`** again.

---

## Step 4.5: Prompt User Review (mandatory wording)

The Agent **must** send a user-facing message containing:

- Ask user to open **absolute path of `BASE.srt`** in local editor.
- Explain they can verify accuracy and **edit subtitle text freely**; burn output will strictly use this file.
- Ask user to **explicitly confirm** before burn (for example: "confirm burn").

**Do not run Step 5 before explicit user confirmation.**

---

## Step 5: Burn Hard Subtitles to Sped-up Video (only after confirmation)

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "SPED.mp4" -vf "subtitles=filename='ABS/PATH/BASE.srt'" -c:v h264_videotoolbox -b:v 6000k -c:a copy "OUT.mp4"
```

Replace `ABS/PATH/BASE.srt` with the **final approved absolute subtitle path**. If path contains spaces, keep `filename='...'` single-quoted.

Common errors: omitting `filename='...'` after `subtitles=`, or using non-ffmpeg-full binary.

---

## Optional: Soft Subtitles (mux only, no video re-encode)

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
"$FFMPEG" -y -i "SPED.mp4" -i "BASE.srt" -map 0:v:0 -map 0:a:0 -map 1:0 -c:v copy -c:a copy -c:s mov_text -metadata:s:s:0 language=chi "OUT_soft.mp4"
```

Still recommended only after user confirms final subtitle text.

---

## PATH and system ffmpeg note

- **`/opt/homebrew/bin/ffmpeg`** may be a slim build. For hard burn, always use ffmpeg-full absolute path.

## Prohibited Actions (avoid regressions)

- Do **not** burn hard/soft subtitles before user confirms **`BASE.srt`** (unless user explicitly waives review).
- Do **not** use original-speed transcription to burn onto **1.5x** `SPED.mp4`.
- Do **not** break SRT structure or arbitrarily rewrite timeline during Agent correction (default: text-only edits).
- Do **not** use `subtitles=/path/file.srt` without `filename=`.
- Do **not** assume `brew install ffmpeg` includes libass; use **ffmpeg-full** for hard subtitles.

## One-liner Pipeline Example (automation only; excludes Agent correction and user confirmation)

**Must stop after whisper**: insert Steps 4-4.5, and only run final burn command after user confirmation.

```bash
FFMPEG="$(brew --prefix ffmpeg-full)/bin/ffmpeg"
INPUT="/path/in.mp4" SPED="/path/sped.mp4" WAV="/path/sped.wav" MODEL="/path/ggml-small.bin" BASE="/path/sped" OUT="/path/out_hardsub.mp4" && \
"$FFMPEG" -y -i "$INPUT" -filter_complex "[0:v]setpts=PTS/1.5[v];[0:a]atempo=1.5[a]" -map "[v]" -map "[a]" -c:v h264_videotoolbox -b:v 6000k -c:a aac -b:a 192k "$SPED" && \
"$FFMPEG" -y -i "$SPED" -ar 16000 -ac 1 -c:a pcm_s16le "$WAV" && \
whisper-cli -m "$MODEL" -f "$WAV" -l zh -osrt -of "$BASE"
# Stop here: Agent correction + user confirmation on BASE.srt, then run:
# "$FFMPEG" -y -i "$SPED" -vf "subtitles=filename='${BASE}.srt'" -c:v h264_videotoolbox -b:v 6000k -c:a copy "$OUT"
```

`BASE` must not include `.srt`; in burn command, `${BASE}.srt` must resolve to the final approved absolute subtitle path.
