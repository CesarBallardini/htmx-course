# Chapter 4 -- Your First HTMX Interaction

## Learning Objectives

By the end of this chapter, you will:

- Understand the core idea behind HTMX: **any HTML element can make HTTP requests**
- Know the three foundational HTMX attributes: `hx-get`, `hx-target`, and `hx-swap`
- Use the `hx` Gleam library to add these attributes in a type-safe way
- Build a server endpoint that returns an **HTML fragment** instead of a full page
- See the complete request-response cycle in your browser's developer tools
- Grasp the difference between `element.to_string` (fragments) and `element.to_document_string` (full pages)

---

## Theory

### The Key Insight of HTMX

In traditional HTML, only two elements can make HTTP requests:

- **`<a>`** tags make GET requests when clicked (navigating to a new page)
- **`<form>`** tags make GET or POST requests when submitted (also navigating to a new page)

Both of them cause a **full page reload**. The browser throws away the entire current page, sends a request, and renders a completely new page from scratch.

HTMX changes this with one powerful idea:

> **Any element can make HTTP requests, and the server's response replaces just a piece of the page -- not the whole thing.**

A `<button>`, a `<div>`, a `<span>`, a `<tr>` -- literally any HTML element can trigger an HTTP request. And when the server responds, HTMX surgically swaps the response into a specific part of the DOM. No full page reload. No JavaScript written by you.

All of this is controlled through HTML attributes.

### The Core Cycle

Here is how every HTMX interaction works:

```
    Browser                                    Server
    ───────                                    ──────

    1. User clicks a button with
       hx-get="/tasks"
              │
              ▼
    2. HTMX intercepts the click and
       makes an AJAX GET request
              │
              │   GET /tasks
              │──────────────────────────────►│
              │                               │
              │                               │  3. Server builds an
              │                               │     HTML *fragment*
              │                               │     (just a <ul> with
              │                               │     some <li> items)
              │                               │
              │   <ul><li>Task 1</li>...</ul>  │
              │◄──────────────────────────────│
              │
              ▼
    4. HTMX finds the target element
       (from hx-target) and swaps the
       response HTML into it
       (using the strategy from hx-swap)
              │
              ▼
    5. The page updates instantly.
       No reload. No flicker.
```

That is the entire mental model. Every HTMX feature you will learn in this course is a variation of this cycle. Master it here and everything else will feel natural.

### The Core Trio: `hx-get`, `hx-target`, `hx-swap`

These three attributes work together. Let us look at each one.

#### `hx-get` -- What to Request

The `hx-get` attribute tells HTMX: "When this element is triggered (clicked, by default for buttons), make a GET request to this URL."

```html
<button hx-get="/tasks">Load Tasks</button>
```

When the user clicks this button, HTMX sends `GET /tasks` to your server. There are also `hx-post`, `hx-put`, `hx-patch`, and `hx-delete` for other HTTP methods, but we will start with `hx-get`.

#### `hx-target` -- Where to Put the Response

By default, HTMX replaces the **inner HTML of the element that made the request**. So if you click a button with `hx-get`, the response would go inside that button -- probably not what you want.

The `hx-target` attribute lets you specify a different element to receive the response, using a CSS selector:

```html
<button hx-get="/tasks" hx-target="#task-list">Load Tasks</button>
<div id="task-list"></div>
```

Now the response from `/tasks` will be placed inside the `<div id="task-list">` element.

#### `hx-swap` -- How to Insert the Response

The `hx-swap` attribute controls *how* the response is inserted into the target. The most common values are:

| Value         | Behavior                                                |
|---------------|---------------------------------------------------------|
| `innerHTML`   | Replace the **contents** of the target (this is the default) |
| `outerHTML`   | Replace the **entire target element** including itself   |
| `beforebegin` | Insert before the target element                        |
| `afterbegin`  | Insert before the first child of the target             |
| `beforeend`   | Insert after the last child of the target               |
| `afterend`    | Insert after the target element                         |
| `delete`      | Remove the target element                               |
| `none`        | Do not swap (useful for side-effect-only requests)      |

For now, `innerHTML` (the default) is all you need. It replaces the contents of the target element with the server's response.

### HTML Fragments vs Full Pages

This is a concept that trips up many beginners, so let us be very clear about it.

When your browser navigates to a URL normally, the server returns a **full HTML page** -- a complete document with `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`, and everything inside. In Chapters 1 through 3, that is what our Gleam server has been returning. We used `element.to_document_string` from Lustre to produce these full pages.

