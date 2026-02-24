# Chapter 10 -- Lists, Filters, and Search

**Phase:** Intermediate
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapter 9 introduced OOB swaps and CSS transitions. The task board can now add, delete, and toggle tasks with multi-area updates.

---

## Learning Objectives

By the end of this chapter you will be able to:

- Build a live search input that filters tasks as the user types, without a full page reload.
- Explain what debouncing is and why it matters for search UX.
- Use `hx-trigger` with the `delay` modifier to debounce requests.
- Use `hx-include` to send values from elements outside the triggering element.
- Use `hx-push-url` to update the browser's address bar, making filtered views bookmarkable and back-button friendly.
- Parse query parameters on the server with `wisp.get_query`.
- Filter lists in Gleam using `list.filter` and `list.key_find`.

---

## 1. Theory

### 1.1 Live Search

Traditional search works like this: the user types a query, clicks a "Search"
button, and the page reloads with results. It works, but it feels sluggish.
Every search requires a deliberate action and a full round-trip that replaces
the entire page.

Live search (sometimes called "instant search" or "search-as-you-type") is
different. As the user types, results update in real time. There is no submit
button, no page reload. The feedback loop is tight: type a few characters, see
the results narrow. Delete a character, see the results expand. It feels
responsive and modern.

With a JavaScript framework, building this requires setting up event listeners,
managing a request queue, cancelling stale requests, and manually updating the
DOM. With HTMX, it requires a single HTML attribute:

```
hx-trigger="keyup changed delay:300ms"
```

That is the entire client-side implementation. The server still does all the
real work -- receiving a query, filtering the data, rendering the HTML, and
sending it back. HTMX just wires the input event to the HTTP request and swaps
the response into the page.

### 1.2 Debouncing

If you fire an HTTP request on every single keystroke, a user typing "design"
would trigger six requests in rapid succession: `d`, `de`, `des`, `desi`,
`desig`, `design`. Most of those intermediate requests are wasted -- by the
time the server responds to the `d` request, the user has already typed three
more characters and the results are stale.

**Debouncing** solves this. The idea is simple: wait for the user to stop
typing before sending the request. In HTMX, you add `delay:300ms` to the
`hx-trigger` attribute:

```
hx-trigger="keyup changed delay:300ms"
```

This means: "When a `keyup` event fires, start a 300-millisecond timer. If
another `keyup` fires before the timer expires, reset the timer. Only send
the request when 300 milliseconds have passed with no new keyup events."

The `changed` modifier adds one more optimization: only fire if the input's
value actually changed. This prevents duplicate requests when the user presses
an arrow key or Shift without altering the text.

Three hundred milliseconds is a good default. It is short enough that the user
perceives the response as instant, but long enough to batch a burst of
keystrokes into a single request. For slower servers or heavier queries, you
might increase it to 500ms.

### 1.3 `hx-include` -- Sending Values from Other Elements

By default, when HTMX sends a request from an element, it includes only that
element's value (if it is a form input) or the values of inputs inside it (if
it is a form). But what happens when you have a search box *and* a set of
filter tabs, and you want both values to travel together in every request?

This is what `hx-include` solves. It accepts a CSS selector and tells HTMX:
"When you send this request, also include the values of any elements matching
this selector."

For example, if your search box has `name="q"` and your filter button has
`name="status"`, the search box can include the filter value:

```
hx-include="[name='status']"
```

And the filter button can include the search value:

```
hx-include="[name='q']"
```

This way, no matter which element triggers the request, both `q` and `status`
are sent as query parameters. The server always sees the complete filter state.

### 1.4 `hx-push-url` -- Bookmarkable Filter State

When the user searches for "design" and filters by "active" tasks, the page
shows exactly the right results. But if they copy the URL and send it to a
colleague, the colleague sees the unfiltered page. The URL in the browser's
address bar has not changed -- it still says `/tasks`.

`hx-push-url="true"` fixes this. When HTMX completes a request, it updates the
browser's URL bar to match the request URL. So after the user types "design"
with the "active" filter, the URL becomes:

```
/tasks?q=design&status=active
```

This gives you three things for free:

1. **Bookmarkability.** The user can bookmark the filtered view and return to it later.
2. **Shareability.** The URL captures the full filter state, so sharing it sends the recipient to the exact same view.
3. **Back button support.** The browser's history stack records each URL change. Pressing Back restores the previous filter state.

