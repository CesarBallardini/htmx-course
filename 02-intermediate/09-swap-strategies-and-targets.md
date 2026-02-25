# Chapter 9 -- Swap Strategies and Targets: Out-of-Band Updates

**Phase:** Intermediate
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the "one request, one target" limitation of standard HTMX swaps and why real applications need more.
- Use `hx-swap-oob` to update multiple, unrelated areas of the page from a single server response.
- Build server responses in Gleam that combine a primary swap target with out-of-band elements using `element.fragment`.
- Use `hx-select` to cherry-pick parts of a server response on the client side.
- Apply CSS transitions with HTMX's swap lifecycle classes (`.htmx-added`, `.htmx-settling`, `.htmx-swapping`) for smooth animations.
- Control swap timing with `hx-swap` modifiers to coordinate CSS transitions with DOM updates.

---

## 1. Theory

### 1.1 The One-Target Problem

Up to this point, every HTMX interaction we have built follows the same pattern:

1. An element triggers an HTTP request.
2. The server returns an HTML fragment.
3. HTMX swaps that fragment into a single target element.

This works beautifully for isolated updates. But think about what happens when
you add a new task to the Teamwork board. The task list needs a new item --
that is the obvious target. But what else should change?

- The **task counter** ("3 of 7 tasks completed") needs to update.
- The **form** should clear itself so the user can immediately add another task.
- Maybe a **progress bar** should advance.

With what we know so far, one request can only update one target. To update the
counter, you would need a second HTMX request. To clear the form, a third. That
is three round trips to the server for what the user perceives as a single
action. It is wasteful, it introduces race conditions, and it creates visual
jank as different parts of the page update at different times.

Here is the problem, visualised:

```
  BEFORE: Standard HTMX (one request, one target)
  ┌──────────────────────────────────────────────────────┐
  │                    Page                               │
  │                                                       │
  │  ┌──────────────────┐   ┌──────────────────────────┐ │
  │  │  Task Counter     │   │  Add Task Form           │ │
  │  │  "2 of 5 done"   │   │  [________] [Add]        │ │
  │  │  (STALE!)         │   │  (NOT CLEARED!)          │ │
  │  └──────────────────┘   └──────────────────────────┘ │
  │                                                       │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  Task List (hx-target)         <── UPDATED       │ │
  │  │  ┌────────────────────────────┐                  │ │
  │  │  │ ☐ Buy groceries            │                  │ │
  │  │  │ ☑ Write report             │                  │ │
  │  │  │ ☐ Call dentist             │                  │ │
  │  │  │ ☐ Fix bug #42             │  <── new task!   │ │
  │  │  └────────────────────────────┘                  │ │
  │  └──────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────┘

  The task list updated, but the counter still says "2 of 5"
  (should be "2 of 6") and the form still shows the text we typed.
```

We need a way for a single server response to update multiple, unrelated parts
of the page. That is exactly what Out-of-Band swaps do.

### 1.2 Out-of-Band (OOB) Swaps

The `hx-swap-oob` attribute is HTMX's answer to the multi-target problem. It
lets the server include extra HTML elements in its response that will be swapped
into the DOM by matching `id`, completely independent of the main `hx-target`.

Here is how it works:

1. The browser sends a request as usual (triggered by `hx-post`, `hx-get`, etc.).
2. The server builds its response with the primary content (which goes to `hx-target` as normal).
3. The server **also** includes additional elements in the response, each marked with `hx-swap-oob="true"` and carrying an `id` attribute.
4. When HTMX receives the response, it scans for any elements with `hx-swap-oob`. It **pulls them out** of the response before the main swap happens.
5. Each OOB element is matched to an existing element in the DOM that has the same `id`.
6. The OOB element replaces (or is swapped into) the matched DOM element.
7. The remaining response content (everything that was not OOB) is swapped into `hx-target` as usual.

Visually:

```
  AFTER: With OOB swaps (one request, multiple targets)

  Server Response:
  ┌──────────────────────────────────────────────────────┐
  │  <li>Fix bug #42</li>              ← main content   │
  │                                       (→ hx-target) │
  │  <div id="task-counter"                              │
  │       hx-swap-oob="true">          ← OOB element 1  │
  │    2 of 6 tasks completed              (→ #counter)  │
  │  </div>                                              │
  │                                                      │
  │  <div id="form-container"                            │
  │       hx-swap-oob="innerHTML">     ← OOB element 2  │
  │    <form>...</form>                    (→ #form)     │
  │  </div>                                              │
  └──────────────────────────────────────────────────────┘

  HTMX processes the response:

  Step 1: Pull out OOB elements (anything with hx-swap-oob)
  Step 2: Swap each OOB element into the DOM element with matching id
  Step 3: Swap the remaining content into hx-target

  Result: THREE areas of the page update from ONE request.

  ┌──────────────────────────────────────────────────────┐
  │                    Page                               │
  │                                                       │
  │  ┌──────────────────┐   ┌──────────────────────────┐ │
  │  │  Task Counter     │   │  Add Task Form           │ │
  │  │  "2 of 6 done"   │   │  [________] [Add]        │ │
  │  │  ✓ UPDATED        │   │  ✓ CLEARED               │ │
  │  └──────────────────┘   └──────────────────────────┘ │
  │                                                       │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  Task List                       ✓ UPDATED       │ │
  │  │  ┌────────────────────────────┐                  │ │
  │  │  │ ☐ Buy groceries            │                  │ │
  │  │  │ ☑ Write report             │                  │ │
  │  │  │ ☐ Call dentist             │                  │ │
  │  │  │ ☐ Fix bug #42             │                  │ │
  │  │  └────────────────────────────┘                  │ │
  │  └──────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────┘
```

