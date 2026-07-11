---
title: "Solving AI code slop with a deterministic quality gate"
tags:
  - AI
  - Coding Agents
  - Claude Code
  - Codex
  - Code Smells
  - Complexity
  - Static Analysis
---

Hey there! This one's a bit different from the AWS posts. I want to talk about how I've set up my coding agents (Claude Code and Codex, mostly) to stop producing what everyone's started calling "AI code slop." Specifically I want to get into the part I've spent the most time getting wrong before getting right: the quality gate, and what "determinism" even means when half your pipeline is an LLM guessing the next token.

## The actual problem

Across my projects I use the same shell for running coding agents: a shared behavior file, a handful of subagents (planner, implementer, code reviewer, bug diagnostician), and a standard way of wiring them together so I don't have to redefine "how should the agent behave here" every single time I start something new.

The workflow looks like this: planner writes a spec, I approve it, implementer writes code in small slices using TDD, red, green, refactor, in a loop, then a code-reviewer subagent looks at the diff and returns APPROVE, WARNING, or BLOCK. BLOCK sends it back to the implementer. That loop is where things kept going wrong for me early on.

LLMs are bad at catching their own mistakes but pretty good at fixing them once something concrete points at the problem. I noticed this over and over: ask an implementer agent to review its own diff and it'll wave everything through. Point a linter or a static analyzer at the same diff and it finds three real issues in two seconds. So the fix isn't "make the model smarter," it's "stop asking the model to do the mechanical part."

That split ended up as three layers, each catching what the one below it can't. Lint handles the boring stuff, formatting, unused symbols, one file at a time, fast. Static analysis is the layer that actually measures whether the codebase is getting worse: complexity, duplication across files, dependency direction, dead code. And agent review is left with the one job tools genuinely can't do, judging whether the abstraction is right, whether something's misnamed, whether the logic is subtly off in a way no linter will ever catch.

None of this touches LLM sampling. There's no seed, no temperature pinning, no replay of tool calls. When I say "determinism" here I mean something narrower: replacing the model's judgment with mechanical, repeatable checks wherever a mechanical check exists, so quality doesn't quietly drift depending on how the agent happened to be feeling that day.

## The ratchet

You can't just turn on strict static analysis against a repo that already has a year of debt in it. Everything trips on day one and the agent (or you) tunes it out. So the gate works on a ratchet instead: measure the current state, commit it as a baseline, and only fail on regressions from there.

Concretely that's a small set of baseline files: one for suppressed static-analysis findings, one holding a duplication-percentage threshold seeded from whatever the codebase measured at when the gate was turned on, and one for grandfathered architecture violations. The gate script itself never touches these. Generating or loosening a baseline is a separate, explicitly approved step, not something that happens automatically mid-review. I learned this one the hard way after watching an agent "fix" a static analysis failure by adding the failing line to the baseline file instead of, you know, fixing it.

## Where I had it wrong: the hook

My first version of this hooked into every single edit. A `PostToolUse` hook fired after any `Edit`, `Write`, or `MultiEdit` call and ran the quality script right then:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "sh quality-gate.sh" }
        ]
      }
    ]
  }
}
```

It sort of worked, but it was noisy and slow. Duplication and architecture checks only mean anything across the whole repo, so running them after every tiny edit was wasted CPU. Worse, firing on every edit instead of once per review made it genuinely unclear whether a given report reflected the current diff or a stale run from ten edits ago. I'd written the comment in the original script myself: "the edit already happened, exit 2 just surfaces feedback, it does not undo the change." That's fine in principle, but a gate nobody can trust the freshness of isn't much of a gate.

So I pulled the hooks out entirely and moved gate execution to be the code reviewer's job: run once, fresh, at the start of every review pass, before any semantic judgment happens.

## What the gate actually runs

For the PHP/Laravel projects I've applied this to, the concrete implementation is a POSIX shell script that wipes the previous report output and runs six checks in a fixed order:

| Check | Tool | Report |
|---|---|---|
| style | Pint | `style.txt` |
| static | PHPStan/Larastan | `static-analysis.json` |
| tests | Sail (`sail artisan test`) | `tests.txt` |
| duplication | jscpd | `duplication.txt` |
| architecture | Deptrac | `architecture.json` |
| audit | `composer audit` | `dependency-audit.json` |

Tests go through Sail on purpose, not a bare `phpunit` call, so the suite runs inside the same container with the same PHP version and services (database, cache, queue) the app actually uses in production. That's the one spot where "determinism" means something closer to its usual sense: same environment every run, not just same logic.

Every run also writes a manifest with a status per check (`pass`, `fail`, or `skipped`) and an exit code. If a tool isn't on the machine, or a script isn't wired up yet, the check gets marked `skipped` rather than failing hard, that's on purpose so a half-set-up project doesn't get permanently stuck. But here's the part I keep having to repeat to myself when writing reviewer instructions: exit code 0 does not mean clean. A skipped check is still missing verification, even though the script exits happily. The reviewer has to read the manifest, not just check the exit code.

## How the reviewer actually uses it

Having six reports sitting around doesn't do anything by itself. The code-reviewer subagent has to know what counts as a problem, and "any finding at all" turns out to be the wrong answer. A codebase with a year of history has existing complexity and duplication everywhere; if every pre-existing smell blocked every PR, nothing would ever ship.

So the reviewer applies scope discipline on top of the raw findings. Anything new introduced by the diff, a new function, a new class, has to come in clean: no new duplication, complexity under the threshold, no architecture violations. Anything the diff touched but didn't fully rewrite can't get worse than it already was, that's the regression check. But if a piece of code is explicitly called out as in scope, because the spec named it, or the bug report pointed at it, or the refactor brief is literally about that file, then every finding in it has to be resolved, not just "not worsened." Reducing a function's cognitive complexity from 40 to 25 still blocks the review if the threshold is 20 and that function was the whole point of the change, unless I explicitly signed off on a partial-reduction goal going in.

Anything else, pre-existing debt the diff never touched, gets left alone. The reviewer isn't supposed to turn a small bug fix into an opportunistic rewrite of a file it happened to open. That debt stays visible in the ratchet baseline for whenever someone decides to deal with it on purpose.

This is what actually solves the "AI code slop" problem for me, more than any single tool. PHPStan and the cognitive-complexity check catch the smells: god functions, deep nesting, duplicated logic jscpd would also flag, dependencies pointing the wrong way per Deptrac's rules. None of that requires the reviewer to be clever. What requires judgment is deciding which of those findings are this diff's responsibility versus the codebase's pre-existing baggage, and that's exactly the kind of call I don't want an agent making unsupervised, so it's spelled out as an explicit policy instead of left to vibes.

## The highest-signal thing to watch for

The single most useful check I've built into the reviewer instructions isn't a code check at all, it's auditing the baseline files themselves. If the static-analysis baseline, the duplication threshold, or the architecture baseline changed in a diff, that's suspicious by default. An agent under pressure to get a green gate has an easy way out: instead of fixing the flagged issue, just add it to the suppression list. The gate goes green either way. So the reviewer is explicitly told to diff the baseline files first, before looking at anything else, and treat any change there as BLOCK unless the scope specifically called for it.

