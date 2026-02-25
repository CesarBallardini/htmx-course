# Chapter 21 -- Inline Editing (Click-to-Edit)

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 20](20-response-headers-and-server-control.md) introduced response headers (`HX-Trigger`,
`HX-Retarget`, `HX-Reswap`) for server-driven control of HTMX behaviour.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Implement the click-to-edit pattern: a read-only display view transforms into an editable form on click, and reverts to the display view on save or cancel.
2. Use `hx-get` to fetch the edit form and `hx-put` (via method override) to save, both with `hx-swap="outerHTML"` targeting the current element via `hx.target(hx.This)`.
3. Handle the Escape key to cancel editing using `_hyperscript` with `on keyup[key === 'Escape']`.
4. Build multi-field inline editing with proper tab order and autofocus on the first field.
5. Validate inline edits server-side, displaying errors within the inline form without losing user input.
6. Use the `HX-Trigger` response header to notify other page sections when an item is updated.

---

## 1. Theory

### 1.1 The Click-to-Edit Pattern

Think about every task management tool you have ever used. You click on a task
title, and it transforms into an editable text field right there in the list.
You make your change, press Enter or click Save, and the field transforms back
into plain text showing your update. This is the **click-to-edit** pattern, and
it is one of the most satisfying interactions in web applications.

The pattern has two states:

```
  ┌─────────────────────────────────────────────┐
  │  DISPLAY STATE (read-only)                  │
  │                                             │
  │  "Fix bug in login flow"         [Edit]     │
  │                                             │
  │  Click the text or the Edit button...       │
  └──────────────────────┬──────────────────────┘
                         │
                         v
  ┌─────────────────────────────────────────────┐
  │  EDIT STATE (form)                          │
  │                                             │
  │  ┌──────────────────────────────────────┐   │
  │  │ Fix bug in login flow                │   │
  │  └──────────────────────────────────────┘   │
  │                                             │
  │               [Cancel]  [Save]              │
  │                                             │
  │  Click Save, press Enter, press Escape...   │
  └──────────────────────┬──────────────────────┘
                         │
                         v
  ┌─────────────────────────────────────────────┐
  │  DISPLAY STATE (read-only, updated)         │
  │                                             │
  │  "Fix bug in login flow"         [Edit]     │
  │                                             │
  └─────────────────────────────────────────────┘
```

**When to use inline editing:**

- Editing a single field or a small group of related fields (title, description).
- The item is part of a list and navigating to a separate edit page would break
  the user's flow.
- Quick corrections are common -- fixing a typo, updating a status, renaming
  something.

**When NOT to use inline editing:**

- The edit form is complex with many fields, file uploads, or rich text editors.
  Use a separate page or a modal dialog (which we will cover in [Chapter 22](22-modal-dialogs.md)).
- The item needs context that is not visible in the list view. A dedicated edit
  page can provide that context.

The beauty of the click-to-edit pattern with HTMX is that it requires no
client-side state management. The server renders the display view. The server
renders the edit form. The server processes the save. HTMX just swaps the HTML
back and forth. Two states, three endpoints, zero client-side JavaScript
(beyond HTMX and a touch of `_hyperscript` for keyboard handling).

### 1.2 The Request Cycle

The click-to-edit pattern involves three HTTP requests, each returning an HTML
fragment that replaces the current element:

```
  1. GET /tasks/:id/edit     → Returns the edit form
  2. PUT /tasks/:id          → Saves and returns the display view
  3. GET /tasks/:id          → Returns the display view (cancel)

  ┌──────────┐   GET /edit    ┌──────────┐   PUT (save)    ┌──────────┐
  │ Display  │ ───────────> │  Edit    │ ───────────> │ Display  │
  │  View    │              │  Form    │              │  View    │
  │          │ <─────────── │          │              │ (updated)│
  └──────────┘   GET (cancel) └──────────┘              └──────────┘
```

All three requests use the same swap strategy: `hx-swap="outerHTML"` with
`hx-target="this"`. This means the entire element -- display or form -- is
replaced by the server's response. The element itself is the target, and it
gets completely swapped out.

Why `outerHTML` instead of `innerHTML`? Because the display view and the edit
form have different root elements. The display view might be a `<div>` with
click handlers. The edit form is a `<form>` with input fields and buttons. We
need to replace the entire thing, not just its contents.

Why `hx.target(hx.This)` instead of an explicit selector? Because we want each
task item to be self-contained. If you have a list of 20 tasks, each one has
its own edit button, its own form, its own save and cancel buttons. Each one
targets itself. No ID coordination is needed between the list container and
the individual items.

### 1.3 Method Override for PUT

HTML forms only support `GET` and `POST`. But semantically, updating an
existing resource should be a `PUT` request. We faced this in [Chapter 11](../02-intermediate/11-multiple-boards-navigation.md) with
`DELETE`, and the solution is the same: the **method override** pattern.

The form declares `method="post"` (because HTML requires it), but includes a
hidden input that tells the server the intended method:

```html
<form method="post" action="/tasks/42">
  <input type="hidden" name="_method" value="put">
  <!-- form fields -->
  <button type="submit">Save</button>
</form>
```

On the server, the middleware calls `wisp.method_override(req)` before
routing. Wisp reads the `_method` field from the POST body and rewrites the
request method. By the time your router sees the request, it looks like a
genuine PUT.

When using HTMX, there is a subtlety. The HTMX `hx-put` attribute sends a
real HTTP PUT request directly -- no method override needed. But `hx-put`
bypasses the `<form>` element entirely. It sends the request with the
element's closest form data (or the `hx-include` contents), but it is not a
form submission.

For inline editing, we want the form semantics: form data serialisation,
`required` attribute enforcement, the browser's built-in validation UI. So
we use `hx-post` on the form combined with the hidden `_method=put` field.
This gives us form semantics on the client and PUT semantics on the server.

```
  Client side:                Server side:
  ┌─────────────────┐        ┌─────────────────────────────────┐
  │ <form>          │        │ wisp.method_override(req)       │
  │   hx-post       │ POST   │   → reads _method=put           │
  │   _method=put   │ ────>  │   → rewrites to PUT             │
  │   [fields]      │        │                                 │
  │ </form>         │        │ router matches http.Put         │
  └─────────────────┘        └─────────────────────────────────┘
```

### 1.4 Focus Management

When the edit form appears, the user expects the cursor to be in the first
input field, ready to type. If they have to click into the field manually, the
interaction feels clunky. Autofocus is essential.

HTML provides the `autofocus` attribute, and it works for page loads. But HTMX
swaps happen after the page has loaded. Depending on the browser and the timing
of the swap, `autofocus` may or may not work on swapped content.

The reliable solution is `_hyperscript`. We use the `on load` event, which
fires when the element is added to the DOM (including after an HTMX swap):

```html
<form _="on load set focus to the first <input/> in me">
  <input type="text" name="title" value="Fix bug">
  <input type="text" name="description" value="In the login flow">
</form>
```

The `_hyperscript` reads naturally: "On load, set focus to the first `<input>`
inside me." The `<input/>` is a CSS selector inside `_hyperscript` syntax
(angle brackets denote a query selector). The `in me` scopes the query to the
current element.

This works consistently because `_hyperscript` listens for the `htmx:load`
event, which fires after every HTMX swap. When the edit form is swapped into
the DOM, `_hyperscript` initialises it and runs the `on load` handler.

### 1.5 Cancel and Escape

Users need two ways to cancel an inline edit:

1. **Cancel button.** A visible button that fetches the display view and swaps
   it back. This is straightforward -- it is just another `hx-get` request.

2. **Escape key.** Pressing Escape should cancel the edit immediately. This is
   a keyboard convention that users expect. Every modal, dropdown, and inline
   edit in every application they have ever used responds to Escape.

The Escape key handler lives in `_hyperscript`:

```html
<form _="on keyup[key === 'Escape'] trigger click on .cancel-btn in me">
```

This says: "When a keyup event fires and the key is Escape, trigger a click
on the element with class `.cancel-btn` inside me." The cancel button already
has `hx-get` configured to fetch the display view, so triggering its click
event fires the HTMX request.

Why trigger a click on the cancel button instead of making the `hx-get`
request directly? Because the cancel button already has the correct URL,
swap strategy, and target configured. By reusing it, we avoid duplicating
that configuration.

An alternative approach uses the `htmx:abort` event:

```html
<form _="on keyup[key === 'Escape'] send htmx:abort to me
                                     then trigger click on .cancel-btn in me">
```

The `htmx:abort` cancels any in-flight HTMX request on the form (in case the
user pressed Escape while a save was pending). Then the click on the cancel
button fires the cancel request. In practice, the simpler version (without
`htmx:abort`) works fine for most cases.

### 1.6 Optimistic Feedback

When the user clicks Save, there is a brief moment while the request travels
to the server and back. During this time, the user should see visual feedback
that something is happening. Otherwise, they might click Save again, thinking
the first click did not register.

The simplest approach is to add a `.saving` class to the form during the
request, which dims the form and disables interaction:

```html
<form _="on htmx:beforeRequest add .saving to me
         on htmx:afterRequest remove .saving from me">
```

With accompanying CSS:

```css
.saving {
  opacity: 0.6;
  pointer-events: none;
}
```

The `htmx:beforeRequest` event fires just before HTMX sends the request.
The `htmx:afterRequest` event fires when the response arrives (regardless of
success or failure). Between those two events, the form appears dimmed and
does not accept clicks or keypresses.

This is not truly "optimistic" in the sense of showing the result before the
server confirms it. It is more accurately described as a loading indicator. But
the user experience is similar: the interface responds immediately to their
action (by dimming), so they know their click was registered.

For true optimistic updates -- showing the updated value before the server
responds -- you would need client-side JavaScript to temporarily update the
DOM. That is outside the scope of HTMX's server-driven model. The loading
indicator approach is the pragmatic sweet spot: immediate feedback without
client-side state.

---

## 2. Code Walkthrough

We are going to add inline editing to the tasks on a board. Each task will
show its title and description as plain text. Clicking the task or an Edit
button will swap in a form. Saving or cancelling will swap back to the
display view.

### Step 1 -- Display Component

The display component is a `<div>` that shows the task's title and
description, with an Edit button that fetches the edit form:

```gleam
// src/teamwork/tasks.gleam

import gleam/string
import hx
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import teamwork/state.{type Task}

/// Renders a task in its read-only display state.
/// Clicking the Edit button fetches the edit form and swaps it in.
pub fn task_display(task: Task) -> Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-display"),
    ],
    [
      html.div([attribute.class("task-display-content")], [
        html.h3([attribute.class("task-display-title")], [
          element.text(task.title),
        ]),
        case string.is_empty(task.description) {
          True -> element.none()
          False ->
            html.p([attribute.class("task-display-description")], [
              element.text(task.description),
            ])
        },
      ]),
      html.div([attribute.class("task-display-actions")], [
        html.span(
          [attribute.class("task-status badge-" <> task.status)],
          [element.text(task.status)],
        ),
        html.button(
          [
            hx.get("/tasks/" <> task.id <> "/edit"),
            hx.swap(hx.OuterHTML),
            hx.target(hx.This),
            attribute.class("btn btn-sm btn-secondary"),
          ],
          [element.text("Edit")],
        ),
      ]),
    ],
  )
}
```

Let's look at the HTMX attributes on the Edit button:

- `hx.get("/tasks/" <> task.id <> "/edit")` -- When clicked, send a GET
  request to fetch the edit form for this task.
- `hx.swap(hx.OuterHTML)` -- Replace the entire element (not just its
  contents) with the server's response.
- `hx.target(hx.This)` -- The target is the button itself? No -- `hx.This`
  refers to the element that triggered the request. But wait, we want to
  replace the entire `task-display` div, not just the button.

Here is the important detail: HTMX's `hx-target="this"` targets the element
that has the `hx-get` attribute, which is the button. But we want to replace
the parent `<div>`. There are two approaches:

**Approach A:** Use `hx.target(hx.Closest("div"))` to target the closest
parent `<div>`:

```gleam
hx.target(hx.Closest("div")),
```

**Approach B:** Move the `hx-get` to the parent `<div>` and use a different
trigger. We will use Approach B because it makes the entire task display
clickable, which feels more natural:

```gleam
pub fn task_display(task: Task) -> Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-display"),
      hx.get("/tasks/" <> task.id <> "/edit"),
      hx.swap(hx.OuterHTML),
      hx.target(hx.This),
      hx.trigger([hx.click()]),
    ],
    [
      html.div([attribute.class("task-display-content")], [
        html.h3([attribute.class("task-display-title")], [
          element.text(task.title),
        ]),
        case string.is_empty(task.description) {
          True -> element.none()
          False ->
            html.p([attribute.class("task-display-description")], [
              element.text(task.description),
            ])
        },
      ]),
      html.div([attribute.class("task-display-actions")], [
        html.span(
          [attribute.class("task-status badge-" <> task.status)],
          [element.text(task.status)],
        ),
      ]),
    ],
  )
}
```

Now the entire `<div>` is the HTMX-active element. `hx.target(hx.This)`
refers to the `<div>` itself. Clicking anywhere on the task display triggers
the GET request, and the server's response replaces the entire `<div>` with
the edit form. No separate Edit button needed -- the whole row is clickable.

We still show the Edit button visually (as a cursor hint), but it does not
need its own HTMX attributes.

Actually, let's keep a dedicated Edit button for accessibility. Not every user
discovers that clicking arbitrary text is an action. A visible button is
clearer. We will put the `hx-get` on the button but target the parent:

```gleam
pub fn task_display(task: Task) -> Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-display"),
    ],
    [
      html.div([attribute.class("task-display-content")], [
        html.h3([attribute.class("task-display-title")], [
          element.text(task.title),
        ]),
        case string.is_empty(task.description) {
          True -> element.none()
          False ->
            html.p([attribute.class("task-display-description")], [
              element.text(task.description),
            ])
        },
      ]),
      html.div([attribute.class("task-display-actions")], [
        html.span(
          [attribute.class("task-status badge-" <> task.status)],
          [element.text(task.status)],
        ),
        html.button(
          [
            hx.get("/tasks/" <> task.id <> "/edit"),
            hx.swap(hx.OuterHTML),
            hx.target(hx.Selector("#task-" <> task.id)),
            attribute.class("btn btn-sm btn-secondary"),
          ],
          [element.text("Edit")],
        ),
      ]),
    ],
  )
}
```

Now the button targets `#task-42` (the parent div) using `hx.target(hx.Selector("#task-" <> task.id))`. When clicked, the entire task display div is replaced by the edit form.

### Step 2 -- Edit Form Component

The edit form replaces the display view. It has the same `id` as the display
view so the swap works correctly. It includes input fields for the title and
description, plus Save and Cancel buttons:

```gleam
/// Renders the inline edit form for a task.
/// The form uses POST with _method=put for the save action.
/// Cancel fetches the display view and swaps it back.
pub fn task_edit_form(
  task: Task,
  errors: List(String),
) -> Element(t) {
  html.form(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-edit-form"),
      attribute.method("post"),
      attribute.action("/tasks/" <> task.id),
      hx.post("/tasks/" <> task.id),
      hx.swap(hx.OuterHTML),
      hx.target(hx.This),
      // Focus management: put cursor in the first input on load
      attribute("_", "on load set focus to the first <input/> in me"),
    ],
    [
      // Hidden _method field for PUT override
      html.input([
        attribute.type_("hidden"),
        attribute.name("_method"),
        attribute.value("put"),
      ]),
      // Title field
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("title-" <> task.id)], [
          element.text("Title"),
        ]),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title-" <> task.id),
          attribute.value(task.title),
          attribute.required(True),
          attribute.class("form-control"),
          attribute.autofocus(True),
        ]),
      ]),
      // Description field
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("desc-" <> task.id)], [
          element.text("Description"),
        ]),
        html.textarea(
          [
            attribute.name("description"),
            attribute.id("desc-" <> task.id),
            attribute.rows(2),
            attribute.class("form-control"),
          ],
          task.description,
        ),
      ]),
      // Validation errors
      case errors {
        [] -> element.none()
        _ ->
          html.div(
            [attribute.class("error-list")],
            list.map(errors, fn(err) {
              html.p([attribute.class("error")], [element.text(err)])
            }),
          )
      },
      // Action buttons
      html.div([attribute.class("form-actions-inline")], [
        html.button(
          [
            attribute.type_("button"),
            hx.get("/tasks/" <> task.id),
            hx.swap(hx.OuterHTML),
            hx.target(hx.Selector("#task-" <> task.id)),
            attribute.class("btn btn-sm btn-secondary cancel-btn"),
          ],
          [element.text("Cancel")],
        ),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn btn-sm btn-primary"),
          ],
          [element.text("Save")],
        ),
      ]),
    ],
  )
}
```

