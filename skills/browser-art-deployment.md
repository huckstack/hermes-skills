---
name: browser-art-deployment
description: Workflow for deploying HTML/p5.js creative projects that require a local server to bypass CORS/security restrictions when loading local assets.
---

# Skill: Browser-Based Generative Art Deployment

This skill outlines the workflow for developing and testing interactive HTML/p5.js creative projects that require loading local assets (audio, images, or JSON) which are often blocked by browser security policies.

## Trigger Conditions
- Developing p5.js, Three.js, or vanilla JS art in HTML files.
- Encountering "CORS" or "Cross-Origin" errors in the browser console.
  - Error examples: `Access to XMLHttpRequest at 'file:///...' from origin 'null' has been blocked by CORS policy`.
- Audio or images failing to load when opening an HTML file directly via `file://`.

## Deployment Protocol

### 1. Identify the Root Cause
If assets fail to load, check the browser's Developer Tools (F12) Console. If you see a "CORS" or "Cross-Origin" error, the browser is blocking the request because you are using the `file://` protocol.

### 2. Launch Local Development Server
Instead of double-clicking the `.html` file, serve the directory using a lightweight HTTP server.

**Option A: Python (Recommended)**
Navigate to your project directory in the terminal and run:
```bash
cd /path/to/your/project
python3 -m http.server 8000
```
*Note: If you get `OSError: [Errno 98] Address already in use`, change the port (e.g., `8080`).*

**Option B: Node.js (If installed)**
```bash
npx serve .
```

### 3. Access via Localhost
Open your browser and navigate to the local address:
- `http://localhost:8000/your_file.html`
- Or `http://localhost:8080/your_file.html`

### 4. Asset Pathing
Ensure your HTML code uses relative paths for assets.
- **Correct**: `loadSound('audio/track.mp3');`
- **Incorrect**: `loadSound('/home/user/music/track.mp3');`

## Project-Specific: MusicVisualizer Server
The MusicVisualizer project uses a systemd service (`music-visualizer`) to manage the server.

**The `viz-server` CLI (`~/bin/viz-server`) is BROKEN** — positional args are misparsed (`$1` gets assigned to PORT instead of ACTION). Do NOT use it.

**Correct approach:**
```bash
# Check status
sudo systemctl status music-visualizer
# Restart if needed
sudo systemctl restart music-visualizer
```

**If systemd is not set up, start manually:**
```bash
cd ~/MusicVisualizer
python3 server.py
```

**Default serving:** The visualizer file is named `index.html` (renamed from `visualizer.html`), so `http://localhost:8123/` serves it directly.

**Setting up a systemd service for a new project:**
1. Write the `.service` file to `~/` (NOT `/etc/systemd/system/` — heredocs break due to terminal whitespace preservation)
2. Have user run: `sudo cp ~/myservice.service /etc/systemd/system/ && sudo systemctl daemon-reload && sudo systemctl enable myservice && sudo systemctl start myservice`
3. Verify: `sudo systemctl status myservice --no-pager`
4. Clean up: `rm ~/myservice.service`

## Pitfalls
- **Keeping the server alive**: If you close the terminal running the Python command, the \"website\" will go down and assets will stop loading.
- **Hardcoded Absolute Paths**: Avoid using absolute system paths in your code; always use paths relative to the HTML file's location.
- **viz-server parameter ordering**: The viz-server CLI is BROKEN — do not use it. Use systemd service or manual `python3 server.py` instead.
- **server.py missing**: If `server.py` was deleted, fall back to `python3 -m http.server 8123` in the project directory.
- **JSON structure mismatch**: The orchestrator pipeline writes lyrics under `data.lyrics` (not `data.segments` like raw Whisper). Always check the actual JSON structure produced by your pipeline before coding the consumer. Check `data.lyrics` first, then `data.segments`, then raw array.
- **Path resolution for server root**: When the server serves FROM the project directory, `getWebPath()` must strip the FULL absolute path (`/home/user/MusicVisualizer`), not a partial marker (`/MusicVisualizer`). The result should be `/processed/...`, not `/MusicVisualizer/processed/...`.
- **File naming for default serve**: If the HTML file isn't named `index.html`, the root URL (`/`) will show a directory listing instead of loading the visualizer. Rename to `index.html` for clean URLs.
- **Dynamic tracklists**: Never hardcode a tracklist in JS. Add a `/api/tracks` endpoint to the server that scans the directory for `.mp3` files and returns `{filename, processed}`. The visualizer fetches this on load, populates the dropdown, and disables unprocessed tracks with a `[NOT PROCESSED]` label. Unprocessed tracks should show a hint to run the orchestrator.
