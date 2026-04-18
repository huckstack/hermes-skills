# Skill: rsi-logging-protocol

## Trigger
After EVERY fix/patch/write_file/cron action. Mandatory. No exceptions.

## Steps
1. After applying a fix, call `rsi-log fix --summary "<what was fixed>" --category <server_config|pipeline|maintenance|memory|cron> --files <comma-separated paths>`
2. If the fix is a lesson learned, also call `rsi-log lesson --text "<lesson>" --category <same> --severity <high|medium|low>`
3. For recurring issues, call `rsi-log pattern --description "<pattern>" --status <observed|applied|resolved> --severity <high|medium|low> --applied "<fix applied>"`

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
