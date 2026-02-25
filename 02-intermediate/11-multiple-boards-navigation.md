# Chapter 11 -- Multiple Boards and Navigation

**Phase:** Intermediate
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter you will be able to:

- Use `hx-boost="true"` to convert standard links and forms into AJAX requests, giving your app SPA-like navigation without client-side JavaScript.
- Explain how `hx-boost` manages browser history automatically (back/forward buttons, URL updates).
- Use `hx-replace-url` to replace the current history entry instead of pushing a new one.
- Design nested routes in Wisp for resources like `/boards/:id/tasks/:task_id`.
- Use `wisp.method_override` to support PUT and DELETE from HTML forms.
- Split a growing Gleam application into multiple modules for maintainability.
- Build a multi-board task application that feels like a single-page app while remaining fully server-rendered.

---

## 1. Theory

### 1.1 The Evolution: From One List to Many Boards

Up to this point our Teamwork app has a single, flat task list. You can create
tasks, edit them, delete them, search and filter them, and the URL updates as
you interact. That is a solid foundation.

But real task management tools do not have just one list. They have **boards** --
separate workspaces for different projects, sprints, or contexts. "Sprint 1",
"Backlog", "Design Review", "Done" -- each board has its own set of tasks, its
own identity, its own URL.

In this chapter we introduce multiple boards. This brings three challenges:

1. **Navigation.** Users need to move between boards without full page reloads.
2. **Nested routing.** A task now lives inside a board: `/boards/abc123/tasks/xyz789`.
3. **Code organization.** Our single `teamwork.gleam` file is getting too large.

HTMX and Gleam give us elegant solutions to all three.

### 1.2 `hx-boost="true"` -- SPA-Like Navigation for Free

When you place `hx-boost="true"` on a container element (typically the
`<body>`), HTMX intercepts every standard link click and form submission inside
that container and converts them into AJAX requests. Here is what changes:

| Without `hx-boost`                          | With `hx-boost="true"`                        |
|---------------------------------------------|-----------------------------------------------|
| Clicking `<a href="/boards">` triggers a full page load. The browser throws away the current page, sends a GET request, and renders the entire response from scratch. | Clicking the same `<a>` triggers an AJAX GET request. HTMX takes the response and swaps just the `<body>` content. No white flash, no scroll reset. |
| Submitting a `<form>` causes a full page reload. | The form is submitted via AJAX. The response replaces the `<body>`. |
| The URL changes because the browser navigated. | HTMX updates the URL using the History API. |

The mental model is simple: **`hx-boost` makes every link and form behave as if
you had written `hx-get` or `hx-post` on it, with `hx-target="body"` and
`hx-push-url="true"`.**

You do not need to change your server at all. The server continues to return
full HTML pages. HTMX is smart enough to extract the `<body>` content from the
response and swap it in. The `<head>` is merged intelligently -- new stylesheets
and scripts are loaded, existing ones are kept.

This is the simplest path to SPA-like navigation. One attribute on the body tag,
and every link in your app becomes an AJAX-powered, history-aware navigation
action.

```html
<body hx-boost="true">
  <!-- Every <a> and <form> inside here is now boosted -->
  <nav>
    <a href="/">Home</a>
    <a href="/boards">Boards</a>
  </nav>
  <main>
    <!-- page content -->
  </main>
</body>
```

### 1.3 Browser History and `hx-boost`

One of the most frustrating problems with hand-rolled AJAX navigation is browser
history. Users expect the back button to work. They expect to bookmark a URL and
return to the same page. They expect to open a link in a new tab.

`hx-boost` handles all of this automatically:

- **URL updates.** When a boosted link is clicked, HTMX pushes the new URL onto
  the browser's history stack using `history.pushState`. The address bar shows
  the correct URL.
- **Back/forward buttons.** When the user clicks the back button, HTMX restores
  the previous page. It does this by caching the previous page's HTML in memory
  and restoring it, or by re-fetching it from the server if the cache has
  expired.
- **New tab / right-click.** Boosted links are still regular `<a>` tags with
  `href` attributes. Right-click and "Open in new tab" works exactly as
  expected -- it triggers a normal full-page navigation, bypassing HTMX entirely.
- **Bookmarks.** Since the URL always reflects the current page, bookmarks work
  perfectly.

This is the power of progressive enhancement. The links are real links. HTMX
enhances them when JavaScript is available, but they degrade gracefully when it
is not.

### 1.4 `hx-replace-url` -- Replacing Instead of Pushing

Sometimes you do not want to add a new entry to the history stack. You want to
**replace** the current entry. The most common scenario is the
POST-Redirect-GET pattern:

1. The user submits a form (POST to `/boards/new`).
2. The server creates the board and redirects to `/boards/abc123`.
3. The browser follows the redirect and displays the new board.

If every step pushed to the history stack, the user would end up with the form
URL in their history. Clicking "back" would take them to the empty form again --
confusing at best, a duplicate submission at worst.

`hx-replace-url` solves this. When placed on an element, it tells HTMX to
replace the current history entry instead of pushing a new one:

```html
<form hx-post="/boards" hx-replace-url="true">
  <!-- After submission, the current history entry is replaced -->
</form>
```

You can also set it to a specific URL:

```html
<div hx-get="/boards/abc123" hx-replace-url="/boards/abc123">
```

The difference between `hx-push-url` and `hx-replace-url`:

| Attribute         | Effect on history stack              | Use case                              |
|-------------------|--------------------------------------|---------------------------------------|
| `hx-push-url`    | Adds a new entry (back button goes to previous page) | Normal navigation between pages |
| `hx-replace-url` | Replaces current entry (back button skips this page) | Redirects after form submission, filter changes |

### 1.5 Nested Routing

With boards in the picture, our URL structure becomes hierarchical:

```
/                           -- Home page
/boards                     -- List all boards
/boards/new                 -- Create a new board (form)
/boards/:board_id           -- Show a specific board and its tasks
/boards/:board_id/tasks     -- Operations on a board's tasks
/boards/:board_id/tasks/:task_id  -- A specific task on a specific board
```

