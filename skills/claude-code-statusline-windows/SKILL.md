---
name: claude-code-statusline-windows
description: Use when configuring Claude Code statusLine on Windows with PowerShell, especially when statusLine shows blank/empty or causes terminal layout corruption on exit.
---

# Claude Code StatusLine on Windows (PowerShell)

## Overview

Configuring `statusLine` in Claude Code on Windows has several non-obvious pitfalls that cause the status bar to appear blank. Core principle: **invoke the script with `-Command` mode and read the stdin JSON using `[Console]::In.ReadToEnd()`.**

## Symptoms

- Status bar is completely blank (nothing displayed)
- Terminal layout corrupts on Claude Code exit → **the script IS being called**, but output is not captured
- Script works fine when run manually

## Three Key Pitfalls

### Pitfall 1: `Write-Host` does not write to stdout

Claude Code captures **standard output (stdout)**. `Write-Host` writes to the host console stream, not stdout.

```powershell
# ❌ Wrong - Claude Code cannot read this
Write-Host -NoNewline "status text"

# ✅ Correct - goes to stdout
Write-Output "status text"
# or simply output the string directly
"status text"
```

### Pitfall 2: stdin reading is unreliable in `-File` mode

Claude Code passes context data as JSON via stdin. When running a script with `-File` mode, both `[Console]::In.ReadToEnd()` and `$input` inside `try/catch` blocks behave unreliably.

**Fix: use `-Command` mode to invoke the script file:**

```json
// ~/.claude/settings.json
{
  "statusLine": {
    "type": "command",
    "command": "pwsh -NoProfile -NonInteractive -Command \"& 'C:\\Users\\YourName\\.claude\\statusline.ps1'\""
  }
}
```

Inside the script, read stdin with `[Console]::In.ReadToEnd()`:

```powershell
$json = $null
try {
    $raw = [Console]::In.ReadToEnd().Trim()
    if ($raw) { $json = $raw | ConvertFrom-Json }
} catch { }
```

### Pitfall 3: Context comes from stdin JSON, not environment variables

Claude Code does **not** inject `CLAUDE_MODEL` or similar environment variables. Context is passed via stdin:

```json
{
  "model": { "id": "claude-sonnet-4-5", "display_name": "Sonnet" },
  "workspace": { "current_dir": "C:/Users/..." },
  "context_window": {
    "context_window_size": 200000,
    "current_usage": {
      "input_tokens": 8500,
      "cache_creation_input_tokens": 5000
    },
    "used_percentage": 6.75
  }
}
```

## Complete Working Template

```powershell
# ~/.claude/statusline.ps1 - invoke via -Command mode

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# ANSI colors
$reset    = "`e[0m"
$bold     = "`e[1m"
$dimPath  = "`e[38;5;244m"
$colDir   = "`e[38;5;75m"
$colGit   = "`e[38;5;214m"
$colDirty = "`e[38;5;196m"
$colModel = "`e[38;5;141m"

function Get-TokenColor($pct) {
    if ($pct -lt 30)  { return "`e[38;5;78m"  }   # green
    if ($pct -lt 60)  { return "`e[38;5;220m" }   # yellow
    if ($pct -lt 80)  { return "`e[38;5;208m" }   # orange
    return "`e[38;5;196m"                           # red
}

# Read stdin JSON (-Command mode required for reliable stdin reading)
$json = $null
try {
    $raw = [Console]::In.ReadToEnd().Trim()
    if ($raw) { $json = $raw | ConvertFrom-Json }
} catch { }

# Current directory (truncated to last 2 segments)
$cwd = if ($json?.workspace?.current_dir) { $json.workspace.current_dir } else { (Get-Location).Path }
$parts = $cwd -split '[/\\]' | Where-Object { $_ -ne '' }
$dirDisplay = if ($parts.Count -le 2) {
    "${colDir}${bold}$cwd${reset}"
} else {
    "${dimPath}...\$($parts[-2])\${reset}${colDir}${bold}$($parts[-1])${reset}"
}

# Git branch and dirty state
$gitPart = ""
try {
    $branch = git -C $cwd rev-parse --abbrev-ref HEAD 2>$null
    if ($branch) {
        $dirty = if (git -C $cwd status --porcelain 2>$null) { "${colDirty}*${reset}" } else { "" }
        $gitPart = "  ${colGit}${branch}${reset}${dirty}"
    }
} catch { }

# Model name
$modelName = if ($json?.model?.display_name) { $json.model.display_name }
             elseif ($json?.model?.id) { $json.model.id -replace '^claude-', '' }
             else { "?" }

# Token usage
$tokenPart = ""
if ($json?.context_window) {
    $ctx = $json.context_window
    $size = [int]$ctx.context_window_size
    $used = [int]$ctx.current_usage.input_tokens + [int]$ctx.current_usage.cache_creation_input_tokens
    if ($size -gt 0) {
        $pct = [math]::Round($used / $size * 100, 0)
        $col = Get-TokenColor $pct
        $tokenPart = "  ${col}$([math]::Round($used/1000,1))k/$([math]::Round($size/1000,0))k (${pct}%)${reset}"
    }
}

Write-Output " $dirDisplay$gitPart  ${colModel}${modelName}${reset}$tokenPart "
```

## Debugging Tips

**Verify the statusLine mechanism works first (use cmd):**

```json
{ "statusLine": { "type": "command", "command": "cmd /c echo STATUS_OK" } }
```

Restart Claude Code — if `STATUS_OK` appears, the mechanism works and the problem is in your script.

**Terminal layout corruption on exit = script is running but output is being swallowed** (the script runs; the output just isn't captured).

## Quick Troubleshooting Table

| Symptom | Cause | Fix |
|---------|-------|-----|
| Blank status bar, clean exit | Script not being executed | Check settings.json format |
| Blank status bar, layout corrupts on exit | Script runs but output is swallowed | Check Pitfalls 1 & 2 |
| Garbled characters | Encoding issue | Add `[Console]::OutputEncoding = UTF8` |
| Shows `unknown` for model | Reading env vars instead of stdin | Use stdin JSON approach (Pitfall 3) |
| JSON not read in `-File` mode | PowerShell stdin behavior difference | Switch to `-Command` mode |
