# CLAUDE.md

This file provides global rules to Claude Code (claude.ai/code) when working in this repository. It is a template foundation for **Agentic Rails** projects built around the **Plan-Implement-Validate (PIV)** loop.

> Terminology constraint: when generating PRDs, plans, stories, or any planning document, do **not** use the word "stakeholders". Use "users", "the team", or "project requirements" instead.

---

## Project Overview

This repository is an opinionated Rails 8 boilerplate for agent-assisted development. Derived projects describe their own purpose at the top of their own `CLAUDE.md`; this template stays generic so the PIV workflow drops in cleanly.

The agent is expected to drive work through the PIV loop:

1. **Plan** — capture intent (`/create_prd`) and decompose it into ordered stories (`/plan_feature`).
2. **Implement** — execute one story at a time (`/implement`), making the smallest coherent change.
3. **Validate** — run the full validation gate (`/commit`) before any code lands.

---

## Tech Stack — Authoritative

This stack is **strict**. Do not introduce alternatives without an explicit instruction in the conversation.

| Layer | Choice | Notes |
|---|---|---|
| Language | Ruby (see `.ruby-version`) | Idiomatic Ruby, no monkey-patching of core classes |
| Framework | Ruby on Rails 8 | Convention over configuration; follow Rails idioms |
| Database | PostgreSQL | Use Postgres features (`jsonb`, `citext`, partial indexes) where they fit |
| ORM | ActiveRecord | No raw SQL except via `find_by_sql` / `connection.execute` with clear comment |
| Frontend HTML | ERB templates + HTML5 | Server-rendered; avoid client-side rendering frameworks |
| CSS | Bootstrap 5 + CSS3 | No Tailwind, no utility-first replacements |
| JS sprinkles | Stimulus.js (Hotwire) | No React, Vue, Svelte, jQuery, Alpine, or TypeScript |
| Navigation | Turbo Drive / Frames / Streams | Default to Turbo before reaching for custom JS |
| Asset pipeline | `esbuild`, `sass`, `postcss` via `yarn` scripts | See `package.json` for build commands |
| Testing | Minitest (Rails default) + Capybara + Selenium | No RSpec unless the derived project adds it explicitly |
| Linting / style | RuboCop with `rubocop-rails-omakase` | See `.rubocop.yml` |
| Security scans | Brakeman, bundler-audit | Run on validation |
| Deployment | Kamal | See `.kamal/` |

**Forbidden, unless explicitly requested:** Bun, npm/pnpm as the primary package manager (yarn is used only for JS assets), TypeScript, React/Next.js/Vue/Svelte/Solid, Tailwind/UnoCSS, Drizzle/Prisma, SQLite (test or otherwise), Vitest/Jest/Playwright, shadcn, any Node-server framework.

---

## Commands — Rails Lifecycle

Use the dedicated tool (Read, Edit, Bash) — these are the canonical commands the agent should invoke during the PIV loop.

```bash
# First-time setup or after a fresh clone
bundle install
yarn install
bin/rails db:prepare           # creates the DB if missing, then migrates + seeds

# Routine setup after pulling
bundle install
yarn install
bin/rails db:migrate

# Development
bin/dev                        # runs Procfile.dev (rails server + asset watchers)
bin/rails server               # server only
bin/rails console              # REPL

# Database
bin/rails db:migrate
bin/rails db:rollback
bin/rails db:seed
bin/rails db:prepare           # idempotent: create + migrate (preferred in CI/onboarding)
bin/rails db:reset             # destructive: drops, recreates, seeds

# Generators (prefer over hand-rolled scaffolding)
bin/rails generate migration AddFooToBars foo:string
bin/rails generate model Foo name:string
bin/rails generate controller Foos index show
bin/rails generate stimulus foo

# Testing
bin/rails test                 # all Minitest unit/integration tests
bin/rails test:system          # Capybara + Selenium system tests
bin/rails test test/models/foo_test.rb       # single file
bin/rails test test/models/foo_test.rb:42    # single test by line

# Linting / formatting
bundle exec rubocop                          # report
bundle exec rubocop -A                       # autocorrect (use deliberately)

# Security
bundle exec brakeman --no-pager -q
bundle exec bundler-audit check --update
```

