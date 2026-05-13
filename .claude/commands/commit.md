---
description: Run the full validation gate; on green, create one conventional commit
argument-hint: (no arguments — uses current uncommitted changes)
---

# /commit — Validate and commit

Your job is to run the **full** Rails validation gate and, only if it passes cleanly, create one well-formed git commit covering the current uncommitted work. This is the "V" in PIV.

## Steps

1. **Survey what's uncommitted.** In parallel:
   - `git status` (no `-uall`)
   - `git diff` (working tree vs index)
   - `git diff --cached` (staged)
   - `git log --oneline -5` for commit-message style

   If there are no changes, stop and report "nothing to commit".

2. **Identify the story being closed.** Look for an `in-progress` or just-marked-`done` story under `.agents/stories/`. If found, capture its slug and title for the commit message. If not found, ask the user what this commit covers.

3. **Run the validation gate, in this order, stopping on first failure.** Use Bash with one command per call so output is readable:

   ```bash
   bundle exec rubocop
   ```
   Pass criterion: ends with `no offenses detected`. Any `C:` / `W:` / `E:` line is a failure.

   ```bash
   bin/rails test
   ```
   Pass criterion: summary line shows `0 failures, 0 errors`. Skips are acceptable; report their count.

   ```bash
   bin/rails test:system
   ```
   Run only if the story touched views, controllers serving HTML, or Stimulus. Same pass criterion as above. If the test environment cannot launch a browser, surface that and ask whether to proceed.

   ```bash
   bundle exec brakeman --no-pager -q
   ```
   Pass criterion: `No warnings found`. Any new warning is a failure even if old ones already exist — diff against the prior baseline if there's a `brakeman.ignore` file.

   ```bash
   bundle exec bundler-audit check --update
   ```
   Pass criterion: `No vulnerabilities found`. Any CVE entry is a failure.

4. **On any failure:**
   - Report the failing tool, the first offending location (file:line), and a one-paragraph diagnosis.
   - Do **not** commit.
   - Do **not** run `rubocop -A` automatically — propose it, but only run with the user's go-ahead, and re-run the full gate after.
   - Suggest a fix or hand back to the user.
   - Mark the story status back to `in-progress` if it was prematurely flipped to `done`.

5. **On full pass:**

   a. **Choose what to stage.** Prefer named paths over `git add -A`. Stage:
      - Files in `app/`, `config/`, `db/migrate/`, `test/`, `lib/`, `Gemfile`, `Gemfile.lock`, `package.json`, `yarn.lock` that the story touched.
      - The story file under `.agents/stories/` if its status changed.
      - Do **not** stage: secrets (`config/credentials/*`, `.env*`), local-only files, generator outputs that weren't part of the story.

   b. **Write the commit message.** Format:

      ```
      <type>(<scope>): <subject under 70 chars>

      <body: 1-3 short paragraphs explaining the WHY, not the what. Reference the PRD slug.>

      Refs: .agents/PRDs/<prd-slug>.md
      Story: .agents/stories/<prd-slug>/<NN>-<task>.md

      Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
      ```

      Conventional types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `style`.
      Scope is the Rails layer or component (e.g. `feat(users): add email verification`, `fix(admin): correct CSV export charset`).

   c. **Create the commit** using a HEREDOC for the message (preserves multi-line formatting). Confirm with `git status` afterwards.

   d. **Do not push.** Pushing is a separate, user-initiated step.

6. **Report.** Output the commit hash, the staged file list, and any warnings that were below failure threshold (skipped tests, brakeman ignores, etc.). Suggest the next story (`/implement <next-path>`) or `/plan_feature` if the PRD is complete.

## Output expectations — exact tokens to watch for

| Tool | "Pass" tokens | "Fail" tokens |
|---|---|---|
| RuboCop | `no offenses detected` | `Offenses:`, lines beginning `<file>:<line>:<col>: C\|W\|E:` |
| Minitest | `0 failures, 0 errors` | any `F` or `E` in the progress bar, or `N failures` / `N errors` in summary where N > 0 |
| Brakeman | `No warnings found` | `== Warnings ==`, lines like `Confidence: High` |
| bundler-audit | `No vulnerabilities found` | `Name:`, `Version:`, `CVE:`, `Criticality:` |

Do not commit based on a partial run — every tool must complete and report cleanly.

## Constraints

- **Never** skip hooks (`--no-verify`) or bypass signing unless the user explicitly asks.
- **Never** force-push from this command.
- **Never** amend an existing commit; create a new one if changes are needed after a failed hook.
- **Never** use the word "stakeholders" in the commit message.
- One commit per `/commit`. If the working tree contains two unrelated changes, ask the user to split.
- If a pre-commit hook fails, the commit did **not** happen — fix the issue, re-stage, and create a **new** commit (do not `--amend`).
