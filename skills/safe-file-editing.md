---
name: safe-file-editing
description: Protocol for safely modifying existing files without corruption. Covers when to use patch, sed, write_file, and execute_code safely. Includes the Fresh Rebuild Strategy for when reconstruction fails.
version: 1.1.0
tags: [files, editing, safety, tools, reconstruction]
---

# Safe File Editing

## The Golden Rule

**NEVER use `execute_code` with `read_file` + `write_file` to modify existing files.** The `read_file` output contains line-number prefixes (e.g., `     1|content`) that corrupt the content when split/joined in Python.

## Tool Selection Guide

| Edit Type | Tool | Why |
|-----------|------|-----|
| Targeted replacement (1-3 lines) | `patch` (replace mode) | Fuzzy matching, verified, safe |
| Simple line-level change | `terminal` with `sed` | Reliable for single-line replacements |
| Full file rewrite | `write_file` | ONLY when writing NEW files or when you have the COMPLETE original content |
| Complex multi-file edits | `execute_code` with `patch` or `sed` | Never with `read_file`+`write_file` |

**User preference:** For JS files with single quotes that break `patch` JSON parsing, prefer `write_file` (complete rewrite) or `sed` over `patch`.

## Corruption Pattern #1: Line-Number Prefixes (DO NOT REPEAT)

```python
# WRONG — line-number prefixes corrupt content:
r = read_file(path="file.js")      # Returns "     1|line1\n     2|line2\n..."
lines = r['content'].split('\n')   # Splits on \n but line numbers stay embedded
lines[10:20] = new_content         # Modifies slice
write_file(path="file.js", content='\n'.join(lines))  # Writes corrupted content
```

The line-number prefixes (`     1|`, `     2|`, etc.) get mixed into the content, causing data loss and corruption.

## Corruption Pattern #2: Truncated Content (DO NOT REPEAT)

```python
# WRONG — read_file has line limits and may truncate:
r = read_file(path="large_file.js")  # May return only first N lines
# If truncated, you're missing data. Modifying and writing back COLLAPSES the file.
lines = r['content'].split('\n')
lines[10:20] = new_content
write_file(path="large_file.js", content='\n'.join(lines))  # File collapsed to partial content
```

**This is the silent killer.** When `read_file` returns truncated content (due to line limits, 8000+ char snapshot limits, or other truncation), you have NO IDEA data is missing. Modifying the partial content and writing it back destroys the parts you didn't read.

**Detection:** Compare `r.get('total_lines', 0)` or file size before/after. If the file was 1098 lines and you only read 500, you're missing 598 lines.

**Rule:** If `read_file` truncated (check `total_lines` vs expected), NEVER use `write_file` on that content. Use `sed` or `patch` for targeted edits instead.

## Correct Approaches

### 1. `patch` (replace mode) — Preferred for targeted edits
```
# Find the exact string and replace it. Works with fuzzy matching.
```

### 2. `sed` via `terminal` — For simple line-level changes
```bash
# Replace a specific string in a file
sed -i 's/old_string/new_string/g' file.js

# Delete specific lines
sed -i '415,427d' file.js

# Fix indentation on specific lines
sed -i '471s/^           /            /' file.js
```

### 3. `write_file` — Only for complete rewrites
```
# Safe ONLY when:
# - Writing a NEW file (doesn't exist yet)
# - You have the COMPLETE original content (not via read_file)
# - You're intentionally replacing the entire file
```

### 4. `execute_code` with `patch` — For complex edits
```python
from hermes_tools import read_file, patch

# Read to find the location
r = read_file(path="file.js")

# Use patch() for the actual edit (not write_file)
patch(path="file.js", old_string="old code", new_string="new code")
```

## Verification Step

After ANY file edit:
1. Check file size: `ls -la file.js` — did it change by expected amount?
2. Spot-check a few lines: `head -5 file.js` and `tail -5 file.js`
3. If file shrunk dramatically → RESTORE IMMEDIATELY (git checkout, backup, or re-read from source)

## Recovery

