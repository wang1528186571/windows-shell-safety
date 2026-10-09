# windows-shell-safety

A skill for AI coding agents (ZCode / Claude Code / any `SKILL.md`-compatible agent) that stops the classic **Windows PowerShell mistakes** agents make — commands that parse fine in bash but silently fail or corrupt data in PowerShell 5.1.

一个给 AI 编码智能体用的 Skill：避免 Windows PowerShell 下的高频翻车操作——在 bash 里没问题的写法，到 PowerShell 5.1 里要么报错、要么**静默执行但结果错误**。

## What it covers

- `&&` / `||` are not valid in Windows PowerShell 5.1 — the #1 agent mistake; correct patterns with `$LASTEXITCODE`
- Console encoding left at GBK → garbled Chinese output; `>` redirection writes **UTF-16LE** in 5.1
- `curl` / `wget` are aliases of `Invoke-WebRequest` in 5.1 — different flags entirely, call `curl.exe`
- `$?` vs `$LASTEXITCODE` confusion; stderr progress output ≠ failure
- Chinese-path silent failures (e.g. OpenCV `imwrite` returning false)
- `Invoke-WebRequest` progress-bar slowdown (`$ProgressPreference`)
- Quoting/escaping rules (backtick, single vs double quotes), prefer `-File` over `-Command`
- Machine-readable output (`ConvertTo-Json`) instead of truncated tables
- A bash-habit → PowerShell equivalent cheat sheet

## Install

Copy the folder into your agent's user-level skill directory:

| Agent | Path |
|---|---|
| ZCode | `~/.agents/skills/windows-shell-safety/` |
| Claude Code | `~/.claude/skills/windows-shell-safety/` |

The skill auto-triggers whenever the agent runs commands on Windows or hits PowerShell errors ("`&&` is not a valid statement separator", garbled output, unrecognized cmdlets) — even when the user never mentions PowerShell.

## License

MIT