There is a lot going on here. Let's break it down piece by piece.

**The form element itself** has both `attribute.method("post")` and
`hx.post("/tasks/" <> task.id)`. The `method` and `action` attributes are
there for progressive enhancement -- if JavaScript is disabled, the form
still submits as a POST. When HTMX is active, the `hx-post` attribute takes
over and sends the request via AJAX.

**The hidden `_method` field** with `attribute.value("put")` tells the
server's `wisp.method_override` middleware to treat this POST as a PUT. This
is the method override pattern from [Chapter 11](../02-intermediate/11-multiple-boards-navigation.md).

**The `_hyperscript` on load handler** sets focus to the first input field
when the form appears. We also set `attribute.autofocus(True)` on the title
input as a fallback. Belt and suspenders.

**The Cancel button** has `attribute.type_("button")` -- not `"submit"`. This
is critical. Without `type="button"`, clicking Cancel would submit the form
(because buttons inside forms default to `type="submit"`). The
`type="button"` prevents form submission. Instead, the button's `hx-get`
fires and fetches the display view.

**The Cancel button targets `#task-42`** (the form's own ID). When the
display view HTML arrives, it replaces the entire form with `outerHTML`. The
form disappears, the display view returns. The user is back where they
started.

**The Save button** is `type="submit"`. Clicking it submits the form via
`hx-post`. The server validates the input, saves the changes, and returns
either the updated display view (on success) or the form with error messages
(on validation failure).

### Step 3 -- Server Handlers

We need three route handlers:

1. `GET /tasks/:id/edit` -- Return the edit form.
2. `PUT /tasks/:id` -- Validate and save, return the display view or errors.
3. `GET /tasks/:id` -- Return the display view (used by the Cancel button).

First, let's add the necessary messages to the state actor. Our Task type
needs a `description` field (if it does not have one already):

```gleam
// src/teamwork/state.gleam

pub type Task {
  Task(
    id: String,
    board_id: String,
    title: String,
    description: String,
    status: String,
  )
}

pub type Message {
  // ... existing messages ...
  GetTask(id: String, reply_to: process.Subject(Result(Task, Nil)))
  UpdateTask(
    id: String,
    title: String,
    description: String,
    reply_to: process.Subject(Result(Task, Nil)),
  )
}
```

And the handler in the actor:

```gleam
fn handle_message(
  state: State,
  message: Message,
) -> actor.Next(State, Message) {
  case message {
    // ... existing handlers ...

    GetTask(id, reply_to) -> {
      process.send(reply_to, dict.get(state.tasks, id))
      actor.continue(state)
    }

    UpdateTask(id, title, description, reply_to) -> {
      case dict.get(state.tasks, id) {
        Ok(task) -> {
          let updated = Task(..task, title: title, description: description)
          let tasks = dict.insert(state.tasks, id, updated)
          process.send(reply_to, Ok(updated))
          actor.continue(State(..state, tasks: tasks))
        }
        Error(Nil) -> {
          process.send(reply_to, Error(Nil))
          actor.continue(state)
        }
      }
    }
  }
}
```

Now the route handlers. Update the router to include the new paths:

```gleam
// src/teamwork/router.gleam

pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  case wisp.path_segments(req) {
    // ... existing routes ...
    ["tasks", task_id, "edit"] -> tasks.edit(req, ctx, task_id)
    ["tasks", task_id] -> tasks.handle(req, ctx, task_id)
    // ...
  }
}
```

And the task handler module:

```gleam
// src/teamwork/tasks.gleam

import gleam/http
import gleam/list
import gleam/otp/actor
import gleam/string
import hx
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import teamwork/state.{type Task}
import teamwork/web.{type Context}
import wisp

/// GET /tasks/:id/edit -- returns the edit form
pub fn edit(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  use <- wisp.require_method(req, http.Get)

  let task_result =
    actor.call(ctx.state, 1000, state.GetTask(task_id, _))

  case task_result {
    Error(Nil) -> wisp.not_found()
    Ok(task) -> {
      let html = task_edit_form(task, [])
      wisp.html_response(element.to_string(html), 200)
    }
  }
}

/// GET /tasks/:id -- returns the display view
/// PUT /tasks/:id -- validates and saves, returns display or errors
pub fn handle(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  case req.method {
    http.Get -> show_task(ctx, task_id)
    http.Put -> update_task(req, ctx, task_id)
    _ -> wisp.method_not_allowed([http.Get, http.Put])
  }
}

fn show_task(ctx: Context, task_id: String) -> wisp.Response {
  let task_result =
    actor.call(ctx.state, 1000, state.GetTask(task_id, _))

  case task_result {
    Error(Nil) -> wisp.not_found()
    Ok(task) -> {
      let html = task_display(task)
      wisp.html_response(element.to_string(html), 200)
    }
  }
}

fn update_task(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  let title = case list.key_find(formdata.values, "title") {
    Ok(value) -> value
    Error(Nil) -> ""
  }
  let description = case list.key_find(formdata.values, "description") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  // Validate
  let errors = validate_task(title)

  case errors {
    [] -> {
      // Save the update
      let result =
        actor.call(
          ctx.state,
          1000,
          state.UpdateTask(task_id, string.trim(title), string.trim(description), _),
        )

      case result {
        Ok(updated_task) -> {
          // Return the display view with an HX-Trigger header
          let html = task_display(updated_task)
          wisp.html_response(element.to_string(html), 200)
          |> wisp.set_header("HX-Trigger", "taskUpdated")
        }
        Error(Nil) -> wisp.not_found()
      }
    }
    _ -> {
      // Validation failed -- return the form with errors
      // We need to reconstruct the task with the user's input
      let task_with_input =
        state.Task(
          id: task_id,
          board_id: "",
          title: title,
          description: description,
          status: "",
        )
      let html = task_edit_form(task_with_input, errors)
      wisp.html_response(element.to_string(html), 422)
    }
  }
}

fn validate_task(title: String) -> List(String) {
  let trimmed = string.trim(title)
  let errors = []

  let errors = case string.is_empty(trimmed) {
    True -> ["Title cannot be empty." , ..errors]
    False -> errors
  }

  let errors = case string.length(trimmed) > 200 {
    True -> ["Title must be 200 characters or less.", ..errors]
    False -> errors
  }

  errors
}
```

Let's trace the complete lifecycle of an edit:

```
  1. User clicks "Edit" on task 42
     → Browser: GET /tasks/42/edit
     → Server: looks up task, returns task_edit_form(task, [])
     → HTMX: swaps the form into #task-42 (outerHTML)
     → _hyperscript: focuses the first input

  2. User changes the title, clicks "Save"
     → Browser: POST /tasks/42 with _method=put, title=..., description=...
     → Server: wisp.method_override → becomes PUT /tasks/42
     → Server: validates, saves, returns task_display(updated_task)
     → Server: sets HX-Trigger: taskUpdated header
     → HTMX: swaps the display view into #task-42 (outerHTML)
     → HTMX: fires "taskUpdated" event on the document

  3. (Alternative) User clicks "Cancel" or presses Escape
     → Browser: GET /tasks/42
     → Server: looks up task, returns task_display(task)
     → HTMX: swaps the display view into #task-42 (outerHTML)
```

Notice the symmetry. Steps 2 and 3 both end the same way: the server returns
the display view, HTMX swaps it in. The only difference is whether the task
data has been updated.

### Step 4 -- Focus and Keyboard Handling

We already added `_hyperscript` to the form for focus management. Let's now
add Escape key handling. We want pressing Escape anywhere in the form to
cancel the edit:

```gleam
pub fn task_edit_form(
  task: Task,
  errors: List(String),
) -> Element(t) {
  html.form(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-edit-form"),
      attribute.method("post"),
      attribute.action("/tasks/" <> task.id),
      hx.post("/tasks/" <> task.id),
      hx.swap(hx.OuterHTML),
      hx.target(hx.This),
      attribute("_",
        "on load set focus to the first <input/> in me "
        <> "on keyup[key === 'Escape'] trigger click on .cancel-btn in me"
      ),
    ],
    [
      // ... form fields (same as Step 2) ...
    ],
  )
}
```

The `_hyperscript` attribute now contains two event handlers, separated by a
space and a new `on` keyword:

1. `on load set focus to the first <input/> in me` -- Focus the first input
   when the form appears.
2. `on keyup[key === 'Escape'] trigger click on .cancel-btn in me` -- When
   Escape is pressed, simulate a click on the Cancel button.

The `[key === 'Escape']` part is an event filter. It only fires when the
`key` property of the `keyup` event equals `'Escape'`. All other keyup events
are ignored.

The `trigger click on .cancel-btn in me` part finds the element with class
`cancel-btn` inside the form and dispatches a `click` event on it. That click
event triggers the Cancel button's `hx-get`, which fetches the display view
and swaps it in.

Here is the flow:

```
  User presses Escape while editing
    |
    v
  keyup event fires on the <form>
    |
    v
  _hyperscript checks: key === 'Escape'? Yes.
    |
    v
  _hyperscript: trigger click on .cancel-btn
    |
    v
  Cancel button's hx-get fires: GET /tasks/42
    |
    v
  Server returns task_display(task)
    |
    v
  HTMX swaps display view into #task-42
    |
    v
  Edit is cancelled, display view is restored
```

### Step 5 -- Multi-Field Editing

Our edit form already has two fields: title and description. Let's make sure
the tab order and keyboard interactions work smoothly.

The natural tab order follows the DOM order: title input, description textarea,
Cancel button, Save button. This is correct -- the user tabs through the
fields first, then through the actions.

For a better keyboard experience, we can submit the form with Enter while in
the title field (single-line input). But pressing Enter in the textarea should
insert a newline, not submit. This is the browser's default behaviour, so we
get it for free:

- `<input type="text">` -- Enter submits the form.
- `<textarea>` -- Enter inserts a newline.

If you want Ctrl+Enter or Cmd+Enter to submit from the textarea, add another
`_hyperscript` handler:

```gleam
attribute("_",
  "on load set focus to the first <input/> in me "
  <> "on keyup[key === 'Escape'] trigger click on .cancel-btn in me "
  <> "on keydown[key === 'Enter' and (ctrlKey or metaKey)] "
  <> "from the first <textarea/> in me "
  <> "submit() me"
),
```

This handler listens for `keydown` events with `Enter` plus `Ctrl` (Linux/
Windows) or `Meta` (macOS) coming from the textarea, and calls `submit()` on
the form. The `from` clause scopes the event listener to the textarea only.

Here is the complete multi-field form with all keyboard handlers:

```gleam
pub fn task_edit_form(
  task: Task,
  errors: List(String),
) -> Element(t) {
  let hyperscript_handlers =
    "on load set focus to the first <input/> in me "
    <> "on keyup[key === 'Escape'] trigger click on .cancel-btn in me "
    <> "on keydown[key === 'Enter' and (ctrlKey or metaKey)] "
    <> "from the first <textarea/> in me "
    <> "submit() me"

  html.form(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-edit-form"),
      attribute.method("post"),
      attribute.action("/tasks/" <> task.id),
      hx.post("/tasks/" <> task.id),
      hx.swap(hx.OuterHTML),
      hx.target(hx.This),
      attribute("_", hyperscript_handlers),
    ],
    [
      html.input([
        attribute.type_("hidden"),
        attribute.name("_method"),
        attribute.value("put"),
      ]),
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("title-" <> task.id)], [
          element.text("Title"),
        ]),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title-" <> task.id),
          attribute.value(task.title),
          attribute.required(True),
          attribute.class("form-control"),
          attribute.autofocus(True),
        ]),
      ]),
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("desc-" <> task.id)], [
          element.text("Description"),
        ]),
        html.textarea(
          [
            attribute.name("description"),
            attribute.id("desc-" <> task.id),
            attribute.rows(2),
            attribute.class("form-control"),
            attribute.placeholder("Optional description..."),
          ],
          task.description,
        ),
      ]),
      case errors {
        [] -> element.none()
        _ ->
          html.div(
            [attribute.class("error-list")],
            list.map(errors, fn(err) {
              html.p([attribute.class("error")], [element.text(err)])
            }),
          )
      },
      html.div([attribute.class("form-actions-inline")], [
        html.button(
          [
            attribute.type_("button"),
            hx.get("/tasks/" <> task.id),
            hx.swap(hx.OuterHTML),
            hx.target(hx.Selector("#task-" <> task.id)),
            attribute.class("btn btn-sm btn-secondary cancel-btn"),
          ],
          [element.text("Cancel")],
        ),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn btn-sm btn-primary"),
          ],
          [element.text("Save")],
        ),
      ]),
    ],
  )
}
```

### Step 6 -- Event Coordination with HX-Trigger

When a task is updated, other parts of the page might need to know about it.
For example, a task summary sidebar, a progress counter, or a colleague's
browser window connected via SSE ([Chapter 13](../03-advanced/13-real-time-with-sse.md)).

In Step 3, our `update_task` handler already sets the response header:

```gleam
wisp.set_header("HX-Trigger", "taskUpdated")
```

This tells HTMX to fire a `taskUpdated` event on the document after the swap
completes. Any element on the page can listen for this event and react:

```gleam
/// A task counter that refreshes itself when a task is updated.
pub fn task_counter(board_id: String, count: Int) -> Element(t) {
  html.div(
    [
      attribute.id("task-counter"),
      attribute.class("task-counter"),
      hx.get("/boards/" <> board_id <> "/task-count"),
      hx.swap(hx.OuterHTML),
      hx.target(hx.This),
      hx.trigger([hx.custom("taskUpdated") |> hx.with_from(hx.Document)]),
    ],
    [
      element.text(
        int.to_string(count) <> " tasks",
      ),
    ],
  )
}
```

This counter element has `hx-trigger="taskUpdated from:document"`. When the
`taskUpdated` event fires (because the server sent the `HX-Trigger: taskUpdated`
header), this element automatically sends a GET request to refresh itself.

The flow:

```
  User saves an inline edit
    |
    v
  Server returns updated task_display + HX-Trigger: taskUpdated
    |
    v
  HTMX swaps in the display view (main swap)
    |
    v
  HTMX fires "taskUpdated" event on the document
    |
    v
  Task counter hears "taskUpdated", sends GET /boards/.../task-count
    |
    v
  Server returns updated counter HTML
    |
    v
  Counter refreshes itself
```

Two elements update from a single user action. The inline edit returns the
display view directly (via the main swap). The counter refreshes itself via
a triggered request. This is the event coordination pattern from [Chapter 20](20-response-headers-and-server-control.md)
applied to inline editing.

For more complex payloads, you can use JSON in the `HX-Trigger` header:

```gleam
wisp.set_header(
  "HX-Trigger",
  "{\"taskUpdated\": {\"taskId\": \"" <> task_id <> "\"}}",
)
```

This passes the task ID along with the event, which listeners can access via
`event.detail`. But for most cases, a simple event name is sufficient.

### Step 7 -- Loading State with _hyperscript

Finally, let's add visual feedback during the save request. We will add a
`.saving` class to the form while the HTMX request is in flight:

```gleam
let hyperscript_handlers =
  "on load set focus to the first <input/> in me "
  <> "on keyup[key === 'Escape'] trigger click on .cancel-btn in me "
  <> "on keydown[key === 'Enter' and (ctrlKey or metaKey)] "
  <> "from the first <textarea/> in me "
  <> "submit() me "
  <> "on htmx:beforeRequest add .saving to me "
  <> "on htmx:afterRequest remove .saving from me"
```

Two new handlers:

- `on htmx:beforeRequest add .saving to me` -- When an HTMX request starts
  (the save), add the `.saving` class to the form.
- `on htmx:afterRequest remove .saving from me` -- When the response arrives
  (success or failure), remove the `.saving` class.

The CSS for `.saving`:

```css
.task-edit-form.saving {
  opacity: 0.6;
  pointer-events: none;
  position: relative;
}

.task-edit-form.saving::after {
  content: "Saving...";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
}
```

The form dims and shows a "Saving..." overlay. The `pointer-events: none`
prevents any interaction while the save is in progress. Once the server
responds, the form either gets replaced by the display view (success) or
gets the `.saving` class removed and shows validation errors (failure).

Note that on success, the `.saving` class removal is irrelevant because the
entire form is replaced by the display view. But on validation failure (422),
the form is replaced by itself with error messages, and the `on load` handler
fires again to set focus. The UX remains smooth either way.

---

## 3. Full Code Listing

Here is every file needed for the inline editing feature, ready to copy and
paste into the Teamwork project.

### Project Structure (relevant files)

```
teamwork/
├── gleam.toml
├── priv/
│   └── static/
│       └── css/
│           └── style.css
├── src/
│   ├── teamwork.gleam
│   └── teamwork/
│       ├── web.gleam
│       ├── router.gleam
│       ├── layout.gleam
│       ├── state.gleam
│       └── tasks.gleam          ← NEW: inline editing views + handlers
└── test/
    └── teamwork_test.gleam
```

### `gleam.toml`

```toml
name = "teamwork"
version = "1.0.0"
target = "erlang"

[dependencies]
gleam_stdlib = ">= 0.50.0 and < 1.0.0"
gleam_erlang = ">= 1.0.0 and < 2.0.0"
gleam_http = ">= 4.0.0 and < 5.0.0"
gleam_otp = ">= 1.0.0 and < 2.0.0"
mist = ">= 5.0.0 and < 6.0.0"
wisp = ">= 2.0.0 and < 3.0.0"
lustre = ">= 5.0.0 and < 6.0.0"
hx = ">= 3.0.0 and < 4.0.0"

[dev-dependencies]
gleeunit = ">= 1.0.0 and < 2.0.0"
```

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process
import mist
import teamwork/router
import teamwork/state
import teamwork/web.{Context}
import wisp
import wisp/wisp_mist

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let assert Ok(state_subject) = state.start()

  let ctx = Context(
    state: state_subject,
    static_directory: static_directory(),
  )

  let handler = fn(req) {
    router.middleware(req, ctx, fn(req) { router.handle_request(req, ctx) })
  }

  let assert Ok(_) =
    wisp_mist.handler(handler, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

fn static_directory() -> String {
  let assert Ok(priv_directory) = wisp.priv_directory("teamwork")
  priv_directory <> "/static"
}
```

### `src/teamwork/web.gleam`

```gleam
import gleam/erlang/process
import teamwork/state

pub type Context {
  Context(state: process.Subject(state.Message), static_directory: String)
}
```

### `src/teamwork/state.gleam`

```gleam
import gleam/dict.{type Dict}
import gleam/erlang/process
import gleam/int
import gleam/list
import gleam/otp/actor

pub type Task {
  Task(
    id: String,
    board_id: String,
    title: String,
    description: String,
    status: String,
  )
}

pub type Message {
  GetBoards(reply_to: process.Subject(List(Board)))
  GetBoard(id: String, reply_to: process.Subject(Result(Board, Nil)))
  CreateBoard(
    name: String,
    description: String,
    reply_to: process.Subject(Board),
  )
  DeleteBoard(id: String, reply_to: process.Subject(Result(Nil, Nil)))
  GetTasks(board_id: String, reply_to: process.Subject(List(Task)))
  GetTask(id: String, reply_to: process.Subject(Result(Task, Nil)))
  CreateTask(
    board_id: String,
    title: String,
    reply_to: process.Subject(Task),
  )
  UpdateTask(
    id: String,
    title: String,
    description: String,
    reply_to: process.Subject(Result(Task, Nil)),
  )
  DeleteTask(task_id: String, reply_to: process.Subject(Result(Nil, Nil)))
}

pub type Board {
  Board(id: String, name: String, description: String)
}

pub type State {
  State(
    boards: Dict(String, Board),
    tasks: Dict(String, Task),
    next_id: Int,
  )
}

fn new_state() -> State {
  State(boards: dict.new(), tasks: dict.new(), next_id: 1)
}

fn next_id(state: State) -> #(String, State) {
  let id = int.to_string(state.next_id)
  #(id, State(..state, next_id: state.next_id + 1))
}

fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}

