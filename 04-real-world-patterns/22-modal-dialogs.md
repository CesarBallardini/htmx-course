# Chapter 22 -- Modal Dialogs via HTMX

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapter 21 introduced inline editing with `hx-get` and `hx-put`,
swapping between read and edit views inside individual task rows.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Use the native `<dialog>` element with `showModal()` and `close()`, and explain why it is preferred over custom overlay implementations.
2. Load modal content from the server using `hx-get` targeting a `#modal` container with an `InnerHTML` swap.
3. Open and close modals using `_hyperscript` responding to custom events triggered by the `HX-Trigger-After-Swap` response header.
4. Implement focus trapping and Escape-to-close behaviour automatically via the `<dialog>` element's built-in capabilities.
5. Submit forms inside modals where a 422 response re-renders the form in the modal and a 201 response closes the modal and updates the page.
6. Return focus to the element that triggered the modal when the modal closes.

---

## 1. Theory

### 1.1 Modals in Hypermedia Applications

In a JavaScript SPA, a modal dialog is a piece of client-side state. You have a
variable somewhere -- `isModalOpen: true` -- and a component that renders based
on that flag. The modal's content might be fetched from an API, or it might be
baked into the JavaScript bundle, or it might be computed from the application's
state tree.

In a hypermedia application, there is no client-side state tree. The server is
the source of truth. This changes how modals work:

- **The modal content comes from the server.** When the user clicks "New Task",
  the browser sends a GET request, and the server returns the form HTML. The
  browser swaps that HTML into a container and opens the dialog.
- **The modal shell is part of the layout.** An empty `<dialog>` element sits in
  the page at all times. It is invisible until something tells it to open.
  Content is loaded into it on demand.
- **Opening and closing are side effects of HTTP responses.** The server decides
  when the modal opens (by sending an `HX-Trigger-After-Swap` header) and when
  it closes (by sending an `HX-Trigger` header).

This approach has several advantages over the SPA pattern:

| Aspect                | SPA Modal                           | Hypermedia Modal                      |
|-----------------------|-------------------------------------|---------------------------------------|
| **Content source**    | Client bundle or API call           | Server HTML fragment                  |
| **State management**  | Client state variable               | None -- server decides                |
| **Form validation**   | Client-side + API call              | Server-side, re-rendered in modal     |
| **Code duplication**  | Often duplicates server validation  | Single source of truth                |
| **Bundle size**       | Modal component shipped to client   | Zero -- content loaded on demand      |
| **Deep linking**      | Requires URL sync logic             | Works if server supports GET          |

The hypermedia modal is simpler because there is less to coordinate. The server
returns HTML, HTMX swaps it in, a tiny bit of client-side scripting opens the
dialog, and the browser's native `<dialog>` handles the rest.

### 1.2 The HTML `<dialog>` Element

The `<dialog>` element is a native HTML element designed specifically for modal
and non-modal dialogs. It has been supported in all major browsers since 2022
(Chrome 37+, Firefox 98+, Safari 15.4+, Edge 79+). There is no reason to use a
custom overlay in new projects.

Here is what `<dialog>` gives you for free:

**Top layer rendering.** When you call `showModal()`, the browser moves the
dialog to a special rendering layer that sits above everything else in the page.
No `z-index` battles. No worrying about stacking contexts. The dialog is always
on top, period.

**A built-in backdrop.** The `::backdrop` pseudo-element creates a dimming
overlay behind the dialog. You style it with CSS:

```css
dialog::backdrop {
  background-color: rgba(0, 0, 0, 0.5);
}
```

No extra `<div class="overlay">` needed.

**Automatic focus trapping.** When a modal dialog is open, Tab and Shift+Tab
cycle only through focusable elements inside the dialog. The user cannot Tab out
to the page behind it. This is the number one accessibility requirement that
custom overlay implementations get wrong.

**Escape key closes the dialog.** Pressing Escape fires a `cancel` event and
closes the dialog. You do not need a keyboard event listener.

**The `close` event.** When the dialog closes (by Escape, by calling `.close()`,
or by a form with `method="dialog"`), it fires a `close` event. You can listen
for this event to clean up or return focus.

**The `open` attribute.** The dialog has an `open` boolean attribute that
reflects its state. You can use CSS `dialog[open]` selectors for styling.

Here is the minimal HTML:

```html
<dialog id="modal">
  <p>Hello from the modal!</p>
  <button onclick="this.closest('dialog').close()">Close</button>
</dialog>

<button onclick="document.getElementById('modal').showModal()">Open</button>
```

That is a fully functional, accessible modal dialog in six lines. No library.
No framework. No JavaScript module. Just the platform.

> **`show()` vs `showModal()`:** The `<dialog>` element has two methods.
> `show()` opens the dialog as a non-modal element -- it appears on the page but
> does not trap focus, does not show a backdrop, and does not block interaction
> with the rest of the page. `showModal()` opens it as a true modal with all the
> features listed above. For modal dialogs, always use `showModal()`.

### 1.3 Architecture: Shell vs Content

The architecture we will build separates the dialog into two parts:

```
Layout (always present)
├── <header> ... </header>
├── <main> ... </main>
├── <dialog id="modal">          <── Shell: empty container
│     ← content loaded here via HTMX
│   </dialog>
```

The **shell** is the `<dialog>` element itself. It lives in the page layout and
is present on every page. It has no visible content. It carries the `_hyperscript`
attributes that control opening and closing.

The **content** is loaded on demand. When the user clicks "New Task", HTMX
fetches the form HTML from the server and swaps it into the dialog's innerHTML.
When the user clicks "Delete Task", HTMX fetches a confirmation message. Each
modal interaction loads different content into the same shell.

Here is the flow:

```
┌─────────────────────────────────────────────────────────────┐
│  Page Layout                                                │
│                                                             │
│  ┌───────────────────────────────────┐                      │
│  │  Main Content                     │                      │
│  │                                   │                      │
│  │  [+ New Task]  ─── hx-get ──►    │                      │
│  │                     /tasks/new    │                      │
│  │                                   │                      │
│  └───────────────────────────────────┘                      │
│                                                             │
│  ┌───────────────────────────────────┐                      │
│  │  <dialog id="modal">             │  ◄── Shell            │
│  │     (empty until content loaded)  │                      │
│  │  </dialog>                        │                      │
│  └───────────────────────────────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘

                    │
                    │  Server responds with:
                    │  1. Form HTML (body)
                    │  2. HX-Trigger-After-Swap: openModal (header)
                    │
                    ▼

┌─────────────────────────────────────────────────────────────┐
│  Page Layout                                                │
│                                                             │
│  ┌───────────────────────────────────┐                      │
│  │  <dialog id="modal" open>        │  ◄── Shell (open)     │
│  │     ┌───────────────────────────┐ │                      │
│  │     │  New Task Form            │ │  ◄── Content          │
│  │     │  Title: [____________]    │ │      (from server)    │
│  │     │  [Cancel]  [Create Task]  │ │                      │
│  │     └───────────────────────────┘ │                      │
│  │  </dialog>                        │                      │
│  └───────────────────────────────────┘                      │
│                                                             │
│  ::backdrop (dimmed)                                        │
└─────────────────────────────────────────────────────────────┘
```

