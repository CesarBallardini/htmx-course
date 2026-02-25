# Chapter 1 -- How the Web Actually Works

**Phase:** Beginner
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Describe the HTTP request/response cycle and identify its core components (methods, status codes, headers, body).
- Explain the HTMX philosophy and how it differs from the JSON-API-plus-JavaScript-framework approach.
- Scaffold a new Gleam project with `gleam new`.
- Wire up a minimal HTTP server using Mist and Wisp.
- Write a `main()` function that starts the server and a `handle_request` function that returns an HTML response.
- Run the server and inspect what the browser actually sends and receives.

---

## 1. Theory

### 1.1 The HTTP Request/Response Cycle

Every interaction you have with a website follows a deceptively simple pattern.
Your browser sends a **request**, and a server somewhere sends back a **response**.
That is the entire conversation. There is no persistent channel, no magical
connection that stays open (we will ignore WebSockets for now). Each exchange is
independent, stateless, and self-contained.

Here is what that looks like:

```
           REQUEST
 ┌────────┐ ─────────────────────────► ┌────────┐
 │        │  GET /tasks HTTP/1.1       │        │
 │Browser │  Host: localhost:8000      │ Server │
 │        │  Accept: text/html         │(Gleam) │
 │        │                            │        │
 │        │ ◄───────────────────────── │        │
 └────────┘        RESPONSE            └────────┘
              HTTP/1.1 200 OK
              Content-Type: text/html

              <h1>Your Tasks</h1>
              <ul>
                <li>Buy milk</li>
              </ul>
```

Let's break down the pieces.

**The Request** consists of:

| Part          | Example                     | Purpose                                       |
|---------------|-----------------------------|-----------------------------------------------|
| **Method**    | `GET`                       | What you want to do (read, create, update...)  |
| **Path**      | `/tasks`                    | Which resource you are asking about            |
| **Headers**   | `Accept: text/html`         | Metadata about the request                     |
| **Body**      | *(empty for GET)*           | Data you are sending (used in POST, PUT, etc.) |

**The Response** consists of:

| Part            | Example                   | Purpose                                   |
|-----------------|---------------------------|-------------------------------------------|
| **Status Code** | `200 OK`                  | Did it work? What happened?               |
| **Headers**     | `Content-Type: text/html` | Metadata about the response               |
| **Body**        | `<h1>Your Tasks</h1>...`  | The actual content -- HTML, JSON, an image, etc. |

**Common HTTP Methods:**

| Method   | Meaning                  | Typical Use                     |
|----------|--------------------------|---------------------------------|
| `GET`    | Read / retrieve          | Loading a page, fetching data   |
| `POST`   | Create                   | Submitting a form, adding a task |
| `PUT`    | Replace entirely         | Updating a full resource        |
| `PATCH`  | Partial update           | Changing one field              |
| `DELETE` | Remove                   | Deleting a task                 |

**Common Status Codes:**

| Code  | Meaning               | When You See It                          |
|-------|-----------------------|------------------------------------------|
| `200` | OK                    | Everything went fine                     |
| `201` | Created               | A new resource was successfully created  |
| `204` | No Content            | Success, but nothing to send back        |
| `301` | Moved Permanently     | The URL changed, follow the redirect     |
| `400` | Bad Request           | The server cannot understand your request|
| `404` | Not Found             | That resource does not exist             |
| `500` | Internal Server Error | Something broke on the server            |

This is the foundation. Every web framework, every API, every website in
existence is built on top of this simple exchange. Understand this, and
everything else is details.

### 1.2 How Traditional Web Apps Work

In the early web (and still in many applications today), the cycle looks like this:

1. The user clicks a link or submits a form.
2. The browser sends an HTTP request to the server.
3. The server processes the request, builds a **complete HTML page**, and sends it back.
4. The browser **throws away** the current page and renders the new one from scratch.

This is the "multi-page application" (MPA) model. It works. It has worked since
1993. But every interaction triggers a full page reload -- a white flash, a
scroll-to-top, a jarring interruption.

To solve this, the industry moved toward "single-page applications" (SPAs):

1. The server sends a mostly empty HTML shell plus a large JavaScript bundle.
2. JavaScript takes over. It makes `fetch()` calls to a JSON API.
3. The client-side framework (React, Vue, Angular, Svelte, etc.) receives JSON, maintains its own state, and updates the DOM.