HTMX endpoints are different. When HTMX makes a request, the server should return just the **HTML fragment** that will be swapped in -- no doctype, no `<html>` wrapper, no `<head>`, no `<body>`. Just the raw piece of HTML that changed.

```
Full page (for normal navigation):
┌─────────────────────────┐
│ <!DOCTYPE html>         │
│ <html>                  │
│   <head>...</head>      │
│   <body>                │
│     <nav>...</nav>      │
│     <main>              │
│       <h1>...</h1>      │
│       <ul>              │  ◄── This is all
│         <li>Task 1</li> │      that HTMX needs
│         <li>Task 2</li> │
│       </ul>             │
│     </main>             │
│   </body>               │
│ </html>                 │
└─────────────────────────┘

Fragment (for HTMX requests):
┌─────────────────────────┐
│ <ul>                    │
│   <li>Task 1</li>       │
│   <li>Task 2</li>       │
│ </ul>                   │
└─────────────────────────┘
```

In Gleam with Lustre:

- **Full page**: use `element.to_document_string` -- wraps content in `<!doctype html>`, `<html>`, `<head>`, `<body>`
- **Fragment**: use `element.to_string` -- renders just the element and its children, nothing more

This distinction is critical. If you accidentally return a full page to an HTMX request, you will end up with a `<html>` tag nested inside a `<div>` -- not what you want.

---

## Code Walkthrough

Let us make the Teamwork app interactive. We will add a "Load Tasks" button that fetches a list of tasks from the server without reloading the page.

### Step 1: Add the `hx` Library

The `hx` library provides type-safe Gleam bindings for HTMX attributes. It integrates directly with Lustre's `attribute.Attribute` type, so HTMX attributes work alongside regular HTML attributes seamlessly.

Open your `gleam.toml` and add `hx` to the dependencies:

```toml
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

Then fetch the new dependency:

```shell
gleam deps download
```

### Step 2: Update the Home Page with an HTMX Button

Now let us update the home page to include a button that will load tasks dynamically. Here is the updated `home_content` function:

```gleam
import hx
import lustre/attribute
import lustre/element.{type Element}
import lustre/element/html

fn home_content() -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    html.p([], [element.text("Your collaborative task board.")]),
    html.button(
      [hx.get("/tasks"), hx.target(hx.Selector("#task-list")), hx.swap(hx.InnerHTML)],
      [element.text("Load Tasks")],
    ),
    html.div([attribute.id("task-list")], []),
  ])
}
```

Let us break down the three HTMX attributes on that button:

- **`hx.get("/tasks")`** -- produces the HTML attribute `hx-get="/tasks"`. When the button is clicked, HTMX will send a GET request to `/tasks`.
- **`hx.target(hx.Selector("#task-list"))`** -- produces `hx-target="#task-list"`. The response will be placed in the element with `id="task-list"`. Notice we use `hx.Selector(...)` to create a CSS selector. The `hx` library has a `Selector` type with several variants: `hx.Selector("#some-id")` for a CSS selector, `hx.This` for the element itself, `hx.Closest(".class")` for the nearest ancestor, and more. For now, `hx.Selector` with a CSS string is all you need.
- **`hx.swap(hx.InnerHTML)`** -- produces `hx-swap="innerHTML"`. The response HTML will replace the *contents* of the target element. Since `innerHTML` is the default, you could actually omit this attribute entirely, but being explicit is good practice while you are learning.

Below the button, we have an empty `<div id="task-list">`. This is the container that will receive the tasks when the button is clicked. Initially it is empty -- the page loads with no tasks visible.

### Step 3: Add the `/tasks` Endpoint

Now we need the server to handle `GET /tasks` and return an HTML fragment. Add a new case to your router:

```gleam
import gleam/http
import wisp.{type Request, type Response}