This separation means:

1. **The shell is defined once** in the layout. Every page gets it automatically.
2. **The content is reusable.** The same form fragment can be loaded in the
   modal or used in a full-page fallback for non-JavaScript browsers.
3. **The server controls everything.** It decides what content appears and
   when the dialog opens.

### 1.4 The Open/Close Lifecycle

The lifecycle has four phases. Understanding them is essential for building
reliable modals.

**Phase 1: Fetch.** The user clicks a trigger element (a button, a link). HTMX
sends a GET request to the server. The target is `#modal` and the swap strategy
is `InnerHTML`.

**Phase 2: Swap.** The server responds with an HTML fragment (just the modal
content -- no `<dialog>` wrapper, no layout). The response includes an
`HX-Trigger-After-Swap: openModal` header. HTMX swaps the content into the
dialog.

**Phase 3: Open.** After the swap completes, HTMX dispatches the `openModal`
custom event (triggered by the `HX-Trigger-After-Swap` header). A `_hyperscript`
listener on the dialog hears this event and calls `showModal()`.

**Phase 4: Close.** Closing can happen in several ways:
- The user presses Escape (built-in `<dialog>` behaviour).
- The user clicks a Cancel button that calls `close()`.
- A form submission succeeds and the server responds with an `HX-Trigger:
  closeModal` header.
- The `_hyperscript` listener on the dialog hears the `closeModal` event and
  calls `close()`.

Here is the timeline:

```
User clicks "New Task"
  │
  ├─ HTMX: GET /tasks/new → target #modal, swap InnerHTML
  │
  ├─ Server: returns form HTML
  │          + HX-Trigger-After-Swap: openModal
  │
  ├─ HTMX: swaps form into <dialog id="modal">
  │
  ├─ HTMX: dispatches "openModal" event (from HX-Trigger-After-Swap)
  │
  ├─ _hyperscript: "on openModal" fires → calls dialog.showModal()
  │
  ├─ Browser: dialog opens, focus traps, backdrop appears
  │
  │  ... user fills form, clicks "Create Task" ...
  │
  ├─ HTMX: POST /tasks → target #modal, swap InnerHTML
  │
  ├─ Server (validation fails): returns form with errors, status 422
  │  ├─ + HX-Trigger-After-Swap: openModal
  │  └─ HTMX: swaps error form into dialog (dialog stays open)
  │
  │  ... user fixes errors, submits again ...
  │
  ├─ HTMX: POST /tasks → target #modal, swap InnerHTML
  │
  ├─ Server (success): returns empty body, status 201
  │  └─ + HX-Trigger: closeModal, taskCreated
  │
  ├─ HTMX: dispatches "closeModal" and "taskCreated" events
  │
  ├─ _hyperscript: "on closeModal" fires → calls dialog.close()
  │
  ├─ Browser: dialog closes, focus returns
  │
  └─ Task list: "on taskCreated" fires → hx-get refreshes the list
```

Two things to notice:

First, the `HX-Trigger-After-Swap` header triggers the event *after* the
content is swapped into the DOM. This is critical. If you used `HX-Trigger`
instead, the event would fire *before* the swap, and `showModal()` would open
an empty dialog. The "After-Swap" timing ensures the content is visible when the
dialog opens.

Second, on success the server uses `HX-Trigger` (not `HX-Trigger-After-Swap`)
for the `closeModal` event. This is correct because we do not need to wait for
a swap -- we want to close immediately. The `taskCreated` event rides along in
the same header, and other elements on the page can listen for it.

### 1.5 Forms Inside Modals

Forms inside modals follow the same validation pattern from Chapter 8, with one
twist: the form lives inside the dialog, and the server response must stay
inside the dialog.

The key is the target and swap configuration on the form:

```
<form hx-post="/tasks"
      hx-target="#modal"
      hx-swap="innerHTML">
```

The form targets the dialog itself (`#modal`) with an `InnerHTML` swap. This
means:

- **On validation failure (422):** The server returns the form with error
  messages. HTMX swaps it into the dialog. The dialog stays open. The user sees
  the errors.
- **On success (201):** The server returns a minimal (or empty) response. HTMX
  swaps it into the dialog (clearing the form). The `HX-Trigger: closeModal`
  header fires the close event.

Here is the decision tree on the server:

```
POST /tasks
  │
  ├─ Parse form data
  │
  ├─ Validate
  │   │
  │   ├─ FAIL: Return form HTML with errors
  │   │        Status: 422
  │   │        Header: HX-Trigger-After-Swap: openModal
  │   │        (keeps dialog open, re-renders form)
  │   │
  │   └─ OK: Create task in database
  │          Return empty or confirmation HTML
  │          Status: 201
  │          Header: HX-Trigger: closeModal, taskCreated
  │          (closes dialog, refreshes task list)
```

Why does the 422 response include `HX-Trigger-After-Swap: openModal`? Because
the dialog is already open, calling `showModal()` again on an open dialog is
a no-op. It does not hurt. But including it makes the response self-contained --
the same endpoint works whether the form is loaded for the first time or
re-rendered after a validation error. Consistency is worth more than a
micro-optimization.

### 1.6 Accessibility

The `<dialog>` element handles the hard accessibility problems for you:

**Focus trapping:** Automatic. The browser restricts Tab navigation to elements
inside the dialog.

**Escape to close:** Automatic. The browser fires a `cancel` event and closes
the dialog.

**Role and aria:** The `<dialog>` element has an implicit `role="dialog"`. You
do not need to add `role="dialog"` manually.

**`aria-label` or `aria-labelledby`:** This is the one thing you DO need to add.
The dialog needs a label that screen readers announce when the dialog opens.
Either use `aria-label` for a short description:

```html
<dialog id="modal" aria-label="Create new task">
```

Or use `aria-labelledby` to point to a heading inside the dialog:

```html
<dialog id="modal" aria-labelledby="modal-title">
  <h2 id="modal-title">Create New Task</h2>
  ...
</dialog>
```

Since our dialog content is loaded dynamically, we will use `aria-labelledby`
and ensure every modal content fragment includes an element with
`id="modal-title"`.

**Return focus to the trigger element.** When a modal closes, focus should
return to the element that opened it. The browser does this automatically in most
cases when you use `showModal()` and `close()`. The element that was focused
before `showModal()` was called receives focus again when the dialog closes.

However, there is a nuance with HTMX. The user clicks a button, HTMX sends a
request, and the swap happens. If the swap replaces the button, the browser
loses track of the original focus target. To handle this edge case, we will
store a reference to the trigger element before the request and restore focus
after close.

The `_hyperscript` approach:

```html
<dialog id="modal"
  _="on openModal
       set my @data-opener to (the closest <button/> to the target of the event).id
       call me.showModal()
     on closeModal
       call me.close()
     on close
       set opener to #{my @data-opener}
       if opener exists call opener.focus()">
```

This stores the triggering button's ID in a data attribute, then uses it to
return focus after close.

