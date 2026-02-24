# Chapter 13 -- Real-Time with Server-Sent Events

## Learning Objectives

By the end of this chapter, you will:

- Understand the three approaches to real-time updates: polling, SSE, and WebSockets
- Know why **Server-Sent Events** are the natural fit for HTMX applications
- Use the HTMX SSE extension with `sse-connect`, `sse-swap`, and `sse-close`
- Build an SSE endpoint in Gleam using Mist's built-in SSE support
- Implement a **pub/sub broadcast pattern** using BEAM processes and subjects
- See changes made by one user appear on every other connected user's screen in real time
- Turn the Teamwork task board into a genuinely collaborative application

---

## Theory

### The Problem: Stale Screens

Up to this point, Teamwork works well for a single user. You can create boards,
add tasks, search, filter, drag things around with OOB swaps, and navigate like
a single-page app. But open the same board in two browser tabs and add a task in
one of them. Switch to the other tab. Nothing changed. The second tab is stale.

For a tool called "Teamwork," that is a serious problem. If Alice adds a task,
Bob should see it appear on his screen without refreshing. If Carol marks
something as done, everyone looking at that board should see the change
immediately.

This is the domain of **real-time web applications**.

### Three Approaches to Real-Time

There are three standard techniques for pushing updates from a server to
connected clients. Each trades off simplicity against capability.

#### 1. Polling

The simplest approach. The client asks the server for updates on a fixed
interval.

```html
<div hx-get="/tasks" hx-trigger="every 5s" hx-swap="innerHTML">
  <!-- Refreshes every 5 seconds -->
</div>
```

HTMX makes polling trivially easy with `hx-trigger="every 5s"`. We used this
pattern back in Chapter 5 for a timestamp display.

**Advantages:**
- Dead simple to implement -- no special server infrastructure
- Works with any HTTP server
- Stateless on the server side

**Disadvantages:**
- Wasteful. If nothing changed, you still made a request and got the same HTML back
- Not truly real-time. With a 5-second interval, changes take up to 5 seconds to appear
- Scaling problem. 100 users polling every 5 seconds means 20 requests per second, even when nobody is doing anything

Polling is a reasonable choice for dashboards that update infrequently. For a
collaborative task board where responsiveness matters, we need something better.

#### 2. Server-Sent Events (SSE)

The client opens a long-lived HTTP connection to the server. The server holds
that connection open and pushes events down to the client whenever something
changes. The connection is one-directional: server to client only.

**Advantages:**
- Truly real-time. Events arrive the instant they happen
- Built on standard HTTP. Works through proxies, load balancers, and firewalls
- Auto-reconnects. If the connection drops, the browser automatically reconnects
- Simple protocol. Just a specific text format over a standard HTTP response
- Lightweight. One connection per client, no periodic overhead

**Disadvantages:**
- One-directional. The client cannot send messages back through the same connection (but it can still make normal HTTP requests)
- Limited to text data (not binary)
- Some older browsers have connection limits (6 per domain), though modern browsers handle this well

#### 3. WebSockets

A full-duplex communication channel. Both client and server can send messages at
any time over a single persistent connection.

**Advantages:**
- Bidirectional. Either side can send messages at any time
- Very low latency
- Supports binary data

**Disadvantages:**
- More complex protocol. Requires a protocol upgrade from HTTP
- Does not auto-reconnect natively (you need a library or custom code)
- Some proxies and firewalls can interfere
- Overkill for most HTMX applications

#### SSE Is the Sweet Spot for HTMX

Think about what HTMX does: the server sends HTML fragments, and the client
swaps them into the page. The client's role is to *receive* HTML and display it.
The client already sends data to the server through normal `hx-post`, `hx-put`,
and `hx-delete` requests.

This maps perfectly onto SSE. The server pushes HTML fragments through the SSE
connection when things change. The client receives them and swaps them in. The
client continues using normal HTTP requests for actions like adding or deleting
tasks.

```
    Alice's Browser                  Server                  Bob's Browser
    ───────────────                  ──────                  ─────────────
         │                              │                          │
         │  SSE connection (open)       │    SSE connection (open) │
         │◄─────────────────────────────│─────────────────────────►│
         │                              │                          │
         │  POST /tasks (add task)      │                          │
         │─────────────────────────────►│                          │
         │                              │                          │
         │  SSE: task-list HTML         │   SSE: task-list HTML    │
         │◄─────────────────────────────│─────────────────────────►│
         │                              │                          │
         │  (page updates)              │            (page updates)│
         │                              │                          │
```

Alice adds a task with a normal POST request. The server processes it, then
broadcasts the updated task list as an SSE event to every connected client --
including Alice and Bob. Both screens update simultaneously.

### How SSE Works Under the Hood

The SSE protocol is remarkably simple. It is just a long-lived HTTP response
with a specific content type and text format.

**Step 1: The client opens a connection.**

The browser sends a GET request with `Accept: text/event-stream`:

```
GET /events HTTP/1.1
Accept: text/event-stream
```

**Step 2: The server holds the connection open.**

Instead of sending a response and closing the connection, the server keeps it
open. It sets the content type to `text/event-stream` and sends events whenever
it has something to report:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: task-list
data: <ul id="task-list"><li>Task A</li><li>Task B</li></ul>

event: task-counter
data: <span id="task-counter">2 tasks</span>

