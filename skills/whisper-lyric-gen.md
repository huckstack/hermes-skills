---
name: whisper-lyric-gen
description: Uses OpenAI Whisper to generate word-level timestamped JSON lyrics from an MP3 file for use in generative art visualizers.
---
# Whisper Lyric Generator

This skill automates the creation of high-precision lyric data for generative art visualizers using the OpenAI Whisper model.

## Prerequisites
- Python 3.12+
- A dedicated virtual environment (recommended to avoid system package conflicts on Linux/Pop!_OS).
- `openai-whisper` installed in the environment.

## Setup (One-time)
To keep your system clean, create a dedicated environment:
```bash
mkdir -p ~/creative/tools/whisper_env
python3 -m venv ~/creative/tools/whisper_env/venv
~/creative/tools/whisper_env/venv/bin/pip install openai-whisper setuptools
```

## Usage
Run the script using the virtual environment's python interpreter to generate a timestamped JSON file.

### Command Template
`~/creative/tools/whisper_env/venv/bin/python ~/creative/tools/whisper_env/lyric_gen.py <input_mp3_path> <output_json_path>`

### Example
`~/creative/tools/whisper_env/venv/bin/python ~/creative/tools/whisper_env/lyric_gen.py ~/creative/generative-art/audio/song.mp3 ~/creative/generative-art/audio/lyrics_song.json`

## Output Format
Generates a JSON array of objects, perfect for p5.js `loadJSON()` integration:
```json
[
    {
        "text": "word",
        "start": 0.0,
        "end": 0.5
    }
]
```

## Pitfalls & Tips
- **Model Accuracy**: The script uses `whisper.load_model("base")` by default for speed. For complex music or low-quality audio, edit `lyric_gen.py` to use `"small"` or `"medium"`.
- **Timestamp Precision**: The script uses `word_timestamps=True` to ensure word-level synchronization for "karaoke" style effects.
- **Integration**: When using these lyrics in HTML/p5.js, ensure the path in your `fetch()` call matches the generated JSON file name.
- **Browser Security**: When testing locally, always use a local web server (e.g., `python3 -m http.server`) to avoid CORS/File-protocol errors when loading audio and JSON.
