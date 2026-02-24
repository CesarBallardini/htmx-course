# Chapter 6 -- Server State: Data That Lives Between Requests

**Phase:** Beginner (final beginner chapter)
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why hardcoded data is insufficient for real applications and describe the options for server-side state.
- Describe what a BEAM process is and how it differs from an operating system thread.
- Use `gleam/otp/actor` to create a stateful actor that holds data in memory and responds to messages.
- Define custom Gleam types to model your application's domain (tasks, messages).
- Implement the Context pattern to pass shared resources (like actor references) through your request handlers.
- Build a task board where adding, deleting, and listing tasks all persist across page refreshes -- without a database.

---

## Theory

### The Problem with Hardcoded Data

In every chapter so far, the tasks on our board have been hardcoded directly
in the route handler functions. Every time the page loads, you see the same
tasks. If you "add" a task, it appears momentarily but vanishes the next time
the page renders. If you "delete" one, it comes right back.

This is because our handlers are pure functions with no memory. They run, they
return HTML, and they forget everything. The request/response cycle is
stateless by design -- the server does not remember what happened last time.

A real application needs data that **lives between requests**. When a user adds
a task, it should still be there when they refresh the page. When they delete
one, it should stay gone.

### Options for Server-Side State

There are three common approaches to persisting data on the server:

| Approach        | Durability               | Speed        | Complexity |
|-----------------|--------------------------|--------------|------------|
| **Database**    | Survives restarts        | Moderate     | Higher     |
| **Files**       | Survives restarts        | Slow         | Moderate   |
| **In-memory**   | Lost on restart          | Very fast    | Lower      |

For production applications, a database is almost always the right choice. We
will add SQLite in Chapter 17. But for learning purposes, in-memory state is
perfect: it is fast, requires no setup, and lets us focus on the patterns
rather than the plumbing.

The question is: where does this in-memory state live?

In many languages, you might reach for a global variable or a singleton object.
Gleam does not have global mutable state -- all values are immutable. Instead,
we use a feature of the runtime: **BEAM processes**.

### BEAM Processes

The BEAM virtual machine (the runtime that powers Erlang, Elixir, and Gleam)
has its own concept of processes that is completely separate from operating
system processes or threads.

A BEAM process is:

- **Lightweight.** Each process uses only a few hundred bytes of memory to
  start. You can run millions of them on a single machine.
- **Isolated.** Processes do not share memory. One process cannot reach into
  another and modify its data. This eliminates an entire class of concurrency
  bugs.
- **Concurrent.** The BEAM scheduler runs processes across all available CPU
  cores, preemptively switching between them. No process can monopolize the
  CPU.
- **Communicating.** Processes talk to each other by **sending messages**.
  Each process has a mailbox where incoming messages queue up, and the process
  handles them one at a time, in order.

When you started your Mist server in Chapter 1, the BEAM was already creating
a new process for every incoming HTTP request. You just did not need to think
about it. Now we are going to create a process deliberately -- one that holds
our task data and responds to requests for it.

### OTP Actors

An **actor** is a design pattern where a long-lived process holds state and
updates it in response to messages. The `gleam/otp/actor` module provides
this pattern.

Think of an actor like a person sitting at a desk with a notebook:

- The **notebook** is the actor's state (our list of tasks).
- People send **letters** to the person's mailbox (messages like "give me all
  the tasks" or "add this new task").
- The person reads each letter one at a time, writes in the notebook if needed,
  and sends a reply if the letter asked for one.
- Because only one letter is processed at a time, there are no conflicts. Two
  people cannot write in the notebook simultaneously.

This is the actor model. It gives us mutable state in an immutable language,
safely, without locks or mutexes.

Here is the lifecycle:

```
  1. Start the actor with initial state
                │
                ▼
  2. Actor waits for a message
                │
                ▼
  3. Message arrives in the mailbox
                │
                ▼
  4. Handle the message:
     - Read the current state
     - Optionally send a reply
     - Return the new state
                │
                ▼
  5. Go back to step 2
```

The actor runs in its own BEAM process, completely independent of the HTTP
request processes. When a request handler needs data, it sends a message to the
actor and waits for a reply. The actor processes the message and responds. This
all happens in microseconds.

### The `actor.call` Pattern

When you need a reply from an actor (for example, "give me the list of tasks"),
you use `actor.call`. This function:

1. Creates a temporary reply channel (a `Subject`).
2. Sends a message to the actor that includes the reply channel.
3. Waits for the actor to send a response on that channel.
4. Returns the response.

The signature looks like this:

