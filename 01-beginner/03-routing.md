# Chapter 3 -- Routing: Multiple Pages

## Learning Objectives

By the end of this chapter you will be able to:

- Describe the anatomy of a URL: scheme, host, port, path, and query string.
- Use `wisp.path_segments` and Gleam pattern matching to route requests to different handler functions.
- Explain the `use` keyword in Gleam and how it enables middleware composition.
- Create a middleware pipeline that handles logging, static files, and routing.
- Return a styled 404 page for unknown paths.
- Understand HTTP methods and how they map to CRUD operations.

---

## Theory

### The Anatomy of a URL

Every time you type an address into your browser you are constructing a URL
(Uniform Resource Locator). A URL has several parts, and understanding them is
essential before we talk about routing.

```
  https://localhost:8000/boards/42?sort=date#tasks
  \___/   \_______/ \__/ \________/ \_______/ \___/
scheme      host   port    path      query   fragment
```

| Part       | Example          | Purpose                                                  |
|------------|------------------|----------------------------------------------------------|
| **Scheme** | `https`          | The protocol. We use `http` during local development.    |
| **Host**   | `localhost`      | The server's domain name or IP address.                  |
| **Port**   | `8000`           | The network port. Defaults to 80 (http) or 443 (https).  |
| **Path**   | `/boards/42`     | Identifies *which resource* on the server we want.       |
| **Query**  | `?sort=date`     | Extra parameters, usually filters or options.            |
| **Fragment**| `#tasks`        | Client-side only; the server never sees it.              |

For routing purposes, the **path** is the star of the show. Our server inspects
the path and decides which handler function should produce the response.

### How Web Frameworks Do Routing

Routing is the process of mapping an incoming URL path to the function that
should handle it. Different frameworks take different approaches:

- **Decorator-based** (Python/Flask): you annotate a function with `@app.route("/about")`.
- **File-based** (Next.js, SvelteKit): the file system itself defines the routes.
- **Pattern-matching** (Gleam/Wisp): you match on the list of path segments directly.

Gleam's approach is beautifully simple. The function `wisp.path_segments(req)`
takes a request and returns a `List(String)` -- the path split on `/` with
leading and trailing slashes removed:

| URL Path         | `wisp.path_segments(req)` returns |
|------------------|-----------------------------------|
| `/`              | `[]`                              |
| `/about`         | `["about"]`                       |
| `/boards/1`      | `["boards", "1"]`                 |
| `/boards/1/tasks`| `["boards", "1", "tasks"]`        |

Because this is a plain list, we can use Gleam's `case` expression to match on
it. There is no routing DSL to learn, no regular expressions to puzzle over --
just pattern matching, the same tool you already use for everything else in
Gleam.

### HTTP Methods

Before we write our router, let's talk briefly about HTTP methods. Every HTTP
request carries a *method* (sometimes called a *verb*) that tells the server
what kind of operation the client wants to perform:

| Method     | Typical Meaning            | CRUD Operation |
|------------|----------------------------|----------------|
| **GET**    | Retrieve a resource        | Read           |
| **POST**   | Create a new resource      | Create         |
| **PUT**    | Replace a resource entirely| Update         |
| **PATCH**  | Partially update a resource| Update         |
| **DELETE** | Remove a resource          | Delete         |

For now, all of our pages will respond to GET requests. In later chapters we
will handle POST for form submissions and DELETE for removing tasks. Wisp lets
you match on the method with `req.method`:

```gleam
case req.method {
  http.Get -> show_page()
  http.Post -> create_thing()
  _ -> wisp.method_not_allowed([http.Get, http.Post])
}
```

We will not use method matching yet, but keep it in mind -- it becomes important
the moment we start handling forms.

### The `use` Keyword in Gleam

Before we look at middleware, we need to understand one of Gleam's most
distinctive features: the `use` keyword.

In many languages, middleware involves decorators, inheritance, or framework
magic. In Gleam, middleware is just a function that takes a callback -- and
`use` is syntactic sugar that makes calling such functions ergonomic.