For simpler cases where the triggering button is never replaced by a swap, the
browser handles focus return automatically and you do not need this extra logic.

---

## 2. Code Walkthrough

We are going to add modal dialogs to the Teamwork task board. By the end, the
"New Task" button will open a modal with a form, the form will validate inside
the modal, and successful creation will close the modal and refresh the task
list. We will also add a confirmation modal for deleting tasks.

### Step 1 -- Add the Modal Shell to the Layout

The dialog shell goes in the layout so every page has it. It is an empty
`<dialog>` element with `_hyperscript` listeners.

First, let us look at the `_hyperscript` we need:

```
on openModal call me.showModal()
on closeModal call me.close()
```

That is the minimal version. The dialog listens for two custom events:
`openModal` opens it, `closeModal` closes it. Let us add focus return:

```
on openModal
  set my @data-opener to (the closest <button/> to the target of the event).id
  call me.showModal()
on closeModal call me.close()
on close
  set openerEl to the first <#{my @data-opener} /> in the document
  if openerEl exists call openerEl.focus()
```

When `openModal` fires, we find the button that triggered the event (the
closest `<button>` ancestor of the event target) and store its ID. When the
dialog closes (the browser fires the `close` event after `close()` is called
or the user presses Escape), we look up that button by ID and return focus to it.

Here is the layout function in Gleam:

```gleam
fn layout(content: element.Element(Nil)) -> wisp.Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork"),
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute.attribute(
            "content",
            "width=device-width, initial-scale=1",
          ),
        ]),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
        html.script(
          [attribute.src("/static/js/htmx.min.js")],
          "",
        ),
        html.script(
          [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
          "",
        ),
      ]),
      html.body([hx.boost(True)], [
        html.div([attribute.class("container")], [
          html.h1([], [element.text("Teamwork Task Board")]),
          content,
        ]),
        // Modal shell -- always present, always empty until loaded
        modal_shell(),
      ]),
    ])

  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

And the `modal_shell` function:

```gleam
fn modal_shell() -> element.Element(Nil) {
  element.element(
    "dialog",
    [
      attribute.id("modal"),
      attribute.class("modal-dialog"),
      attribute.attribute("aria-labelledby", "modal-title"),
      attribute.attribute(
        "_",
        "on openModal
           set my @data-opener to (the closest <button/> to the target of the event).id
           call me.showModal()
         on closeModal call me.close()
         on close
           set openerEl to the first <#{my @data-opener} /> in the document
           if openerEl exists call openerEl.focus()",
      ),
    ],
    [],
  )
}
```

A few things to notice:

**`element.element("dialog", attrs, children)`.** We use the generic
`element.element` function to create the `<dialog>` tag. Lustre provides
`html.dialog(attrs, children)` as well, but `element.element` works as a
reliable fallback if your version of the `lustre` library does not include
`html.dialog`.

**`aria-labelledby="modal-title"`.** This tells screen readers to announce the
element with `id="modal-title"` as the dialog's label. Every modal content
fragment we create will include an `<h2 id="modal-title">` heading.

**The `_hyperscript` is on the dialog, not on individual buttons.** This is
important. The dialog listens for events that bubble up from anywhere in the
page. The buttons do not need to know anything about the dialog -- they just
trigger HTMX requests that target `#modal`.

**The dialog starts empty.** No children. Content is loaded on demand.

### Step 2 -- The "New Task" Button

The button that opens the modal needs three things:

1. An `hx-get` that fetches the form.
2. An `hx-target` pointing to `#modal`.
3. An `hx-swap` of `InnerHTML`.

```gleam
fn new_task_button(board_id: String) -> element.Element(Nil) {
  html.button(
    [
      attribute.id("new-task-btn"),
      attribute.class("btn btn-primary"),
      hx.get("/boards/" <> board_id <> "/tasks/new"),
      hx.target(hx.Selector("#modal")),
      hx.swap(hx.InnerHTML),
    ],
    [element.text("+ New Task")],
  )
}
```

That is the entire client-side setup for opening a modal. No `onclick` handler.
No JavaScript. HTMX fetches the content, swaps it in, and the server's response
header tells the dialog to open.

The button has an `id` of `"new-task-btn"`. This is important for focus return
-- when the dialog closes, the `_hyperscript` on the dialog will look up this
ID and return focus to it.

### Step 3 -- Server Handler Returning Modal Content

The server needs a handler for `GET /boards/:board_id/tasks/new` that returns
the form HTML as a fragment (no layout wrapper) and includes the
`HX-Trigger-After-Swap: openModal` header.

```gleam
fn new_task_form_handler(
  _req: wisp.Request,
  board_id: String,
) -> wisp.Response {
  let html = new_task_modal_content(board_id)

  wisp.html_response(element.to_string(html), 200)
  |> wisp.set_header("HX-Trigger-After-Swap", "openModal")
}
```

The response body is a fragment -- just the form, no `<html>`, no `<body>`, no
layout. HTMX will swap this fragment into the dialog's innerHTML.

The `HX-Trigger-After-Swap: openModal` header tells HTMX to dispatch an
`openModal` event after the swap completes. The `_hyperscript` listener on the
dialog responds by calling `showModal()`.

Now the modal content function:

```gleam
fn new_task_modal_content(board_id: String) -> element.Element(Nil) {
  html.div([attribute.class("modal-content")], [
    html.div([attribute.class("modal-header")], [
      html.h2(
        [attribute.id("modal-title")],
        [element.text("New Task")],
      ),
      close_button(),
    ]),
    task_form_for_modal(board_id, "", []),
  ])
}
```

Every modal content fragment follows the same structure:

1. A `modal-content` wrapper div.
2. A `modal-header` with an `<h2 id="modal-title">` and a close button.
3. The body content (a form, a confirmation message, etc.).

The `close_button` is a simple button that closes the dialog:

```gleam
fn close_button() -> element.Element(Nil) {
  html.button(
    [
      attribute.class("btn-close"),
      attribute.attribute("aria-label", "Close dialog"),
      attribute.attribute(
        "_",
        "on click call the closest <dialog/> to me .close()",
      ),
    ],
    [element.text("X")],
  )
}
```

The `_hyperscript` here finds the closest `<dialog>` ancestor and calls
`close()` on it. This is more resilient than hard-coding `#modal` -- if you
ever have nested dialogs, each close button closes its own parent.

### Step 4 -- The Form Inside the Modal

The form needs to POST to the task creation endpoint, target the dialog, and
swap its content:

```gleam
fn task_form_for_modal(
  board_id: String,
  title_value: String,
  errors: List(String),
) -> element.Element(Nil) {
  html.form(
    [
      attribute.id("task-form"),
      attribute.class("modal-form"),
      hx.post("/boards/" <> board_id <> "/tasks"),
      hx.target(hx.Selector("#modal")),
      hx.swap(hx.InnerHTML),
    ],
    [
      error_list(errors),
      html.div([attribute.class("form-group")], [
        html.label(
          [attribute.for("title")],
          [element.text("Task title")],
        ),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.value(title_value),
          attribute.required(True),
          attribute.autofocus(True),
          attribute.placeholder("What needs to be done?"),
        ]),
      ]),
      html.div([attribute.class("modal-actions")], [
        html.button(
          [
            attribute.type_("button"),
            attribute.class("btn btn-secondary"),
            attribute.attribute(
              "_",
              "on click call the closest <dialog/> to me .close()",
            ),
          ],
          [element.text("Cancel")],
        ),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn btn-primary"),
          ],
          [element.text("Create Task")],
        ),
      ]),
    ],
  )
}
```