```gleam
actor.call(subject, timeout_in_ms, fn(reply_subject) { YourMessage(reply_subject) })
```

The third argument is a function that constructs the message. The function
receives a reply subject, which you embed in the message so the actor knows
where to send the answer. The timeout (in milliseconds) prevents your request
from hanging forever if the actor is stuck or dead.

For messages that do not need a reply (like "delete task 5"), you use
`actor.send`, which fires the message and immediately returns without waiting.

### The Context Pattern

We now have an actor that holds state, and request handlers that need to talk
to it. How do we connect them?

Some frameworks use global state or dependency injection containers. Gleam
takes a simpler approach: we define a **Context** type that holds references
to all shared resources, and we pass it explicitly through every handler.

```
  main()
    │
    ├── Starts the task actor → gets a Subject(Message)
    │
    ├── Creates Context { tasks: subject }
    │
    └── Passes Context into the request handler via a closure
            │
            ├── home_page(req, ctx)  → uses ctx.tasks
            ├── tasks_route(req, ctx) → uses ctx.tasks
            └── task_route(req, ctx, id) → uses ctx.tasks
```

This is dependency injection without the framework. Every handler declares
exactly what it needs, the compiler verifies it, and there is no hidden magic.

One important note: **in-memory state is lost when the server restarts.** If
you stop `gleam run` and start it again, the actor is created fresh with the
initial seed data. This is fine for development. In Chapter 17, we will add
SQLite for persistence that survives restarts.

---

## Code Walkthrough

We are going to make four changes to the project:

1. Define a `Task` custom type.
2. Create a state actor that manages a list of tasks.
3. Define a `Context` type that holds the actor reference.
4. Update `main()` and the route handlers to use the context.

### Step 1 -- Define the Task Type

Create a new file `src/teamwork/task.gleam`. This module defines what a task
looks like in our application.

```gleam
// src/teamwork/task.gleam

pub type Task {
  Task(id: String, title: String, done: Bool)
}
```

This is a Gleam **custom type** with a single variant called `Task`. It has
three fields:

- `id` -- a unique identifier so we can target specific tasks for deletion or
  toggling.
- `title` -- the text the user sees.
- `done` -- whether the task has been completed.

Using a `String` for the ID keeps things simple. We will generate IDs using
random integers converted to strings.

### Step 2 -- Create the State Actor

Create `src/teamwork/state.gleam`. This is the heart of the chapter.

```gleam
// src/teamwork/state.gleam

import gleam/bool
import gleam/list
import gleam/otp/actor
import gleam/erlang/process.{type Subject}
import teamwork/task.{type Task, Task}

pub type Message {
  GetTasks(reply_to: Subject(List(Task)))
  AddTask(task: Task)
  DeleteTask(id: String)
  ToggleTask(id: String)
}
```

The `Message` type defines every kind of request our actor understands:

- **`GetTasks`** carries a `reply_to` subject. The actor will send the full
  list of tasks back on this channel. This is the "letter that expects a reply."
- **`AddTask`** carries a complete `Task` value to be added.
- **`DeleteTask`** carries the ID of the task to remove.
- **`ToggleTask`** carries the ID of the task whose `done` status should flip.

Now the start function:

```gleam
pub fn start() -> Result(Subject(Message), actor.StartError) {
  let initial_tasks = [
    Task(id: "1", title: "Set up project", done: True),
    Task(id: "2", title: "Design the board", done: False),
    Task(id: "3", title: "Add task management", done: False),
  ]

  actor.new(initial_tasks)
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}
```

Let us break this down:

1. We create a list of seed tasks so the board is not empty on first launch.
2. `actor.new(initial_tasks)` creates an actor builder with our initial state.
3. `|> actor.on_message(handle_message)` tells the actor which function to call
   when a message arrives.
4. `|> actor.start` starts the actor in a new BEAM process. It returns
   `Result(Started(Subject(Message)), StartError)`.

The `start` function returns a `Started` record that wraps the subject. We need
a small helper to extract just the subject:

```gleam
fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}
```

This unwraps `Started(data: subject)` into just the `subject`, which is what
the rest of our application needs. The `Subject(Message)` is the address other
processes use to send messages to this actor.

Now the message handler:

```gleam
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

    ToggleTask(id) -> {
      let updated = list.map(tasks, fn(t) {
        case t.id == id {
          True -> Task(..t, done: bool.negate(t.done))
          False -> t
        }
      })
      actor.continue(updated)
    }
  }
}
```

