---
name: pre-post-hooks
description: >
  Guide for configuring PreToolUse, PostToolUse, and Stop hooks in Claude Code.
  Use when the user asks about hooks, wants to run commands before/after tool calls,
  needs auto-formatting, auto-linting, validation gates, or mentions settings.json hook config.
---

# Pre/Post Hooks

Claude Code hooks are shell commands that execute automatically in response to tool events. They enable validation, formatting, and guardrails without manual intervention.

## When to Use
- User wants to run a command before or after a tool executes
- User asks about PreToolUse, PostToolUse, or Stop hooks
- User wants to add validation gates or blockers
- User needs to configure hooks in settings.json
- User mentions auto-running commands on file edit, git push, or session end

## Core Patterns

### Hook Types and Lifecycle

```
PreToolUse  --> Tool Executes --> PostToolUse
                                       |
                                  (session ends)
                                       |
                                     Stop
```

- **PreToolUse**: Runs before a tool call. Can block with exit code `2` or a structured JSON denial. Exit code `1` alone is non-blocking.
- **PostToolUse**: Runs after a tool call completes. Used for side effects like formatting or checks.
- **Stop**: Runs before the agent stops. Exit code `2` prevents stopping; use bounded verification to avoid loops.

### Hook Configuration in settings.json

Hooks live under `hooks` in `~/.claude/settings.json`. Event entries contain matcher groups, and each group contains an array of typed hook handlers:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "echo 'About to write' >&2", "timeout": 10 }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          { "type": "command", "command": "echo 'Edited a file' >&2", "timeout": 10 }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "echo 'Session ended' >&2", "timeout": 10 }
        ]
      }
    ]
  }
}
```

See [Anthropic's current hook configuration reference](https://code.claude.com/docs/en/hooks#configuration).

### Matcher Patterns

The `matcher` filters by tool name. A single matcher can select `Edit` or several tools using `|`, such as `"Write|Edit"`. Omit the matcher for `Stop` hooks.

```json
{
  "matcher": "Edit",
  "hooks": [
    { "type": "command", "command": "echo 'Edited file' >&2" }
  ]
}
```

Common tool names are `Write`, `Edit`, `Bash`, `Read`, `Glob`, and `Grep`.

### Hook Input Fields

Command hooks receive event JSON on **stdin**. File paths and tool names are not universally injected as environment variables. Parse `.tool_input.file_path` (for tools that provide it), `.tool_name`, and `.session_id` from that JSON. The documented environment variable `CLAUDE_PROJECT_DIR` names the project root. The shell examples below require `jq`; substitute a suitable JSON parser when it is unavailable.

| JSON input field | Description |
|---|---|
| `.tool_input.file_path` | File path on a Write/Edit hook, if supplied |
| `.tool_name` | Tool being invoked |
| `.session_id` | Session identifier |

### Blocking with PreToolUse

A `PreToolUse` hook blocks with `exit 2`. Exit code `1` without valid decision JSON is **non-blocking**, even when a script intended it as a policy failure. The following `Write` hook blocks Markdown writes and also blocks if its expected JSON input cannot be parsed:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "path=$(jq -er '.tool_input.file_path' 2>/dev/null) || { echo 'Missing file_path' >&2; exit 2; }; case \"$path\" in *.md) echo 'Blocked: Markdown write' >&2; exit 2;; esac",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

This applies to *any* Write of a Markdown path, not just a new file. An alternative is exiting 0 with a valid `hookSpecificOutput.permissionDecision: "deny"` JSON object. Consult the [official per-event exit-code table](https://code.claude.com/docs/en/hooks#exit-code-2-behavior-per-event).

### PostToolUse Side Effects

`PostToolUse` runs after the edit and cannot undo it. Parse the path from the hook event on stdin rather than assuming an undocumented file-path environment variable:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "path=$(jq -r '.tool_input.file_path // empty'); if [ -n \"$path\" ]; then npx prettier --write \"$path\"; fi",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

### Stop Hook for Final Audit

A Stop hook can report a finding without blocking, as in this example. To enforce a verification gate, exit 2 with a clear stderr reason and bound rechecks so Claude cannot be trapped in an endless stopping loop.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "git diff --cached --name-only | xargs -r grep -l 'console.log' >&2 || true",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

## Anti-Patterns

- **Slow hooks on hot paths**: Do not run expensive commands (full test suite, large builds) in PostToolUse on every Edit. It blocks the agent loop. Use targeted, fast checks instead.
- **Blocking reads**: Do not use PreToolUse to block Read or Grep calls. These are non-destructive and blocking them disrupts exploration.
- **Missing error handling**: Hook commands that fail silently hide problems. Always include meaningful error messages in blocking hooks.
- **Hardcoded paths or assumed variables**: Read the event JSON from stdin; do not assume file-path environment variables exist.
- **Too many hooks**: Each hook adds latency to every matching tool call. Keep the hook list focused on high-value checks.

## Quick Reference

| Hook Type | Timing | Can Block? | Use Case |
|-----------|--------|------------|----------|
| PreToolUse | Before tool | Yes (exit 2) | Validation, gates, warnings |
| PostToolUse | After tool | No | Formatting, type-checking, linting |
| Stop | Before stopping | Yes (exit 2) | Bounded final verification, audits |

**Config location**: `~/.claude/settings.json` > `hooks`
**Matcher syntax**: Tool name string, pipe-separated for multiple
**Blocking**: PreToolUse can block with exit 2 or valid JSON denial. Stop can prevent stopping with exit 2. Exit 1 by itself does not block either.
