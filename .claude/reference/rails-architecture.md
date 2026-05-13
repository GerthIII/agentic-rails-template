# Reference: Rails MVC + Routing Architecture

Load this when a story touches controllers, routes, request handling, or the overall app structure.

---

## Request lifecycle (one HTTP request)

```
Rack server (Puma)
  → Rails routing (config/routes.rb)
    → Middleware stack (sessions, CSRF, params, etc.)
      → Controller#action
        → Strong params
        → Model / service call
        → Render (view) or redirect_to
          → Response (HTML / Turbo Stream / JSON)
```

The agent's job in a typical story: add or modify the route, controller action, and view. Models and migrations are usually a separate, earlier story.

---

## Directory layout — what goes where

| Path | Contents | Don't put here |
|---|---|---|
| `app/controllers/` | Thin orchestration: parse params, call models, render | Business logic, queries beyond `Model.find(params[:id])` |
| `app/models/` | ActiveRecord models, validations, scopes, associations, instance methods | View formatting, request-aware code |
| `app/views/<resource>/` | ERB templates per action; partials as `_name.html.erb` | Ruby logic beyond simple iteration / conditionals |
| `app/helpers/` | View helper methods (format dates, render badges) | Business logic |
| `app/javascript/controllers/` | Stimulus controllers (`*_controller.js`) | Anything beyond DOM-level behavior |
| `app/assets/stylesheets/` | SCSS, Bootstrap overrides | Per-component CSS-in-JS |
| `app/jobs/` | ActiveJob classes (background work) | Sync code that belongs in a model |
| `app/mailers/` | ActionMailer classes | HTML composition (use mailer views) |
| `app/channels/` | ActionCable channels (only if Turbo Streams aren't enough) | Default chat/notification use cases (Turbo Streams handle these) |
| `lib/` | Code not tied to ActiveRecord or controllers (parsers, integrations) | Anything that should be in `app/` |
| `config/initializers/` | Boot-time config | Per-request behavior |

---

## Routing patterns

### Resourceful (preferred)

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"

  resources :posts do
    resources :comments, only: %i[create destroy]
    member do
      post :publish
    end
  end

  namespace :admin do
    resources :users
  end
end
```

This produces conventional URL/controller mappings: `GET /posts → posts#index`, `POST /posts/:id/publish → posts#publish`, `POST /admin/users → admin/users#create`, etc.

### When NOT to use `resources`

- Single-action endpoints: prefer `get "/healthz", to: "health#show"`.
- Webhook receivers: `post "/webhooks/stripe", to: "webhooks#stripe"`.

Avoid `match` and wildcard `get "*path"` routes outside of error handling.

### Inspecting routes

```bash
bin/rails routes                       # all routes
bin/rails routes -g posts              # grep
bin/rails routes -c PostsController    # by controller
```

---

## Controller patterns

### Canonical RESTful controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_post, only: %i[show edit update destroy]

  def index
    @posts = Post.published.order(created_at: :desc)
  end

  def show; end

  def new
    @post = Post.new
  end

  def create
    @post = current_user.posts.new(post_params)
    if @post.save
      redirect_to @post, notice: "Post created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit; end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: "Post updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: "Post deleted."
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body)
  end
end
```

Notes:
- `before_action` for shared setup, not for business logic.
- `status: :unprocessable_entity` on form-error renders so Turbo correctly handles the response.
- Strong params is mandatory. Never `params[:post]` directly into `update`.
- Use `current_user.posts.new(...)` (scoping through association) instead of `Post.new(user_id: current_user.id, ...)` — avoids mass-assignment of `user_id`.

### Multi-format responses

```ruby
def show
  @post = Post.find(params[:id])
  respond_to do |format|
    format.html
    format.json { render json: @post }
  end
