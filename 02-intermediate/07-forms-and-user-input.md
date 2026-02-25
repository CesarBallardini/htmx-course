# Chapter 7 -- Forms and User Input

**Phase:** Intermediate
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter you will be able to:

- Describe how HTML forms encode and submit data to a server.
- Use `hx-post` on a form to submit data via AJAX and swap the response into the page without a full reload.
- Parse form data on the server using `wisp.require_form` and extract fields with `list.key_find`.
- Apply `hx-boost="true"` to progressively enhance existing forms and links.
- Build form elements (inputs, labels, buttons) using Lustre's `html` module.
- Return an HTML fragment from a POST endpoint so HTMX can append it to the task list.

---

## 1. Theory

### 1.1 HTML Forms -- The Original Way to Send Data

Before JavaScript existed, HTML forms were the *only* way to send user input to
a server. The mechanism is simple and has not changed in thirty years:

```html
<form method="post" action="/tasks">
  <input type="text" name="title" />
  <button type="submit">Add Task</button>
</form>
```

When the user clicks "Add Task", the browser:

1. Collects every named input inside the form.
2. Encodes them as key-value pairs: `title=Buy+milk`.
3. Sends an HTTP POST request to `/tasks` with those pairs in the body.
4. **Replaces the entire page** with whatever the server sends back.

That last step is the problem. The full page reload causes a white flash, resets
scroll position, and destroys any client-side state. It works, but it feels
clunky.

### 1.2 Form Data Encoding

By default, a form sends its data as `application/x-www-form-urlencoded`. This
is a flat list of key-value pairs separated by `&`, with special characters
percent-encoded:

```
title=Buy+milk&description=We+need+2%25+milk
```

| Character | Encoded as |
|-----------|------------|
| space     | `+`        |
| `&`       | `%26`      |
| `=`       | `%3D`      |
| `%`       | `%25`      |

There is also `multipart/form-data` for file uploads, but we will not need that
until much later. For text-based forms, url-encoded is the standard.

### 1.3 How HTMX Enhances Forms

HTMX gives you a way to submit forms without a full page reload. The key
attribute is `hx-post`:

```html
<form hx-post="/tasks" hx-target="#task-list" hx-swap="beforeend">
  <input type="text" name="title" />
  <button type="submit">Add Task</button>
</form>
```

When the user submits this form, HTMX:

1. Intercepts the native form submission (prevents the full page reload).
2. Collects the form data, exactly as the browser would.
3. Sends an AJAX POST request to `/tasks` with the form data in the body.
4. Takes the HTML that the server returns and **swaps it into** the element
   matching `#task-list`, using the `beforeend` swap strategy (append as the
   last child).

The server does not need to know whether the request came from a normal form
submission or from HTMX. It receives the same form data either way. The only
difference is what it sends back:

- For a normal submission, the server would return a full HTML page.
- For an HTMX submission, the server returns just the HTML fragment that needs
  to change -- in our case, a single `<li>` for the new task.

This is the HTMX pattern at its core. The server is still the single source of
truth. It still returns HTML. HTMX just makes the exchange surgical instead of
whole-page.

### 1.4 `hx-boost` -- Progressive Enhancement for Free

Sometimes you have a form that already works the traditional way (with a
`method` and `action`), and you simply want to upgrade it to use AJAX. That is
what `hx-boost` does:

```html
<form method="post" action="/tasks" hx-boost="true">
  <!-- fields -->
</form>
```

With `hx-boost="true"`:

- **If JavaScript is enabled** (which it almost always is), HTMX intercepts the
  submission and sends it via AJAX. The response replaces the `<body>` of the
  current page, giving you a smooth, no-reload transition.
- **If JavaScript is disabled** (rare, but possible -- screen readers, corporate
  proxies, search engine crawlers), the form submits normally. The user still
  gets a working application, just with full page reloads.

This is **progressive enhancement**: start with something that works everywhere,
then layer on improvements for browsers that support them. It is one of the
oldest and most reliable patterns in web development.

You can also put `hx-boost="true"` on a parent element (like `<body>` or a
`<nav>`) and it will boost all links and forms inside it.

### 1.5 Parsing Form Data in Wisp

