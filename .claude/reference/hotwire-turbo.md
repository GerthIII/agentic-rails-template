# Reference: Hotwire — Turbo Drive, Frames, Streams

Load this when a story involves page navigation behavior, in-place updates, real-time UI, or you're tempted to reach for a SPA framework.

Hotwire = **Turbo** (navigation + partial updates over HTML) + **Stimulus** (sprinkles of JS behavior). This doc covers Turbo. See `stimulus-bootstrap.md` for Stimulus.

The mental model:
- **Turbo Drive** intercepts full-page navigation and makes it feel SPA-fast, without a client-side router.
- **Turbo Frames** scope a part of the page to its own navigation — clicks and form submits inside a frame stay inside that frame.
- **Turbo Streams** let the server push HTML fragments that update specific parts of the page, in response to a request or over a WebSocket.

Always try in this order: **Drive → Frames → Streams → Stimulus → (last resort) full custom JS.**

---

## Turbo Drive — full-page navigation, accelerated

It's on by default. Every `<a>` and `<form>` is intercepted. The server still returns full HTML pages — Turbo just replaces the `<body>` without a full reload.

### Opting out

```erb
<%= link_to "Native", external_path, data: { turbo: false } %>
<%= form_with model: @post, data: { turbo: false } %>

<a href="/admin" data-turbo="false">Admin</a>
```

Use `false` for: file downloads, third-party redirects, anything that mustn't be intercepted.

### Tracking assets across reloads

```erb
<%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
```

If `application.css` changes between visits, Turbo forces a full reload so the new CSS loads.

### Preserving elements across visits

```erb
<div id="player" data-turbo-permanent>...</div>
```

The element with the same ID is kept across page transitions. Use for media players, chat widgets, etc.

### Progress bar

Built-in. Customize in CSS:

```css
.turbo-progress-bar { background-color: #0d6efd; }
```

### Form responses

Forms submitted via Turbo expect:
- **2xx** + HTML → replace body
- **422** + HTML → render the error state (this is why controllers use `status: :unprocessable_entity` on validation errors)
- **3xx** redirect → follow

```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post                                 # 303 See Other (Turbo handles it)
  else
    render :new, status: :unprocessable_entity        # 422 — Turbo replaces the form
  end
end
```

---

## Turbo Frames — scoped navigation

A `<turbo-frame>` is an island. Clicks and form submits inside it only update its contents.

### Basic frame

```erb
<%# app/views/posts/show.html.erb %>
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
  <%= link_to "Edit", edit_post_path(@post) %>
<% end %>
```

When the user clicks "Edit", Turbo fetches `edit_post_path` and looks for a frame with the same id (`post_<id>`) in the response — and swaps just that frame.

```erb
<%# app/views/posts/edit.html.erb %>
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <%= render "form", post: @post %>
<% end %>
```

The URL bar does **not** change. The rest of the page stays put.

### Lazy-loaded frames

```erb
<%= turbo_frame_tag "comments", src: post_comments_path(@post), loading: "lazy" do %>
  <p>Loading comments…</p>
<% end %>
```

Frame fetches its contents from `src` when it scrolls into view. Useful for expensive sections.

### Targeting from outside the frame

```erb
<%= link_to "Edit", edit_post_path(@post), data: { turbo_frame: "post_#{@post.id}" } %>
```

This link is outside the frame but tells Turbo: "replace that frame's contents with what comes back from this URL."

### Breaking out of a frame

```erb
<%= link_to "View full", post_path(@post), data: { turbo_frame: "_top" } %>
```

`_top` does a full-page navigation instead of frame-scoped.

### When to use a frame

- An inline edit on a card without leaving the page.
- A search-and-filter section that refreshes independently.
- A modal whose content is fetched on demand.
- A list with pagination that updates in place.

### When NOT to use a frame

- The entire page should update — that's just Turbo Drive (no frame needed).
- Multiple disconnected parts should update from one action — use Turbo Streams.
- The update should reach other connected clients — use Turbo Streams over ActionCable.

---

## Turbo Streams — server-pushed HTML fragments

A Turbo Stream is a `<turbo-stream>` element that names an action (append, prepend, replace, update, remove, before, after) and a target id, with HTML content.

### Stream actions

| Action | Effect |
|---|---|
| `append` | Add content as last child of target |
| `prepend` | Add content as first child |
| `replace` | Replace target element entirely |
| `update` | Replace target's inner HTML |
| `remove` | Delete target element |
| `before` | Insert before target |
| `after` | Insert after target |