**OOB swap strategies.** The `hx-swap-oob` attribute accepts any of the
standard swap strategies:

| Value                          | Behaviour                                                     |
|--------------------------------|---------------------------------------------------------------|
| `hx-swap-oob="true"`          | Shorthand for `outerHTML` -- replaces the entire target element |
| `hx-swap-oob="outerHTML"`     | Replaces the entire target element (same as `true`)            |
| `hx-swap-oob="innerHTML"`     | Replaces only the children of the target element               |
| `hx-swap-oob="beforebegin"`   | Inserts before the target element                              |
| `hx-swap-oob="afterbegin"`    | Inserts as the first child of the target                       |
| `hx-swap-oob="beforeend"`     | Inserts as the last child of the target                        |
| `hx-swap-oob="afterend"`      | Inserts after the target element                               |
| `hx-swap-oob="delete"`        | Deletes the target element                                     |

The most common choices are `outerHTML` (replace the whole thing) and
`innerHTML` (replace only the contents, keeping the wrapper).

**The golden rule:** Every OOB element **must** have an `id`. Without an `id`,
HTMX has no way to find the matching element in the DOM, and the OOB element
will simply be discarded.

### 1.3 `hx-select` -- Filtering the Response on the Client

Sometimes the opposite problem arises: the server returns **more** HTML than
you need, and you want to use only a piece of it. The `hx-select` attribute
lets the client filter the server response using a CSS selector.

For example, imagine you have a `/dashboard` endpoint that returns a full page.
But you only want to refresh the statistics section:

```html
<button hx-get="/dashboard"
        hx-target="#stats"
        hx-swap="innerHTML"
        hx-select="#stats-content">
  Refresh Stats
</button>
```

When HTMX receives the full `/dashboard` response, it applies `hx-select` as a
filter. Only the element(s) matching `#stats-content` are extracted from the
response. Everything else is thrown away. The extracted content is then swapped
into `#stats` as usual.

This is useful when:

- You want to reuse an existing endpoint without creating a dedicated fragment endpoint.
- You are doing progressive enhancement and want the same URL to work for both full-page loads and HTMX partial updates.
- The server returns a full page but you only care about one section.

**`hx-select-oob`** is the companion to `hx-select`. It lets you select
additional elements from the response and swap them into different targets, much
like `hx-swap-oob` but driven from the client side rather than the server side.
We will stick with server-driven OOB in this chapter because it gives the
server more control and keeps the client-side markup simpler.

### 1.4 CSS Transitions with HTMX

HTMX automatically adds CSS classes during the swap lifecycle. You can hook
into these classes with CSS transitions to create smooth animations -- no
JavaScript required.

The three key classes are:

| Class              | When it is applied                                                         |
|--------------------|----------------------------------------------------------------------------|
| `.htmx-added`     | Applied to **new content** immediately when it is added to the DOM. Removed after one frame. |
| `.htmx-settling`  | Applied to the **target element** after the new content is added. Removed after the settle delay. |
| `.htmx-swapping`  | Applied to the **target element** before the old content is removed. Removed after the swap delay. |

The pattern for a **fade-in** effect:

1. Define the "starting state" for new elements: `opacity: 0`.
2. Define the "end state" with a transition: `opacity: 1; transition: opacity 0.3s`.
3. HTMX adds the element with `.htmx-added` (opacity: 0), then removes the class on the next frame, triggering the transition to opacity: 1.

```css
#task-list li {
  transition: opacity 0.3s ease-in;
}
#task-list li.htmx-added {
  opacity: 0;
}
```

The pattern for a **fade-out** effect on deletion:

1. Define the "leaving state" on `.htmx-swapping`: `opacity: 0; transition: opacity 0.5s`.
2. Tell HTMX to **wait** before performing the swap, giving the CSS transition time to complete: `hx-swap="outerHTML swap:500ms"`.

```css
.htmx-swapping {
  opacity: 0;
  transition: opacity 0.5s ease-out;
}
```

The `swap:500ms` modifier in `hx-swap` tells HTMX: "After adding the
`.htmx-swapping` class, wait 500 milliseconds before actually removing the old
content." This delay is what gives your CSS transition time to play out.

Without the delay, HTMX would remove the old content immediately, and you would
never see the fade-out -- the element would just vanish.

---

## 2. Code Walkthrough

We are going to enhance the Teamwork task board with three improvements:

1. A **task counter** that stays in sync with every mutation via OOB swaps.
2. **Form clearing** after successful task creation via OOB swaps.
3. **CSS transitions** for smooth task additions and deletions.