fn handle_request(req: Request) -> Response {
  case wisp.path_segments(req) {
    [] -> {
      use <- wisp.require_method(req, http.Get)
      let body = element.to_document_string(home_page())
      wisp.html_response(body, 200)
    }

    ["tasks"] -> {
      use <- wisp.require_method(req, http.Get)
      let tasks_html =
        html.ul([], [
          html.li([], [element.text("Set up project repository")]),
          html.li([], [element.text("Design the database schema")]),
          html.li([], [element.text("Build the landing page")]),
        ])
      let body = element.to_string(tasks_html)
      wisp.html_response(body, 200)
    }

    _ -> wisp.not_found()
  }
}
```

Pay close attention to the `/tasks` handler:

1. **`use <- wisp.require_method(req, http.Get)`** -- We only accept GET requests for this route, which matches the `hx-get` on the button. You need `import gleam/http` for the `http.Get` value.
2. **`html.ul([], [...])`** -- We build just a `<ul>` with some `<li>` items. No page wrapper. No `<html>`, no `<head>`, no `<body>`. Just the fragment.
3. **`element.to_string(tasks_html)`** -- Notice this is `element.to_string`, **not** `element.to_document_string`. This renders just the element as-is, producing something like:

```html
<ul><li>Set up project repository</li><li>Design the database schema</li><li>Build the landing page</li></ul>
```

That raw HTML fragment is exactly what HTMX expects.

### Step 4: Make Sure HTMX Is Loaded

For any of this to work, the HTMX JavaScript library must be loaded on the page. If you have been following along from the previous chapters, your page layout should already include the HTMX script from a CDN in the `<head>`:

```gleam
fn home_page() -> Element(t) {
  html.html([], [
    html.head([], [
      html.meta([attribute.attribute("charset", "utf-8")]),
      html.meta([
        attribute.name("viewport"),
        attribute.attribute("content", "width=device-width, initial-scale=1"),
      ]),
      html.title([], "Teamwork"),
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
    ]),
    html.body([], [home_content()]),
  ])
}
```

The `<script>` tag loads HTMX from a CDN. When the page loads, HTMX scans the DOM for elements with `hx-*` attributes and sets up the event listeners automatically. You do not need to write any JavaScript -- HTMX handles everything.

### Step 5: Watch It Work

Start your server:

```shell
gleam run
```

Open your browser to `http://localhost:8000`. You should see:

- The "Teamwork" heading
- The paragraph text
- A "Load Tasks" button
- An empty area below the button (the `#task-list` div)

Click the "Load Tasks" button. The task list appears instantly below the button, without a page reload.

### What Happened, Step by Step

1. **The page loaded normally.** The browser requested `GET /` and received a full HTML page (with doctype, head, body, and the HTMX script). HTMX initialized and noticed the button has `hx-get="/tasks"`.

2. **You clicked the button.** HTMX intercepted the click event (the default trigger for buttons).

3. **HTMX sent an AJAX request.** It made `GET /tasks` in the background -- you can see this in the Network tab of your browser's developer tools. The request includes a header `HX-Request: true` so the server knows it came from HTMX.

4. **The server returned a fragment.** Our `/tasks` handler returned just `<ul><li>...</li></ul>` -- raw HTML, no page wrapper.

5. **HTMX swapped the response into the DOM.** It found the element matching `#task-list` (from `hx-target`) and replaced its inner HTML (from `hx-swap="innerHTML"`) with the server's response.

6. **The page updated.** The task list appeared inside the previously empty div. No page reload. No flicker. No JavaScript written by you.

### Verifying in Browser Developer Tools

Open your browser's developer tools (F12 or Ctrl+Shift+I) and go to the **Network** tab. Click the "Load Tasks" button and observe:

- A new request appears: `GET /tasks`, type `xhr` (XMLHttpRequest)
- Click on it to inspect the details:
  - **Request Headers** include `HX-Request: true`, `HX-Target: task-list`, and `HX-Trigger: null` -- HTMX adds these automatically
  - **Response Headers** show `content-type: text/html`
  - **Response Body** is just the HTML fragment: `<ul><li>Set up project repository</li>...</ul>`

This is a great habit to build. Whenever something does not work as expected with HTMX, the Network tab is your first debugging tool. Check whether the request is being made, what URL it hits, and what the server returns.

### Why the `hx` Library Matters

You might wonder: why not just use raw string attributes?

```gleam
// Without hx library -- works but fragile:
attribute.attribute("hx-get", "/tasks")
attribute.attribute("hx-target", "#task-list")
attribute.attribute("hx-swap", "innerHTML")

// With hx library -- type-safe and compiler-checked:
hx.get("/tasks")
hx.target(hx.Selector("#task-list"))
hx.swap(hx.InnerHTML)
```

Both approaches produce identical HTML. But the `hx` library gives you:

- **Compile-time checking.** If you write `hx.swap(hx.InnerHTML)`, the Gleam compiler ensures `InnerHTML` is a valid swap strategy. If you write `attribute.attribute("hx-swap", "innerHtml")` with a typo, the compiler cannot help you.
- **Discoverability.** Your editor's autocomplete shows all valid `Swap` variants (`InnerHTML`, `OuterHTML`, `TextContent`, `Beforebegin`, `Afterbegin`, `Beforeend`, `Afterend`, `Delete`, `SwapNone`). No need to memorize them or check the HTMX docs.
- **The `Selector` type.** Instead of remembering the special HTMX extended CSS selector syntax, you use Gleam values like `hx.Selector("#id")`, `hx.This`, `hx.Closest(".class")`, `hx.Find("span")`, and so on.