pub fn start() -> Result(process.Subject(Message), actor.StartError) {
  actor.new(new_state())
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}

fn handle_message(
  state: State,
  message: Message,
) -> actor.Next(State, Message) {
  case message {
    GetBoards(reply_to) -> {
      process.send(reply_to, dict.values(state.boards))
      actor.continue(state)
    }

    GetBoard(id, reply_to) -> {
      process.send(reply_to, dict.get(state.boards, id))
      actor.continue(state)
    }

    CreateBoard(name, description, reply_to) -> {
      let #(id, state) = next_id(state)
      let new_board = Board(id: id, name: name, description: description)
      let boards = dict.insert(state.boards, id, new_board)
      process.send(reply_to, new_board)
      actor.continue(State(..state, boards: boards))
    }

    DeleteBoard(id, reply_to) -> {
      case dict.get(state.boards, id) {
        Ok(_) -> {
          let boards = dict.delete(state.boards, id)
          let tasks =
            state.tasks
            |> dict.to_list
            |> list.filter(fn(pair) { { pair.1 }.board_id != id })
            |> dict.from_list
          process.send(reply_to, Ok(Nil))
          actor.continue(State(..state, boards: boards, tasks: tasks))
        }
        Error(Nil) -> {
          process.send(reply_to, Error(Nil))
          actor.continue(state)
        }
      }
    }

    GetTasks(board_id, reply_to) -> {
      let board_tasks =
        state.tasks
        |> dict.values
        |> list.filter(fn(task) { task.board_id == board_id })
      process.send(reply_to, board_tasks)
      actor.continue(state)
    }

    GetTask(id, reply_to) -> {
      process.send(reply_to, dict.get(state.tasks, id))
      actor.continue(state)
    }

    CreateTask(board_id, title, reply_to) -> {
      let #(id, state) = next_id(state)
      let task = Task(
        id: id,
        board_id: board_id,
        title: title,
        description: "",
        status: "todo",
      )
      let tasks = dict.insert(state.tasks, id, task)
      process.send(reply_to, task)
      actor.continue(State(..state, tasks: tasks))
    }

    UpdateTask(id, title, description, reply_to) -> {
      case dict.get(state.tasks, id) {
        Ok(task) -> {
          let updated = Task(..task, title: title, description: description)
          let tasks = dict.insert(state.tasks, id, updated)
          process.send(reply_to, Ok(updated))
          actor.continue(State(..state, tasks: tasks))
        }
        Error(Nil) -> {
          process.send(reply_to, Error(Nil))
          actor.continue(state)
        }
      }
    }

    DeleteTask(task_id, reply_to) -> {
      case dict.get(state.tasks, task_id) {
        Ok(_) -> {
          let tasks = dict.delete(state.tasks, task_id)
          process.send(reply_to, Ok(Nil))
          actor.continue(State(..state, tasks: tasks))
        }
        Error(Nil) -> {
          process.send(reply_to, Error(Nil))
          actor.continue(state)
        }
      }
    }
  }
}
```

### `src/teamwork/layout.gleam`

```gleam
import gleam/list
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import lustre/element/keyed