This gives you smooth, no-reload interactions, but at a steep cost:

- You now maintain **two** applications: a JSON API backend and a JavaScript frontend.
- You need client-side state management (Redux, Zustand, signals, stores...).
- You need a build toolchain (Webpack, Vite, esbuild...).
- You need to handle client-side routing, error states, loading states, cache invalidation...
- The initial page load is heavy because the browser must download, parse, and execute a JavaScript bundle before anything appears.

For many applications -- especially CRUD apps, dashboards, internal tools, and content-driven sites -- this complexity is not justified.

### 1.3 The HTMX Philosophy

HTMX offers a third path. Its core idea can be stated in one sentence:

> **The server returns HTML, and HTMX swaps it into the page.**

Instead of the browser making JSON API calls and a JavaScript framework
rendering the result, HTMX extends HTML itself so that **any element** can make
HTTP requests and **any part of the page** can be updated with the HTML that
comes back.

Here is the conceptual shift:

```
  Traditional SPA:
    Browser  ──GET /api/tasks──►  Server
    Browser  ◄──── JSON ────────  Server
    JavaScript renders the JSON into DOM nodes.

  HTMX:
    Browser  ──GET /tasks──────►  Server
    Browser  ◄──── HTML ────────  Server
    HTMX swaps the HTML into a target element. Done.
```

With HTMX, you write your application logic **once**, on the server. The server
decides what HTML to return, and the client simply displays it. There is no
client-side state to manage, no build tools to configure, no framework to learn.
You just write HTML on the server.

This is what the HTMX documentation means when it says **"HTML as the engine
of application state."** The state of your application is always reflected in
the HTML the server produces. The browser is just a rendering engine -- which
is exactly what it was designed to be.

**What HTMX gives you:**

- Any element can issue an HTTP request (not just links and forms).
- Any HTTP method can be used (not just GET and POST).
- Any element on the page can be the target for the server's response.
- Requests can be triggered by any event (click, hover, keyup, scroll, load...).
- All of this is declared in HTML attributes -- no JavaScript required.

We will explore each of these capabilities in later chapters. For now, the
important thing is the mental model: **your server returns HTML fragments, and
HTMX puts them on the page.**

### 1.4 Why Hypermedia Is Powerful

HTMX is part of a broader movement sometimes called the "hypermedia renaissance."
The idea is that hypermedia (HTML with links and forms) was always the right
architecture for most web applications. The industry just got sidetracked by
the allure of thick clients and JSON APIs.

Hypermedia-driven applications have some compelling properties:

- **Simplicity.** One codebase, one language, one mental model. The server is
  the single source of truth.
- **Resilience.** If JavaScript fails to load, your links and forms still work
  (graceful degradation).
- **SEO.** Search engines see real HTML, not an empty `<div id="root">`.
- **Performance.** No multi-megabyte JavaScript bundles. HTML is fast to parse.
- **Accessibility.** Semantic HTML is inherently accessible. Screen readers
  understand `<a>`, `<button>`, `<form>` -- they do not understand
  `<div onClick={...}>`.

### 1.5 Enter Gleam

Gleam is a functional programming language that runs on the BEAM virtual machine
-- the same battle-tested runtime that powers Erlang and Elixir.

Here is what makes Gleam interesting for web development:

- **Type safety.** Gleam has a strong, static type system. If your code compiles,
  a whole class of bugs simply cannot happen. No `undefined is not a function`,
  no `null pointer exception`.
- **Friendly syntax.** If you have used Rust, TypeScript, or Elixir, Gleam will
  feel familiar. It is designed to be approachable.
- **BEAM concurrency.** The BEAM VM handles millions of lightweight processes.
  Each incoming HTTP request gets its own process, completely isolated from the
  others. One slow request cannot block the rest. One crash cannot take down the
  server.
- **Fault tolerance.** The BEAM's supervisor trees mean your application can
  recover from errors automatically. A single failing request does not bring
  down your server -- the process crashes, a supervisor restarts it, and
  everything else keeps running.
- **Interop.** Gleam can call Erlang and Elixir libraries directly. The entire
  BEAM ecosystem is at your disposal.

### 1.6 Why Gleam + HTMX Is a Good Combination