end
```

Use `respond_to` only when you actually serve multiple formats. Don't add it speculatively.

### Turbo Stream responses

See `.claude/reference/hotwire-turbo.md` — the short version: render `*.turbo_stream.erb` templates from create/update/destroy for in-place updates.

---

## View patterns

### Partials

```erb
<%# app/views/posts/index.html.erb %>
<div class="container">
  <h1>Posts</h1>
  <%= render @posts %>          <%# renders _post.html.erb once per post %>
</div>

<%# app/views/posts/_post.html.erb %>
<article class="card mb-3">
  <div class="card-body">
    <h2 class="card-title"><%= post.title %></h2>
    <p class="card-text"><%= post.body %></p>
  </div>
</article>
```

`render @posts` uses Rails' implicit partial convention. For collections, use this instead of an explicit `each`.

### Forms

```erb
<%= form_with model: @post, class: "needs-validation" do |f| %>
  <% if @post.errors.any? %>
    <div class="alert alert-danger">
      <%= @post.errors.full_messages.to_sentence %>
    </div>
  <% end %>

  <div class="mb-3">
    <%= f.label :title, class: "form-label" %>
    <%= f.text_field :title, class: "form-control", required: true %>
  </div>

  <div class="mb-3">
    <%= f.label :body, class: "form-label" %>
    <%= f.text_area :body, class: "form-control", rows: 6 %>
  </div>

  <%= f.submit class: "btn btn-primary" %>
<% end %>
```

`form_with` emits a Turbo-aware form by default — submissions go through Turbo Drive unless you opt out with `data: { turbo: false }`.

---

## Layouts

`app/views/layouts/application.html.erb` is the default wrapper. Per-controller layouts (`layout "admin"`) for sections with distinct chrome.

```erb
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "App" %></title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>      <%# or javascript_include_tag if using jsbundling-rails %>
  </head>
  <body>
    <%= render "shared/navbar" %>
    <main class="container py-4">
      <% if notice %><div class="alert alert-success"><%= notice %></div><% end %>
      <% if alert %><div class="alert alert-warning"><%= alert %></div><% end %>
      <%= yield %>
    </main>
  </body>
</html>
```

---

## Service objects (when controllers get heavy)

If a controller action exceeds ~10 lines of logic or coordinates more than one model, extract a service.

```ruby
# app/services/posts/publish.rb
module Posts
  class Publish
    def initialize(post:, user:)
      @post = post
      @user = user
    end

    def call
      return false unless @post.draft?

      Post.transaction do
        @post.update!(published_at: Time.current, published_by: @user)
        NotifySubscribersJob.perform_later(@post.id)
      end
      true
    end
  end
end
```

Call sites: `Posts::Publish.new(post: @post, user: current_user).call`. Keep service objects to one public method (`#call`) and inject all dependencies.

---

## Concerns (shared behavior)

Use `app/controllers/concerns/` and `app/models/concerns/` for **truly shared** behavior, not as a hiding place for long files.

```ruby
# app/models/concerns/sluggable.rb
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_validation :set_slug
    validates :slug, presence: true, uniqueness: true
  end

  private

  def set_slug
    self.slug ||= name.parameterize
  end
end
```

`include Sluggable` in the model.

---

## Error handling

Default Rails responses are usually fine. Custom handling only when needed:

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :render_404

  private

  def render_404
    render file: Rails.root.join("public/404.html"), status: :not_found, layout: false
  end
end
```

---

## Anti-patterns to avoid

- **Fat controllers**: anything beyond parse-params / call-model / render belongs in a model or service.
- **Skinny models, fat helpers**: helpers are for view formatting only.
- **N+1 queries**: use `includes(:association)` when iterating; Bullet gem flags them.
- **Custom routes for resourceful actions**: if you find yourself writing `get "/posts/new_form"`, you want `new` on a resource.
- **Bypassing strong params**: `params[:thing]` straight into `update` is a security hole.
- **Editing `db/schema.rb` by hand**: generate a migration instead.
- **Reaching for client-side rendering**: try Turbo Frames + Streams + a Stimulus controller first.