pub type Breadcrumb {
  Breadcrumb(label: String, href: String)
}

pub fn layout(
  title: String,
  breadcrumbs: List(Breadcrumb),
  content: Element(t),
) -> Element(t) {
  html.html([attribute("lang", "en")], [
    html.head([], [
      html.meta([attribute("charset", "UTF-8")]),
      html.meta([
        attribute("name", "viewport"),
        attribute("content", "width=device-width, initial-scale=1.0"),
      ]),
      html.title([], title),
      html.link([
        attribute("rel", "stylesheet"),
        attribute("href", "/static/css/style.css"),
      ]),
      html.script(
        [attribute("src", "https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
      html.script(
        [attribute("src", "https://unpkg.com/hyperscript.org@0.9.14")],
        "",
      ),
    ]),
    html.body([attribute("hx-boost", "true")], [
      nav_bar(),
      render_breadcrumbs(breadcrumbs),
      html.main([attribute.class("container")], [content]),
    ]),
  ])
}

fn nav_bar() -> Element(t) {
  html.nav([attribute.class("nav-bar")], [
    html.div([attribute.class("nav-brand")], [
      html.a([attribute.href("/")], [element.text("Teamwork")]),
    ]),
    html.div([attribute.class("nav-links")], [
      html.a([attribute.href("/boards")], [element.text("Boards")]),
    ]),
  ])
}

fn render_breadcrumbs(crumbs: List(Breadcrumb)) -> Element(t) {
  case crumbs {
    [] -> element.none()
    _ ->
      html.nav([attribute.class("breadcrumbs")], [
        keyed.ol(
          [],
          list.map(crumbs, fn(crumb) {
            #(
              crumb.href,
              html.li([], [
                html.a([attribute.href(crumb.href)], [
                  element.text(crumb.label),
                ]),
              ]),
            )
          }),
        ),
      ])
  }
}
```

### `src/teamwork/router.gleam`

```gleam
import gleam/http
import gleam/list
import gleam/otp/actor
import lustre/attribute.{attribute}
import lustre/element
import lustre/element/html
import teamwork/layout.{Breadcrumb}
import teamwork/state
import teamwork/tasks
import teamwork/web.{type Context}
import wisp

pub fn middleware(
  req: wisp.Request,
  ctx: Context,
  handle_request: fn(wisp.Request) -> wisp.Response,
) -> wisp.Response {
  let req = wisp.method_override(req)
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: ctx.static_directory)

  handle_request(req)
}

pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  case wisp.path_segments(req) {
    [] -> home_page(req)
    ["boards"] -> boards_index(req, ctx)
    ["boards", board_id] -> board_show(req, ctx, board_id)
    ["boards", board_id, "tasks"] -> tasks.create(req, ctx, board_id)
    ["tasks", task_id, "edit"] -> tasks.edit(req, ctx, task_id)
    ["tasks", task_id] -> tasks.handle(req, ctx, task_id)
    _ -> not_found_page()
  }
}

