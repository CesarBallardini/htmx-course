# Chapter 5 -- Click, Trigger, Swap: The HTMX Triad

## Learning Objectives

By the end of this chapter, you will:

- Understand how HTMX decides **when** to fire a request using `hx-trigger`
- Know every swap strategy and **when to use each one**
- Be able to delete elements from the page without a full reload
- Show loading indicators so users know something is happening
- Combine triggers, targets, and swaps to build rich interactions for the Teamwork task board

---

## Theory

In the previous chapters we made our first HTMX requests with `hx-get` and aimed
them with `hx-target` and `hx-swap`. Those three attributes form the core triad of
every HTMX interaction:

1. **Trigger** -- *When* does the request fire?
2. **Request** -- *What* HTTP request is sent? (`hx-get`, `hx-post`, `hx-delete`, ...)
3. **Swap** -- *Where and how* does the response HTML land on the page?

We touched on targets and swaps briefly. This chapter dives deep into triggers and
swap strategies, and introduces loading indicators -- the visual glue that makes
the experience feel polished.

---

### 1. Triggers: Controlling *When* Requests Fire

Every HTMX-powered element has a trigger -- the event that causes the HTTP request.
If you do not specify `hx-trigger`, HTMX uses sensible defaults:

| Element type           | Default trigger |
|------------------------|-----------------|
| `<button>`, `<input type="submit">` | `click` |
| `<form>`               | `submit`        |
| `<input>`, `<select>`, `<textarea>` | `change` |
| Everything else         | `click`         |

You override the default by adding `hx-trigger` (or `hx.trigger(...)` in Gleam).

#### Common Trigger Events

| Trigger         | When it fires                                      |
|-----------------|----------------------------------------------------|
| `click`         | User clicks the element                            |
| `submit`        | Form is submitted                                  |
| `keyup`         | User releases a key while element is focused       |
| `mouseenter`    | Mouse pointer enters the element                   |
| `load`          | Fires immediately when the element is loaded in the DOM |
| `revealed`      | Fires when the element is scrolled into the viewport |
| `every Ns`      | Fires repeatedly on an interval (polling)          |

#### Trigger Modifiers

Modifiers let you fine-tune when a trigger actually fires. You append them after the
event name, separated by spaces.

| Modifier          | Effect                                                    |
|-------------------|-----------------------------------------------------------|
| `changed`         | Only fire if the element's value has actually changed     |
| `once`            | Fire at most one time                                     |
| `delay:Nms`       | Wait N milliseconds after the event before firing (debounce). If the event fires again during the delay, the timer resets. |
| `throttle:Nms`    | Fire at most once every N milliseconds                    |
| `from:<selector>` | Listen for the event on a different element               |

#### Combining Modifiers: A Practical Example

Imagine a live-search input. You want to send a request on `keyup`, but only if the
value actually changed, and you want to wait 300ms after the user stops typing to
avoid flooding the server:

```html
<input type="text"
       name="search"
       hx-get="/search"
       hx-target="#results"
       hx-trigger="keyup changed delay:300ms" />
```

Reading that trigger aloud: "On **keyup**, but only if the value **changed**, and
with a **300ms debounce delay**."

This pattern is extremely common for search-as-you-type interfaces.

#### Polling with `every`

Need to refresh data periodically? Use `every`:

```html
<div hx-get="/status" hx-trigger="every 5s" hx-swap="innerHTML">
  Waiting for update...
</div>
```

HTMX will send `GET /status` every 5 seconds and swap the response into the div.
No JavaScript `setInterval` needed.

---

### 2. Swap Strategies: Controlling *Where* the Response Lands

When HTMX receives an HTML response from the server, it needs to know **how** to
insert that HTML relative to the target element. The `hx-swap` attribute controls
this. There are eight strategies.

Let us visualize every one of them. In the diagrams below, the **target** is marked,
and the **response HTML** is what the server returns.

Assume the current DOM looks like this:

```
<ul id="task-list">          <!-- target -->
  <li>Existing Task A</li>
  <li>Existing Task B</li>
</ul>
```

And the server response is:

```html
<li>New Task</li>
```

---

#### `innerHTML` (default)

