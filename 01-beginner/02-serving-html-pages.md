# Chapter 2 -- Serving HTML Pages

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why generating HTML through string concatenation is fragile and why type-safe HTML matters.
- Use Lustre's `html`, `element`, and `attribute` modules to build complete HTML documents in Gleam.
- Convert a Lustre element tree into an HTML string with `element.to_document_string`.
- Serve static files (CSS, images) using `wisp.serve_static`.
- Understand how the `Content-Type` header tells a browser what it is receiving.
- Load HTMX from a CDN so it is ready for the chapters ahead.

---

## Theory

### The Problem with String Concatenation

In Chapter 1 we returned a tiny HTML snippet as a raw string:

```gleam
"<h1>Hello from Teamwork!</h1>"
```

That worked fine for a single line, but imagine building an entire page this way:

```gleam
// DO NOT do this -- it is here only to illustrate the problem
let page =
  "<html><head><title>"
  <> title
  <> "</title></head><body><div class=\"container\"><h1>"
  <> heading
  <> "</h1><p>"
  <> body_text
  <> "</p></div></body></html>"
```

Several things can go wrong:

1. **Mismatched tags.** A missing `</div>` somewhere in the middle produces broken HTML, and the compiler cannot catch it because, to Gleam, it is all just a `String`.
2. **Escaping issues.** If `body_text` contains a `<` character, the browser will try to interpret it as the start of an HTML tag. You need to remember to escape every dynamic value, every time.
3. **Readability.** The structure of the document is buried inside quote characters and concatenation operators. Reviewing it in a pull request is painful.

We need a way to describe HTML that is **checked at compile time** and **mirrors the tree structure** of an actual document.

### Lustre -- A Type-Safe HTML DSL

Lustre is Gleam's primary UI library. It was designed as a full client-side framework (think Elm or React), but its HTML module works perfectly on the server side as a **domain-specific language for generating HTML**. When you write:

```gleam
html.div([attribute.class("container")], [
  html.h1([], [element.text("Hello")]),
])
```

Lustre builds an in-memory tree of elements. Each function enforces the correct shape:

- The first argument is always a `List(Attribute(t))` -- the element's attributes.
- The second argument is always a `List(Element(t))` -- the element's children.
- A few void elements like `html.meta` take only attributes (no children list).
- `html.title` takes a `List(Attribute(t))` and a `String` (the document title text), not a list of children.

If you pass the wrong type -- say, a `String` where an `Element` is expected -- the Gleam compiler will reject the code before it ever runs.

### Converting an Element Tree to HTML

Once you have built your tree, you call:

```gleam
element.to_document_string(tree)
```

This walks the tree and produces a `String` that begins with `<!doctype html>` followed by the serialised HTML. You then hand that string to Wisp, which sends it to the browser.

There is also `element.to_string`, which produces the HTML without the doctype. We will always use `to_document_string` for full pages because browsers behave more predictably in standards mode, which the doctype triggers.

### Content Types

When a web server sends a response, it includes a `Content-Type` header that tells the browser how to interpret the bytes. The three most common types you will encounter are:

| Content-Type           | What the browser does                          |
|------------------------|------------------------------------------------|
| `text/html`            | Parses and renders the response as a web page. |
| `text/plain`           | Displays the raw text with no formatting.      |
| `application/json`     | Treats the response as structured data (APIs). |

In Chapter 1, `wisp.html_response` set the content type to `text/html` for us automatically. We will continue using that helper. If you ever need to return JSON (for an API endpoint, for example), Wisp has `wisp.json_response`.

### A Note on the `hx` Library

Later in this course (Chapter 4) we will install a small library called `hx` that gives us **typed HTMX attributes** for Lustre. Instead of writing:

```gleam
attribute("hx-get", "/tasks")
```

you will write:

```gleam
hx.get("/tasks")
```

We mention it now so you know it exists. For this chapter we only need standard HTML attributes, so `hx` stays on the shelf for now.

---

## Code Walkthrough

We are going to make three changes to the project from Chapter 1:

1. Add Lustre as a dependency.
2. Rewrite `handle_request` to return a full HTML page built with Lustre.
3. Serve a static CSS file so the page looks presentable.