This function receives two arguments: the current state (our list of tasks) and
the message. It returns `actor.Next`, which tells the actor what to do next.
`actor.continue(new_state)` means "keep running with this updated state."

Walking through each case:

- **`GetTasks(reply_to)`** -- Sends the current task list back to whoever asked,
  then continues with the state unchanged.
- **`AddTask(task)`** -- Prepends the new task to the front of the list
  (`[task, ..tasks]` is Gleam's list cons syntax) and continues.
- **`DeleteTask(id)`** -- Filters out the task with the matching ID using
  `list.filter` and continues with the remaining tasks.
- **`ToggleTask(id)`** -- Maps over the list. When we find the task with the
  matching ID, we use the record update syntax `Task(..t, done: bool.negate(t.done))`
  to create a new `Task` with the `done` field flipped. All other tasks pass
  through unchanged. Note that Gleam has no `!` operator for booleans; we use
  `bool.negate` instead.

### Step 3 -- Define the Context Type

Create `src/teamwork/web.gleam`:

```gleam
// src/teamwork/web.gleam

import gleam/erlang/process.{type Subject}
import teamwork/state.{type Message}

pub type Context {
  Context(tasks: Subject(Message))
}
```

The `Context` type has a single field: `tasks`, which is the subject (address)
of our state actor. As the application grows, you would add more fields here --
a database connection pool, a configuration record, a session store, etc.

This is the dependency injection pattern, Gleam style. Instead of hiding
dependencies behind a global or a framework-managed container, we declare them
in a type and pass them explicitly. The compiler ensures every handler that
needs the context actually receives it.

### Step 4 -- Update the Main Module

Now we wire everything together. Here is the updated `src/teamwork.gleam`:

```gleam
import gleam/erlang/process
import gleam/int
import gleam/list
import gleam/otp/actor
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist
import teamwork/state.{type Message, AddTask, DeleteTask, GetTasks}
import teamwork/task.{type Task, Task}
import teamwork/web.{type Context, Context}
```

The imports now include our new modules: `state`, `task`, and `web`. Notice how
we import specific constructors like `GetTasks`, `AddTask`, and `DeleteTask`
so we can use them directly in our code without the `state.` prefix.

**The `main()` function:**

```gleam
pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the task actor -- this creates a BEAM process that holds our state
  let assert Ok(task_store) = state.start()

  // Build the context that will be passed to every request handler
  let ctx = Context(tasks: task_store)

  let assert Ok(_) =
    wisp_mist.handler(fn(req) { handle_request(req, ctx) }, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

Two things are new here:

1. We start the state actor before the HTTP server. `state.start()` returns
   a `Subject(Message)` -- the address we use to send messages to the actor.
   We use `let assert Ok(...)` because if the actor fails to start, there is
   no point running the server.

2. We create a `Context` and pass it into the handler using a closure.
   `wisp_mist.handler` expects a function with the signature
   `fn(Request) -> Response`, so we cannot just pass `handle_request` directly
   (it now takes two arguments). Instead, we wrap it in `fn(req) { handle_request(req, ctx) }`.
   The closure captures `ctx`, making it available to every request.

**The request handler:**

```gleam
fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["tasks"] -> tasks_route(req, ctx)
    ["tasks", id] -> task_route(req, ctx, id)
    _ -> not_found_page()
  }
}
```

This is the same routing pattern from previous chapters, but every route now
receives `ctx` so it can talk to the state actor.

**Fetching tasks from the actor:**

```gleam
fn home_page(_req: wisp.Request, ctx: Context) -> wisp.Response {
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let page = layout("Teamwork", task_board(tasks))
  let html_string = element.to_document_string(page)
  wisp.html_response(html_string, 200)
}
```

`actor.call(ctx.tasks, 1000, GetTasks)` is the key line. Let us unpack it:

- `ctx.tasks` -- the subject (address) of the state actor.
- `1000` -- timeout in milliseconds. If the actor does not respond within one
  second, the call crashes. In practice, an in-memory actor responds in
  microseconds, so this is extremely generous.
- `GetTasks` -- this is the message constructor. Remember, `GetTasks` has the
  type `fn(Subject(List(Task))) -> Message`. The `actor.call` function creates
  a reply subject, passes it to `GetTasks` to construct the message, sends the
  message to the actor, and waits for the reply. It returns `List(Task)`.

This is synchronous from the handler's perspective: the line blocks until the
actor responds. But it is non-blocking at the BEAM level -- the handler's
process simply yields while waiting, allowing other processes to run.

**The task board view:**

```gleam
fn task_board(tasks: List(Task)) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    html.p([attribute.class("task-count")], [
      element.text(task_count_text(tasks)),
    ]),
    add_task_form(),
    task_list(tasks),
  ])
}