```

Each event has two parts:
- **`event:`** -- A name that identifies what kind of event this is
- **`data:`** -- The payload. For HTMX, this is an HTML fragment

Events are separated by a blank line. The server can send as many events as it
wants over the lifetime of the connection.

**Step 3: The client receives events and processes them.**

The browser's built-in `EventSource` API handles the connection, parsing, and
auto-reconnection. The HTMX SSE extension builds on top of `EventSource` to
automatically swap received HTML into the correct elements.

**Step 4: Auto-reconnection.**

If the connection drops (network blip, server restart, etc.), the browser
automatically reconnects after a short delay. The server can control the retry
interval by sending a `retry:` field. This resilience is built into the protocol
-- you get it for free.

### The HTMX SSE Extension

HTMX does not include SSE support in its core library. Instead, it provides an
official extension called `htmx-ext-sse`. You load it as a separate script and
activate it with three attributes.

**`hx-ext="sse"`** -- Activates the SSE extension on an element and its
descendants.

**`sse-connect="/events"`** -- Opens an SSE connection to the specified URL.
Place this on a container element. All children can then receive events from
this connection.

**`sse-swap="event-name"`** -- Tells a child element to listen for a specific
SSE event name and swap the event's data (HTML) into itself. The swap strategy
defaults to `innerHTML` but can be overridden with `hx-swap`.

**`sse-close="event-name"`** -- Closes the SSE connection when the specified
event is received. Useful for finite streams.

Here is the pattern:

```html
<!-- Load the SSE extension -->
<script src="https://unpkg.com/htmx-ext-sse@2.2.2/sse.js"></script>

<!-- Container: activates SSE and connects to /events -->
<div hx-ext="sse" sse-connect="/events">

  <!-- This div receives "task-list" events -->
  <div id="task-list" sse-swap="task-list">
    <!-- Initial content here; replaced when server sends a task-list event -->
  </div>

  <!-- This div receives "task-counter" events -->
  <div id="task-counter" sse-swap="task-counter">
    <!-- Replaced when server sends a task-counter event -->
  </div>

</div>
```

When the server sends an event like this:

```
event: task-list
data: <ul><li>Buy milk</li><li>Write tests</li></ul>

```

The HTMX SSE extension finds the element with `sse-swap="task-list"` and
replaces its `innerHTML` with the event's data. The same happens independently
for each event name.

This is the core mechanism. The server pushes HTML. HTMX swaps it in. No
JavaScript needed on your part.

---

## Code Walkthrough

We are going to add real-time updates to the Teamwork task board. When any user
adds, deletes, or modifies a task, every connected user will see the change
instantly.

The architecture has four parts:

1. A **state actor** that holds the task list and a list of subscribers
2. An **SSE endpoint** built with Mist's SSE support (not Wisp -- Wisp does not support streaming responses)
3. A **hybrid router** that sends SSE requests to Mist's handler and everything else to Wisp
4. **HTMX SSE markup** on the client side

### Project Structure

```
teamwork/
  src/
    teamwork.gleam          -- main entry point, hybrid router
    teamwork/router.gleam   -- Wisp request handler
    teamwork/state.gleam    -- state actor with pub/sub
    teamwork/views.gleam    -- HTML rendering functions
  gleam.toml
```

### Step 1: The State Actor with Pub/Sub

The state actor is the heart of the real-time system. It holds the task list,
accepts commands to add and remove tasks, maintains a list of SSE subscribers,
and broadcasts changes to all of them.

This is a natural fit for the BEAM. Each SSE connection is a lightweight process.
The state actor is another process. They communicate through message passing.
The BEAM was designed for exactly this kind of concurrent, message-driven
architecture.

**`src/teamwork/state.gleam`**

```gleam
import gleam/bool
import gleam/erlang/process.{type Subject}
import gleam/list
import gleam/otp/actor

// ── Types ────────────────────────────────────────────────────────────────

pub type Task {
  Task(id: Int, title: String, done: Bool)
}

/// Messages that the state actor can receive.
pub type Message {
  /// Add a new task. The subject receives the created task back.
  AddTask(title: String, reply_to: Subject(Task))
  /// Delete a task by ID.
  DeleteTask(id: Int)
  /// Toggle a task's done status.
  ToggleTask(id: Int)
  /// Get all tasks. The subject receives the list.
  GetTasks(reply_to: Subject(List(Task)))
  /// Register an SSE connection to receive broadcasts.
  Subscribe(subject: Subject(Update))
  /// Unregister an SSE connection.
  Unsubscribe(subject: Subject(Update))
}

/// Events broadcast to SSE subscribers.
pub type Update {
  TasksChanged
}

/// Internal state of the actor.
pub type State {
  State(
    tasks: List(Task),
    next_id: Int,
    subscribers: List(Subject(Update)),
  )
}

// ── Actor Setup ──────────────────────────────────────────────────────────

fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}

/// Start the state actor with some seed data.
pub fn start() -> Result(Subject(Message), actor.StartError) {
  let initial_state =
    State(
      tasks: [
        Task(id: 1, title: "Set up project repository", done: False),
        Task(id: 2, title: "Design database schema", done: False),
        Task(id: 3, title: "Build task API endpoints", done: True),
        Task(id: 4, title: "Create front-end layout", done: False),
      ],
      next_id: 5,
      subscribers: [],
    )

  actor.new(initial_state)
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}

// ── Message Handler ──────────────────────────────────────────────────────

fn handle_message(state: State, message: Message) -> actor.Next(State, Message) {
  case message {
    AddTask(title, reply_to) -> {
      let task = Task(id: state.next_id, title: title, done: False)
      let new_state =
        State(
          ..state,
          tasks: list.append(state.tasks, [task]),
          next_id: state.next_id + 1,
        )
      process.send(reply_to, task)
      broadcast(new_state.subscribers, TasksChanged)
      actor.continue(new_state)
    }

    DeleteTask(id) -> {
      let new_tasks = list.filter(state.tasks, fn(t) { t.id != id })
      let new_state = State(..state, tasks: new_tasks)
      broadcast(new_state.subscribers, TasksChanged)
      actor.continue(new_state)
    }

    ToggleTask(id) -> {
      let new_tasks =
        list.map(state.tasks, fn(t) {
          case t.id == id {
            True -> Task(..t, done: bool.negate(t.done))
            False -> t
          }
        })
      let new_state = State(..state, tasks: new_tasks)
      broadcast(new_state.subscribers, TasksChanged)
      actor.continue(new_state)
    }

    GetTasks(reply_to) -> {
      process.send(reply_to, state.tasks)
      actor.continue(state)
    }

    Subscribe(subject) -> {
      let new_state =
        State(..state, subscribers: [subject, ..state.subscribers])
      actor.continue(new_state)
    }

    Unsubscribe(subject) -> {
      let new_subscribers =
        list.filter(state.subscribers, fn(s) { s != subject })
      let new_state = State(..state, subscribers: new_subscribers)
      actor.continue(new_state)
    }
  }
}