On the Gleam side, Wisp provides a clean API for parsing form data:

```gleam
use form_data <- wisp.require_form(req)
```

The `wisp.require_form` function reads the request body, parses the url-encoded
data, and gives you a `FormData` value. If the body cannot be parsed (wrong
content type, too large, malformed data), Wisp automatically returns a `400 Bad
Request` response -- you do not need to handle that case yourself.

The `form_data` value has a `values` field of type `List(#(String, String))` --
a list of key-value tuples. To extract a specific field:

```gleam
case list.key_find(form_data.values, "title") {
  Ok(title) -> // use the title
  Error(_) -> wisp.bad_request()  // field was missing
}
```

`list.key_find` searches the list for a tuple whose first element matches the
given key and returns the second element wrapped in `Ok`. If no match is found,
it returns `Error(Nil)`.

This is a deliberate design choice. Form data is inherently untyped -- it is
just strings from the network. Gleam forces you to handle the case where a field
is missing or has an unexpected value. You cannot accidentally forget.

---

## 2. Code Walkthrough

We are going to add a form that lets users create new tasks. By the end of this
section, you will be able to type a task title, hit "Add Task", and see it
appear in the list instantly -- without a page reload.

### Where We Left Off

After [Chapter 6](../01-beginner/06-server-state.md), our Teamwork app has:

- A Mist/Wisp server with routing.
- A layout function that renders full pages with Lustre.
- A task list rendered from in-memory state managed by a BEAM actor.
- Delete buttons on each task that use `hx-delete` to remove tasks.
- Loading indicators using `hx-indicator`.

The task list is currently read-only aside from deletion. We need a way to
add tasks.

### Step 1 -- The Add Task Form

Create a function that builds the form element. This goes in your main module
(or in a views module if you have extracted one):

```gleam
fn add_task_form() -> Element(t) {
  html.form(
    [
      hx.post("/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
      attribute.id("add-task-form"),
      attribute.class("add-task-form"),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("title")], [element.text("Task title")]),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.placeholder("What needs to be done?"),
          attribute.required(True),
        ]),
      ]),
      html.button(
        [attribute.type_("submit"), attribute.class("btn-primary")],
        [element.text("Add Task")],
      ),
    ],
  )
}
```

Let us walk through the key details.

**`hx.post("/tasks")`** -- When this form is submitted, HTMX will send a POST
request to `/tasks`. The form data (all named inputs) will be included in the
request body, encoded as `application/x-www-form-urlencoded`.

**`hx.target(hx.Selector("#task-list"))`** -- The server's response will be inserted into the
element with `id="task-list"`. This is the `<ul>` or `<div>` that holds our
existing tasks.

**`hx.swap(hx.Beforeend)`** -- The response will be appended as the last child
of the target. This means the new task appears at the bottom of the list, which
is the behaviour users expect.

**`attribute.name("title")`** -- This is critical. The `name` attribute
determines the key in the form data. Without it, the input's value will not be
sent to the server. This is standard HTML behaviour, not an HTMX thing.

**`attribute.required(True)`** -- The HTML `required` attribute. The browser
will prevent form submission if this field is empty, showing a native validation
tooltip. This is client-side validation -- we will also validate on the server.

**`attribute.for("title")` on the label** -- Links the label to the input with
`id="title"`. Clicking the label focuses the input. This is important for
accessibility: screen readers use this association to announce what each input
is for.

### Step 2 -- Place the Form on the Page

Update your page function to include the form above the task list:

```gleam
fn tasks_page(tasks: List(Task)) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    add_task_form(),
    html.div([attribute.id("task-list")], list.map(tasks, task_item)),
  ])
}
```

The form sits between the heading and the task list. When a user submits it,
HTMX will POST to `/tasks`, and the response will be appended inside the
`#task-list` div.

### Step 3 -- Handle the POST on the Server

We need to update our router to accept POST requests on the `/tasks` path.
Previously, we only handled GET:

```gleam
fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    [] -> home_page_response(req, ctx)
    ["tasks"] -> {
      case req.method {
        http.Get -> list_tasks(req, ctx)
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }
    }
    ["tasks", id] -> {
      case req.method {
        http.Delete -> delete_task(req, ctx, id)
        _ -> wisp.method_not_allowed([http.Delete])
      }
    }
    _ -> wisp.not_found()
  }
}
```