fn home_page(_req: wisp.Request) -> wisp.Response {
  let content =
    layout.layout(
      "Teamwork",
      [],
      html.div([], [
        html.h1([], [element.text("Welcome to Teamwork")]),
        html.p([], [
          element.text(
            "Organize your team's work across multiple boards. "
            <> "Click any task to edit it inline.",
          ),
        ]),
        html.a(
          [attribute.href("/boards"), attribute.class("btn btn-primary")],
          [element.text("View Your Boards")],
        ),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn boards_index(req: wisp.Request, ctx: Context) -> wisp.Response {
  // Simplified -- see Chapter 11 for full board listing
  use <- wisp.require_method(req, http.Get)

  let content =
    layout.layout(
      "Boards -- Teamwork",
      [Breadcrumb("Home", "/")],
      html.div([], [
        html.h1([], [element.text("Your Boards")]),
        html.p([], [element.text("Select a board to view and edit tasks.")]),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn board_show(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  // Simplified -- focuses on inline editing of tasks
  use <- wisp.require_method(req, http.Get)

  let board_tasks =
    actor.call(ctx.state, 1000, state.GetTasks(board_id, _))

  let content =
    layout.layout(
      "Board -- Teamwork",
      [Breadcrumb("Home", "/"), Breadcrumb("Boards", "/boards")],
      html.div([], [
        html.h1([], [element.text("Board Tasks")]),
        html.div(
          [attribute.id("task-list"), attribute.class("task-list")],
          list.map(board_tasks, tasks.task_display),
        ),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn not_found_page() -> wisp.Response {
  let content =
    layout.layout(
      "Not Found -- Teamwork",
      [Breadcrumb("Home", "/")],
      html.div([], [
        html.h1([], [element.text("404 -- Page Not Found")]),
        html.p([], [
          element.text("The page you are looking for does not exist."),
        ]),
        html.a(
          [attribute.href("/"), attribute.class("btn btn-primary")],
          [element.text("Back to Home")],
        ),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    404,
  )
}
```

### `src/teamwork/tasks.gleam`

This is the main file for this chapter. It contains the display component,
the edit form component, the server handlers, and the validation logic.

```gleam
import gleam/http
import gleam/list
import gleam/otp/actor
import gleam/string
import hx
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import teamwork/state.{type Task}
import teamwork/web.{type Context}
import wisp

// ── Display Component ──────────────────────────────────────────

/// Renders a task in its read-only display state.
/// The Edit button fetches the edit form and replaces this element.
pub fn task_display(task: Task) -> Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-display"),
    ],
    [
      html.div([attribute.class("task-display-content")], [
        html.h3([attribute.class("task-display-title")], [
          element.text(task.title),
        ]),
        case string.is_empty(task.description) {
          True -> element.none()
          False ->
            html.p([attribute.class("task-display-description")], [
              element.text(task.description),
            ])
        },
      ]),
      html.div([attribute.class("task-display-actions")], [
        html.span(
          [attribute.class("task-status badge-" <> task.status)],
          [element.text(task.status)],
        ),
        html.button(
          [
            hx.get("/tasks/" <> task.id <> "/edit"),
            hx.swap(hx.OuterHTML),
            hx.target(hx.Selector("#task-" <> task.id)),
            attribute.class("btn btn-sm btn-secondary"),
          ],
          [element.text("Edit")],
        ),
      ]),
    ],
  )
}

// ── Edit Form Component ────────────────────────────────────────

/// Renders the inline edit form for a task.
/// Uses POST + _method=put for method override.
/// Includes _hyperscript for focus, Escape handling, and saving feedback.
pub fn task_edit_form(task: Task, errors: List(String)) -> Element(t) {
  let hyperscript_handlers =
    "on load set focus to the first <input/> in me "
    <> "on keyup[key === 'Escape'] trigger click on .cancel-btn in me "
    <> "on keydown[key === 'Enter' and (ctrlKey or metaKey)] "
    <> "from the first <textarea/> in me "
    <> "submit() me "
    <> "on htmx:beforeRequest add .saving to me "
    <> "on htmx:afterRequest remove .saving from me"

  html.form(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-edit-form"),
      attribute.method("post"),
      attribute.action("/tasks/" <> task.id),
      hx.post("/tasks/" <> task.id),
      hx.swap(hx.OuterHTML),
      hx.target(hx.This),
      attribute("_", hyperscript_handlers),
    ],
    [
      // Hidden _method field for PUT override
      html.input([
        attribute.type_("hidden"),
        attribute.name("_method"),
        attribute.value("put"),
      ]),
      // Title field
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("title-" <> task.id)], [
          element.text("Title"),
        ]),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title-" <> task.id),
          attribute.value(task.title),
          attribute.required(True),
          attribute.class("form-control"),
          attribute.autofocus(True),
        ]),
      ]),
      // Description field
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("desc-" <> task.id)], [
          element.text("Description"),
        ]),
        html.textarea(
          [
            attribute.name("description"),
            attribute.id("desc-" <> task.id),
            attribute.rows(2),
            attribute.class("form-control"),
            attribute.placeholder("Optional description..."),
          ],
          task.description,
        ),
      ]),
      // Validation errors
      case errors {
        [] -> element.none()
        _ ->
          html.div(
            [attribute.class("error-list")],
            list.map(errors, fn(err) {
              html.p([attribute.class("error")], [element.text(err)])
            }),
          )
      },
      // Action buttons
      html.div([attribute.class("form-actions-inline")], [
        html.button(
          [
            attribute.type_("button"),
            hx.get("/tasks/" <> task.id),
            hx.swap(hx.OuterHTML),
            hx.target(hx.Selector("#task-" <> task.id)),
            attribute.class("btn btn-sm btn-secondary cancel-btn"),
          ],
          [element.text("Cancel")],
        ),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn btn-sm btn-primary"),
          ],
          [element.text("Save")],
        ),
      ]),
    ],
  )
}

// ── Validation ─────────────────────────────────────────────────

fn validate_task(title: String) -> List(String) {
  let trimmed = string.trim(title)
  let errors = []

  let errors = case string.is_empty(trimmed) {
    True -> ["Title cannot be empty.", ..errors]
    False -> errors
  }

  let errors = case string.length(trimmed) > 200 {
    True -> ["Title must be 200 characters or less.", ..errors]
    False -> errors
  }

  errors
}

// ── Route Handlers ─────────────────────────────────────────────

/// GET /tasks/:id/edit -- returns the edit form fragment
pub fn edit(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  use <- wisp.require_method(req, http.Get)

  let task_result =
    actor.call(ctx.state, 1000, state.GetTask(task_id, _))

  case task_result {
    Error(Nil) -> wisp.not_found()
    Ok(task) -> {
      let html = task_edit_form(task, [])
      wisp.html_response(element.to_string(html), 200)
    }
  }
}

/// GET /tasks/:id -- returns the display view fragment
/// PUT /tasks/:id -- validates, saves, returns display or errors
pub fn handle(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  case req.method {
    http.Get -> show(ctx, task_id)
    http.Put -> update(req, ctx, task_id)
    _ -> wisp.method_not_allowed([http.Get, http.Put])
  }
}

