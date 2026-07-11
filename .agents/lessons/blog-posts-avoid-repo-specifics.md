---
name: blog-posts-avoid-repo-specifics
description: Blog posts about tooling/workflow should describe the general approach, not name specific repos or internal folder paths
metadata:
  type: feedback
---

When drafting a blog post about a technique, tool, or workflow the user built, describe it generally — do not name the specific repo it lives in or reference its internal folder/file paths (e.g. `docs/spec/`, `scripts/quality-gate`, `.quality-reports/`).

**Why:** The user corrected a draft that was written as a tour of one particular repo (`coding-agent-baseline`), full of its folder layout and exact filenames. They want posts to read as "here's an approach that works," reusable by any reader, not "here's what's in my repo."

**How to apply:** Keep concrete technical details (tool names, config concepts, illustrative snippets) since they add credibility, but phrase them as generic patterns ("a baseline file for suppressed findings") rather than literal paths tied to one project's structure. Never mention the source repo's name in post prose.