// ── Broadcasting ─────────────────────────────────────────────────────────

/// Send an update to every subscriber.
fn broadcast(subscribers: List(Subject(Update)), update: Update) -> Nil {
  list.each(subscribers, fn(subscriber) {
    process.send(subscriber, update)
  })
}
```

Let us walk through the key design decisions.

**The `Subscribe` and `Unsubscribe` messages.** When a new SSE connection opens,
Mist provides a `Subject(Update)` for that connection. The `init` callback sends
a `Subscribe` message to the state actor, passing that subject. The actor adds
it to its subscriber list. When the SSE connection closes (the user navigates
away, closes the tab, or the network drops), the handler sends `Unsubscribe` to
clean up.

**The `broadcast` function.** After every mutation (add, delete, toggle), the
actor calls `broadcast`, which sends a `TasksChanged` message to every
subscriber. Each subscriber is a process handling an SSE connection, so it will
receive the message and push an updated HTML fragment to the client.

**Why `TasksChanged` instead of sending the HTML directly?** Separation of
concerns. The state actor manages data. The SSE handler decides how to render
that data into HTML. The state actor does not need to know about HTML.

### Step 2: The SSE Endpoint with Mist

Wisp is built on top of Mist, but Wisp's response model assumes you build the
full response body and return it. SSE needs a long-lived connection where you
send data over time. For this, we drop down to Mist's level.

Mist provides a `server_sent_events` function that handles the SSE protocol for
you. You give it an `init` callback (to subscribe and set up initial state) and
a `loop` callback (to handle incoming messages and send events to the client).

**`src/teamwork.gleam`** -- the main module with the hybrid router:

```gleam
import gleam/erlang/process.{type Subject}
import gleam/http/request.{type Request}
import gleam/http/response
import gleam/otp/actor
import gleam/string_tree
import mist.{type Connection, type ResponseData, type SSEConnection}
import teamwork/state.{type Message, type Task, type Update}
import teamwork/router
import teamwork/views
import wisp
import wisp/wisp_mist

// ── Context ──────────────────────────────────────────────────────────────

pub type Context {
  Context(state: Subject(Message))
}

// ── Main ─────────────────────────────────────────────────────────────────

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the state actor
  let assert Ok(state_subject) = state.start()
  let ctx = Context(state: state_subject)

  // Build a hybrid handler:
  //   - /events goes to the SSE handler (Mist level)
  //   - Everything else goes to Wisp
  let handler = fn(req: Request(Connection)) -> response.Response(ResponseData) {
    case request.path_segments(req) {
      ["events"] -> handle_sse(req, ctx)
      _ -> {
        let wisp_handler = fn(req) { router.handle_request(req, ctx) }
        wisp_mist.handler(wisp_handler, secret_key_base)(req)
      }
    }
  }

  let assert Ok(_) =
    handler
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

// ── SSE Handler ──────────────────────────────────────────────────────────