In Wisp, this maps to pattern matching on path segments. The `:board_id` and
`:task_id` parts are dynamic -- they match any string, and we bind them to
variables:

```gleam
case wisp.path_segments(req) {
  []                                    -> home(req, ctx)
  ["boards"]                            -> boards_index(req, ctx)
  ["boards", "new"]                     -> boards_new(req, ctx)
  ["boards", board_id]                  -> board_show(req, ctx, board_id)
  ["boards", board_id, "tasks"]         -> board_tasks(req, ctx, board_id)
  ["boards", board_id, "tasks", task_id] ->
    board_task(req, ctx, board_id, task_id)
  _                                     -> not_found()
}
```

**Order matters.** The pattern `["boards", "new"]` must come before
`["boards", board_id]` because `board_id` would match the string `"new"`. Gleam
evaluates patterns top to bottom and picks the first match.

### 1.6 `wisp.method_override` -- PUT and DELETE from HTML Forms

HTML forms only support two methods: GET and POST. If you need to send a PUT or
DELETE request from a form, you have two options:

1. Use HTMX attributes like `hx-delete` or `hx-put` on the form or a button.
2. Use the **method override** pattern: submit a POST with a hidden `_method`
   field, and have the server rewrite the method before routing.

Option 2 is the classic approach, and Wisp supports it out of the box with
`wisp.method_override(req)`. Here is how it works:

```html
<form method="post" action="/boards/abc123">
  <input type="hidden" name="_method" value="delete">
  <button type="submit">Delete Board</button>
</form>
```

On the server, in your middleware:

```gleam
let req = wisp.method_override(req)
```

When Wisp sees a POST request with a `_method` field in the body, it changes
the request method to whatever `_method` specifies. Your router then sees a
DELETE request and routes it accordingly.

This is especially useful when `hx-boost` is active. The boosted form will be
submitted via AJAX, the method override will convert it to DELETE on the server,
and the response will be swapped into the page seamlessly.

### 1.7 Code Organization -- Splitting into Modules

A single file was fine for three routes. With boards, tasks, a layout module,
and a state actor, it becomes hard to navigate. Gleam's module system makes
splitting straightforward.

The convention we will follow:

```
src/
  teamwork.gleam              -- Entry point: main(), starts server
  teamwork/
    web.gleam                 -- Context type, shared types
    router.gleam              -- Top-level routing, middleware
    layout.gleam              -- Shared layout, nav bar, breadcrumbs
    boards.gleam              -- Board handlers and views
    tasks.gleam               -- Task handlers and views
    state.gleam               -- Actor that manages boards + tasks
```

Each module has a clear responsibility. The router imports the handlers from
`boards` and `tasks`. The handlers import the layout. The state actor is shared
through the context.

In Gleam, a module's name corresponds to its file path under `src/`. The module
`teamwork/router` lives at `src/teamwork/router.gleam` and is imported as:

```gleam
import teamwork/router
```

Public functions are exported with `pub fn`. Private functions (without `pub`)
are only visible within their own module. This gives you real encapsulation --
you choose exactly what each module exposes.

---

## 2. Code Walkthrough

Let's build the multi-board version of Teamwork step by step. We will start
with the data model, set up the module structure, wire up routing, and make
navigation feel instant with `hx-boost`.

### Step 1 -- The Board Data Model

First, we define what a board looks like. Create a new file for the board type:

```gleam
// src/teamwork/board.gleam

pub type Board {
  Board(id: String, name: String, description: String)
}
```

This is a simple record type. Each board has a unique identifier, a name, and
a description. We use `String` for the ID -- in a real app this might be a UUID.

### Step 2 -- Update the State Actor

Our state actor (from previous chapters) managed a flat list of tasks. Now it
needs to manage boards and associate tasks with boards. Update the state module:

```gleam
// src/teamwork/state.gleam

import gleam/dict.{type Dict}
import gleam/erlang/process
import gleam/int
import gleam/list
import gleam/otp/actor
import teamwork/board.{type Board, Board}

pub type Task {
  Task(
    id: String,
    board_id: String,
    title: String,
    status: String,
  )
}

pub type Message {
  GetBoards(reply_to: process.Subject(List(Board)))
  GetBoard(id: String, reply_to: process.Subject(Result(Board, Nil)))
  CreateBoard(name: String, description: String, reply_to: process.Subject(Board))
  DeleteBoard(id: String, reply_to: process.Subject(Result(Nil, Nil)))
  GetTasks(board_id: String, reply_to: process.Subject(List(Task)))
  CreateTask(board_id: String, title: String, reply_to: process.Subject(Task))
  DeleteTask(task_id: String, reply_to: process.Subject(Result(Nil, Nil)))
  UpdateTaskStatus(task_id: String, status: String, reply_to: process.Subject(Result(Task, Nil)))
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

fn handle_message(state: State, message: Message) -> actor.Next(State, Message) {
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
      let board = Board(id: id, name: name, description: description)
      let boards = dict.insert(state.boards, id, board)
      process.send(reply_to, board)
      actor.continue(State(..state, boards: boards))
    }

    DeleteBoard(id, reply_to) -> {
      case dict.get(state.boards, id) {
        Ok(_) -> {
          let boards = dict.delete(state.boards, id)
          // Also delete all tasks belonging to this board
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

    CreateTask(board_id, title, reply_to) -> {
      let #(id, state) = next_id(state)
      let task = Task(id: id, board_id: board_id, title: title, status: "todo")
      let tasks = dict.insert(state.tasks, id, task)
      process.send(reply_to, task)
      actor.continue(State(..state, tasks: tasks))
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

    UpdateTaskStatus(task_id, status, reply_to) -> {
      case dict.get(state.tasks, task_id) {
        Ok(task) -> {
          let updated = Task(..task, status: status)
          let tasks = dict.insert(state.tasks, task_id, updated)
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

A few things to notice:

- **`Task` now has a `board_id` field** that associates it with a board.
- **`next_id`** generates simple incrementing string IDs. In production you
  would use UUIDs, but incrementing IDs are easier to read in a tutorial.
- **`DeleteBoard` cascades** -- when a board is deleted, all its tasks are
  deleted too. This is a simple approach. A production app might prompt the user
  for confirmation.
- **Every message carries a `reply_to` subject.** This is the OTP actor pattern
  for request/response communication. The caller sends a message containing a
  `process.Subject` to send the result back to.

### Step 3 -- The Context Type

The context carries shared dependencies that every handler needs. Create the
web module:

```gleam
// src/teamwork/web.gleam