If a file got corrupted:
1. Check `git checkout -- file` first
2. Check for backups (previous session's read_file content, local copies)
3. If no recovery available: ask the user — DO NOT guess the content back
4. For reconstruction from memory: be honest about what you remember vs. what you're guessing. Mark reconstructed sections clearly.

## Corruption Pattern #3: write_file Truncation (DO NOT REPEAT)

```python
# WRONG — write_file can silently truncate very large content:
write_file(path="large_file.js", content=very_long_string)  # May only write first N chars
```

**Detection:** After write_file, immediately check `ls -la` or `wc -c` to verify file size matches expected. If the file is significantly smaller than expected, the write was truncated.

**Rule:** For files >~500 lines, prefer `sed` for targeted edits or `patch` for multi-line changes. Only use `write_file` for complete rewrites when you have verified full content.

## Corruption Pattern #4: Heredoc and Python Script Truncation (DO NOT REPEAT)

```bash
# WRONG — heredoc in terminal also truncates at ~30KB:
cat > file << 'EOF'
very_long_content_here...
EOF

# WRONG — writing a Python script to disk and executing it also truncates:
write_file(path="/tmp/build.py", content=very_long_python_script)  # Truncated
terminal(command="python3 /tmp/build.py")  # Runs truncated script
```

**Root cause:** The write_file tool has a hard limit of ~30KB. This affects ALL write_file calls, regardless of whether the content is HTML, Python, bash, or any other format. Heredocs via terminal inherit this because the content is passed through the same tool chain.

**Detection:** After any large write, immediately check file size with `wc -c` or `wc -l`. Compare to expected size.

**Rule:** If you need to write a file >30KB:
1. Ask the user to provide the content (paste in chunks)
2. Use `sed` for targeted edits instead of full rewrites
3. If you must build a large file, write a small Python script that downloads/generates the content (don't embed the content in the script)
4. Verify file size after every write

## Corruption Pattern #5: Reconstruction from Memory

```python
# WRONG — reconstructing large files from memory produces incomplete results:
# You remember the structure but miss details, variable names, edge cases.
# Result: file has correct skeleton but broken functionality.
```

**Rule:** NEVER reconstruct a file from memory if it has >~300 lines. The probability of subtle bugs is too high. Instead:
1. Ask the user to provide the original file (paste in chunks)
2. Check for backups (local machine, other repos, cloud storage)
3. If you must reconstruct, clearly mark reconstructed sections and test thoroughly

## The Fresh Rebuild Strategy (When Reconstruction Fails)

When a file is broken beyond repair (corrupted, truncated, or unrecoverable), **don't try to patch or reconstruct it.** Instead:

1. **Audit what exists** — List all files, check sizes, identify what works
2. **Keep working code** — server.py, orchestrator.py, configs, etc. are fine
3. **Rewrite broken code fresh** — Write new files from known-good patterns
4. **Commit and push immediately** — Version control the working state
5. **Set up backups** — Cron, hooks, or manual before continuing

**Why this is faster:**
- Reconstruction from memory is error-prone and slow (trial-and-error)
- Patching broken code compounds errors
- Fresh build uses known-good patterns + what we learned
- Git gives you a clean starting point

**When to use:**
- User says "redo" or "start fresh"
- File is corrupted and git doesn't have a good version
- You've hit 3+ failed reconstruction attempts
- The broken code is a symptom of a deeper issue (tool limitation, wrong approach)

**Example:** MusicVisualizer index.html was 293 lines (should be 1098). Instead of trying to reconstruct from memory, we audited what worked (server.py, orchestrator.py, watcher.sh), rewrote index.html fresh with all 6 modes from known-good patterns, committed to git, and set up backups. Done in one pass.

## False Positive: Regex Literals in Brace Counting

Simple `{`/`}` counting fails on regex literals like `/^[^{]*\\{/`. These contain braces inside regex delimiters that are NOT code braces.

**Fix:** Strip regex literals before counting, or use a proper parser. A quick check: `python3 -c "import re; c=open('f').read(); c2=re.sub(r'/\\^?\\[.*?\\]/[gim]*','R',c); print('OK' if c2.count('{')==c2.count('}') else 'MISMATCH')"`

## JS Name Collision Trap (SILENT BROWSER FAILURE)

**Problem:** Declaring a `const` variable and a `function` with the same name causes a `SyntaxError: Identifier 'X' has already been declared` in Node.js, but in browsers the error may be silent — the script fails to execute entirely with no console errors.

**Example:**
```javascript
const debugLog = document.getElementById('debug-log');  // line 106
// ... 400+ lines of code ...
function debugLog(msg) { ... }  // line 566 — COLLISION!
```

**Detection:** Run Node.js validation before deploying:
```bash
node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*?)<\/script>/);try{new Function(m[1])();console.log('JS OK')}catch(e){console.error('JS ERROR:',e.message)}"
```
If error mentions "already been declared", you have a name collision.

**Fix:** Use distinct names:
- `const` for DOM element references: `debugLog`, `statusEl`, `btnPlay`
- `function` for logic: `addDebugLog()`, `updateStatus()`, `togglePlay()`

**Prevention:** When writing JS, never reuse a name between a `const`/`let`/`var` declaration and a `function` declaration. Keep a mental list of your top-level declarations.

## JS Regex with `}` Breaking HTML Parsing

**Problem:** Regex patterns containing `}` (like `/\};?\s*$/`) can break JavaScript parsing when embedded in HTML. The `}` inside the regex may be misinterpreted by parsers.

**Fix:** Use string methods instead of regex for stripping wrappers:
```javascript
// WRONG — regex with } can break parsing:
let jsonStr = data.replace(/\};?\s*$/, '}');

// RIGHT — use string methods:
let ci = jsonStr.lastIndexOf('}');
if (ci > 0) jsonStr = jsonStr.substring(0, ci + 1);
```

## JS Validation Before Deploy (Quick Checklist)

Before deploying any HTML file with embedded JS:
1. [ ] Run Node.js syntax check: `node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*?)<\/script>/);try{new Function(m[1])();console.log('OK')}catch(e){console.error(e.message)}"`
2. [ ] Check for name collisions: any `const X` and `function X` pair?
3. [ ] Check regex patterns: any `}` or `{` inside regex literals?
4. [ ] Verify browser loads: open in browser, check console for errors
5. [ ] Verify functionality: check that key functions are defined (`typeof loadTracks !== 'undefined'`)

## Browser Caching Gotcha

`?v=$(date +%s)` does NOT work in `browser_navigate` URLs — the shell doesn't expand `$()` inside the tool call. Use a static value like `?v=2` or `?nocache=1` instead. The browser tool may cache aggressively, so always use a cache-busting query param after file changes.

## Lyrics Data Loading Pattern

When the orchestrator generates `master_data.json` (not `.js` files), the visualizer must:
1. Derive project_id from filename: `track.filename.replace('.mp3','').replace(/ /g,'_').toLowerCase()`
2. Fetch: `/processed/<project_id>/master_data.json`
3. Parse lyrics from `parsed.lyrics` field (not `parsed.segments`)
4. Store in `lyricsSegments` array for `updateLyrics()` to match against audio currentTime

## Prevention Checklist

Before modifying any file:
- [ ] Is this a file I have COMPLETE content for? (Check `total_lines` from read_file)
- [ ] Would `patch` or `sed` handle this edit safely?
- [ ] If using `write_file`, do I have the full original content?
- [ ] Will I verify file size after writing?
- [ ] Is this file >500 lines? If yes, prefer `sed`/`patch` over `write_file`
- [ ] Am I reconstructing from memory? If yes, ask the user for the original first
- [ ] If reconstruction failed 3+ times, switch to Fresh Rebuild Strategy

## Tool Limitations Summary

| Tool | Limit | Workaround |
|------|-------|------------|
| `write_file` | ~30KB per call | Use `sed`/`patch` for targeted edits; ask user for content |
| `read_file` | ~500 lines per call | Use `offset`/`limit` to read in chunks; check `total_lines` |
| `patch` (replace) | JS single quotes break JSON | Use `sed` or `write_file` (complete rewrite) instead |
| `cat > file << 'EOF'` | ~30KB (same tool chain) | Same as write_file — use targeted edits |
| `cronjob` script path | Must be relative, in `~/.hermes/scripts/` | Copy script there first, use just the filename |