fn handle_sse(
  req: Request(Connection),
  ctx: Context,
) -> response.Response(ResponseData) {
  mist.server_sent_events(
    request: req,
    initial_response: response.new(200),
    // The `init` callback runs once when the connection opens.
    // Mist passes us a subject we use to receive messages.
    init: fn(subject) {
      // Subscribe this connection to state changes.
      actor.send(ctx.state, state.Subscribe(subject))
      // Send ourselves an initial TasksChanged so the client gets data immediately.
      process.send(subject, state.TasksChanged)
      Ok(actor.initialised(subject))
    },
    // The `loop` callback runs every time a message arrives.
    loop: fn(subject, message, conn) {
      case message {
        state.TasksChanged -> {
          // Fetch the updated task list.
          let tasks = actor.call(ctx.state, 1000, state.GetTasks)

          // Send the updated task list and counter as SSE events.
          let task_list_event =
            mist.event(string_tree.from_string(views.render_task_list(tasks)))
            |> mist.event_name("task-list")

          let counter_event =
            mist.event(string_tree.from_string(views.render_task_counter(tasks)))
            |> mist.event_name("task-counter")

          case mist.send_event(conn, task_list_event) {
            Ok(_) -> {
              case mist.send_event(conn, counter_event) {
                Ok(_) -> actor.continue(subject)
                Error(_) -> {
                  actor.send(ctx.state, state.Unsubscribe(subject))
                  actor.stop()
                }
              }
            }
            Error(_) -> {
              // Connection closed. Unsubscribe and stop.
              actor.send(ctx.state, state.Unsubscribe(subject))
              actor.stop()
            }
          }
        }
      }
    },
  )
}
```

Let us break this down carefully.

**The hybrid router.** At the Mist level, we intercept requests to `/events`
and handle them directly. Everything else is forwarded to Wisp through
`wisp_mist.handler`. This is the key architectural pattern: Mist handles SSE,
Wisp handles normal requests.

**`handle_sse` function.** When a client connects to `/events`:
1. We call `mist.server_sent_events`, which takes over the HTTP connection.
2. In `init`, Mist gives us a `Subject(Update)` for this connection. We
   subscribe it to the state actor, then send ourselves a `TasksChanged`
   message so the client immediately receives the current state.
3. In `loop`, we wait for `TasksChanged` messages. When one arrives, we fetch
   the updated tasks, render them to HTML, and send them as SSE events.
4. If `mist.send_event` fails (the client disconnected), we unsubscribe and
   call `actor.stop()` to end the loop.

### Step 3: The Wisp Router

The Wisp router handles normal page loads and form submissions. When a task is
added or deleted, the router sends a message to the state actor, which triggers
a broadcast to all SSE subscribers.

**`src/teamwork/router.gleam`**

```gleam
import gleam/http
import gleam/int
import gleam/list
import gleam/erlang/process
import gleam/otp/actor
import teamwork.{type Context}
import teamwork/state
import teamwork/views
import wisp.{type Request, type Response}

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use req <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    // Home page -- full HTML document
    [] -> {
      use <- wisp.require_method(req, http.Get)
      let tasks = actor.call(ctx.state, 1000, state.GetTasks)
      views.home_page(tasks)
    }

    // POST /tasks -- add a new task
    ["tasks"] -> {
      use <- wisp.require_method(req, http.Post)
      use formdata <- wisp.require_form(req)

      let title =
        list.find(formdata.values, fn(pair) { pair.0 == "title" })
      case title {
        Ok(#(_, value)) -> {
          // Tell the state actor to add the task.
          // The actor will broadcast to all SSE subscribers.
          let _task =
            actor.call(ctx.state, 1000, state.AddTask(value, _))

          // Return an empty response. The SSE broadcast handles the UI update.
          // We also clear the form input with an OOB swap.
          let body =
              "<input id=\"task-input\" name=\"title\" type=\"text\" "
              <> "placeholder=\"What needs to be done?\" value=\"\" "
              <> "hx-swap-oob=\"true\" />"
          wisp.html_response(body, 200)
        }
        Error(_) -> wisp.bad_request()
      }
    }

    // DELETE /tasks/:id -- delete a task
    ["tasks", id_str] -> {
      use <- wisp.require_method(req, http.Delete)
      case int.parse(id_str) {
        Ok(id) -> {
          // Tell the state actor to delete the task.
          // The actor will broadcast to all SSE subscribers.
          actor.send(ctx.state, state.DeleteTask(id))
          wisp.html_response("", 200)
        }
        Error(_) -> wisp.bad_request()
      }
    }

    // POST /tasks/:id/toggle -- toggle done status
    ["tasks", id_str, "toggle"] -> {
      use <- wisp.require_method(req, http.Post)
      case int.parse(id_str) {
        Ok(id) -> {
          actor.send(ctx.state, state.ToggleTask(id))
          wisp.html_response("", 200)
        }
        Error(_) -> wisp.bad_request()
      }
    }

    _ -> wisp.not_found()
  }
}
```

Notice what happens in the `POST /tasks` handler. We call
`actor.call(ctx.state, 1000, state.AddTask(value, _))` to add the task. The
state actor adds it to the list and then calls `broadcast(subscribers,
TasksChanged)`. Every SSE connection receives the `TasksChanged` message and
pushes the updated task list to its client. The POST handler itself returns only
a small OOB swap to clear the form input -- the real UI update comes through SSE.

This is a fundamental shift in how data flows through the application. In
previous chapters, the response to a POST request contained the updated HTML.
Now, the POST response is minimal. The updated HTML arrives separately through
the SSE channel, and it arrives for **every** connected client, not just the one
who submitted the form.

### Step 4: The Views

The views module renders HTML fragments. These are used both for the initial
page load and for SSE events.

**`src/teamwork/views.gleam`**

```gleam
import gleam/int
import gleam/list
import lustre/attribute
import lustre/element
import lustre/element/html
import teamwork/state.{type Task}
import wisp.{type Response}

// ── Full Page ────────────────────────────────────────────────────────────

pub fn home_page(tasks: List(Task)) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute.attribute("content", "width=device-width, initial-scale=1"),
        ]),
        html.title([], "Teamwork -- Collaborative Task Board"),
        // HTMX core
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        // HTMX SSE extension
        html.script(
          [attribute.src("https://unpkg.com/htmx-ext-sse@2.2.2/sse.js")],
          "",
        ),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/styles.css"),
        ]),
      ]),
      html.body([], [
        html.h1([], [element.text("Teamwork")]),

        // ── SSE container ──
        // Everything inside this div receives real-time updates.
        html.div(
          [
            attribute.attribute("hx-ext", "sse"),
            attribute.attribute("sse-connect", "/events"),
          ],
          [
            // ── Task counter: receives "task-counter" events ──
            html.div(
              [
                attribute.id("task-counter"),
                attribute.attribute("sse-swap", "task-counter"),
              ],
              // SSE populates this on connect (see handle_sse init)
              [],
            ),

            // ── Add task form ──
            // This form uses a normal hx-post. The SSE broadcast
            // handles updating the task list for all clients.
            html.form(
              [
                attribute.attribute("hx-post", "/tasks"),
                attribute.attribute("hx-swap", "none"),
                attribute.class("add-task-form"),
              ],
              [
                html.input([
                  attribute.id("task-input"),
                  attribute.name("title"),
                  attribute.type_("text"),
                  attribute.placeholder("What needs to be done?"),
                  attribute.required(True),
                ]),
                html.button(
                  [attribute.type_("submit"), attribute.class("btn btn-primary")],
                  [element.text("Add Task")],
                ),
              ],
            ),

            // ── Task list: receives "task-list" events ──
            html.div(
              [
                attribute.id("task-list"),
                attribute.attribute("sse-swap", "task-list"),
              ],
              // SSE populates this on connect (see handle_sse init)
              [],
            ),
          ],
        ),

        // ── Connection status indicator ──
        html.div(
          [attribute.id("connection-status"), attribute.class("connected")],
          [element.text("Connected -- real-time updates active")],
        ),
      ]),
    ])

  let body =
    page
    |> element.to_document_string

  wisp.html_response(body, 200)
}

// ── Fragments ────────────────────────────────────────────────────────────

