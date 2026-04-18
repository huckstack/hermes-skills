---
name: llama-cpp-memory-workaround
category: devops
description: Workaround for llama.cpp memory save tool call HTTP 500 errors caused by unescaped special characters in summary text
---

# llama-cpp-memory-workaround

When running on local llama.cpp endpoints (llama-server), memory save tool calls fail with HTTP 500 if summary text contains apostrophes, slashes, or special characters. Args are not escaped before JSON serialization.

**Issue:** #12068 (open)

## Rule
ALWAYS use plain descriptive text without punctuation in memory saves when running on llama.cpp.

### DO
- `llama-cpp-memory-bug: Memory save tool calls fail with HTTP 500 when summary text contains apostrophes, slashes, or special chars. Args not escaped before JSON serialization. Issue #12068 open. Workaround: use plain descriptive text without punctuation in memory saves.`

### DON'T
- `User's preference: music visualizer uses p5.js with 6 modes and 13 mood palettes. Beat detection, particles, VHS glitch, lyrics, playlist w/ filter tags.` (contains apostrophe, slash)

## Checklist
1. Before calling memory tool, scan content for: `' " / \ * ? [ ] { }`
2. Replace with plain text: `w/` → `with`, `don't` → `do not`, remove quotes
3. Save only if clean