# whisp

Transcribe audio/video files using [OpenAI Whisper](https://github.com/openai/whisper), with automatic hallucination-loop removal and optional YouTube caption merging for better proper-noun accuracy.

## Setup

```bash
git clone git@github.com:esaruoho/whisp.git
cd whisp
```

Add to your PATH (pick one):

```bash
# Symlink into a directory already on your PATH
ln -s "$(pwd)/whisp" /usr/local/bin/whisp

# Or add this repo to PATH in your shell profile
echo 'export PATH="$HOME/path/to/whisp:$PATH"' >> ~/.zshrc
```

### Dependencies

whisp will **auto-install missing dependencies** on first run (prompts before installing):

| Dependency | What it does | Install command |
|---|---|---|
| [ffmpeg](https://ffmpeg.org/) | Audio/video processing | `brew install ffmpeg` |
| [openai-whisper](https://github.com/openai/whisper) | Speech-to-text | `brew install openai-whisper` |
| [yt-dlp](https://github.com/yt-dlp/yt-dlp) | YouTube download (only needed for URL mode) | `brew install yt-dlp` |

On macOS, whisp uses Homebrew for all dependencies (avoids pip/venv issues). If Homebrew itself is missing, whisp will offer to install it too.

On Linux, it falls back to `apt-get`.

Or install everything upfront:

```bash
brew install ffmpeg openai-whisper yt-dlp
```

## Usage

```
whisp [OPTIONS] [FILE...] [YOUTUBE_URL...]
```

### Basic transcription

```bash
whisp recording.mp3
whisp lecture.mp4
whisp *.m4a
```

Output goes to your **current working directory** (not the source file's directory).

### Force a language

```bash
whisp --en interview.m4a    # English
whisp --fi luento.mp4        # Finnish
whisp --de vortrag.ogg       # German
whisp --ru запись.wav        # Russian
whisp --sv föreläsning.mp3  # Swedish
```

### YouTube mode

Download audio from YouTube, transcribe, and merge with YouTube's captions for better proper nouns:

```bash
whisp https://www.youtube.com/watch?v=ABC123
whisp --en https://youtu.be/ABC123
```

Pair a local file with its YouTube URL for caption merging without re-downloading:

```bash
whisp video.mp4 https://www.youtube.com/watch?v=ABC123
```

### Options

| Flag | Description |
|---|---|
| `--en`, `--fi`, `--de`, `--ru`, `--sv` | Force language |
| `--model NAME` | Whisper model: `tiny`, `base`, `small`, `medium`, `large`, `turbo` (default: `turbo`) |
| `--out DIR` | Output directory (default: current directory) |
| `--condition-prev` | Enable conditioning on previous text (off by default to prevent hallucination loops) |
| `--no-halluc-filter` | Disable hallucination silence filter |
| `--no-merge` | Skip YouTube caption merge step |

### Supported formats

mp3, m4a, ogg, wav, flac, mp4, mkv, webm, avi, mov

### Recording mode

Record audio from your microphone, see transcripts streaming live as you speak, and get a full high-quality transcript the moment you stop. Useful for lectures, interviews, meetings, and any other "I want to participate AND have a transcript" situation.

```bash
whisp record                     # start recording; Ctrl+C to stop
whisp record --fi                # Finnish lecture
whisp record --out lecture       # lecture.wav, lecture.live.md, lecture.txt, ...
whisp record --list-devices      # see available mics
whisp record --no-live           # just record; transcribe only at the end
whisp record --no-final          # live only; no final high-quality pass
```

**How it works.** `whisp record` starts `ffmpeg` capturing your mic to a `.wav` file (16 kHz mono PCM). While recording, a background loop transcribes the audio in short segments using the same `turbo` model whisp uses for files, dedupes overlap between segments, and appends the new text to a `<base>.live.md` file. Hit `Ctrl+C` and it stops the recording, processes any trailing audio, then runs a final full-quality pass on the complete `.wav` through the normal whisp pipeline (Whisper + deloop), producing the usual 5 output formats.

**Live latency** is on the order of 15–25 seconds between "someone said it" and "you see it on screen." Good for grabbing quotes into your notes; too slow for responding to something you missed mid-conversation.

**Where to watch the live transcript.** The `.live.md` file is appended to continuously and is also printed to the `whisp record` terminal as it arrives. Open it in Obsidian with auto-reload, `less +F` in another pane, or any text editor that auto-refreshes.

**Output files.**

```
whisp-recording-20260410-143000.wav       # raw audio (always kept)
whisp-recording-20260410-143000.live.md   # live streaming transcript (always kept)
whisp-recording-20260410-143000.txt       # final high-quality transcript
.srt / .vtt / .tsv / .json                # (same base name)
```

The `.live.md` is a permanent historical record of what you saw in the room during the session — it's not deleted when the final pass runs. The final `.txt` / `.srt` / etc. are the authoritative transcripts.

**macOS microphone permission.** First run will prompt macOS to grant your terminal app (Terminal.app, iTerm2, etc.) microphone access. If `ffmpeg` exits immediately with "Operation not permitted", grant access in `System Settings → Privacy & Security → Microphone` and re-run.

**Archive is off by default for record mode.** In-person recordings are more privacy-sensitive than file-based transcription runs. Pass `--archive` if you explicitly want the final transcript pushed to `~/work/whisp-transcripts`.

**Options:**

| Flag | Description |
|---|---|
| `--en`, `--fa`, `--fi`, `--de`, `--ru`, `--sv` | Force language |
| `--model NAME` | Whisper model (default: `turbo` — same as whisp) |
| `--device DEV` | Audio input (macOS: `:0`, `:1`, ...; Linux: PulseAudio source name) |
| `--out NAME` | Output base filename (default: `whisp-recording-<timestamp>`) |
| `--out-dir DIR` | Output directory (default: current directory) |
| `--no-live` | Skip live transcription — just record audio |
| `--no-final` | Skip the final high-quality pass |
| `--archive` | Push final transcript to `~/work/whisp-transcripts` (opt-in) |
| `--list-devices` | List available audio input devices and exit |

## Output

Generates five output formats per file:

- `.txt` — plain text transcript
- `.srt` — SubRip subtitles
- `.vtt` — WebVTT subtitles
- `.tsv` — tab-separated (start/end/text in milliseconds)
- `.json` — full Whisper output with segments and word-level data

## How it works

1. **Whisper transcription** — runs OpenAI Whisper with hallucination silence filtering (2s threshold by default) and conditioning on previous text disabled
2. **Deloop** (`whisp-deloop`) — post-processes the JSON output to strip hallucination loops (3+ consecutive identical segments like "Thank you. Thank you. Thank you.")
3. **YouTube merge** (`whisp-merge`, optional) — when a YouTube URL is provided, downloads captions in parallel with Whisper, then merges proper nouns from YouTube's captions into the Whisper output using word-level timestamps and confidence scores
4. **Live recording** (`whisp-record` + `whisp-stream-merge`, optional) — `whisp record` captures mic audio to a WAV, transcribes it in overlapping chunks with the same `turbo` model whisp uses for files, dedupes overlap against the previous chunk's emitted timestamps, and appends new text to a `.live.md` file. On Ctrl+C, runs the normal pipeline (Whisper + deloop) on the full recording for the authoritative final transcript.

## License

MIT