fn show(ctx: Context, task_id: String) -> wisp.Response {
  let task_result =
    actor.call(ctx.state, 1000, state.GetTask(task_id, _))

  case task_result {
    Error(Nil) -> wisp.not_found()
    Ok(task) -> {
      let html = task_display(task)
      wisp.html_response(element.to_string(html), 200)
    }
  }
}

fn update(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  let title = case list.key_find(formdata.values, "title") {
    Ok(value) -> value
    Error(Nil) -> ""
  }
  let description = case list.key_find(formdata.values, "description") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  // Validate
  let errors = validate_task(title)

  case errors {
    [] -> {
      // Validation passed -- save and return display view
      let result =
        actor.call(
          ctx.state,
          1000,
          state.UpdateTask(
            task_id,
            string.trim(title),
            string.trim(description),
            _,
          ),
        )

      case result {
        Ok(updated_task) -> {
          let html = task_display(updated_task)
          wisp.html_response(element.to_string(html), 200)
          |> wisp.set_header("HX-Trigger", "taskUpdated")
        }
        Error(Nil) -> wisp.not_found()
      }
    }
    _ -> {
      // Validation failed -- return form with errors and user input
      let task_with_input =
        state.Task(
          id: task_id,
          board_id: "",
          title: title,
          description: description,
          status: "",
        )
      let html = task_edit_form(task_with_input, errors)
      wisp.html_response(element.to_string(html), 422)
    }
  }
}

/// POST /boards/:board_id/tasks -- create a new task on a board
pub fn create(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use <- wisp.require_method(req, http.Post)
  use formdata <- wisp.require_form(req)

  let title = case list.key_find(formdata.values, "title") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  case string.is_empty(string.trim(title)) {
    True -> wisp.redirect("/boards/" <> board_id)
    False -> {
      let _ =
        actor.call(
          ctx.state,
          1000,
          state.CreateTask(board_id, string.trim(title), _),
        )
      wisp.redirect("/boards/" <> board_id)
    }
  }
}
```

### `priv/static/css/style.css`

Add these styles to your existing stylesheet. They handle both the display
view and the edit form, including transitions and the saving state:

```css
/* ── Base Reset ───────────────────────────────────── */

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: system-ui, -apple-system, sans-serif;
  line-height: 1.6;
  color: #1a1a2e;
  background-color: #f0f0f5;
}

/* ── Navigation ───────────────────────────────────── */

.nav-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 2rem;
  background: #16213e;
  color: #fff;
}

.nav-brand a {
  color: #fff;
  text-decoration: none;
  font-size: 1.25rem;
  font-weight: 700;
}

.nav-links a {
  color: #ccc;
  text-decoration: none;
  margin-left: 1.5rem;
}

.nav-links a:hover {
  color: #fff;
}

/* ── Breadcrumbs ──────────────────────────────────── */

.breadcrumbs {
  padding: 0.75rem 2rem;
  background: #e8e8f0;
  font-size: 0.875rem;
}

.breadcrumbs ol {
  list-style: none;
  display: flex;
  gap: 0.5rem;
}

.breadcrumbs li:not(:last-child)::after {
  content: ">";
  margin-left: 0.5rem;
  color: #999;
}

.breadcrumbs a {
  color: #16213e;
  text-decoration: none;
}

.breadcrumbs a:hover {
  text-decoration: underline;
}

/* ── Layout ───────────────────────────────────────── */

.container {
  max-width: 900px;
  margin: 2rem auto;
  padding: 0 1rem;
}

/* ── Buttons ──────────────────────────────────────── */

.btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.btn-primary {
  background: #16213e;
  color: #fff;
}

.btn-primary:hover {
  background: #1a2744;
}

.btn-secondary {
  background: #e0e0e8;
  color: #333;
}

.btn-secondary:hover {
  background: #d0d0d8;
}

.btn-danger {
  background: #dc3545;
  color: #fff;
}

.btn-danger:hover {
  background: #c82333;
}

.btn-sm {
  padding: 0.25rem 0.5rem;
  font-size: 0.75rem;
}

/* ── Forms ────────────────────────────────────────── */

.form-group {
  margin-bottom: 0.75rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.25rem;
  font-weight: 600;
  font-size: 0.8rem;
  color: #16213e;
}

.form-control {
  width: 100%;
  padding: 0.4rem 0.6rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 0.9rem;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.form-control:focus {
  outline: none;
  border-color: #4a90d9;
  box-shadow: 0 0 0 2px rgba(74, 144, 217, 0.25);
}

.form-actions-inline {
  display: flex;
  gap: 0.5rem;
  justify-content: flex-end;
  margin-top: 0.5rem;
}

/* ── Validation Errors ────────────────────────────── */

.error-list {
  margin-bottom: 0.5rem;
}

.error {
  color: #dc3545;
  font-size: 0.8rem;
  margin-bottom: 0.25rem;
}

/* ── Task List ────────────────────────────────────── */

.task-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

/* ── Task Display (read-only state) ───────────────── */

.task-display {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  padding: 0.75rem 1rem;
  background: #fff;
  border: 1px solid #e0e0e8;
  border-radius: 6px;
  transition: box-shadow 0.15s ease;
}

.task-display:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.task-display-content {
  flex: 1;
  min-width: 0;
}

.task-display-title {
  font-size: 0.95rem;
  font-weight: 600;
  color: #1a1a2e;
  margin-bottom: 0.15rem;
}

.task-display-description {
  font-size: 0.8rem;
  color: #666;
  line-height: 1.4;
}

.task-display-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-shrink: 0;
}

.task-status {
  display: inline-block;
  padding: 0.125rem 0.5rem;
  border-radius: 12px;
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.03em;
}

.badge-todo {
  background: #ffeeba;
  color: #856404;
}

.badge-done {
  background: #d4edda;
  color: #155724;
}

.badge-in-progress {
  background: #cce5ff;
  color: #004085;
}

/* ── Task Edit Form (editing state) ───────────────── */

.task-edit-form {
  padding: 0.75rem 1rem;
  background: #fff;
  border: 2px solid #4a90d9;
  border-radius: 6px;
  transition: opacity 0.2s ease;
}

/* ── Saving State ─────────────────────────────────── */

.task-edit-form.saving {
  opacity: 0.6;
  pointer-events: none;
  position: relative;
}

.task-edit-form.saving::after {
  content: "Saving...";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
}

/* ── Transition: Display → Edit (swap in) ─────────── */

.task-edit-form.htmx-added {
  opacity: 0;
  transform: scaleY(0.95);
}

.task-edit-form {
  transition: opacity 0.15s ease, transform 0.15s ease;
}

/* ── Transition: Edit → Display (swap back) ───────── */

.task-display.htmx-added {
  opacity: 0;
}

