# Chapter 16 -- Extensions and Advanced Patterns

**Phase:** Advanced
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain how the HTMX extension system works and how to load extensions.
- Use the `response-targets` extension to swap different content based on HTTP status codes.
- Use the `preload` extension to make navigation feel instant.
- Implement infinite scroll using the `revealed` trigger and a sentinel element.
- Build polling patterns -- from basic to visibility-aware to server-controlled.
- Coordinate independent page sections using the `HX-Trigger` response header and custom events.

---

## 1. Theory

### 1.1 The HTMX Extension Ecosystem

HTMX ships with a small, focused core. It handles triggers, requests, and swaps.
Everything else -- preloading, status-code-based targeting, loading states, multi-swap
tricks -- lives in **extensions**.

Extensions are separate JavaScript files that hook into HTMX's internal event system.
You load them with a `<script>` tag, then activate them on an element (or the entire
page) with the `hx-ext` attribute:

```html
<!-- Step 1: Load the extension script -->
<script src="https://unpkg.com/htmx-ext-response-targets@2.0.2/response-targets.js"></script>

<!-- Step 2: Activate it on a container -->
<body hx-ext="response-targets">
  ...
</body>
```

Once activated, the extension applies to that element and all its descendants. You
can activate multiple extensions by separating them with commas:

```html
<body hx-ext="response-targets, preload">
```

This design keeps HTMX lightweight. You only pay for the features you use. And
because extensions are just JavaScript, the Gleam side of things is straightforward
-- you include the script tags and set the right attributes.

### 1.2 The `response-targets` Extension

In [Chapter 8](../02-intermediate/08-validation-and-error-feedback.md) we learned that HTMX swaps the response into the target regardless
of the HTTP status code. That default behaviour is useful for simple forms: a 422
response replaces the form with error-annotated HTML, and the user sees what went
wrong.

But what if you want *different* targets for different status codes? For example:

- **200 or 201:** Swap the new task into `#task-list`.
- **422:** Swap validation errors into `#form-errors` (right next to the form).
- **500:** Show an error banner at the top of the page in `#error-banner`.

The `response-targets` extension makes this possible. It introduces new attributes
that follow the pattern `hx-target-<status>`:

| Attribute           | Triggers when           |
|---------------------|-------------------------|
| `hx-target-422`     | Server returns 422      |
| `hx-target-500`     | Server returns 500      |
| `hx-target-404`     | Server returns 404      |
| `hx-target-5*`      | Server returns any 5xx  |
| `hx-target-error`   | Server returns any 4xx or 5xx |

The wildcard `*` matches any digit. So `hx-target-5*` catches 500, 502, 503, and
every other 5xx code. And `hx-target-error` is the catch-all for anything that is
not a 2xx or 3xx.

Without this extension, you are stuck with a single target for all responses. With
it, your error handling becomes surgical.

### 1.3 The `preload` Extension

When a user clicks a link, there is always a delay -- the browser sends the request,
waits for the response, and then renders it. Even if that delay is 100 milliseconds,
the user perceives a lag.

The `preload` extension eliminates this perceived delay by fetching the content
*before* the user clicks. It listens for hover or focus events and fires the request
early. By the time the user actually clicks, the response is already cached and the
swap feels instant.

There are three preload strategies:

| Strategy      | When it fires                  | Trade-off                           |
|---------------|--------------------------------|-------------------------------------|
| `mousedown`   | User presses the mouse button  | ~100ms head start; fewest wasted requests |
| `mouseover`   | Mouse hovers over the element  | ~200-400ms head start; some wasted requests |
| `init`        | Element enters the DOM         | Maximum speed; most wasted requests  |

The best default for most applications is `mousedown`. It gives you a meaningful
head start (the delay between pressing and releasing the mouse button) with
essentially zero wasted requests -- if the user pressed the button, they intend
to click.

### 1.4 Infinite Scroll

Pagination is a common pattern. Traditional pagination uses numbered links at the
bottom of the page: "1 2 3 4 ... 47 Next". This forces the user to click, wait,
re-orient themselves on the new page, and repeat.

Infinite scroll replaces this with a smoother experience. When the user scrolls to
the bottom of the current content, more content loads automatically. The key HTMX
ingredient is the `revealed` trigger.

The pattern works like this:

1. Render the first page of items.
2. At the end of the list, place a **sentinel element** -- a div that triggers a
   request when it scrolls into view.
3. The server responds with the next page of items **plus a new sentinel** (if
   there are more pages).
4. The sentinel replaces itself (`outerHTML` swap) with the new items and a new
   sentinel.
5. Repeat until there are no more items. The last response omits the sentinel and
   shows an "end of list" message instead.

The sentinel element is the clever part. It is not visible content -- it is a
trigger mechanism that loads the next batch and then replaces itself. It is a
self-replicating loader.

### 1.5 Advanced Polling Patterns

In [Chapter 5](../01-beginner/05-click-trigger-swap.md) we covered basic polling with `hx-trigger="every 5s"`. That works,
but it has limitations:

**Problem 1: Polling when the tab is hidden.** If the user switches to another
browser tab, your application keeps polling. This wastes bandwidth, server
resources, and battery life on mobile devices.