fn task_count_text(tasks: List(Task)) -> String {
  let total = list.length(tasks)
  let completed = list.length(list.filter(tasks, fn(t) { t.done }))
  int.to_string(total)
  <> " tasks, "
  <> int.to_string(completed)
  <> " completed"
}
```

**The task list rendering:**

```gleam
fn task_list(tasks: List(Task)) -> Element(t) {
  html.ul(
    [attribute.id("task-list"), attribute.class("task-list")],
    list.map(tasks, task_item),
  )
}

fn task_item(task: Task) -> Element(t) {
  let done_class = case task.done {
    True -> "task-item done"
    False -> "task-item"
  }

  html.li([attribute.class(done_class)], [
    html.span([attribute.class("task-title")], [element.text(task.title)]),
    html.button(
      [
        attribute.class("delete-btn"),
        attribute("hx-delete", "/tasks/" <> task.id),
        attribute("hx-target", "#task-list"),
        attribute("hx-swap", "outerHTML"),
      ],
      [element.text("Delete")],
    ),
  ])
}
```

Each task renders as a list item with the task title and a delete button. The
delete button uses the HTMX attributes we learned in earlier chapters:

- `hx-delete="/tasks/3"` -- sends a DELETE request to `/tasks/3`.
- `hx-target="#task-list"` -- the response should replace the element with
  `id="task-list"`.
- `hx-swap="outerHTML"` -- replace the entire target element (not just its
  contents).

**The add task form:**

```gleam
fn add_task_form() -> Element(t) {
  html.form(
    [
      attribute.class("add-task-form"),
      attribute("hx-post", "/tasks"),
      attribute("hx-target", "#task-list"),
      attribute("hx-swap", "outerHTML"),
    ],
    [
      html.input([
        attribute.type_("text"),
        attribute.name("title"),
        attribute.placeholder("Add a new task..."),
        attribute.required(True),
      ]),
      html.button([attribute.type_("submit")], [element.text("Add")]),
    ],
  )
}
```

This form sends a POST request to `/tasks` when submitted. HTMX intercepts the
submit event, sends the form data via an AJAX request, and swaps the response
into `#task-list`.

**The tasks route (POST -- add a task):**

```gleam
fn tasks_route(req: wisp.Request, ctx: Context) -> wisp.Response {
  case req.method {
    http.Post -> add_task(req, ctx)
    _ -> wisp.method_not_allowed([http.Post])
  }
}

fn add_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  case list.key_find(formdata.values, "title") {
    Ok(title) -> {
      let id = int.to_string(int.random(1_000_000_000))
      let task = Task(id: id, title: title, done: False)
      actor.send(ctx.tasks, AddTask(task))

      // Fetch the updated list and return the rendered task list
      let tasks = actor.call(ctx.tasks, 1000, GetTasks)
      let html_string = element.to_string(task_list(tasks))
      wisp.html_response(html_string, 200)
    }
    Error(_) -> wisp.bad_request()
  }
}
```

A few things to notice:

1. `wisp.require_form(req)` parses the request body as form data. The `use`
   syntax means "if parsing fails, return a 400 Bad Request automatically."

2. We generate a simple ID with `int.random(1_000_000_000)` -- a random integer
   up to one billion, converted to a string. This is not globally unique, but
   it is good enough for a development task board. In Chapter 17, the database
   will handle ID generation.

3. We use `actor.send` (not `actor.call`) for `AddTask` because we do not need
   a reply -- we just fire the message and move on. Then we immediately call
   `GetTasks` to fetch the updated list for rendering.

4. We use `element.to_string` (not `to_document_string`) because we are
   returning an HTML **fragment**, not a full page. HTMX will swap this fragment
   into the existing page.

**The single-task route (DELETE):**

```gleam
fn task_route(req: wisp.Request, ctx: Context, id: String) -> wisp.Response {
  case req.method {
    http.Delete -> delete_task(ctx, id)
    _ -> wisp.method_not_allowed([http.Delete])
  }
}

fn delete_task(ctx: Context, id: String) -> wisp.Response {
  actor.send(ctx.tasks, DeleteTask(id))

  // Return the updated task list
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let html_string = element.to_string(task_list(tasks))
  wisp.html_response(html_string, 200)
}
```

Same pattern: send the `DeleteTask` message, fetch the updated list, return
the rendered fragment.

**The helper functions:**

