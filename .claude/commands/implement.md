---
description: Execute one story end-to-end — code, tests, and self-validation
argument-hint: <path to story file, e.g. .agents/stories/user-email-verification/01-add-verified-at.md>
---

# /implement — Execute one story

Your job is to take a single story file and complete the work it describes, leaving the test suite green and the lint clean. Stop after one story — do not chain into the next.

Target story: **$ARGUMENTS**

## Steps

1. **Read the story.** If `$ARGUMENTS` is missing or the file does not exist, list available stories under `.agents/stories/` and ask which one. If the story's status is already `done`, confirm with the user before re-running.

2. **Read the parent PRD** referenced at the top of the story to ground the "why".

3. **Read the on-demand references** the story or PRD implicates:
   - Migrations / models → `.claude/reference/active-record-postgres.md`
   - Routing / controllers → `.claude/reference/rails-architecture.md`
   - Views / Stimulus → `.claude/reference/stimulus-bootstrap.md`
   - Turbo Frames / Streams → `.claude/reference/hotwire-turbo.md`

   Read only the ones that apply.

4. **Update the story file's status** to `in-progress` before starting work.

5. **Execute the story's Steps**, in order, with these Rails-specific defaults:

   - **Migrations:** generate with `bin/rails generate migration ...`, hand-edit if needed, run `bin/rails db:migrate`, verify with `bin/rails db:migrate:status`. If the change is destructive, write a `down` block or use `reversible`.
   - **Models:** add validations + associations; back validations with DB constraints in the migration (`null: false`, `add_index ..., unique: true`, foreign keys).
   - **Controllers:** RESTful actions; strong params; render Turbo Streams for in-place updates, redirect for full transitions.
   - **Routes:** prefer `resources :foos` and nested resources over custom routes.
   - **Views:** ERB partials in `_underscored.html.erb`; Bootstrap classes; minimal custom CSS.
   - **Stimulus:** generate with `bin/rails generate stimulus name`; one controller per concern; values/targets/actions via `data-*` attributes.
   - **Tests:** add the Minitest cases listed under "Test plan" before the corresponding code where practical; otherwise immediately after.

6. **Run focused validation as you go.** After each meaningful change, run the narrowest applicable check:
   - After a model change: `bin/rails test test/models/foo_test.rb`
   - After a controller change: `bin/rails test test/controllers/foos_controller_test.rb`
   - After a migration: `bin/rails db:migrate` and check `db/schema.rb` diff
   - After a Stimulus controller: open the dev server (`bin/dev`) only if you can manually verify, otherwise rely on a system test

   Save the full validation gate for `/commit`.

7. **Tick off the "Done when" checklist** in the story file as each item is satisfied.

8. **Update the story's status** to `done` (only if all "Done when" items are checked) or leave as `in-progress` with a note explaining what is blocked.

9. **Summarize** in 3-6 bullets: what changed, which files, which tests were added, any deviations from the plan. Suggest `/commit` as the next step.

## Reading Minitest output during focused runs

```
Run options: --seed 12345
# Running:
....F.E

Finished in 0.42s, 7 runs, 9 assertions, 1 failure, 1 error.
```

- Any `F` (failure) or `E` (error) means stop and fix. Do not continue to the next step on a red bar.
- Read the stack trace from the bottom up; the first frame inside `test/` is your assertion, the first frame inside `app/` is your bug.

## Constraints

- **One story per invocation.** Do not chain to the next story even if it looks short.
- **Do not commit.** `/commit` runs the full validation gate. This command stops at "tests green for this story".
- **Do not modify the PRD** or other story files — only the one you are implementing.
- **Do not introduce dependencies** not in `Gemfile` / `package.json` without first proposing the addition to the user.
- **Stack discipline:** if a step seems to call for React/TypeScript/Tailwind/etc., stop and re-plan with Stimulus + Bootstrap + ERB instead.
- **Never** use the word "stakeholders" in updates to the story file or summaries.
- If the story turns out to be wrong (e.g. the migration conflicts with existing schema), update the story file with what you learned and ask the user before improvising a new plan.