The new branch is `http.Post -> create_task(req, ctx)`. When a POST request
arrives at `/tasks`, we hand it off to our `create_task` function.

Notice the `wisp.method_not_allowed([http.Get, http.Post])` call. It returns a
`405 Method Not Allowed` response and includes an `Allow` header listing the
permitted methods. This is proper HTTP behaviour -- if someone sends a PUT to
`/tasks`, they get told what they *can* do instead.

### Step 4 -- The `create_task` Function

Here is the function that does the actual work:

```gleam
fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  // Parse the form data from the request body.
  // If parsing fails, Wisp returns 400 Bad Request automatically.
  use form_data <- wisp.require_form(req)

  // Extract the "title" field from the form data.
  case list.key_find(form_data.values, "title") {
    Ok(title) -> {
      // Guard against empty titles (the "required" attribute handles this
      // on the client, but never trust the client).
      case string.trim(title) {
        "" -> wisp.bad_request()
        trimmed_title -> {
          // Generate a unique ID and create the task.
          let id = new_id()
          let task = Task(id: id, title: trimmed_title, done: False)

          // Tell the actor to store the new task.
          actor.send(ctx.tasks, AddTask(task))

          // Return just the HTML fragment for the new task item.
          // HTMX will append this into #task-list.
          let html = task_item(task)
          wisp.html_response(element.to_string(html), 201)
        }
      }
    }
    Error(_) -> wisp.bad_request()
  }
}
```

There is a lot happening here, so let us take it apart.

**`use form_data <- wisp.require_form(req)`** -- This is the `use` pattern we
first saw in [Chapter 1](../01-beginner/01-how-the-web-works.md) with `wisp.log_request`. The `wisp.require_form`
function reads the request body and parses it. If parsing succeeds, it calls our
continuation with the parsed `FormData`. If it fails (bad content type, body too
large, parse error), it short-circuits and returns a `400` response. We never
see the error case -- Wisp handles it for us.

**`list.key_find(form_data.values, "title")`** -- Searches the list of
key-value pairs for a tuple with key `"title"`. The `form_data.values` field
has type `List(#(String, String))`. If found, we get `Ok(title)` where `title`
is the value the user typed. If not found, we get `Error(Nil)`.

**`string.trim(title)`** -- Even though we have a `required` attribute on the
input, we validate again on the server. Client-side validation is a convenience
for the user; server-side validation is a requirement for correctness. Someone
can bypass the form entirely with `curl`. The `string.trim` call strips
leading and trailing whitespace, and we reject empty strings.

**`new_id()`** -- Generates a unique identifier for the task. You might use
`int.to_string(int.random(1_000_000))` or a simple counter from your actor.
The implementation is not critical for this chapter.

**`actor.send(ctx.tasks, AddTask(task))`** -- Sends a message to the task
actor telling it to store the new task. This is the same actor pattern from
[Chapter 6](../01-beginner/06-server-state.md). The actor holds the canonical list of tasks in memory.

**Status code `201`** -- We return `201 Created` instead of `200 OK`. This is
semantically correct: a new resource was created. HTMX does not care about the
status code for swapping purposes (any 2xx triggers a swap), but using the right
code is good practice. It makes your server behave correctly for non-HTMX
clients too.

**`element.to_string(html)`** -- Note that we use `to_string`, not
`to_document_string`. We are returning a fragment (a single `<li>` or `<div>`),
not a full HTML document. There should be no `<!doctype html>` in a fragment
response.

### Step 5 -- Progressive Enhancement with `hx-boost`

The form we built in Step 1 uses `hx-post` directly, which means it only works
with JavaScript enabled. If you want a form that works *without* JavaScript and
gets enhanced *with* it, use `hx-boost` instead:

```gleam
fn add_task_form_boosted() -> Element(t) {
  html.form(
    [
      attribute.method("post"),
      attribute.action("/tasks"),
      hx.boost(True),
      attribute.id("add-task-form"),
      attribute.class("add-task-form"),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("title")], [element.text("Task title")]),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.placeholder("What needs to be done?"),
          attribute.required(True),
        ]),
      ]),
      html.button(
        [attribute.type_("submit"), attribute.class("btn-primary")],
        [element.text("Add Task")],
      ),
    ],
  )
}
```