Never substitute these with `npm`, `pnpm`, `bun`, or `tsc` equivalents.

---

## Architecture

Standard Rails MVC. Derived projects may add directories; do not move these.

```
app/
├── controllers/         # Thin: parse params, call models/services, render
├── models/              # ActiveRecord models, validations, scopes, callbacks (sparingly)
├── views/               # ERB templates, partials in _underscored.html.erb
├── helpers/             # View helpers only; no business logic
├── javascript/
│   └── controllers/     # Stimulus controllers, *_controller.js
├── assets/
│   └── stylesheets/     # SCSS, Bootstrap overrides
├── jobs/                # ActiveJob background work
├── mailers/             # ActionMailer
└── channels/            # ActionCable (only if pushed beyond Turbo Streams)
config/
├── routes.rb            # RESTful resources by default
├── database.yml         # PostgreSQL config
└── application.rb
db/
├── migrate/             # Timestamped migrations
├── schema.rb            # Generated; never edit by hand
└── seeds.rb
test/                    # Minitest, mirrors app/
.agents/
├── PRDs/                # One file per feature intent
└── stories/             # One subdirectory per PRD, ordered story files
.claude/
├── commands/            # PIV slash commands
└── reference/           # On-demand reference docs (loaded only when needed)
```

Request lifecycle reference: `.claude/reference/rails-architecture.md`.

---

## Code Patterns

### ActiveRecord
- Validations live on the model; database constraints back them up via migration (`null: false`, indexes, foreign keys).
- Prefer `has_many`/`belongs_to` with `inverse_of` declared when ambiguous.
- Use scopes for reusable queries; avoid embedding business logic in controllers.
- Reach for `find_each` for batches > 1k rows.
- Use strong migrations practices: add columns nullable, backfill in a separate migration, then add the not-null constraint.

### Controllers
- RESTful resources: `index`, `show`, `new`, `create`, `edit`, `update`, `destroy`. Custom actions need justification.
- Use strong parameters (`params.require(...).permit(...)`).
- Render Turbo Streams for in-place updates; redirect on full-page transitions.
- Keep controllers under ~80 lines — extract to a model method or service object beyond that.

### Views & Stimulus
- ERB partials in `app/views/**/_name.html.erb`, rendered with `render "name"` or `render partial: "name"`.
- Stimulus controllers in `app/javascript/controllers/foo_controller.js`. Use `data-controller`, `data-action`, `data-foo-target`, `data-foo-{name}-value`. See `.claude/reference/stimulus-bootstrap.md`.
- Use Bootstrap component classes directly; do not invent new CSS until a pattern repeats 3+ times.

### Routing
- RESTful first. Namespace by concern (`namespace :admin`), not by tech.
- Avoid `match` and wildcard routes unless explicitly needed.

### Naming
- Models singular CamelCase (`User`); tables plural snake_case (`users`).
- Controllers plural (`UsersController`); routes plural (`resources :users`).
- Stimulus controllers use kebab-case identifiers (`data-controller="user-form"` → `user_form_controller.js`).

### Error handling
- Let Rails handle 404/422 via `rescue_from` in `ApplicationController` only when you need a custom render.
- Use `ActiveRecord::RecordNotFound` for missing records; do not return `nil`-and-check.

---

## Testing — Minitest

- **Run all tests:** `bin/rails test`
- **Test location:** `test/`, mirroring `app/`
- **Pattern:** Minitest subclasses of `ActiveSupport::TestCase`, `ActionDispatch::IntegrationTest`, `ApplicationSystemTestCase`.
- **Fixtures:** `test/fixtures/*.yml` (Rails-loaded automatically).
- **System tests:** Capybara + Selenium, in `test/system/`.

### Reading Minitest output

Standard run produces progress markers, then a summary:

```
Run options: --seed 12345
# Running:
....F.E..

Finished in 1.23s, 8 runs, 12 assertions, 1 failure, 1 error.
```

