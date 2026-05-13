# Reference: Stimulus.js + Bootstrap 5 Patterns

Load this when a story adds or modifies client-side behavior, forms, modals, dropdowns, or any interactive UI.

The mental model: **Bootstrap provides the markup vocabulary; Stimulus wires behavior to that markup; Turbo handles navigation and partial updates.** Reach for custom JS only when those three together can't express what you need.

---

## Stimulus controllers — the basics

### File location and naming

```
app/javascript/controllers/
├── application.js              # Stimulus instance (boilerplate)
├── index.js                    # auto-loader (boilerplate)
├── hello_controller.js         # → data-controller="hello"
└── user_form_controller.js     # → data-controller="user-form"
```

Generate with `bin/rails generate stimulus user_form`. The generator handles the registration.

### Minimal controller

```javascript
// app/javascript/controllers/hello_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["name", "output"]
  static values = { greeting: { type: String, default: "Hello" } }
  static classes = ["active"]

  connect() {
    // Called when the controller is attached to the DOM
  }

  disconnect() {
    // Called when removed. Clean up listeners, timers, etc.
  }

  greet(event) {
    event.preventDefault()
    this.outputTarget.textContent = `${this.greetingValue}, ${this.nameTarget.value}!`
    this.element.classList.add(this.activeClass)
  }
}
```

### Wiring it in markup

```erb
<div data-controller="hello"
     data-hello-greeting-value="Hi"
     data-hello-active-class="bg-success">
  <input type="text" data-hello-target="name" class="form-control">
  <button data-action="click->hello#greet" class="btn btn-primary">Greet</button>
  <p data-hello-target="output" class="mt-2"></p>
</div>
```

### `data-*` attribute reference

| Attribute | Purpose |
|---|---|
| `data-controller="foo"` | Attach the `foo` Stimulus controller to this element |
| `data-controller="foo bar"` | Attach multiple controllers |
| `data-action="click->foo#bar"` | On click, call `bar()` on the `foo` controller |
| `data-action="submit->foo#save keydown.enter->foo#save"` | Multiple actions |
| `data-foo-target="thing"` | Mark this element as the `thingTarget` for controller `foo` |
| `data-foo-thing-value="42"` | Set the `thingValue` on the `foo` controller (typed via `static values`) |
| `data-foo-active-class="bg-success"` | Set the `activeClass` (resolved via `this.activeClass`) |
| `data-foo-active-class-active="bg-info"` | Override per-element |

### Targets

```javascript
static targets = ["item", "form"]

// In methods:
this.itemTarget       // first match, throws if missing
this.itemTargets      // all matches as Array
this.hasItemTarget    // boolean
```

### Values

```javascript
static values = {
  url: String,
  count: { type: Number, default: 0 },
  ids: Array,
  open: { type: Boolean, default: false }
}

// Read:
this.urlValue
// Write (triggers data-attribute update + optional changed callback):
this.countValue = this.countValue + 1
// Watch:
countValueChanged(newValue, oldValue) {
  // called on init and on every change
}
```

### Actions — event filtering

```html
<button data-action="click->foo#bar">Default</button>
<form data-action="submit->foo#save">           <!-- preventDefault is your job -->
<input data-action="keydown.enter->foo#submit  keydown.esc->foo#cancel">
<button data-action="click->foo#delete:prevent">Delete</button>  <!-- :prevent / :stop -->
```

### Lifecycle methods

| Method | When |
|---|---|
| `initialize()` | Once per controller instance, before connect |
| `connect()` | When the element enters the DOM (or Turbo re-renders) |
| `disconnect()` | When the element leaves the DOM |
| `<name>TargetConnected(el)` | When a target appears (e.g. Turbo Stream insert) |
| `<name>TargetDisconnected(el)` | When a target is removed |

`connect`/`disconnect` are the main entry/cleanup points. Always pair `addEventListener` with `removeEventListener` in disconnect — or use `data-action` so Stimulus manages it for you.

---

## Bootstrap 5 — component cookbook