Replaces the **inner content** of the target. The target element itself remains.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">             <ul id="task-list">
  <li>Existing Task A</li>  ->   <li>New Task</li>
  <li>Existing Task B</li>     </ul>
</ul>
```

The `<ul>` stays. Its children are replaced entirely.

**Use when:** You want to reload an entire section's content.

---

#### `outerHTML`

Replaces the **entire target element**, including the element itself.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">
  <li>Existing Task A</li>  ->  <li>New Task</li>
  <li>Existing Task B</li>
</ul>
```

The `<ul>` is gone. It has been replaced by whatever the server returned.

**Use when:** You want to replace an element wholesale -- for example, replacing a
task item with an updated version of itself, or deleting it by returning nothing.

---

#### `beforebegin`

Inserts the response **before** the target element, as a sibling.

```
BEFORE                          AFTER
------                          -----
                                <li>New Task</li>        <-- inserted here
<ul id="task-list">             <ul id="task-list">
  <li>Existing Task A</li>       <li>Existing Task A</li>
  <li>Existing Task B</li>       <li>Existing Task B</li>
</ul>                           </ul>
```

**Use when:** You want to insert content just above an element in the DOM tree.

---

#### `afterbegin`

Inserts the response as the **first child** of the target.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">             <ul id="task-list">
                                  <li>New Task</li>      <-- inserted here
  <li>Existing Task A</li>       <li>Existing Task A</li>
  <li>Existing Task B</li>       <li>Existing Task B</li>
</ul>                           </ul>
```

**Use when:** You want to prepend to a list -- newest items first.

---

#### `beforeend`

Inserts the response as the **last child** of the target.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">             <ul id="task-list">
  <li>Existing Task A</li>       <li>Existing Task A</li>
  <li>Existing Task B</li>       <li>Existing Task B</li>
                                  <li>New Task</li>      <-- inserted here
</ul>                           </ul>
```

**Use when:** You want to **append** to a list. This is one of the most useful swap
strategies. Think: adding a new chat message, a new task, a new row to a table.

---

#### `afterend`

Inserts the response **after** the target element, as a sibling.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">             <ul id="task-list">
  <li>Existing Task A</li>       <li>Existing Task A</li>
  <li>Existing Task B</li>       <li>Existing Task B</li>
</ul>                           </ul>
                                <li>New Task</li>        <-- inserted here
```

**Use when:** You want to insert content just below an element in the DOM tree.

---

#### `delete`

**Deletes** the target element from the DOM. The server response body is ignored.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">
  <li>Existing Task A</li>  ->  (nothing -- the <ul> is removed)
  <li>Existing Task B</li>
</ul>
```

**Use when:** You want to remove an element. Commonly used for delete operations
where you target the specific item being deleted.

---

#### `none`

Does **nothing** with the response. The DOM is not touched.

```
BEFORE                          AFTER
------                          -----
<ul id="task-list">             <ul id="task-list">
  <li>Existing Task A</li>  ->   <li>Existing Task A</li>
  <li>Existing Task B</li>       <li>Existing Task B</li>
</ul>                           </ul>
```

**Use when:** You only want a side effect on the server (logging, analytics, etc.)
and do not need to update the page. Also useful with response headers like
`HX-Trigger` for event-driven workflows (covered in later chapters).

---

#### Quick Reference Table

| Strategy      | Target stays? | Children stay? | Where response goes          |
|---------------|---------------|----------------|------------------------------|
| `innerHTML`   | Yes           | No (replaced)  | Replaces target's children   |
| `outerHTML`   | No (replaced) | No (replaced)  | Replaces the target itself   |
| `beforebegin` | Yes           | Yes            | Before the target (sibling)  |
| `afterbegin`  | Yes           | Yes            | First child of target        |
| `beforeend`   | Yes           | Yes            | Last child of target         |
| `afterend`    | Yes           | Yes            | After the target (sibling)   |
| `delete`      | No (removed)  | No (removed)   | Response ignored             |
| `none`        | Yes           | Yes            | Response ignored             |

---

### 3. Loading Indicators: Telling Users "Something Is Happening"

When a network request takes more than an instant, users need visual feedback.
HTMX has a built-in mechanism for this: **`hx-indicator`**.

