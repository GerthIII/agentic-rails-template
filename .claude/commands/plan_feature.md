---
description: Decompose a PRD into ordered, implementable stories in .agents/stories/{prd-slug}/
argument-hint: <prd-slug>
---

# /plan_feature — Decompose a PRD into stories

Your job is to take an existing PRD and produce a sequence of small, ordered, independently-validatable stories. Each story should be implementable in one `/implement` invocation and validatable by `/commit`.

Target PRD slug: **$ARGUMENTS**

## Steps

1. **Read the PRD** at `.agents/PRDs/$ARGUMENTS.md`. If it does not exist, list available PRDs and ask the user which one. If the PRD's "Open questions" section is non-empty, surface those questions to the user via AskUserQuestion and incorporate the answers before continuing.

2. **Read the global rules** at `CLAUDE.md` and skim the relevant on-demand reference docs in `.claude/reference/` based on what the PRD touches:
   - Touches DB / models → `active-record-postgres.md`
   - Touches routes / controllers → `rails-architecture.md`
   - Touches UI / Stimulus → `stimulus-bootstrap.md`
   - Touches Turbo Frames / Streams → `hotwire-turbo.md`

3. **Read the existing codebase** to ground the plan. Grep for relevant models, controllers, routes. The plan must reference real file paths.

4. **Decompose into 2-6 stories.** Each story is a single coherent change with its own validation. Order them so the project compiles and tests pass after each one — never plan a story that leaves the suite red as a "stepping stone".

   Typical Rails ordering:
   1. Migration (schema only, with backfill if needed)
   2. Model (validations, associations, scopes) + model tests
   3. Controller + routes + integration tests
   4. Views + Stimulus + system tests
   5. Background jobs / mailers (if any)
   6. Cleanup / docs

   Merge steps that are trivially small. Split steps that touch >5 files.

5. **Write each story** to `.agents/stories/{prd-slug}/{NN}-{kebab-task-name}.md` where `NN` is a zero-padded 2-digit index starting at `01`. Use the story template below.

6. **Write a plan index** at `.agents/stories/{prd-slug}/README.md` listing the stories in order with one-line summaries.

7. **Summarize** the plan (story count, what each one does in 5-10 words) and suggest `/implement .agents/stories/{prd-slug}/01-*.md` as the next step.

## Story template

```markdown
# Story: <Title>

**PRD:** [.agents/PRDs/<prd-slug>.md](../../PRDs/<prd-slug>.md)
**Index:** NN of TT
**Status:** open
**Estimated scope:** <small | medium> (medium = >5 files or new migration)

## Goal

<One sentence. What changes from the user's perspective, or what capability is added.>

## Context

<What the implementer needs to know that isn't obvious from the PRD. Reference real files.>

- Builds on: <file:line or "none">
- Touches: <file paths>

## Inputs

<What needs to exist before this story runs. List prior stories or "none".>

## Steps

<Numbered, concrete actions. Each should map to one or two tool calls during /implement.>

1. <action>
2. <action>

## Expected files

<Bulleted list of files that will be created or modified.>

- `app/models/foo.rb` — new
- `db/migrate/YYYYMMDDHHMMSS_create_foos.rb` — new
- `test/models/foo_test.rb` — new

## Test plan

<Specific tests to add. Use Minitest naming. Include both happy path and edge cases.>

- `test/models/foo_test.rb`
  - `test "validates presence of name"`
  - `test "scope :active returns only enabled foos"`
- `test/controllers/foos_controller_test.rb`
  - `test "GET /foos returns success"`

## Validation gate

<Which checks must pass for this story. Default: all four. List exceptions only.>

- [ ] `bundle exec rubocop` clean
- [ ] `bin/rails test` green
- [ ] `bundle exec brakeman --no-pager -q` clean
- [ ] `bundle exec bundler-audit check` clean
- [ ] (if UI) `bin/rails test:system` green

## Done when

<Checklist the implementer ticks off. Each item must be observable.>

- [ ] Migration runs and rolls back cleanly
- [ ] Model tests pass
- [ ] <observable behavior from PRD success criteria>
```

## Constraints

- **Never** use the word "stakeholders" in the plan or stories. Use "users" or "project requirements".
- Do not write code in this command — only plans.
- Every story must be commit-shaped: ends with a green test suite, lints clean, is one logical change.
- If the PRD is too vague to plan, push back via AskUserQuestion before producing a thin plan.
- If a story would touch more than ~10 files, split it.
