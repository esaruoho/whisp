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

## License

MIT
