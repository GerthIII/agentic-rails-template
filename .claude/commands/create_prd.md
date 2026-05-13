---
description: Capture feature intent as a structured PRD in .agents/PRDs/
argument-hint: [short feature description]
---

# /create_prd — Capture feature intent

Your job is to interview the user enough to produce a focused, decision-ready Product Requirement Document and write it to `.agents/PRDs/{slug}.md`.

User's initial feature description: **$ARGUMENTS**

## Terminology — strict

- **Never** use the word "stakeholders" anywhere in the PRD. Use "users", "the team", or "project requirements".
- Write for an engineer who will implement this without further conversation.
- Plain language. No marketing voice. No emojis unless the user used them first.

## Steps

1. **Read the global rules** at `CLAUDE.md` so the PRD's technical notes align with the actual stack. If the feature description is missing entirely, ask the user for it before continuing.

2. **Ask clarifying questions, up to 4 at once, only the ones that block a good PRD.** Use the AskUserQuestion tool. Skip anything you can reasonably infer. Examples of questions worth asking:
   - Who are the users? (end-users, admins, internal team, external API consumer)
   - What is the success criterion? (a behavior the user will see, not a metric)
   - What is explicitly out of scope?
   - Are there existing models, controllers, or routes this builds on?
   - Any constraints on the data layer (new table, new column, new association)?
   - Any UI requirements beyond a standard Bootstrap form/table?

   Do **not** ask about: framework choice, testing framework, linter, deployment — those are fixed by `CLAUDE.md`.

3. **Pick a slug.** Kebab-case, 2-5 words, no date prefix. Examples: `user-email-verification`, `admin-export-csv`. Confirm with the user only if ambiguous.

4. **Write the PRD** to `.agents/PRDs/{slug}.md` using the template below. Fill every section — if a section truly has nothing, write `None.` rather than deleting it.

5. **Briefly summarize** what you wrote (3-5 bullets) and suggest `/plan_feature {slug}` as the next step.

## PRD template

```markdown
# PRD: <Title Case Feature Name>

**Slug:** <slug>
**Created:** <YYYY-MM-DD>
**Status:** draft

## Problem

<2-4 sentences. What is the user-visible problem this solves? Why now?>

## Users

<Who are the affected users? Be concrete: "signed-in customers", "admin operators", "the developer running rake tasks". Avoid "stakeholders".>

## Success criteria

<Bulleted list of behaviors a user will observe when this is done. Each bullet should be testable.>

- <observable behavior>
- <observable behavior>

## Scope

### In scope
- <feature element>
- <feature element>

### Out of scope (explicitly)
- <thing we are NOT doing — and why, in one phrase>

## Technical notes

<What the implementer needs to know that isn't obvious from the codebase. Stack is fixed by CLAUDE.md, so don't repeat it.>

- **Models / migrations:** <new table? new column? new association? or "none">
- **Routes / controllers:** <new resource? new action on existing controller? or "none">
- **Views / Stimulus:** <new view? new Stimulus controller? Turbo Frame / Stream? or "none">
- **Background work:** <ActiveJob? scheduled? or "none">
- **External services:** <API calls? webhooks? or "none">

## Open questions

<Anything the user could not answer yet. The plan_feature step will surface these again.>

- <question>

## Out of band

<Cross-cutting concerns: feature flag? data migration? observability? security review?>

- <item or "none">
```

## Constraints

- Write **one** PRD per invocation. If the user describes multiple unrelated features, propose splitting and confirm before writing.
- Do not begin implementation. Do not generate migrations, models, or stories — that is `/plan_feature`'s job.
- If `.agents/PRDs/{slug}.md` already exists, ask before overwriting.