On the server side, you need to handle these query parameters in your route
handler so that a direct visit to `/tasks?q=design&status=active` renders the
correct filtered view. This is called "server-side rendering of URL state," and
it is one of the key advantages of the hypermedia approach: the URL is the
source of truth, and the server always knows how to render it.

### 1.5 Query Parameters in Wisp

Wisp provides a simple function for reading query parameters:

```gleam
wisp.get_query(req)
```

This returns a `List(#(String, String))` -- a list of key-value tuples. For the
URL `/tasks?q=design&status=active`, it returns:

```gleam
[#("q", "design"), #("status", "active")]
```

You can extract individual values with `list.key_find`:

```gleam
let search_query = list.key_find(query_params, "q") |> result.unwrap("")
```

`list.key_find` returns a `Result(String, Nil)`. If the key is present, you get
`Ok(value)`. If not, you get `Error(Nil)`. Piping through `result.unwrap("")`
gives you the value if it exists, or an empty string as a default.

### 1.6 Filter Patterns

There are several common patterns for filtering lists in a web application:

- **Filter tabs** (All / Active / Done): A row of buttons or links. Only one is active at a time. Clicking one replaces the list with the filtered results.
- **Search box**: A text input that filters by keyword. Results update as the user types.
- **Combined filters**: Both tabs and search work together. The user can search within "Active" tasks, for example.
- **Sort controls**: A dropdown or set of buttons that change the ordering without changing which items are shown.

In this chapter we will implement all of these for the Teamwork task board.
The key insight is that all of these controls ultimately produce the same thing:
query parameters that the server uses to filter and sort the task list.

---

## 2. Code Walkthrough

We are going to add three features to the Teamwork task board:

1. A **search box** that filters tasks by title as the user types.
2. **Filter tabs** (All / Active / Done) that filter tasks by completion status.
3. **URL synchronization** so that filters are bookmarkable and the back button works.

These three features work together. Searching while a filter is active narrows
the results further. Changing the filter preserves the search query. The URL
always reflects the current state.

### 2.1 The Search Box with Debounce

Let's start with the search input. This is a single `<input>` element with
several HTMX attributes:

```gleam
fn search_box(current_query: String) -> Element(t) {
  html.input([
    attribute.type_("search"),
    attribute.name("q"),
    attribute.value(current_query),
    attribute.placeholder("Search tasks..."),
    attribute.class("search-input"),
    hx.get("/tasks"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    hx.trigger([
      hx.keyup()
      |> hx.with_changed
      |> hx.with_delay(duration.milliseconds(300)),
    ]),
    hx.push_url(True),
    hx.include(hx.Selector("[name='status']")),
  ])
}
```

Let's walk through each attribute:

- **`attribute.type_("search")`** -- Uses the HTML5 `search` input type. Browsers render this with a small "x" button to clear the field, which is a nice UX touch. Note the trailing underscore on `type_` because `type` is a reserved word in Gleam.

- **`attribute.name("q")`** -- The parameter name that will be sent with the request. HTMX reads the input's `name` and `value` and sends them as query parameters.

- **`attribute.value(current_query)`** -- Pre-fills the input with the current search query. This is important for two reasons: (1) when the page loads from a bookmarked URL like `/tasks?q=design`, the search box shows "design", and (2) after a filter change, the search box retains its value.

- **`attribute.placeholder("Search tasks...")`** -- Placeholder text shown when the input is empty.

- **`hx.get("/tasks")`** -- When triggered, send a GET request to `/tasks`.

- **`hx.target(hx.Selector("#task-list"))`** -- Swap the response into the element with `id="task-list"`.

- **`hx.swap(hx.InnerHTML)`** -- Replace the inner HTML of the target.

- **`hx.trigger`** -- Fire on keyup events, but only if the value changed, and debounce by 300 milliseconds. The `hx` library uses a typed API: `hx.keyup()` creates the event, `hx.with_changed` adds the `changed` modifier, and `hx.with_delay(duration.milliseconds(300))` adds the debounce. These are chained with Gleam's pipe operator.

- **`hx.push_url(True)`** -- Update the browser's URL bar after the request succeeds.