Here is how it works:

1. You create an element with the CSS class `htmx-indicator`. By default, this
   element is invisible (opacity 0).
2. You point to that element using `hx-indicator="#my-spinner"` on the element
   that makes the request.
3. When the request starts, HTMX adds the class `htmx-request` to the indicator
   element, making it visible.
4. When the response arrives, HTMX removes `htmx-request`, hiding it again.

The default CSS you need:

```css
.htmx-indicator {
    opacity: 0;
    transition: opacity 200ms ease-in;
}

.htmx-request .htmx-indicator,
.htmx-request.htmx-indicator {
    opacity: 1;
}
```

The first rule (`.htmx-request .htmx-indicator`) handles the case where the
indicator is a **child** of the requesting element. The second rule
(`.htmx-request.htmx-indicator`) handles the case where the indicator **is**
the requesting element (or is targeted explicitly via `hx-indicator`).

> **Note:** If you include the standard htmx CSS or the htmx script tag, most of
> this CSS is already applied. But it is good to understand what is happening under
> the hood.

---

## Code Walkthrough

Let us put all of this into practice by extending our **Teamwork** task board. We
will build:

1. A task list that loads automatically when the page opens
2. A button to add new tasks (appended to the end of the list)
3. Delete buttons on each task (removed with outerHTML swap)
4. A loading indicator while the task list loads
5. A "last updated" timestamp that refreshes every 5 seconds

### Project Structure

```
teamwork/
  src/
    teamwork.gleam        -- main application, router, handlers
  priv/
    static/
      styles.css          -- CSS including htmx-indicator styles
  gleam.toml
```

### Step 1: Building the Home Page

The home page wires together all our interactions. Study the comments in the code
(see the full listing below), and use this table as a reference:

| Element          | Trigger             | Target         | Swap         | Behavior                                  |
|------------------|---------------------|----------------|--------------|-------------------------------------------|
| "Load Tasks" btn | `click` (default)   | `#task-list`   | `innerHTML`  | Replaces entire list content               |
| "Add Random" btn | `click` (default)   | `#task-list`   | `beforeend`  | Appends new task at end of list            |
| `#task-list` ul  | `load`              | itself         | `innerHTML`  | Auto-loads tasks when page opens           |
| `#hover-preview` | `mouseenter once`   | itself         | `innerHTML`  | Loads preview on first hover only          |
| `#last-updated`  | `load, every 5s`    | itself         | `innerHTML`  | Loads immediately, then polls every 5s     |

Key observations:

- The `#task-list` has `hx-trigger="load"`. As soon as the page loads, HTMX fires
  a GET to `/tasks` and fills the list automatically. The user sees "Loading
  tasks..." briefly, then the tasks appear.

- The timestamp div uses `hx-trigger="load, every 5s"`. You can combine multiple
  triggers with a comma -- this fires immediately on load, then again every 5s.

- The spinner `<span>` has class `htmx-indicator` (hidden by default) and is
  referenced via `hx.indicator(hx.Selector("#spinner"))` from both the button and the list.

### Step 2: Task Item Component with Delete

Each task item is a self-contained component. The delete button targets its own
parent `<li>` and uses `outerHTML` swap. When the server returns an empty response,
the entire `<li>` is replaced with nothing -- effectively removing it.

```gleam
fn task_item(id: String, title: String) -> element.Element(t) {
  html.li([attribute.id("task-" <> id)], [
    html.span([], [element.text(title)]),
    html.button(
      [
        hx.delete("/tasks/" <> id),
        hx.target(hx.Selector("#task-" <> id)),
        hx.swap(hx.OuterHTML),
      ],
      [element.text("Delete")],
    ),
  ])
}
```

Here is the mental model for what happens when the user clicks "Delete" on task 3:

```
1. User clicks "Delete" on task-3.

2. HTMX sends:  DELETE /tasks/3

3. Server deletes the task and returns an empty response (200 OK, no body).

4. HTMX applies outerHTML swap on #task-3:

   BEFORE                              AFTER
   ------                              -----
   <ul id="task-list">                 <ul id="task-list">
     <li id="task-1">Task A ...</li>     <li id="task-1">Task A ...</li>
     <li id="task-2">Task B ...</li>     <li id="task-2">Task B ...</li>
     <li id="task-3">Task C ...</li>     <!-- task-3 is gone! -->
   </ul>                               </ul>
```