### 2.1 The Task Counter Component

First, we need a reusable function that renders the task counter. This function
will be called both when rendering the initial page and when building OOB
response fragments.

```gleam
fn task_counter(tasks: List(Task)) -> Element(t) {
  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  html.div([attribute.id("task-counter")], [
    element.text(
      int.to_string(done) <> " of " <> int.to_string(total) <> " tasks completed",
    ),
  ])
}
```

Two things matter here:

- The element has `attribute.id("task-counter")`. This `id` is the anchor that
  OOB swaps will use to find and replace this element in the DOM.
- The function takes the full task list as input so it can compute both the
  total count and the completed count. This means the counter is always derived
  from the current state -- there is no separate "counter state" to keep in
  sync.

### 2.2 The Home Page with Counter

Update the home page to include the task counter above the task list:

```gleam
fn home_page(tasks: List(Task)) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    task_counter(tasks),
    html.div([attribute.id("form-container")], [
      add_task_form([], []),
    ]),
    html.ul([attribute.id("task-list")], {
      list.map(tasks, task_item)
    }),
  ])
}
```

Notice that both the counter (`#task-counter`), the form container
(`#form-container`), and the task list (`#task-list`) have stable `id`
attributes. These IDs are the hooks that OOB swaps will use.

### 2.3 OOB Response for Task Creation

Here is the key change. When a user creates a task, the server returns not just
the new task item (for the main swap target), but also OOB elements for the
counter and the form:

```gleam
fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.find(form_data.values, fn(pair) { pair.0 == "title" })
    |> result.map(fn(pair) { pair.1 })
    |> result.unwrap("")

  case validate_task_title(title) {
    Ok(valid_title) -> {
      let task = Task(id: new_id(), title: valid_title, done: False)
      actor.send(ctx.tasks, AddTask(task))
      let tasks = actor.call(ctx.tasks, 1000, GetTasks)

      // Build the response with main content + OOB elements
      let response_html =
        element.fragment([
          // 1. Main content: the new task item
          //    This goes to hx-target="#task-list" via the normal swap
          task_item(task),

          // 2. OOB: Updated task counter
          //    Replaces the existing #task-counter anywhere on the page
          html.div(
            [attribute.id("task-counter"), hx.swap_oob(hx.OuterHTML, None)],
            [
              element.text(
                int.to_string(list.count(tasks, fn(t) { t.done }))
                <> " of "
                <> int.to_string(list.length(tasks))
                <> " tasks completed",
              ),
            ],
          ),

          // 3. OOB: Fresh form (clears the input)
          //    Replaces the contents of #form-container
          html.div(
            [attribute.id("form-container"), hx.swap_oob(hx.InnerHTML, None)],
            [add_task_form([], [])],
          ),
        ])

      wisp.html_response(
        element.to_string(response_html),
        201,
      )
    }
    Error(errors) -> {
      // Validation failed -- return the form with errors (no OOB needed)
      let response_html = add_task_form(errors, [#("title", title)])
      wisp.html_response(
        element.to_string(response_html),
        422,
      )
    }
  }
}
```

Let's trace what happens when this response reaches the browser:

```
  Server response body (simplified):
  ┌──────────────────────────────────────────────────────┐
  │  <li id="task-6">Fix bug #42 [Delete] [Toggle]</li> │
  │  <div id="task-counter" hx-swap-oob="outerHTML">     │
  │    2 of 6 tasks completed                            │
  │  </div>                                              │
  │  <div id="form-container" hx-swap-oob="innerHTML">   │
  │    <form hx-post="/tasks" ...>...</form>              │
  │  </div>                                              │
  └──────────────────────────────────────────────────────┘

  HTMX processing:
  1. Scan response for hx-swap-oob elements → found 2
  2. Pull out <div id="task-counter" hx-swap-oob="outerHTML">
     → Find #task-counter in the DOM → replace it entirely
  3. Pull out <div id="form-container" hx-swap-oob="innerHTML">
     → Find #form-container in the DOM → replace its children
  4. Remaining content: <li id="task-6">...</li>
     → Swap into hx-target="#task-list" using hx-swap="beforeend"
```

**`element.fragment`** is the Lustre function that groups multiple elements
without wrapping them in a container tag. When serialised with
`element.to_string`, a fragment simply concatenates its children. This is
essential for OOB responses because the OOB elements must be siblings in the
response, not nested inside a wrapper div.

**`hx.swap_oob(hx.OuterHTML, None)`** is the `hx` library function that produces the
`hx-swap-oob="outerHTML"` attribute. If you are not using the `hx` library, you
can use `attribute("hx-swap-oob", "outerHTML")` instead.

### 2.4 The Form and Its Trigger

The add task form needs to target the task list and use `beforeend` to append
new tasks at the bottom:

```gleam
fn add_task_form(
  errors: List(String),
  values: List(#(String, String)),
) -> Element(t) {
  let title_value =
    list.find(values, fn(pair) { pair.0 == "title" })
    |> result.map(fn(pair) { pair.1 })
    |> result.unwrap("")

  html.form(
    [
      hx.post("/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.input([
          attribute("type", "text"),
          attribute("name", "title"),
          attribute("placeholder", "What needs to be done?"),
          attribute.value(title_value),
        ]),
        html.button([attribute("type", "submit")], [
          element.text("Add Task"),
        ]),
      ]),
      // Display validation errors if any
      ..list.map(errors, fn(error) {
        html.p([attribute.class("error")], [element.text(error)])
      })
    ],
  )
}
```