```gleam
fn not_found_page() -> wisp.Response {
  let page =
    layout("Not Found", html.div([attribute.class("container")], [
      html.h1([], [element.text("404 -- Not Found")]),
      html.p([], [element.text("That page does not exist.")]),
    ]))
  let html_string = element.to_document_string(page)
  wisp.html_response(html_string, 404)
}

fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}
```

And the layout function (same as previous chapters, included here for
completeness):

```gleam
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
```

---

## Full Code Listing

Here is every file you need for this chapter, ready to copy and paste.

### File Structure

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
│       ├── task.gleam
│       ├── state.gleam
│       └── web.gleam
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

### `src/teamwork/task.gleam`

```gleam
pub type Task {
  Task(id: String, title: String, done: Bool)
}
```

### `src/teamwork/state.gleam`

```gleam
import gleam/bool
import gleam/list
import gleam/otp/actor
import gleam/erlang/process.{type Subject}
import teamwork/task.{type Task, Task}

pub type Message {
  GetTasks(reply_to: Subject(List(Task)))
  AddTask(task: Task)
  DeleteTask(id: String)
  ToggleTask(id: String)
}

pub fn start() -> Result(Subject(Message), actor.StartError) {
  let initial_tasks = [
    Task(id: "1", title: "Set up project", done: True),
    Task(id: "2", title: "Design the board", done: False),
    Task(id: "3", title: "Add task management", done: False),
  ]

  actor.new(initial_tasks)
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

    ToggleTask(id) -> {
      let updated = list.map(tasks, fn(t) {
        case t.id == id {
          True -> Task(..t, done: bool.negate(t.done))
          False -> t
        }
      })
      actor.continue(updated)
    }
  }
}
```

### `src/teamwork/web.gleam`

```gleam
import gleam/erlang/process.{type Subject}
import teamwork/state.{type Message}

pub type Context {
  Context(tasks: Subject(Message))
}
```

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process
import gleam/http
import gleam/int
import gleam/list
import gleam/otp/actor
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist
import teamwork/state.{type Message, AddTask, DeleteTask, GetTasks}
import teamwork/task.{type Task, Task}
import teamwork/web.{type Context, Context}

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the task actor
  let assert Ok(task_store) = state.start()

  // Build the context
  let ctx = Context(tasks: task_store)

  let assert Ok(_) =
    wisp_mist.handler(fn(req) { handle_request(req, ctx) }, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["tasks"] -> tasks_route(req, ctx)
    ["tasks", id] -> task_route(req, ctx, id)
    _ -> not_found_page()
  }
}

fn home_page(_req: wisp.Request, ctx: Context) -> wisp.Response {
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)

  let page = layout("Teamwork", task_board(tasks))
  let html_string = element.to_document_string(page)
  wisp.html_response(html_string, 200)
}

fn tasks_route(req: wisp.Request, ctx: Context) -> wisp.Response {
  case req.method {
    http.Post -> add_task(req, ctx)
    _ -> wisp.method_not_allowed([http.Post])
  }
}

fn add_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  case list.key_find(formdata.values, "title") {
    Ok(title) -> {
      let id = int.to_string(int.random(1_000_000_000))
      let task = Task(id: id, title: title, done: False)
      actor.send(ctx.tasks, AddTask(task))

      let tasks = actor.call(ctx.tasks, 1000, GetTasks)
      let html_string = element.to_string(task_list(tasks))
      wisp.html_response(html_string, 200)
    }
    Error(_) -> wisp.bad_request()
  }
}

fn task_route(req: wisp.Request, ctx: Context, id: String) -> wisp.Response {
  case req.method {
    http.Delete -> delete_task(ctx, id)
    _ -> wisp.method_not_allowed([http.Delete])
  }
}

fn delete_task(ctx: Context, id: String) -> wisp.Response {
  actor.send(ctx.tasks, DeleteTask(id))

  let tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let html_string = element.to_string(task_list(tasks))
  wisp.html_response(html_string, 200)
}

// ── Views ──────────────────────────────────────────────────────────

fn task_board(tasks: List(Task)) -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    html.p([attribute.class("task-count"), attribute.id("task-count")], [
      element.text(task_count_text(tasks)),
    ]),
    add_task_form(),
    task_list(tasks),
  ])
}

fn task_count_text(tasks: List(Task)) -> String {
  let total = list.length(tasks)
  let completed = list.length(list.filter(tasks, fn(t) { t.done }))
  int.to_string(total)
  <> " tasks, "
  <> int.to_string(completed)
  <> " completed"
}