import gleam/erlang/process
import teamwork/state

pub type Context {
  Context(state: process.Subject(state.Message), static_directory: String)
}
```

This is the same pattern we have used in previous chapters. The context holds
a reference to the state actor and the path to the static files directory.

### Step 4 -- The Shared Layout with `hx-boost`

This is where the magic happens. We add `hx-boost="true"` to the `<body>` tag,
and every link in the app becomes an AJAX-powered navigation action:

```gleam
// src/teamwork/layout.gleam

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

Let's look at the key line:

```gleam
html.body([attribute("hx-boost", "true")], [
```

That single attribute transforms the entire application's navigation behavior.
Every `<a>` tag inside the body will now trigger an AJAX request instead of a
full page load. Every `<form>` will submit via AJAX. The URL will update. The
back button will work. And the user will see smooth, instant page transitions.

The `render_breadcrumbs` function takes a list of `Breadcrumb` records and
renders them as an ordered list. Each page handler will pass the appropriate
breadcrumbs for its position in the hierarchy. When the list is empty, we render
nothing using `element.none()`.

### Step 5 -- The Router and Middleware

The router is the nerve center. It inspects the URL and delegates to the right
handler module:

```gleam
// src/teamwork/router.gleam

import lustre/element
import teamwork/boards
import teamwork/layout.{Breadcrumb}
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
    ["boards"] -> boards.index(req, ctx)
    ["boards", "new"] -> boards.new_form(req, ctx)
    ["boards", board_id] -> boards.show(req, ctx, board_id)
    ["boards", board_id, "tasks"] -> boards.tasks(req, ctx, board_id)
    ["boards", board_id, "tasks", task_id] ->
      boards.task(req, ctx, board_id, task_id)
    _ -> not_found_page()
  }
}

fn home_page(_req: wisp.Request) -> wisp.Response {
  let content =
    layout.layout("Teamwork", [], {
      element.fragment([
        lustre_html_h1("Welcome to Teamwork"),
        lustre_html_p(
          "Organize your team's work across multiple boards. "
          <> "Create a board for each project, sprint, or workflow.",
        ),
        lustre_html_link("Get Started", "/boards"),
      ])
    })
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
      {
        element.fragment([
          lustre_html_h1("404 -- Page Not Found"),
          lustre_html_p("The page you are looking for does not exist."),
          lustre_html_link("Back to Home", "/"),
        ])
      },
    )
  wisp.html_response(
    element.to_document_string(content),
    404,
  )
}
```

Note the use of helper functions `lustre_html_h1`, `lustre_html_p`, and
`lustre_html_link`. These are small wrappers to keep the router code concise.
You can define them in a shared utilities module or inline them:

```gleam
import lustre/element/html
import lustre/attribute

fn lustre_html_h1(text: String) -> element.Element(t) {
  html.h1([], [element.text(text)])
}

fn lustre_html_p(text: String) -> element.Element(t) {
  html.p([], [element.text(text)])
}

fn lustre_html_link(text: String, href: String) -> element.Element(t) {
  html.a([attribute.href(href), attribute.class("btn btn-primary")], [
    element.text(text),
  ])
}
```

The important thing to see in the router is the **order of pattern matches**.
`["boards", "new"]` comes before `["boards", board_id]`. If we reversed them,
the request `GET /boards/new` would match the second pattern with
`board_id = "new"`, and we would try to look up a board with that ID instead of
showing the creation form.

Also notice that `wisp.method_override(req)` is called at the top of the
middleware. This means that by the time our router sees the request, a POST with
`_method=delete` has already been converted to a DELETE request.

### Step 6 -- Board Handlers and Views

This is the largest module. It handles listing boards, showing a single board
with its tasks, creating boards, and deleting boards:

```gleam
// src/teamwork/boards.gleam

import gleam/http
import gleam/list
import gleam/otp/actor
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import teamwork/board.{type Board, Board}
import teamwork/layout.{Breadcrumb}
import teamwork/state.{type Task}
import teamwork/web.{type Context}
import wisp

// ── Boards Index ──────────────────────────────────────────────────────

pub fn index(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.require_method(req, http.Get)

  let boards = actor.call(ctx.state, 1000, state.GetBoards)

  let content =
    layout.layout(
      "Boards -- Teamwork",
      [Breadcrumb("Home", "/")],
      html.div([], [
        html.div([attribute.class("page-header")], [
          html.h1([], [element.text("Your Boards")]),
          html.a(
            [attribute.href("/boards/new"), attribute.class("btn btn-primary")],
            [element.text("New Board")],
          ),
        ]),
        board_grid(boards),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn board_grid(boards: List(Board)) -> Element(t) {
  case boards {
    [] ->
      html.div([attribute.class("empty-state")], [
        html.p([], [element.text("No boards yet. Create your first one!")]),
      ])
    _ ->
      html.div(
        [attribute.class("board-grid")],
        list.map(boards, board_card),
      )
  }
}

fn board_card(board: Board) -> Element(t) {
  html.a(
    [
      attribute.href("/boards/" <> board.id),
      attribute.class("board-card"),
    ],
    [
      html.h3([], [element.text(board.name)]),
      html.p([], [element.text(board.description)]),
    ],
  )
}

// ── New Board Form ────────────────────────────────────────────────────

pub fn new_form(req: wisp.Request, ctx: Context) -> wisp.Response {
  case req.method {
    http.Get -> render_new_board_form()
    http.Post -> create_board(req, ctx)
    _ -> wisp.method_not_allowed([http.Get, http.Post])
  }
}

fn render_new_board_form() -> wisp.Response {
  let content =
    layout.layout(
      "New Board -- Teamwork",
      [Breadcrumb("Home", "/"), Breadcrumb("Boards", "/boards")],
      html.div([attribute.class("form-container")], [
        html.h1([], [element.text("Create a New Board")]),
        html.form([attribute.method("post"), attribute.action("/boards/new")], [
          html.div([attribute.class("form-group")], [
            html.label([attribute.for("name")], [element.text("Board Name")]),
            html.input([
              attribute.type_("text"),
              attribute.name("name"),
              attribute.id("name"),
              attribute.required(True),
              attribute.placeholder("e.g., Sprint 1"),
              attribute.class("form-control"),
            ]),
          ]),
          html.div([attribute.class("form-group")], [
            html.label([attribute.for("description")], [
              element.text("Description"),
            ]),
            html.textarea(
              [
                attribute.name("description"),
                attribute.id("description"),
                attribute.rows(3),
                attribute.placeholder("What is this board for?"),
                attribute.class("form-control"),
              ],
              "",
            ),
          ]),
          html.div([attribute.class("form-actions")], [
            html.a(
              [attribute.href("/boards"), attribute.class("btn btn-secondary")],
              [element.text("Cancel")],
            ),
            html.button(
              [attribute.type_("submit"), attribute.class("btn btn-primary")],
              [element.text("Create Board")],
            ),
          ]),
        ]),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn create_board(req: wisp.Request, ctx: Context) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  let name = case list.key_find(formdata.values, "name") {
    Ok(value) -> value
    Error(Nil) -> ""
  }
  let description = case list.key_find(formdata.values, "description") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  case name {
    "" -> wisp.redirect("/boards/new")
    _ -> {
      let board = actor.call(
        ctx.state,
        1000,
        state.CreateBoard(name, description, _),
      )
      wisp.redirect("/boards/" <> board.id)
    }
  }
}

// ── Board Detail ──────────────────────────────────────────────────────

pub fn show(req: wisp.Request, ctx: Context, board_id: String) -> wisp.Response {
  case req.method {
    http.Get -> render_board(ctx, board_id)
    http.Delete -> delete_board(ctx, board_id)
    _ -> wisp.method_not_allowed([http.Get, http.Delete])
  }
}

fn render_board(ctx: Context, board_id: String) -> wisp.Response {
  let board_result = actor.call(ctx.state, 1000, state.GetBoard(board_id, _))

  case board_result {
    Error(Nil) -> wisp.not_found()
    Ok(board) -> {
      let tasks = actor.call(ctx.state, 1000, state.GetTasks(board_id, _))

      let content =
        layout.layout(
          board.name <> " -- Teamwork",
          [
            Breadcrumb("Home", "/"),
            Breadcrumb("Boards", "/boards"),
            Breadcrumb(board.name, "/boards/" <> board.id),
          ],
          html.div([], [
            html.div([attribute.class("page-header")], [
              html.div([], [
                html.h1([], [element.text(board.name)]),
                html.p([attribute.class("board-description")], [
                  element.text(board.description),
                ]),
              ]),
              html.div([attribute.class("board-actions")], [
                delete_board_button(board.id),
              ]),
            ]),
            html.hr([]),
            html.div([attribute.class("tasks-section")], [
              html.div([attribute.class("tasks-header")], [
                html.h2([], [element.text("Tasks")]),
              ]),
              add_task_form(board.id),
              task_list(board.id, tasks),
            ]),
          ]),
        )

      wisp.html_response(
        element.to_document_string(content),
        200,
      )
    }
  }
}

fn delete_board_button(board_id: String) -> Element(t) {
  html.form(
    [
      attribute.method("post"),
      attribute.action("/boards/" <> board_id),
    ],
    [
      html.input([
        attribute.type_("hidden"),
        attribute.name("_method"),
        attribute.value("delete"),
      ]),
      html.button(
        [attribute.type_("submit"), attribute.class("btn btn-danger")],
        [element.text("Delete Board")],
      ),
    ],
  )
}

fn delete_board(ctx: Context, board_id: String) -> wisp.Response {
  let _ = actor.call(ctx.state, 1000, state.DeleteBoard(board_id, _))
  wisp.redirect("/boards")
}

// ── Tasks ─────────────────────────────────────────────────────────────

fn add_task_form(board_id: String) -> Element(t) {
  html.form(
    [
      attribute.method("post"),
      attribute.action("/boards/" <> board_id <> "/tasks"),
      attribute.class("add-task-form"),
    ],
    [
      html.input([
        attribute.type_("text"),
        attribute.name("title"),
        attribute.required(True),
        attribute.placeholder("Add a new task..."),
        attribute.class("form-control"),
      ]),
      html.button(
        [attribute.type_("submit"), attribute.class("btn btn-primary")],
        [element.text("Add")],
      ),
    ],
  )
}

fn task_list(board_id: String, tasks: List(Task)) -> Element(t) {
  case tasks {
    [] ->
      html.div([attribute.class("empty-state")], [
        html.p([], [element.text("No tasks yet. Add one above!")]),
      ])
    _ ->
      html.ul(
        [attribute.class("task-list")],
        list.map(tasks, fn(task) { task_item(board_id, task) }),
      )
  }
}

fn task_item(board_id: String, task: Task) -> Element(t) {
  html.li([attribute.class("task-item")], [
    html.span([attribute.class("task-title")], [element.text(task.title)]),
    html.span([attribute.class("task-status badge-" <> task.status)], [
      element.text(task.status),
    ]),
    html.form(
      [
        attribute.method("post"),
        attribute.action(
          "/boards/" <> board_id <> "/tasks/" <> task.id,
        ),
      ],
      [
        html.input([
          attribute.type_("hidden"),
          attribute.name("_method"),
          attribute.value("delete"),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn btn-sm btn-danger")],
          [element.text("Remove")],
        ),
      ],
    ),
  ])
}

// ── Task Actions ──────────────────────────────────────────────────────

pub fn tasks(req: wisp.Request, ctx: Context, board_id: String) -> wisp.Response {
  case req.method {
    http.Post -> create_task(req, ctx, board_id)
    _ -> wisp.method_not_allowed([http.Post])
  }
}

fn create_task(req: wisp.Request, ctx: Context, board_id: String) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  let title = case list.key_find(formdata.values, "title") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  case title {
    "" -> wisp.redirect("/boards/" <> board_id)
    _ -> {
      let _ = actor.call(
        ctx.state,
        1000,
        state.CreateTask(board_id, title, _),
      )
      wisp.redirect("/boards/" <> board_id)
    }
  }
}

pub fn task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
  task_id: String,
) -> wisp.Response {
  case req.method {
    http.Delete -> delete_task(ctx, board_id, task_id)
    _ -> wisp.method_not_allowed([http.Delete])
  }
}

fn delete_task(ctx: Context, board_id: String, _task_id: String) -> wisp.Response {
  let _ = actor.call(ctx.state, 1000, state.DeleteTask(_task_id, _))
  wisp.redirect("/boards/" <> board_id)
}
```

