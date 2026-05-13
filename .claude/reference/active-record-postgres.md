# Reference: ActiveRecord + PostgreSQL Conventions

Load this when a story touches migrations, models, validations, queries, or database performance.

---

## Migrations

### Generate, don't hand-write

```bash
bin/rails generate migration AddPublishedAtToPosts published_at:datetime
bin/rails generate migration CreatePosts title:string body:text user:references
bin/rails generate model Comment post:references body:text     # model + migration
```

The generator names the migration class and timestamps the file correctly. Then hand-edit the file for indexes, defaults, and constraints.

### Safe migration template

```ruby
class CreatePosts < ActiveRecord::Migration[8.0]
  def change
    create_table :posts do |t|
      t.references :user, null: false, foreign_key: true
      t.string :title, null: false
      t.text :body
      t.string :slug, null: false
      t.datetime :published_at
      t.timestamps
    end

    add_index :posts, :slug, unique: true
    add_index :posts, :published_at        # for ORDER BY / WHERE filters
  end
end
```

### Adding a NOT NULL column to an existing table

**Wrong** — locks the table while every row is rewritten:

```ruby
add_column :users, :role, :string, null: false, default: "member"
```

**Right** — three migrations:

```ruby
# Migration 1: add as nullable, with a default for new rows
class AddRoleToUsers < ActiveRecord::Migration[8.0]
  def change
    add_column :users, :role, :string, default: "member"
  end
end

# Migration 2: backfill (separate migration so it can be retried)
class BackfillUserRole < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!
  def up
    User.in_batches.update_all(role: "member")
  end
end

# Migration 3: enforce
class MakeUserRoleNotNull < ActiveRecord::Migration[8.0]
  def change
    change_column_null :users, :role, false
  end
end
```

### Adding indexes to large tables

```ruby
class AddIndexConcurrently < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!
  def change
    add_index :posts, :user_id, algorithm: :concurrently
  end
end
```

`algorithm: :concurrently` lets PG add the index without an exclusive lock. Requires `disable_ddl_transaction!`.

### Rolling back

```bash
bin/rails db:rollback                  # one step
bin/rails db:rollback STEP=3           # three steps
bin/rails db:migrate:status            # see what's applied
```

Every migration should be reversible. Use `reversible` blocks for non-DDL operations:

```ruby
def change
  reversible do |dir|
    dir.up   { execute "UPDATE posts SET status = 'draft' WHERE status IS NULL" }
    dir.down { } # no-op
  end
end
```

---

## Models — anatomy

```ruby
class Post < ApplicationRecord
  # 1. Associations
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_one_attached :cover_image

  # 2. Enums (PostgreSQL: string-backed is safest)
  enum :status, { draft: "draft", published: "published", archived: "archived" }

  # 3. Validations
  validates :title, presence: true, length: { maximum: 255 }
  validates :slug, presence: true, uniqueness: true, format: { with: /\A[a-z0-9-]+\z/ }
  validate :published_at_in_past, if: :published?

  # 4. Callbacks (use sparingly — prefer service objects)
  before_validation :generate_slug, on: :create

  # 5. Scopes
  scope :recent, -> { order(created_at: :desc) }
  scope :by_user, ->(user) { where(user: user) }

  # 6. Instance methods
  def summary
    body.to_s.truncate(140)
  end

  private

  def generate_slug
    self.slug ||= title.to_s.parameterize
  end

  def published_at_in_past
    return if published_at.nil? || published_at <= Time.current
    errors.add(:published_at, "must not be in the future")
  end
end
```

Order sections consistently in every model — easier to read.

### Associations — common gotchas

```ruby
belongs_to :user                              # required by default (Rails 5+)
belongs_to :user, optional: true              # allow nil
has_many :comments, dependent: :destroy       # delete children when parent is destroyed
has_many :comments, dependent: :delete_all    # bypass callbacks — faster, no callbacks fire
has_many :through                             # join through an intermediate model
has_one_attached :avatar                      # Active Storage
```

When an association is ambiguous, declare `inverse_of`:

```ruby
class Post < ApplicationRecord
  has_many :comments, inverse_of: :post
end

class Comment < ApplicationRecord
  belongs_to :post, inverse_of: :comments
end
```

### Validations — match with DB constraints

A validation in Ruby is not enough. Mirror it in the migration:

| Validation | Migration |
|---|---|
| `validates :name, presence: true` | `t.string :name, null: false` |
| `validates :email, uniqueness: true` | `add_index :users, :email, unique: true` |
| `validates :role, inclusion: { in: ROLES }` | CHECK constraint or `enum` |
| `belongs_to :user` (default required) | `t.references :user, null: false, foreign_key: true` |