fn add_task_form() -> Element(t) {
  html.form(
    [
      attribute.class("add-task-form"),
      attribute("hx-post", "/tasks"),
      attribute("hx-target", "#task-list"),
      attribute("hx-swap", "outerHTML"),
    ],
    [
      html.input([
        attribute.type_("text"),
        attribute.name("title"),
        attribute.placeholder("Add a new task..."),
        attribute.required(True),
      ]),
      html.button([attribute.type_("submit")], [element.text("Add")]),
    ],
  )
}

fn task_list(tasks: List(Task)) -> Element(t) {
  html.ul(
    [attribute.id("task-list"), attribute.class("task-list")],
    list.map(tasks, task_item),
  )
}

fn task_item(task: Task) -> Element(t) {
  let done_class = case task.done {
    True -> "task-item done"
    False -> "task-item"
  }

  html.li([attribute.class(done_class)], [
    html.span([attribute.class("task-title")], [element.text(task.title)]),
    html.button(
      [
        attribute.class("delete-btn"),
        attribute("hx-delete", "/tasks/" <> task.id),
        attribute("hx-target", "#task-list"),
        attribute("hx-swap", "outerHTML"),
      ],
      [element.text("Delete")],
    ),
  ])
}

// ── Layout ─────────────────────────────────────────────────────────

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

fn not_found_page() -> wisp.Response {
  let page =
    layout(
      "Not Found",
      html.div([attribute.class("container")], [
        html.h1([], [element.text("404 -- Not Found")]),
        html.p([], [element.text("That page does not exist.")]),
      ]),
    )
  let html_string = element.to_document_string(page)
  wisp.html_response(html_string, 404)
}

fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}
```

### `priv/static/css/style.css`

Add the following rules to your existing stylesheet:

```css
/* ── Base ─────────────────────────────────────────────── */

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
  margin-bottom: 0.5rem;
}

/* ── Task Count ───────────────────────────────────────── */

.task-count {
  color: #666;
  font-size: 0.9rem;
  margin-bottom: 1.5rem;
}

/* ── Add Task Form ────────────────────────────────────── */

.add-task-form {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}

.add-task-form input[type="text"] {
  flex: 1;
  padding: 0.5rem 0.75rem;
  border: 2px solid #e0e0e8;
  border-radius: 6px;
  font-size: 1rem;
  font-family: inherit;
}

.add-task-form input[type="text"]:focus {
  outline: none;
  border-color: #16213e;
}

.add-task-form button {
  padding: 0.5rem 1.25rem;
  background-color: #16213e;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
}

.add-task-form button:hover {
  background-color: #1a2a4e;
}

/* ── Task List ────────────────────────────────────────── */

.task-list {
  list-style: none;
}

.task-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1rem;
  margin-bottom: 0.5rem;
  background-color: white;
  border-radius: 6px;
  border: 1px solid #e0e0e8;
}

.task-item.done .task-title {
  text-decoration: line-through;
  color: #999;
}

.task-title {
  flex: 1;
}

.delete-btn {
  padding: 0.25rem 0.75rem;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 0.85rem;
  cursor: pointer;
}

.delete-btn:hover {
  background-color: #c0392b;
}
```

---

## How It All Fits Together

Let us trace what happens when a user adds a task:

```
Browser                         Server
  |                                |
  |  User types "Write tests"     |
  |  and clicks "Add"             |
  |                                |
  |  POST /tasks  HTTP/1.1        |
  |  title=Write+tests            |
  |------------------------------->|
  |                                |
  |                     handle_request(req, ctx)
  |                       |
  |                       ├── wisp.log_request
  |                       ├── wisp.serve_static (not /static, skip)
  |                       ├── path = ["tasks"], method = POST
  |                       └── add_task(req, ctx)
  |                             |
  |                             ├── Parse form data → title = "Write tests"
  |                             ├── Generate id = "847291563"
  |                             ├── Create Task(id, title, done: False)
  |                             |
  |                             ├── actor.send(ctx.tasks, AddTask(task))
  |                             |     └─── message arrives in actor mailbox
  |                             |           actor prepends task to list
  |                             |           actor.continue(new_list)
  |                             |
  |                             ├── actor.call(ctx.tasks, 1000, GetTasks)
  |                             |     └─── actor receives GetTasks
  |                             |           sends task list on reply channel
  |                             |           returns List(Task) with 4 items
  |                             |
  |                             └── Render task_list(tasks)
  |                                  → element.to_string → HTML fragment
  |                                  → wisp.html_response(fragment, 200)
  |                                |
  |  200 OK                        |
  |  <ul id="task-list">...</ul>   |
  |<-------------------------------|
  |                                |
  |  HTMX receives the fragment   |
  |  Finds #task-list on the page  |
  |  Replaces it with the new HTML |
  |                                |
  |  User sees "Write tests"       |
  |  appear in the list.           |