### Responding to a form submit with a stream

```ruby
def create
  @comment = @post.comments.create(comment_params)
  respond_to do |format|
    format.turbo_stream
    format.html { redirect_to @post }
  end
end
```

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments", partial: "comments/comment", locals: { comment: @comment } %>
<%= turbo_stream.update "comments_count", @post.comments.count %>
<%= turbo_stream.replace "comment_form", partial: "comments/form", locals: { comment: Comment.new } %>
```

The form submit returns multiple updates in one response. Turbo applies them all.

### Streams from `button_to` / destroys

```erb
<%= button_to "Delete", comment_path(comment), method: :delete, class: "btn btn-link" %>
```

```ruby
def destroy
  @comment.destroy
  respond_to do |format|
    format.turbo_stream { render turbo_stream: turbo_stream.remove(@comment) }
    format.html         { redirect_to @comment.post }
  end
end
```

`turbo_stream.remove(@comment)` uses the model's `dom_id` (e.g. `comment_42`) automatically.

### Broadcasting from the model (real-time)

```ruby
class Comment < ApplicationRecord
  belongs_to :post
  broadcasts_to ->(comment) { [comment.post, "comments"] }, inserts_by: :append
end
```

```erb
<%# app/views/posts/show.html.erb %>
<%= turbo_stream_from @post, "comments" %>
<div id="comments">
  <%= render @post.comments %>
</div>
```

Now every connected viewer of this post sees new comments append in real time via ActionCable. No JS code in the app.

### Choosing inserts_by

- `:append` — new items at the bottom (chat-like).
- `:prepend` — new items at the top (feed-like).

For more control:

```ruby
after_create_commit -> { broadcast_prepend_to [post, "comments"], partial: "comments/comment" }
after_update_commit -> { broadcast_replace_to [post, "comments"] }
after_destroy_commit -> { broadcast_remove_to [post, "comments"] }
```

---

## DOM IDs — `dom_id` helper

```ruby
dom_id(@post)              # "post_42"
dom_id(@post, :edit)       # "edit_post_42"
dom_id(Post.new)           # "new_post"
```

```erb
<div id="<%= dom_id(@post) %>" class="card">...</div>
<%= turbo_frame_tag @post do %>...<% end %>
<%= turbo_stream.replace @post, partial: "posts/post", locals: { post: @post } %>
```

When `turbo_stream.replace @post` takes a model, it uses `dom_id(@post)` automatically. Match the wrapper id in your partial.

---

## Putting it together — example flow

**User adds a comment without leaving the page:**

1. `_form.html.erb` posts via Turbo to `POST /posts/:id/comments`.
2. Controller creates the comment, renders `create.turbo_stream.erb`.
3. The template returns three streams: append to `#comments`, update `#comments_count`, reset the form.
4. Turbo applies all three. Page never reloaded; URL didn't change.

**Real-time for other viewers:**

1. Model declares `broadcasts_to ...`.
2. Layout has `<%= turbo_stream_from @post, "comments" %>`.
3. ActionCable pushes the same `<turbo-stream>` payload to all connected viewers.

No custom JS for either path.

---

## When you genuinely need Stimulus or custom JS

- Client-only state that doesn't need the server (e.g. show/hide a panel, count characters as the user types).
- Bridging a third-party JS library to Bootstrap markup.
- Form behaviors before submit (e.g. validate format, enable submit only when valid).

Anything that requires server data → Turbo Frame or Stream first.

---

## Anti-patterns to avoid

- **Custom AJAX with `fetch()` + `innerHTML`** when a Turbo Stream would do.
- **Returning JSON from controllers for internal UI updates** — return Turbo Streams.
- **Heavy Stimulus controllers that re-render lists** — let Turbo Streams do the rendering on the server.
- **Disabling Turbo globally** because one form broke — fix the form (`status: :unprocessable_entity`, missing `dom_id`, etc.).
- **Mixing frame and stream responses for the same form** — pick one. Frames replace one region; streams update many.
- **Forgetting `dom_id` consistency** — if `broadcast_replace_to` sends `post_42` and your partial wraps in `<div id="post-42">`, nothing happens. Use `dom_id(@post)` everywhere.
- **Polling intervals to fake real-time** — use `turbo_stream_from` over ActionCable.