Let us break this down:

**`hx.post("/boards/" <> board_id <> "/tasks")`.** The form submits to the same
endpoint used for creating tasks. The same handler processes the request.

**`hx.target(hx.Selector("#modal"))`.** The response replaces the dialog's
content. For both success and failure, the response goes into the dialog.

**`hx.swap(hx.InnerHTML)`.** We replace the innerHTML of the dialog, not the
dialog element itself (which would be `OuterHTML`). This keeps the dialog's
`_hyperscript` listeners intact.

**`attribute.value(title_value)`.** When the form is re-rendered after a
validation error, the user's input is preserved. On first render,
`title_value` is `""`.

**`attribute.autofocus(True)`.** The title input gets focus when the form
appears in the dialog. This works because `<dialog>` respects `autofocus` on
its descendants when `showModal()` is called.

**The Cancel button.** It has `type_("button")` (not `"submit"`) so clicking it
does not submit the form. Its `_hyperscript` closes the dialog.

The `error_list` helper renders validation errors:

```gleam
fn error_list(errors: List(String)) -> element.Element(Nil) {
  case errors {
    [] -> element.none()
    _ ->
      html.div(
        [
          attribute.class("error-list"),
          attribute.attribute("role", "alert"),
        ],
        list.map(errors, fn(error) {
          html.p([attribute.class("error-message")], [
            element.text(error),
          ])
        }),
      )
  }
}
```

When there are no errors, `element.none()` renders nothing. When there are
errors, they appear in a div with `role="alert"` so screen readers announce them
immediately.

### Step 5 -- Form Submission: Success and Failure

The form POST handler is the heart of the modal workflow. It must handle two
outcomes:

**Validation failure (422):** Re-render the form with errors, keeping the
dialog open.

**Success (201):** Create the task, close the dialog, and trigger a refresh of
the task list.

```gleam
fn create_task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  // Validate
  case validate_title(title) {
    Error(errors) -> {
      // 422: Re-render form with errors inside the modal
      let html = new_task_modal_content_with_errors(board_id, title, errors)

      wisp.html_response(element.to_string(html), 422)
      |> wisp.set_header("HX-Trigger-After-Swap", "openModal")
    }

    Ok(valid_title) -> {
      // Create the task
      let task_id = wisp.random_string(16)
      let task =
        db.Task(
          id: task_id,
          title: valid_title,
          description: "",
          done: False,
          due_date: option.None,
        )
      let assert Ok(_) = db.create_task(ctx.db, board_id, task)

      // 201: Close modal and trigger task list refresh
      wisp.html_response("", 201)
      |> wisp.set_header("HX-Trigger", "closeModal, taskCreated")
    }
  }
}
```

Let us trace through both paths.

**Failure path:**

1. `validate_title` returns `Error(["Title must be at least 3 characters"])`.
2. The server builds the form HTML with the user's input preserved and the
   error messages displayed.
3. Status 422 tells the client the request was understood but invalid.
4. `HX-Trigger-After-Swap: openModal` fires after the swap. Since the dialog
   is already open, `showModal()` is a no-op. But the swap replaces the form
   content with the error-annotated version.
5. The user sees their input plus the error messages. They fix the input and
   submit again.

**Success path:**

1. `validate_title` returns `Ok("Clean up the backlog")`.
2. The task is created in the database.
3. The response body is empty (or could be a success message -- the dialog will
   close immediately either way).
4. Status 201 signals successful creation.
5. `HX-Trigger: closeModal, taskCreated` fires TWO events:
   - `closeModal` -- the dialog's `_hyperscript` listener calls `close()`.
   - `taskCreated` -- the task list's trigger listener refetches the list.

The helper function for rendering the form with errors:

```gleam
fn new_task_modal_content_with_errors(
  board_id: String,
  title_value: String,
  errors: List(String),
) -> element.Element(Nil) {
  html.div([attribute.class("modal-content")], [
    html.div([attribute.class("modal-header")], [
      html.h2(
        [attribute.id("modal-title")],
        [element.text("New Task")],
      ),
      close_button(),
    ]),
    task_form_for_modal(board_id, title_value, errors),
  ])
}
```

This is the same structure as `new_task_modal_content`, but it passes the
user's title and the error list to the form. In practice, you would likely
collapse these two functions into one. We keep them separate here for clarity.

The validation function:

```gleam
fn validate_title(title: String) -> Result(String, List(String)) {
  let trimmed = string.trim(title)
  let errors = []

  let errors = case string.length(trimmed) {
    0 -> ["Title is required." , ..errors]
    n if n < 3 -> ["Title must be at least 3 characters." , ..errors]
    n if n > 200 -> ["Title must be 200 characters or fewer." , ..errors]
    _ -> errors
  }

  case errors {
    [] -> Ok(trimmed)
    _ -> Error(errors)
  }
}
```

### Step 6 -- The Task List Listening for `taskCreated`

The task list needs to refresh itself when a task is created. It listens for
the `taskCreated` event using an HTMX trigger:

```gleam
fn task_list(board_id: String, tasks: List(db.Task)) -> element.Element(Nil) {
  let today = date_helpers.today()

  html.div(
    [
      attribute.id("task-list"),
      hx.get("/boards/" <> board_id <> "/tasks"),
      hx.swap(hx.InnerHTML),
      hx.trigger([hx.custom("taskCreated") |> hx.with_from(hx.Document)]),
    ],
    list.map(tasks, fn(task) { task_item(task, today) }),
  )
}
```

The `hx.trigger` is the key part. It sets up:

```
hx-trigger="taskCreated from:body"
```

When the `POST /tasks` handler returns `HX-Trigger: closeModal, taskCreated`,
HTMX dispatches the `taskCreated` event on the body. The task list hears it
and fires a GET request to reload itself. The modal closes and the task list
updates -- all from a single server response.

This is the same event coordination pattern from Chapter 16, now applied to
modal workflows.

### Step 7 -- Confirmation Modal for Delete

Delete actions deserve a confirmation step. Instead of using the browser's
`window.confirm()` (which is ugly and not styleable), we will load a
confirmation dialog from the server.

The delete button on each task:

```gleam
fn delete_button(task: db.Task) -> element.Element(Nil) {
  html.button(
    [
      attribute.id("delete-btn-" <> task.id),
      attribute.class("btn btn-danger btn-sm"),
      hx.get("/tasks/" <> task.id <> "/confirm-delete"),
      hx.target(hx.Selector("#modal")),
      hx.swap(hx.InnerHTML),
    ],
    [element.text("Delete")],
  )
}
```

The server handler for the confirmation:

```gleam
fn confirm_delete_handler(
  _req: wisp.Request,
  task_id: String,
  ctx: Context,
) -> wisp.Response {
  // Look up the task to show its title in the confirmation
  let assert Ok(task) = db.get_task(ctx.db, task_id)

  let html = confirm_delete_modal_content(task)

  wisp.html_response(element.to_string(html), 200)
  |> wisp.set_header("HX-Trigger-After-Swap", "openModal")
}
```

The confirmation content:

```gleam
fn confirm_delete_modal_content(task: db.Task) -> element.Element(Nil) {
  html.div([attribute.class("modal-content")], [
    html.div([attribute.class("modal-header")], [
      html.h2(
        [attribute.id("modal-title")],
        [element.text("Delete Task")],
      ),
      close_button(),
    ]),
    html.div([attribute.class("modal-body")], [
      html.p([], [
        element.text("Are you sure you want to delete "),
        html.strong([], [element.text(task.title)]),
        element.text("? This action cannot be undone."),
      ]),
    ]),
    html.div([attribute.class("modal-actions")], [
      html.button(
        [
          attribute.type_("button"),
          attribute.class("btn btn-secondary"),
          attribute.attribute(
            "_",
            "on click call the closest <dialog/> to me .close()",
          ),
        ],
        [element.text("Cancel")],
      ),
      html.button(
        [
          attribute.class("btn btn-danger"),
          hx.delete("/tasks/" <> task.id),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
        ],
        [element.text("Delete")],
      ),
    ]),
  ])
}
```

The "Delete" button inside the confirmation modal does something different from
the task form. Instead of targeting `#modal`, it targets the task element in the
list (`#task-<id>`) and swaps it with `OuterHTML`. This is because a successful
delete should remove the task from the list, not update the modal content.

The delete handler:

```gleam
fn delete_task(
  _req: wisp.Request,
  task_id: String,
  ctx: Context,
) -> wisp.Response {
  let assert Ok(_) = db.delete_task(ctx.db, task_id)

  // Return empty content to remove the task element
  wisp.html_response("", 200)
  |> wisp.set_header("HX-Trigger", "closeModal, taskDeleted")
}
```

The response body is empty because the `OuterHTML` swap with an empty body
removes the target element from the DOM. The `closeModal` event closes the
dialog. The `taskDeleted` event can be used by counters or other elements that
need to refresh.

Notice the two-step flow:

1. **First click (delete button on task):** Opens confirmation modal.
2. **Second click (confirm button in modal):** Executes the delete, closes
   modal, removes task from list.

This is safer than a single-click delete with `hx.confirm()`. The confirmation
modal can show the task title, warn about consequences, and give the user a
clear Cancel button.

### Step 8 -- Wire It into the Router

Update the router to handle the new modal endpoints:

```gleam
pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)

    ["boards", board_id] ->
      board_page(req, ctx, board_id)

    ["boards", board_id, "tasks"] ->
      case req.method {
        http.Get -> list_tasks(req, ctx, board_id)
        http.Post -> create_task(req, ctx, board_id)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }

    ["boards", board_id, "tasks", "new"] -> {
      use <- wisp.require_method(req, http.Get)
      new_task_form_handler(req, board_id)
    }

    ["tasks", task_id] ->
      case req.method {
        http.Delete -> delete_task(req, task_id, ctx)
        _ -> wisp.method_not_allowed([http.Delete])
      }

    ["tasks", task_id, "confirm-delete"] -> {
      use <- wisp.require_method(req, http.Get)
      confirm_delete_handler(req, task_id, ctx)
    }

    _ -> wisp.not_found()
  }
}
```

Two new routes:

| Route                                  | Method | Purpose                         |
|----------------------------------------|--------|---------------------------------|
| `/boards/:board_id/tasks/new`          | GET    | Returns the new task form       |
| `/tasks/:task_id/confirm-delete`       | GET    | Returns the delete confirmation |

Both return HTML fragments (no layout) with `HX-Trigger-After-Swap: openModal`.

---

## 3. Full Code Listing

Here are the complete files with modal support integrated.

### `src/teamwork/web.gleam`