When this form is submitted:

1. HTMX sends a POST to `/tasks`.
2. The server responds with the new `<li>` (main content) plus OOB elements.
3. HTMX appends the `<li>` to `#task-list` (because of `hx-swap="beforeend"`).
4. HTMX swaps the OOB counter into `#task-counter`.
5. HTMX swaps the OOB form into `#form-container`, effectively clearing the input.

All from a single HTTP request.

### 2.5 OOB on Delete

The same pattern applies to deletion. When a task is deleted, we need to remove
the task from the list and update the counter:

```gleam
fn delete_task(task_id: String, ctx: Context) -> wisp.Response {
  actor.send(ctx.tasks, RemoveTask(task_id))
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  // Return empty string for the main swap (removes the task item)
  // plus an OOB counter update
  let response_html =
    element.fragment([
      // OOB: Updated counter
      html.div(
        [attribute.id("task-counter"), hx.swap_oob(hx.OuterHTML, None)],
        [
          element.text(
            int.to_string(list.count(tasks, fn(t) { t.done }))
            <> " of "
            <> int.to_string(list.length(tasks))
            <> " tasks completed",
          ),
        ],
      ),
    ])

  wisp.html_response(
    element.to_string(response_html),
    200,
  )
}
```

The delete button on each task uses `outerHTML` with a swap delay for the
fade-out transition:

```gleam
fn task_item(task: Task) -> Element(t) {
  let done_class = case task.done {
    True -> "task-done"
    False -> ""
  }
  html.li([attribute.id("task-" <> task.id), attribute.class(done_class)], [
    html.span([attribute.class("task-title")], [element.text(task.title)]),
    html.button(
      [
        hx.delete("/tasks/" <> task.id),
        hx.target(hx.Selector("#task-" <> task.id)),
        attribute("hx-swap", "outerHTML swap:500ms"),
      ],
      [element.text("Delete")],
    ),
    html.button(
      [
        hx.patch("/tasks/" <> task.id <> "/toggle"),
        hx.target(hx.Selector("#task-" <> task.id)),
        hx.swap(hx.OuterHTML),
      ],
      [element.text(case task.done {
        True -> "Undo"
        False -> "Done"
      })],
    ),
  ])
}
```

The `attribute("hx-swap", "outerHTML swap:500ms")` tells HTMX to wait 500
milliseconds after adding the `.htmx-swapping` class before actually removing
the element. This gives the CSS transition time to fade the task out.

### 2.6 OOB on Toggle

Toggling a task's done status also needs to update the counter:

```gleam
fn toggle_task(task_id: String, ctx: Context) -> wisp.Response {
  actor.send(ctx.tasks, ToggleTask(task_id))
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let task =
    list.find(tasks, fn(t) { t.id == task_id })
    |> result.lazy_unwrap(fn() { panic as "Task not found" })

  let response_html =
    element.fragment([
      // Main content: the updated task item
      task_item(task),

      // OOB: Updated counter
      html.div(
        [attribute.id("task-counter"), hx.swap_oob(hx.OuterHTML, None)],
        [
          element.text(
            int.to_string(list.count(tasks, fn(t) { t.done }))
            <> " of "
            <> int.to_string(list.length(tasks))
            <> " tasks completed",
          ),
        ],
      ),
    ])

  wisp.html_response(
    element.to_string(response_html),
    200,
  )
}
```

### 2.7 Extracting the OOB Counter

You may have noticed that we are duplicating the counter HTML in every handler.
That is a code smell. Let's extract a helper function that wraps the counter
with the OOB attribute:

```gleam
fn task_counter_oob(tasks: List(Task)) -> Element(t) {
  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  html.div(
    [attribute.id("task-counter"), hx.swap_oob(hx.OuterHTML, None)],
    [
      element.text(
        int.to_string(done) <> " of " <> int.to_string(total) <> " tasks completed",
      ),
    ],
  )
}
```

Now every handler that modifies tasks can simply include `task_counter_oob(tasks)`
in its response fragment. The counter logic lives in one place. If you decide to
change the counter's markup or add a progress bar, you change it once.

### 2.8 Using `hx-select`

Here is a practical use of `hx-select` in the Teamwork app. Suppose you want a
"Refresh" button that reloads the entire task list from the home page endpoint
without creating a dedicated fragment route:

```gleam
html.button(
  [
    hx.get("/"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    attribute("hx-select", "#task-list li"),
  ],
  [element.text("Refresh")],
)
```

When clicked:

1. HTMX sends a GET to `/` (the full home page).
2. The server returns the complete page HTML.
3. HTMX applies `hx-select="#task-list li"` -- it extracts only the `<li>`
   elements inside `#task-list` from the response.
4. Those `<li>` elements are swapped into the current `#task-list` using `innerHTML`.
5. Everything else in the response (the head, nav, counter, form) is discarded.

