# Chapter 19 -- Error Handling and Graceful Degradation

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 18](../03-advanced/18-flatpickr-and-third-party-js.md) integrated Flatpickr as a third-party JavaScript library,
completing the Advanced phase. This chapter begins the Real-World Patterns section,
where we shift focus from building features to making them resilient. Everything
breaks eventually -- networks fail, servers crash, users disable JavaScript. The
question is not whether errors will happen, but how gracefully your application
handles them when they do.

---

## Learning Objectives

By the end of this chapter you will be able to:

- Describe the four error categories (network, server, validation, client) and the response strategy for each.
- Use HTMX error events (`htmx:sendError`, `htmx:responseError`, `htmx:timeout`) with `_hyperscript` to display toast notifications.
- Build error-handling middleware in Gleam that returns styled 500 fragments for HTMX requests and full error pages for regular requests.
- Configure request timeouts with `hx-request='{"timeout": 5000}'` and handle the timeout event.
- Structure forms for progressive enhancement: `method`/`action` for no-JS, `hx-post`/`hx-target` for HTMX.
- Implement a retry pattern using `_hyperscript` to re-trigger failed requests.

---

## 1. Theory

### 1.1 The Four Error Categories

Not all errors are alike. When something goes wrong in an HTMX application, the
failure falls into one of four categories. Each has a different cause, a different
user experience, and a different recovery strategy.

```
                         ┌────────────────────────────────┐
                         │        User clicks button      │
                         └───────────────┬────────────────┘
                                         │
                                         ▼
                         ┌────────────────────────────────┐
                   ┌─ No │  Does the request reach the    │ Yes ─┐
                   │     │  server?                        │      │
                   │     └────────────────────────────────┘      │
                   │                                              │
                   ▼                                              ▼
          ┌────────────────┐                        ┌────────────────────────┐
          │  NETWORK ERROR │                        │  Does the server crash │
          │  (Category 1)  │                  ┌─ Yes│  internally?           │─ No ─┐
          └────────────────┘                  │     └────────────────────────┘      │
                                              │                                     │
                                              ▼                                     ▼
                                   ┌──────────────────┐             ┌─────────────────────────┐
                                   │  SERVER ERROR     │             │  Is the user input valid? │
                                   │  (Category 2)     │       ┌─ No│                           │─ Yes ─┐
                                   └──────────────────┘       │     └─────────────────────────┘        │
                                                               │                                        │
                                                               ▼                                        ▼
                                                    ┌───────────────────┐                    ┌──────────────┐
                                                    │  VALIDATION ERROR │                    │  SUCCESS     │
                                                    │  (Category 3)     │                    │  (200 / 201) │
                                                    └───────────────────┘                    └──────────────┘
```

Here is the full breakdown:

| Category           | HTTP Status        | Cause                                     | HTMX Event             | User Sees                     | Recovery Strategy            |
|--------------------|--------------------|-------------------------------------------|------------------------|-------------------------------|------------------------------|
| **Network error**  | None (no response) | Wi-Fi drops, DNS fails, server unreachable | `htmx:sendError`       | Toast: "Network error"        | Retry button                 |
| **Server error**   | 500, 502, 503      | Unhandled exception, database crash        | `htmx:responseError`   | Toast or inline error banner  | Retry after delay            |
| **Validation error**| 422               | Bad user input                             | (Normal swap)          | Inline form errors            | User corrects and resubmits  |
| **Client error**   | 400, 403, 404, 409 | Bad request, forbidden, not found          | `htmx:responseError`   | Context-specific message      | Depends on the cause         |
| **Timeout**        | None (aborted)     | Server too slow, network congestion        | `htmx:timeout`         | Toast: "Request timed out"    | Retry button                 |

The first thing to notice: validation errors are not really "errors" in the
operational sense. They are expected outcomes of user input. We already handle them
beautifully with 422 responses and inline form re-rendering ([Chapter 8](../02-intermediate/08-validation-and-error-feedback.md)). The
`response-targets` extension ([Chapter 16](../03-advanced/16-extensions-and-patterns.md)) lets us route 422 responses to a different
target than 200 responses. Validation errors are a solved problem.

The unsolved problems are the other three categories. Network errors produce no HTTP
response at all. Server errors produce a response but with an unhelpful status code.
Timeouts produce no response because the request was aborted. For all three, the
default HTMX behaviour is to do nothing -- the user clicks a button, waits, and
nothing happens. That is the worst possible user experience.

This chapter fixes that.

### 1.2 HTMX Error Events

HTMX fires events at every stage of the request lifecycle. For error handling, four
events matter:

| Event                  | When it fires                                              | `event.detail` contains                  |
|------------------------|------------------------------------------------------------|------------------------------------------|
| `htmx:sendError`       | The XMLHttpRequest failed to send (network down)          | `{ elt, xhr }`                           |
| `htmx:responseError`   | The server returned a non-2xx status code                 | `{ elt, xhr, response }`                 |
| `htmx:timeout`         | The request exceeded the configured timeout               | `{ elt, xhr }`                           |
| `htmx:afterRequest`    | Every request completes (success or failure)              | `{ elt, xhr, successful, failed }`       |

These events bubble up the DOM tree. A `htmx:sendError` event fired on a button
inside a form inside a div will bubble up through the form, the div, the body, and
finally the document. This means you can listen for errors at any level:

- **On the element:** Handle errors for a specific button or form.
- **On a container:** Handle errors for all requests within a section.
- **On the `<body>`:** Handle errors globally for the entire page.

Global error handling on the `<body>` is the most practical approach. You add one
listener and it catches every error from every HTMX request on the page. Individual
elements can still override the global handler by stopping event propagation.

Here is the event lifecycle for a failed request:

```
Request initiated
    │
    ├── Network fails before response ──► htmx:sendError ──► htmx:afterRequest
    │
    ├── Timeout exceeded ──► htmx:timeout ──► htmx:afterRequest
    │
    ├── Server returns 500 ──► htmx:responseError ──► htmx:afterRequest
    │
    └── Server returns 200 ──► (normal swap) ──► htmx:afterRequest
```

Notice that `htmx:afterRequest` fires in every case. Its `detail` object includes
`successful` (boolean) and `failed` (boolean), so you can use it as a single
catch-all. But for fine-grained handling, the specific events are better because
they tell you exactly what went wrong.

### 1.3 The Error Toast Pattern

A "toast" is a small notification that appears temporarily, usually in a corner of
the screen, and then disappears. It is the standard UI pattern for non-blocking error
messages -- errors that the user should see but that do not require immediate action.

The pattern has three parts:

1. **A fixed-position container** in the layout that holds toast messages.
2. **`_hyperscript` listeners** on the `<body>` that catch HTMX error events and
   create toast elements.
3. **CSS animations** that slide the toast in, hold it for a few seconds, and slide
   it out.

```
  ┌───────────────────────────────────────────────────┐
  │  Teamwork -- Task Board                           │
  │                                                   │
  │  ┌───────────────────────────────────────┐        │
  │  │  Task form                            │        │
  │  │  [title input] [Add Task]             │        │
  │  └───────────────────────────────────────┘        │
  │                                                   │
  │  ┌───────────────────────────────────────┐        │
  │  │  Task 1                               │        │
  │  │  Task 2                               │  ┌─────┤
  │  │  Task 3                               │  │Toast│
  │  │  Task 4                               │  │msg  │
  │  └───────────────────────────────────────┘  └─────┤
  │                                                   │
  └───────────────────────────────────────────────────┘
```

Why a toast and not an inline error? Because network errors and server errors are
not tied to a specific form or element. They are infrastructure problems. An inline
error next to a button would look strange when the problem is that the Wi-Fi is down.
A toast says "something went wrong with the system" without pointing fingers at a
specific form field.