The differences from the `hx-post` version:

- We use standard HTML `method="post"` and `action="/tasks"` attributes.
- We add `hx-boost="true"` via `hx.boost(True)`.
- We do **not** set `hx-target` or `hx-swap`. With `hx-boost`, HTMX replaces
  the entire `<body>` of the response into the current page's `<body>`.

The trade-off is clear:

| Approach     | Without JS              | With JS                         | Granularity          |
|-------------|-------------------------|----------------------------------|----------------------|
| `hx-post`   | Form does nothing       | Swaps a fragment into a target   | Surgical (one element) |
| `hx-boost`  | Full page reload (works)| Replaces `<body>` via AJAX       | Whole body           |

For our task board, we will use `hx-post` because we want the surgical swap
(append just the new task). But `hx-boost` is the right choice when you want
to enhance navigation links or forms where a full body replacement is
acceptable.

Here is an example of boosting navigation links:

```gleam
fn nav_bar() -> Element(t) {
  html.nav([hx.boost(True), attribute.class("nav")], [
    html.a([attribute.href("/")], [element.text("Home")]),
    html.a([attribute.href("/tasks")], [element.text("Tasks")]),
    html.a([attribute.href("/about")], [element.text("About")]),
  ])
}
```

With `hx-boost` on the `<nav>`, every link inside it becomes an AJAX request.
The user sees smooth page transitions instead of full reloads. Without
JavaScript, the links work normally. Zero downside.

### Step 6 -- Clearing the Form After Submission

There is one usability problem with our current form: after submitting, the
input field still contains the text the user just typed. They have to manually
clear it before adding another task. That is annoying.

There are several ways to solve this. Here is the simplest approach that does
not require features we have not covered yet:

**Approach: Use `hx-on::after-request` to reset the form.**

HTMX fires custom events during the request lifecycle. We can listen for the
`htmx:afterRequest` event and reset the form:

```gleam
fn add_task_form() -> Element(t) {
  html.form(
    [
      hx.post("/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
      attribute.id("add-task-form"),
      attribute.class("add-task-form"),
      attribute("hx-on::after-request", "this.reset()"),
    ],
    [
      // ... fields ...
    ],
  )
}
```

The `hx-on::after-request` attribute tells HTMX: "After the request completes,
run this JavaScript on the form element." The `this.reset()` call is a native
DOM method that clears all inputs in a form back to their default values.

This is a small amount of inline JavaScript, which might feel like it
contradicts the "no JavaScript" philosophy. But it is a one-liner that handles
a common UX need. HTMX is pragmatic, not dogmatic. The alternative -- having the
server return a fresh form -- requires out-of-band swaps, which we will cover in
[Chapter 9](09-swap-strategies-and-targets.md).

### Step 7 -- Styling the Form

Add the following to your `priv/static/css/style.css`:

```css
.add-task-form {
  margin-bottom: 2rem;
  padding: 1.5rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.25rem;
  font-weight: 600;
  color: #1a1a2e;
}

.form-group input[type="text"] {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  line-height: 1.5;
  transition: border-color 0.15s ease;
}

.form-group input[type="text"]:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.15);
}

.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #16213e;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.btn-primary:hover {
  background: #1a2744;
}

.btn-primary:active {
  background: #0f1729;
}
```

Nothing surprising here. The focus style on the input gives users a clear visual
indicator of where they are typing. The button hover and active states provide
tactile feedback. These small details make the difference between an application
that feels polished and one that feels like a prototype.

---

## 3. Full Code Listing

Here is the complete updated code after this chapter. Files that have not changed
from [Chapter 6](../01-beginner/06-server-state.md) are omitted.

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
import gleam/http
import gleam/int
import gleam/list
import gleam/otp/actor
import gleam/string
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import hx
import mist
import wisp
import wisp/wisp_mist

// ---------------------------------------------------------------------------
// Types
// ---------------------------------------------------------------------------

pub type Task {
  Task(id: String, title: String, done: Bool)
}

