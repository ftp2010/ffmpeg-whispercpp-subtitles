# ffmpeg + whisper.cpp：变速、转写、Agent 纠错、人工确认、烧录字幕（macOS）

[中文](README.md) | [English](README.en.md)

## 适用场景

- 把本地视频 1.5 倍速（画面 + 人声同步）。
- 提取音频供 whisper.cpp 识别。
- 生成 SRT 后，由当前会话 Agent 所用的大语言模型（与对话中模型一致，不是再跑 whisper 权重）通读整份字幕做上下文纠错。
- 提示用户在本地打开字幕文件核对并可手工修改；用户确认后再把硬字幕烧进 MP4。
- 变速与烧录用 ffmpeg-full；识别用 whisper-cli（whisper.cpp）；字幕文本润色由 Agent 在对话内完成，不把字幕内容上传到 whisper 或第三方 ASR（仅 Agent 处理文本，是否经云端取决于所用平台设置）。