- **`hx.include(hx.Selector("[name='status']"))`** -- Also include the value of any element with `name="status"` (our filter tabs). This ensures the status filter is sent along with the search query.

### 2.2 Filter Tabs

The filter tabs are a row of buttons. Each button has a `name` of `"status"` and
a `value` of `"all"`, `"active"`, or `"done"`. Clicking a button sends a GET
request with that status, along with the current search query:

```gleam
fn filter_tabs(current_status: String) -> Element(t) {
  html.div([attribute.class("filter-tabs")], [
    filter_tab("all", "All", current_status),
    filter_tab("active", "Active", current_status),
    filter_tab("done", "Done", current_status),
  ])
}

fn filter_tab(
  value: String,
  label: String,
  current: String,
) -> Element(t) {
  let is_active = value == current
  let class = case is_active {
    True -> "tab active"
    False -> "tab"
  }
  html.button([
    attribute.class(class),
    attribute.name("status"),
    attribute.value(value),
    hx.get("/tasks"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    hx.include(hx.Selector("[name='q']")),
    hx.push_url(True),
  ], [element.text(label)])
}
```

A few things to notice:

- **Active state.** The current tab gets the CSS class `"tab active"`, which we style differently so the user can see which filter is selected. The comparison `value == current` returns a `Bool`, and we use `case` to choose the appropriate class string.

- **`hx.include(hx.Selector("[name='q']"))`** -- This is the mirror of the search box's `hx-include`. The filter button includes the search box's value so that the search query is not lost when the user changes the status filter.

- **`attribute.name("status")` and `attribute.value(value)`** -- HTMX reads these to send `status=active` (or whichever tab was clicked) as a query parameter.

- **No `hx-trigger`** -- Buttons trigger on click by default, which is exactly what we want.

### 2.3 The Toolbar

Let's compose the search box and filter tabs into a toolbar:

```gleam
fn task_toolbar(
  current_query: String,
  current_status: String,
) -> Element(t) {
  html.div([attribute.class("task-toolbar")], [
    search_box(current_query),
    filter_tabs(current_status),
  ])
}
```

### 2.4 Server-Side Filtering

On the server side, we need to read the query parameters, filter the task list,
and return the filtered HTML. Here is the updated route handler:

```gleam
fn get_tasks(req: wisp.Request, ctx: Context) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let search_query =
    list.key_find(query_params, "q")
    |> result.unwrap("")

  let status_filter =
    list.key_find(query_params, "status")
    |> result.unwrap("all")

  // Fetch all tasks from the actor
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  // Apply filters
  let filtered =
    tasks
    |> filter_by_status(status_filter)
    |> filter_by_search(search_query)

  // Render the HTML
  let html = task_list_html(filtered)
  wisp.html_response(
    element.to_string(html),
    200,
  )
}
```

This function does four things:

1. **Reads query parameters.** `wisp.get_query(req)` returns all query parameters as a list of tuples. We extract `q` and `status`, defaulting to empty string and `"all"` respectively.

2. **Fetches all tasks.** We get the full list from the actor, just as in previous chapters.

3. **Applies filters.** The pipeline filters by status first, then by search query. The order does not matter for correctness, but filtering by status first usually reduces the list more aggressively, making the subsequent string search faster.

4. **Renders and returns HTML.** We convert the filtered list to HTML and send it back. Note that we use `element.to_string` (not `to_document_string`) because this is a partial response -- just the task list, not a full page.

### 2.5 The Filter Functions

The two filter functions are straightforward applications of `list.filter`:

```gleam
fn filter_by_status(
  tasks: List(Task),
  status: String,
) -> List(Task) {
  case status {
    "active" -> list.filter(tasks, fn(t) { bool.negate(t.done) })
    "done" -> list.filter(tasks, fn(t) { t.done })
    _ -> tasks
  }
}

fn filter_by_search(
  tasks: List(Task),
  query: String,
) -> List(Task) {
  case query {
    "" -> tasks
    q -> {
      let lower_q = string.lowercase(q)
      list.filter(tasks, fn(t) {
        string.contains(string.lowercase(t.title), lower_q)
      })
    }
  }
}
```

Let's look at these closely.

**`filter_by_status`** pattern-matches on the status string:
- `"active"` keeps only tasks where `t.done` is `False`. `bool.negate` from `gleam/bool` flips the boolean value.
- `"done"` keeps only tasks where `t.done` is `True`.
- Anything else (including `"all"`) returns the full list unchanged.