Validation errors (422) are different -- they should remain inline, next to the
field that has the problem. We handled that in Chapters 8 and 16. The toast pattern
is only for infrastructure errors.

### 1.4 Server-Side Error Middleware in Gleam

When your server code panics (in Erlang/OTP terms, when a process crashes), Wisp
catches the crash and returns a generic 500 response. But that generic response is
not helpful to the user. We want two different responses depending on who is asking:

| Request type     | How to detect                           | Response                                              |
|------------------|-----------------------------------------|-------------------------------------------------------|
| **HTMX request** | `HX-Request: true` header present       | HTML fragment for the toast or error banner            |
| **Regular request** | No `HX-Request` header                | Full error page with navigation, "go back" link, etc. |

This distinction is critical. If an HTMX request gets back a full HTML page, HTMX
will swap the entire page into the target element, creating a page-inside-a-page
mess. If a regular browser navigation gets back a bare HTML fragment, the user sees
raw markup with no layout.

The middleware pattern in Gleam looks like this:

```
Request arrives
    │
    ▼
┌────────────────────────┐
│  Error middleware       │
│  (wraps the handler)   │
│                        │
│  1. Call the handler   │
│  2. If it panics:      │
│     - Check HX-Request │
│     - Return fragment  │
│       or full page     │
└────────────────────────┘
    │
    ▼
┌────────────────────────┐
│  Route handler         │
│  (may crash)           │
└────────────────────────┘
```

Wisp provides `wisp.rescue_crashes` for wrapping handlers that might crash. We
combine that with an HX-Request check to produce the right response format.

### 1.5 Timeouts and Retries

By default, HTMX does not set a timeout on requests. An XMLHttpRequest will wait
indefinitely for a response. If your server is stuck processing a query that will
never finish, the user's button just shows the loading indicator forever.

Setting a timeout tells the browser: "If you have not received a response within
this many milliseconds, abort the request and fire `htmx:timeout`."

You configure timeouts with the `hx-request` attribute, which accepts a JSON object:

```html
<body hx-request='{"timeout": 5000}'>
```

This sets a 5-second timeout for all HTMX requests inside the `<body>`. Individual
elements can override it:

```html
<form hx-post="/upload" hx-request='{"timeout": 30000}'>
  <!-- File upload gets 30 seconds -->
</form>
```

The `hx-request` attribute also supports other configuration:

| Property       | Type    | Default | Description                                         |
|----------------|---------|---------|-----------------------------------------------------|
| `timeout`      | Number  | 0 (none)| Milliseconds before the request is aborted          |
| `credentials`  | Boolean | `true`  | Whether to send cookies with cross-origin requests   |
| `noHeaders`    | Boolean | `false` | If `true`, HTMX does not add its standard headers    |

For retries, HTMX does not have a built-in retry mechanism. You have two options:

1. **Manual retry with `_hyperscript`.** After a failure, show a "Retry" button
   that re-triggers the original request.
2. **Automatic retry with `_hyperscript`.** After a failure, wait a few seconds
   and re-trigger the request automatically.

Manual retry is almost always better. Automatic retry can cause a cascade of
requests that overwhelm an already struggling server. When the server returns a
500, it is usually because something is broken -- retrying immediately will just
produce another 500. A manual retry gives the server time to recover and gives the
user control.

### 1.6 Progressive Enhancement

Progressive enhancement is the practice of building a baseline experience that
works without JavaScript, then layering enhancements on top for users who have it.

For HTMX applications, progressive enhancement means:

1. **Every form works as a standard HTML form.** It has `method="post"` and
   `action="/tasks"` so that submitting it without JavaScript performs a full page
   navigation and the server processes the request normally.
2. **HTMX attributes add the enhanced experience.** `hx-post="/tasks"` and
   `hx-target="#task-list"` make the same form submit asynchronously and swap the
   result into the page without a full reload.

When both sets of attributes are present, HTMX intercepts the form submission and
handles it. When JavaScript is disabled (or fails to load, or HTMX has not loaded
yet), the browser falls back to the standard `method`/`action` behaviour.

Here is a comparison:

| Attribute             | Purpose                          | Used by       |
|-----------------------|----------------------------------|---------------|
| `method="post"`       | Standard HTML form method        | Browser       |
| `action="/tasks"`     | Standard HTML form action        | Browser       |
| `hx-post="/tasks"`    | HTMX-enhanced POST               | HTMX          |
| `hx-target="#list"`   | Where to swap the response       | HTMX          |
| `hx-swap="beforeend"` | How to swap                      | HTMX          |

The form has **dual attributes**: standard HTML attributes for the no-JS fallback,
and HTMX attributes for the enhanced experience. Both point to the same server
endpoint.

The server needs to handle both cases too. When an HTMX request comes in (the
`HX-Request: true` header is present), the server returns an HTML fragment. When
a standard form submission comes in (no `HX-Request` header), the server processes
the data, then redirects to the full page (the standard Post/Redirect/Get pattern).

```
                  ┌──────────────────────────────────────┐
                  │        Form submitted                 │
                  └──────────────────┬───────────────────┘
                                     │
                          ┌──────────┴──────────┐
                          │  HX-Request header?  │
                          └──────────┬──────────┘
                                     │
                         ┌───────────┴───────────┐
                     Yes │                       │ No
                         ▼                       ▼
              ┌──────────────────┐    ┌────────────────────┐
              │ Return fragment  │    │ Process + redirect  │
              │ (200 or 422)    │    │ (303 See Other)     │
              └──────────────────┘    └────────────────────┘
```

### 1.7 Graceful Degradation vs Progressive Enhancement

These two terms are often used interchangeably, but they describe opposite
approaches:

| Concept                      | Starting point                   | Direction       |
|------------------------------|----------------------------------|-----------------|
| **Progressive enhancement**  | Start with basic HTML            | Add JS features |
| **Graceful degradation**     | Start with full JS experience    | Handle JS failure |

**Progressive enhancement** says: "Build the floor first, then add the ceiling."
You start with a working HTML form, then add HTMX for a smoother experience.

**Graceful degradation** says: "Build the whole house, but make sure the floor
holds if the ceiling collapses." You build the full HTMX experience, then make
sure the application does not break catastrophically when something fails.

In practice, you do both:

- **Progressive enhancement for forms.** Dual attributes (`method`/`action` +
  `hx-post`/`hx-target`) so forms work without JavaScript.
- **Graceful degradation for everything else.** Error toasts, retry buttons,
  timeout handling, and fallback messages for when HTMX requests fail.

Here is a checklist for evaluating your application:

```
Progressive Enhancement Checklist
─────────────────────────────────
[ ] Forms have method and action attributes
[ ] Links use <a href="..."> with real URLs
[ ] Server returns full pages for non-HTMX requests
[ ] Server returns fragments for HTMX requests
[ ] Critical actions work without JavaScript

Graceful Degradation Checklist
──────────────────────────────
[ ] Network errors show a toast notification
[ ] Server errors show a toast notification
[ ] Timeouts show a toast with retry option
[ ] Failed requests do not leave the UI in a broken state
[ ] Loading indicators disappear after errors (not just success)
[ ] Retry buttons are available for failed requests
[ ] Error messages are human-readable, not stack traces
```

---

## 2. Code Walkthrough

We are going to add five things to the Teamwork application:

1. A global error toast component with `_hyperscript` event listeners.
2. A request timeout configuration on the `<body>`.
3. Server-side error middleware that is HTMX-aware.
4. Progressive enhancement for the task creation form.
5. A retry button pattern for failed requests.
6. Manual testing of each error path.

### Step 1 -- Build the Error Toast Component