pub type Message {
  AddTask(Task)
  DeleteTask(String)
  GetTasks(process.Subject(List(Task)))
}

pub type Context {
  Context(tasks: process.Subject(Message), static: String)
}

// ---------------------------------------------------------------------------
// Entry point
// ---------------------------------------------------------------------------

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the task actor with an empty list.
  let assert Ok(tasks) = start_task_actor()

  let assert Ok(priv) = wisp.priv_directory("teamwork")
  let static = priv <> "/static"

  let ctx = Context(tasks: tasks, static: static)

  let assert Ok(_) =
    wisp_mist.handler(fn(req) { handle_request(req, ctx) }, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

// ---------------------------------------------------------------------------
// Task actor
// ---------------------------------------------------------------------------

fn start_task_actor() -> Result(process.Subject(Message), actor.StartError) {
  actor.new([])
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}

fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}

fn handle_message(
  tasks: List(Task),
  message: Message,
) -> actor.Next(List(Task), Message) {
  case message {
    GetTasks(reply_to) -> {
      process.send(reply_to, tasks)
      actor.continue(tasks)
    }
    AddTask(task) -> {
      actor.continue([task, ..tasks])
    }
    DeleteTask(id) -> {
      let remaining = list.filter(tasks, fn(t) { t.id != id })
      actor.continue(remaining)
    }
  }
}

fn get_all_tasks(ctx: Context) -> List(Task) {
  actor.call(ctx.tasks, 1000, GetTasks)
}

// ---------------------------------------------------------------------------
// ID generation
// ---------------------------------------------------------------------------

fn new_id() -> String {
  int.to_string(int.random(1_000_000))
}

// ---------------------------------------------------------------------------
// Router
// ---------------------------------------------------------------------------

fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: ctx.static)

  case wisp.path_segments(req) {
    [] -> {
      let tasks = get_all_tasks(ctx)
      let page = layout("Teamwork", tasks_page(tasks))
      let html_string = element.to_document_string(page)
      wisp.html_response(html_string, 200)
    }
    ["tasks"] -> {
      case req.method {
        http.Get -> {
          let tasks = get_all_tasks(ctx)
          let html_string =
            list.map(tasks, task_item)
            |> element.fragment
            |> element.to_string
          wisp.html_response(html_string, 200)
        }
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }
    }
    ["tasks", id] -> {
      case req.method {
        http.Delete -> delete_task(ctx, id)
        _ -> wisp.method_not_allowed([http.Delete])
      }
    }
    _ -> wisp.not_found()
  }
}

// ---------------------------------------------------------------------------
// Create task handler
// ---------------------------------------------------------------------------

fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  case list.key_find(form_data.values, "title") {
    Ok(title) -> {
      case string.trim(title) {
        "" -> wisp.bad_request()
        trimmed_title -> {
          let id = new_id()
          let task = Task(id: id, title: trimmed_title, done: False)
          actor.send(ctx.tasks, AddTask(task))

          let html = task_item(task)
          wisp.html_response(element.to_string(html), 201)
        }
      }
    }
    Error(_) -> wisp.bad_request()
  }
}

// ---------------------------------------------------------------------------
// Delete task handler
// ---------------------------------------------------------------------------

fn delete_task(ctx: Context, id: String) -> wisp.Response {
  actor.send(ctx.tasks, DeleteTask(id))
  wisp.html_response("", 200)
}

// ---------------------------------------------------------------------------
// Views
// ---------------------------------------------------------------------------

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
    html.body([], [
      html.nav([hx.boost(True), attribute.class("nav")], [
        html.a([attribute.href("/")], [element.text("Home")]),
        html.a([attribute.href("/tasks")], [element.text("Tasks")]),
        html.a([attribute.href("/about")], [element.text("About")]),
      ]),
      content,
      html.footer([attribute.class("footer")], [
        html.p([], [element.text("Built with Gleam and HTMX")]),
      ]),
    ]),
  ])
}

fn tasks_page(tasks: List(Task)) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    add_task_form(),
    html.div([attribute.id("task-list")], list.map(tasks, task_item)),
  ])
}