**`filter_by_search`** handles the search query:
- If the query is empty, return all tasks. This is the base case -- no search means no filtering.
- Otherwise, convert both the query and each task title to lowercase, then check if the title contains the query substring. `string.contains` does the heavy lifting.

We compute `string.lowercase(q)` once, outside the filter function, rather than
inside it. This avoids converting the query string on every iteration. A small
optimization, but good practice.

### 2.6 Rendering the Task List

The task list renderer needs to handle the empty case gracefully. When the
user's filters match no tasks, showing a blank area is confusing. A clear
"no results" message is much better:

```gleam
fn task_list_html(tasks: List(Task)) -> Element(t) {
  case tasks {
    [] ->
      html.p([attribute.class("empty-state")], [
        element.text("No tasks found."),
      ])
    tasks ->
      html.ul(
        [attribute.id("task-list-items")],
        list.map(tasks, task_item),
      )
  }
}
```

`list.map(tasks, task_item)` converts each `Task` into an `Element` using
the `task_item` function (which we built in earlier chapters). The result is a
list of `<li>` elements that become the children of the `<ul>`.

### 2.7 The Full Page vs. Partial Responses

There is an important distinction between two kinds of requests:

1. **Full page load.** The user navigates to `/tasks?q=design&status=active` directly (by typing the URL, clicking a bookmark, or refreshing the page). The server must return a complete HTML document: `<!doctype html>`, `<head>`, `<body>`, the toolbar, the task list, and everything else.

2. **HTMX partial request.** The user types in the search box or clicks a filter tab. HTMX sends a request to `/tasks?q=design&status=active`, but it only needs the task list fragment to swap into `#task-list`.

How do you tell the difference? HTMX sets a request header called `HX-Request`
with the value `true` on every request it makes. You can check for this header
to decide whether to return a full page or a fragment:

```gleam
fn handle_tasks(req: wisp.Request, ctx: Context) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let search_query =
    list.key_find(query_params, "q")
    |> result.unwrap("")

  let status_filter =
    list.key_find(query_params, "status")
    |> result.unwrap("all")

  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let filtered =
    tasks
    |> filter_by_status(status_filter)
    |> filter_by_search(search_query)

  let is_htmx_request =
    list.key_find(req.headers, "hx-request")
    |> result.unwrap("")
    == "true"

  case is_htmx_request {
    True -> {
      // HTMX request: return just the task list fragment
      let html = task_list_html(filtered)
      wisp.html_response(
        element.to_string(html),
        200,
      )
    }
    False -> {
      // Full page load: return the complete page
      let page = tasks_page(filtered, search_query, status_filter)
      wisp.html_response(
        element.to_document_string(page),
        200,
      )
    }
  }
}
```

The `tasks_page` function builds the full page layout with the toolbar and the
task list:

```gleam
fn tasks_page(
  tasks: List(Task),
  current_query: String,
  current_status: String,
) -> Element(t) {
  layout("Tasks -- Teamwork", [
    html.div([attribute.class("container")], [
      html.h1([], [element.text("Tasks")]),
      task_toolbar(current_query, current_status),
      html.div([attribute.id("task-list")], [
        task_list_html(tasks),
      ]),
    ]),
  ])
}
```

Notice that the `#task-list` div wraps the task list. This is the swap target
that HTMX replaces when the user searches or filters.

### 2.8 CSS for the Toolbar

Here is the CSS that makes the search box and filter tabs look polished:

```css
/* Task toolbar */
.task-toolbar {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

/* Search input */
.search-input {
  width: 100%;
  padding: 0.625rem 0.75rem;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 0.9375rem;
  color: #1a1a2e;
  background: white;
  transition: border-color 0.15s ease;
}

.search-input:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.1);
}

.search-input::placeholder {
  color: #999;
}

/* Filter tabs */
.filter-tabs {
  display: flex;
  gap: 0.5rem;
}

.tab {
  padding: 0.5rem 1rem;
  border: 1px solid #ddd;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  font-weight: 500;
  color: #555;
  transition: all 0.15s ease;
}

.tab:hover {
  background: #f5f5f5;
  border-color: #ccc;
}

.tab.active {
  background: #16213e;
  color: white;
  border-color: #16213e;
}

.tab.active:hover {
  background: #1a2745;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 2rem 1rem;
  color: #888;
  font-style: italic;
}
```