```gleam
import gleam/http
import gleam/list
import gleam/option
import gleam/result
import gleam/string
import hx
import lustre/attribute
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}
import teamwork/context.{type Context}
import teamwork/date_helpers
import teamwork/db

// ── Layout ──────────────────────────────────────────────────────────────

fn layout(content: element.Element(Nil)) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork"),
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute.attribute(
            "content",
            "width=device-width, initial-scale=1",
          ),
        ]),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
        html.script(
          [attribute.src("/static/js/htmx.min.js")],
          "",
        ),
        html.script(
          [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
          "",
        ),
      ]),
      html.body([hx.boost(True)], [
        html.div([attribute.class("container")], [
          html.h1([], [element.text("Teamwork Task Board")]),
          content,
        ]),
        // Modal shell
        modal_shell(),
      ]),
    ])

  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

// ── Modal Shell ─────────────────────────────────────────────────────────

fn modal_shell() -> element.Element(Nil) {
  element.element(
    "dialog",
    [
      attribute.id("modal"),
      attribute.class("modal-dialog"),
      attribute.attribute("aria-labelledby", "modal-title"),
      attribute.attribute(
        "_",
        "on openModal
           set my @data-opener to (the closest <button/> to the target of the event).id
           call me.showModal()
         on closeModal call me.close()
         on close
           set openerEl to the first <#{my @data-opener} /> in the document
           if openerEl exists call openerEl.focus()",
      ),
    ],
    [],
  )
}

// ── Close Button ────────────────────────────────────────────────────────

fn close_button() -> element.Element(Nil) {
  html.button(
    [
      attribute.class("btn-close"),
      attribute.attribute("aria-label", "Close dialog"),
      attribute.attribute(
        "_",
        "on click call the closest <dialog/> to me .close()",
      ),
    ],
    [element.text("X")],
  )
}

// ── Error List ──────────────────────────────────────────────────────────

fn error_list(errors: List(String)) -> element.Element(Nil) {
  case errors {
    [] -> element.none()
    _ ->
      html.div(
        [
          attribute.class("error-list"),
          attribute.attribute("role", "alert"),
        ],
        list.map(errors, fn(error) {
          html.p([attribute.class("error-message")], [
            element.text(error),
          ])
        }),
      )
  }
}

// ── New Task Button ─────────────────────────────────────────────────────

fn new_task_button(board_id: String) -> element.Element(Nil) {
  html.button(
    [
      attribute.id("new-task-btn"),
      attribute.class("btn btn-primary"),
      hx.get("/boards/" <> board_id <> "/tasks/new"),
      hx.target(hx.Selector("#modal")),
      hx.swap(hx.InnerHTML),
    ],
    [element.text("+ New Task")],
  )
}

// ── Task Form (for modal) ───────────────────────────────────────────────

fn task_form_for_modal(
  board_id: String,
  title_value: String,
  errors: List(String),
) -> element.Element(Nil) {
  html.form(
    [
      attribute.id("task-form"),
      attribute.class("modal-form"),
      hx.post("/boards/" <> board_id <> "/tasks"),
      hx.target(hx.Selector("#modal")),
      hx.swap(hx.InnerHTML),
    ],
    [
      error_list(errors),
      html.div([attribute.class("form-group")], [
        html.label(
          [attribute.for("title")],
          [element.text("Task title")],
        ),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.value(title_value),
          attribute.required(True),
          attribute.autofocus(True),
          attribute.placeholder("What needs to be done?"),
        ]),
      ]),
      html.div([attribute.class("modal-actions")], [
        html.button(
          [
            attribute.type_("button"),
            attribute.class("btn btn-secondary"),
            attribute.attribute(
              "_",
              "on click call the closest <dialog/> to me .close()",
            ),
          ],
          [element.text("Cancel")],
        ),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn btn-primary"),
          ],
          [element.text("Create Task")],
        ),
      ]),
    ],
  )
}

// ── Modal Content: New Task ─────────────────────────────────────────────

fn new_task_modal_content(
  board_id: String,
  title_value: String,
  errors: List(String),
) -> element.Element(Nil) {
  html.div([attribute.class("modal-content")], [
    html.div([attribute.class("modal-header")], [
      html.h2(
        [attribute.id("modal-title")],
        [element.text("New Task")],
      ),
      close_button(),
    ]),
    task_form_for_modal(board_id, title_value, errors),
  ])
}

// ── Modal Content: Confirm Delete ───────────────────────────────────────

fn confirm_delete_modal_content(task: db.Task) -> element.Element(Nil) {
  html.div([attribute.class("modal-content")], [
    html.div([attribute.class("modal-header")], [
      html.h2(
        [attribute.id("modal-title")],
        [element.text("Delete Task")],
      ),
      close_button(),
    ]),
    html.div([attribute.class("modal-body")], [
      html.p([], [
        element.text("Are you sure you want to delete "),
        html.strong([], [element.text(task.title)]),
        element.text("? This action cannot be undone."),
      ]),
    ]),
    html.div([attribute.class("modal-actions")], [
      html.button(
        [
          attribute.type_("button"),
          attribute.class("btn btn-secondary"),
          attribute.attribute(
            "_",
            "on click call the closest <dialog/> to me .close()",
          ),
        ],
        [element.text("Cancel")],
      ),
      html.button(
        [
          attribute.class("btn btn-danger"),
          hx.delete("/tasks/" <> task.id),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
        ],
        [element.text("Delete")],
      ),
    ]),
  ])
}

// ── Delete Button (on each task) ────────────────────────────────────────

fn delete_button(task: db.Task) -> element.Element(Nil) {
  html.button(
    [
      attribute.id("delete-btn-" <> task.id),
      attribute.class("btn btn-danger btn-sm"),
      hx.get("/tasks/" <> task.id <> "/confirm-delete"),
      hx.target(hx.Selector("#modal")),
      hx.swap(hx.InnerHTML),
    ],
    [element.text("Delete")],
  )
}

// ── Task Item ───────────────────────────────────────────────────────────

fn task_item(task: db.Task, today: String) -> element.Element(Nil) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item"),
    ],
    [
      html.span(
        [attribute.class("task-title")],
        [element.text(task.title)],
      ),
      delete_button(task),
    ],
  )
}

// ── Task List ───────────────────────────────────────────────────────────

fn task_list(
  board_id: String,
  tasks: List(db.Task),
) -> element.Element(Nil) {
  let today = date_helpers.today()

  html.div(
    [
      attribute.id("task-list"),
      hx.get("/boards/" <> board_id <> "/tasks"),
      hx.swap(hx.InnerHTML),
      hx.trigger([hx.custom("taskCreated") |> hx.with_from(hx.Document)]),
    ],
    list.map(tasks, fn(task) { task_item(task, today) }),
  )
}

// ── Board Page ──────────────────────────────────────────────────────────

fn board_page(
  _req: Request,
  ctx: Context,
  board_id: String,
) -> Response {
  let assert Ok(tasks) = db.list_tasks(ctx.db, board_id)

  let content =
    html.div([], [
      new_task_button(board_id),
      task_list(board_id, tasks),
    ])

  layout(content)
}

// ── Validation ──────────────────────────────────────────────────────────

fn validate_title(title: String) -> Result(String, List(String)) {
  let trimmed = string.trim(title)
  let errors = []

  let errors = case string.length(trimmed) {
    0 -> ["Title is required.", ..errors]
    n if n < 3 -> ["Title must be at least 3 characters.", ..errors]
    n if n > 200 -> ["Title must be 200 characters or fewer.", ..errors]
    _ -> errors
  }

  case errors {
    [] -> Ok(trimmed)
    _ -> Error(errors)
  }
}

// ── Request Handlers ────────────────────────────────────────────────────

fn new_task_form_handler(
  _req: Request,
  board_id: String,
) -> Response {
  let html = new_task_modal_content(board_id, "", [])

  wisp.html_response(element.to_string(html), 200)
  |> wisp.set_header("HX-Trigger-After-Swap", "openModal")
}

fn create_task(
  req: Request,
  ctx: Context,
  board_id: String,
) -> Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validate_title(title) {
    Error(errors) -> {
      let html = new_task_modal_content(board_id, title, errors)

      wisp.html_response(element.to_string(html), 422)
      |> wisp.set_header("HX-Trigger-After-Swap", "openModal")
    }

    Ok(valid_title) -> {
      let task_id = wisp.random_string(16)
      let task =
        db.Task(
          id: task_id,
          title: valid_title,
          description: "",
          done: False,
          due_date: option.None,
        )
      let assert Ok(_) = db.create_task(ctx.db, board_id, task)

      wisp.html_response("", 201)
      |> wisp.set_header("HX-Trigger", "closeModal, taskCreated")
    }
  }
}

fn confirm_delete_handler(
  _req: Request,
  task_id: String,
  ctx: Context,
) -> Response {
  let assert Ok(task) = db.get_task(ctx.db, task_id)

  let html = confirm_delete_modal_content(task)

  wisp.html_response(element.to_string(html), 200)
  |> wisp.set_header("HX-Trigger-After-Swap", "openModal")
}

fn delete_task(
  _req: Request,
  task_id: String,
  ctx: Context,
) -> Response {
  let assert Ok(_) = db.delete_task(ctx.db, task_id)

  wisp.html_response("", 200)
  |> wisp.set_header("HX-Trigger", "closeModal, taskDeleted")
}

fn list_tasks(
  _req: Request,
  ctx: Context,
  board_id: String,
) -> Response {
  let assert Ok(tasks) = db.list_tasks(ctx.db, board_id)
  let today = date_helpers.today()

  let html =
    element.fragment(
      list.map(tasks, fn(task) { task_item(task, today) }),
    )

  wisp.html_response(element.to_string(html), 200)
}

// ── Router ──────────────────────────────────────────────────────────────

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> board_page(req, ctx, "default")

    ["boards", board_id] ->
      board_page(req, ctx, board_id)

    ["boards", board_id, "tasks"] ->
      case req.method {
        http.Get -> list_tasks(req, ctx, board_id)
        http.Post -> create_task(req, ctx, board_id)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }

    ["boards", board_id, "tasks", "new"] -> {
      use <- wisp.require_method(req, http.Get)
      new_task_form_handler(req, board_id)
    }

    ["tasks", task_id] ->
      case req.method {
        http.Delete -> delete_task(req, task_id, ctx)
        _ -> wisp.method_not_allowed([http.Delete])
      }

    ["tasks", task_id, "confirm-delete"] -> {
      use <- wisp.require_method(req, http.Get)
      confirm_delete_handler(req, task_id, ctx)
    }

    _ -> wisp.not_found()
  }
}
```