**Solution:** Use a JavaScript condition in the trigger:

```
hx-trigger="every 10s [document.visibilityState === 'visible']"
```

The expression in square brackets is evaluated before each poll. If it returns
false, the request is skipped. The `document.visibilityState` API tells you whether
the tab is currently visible to the user.

**Problem 2: Polling when nothing has changed.** If the data has not changed since
the last poll, the server is doing work and sending bytes for no reason.

**Solution:** Use ETags or content hashing. The server includes a hash of the
current data in the response (via the `ETag` header). HTMX automatically sends
`If-None-Match` on subsequent requests. If the data has not changed, the server
returns `304 Not Modified` with an empty body, and HTMX skips the swap.

**Problem 3: Coordinating updates between independent sections.** Polling a
dashboard every few seconds works, but what about updating a section immediately
when something happens? For example, when a user creates a new task, the task
counter should update right away -- not wait for the next poll cycle.

**Solution:** Use the `HX-Trigger` response header. When the server returns a
response, it can include a header that fires a custom event on the client. Other
elements can listen for that event and refresh themselves.

### 1.6 Other Notable Extensions

The HTMX ecosystem has many extensions. Here are a few worth knowing about, even
if we will not use them all in this chapter:

| Extension        | What it does                                                   |
|------------------|----------------------------------------------------------------|
| `loading-states` | Fine-grained control over loading indicators (disable buttons, show spinners, add CSS classes) |
| `multi-swap`     | Swap different parts of the response into different targets in a single response |
| `path-deps`     | Automatically refresh elements when a related path is modified (e.g., refresh `/tasks` list when POST to `/tasks` succeeds) |
| `restored`       | Fires an event when content is restored from the browser's back/forward cache |
| `head-support`   | Merges `<head>` elements (titles, meta tags, stylesheets) on navigation |

You do not need to memorize these. Just know they exist. When you hit a limitation
with core HTMX, check the extension list before reaching for custom JavaScript.

### 1.7 \_hyperscript -- A Companion Language

While extensions add new *HTMX* capabilities, there is a separate project that
adds an entirely new scripting language for client-side behaviour:
**\_hyperscript**.

\_hyperscript is a front-end scripting language created by Carson Gross -- the
same author as HTMX. It uses a natural-language-like syntax declared in HTML
`_` attributes. Where HTMX covers server communication (requests, swaps,
triggers), \_hyperscript covers **client-side behaviour** -- class toggling,
animations, DOM manipulation, and small interactions that do not need a server
round-trip.

Here are a few examples to show what it looks like:

**Toggle a CSS class on click:**

```html
<button _="on click toggle .dark-mode on <body/>">Toggle Dark Mode</button>
```

**Fade out and remove an element:**

```html
<div class="flash-message"
     _="on click transition *opacity to 0 over 300ms then remove me">
  Task created successfully!
</div>
```

**Disable a button while an HTMX request is in flight:**

```html
<button hx-post="/tasks"
        hx-target="#task-list"
        _="on htmx:beforeRequest add @disabled to me
           on htmx:afterRequest remove @disabled from me">
  Add Task
</button>
```

The last example shows the real power of the pairing: HTMX handles the server
request and DOM swap, while \_hyperscript handles the loading-state feedback --
all declared inline in the HTML.

\_hyperscript is **pre-1.0 software** (currently at v0.9.14). The features
covered here are stable, but the language is still evolving. Pin your version
in production.

For a complete reference -- core syntax, essential commands, HTMX integration
patterns, and hands-on exercises -- see **[Appendix B](../05-appendices/A2-hyperscript.md): \_hyperscript**.

---

## 2. Code Walkthrough

We are going to add four features to the Teamwork board:

1. **response-targets** for differentiated error handling.
2. **preload** for instant-feeling navigation.
3. **Infinite scroll** for the task list.
4. **Smart polling** with visibility awareness and event-driven refresh.

### Step 1 -- Load the Extensions

First, update the layout to include the extension scripts and activate them.

```gleam
fn layout(content: element.Element(t)) -> wisp.Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork -- Task Board"),
        // Core HTMX
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        // Extension: response-targets
        html.script(
          [attribute.src(
            "https://unpkg.com/htmx-ext-response-targets@2.0.2/response-targets.js",
          )],
          "",
        ),
        // Extension: preload
        html.script(
          [attribute.src(
            "https://unpkg.com/htmx-ext-preload@2.1.0/preload.js",
          )],
          "",
        ),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body(
        [
          // Activate both extensions on the body.
          // They apply to all descendant elements.
          attribute("hx-ext", "response-targets, preload"),
        ],
        [
          // Error banner for server errors (hidden by default)
          html.div(
            [
              attribute.id("error-banner"),
              attribute.class("error-banner hidden"),
            ],
            [],
          ),
          // Main content
          html.div([attribute.class("container")], [
            html.h1([], [element.text("Teamwork Task Board")]),
            content,
          ]),
        ],
      ),
    ])

  let body = element.to_document_string(page)

  wisp.html_response(body, 200)
}
```

Two things to notice. First, both extensions are loaded as script tags in the
`<head>`. Second, they are activated on the `<body>` with a single `hx-ext`
attribute containing both names separated by a comma. Every element inside the body
can now use features from both extensions.