The `<li id="task-3">` is replaced with the empty response, so it vanishes.

### Step 3: The Router and Handlers

Now let us write the server-side code. Each route returns the appropriate HTML
fragment. Here are the key patterns to understand (see the full code listing
below for the complete implementation):

- **`get_tasks`** -- Returns all tasks as `<li>` elements. Includes
  `process.sleep(1000)` to simulate a slow database query so you can see the
  loading indicator. **Remove this delay in production code.**

- **`get_random_task`** -- Returns a single `<li>` element. Because the "Add Random
  Task" button uses `hx-swap="beforeend"`, this `<li>` gets appended as the last
  child of the `<ul>`. Existing tasks remain untouched.

- **`delete_task`** -- Returns an **empty 200 response**. This is the key to the
  delete pattern: when the target element's `outerHTML` is replaced with an empty
  string, the element is removed from the DOM.

- **`get_preview`** -- Returns HTML for the hover preview box.

- **`get_timestamp`** -- Returns the current server time as text. Called every 5
  seconds by the polling div.

---

### Gleam `hx` Module Quick Reference

The Gleam `hx` module provides typed helpers for HTMX attributes. Here is a
summary of the ones used in this chapter:

```gleam
// Triggers (typed events)
hx.trigger([hx.click()])                     // click event
hx.trigger([hx.load()])                      // fire on page load
hx.trigger([hx.custom("mouseenter")          // mouseenter with once modifier
  |> hx.with_once])
hx.trigger([hx.keyup()                       // keyup with changed + debounce
  |> hx.with_changed
  |> hx.with_delay(duration.milliseconds(300))])
hx.trigger_polling(                          // poll every 5s, also on load
  timing: duration.seconds(5),
  filters: None,
  on_load: True,
)

// Swap strategies (typed variants)
hx.swap(hx.InnerHTML)    hx.swap(hx.OuterHTML)     hx.swap(hx.Beforeend)
hx.swap(hx.Afterbegin)  hx.swap(hx.Beforebegin)   hx.swap(hx.Afterend)
hx.swap(hx.Delete)      hx.swap(hx.SwapNone)

// HTTP methods, target, and indicator
hx.get("/path")    hx.post("/path")    hx.delete("/path")
hx.put("/path")    hx.patch("/path")
hx.target(hx.Selector("#id"))    hx.indicator(hx.Selector("#spinner"))
```

Using typed helpers instead of raw attribute strings gives you compile-time
guarantees. If you misspell `hx.Beforeend`, Gleam catches it. If you misspell
`"beforend"` in a raw string, nobody catches it until you test in the browser.

---

## Full Code Listing

Below is the complete `src/teamwork.gleam` file. For the CSS file
(`priv/static/styles.css`), use the indicator styles from the Theory section
combined with your own layout styles. The key CSS rules are the `.htmx-indicator`
and `.htmx-request` selectors shown earlier.

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process
import gleam/http
import gleam/int
import gleam/list
import gleam/option.{None}
import gleam/string
import gleam/time/duration
import gleam/time/timestamp
import hx
import lustre/attribute
import lustre/element
import lustre/element/html
import mist
import wisp.{type Request, type Response}
import wisp/wisp_mist

// ── Main ─────────────────────────────────────────────────────────────────

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let assert Ok(_) =
    wisp_mist.handler(handle_request, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

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
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/styles.css"),
        ]),
      ]),
      html.body([], [
        html.h1([], [element.text("Teamwork Task Board")]),
        content,
      ]),
    ])

  let body =
    page
    |> element.to_document_string

  wisp.html_response(body, 200)
}

// ── Components ──────────────────────────────────────────────────────────

fn task_item(id: String, title: String) -> element.Element(t) {
  html.li([attribute.id("task-" <> id)], [
    html.span([], [element.text(title)]),
    html.button(
      [
        hx.delete("/tasks/" <> id),
        hx.target(hx.Selector("#task-" <> id)),
        hx.swap(hx.OuterHTML),
      ],
      [element.text("Delete")],
    ),
  ])
}