The toast component is a container div that lives at the bottom-right of the page.
It starts empty. When an HTMX error event fires, `_hyperscript` creates a toast
element inside it, shows it, waits, and removes it.

Add the toast container and its event listeners to the layout. The `_hyperscript`
code goes on the `<body>` element because HTMX error events bubble up to the body.

```gleam
// In src/teamwork/web.gleam -- update the layout function

fn layout(content: element.Element(Nil)) -> wisp.Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork"),
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute.attribute("content", "width=device-width, initial-scale=1"),
        ]),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
        // HTMX
        html.script(
          [attribute.src("/static/js/htmx.min.js")],
          "",
        ),
        // response-targets extension
        html.script(
          [attribute.src(
            "https://unpkg.com/htmx-ext-response-targets@2.0.2/response-targets.js",
          )],
          "",
        ),
        // _hyperscript
        html.script(
          [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
          "",
        ),
      ]),
      html.body(
        [
          attribute.attribute("hx-ext", "response-targets"),
          // _hyperscript listeners for HTMX error events
          attribute.attribute(
            "_",
            "on htmx:sendError "
            <> "call makeToast('Network error. Check your connection and try again.', 'error') "
            <> "end "
            <> "on htmx:responseError "
            <> "if event.detail.xhr.status >= 500 "
            <> "call makeToast('Server error. Please try again in a moment.', 'error') "
            <> "end "
            <> "on htmx:timeout "
            <> "call makeToast('Request timed out. The server may be busy.', 'warning') "
            <> "end",
          ),
          // Global timeout: 10 seconds for all HTMX requests
          attribute.attribute(
            "hx-request",
            "{\"timeout\": 10000}",
          ),
        ],
        [
          // Error toast container (fixed position, bottom-right)
          html.div(
            [
              attribute.id("toast-container"),
              attribute.class("toast-container"),
            ],
            [],
          ),
          // Main content
          html.div([attribute.class("container")], [content]),
        ],
      ),
    ])

  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

That `_hyperscript` string on the body is doing a lot. Let us break it down piece
by piece.

**Listener 1: `on htmx:sendError`**

```
on htmx:sendError
  call makeToast('Network error. Check your connection and try again.', 'error')
end
```

This fires when the browser cannot send the request at all -- the network is down,
the DNS lookup failed, or the server is unreachable. There is no HTTP response to
examine, so we show a generic network error message.

**Listener 2: `on htmx:responseError`**

```
on htmx:responseError
  if event.detail.xhr.status >= 500
    call makeToast('Server error. Please try again in a moment.', 'error')
  end
```

This fires when the server responds with a non-2xx status code. We check for 5xx
specifically because 4xx errors (like 422 validation errors) are already handled by
the `response-targets` extension. We do not want to show a toast for a validation
error -- that would be confusing.

**Listener 3: `on htmx:timeout`**

```
on htmx:timeout
  call makeToast('Request timed out. The server may be busy.', 'warning')
end
```

This fires when the request exceeds the timeout configured in `hx-request`. We use
the `'warning'` type instead of `'error'` because a timeout is not necessarily a
failure -- the server might still be processing the request. The user should retry,
not panic.

Now we need the `makeToast` JavaScript function. Add it as an inline script in the
layout's `<head>`:

```gleam
// Add this script element inside the <head>, after the _hyperscript script tag

html.script([], "
  function makeToast(message, type) {
    var container = document.getElementById('toast-container');
    if (!container) return;

    var toast = document.createElement('div');
    toast.className = 'toast toast-' + type;
    toast.textContent = message;

    // Add a dismiss button
    var dismiss = document.createElement('button');
    dismiss.className = 'toast-dismiss';
    dismiss.textContent = '\\u00D7';
    dismiss.onclick = function() {
      toast.classList.add('toast-exit');
      setTimeout(function() { toast.remove(); }, 300);
    };
    toast.appendChild(dismiss);

    container.appendChild(toast);

    // Auto-remove after 8 seconds
    setTimeout(function() {
      if (toast.parentNode) {
        toast.classList.add('toast-exit');
        setTimeout(function() { toast.remove(); }, 300);
      }
    }, 8000);
  }
"),
```

The `makeToast` function does four things:

1. Creates a div with the message and a CSS class based on the error type.
2. Adds a dismiss button (the multiplication sign character, which looks like an X).
3. Appends the toast to the container.
4. Sets a timer to auto-remove the toast after 8 seconds.

The 300ms delay before `toast.remove()` gives the exit animation time to play. We
add the `toast-exit` class first (which triggers a CSS transition), wait 300ms, then
remove the element from the DOM.

Why a plain JavaScript function instead of pure `_hyperscript`? Because creating DOM
elements with `_hyperscript` requires several lines of `make` and `put` commands, and
the result is harder to read than the equivalent JavaScript. The `_hyperscript`
listeners handle the event-to-function bridge. The JavaScript function handles DOM
creation. Each tool does what it is best at.

### Step 2 -- Add Request Timeout Configuration

We already added the timeout in Step 1 with this attribute on the `<body>`:

```gleam
attribute.attribute(
  "hx-request",
  "{\"timeout\": 10000}",
),
```

This sets a 10-second timeout for every HTMX request in the application. Let us
discuss why 10 seconds is a reasonable default.

| Timeout value | Trade-off                                                   |
|---------------|-------------------------------------------------------------|
| 1-2 seconds   | Too aggressive. Slow mobile connections will timeout often. |
| 5 seconds     | Good for fast operations. May timeout on slow queries.      |
| 10 seconds    | Balanced default. Most operations finish well within this.  |
| 30 seconds    | Only for known-slow operations (file upload, report gen).   |
| 0 (none)      | The HTMX default. Request waits forever. Not recommended.   |

For specific operations that you know are slow, override the timeout on the element:

```gleam
// A file upload form that needs more time
html.form(
  [
    hx.post("/upload"),
    hx.target(hx.Selector("#upload-result")),
    attribute.attribute(
      "hx-request",
      "{\"timeout\": 30000}",
    ),
  ],
  [
    // ... file input ...
  ],
)
```

The element-level `hx-request` attribute merges with the inherited one. Only the
properties you specify are overridden -- other properties keep their inherited values.

### Step 3 -- Server-Side Error Middleware

Now let us build the Gleam middleware that catches server crashes and returns the
right response format.

First, a helper function to check whether a request came from HTMX:

```gleam
// In src/teamwork/middleware.gleam

import gleam/int
import gleam/list
import lustre/attribute
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}

/// Returns True if the request was made by HTMX (has the HX-Request header).
pub fn is_htmx_request(req: Request) -> Bool {
  case list.key_find(req.headers, "hx-request") {
    Ok("true") -> True
    _ -> False
  }
}
```

Now the error middleware. Wisp's `wisp.rescue_crashes` wraps a handler and returns
a 500 response if the handler crashes. We build on top of that to customize the 500
response:

```gleam
/// Middleware that catches crashes and returns an appropriate error response.
/// For HTMX requests: returns an HTML fragment suitable for swapping.
/// For regular requests: returns a full error page with layout.
pub fn handle_errors(
  req: Request,
  handler: fn() -> Response,
) -> Response {
  let response = wisp.rescue_crashes(handler)

  case response.status {
    500 -> error_response(req, 500, "Something went wrong on our end.")
    503 -> error_response(req, 503, "The service is temporarily unavailable.")
    _ -> response
  }
}