---

## Queries — patterns to use

### Basic finders

```ruby
Post.find(1)                          # raises RecordNotFound if missing
Post.find_by(slug: "hello")           # returns nil if missing
Post.find_by!(slug: "hello")          # raises if missing
Post.where(user: current_user)        # AR composes the right WHERE
Post.where("created_at > ?", 1.week.ago)
Post.where(status: %w[published archived])
Post.order(created_at: :desc).limit(10)
```

### Avoiding N+1

```ruby
# WRONG — N+1: one query per post for its user
Post.recent.each { |p| puts p.user.name }

# RIGHT — eager-loaded
Post.recent.includes(:user).each { |p| puts p.user.name }
```

`includes` joins or pre-fetches as needed. The `bullet` gem (if installed) flags N+1 in development.

### Batching

```ruby
User.where(active: true).find_each(batch_size: 1000) do |user|
  user.recalculate_score!
end
```

`find_each` paginates — never iterate `.all` on large tables.

### Aggregations and grouping

```ruby
Post.published.group(:user_id).count                # { user_id => count }
Post.where(user: u).sum(:word_count)
Post.average(:rating)
```

### Raw SQL — only when AR can't express it

```ruby
Post.find_by_sql([
  "SELECT * FROM posts WHERE EXTRACT(YEAR FROM created_at) = ?", 2024
])

ActiveRecord::Base.connection.execute("VACUUM ANALYZE posts")
```

Comment why AR couldn't do it. Audit for SQL injection — always parameterize.

### Transactions

```ruby
ActiveRecord::Base.transaction do
  post.update!(status: "published")
  PostPublished.create!(post: post, user: current_user)
end
```

Use the bang methods (`update!`, `create!`) inside transactions so failures raise and trigger rollback.

---

## PostgreSQL features worth using

### JSONB

```ruby
# Migration
add_column :events, :payload, :jsonb, default: {}, null: false
add_index :events, :payload, using: :gin
```

```ruby
# Querying JSONB
Event.where("payload @> ?", { type: "click" }.to_json)
Event.where("payload->>'user_id' = ?", user.id.to_s)
```

Use JSONB for flexible event payloads, settings, or API responses. Don't use it as a substitute for normalized columns.

### citext (case-insensitive text)

```ruby
# Enable once
enable_extension "citext"

# Migration
add_column :users, :email, :citext, null: false
add_index :users, :email, unique: true
```

Then `User.where(email: "Alice@Example.com")` matches `alice@example.com` automatically.

### Arrays

```ruby
add_column :posts, :tags, :string, array: true, default: [], null: false
add_index :posts, :tags, using: :gin
```

```ruby
Post.where("? = ANY(tags)", "rails")
Post.where("tags @> ARRAY[?]::varchar[]", ["rails", "ruby"])
```

### Partial indexes

```ruby
add_index :users, :email, unique: true, where: "deleted_at IS NULL"
```

Useful for unique-only-while-active patterns.

### Generated columns

```ruby
add_column :posts, :search_vector, :tsvector, as: "to_tsvector('english', title || ' ' || coalesce(body, ''))", stored: true
add_index :posts, :search_vector, using: :gin
```

### Enums via CHECK constraints

```ruby
add_column :posts, :status, :string, null: false, default: "draft"
add_check_constraint :posts, "status IN ('draft', 'published', 'archived')", name: "posts_status_check"
```

Backs the model's `enum :status` declaration at the DB level.

---

## Configuration

`config/database.yml` for this template:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: agentic_rails_development

test:
  <<: *default
  database: agentic_rails_test

production:
  <<: *default
  url: <%= ENV["DATABASE_URL"] %>
```

Never commit credentials. Use `RAILS_MASTER_KEY` (or Kamal secrets) for production.

---

## Anti-patterns to avoid

- **`Model.all.each`** on tables of any size. Use `find_each`.
- **`Model.where(...).count` inside loops** — pre-aggregate with `group`.
- **Migrations that combine schema + data changes** — split them; data migrations should be retriable.
- **`null: true` everywhere** — declare not-null on the migration, not just the model.
- **Hand-editing `db/schema.rb`** — generate a migration.
- **Using SQLite for tests** — this project is Postgres-only. Test against the real DB.
- **Putting business logic in callbacks** — `after_create` chains become impossible to debug. Use a service object.
- **`rescue => e` swallowing errors** — let exceptions propagate; let the request layer handle them.
