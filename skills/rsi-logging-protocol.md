# Skill: rsi-logging-protocol

## Trigger
After EVERY fix/patch/write_file/cron action. Mandatory. No exceptions.

## Steps
1. After applying a fix, call `rsi-log fix --summary "<what was fixed>" --category <server_config|pipeline|maintenance|memory|cron> --files <comma-separated paths>`
2. If the fix is a lesson learned, also call `rsi-log lesson --text "<lesson>" --category <same> --severity <high|medium|low>`
3. For recurring issues, call `rsi-log pattern --description "<pattern>" --status <observed|applied|resolved> --severity <high|medium|low> --applied "<fix applied>"`
4. When something surprising or interesting happens (not a bug, but a pattern, unexpected behavior, or something worth investigating), call:
   `rsi-log surprise --text "<what was surprising>" --category <same> --signal <high|medium|low> [--investigate]`

## Categories
- `server_config` — llama.cpp, systemd, config.yaml changes
- `pipeline` — orchestrator, whisper, demucs, visualizer changes
- `maintenance` — cleanup, logging, monitoring, config hygiene
- `memory` — context compaction, memory/learning_db changes
- `cron` — cron job creation/modification/removal

## Severity
- `high` — affects system stability, causes crashes, data loss
- `medium` — performance, accuracy, reliability improvements
- `low` — cleanup, documentation, minor fixes

## Examples
```bash
# Fix with files
~/bin/rsi-log fix --summary "reasoning_effort: none in config.yaml" --category server_config --files ~/.hermes/config.yaml

# Fix with lesson
~/bin/rsi-log fix --summary "systemd KillMode=control-group" --category server_config --files /etc/systemd/system/llama-server.service
~/bin/rsi-log lesson --text "llama-server with tee pipe: default KillMode=process leaves children alive holding VRAM. Must use KillMode=control-group and RestartSec=30 for CUDA release." --category server_config --severity critical

# Pattern
~/bin/rsi-log pattern --description "Config files accumulate stale entries (e.g., dead cron jobs in config.yaml)" --status observed --severity medium
```

## Verification
After logging, check:
```bash
tail -1 ~/.hermes/logs/rsi_audit.jsonl | python3 -m json.tool
```

## Anti-patterns (DON'T)
- Don't log manually (copy-paste JSON into files)
- Don't skip logging "small" fixes
- Don't batch-logged at end of session (forget, incomplete)
- Don't use vague summaries ("fixed things")
- Don't only report failures — also report surprises, patterns, and interesting findings
- Don't ignore unexpected behaviors — they often reveal deeper patterns

## When to Report Surprises
Report a surprise when:
- Something behaves differently than expected (even if it works)
- You notice a pattern that wasn't in your mental model
- A tool or system does something clever or unexpected
- There's a discrepancy between documentation and reality
- You find something worth investigating later
- A "bug" turns out to be a feature or design choice
- Two systems interact in an unexpected way

Signal levels:
- `high` — reveals a fundamental misunderstanding, affects multiple systems, worth deep investigation
- `medium` — interesting pattern, might be useful later
- `low` — minor curiosity, worth noting but not urgent

Add `--investigate` when you want to revisit this later. High-signal surprises are also added to the learning_db patterns table.