There is a lot happening in this module, so let's highlight the key patterns.

**Boosted navigation through board cards.** Each board card is a plain `<a>`
tag. Because `hx-boost` is on the body, clicking a card makes an AJAX request
to `/boards/abc123`. HTMX swaps the response body into the page. The URL
updates. The transition is instant. No JavaScript was written to make this
happen.

**Delete with method override.** The `delete_board_button` function renders a
form with `method="post"` and a hidden `_method` field set to `"delete"`. When
`hx-boost` intercepts the form submission, it sends a POST. The middleware calls
`wisp.method_override`, which sees the `_method` field and converts the request
to DELETE. The router matches `http.Delete` in the `show` handler.

**POST-Redirect-GET.** When a board is created, `create_board` returns
`wisp.redirect("/boards/" <> board.id)`. This sends an HTTP 303 redirect. HTMX
follows the redirect automatically and loads the new board page. Because
`hx-boost` is active, the redirect happens via AJAX -- the user sees the new
board appear without a full page reload.

**Breadcrumbs.** Each handler passes the appropriate breadcrumb trail to the
layout. The home page has no breadcrumbs. The boards index has
`[Home]`. The board detail page has `[Home > Boards > Sprint 1]`. This gives
the user a clear sense of where they are in the hierarchy.

### Step 7 -- The Entry Point

Finally, wire everything together in the main module:

```gleam
// src/teamwork.gleam

import gleam/erlang/process
import teamwork/router
import teamwork/state
import teamwork/web.{Context}
import mist
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

The main function:

1. Starts the state actor.
2. Creates a context containing the actor subject and static directory path.
3. Builds a handler closure that wires middleware and routing together.
4. Starts the HTTP server on port 8000.

### Step 8 -- CSS for the New Pages

Add these styles to `priv/static/css/style.css` to support the board grid,
cards, breadcrumbs, and task list:

```css
/* ── Navigation ────────────────────────────────── */

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

/* ── Breadcrumbs ───────────────────────────────── */

.breadcrumbs {
  padding: 0.75rem 2rem;
  background: #e8e8f0;
  font-size: 0.875rem;
}

.breadcrumbs ol {
  list-style: none;
  display: flex;
  gap: 0.5rem;
  margin: 0;
  padding: 0;
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

/* ── Layout ────────────────────────────────────── */

.container {
  max-width: 900px;
  margin: 2rem auto;
  padding: 0 1rem;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1.5rem;
}

/* ── Board Grid ────────────────────────────────── */

.board-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
  margin-top: 1rem;
}

.board-card {
  display: block;
  padding: 1.5rem;
  background: #fff;
  border: 1px solid #e0e0e8;
  border-radius: 8px;
  text-decoration: none;
  color: inherit;
  transition: box-shadow 0.15s ease, transform 0.15s ease;
}

.board-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.board-card h3 {
  margin: 0 0 0.5rem;
  color: #16213e;
}

.board-card p {
  margin: 0;
  color: #666;
  font-size: 0.875rem;
}

/* ── Forms ─────────────────────────────────────── */

.form-container {
  max-width: 500px;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.25rem;
  font-weight: 600;
  color: #16213e;
}

.form-control {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
}

.form-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 1.5rem;
}

/* ── Buttons ───────────────────────────────────── */

.btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;
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

/* ── Tasks ─────────────────────────────────────── */

.tasks-section {
  margin-top: 1.5rem;
}

.add-task-form {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.add-task-form .form-control {
  flex: 1;
}

.task-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  background: #fff;
  border: 1px solid #e0e0e8;
  border-radius: 4px;
  margin-bottom: 0.5rem;
}

.task-title {
  flex: 1;
}

.task-status {
  padding: 0.125rem 0.5rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 600;
}

.badge-todo {
  background: #ffeeba;
  color: #856404;
}

.badge-done {
  background: #d4edda;
  color: #155724;
}

/* ── Empty State ───────────────────────────────── */

.empty-state {
  text-align: center;
  padding: 3rem 1rem;
  color: #888;
}
```

---

## 3. Full Code Listing

Here is every file in the reorganized project, ready to copy and paste.

### Project Structure

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
│       ├── board.gleam
│       ├── boards.gleam
│       ├── state.gleam
│       └── tasks.gleam       (optional -- for when you split tasks out)
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

  // Start the state actor that manages boards and tasks.
  let assert Ok(state_subject) = state.start()

  let ctx = Context(
    state: state_subject,
    static_directory: static_directory(),
  )

  // Wire middleware and routing together.
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

### `src/teamwork/board.gleam`

```gleam
pub type Board {
  Board(id: String, name: String, description: String)
}
```

### `src/teamwork/state.gleam`

```gleam
import gleam/dict.{type Dict}
import gleam/erlang/process
import gleam/int
import gleam/list
import gleam/otp/actor
import teamwork/board.{type Board, Board}

