---
title: "Coding agent conventions: grill the team, don't scan the repo"
tags:
  - AI
  - Coding Agents
---

A coding agent can read your whole repo in seconds. It still can't tell you which of your two error-handling patterns is the one you want. This post explains why I write agent conventions by interviewing a person instead of scanning the code, and what a skill that does this looks like.

## A scan records what is common, not what is right

Many people write the code in a repo. Some of them wrote it in a hurry, and some of them left years ago. When an agent scans that code and writes "the conventions", it records the most frequent pattern. Frequency says nothing about quality.

Say 12 services throw raw exceptions and 3 return a `Result` type. The team agreed last year that new code returns `Result`. A scan writes "services throw exceptions" and turns the bad pattern into policy. Every agent that reads the file then copies it.

So the rule I follow is simple to state. The repo is evidence. The person is the authority.

## How the skill works

The skill, `define-conventions`, runs as an interview. I call it a grill, because it asks one focused question at a time until the answer is clear. It goes through four steps:

1. It scans the repo, the README, the contributing guide, and the CI config for evidence.
2. It shows the areas it plans to cover and asks what agents or new contributors keep getting wrong.
3. It grills one area at a time: change process, testing, deployment, then coding standards.
4. It drafts each file, shows it, and writes it only after you confirm.

Each area becomes its own file, so an agent that makes a commit doesn't load the deployment rules. A file exists only when it has content. A library with no deployments gets no deployment file.

## The evidence must not pick the answer

A grill usually words each question so "yes" accepts its recommendation. That saves time, but it has a trap. If the recommendation comes from frequency, "yes" makes the common pattern official, good or bad.

The skill avoids this on questions about what *should* be true. It shows the split ("12 files do X, 3 do Y"), recommends on quality, and asks an open question when no option is good. It can also propose a rule the code doesn't follow yet. Only the person can adopt it.

A rule like that needs a scope, or agents misread it. Some agents distrust a rule the code ignores. Others start to migrate old code that nobody asked them to touch. A scoped rule prevents both: "New code returns `Result`. The legacy module throws. Don't copy it, and don't migrate it unless asked."

## Every line has to earn its place

A conventions file costs tokens on every read, and agents follow what they read. A useless line is worse than a missing one. The skill keeps a line only if a competent agent, reading the repo for its task, would otherwise do one of three things:

- guess wrong
- spend real effort to find the answer
- copy the wrong pattern

Here is how that test sorts some typical lines:

| Fails | Passes |
| --- | --- |
| Tests run with `npm test`. | `test:integration` must also pass before merge, though CI skips it. |
| Use meaningful variable names. | Services return `Result`. Only controllers turn errors into HTTP responses. |
| We use GitHub Actions. | A merge to `main` deploys to stage. Prod needs a manual approval. |

The lines on the left are true, but the agent can find them in `package.json` or the CI config, or would do them anyway. The lines on the right are invisible in the code.

A few more limits follow from the same idea:

- No directory maps or stack overviews.
- No links to other repo documents. Agents don't reliably follow links, and links rot. Each rule stands alone.
- No secrets. The file names environment variables, never their values.
- No commands that the person didn't state or confirm. The skill never runs anything that changes state.

## What the research says

The case against generated context files is fairly strong:

- Gloaguen et al., in [*Evaluating AGENTS.md*](https://arxiv.org/abs/2602.11988), found that LLM-generated context files slightly reduced task success and raised cost by 20% to 23%. Developer-written files did significantly better than generated ones. The generated files mostly restated existing documentation.
- The same study found that codebase overviews didn't help agents reach the right files sooner.
- Lulla et al. ([arXiv:2601.20404](https://arxiv.org/abs/2601.20404)) found 28.64% lower median runtime and 16.58% fewer output tokens with developer-written `AGENTS.md` files, with similar completion rates.
- Addy Osmani, in [*Stop Using /init for AGENTS.md*](https://addyosmani.com/blog/agents-md/), recommends that you include only what an agent can't discover from the code.

None of these studies tests conventions written by interview, so treat the case for this skill as a direction, not a result.

## Why this matters

I went through three earlier versions before this one. The first generated architecture overviews, which the research says don't help. The second filled bare templates, and the agent wrote what is usually true for the stack. The third gathered evidence carefully but still treated the repo as the source of truth, so it wrote down existing practice whether or not that practice was any good.

Every version failed for the same reason. It asked the code a question that only a person can answer. The code shows what the team did. It can't show what the team meant, which of two patterns is current, or what should stop spreading.

An agent copies whatever it sees most. The conventions file is the one place where you can tell it otherwise, so it should hold your intent, not a summary of your history.

If you want to check the trade-off in your own project, run a handful of real tasks with and without the conventions files. Compare the tokens, the steps, and the results.