The `#error-banner` div sits at the top of the page, outside the main content area.
It starts with the class `hidden` (which we will define as `display: none` in CSS).
When a 500 error occurs, the `response-targets` extension will swap error HTML into
this div, making it visible.

### Step 2 -- response-targets for Better Error Handling

Now update the task creation form to use different targets for different status
codes.

```gleam
fn add_task_form(
  errors: List(ValidationError),
  values: List(#(String, String)),
) -> element.Element(t) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    // Inline error area for validation errors
    html.div([attribute.id("form-errors")], case title_errors {
      [] -> []
      errs ->
        list.map(errs, fn(e) {
          html.span(
            [attribute.class("error-text")],
            [element.text(validation.error_message(e))],
          )
        })
    }),

    html.form(
      [
        hx.post("/tasks"),
        // On success (200/201), swap the new task into the list
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        // On 422 (validation error), swap errors into the form error area
        attribute("hx-target-422", "#form-container"),
        attribute("hx-swap-422", "outerHTML"),
        // On any 5xx, swap the error message into the banner
        attribute("hx-target-5*", "#error-banner"),
        attribute("hx-swap-5*", "innerHTML"),
        // Catch-all for other errors
        attribute("hx-target-error", "#error-banner"),
      ],
      [
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
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}
```

Read the attributes on the form carefully. There are now four possible outcomes:

| Status | Target            | Swap        | What happens                                |
|--------|-------------------|-------------|---------------------------------------------|
| 201    | `#task-list`      | `beforeend` | New task item appended to the list           |
| 422    | `#form-container` | `outerHTML` | Form re-rendered with error messages         |
| 5xx    | `#error-banner`   | `innerHTML` | Error message shown in the top banner        |
| Other  | `#error-banner`   | (default)   | Generic error message in the top banner      |

Now update the `create_task` handler to return appropriate responses for each case:

```gleam
fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      let task = Task(id: new_id(), title: valid_title, done: False)
      actor.send(ctx.tasks, AddTask(task))

      // Return just the new task item (201).
      // response-targets routes this to #task-list with beforeend swap.
      let task_html = task_item(task.id, task.title)
      wisp.html_response(element.to_string(task_html), 201)
      |> wisp.set_header("HX-Trigger", "taskAdded")
    }

    Error(errors) -> {
      // Return the form with errors (422).
      // response-targets routes this to #form-container with outerHTML swap.
      let form_html = add_task_form(errors, form_data.values)
      wisp.html_response(element.to_string(form_html), 422)
    }
  }
}
```

The success path now returns only the task item HTML, not a fresh form. The
response-targets extension sends it to `#task-list` with `beforeend` swap, so the
new task appears at the bottom of the list. The error path returns the full form
container with error annotations, targeted at `#form-container` with `outerHTML`
swap.

Notice the `HX-Trigger: taskAdded` header on the success response. We will use
this in Step 5 to refresh other parts of the page.

For server errors, create a simple error handler:

```gleam
fn server_error_html() -> String {
  let html =
    html.div([attribute.class("error-banner-content")], [
      html.strong([], [element.text("Something went wrong.")]),
      element.text(" Please try again in a moment."),
      html.button(
        [
          attribute.class("btn btn-small"),
          attribute("onclick", "this.parentElement.parentElement.classList.add('hidden')"),
        ],
        [element.text("Dismiss")],
      ),
    ])
  element.to_string(html)
}
```

### Step 3 -- Preload for Instant Navigation

Add preload to the navigation links. This is the easiest win in the chapter --
two attributes and navigation feels instant.

```gleam
fn nav_bar() -> element.Element(t) {
  html.nav([attribute.class("nav")], [
    html.a(
      [
        attribute.href("/"),
        hx.boost(True),
        attribute("preload", "mousedown"),
      ],
      [element.text("Board")],
    ),
    html.a(
      [
        attribute.href("/boards"),
        hx.boost(True),
        attribute("preload", "mousedown"),
      ],
      [element.text("All Boards")],
    ),
    html.a(
      [
        attribute.href("/settings"),
        hx.boost(True),
        attribute("preload", "mouseover"),
      ],
      [element.text("Settings")],
    ),
  ])
}
```

Notice the difference in preload strategies. The "Board" and "All Boards" links
use `mousedown` -- minimal wasted requests, but enough of a head start to feel
snappy. The "Settings" link uses `mouseover` -- it preloads as soon as the user
hovers. This is useful for pages that are cheap to generate but slightly slower
to render.

Because we activated the `preload` extension on the `<body>` in Step 1, these
attributes work on any element inside the body. No additional setup needed.

### Step 4 -- Infinite Scroll for Long Task Lists

This is the most interesting pattern in the chapter. We need three things:

1. A function that renders a page of tasks with a sentinel at the end.
2. A server handler that accepts a `page` query parameter.
3. Pagination logic to determine if more pages exist.

First, the sentinel-based task list renderer:

```gleam
fn task_list_page(
  tasks: List(Task),
  page: Int,
  has_more: Bool,
) -> element.Element(t) {
  element.fragment([
    // Render this page's tasks
    ..list.append(
      list.map(tasks, fn(task) { task_item(task.id, task.title) }),
      // After the tasks, add either a sentinel or an end message
      [
        case has_more {
          True ->
            // The sentinel: triggers loading the next page when scrolled into view
            html.div(
              [
                attribute.class("loading-sentinel"),
                hx.get(
                  "/tasks?page=" <> int.to_string(page + 1),
                ),
                hx.trigger([hx.revealed()]),
                hx.swap(hx.OuterHTML),
                hx.indicator(hx.Selector("#load-more-spinner")),
              ],
              [
                html.span(
                  [
                    attribute.id("load-more-spinner"),
                    attribute.class("htmx-indicator"),
                  ],
                  [element.text("Loading more tasks...")],
                ),
              ],
            )
          False ->
            html.p(
              [attribute.class("end-of-list")],
              [element.text("You have reached the end of the list.")],
            )
        },
      ],
    ),
  ])
}
```

Let us trace through what happens when the user scrolls.

**Initial page load:** The server renders tasks 1-20 and a sentinel div at the end.
The sentinel is off-screen, below the visible area.

```
Viewport
+----------------------------------+
| Task 1                           |
| Task 2                           |
| Task 3                           |
| ...                              |
| Task 15                          |
+----------------------------------+
  Task 16
  Task 17
  Task 18
  Task 19
  Task 20
  [sentinel: GET /tasks?page=2]     <-- not yet visible
```

**User scrolls down:** The sentinel enters the viewport. HTMX fires the `revealed`
trigger, sending `GET /tasks?page=2`.

```
Viewport
+----------------------------------+
| Task 16                          |
| Task 17                          |
| Task 18                          |
| Task 19                          |
| Task 20                          |
| [Loading more tasks...]          | <-- sentinel visible, request fired
+----------------------------------+
```

**Server responds:** The response contains tasks 21-40 and a *new* sentinel. The
old sentinel replaces itself (`outerHTML` swap) with the new content.

```
Viewport
+----------------------------------+
| Task 18                          |
| Task 19                          |
| Task 20                          |
| Task 21                          | <-- new content
| Task 22                          |
| Task 23                          |
+----------------------------------+
  Task 24
  ...
  Task 40
  [sentinel: GET /tasks?page=3]     <-- new sentinel, not yet visible
```

**Last page:** When there are no more tasks, the server responds with the final
batch and an "end of list" message instead of a sentinel. No more requests will
fire.

Now the server handler with pagination:

```gleam
fn get_tasks_paginated(req: wisp.Request, ctx: Context) -> wisp.Response {
  let query = wisp.get_query(req)

  // Parse the page number from the query string, defaulting to 1
  let page =
    list.key_find(query, "page")
    |> result.unwrap("1")
    |> int.parse
    |> result.unwrap(1)

  let per_page = 20
  let offset = { page - 1 } * per_page

  // Get all tasks from the actor
  let all_tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let total = list.length(all_tasks)

  // Slice the list for the current page
  let page_tasks =
    all_tasks
    |> list.drop(offset)
    |> list.take(per_page)

  // Determine if there are more pages
  let has_more = offset + per_page < total

  let html = task_list_page(page_tasks, page, has_more)
  wisp.html_response(element.to_string(html), 200)
}
```

The pagination logic is straightforward. We parse the `page` query parameter,
calculate the offset, slice the task list, and check whether more items exist
beyond the current page.

In a real application you would push this pagination into your database query
(`LIMIT` and `OFFSET` in SQL). Slicing an in-memory list works for demonstration
purposes but does not scale.

The initial page needs to render the task list container with the first batch:

```gleam
fn home_page(req: wisp.Request, ctx: Context) -> wisp.Response {
  let all_tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let first_page = list.take(all_tasks, 20)
  let has_more = list.length(all_tasks) > 20

  let content =
    html.div([], [
      nav_bar(),
      add_task_form([], []),

      // Task list container
      html.div(
        [attribute.id("task-list"), attribute.class("task-list")],
        [task_list_page(first_page, 1, has_more)],
      ),

      // Task counter (refreshes via events and polling)
      html.div(
        [
          attribute.id("task-counter"),
          hx.get("/stats"),
          attribute("hx-trigger", "load, taskAdded from:body, every 30s [document.visibilityState === 'visible']"),
          hx.swap(hx.InnerHTML),
        ],
        [element.text("Loading stats...")],
      ),
    ])

  layout(content)
}
```

Notice the `#task-counter` div near the bottom. We will explain that trigger in
the next step.

### Step 5 -- Smart Polling and Event Coordination

The task counter div has the most interesting trigger in the entire project. Let
us break it down piece by piece:

```
hx-trigger="load, taskAdded from:body, every 30s [document.visibilityState === 'visible']"
```

This trigger is actually three triggers combined with commas:

| Trigger | What it does |
|---------|-------------|
| `load` | Fire once immediately when the element enters the DOM. |
| `taskAdded from:body` | Fire whenever a `taskAdded` event bubbles up to the `<body>`. This event is triggered by the `HX-Trigger: taskAdded` response header when a task is created. |
| `every 30s [document.visibilityState === 'visible']` | Poll every 30 seconds, but only if the browser tab is currently visible. |

The combination gives us three refresh behaviours:

1. **On page load:** The counter appears immediately.
2. **On task creation:** The counter updates within milliseconds of a new task
   being created, because the `HX-Trigger` header fires the `taskAdded` event.
3. **On a slow poll:** Even if no tasks are created, the counter refreshes every
   30 seconds -- but only when the user is actually looking at the tab.

The server handler for stats is simple:

```gleam
fn get_stats(ctx: Context) -> wisp.Response {
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let total = list.length(tasks)
  let done =
    tasks
    |> list.filter(fn(t) { t.done })
    |> list.length

  let html =
    html.div([], [
      html.span([attribute.class("stat")], [
        element.text(int.to_string(total) <> " total"),
      ]),
      html.span([attribute.class("stat")], [
        element.text(int.to_string(done) <> " done"),
      ]),
      html.span([attribute.class("stat")], [
        element.text(int.to_string(total - done) <> " remaining"),
      ]),
    ])

  wisp.html_response(element.to_string(html), 200)
}
```

The important part is on the *sending* side. Recall that the `create_task` handler
returns this header on success:

```gleam
wisp.set_header("HX-Trigger", "taskAdded")
```

When HTMX processes a response with this header, it dispatches a `taskAdded` event
on the document body. The task counter is listening for exactly that event via
`taskAdded from:body` in its trigger. The result: the counter updates automatically
whenever a task is created, without any polling delay.

You can trigger multiple events by separating them with commas in the header value:

```gleam
wisp.set_header("HX-Trigger", "taskAdded, statsChanged")
```

Or include event data as JSON:

```gleam
wisp.set_header("HX-Trigger", "{\"taskAdded\":{\"id\":\"42\"}}")
```

For our purposes, the simple event name is enough.

### Step 6 -- Wire It into the Router

Update the router to handle the new endpoints:

```gleam
pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)

    ["tasks"] ->
      case req.method {
        http.Get -> get_tasks_paginated(req, ctx)
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }

    ["tasks", id] ->
      case req.method {
        http.Delete -> delete_task(req, id, ctx)
        _ -> wisp.method_not_allowed([http.Delete])
      }

    ["stats"] -> {
      use <- wisp.require_method(req, http.Get)
      get_stats(ctx)
    }

    ["boards"] -> boards_page(req, ctx)
    ["settings"] -> settings_page(req, ctx)

    _ -> wisp.not_found()
  }
}
```

---

## 3. Full Code Listing

Here is the complete updated `src/teamwork/web.gleam` with all extensions and
patterns integrated.

### `src/teamwork/web.gleam`