This is a pragmatic approach. You avoid creating a separate `/tasks/list`
endpoint. The trade-off is that the server does more work than necessary
(rendering a full page when the client only needs a fragment), but for many
applications that cost is negligible.

### 2.9 CSS Transitions

Add these styles to your `priv/static/css/style.css` to make task additions
and deletions feel smooth:

```css
/* ── Transition: New tasks fade in ────────────────────── */

#task-list li {
  transition: opacity 0.3s ease-in;
}

#task-list li.htmx-added {
  opacity: 0;
}

/*
  When .htmx-added is removed (after one animation frame),
  the transition kicks in, smoothly changing opacity from 0 to 1.
  No .htmx-settling rule is needed here -- the default state
  (opacity: 1, from the normal #task-list li rule) is the end state.
*/

/* ── Transition: Deleted tasks fade out ───────────────── */

#task-list li.htmx-swapping {
  opacity: 0;
  transition: opacity 0.5s ease-out;
}

/*
  This requires hx-swap="outerHTML swap:500ms" on the delete button.
  The swap:500ms delay gives the transition time to play out before
  HTMX removes the element from the DOM.
*/

/* ── Transition: Counter updates pulse ────────────────── */

#task-counter {
  transition: color 0.3s ease;
}

#task-counter.htmx-settling {
  color: #4a90d9;
}

/*
  When the counter is swapped via OOB, it briefly flashes blue
  to draw the user's attention to the change, then fades back
  to the default colour.
*/
```

The lifecycle of these classes during a swap:

```
  Delete button clicked on "Buy groceries"
  ─────────────────────────────────────────────────

  Time 0ms:    HTMX adds .htmx-swapping to <li id="task-3">
               CSS transition starts: opacity 1 → 0 over 500ms

  Time 500ms:  HTMX removes <li id="task-3"> from the DOM
               (because of swap:500ms delay)

  Time 500ms:  HTMX swaps in the response (empty string → element gone)
               OOB counter element is processed:
                 - New #task-counter is inserted
                 - .htmx-settling is added to new #task-counter
                 - Counter text flashes blue

  Time 800ms:  .htmx-settling is removed from #task-counter
               Counter colour fades back to default
```

---

## 3. Full Code Listing

Here is the complete updated module with OOB support and CSS transitions.

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
```

### `src/teamwork.gleam`

```gleam
import gleam/bool
import gleam/erlang/process
import gleam/http
import gleam/int
import gleam/list
import gleam/option.{None, Some}
import gleam/otp/actor
import gleam/result
import gleam/string
import hx
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist

// ── Types ────────────────────────────────────────────────

pub type Task {
  Task(id: String, title: String, done: Bool)
}

pub type TaskMessage {
  AddTask(Task)
  RemoveTask(String)
  ToggleTask(String)
  GetTasks(reply_to: process.Subject(List(Task)))
}

pub type Context {
  Context(tasks: process.Subject(TaskMessage))
}

// ── Main ─────────────────────────────────────────────────

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let assert Ok(tasks_actor) =
    actor.new([])
    |> actor.on_message(handle_task_message)
    |> actor.start
    |> result_map_started
  let ctx = Context(tasks: tasks_actor)

  let assert Ok(_) =
    wisp_mist.handler(fn(req) { handle_request(req, ctx) }, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

// ── Actor ────────────────────────────────────────────────

fn handle_task_message(
  tasks: List(Task),
  message: TaskMessage,
) -> actor.Next(List(Task), TaskMessage) {
  case message {
    AddTask(task) -> actor.continue(list.append(tasks, [task]))
    RemoveTask(id) ->
      actor.continue(list.filter(tasks, fn(t) { t.id != id }))
    ToggleTask(id) ->
      actor.continue(
        list.map(tasks, fn(t) {
          case t.id == id {
            True -> Task(..t, done: bool.negate(t.done))
            False -> t
          }
        }),
      )
    GetTasks(reply_to) -> {
      process.send(reply_to, tasks)
      actor.continue(tasks)
    }
  }
}

fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}

// ── ID generation ────────────────────────────────────────

fn new_id() -> String {
  int.to_string(int.random(1_000_000))
}

// ── Validation ───────────────────────────────────────────

fn validate_task_title(title: String) -> Result(String, List(String)) {
  let trimmed = string.trim(title)
  case trimmed {
    "" -> Error(["Title cannot be empty"])
    t if string.length(t) > 100 ->
      Error(["Title must be 100 characters or less"])
    t -> Ok(t)
  }
}

// ── Routing ──────────────────────────────────────────────

fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req), req.method {
    [], http.Get -> {
      let tasks = actor.call(ctx.tasks, 1000, GetTasks)
      let page = layout("Teamwork", home_page(tasks))
      wisp.html_response(
        element.to_document_string(page),
        200,
      )
    }
    ["tasks"], http.Post -> create_task(req, ctx)
    ["tasks", id], http.Delete -> delete_task(id, ctx)
    ["tasks", id, "toggle"], http.Patch -> toggle_task(id, ctx)
    _, _ -> wisp.not_found()
  }
}

fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}

// ── Handlers ─────────────────────────────────────────────

fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.find(form_data.values, fn(pair) { pair.0 == "title" })
    |> result.map(fn(pair) { pair.1 })
    |> result.unwrap("")

  case validate_task_title(title) {
    Ok(valid_title) -> {
      let task = Task(id: new_id(), title: valid_title, done: False)
      actor.send(ctx.tasks, AddTask(task))
      let tasks = actor.call(ctx.tasks, 1000, GetTasks)

      let response_html =
        element.fragment([
          task_item(task),
          task_counter_oob(tasks),
          html.div(
            [attribute.id("form-container"), hx.swap_oob(hx.InnerHTML, None)],
            [add_task_form([], [])],
          ),
        ])

      wisp.html_response(
        element.to_string(response_html),
        201,
      )
    }
    Error(errors) -> {
      let response_html = add_task_form(errors, [#("title", title)])
      wisp.html_response(
        element.to_string(response_html),
        422,
      )
    }
  }
}

fn delete_task(task_id: String, ctx: Context) -> wisp.Response {
  actor.send(ctx.tasks, RemoveTask(task_id))
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let response_html = element.fragment([task_counter_oob(tasks)])

  wisp.html_response(
    element.to_string(response_html),
    200,
  )
}

fn toggle_task(task_id: String, ctx: Context) -> wisp.Response {
  actor.send(ctx.tasks, ToggleTask(task_id))
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let task =
    list.find(tasks, fn(t) { t.id == task_id })
    |> result.lazy_unwrap(fn() { panic as "Task not found" })

  let response_html =
    element.fragment([
      task_item(task),
      task_counter_oob(tasks),
    ])

  wisp.html_response(
    element.to_string(response_html),
    200,
  )
}

// ── Components ───────────────────────────────────────────

fn layout(page_title: String, content: Element(t)) -> Element(t) {
  html.html([], [
    html.head([], [
      html.meta([attribute("charset", "UTF-8")]),
      html.meta([
        attribute("name", "viewport"),
        attribute("content", "width=device-width, initial-scale=1.0"),
      ]),
      html.title([], page_title),
      html.link([
        attribute("rel", "stylesheet"),
        attribute("href", "/static/css/style.css"),
      ]),
      html.script(
        [attribute("src", "https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
    ]),
    html.body([], [content]),
  ])
}

fn home_page(tasks: List(Task)) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    task_counter(tasks),
    html.div([attribute.id("form-container")], [
      add_task_form([], []),
    ]),
    html.ul([attribute.id("task-list")], {
      list.map(tasks, task_item)
    }),
  ])
}

fn task_counter(tasks: List(Task)) -> Element(t) {
  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  html.div([attribute.id("task-counter"), attribute.class("counter")], [
    element.text(
      int.to_string(done) <> " of " <> int.to_string(total) <> " tasks completed",
    ),
  ])
}

/// OOB version of the counter -- includes hx-swap-oob so it can ride
/// along with any response and automatically replace the existing counter.
fn task_counter_oob(tasks: List(Task)) -> Element(t) {
  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  html.div(
    [
      attribute.id("task-counter"),
      attribute.class("counter"),
      hx.swap_oob(hx.OuterHTML, None),
    ],
    [
      element.text(
        int.to_string(done)
        <> " of "
        <> int.to_string(total)
        <> " tasks completed",
      ),
    ],
  )
}

fn add_task_form(
  errors: List(String),
  values: List(#(String, String)),
) -> Element(t) {
  let title_value =
    list.find(values, fn(pair) { pair.0 == "title" })
    |> result.map(fn(pair) { pair.1 })
    |> result.unwrap("")

  html.form(
    [
      hx.post("/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.input([
          attribute("type", "text"),
          attribute("name", "title"),
          attribute("placeholder", "What needs to be done?"),
          attribute.value(title_value),
          attribute.class("task-input"),
        ]),
        html.button(
          [attribute("type", "submit"), attribute.class("btn btn-primary")],
          [element.text("Add Task")],
        ),
      ]),
      ..list.map(errors, fn(error) {
        html.p([attribute.class("error")], [element.text(error)])
      })
    ],
  )
}

fn task_item(task: Task) -> Element(t) {
  let done_class = case task.done {
    True -> "task-item task-done"
    False -> "task-item"
  }
  html.li([attribute.id("task-" <> task.id), attribute.class(done_class)], [
    html.span([attribute.class("task-title")], [element.text(task.title)]),
    html.div([attribute.class("task-actions")], [
      html.button(
        [
          hx.patch("/tasks/" <> task.id <> "/toggle"),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
          attribute.class("btn btn-small"),
        ],
        [element.text(case task.done {
          True -> "Undo"
          False -> "Done"
        })],
      ),
      html.button(
        [
          hx.delete("/tasks/" <> task.id),
          hx.target(hx.Selector("#task-" <> task.id)),
          attribute("hx-swap", "outerHTML swap:500ms"),
          attribute.class("btn btn-small btn-danger"),
        ],
        [element.text("Delete")],
      ),
    ]),
  ])
}
```

### `priv/static/css/style.css`

```css
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
  padding: 2rem;
}