Here is a concrete example. Suppose you have a function `log_request` with this
signature:

```gleam
fn log_request(
  req: Request,
  next: fn() -> Response,
) -> Response
```

It takes a request and a *continuation* -- a function to call after it finishes
logging. Without `use`, calling it looks like this:

```gleam
fn handle_request(req) {
  log_request(req, fn() {
    // ... the rest of your handler, nested inside a callback ...
    wisp.ok()
  })
}
```

With `use`, you can flatten the nesting:

```gleam
fn handle_request(req) {
  use <- log_request(req)
  // ... the rest of your handler, no nesting ...
  wisp.ok()
}
```

The line `use <- log_request(req)` means: "Call `log_request` with `req` and a
callback. Everything below this line *is* the callback."

The `<-` arrow tells Gleam: "the last argument to `log_request` is a function
whose body is the remainder of this block."

This is not special framework syntax. It works with *any* function whose last
parameter is a callback. You will see it used for middleware, for database
transactions, for resource cleanup -- anywhere the "wrap some work around other
work" pattern appears.

### Middleware Composition

A middleware is a function that wraps a handler. It can:

1. Inspect or modify the request *before* the handler runs.
2. Inspect or modify the response *after* the handler runs.
3. Short-circuit and return a response without calling the handler at all.

With `use`, we can chain several middleware functions in sequence, and each one
wraps the rest:

```gleam
fn middleware(
  req: wisp.Request,
  handle_request: fn(wisp.Request) -> wisp.Response,
) -> wisp.Response {
  let req = wisp.method_override(req)
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  handle_request(req)
}
```

Reading this from top to bottom:

1. `wisp.method_override(req)` checks for a `_method` field in form data and
   overrides the HTTP method accordingly. (HTML forms only support GET and POST,
   so this lets us simulate PUT and DELETE.) It returns a new request.
2. `wisp.log_request(req)` logs the request method and path, then calls the
   continuation (everything below it).
3. `wisp.serve_static` checks if the request path starts with `/static`. If it
   does, it serves the file and never calls the continuation. If it does not
   match, it calls the continuation.
4. If we get past the static file check, we call `handle_request(req)` -- our
   actual router.

The beauty of this design is that each middleware is independent. You can
reorder them, add new ones, or remove them without touching the others.

### Why 404 Pages Matter

When a user visits a path that does not exist, the server should respond with
HTTP status code 404 (Not Found). A bare 404 with no body is technically valid,
but it creates a terrible experience: the user sees a blank page or a generic
browser error.

A good 404 page should:

- Clearly tell the user the page was not found.
- Maintain the site's look and feel (use the same layout).
- Provide a link back to the home page or other useful navigation.
- Return the 404 status code so search engines know not to index the page.

---

## Code Walkthrough

We are going to refactor the code from Chapter 2. In that chapter, our
`handle_request` function returned the same HTML page for every path. Now we
will give our Teamwork app three distinct pages -- Home, About, and a custom 404
-- and set up a proper middleware pipeline.

### Step 1: Separate Middleware from Routing

In Chapter 2 everything lived inside `handle_request`. Let's split it into two
functions: `middleware` (cross-cutting concerns) and `handle_request` (routing).

Open `src/teamwork.gleam` and update it. First, make sure you have the
necessary imports at the top:

```gleam
import gleam/erlang/process
import lustre/attribute
import lustre/element
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist
```

Now let's write the middleware function. This function will be the entry point
that the Mist server calls. It receives the raw request, applies cross-cutting
concerns, and then delegates to our router:

```gleam
fn middleware(
  req: wisp.Request,
  handle_request: fn(wisp.Request) -> wisp.Response,
) -> wisp.Response {
  let req = wisp.method_override(req)
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  handle_request(req)
}
```

Notice the signature of `middleware`: it takes a request *and* a function
`handle_request`. This is the standard Wisp middleware pattern. In our `main`
function we will wire them together:

```gleam
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
```

Wisp calls `middleware` internally as part of its handler pipeline. The function
`wisp_mist.handler` takes our `handle_request` function and the secret key. The
middleware wrapping happens because Wisp's built-in handler already applies the
standard middleware. However, if you want explicit control (which we do), you
can pass `middleware` as a wrapper. Let's use the explicit pattern:

```gleam
pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let handler = fn(req) { middleware(req, handle_request) }

  let assert Ok(_) =
    wisp_mist.handler(handler, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

Here we create a `handler` closure that wires middleware and routing together.
When a request comes in, Mist passes it to `wisp_mist.handler`, which converts
it into a Wisp request and calls our `handler` closure. That closure calls
`middleware`, which applies logging and static file serving, and then calls
`handle_request` -- our router.

### Step 2: The Router

The `handle_request` function is now purely about routing. It inspects the path
and dispatches to the right page handler:

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    [] -> home_page(req)
    ["about"] -> about_page(req)
    _ -> not_found_page()
  }
}
```

Let's trace through a few example requests:

| Request Path | `path_segments` returns | Matches         | Handler called   |
|--------------|-------------------------|-----------------|------------------|
| `/`          | `[]`                    | `[]`            | `home_page`      |
| `/about`     | `["about"]`             | `["about"]`     | `about_page`     |
| `/contact`   | `["contact"]`           | `_` (catch-all) | `not_found_page` |
| `/boards/1`  | `["boards", "1"]`       | `_` (catch-all) | `not_found_page` |

The `_` pattern at the bottom is the catch-all. Any path that does not match a
defined route will hit this branch and receive a 404 response. This is why order
matters: put specific matches first and the catch-all last.

### Step 3: The Layout Function

Before writing our page handlers, let's revisit the `layout` function from
Chapter 2. We need it to accept a page title and a list of child elements that
form the page's content:

```gleam
fn layout(title: String, content: List(element.Element(t))) -> element.Element(t) {
  html.html([], [
    html.head([], [
      html.title([], title),
      html.meta([attribute.attribute("charset", "UTF-8")]),
      html.meta([
        attribute.name("viewport"),
        attribute.attribute("content", "width=device-width, initial-scale=1"),
      ]),
      html.link([
        attribute.rel("stylesheet"),
        attribute.href("/static/css/style.css"),
      ]),
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
    ]),
    html.body([], [
      html.nav([], [
        html.ul([], [
          html.li([], [
            html.a([attribute.href("/")], [element.text("Home")]),
          ]),
          html.li([], [
            html.a([attribute.href("/about")], [element.text("About")]),
          ]),
        ]),
      ]),
      html.main([], content),
    ]),
  ])
}
```

This function:

1. Takes a `title` string for the `<title>` tag.
2. Takes a `content` list of Lustre elements to render inside `<main>`.
3. Includes a `<nav>` with links to our two pages.
4. Loads our CSS from `/static/css/style.css` (served by the static middleware).
5. Loads HTMX from a CDN. We are not using HTMX features yet, but having it
   loaded means we are ready for the next chapter.

The return type `element.Element(t)` uses a type parameter `t`. Lustre elements
are generic over the type of messages they can produce (used in client-side
Lustre apps). On the server side we do not use messages, so we leave it as a
type variable and let the compiler infer it.

### Step 4: Page Handlers

Now let's write the individual page handlers. Each one builds content using
Lustre, passes it to `layout`, converts the result to an HTML string, and
returns a Wisp response.

**Home page:**

