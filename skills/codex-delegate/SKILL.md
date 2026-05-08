---
name: codex-delegate
description: Delegate tasks to the local Codex CLI executable. Activate when the user message starts with "交给codex" or explicitly asks to hand off work to Codex. Runs the Codex CLI, waits for completion, checks output, and reports a summary back to the user.
---

# Codex Delegate

## When This Skill Activates

Trigger when the user's message **starts with** "交给codex" (literally "hand to codex"). The remainder of the user message after the trigger phrase is the input to be sent to Codex CLI.

## Workflow

### 1. Extract the input

Strip the trigger prefix "交给codex" from the user message. Trim leading/trailing whitespace. The remaining text is the **task input** for Codex.

Example:
- User: "交给codex 整理flash attention每一代算法的详细介绍"
- Task input: "整理flash attention每一代算法的详细介绍"

### 2. Execute Codex CLI

Run the Codex CLI executable with the task input as a command-line argument:

```
/mnt/c/Users/Administrator/.codex/.sandbox-bin/codex.exe <task-input>
```

Use `exec` with appropriate timeout. Codex may take a while for complex tasks — set timeout to 300 seconds (5 minutes) as a starting point. If it times out, retry with longer timeout or inform the user.

### 3. Wait and monitor

- Capture stdout and stderr from the Codex process
- Monitor for completion (exit code 0 usually means success)
- If the process hangs or takes an unusually long time, consider aborting and reporting partial results

### 4. Check the results

After Codex finishes:
- Check the exit code
- Review the output content
- See if Codex created any files in the workspace (it typically writes to the working directory)

### 5. Summarize and report

Write a concise summary for the user covering:
- What task was given to Codex
- Whether it completed successfully
- Key outputs or files produced
- Any notable findings or issues

## Reference

Codex CLI executable path: `/mnt/c/Users/Administrator/.codex/.sandbox-bin/codex.exe`

This is a cross-platform binary (Windows executable accessed via WSL2). It accepts task instructions as command-line arguments and runs Codex as an interactive agent that can:
- Research topics via web search
- Read and write files
- Generate code and documentation
- Execute commands in a sandboxed environment