/// Render the task list as an HTML string.
/// Used for both the initial page load and SSE events.
pub fn render_task_list(tasks: List(Task)) -> String {
  let items =
    list.map(tasks, fn(task) {
      let id_str = int.to_string(task.id)
      let done_class = case task.done {
        True -> " done"
        False -> ""
      }
      let done_text = case task.done {
        True -> " [done]"
        False -> ""
      }

      "<li id=\"task-"
      <> id_str
      <> "\" class=\"task-item"
      <> done_class
      <> "\">"
      <> "<span class=\"task-title\">"
      <> task.title
      <> done_text
      <> "</span>"
      <> "<div class=\"task-actions\">"
      <> "<button hx-post=\"/tasks/"
      <> id_str
      <> "/toggle\" hx-swap=\"none\">Toggle</button>"
      <> "<button hx-delete=\"/tasks/"
      <> id_str
      <> "\" hx-swap=\"none\">Delete</button>"
      <> "</div>"
      <> "</li>"
    })

  "<ul class=\"task-list\">" <> string_join(items, "") <> "</ul>"
}

/// Render the task counter as an HTML string.
pub fn render_task_counter(tasks: List(Task)) -> String {
  let total = list.length(tasks)
  let done = list.length(list.filter(tasks, fn(t) { t.done }))
  let remaining = total - done

  "<span class=\"counter\">"
  <> int.to_string(remaining)
  <> " remaining, "
  <> int.to_string(done)
  <> " done, "
  <> int.to_string(total)
  <> " total</span>"
}

// ── Helpers ──────────────────────────────────────────────────────────────

fn string_join(items: List(String), separator: String) -> String {
  case items {
    [] -> ""
    [first, ..rest] ->
      list.fold(rest, first, fn(acc, item) { acc <> separator <> item })
  }
}
```

A few things to notice about the views.

**The SSE container wraps the dynamic content.** The `div` with `hx-ext="sse"`
and `sse-connect="/events"` is the parent of both the task list and the task
counter. When the page loads, HTMX opens an SSE connection to `/events`. Any
child element with `sse-swap` will receive updates from that connection.

**The form uses `hx-swap="none"`.** The add-task form submits via `hx-post` but
does not need to swap anything into the DOM directly. The SSE broadcast will
handle updating the task list. The only thing the POST response does is clear
the form input with an OOB swap.

**The task buttons also use `hx-swap="none"`.** The toggle and delete buttons
send their requests, the state actor processes them and broadcasts, and the SSE
channel pushes the updated list. The buttons themselves do not need to swap any
response.

**The `render_task_list` and `render_task_counter` functions return raw HTML
strings.** These are used both for the initial page render (embedded in the
full page) and for SSE event payloads. Having a single rendering function for
both ensures consistency -- the SSE update produces exactly the same HTML as the
initial load.

### Step 5: Testing the Real-Time Flow

Start the server:

```shell
gleam run
```

Open `http://localhost:8000` in two browser tabs, side by side. Now:

1. **In Tab A**, type a task title and click "Add Task."
2. **Watch Tab B.** The new task appears instantly, without any action from the
   user in Tab B. The task counter updates too.
3. **In Tab B**, click "Delete" on a task.
4. **Watch Tab A.** The task disappears.
5. **In Tab A**, click "Toggle" on a task.
6. **Watch Tab B.** The task's done status changes.

This is the moment where Teamwork becomes truly collaborative. Two users, two
browser windows, one shared task board, real-time synchronization.

### Step 6: What Happens in the Network Tab

Open developer tools in one of the tabs and look at the Network tab. You will
see:

1. A long-lived request to `/events` with type `eventsource`. This is the SSE
   connection. It stays open indefinitely.
2. When you click "Add Task," a normal `POST /tasks` request appears and
   completes quickly.
3. On the `/events` connection, new data appears -- the SSE events pushed by the
   server. You can see the `event:` and `data:` fields in the EventStream tab.

The POST requests are short-lived (milliseconds). The SSE connection lives for
the duration of the session. This is efficient: one persistent connection
replaces what would be hundreds of polling requests.

---

## Full Code Listing

Below are all four modules, complete and ready to run.

### `gleam.toml`

```toml
name = "teamwork"
version = "1.0.0"
target = "erlang"

[dependencies]
gleam_stdlib = ">= 0.50.0 and < 1.0.0"
gleam_http = ">= 4.0.0 and < 5.0.0"
gleam_erlang = ">= 1.0.0 and < 2.0.0"
gleam_otp = ">= 1.0.0 and < 2.0.0"
mist = ">= 5.0.0 and < 6.0.0"
wisp = ">= 2.0.0 and < 3.0.0"
lustre = ">= 5.0.0 and < 6.0.0"

[dev-dependencies]
gleeunit = ">= 1.0.0 and < 2.0.0"
```

### `src/teamwork/state.gleam`

