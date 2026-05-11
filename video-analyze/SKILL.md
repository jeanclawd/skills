---
name: video-analyze
description: Analyze the content of a web video — transcribe, summarize, answer questions about what happens or what's said. Trigger when the user gives a URL and asks to "summarize", "transcribe", "explain what's in", "extract the key points from", or "what does this video say about X". Different from the `video-export` skill, which only downloads the file. Skip if the user only wants the file or a thumbnail.
---

# video-analyze

End-to-end pipeline for turning a public web video into structured analysis you can answer questions about. Uses Copilot-CLI-style agent loops (or run it directly from Claude Code — same pipeline either way).

## Pipeline

```
URL ─► yt-dlp ─► audio.m4a ─► whisper.cpp / faster-whisper ─► transcript.txt
                                                              │
                                                              ▼
                                           Claude (or any LLM) ─► summary / Q&A
```

## Steps

1. **Identify the platform.** YouTube and LinkedIn are the common cases; both go through yt-dlp. YouTube increasingly bot-blocks cloud-provider IPs (see `Known failure modes` below).

2. **Get just the audio.** Full-quality video is wasteful when you only need text.

   ```bash
   command -v yt-dlp >/dev/null || uv tool install yt-dlp

   mkdir -p /tmp/vid && cd /tmp/vid
   yt-dlp --no-warnings \
          -f "bestaudio[ext=m4a]/bestaudio" \
          --extract-audio --audio-format m4a \
          -o "%(id)s.%(ext)s" \
          "<URL>"
   ```

   For login-walled content, add `--cookies-from-browser firefox` (or the browser the user is logged into).

3. **Transcribe.** Three viable engines in 2026, ranked:

   - **faster-whisper** (Python wrapper around CTranslate2) — fastest, ~10× real-time on CPU for the `tiny.en` model, ~2× on `medium`.

     ```bash
     uv run --with faster-whisper python3 -c "
     from faster_whisper import WhisperModel
     m = WhisperModel('small', device='cpu', compute_type='int8')
     segs, info = m.transcribe('VIDEO_ID.m4a', vad_filter=True)
     with open('transcript.txt', 'w') as f:
         for s in segs: f.write(f'[{s.start:.1f}] {s.text.strip()}\n')
     print(f'lang={info.language}, duration={info.duration:.0f}s')
     "
     ```

   - **whisper.cpp** — if no Python, C++ binary you can curl-install.

   - **OpenAI Whisper API** — paid, but useful for batch and for getting word-level timestamps.

4. **Summarize / answer the question.** Feed the transcript to Claude (or whatever model you're driving). Pick the prompt to the task:

   - **Summary:** "Summarize this video in 5 bullet points covering what the speaker demonstrates and what they conclude."
   - **Specific question:** "Looking at this transcript, what tool does the speaker recommend for X?"
   - **Feature extraction:** "List every CLI command and tool name mentioned, with the timestamp of first mention."
   - **Style match:** "Was this a tutorial, a marketing demo, or a conference talk? Give one supporting timestamp."

5. **Deliver back.** Send the summary as a chat message; attach the transcript file if the user wants it for reference. Don't attach the audio.

## When you also need the visuals (not just audio)

For demos where the speaker shows their screen and the relevant content is what's on the screen (CLI demos, IDE walkthroughs):

```bash
# Sample one frame per N seconds
ffmpeg -i VIDEO_ID.mp4 -vf "fps=1/5" /tmp/vid/frame_%04d.png
```

Then hand the most relevant frames to a vision-capable model (Claude Sonnet 4.6 reads images directly) together with the transcript:

> "These frames are sampled at 5-second intervals from a CLI demo. Tell me every command the speaker types."

For a 5-minute video at 1 frame / 5 s that's 60 frames; for an hour, downsample harder (1 frame / 30 s, then upsample around interesting transcript spans).

## Known failure modes

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Sign in to confirm you're not a bot` | YouTube bot wall — most aggressive against cloud-provider IPs | `--cookies-from-browser firefox`, OR ask the user to paste a cookies.txt, OR run from a residential IP |
| `RequestBlocked` from youtube-transcript-api | Same IP block, different endpoint | Same fix as above |
| Whisper hallucinates after silence | Long silent stretches confuse it | Enable VAD: `vad_filter=True` (already in the snippet above) |
| Wrong language detected | Mixed-language content | Force language: `transcribe(..., language='en')` |
| Cadence stutter on a fast speaker | `tiny` model on rapid speech | Step up to `small` or `medium` — costs throughput, fixes accuracy |
| Music-heavy video transcribes to nothing useful | Whisper is for speech | Skip transcription; sample frames + describe visually |

## Don'ts

- **Don't reproduce copyrighted creative content verbatim.** If the video is a song, comedy bit, audiobook, movie, or other creative work, refuse the full transcript and offer a summary or a few short quotes instead.
- **Don't push the raw audio or transcript file to public hosts** without explicit user permission. Treat it as the user's private working copy.
- **Don't run face/identity recognition on sampled frames.** If the user wants to know "who's in this video," ask them — that crosses a different line.
- **Don't trust the transcript for legal or medical advice.** Whisper makes confident errors on technical terms.

## Output template (when summarizing for delivery)

```
**<Inferred title or topic>** — <duration> · <speaker name if obvious>

Key points:
- <point 1, timestamp>
- <point 2, timestamp>
- <point 3, timestamp>

Tools / names mentioned: <list>

What the speaker concludes / recommends: <one sentence>

Caveats: <anything Whisper got wrong that you noticed; anything the
video shows visually that the transcript missed>
```

Trim to taste — short videos get 2–3 bullets, hour-long talks get 6–8.
