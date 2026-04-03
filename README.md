# ClaudeCodeStatusLineWindows

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for configuring `statusLine` on Windows with PowerShell.

## What It Solves

Configuring `statusLine` in Claude Code on Windows has several non-obvious pitfalls that cause the status bar to appear blank or the terminal layout to corrupt on exit. This skill documents the three key pitfalls and provides a ready-to-use template.

### Common Symptoms

- Status bar is completely blank
- Terminal layout corrupts on Claude Code exit (script is running, but output is not captured)
- Script works fine when run manually, but not via Claude Code

## Quick Setup

1. Save the PowerShell script from the skill template to `~/.claude/statusline.ps1`
2. Add the following to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "pwsh -NoProfile -NonInteractive -Command \"& 'C:\\Users\\YourName\\.claude\\statusline.ps1'\""
  }
}
```

3. Restart Claude Code

The status line displays: working directory, git branch (with dirty indicator), model name, and token usage with color-coded percentage.

## Three Key Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| `Write-Host` | Does not write to stdout | Use `Write-Output` or string output |
| `-File` mode | stdin reading is unreliable | Use `-Command` mode to invoke the script |
| Environment variables | Claude Code does not inject env vars | Read context from stdin JSON |

## Installation as a Skill

Copy `skills/claude-code-statusline-windows/` to your Claude Code skills directory:

```
~/.claude/skills/claude-code-statusline-windows/SKILL.md
```

## License

MIT