The `transition` properties on the search input and tab buttons create smooth
visual feedback. The focus styles on the search input use a subtle box-shadow to
indicate the active element without the browser's default outline. The
`.tab.active` style uses the same dark blue as the site heading, tying the
active filter visually to the brand.

### 2.9 Wiring It Up in the Router

Update your router to handle the tasks route:

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    [] -> home_page(req)
    ["tasks"] -> handle_tasks(req, ctx)
    ["tasks", "toggle", id] -> toggle_task(req, ctx, id)
    ["tasks", "delete", id] -> delete_task(req, ctx, id)
    _ -> not_found_page()
  }
}
```

The `["tasks"]` branch now points to `handle_tasks`, which reads query
parameters and returns either the full page or a filtered fragment depending on
whether the request comes from HTMX.

---

## 3. Full Code Listing

Here is the complete code for the search and filter functionality. This listing
shows all the pieces assembled into their respective modules.

### `src/teamwork/task.gleam`

The task type and filter functions:

```gleam
// src/teamwork/task.gleam

import gleam/bool
import gleam/list
import gleam/string

pub type Task {
  Task(id: String, title: String, done: Bool)
}

pub fn filter_by_status(
  tasks: List(Task),
  status: String,
) -> List(Task) {
  case status {
    "active" -> list.filter(tasks, fn(t) { bool.negate(t.done) })
    "done" -> list.filter(tasks, fn(t) { t.done })
    _ -> tasks
  }
}

pub fn filter_by_search(
  tasks: List(Task),
  query: String,
) -> List(Task) {
  case query {
    "" -> tasks
    q -> {
      let lower_q = string.lowercase(q)
      list.filter(tasks, fn(t) {
        string.contains(string.lowercase(t.title), lower_q)
      })
    }
  }
}
```

### `src/teamwork/views/tasks_view.gleam`

The view functions for rendering tasks, search, and filters:

```gleam
// src/teamwork/views/tasks_view.gleam

import gleam/list
import gleam/time/duration
import hx
import lustre/attribute
import lustre/element.{type Element}
import lustre/element/html
import teamwork/task.{type Task}

pub fn tasks_page(
  tasks: List(Task),
  current_query: String,
  current_status: String,
) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Tasks")]),
    task_toolbar(current_query, current_status),
    html.div([attribute.id("task-list")], [
      task_list_html(tasks),
    ]),
  ])
}

fn task_toolbar(
  current_query: String,
  current_status: String,
) -> Element(t) {
  html.div([attribute.class("task-toolbar")], [
    search_box(current_query),
    filter_tabs(current_status),
  ])
}