```gleam
import gleam/bool
import gleam/erlang/process.{type Subject}
import gleam/list
import gleam/otp/actor

pub type Task {
  Task(id: Int, title: String, done: Bool)
}

pub type Message {
  AddTask(title: String, reply_to: Subject(Task))
  DeleteTask(id: Int)
  ToggleTask(id: Int)
  GetTasks(reply_to: Subject(List(Task)))
  Subscribe(subject: Subject(Update))
  Unsubscribe(subject: Subject(Update))
}

pub type Update {
  TasksChanged
}

pub type State {
  State(
    tasks: List(Task),
    next_id: Int,
    subscribers: List(Subject(Update)),
  )
}

fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}

pub fn start() -> Result(Subject(Message), actor.StartError) {
  let initial_state =
    State(
      tasks: [
        Task(id: 1, title: "Set up project repository", done: False),
        Task(id: 2, title: "Design database schema", done: False),
        Task(id: 3, title: "Build task API endpoints", done: True),
        Task(id: 4, title: "Create front-end layout", done: False),
      ],
      next_id: 5,
      subscribers: [],
    )

  actor.new(initial_state)
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}

fn handle_message(state: State, message: Message) -> actor.Next(State, Message) {
  case message {
    AddTask(title, reply_to) -> {
      let task = Task(id: state.next_id, title: title, done: False)
      let new_state =
        State(
          ..state,
          tasks: list.append(state.tasks, [task]),
          next_id: state.next_id + 1,
        )
      process.send(reply_to, task)
      broadcast(new_state.subscribers, TasksChanged)
      actor.continue(new_state)
    }

    DeleteTask(id) -> {
      let new_tasks = list.filter(state.tasks, fn(t) { t.id != id })
      let new_state = State(..state, tasks: new_tasks)
      broadcast(new_state.subscribers, TasksChanged)
      actor.continue(new_state)
    }

    ToggleTask(id) -> {
      let new_tasks =
        list.map(state.tasks, fn(t) {
          case t.id == id {
            True -> Task(..t, done: bool.negate(t.done))
            False -> t
          }
        })
      let new_state = State(..state, tasks: new_tasks)
      broadcast(new_state.subscribers, TasksChanged)
      actor.continue(new_state)
    }

    GetTasks(reply_to) -> {
      process.send(reply_to, state.tasks)
      actor.continue(state)
    }

    Subscribe(subject) -> {
      let new_state =
        State(..state, subscribers: [subject, ..state.subscribers])
      actor.continue(new_state)
    }

    Unsubscribe(subject) -> {
      let new_subscribers =
        list.filter(state.subscribers, fn(s) { s != subject })
      let new_state = State(..state, subscribers: new_subscribers)
      actor.continue(new_state)
    }
  }
}

fn broadcast(subscribers: List(Subject(Update)), update: Update) -> Nil {
  list.each(subscribers, fn(subscriber) {
    process.send(subscriber, update)
  })
}
```

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process.{type Subject}
import gleam/http/request.{type Request}
import gleam/http/response
import gleam/otp/actor
import gleam/string_tree
import mist.{type Connection, type ResponseData, type SSEConnection}
import teamwork/state.{type Message, type Task, type Update}
import teamwork/router
import teamwork/views
import wisp
import wisp/wisp_mist

pub type Context {
  Context(state: Subject(Message))
}

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let assert Ok(state_subject) = state.start()
  let ctx = Context(state: state_subject)

  let handler = fn(req: Request(Connection)) -> response.Response(ResponseData) {
    case request.path_segments(req) {
      ["events"] -> handle_sse(req, ctx)
      _ -> {
        let wisp_handler = fn(req) { router.handle_request(req, ctx) }
        wisp_mist.handler(wisp_handler, secret_key_base)(req)
      }
    }
  }

  let assert Ok(_) =
    handler
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

fn handle_sse(
  req: Request(Connection),
  ctx: Context,
) -> response.Response(ResponseData) {
  mist.server_sent_events(
    request: req,
    initial_response: response.new(200),
    init: fn(subject) {
      actor.send(ctx.state, state.Subscribe(subject))
      process.send(subject, state.TasksChanged)
      Ok(actor.initialised(subject))
    },
    loop: fn(subject, message, conn) {
      case message {
        state.TasksChanged -> {
          let tasks = actor.call(ctx.state, 1000, state.GetTasks)

          let task_list_event =
            mist.event(string_tree.from_string(views.render_task_list(tasks)))
            |> mist.event_name("task-list")

          let counter_event =
            mist.event(string_tree.from_string(views.render_task_counter(tasks)))
            |> mist.event_name("task-counter")

          case mist.send_event(conn, task_list_event) {
            Ok(_) -> {
              case mist.send_event(conn, counter_event) {
                Ok(_) -> actor.continue(subject)
                Error(_) -> {
                  actor.send(ctx.state, state.Unsubscribe(subject))
                  actor.stop()
                }
              }
            }
            Error(_) -> {
              actor.send(ctx.state, state.Unsubscribe(subject))
              actor.stop()
            }
          }
        }
      }
    },
  )
}
```

### `src/teamwork/router.gleam`

```gleam
import gleam/http
import gleam/int
import gleam/list
import gleam/erlang/process
import gleam/otp/actor
import teamwork.{type Context}
import teamwork/state
import teamwork/views
import wisp.{type Request, type Response}

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use req <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> {
      use <- wisp.require_method(req, http.Get)
      let tasks = actor.call(ctx.state, 1000, state.GetTasks)
      views.home_page(tasks)
    }

    ["tasks"] -> {
      use <- wisp.require_method(req, http.Post)
      use formdata <- wisp.require_form(req)

      let title =
        list.find(formdata.values, fn(pair) { pair.0 == "title" })
      case title {
        Ok(#(_, value)) -> {
          let _task =
            actor.call(ctx.state, 1000, state.AddTask(value, _))
          let body =
              "<input id=\"task-input\" name=\"title\" type=\"text\" "
              <> "placeholder=\"What needs to be done?\" value=\"\" "
              <> "hx-swap-oob=\"true\" />"
          wisp.html_response(body, 200)
        }
        Error(_) -> wisp.bad_request()
      }
    }

    ["tasks", id_str] -> {
      use <- wisp.require_method(req, http.Delete)
      case int.parse(id_str) {
        Ok(id) -> {
          actor.send(ctx.state, state.DeleteTask(id))
          wisp.html_response("", 200)
        }
        Error(_) -> wisp.bad_request()
      }
    }

    ["tasks", id_str, "toggle"] -> {
      use <- wisp.require_method(req, http.Post)
      case int.parse(id_str) {
        Ok(id) -> {
          actor.send(ctx.state, state.ToggleTask(id))
          wisp.html_response("", 200)
        }
        Error(_) -> wisp.bad_request()
      }
    }

    _ -> wisp.not_found()
  }
}
```

### `src/teamwork/views.gleam`

```gleam
import gleam/int
import gleam/list
import lustre/attribute
import lustre/element
import lustre/element/html
import teamwork/state.{type Task}
import wisp.{type Response}