```gleam
import gleam/erlang/process
import gleam/http
import gleam/int
import gleam/list
import gleam/otp/actor
import gleam/result
import lustre/attribute.{attribute}
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}
import hx

import teamwork/context.{type Context}
import teamwork/task.{type Task, Task}
import teamwork/validation

// ── Layout ──────────────────────────────────────────────────────────────

fn layout(content: element.Element(t)) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork -- Task Board"),
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        html.script(
          [attribute.src(
            "https://unpkg.com/htmx-ext-response-targets@2.0.2/response-targets.js",
          )],
          "",
        ),
        html.script(
          [attribute.src(
            "https://unpkg.com/htmx-ext-preload@2.1.0/preload.js",
          )],
          "",
        ),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body(
        [attribute("hx-ext", "response-targets, preload")],
        [
          // Global error banner for 5xx errors
          html.div(
            [
              attribute.id("error-banner"),
              attribute.class("error-banner hidden"),
            ],
            [],
          ),
          html.div([attribute.class("container")], [
            html.h1([], [element.text("Teamwork Task Board")]),
            content,
          ]),
        ],
      ),
    ])

  let body = element.to_document_string(page)

  wisp.html_response(body, 200)
}

// ── Navigation ──────────────────────────────────────────────────────────

fn nav_bar() -> element.Element(t) {
  html.nav([attribute.class("nav")], [
    html.a(
      [
        attribute.href("/"),
        hx.boost(True),
        attribute("preload", "mousedown"),
      ],
      [element.text("Board")],
    ),
    html.a(
      [
        attribute.href("/boards"),
        hx.boost(True),
        attribute("preload", "mousedown"),
      ],
      [element.text("All Boards")],
    ),
    html.a(
      [
        attribute.href("/settings"),
        hx.boost(True),
        attribute("preload", "mouseover"),
      ],
      [element.text("Settings")],
    ),
  ])
}

// ── Components ──────────────────────────────────────────────────────────

fn task_item(id: String, title: String) -> element.Element(t) {
  html.li([attribute.id("task-" <> id), attribute.class("task-item")], [
    html.span([], [element.text(title)]),
    html.button(
      [
        attribute.class("btn btn-danger btn-small"),
        hx.delete("/tasks/" <> id),
        hx.target(hx.Selector("#task-" <> id)),
        hx.swap(hx.OuterHTML),
      ],
      [element.text("Delete")],
    ),
  ])
}

fn task_list_page(
  tasks: List(Task),
  page: Int,
  has_more: Bool,
) -> element.Element(t) {
  element.fragment([
    ..list.append(
      list.map(tasks, fn(task) { task_item(task.id, task.title) }),
      [
        case has_more {
          True ->
            html.div(
              [
                attribute.class("loading-sentinel"),
                hx.get("/tasks?page=" <> int.to_string(page + 1)),
                hx.trigger([hx.revealed()]),
                hx.swap(hx.OuterHTML),
                hx.indicator(hx.Selector("#load-more-spinner")),
              ],
              [
                html.span(
                  [
                    attribute.id("load-more-spinner"),
                    attribute.class("htmx-indicator"),
                  ],
                  [element.text("Loading more tasks...")],
                ),
              ],
            )
          False ->
            html.p(
              [attribute.class("end-of-list")],
              [element.text("You have reached the end of the list.")],
            )
        },
      ],
    ),
  ])
}

fn add_task_form(
  errors: List(validation.ValidationError),
  values: List(#(String, String)),
) -> element.Element(t) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.div([attribute.id("form-errors")], case title_errors {
      [] -> []
      errs ->
        list.map(errs, fn(e) {
          html.span(
            [attribute.class("error-text")],
            [element.text(validation.error_message(e))],
          )
        })
    }),
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        attribute("hx-target-422", "#form-container"),
        attribute("hx-swap-422", "outerHTML"),
        attribute("hx-target-5*", "#error-banner"),
        attribute("hx-swap-5*", "innerHTML"),
        attribute("hx-target-error", "#error-banner"),
      ],
      [
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
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}

// ── Pages ───────────────────────────────────────────────────────────────

fn home_page(req: Request, ctx: Context) -> Response {
  let all_tasks = actor.call(ctx.tasks, 1000, task.GetTasks)
  let first_page = list.take(all_tasks, 20)
  let has_more = list.length(all_tasks) > 20

  let content =
    html.div([], [
      nav_bar(),
      add_task_form([], []),

      // Task list with infinite scroll
      html.div(
        [attribute.id("task-list"), attribute.class("task-list")],
        [task_list_page(first_page, 1, has_more)],
      ),

      // Task counter: refreshes on load, on task events, and via polling
      html.div(
        [
          attribute.id("task-counter"),
          attribute.class("stats-panel"),
          hx.get("/stats"),
          attribute("hx-trigger", "load, taskAdded from:body, every 30s [document.visibilityState === 'visible']"),
          hx.swap(hx.InnerHTML),
        ],
        [element.text("Loading stats...")],
      ),
    ])

  layout(content)
}

// ── Handlers ────────────────────────────────────────────────────────────

fn get_tasks_paginated(req: Request, ctx: Context) -> Response {
  let query = wisp.get_query(req)
  let page =
    list.key_find(query, "page")
    |> result.unwrap("1")
    |> int.parse
    |> result.unwrap(1)

  let per_page = 20
  let offset = { page - 1 } * per_page

  let all_tasks = actor.call(ctx.tasks, 1000, task.GetTasks)
  let total = list.length(all_tasks)
  let page_tasks =
    all_tasks
    |> list.drop(offset)
    |> list.take(per_page)
  let has_more = offset + per_page < total

  let html = task_list_page(page_tasks, page, has_more)
  wisp.html_response(element.to_string(html), 200)
}

fn create_task(req: Request, ctx: Context) -> Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      let id = int.to_string(int.random(100_000))
      let new_task = Task(id: id, title: valid_title, done: False)
      actor.send(ctx.tasks, task.AddTask(new_task))

      let task_html = task_item(id, valid_title)
      wisp.html_response(element.to_string(task_html), 201)
      |> wisp.set_header("HX-Trigger", "taskAdded")
    }

    Error(errors) -> {
      let form_html = add_task_form(errors, form_data.values)
      wisp.html_response(element.to_string(form_html), 422)
    }
  }
}

fn delete_task(_req: Request, id: String, ctx: Context) -> Response {
  actor.send(ctx.tasks, task.DeleteTask(id))

  wisp.html_response("", 200)
  |> wisp.set_header("HX-Trigger", "taskAdded")
}

fn get_stats(ctx: Context) -> Response {
  let tasks = actor.call(ctx.tasks, 1000, task.GetTasks)
  let total = list.length(tasks)
  let done =
    tasks
    |> list.filter(fn(t) { t.done })
    |> list.length

  let html =
    html.div([], [
      html.span([attribute.class("stat")], [
        element.text(int.to_string(total) <> " total"),
      ]),
      html.span([attribute.class("stat-separator")], [element.text(" | ")]),
      html.span([attribute.class("stat")], [
        element.text(int.to_string(done) <> " done"),
      ]),
      html.span([attribute.class("stat-separator")], [element.text(" | ")]),
      html.span([attribute.class("stat")], [
        element.text(int.to_string(total - done) <> " remaining"),
      ]),
    ])

  wisp.html_response(element.to_string(html), 200)
}

// ── Router ──────────────────────────────────────────────────────────────

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)

    ["tasks"] ->
      case req.method {
        http.Get -> get_tasks_paginated(req, ctx)
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }

    ["tasks", id] -> {
      use <- wisp.require_method(req, http.Delete)
      delete_task(req, id, ctx)
    }

    ["stats"] -> {
      use <- wisp.require_method(req, http.Get)
      get_stats(ctx)
    }

    _ -> wisp.not_found()
  }
}
```

### `priv/static/css/style.css` (additions)