fn search_box(current_query: String) -> Element(t) {
  html.input([
    attribute.type_("search"),
    attribute.name("q"),
    attribute.value(current_query),
    attribute.placeholder("Search tasks..."),
    attribute.class("search-input"),
    hx.get("/tasks"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    hx.trigger([
      hx.keyup()
      |> hx.with_changed
      |> hx.with_delay(duration.milliseconds(300)),
    ]),
    hx.push_url(True),
    hx.include(hx.Selector("[name='status']")),
  ])
}

fn filter_tabs(current_status: String) -> Element(t) {
  html.div([attribute.class("filter-tabs")], [
    filter_tab("all", "All", current_status),
    filter_tab("active", "Active", current_status),
    filter_tab("done", "Done", current_status),
  ])
}

fn filter_tab(
  value: String,
  label: String,
  current: String,
) -> Element(t) {
  let is_active = value == current
  let class = case is_active {
    True -> "tab active"
    False -> "tab"
  }
  html.button([
    attribute.class(class),
    attribute.name("status"),
    attribute.value(value),
    hx.get("/tasks"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    hx.include(hx.Selector("[name='q']")),
    hx.push_url(True),
  ], [element.text(label)])
}

pub fn task_list_html(tasks: List(Task)) -> Element(t) {
  case tasks {
    [] ->
      html.p([attribute.class("empty-state")], [
        element.text("No tasks found."),
      ])
    tasks ->
      html.ul(
        [attribute.id("task-list-items")],
        list.map(tasks, task_item),
      )
  }
}

fn task_item(task: Task) -> Element(t) {
  let class = case task.done {
    True -> "task-item done"
    False -> "task-item"
  }
  html.li([attribute.class(class), attribute.id("task-" <> task.id)], [
    html.span([attribute.class("task-title")], [
      element.text(task.title),
    ]),
    html.div([attribute.class("task-actions")], [
      html.button([
        attribute.class("btn-toggle"),
        hx.patch("/tasks/toggle/" <> task.id),
        hx.target(hx.Selector("#task-" <> task.id)),
        hx.swap(hx.OuterHTML),
      ], [
        element.text(case task.done {
          True -> "Undo"
          False -> "Done"
        }),
      ]),
      html.button([
        attribute.class("btn-delete"),
        hx.delete("/tasks/delete/" <> task.id),
        hx.target(hx.Selector("#task-" <> task.id)),
        hx.swap(hx.OuterHTML),
      ], [element.text("Delete")]),
    ]),
  ])
}
```

### `src/teamwork/router.gleam`

The route handler for tasks with query parameter parsing:

```gleam
// src/teamwork/router.gleam (relevant excerpt)

import gleam/list
import gleam/otp/actor
import gleam/result
import lustre/element
import wisp
import teamwork/task
import teamwork/views/tasks_view

fn handle_tasks(req: wisp.Request, ctx: Context) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let search_query =
    list.key_find(query_params, "q")
    |> result.unwrap("")

  let status_filter =
    list.key_find(query_params, "status")
    |> result.unwrap("all")

  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let filtered =
    tasks
    |> task.filter_by_status(status_filter)
    |> task.filter_by_search(search_query)

  // Check if this is an HTMX request
  let is_htmx =
    list.key_find(req.headers, "hx-request")
    |> result.unwrap("")
    == "true"

  case is_htmx {
    True -> {
      // Return just the task list fragment
      let html = tasks_view.task_list_html(filtered)
      wisp.html_response(
        element.to_string(html),
        200,
      )
    }
    False -> {
      // Return the full page
      let page =
        layout("Tasks -- Teamwork", [
          tasks_view.tasks_page(filtered, search_query, status_filter),
        ])
      wisp.html_response(
        element.to_document_string(page),
        200,
      )
    }
  }
}
```

### `priv/static/css/style.css` (additions)

Append these rules to your existing stylesheet:

```css
/* Search and filter toolbar */
.task-toolbar {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.search-input {
  width: 100%;
  padding: 0.625rem 0.75rem;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 0.9375rem;
  color: #1a1a2e;
  background: white;
  transition: border-color 0.15s ease;
}

.search-input:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.1);
}

.search-input::placeholder {
  color: #999;
}

.filter-tabs {
  display: flex;
  gap: 0.5rem;
}

.tab {
  padding: 0.5rem 1rem;
  border: 1px solid #ddd;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  font-weight: 500;
  color: #555;
  transition: all 0.15s ease;
}

.tab:hover {
  background: #f5f5f5;
  border-color: #ccc;
}

.tab.active {
  background: #16213e;
  color: white;
  border-color: #16213e;
}

.tab.active:hover {
  background: #1a2745;
}

.empty-state {
  text-align: center;
  padding: 2rem 1rem;
  color: #888;
  font-style: italic;
}

/* Task list items */
#task-list-items {
  list-style: none;
  padding: 0;
}

.task-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 1rem;
  border: 1px solid #eee;
  border-radius: 6px;
  margin-bottom: 0.5rem;
  background: white;
  transition: opacity 0.2s ease;
}

.task-item.done .task-title {
  text-decoration: line-through;
  color: #999;
}

.task-actions {
  display: flex;
  gap: 0.375rem;
}

.btn-toggle,
.btn-delete {
  padding: 0.25rem 0.625rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8125rem;
  background: white;
}

.btn-toggle:hover {
  background: #e8f5e9;
  border-color: #4caf50;
}