pub fn home_page(tasks: List(Task)) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute.attribute("content", "width=device-width, initial-scale=1"),
        ]),
        html.title([], "Teamwork -- Collaborative Task Board"),
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        html.script(
          [attribute.src("https://unpkg.com/htmx-ext-sse@2.2.2/sse.js")],
          "",
        ),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/styles.css"),
        ]),
      ]),
      html.body([], [
        html.h1([], [element.text("Teamwork")]),
        html.div(
          [
            attribute.attribute("hx-ext", "sse"),
            attribute.attribute("sse-connect", "/events"),
          ],
          [
            html.div(
              [
                attribute.id("task-counter"),
                attribute.attribute("sse-swap", "task-counter"),
              ],
              // SSE populates this on connect (see handle_sse init)
              [],
            ),
            html.form(
              [
                attribute.attribute("hx-post", "/tasks"),
                attribute.attribute("hx-swap", "none"),
                attribute.class("add-task-form"),
              ],
              [
                html.input([
                  attribute.id("task-input"),
                  attribute.name("title"),
                  attribute.type_("text"),
                  attribute.placeholder("What needs to be done?"),
                  attribute.required(True),
                ]),
                html.button(
                  [attribute.type_("submit"), attribute.class("btn btn-primary")],
                  [element.text("Add Task")],
                ),
              ],
            ),
            html.div(
              [
                attribute.id("task-list"),
                attribute.attribute("sse-swap", "task-list"),
              ],
              // SSE populates this on connect (see handle_sse init)
              [],
            ),
          ],
        ),
        html.div(
          [attribute.id("connection-status"), attribute.class("connected")],
          [element.text("Connected -- real-time updates active")],
        ),
      ]),
    ])

  let body =
    page
    |> element.to_document_string

  wisp.html_response(body, 200)
}

pub fn render_task_list(tasks: List(Task)) -> String {
  let items =
    list.map(tasks, fn(task) {
      let id_str = int.to_string(task.id)
      let done_class = case task.done {
        True -> " done"
        False -> ""
      }
      let done_text = case task.done {
        True -> " [done]"
        False -> ""
      }

      "<li id=\"task-"
      <> id_str
      <> "\" class=\"task-item"
      <> done_class
      <> "\">"
      <> "<span class=\"task-title\">"
      <> task.title
      <> done_text
      <> "</span>"
      <> "<div class=\"task-actions\">"
      <> "<button hx-post=\"/tasks/"
      <> id_str
      <> "/toggle\" hx-swap=\"none\">Toggle</button>"
      <> "<button hx-delete=\"/tasks/"
      <> id_str
      <> "\" hx-swap=\"none\">Delete</button>"
      <> "</div>"
      <> "</li>"
    })

  "<ul class=\"task-list\">" <> string_join(items, "") <> "</ul>"
}

pub fn render_task_counter(tasks: List(Task)) -> String {
  let total = list.length(tasks)
  let done = list.length(list.filter(tasks, fn(t) { t.done }))
  let remaining = total - done

  "<span class=\"counter\">"
  <> int.to_string(remaining)
  <> " remaining, "
  <> int.to_string(done)
  <> " done, "
  <> int.to_string(total)
  <> " total</span>"
}

fn string_join(items: List(String), separator: String) -> String {
  case items {
    [] -> ""
    [first, ..rest] ->
      list.fold(rest, first, fn(acc, item) { acc <> separator <> item })
  }
}
```

---

## Exercise

Now it is your turn to extend the real-time task board. Each exercise builds on
the code from this chapter.

### Task 1: Broadcast Task Deletions

The current code already broadcasts deletions through the state actor. Verify
this works end to end:

1. Open the app in two browser tabs
2. Delete a task in Tab A
3. Confirm the task disappears in Tab B in real time

If it does not work, trace the flow: Does `DeleteTask` call `broadcast`? Does
the SSE loop handle `TasksChanged`? Does `render_task_list` omit the deleted
task?

### Task 2: "User Is Typing" Indicator

Add a visual indicator that shows when someone is typing in the add-task input
field. When a user types in Tab A, Tab B should show "Someone is typing..."

Hints:
- Add a new SSE event type called `typing-indicator`
- Add a `<div>` with `sse-swap="typing-indicator"` to the page
- Use `hx-trigger="keyup changed delay:100ms"` on the input to POST to a
  `/typing` endpoint
- The `/typing` endpoint broadcasts a typing event through the state actor
- Use a timeout: after 2 seconds of no keystrokes, clear the indicator

### Task 3: Notification Toasts

When another user adds a task, show a notification toast that says something
like "A task was added: Design the logo." The toast should appear at the top of
the page and disappear after a few seconds.

Hints:
- Add a new SSE event type called `notification`
- Add a `<div>` with `sse-swap="notification"` and `hx-swap="afterbegin"` so
  new notifications stack at the top
- In the state actor, include the task title in the broadcast so the SSE handler
  can render a meaningful message
- Use CSS animations or `setTimeout` (via a small inline script) to fade out
  the toast after 3 seconds

### Task 4: Connected Users Counter

Display a count of how many users are currently viewing the board. The count
should update in real time as users connect and disconnect.

Hints:
- The state actor already tracks subscribers. Add a `GetSubscriberCount`
  message
- Add a new SSE event type called `user-count`
- Broadcast `user-count` events whenever a user subscribes or unsubscribes
- Add a `<div>` with `sse-swap="user-count"` to the page

### Task 5: Reconnection Resilience

Test SSE reconnection:

1. Open the app in a browser tab
2. Stop the server (Ctrl+C)
3. Wait a few seconds
4. Restart the server (`gleam run`)
5. Add a task. Does the tab receive the update?

The browser's `EventSource` API automatically reconnects. HTMX's SSE extension
inherits this behavior. However, after reconnection the client might have stale
data. Think about how to handle this -- the `init` callback in the SSE handler
already sends the full current state, which solves the problem naturally.

---

## Exercise Solution Hints

### Hint for Task 1: Tracing the Broadcast

If deletions are not broadcasting, check three things:

1. The `DeleteTask` handler in `state.gleam` must call `broadcast`:
   ```gleam
   DeleteTask(id) -> {
     let new_tasks = list.filter(state.tasks, fn(t) { t.id != id })
     let new_state = State(..state, tasks: new_tasks)
     broadcast(new_state.subscribers, TasksChanged)  // <-- Must be here
     actor.continue(new_state)
   }
   ```

2. The delete button must use `hx-swap="none"` so HTMX does not try to swap the
   empty response into the DOM. The real update comes through SSE.

3. The SSE loop handles `TasksChanged` by fetching the full task list and
   sending it. Since the deleted task is no longer in the list, the rendered HTML
   will not include it.

### Hint for Task 2: Typing Indicator

Add a new message type to the state actor:

```gleam
pub type Message {
  // ... existing messages ...
  NotifyTyping
}