Markers: `.` = pass, `F` = failure (assertion failed), `E` = error (exception raised), `S` = skip. A run with **any** `F` or `E` is **failing** — do not proceed to commit. After the summary, each failure/error gets a stack trace; quote the file:line of the first frame in `test/` to locate the assertion.

---

## Validation — The "V" in PIV

The validation gate **must** pass before any commit lands. The `/commit` slash command runs all of these. Manually, the equivalent is:

```bash
bundle exec rubocop                 && \
bin/rails test                      && \
bundle exec brakeman --no-pager -q  && \
bundle exec bundler-audit check
```

### What "pass" means

| Tool | Pass criterion |
|---|---|
| RuboCop | `no offenses detected` (or `X files inspected, no offenses detected`). Any `C:` / `W:` / `E:` line is a failure. |
| Minitest | Summary line shows `0 failures, 0 errors`. Skips (`S`) are acceptable but flag them. |
| Brakeman | `No warnings found`. New warnings are a failure even if old ones exist. |
| bundler-audit | `No vulnerabilities found`. Any CVE line is a failure. |

### When validation fails

- **Do not commit.** Report the failing tool, the first offending location (file:line), and propose a fix.
- If RuboCop fails on style only, consider `bundle exec rubocop -A` deliberately — but **re-run the test suite** after autocorrect.
- If a test fails, read the assertion and the code under test; do not edit the test to make it pass without confirming intent.

---

## Agentic Workflow — The PIV Loop

| Phase | Command | Output |
|---|---|---|
| Prime | `/prime` | Session briefing: loaded context, open PRDs, in-flight stories, git state |
| Plan (intent) | `/create_prd <feature>` | New file in `.agents/PRDs/{slug}.md` |
| Plan (decomposition) | `/plan_feature <prd-slug>` | Ordered story files in `.agents/stories/{prd-slug}/NN-*.md` |
| Implement | `/implement <story-path>` | Code changes, story file updated with progress |
| Validate + Commit | `/commit` | All checks pass, single conventional commit |

Run `/prime` at the start of every session. Run `/commit` after every story.

---

## Key Files

| File | Purpose |
|---|---|
| `CLAUDE.md` | This file — global rules |
| `.claude/commands/*.md` | PIV slash commands |
| `.claude/reference/*.md` | On-demand reference docs (only loaded when needed) |
| `.agents/PRDs/` | Feature intent documents |
| `.agents/stories/` | Decomposed implementation tasks |
| `Gemfile` | Ruby dependencies |
| `package.json` | JS asset build dependencies (yarn) |
| `config/routes.rb` | URL → controller mapping |
| `config/database.yml` | PostgreSQL connection |
| `.rubocop.yml` | Lint rules (uses rubocop-rails-omakase) |
| `Procfile.dev` | Dev process group (run via `bin/dev`) |
| `devise.rb`, `minimal.rb` | Rails app templates (used with `rails new -m`) |

---

## On-Demand Context

Load these only when the current task requires them — they exist to keep deep context out of the default token budget.

| Topic | File |
|---|---|
| Rails MVC, routing, request lifecycle | `.claude/reference/rails-architecture.md` |
| Stimulus controllers + Bootstrap component patterns | `.claude/reference/stimulus-bootstrap.md` |
| ActiveRecord + PostgreSQL conventions | `.claude/reference/active-record-postgres.md` |
| Hotwire / Turbo (Drive, Frames, Streams) | `.claude/reference/hotwire-turbo.md` |

---

## Notes & Constraints

- Never use the word "stakeholders" in generated PRDs, plans, or stories. Use "users" or "project requirements".
- Never introduce a new dependency without updating `Gemfile` or `package.json` and re-running `bundle install` / `yarn install`.
- Never edit `db/schema.rb` by hand — generate a migration.
- Never skip the validation gate. If a check is broken (not failing — broken), say so and ask before disabling it.
- Prefer Turbo Frames + Streams over JSON APIs for internal UI updates.
- Prefer Rails generators over hand-written scaffolding for models, controllers, migrations, and Stimulus controllers.
- When in doubt about idiom, follow `rubocop-rails-omakase` and the Rails guides.