fn error_response(req: Request, status: Int, message: String) -> Response {
  case is_htmx_request(req) {
    True -> htmx_error_fragment(status, message)
    False -> full_error_page(status, message)
  }
}
```

The HTMX error fragment is a small HTML snippet. The `response-targets` extension
or the toast system will handle displaying it:

```gleam
fn htmx_error_fragment(status: Int, message: String) -> Response {
  let fragment =
    html.div([attribute.class("error-fragment")], [
      html.strong([], [element.text("Error " <> int.to_string(status))]),
      element.text(" -- "),
      element.text(message),
    ])

  wisp.html_response(element.to_string(fragment), status)
}
```

The full error page is for non-HTMX requests -- when the user navigates directly
to a broken URL or submits a form with JavaScript disabled:

```gleam
fn full_error_page(status: Int, message: String) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Error -- Teamwork"),
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body([], [
        html.div([attribute.class("container")], [
          html.div([attribute.class("error-page")], [
            html.h1([], [
              element.text("Error " <> int.to_string(status)),
            ]),
            html.p([attribute.class("error-message")], [
              element.text(message),
            ]),
            html.p([], [
              element.text("You can "),
              html.a([attribute.href("/")], [element.text("go back to the board")]),
              element.text(" or try again in a moment."),
            ]),
          ]),
        ]),
      ]),
    ])

  wisp.html_response(element.to_document_string(page), status)
}
```

Wire the middleware into the router. In Wisp, middleware is applied by wrapping the
handler in a `use` expression:

```gleam
// In src/teamwork/router.gleam

import teamwork/middleware

pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- middleware.handle_errors(req)
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["boards", board_id, "tasks"] -> handle_tasks(req, ctx, board_id)
    // ... other routes ...
    _ -> wisp.not_found()
  }
}
```

The order matters. `handle_errors` wraps everything, including static file serving.
If any handler inside the `use` chain crashes, `handle_errors` catches it and returns
the appropriate response.

### Step 4 -- Progressive Enhancement for the Task Form

Let us update the task creation form to work with and without JavaScript. The form
needs dual attributes: standard HTML attributes for the no-JS fallback, and HTMX
attributes for the enhanced experience.

```gleam
fn task_form(board_id: String) -> element.Element(Nil) {
  html.div([attribute.id("form-container")], [
    html.form(
      [
        attribute.id("task-form"),
        // Standard HTML attributes (no-JS fallback)
        attribute.method("post"),
        attribute.action("/boards/" <> board_id <> "/tasks"),
        // HTMX attributes (enhanced experience)
        hx.post("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        // Route validation errors to the form container
        attribute.attribute("hx-target-422", "#form-container"),
        attribute.attribute("hx-swap-422", "outerHTML"),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label([attribute.for("title")], [element.text("Title")]),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.required(True),
            attribute.placeholder("What needs to be done?"),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}
```

Both `method="post"` / `action="/boards/abc/tasks"` and `hx-post="/boards/abc/tasks"`
point to the same endpoint. When HTMX is available, it intercepts the form submission
and handles it with an AJAX request. When HTMX is not available, the browser submits
the form normally.

The server handler needs to detect which case it is and respond accordingly:

```gleam
fn create_task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      let task_id = wisp.random_string(16)
      let task =
        db.Task(
          id: task_id,
          title: valid_title,
          description: "",
          done: False,
          due_date: option.None,
        )
      let assert Ok(_) = db.create_task(ctx.db, board_id, task)

      case middleware.is_htmx_request(req) {
        True -> {
          // HTMX request: return just the task item fragment
          let today = date_helpers.today()
          wisp.html_response(
            element.to_string(task_item(task, today)),
            201,
          )
          |> wisp.set_header("HX-Trigger", "taskAdded")
        }
        False -> {
          // Standard form submission: redirect to the board page (PRG pattern)
          wisp.redirect("/boards/" <> board_id)
        }
      }
    }

    Error(errors) -> {
      case middleware.is_htmx_request(req) {
        True -> {
          // HTMX request: return re-rendered form with errors (422)
          let form_html =
            task_form_with_errors(board_id, errors, form_data.values)
          wisp.html_response(element.to_string(form_html), 422)
        }
        False -> {
          // Standard form submission: re-render full page with errors
          let content =
            board_page_content(board_id, ctx, errors, form_data.values)
          full_page_response(content, 422)
        }
      }
    }
  }
}
```

There are four paths through this function:

| HTMX? | Valid? | Action                                     | Status |
|--------|--------|--------------------------------------------|--------|
| Yes    | Yes    | Return task item fragment                  | 201    |
| Yes    | No     | Return form with inline errors             | 422    |
| No     | Yes    | Redirect to board page (PRG)               | 303    |
| No     | No     | Re-render full page with errors            | 422    |

The PRG (Post/Redirect/Get) pattern in the no-JS success path prevents the
"resubmit form?" dialog that browsers show when you refresh a page after a POST.
The `wisp.redirect` function returns a 303 See Other response, which tells the
browser to issue a GET to the redirect URL.

The form-with-errors helper renders the form container with validation messages:

```gleam
fn task_form_with_errors(
  board_id: String,
  errors: List(validation.ValidationError),
  values: List(#(String, String)),
) -> element.Element(Nil) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.div(
      [attribute.id("form-errors"), attribute.class("error-list")],
      list.map(title_errors, fn(e) {
        html.span(
          [attribute.class("error-text")],
          [element.text(validation.error_message(e))],
        )
      }),
    ),
    html.form(
      [
        attribute.id("task-form"),
        attribute.method("post"),
        attribute.action("/boards/" <> board_id <> "/tasks"),
        hx.post("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        attribute.attribute("hx-target-422", "#form-container"),
        attribute.attribute("hx-swap-422", "outerHTML"),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label([attribute.for("title")], [element.text("Title")]),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.required(True),
            attribute.value(title_value),
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}
```

### Step 5 -- Retry Button Pattern

When an HTMX request fails, the user needs a way to try again. We will build a
retry pattern using `_hyperscript` that works for any element.

The idea: when a request fails, replace the target content with a "Something went
wrong" message that includes a Retry button. Clicking the button re-sends the
original request.

First, let us build a reusable retry component:

```gleam
/// Renders a retry button that re-triggers the original request.
/// The `retry_url` is the URL to GET when the user clicks Retry.
/// The `retry_target` is the CSS selector of the element to swap into.
fn retry_panel(
  retry_url: String,
  retry_target: String,
  message: String,
) -> element.Element(Nil) {
  html.div([attribute.class("retry-panel")], [
    html.p([attribute.class("retry-message")], [
      element.text(message),
    ]),
    html.button(
      [
        attribute.class("btn btn-outline"),
        hx.get(retry_url),
        hx.target(hx.Selector(retry_target)),
        hx.swap(hx.InnerHTML),
      ],
      [element.text("Retry")],
    ),
  ])
}
```

This component is server-rendered. When the server knows a request failed (for
example, a database query timed out), it returns the retry panel instead of the
normal content:

```gleam
fn list_tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  case db.list_tasks(ctx.db, board_id) {
    Ok(tasks) -> {
      let today = date_helpers.today()
      let task_html =
        element.fragment(
          list.map(tasks, fn(task) { task_item(task, today) }),
        )
      wisp.html_response(element.to_string(task_html), 200)
    }
    Error(_db_error) -> {
      let panel =
        retry_panel(
          "/boards/" <> board_id <> "/tasks",
          "#task-list",
          "Could not load tasks. The database may be temporarily unavailable.",
        )
      wisp.html_response(element.to_string(panel), 500)
    }
  }
}
```

For client-side retries (when the request itself fails before reaching the server),
we use `_hyperscript` on specific elements. Here is a task list loader that retries
on failure:

```gleam
fn task_list_loader(board_id: String) -> element.Element(Nil) {
  html.div(
    [
      attribute.id("task-list"),
      attribute.class("task-list"),
      hx.get("/boards/" <> board_id <> "/tasks"),
      hx.trigger([hx.load()]),
      hx.swap(hx.InnerHTML),
      attribute.attribute(
        "_",
        "on htmx:sendError from me "
        <> "put 'Failed to load tasks. ' into me "
        <> "then make <button.btn.btn-outline/> called retryBtn "
        <> "then put 'Retry' into retryBtn "
        <> "then set retryBtn@_ to 'on click trigger load on #task-list' "
        <> "then put retryBtn at end of me",
      ),
    ],
    [
      html.span(
        [attribute.class("htmx-indicator")],
        [element.text("Loading tasks...")],
      ),
    ],
  )
}
```

This is more complex than the server-side retry panel, and the `_hyperscript` is
harder to read at this length. If you find yourself writing long `_hyperscript`
handlers, consider moving the logic to a JavaScript function (like we did with
`makeToast`) and calling it from a shorter `_hyperscript` expression:

```gleam
// Simpler approach: delegate to JavaScript
attribute.attribute(
  "_",
  "on htmx:sendError from me call showRetry(me)",
),
```

With the JavaScript function:

```javascript
function showRetry(element) {
  element.innerHTML =
    '<div class="retry-panel">' +
    '<p>Failed to load. </p>' +
    '<button class="btn btn-outline" onclick="htmx.trigger(this.closest(\'[hx-get]\'), \'load\')">Retry</button>' +
    '</div>';
}
```

Both approaches work. Use whichever is more readable in your context.

### Step 6 -- Test Each Error Path

Testing error handling requires simulating failures. Here are four test scenarios
you can run manually using your browser's developer tools.

**Test 1: Network Error**

1. Open the application in your browser.
2. Open DevTools (F12) and go to the Network tab.
3. Click the "Offline" checkbox (or select "Offline" from the throttling dropdown).
4. Try to add a task.
5. **Expected:** A toast appears saying "Network error. Check your connection and
   try again."
6. Uncheck "Offline" and verify the application works again.

**Test 2: Server Error (500)**

Create a temporary test endpoint that always crashes:

```gleam
// Add to your router temporarily for testing
["test-error"] -> {
  // This will cause a crash that the middleware catches
  let assert Ok(_) = Error("intentional crash for testing")
  wisp.ok()
}
```

1. Add a button that hits the test endpoint:

```html
<button hx-get="/test-error" hx-target="#test-result">
  Test Server Error
</button>
<div id="test-result"></div>
```

2. Click the button.
3. **Expected:** A toast appears saying "Server error. Please try again in a
   moment." The `#test-result` div shows the error fragment from the middleware.

**Test 3: Timeout**

Create a test endpoint that sleeps longer than the configured timeout:

```gleam
// This endpoint simulates a slow operation
["test-slow"] -> {
  // Sleep for 15 seconds (longer than the 10-second timeout)
  process.sleep(15_000)
  wisp.html_response("<p>This response is too slow</p>", 200)
}
```

1. Add a button that hits the slow endpoint.
2. Click the button.
3. **Expected:** After 10 seconds, a toast appears saying "Request timed out. The
   server may be busy."

**Test 4: JavaScript Disabled (Progressive Enhancement)**

1. Open DevTools and go to Settings (gear icon or F1 in Chrome).
2. Under "Debugger" (or "Preferences"), check "Disable JavaScript."
3. Navigate to the board page.
4. Fill in the task form and submit.
5. **Expected:** The form submits as a standard HTML form. The page reloads. The
   new task appears in the list. No HTMX, no fragments, no AJAX.
6. Re-enable JavaScript.

For automated testing, you can verify the error middleware in unit tests:

```gleam
// In test/teamwork_test.gleam

import teamwork/middleware
import wisp/testing

pub fn htmx_error_response_returns_fragment_test() {
  let req =
    testing.get("/test-error", [])
    |> testing.set_header("hx-request", "true")

  let response = middleware.handle_errors(req, fn() {
    panic as "test crash"
  })

  // Should return 500 with an HTML fragment (not a full page)
  let assert 500 = response.status
  // The body should contain "Error 500" but NOT "<html>" (no full page)
}

pub fn regular_error_response_returns_full_page_test() {
  let req = testing.get("/test-error", [])

  let response = middleware.handle_errors(req, fn() {
    panic as "test crash"
  })

  // Should return 500 with a full HTML page
  let assert 500 = response.status
  // The body should contain "<html>" (full page with layout)
}
```

---

## 3. Full Code Listing

Here are the complete files incorporating all changes from the walkthrough.

### `src/teamwork/middleware.gleam`

```gleam
import gleam/int
import gleam/list
import lustre/attribute
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}

// ── HTMX Detection ──────────────────────────────────────────────────────

/// Returns True if the request was made by HTMX (has the HX-Request header).
pub fn is_htmx_request(req: Request) -> Bool {
  case list.key_find(req.headers, "hx-request") {
    Ok("true") -> True
    _ -> False
  }
}

// ── Error Middleware ─────────────────────────────────────────────────────

/// Middleware that catches crashes and returns an appropriate error response.
/// For HTMX requests: returns an HTML fragment suitable for swapping.
/// For regular requests: returns a full error page with layout.
pub fn handle_errors(
  req: Request,
  handler: fn() -> Response,
) -> Response {
  let response = wisp.rescue_crashes(handler)

  case response.status {
    500 -> error_response(req, 500, "Something went wrong on our end.")
    503 -> error_response(req, 503, "The service is temporarily unavailable.")
    _ -> response
  }
}

fn error_response(req: Request, status: Int, message: String) -> Response {
  case is_htmx_request(req) {
    True -> htmx_error_fragment(status, message)
    False -> full_error_page(status, message)
  }
}

// ── HTMX Error Fragment ─────────────────────────────────────────────────

fn htmx_error_fragment(status: Int, message: String) -> Response {
  let fragment =
    html.div([attribute.class("error-fragment")], [
      html.strong([], [
        element.text("Error " <> int.to_string(status)),
      ]),
      element.text(" -- "),
      element.text(message),
    ])

  wisp.html_response(element.to_string(fragment), status)
}

// ── Full Error Page ─────────────────────────────────────────────────────

fn full_error_page(status: Int, message: String) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Error -- Teamwork"),
        html.meta([attribute.attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute.attribute("content", "width=device-width, initial-scale=1"),
        ]),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body([], [
        html.div([attribute.class("container")], [
          html.div([attribute.class("error-page")], [
            html.h1([], [
              element.text("Error " <> int.to_string(status)),
            ]),
            html.p([attribute.class("error-message")], [
              element.text(message),
            ]),
            html.p([], [
              element.text("You can "),
              html.a(
                [attribute.href("/")],
                [element.text("go back to the board")],
              ),
              element.text(" or try again in a moment."),
            ]),
          ]),
        ]),
      ]),
    ])

  wisp.html_response(element.to_document_string(page), status)
}
```

### `src/teamwork/web.gleam`

```gleam
import gleam/bool
import gleam/http
import gleam/int
import gleam/list
import gleam/option
import gleam/order
import gleam/result
import gleam/string
import hx
import lustre/attribute.{attribute}
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}

import teamwork/context.{type Context}
import teamwork/date_helpers
import teamwork/db
import teamwork/middleware
import teamwork/validation

// ── Layout ──────────────────────────────────────────────────────────────

fn layout(content: element.Element(Nil)) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork"),
        html.meta([attribute("charset", "utf-8")]),
        html.meta([
          attribute.name("viewport"),
          attribute("content", "width=device-width, initial-scale=1"),
        ]),
        // Application CSS
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
        // HTMX (self-hosted)
        html.script(
          [attribute.src("/static/js/htmx.min.js")],
          "",
        ),
        // response-targets extension
        html.script(
          [attribute.src(
            "https://unpkg.com/htmx-ext-response-targets@2.0.2/response-targets.js",
          )],
          "",
        ),
        // _hyperscript
        html.script(
          [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
          "",
        ),
        // Toast helper function
        html.script([], "
          function makeToast(message, type) {
            var container = document.getElementById('toast-container');
            if (!container) return;

            var toast = document.createElement('div');
            toast.className = 'toast toast-' + type;
            toast.textContent = message;

            var dismiss = document.createElement('button');
            dismiss.className = 'toast-dismiss';
            dismiss.textContent = '\\u00D7';
            dismiss.onclick = function() {
              toast.classList.add('toast-exit');
              setTimeout(function() { toast.remove(); }, 300);
            };
            toast.appendChild(dismiss);

            container.appendChild(toast);

            setTimeout(function() {
              if (toast.parentNode) {
                toast.classList.add('toast-exit');
                setTimeout(function() { toast.remove(); }, 300);
              }
            }, 8000);
          }
        "),
      ]),
      html.body(
        [
          // Activate response-targets extension
          attribute("hx-ext", "response-targets"),
          // Global error event listeners (_hyperscript)
          attribute(
            "_",
            "on htmx:sendError "
            <> "call makeToast('Network error. Check your connection and try again.', 'error') "
            <> "end "
            <> "on htmx:responseError "
            <> "if event.detail.xhr.status >= 500 "
            <> "call makeToast('Server error. Please try again in a moment.', 'error') "
            <> "end "
            <> "on htmx:timeout "
            <> "call makeToast('Request timed out. The server may be busy.', 'warning') "
            <> "end",
          ),
          // Global request timeout: 10 seconds
          attribute("hx-request", "{\"timeout\": 10000}"),
        ],
        [
          // Toast container (fixed position, bottom-right)
          html.div(
            [
              attribute.id("toast-container"),
              attribute.class("toast-container"),
            ],
            [],
          ),
          // Main content
          html.div([attribute.class("container")], [content]),
        ],
      ),
    ])

  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

// ── Components ──────────────────────────────────────────────────────────

fn task_item(task: db.Task, today: String) -> element.Element(Nil) {
  let is_overdue = case task.due_date {
    option.Some(date) -> {
      case string.compare(date, today) {
        order.Lt -> bool.negate(task.done)
        _ -> False
      }
    }
    option.None -> False
  }

  let overdue_class = case is_overdue {
    True -> " overdue"
    False -> ""
  }

  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item" <> overdue_class),
    ],
    [
      html.span(
        [attribute.class("task-title")],
        [element.text(task.title)],
      ),
      html.button(
        [
          attribute.class("btn btn-danger btn-small"),
          hx.delete("/tasks/" <> task.id),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
          attribute("hx-confirm", "Delete this task?"),
        ],
        [element.text("Delete")],
      ),
    ],
  )
}

fn task_form(board_id: String) -> element.Element(Nil) {
  html.div([attribute.id("form-container")], [
    html.form(
      [
        attribute.id("task-form"),
        // Standard HTML attributes (no-JS fallback)
        attribute.method("post"),
        attribute.action("/boards/" <> board_id <> "/tasks"),
        // HTMX attributes (enhanced experience)
        hx.post("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        // Route validation errors to the form container
        attribute("hx-target-422", "#form-container"),
        attribute("hx-swap-422", "outerHTML"),
        // Route server errors to the error banner
        attribute("hx-target-5*", "#error-banner"),
        attribute("hx-swap-5*", "innerHTML"),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("title")],
            [element.text("Title")],
          ),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.required(True),
            attribute.placeholder("What needs to be done?"),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}

fn task_form_with_errors(
  board_id: String,
  errors: List(validation.ValidationError),
  values: List(#(String, String)),
) -> element.Element(Nil) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.div(
      [attribute.id("form-errors"), attribute.class("error-list")],
      list.map(title_errors, fn(e) {
        html.span(
          [attribute.class("error-text")],
          [element.text(validation.error_message(e))],
        )
      }),
    ),
    html.form(
      [
        attribute.id("task-form"),
        attribute.method("post"),
        attribute.action("/boards/" <> board_id <> "/tasks"),
        hx.post("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        attribute("hx-target-422", "#form-container"),
        attribute("hx-swap-422", "outerHTML"),
        attribute("hx-target-5*", "#error-banner"),
        attribute("hx-swap-5*", "innerHTML"),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("title")],
            [element.text("Title")],
          ),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.required(True),
            attribute.value(title_value),
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}

/// Renders a retry panel with a message and a button that re-triggers the request.
fn retry_panel(
  retry_url: String,
  retry_target: String,
  message: String,
) -> element.Element(Nil) {
  html.div([attribute.class("retry-panel")], [
    html.p([attribute.class("retry-message")], [
      element.text(message),
    ]),
    html.button(
      [
        attribute.class("btn btn-outline"),
        hx.get(retry_url),
        hx.target(hx.Selector(retry_target)),
        hx.swap(hx.InnerHTML),
      ],
      [element.text("Retry")],
    ),
  ])
}

// ── Pages ───────────────────────────────────────────────────────────────

fn home_page(req: Request, ctx: Context) -> Response {
  let content =
    html.div([], [
      html.h1([], [element.text("Teamwork Task Board")]),
      task_form("default"),
      html.div(
        [
          attribute.id("task-list"),
          attribute.class("task-list"),
          hx.get("/boards/default/tasks"),
          hx.trigger([hx.load()]),
          hx.swap(hx.InnerHTML),
        ],
        [
          html.span(
            [attribute.class("htmx-indicator")],
            [element.text("Loading tasks...")],
          ),
        ],
      ),
      // Error banner for server errors (hidden by default)
      html.div(
        [
          attribute.id("error-banner"),
          attribute.class("error-banner hidden"),
        ],
        [],
      ),
    ])

  layout(content)
}

// ── Handlers ────────────────────────────────────────────────────────────

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use <- middleware.handle_errors(req)
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)

    ["boards", board_id, "tasks"] ->
      case req.method {
        http.Get -> list_tasks(req, ctx, board_id)
        http.Post -> create_task(req, ctx, board_id)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }

    ["tasks", task_id] ->
      case req.method {
        http.Delete -> delete_task(req, ctx, task_id)
        _ -> wisp.method_not_allowed([http.Delete])
      }

    _ -> wisp.not_found()
  }
}

fn list_tasks(
  req: Request,
  ctx: Context,
  board_id: String,
) -> Response {
  case db.list_tasks(ctx.db, board_id) {
    Ok(tasks) -> {
      let today = date_helpers.today()
      let task_html =
        element.fragment(
          list.map(tasks, fn(task) { task_item(task, today) }),
        )
      wisp.html_response(element.to_string(task_html), 200)
    }
    Error(_db_error) -> {
      let panel =
        retry_panel(
          "/boards/" <> board_id <> "/tasks",
          "#task-list",
          "Could not load tasks. The database may be temporarily unavailable.",
        )
      wisp.html_response(element.to_string(panel), 500)
    }
  }
}

fn create_task(
  req: Request,
  ctx: Context,
  board_id: String,
) -> Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      let task_id = wisp.random_string(16)
      let task =
        db.Task(
          id: task_id,
          title: valid_title,
          description: "",
          done: False,
          due_date: option.None,
        )
      let assert Ok(_) = db.create_task(ctx.db, board_id, task)

      case middleware.is_htmx_request(req) {
        True -> {
          let today = date_helpers.today()
          wisp.html_response(
            element.to_string(task_item(task, today)),
            201,
          )
          |> wisp.set_header("HX-Trigger", "taskAdded")
        }
        False -> {
          wisp.redirect("/boards/" <> board_id)
        }
      }
    }

    Error(errors) -> {
      case middleware.is_htmx_request(req) {
        True -> {
          let form_html =
            task_form_with_errors(board_id, errors, form_data.values)
          wisp.html_response(element.to_string(form_html), 422)
        }
        False -> {
          // For non-HTMX: re-render the full page with the form errors.
          // In a real app, you would pass the errors into the page template.
          // For simplicity, redirect back with an error flash message.
          wisp.redirect("/boards/" <> board_id)
        }
      }
    }
  }
}

fn delete_task(
  req: Request,
  ctx: Context,
  task_id: String,
) -> Response {
  case db.delete_task(ctx.db, task_id) {
    Ok(_) -> {
      case middleware.is_htmx_request(req) {
        True ->
          wisp.html_response("", 200)
          |> wisp.set_header("HX-Trigger", "taskDeleted")
        False ->
          wisp.redirect("/")
      }
    }
    Error(_) -> {
      case middleware.is_htmx_request(req) {
        True -> {
          let panel =
            retry_panel(
              "/tasks/" <> task_id,
              "#task-" <> task_id,
              "Could not delete this task.",
            )
          wisp.html_response(element.to_string(panel), 500)
        }
        False ->
          wisp.redirect("/")
      }
    }
  }
}
```

### `priv/static/css/style.css` (additions)

```css
/* ============================================================
   Toast Notifications
   ============================================================ */

.toast-container {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-width: 400px;
  pointer-events: none;
}

.toast {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 12px 16px;
  border-radius: 6px;
  font-size: 0.9em;
  line-height: 1.4;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  pointer-events: auto;

  /* Entry animation */
  animation: toast-enter 0.3s ease-out;
}

.toast-error {
  background-color: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
}

.toast-warning {
  background-color: #fffbeb;
  border: 1px solid #fde68a;
  color: #92400e;
}

.toast-info {
  background-color: #eff6ff;
  border: 1px solid #bfdbfe;
  color: #1e40af;
}

.toast-dismiss {
  background: none;
  border: none;
  font-size: 1.2em;
  cursor: pointer;
  color: inherit;
  opacity: 0.6;
  padding: 0 4px;
  line-height: 1;
}

.toast-dismiss:hover {
  opacity: 1;
}

/* Exit animation */
.toast-exit {
  animation: toast-exit 0.3s ease-in forwards;
}

@keyframes toast-enter {
  from {
    opacity: 0;
    transform: translateX(100%);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes toast-exit {
  from {
    opacity: 1;
    transform: translateX(0);
  }
  to {
    opacity: 0;
    transform: translateX(100%);
  }
}

/* ============================================================
   Error Pages and Fragments
   ============================================================ */

.error-page {
  text-align: center;
  padding: 60px 20px;
}

.error-page h1 {
  font-size: 2em;
  color: #991b1b;
  margin-bottom: 16px;
}

.error-page .error-message {
  font-size: 1.1em;
  color: #555;
  margin-bottom: 24px;
}

.error-page a {
  color: #2563eb;
  text-decoration: underline;
}

.error-fragment {
  padding: 12px 16px;
  background-color: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
  color: #991b1b;
  font-size: 0.9em;
}

.error-banner {
  padding: 12px 16px;
  background-color: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
  color: #991b1b;
  margin-bottom: 16px;
}

.error-banner.hidden {
  display: none;
}

/* ============================================================
   Retry Panel
   ============================================================ */

.retry-panel {
  text-align: center;
  padding: 32px 16px;
  background-color: #f9fafb;
  border: 1px dashed #d1d5db;
  border-radius: 6px;
}

.retry-message {
  color: #6b7280;
  margin-bottom: 16px;
}

.btn-outline {
  background: transparent;
  border: 1px solid #2563eb;
  color: #2563eb;
  padding: 8px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9em;
}

.btn-outline:hover {
  background-color: #2563eb;
  color: #fff;
}

/* ============================================================
   Error States for Form Inputs
   ============================================================ */

.error-list {
  margin-bottom: 8px;
}

.error-text {
  display: block;
  color: #991b1b;
  font-size: 0.85em;
  margin-bottom: 4px;
}

.input-error {
  border-color: #f87171;
  box-shadow: 0 0 0 1px #f87171;
}

/* ============================================================
   HTMX Loading Indicator
   ============================================================ */

.htmx-indicator {
  display: none;
  color: #6b7280;
  font-size: 0.9em;
}

.htmx-request .htmx-indicator,
.htmx-request.htmx-indicator {
  display: inline-block;
}
```

---

## 4. Exercises

### Task 1: Custom Toast Messages per Endpoint

Right now, all server errors show the same generic "Server error" toast message.
Modify the `htmx:responseError` handler in `_hyperscript` to read a custom error
message from the response body and display it in the toast.

**Acceptance criteria:**
- When `/boards/abc/tasks` returns a 500, the toast shows "Could not load tasks."
- When `/tasks/xyz` (DELETE) returns a 500, the toast shows "Could not delete task."
- When any other endpoint returns a 500, the toast shows the generic message.
- The `_hyperscript` handler reads `event.detail.xhr.responseText` and passes it
  to `makeToast` if it is not empty.

### Task 2: Auto-Dismiss with Progress Bar

Add a visual progress bar to each toast that shows how much time is left before
auto-dismiss. The progress bar should shrink from 100% to 0% over the 8-second
duration.

**Acceptance criteria:**
- Each toast has a thin bar at the bottom.
- The bar animates from full width to zero width over 8 seconds.
- Clicking the dismiss button removes the toast immediately (cancels the timer).
- The bar uses the same color as the toast border.

### Task 3: Exponential Backoff Retry

Build a retry mechanism that automatically retries a failed request with exponential
backoff. After the first failure, wait 1 second. After the second, wait 2 seconds.
After the third, wait 4 seconds. After the fourth, stop retrying and show a "Give
up" message.

**Acceptance criteria:**
- The task list loader retries automatically on network error.
- Each retry shows a "Retrying in N seconds..." message.
- After 4 failed attempts, the message changes to "Could not connect after 4
  attempts" with a manual Retry button.
- A successful retry clears the error state and shows the task list.

### Task 4: Offline Banner

Add a persistent banner at the top of the page that appears when the browser goes
offline and disappears when it comes back online. Use the browser's `online` and
`offline` events.

**Acceptance criteria:**
- When the browser goes offline, a yellow banner appears: "You are offline. Changes
  will not be saved."
- When the browser comes back online, the banner disappears.
- The banner does not interfere with the toast notifications.
- Implement using `_hyperscript` listeners on the `<body>` for the `offline` and
  `online` events.

### Task 5: No-JS Error Page

Create a `<noscript>` fallback that shows a friendly message when JavaScript is
completely disabled. Also, verify that the task creation form still works without
JavaScript by testing the full Post/Redirect/Get flow.

**Acceptance criteria:**
- A `<noscript>` tag in the layout shows: "This application works best with
  JavaScript enabled, but all core features are available without it."
- With JavaScript disabled, creating a task via the form works (page reloads, task
  appears).
- With JavaScript disabled, the error banner and toast are not visible (they
  require JS).
- The form submission uses the standard `method`/`action` attributes.

---

## 5. Exercise Solution Hints

> Try each exercise on your own before reading these hints.

### Hint for Task 1

Modify the `htmx:responseError` listener to extract the response text:

```
on htmx:responseError
  if event.detail.xhr.status >= 500
    set responseBody to event.detail.xhr.responseText
    if responseBody is not ''
      call makeToast(responseBody, 'error')
    else
      call makeToast('Server error. Please try again in a moment.', 'error')
    end
  end
```

On the server side, make sure your 500 responses return plain text error messages
(or strip the HTML tags). Alternatively, return the error message in a custom
response header:

```gleam
wisp.html_response(element.to_string(fragment), 500)
|> wisp.set_header("X-Error-Message", "Could not load tasks.")
```

Then read `event.detail.xhr.getResponseHeader('X-Error-Message')` in the
`_hyperscript` handler.

### Hint for Task 2

Add a progress bar element inside each toast in the `makeToast` function:

```javascript
function makeToast(message, type) {
  // ... existing code ...

  var progress = document.createElement('div');
  progress.className = 'toast-progress toast-progress-' + type;
  progress.style.animationDuration = '8s';
  toast.appendChild(progress);

  // ... rest of existing code ...
}
```

CSS for the progress bar:

```css
.toast-progress {
  position: absolute;
  bottom: 0;
  left: 0;
  height: 3px;
  width: 100%;
  border-radius: 0 0 6px 6px;
  animation: progress-shrink linear forwards;
}

.toast-progress-error { background-color: #fecaca; }
.toast-progress-warning { background-color: #fde68a; }

@keyframes progress-shrink {
  from { width: 100%; }
  to { width: 0%; }
}
```

Make the toast `position: relative` so the progress bar is positioned relative to it.

### Hint for Task 3

Use a JavaScript function with a counter and `setTimeout`:

```javascript
function retryWithBackoff(element, attempt) {
  if (attempt > 4) {
    element.innerHTML = '<div class="retry-panel">' +
      '<p>Could not connect after 4 attempts.</p>' +
      '<button class="btn btn-outline" onclick="retryWithBackoff(this.closest(\'[hx-get]\'), 0)">Retry</button>' +
      '</div>';
    return;
  }

  var delay = Math.pow(2, attempt - 1) * 1000;  // 1s, 2s, 4s, 8s
  element.innerHTML = '<p>Retrying in ' + (delay / 1000) + ' seconds...</p>';

  setTimeout(function() {
    htmx.trigger(element, 'load');
  }, delay);
}
```

Wire it to the `htmx:sendError` event on the task list element:

```gleam
attribute.attribute(
  "_",
  "on htmx:sendError from me call retryWithBackoff(me, (me.dataset.attempt || 0) + 1)",
),
```

Reset the attempt counter on success by listening for `htmx:afterRequest`:

```gleam
attribute.attribute(
  "_",
  "on htmx:sendError from me "
  <> "set my.dataset.attempt to (parseInt(my.dataset.attempt || '0') + 1) "
  <> "call retryWithBackoff(me, parseInt(my.dataset.attempt)) "
  <> "end "
  <> "on htmx:afterRequest[detail.successful] from me "
  <> "set my.dataset.attempt to '0' "
  <> "end",
),
```

### Hint for Task 4

Add `_hyperscript` listeners on the body for the browser's native `offline` and
`online` events:

```gleam
attribute.attribute(
  "_",
  // ... existing error listeners ...
  <> "on offline from window "
  <> "remove .hidden from #offline-banner "
  <> "end "
  <> "on online from window "
  <> "add .hidden to #offline-banner "
  <> "end",
),
```

Add the offline banner to the layout:

```gleam
html.div(
  [
    attribute.id("offline-banner"),
    attribute.class("offline-banner hidden"),
  ],
  [element.text("You are offline. Changes will not be saved.")],
)
```

CSS:

```css
.offline-banner {
  background-color: #fffbeb;
  border-bottom: 1px solid #fde68a;
  color: #92400e;
  text-align: center;
  padding: 8px 16px;
  font-size: 0.9em;
}

.offline-banner.hidden {
  display: none;
}
```

### Hint for Task 5

Add a `<noscript>` tag in the layout body, before the main content:

```gleam
// Lustre does not have a built-in noscript element.
// Use element.element to create one:
element.element("noscript", [], [
  html.div([attribute.class("noscript-banner")], [
    element.text(
      "This application works best with JavaScript enabled, "
      <> "but all core features are available without it.",
    ),
  ]),
])
```

For the Post/Redirect/Get flow, make sure the non-HTMX branch in `create_task`
uses `wisp.redirect`:

```gleam
False -> {
  // Standard form submission: redirect back to the board
  wisp.redirect("/boards/" <> board_id)
}
```

The redirect returns a 303 status code, which tells the browser to issue a GET
request to the redirect URL. This prevents the "resubmit form?" dialog on refresh.

Test by disabling JavaScript in DevTools (Settings > Debugger > Disable JavaScript
in Chrome, or `about:config` > `javascript.enabled` = `false` in Firefox).

---

## 6. Key Takeaways

1. **Errors fall into four categories, and each needs a different strategy.** Network
   errors need a toast and a retry button. Server errors need a toast and a server-side
   retry panel. Validation errors need inline form feedback. Timeouts need a toast with
   a retry option. One-size-fits-all error handling does not work.

2. **HTMX fires specific events for each error type.** `htmx:sendError` for network
   failures, `htmx:responseError` for non-2xx responses, `htmx:timeout` for exceeded
   timeouts, and `htmx:afterRequest` as a catch-all. Listen for these on the `<body>`
   for global handling.

3. **Always set a request timeout.** The default HTMX timeout is zero (wait forever).
   Use `hx-request='{"timeout": 10000}'` on the `<body>` to set a reasonable global
   default. Override it on specific elements that need more time.

4. **HTMX-aware middleware is essential.** When a request has the `HX-Request: true`
   header, your error responses must return HTML fragments. When the header is absent,
   return full pages. Swapping a full page into a `<div>` target creates a
   page-inside-a-page disaster.

5. **Progressive enhancement costs very little.** Adding `method="post"` and
   `action="/tasks"` to a form that already has `hx-post="/tasks"` takes two lines.
   The server needs a branch that checks `is_htmx_request` and either returns a
   fragment or redirects. The payoff is that your forms work even when JavaScript
   fails to load.

6. **Toast notifications are for infrastructure errors, not validation errors.**
   Validation errors should appear inline, next to the field that has the problem.
   Toasts should appear for problems that are not tied to a specific form element:
   network failures, server crashes, and timeouts.

7. **Server-side retry panels are more reliable than client-side retries.** When the
   server can catch an error (like a database timeout), it returns a retry panel with
   the correct URL and target already set. The user clicks one button and the request
   is re-sent. This is more reliable than client-side retry logic because the server
   knows exactly what failed.

8. **Manual retries are safer than automatic retries.** Automatic exponential backoff
   sounds clever, but it can overwhelm a server that is already struggling. A manual
   retry button gives the user control and gives the server time to recover. Use
   automatic retries only for idempotent read operations (GET requests), never for
   state-changing operations (POST, DELETE).

9. **Test every error path.** Use the browser's DevTools to simulate offline mode,
   create test endpoints that crash or sleep, and disable JavaScript to verify
   progressive enhancement. If you have not tested an error path, it does not work.

---

## What's Next

This chapter gave you a complete error handling strategy: toast notifications for
infrastructure errors, HTMX-aware middleware for server crashes, progressive
enhancement for forms, and retry patterns for failed requests. The application is
no longer fragile -- it communicates problems clearly and gives users a path to
recovery.

But error handling is only half of the server-client communication story. In
[Chapter 20](20-response-headers-and-server-control.md), we will explore **HTMX response headers and server-driven control**.
HTMX provides a set of response headers (`HX-Trigger`, `HX-Redirect`,
`HX-Reswap`, `HX-Retarget`, and more) that let the server dynamically control
what happens on the client after a request completes. Instead of hardcoding
every swap target and trigger in your HTML attributes, you can have the server
decide at response time -- giving you a powerful feedback loop between your Gleam
handlers and the browser.

Together, Chapters 19 and [Chapter 20](20-response-headers-and-server-control.md) form the resilience and control foundation for every
pattern that follows. Error handling tells the user when something went wrong.
Response headers let the server orchestrate what happens next. Both are necessary
for an application that feels responsive and trustworthy.

---