pub type Update {
  TasksChanged
  TypingStarted
}
```

In the SSE loop, handle `TypingStarted`:

```gleam
state.TypingStarted -> {
  let event =
    mist.event(string_tree.from_string(
      "<span class=\"typing\">Someone is typing...</span>",
    ))
    |> mist.event_name("typing-indicator")

  case mist.send_event(conn, event) {
    Ok(_) -> actor.continue(subject)
    Error(_) -> {
      actor.send(ctx.state, state.Unsubscribe(subject))
      actor.stop()
    }
  }
}
```

On the input element:

```html
<input name="title"
       hx-post="/typing"
       hx-trigger="keyup changed delay:100ms"
       hx-swap="none" />
```

To clear the indicator after inactivity, you can either use a server-side timer
(a process that sends a "clear typing" event after 2 seconds) or include a small
inline script in the SSE event data that uses `setTimeout` to clear itself.

### Hint for Task 3: Notification Toasts

The key is sending the task title along with the broadcast. Modify the `Update`
type:

```gleam
pub type Update {
  TasksChanged
  TaskAdded(title: String)
}
```

In the `AddTask` handler, broadcast `TaskAdded(title)` in addition to
`TasksChanged`. In the SSE loop, send a notification event:

```gleam
state.TaskAdded(title) -> {
  let html =
    "<div class=\"toast\">New task added: " <> title <> "</div>"
  let event =
    mist.event(string_tree.from_string(html))
    |> mist.event_name("notification")
  // ... send event ...
}
```

Use `hx-swap="afterbegin"` on the notification container so new toasts stack
at the top.

### Hint for Task 4: Connected Users Counter

Add subscriber count logic to the state actor:

```gleam
Subscribe(subject) -> {
  let new_state =
    State(..state, subscribers: [subject, ..state.subscribers])
  broadcast(new_state.subscribers, UserCountChanged(list.length(new_state.subscribers)))
  actor.continue(new_state)
}

Unsubscribe(subject) -> {
  let new_subscribers =
    list.filter(state.subscribers, fn(s) { s != subject })
  let new_state = State(..state, subscribers: new_subscribers)
  broadcast(new_state.subscribers, UserCountChanged(list.length(new_state.subscribers)))
  actor.continue(new_state)
}
```

The SSE handler sends the count:

```gleam
state.UserCountChanged(count) -> {
  let html =
    "<span>" <> int.to_string(count) <> " user(s) online</span>"
  let event =
    mist.event(string_tree.from_string(html))
    |> mist.event_name("user-count")
  // ... send event ...
}
```

### Hint for Task 5: Reconnection

You do not need to write any code for basic reconnection. The `EventSource` API
handles it automatically, and the HTMX SSE extension preserves this behavior.
When the connection re-establishes, the `init` callback in `handle_sse` runs
again, sending the full current state. This means the client's view is
automatically corrected after a reconnection.

To test this properly, you can add a `retry:` field to your SSE events to
control the reconnection delay. Mist may support this through its event builder.
A 1-second retry is a good default:

```gleam
mist.event(string_tree.from_string(html))
  |> mist.event_name("task-list")
  |> mist.event_retry(1000)
```

---

## Key Takeaways

1. **SSE is the natural real-time transport for HTMX.** The server pushes HTML
   fragments through a persistent connection. HTMX swaps them in. The client
   uses normal HTTP requests for actions. This one-directional push model maps
   perfectly onto how HTMX already works.

2. **The HTMX SSE extension needs three attributes.** `hx-ext="sse"` activates
   the extension. `sse-connect="/events"` opens the connection. `sse-swap="name"`
   on child elements subscribes them to specific event types. That is the entire
   client-side setup.

3. **Mist handles SSE at the HTTP level.** Wisp does not support streaming
   responses, so we intercept SSE requests at the Mist level with a hybrid
   router. Mist's `server_sent_events` function manages the connection
   lifecycle, and we provide `init` and `loop` callbacks.

4. **The pub/sub pattern uses BEAM subjects.** Each SSE connection receives a
   `Subject(Update)` from Mist. The state actor maintains a list of subscribers. After
   every mutation, it broadcasts to all subscribers. This is a textbook use of
   the actor model -- lightweight processes communicating through messages.

5. **Actions go through HTTP, updates come through SSE.** The form POSTs to
   `/tasks`. The state actor processes it, broadcasts `TasksChanged`, and every
   SSE connection pushes the updated HTML. The POST response itself is minimal.
   This separation keeps the architecture clean and ensures every client gets
   the update.

6. **SSE auto-reconnects for free.** The browser's `EventSource` API
   reconnects automatically. The `init` callback in the SSE handler sends the
   full current state on every connection, so after a reconnection the client's
   view is corrected without any extra work.

7. **One rendering function serves two purposes.** Functions like
   `render_task_list` produce HTML used for both the initial page load and SSE
   event payloads. This guarantees consistency -- the real-time update looks
   identical to a fresh page load.

8. **The BEAM makes this architecture natural.** In many languages, managing
   hundreds of concurrent SSE connections and broadcasting between them requires
   external tools (Redis pub/sub, message queues). On the BEAM, it is just
   processes and messages. Each connection is a lightweight process. The state
   actor is a process. Broadcasting is a loop over `process.send`. No external
   dependencies needed.

---

**Next chapter:** With our application now serving real-time updates, Chapter 14
will focus on **security hardening** -- Content-Security-Policy headers, CSRF
protection, rate limiting, and input sanitisation to make the Teamwork board
production-safe.