Gleam and HTMX share a philosophical alignment: **do less, but do it well.**

HTMX removes the need for a complex frontend framework. Gleam gives you a
type-safe, concurrent backend that is a joy to work with. Together, they let
you build interactive web applications with a single codebase, no build tools
on the frontend, and a runtime that can handle serious traffic.

Throughout this course, we will use the following stack:

| Layer         | Tool         | Role                                                 |
|---------------|--------------|------------------------------------------------------|
| HTTP server   | **Mist**     | Accepts TCP connections, parses HTTP                  |
| Web framework | **Wisp**     | Routing, middleware, request/response helpers          |
| HTML          | **Lustre**   | Type-safe HTML generation (server-side rendering)      |
| HTMX bindings | **hx**       | Typed HTMX attribute helpers for Lustre elements       |
| Interactivity | **HTMX**     | Client-side library loaded from CDN, swaps HTML fragments |
| Database      | **sqlight**  | SQLite bindings (introduced in a later chapter)        |

For this first chapter, we only need Mist and Wisp. We will add the other
pieces as we go.

---

## 2. Code Walkthrough

Let's build something. By the end of this section, you will have a running
HTTP server written in Gleam that responds with HTML.

### 2.1 Prerequisites

You need **Gleam** and **Erlang/OTP** installed. The easiest way is with
[mise](https://mise.jdx.dev), a polyglot tool version manager. If you do not
have mise yet, see **[Appendix C](../05-appendices/A3-installing-mise.md)** for detailed installation instructions on
Linux, macOS, and Windows.

Once mise is installed, run this from the project root:

```bash
mise install
```

The repository includes a `mise.toml` that pins compatible versions:

```toml
[tools]
erlang = "27"
gleam = "1"
```

Running `mise install` reads this file and installs Erlang/OTP 27 and the
latest Gleam 1.x. Gleam's build tool downloads **rebar3** (the Erlang build
tool it uses under the hood) automatically on first run, so you do not need
to install it separately.