.container {
  max-width: 800px;
  margin: 0 auto;
}

h1 {
  color: #16213e;
  margin-bottom: 1rem;
}

/* ── Counter ──────────────────────────────────────────── */

.counter {
  padding: 0.75rem 1rem;
  background: #e8ecf1;
  border-radius: 6px;
  margin-bottom: 1rem;
  font-weight: 600;
  color: #333;
  transition: color 0.3s ease, background-color 0.3s ease;
}

.counter.htmx-settling {
  color: #4a90d9;
  background-color: #dbe8f7;
}

/* ── Form ─────────────────────────────────────────────── */

.form-group {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.task-input {
  flex: 1;
  padding: 0.5rem 0.75rem;
  border: 2px solid #d0d0d8;
  border-radius: 6px;
  font-size: 1rem;
  transition: border-color 0.2s;
}

.task-input:focus {
  outline: none;
  border-color: #4a90d9;
}

.error {
  color: #d32f2f;
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
}

/* ── Buttons ──────────────────────────────────────────── */

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  font-weight: 600;
  transition: background-color 0.2s;
}

.btn-primary {
  background-color: #4a90d9;
  color: white;
}

.btn-primary:hover {
  background-color: #3a7bc8;
}

.btn-small {
  padding: 0.25rem 0.5rem;
  font-size: 0.75rem;
}

.btn-danger {
  background-color: #e74c3c;
  color: white;
}

.btn-danger:hover {
  background-color: #c0392b;
}

/* ── Task List ────────────────────────────────────────── */

#task-list {
  list-style: none;
}

.task-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1rem;
  background: white;
  border-radius: 6px;
  margin-bottom: 0.5rem;
  border: 1px solid #e0e0e8;
  transition: opacity 0.3s ease-in, transform 0.3s ease-in;
}

.task-item.task-done .task-title {
  text-decoration: line-through;
  color: #999;
}

.task-actions {
  display: flex;
  gap: 0.25rem;
}

/* ── HTMX Transition: Fade in new tasks ───────────────── */

#task-list li.htmx-added {
  opacity: 0;
  transform: translateY(-10px);
}

/*
  When .htmx-added is removed on the next animation frame,
  the element transitions from opacity:0/translateY(-10px)
  to the default opacity:1/translateY(0) defined on .task-item.
*/

/* ── HTMX Transition: Fade out deleted tasks ──────────── */

#task-list li.htmx-swapping {
  opacity: 0;
  transform: translateX(30px);
  transition: opacity 0.5s ease-out, transform 0.5s ease-out;
}

/*
  This requires hx-swap="outerHTML swap:500ms" on the delete button.
  The 500ms delay gives the fade-out time to complete before
  HTMX removes the element from the DOM.
*/
```

---

## 4. Exercise

Put what you have learned into practice with the following tasks.

### Task 1 -- OOB Counter on Delete

Verify that deleting a task updates the task counter. If you have been following
along, this should already work. Test it:

1. Add three tasks.
2. Delete one.
3. Confirm the counter changes from "0 of 3 tasks completed" to "0 of 2 tasks completed" without a page reload.

If the counter does not update, check that your `delete_task` handler includes
`task_counter_oob(tasks)` in its response fragment.

### Task 2 -- OOB Counter on Toggle

Toggle a task's done status and verify the counter updates:

1. Add two tasks.
2. Click "Done" on one of them.
3. Confirm the counter changes from "0 of 2 tasks completed" to "1 of 2 tasks completed".
4. Click "Undo" and confirm it goes back to "0 of 2".

### Task 3 -- CSS Transitions

Test the transitions:

1. Add a task. It should fade in smoothly (opacity 0 to 1, sliding down slightly).
2. Delete a task. It should fade out (opacity 1 to 0, sliding right) over half a second.
3. If the fade-out is not visible, check that your delete button has `hx-swap="outerHTML swap:500ms"` (not just `hx-swap="outerHTML"`).

### Task 4 -- Progress Bar

Add a visual progress bar that shows what percentage of tasks are completed.
It should update via OOB after every action (add, delete, toggle).

Requirements:

- Create a `progress_bar` function that renders a `<div id="progress-bar">` containing an inner `<div>` whose width is set to the completion percentage.
- Create a `progress_bar_oob` function (like `task_counter_oob`) that includes the `hx-swap-oob` attribute.
- Include the OOB progress bar in every handler response that modifies tasks.
- Add CSS that animates the width change smoothly.

### Task 5 -- Refresh with `hx-select`

Add a "Refresh" button to the page that reloads the task list from the full
home page endpoint using `hx-select`:

1. Add a button with `hx-get="/"`.
2. Set `hx-target="#task-list"` and `hx-swap="innerHTML"`.
3. Add `hx-select="#task-list li"` to extract only the task items from the full page response.
4. Test it: open the app in two browser tabs, add a task in one tab, then click "Refresh" in the other. The new task should appear.

---

## 5. Exercise Solution Hints

Try to solve each exercise on your own before reading these hints.

### Hint for Task 4 -- Progress Bar

The progress bar component:

```gleam
fn progress_bar(tasks: List(Task)) -> Element(t) {
  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  let percentage = case total {
    0 -> 0
    _ -> done * 100 / total
  }

  html.div([attribute.id("progress-bar"), attribute.class("progress-bar")], [
    html.div(
      [
        attribute.class("progress-fill"),
        attribute("style", "width: " <> int.to_string(percentage) <> "%"),
      ],
      [],
    ),
  ])
}
```

The OOB version:

```gleam
fn progress_bar_oob(tasks: List(Task)) -> Element(t) {
  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  let percentage = case total {
    0 -> 0
    _ -> done * 100 / total
  }

  html.div(
    [
      attribute.id("progress-bar"),
      attribute.class("progress-bar"),
      hx.swap_oob(hx.OuterHTML, None),
    ],
    [
      html.div(
        [
          attribute.class("progress-fill"),
          attribute("style", "width: " <> int.to_string(percentage) <> "%"),
        ],
        [],
      ),
    ],
  )
}
```

The CSS:

```css
.progress-bar {
  width: 100%;
  height: 8px;
  background: #e0e0e8;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 1rem;
}