Bootstrap 5 dropped jQuery. Components are activated via `data-bs-*` attributes (which Bootstrap's JS reads on its own) or instantiated via JS API.

### Importing Bootstrap JS

Either:
- `import "bootstrap"` in `app/javascript/application.js` for the full bundle, or
- Import specific components: `import { Modal, Tooltip } from "bootstrap"`.

Choose the second when you only need a few components — it keeps the bundle small.

### Form components

```erb
<div class="mb-3">
  <label for="post_title" class="form-label">Title</label>
  <input type="text" id="post_title" name="post[title]" class="form-control" required>
  <div class="form-text">Keep it under 80 characters.</div>
  <div class="invalid-feedback">Title can't be blank.</div>
</div>

<div class="form-check">
  <input type="checkbox" id="post_published" name="post[published]" class="form-check-input">
  <label for="post_published" class="form-check-label">Published</label>
</div>

<select name="post[category_id]" class="form-select">
  <%= options_from_collection_for_select(Category.all, :id, :name) %>
</select>
```

In Rails form helpers (`form_with`), pass `class: "form-control"` etc. explicitly — Rails doesn't add Bootstrap classes for you.

### Buttons

```erb
<%= link_to "Edit", edit_post_path(@post), class: "btn btn-outline-primary" %>
<%= button_to "Delete", @post, method: :delete,
              class: "btn btn-danger",
              data: { turbo_confirm: "Sure?" } %>
```

### Modals — Bootstrap data API

```erb
<button type="button" class="btn btn-primary"
        data-bs-toggle="modal" data-bs-target="#confirmModal">
  Open
</button>

<div class="modal fade" id="confirmModal" tabindex="-1">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Confirm</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">Are you sure?</div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
        <button type="button" class="btn btn-danger">Confirm</button>
      </div>
    </div>
  </div>
</div>
```

### Modals — Stimulus-controlled (when you need to open from code or after a Turbo Stream)

```javascript
// app/javascript/controllers/modal_controller.js
import { Controller } from "@hotwired/stimulus"
import { Modal } from "bootstrap"

export default class extends Controller {
  connect() {
    this.modal = new Modal(this.element)
    this.modal.show()
    this.element.addEventListener("hidden.bs.modal", () => this.element.remove())
  }
}
```

Pair with a Turbo Stream that appends a modal partial — the controller opens it automatically.

### Tooltips & popovers (opt-in initialization)

Bootstrap won't auto-init these. A small Stimulus controller works:

```javascript
// app/javascript/controllers/tooltip_controller.js
import { Controller } from "@hotwired/stimulus"
import { Tooltip } from "bootstrap"

export default class extends Controller {
  connect() {
    this.tooltip = new Tooltip(this.element)
  }
  disconnect() {
    this.tooltip?.dispose()
  }
}
```

Markup: `<a data-controller="tooltip" data-bs-title="hello">link</a>`.

---

## Common Stimulus + Bootstrap recipes

### Auto-dismissing flash messages

```javascript
// app/javascript/controllers/flash_controller.js
import { Controller } from "@hotwired/stimulus"
import { Alert } from "bootstrap"

export default class extends Controller {
  static values = { delay: { type: Number, default: 4000 } }

  connect() {
    this.timer = setTimeout(() => Alert.getOrCreateInstance(this.element).close(), this.delayValue)
  }
  disconnect() {
    clearTimeout(this.timer)
  }
}
```

```erb
<% if notice %>
  <div class="alert alert-success alert-dismissible fade show"
       role="alert"
       data-controller="flash">
    <%= notice %>
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
<% end %>
```

### Confirm before submit (replaces deprecated `data-confirm`)

```erb
<%= button_to "Delete", post_path(@post),
              method: :delete,
              class: "btn btn-danger",
              data: { turbo_confirm: "Permanently delete?" } %>
```

Turbo handles the confirm prompt; no Stimulus needed.

### Toggle visibility

```javascript
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["panel"]

  toggle() {
    this.panelTarget.classList.toggle("d-none")
  }
}
```

```erb
<div data-controller="toggle">
  <button class="btn btn-link" data-action="toggle#toggle">More</button>
  <div class="d-none" data-toggle-target="panel">…content…</div>
</div>
```

### Submit-on-change select

```javascript
// app/javascript/controllers/auto_submit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  submit() {
    this.element.requestSubmit()
  }
}
```

```erb
<%= form_with url: filter_path, method: :get, data: { controller: "auto-submit" } do |f| %>
  <%= f.select :category, Category.pluck(:name, :id),
               {},
               class: "form-select",
               data: { action: "change->auto-submit#submit" } %>
<% end %>
```

---

## Constraints and anti-patterns

- **Don't fight Turbo.** If a Stimulus controller manipulates the DOM in a way that survives a Turbo Drive visit, you need either a Turbo Frame, a Stream, or `data-turbo-permanent`.
- **Don't add jQuery.** Bootstrap 5 doesn't need it, and Stimulus + vanilla DOM APIs cover the rest.
- **Don't write large Stimulus controllers.** If a controller has more than ~80 lines or 6+ actions, split it.
- **Don't manipulate the DOM outside `this.element`.** Stimulus controllers are scoped — global queries (`document.querySelector`) are a smell.
- **Don't use IDs for hooks.** Use `data-controller` and `data-foo-target` instead — IDs are for users, hooks are for behavior.
- **Don't pass server state via data attributes if it could leak.** Values are public DOM.
- **Don't auto-initialize Bootstrap components globally** when you only need a few. Use Stimulus controllers per component for clean lifecycle.
