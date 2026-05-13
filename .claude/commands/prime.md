---
description: Prime a new session — load global rules, inventory PRDs and stories, summarize git state
argument-hint: (no arguments)
---

# /prime — Session priming

You are starting (or resuming) work on this Agentic Rails project. Your job in this command is **not** to make changes. Your job is to load enough context to start the next PIV phase cleanly, and to brief the user.

## Steps — do these in parallel where possible

1. **Read the global rules.** Read `CLAUDE.md` at the repo root. Internalize the tech stack, validation gate, and PIV workflow. Note especially the forbidden technologies and the "no 'stakeholders'" terminology constraint.

2. **Inventory the PIV state.**
   - List files in `.agents/PRDs/` (each is a feature intent doc).
   - List subdirectories of `.agents/stories/` (each corresponds to a PRD slug) and the story files inside.
   - For each story file, note its status header (`Status: open | in-progress | done`) without reading the full body.

3. **Survey git state.**
   - `git status` (do not pass `-uall`).
   - `git log --oneline -10` to see recent work.
   - Current branch via `git branch --show-current`.

4. **Confirm the toolchain is reachable.** Quickly check:
   - `bundle --version`
   - `bin/rails --version` (only if `bin/rails` exists)
   - `yarn --version`
   Do not run `bundle install` or any setup — just verify the binaries respond. If any are missing, surface that.

## Output — brief the user in this exact shape

```
## Session primed

**Branch:** <branch> (<N> commits since main, <clean | uncommitted changes>)

**Stack:** Rails <version>, Ruby <version>, PostgreSQL, Bootstrap, Stimulus, Minitest, RuboCop

**PRDs (.agents/PRDs/):**
- <slug>.md — <one-line summary pulled from the PRD's title or first heading>

**In-flight stories:**
- <prd-slug>/<NN-task>.md — <status>

**Recent commits:**
- <hash> <subject>
- ...

**Suggested next step:** <one of: /create_prd, /plan_feature <prd-slug>, /implement <story-path>, /commit>
```

Keep this brief — under 30 lines. If a section has nothing (e.g. no PRDs yet), say so in one line rather than listing emptiness.

## Constraints

- Do not modify any files in this command.
- Do not run validation tools (`rubocop`, `rails test`) — that is the `/commit` command's job.
- Do not load any `.claude/reference/*.md` files yet — they are on-demand.
- Do not invent a "next step" if none is obvious; say "ready for `/create_prd`" by default.
