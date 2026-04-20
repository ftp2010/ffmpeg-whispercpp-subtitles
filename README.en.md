# ffmpeg + whisper.cpp: Speed-up, Transcription, Agent Correction, Human Review, and Hard Subtitles (macOS)

[中文](README.md) | [English](README.en.md)

## Use Cases

- Speed up local videos to 1.5x (video and voice stay in sync).
- Extract audio for whisper.cpp transcription.
- After generating the SRT file, use the same LLM backing the current session agent (not whisper weights) to review the full subtitle file and correct context-level errors.
- Prompt the user to open the subtitle file locally for manual review and optional edits; only burn hard subtitles into MP4 after explicit user confirmation.
- Use ffmpeg-full for speed-up and subtitle burning; use whisper-cli (whisper.cpp) for ASR; subtitle polishing is done by the Agent in-chat, without sending subtitle content to whisper or third-party ASR services (whether text goes through cloud inference depends on the platform configuration).