```css
/* ─── Error banner ─── */

.error-banner {
  background-color: #fef2f2;
  border: 1px solid #e74c3c;
  border-radius: 4px;
  padding: 1rem;
  margin-bottom: 1rem;
}

.error-banner.hidden {
  display: none;
}

.error-banner-content {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* ─── Stats panel ─── */

.stats-panel {
  background-color: #f0f4f8;
  border-radius: 4px;
  padding: 0.75rem 1rem;
  margin-top: 1.5rem;
  font-size: 0.9rem;
  color: #4a5568;
}

.stat {
  font-weight: 600;
}

.stat-separator {
  color: #a0aec0;
}

/* ─── Infinite scroll ─── */

.loading-sentinel {
  text-align: center;
  padding: 1rem;
  color: #718096;
}

.end-of-list {
  text-align: center;
  padding: 1rem;
  color: #a0aec0;
  font-style: italic;
}

/* ─── Navigation ─── */

.nav {
  display: flex;
  gap: 1rem;
  margin-bottom: 1.5rem;
  padding-bottom: 0.75rem;
  border-bottom: 1px solid #e2e8f0;
}

.nav a {
  color: #4a6fa5;
  text-decoration: none;
  font-weight: 500;
}

.nav a:hover {
  text-decoration: underline;
}

/* ─── Button variants ─── */

.btn-danger {
  background-color: #e74c3c;
}

.btn-danger:hover {
  background-color: #c0392b;
}

.btn-small {
  padding: 0.25rem 0.75rem;
  font-size: 0.85rem;
}
```

---

## 4. Exercise

### Task 1 -- Add response-targets for Error Handling

Update the task creation form so that:

- On **201**, the new task is appended to `#task-list` (using `beforeend` swap).
- On **422**, the form container is replaced with the error-annotated form (using
  `outerHTML` swap on `#form-container`).
- On **500** or any 5xx, an error message appears in `#error-banner` at the top of
  the page.

Verify by submitting an empty form (should show validation error next to the form)
and then submitting a valid title (should append the task to the list).

To test the 500 path, temporarily add a handler that always returns 500:

```gleam
wisp.html_response("<p>Internal server error. Please try again.</p>", 500)
```

### Task 2 -- Add Preload to Navigation

Add the `preload` extension to your navigation links:

- Use `preload="mousedown"` on the most common links.
- Use `preload="mouseover"` on one link to feel the difference.

Verify by opening the browser's Network tab. When you hover over a link with
`mouseover`, you should see a GET request fire before you click. When you click,
the page should swap almost instantly because the response is already cached.

### Task 3 -- Implement Infinite Scroll

Implement infinite scroll for the task list with 20 tasks per page:

- Seed your task store with at least 50 tasks so you have multiple pages.
- Render the first page with a sentinel element at the bottom.
- Each subsequent page request returns more tasks and a new sentinel (or an end
  message on the last page).
- Add a "Loading more tasks..." indicator that appears while the next page loads.

Verify by scrolling to the bottom of the task list. New tasks should appear
seamlessly. The loading indicator should flash briefly. When you reach the end, you
should see "You have reached the end of the list."

### Task 4 -- Add a Dashboard Panel with Smart Polling

Create a stats panel that shows total tasks, completed tasks, and remaining tasks.
The panel should refresh in three ways:

- **On page load:** Show the stats immediately.
- **On task creation or deletion:** Update within milliseconds using `HX-Trigger`.
- **On a slow poll:** Refresh every 30 seconds, but only if the browser tab is
  visible.

The trigger should look like:

```
hx-trigger="load, taskAdded from:body, every 30s [document.visibilityState === 'visible']"
```

Verify by creating a task and watching the stats update immediately. Then switch
to another tab, wait 30 seconds, switch back, and confirm that the stats refresh
(check the Network tab to see that no requests were made while the tab was hidden).

### Task 5 -- Coordinate Form and Counter with HX-Trigger

Make sure the `create_task` and `delete_task` handlers both return the
`HX-Trigger: taskAdded` header. This way, both creating and deleting tasks cause
the stats panel to refresh automatically.

Verify by deleting a task. The stats panel should update its count immediately
without waiting for the next poll cycle.

---

## 5. Exercise Solution Hints

Try each task on your own before reading these hints.

### Hint for Task 1

The `response-targets` extension uses attributes that follow the pattern
`hx-target-<status>`. For a 422 response:

```gleam
attribute("hx-target-422", "#form-container"),
attribute("hx-swap-422", "outerHTML"),
```

For any 5xx response, use the wildcard:

```gleam
attribute("hx-target-5*", "#error-banner"),
attribute("hx-swap-5*", "innerHTML"),
```

The catch-all for any non-2xx response:

```gleam
attribute("hx-target-error", "#error-banner"),
```

Make sure the `response-targets` extension is loaded as a script tag and activated
with `hx-ext="response-targets"` on a parent element (the `<body>` is the easiest
choice).

### Hint for Task 2

Preload requires two things: the extension script loaded and `hx-ext="preload"`
on a parent element. Then use the `preload` attribute on individual links:

```gleam
attribute("preload", "mousedown")
```

The best balance of speed and efficiency is `mousedown`. The user has already
committed to clicking -- they pressed the button. The response starts loading
during the ~100ms between mouse-down and mouse-up.

