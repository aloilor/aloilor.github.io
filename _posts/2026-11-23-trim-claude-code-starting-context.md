---
title: "Trim Claude Code's starting context: deny the tools you never use"
tags:
  - AI
  - Coding Agents
---

Before you type a word, a Claude Code session already holds thousands of tokens. Most of them are tool definitions and skill listings. Some of those tools are useless in a coding session, and every turn pays for them anyway. This post explains how I cut that starting context with two lines in `settings.json`.

## Run `/context` before you blame the model

Start a fresh session and run `/context`. It breaks the window down by source: system prompt, tools, MCP servers, memory files, and skills. Read the tools row first.

On my Windows machine, two entries stood out:

- **`Artifact`**, the tool that publishes HTML pages to claude.ai. Its description is long, and it brings a set of skills with it: design guidance, diagramming, and runtime capabilities. Each skill adds a listing line to every session.
- **`PowerShell`**, a second shell tool next to `Bash`. Claude Code on Windows loads both, each with its own description and its own syntax rules.

<!-- TODO: add before and after token counts from /context -->

I write code in these sessions. I don't publish web pages from them, and I don't want two shells.

## A deny rule removes the tool, not only the permission

A rule in `permissions.deny` that names a whole tool does more than block calls. Claude Code drops the tool from the session, so its definition never reaches the model. The skills that exist to serve that tool lose their reason to load.

Here is the block I add to `.claude/settings.json` in each project:

```json
{
	"permissions": {
		"deny": [
			"Artifact",
			"PowerShell"
		]
	}
}
```

Merge it with any deny rules you already have. Don't replace them.

## Why deny `PowerShell` and keep `Bash`

Token savings were the first reason. Consistency turned out to be the better one.

With both shells loaded, the agent picks one per command. It writes `$env:VAR` in one call and `$VAR` in the next. A script it writes on Windows breaks on a teammate's Mac. With `PowerShell` denied, every command runs through Git Bash, so commands, scripts, and docs use the same POSIX syntax on every platform.

## Why deny `Artifact`

A coding session needs files on disk, not hosted pages. The `Artifact` tool and its skills cost context on every turn and help with none of them. When I want a page, I open a separate chat for it.

## Pair the trim with a compaction limit

A smaller start leaves more room for the work, but long sessions still fill up. I also set `autoCompactWindow` per model under `modelSettings`, so a session compacts before it gets slow and expensive:

```json
{
	"modelSettings": {
		"claude-opus-5-5": { "autoCompactWindow": 450000 },
		"claude-sonnet-5-5": { "autoCompactWindow": 350000 },
		"claude-haiku-5-5": { "autoCompactWindow": 250000 }
	}
}
```

## Audit your own list

My two tools won't match yours. Run `/context` in a fresh session, find every tool, MCP server, and skill you never call, and deny it. Then run `/context` again and compare. If a deny rule removes something you need, delete the line and restart the session.