```gleam
fn home_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("Teamwork", [
      html.h1([], [element.text("Teamwork")]),
      html.p([], [
        element.text("Your collaborative task board."),
      ]),
      html.p([], [
        element.text(
          "Organize your team's work with boards, lists, and cards.",
        ),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

**About page:**

```gleam
fn about_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("About -- Teamwork", [
      html.h1([], [element.text("About Teamwork")]),
      html.p([], [
        element.text("Built with Gleam, Wisp, and HTMX."),
      ]),
      html.p([], [
        element.text(
          "Teamwork is a progressive project that grows chapter by chapter. "
          <> "Each chapter introduces new HTMX concepts alongside Gleam features.",
        ),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

Notice that we prefix the `req` parameter with an underscore (`_req`) because
we are not using it yet. Gleam's compiler warns about unused variables, and the
underscore prefix tells it: "I know, and that's intentional."

### Step 5: A Custom 404 Page

Instead of returning a bare `wisp.not_found()` (which sends a 404 with an empty
body), we build a full HTML page:

```gleam
fn not_found_page() -> wisp.Response {
  let page =
    layout("Not Found -- Teamwork", [
      html.h1([], [element.text("404 -- Page Not Found")]),
      html.p([], [
        element.text("The page you're looking for doesn't exist."),
      ]),
      html.a([attribute.href("/")], [
        element.text("Go back home"),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 404)
}
```

The key difference from the other handlers: notice the second argument to
`wisp.html_response` is `404` instead of `200`. This ensures the HTTP status
code correctly communicates "not found" to the browser and to search engines,
even though the user sees a friendly, styled page.

Also notice that `not_found_page` does not take a `Request` parameter -- it
does not need any information from the request to render the error page.

### Step 6: The Static Directory Helper

We need the helper function that tells Wisp where our static files live:

```gleam
fn static_directory() -> String {
  let assert Ok(priv_directory) = wisp.priv_directory("teamwork")
  priv_directory <> "/static"
}
```

This uses `wisp.priv_directory` to find the `priv` directory of our `teamwork`
package. During development this points to the `priv/` folder in your project
root. Make sure you have at least the directory structure `priv/static/css/`
in your project.

### Understanding the Request Flow

Let's trace the full lifecycle of a request to `GET /about`:

```
Browser sends: GET /about HTTP/1.1

1. Mist receives the raw HTTP request.
2. wisp_mist.handler converts it to a wisp.Request.
3. Our handler closure calls middleware(req, handle_request).
4. middleware:
   a. wisp.method_override(req) -- no _method field, req unchanged.
   b. wisp.log_request(req) -- logs "GET /about" to the console.
      Then calls the continuation (everything below).
   c. wisp.serve_static(req, ...) -- "/about" does not start with "/static",
      so it calls the continuation (everything below).
   d. handle_request(req) is called.
5. handle_request:
   a. wisp.path_segments(req) returns ["about"].
   b. Pattern match: ["about"] matches the second branch.
   c. about_page(req) is called.
6. about_page:
   a. Builds the HTML using layout + Lustre elements.
   b. Converts to a string with element.to_document_string.
   c. Returns wisp.html_response(..., 200).
7. The response flows back up through the middleware chain.
8. Mist sends the HTTP response to the browser.
```

Now let's trace `GET /static/css/style.css`:

```
1-4b. Same as above.
4c. wisp.serve_static: "/static/css/style.css" starts with "/static".
    Wisp looks for priv/static/css/style.css.
    Found! Returns the file as a response.
    The continuation is NEVER called.
    handle_request is NEVER called.
```

This is the short-circuit behavior of middleware. `wisp.serve_static` handles
the request entirely on its own and never passes control to the router.

---

## Full Code Listing

Here is the complete `src/teamwork.gleam` file incorporating all the changes
from this chapter:

```gleam
// src/teamwork.gleam

import gleam/erlang/process
import lustre/attribute
import lustre/element
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let handler = fn(req) { middleware(req, handle_request) }

  let assert Ok(_) =
    wisp_mist.handler(handler, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}

fn middleware(
  req: wisp.Request,
  handle_request: fn(wisp.Request) -> wisp.Response,
) -> wisp.Response {
  let req = wisp.method_override(req)
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  handle_request(req)
}

fn handle_request(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    [] -> home_page(req)
    ["about"] -> about_page(req)
    _ -> not_found_page()
  }
}

fn static_directory() -> String {
  let assert Ok(priv_directory) = wisp.priv_directory("teamwork")
  priv_directory <> "/static"
}

fn layout(title: String, content: List(element.Element(t))) -> element.Element(t) {
  html.html([], [
    html.head([], [
      html.title([], title),
      html.meta([attribute.attribute("charset", "UTF-8")]),
      html.meta([
        attribute.name("viewport"),
        attribute.attribute("content", "width=device-width, initial-scale=1"),
      ]),
      html.link([
        attribute.rel("stylesheet"),
        attribute.href("/static/css/style.css"),
      ]),
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
    ]),
    html.body([], [
      html.nav([], [
        html.ul([], [
          html.li([], [
            html.a([attribute.href("/")], [element.text("Home")]),
          ]),
          html.li([], [
            html.a([attribute.href("/about")], [element.text("About")]),
          ]),
        ]),
      ]),
      html.main([], content),
    ]),
  ])
}

fn home_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("Teamwork", [
      html.h1([], [element.text("Teamwork")]),
      html.p([], [
        element.text("Your collaborative task board."),
      ]),
      html.p([], [
        element.text(
          "Organize your team's work with boards, lists, and cards.",
        ),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

fn about_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("About -- Teamwork", [
      html.h1([], [element.text("About Teamwork")]),
      html.p([], [
        element.text("Built with Gleam, Wisp, and HTMX."),
      ]),
      html.p([], [
        element.text(
          "Teamwork is a progressive project that grows chapter by chapter. "
          <> "Each chapter introduces new HTMX concepts alongside Gleam features.",
        ),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

fn not_found_page() -> wisp.Response {
  let page =
    layout("Not Found -- Teamwork", [
      html.h1([], [element.text("404 -- Page Not Found")]),
      html.p([], [
        element.text("The page you're looking for doesn't exist."),
      ]),
      html.a([attribute.href("/")], [
        element.text("Go back home"),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 404)
}
```

Run the server with:

```sh
gleam run
```

Then test the following URLs in your browser:

- [http://localhost:8000/](http://localhost:8000/) -- You should see the home page.
- [http://localhost:8000/about](http://localhost:8000/about) -- You should see the about page.
- [http://localhost:8000/anything-else](http://localhost:8000/anything-else) -- You should see the 404 page.

Check your terminal too -- `wisp.log_request` should be printing a log line for
each request, showing the method, path, status code, and response time.

---

## Exercise

Now it is your turn. Extend the Teamwork app with the following changes:

### Task 1: Add a `/contact` Route

Create a `contact_page` handler that renders a page with some placeholder
contact information. Add a `["contact"]` branch to the router and a "Contact"
link to the navigation in `layout`.

### Task 2: Add a `/boards` Route

Create a `boards_page` handler that renders a page with the heading "Boards"
and a paragraph saying "Your boards will appear here." This is a placeholder
for the functionality we will build in later chapters. Add a `["boards"]`
branch to the router and a "Boards" link to the navigation.

### Task 3: Highlight the Current Navigation Link

Right now every navigation link looks the same regardless of which page you are
on. Update the `layout` function so it accepts the current path and applies a
CSS class (for example, `"active"`) to the link that matches the current page.

You will need to:

1. Change the `layout` function signature to accept the current path.
2. Update all callers of `layout` to pass the path.
3. Conditionally apply `attribute.class("active")` to the matching link.
4. Add a CSS rule to `priv/static/css/style.css` to style `.active` links
   (for example, bold text or an underline).

### Task 4: Verify 404 Behavior for Nested Paths

Visit [http://localhost:8000/boards/123](http://localhost:8000/boards/123) in
your browser. You should see the 404 page. We match `["boards"]` but not
`["boards", "123"]` -- the catch-all `_` handles it. In a future chapter we
will handle dynamic segments like this.

---

## Exercise Solution Hints

If you get stuck, here are some hints. Try to solve each task on your own
before reading these.

### Hint for Task 1

Add a new branch to the `case` expression in `handle_request`:

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    [] -> home_page(req)
    ["about"] -> about_page(req)
    ["contact"] -> contact_page(req)
    _ -> not_found_page()
  }
}
```

The handler follows the same pattern as the others:

```gleam
fn contact_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("Contact -- Teamwork", [
      html.h1([], [element.text("Contact")]),
      html.p([], [
        element.text("Get in touch at hello@teamwork.example.com"),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

Don't forget to add the "Contact" link to `layout`'s `<nav>`.

### Hint for Task 2

Same pattern as Task 1. Add `["boards"] -> boards_page(req)` to the router:

```gleam
fn boards_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("Boards -- Teamwork", [
      html.h1([], [element.text("Boards")]),
      html.p([], [
        element.text("Your boards will appear here."),
      ]),
    ])
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

### Hint for Task 3

Change `layout` to accept the current path as a `String`:

```gleam
fn layout(
  title: String,
  current_path: String,
  content: List(element.Element(t)),
) -> element.Element(t) {
  // ...
}
```

For each nav link, check if the current path matches and conditionally add the
`"active"` class. You can write a small helper:

```gleam
fn nav_link(
  href: String,
  label: String,
  current_path: String,
) -> element.Element(t) {
  let attrs = case current_path == href {
    True -> [attribute.href(href), attribute.class("active")]
    False -> [attribute.href(href)]
  }
  html.a(attrs, [element.text(label)])
}
```

Then use it in the nav:

```gleam
html.nav([], [
  html.ul([], [
    html.li([], [nav_link("/", "Home", current_path)]),
    html.li([], [nav_link("/about", "About", current_path)]),
    html.li([], [nav_link("/boards", "Boards", current_path)]),
    html.li([], [nav_link("/contact", "Contact", current_path)]),
  ]),
])
```

You can get the current path from the request using `req.path` or reconstruct
it from `wisp.path_segments(req)`. The simplest approach is to pass the path
string from each handler:

```gleam
fn home_page(_req: wisp.Request) -> wisp.Response {
  let page =
    layout("Teamwork", "/", [
      // content...
    ])
  // ...
}
```

For the CSS, add something like this to `priv/static/css/style.css`:

```css
nav a.active {
  font-weight: bold;
  text-decoration: underline;
}
```

### Hint for Task 4

No code changes needed. Just visit `http://localhost:8000/boards/123` and
confirm you see your 404 page. The path `["boards", "123"]` does not match
`["boards"]`, so it falls through to `_`.

---

## Key Takeaways

1. **URLs have structure.** The path identifies the resource; the query string
   provides parameters. Routing means mapping paths to handlers.

2. **`wisp.path_segments(req)` splits the path into a `List(String)`.** This
   turns routing into pattern matching -- no special DSL required.

3. **The `use` keyword flattens callbacks.** `use <- some_function(args)` calls
   `some_function` with a callback made from the rest of the function body.
   This is how Gleam avoids deeply nested middleware chains.

4. **Middleware is just function composition.** Each middleware wraps the next
   handler, optionally modifying the request or response, or short-circuiting
   entirely (as `serve_static` does when it finds a matching file).

5. **Always define a catch-all route.** The `_` pattern at the end of your
   router ensures unknown paths get a proper 404 response instead of a crash.

6. **Custom 404 pages matter.** Return the correct status code (404) but render
   a full, styled page that helps the user find their way back.

7. **HTTP methods indicate intent.** GET reads, POST creates, PUT/PATCH
   updates, DELETE removes. For now we only handle GET; forms and mutations
   come in later chapters.

---

## What's Next

In [Chapter 4](04-first-htmx-interaction.md) we will introduce HTMX's `hx-get` attribute to load content
dynamically without full page reloads. We will add a board list to the
`/boards` page and load board details with HTMX partial responses -- your first
taste of hypermedia-driven interactivity.