pub type Task {
  Task(id: String, board_id: String, title: String, status: String)
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
  CreateTask(
    board_id: String,
    title: String,
    reply_to: process.Subject(Task),
  )
  DeleteTask(task_id: String, reply_to: process.Subject(Result(Nil, Nil)))
  UpdateTaskStatus(
    task_id: String,
    status: String,
    reply_to: process.Subject(Result(Task, Nil)),
  )
}

pub type State {
  State(boards: Dict(String, Board), tasks: Dict(String, Task), next_id: Int)
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

    CreateTask(board_id, title, reply_to) -> {
      let #(id, state) = next_id(state)
      let task = Task(
        id: id,
        board_id: board_id,
        title: title,
        status: "todo",
      )
      let tasks = dict.insert(state.tasks, id, task)
      process.send(reply_to, task)
      actor.continue(State(..state, tasks: tasks))
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

    UpdateTaskStatus(task_id, status, reply_to) -> {
      case dict.get(state.tasks, task_id) {
        Ok(task) -> {
          let updated = Task(..task, status: status)
          let tasks = dict.insert(state.tasks, task_id, updated)
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
import lustre/attribute
import lustre/element
import lustre/element/html
import teamwork/boards
import teamwork/layout.{Breadcrumb}
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
    ["boards"] -> boards.index(req, ctx)
    ["boards", "new"] -> boards.new_form(req, ctx)
    ["boards", board_id] -> boards.show(req, ctx, board_id)
    ["boards", board_id, "tasks"] -> boards.tasks(req, ctx, board_id)
    ["boards", board_id, "tasks", task_id] ->
      boards.task(req, ctx, board_id, task_id)
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
            <> "Create a board for each project, sprint, or workflow.",
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

### `src/teamwork/boards.gleam`

```gleam
import gleam/http
import gleam/list
import gleam/otp/actor
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import teamwork/board.{type Board}
import teamwork/layout.{Breadcrumb}
import teamwork/state.{type Task}
import teamwork/web.{type Context}
import wisp

// ── Boards Index ───────────────────────────────────────────────

pub fn index(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.require_method(req, http.Get)
  let boards = actor.call(ctx.state, 1000, state.GetBoards)

  let content =
    layout.layout(
      "Boards -- Teamwork",
      [Breadcrumb("Home", "/")],
      html.div([], [
        html.div([attribute.class("page-header")], [
          html.h1([], [element.text("Your Boards")]),
          html.a(
            [
              attribute.href("/boards/new"),
              attribute.class("btn btn-primary"),
            ],
            [element.text("New Board")],
          ),
        ]),
        board_grid(boards),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn board_grid(boards: List(Board)) -> Element(t) {
  case boards {
    [] ->
      html.div([attribute.class("empty-state")], [
        html.p([], [
          element.text("No boards yet. Create your first one!"),
        ]),
      ])
    _ ->
      html.div(
        [attribute.class("board-grid")],
        list.map(boards, board_card),
      )
  }
}

fn board_card(b: Board) -> Element(t) {
  html.a(
    [
      attribute.href("/boards/" <> b.id),
      attribute.class("board-card"),
    ],
    [
      html.h3([], [element.text(b.name)]),
      html.p([], [element.text(b.description)]),
    ],
  )
}

// ── New Board Form ─────────────────────────────────────────────

pub fn new_form(req: wisp.Request, ctx: Context) -> wisp.Response {
  case req.method {
    http.Get -> render_new_board_form()
    http.Post -> create_board(req, ctx)
    _ -> wisp.method_not_allowed([http.Get, http.Post])
  }
}

fn render_new_board_form() -> wisp.Response {
  let content =
    layout.layout(
      "New Board -- Teamwork",
      [
        Breadcrumb("Home", "/"),
        Breadcrumb("Boards", "/boards"),
      ],
      html.div([attribute.class("form-container")], [
        html.h1([], [element.text("Create a New Board")]),
        html.form(
          [
            attribute.method("post"),
            attribute.action("/boards/new"),
          ],
          [
            html.div([attribute.class("form-group")], [
              html.label([attribute.for("name")], [
                element.text("Board Name"),
              ]),
              html.input([
                attribute.type_("text"),
                attribute.name("name"),
                attribute.id("name"),
                attribute.required(True),
                attribute.placeholder("e.g., Sprint 1"),
                attribute.class("form-control"),
              ]),
            ]),
            html.div([attribute.class("form-group")], [
              html.label([attribute.for("description")], [
                element.text("Description"),
              ]),
              html.textarea(
                [
                  attribute.name("description"),
                  attribute.id("description"),
                  attribute.rows(3),
                  attribute.placeholder(
                    "What is this board for?",
                  ),
                  attribute.class("form-control"),
                ],
                "",
              ),
            ]),
            html.div([attribute.class("form-actions")], [
              html.a(
                [
                  attribute.href("/boards"),
                  attribute.class("btn btn-secondary"),
                ],
                [element.text("Cancel")],
              ),
              html.button(
                [
                  attribute.type_("submit"),
                  attribute.class("btn btn-primary"),
                ],
                [element.text("Create Board")],
              ),
            ]),
          ],
        ),
      ]),
    )

  wisp.html_response(
    element.to_document_string(content),
    200,
  )
}

fn create_board(
  req: wisp.Request,
  ctx: Context,
) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  let name = case list.key_find(formdata.values, "name") {
    Ok(value) -> value
    Error(Nil) -> ""
  }
  let description = case list.key_find(formdata.values, "description") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  case name {
    "" -> wisp.redirect("/boards/new")
    _ -> {
      let new_board =
        actor.call(
          ctx.state,
          1000,
          state.CreateBoard(name, description, _),
        )
      // Redirect to the new board.
      // Because hx-boost is active, this redirect happens via AJAX.
      // The browser URL updates to /boards/<id>.
      wisp.redirect("/boards/" <> new_board.id)
    }
  }
}

// ── Board Detail ───────────────────────────────────────────────

pub fn show(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  case req.method {
    http.Get -> render_board(ctx, board_id)
    http.Delete -> delete_board(ctx, board_id)
    _ -> wisp.method_not_allowed([http.Get, http.Delete])
  }
}

fn render_board(ctx: Context, board_id: String) -> wisp.Response {
  let board_result =
    actor.call(ctx.state, 1000, state.GetBoard(board_id, _))

  case board_result {
    Error(Nil) -> wisp.not_found()
    Ok(board) -> {
      let board_tasks =
        actor.call(ctx.state, 1000, state.GetTasks(board_id, _))

      let content =
        layout.layout(
          board.name <> " -- Teamwork",
          [
            Breadcrumb("Home", "/"),
            Breadcrumb("Boards", "/boards"),
            Breadcrumb(board.name, "/boards/" <> board.id),
          ],
          html.div([], [
            html.div([attribute.class("page-header")], [
              html.div([], [
                html.h1([], [element.text(board.name)]),
                html.p(
                  [attribute.class("board-description")],
                  [element.text(board.description)],
                ),
              ]),
              html.div([attribute.class("board-actions")], [
                delete_board_button(board.id),
              ]),
            ]),
            html.hr([]),
            html.div([attribute.class("tasks-section")], [
              html.h2([], [element.text("Tasks")]),
              add_task_form(board.id),
              task_list(board.id, board_tasks),
            ]),
          ]),
        )

      wisp.html_response(
        element.to_document_string(content),
        200,
      )
    }
  }
}

fn delete_board_button(board_id: String) -> Element(t) {
  html.form(
    [
      attribute.method("post"),
      attribute.action("/boards/" <> board_id),
    ],
    [
      html.input([
        attribute.type_("hidden"),
        attribute.name("_method"),
        attribute.value("delete"),
      ]),
      html.button(
        [
          attribute.type_("submit"),
          attribute.class("btn btn-danger"),
        ],
        [element.text("Delete Board")],
      ),
    ],
  )
}

fn delete_board(ctx: Context, board_id: String) -> wisp.Response {
  let _ =
    actor.call(ctx.state, 1000, state.DeleteBoard(board_id, _))
  wisp.redirect("/boards")
}

// ── Task Views ─────────────────────────────────────────────────

fn add_task_form(board_id: String) -> Element(t) {
  html.form(
    [
      attribute.method("post"),
      attribute.action(
        "/boards/" <> board_id <> "/tasks",
      ),
      attribute.class("add-task-form"),
    ],
    [
      html.input([
        attribute.type_("text"),
        attribute.name("title"),
        attribute.required(True),
        attribute.placeholder("Add a new task..."),
        attribute.class("form-control"),
      ]),
      html.button(
        [
          attribute.type_("submit"),
          attribute.class("btn btn-primary"),
        ],
        [element.text("Add")],
      ),
    ],
  )
}

fn task_list(board_id: String, items: List(Task)) -> Element(t) {
  case items {
    [] ->
      html.div([attribute.class("empty-state")], [
        html.p([], [
          element.text("No tasks yet. Add one above!"),
        ]),
      ])
    _ ->
      html.ul(
        [attribute.class("task-list")],
        list.map(items, fn(task) { task_item(board_id, task) }),
      )
  }
}

fn task_item(board_id: String, task: Task) -> Element(t) {
  html.li([attribute.class("task-item")], [
    html.span([attribute.class("task-title")], [
      element.text(task.title),
    ]),
    html.span(
      [attribute.class("task-status badge-" <> task.status)],
      [element.text(task.status)],
    ),
    html.form(
      [
        attribute.method("post"),
        attribute.action(
          "/boards/"
            <> board_id
            <> "/tasks/"
            <> task.id,
        ),
      ],
      [
        html.input([
          attribute.type_("hidden"),
          attribute.name("_method"),
          attribute.value("delete"),
        ]),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn btn-sm btn-danger"),
          ],
          [element.text("Remove")],
        ),
      ],
    ),
  ])
}

// ── Task Actions ───────────────────────────────────────────────

pub fn tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  case req.method {
    http.Post -> create_task(req, ctx, board_id)
    _ -> wisp.method_not_allowed([http.Post])
  }
}

fn create_task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  let title = case list.key_find(formdata.values, "title") {
    Ok(value) -> value
    Error(Nil) -> ""
  }

  case title {
    "" -> wisp.redirect("/boards/" <> board_id)
    _ -> {
      let _ =
        actor.call(
          ctx.state,
          1000,
          state.CreateTask(board_id, title, _),
        )
      wisp.redirect("/boards/" <> board_id)
    }
  }
}

pub fn task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
  task_id: String,
) -> wisp.Response {
  case req.method {
    http.Delete -> {
      let _ =
        actor.call(ctx.state, 1000, state.DeleteTask(task_id, _))
      wisp.redirect("/boards/" <> board_id)
    }
    _ -> wisp.method_not_allowed([http.Delete])
  }
}
```

---

## 4. Exercise

Now it is your turn. Work through the following tasks to deepen your
understanding of boosted navigation, nested routing, and code organization.

### Task 1: Create and Navigate Between Boards

1. Start the server with `gleam run`.
2. Navigate to `/boards` and create two or three boards using the "New Board" form.
3. Click between boards and observe the navigation. Notice how:
   - The page transitions are instant -- no white flash, no full page reload.
   - The URL in the address bar updates with each click.
   - The breadcrumbs reflect your position in the hierarchy.
4. Use the browser's back and forward buttons. Confirm they navigate through
   your history correctly.

**Acceptance criteria:** You can create boards, navigate between them, and use
the back/forward buttons without any full page reloads.

### Task 2: Add Tasks to Different Boards

1. Navigate to a board and add several tasks using the task form.
2. Navigate to a different board and add different tasks.
3. Return to the first board and confirm its tasks are still there -- each board
   has its own independent task list.

**Acceptance criteria:** Each board maintains its own tasks. Adding a task to
Board A does not affect Board B.

### Task 3: Use `hx-replace-url` on the Board Creation Form

Right now, after creating a board, the redirect pushes a new URL onto the
history stack. This means clicking "back" after creating a board takes you to
the empty form again. That is not ideal.

Modify the "Create Board" form to use `hx-replace-url` so that after the
redirect, the form URL is replaced in history. Clicking "back" should take you
to the boards list, not back to the form.

**Hint:** Add the `hx-replace-url="true"` attribute to the `<form>` element in
`render_new_board_form`.

**Acceptance criteria:** After creating a board, clicking the back button takes
you to the boards list, not the creation form.

### Task 4: Add Breadcrumb Navigation

Verify that breadcrumbs work correctly on every page:

- Home: no breadcrumbs.
- Boards index: `Home`.
- New board form: `Home > Boards`.
- Board detail: `Home > Boards > [Board Name]`.

Click each breadcrumb link and confirm it navigates correctly via `hx-boost`
(no full page reload).

### Task 5: Split Task Handlers into Their Own Module

As an extra challenge, move all task-related handlers and views from
`boards.gleam` into a separate `src/teamwork/tasks.gleam` module. Update the
imports in `boards.gleam` and `router.gleam` accordingly.

**Acceptance criteria:** The project compiles, all task CRUD operations work,
and `boards.gleam` only contains board-related code.

---

## 5. Exercise Solution Hints

If you get stuck, here are some pointers. Try to work through the exercises on
your own first.

### Hint for Task 1

If navigation does not feel instant, check that `hx-boost="true"` is present on
the `<body>` tag. Open your browser's developer tools, go to the Network tab,
and watch the requests. When you click a link, you should see an XHR/Fetch
request instead of a full document navigation. The request type column in the
Network tab will show "xhr" or "fetch" for boosted requests.

### Hint for Task 3

Modify the form in `render_new_board_form` to include `hx-replace-url`:

```gleam
html.form(
  [
    attribute.method("post"),
    attribute.action("/boards/new"),
    attribute("hx-replace-url", "true"),
  ],
  [
    // ... form fields ...
  ],
)
```

When `hx-boost` intercepts the form submission and the server responds with a
redirect, HTMX follows the redirect via AJAX. The `hx-replace-url` attribute
tells HTMX to replace the current history entry instead of pushing a new one.
The result is that the form URL (`/boards/new`) is never in the history stack --
the back button goes straight to `/boards`.

Note that the redirect from the server (`wisp.redirect("/boards/" <> new_id)`)
produces an HTTP 303 response. HTMX follows this redirect automatically. The
combination of `hx-boost`, `hx-replace-url`, and the server redirect gives you
the POST-Redirect-GET pattern with clean history behavior.

### Hint for Task 5

Create `src/teamwork/tasks.gleam` and move these functions into it:

- `add_task_form`
- `task_list`
- `task_item`
- `create_task`

Then export the functions you need with `pub fn` and import the module in
`boards.gleam`:

```gleam
import teamwork/tasks
```

Use the functions as `tasks.add_task_form(board_id)`,
`tasks.task_list(board_id, board_tasks)`, etc.

For the route handlers (`tasks`, `task`), you can either keep them in
`boards.gleam` (since they are board-scoped routes) or move them to
`tasks.gleam` and update the router. Either approach is valid. The key is that
each module has a clear, focused responsibility.

---

## 6. Key Takeaways

1. **`hx-boost="true"` is the simplest path to SPA-like navigation.** Place it
   on the `<body>` and every link and form inside becomes an AJAX request. The
   server returns full HTML pages, HTMX swaps the body content, and the URL
   updates automatically. One attribute, zero JavaScript.

2. **Browser history works automatically with `hx-boost`.** Back and forward
   buttons work. URLs are bookmarkable. Right-click "Open in new tab" works.
   This is progressive enhancement at its best -- the links are real links, and
   HTMX enhances them when JavaScript is available.

3. **`hx-replace-url` prevents form URLs from polluting the history stack.**
   Use it on forms that redirect after submission (the POST-Redirect-GET
   pattern) so the back button skips the form and goes to the previous real
   page.

4. **Nested routing is just deeper pattern matching.** `["boards", board_id, "tasks", task_id]`
   captures both the board and task IDs from the URL. Order matters: put
   specific patterns (like `["boards", "new"]`) before generic ones (like
   `["boards", board_id]`).

5. **`wisp.method_override` unlocks PUT and DELETE from HTML forms.** HTML forms
   only support GET and POST. By including a hidden `_method` field and calling
   `wisp.method_override(req)` in your middleware, you can use the full range of
   HTTP methods while keeping your forms simple.

6. **Split your code into modules as it grows.** A module per concern --
   `router.gleam` for routing, `boards.gleam` for board logic, `state.gleam`
   for data management, `layout.gleam` for shared UI. Each module has a clear
   responsibility, and `pub fn` controls what is visible to the outside.

7. **The server does not know or care about `hx-boost`.** Your handlers return
   the same full HTML pages they always did. HTMX on the client side is what
   makes navigation smooth. This means your app works perfectly without
   JavaScript -- it just reloads the full page instead of swapping the body.
   True progressive enhancement.

---

## What's Next

In [Chapter 12](12-cookies-sessions-auth.md), we will add **cookies, sessions, and authentication**. We will
use signed cookies to store session data, build a login form, protect routes
with authentication middleware, and display different UI based on whether the
user is logged in. The Teamwork app gets its first layer of access control.