### Step 1 -- Add Lustre to Your Dependencies

Open `gleam.toml` and add `lustre` to the `[dependencies]` section:

```toml
[dependencies]
gleam_stdlib = ">= 0.50.0 and < 1.0.0"
gleam_erlang = ">= 1.0.0 and < 2.0.0"
gleam_http = ">= 4.0.0 and < 5.0.0"
gleam_otp = ">= 1.0.0 and < 2.0.0"
mist = ">= 5.0.0 and < 6.0.0"
wisp = ">= 2.0.0 and < 3.0.0"
lustre = ">= 5.0.0 and < 6.0.0"
```

Then run:

```sh
gleam deps download
```

Gleam will fetch Lustre and all of its transitive dependencies. You will see them appear in the `build/packages` directory.

### Step 2 -- Build a Layout Function

A **layout** is the outer shell that wraps every page: the `<html>`, `<head>`, and `<body>` tags, the stylesheet link, the HTMX script tag, and so on. We write it once and reuse it everywhere.

Create (or replace the contents of) `src/teamwork.gleam`. Start with the imports:

```gleam
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist
```

Now add the layout function:

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

Let us walk through what each piece does.

**`html.html([], [...])`** -- Creates the root `<html>` element. The first argument is an empty attribute list (we do not need `lang` yet, but you could add `attribute("lang", "en")` if you like). The second argument is the list of children: `<head>` and `<body>`.

**`html.meta([attribute("charset", "UTF-8")])`** -- A void element. It takes only a list of attributes; there is no children argument because `<meta>` tags never have children in HTML.

**`html.title([], page_title)`** -- Sets the document title that appears in the browser tab. Notice that the second argument is a plain `String`, not a list of child elements. This is a special case in Lustre's API because the `<title>` element only ever contains text.

**`html.link([...])`** -- Another void element. We point it at `/static/css/style.css`. The `/static` prefix is important: Wisp will intercept any request that starts with `/static` and try to serve a matching file from disk.

**`html.script([attribute("src", "...")], "")`** -- Loads HTMX from the unpkg CDN. The second argument is the inline script body, which we leave as an empty string because all the code lives in the external file. We are loading HTMX now so that it is available the moment we need it in Chapter 4.

**`html.body([], [content])`** -- Wraps whatever page-specific content we pass in.

### Step 3 -- Create a Home Page

The home page is simple for now. We will grow it into a full task board over the coming chapters.

```gleam
fn home_page() -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    html.p([], [
      element.text("A collaborative task board — coming soon!"),
    ]),
  ])
}
```

Two small details worth noticing:

- **`attribute.class("container")`** is a convenience function. You could also write `attribute("class", "container")`, but `attribute.class` reads more naturally and is checked by the compiler.
- **`element.text("...")`** creates a text node. You cannot place a bare `String` inside a children list; it must be wrapped in `element.text`. This prevents accidental mixing of types.

### Step 4 -- Update `handle_request`

Replace the handler from Chapter 1 with:

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  let page = layout("Teamwork", home_page())
  let html_string = element.to_document_string(page)

  wisp.html_response(html_string, 200)
}
```

Line by line:

1. **`use <- wisp.log_request(req)`** -- Logs every incoming request to the console. This is middleware: it runs some code, then calls the rest of our handler.

2. **`use <- wisp.serve_static(req, under: "/static", from: static_directory())`** -- Another middleware. If the request path starts with `/static`, Wisp looks for a matching file in the directory returned by `static_directory()` and sends it back. If no file matches, or if the path does not start with `/static`, execution continues to the next line.

3. **`let page = layout("Teamwork", home_page())`** -- Builds the Lustre element tree.

4. **`let html_string = element.to_document_string(page)`** -- Serialises the tree into a `String` beginning with `<!doctype html>`.

5. **`wisp.html_response(html_string, 200)`** -- Creates a `wisp.Response` with status `200`, the `Content-Type` header set to `text/html`, and our HTML as the body. `html_response` takes a `String` directly, so we pass `html_string` as-is.

### Step 5 -- The Static Directory Helper

Wisp's convention is to store static assets inside a `priv/static` directory. The `priv` directory is special in the BEAM ecosystem: it is included alongside your compiled code when you build a release, so your assets travel with your application.

```gleam
fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}
```

`wisp.priv_directory("teamwork")` returns the absolute path to the `priv` folder of the `teamwork` application. We append `"/static"` because we want to serve only the contents of that subdirectory, not the entire `priv` folder.

The `let assert Ok(priv)` will crash at startup if the directory does not exist. That is intentional: there is no reasonable way to continue without static assets, so failing loudly is the right choice.

### Step 6 -- Write the CSS

Create the directory structure and file:

```
priv/
  static/
    css/
      style.css
