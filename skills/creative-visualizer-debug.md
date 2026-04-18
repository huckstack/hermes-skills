---
name: creative-visualizer-debug
description: A specialized workflow for troubleshooting and deploying interactive, audio-reactive generative art (p5.js/Web Audio API) in a Linux environment. Covers JS debugging, p5.js pitfalls, and deployment.
version: 2.0.0
tags: [creative, p5js, audio, troubleshooting, linux, javascript]
---

# Creative Visualizer Debugging & Deployment

A specialized workflow for troubleshooting and deploying interactive, audio-reactive generative art (p5.js/Web Audio API) within a Linux environment.

## Trigger Conditions
- User reports a visualizer is "not working" or "not reacting" despite hardware being connected.
- User is attempting to deploy interactive web-based art in a headless or remote environment.
- Troubleshooting "silent" audio-reactive elements in a browser.

## Debugging Protocol (The Golden Rule)

**Read the file → one hypothesis → one action → check result. Cycle <60s. If >5min, hypothesis is wrong.**

### Phase 1: Syntax Validation (BEFORE Browser Test)

**Never test broken code in the browser.** Run Node.js validation first:

```bash
node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*?)<\/script>/);try{new Function(m[1])();console.log('JS OK')}catch(e){console.error('JS ERROR:',e.message)}"
```

Common errors caught:
- `Identifier 'X' has already been declared` → Name collision between const and function
- `Unexpected token '<'` → Trying to parse HTML as JS (wrong file)
- `Unexpected identifier` → Syntax error in JS

### Phase 2: Name Collision Check

**Before declaring any function, grep for existing const/let/var with the same name.** DOM references and functions share the same scope.

```bash
grep -n 'const debugLog\|function debugLog' index.html
```

**Fix:** Use distinct names:
- `const` for DOM refs: `debugLog`, `statusEl`, `btnPlay`
- `function` for logic: `addDebugLog()`, `updateStatus()`, `togglePlay()`

### Phase 3: Browser Testing

**Always append `?v=$(date +%s)` to URLs after file changes.** Never assume browser serves latest version.

```bash
open "http://localhost:8123/?v=$(date +%s)"
```

Check:
1. Console for errors: `browser_console`
2. Key functions defined: `typeof loadTracks !== 'undefined'`
3. DOM elements present: `document.getElementById('playlist-select')`

### Phase 4: Targeted Logging

**NEVER add logging inside draw loops, animation frames, render callbacks, or any function called >1/sec.** These flood buffers and make all subsequent debugging impossible.

**One targeted log, checked once, then removed.** Log on state changes only (track load, mode switch, errors).

## Implementation Best Practices

### The "Hardened" Template
When creating audio-reactive art, always include:
- **User Activation Trigger:** Wrap initialization in a `click` event to satisfy browser "Autoplay" policies.
- **Status Logging:** Include a visible `div` (e.g., `#status`) that prints real-time error messages.
- **Async Initialization:** Use `async/await` with `audioContext.resume()` to handle async audio setup.
- **Signal Verification:** Check `analyser` is connected before reading frequency data.

## Pitfalls

### The "Default" Trap
Browsers often default to the built-in laptop mic even if a USB mic is plugged in. Always check the **Recording** tab in `pavucontrol`.

### The "Silent" Error
A script may run perfectly but produce no visual output if the audio context is suspended. Always check the console for `AudioContext` warnings.

### The "ReferenceError" Kill-Switch
In single-file HTML/JS apps, referencing a variable before its declaration (e.g., `let palette = MOOD_PALETTES.default;` where `MOOD_PALETTES` is defined 50 lines later) throws a `ReferenceError` that silently kills the entire script. **Symptom:** UI renders but nothing works — no tracklist, no buttons, no console output. **Fix:** Check `browser_console` for `Cannot access 'X' before initialization` errors. Either move the definition above all references, or use a primitive default value at declaration time.

### p5.js `get()` / `loadPixels()` Silent Failure
In certain browser contexts (headless, specific sandbox configs), p5.js canvas readback operations (`get()`, `loadPixels()`, `updatePixels()`) fail silently — they don't throw, but return `undefined` or corrupt state. **Symptom:** Canvas is completely black/empty, draw() runs but nothing renders. **Fix:** Replace all canvas readback with pure draw calls — use `rect()`, `fill()`, `stroke()` instead of `get()`+`image()`, use random `rect()` calls instead of `loadPixels()`+`pixels[]` manipulation. **Permanent: do not add these back.**

### Vignette Alpha Compounding
Drawing a black `rect()` over the entire canvas in a loop with increasing alpha compounds multiplicatively — after 20 iterations the overlay is ~81% black, completely obscuring all visualization. **Fix:** Either remove full-canvas vignettes, use corner-only gradients, or cap total alpha at ~10-15% max per frame.

### Regex with `}` Breaking JS Parsing
Regex patterns containing `}` (like `/\};?\s*$/`) can break JavaScript parsing when embedded in HTML. The `}` inside the regex may be misinterpreted by parsers. **Fix:** Use string methods instead of regex:
```javascript
// WRONG — regex with } can break parsing:
let jsonStr = data.replace(/\};?\s*$/, '}');

// RIGHT — use string methods:
let ci = jsonStr.lastIndexOf('}');
if (ci > 0) jsonStr = jsonStr.substring(0, ci + 1);
```

### File Editing Safety
- **NEVER use `patch` for JS files with single quotes** — it breaks JSON parsing. Use `sed` or `write_file` (complete rewrite) instead.
- **NEVER use `write_file` on partial content** — if `read_file` truncated, you're missing data. Verify file_size after every write.
- **After 6+ patches in one session, rewrite from scratch** — patching compounds errors. Fresh build is faster.

## Tool Selection for JS Edits

| Edit Type | Tool | Why |
|-----------|------|-----|
| Targeted line change | `sed -i 's/old/new/g' file.js` | Safe, no JSON parsing issues |
| Function rename | `sed -i 's/oldName/newName/g' file.js` | Handles all references at once |
| Delete lines | `sed -i '415,427d' file.js` | Clean line removal |
| Full rewrite | `write_file` | Only when you have COMPLETE content |
| Multi-file edit | `execute_code` with `sed` calls | Never with `patch` on JS |

## Debugging Checklist

Before declaring any function:
- [ ] Run Node.js syntax check
- [ ] Grep for name collisions: `grep -n 'const X\|function X' file.js`
- [ ] Check regex patterns for `{` or `}` inside delimiters
- [ ] Verify browser loads: open with `?v=$(date +%s)`
- [ ] Verify key functions defined: `typeof functionName !== 'undefined'`