### `priv/static/css/style.css` (modal additions)

```css
/* ============================================================
   Modal Dialog
   ============================================================ */

/* The dialog element itself */
.modal-dialog {
  border: none;
  border-radius: 8px;
  padding: 0;
  max-width: 500px;
  width: 90vw;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
  animation: modal-fade-in 150ms ease-out;
}

/* Backdrop */
.modal-dialog::backdrop {
  background-color: rgba(0, 0, 0, 0.5);
  animation: backdrop-fade-in 150ms ease-out;
}

/* Fade-in animations */
@keyframes modal-fade-in {
  from {
    opacity: 0;
    transform: translateY(-10px) scale(0.98);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes backdrop-fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

/* Content wrapper inside the dialog */
.modal-content {
  padding: 0;
}

/* Header with title and close button */
.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  border-bottom: 1px solid #e5e7eb;
}

.modal-header h2 {
  margin: 0;
  font-size: 1.25rem;
  font-weight: 600;
  color: #111827;
}

/* Close button (X) */
.btn-close {
  background: none;
  border: none;
  font-size: 1.25rem;
  color: #6b7280;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
  line-height: 1;
}

.btn-close:hover {
  background-color: #f3f4f6;
  color: #111827;
}

/* Body content */
.modal-body {
  padding: 16px 24px;
}

/* Form inside modal */
.modal-form {
  padding: 16px 24px;
}

.modal-form .form-group {
  margin-bottom: 16px;
}

.modal-form label {
  display: block;
  margin-bottom: 4px;
  font-weight: 500;
  font-size: 0.9rem;
  color: #374151;
}

.modal-form input[type="text"] {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 1rem;
  box-sizing: border-box;
}

.modal-form input[type="text"]:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

/* Action buttons row at the bottom */
.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  padding: 16px 24px;
  border-top: 1px solid #e5e7eb;
}

/* Error messages inside modal */
.error-list {
  padding: 12px 16px;
  margin: 0 0 8px 0;
  background-color: #fef2f2;
  border-left: 3px solid #ef4444;
  border-radius: 0 4px 4px 0;
}

.error-message {
  margin: 0;
  padding: 2px 0;
  color: #b91c1c;
  font-size: 0.9rem;
}

/* ── Buttons ──────────────────────────────────────────────── */

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 150ms ease;
}

.btn-primary {
  background-color: #3b82f6;
  color: #fff;
}

.btn-primary:hover {
  background-color: #2563eb;
}

.btn-secondary {
  background-color: #f3f4f6;
  color: #374151;
}

.btn-secondary:hover {
  background-color: #e5e7eb;
}

.btn-danger {
  background-color: #ef4444;
  color: #fff;
}

.btn-danger:hover {
  background-color: #dc2626;
}

.btn-sm {
  padding: 4px 10px;
  font-size: 0.8rem;
}

/* ── HTMX indicator ──────────────────────────────────────── */

.htmx-indicator {
  display: none;
}

.htmx-request .htmx-indicator {
  display: inline-block;
}
```

---

## 4. Exercises

### Exercise 1: Edit Task Modal

Add an "Edit" button to each task item that opens a modal with the task's
current title pre-filled in an input. Submitting the form should update the
task title and close the modal.

1. Add a `GET /tasks/:id/edit` route that returns an edit form fragment.
2. The form uses `hx-put` (or `hx-post` with a method override) to update the task.
3. On success (200), return `HX-Trigger: closeModal, taskUpdated`.
4. The task list listens for `taskUpdated from:body` to refresh.
5. On validation failure (422), re-render the form in the modal.

**Acceptance criteria:** Clicking "Edit" opens a modal with the current title.
Submitting updates the task. Validation errors appear in the modal. Success
closes the modal and the task list shows the updated title.

### Exercise 2: Click-Outside-to-Close

Add behaviour so that clicking the backdrop (outside the dialog content) closes
the modal. The native `<dialog>` does NOT do this by default.

1. Use `_hyperscript` on the dialog: listen for `click` events where the
   click target is the dialog itself (not a descendant).
2. Call `close()` when the click is on the backdrop.

**Hint:** When the user clicks the backdrop of a `<dialog>`, the click event's
`target` is the `<dialog>` element itself. When they click inside the dialog
content, the target is a descendant.

**Acceptance criteria:** Clicking the dimmed area behind the modal closes it.
Clicking inside the modal does not.

### Exercise 3: Loading Indicator

Add a loading spinner or "Loading..." text that appears in the dialog while the
server is processing the modal content request.

1. Put a default loading indicator inside the `<dialog>` element (as initial
   content).
2. When HTMX swaps the response in, the loading indicator is replaced.
3. Style it to be centered in the dialog.

**Hint:** The dialog shell can have default children that get replaced by the
`InnerHTML` swap. Put a spinner `<div>` as the initial child.

**Acceptance criteria:** When clicking "New Task" or "Delete", a brief loading
indicator appears before the server content loads.

### Exercise 4: Multiple Modal Sizes

Support small, medium, and large modals. The server determines the size by
including a CSS class in the response.

1. Add `.modal-sm`, `.modal-md`, and `.modal-lg` CSS classes with different
   `max-width` values.
2. The modal content wrapper includes the size class:
   `<div class="modal-content modal-lg">`.
3. Use CSS to apply the width based on the content wrapper class.

**Acceptance criteria:** Different modal endpoints can produce different-sized
dialogs. The "New Task" form uses medium size, and a future "Task Details" view
uses large.

### Exercise 5: Stacked Confirmation

From the "Edit Task" modal, add a "Delete" button that opens a second
confirmation dialog on top. The confirmation dialog should close back to the
edit modal, or delete the task and close both.

1. Add a second `<dialog id="modal-confirm">` to the layout.
2. The "Delete" button inside the edit modal targets `#modal-confirm`.
3. On cancel, close only `#modal-confirm`.
4. On confirm, delete the task and close both dialogs.

**Hint:** You will need a second set of events (`openConfirm`, `closeConfirm`)
and a second `_hyperscript` listener on the second dialog.

**Acceptance criteria:** The edit modal stays visible behind the confirmation
dialog. Cancelling returns to the edit modal. Confirming deletion closes both.

---

## 5. Exercise Solution Hints

Try each exercise on your own before reading these hints.

### Hint for Exercise 1

The edit form is similar to `task_form_for_modal` but pre-fills the input:

```gleam
fn edit_task_modal_content(
  task: db.Task,
  errors: List(String),
) -> element.Element(Nil) {
  html.div([attribute.class("modal-content")], [
    html.div([attribute.class("modal-header")], [
      html.h2(
        [attribute.id("modal-title")],
        [element.text("Edit Task")],
      ),
      close_button(),
    ]),
    html.form(
      [
        hx.post("/tasks/" <> task.id <> "/update"),
        hx.target(hx.Selector("#modal")),
        hx.swap(hx.InnerHTML),
      ],
      [
        error_list(errors),
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("title")],
            [element.text("Task title")],
          ),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.value(task.title),
            attribute.required(True),
            attribute.autofocus(True),
          ]),
        ]),
        html.div([attribute.class("modal-actions")], [
          html.button(
            [
              attribute.type_("button"),
              attribute.class("btn btn-secondary"),
              attribute.attribute(
                "_",
                "on click call the closest <dialog/> to me .close()",
              ),
            ],
            [element.text("Cancel")],
          ),
          html.button(
            [
              attribute.type_("submit"),
              attribute.class("btn btn-primary"),
            ],
            [element.text("Save Changes")],
          ),
        ]),
      ],
    ),
  ])
}
```

On the server, the update handler follows the same pattern as `create_task`:
validate, return 422 with errors or 200 with `HX-Trigger: closeModal,
taskUpdated`. Add `taskUpdated from:body` to the task list's trigger.

### Hint for Exercise 2

Add this to the dialog's `_hyperscript`:

```
on click
  if the target of the event is me
    call me.close()
```

When the user clicks the backdrop, the event target is the `<dialog>` element
itself (`me`). When they click inside the content, the target is a child
element. So checking `if the target of the event is me` only matches backdrop
clicks.

The full `_hyperscript` on the dialog becomes:

```
on openModal
  set my @data-opener to (the closest <button/> to the target of the event).id
  call me.showModal()
on closeModal call me.close()
on click
  if the target of the event is me
    call me.close()
on close
  set openerEl to the first <#{my @data-opener} /> in the document
  if openerEl exists call openerEl.focus()
```

### Hint for Exercise 3

Change the `modal_shell` function to include a default child:

```gleam
fn modal_shell() -> element.Element(Nil) {
  element.element(
    "dialog",
    [
      attribute.id("modal"),
      attribute.class("modal-dialog"),
      attribute.attribute("aria-labelledby", "modal-title"),
      attribute.attribute("_", "..."),
    ],
    [
      html.div([attribute.class("modal-loading")], [
        html.p([], [element.text("Loading...")]),
      ]),
    ],
  )
}
```

When HTMX swaps the response with `InnerHTML`, it replaces the loading div with
the actual content. Add CSS to center the loading text:

```css
.modal-loading {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 48px 24px;
  color: #9ca3af;
}
```

### Hint for Exercise 4

Use CSS custom properties or descendant selectors. The dialog width adapts
based on the content class:

```css
.modal-dialog:has(.modal-sm) { max-width: 360px; }
.modal-dialog:has(.modal-md) { max-width: 500px; }
.modal-dialog:has(.modal-lg) { max-width: 720px; }
```

The `:has()` selector checks if the dialog contains a child with the given
class. Browser support for `:has()` is excellent in all modern browsers
(Chrome 105+, Firefox 121+, Safari 15.4+).

Then on the server, simply add the class to the content wrapper:

```gleam
html.div([attribute.class("modal-content modal-lg")], [...])
```

### Hint for Exercise 5

Add a second dialog to the layout:

```gleam
fn confirm_shell() -> element.Element(Nil) {
  element.element(
    "dialog",
    [
      attribute.id("modal-confirm"),
      attribute.class("modal-dialog modal-dialog-sm"),
      attribute.attribute("aria-labelledby", "confirm-title"),
      attribute.attribute(
        "_",
        "on openConfirm call me.showModal()
         on closeConfirm call me.close()",
      ),
    ],
    [],
  )
}
```

The "Delete" button inside the edit modal targets `#modal-confirm`:

```gleam
html.button(
  [
    attribute.class("btn btn-danger"),
    hx.get("/tasks/" <> task.id <> "/confirm-delete"),
    hx.target(hx.Selector("#modal-confirm")),
    hx.swap(hx.InnerHTML),
  ],
  [element.text("Delete")],
)
```

The server handler for `/tasks/:id/confirm-delete` returns:

```gleam
wisp.set_header("HX-Trigger-After-Swap", "openConfirm")
```

On successful delete, close both:

```gleam
wisp.set_header("HX-Trigger", "closeConfirm, closeModal, taskDeleted")
```

The events fire in order: `closeConfirm` closes the top dialog, `closeModal`
closes the edit dialog, `taskDeleted` refreshes the task list.

---

## 6. Key Takeaways

1. **The native `<dialog>` element is the right tool for modals.** It provides
   focus trapping, backdrop, Escape-to-close, and top-layer rendering for free.
   Custom overlay implementations reinvent all of this and usually get
   accessibility wrong.

2. **Separate the shell from the content.** The `<dialog>` element lives in
   the layout (always present, always empty). Content is loaded on demand via
   HTMX. This keeps the initial page lightweight and ensures every page can
   open a modal.

3. **Use `HX-Trigger-After-Swap` to open the dialog.** The "After-Swap"
   timing is critical -- it ensures the content is in the DOM before
   `showModal()` is called. Using `HX-Trigger` (which fires before the swap)
   would open an empty dialog.

4. **Use `HX-Trigger` (not After-Swap) to close the dialog.** On success, the
   server returns `HX-Trigger: closeModal, taskCreated`. There is no content to
   wait for, so the close event can fire immediately. The second event
   (`taskCreated`) triggers the task list to refresh.

5. **Forms inside modals follow the standard validation pattern.** The form
   targets the dialog with `InnerHTML` swap. A 422 response re-renders the form
   with errors (dialog stays open). A 201 response closes the dialog. The same
   handler works whether the form is in a modal or on a standalone page.

6. **Confirmation modals are just another modal content load.** Delete buttons
   fetch a confirmation fragment. The "Confirm" button in the fragment executes
   the actual delete. This two-step pattern is safer and more user-friendly than
   `window.confirm()`.

7. **Return focus to the trigger element.** Store the trigger button's ID when
   the modal opens and restore focus after close. The native `<dialog>` does
   this automatically in simple cases, but HTMX swaps can disrupt the focus
   chain.

8. **Always include `aria-labelledby` or `aria-label` on the dialog.** The
   `<dialog>` element has an implicit `role="dialog"`, but it needs an
   accessible name. Point `aria-labelledby` at the modal's heading.

9. **The server stays in control.** The server decides what content appears in
   the modal, when it opens (via response headers), and when it closes (via
   response headers). The client-side code is minimal: two `_hyperscript`
   event handlers on the dialog element.

---

## What's Next

In **Chapter 23 -- _hyperscript Practical Patterns**, we will take a deeper look
at the `_hyperscript` language we have been using for client-side behaviour. You
have already seen it for opening/closing dialogs, disabling buttons, and
initializing third-party libraries. Chapter 23 will cover more patterns:
toggling classes, coordinating animations with HTMX swaps, building accessible
dropdowns, and managing small client-side state without JavaScript modules. If
HTMX handles the server communication, `_hyperscript` handles everything else
that needs to happen in the browser.
