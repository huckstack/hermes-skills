---
name: config-change-verify
description: Verify model/server configuration changes with a live curl-based stress test. Use after any temperature, penalty, reasoning_effort, or server parameter change.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [config, verification, testing, llama.cpp, repetition, reasoning]
---

# Config-Change Verification Protocol

When changing model/server configuration, verify the change took effect with a live test — don't just read the config file.

## When to Use

- After any model/server config change (temperature, penalties, reasoning_effort)
- After llama.cpp server parameter changes (`--repeat-penalty`, `--temp`, etc.)
- Before declaring a config fix "done"
- After restarting the server to confirm new params are active

## The Stress Test

Run a curl-based test against the API:

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-local" \
  -d '{
    "model": "unsloth/Qwen3.6-35B-A3B-UD-Q4_K_XL",
    "messages": [
      {"role": "system", "content": "You are a helpful AI assistant. Generate creative, detailed responses. Be thorough."},
      {"role": "user", "content": "Write a detailed technical analysis of a music visualization pipeline that processes MP3 files through vocal extraction, lyric transcription, and generates a p5.js visualizer. Cover the architecture, data flow, error handling, and optimization strategies. Be very thorough and detailed."}
    ],
    "max_tokens": 4096,
    "temperature": 0.8,
    "repeat_penalty": 1.1,
    "frequency_penalty": 0.1,
    "presence_penalty": 0.1
  }' 2>/dev/null | python3 -c "
import sys, json, time
start = time.time()
data = json.load(sys.stdin)
elapsed = time.time() - start
usage = data.get('usage', {})
content = data['choices'][0]['message']['content']
lines = content.strip().split('\n')
sentences = [l.strip() for l in lines if len(l.strip()) > 20]
seen = {}
repetitions = 0
for i, s in enumerate(sentences):
    key = s[:60]
    if key in seen:
        repetitions += 1
        if repetitions <= 3:
            print(f'REP at line {i} (first at {seen[key]}): {s[:80]}...')
    else:
        seen[key] = i
print(f'Elapsed: {elapsed:.1f}s')
print(f'Tokens: {usage.get(\"completion_tokens\", \"?\")} gen / {usage.get(\"prompt_tokens\", \"?\")} prompt / {usage.get(\"total_tokens\", \"?\")} total')
print(f'Sentences: {len(sentences)}, Unique: {len(seen)}, Repetitions: {repetitions}')
print(f'Content length: {len(content)} chars')
"
```

## Pass Criteria

| Metric | Pass Threshold | Failure Indicates |
|--------|---------------|-------------------|
| Repetitions | 0 | Penalties too low, or reasoning loop still active |
| Token count | < 5000 (with 4096 cap) | >10000 means reasoning loops |
| Response time | < 60s for 4096 tokens | >120s = model looping |
| Unique sentences | == total sentences | Any < = repetition |

## Important Notes

- **Config.yaml params** (temperature, penalties) flow through the API — no restart needed
- **Server startup flags** (`--repeat-penalty`, `--temp` on llama.cpp) require server restart to take effect
- Always check BOTH `agent.reasoning_effort` and `delegation.reasoning_effort` in config.yaml
- After changing server flags, restart llama.cpp before running the test

## Known Config Patterns

- `reasoning_effort: medium` on non-reasoning models (Qwen3.6-35B-A3B) → infinite loops → use `none`
- `--repeat-penalty 1.0` (default) → repetitive output → use `1.1`
- `--presence-penalty 1.5` (default) → too high for this model → use `0.1`
- `--temp 0.6` → too low, encourages repetition → use `0.8`
- `--frequency-penalty` is NOT a default llama.cpp sampler — add `frequency_penalty` to `--samplers` list
- `--presence-penalty` is NOT a default llama.cpp sampler — add `presence_penalty` to `--samplers` list