```

The critical insight is the separation of concerns:

- **The actor** owns the data. It processes messages sequentially, ensuring
  consistency.
- **The request handler** coordinates the workflow: parse input, send messages,
  render HTML.
- **HTMX** handles the client-side swap. No JavaScript was written.

If the user refreshes the page, `home_page` calls `GetTasks` again and gets
the same list (including "Write tests") because the actor's state persists
in memory. The data is gone only if the server process itself is stopped.

---

## Exercise

Now it is your turn. Work through the following tasks to reinforce what you
have learned.

### Task 1 -- Add Toggle Functionality

Clicking a task's title should toggle its `done` status. Implement this by:

1. Adding an `hx-patch` attribute to the task title `<span>` element so that
   clicking it sends a PATCH request to `/tasks/<id>`.
2. Adding a `http.Patch` case to `task_route` that sends a `ToggleTask` message
   to the actor.
3. Returning the updated task list fragment, just like the delete handler does.

**Acceptance criteria:** Clicking a task title visually toggles it between
normal and struck-through text. Refreshing the page preserves the toggled
state.

### Task 2 -- Update the Task Count After Every Action

Right now, the task count ("3 tasks, 1 completed") is rendered once on page
load but does not update when you add, delete, or toggle a task. Fix this so
the count stays accurate after every action.

**Hint:** The simplest approach is to include the task count in the HTML
fragment returned by your add, delete, and toggle handlers. You could wrap
both the count paragraph and the task list in a parent `<div>` and target that
instead. Alternatively, you could return two separate fragments using
out-of-band swaps (`hx-swap-oob`), which we will cover properly in Chapter 9.
For now, the wrapper `<div>` approach is recommended.

**Acceptance criteria:** The count text updates immediately after adding,
deleting, or toggling a task, without a full page refresh.

### Task 3 -- Add a "Clear Completed" Button

Add a button below the task list that reads "Clear Completed". When clicked, it
should remove all tasks whose `done` field is `True`.

You will need to:

1. Add a new message variant to the `Message` type in `state.gleam` (for
   example, `ClearCompleted`).
2. Handle that message in `handle_message` by filtering out all done tasks.
3. Add a route and handler that sends this message.
4. Add an HTMX-powered button in the view.

**Acceptance criteria:** Clicking "Clear Completed" removes all done tasks from
the list. The task count updates. Refreshing the page confirms they are gone.

### Task 4 -- Verify State Persistence and Its Limits

1. Add a few tasks and delete one. Refresh the page. Confirm that your changes
   persisted.
2. Stop the server (`Ctrl+C`) and start it again with `gleam run`. Confirm
   that the task list is back to the three seed tasks -- your changes are gone.

This is the fundamental limitation of in-memory state. Understand it now so
that the motivation for adding a database in Chapter 17 is crystal clear.

---

## Exercise Solution Hints

If you get stuck, here are some pointers. Try to solve each task on your own
first.

### Hint for Task 1

Update the `task_item` function to make the title clickable:

```gleam
fn task_item(task: Task) -> Element(t) {
  let done_class = case task.done {
    True -> "task-item done"
    False -> "task-item"
  }

  html.li([attribute.class(done_class)], [
    html.span(
      [
        attribute.class("task-title"),
        attribute("hx-patch", "/tasks/" <> task.id),
        attribute("hx-target", "#task-list"),
        attribute("hx-swap", "outerHTML"),
        attribute("style", "cursor: pointer"),
      ],
      [element.text(task.title)],
    ),
    html.button(
      [
        attribute.class("delete-btn"),
        attribute("hx-delete", "/tasks/" <> task.id),
        attribute("hx-target", "#task-list"),
        attribute("hx-swap", "outerHTML"),
      ],
      [element.text("Delete")],
    ),
  ])
}
```

Then update `task_route` to handle PATCH:

```gleam
fn task_route(req: wisp.Request, ctx: Context, id: String) -> wisp.Response {
  case req.method {
    http.Delete -> delete_task(ctx, id)
    http.Patch -> toggle_task(ctx, id)
    _ -> wisp.method_not_allowed([http.Delete, http.Patch])
  }
}

