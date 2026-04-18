---
name: whisper-vocal-isolation-debug
description: Troubleshooting Whisper missing early/quiet vocals in the Demucs → Whisper pipeline
---

# whisper-vocal-isolation-debug

Troubleshooting Whisper missing early/quiet vocals in the Demucs → Whisper pipeline.

## Problem
Whisper transcription starts late (e.g., first detected word at 14+ seconds) even though vocals are audible from song start.

## Diagnosis Steps

1. Check first timestamp in master_data.json:
   ```
   python3 -c "
   import json
   with open('master_data.json') as f:
       data = json.load(f)
   print('First segment:', data['lyrics'][0])
   "
   ```

2. Check raw Whisper output for earliest segment:
   ```
   python3 -c "
   import json
   with open('vocals.json') as f:
       raw = json.load(f)
   segs = raw['segments']
   print('First 3 raw segments:')
   for s in segs[:3]:
       words = s.get('words', [])
       if words:
           print(f'  {s[\"start\"]:.2f}s first_word=\"{words[0][\"word\"]}\"')
   "
   ```

3. Compare with original MP3 duration:
   ```
   ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 track.mp3
   ```

## Root Cause
Demucs vocal isolation filters out quiet/obscured vocals before Whisper even sees them. Whisper has a **hard confidence floor** (~13-15s for heavily mumbled intros) regardless of model or temperature settings. Even large-v3 + temp 0.1 won't transcribe below this energy threshold.

## Solutions (in order of priority)

### 1. Use mdx family Demucs models (BEST)
The mdx ensemble models have historically best preserved quiet vocals:
- `mdx_extra_q` — best quality, quantized (recommended first try)
- `mdx_extra` — maximum quality, slowest
- `mdx_q` — quantized original mdx ensemble

Change in orchestrator.py step_1_demucs():
```python
"-n", "mdx_extra_q",  # instead of "htdemucs_ft"
```

### 2. Upgrade Whisper model
Medium → large-v3 (more sensitive to quiet vocals):
```python
"--model", "large-v3",  # instead of "medium"
```

### 3. Whisper on original MP3 (fallback)
Run Whisper directly on the original MP3, skipping Demucs entirely. Captures ALL vocals but with more background noise in transcription.

### 4. Hybrid approach
Run Whisper on original MP3 for early lyrics, use Demucs-vocal track for the rest. Merge manually.

## Hard Floor — When Model Upgrades Don't Help

If you've tried mdx models + large-v3 + temp 0.1 and the first segment is still >10s:

**The intro is genuinely below Whisper's detection threshold.** Demucs attenuates quiet vocals further. Options:

1. **Accept the limit** — start lyrics at first confident detection. Clean output, no noise.
2. **Transcribe full audio** — skip Demucs, run Whisper on original MP3. Picks up mumbles but also instruments/noise.
3. **Manual annotation** — add early lyrics by hand to the JSON. Best for short intros with a few mumbled lines.

**Empirical data from testing:**
- `medium` model: first word at ~14.48s
- `large-v3` + temp 0.1: first word at ~13.76s (0.72s improvement, still 13s+ gap)
- Word confidence for first segment: ~0.30 (low, but readable)

## Model Selection Cheat Sheet

| Model | Vocal Quality | Speed | Params | Notes |
|-------|-------------|-------|--------|-------|
| mdx_extra_q | ★★★★★ | Medium | ~167M | Best quality, quantized |
| mdx_extra | ★★★★★ | Slow | ~668M | Max quality |
| mdx_q | ★★★★ | Medium | ~200M | Balanced |
| htdemucs_ft | ★★★★ | Fast | ~168M | Per-source weighting |
| htdemucs | ★★★ | Fastest | ~42M | Default, lighter |
| large-v3 | - | Slow | - | Whisper model, not Demucs |

## Verification
After reprocessing, check:
- First segment timestamp should be near 0s (or at least <5s for typical songs)
- Total segments should increase
- Raw Whisper segments should increase (more detected content)