Throughout this course, we will use the `hx` library exclusively.

---

## Full Code Listing

Here is the complete, updated project.

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
import hx
import lustre/attribute
import lustre/element.{type Element}
import lustre/element/html
import mist
import wisp.{type Request, type Response}
import wisp/wisp_mist

// --- Main ---

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

// --- Router ---

fn handle_request(req: Request) -> Response {
  case wisp.path_segments(req) {
    // Home page -- returns a full HTML document
    [] -> {
      use <- wisp.require_method(req, http.Get)
      let body = element.to_document_string(home_page())
      wisp.html_response(body, 200)
    }

    // Tasks endpoint -- returns an HTML fragment
    ["tasks"] -> {
      use <- wisp.require_method(req, http.Get)
      let tasks_html = tasks_fragment()
      let body = element.to_string(tasks_html)
      wisp.html_response(body, 200)
    }

    // Everything else -- 404
    _ -> wisp.not_found()
  }
}

// --- Pages (full HTML documents) ---

fn home_page() -> Element(t) {
  html.html([], [
    html.head([], [
      html.meta([attribute.attribute("charset", "utf-8")]),
      html.meta([
        attribute.name("viewport"),
        attribute.attribute("content", "width=device-width, initial-scale=1"),
      ]),
      html.title([], "Teamwork"),
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
      html.style([], "
        body { font-family: system-ui, sans-serif; max-width: 800px; margin: 0 auto; padding: 2rem; }
        button { padding: 0.5rem 1rem; font-size: 1rem; cursor: pointer; margin-right: 0.5rem; }
        ul { list-style: none; padding: 0; }
        li { padding: 0.5rem; border-bottom: 1px solid #eee; }
      "),
    ]),
    html.body([], [home_content()]),
  ])
}

fn home_content() -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    html.p([], [element.text("Your collaborative task board.")]),
    html.button(
      [
        hx.get("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.InnerHTML),
      ],
      [element.text("Load Tasks")],
    ),
    html.div([attribute.id("task-list")], []),
  ])
}

// --- Fragments (partial HTML for HTMX responses) ---

fn tasks_fragment() -> Element(t) {
  html.ul([], [
    html.li([], [element.text("Set up project repository")]),
    html.li([], [element.text("Design the database schema")]),
    html.li([], [element.text("Build the landing page")]),
  ])
}
```

### Project Structure

```
teamwork/
├── gleam.toml
├── src/
│   └── teamwork.gleam
└── test/
    └── teamwork_test.gleam
```

---

## Exercise

Now it is your turn. Extend the Teamwork app with the following tasks:

### Task 1: Add a "Load Team Members" Button

Add a second button to the home page labeled "Load Team Members". When clicked, it should:

- Send a GET request to `/team`
- Display the response in a new `<div id="team-list">` on the page
- The `/team` endpoint should return a `<ul>` with at least three team member names

### Task 2: Experiment with `OuterHTML`

Change the swap strategy on your "Load Team Members" button from `hx.InnerHTML` to `hx.OuterHTML`:

```gleam
hx.swap(hx.OuterHTML)
```

Click the button and observe what happens. Then try clicking it a second time. What changed? Why does the second click not work?

### Task 3: Multiple Buttons, Same Target

Add a third button labeled "Load Milestones" that fetches from `/milestones` and displays the result in the **same** `#task-list` div that the "Load Tasks" button uses. Click each button back and forth to see the content swap.

### Task 4: Inspect in Developer Tools

Open your browser's developer tools (F12), go to the Network tab, and:

1. Click "Load Tasks" and inspect the request and response
2. Click "Load Team Members" and compare
3. Look at the request headers -- find the `HX-Request` and `HX-Target` headers
4. Look at the response body -- confirm it is a fragment, not a full page

---

## Exercise Solution Hints

### Hint for Task 1

You need two things: a new button in `home_content` and a new route in `handle_request`.

For the button, add it alongside the existing "Load Tasks" button:

```gleam
html.button(
  [
    hx.get("/team"),
    hx.target(hx.Selector("#team-list")),
    hx.swap(hx.InnerHTML),
  ],
  [element.text("Load Team Members")],
),
html.div([attribute.id("team-list")], []),
```

For the route, add a new pattern match case in `handle_request`:

```gleam
["team"] -> {
  use <- wisp.require_method(req, http.Get)
  let team_html =
    html.ul([], [
      html.li([], [element.text("Alice -- Project Manager")]),
      html.li([], [element.text("Bob -- Backend Developer")]),
      html.li([], [element.text("Carol -- Frontend Developer")]),
    ])
  let body = element.to_string(team_html)
  wisp.html_response(body, 200)
}
```

### Hint for Task 2

When you use `hx.swap(hx.OuterHTML)`, the **entire target element** is replaced by the response -- not just its contents. So after the first click, the `<div id="team-list">` is replaced by the `<ul>` from the server response. Since there is no longer a `<div id="team-list">` in the DOM, the second click has nowhere to put the response.

To make `OuterHTML` work repeatedly, the server's response must include an element with the same ID:

```gleam
["team"] -> {
  use <- wisp.require_method(req, http.Get)
  let team_html =
    html.div([attribute.id("team-list")], [
      html.ul([], [
        html.li([], [element.text("Alice -- Project Manager")]),
        html.li([], [element.text("Bob -- Backend Developer")]),
        html.li([], [element.text("Carol -- Frontend Developer")]),
      ]),
    ])
  let body = element.to_string(team_html)
  wisp.html_response(body, 200)
}
```

Now the response wraps the `<ul>` inside a new `<div id="team-list">`, so the target still exists after the swap. This is an important pattern to remember when using `OuterHTML`.

### Hint for Task 3

The key insight is that `hx-target` is just a CSS selector -- multiple buttons can target the same element. Add the new button:

```gleam
html.button(
  [
    hx.get("/milestones"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
  ],
  [element.text("Load Milestones")],
),
```

And add the corresponding route:

```gleam
["milestones"] -> {
  use <- wisp.require_method(req, http.Get)
  let milestones_html =
    html.ul([], [
      html.li([], [element.text("v0.1 -- Project scaffolding")]),
      html.li([], [element.text("v0.2 -- User authentication")]),
      html.li([], [element.text("v0.3 -- Task management")]),
    ])
  let body = element.to_string(milestones_html)
  wisp.html_response(body, 200)
}
```

Both the "Load Tasks" and "Load Milestones" buttons target `#task-list`, so clicking either one replaces the content of that div. This is a natural and common pattern -- think of it like tabs, where different buttons load different content into the same area.

### Hint for Task 4

In the Network tab, you should see requests of type `xhr`. Click on a request and look at:

- **Request Headers**: `HX-Request: true` confirms it came from HTMX. `HX-Target: task-list` tells the server which element is being targeted (without the `#`).
- **Response Body**: should be just a `<ul>` with `<li>` items -- no `<!DOCTYPE>`, no `<html>`, no `<body>`.

If you see a full HTML page in the response body, you are using `element.to_document_string` instead of `element.to_string` for that route. Switch to `element.to_string`.

---

## Key Takeaways

1. **HTMX lets any element make HTTP requests.** You are not limited to links and forms. A button, a div, a table row -- anything can trigger a request using `hx-get` (or `hx-post`, `hx-put`, etc.).

2. **The core trio is `hx-get`, `hx-target`, and `hx-swap`.** Together they answer three questions: *What URL to request?* *Where does the response go?* *How is it inserted?*

3. **The server returns HTML fragments, not full pages.** HTMX endpoints should return just the piece of HTML that changed. Use `element.to_string` for fragments, not `element.to_document_string`.

4. **The `hx` library provides type-safe HTMX attributes for Lustre.** Functions like `hx.get`, `hx.target`, and `hx.swap` produce the same HTML attributes as writing them by hand, but with compile-time safety. The `Swap` type has variants like `InnerHTML`, `OuterHTML`, `Beforeend`, and others. The `Selector` type has variants like `Selector("#id")`, `This`, `Closest(".class")`, and more.

5. **`innerHTML` vs `outerHTML` matters.** With `innerHTML` (the default), the target element stays in the DOM and its contents are replaced. With `outerHTML`, the target element itself is replaced. If you use `outerHTML`, make sure the response includes a replacement element with the same ID if you want to swap again later.

6. **The Network tab is your best friend.** When debugging HTMX, always check the Network tab to see what request was sent and what HTML came back. HTMX adds helpful headers like `HX-Request` and `HX-Target` that you can inspect.

---

## What's Next

In this chapter, you made a button load content from the server. That is genuinely useful, but the data is still hardcoded. In Chapter 5, we will add `hx-post` to create new tasks from a form, introducing the request-response cycle for *writing* data, not just reading it. The Teamwork board is about to become truly interactive.