fn add_task_form() -> Element(t) {
  html.form(
    [
      hx.post("/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
      attribute.id("add-task-form"),
      attribute.class("add-task-form"),
      attribute("hx-on::after-request", "this.reset()"),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("title")], [element.text("Task title")]),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.placeholder("What needs to be done?"),
          attribute.required(True),
        ]),
      ]),
      html.button(
        [attribute.type_("submit"), attribute.class("btn-primary")],
        [element.text("Add Task")],
      ),
    ],
  )
}

fn task_item(task: Task) -> Element(t) {
  html.div(
    [attribute.class("task-item"), attribute.id("task-" <> task.id)],
    [
      html.span([attribute.class("task-title")], [element.text(task.title)]),
      html.button(
        [
          hx.delete("/tasks/" <> task.id),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
          attribute.class("btn-delete"),
        ],
        [element.text("Delete")],
      ),
    ],
  )
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
  margin-bottom: 1.5rem;
}

/* Navigation */
.nav {
  display: flex;
  gap: 1.5rem;
  padding-bottom: 1rem;
  margin-bottom: 2rem;
  border-bottom: 2px solid #e0e0e8;
}

.nav a {
  color: #16213e;
  text-decoration: none;
  font-weight: 600;
}

.nav a:hover {
  text-decoration: underline;
}

/* Form */
.add-task-form {
  margin-bottom: 2rem;
  padding: 1.5rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.25rem;
  font-weight: 600;
  color: #1a1a2e;
}

.form-group input[type="text"] {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  line-height: 1.5;
  transition: border-color 0.15s ease;
}

.form-group input[type="text"]:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.15);
}

.form-group textarea {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  line-height: 1.5;
  resize: vertical;
  min-height: 4rem;
  transition: border-color 0.15s ease;
}

.form-group textarea:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.15);
}

.form-group select {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  line-height: 1.5;
  background: white;
  transition: border-color 0.15s ease;
}

.form-group select:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.15);
}

/* Buttons */
.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #16213e;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.btn-primary:hover {
  background: #1a2744;
}

.btn-primary:active {
  background: #0f1729;
}

.btn-delete {
  padding: 0.25rem 0.75rem;
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.btn-delete:hover {
  background: #c82333;
}

/* Task list */
.task-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1rem;
  margin-bottom: 0.5rem;
  background: white;
  border-radius: 6px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}

.task-title {
  flex: 1;
  margin-right: 1rem;
}

/* Footer */
.footer {
  margin-top: 3rem;
  padding-top: 1rem;
  border-top: 1px solid #e0e0e8;
  color: #888;
  font-size: 0.875rem;
}
```

---

## 4. How It All Fits Together

Here is the lifecycle of adding a task, from keystroke to screen:

```
Browser                                Server
  |                                       |
  |  User types "Write tests" and         |
  |  clicks "Add Task"                    |
  |                                       |
  |  HTMX intercepts the submit event     |
  |                                       |
  |  POST /tasks  HTTP/1.1                |
  |  Content-Type:                        |
  |    application/x-www-form-urlencoded  |
  |  Body: title=Write+tests              |
  |-------------------------------------->|
  |                                       |  wisp.require_form parses the body
  |                                       |  list.key_find(values, "title")
  |                                       |    -> Ok("Write tests")
  |                                       |  string.trim("Write tests")
  |                                       |    -> "Write tests" (not empty, good)
  |                                       |  new_id() -> "1708876543210"
  |                                       |  Actor ! AddTask(task)
  |                                       |  task_item(task) -> <div>...</div>
  |                                       |  element.to_string -> HTML fragment
  |                                       |
  |  201 Created                          |
  |  Content-Type: text/html              |
  |                                       |
  |  <div class="task-item"               |
  |       id="task-1708876543210">        |
  |    <span>Write tests</span>           |
  |    <button hx-delete="...">           |
  |      Delete                           |
  |    </button>                          |
  |  </div>                               |
  |<--------------------------------------|
  |                                       |
  |  HTMX finds #task-list                |
  |  Appends the fragment as last child   |
  |  (hx-swap="beforeend")               |
  |                                       |
  |  hx-on::after-request fires           |
  |  this.reset() clears the form         |
  |                                       |
  New task visible. Form cleared.
  No page reload.
