---
name: muscle-run-wrapper-protocol
description: "Protocol for executing high-compute or media-heavy tasks via the dedicated RTX 3060 hardware bridge."
---

# Muscle Run Wrapper Protocol

## 📖 Description
When operating in the RSI-enabled environment, certain tasks (e.g., video rendering, heavy tensor math) are assigned to a dedicated hardware-isolated sandbox (the "Muscle" workspace). To interact with this workspace correctly and avoid environment-specific errors (like `subprocess` being restricted in the main venv), all heavy tasks must be executed via the `muscle_run.sh` wrapper.

## 🚀 When to Use
- **Task Type**: Any task involving `torch`, `cv2` (OpenCV), `ffmpeg`, `magick`, or high-VRAM operations.
- **Error Trigger**: If a task fails with `ImportError` (e.g., `torch` not found) or `NameError: name 'subprocess' is not defined` despite being present in the system, it indicates the task is being run in the restricted "Brain" venv instead of the "Muscle" venv.

## 🛠️ Execution Pattern
Always wrap the target Python script in the-muscle-run shell command:

```bash
# Correct Pattern
~/muscle_run.sh /path/to/your_script.py [args]

# Incorrect Pattern (Will likely fail due to venv restrictions)
python3 /path/to/your_script.py [args]
```

## ⚠️ PITFALLS
- **Environment Mismatch**: Running heavy scripts directly via `python3` will attempt to use the restricted "Brain" venv, leading to `ImportError`.
- **Recursive Loop Risk**: When writing code that calls itself, ensure a `max_retries` counter is implemented to prevent infinite recursion in the event of a logic error.
