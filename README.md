# Agentic Rails Template

A Rails 8 boilerplate wired for agent-assisted development with the **Plan-Implement-Validate (PIV)** workflow.

It's an opinionated starting point: Rails + PostgreSQL + Bootstrap + Stimulus + Hotwire, with a set of Claude Code slash commands and reference docs that drive features from a one-line intent to a green, committed change.

---

## Quick start

```bash
# 1. Create your project from this template
#    GitHub: click "Use this template" → "Create a new repository"
#    or:    gh repo create my-app --template GerthIII/agentic-rails-template --clone

cd my-app

# 2. Install dependencies
bundle install
yarn install

# 3. Set up the database
bin/rails db:prepare

# 4. Run the dev server (Rails + asset watchers)
bin/dev
```

Open <http://localhost:3000> — you should see the Rails welcome page.

---

## The PIV workflow

The agent drives work through five slash commands. Run them in order:

| Command | Phase | Output |
|---|---|---|
| `/prime` | Setup | Briefs the agent on the repo, in-flight PRDs, and git state |
| `/create_prd <feature>` | Plan (intent) | A focused PRD in `.agents/PRDs/{slug}.md` |
| `/plan_feature <slug>` | Plan (decomposition) | Ordered stories in `.agents/stories/{slug}/NN-*.md` |
| `/implement <story-path>` | Implement | Code, tests, and self-validation for one story |
| `/commit` | Validate | Full validation gate, then one conventional commit |

The validation gate (`/commit`) runs:

```bash
bundle exec rubocop
bin/rails test
bundle exec brakeman --no-pager -q
bundle exec bundler-audit check
```

Nothing commits unless every check is clean.

---

## Tech stack

Strict by design — these are the only tools the agent will reach for.

| Layer | Choice |
|---|---|
| Language | Ruby (see [`.ruby-version`](.ruby-version)) |
| Framework | Ruby on Rails 8 |
| Database | PostgreSQL |
| ORM | ActiveRecord |
| HTML | ERB templates, HTML5 |
| CSS | Bootstrap 5, CSS3 |
| JS | Stimulus.js (Hotwire) |
| Navigation | Turbo Drive, Frames, Streams |
| Testing | Minitest + Capybara + Selenium |
| Linting | RuboCop (`rubocop-rails-omakase`) |
| Security | Brakeman, bundler-audit |
| Deployment | Kamal |

**Not in this stack:** TypeScript, React/Vue/Svelte, Tailwind, jQuery, Drizzle/Prisma, SQLite, Vitest/Jest, Bun. The agent will push back if asked to introduce them.

---

## Requirements

- Ruby (version in [`.ruby-version`](.ruby-version))
- Node.js (version in [`.node-version`](.node-version)) + Yarn
- PostgreSQL 14+
- [Claude Code](https://docs.claude.com/en/docs/agents-and-tools/claude-code/overview) to use the agentic workflow (the template still works as a plain Rails app without it)

---

## Repository layout

```
.
├── app/                       # Standard Rails MVC
├── config/                    # Routes, database, environments
├── db/                        # Migrations + schema
├── test/                      # Minitest
├── .agents/
│   ├── PRDs/                  # Feature intent documents
│   └── stories/               # Decomposed implementation tasks
├── .claude/
│   ├── commands/              # PIV slash commands
│   └── reference/             # On-demand reference docs
├── .kamal/                    # Deploy hooks + secrets template
├── CLAUDE.md                  # Global rules for the agent
└── README.md                  # This file
```

The agentic configuration lives in two places:

- **[CLAUDE.md](CLAUDE.md)** — the global rules. Tech stack, validation gate, terminology constraints, and workflow definition.
- **[.claude/reference/](.claude/reference/)** — deep technical references loaded only when a story needs them (kept out of the default token budget):
  - [`rails-architecture.md`](.claude/reference/rails-architecture.md) — MVC, routing, request lifecycle
  - [`stimulus-bootstrap.md`](.claude/reference/stimulus-bootstrap.md) — Stimulus controllers + Bootstrap 5 components
  - [`active-record-postgres.md`](.claude/reference/active-record-postgres.md) — Migrations, queries, PostgreSQL features
  - [`hotwire-turbo.md`](.claude/reference/hotwire-turbo.md) — Drive, Frames, Streams

---

## Customizing the template

1. **Project name** — update `config/application.rb`, `config/database.yml`, and `config/cable.yml` (search for the placeholder module name).
2. **Master key** — `config/master.key` is gitignored. Run `bin/rails credentials:edit` to generate a new one in your clone.
3. **`CLAUDE.md`** — replace the generic "Project Overview" section with what your app actually does. The rest of the file is stack-fixed and shouldn't need changes.
4. **Kamal deploy** — fill in `config/deploy.yml` and `.kamal/secrets` once you have hosting set up.

---

## Working without Claude Code

The agentic tooling is additive — none of it is required to run the Rails app. If you're just using this as a regular Rails 8 + Bootstrap + Stimulus starting point, ignore `.claude/`, `.agents/`, and `CLAUDE.md`. The app boots and tests with standard Rails commands.

---

## Conventions worth knowing

- **No "stakeholders".** Generated PRDs, plans, and commits use "users" or "project requirements". The constraint is encoded in [CLAUDE.md](CLAUDE.md) and the command prompts.
- **One commit per story.** `/commit` rejects partial validation runs and refuses to amend.
- **Migrations are split.** Adding a NOT NULL column is three migrations (nullable + default → backfill → enforce). See [`active-record-postgres.md`](.claude/reference/active-record-postgres.md).
- **Turbo before custom JS.** Server-rendered HTML with Turbo Frames and Streams is the default; Stimulus is for client-only behavior.

---

## License

MIT — see [LICENSE](LICENSE).
