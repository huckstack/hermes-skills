---
name: gpu-task-isolation
description: A protocol for running heavy GPU workloads on specific, idle hardware in multi-GPU environments to prevent resource contention and timeouts.
---
# GPU Task Isolation Protocol

Use this skill when performing heavy ML/AI tasks (like Whisper transcription or model fine-tuning) in environments with multiple GPUs where the default device (usually index 0) is heavily utilized.

## Workflow

### 1. Hardware Audit
Before launching a heavy task, inspect the current state of the GPU cluster to find an idle device.
**Command:**
`nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total --format=csv,noheader,nounits`

### 2. Device Isolation
Once an idle index is identified (e.g., index `1`), isolate the task to that specific card. This prevents the task from competing for VRAM or compute cycles on the primary card.

**Bash/Terminal Execution:**
`export CUDA_VISIBLE_DEVICES=1 && [your_command_here]`

**Python/Subprocess Execution:**
If running via a script, inject the environment variable into the subprocess:
`env = os.environ.copy(); env["CUDA_VISIBLE_DEVICES"] = "1"; subprocess.Popen(cmd, env=env, ...)`

### 3. Resilient Monitoring
To avoid "Empty Response" errors or connection timeouts in agentic interfaces, do not run long tasks in the foreground. Use a background process paired with a polling loop to monitor file creation or completion.

## Pitfalls
- **Driver Mismatch**: Ensure the virtual environment's PyTorch installation is compiled with CUDA support that matches the system drivers.
- **Memory Fragmentation**: Even if a card is "idle," ensure the requested model size (e.g., Whisper `medium`) fits within the available VRAM on the target card.