.btn-delete:hover {
  background: #ffebee;
  border-color: #f44336;
}
```

---

## 4. Exercise

Now it is your turn to extend the search and filter system. Work through these
tasks in order.

### Task 1: Add a "Sort By" Dropdown

Add a `<select>` dropdown with three options:

- **Newest first** (default) -- tasks in the order they were created, newest at the top.
- **Oldest first** -- tasks in creation order, oldest at the top.
- **Alphabetical** -- tasks sorted by title, A to Z.

The dropdown should:
- Have `name="sort"`.
- Send a GET request to `/tasks` when the value changes (use `hx-trigger="change"`).
- Include both the search query and the status filter (use `hx-include`).
- Update the URL (use `hx-push-url`).

On the server side, add a `sort` query parameter to `handle_tasks` and use
`list.sort` with an appropriate comparison function.

### Task 2: Show the Result Count

Below the filter tabs, display a line of text showing how many tasks match the
current filters. For example: "Showing 3 of 10 tasks". This gives the user
immediate feedback about how much their filters have narrowed the results.

You will need to pass both the total count and the filtered count to the view
function.

### Task 3: Add a "Clear Filters" Button

When the user has active filters (a non-empty search query or a status other
than "all"), show a "Clear filters" button that resets everything. This button
should:

- Send a GET request to `/tasks` with no query parameters.
- Replace the entire `#task-list` content.
- Update the URL back to `/tasks`.
- Only appear when there are active filters. If there are no active filters, do not render the button.

### Task 4: Verify Back Button Behavior

With all the above working, test the browser's back button:

1. Start at `/tasks` (no filters).
2. Click the "Active" tab. The URL should change to `/tasks?status=active`.
3. Type "design" in the search box. The URL should change to `/tasks?q=design&status=active`.
4. Press the back button. The URL should revert to `/tasks?status=active` and the search box should clear.
5. Press the back button again. The URL should revert to `/tasks`.

If the back button does not restore the filter state, you may need to listen
for the `popstate` event. HTMX handles this automatically when you use
`hx-push-url`, but only for the content swap -- you may need to re-trigger
the request. Look into `htmx.config.refreshOnHistoryMiss` or the
`htmx:historyRestore` event.

### Task 5: Search Highlighting (Stretch Goal)

When the user searches for "des", highlight the matching portion of each task
title. For example, if a task is titled "Create **des**ign mockups", the "des"
portion should be wrapped in a `<mark>` tag.

Implement this on the server side by modifying `task_item` to accept the
current search query and use `string.split` to break the title around the
matching substring.

---

## 5. Exercise Solution Hints

Try each task on your own before reading these hints.

### Hint for Task 1

The dropdown element in Gleam:

```gleam
fn sort_dropdown(current_sort: String) -> Element(t) {
  html.select([
    attribute.name("sort"),
    attribute.class("sort-select"),
    hx.get("/tasks"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
    hx.trigger([hx.change()]),
    hx.push_url(True),
    hx.include(hx.Selector("[name='q'], [name='status']")),
  ], [
    sort_option("newest", "Newest first", current_sort),
    sort_option("oldest", "Oldest first", current_sort),
    sort_option("alpha", "Alphabetical", current_sort),
  ])
}

fn sort_option(
  value: String,
  label: String,
  current: String,
) -> Element(t) {
  html.option(
    [
      attribute.value(value),
      attribute.selected(value == current),
    ],
    label,
  )
}
```

On the server side, the sort function:

```gleam
fn sort_tasks(tasks: List(Task), sort: String) -> List(Task) {
  case sort {
    "oldest" -> list.reverse(tasks)
    "alpha" -> list.sort(tasks, fn(a, b) {
      string.compare(string.lowercase(a.title), string.lowercase(b.title))
    })
    _ -> tasks  // "newest" is the default order
  }
}
```

Remember to update `hx.include` on the search box and filter tabs to also
include `[name='sort']`.

### Hint for Task 2

Pass the total count and filtered count as arguments:

```gleam
fn result_count(
  shown: Int,
  total: Int,
) -> Element(t) {
  let text = "Showing "
    <> int.to_string(shown)
    <> " of "
    <> int.to_string(total)
    <> " tasks"
  html.p([attribute.class("result-count")], [
    element.text(text),
  ])
}
```

Call this inside the task list response, before the `<ul>`. You need
`import gleam/int` for `int.to_string`.

### Hint for Task 3

The clear button only renders when filters are active:

```gleam
fn clear_filters_button(
  search_query: String,
  status_filter: String,
) -> Element(t) {
  let has_filters = search_query != "" || status_filter != "all"
  case has_filters {
    False -> element.none()
    True ->
      html.button([
        attribute.class("btn-clear"),
        hx.get("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.InnerHTML),
        hx.push_url(True),
      ], [element.text("Clear filters")])
  }
}
```

`element.none()` renders nothing -- it produces an empty text node. This is
Lustre's way of conditionally omitting an element.

Notice that the button does not use `hx-include`. By excluding `hx-include`,
the request goes to `/tasks` with no query parameters, effectively resetting
all filters.

### Hint for Task 4

HTMX's `hx-push-url` automatically integrates with the browser's history
API. When you press Back, the browser fires a `popstate` event. By default,
HTMX caches the previous page content and restores it from the cache.

If you find that the page content restores but the input values (search box,
active tab) do not update, you have two options:

1. Add `hx-history-elt` to the element that should be cached and restored.
2. Configure HTMX to re-request the page on history miss:

```javascript
htmx.config.refreshOnHistoryMiss = true;
```

This tells HTMX to make a fresh server request when it cannot find a cached
version, which ensures the server renders the correct state for the URL.

### Hint for Task 5

The key idea is to split the title around the search term. Here is a helper
that wraps matches in `<mark>` tags:

```gleam
fn highlight_text(
  text: String,
  query: String,
) -> List(Element(t)) {
  case query {
    "" -> [element.text(text)]
    q -> {
      let lower_text = string.lowercase(text)
      let lower_q = string.lowercase(q)
      case string.contains(lower_text, lower_q) {
        False -> [element.text(text)]
        True -> {
          // Find the position and split manually
          let parts = string.split(lower_text, lower_q)
          let original_parts = split_preserving_case(text, q)
          interleave_with_mark(original_parts, q)
        }
      }
    }
  }
}
```

This is a stretch goal because the case-insensitive splitting logic requires
careful handling. A simpler approach: if you control the data, just use
`string.split` on the lowercase version and reconstruct with offsets. Or,
accept case-sensitive highlighting as a first pass.

---

## 6. Key Takeaways

1. **Live search with HTMX is one attribute away.** `hx-trigger="keyup changed delay:300ms"` on an input element gives you debounced, live-as-you-type search. No JavaScript event listeners, no request management, no DOM manipulation.

2. **Debouncing prevents request flooding.** The `delay:300ms` modifier waits for the user to pause before sending a request. The `changed` modifier ensures no request is sent if the value did not actually change. Together, they make search efficient and responsive.

3. **`hx-include` coordinates multiple controls.** When your search box and filter tabs need to send their values together, `hx-include` with a CSS selector pulls in values from elements outside the triggering element. This keeps your filter state consistent across interactions.

4. **`hx-push-url` makes filters bookmarkable.** Updating the browser's URL bar after each filter change means the URL is always a complete representation of the current view state. Users can bookmark, share, and navigate back through filter states.

5. **Query parameters are the natural home for filter state.** `/tasks?q=design&status=active` is readable, shareable, and stateless. The server can render the correct view from the URL alone, with no session state or cookies required.

6. **`wisp.get_query` and `list.key_find` parse query parameters.** `wisp.get_query(req)` returns a `List(#(String, String))`, and `list.key_find` extracts individual values. Always provide a default with `result.unwrap` for when a parameter is missing.

7. **`list.filter` is your workhorse for server-side filtering.** Give it a predicate function, and it returns a new list containing only the elements that match. Chain multiple `list.filter` calls with the pipe operator to apply multiple criteria.

8. **Full page vs. partial response.** Check the `hx-request` header to determine whether to return a complete HTML document (direct navigation) or a fragment (HTMX swap). This pattern ensures your app works with and without JavaScript.

9. **Empty states are not optional.** When filters produce no results, show a clear message. A blank space where tasks used to be is confusing. An explicit "No tasks found." tells the user their filters are too restrictive.

10. **URL state and server state are the same thing.** In a hypermedia application, the URL tells the server everything it needs to render the page. There is no hidden client-side state that can get out of sync. This is one of the deepest advantages of the HTMX approach.

---

## What's Next

In Chapter 11 we will add **multiple boards and navigation** -- organizing
tasks into separate boards, adding `hx-boost` for instant page transitions,
nested routing with path parameters, and breadcrumb navigation. The Teamwork
app grows from a single list into a multi-board workspace.