.progress-fill {
  height: 100%;
  background: #4a90d9;
  border-radius: 4px;
  transition: width 0.4s ease;
}
```

The key insight is that the `transition: width 0.4s ease` on `.progress-fill`
handles the animation automatically. When the OOB swap replaces the progress
bar, the browser sees the width change and animates it. You do not need any
HTMX transition classes for this -- standard CSS transitions on the `style`
attribute work fine.

Include both `task_counter_oob(tasks)` and `progress_bar_oob(tasks)` in every
handler that modifies tasks. There is no limit to the number of OOB elements
in a single response.

### Hint for Task 5 -- Refresh with `hx-select`

```gleam
html.button(
  [
    hx.get("/"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    attribute("hx-select", "#task-list li"),
    attribute.class("btn"),
  ],
  [element.text("Refresh")],
)
```

Place this button somewhere visible on the page, such as next to the heading
or above the task list. When clicked, it fetches the full page, extracts only
the `<li>` elements from `#task-list`, and swaps them into the current
`#task-list`.

Note: If the server has no tasks, the response will contain an empty `<ul>`,
and `hx-select="#task-list li"` will find nothing. The `innerHTML` swap will
then clear the task list, which is the correct behaviour.

---

## 6. Key Takeaways

1. **Standard HTMX is one request, one target.** A single `hx-post` or
   `hx-get` updates a single `hx-target` element. For many interactions this
   is all you need. But when one user action should update multiple unrelated
   parts of the page, you need OOB swaps.

2. **Out-of-Band (OOB) swaps let a single response update multiple targets.**
   Any element in the response with `hx-swap-oob` and an `id` is pulled out
   and swapped into the matching DOM element, independent of the main target.
   The rest of the response goes to `hx-target` as usual.

3. **OOB elements must have an `id`.** The `id` is how HTMX matches the
   response element to the DOM element. No `id`, no match, no swap.

4. **OOB supports all swap strategies.** Use `hx-swap-oob="outerHTML"` to
   replace the entire element (most common), `hx-swap-oob="innerHTML"` to
   replace only its children, or any other strategy like `beforeend` or
   `afterbegin`.

5. **`element.fragment` groups elements without a wrapper.** In Lustre,
   `element.fragment([a, b, c])` serialises to the concatenation of `a`, `b`,
   and `c` with no surrounding tag. This is essential for building responses
   that contain both main content and OOB elements as siblings.

6. **Extract OOB helpers to avoid duplication.** Every handler that modifies
   tasks should include the same OOB counter update. Write a single
   `task_counter_oob` function and call it everywhere.

7. **`hx-select` filters the response on the client side.** It applies a CSS
   selector to the server's response and discards everything that does not
   match. Useful for reusing full-page endpoints without creating dedicated
   fragment routes.

8. **HTMX provides CSS transition hooks.** `.htmx-added` is applied to new
   content on insertion (and removed after one frame), `.htmx-swapping` is
   applied before old content is removed, and `.htmx-settling` is applied
   during the settle phase. You combine these with CSS `transition` rules for
   smooth animations.

9. **Use `swap:` timing modifiers for fade-out effects.** The `swap:500ms`
   modifier in `hx-swap="outerHTML swap:500ms"` tells HTMX to wait 500ms
   before removing the old content. This gives your `.htmx-swapping` CSS
   transition time to play out. Without the delay, the element vanishes
   instantly and no animation is visible.

10. **OOB is a server-driven pattern.** The server decides what needs to
    update and includes the appropriate OOB elements. The client does not need
    to know which parts of the page are coupled to which actions. This keeps
    the coupling logic centralised on the server, which aligns perfectly with
    HTMX's philosophy of "HTML as the engine of application state."

---

## What's Next

In [Chapter 10](10-lists-filters-search.md), we will add **list filtering and search** -- letting users
filter tasks by status and search by title with instant results. We will use
`hx-get` with query parameters, active search that fires on keyup with a
debounce, and combine multiple filter controls into a single coordinated UI.
The Teamwork board will feel much more usable with large task lists.