```

The key insight: the server does not know or care that HTMX made the request.
It receives standard form data, processes it, and returns HTML. If you sent
the same POST with `curl`, you would get the same response. The server is a
pure function from request to response. HTMX is just a smarter client.

---

## 5. Exercise

Now it is your turn. These exercises build on the code from this chapter.

### Task 1 -- Add a Description Field

Add a "Description" field to the add-task form using a `<textarea>` element.
Update the `Task` type to include a `description: String` field. Update the
`create_task` function to extract the description from the form data. Update
`task_item` to display the description below the title (when it is not empty).

**Acceptance criteria:** You can create a task with both a title and a
description. The description appears in the task list. Tasks with an empty
description show only the title.

### Task 2 -- Add an Assignee Dropdown

Add an "Assignee" `<select>` dropdown to the form with the following options:

- "Unassigned" (value: `""`)
- "Alice" (value: `"alice"`)
- "Bob" (value: `"bob"`)
- "Carol" (value: `"carol"`)

Update the `Task` type to include an `assignee: String` field. Display the
assignee name next to the task title (when one is assigned).

**Acceptance criteria:** You can assign a task to a team member. The assignee's
name appears in the task list. "Unassigned" tasks show no assignee label.

### Task 3 -- Clear the Form After Submission

If you have not already implemented form clearing from the walkthrough, do so
now. After a successful task creation, the form should reset to its empty state.

For an extra challenge, instead of using `hx-on::after-request`, try a different
approach: have the server return both the new task item AND a fresh empty form.
You can do this by targeting a wrapper `<div>` that contains both the form and
the task list, and returning the full contents. (This is a stepping stone toward
out-of-band swaps, which we will cover properly in [Chapter 9](09-swap-strategies-and-targets.md).)

**Acceptance criteria:** After submitting a task, the title input, description
textarea, and assignee dropdown all return to their default/empty state.

### Task 4 -- Boost the Navigation

Add `hx-boost="true"` to the navigation bar so that clicking links causes a
smooth AJAX page transition instead of a full reload.

**Acceptance criteria:** Clicking navigation links updates the page content
without a white flash or scroll reset. Check the Network tab in your browser's
developer tools -- you should see AJAX requests instead of full document
navigations.

---

## 6. Exercise Solution Hints

Try each exercise on your own before reading these hints.

### Hint for Task 1

For the textarea, use `html.textarea`:

```gleam
html.div([attribute.class("form-group")], [
  html.label([attribute.for("description")], [
    element.text("Description (optional)"),
  ]),
  html.textarea(
    [
      attribute.name("description"),
      attribute.id("description"),
      attribute.placeholder("Add some details..."),
      attribute.rows(3),
    ],
    "",
  ),
])
```

Update the `Task` type:

```gleam
pub type Task {
  Task(id: String, title: String, description: String, done: Bool)
}
```

In `create_task`, extract the description. Since it is optional, treat a
missing field as an empty string:

```gleam
let description = case list.key_find(form_data.values, "description") {
  Ok(desc) -> string.trim(desc)
  Error(_) -> ""
}
let task = Task(
  id: id,
  title: trimmed_title,
  description: description,
  done: False,
)
```

In `task_item`, conditionally render the description:

```gleam
fn task_item(task: Task) -> Element(t) {
  let description_el = case task.description {
    "" -> element.none()
    desc ->
      html.p([attribute.class("task-description")], [element.text(desc)])
  }

  html.div(
    [attribute.class("task-item"), attribute.id("task-" <> task.id)],
    [
      html.div([attribute.class("task-content")], [
        html.span([attribute.class("task-title")], [
          element.text(task.title),
        ]),
        description_el,
      ]),
      html.button(
        [
          hx.delete("/tasks/" <> task.id),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
          attribute.class("btn-delete"),
        ],
        [element.text("Delete")],
      ),
    ],
  )
}
```

### Hint for Task 2

For the select dropdown:

```gleam
html.div([attribute.class("form-group")], [
  html.label([attribute.for("assignee")], [element.text("Assignee")]),
  html.select(
    [attribute.name("assignee"), attribute.id("assignee")],
    [
      html.option([attribute.value("")], "Unassigned"),
      html.option([attribute.value("alice")], "Alice"),
      html.option([attribute.value("bob")], "Bob"),
      html.option([attribute.value("carol")], "Carol"),
    ],
  ),
])
```

Update the `Task` type to include `assignee: String`.

Extract the value in `create_task`:

```gleam
let assignee = case list.key_find(form_data.values, "assignee") {
  Ok(a) -> a
  Error(_) -> ""
}
```

Display it in `task_item`:

```gleam
let assignee_el = case task.assignee {
  "" -> element.none()
  name ->
    html.span(
      [attribute.class("task-assignee")],
      [element.text("@ " <> name)],
    )
}
```

### Hint for Task 3

The simplest approach uses the attribute we showed in the walkthrough:

```gleam
attribute("hx-on::after-request", "this.reset()")
```

For the alternative approach (server returns fresh form + new task), you would
need to wrap the form and task list in a single container, target that container,
and have the POST handler return the complete contents (fresh form + full task
list). This works but is less efficient because it re-renders everything. The
proper solution using `hx-swap-oob` comes in [Chapter 9](09-swap-strategies-and-targets.md).

### Hint for Task 4

On the `<nav>` element in your layout:

```gleam
html.nav([hx.boost(True), attribute.class("nav")], [
  html.a([attribute.href("/")], [element.text("Home")]),
  html.a([attribute.href("/tasks")], [element.text("Tasks")]),
  html.a([attribute.href("/about")], [element.text("About")]),
])
```

That single `hx.boost(True)` attribute is all you need. Every `<a>` and
`<form>` inside the nav will be automatically boosted.

To verify it is working, open the Network tab in your browser's developer tools
before clicking a link. You should see an XHR/Fetch request (not a full document
navigation). The page content will change, but the URL bar will also update
(HTMX uses the History API to push the new URL).

---

## 7. Key Takeaways

1. **HTML forms are the native way to send data to a server.** They encode
   inputs as key-value pairs and send them in the request body. This has worked
   since the beginning of the web, and it still works today.

2. **`hx-post` turns a form into an AJAX request.** Instead of a full page
   reload, HTMX submits the form data via AJAX and swaps the server's HTML
   response into a target element. The server receives the same form data
   either way.

3. **`hx-target` and `hx-swap` control where the response goes.** Target an
   element by CSS selector, and choose a swap strategy (`innerHTML`,
   `outerHTML`, `beforeend`, `afterbegin`, etc.). For adding items to a list,
   `beforeend` appends to the target.

4. **`wisp.require_form` parses form data in Gleam.** It extracts
   `form_data.values` as a `List(#(String, String))`. Use `list.key_find` to
   pull out specific fields. Always handle the `Error` case -- never trust
   client-side validation alone.

5. **`hx-boost="true"` is progressive enhancement.** It converts standard form
   submissions and link clicks into AJAX requests. Without JavaScript, everything
   still works via full page reloads. Put it on a parent element to boost
   everything inside.

6. **Return fragments, not full pages.** When HTMX makes a request, your server
   should return only the HTML that needs to change. Use `element.to_string`
   for fragments, not `element.to_document_string` (which adds a doctype).

7. **Validate on the server, always.** The `required` attribute and other
   client-side validation are a convenience for the user. Server-side validation
   with `string.trim` and proper error responses is a requirement for
   correctness.

8. **Status codes matter.** Return `201 Created` when a resource is created,
   `400 Bad Request` when input is invalid, and `405 Method Not Allowed` when
   the wrong HTTP method is used. HTMX triggers swaps on any 2xx response and
   shows errors on 4xx/5xx.

9. **Form clearing is a UX detail worth getting right.** Use
   `hx-on::after-request="this.reset()"` for a simple one-liner, or return a
   fresh form from the server (covered properly with out-of-band swaps in
   [Chapter 9](09-swap-strategies-and-targets.md)).

---

## What's Next

In [Chapter 8](08-validation-and-error-feedback.md), we will add **server-side validation and error feedback**. Right
now our form accepts anything -- empty titles, absurdly long strings, whatever
the user types. We will introduce the `Result` type for validation, return
HTTP 422 responses for invalid input, and render inline error messages that
guide the user toward valid data.

The Teamwork board is becoming genuinely useful.