> **Without mise.** You can install the tools manually instead:
>
> - **Gleam** (v1.0 or later): [gleam.run/getting-started/installing](https://gleam.run/getting-started/installing/)
> - **Erlang/OTP** (v26 or later): [erlang.org/downloads](https://www.erlang.org/downloads)
>
> Any recent Gleam 1.x and OTP 26+ combination will work.

Verify your installation:

```bash
gleam --version
# gleam 1.x.x

erl -eval 'erlang:display(erlang:system_info(otp_release)), halt().' -noshell
# "27"
```

### 2.2 Scaffold the Project

Open your terminal and run:

```bash
gleam new teamwork
cd teamwork
```

This creates a new Gleam project with the following structure:

```
teamwork/
├── gleam.toml          # Project configuration and dependencies
├── src/
│   └── teamwork.gleam  # Your main module
└── test/
    └── teamwork_test.gleam  # Tests
```

The `gleam.toml` file is where you declare metadata and dependencies. Open it
and replace its contents with:

```toml
name = "teamwork"
version = "1.0.0"
target = "erlang"

[dependencies]
gleam_stdlib = ">= 0.50.0 and < 1.0.0"
gleam_http = ">= 4.0.0 and < 5.0.0"
gleam_erlang = ">= 1.0.0 and < 2.0.0"
gleam_otp = ">= 1.0.0 and < 2.0.0"
wisp = ">= 2.0.0 and < 3.0.0"
mist = ">= 5.0.0 and < 6.0.0"

[dev-dependencies]
gleeunit = ">= 1.0.0 and < 2.0.0"
```

Let's walk through the dependencies:

- **gleam_stdlib**: The standard library. String manipulation, list operations, result handling, and more.
- **gleam_http**: Types for HTTP requests and responses. Wisp and Mist both build on these.
- **gleam_erlang**: Gleam bindings to core Erlang functionality, including the `process` module we need.
- **gleam_otp**: OTP (Open Telecom Platform) bindings. Supervisors, actors, and other concurrency primitives.
- **wisp**: A web framework that provides routing, middleware, and response helpers.
- **mist**: An HTTP server built in Gleam. It handles the low-level work of accepting connections and parsing HTTP.

Now download the dependencies:

```bash
gleam deps download
```

### 2.3 Write the Server

Open `src/teamwork.gleam` and replace its contents with the following. We will
go through it line by line afterward.

```gleam
import gleam/erlang/process
import mist
import wisp
import wisp/wisp_mist

pub fn main() {
  // Set up Wisp's logger so we can see requests in the terminal.
  wisp.configure_logger()

  // Wisp uses a secret key for signing cookies and other security features.
  // In production you would load this from an environment variable.
  // For development, a random string is fine.
  let secret_key_base = wisp.random_string(64)

  // Build the HTTP server:
  // 1. Create a Wisp handler from our handle_request function
  // 2. Wrap it in a Mist server
  // 3. Set the port to 8000
  // 4. Start listening for connections
  let assert Ok(_) =
    wisp_mist.handler(handle_request, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  // The server is now running in the background on BEAM processes.
  // We need to keep the main process alive, or the program will exit.
  process.sleep_forever()
}

fn handle_request(req: wisp.Request) -> wisp.Response {
  // Log every incoming request (method, path, status, timing).
  use <- wisp.log_request(req)

  // Build an HTML response with status 200.
  wisp.html_response("<h1>Hello from Teamwork!</h1>", 200)
}
```

### 2.4 Line-by-Line Explanation

**Imports:**

```gleam
import gleam/erlang/process
import mist
import wisp
import wisp/wisp_mist
```

Gleam uses explicit imports. There is no global namespace pollution -- you
import exactly what you need. The `gleam/erlang/process` module gives us
`process.sleep_forever()`, which we need to keep the main BEAM process alive.
The `wisp/wisp_mist` module is the glue between Wisp and Mist.

**The `main()` function:**

```gleam
pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)
  // ...
}
```

`pub fn` means this function is public -- it can be called from outside the
module. Gleam uses `pub fn main()` as the entry point, just like many other
languages.

`wisp.configure_logger()` hooks into Erlang's built-in logging system so that
Wisp can print request logs to your terminal.

`wisp.random_string(64)` generates a 64-character random string. Wisp needs
this for cryptographic operations like signing cookies. In a real application,
you would store this in an environment variable so it persists across restarts.

**Starting the server:**

```gleam
let assert Ok(_) =
  wisp_mist.handler(handle_request, secret_key_base)
  |> mist.new
  |> mist.port(8000)
  |> mist.start
```

This is a pipeline using the `|>` (pipe) operator. The pipe takes the result
of the expression on the left and passes it as the first argument to the
function on the right. Reading top to bottom:

1. `wisp_mist.handler(handle_request, secret_key_base)` -- Create an HTTP
   handler function that Mist understands. We pass in our `handle_request`
   function (more on that in a moment) and the secret key.
2. `|> mist.new` -- Create a new Mist server configuration from that handler.
3. `|> mist.port(8000)` -- Set the port to 8000.
4. `|> mist.start` -- Start the server. This returns `Ok(...)` on success
   or `Error(...)` on failure.

The `let assert Ok(_) =` line is a pattern match. It says: "I expect this to
return `Ok`. If it returns `Error`, crash immediately." This is appropriate
during startup -- if the server cannot start, there is no point continuing.
The underscore `_` means we do not need the value inside `Ok`.

**Keeping the process alive:**

```gleam
process.sleep_forever()
```

The BEAM runs your `main()` function in a process. When `main()` returns, that
process ends, and if it is the main process, the entire application shuts down.
But our HTTP server is running on *other* BEAM processes managed by Mist. We
need the main process to stay alive so the application keeps running.
`process.sleep_forever()` does exactly what the name says -- it blocks the main
process indefinitely.

**The request handler:**

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  use <- wisp.log_request(req)

  wisp.html_response("<h1>Hello from Teamwork!</h1>", 200)
}
```

This function takes a `wisp.Request` and returns a `wisp.Response`. That is
the entire job of a web server: receive requests, produce responses.

The `use <- wisp.log_request(req)` line deserves special attention. The `use`
keyword in Gleam is syntactic sugar for callbacks. What this line says is:
"Run `wisp.log_request`, and when it is done, continue with the rest of this
function." Under the hood, `wisp.log_request` takes the request, logs it, calls
our continuation (the rest of the function), gets the response back, logs the
status code and timing, and then returns the response.

You can think of `use` as middleware. We will use this pattern extensively in
later chapters for things like authentication, error handling, and database
transactions.

Finally, `wisp.html_response` creates a response with:
- A body from the HTML string we pass as the first argument.
- A status code of `200` (OK).
- A `Content-Type: text/html` header (set automatically by Wisp).

### 2.5 Run It

In your terminal, from the `teamwork` directory, run:

```bash
gleam run
```

The first run will take a moment as Gleam downloads and compiles all
dependencies. You should see output similar to:

```
  Compiling gleam_stdlib
  Compiling gleam_http
  Compiling gleam_erlang
  Compiling gleam_otp
  Compiling mist
  Compiling wisp
  Compiling teamwork
   Compiled in 5.23s
    Running teamwork.main
```

Now open your browser and navigate to:

```
http://localhost:8000
```

You should see:

> # Hello from Teamwork!

That is your first Gleam web server. It is running, it is serving HTML, and it
is logging every request to your terminal.

### 2.6 What Just Happened?

Let's trace the full path of that request:

1. You typed `http://localhost:8000` in your browser.
2. Your browser constructed an HTTP request:
   ```
   GET / HTTP/1.1
   Host: localhost:8000
   Accept: text/html,...
   ```
3. Mist, listening on port 8000, accepted the TCP connection and parsed the HTTP request.
4. Mist passed the request to the Wisp handler.
5. Wisp called your `handle_request` function with a `wisp.Request` value.
6. Your function logged the request, built an HTML string, and returned a `wisp.Response`.
7. Wisp passed the response back to Mist.
8. Mist serialized it into raw HTTP and sent it over the TCP connection:
   ```
   HTTP/1.1 200 OK
   Content-Type: text/html

   <h1>Hello from Teamwork!</h1>
   ```
9. Your browser received the response, saw `Content-Type: text/html`, parsed the HTML, and rendered it.

This is the request/response cycle in action. Every chapter in this course
follows this same pattern -- we are just going to make the request handling
progressively more sophisticated.

---

## 3. Full Code Listing

Here is everything you need, ready to copy and paste.

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
wisp = ">= 2.0.0 and < 3.0.0"
mist = ">= 5.0.0 and < 6.0.0"

[dev-dependencies]
gleeunit = ">= 1.0.0 and < 2.0.0"
```

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process
import mist
import wisp
import wisp/wisp_mist

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

fn handle_request(req: wisp.Request) -> wisp.Response {
  use <- wisp.log_request(req)

  wisp.html_response("<h1>Hello from Teamwork!</h1>", 200)
}
```

---

## 4. Exercise

Now it is your turn. Work through the following tasks to solidify what you have
learned.

### Task 1: Create and Run the Project

1. Run `gleam new teamwork` to scaffold the project.
2. Replace `gleam.toml` and `src/teamwork.gleam` with the code listings above.
3. Run `gleam run` and confirm you can see "Hello from Teamwork!" at `http://localhost:8000`.

**Acceptance criteria:** The browser displays the heading, and you see a request
log in your terminal.

### Task 2: Change the Response

Modify the `handle_request` function to return a more complete HTML page. Your
response should include:

- A proper `<!DOCTYPE html>` declaration.
- An `<html>` tag with a `<head>` containing a `<title>` of "Teamwork".
- A `<body>` containing an `<h1>` with the text "Welcome to Teamwork" and a
  `<p>` with a short description of the project.

**Acceptance criteria:** Viewing the page source in your browser (right-click,
"View Page Source") shows a complete HTML document.

### Task 3: Observe the Lack of Routing

With the server running, try visiting these URLs:

- `http://localhost:8000/`
- `http://localhost:8000/about`
- `http://localhost:8000/tasks`
- `http://localhost:8000/anything/you/want`

**Acceptance criteria:** All four URLs return the exact same response. Observe
this and think about why -- we will fix it in the next chapter when we add
routing.

### Task 4: Inspect HTTP Headers

Open your browser's developer tools (F12 or Ctrl+Shift+I) and go to the
**Network** tab. Reload the page and click on the request to `localhost`.
Examine:

- The **request headers**: What method was used? What `Accept` header did the
  browser send?
- The **response headers**: What `Content-Type` did the server send? What
  status code?

Alternatively, use `curl` from your terminal:

```bash
curl -v http://localhost:8000
```

The `-v` (verbose) flag shows you the full HTTP conversation, including all
headers.

**Acceptance criteria:** You can identify the request method (`GET`), the
response status (`200`), and the `Content-Type` header (`text/html`) in either
the browser dev tools or the `curl` output.

---

## 5. Exercise Solution Hints

If you get stuck, here are some pointers. Try to work through the exercises
on your own first.

### Hint for Task 1

If `gleam run` fails with an error about Erlang not being found, you need to
install Erlang/OTP. On macOS you can use `brew install erlang`. On Ubuntu/Debian,
`sudo apt install erlang`. On Windows, download the installer from
[https://www.erlang.org/downloads](https://www.erlang.org/downloads). Make
sure `erl` is on your PATH.

If you see a dependency resolution error, double-check that your `gleam.toml`
matches the listing exactly. Version constraints matter.

### Hint for Task 2

You need to build a larger HTML string. Remember that Gleam strings use double
quotes and can span multiple lines. Here is a starting point:

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  use <- wisp.log_request(req)

  let html = "
<!DOCTYPE html>
<html lang=\"en\">
<head>
  <meta charset=\"utf-8\">
  <title>Teamwork</title>
</head>
<body>
  <h1>Welcome to Teamwork</h1>
  <p>A collaborative task board built with Gleam and HTMX.</p>
</body>
</html>
"

  wisp.html_response(html, 200)
}
```

Notice how we escape the double quotes inside the HTML attributes with `\"`.
This works, but it is not pretty. In the next chapter, we will introduce
**Lustre** for type-safe HTML generation, which eliminates this problem entirely.

### Hint for Task 3

Every URL returns the same response because our `handle_request` function
ignores the request path entirely. It does not look at `req` (beyond passing it
to the logger). It simply returns the same HTML every time.

In [Chapter 2](02-serving-html-pages.md), we will introduce **Lustre** for generating HTML in a type-safe
way. That is how your server will produce real web pages instead of raw strings.

### Hint for Task 4

When you run `curl -v http://localhost:8000`, look for lines starting with
`>` (what you sent) and `<` (what you received). You should see something like:

```
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/8.x.x
> Accept: */*
>
< HTTP/1.1 200 OK
< content-type: text/html
< ...
<
<h1>Hello from Teamwork!</h1>
```

The `>` lines are the request. The `<` lines are the response. This is the
raw HTTP conversation, exactly as described in the theory section.

---

## 6. Key Takeaways

- **The web is built on HTTP request/response cycles.** A client sends a request
  (method, path, headers, optional body), and a server sends back a response
  (status code, headers, body). That is the whole model.

- **Traditional web apps return full HTML pages** on every request, causing full
  page reloads. SPAs moved rendering to the client with JavaScript, gaining
  interactivity but adding enormous complexity.

- **HTMX is a third path.** The server still returns HTML, but HTMX can swap
  fragments into the page without a full reload. You get SPA-like interactivity
  with server-side simplicity.

- **"HTML as the engine of application state"** means your server is the single
  source of truth. No client-side state management, no JSON serialization, no
  build tools.

- **Gleam is a type-safe, functional language on the BEAM VM.** It gives you
  Erlang's legendary concurrency and fault tolerance with a modern, friendly
  syntax and a strong type system.

- **Mist** is the HTTP server (handles connections and raw HTTP). **Wisp** is
  the web framework (provides routing, middleware, and helpers). They work
  together.

- **`gleam new`** scaffolds a project. **`gleam run`** compiles and runs it.

- **The `|>` pipe operator** threads a value through a chain of function calls,
  making code read top-to-bottom.

- **`use`** is Gleam's syntax for callbacks, commonly used for middleware-like
  patterns in Wisp (logging, authentication, etc.).

- **`let assert Ok(_) =`** is a pattern match that crashes on failure --
  appropriate for startup code where failure means "cannot continue."

- **`process.sleep_forever()`** keeps the main BEAM process alive so the
  server continues running.

---

## What's Next

In [Chapter 2](02-serving-html-pages.md), we will introduce **Lustre** for generating HTML in a type-safe
way and load **HTMX from a CDN** to prepare for our first interactive feature.
Then in [Chapter 3](03-routing.md), we will add **routing** -- pattern matching on the request
path to return different pages for different routes.

The Teamwork task board is just getting started.