fn toggle_task(ctx: Context, id: String) -> wisp.Response {
  actor.send(ctx.tasks, ToggleTask(id))

  let tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let html_string = element.to_string(task_list(tasks))
  wisp.html_response(html_string, 200)
}
```

Remember to add `ToggleTask` to your imports in the main module.

### Hint for Task 2

The simplest approach: wrap the count and the list in a container, and target
that container from all three actions.

Create a helper that renders both:

```gleam
fn task_list_with_count(tasks: List(Task)) -> Element(t) {
  html.div([attribute.id("task-section")], [
    html.p([attribute.class("task-count")], [
      element.text(task_count_text(tasks)),
    ]),
    task_list(tasks),
  ])
}
```

Then change your `hx-target` attributes to `"#task-section"` and return
`element.to_string(task_list_with_count(tasks))` from your add, delete, and
toggle handlers. Also update the `task_board` function to use
`task_list_with_count` so the initial page render includes the wrapper div.

### Hint for Task 3

Add the message variant in `state.gleam`:

```gleam
pub type Message {
  GetTasks(reply_to: Subject(List(Task)))
  AddTask(task: Task)
  DeleteTask(id: String)
  ToggleTask(id: String)
  ClearCompleted
}
```

Handle it:

```gleam
ClearCompleted -> {
  let remaining = list.filter(tasks, fn(t) { !t.done })
  actor.continue(remaining)
}
```

Wait -- Gleam does not have `!`. Use `bool.negate`:

```gleam
ClearCompleted -> {
  let remaining = list.filter(tasks, fn(t) { bool.negate(t.done) })
  actor.continue(remaining)
}
```

Or, equivalently, filter for tasks that are not done:

```gleam
ClearCompleted -> {
  let remaining = list.filter(tasks, fn(t) {
    case t.done {
      True -> False
      False -> True
    }
  })
  actor.continue(remaining)
}
```

Add a route (for example, `POST /tasks/clear-completed` or
`DELETE /tasks/completed`) and a button with matching HTMX attributes.

### Hint for Task 4

This is an observation task -- there is no code to write. Just follow the
steps and notice the behaviour. The key learning is that BEAM process state
lives in memory and is tied to the process lifetime. When the server stops,
the process dies, and the state is gone.

---

## Key Takeaways

1. **Hardcoded data does not survive interactions.** For data to persist across
   requests, it must live somewhere on the server -- in a database, a file, or
   in memory.

2. **BEAM processes are lightweight, isolated units of concurrency.** They do
   not share memory. They communicate by sending messages to each other's
   mailboxes. The BEAM VM can run millions of them.

3. **An OTP actor is a process that holds state.** It receives messages one at
   a time, updates its state, and optionally sends replies. The
   `gleam/otp/actor` module provides this pattern through a builder API:
   `actor.new(state) |> actor.on_message(handler) |> actor.start`.

4. **`actor.call` is synchronous request/reply.** It sends a message with a
   reply channel and waits for the response. Use it when you need data back
   (like fetching the task list). **`actor.send` is fire-and-forget.** Use it
   when you do not need a reply (like telling the actor to delete a task).

5. **The Context pattern is dependency injection, Gleam style.** Define a type
   that holds your shared resources, create it in `main()`, and pass it through
   a closure to your request handlers. No globals, no magic -- the compiler
   verifies everything.

6. **Closures bridge the gap between framework expectations and your needs.**
   `wisp_mist.handler` expects `fn(Request) -> Response`, but your handler
   needs extra context. A closure like `fn(req) { handle_request(req, ctx) }`
   captures the context and satisfies the expected signature.

7. **In-memory state is fast but ephemeral.** It is perfect for development
   and prototyping. For production, you need a database. We will add SQLite in
   Chapter 17.

8. **`element.to_string` vs `element.to_document_string`.** Use
   `to_document_string` for full pages (includes the doctype). Use `to_string`
   for HTML fragments returned to HTMX.

9. **The actor processes messages sequentially.** This eliminates race
   conditions. Two simultaneous requests that both modify the task list will
   be processed one after the other by the actor, never simultaneously. The
   BEAM gives you concurrency without the usual pain.

---

## What's Next

Congratulations -- you have completed the beginner phase. You now have a
working task board with a real server, type-safe HTML rendering, client-side
interactivity via HTMX, and stateful server-side data managed by a BEAM actor.

In the intermediate phase, starting with Chapter 7, we will tackle **form
validation** -- making sure the data users submit is correct before we store
it, and showing helpful error messages when it is not. The patterns you learned
in this chapter (actors for state, context for dependency injection, fragments
for HTMX responses) will carry forward through every chapter that follows.