If you want even more speed at the cost of some wasted requests, use `mouseover`.
The response starts loading as soon as the user hovers, which is typically
200-400ms before they click.

### Hint for Task 3

The key to infinite scroll is the **sentinel element**. It is a div at the bottom
of the current page of results that uses `hx-trigger="revealed"`:

```gleam
html.div(
  [
    hx.get("/tasks?page=" <> int.to_string(page + 1)),
    hx.trigger([hx.revealed()]),
    hx.swap(hx.OuterHTML),
  ],
  [element.text("Loading...")],
)
```

The `revealed` trigger fires when the element scrolls into the viewport. The
`outerHTML` swap is critical: the sentinel replaces *itself* with the new content,
which includes the next batch of tasks and (if needed) a new sentinel.

For pagination on the server, parse the `page` query parameter:

```gleam
let page =
  list.key_find(query, "page")
  |> result.unwrap("1")
  |> int.parse
  |> result.unwrap(1)
```

Calculate `offset = (page - 1) * per_page` and use `list.drop(offset)` followed
by `list.take(per_page)`.

### Hint for Task 4

The visibility-aware polling trigger uses a JavaScript condition in square brackets:

```gleam
attribute("hx-trigger", "every 30s [document.visibilityState === 'visible']")
```

HTMX evaluates the expression before each poll. If it returns `false`, the request
is skipped. The `document.visibilityState` property returns `"visible"` when the
tab is active and `"hidden"` when it is not.

Combine this with `load` (for the initial fetch) and a custom event (for
immediate updates):

```gleam
attribute("hx-trigger", "load, taskAdded from:body, every 30s [document.visibilityState === 'visible']")
```

### Hint for Task 5

The `HX-Trigger` header is set on the *response*, not the request. In Gleam with
wisp:

```gleam
wisp.html_response(body, 201)
|> wisp.set_header("HX-Trigger", "taskAdded")
```

Any element with `taskAdded from:body` in its `hx-trigger` will fire a request
when this response is processed by HTMX. The event name is arbitrary -- just make
sure the header value matches the trigger name exactly.

You can reuse the same event name (`taskAdded`) for both creation and deletion. The
stats panel does not care *why* it is refreshing, only *that* it should refresh.

---

## 6. Key Takeaways

1. **Extensions keep HTMX small.** The core library handles triggers, requests,
   and swaps. Everything else is an extension. You load them with `<script>` tags
   and activate them with `hx-ext`. You only pay for what you use.

2. **`response-targets` gives you surgical error handling.** Map HTTP status codes
   to different swap targets: `hx-target-422` for validation errors,
   `hx-target-5*` for server errors, `hx-target-error` for the catch-all. Each
   error type can land in the right place on the page.

3. **`preload` makes navigation feel instant.** Add `preload="mousedown"` to links
   and the response starts loading before the user finishes clicking. For the user,
   page transitions feel like they happen in zero time.

4. **Infinite scroll is a sentinel element with `revealed`.** Place a div at the
   bottom of the list with `hx-trigger="revealed"` and `hx-swap="outerHTML"`. When
   it scrolls into view, it loads the next page and replaces itself with the new
   content. The pattern is self-replicating: each response includes a new sentinel
   until there are no more pages.

5. **Visibility-aware polling saves resources.** The condition
   `[document.visibilityState === 'visible']` in a trigger expression prevents
   requests when the tab is hidden. This reduces unnecessary server load, saves
   bandwidth, and improves battery life on mobile.

6. **`HX-Trigger` coordinates independent page sections.** When a server response
   includes the `HX-Trigger` header, HTMX fires a custom event on the body. Any
   element listening for that event (via `from:body` in its trigger) will refresh
   itself. This lets you update multiple parts of the page from a single server
   response without Out-of-Band swaps.

7. **Combine triggers for layered refresh strategies.** A single element can have
   `load` (initial), a custom event (immediate), and `every Ns` (background poll)
   all in one trigger. Each layer covers a different use case, and together they
   ensure the user always sees current data.

8. **Server-side pagination is a `page` query parameter.** Parse it, calculate the
   offset, slice your data, and check if more items exist. In production, push the
   `LIMIT` and `OFFSET` into your SQL query rather than slicing an in-memory list.

9. **\_hyperscript complements HTMX for client-side behaviour.** HTMX talks to
   the server; \_hyperscript handles DOM manipulation, class toggling, and
   animations. Together they cover most interactions without writing vanilla
   JavaScript. See [Appendix B](../05-appendices/A2-hyperscript.md) for the full reference.

---

## What's Next

The Teamwork board now has extensions for error handling and preloading, infinite
scroll for large lists, and smart polling with event coordination. These patterns
cover the vast majority of real-world HTMX applications.

In [Chapter 17](17-deployment-and-production.md) we will look at **deployment and production readiness**: building
releases, configuring environment variables, serving behind a reverse proxy, and
the final touches that take the Teamwork board from a development project to a
production application.

If you want to explore \_hyperscript further -- core syntax, essential commands,
HTMX integration patterns, and hands-on exercises -- head to **[Appendix B](../05-appendices/A2-hyperscript.md):
\_hyperscript**.