// ── Pages ───────────────────────────────────────────────────────────────

fn home_page() -> Response {
  let content =
    html.div([], [
      // ── Controls ──
      html.div([], [
        html.button(
          [
            attribute.class("btn btn-primary"),
            hx.get("/tasks"),
            hx.target(hx.Selector("#task-list")),
            hx.swap(hx.InnerHTML),
            hx.indicator(hx.Selector("#spinner")),
          ],
          [element.text("Load Tasks")],
        ),
        html.button(
          [
            attribute.class("btn btn-primary"),
            hx.get("/tasks/random"),
            hx.target(hx.Selector("#task-list")),
            hx.swap(hx.Beforeend),
          ],
          [element.text("Add Random Task")],
        ),
        html.span(
          [attribute.id("spinner"), attribute.class("htmx-indicator")],
          [element.text(" Loading...")],
        ),
      ]),

      // ── Task list: auto-loads on page load ──
      html.ul(
        [
          attribute.id("task-list"),
          hx.get("/tasks"),
          hx.trigger([hx.load()]),
          hx.swap(hx.InnerHTML),
          hx.indicator(hx.Selector("#spinner")),
        ],
        [element.text("Loading tasks...")],
      ),

      // ── Hover preview ──
      html.div([], [
        html.p([], [
          element.text("Hover over the box below to load a preview:"),
        ]),
        html.div(
          [
            attribute.id("hover-preview"),
            hx.get("/preview"),
            hx.trigger([hx.custom("mouseenter") |> hx.with_once]),
            hx.swap(hx.InnerHTML),
          ],
          [element.text("[ Hover over me ]")],
        ),
      ]),

      // ── Polling timestamp ──
      html.div(
        [
          attribute.id("last-updated"),
          hx.get("/timestamp"),
          hx.trigger_polling(
            timing: duration.seconds(5),
            filters: None,
            on_load: True,
          ),
          hx.swap(hx.InnerHTML),
        ],
        [element.text("Loading timestamp...")],
      ),
    ])

  layout(content)
}

// ── Handlers ────────────────────────────────────────────────────────────

fn get_tasks(_req: Request) -> Response {
  // Simulate a slow database query (remove in production!)
  process.sleep(1000)

  let tasks = [
    task_item("1", "Set up project repository"),
    task_item("2", "Design database schema"),
    task_item("3", "Build task API endpoints"),
    task_item("4", "Create front-end layout"),
    task_item("5", "Write integration tests"),
  ]

  let body =
    tasks
    |> list.map(element.to_string)
    |> string.join("")

  wisp.html_response(body, 200)
}

fn get_random_task(_req: Request) -> Response {
  let id = int.to_string(int.random(10_000))
  let title = "New task #" <> id

  let body =
    task_item(id, title)
    |> element.to_string

  wisp.html_response(body, 200)
}

fn delete_task(_req: Request, _id: String) -> Response {
  // In a real app: delete from database here
  // let _ = db.delete_task(id)
  wisp.html_response("", 200)
}

fn get_preview(_req: Request) -> Response {
  let body =
    html.div([], [
      html.strong([], [element.text("Preview loaded!")]),
      element.text(" This content appeared because you hovered. "),
      element.text("Thanks to the "),
      html.code([], [element.text("once")]),
      element.text(" modifier, hovering again will not trigger another request."),
    ])
    |> element.to_string

  wisp.html_response(body, 200)
}

fn get_timestamp(_req: Request) -> Response {
  let now = timestamp.to_unix_seconds(timestamp.system_time())
  let body =
    "Last updated: " <> int.to_string(now) <> " (refreshes every 5s)"

  wisp.html_response(body, 200)
}

// ── Helpers ─────────────────────────────────────────────────────────────

fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}

// ── Router ──────────────────────────────────────────────────────────────