.task-display {
  transition: opacity 0.15s ease;
}
```

---

## 4. Exercises

Put what you have learned into practice. Each exercise builds on the previous
one.

### Exercise 1: Basic Click-to-Edit

Implement the click-to-edit pattern for a single task.

1. Start the server, navigate to a board, and create a task.
2. Click the "Edit" button on the task.
3. Verify that the edit form appears with the task's title pre-filled.
4. Change the title, click "Save".
5. Verify that the display view reappears with the updated title.
6. Open the browser's Network tab and confirm that:
   - Clicking "Edit" sends `GET /tasks/:id/edit`.
   - Clicking "Save" sends `POST /tasks/:id` with `_method=put`.
   - The POST is converted to PUT by the middleware.

**Acceptance criteria:** You can edit a task title inline and see the change
immediately without a page reload.

### Exercise 2: Cancel and Escape

Test both cancel mechanisms.

1. Click "Edit" on a task.
2. Change the title to something new but do NOT save.
3. Click "Cancel".
4. Verify the original title is restored -- your unsaved change is discarded.
5. Click "Edit" again.
6. Press the Escape key.
7. Verify the edit is cancelled and the original display view is restored.

**Acceptance criteria:** Both the Cancel button and the Escape key discard
changes and restore the display view.

### Exercise 3: Validation Errors

Test server-side validation within the inline form.

1. Click "Edit" on a task.
2. Clear the title field (leave it empty).
3. Click "Save".
4. Verify that the form reappears with an error message: "Title cannot be
   empty."
5. Verify that the description field still contains whatever you typed.
6. Type a new valid title and save again.
7. Verify that the save succeeds and the display view appears.

**Acceptance criteria:** Validation errors are displayed inline without
disrupting the layout. The form preserves user input on error.

### Exercise 4: Event Coordination

Add a task counter to the board page that updates when a task is edited.

1. Add a `<div id="task-counter">` to the board page that shows "N tasks".
2. Give it `hx-get="/boards/:board_id/task-count"` with
   `hx-trigger="taskUpdated from:document"`.
3. Create a `GET /boards/:board_id/task-count` endpoint that returns the
   counter HTML.
4. Edit a task and verify that the counter refreshes automatically after
   the save.

**Hint:** The `HX-Trigger: taskUpdated` header is already set in the update
handler.

**Acceptance criteria:** The task counter refreshes itself after every inline
edit without requiring a separate user action.

### Exercise 5: Inline Status Toggle

Extend inline editing to include a status dropdown.

1. Add a `<select>` field to the edit form with options: "todo",
   "in-progress", "done".
2. Pre-select the task's current status.
3. Save the status along with the title and description.
4. Update the `UpdateTask` message and handler to accept a status field.
5. Verify that the status badge in the display view reflects the new status
   after saving.

**Acceptance criteria:** You can change a task's status through inline editing.
The status badge updates to show the correct colour and label.

---

## 5. Exercise Solution Hints

Try to solve each exercise on your own before reading these hints.

### Hint for Exercise 1

If clicking "Edit" does not work, check:

- The `hx-target` on the Edit button matches the `id` of the parent div.
  Use `hx.target(hx.Selector("#task-" <> task.id))` on the button.
- The edit form's `id` matches the display div's `id`. Both should be
  `"task-" <> task.id`.
- The middleware calls `wisp.method_override(req)` before routing.

If the PUT is not reaching your handler, check the hidden input:

```gleam
html.input([
  attribute.type_("hidden"),
  attribute.name("_method"),
  attribute.value("put"),
])
```

And confirm the router matches `http.Put`:

```gleam
http.Put -> update(req, ctx, task_id)
```

### Hint for Exercise 2

If Escape does not cancel, check that:

- `_hyperscript` is loaded in the `<head>`: `https://unpkg.com/hyperscript.org@0.9.14`
- The `_` attribute on the form contains the `on keyup[key === 'Escape']`
  handler.
- The Cancel button has the class `cancel-btn` (matching the `_hyperscript`
  selector `.cancel-btn`).

If the Cancel button does not work, check that it has `attribute.type_("button")`
and NOT `attribute.type_("submit")`. A submit button inside a form will trigger
the form's `hx-post`, not the button's `hx-get`.

### Hint for Exercise 3

The key to preserving user input on validation failure is reconstructing the
task with the submitted values:

```gleam
let task_with_input =
  state.Task(
    id: task_id,
    board_id: "",
    title: title,
    description: description,
    status: "",
  )
let html = task_edit_form(task_with_input, errors)
wisp.html_response(element.to_string(html), 422)
```

The `task_edit_form` function reads `task.title` and `task.description` to
populate the form fields. By passing the submitted values (not the saved
values), the form preserves what the user typed.

The `422` status code tells HTMX that validation failed. Because HTMX swaps
422 responses the same way as 200 responses (by default), the form with error
messages replaces the current form. No special client-side error handling
needed.

### Hint for Exercise 4

The counter endpoint:

```gleam
// In router.gleam, add:
["boards", board_id, "task-count"] -> tasks.task_count(req, ctx, board_id)

// In tasks.gleam, add:
pub fn task_count(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use <- wisp.require_method(req, http.Get)

  let board_tasks =
    actor.call(ctx.state, 1000, state.GetTasks(board_id, _))
  let count = list.length(board_tasks)

  let html =
    html.div(
      [
        attribute.id("task-counter"),
        attribute.class("task-counter"),
        hx.get("/boards/" <> board_id <> "/task-count"),
        hx.swap(hx.OuterHTML),
        hx.target(hx.This),
        hx.trigger([hx.custom("taskUpdated") |> hx.with_from(hx.Document)]),
      ],
      [element.text(int.to_string(count) <> " tasks")],
    )

  wisp.html_response(element.to_string(html), 200)
}
```

Notice that the counter element includes its own `hx-get` and `hx-trigger`
attributes in the response. When HTMX swaps in the new counter, the new
element is still listening for `taskUpdated` events. This is self-refreshing
HTML -- the element carries its own update logic.

### Hint for Exercise 5

Add the status select to `task_edit_form`:

```gleam
html.div([attribute.class("form-group")], [
  html.label([attribute.for("status-" <> task.id)], [
    element.text("Status"),
  ]),
  html.select(
    [
      attribute.name("status"),
      attribute.id("status-" <> task.id),
      attribute.class("form-control"),
    ],
    [
      html.option(
        [
          attribute.value("todo"),
          attribute.selected(task.status == "todo"),
        ],
        "To Do",
      ),
      html.option(
        [
          attribute.value("in-progress"),
          attribute.selected(task.status == "in-progress"),
        ],
        "In Progress",
      ),
      html.option(
        [
          attribute.value("done"),
          attribute.selected(task.status == "done"),
        ],
        "Done",
      ),
    ],
  ),
]),
```

Update the `UpdateTask` message to include a `status` field, and read it from
the form data in the `update` handler:

```gleam
let status = case list.key_find(formdata.values, "status") {
  Ok(value) -> value
  Error(Nil) -> "todo"
}
```

---

## 6. Key Takeaways

1. **The click-to-edit pattern has two HTML states: display and form.** The
   server renders both. HTMX swaps between them using `outerHTML`. No
   client-side state management is needed -- the DOM IS the state.

2. **Three endpoints power the entire interaction.** `GET /tasks/:id/edit`
   returns the form. `PUT /tasks/:id` saves and returns the display view.
   `GET /tasks/:id` returns the display view for cancel. The symmetry is
   elegant: edit and cancel both end with the server returning HTML that
   HTMX swaps into place.

3. **Method override bridges the gap between HTML forms and REST semantics.**
   HTML forms only support GET and POST. By including a hidden
   `_method=put` field and calling `wisp.method_override(req)` in the
   middleware, you get PUT semantics on the server while keeping form
   semantics on the client. The same pattern works for PATCH and DELETE.

4. **`hx.target(hx.This)` and `hx.target(hx.Selector("#id"))` serve
   different needs.** Use `hx.target(hx.This)` on the form itself (which
   has the same ID as the display div it replaced). Use
   `hx.target(hx.Selector("#task-" <> id))` on child elements (like the
   Edit button) that need to target their parent.

5. **Focus management requires `_hyperscript`, not just `autofocus`.** The
   HTML `autofocus` attribute is unreliable for content swapped in after page
   load. The `_hyperscript` `on load set focus to the first <input/> in me`
   handler works consistently because it hooks into the `htmx:load` event.

6. **Escape key handling reuses the Cancel button.** Instead of duplicating
   the cancel URL and swap configuration, the `_hyperscript`
   `on keyup[key === 'Escape'] trigger click on .cancel-btn in me` handler
   simulates a click on the existing Cancel button. One source of truth for
   the cancel behaviour.

7. **Validation errors stay inline.** When the server returns a 422, HTMX
   swaps the form-with-errors into the same spot where the form was. The
   user sees the error messages within the inline form, not in a separate
   location. Their input is preserved because the server populates the form
   fields with the submitted values.

8. **`HX-Trigger` response headers enable event coordination.** After a
   successful save, the server sends `HX-Trigger: taskUpdated`. Any element
   on the page listening for `taskUpdated` (via `hx-trigger`) can refresh
   itself. This decouples the inline edit from other page components -- the
   edit handler does not need to know about the counter, sidebar, or any
   other element that cares about task changes.

9. **The `.saving` class provides immediate feedback during the request
   lifecycle.** Adding a class on `htmx:beforeRequest` and removing it on
   `htmx:afterRequest` gives the user a visual indicator that their save is
   being processed. This prevents double-clicks and communicates
   responsiveness without any client-side state.

---

## What's Next

In [Chapter 22](22-modal-dialogs.md), we will build **modal dialogs** -- overlay panels that appear
on top of the current page for creating and editing resources. We will learn
how to load modal content from the server with HTMX, manage focus trapping
with `_hyperscript`, handle the browser's Escape key and backdrop clicks for
dismissal, and return data from the modal to the underlying page using
`HX-Trigger` events. Where inline editing works for quick single-field
changes, modals are the right tool for complex forms that need more visual
space and user attention.