```

On your terminal:

```sh
mkdir -p priv/static/css
```

Then write `priv/static/css/style.css`:

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

p {
  color: #333;
  margin-bottom: 0.75rem;
}
```

Nothing fancy -- just clean defaults that make the page readable. We will extend this stylesheet as the project grows.

### Step 7 -- Run It

```sh
gleam run
```

Open your browser to `http://localhost:8000`. You should see:

- A styled heading that reads **Teamwork**.
- A paragraph that reads "A collaborative task board -- coming soon!".
- The browser tab should say "Teamwork".

If you view the page source (`Ctrl+U` or `Cmd+U`), you will see clean, properly nested HTML starting with `<!doctype html>`. That is `element.to_document_string` at work.

---

## Full Code Listing

Here is every file you should have after completing this chapter.

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
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
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
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  let page = layout("Teamwork", home_page())
  let html_string = element.to_document_string(page)

  wisp.html_response(html_string, 200)
}

fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}

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

fn home_page() -> Element(t) {
  html.div([attribute.class("container")], [
    html.h1([], [element.text("Teamwork")]),
    html.p([], [
      element.text("A collaborative task board — coming soon!"),
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

p {
  color: #333;
  margin-bottom: 0.75rem;
}
```

---

## How It All Fits Together

Here is the journey of a single request, from browser to response:

```
Browser                         Server
  |                                |
  |  GET /  HTTP/1.1               |
  |------------------------------->|
  |                                |  wisp.log_request  -- logs "GET /"
  |                                |  wisp.serve_static -- path "/" does not
  |                                |                       start with "/static",
  |                                |                       so skip
  |                                |  layout("Teamwork", home_page())
  |                                |    -> builds Element tree
  |                                |  element.to_document_string(tree)
  |                                |    -> "<!doctype html><html>..."
  |                                |  wisp.html_response(body, 200)
  |                                |    -> Response {
  |                                |         status: 200,
  |                                |         headers: [
  |                                |           ("content-type", "text/html")
  |                                |         ],
  |                                |         body: "<!doctype html>..."
  |                                |       }
  |  200 OK                        |
  |  content-type: text/html       |
  |  <!doctype html><html>...      |
  |<-------------------------------|
  |                                |
  |  GET /static/css/style.css     |
  |------------------------------->|
  |                                |  wisp.log_request  -- logs the request
  |                                |  wisp.serve_static -- path starts with
  |                                |                       "/static", found file
  |                                |    -> serves priv/static/css/style.css
  |  200 OK                        |
  |  content-type: text/css        |
  |  * { margin: 0; ...            |
  |<-------------------------------|
  |                                |
  |  GET https://unpkg.com/...     |  (handled by unpkg CDN, not our server)
  |                                |
  Browser renders the page.
```

Notice the two separate requests to our server. The browser parses the HTML, finds the `<link>` tag, and fires a second request for the stylesheet. Wisp's `serve_static` middleware catches that second request and serves the file directly.

The HTMX script is loaded from an external CDN, so that request never hits our server at all.

---

## Exercise

Practice what you have learned by making the following changes. Work through them in order.

### Task 1 -- Add a Navigation Bar

Add a `<nav>` element at the very top of the `<body>`, before the page content. It should contain two links:

- A link to `"/"` with the text "Home".
- A link to `"/about"` with the text "About".

These links will not lead anywhere useful yet (clicking "About" will show the same home page because we have not added routing). We will fix that in Chapter 3.

**Hint:** Modify the `layout` function so that `html.body` contains the nav first and then the content.

### Task 2 -- Add a Footer

Add a `<footer>` element after the content inside `html.body`. The footer should contain a paragraph that reads "Built with Gleam and HTMX".

### Task 3 -- Style the New Elements

Open `priv/static/css/style.css` and add rules for `nav`, `nav a`, and `footer`. Here are some ideas:

- Give the nav a bottom border and some padding.
- Style the links with a colour that matches the heading, and remove the default underline.
- Push the footer to the bottom with a top margin, and give it a lighter text colour.

### Task 4 -- Inspect the Response Headers

Open your browser's developer tools (press `F12`). Go to the **Network** tab, refresh the page, and click on the initial document request (the one for `/`).

1. Find the `Content-Type` response header. Confirm it says `text/html; charset=utf-8` (Wisp adds the charset automatically).
2. Click on the request for `style.css`. Its `Content-Type` should be `text/css`.

### Task 5 -- View the Generated Source

Right-click the page and choose **View Page Source** (or press `Ctrl+U`). Read through the HTML that Lustre generated. Notice that:

- It starts with `<!doctype html>`.
- All tags are properly opened and closed.
- Attribute values are correctly quoted.

Compare this with what you would get from manual string concatenation and appreciate the work Lustre is doing for you.

---

## Exercise Solution Hints

If you get stuck, here are some pointers. Try to solve each task on your own before reading these.

### Hint for Task 1

```gleam
fn layout(page_title: String, content: Element(t)) -> Element(t) {
  html.html([], [
    html.head([], [
      // ... same as before ...
    ]),
    html.body([], [
      html.nav([attribute.class("nav")], [
        html.a([attribute.href("/")], [element.text("Home")]),
        html.a([attribute.href("/about")], [element.text("About")]),
      ]),
      content,
      // ... footer goes here (Task 2) ...
    ]),
  ])
}
```

Key points:

- `html.nav` takes attributes and children, just like `html.div`.
- `attribute.href("/")` is a convenience function, similar to `attribute.class`.
- The nav element goes before `content` in the children list of `html.body`.

### Hint for Task 2

Inside the children list of `html.body`, after `content`:

```gleam
html.footer([attribute.class("footer")], [
  html.p([], [element.text("Built with Gleam and HTMX")]),
]),
```

### Hint for Task 3

Some CSS to get you started:

```css
nav {
  display: flex;
  gap: 1.5rem;
  padding-bottom: 1rem;
  margin-bottom: 2rem;
  border-bottom: 2px solid #e0e0e8;
}

nav a {
  color: #16213e;
  text-decoration: none;
  font-weight: 600;
}

nav a:hover {
  text-decoration: underline;
}

footer {
  margin-top: 3rem;
  padding-top: 1rem;
  border-top: 1px solid #e0e0e8;
  color: #888;
  font-size: 0.875rem;
}
```

### Hint for Tasks 4 and 5

These are observation tasks -- there is no code to write. Just open your browser's developer tools and explore. Getting comfortable with the Network tab and View Source will be invaluable throughout this course.

---

## Key Takeaways

1. **String concatenation for HTML is a trap.** It is easy to start with, but it scales poorly and introduces bugs that the compiler cannot catch. Always prefer a structured approach.

2. **Lustre gives you type-safe HTML.** Every element function enforces the correct shape: attributes first, then children. If you make a structural mistake, the Gleam compiler tells you before you ever open a browser.

3. **`element.to_document_string` produces a full HTML document.** It prepends `<!doctype html>` so the browser enters standards mode. Use `element.to_string` (without "document") only when you need an HTML fragment.

4. **`wisp.serve_static` handles static files.** Point it at a directory prefix (`"/static"`) and a filesystem path (your `priv/static` folder), and it takes care of reading files, setting the correct `Content-Type`, and sending the response.

5. **Content types matter.** The `Content-Type` header tells the browser whether it is looking at HTML, plain text, JSON, CSS, or something else. Wisp sets the correct type for you when you use helpers like `html_response`, and it infers the type from the file extension when serving static files.

6. **HTMX is loaded and waiting.** We included the HTMX script in our layout. It is not doing anything yet, but starting in Chapter 4 we will add `hx-*` attributes to our elements and watch the page come alive without writing any client-side JavaScript ourselves.

---

**Next up:** In Chapter 3 we will add routing so that different URLs show different pages, and we will give the "About" link somewhere to go.