pub fn handle_request(req: Request) -> Response {
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    [] -> home_page()

    ["tasks"] -> {
      use <- wisp.require_method(req, http.Get)
      get_tasks(req)
    }

    ["tasks", "random"] -> {
      use <- wisp.require_method(req, http.Get)
      get_random_task(req)
    }

    ["tasks", id] -> {
      use <- wisp.require_method(req, http.Delete)
      delete_task(req, id)
    }

    ["preview"] -> {
      use <- wisp.require_method(req, http.Get)
      get_preview(req)
    }

    ["timestamp"] -> {
      use <- wisp.require_method(req, http.Get)
      get_timestamp(req)
    }

    _ -> wisp.not_found()
  }
}
```

---

## Exercise

Starting from the code above, complete the following five tasks:

**Task 1 -- Delete with Smooth Removal:** Add a "Delete" button to each task that
sends `DELETE /tasks/:id`, targets the specific `<li>`, and uses `outerHTML` swap.
Verify: clicking delete removes that task without affecting others.

**Task 2 -- Append New Tasks:** Add an "Add Task" button that sends
`GET /tasks/random`, targets the `<ul>`, and uses `beforeend` swap. Verify:
clicking it multiple times appends a new task each time.

**Task 3 -- Loading Indicator:** Add a `<span>` with class `htmx-indicator` and
point to it with `hx-indicator`. Add `process.sleep(1000)` in the handler.
Verify: "Loading..." appears during the request and disappears when done.

**Task 4 -- Load on Hover (Once):** Create a `<div>` with
`hx-trigger="mouseenter once"` that fetches `/preview` on first hover. Verify:
content loads once; subsequent hovers do not trigger new requests.

**Task 5 -- Polling Timestamp:** Add an element with
`hx-trigger="load, every 5s"` that fetches `/timestamp`. Verify: the timestamp
appears immediately, then updates every 5 seconds automatically.

---

## Exercise Solution Hints

If you get stuck, here are some pointers:

- **Delete**: The handler returns `wisp.html_response("", 200)` --
  an empty 200 response. The delete button must target the **specific** `<li>`
  with `hx.target(hx.Selector("#task-" <> id))`, not the whole list.

- **Append**: The `/tasks/random` handler returns exactly one `<li>` element.
  The `beforeend` swap inserts it as the last child of the `<ul>`.

- **Polling**: Use `hx.trigger_polling(timing: duration.seconds(5), filters: None, on_load: True)`.
  Setting `on_load: True` fires immediately, then polls every 5 seconds.

- **Once modifier**: `hx.trigger([hx.custom("mouseenter") |> hx.with_once])` -- HTMX fires on the first
  `mouseenter` and then removes the listener.

- **Indicator CSS**: Make sure your CSS includes **both** selectors:
  `.htmx-request .htmx-indicator` (child) and `.htmx-request.htmx-indicator`
  (direct). Missing one may cause the indicator to not show.

---

## Key Takeaways

1. **The HTMX Triad**: Every interaction is defined by three questions: *When*
   does it fire (trigger)? *What* request is sent (hx-get/post/delete/...)? *How*
   is the response placed on the page (swap)?

2. **Default triggers are smart**: Buttons trigger on `click`, forms on `submit`,
   inputs on `change`. You only need `hx-trigger` when you want something different.

3. **Trigger modifiers are powerful**: `changed`, `once`, `delay:Nms`, and
   `throttle:Nms` let you fine-tune behavior without writing any JavaScript.

4. **Swap strategies give you surgical control**: `innerHTML` replaces children.
   `outerHTML` replaces the whole element. `beforeend` appends. `delete` removes.
   Choose the right one for the job.

5. **Delete pattern**: Send a DELETE request, return an empty response, use
   `outerHTML` swap on the target element. The element is replaced with nothing
   and disappears.

6. **Loading indicators are free**: Add `hx-indicator`, point it at an element
   with class `htmx-indicator`, and HTMX handles the show/hide automatically
   via the `htmx-request` class.

7. **Polling is one attribute**: `hx-trigger="every 5s"` gives you polling with
   zero JavaScript. Combine with `load` to also fire immediately:
   `hx-trigger="load, every 5s"`.

8. **Gleam's typed helpers catch mistakes at compile time**: Using `hx.swap(hx.Beforeend)`
   instead of a raw string `"beforeend"` means the compiler catches typos before
   your users do.

---

**[Next chapter:](06-server-state.md)** We will track state on the server using Gleam actors, storing our task list in memory so every request works with the same data.
