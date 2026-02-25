# Chapter 20 -- HTMX Response Headers and Server-Driven Control

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 19](19-error-handling-and-graceful-degradation.md) introduced structured error handling with
`response-targets`, flash messages, and recovery patterns. This chapter
moves from reacting to errors to **controlling the client from the server**
-- deciding where content lands, how the URL bar updates, and when events
fire, all through HTTP response headers.

---

## Learning Objectives

By the end of this chapter you will be able to:

- List every HTMX response header and explain when each is appropriate.
- Use `HX-Retarget` and `HX-Reswap` to override client-side target/swap from the server.
- Use `HX-Push-Url` and `HX-Replace-Url` for server-controlled browser history.
- Use `HX-Trigger-After-Settle` to fire events after CSS transitions complete.
- Distinguish `HX-Redirect` (full navigation) from `HX-Location` (HTMX-powered navigation).
- Build a typed `htmx_headers` helper module in Gleam using `wisp.set_header`.

---

## 1. Theory

### 1.1 The Server-Driven Philosophy

Most JavaScript frameworks put the browser in charge. The client decides where
fetched data goes, how the URL changes, and which animations play. The server
is reduced to a JSON endpoint -- a data vending machine that has no opinion
about presentation.

HTMX inverts this. The server returns HTML, and HTMX puts it into the DOM.
But HTMX goes further than simple "fetch HTML, swap it in." Through
**response headers**, the server can tell HTMX:

- *Where* to put the content (overriding the client's `hx-target`).
- *How* to swap it (overriding the client's `hx-swap`).
- *What URL* to show in the browser's address bar.
- *Whether* to navigate to a completely different page.
- *Which events* to fire, and *when* to fire them.

This is the **server-driven philosophy**. The server is not just a data source.
It is the decision-maker. It sees the full picture -- the database state, the
user's permissions, the result of the operation -- and it makes the call about
what the client should do next.

Why does this matter? Consider a task creation form:

```
Client says:  POST /tasks
              hx-target="#task-list"
              hx-swap="beforeend"

Server thinks: "The task was created successfully.
                But wait -- the user just created their 100th task.
                I want to show a congratulations banner AND
                update the task list AND push the new URL."
```

With response headers, the server can do all of this. It can override the
target, push a URL, fire custom events, and trigger animations -- all from
a single response. The client's attributes are *defaults*. The server's
headers are *overrides*.

This is a fundamentally different power dynamic from the SPA model, and it
is one of the most compelling features of HTMX.

Here is the full picture:

```
  ┌─────────────────────────────────────────────────────┐
  │                    BROWSER                           │
  │                                                      │
  │  hx-target="#task-list"     ← client default         │
  │  hx-swap="beforeend"       ← client default         │
  │                                                      │
  │  ──────── HTMX Request ──────────────────────►       │
  │                                                      │
  │                               ┌──────────────────┐  │
  │                               │     SERVER       │  │
  │                               │                  │  │
  │                               │  Decides:        │  │
  │                               │  - HX-Retarget   │  │
  │                               │  - HX-Reswap     │  │
  │                               │  - HX-Push-Url   │  │
  │                               │  - HX-Trigger    │  │
  │                               │                  │  │
  │                               └──────────────────┘  │
  │                                                      │
  │  ◄─────── Response + Headers ────────────────────    │
  │                                                      │
  │  HTMX checks:                                        │
  │  1. HX-Retarget? → overrides hx-target               │
  │  2. HX-Reswap?   → overrides hx-swap                 │
  │  3. HX-Push-Url? → updates address bar               │
  │  4. HX-Trigger?  → fires custom events               │
  │                                                      │
  │  Client attributes are DEFAULTS.                     │
  │  Server headers are OVERRIDES.                       │
  └─────────────────────────────────────────────────────┘
```

### 1.2 Target and Swap Override Headers (HX-Retarget, HX-Reswap)

In [Chapter 9](../02-intermediate/09-swap-strategies-and-targets.md) we learned that `hx-target` tells HTMX which DOM element should
receive the response content, and `hx-swap` tells HTMX *how* to put it there
(innerHTML, outerHTML, beforeend, etc.). These are set on the client as HTML
attributes.

But sometimes the server needs to change those decisions. The classic example
is a form that can succeed or fail:

- **Success:** The new task should be appended to `#task-list` using `beforeend`.
- **Validation error:** The error messages should replace the content inside
  `#form-container` using `innerHTML`.

You *could* solve this with the `response-targets` extension ([Chapter 16](../03-advanced/16-extensions-and-patterns.md)),
which maps HTTP status codes to different targets on the client side. But
`response-targets` requires you to decide the mapping at HTML-authoring time.
What if the server needs to make a decision that the client cannot anticipate?

That is where `HX-Retarget` and `HX-Reswap` come in.

**`HX-Retarget`** overrides the element that receives the response. Its value
is a CSS selector:

```
HX-Retarget: #error-banner
```

When HTMX sees this header, it ignores whatever `hx-target` was set on the
triggering element and instead swaps the content into the element matching
`#error-banner`.

**`HX-Reswap`** overrides the swap strategy. Its value is any valid
`hx-swap` value:

```
HX-Reswap: innerHTML
```

You can combine them:

```
HX-Retarget: #form-errors
HX-Reswap: innerHTML
```

This tells HTMX: "Forget what the client said. Put this content inside
`#form-errors` using `innerHTML`."

Here is a concrete scenario. The task form has these attributes:

```html
<form hx-post="/tasks"
      hx-target="#task-list"
      hx-swap="beforeend">
```

On success, the server returns the new task `<li>` and everything works
as expected -- the task appears at the bottom of the list.

On validation failure, the server returns error HTML and overrides the
target:

```
HTTP/1.1 422 Unprocessable Content
HX-Retarget: #form-container
HX-Reswap: innerHTML

<form hx-post="/tasks" ...>
  <div class="error">Title cannot be empty</div>
  <input name="title" value="" />
  <button type="submit">Add Task</button>
</form>
```

The error-annotated form replaces the original form, and the task list
is untouched. The server made the decision, not the client.

**HX-Reswap modifiers.** The `HX-Reswap` header supports the same modifiers
as the `hx-swap` attribute:

```
HX-Reswap: innerHTML swap:200ms settle:100ms
HX-Reswap: beforeend show:bottom
HX-Reswap: outerHTML transition:true
```

This gives the server full control over swap timing and scroll behaviour.

**When to use HX-Retarget/HX-Reswap vs. response-targets:**

| Scenario | Recommendation |
|----------|---------------|
| Error handling with predictable status codes (422, 500) | `response-targets` extension -- client-side, declarative |
| Server-side conditional logic (permissions, business rules) | `HX-Retarget` / `HX-Reswap` -- server-side, dynamic |
| Different targets for success vs. failure | Either works; `response-targets` is simpler |
| Target depends on *data* (e.g., which board the task belongs to) | `HX-Retarget` -- only the server knows |

### 1.3 URL Management Headers (HX-Push-Url, HX-Replace-Url)

When a user navigates a traditional website, the browser updates the address
bar with each page. This is important for three reasons:

1. **Bookmarking.** The user can save the URL and return to the same page later.
2. **Sharing.** The user can copy the URL and send it to someone else.
3. **Back button.** The browser's history lets the user go back to where
   they were.

HTMX requests are AJAX calls. By default, they do not change the URL. The
address bar stays wherever it was when the page first loaded. For small
interactions -- toggling a checkbox, adding a task -- this is fine. The user
does not expect the URL to change when they check off a to-do item.

But for larger interactions -- navigating to a different board, opening a task
detail view, applying a complex filter -- the URL *should* change. HTMX
provides two client-side attributes for this:

- `hx-push-url="true"` adds a new entry to the browser history.
- `hx-replace-url="true"` replaces the current entry (no new back-button step).

The server-side equivalents are response headers:

**`HX-Push-Url`** adds a new entry to the browser's history stack:

```
HX-Push-Url: /boards/abc/tasks/42
```

The address bar changes to `/boards/abc/tasks/42`, and pressing the back
button returns to the previous URL. You can also pass `false` to explicitly
*prevent* a URL push that the client requested:

```
HX-Push-Url: false
```

**`HX-Replace-Url`** replaces the current history entry without adding a
new one:

```
HX-Replace-Url: /boards/abc?filter=active
```

The address bar updates, but the back button still goes to wherever the user
was *before* the current page. This is useful for filter changes -- the user
does not want to press "back" fifteen times to undo each filter tweak.

When to use which:

| Action | Header | Why |
|--------|--------|-----|
| Navigate to a new page/view | `HX-Push-Url` | User expects back button to work |
| Apply a filter or sort | `HX-Replace-Url` | Filter changes are not navigation steps |
| Create a resource and show it | `HX-Push-Url` | The new resource has its own URL |
| Tab switching within a page | `HX-Replace-Url` | Tabs are a view change, not navigation |
| Prevent URL change on error | `HX-Push-Url: false` | Failed actions should not alter the URL |

Here is a comparison of all four URL mechanisms:

```
                    Client-side              Server-side
                    ───────────              ───────────
  Push (new entry)  hx-push-url="true"       HX-Push-Url: /path
  Replace (same)    hx-replace-url="true"    HX-Replace-Url: /path
  Prevent push      hx-push-url="false"      HX-Push-Url: false
  Prevent replace   hx-replace-url="false"   HX-Replace-Url: false
```

The server headers always win. If the client says `hx-push-url="true"` but
the server responds with `HX-Push-Url: false`, the URL does not change.

**Important:** `HX-Push-Url` and `HX-Replace-Url` only change the address
bar. They do not trigger a page load. The content is still swapped via AJAX.
If you need a full page navigation, use `HX-Redirect` or `HX-Location`
(covered in the next section).

### 1.4 Navigation Headers (HX-Redirect, HX-Location, HX-Refresh)

Sometimes the server does not want to return a fragment for HTMX to swap.
It wants to send the user somewhere else entirely. HTMX provides three
headers for this, and they behave very differently.

**`HX-Redirect`** -- Full page navigation.

We first saw this in [Chapter 12](../02-intermediate/12-cookies-sessions-auth.md). When HTMX receives a response with the
`HX-Redirect` header, it performs a **full page navigation** -- equivalent
to `window.location.href = url`. The entire page reloads. The browser
fetches the new URL as a normal request (not an HTMX request).

```
HX-Redirect: /boards
```

Use this when:
- The user logs in and should go to the dashboard.
- The user's session expires and they need to re-authenticate.
- You need a completely fresh page (all state reset).

**`HX-Location`** -- HTMX-powered navigation.

`HX-Location` is the HTMX-native alternative to `HX-Redirect`. Instead of
a full page reload, HTMX makes a **new HTMX request** to the specified URL.
This is faster because the browser does not need to re-parse the entire page
-- only the swapped content changes.

In its simplest form:

```
HX-Location: /boards/abc
```

This is equivalent to the user clicking an element with `hx-get="/boards/abc"
hx-target="body" hx-swap="innerHTML" hx-push-url="true"`. The page content
updates, the URL bar changes, and a history entry is created -- all without
a full reload.

For more control, `HX-Location` accepts a JSON value:

```
HX-Location: {"path": "/boards/abc", "target": "#main-content", "swap": "innerHTML"}
```

The JSON object supports these keys:

| Key | Description | Default |
|-----|------------|---------|
| `path` | The URL to navigate to | (required) |
| `target` | CSS selector for the swap target | `body` |
| `swap` | Swap strategy | `innerHTML` |
| `source` | Source element for the request | The element that triggered the original request |
| `event` | Event that triggered the request | (none) |
| `handler` | Event handler for the request | (none) |
| `select` | CSS selector to cherry-pick from the response | (none) |
| `headers` | Additional headers to include | `{}` |
| `values` | Values to submit with the request | `{}` |

The JSON form is powerful. It lets the server say "navigate to this page,
but only update the main content area, and use this specific swap strategy."
This is the foundation for modal-to-page transitions, which we will explore
in [Chapter 22](22-modal-dialogs.md).

**`HX-Refresh`** -- Reload the current page.

```
HX-Refresh: true
```

This is the nuclear option. HTMX tells the browser to reload the entire
current page -- equivalent to `location.reload()`. Use it sparingly:

- Session invalidation that requires a fresh page.
- Database migration that changes the page structure.
- A rare "something went so wrong that we need to start over" scenario.

In practice, you will reach for `HX-Refresh` far less often than the
other headers. It exists as an escape hatch.

Here is a comparison table:

| Header | Behaviour | URL changes? | Full reload? | Back button? |
|--------|-----------|-------------|-------------|-------------|
| `HX-Redirect` | Navigate to new URL (full) | Yes | Yes | Yes (new entry) |
| `HX-Location` | Navigate via HTMX (AJAX) | Yes (push) | No | Yes (new entry) |
| `HX-Location` (JSON) | Navigate via HTMX with options | Configurable | No | Configurable |
| `HX-Refresh` | Reload current page | No | Yes | No |

**When to use HX-Redirect vs. HX-Location:**

```
  Need full page reload?
  │
  ├── Yes → HX-Redirect
  │         (login, logout, session expiry)
  │
  └── No → HX-Location
           (navigation within the app, modal-to-page)
```

`HX-Location` is almost always the better choice within a running
application. It is faster because HTMX only swaps the content region
instead of re-parsing the entire HTML document, re-loading CSS, and
re-running JavaScript. Save `HX-Redirect` for transitions that truly
need a clean slate.

### 1.5 Trigger Timing Headers (HX-Trigger, After-Swap, After-Settle)

In [Chapter 16](../03-advanced/16-extensions-and-patterns.md) we learned that the `HX-Trigger` response header fires a
custom event on the element that made the request. Other elements can
listen for that event and refresh themselves. This is the coordination
mechanism that lets independent page sections stay in sync.

But *when* does the event fire? HTMX provides three timing options:

| Header | When it fires |
|--------|---------------|
| `HX-Trigger` | Immediately upon receiving the response, **before** the swap |
| `HX-Trigger-After-Swap` | After the new content has been swapped into the DOM |
| `HX-Trigger-After-Settle` | After the settle phase completes (CSS transitions finish) |

This distinction matters when your event handlers interact with the DOM.

**`HX-Trigger`** fires first. At this point, the new content is **not yet**
in the DOM. The old content is still there. This is useful for events that
do not need the new DOM -- for example, triggering a notification toast or
updating a counter badge.

**`HX-Trigger-After-Swap`** fires after the new content is in the DOM, but
before CSS transitions have completed. The new elements exist, and you can
query them with JavaScript, but they may still have transition classes
applied (`htmx-added`, `htmx-settling`).

**`HX-Trigger-After-Settle`** fires after the settle phase. This is the
moment when:
- All CSS transition classes have been removed.
- The DOM is in its final, stable state.
- Elements have their final positions and dimensions.

This is the right timing for:
- Scrolling to a newly added element (you need its final position).
- Initializing a third-party library on new content.
- Measuring element dimensions for layout calculations.
- Focusing an input field in newly swapped content.

Here is the timeline:

```
  Response received
  │
  ├── HX-Trigger fires               ← DOM has OLD content
  │
  ├── Swap happens                    ← New content enters DOM
  │   (htmx-added class applied)
  │
  ├── HX-Trigger-After-Swap fires    ← New content is in DOM
  │                                     (but may have transition classes)
  │
  ├── Settle phase                    ← CSS transitions play out
  │   (htmx-settling class applied,
  │    then removed)
  │
  └── HX-Trigger-After-Settle fires  ← DOM is stable, transitions done
```

All three headers accept the same value formats:

**Single event:**
```
HX-Trigger: taskCreated
```

**Multiple events:**
```
HX-Trigger: taskCreated, statsUpdated
```

**Events with data:**
```
HX-Trigger: {"taskCreated": {"id": "42", "title": "Fix bug"}}
```

When events carry data, event listeners receive it via `event.detail`:

```javascript
document.addEventListener("taskCreated", function(event) {
  console.log(event.detail.id);    // "42"
  console.log(event.detail.title); // "Fix bug"
});
```

Or in `_hyperscript`:
```html
<div _="on taskCreated(id, title) from body
        put title into my.textContent">
```

**Common combinations:**

```
// Refresh stats immediately, scroll to new task after settle
HX-Trigger: statsUpdated
HX-Trigger-After-Settle: scrollToNew
```

```
// Show a flash message before swap, initialize widgets after settle
HX-Trigger: showFlash
HX-Trigger-After-Settle: initWidgets
```

### 1.6 Combining Multiple Headers (Precedence, Common Combinations)

In real applications, you often need to set several response headers at once.
A single response to "create a task" might need to:

1. Return the new task HTML (the body).
2. Override the target to append to the task list (`HX-Retarget`).
3. Push the new task's URL (`HX-Push-Url`).
4. Fire an event to refresh the stats counter (`HX-Trigger`).
5. Fire an event to scroll to the new task after CSS transitions
   (`HX-Trigger-After-Settle`).

All of these can coexist on a single response:

```
HTTP/1.1 201 Created
Content-Type: text/html
HX-Retarget: #task-list
HX-Reswap: beforeend
HX-Push-Url: /boards/abc/tasks/42
HX-Trigger: statsUpdated
HX-Trigger-After-Settle: scrollToNew

<li id="task-42" class="task-item">
  <span>Fix the login bug</span>
</li>
```

**Precedence rules:**

HTMX processes response headers in a specific order. Understanding this order
helps you predict behaviour when headers interact:

1. **Navigation headers are checked first.** If `HX-Redirect`, `HX-Location`,
   or `HX-Refresh` is present, HTMX stops processing and navigates. No swap
   happens, no events fire.
2. **Target override is applied next.** `HX-Retarget` replaces the resolved
   `hx-target`.
3. **Swap override is applied.** `HX-Reswap` replaces the resolved `hx-swap`.
4. **The swap is performed.** Content enters the DOM.
5. **URL headers are processed.** `HX-Push-Url` or `HX-Replace-Url` updates
   the address bar.
6. **Trigger headers fire.** In order: `HX-Trigger`, then
   `HX-Trigger-After-Swap`, then `HX-Trigger-After-Settle`.

```
  Response arrives
  │
  ├── HX-Redirect / HX-Location / HX-Refresh?
  │   └── Yes → navigate, STOP (no swap, no triggers)
  │
  ├── HX-Retarget? → override target
  ├── HX-Reswap?   → override swap strategy
  │
  ├── Perform swap
  │
  ├── HX-Push-Url / HX-Replace-Url? → update address bar
  │
  ├── HX-Trigger         → fire events (pre-swap timing)
  ├── HX-Trigger-After-Swap  → fire events
  └── HX-Trigger-After-Settle → fire events
```

**Important:** Do not combine navigation headers with swap headers. If you
set both `HX-Redirect` and `HX-Retarget`, the redirect wins and the retarget
is ignored. This is a common mistake:

```
// BAD: HX-Redirect cancels the swap entirely
HX-Redirect: /boards
HX-Retarget: #task-list    ← ignored!
HX-Trigger: statsUpdated   ← ignored!
```

**Common header combinations for the Teamwork app:**

| Scenario | Headers |
|----------|---------|
| Create task (success) | `HX-Trigger: statsUpdated` + `HX-Trigger-After-Settle: scrollToNew` |
| Create task (with URL) | `HX-Push-Url: /boards/:id/tasks/:tid` + `HX-Trigger: statsUpdated` |
| Validation error | `HX-Retarget: #form-container` + `HX-Reswap: innerHTML` |
| Delete task | `HX-Trigger: statsUpdated` |
| Apply filter | `HX-Replace-Url: /boards/:id?filter=active` |
| Login success | `HX-Redirect: /boards` |
| Session expired | `HX-Refresh: true` |
| Modal to full page | `HX-Location: {"path": "/tasks/42", "target": "#main"}` |

### 1.7 Helper Functions in Gleam (`htmx_headers.gleam` Module)

Setting response headers with raw strings is error-prone. You can misspell
a header name, forget the `HX-` prefix, or pass an invalid value. Gleam's
type system cannot help you if the header names and values are just strings.

The solution is a small helper module that wraps each header in a typed
function. Each function takes a `wisp.Response` and returns a `wisp.Response`
with the appropriate header set. This gives you:

1. **Autocomplete.** Your editor shows available headers as function names.
2. **Typo protection.** A misspelled function name is a compile error.
3. **Documentation.** Each function can have a doc comment explaining when
   to use it.
4. **Consistency.** Header names are defined in one place.

Here is the design:

```gleam
// src/teamwork/htmx_headers.gleam

import wisp

/// Override the target element for the swap.
/// The selector should be a valid CSS selector (e.g., "#task-list").
pub fn retarget(
  response: wisp.Response,
  selector: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Retarget", selector)
}

/// Override the swap strategy.
/// Accepts any valid hx-swap value (e.g., "innerHTML", "beforeend").
pub fn reswap(
  response: wisp.Response,
  strategy: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Reswap", strategy)
}
```

The pattern is simple: one function per header, each taking a `wisp.Response`
and returning a `wisp.Response`. This means they compose with the pipe
operator:

```gleam
wisp.html_response(body, 201)
|> htmx_headers.retarget("#task-list")
|> htmx_headers.reswap("beforeend")
|> htmx_headers.push_url("/boards/abc/tasks/42")
|> htmx_headers.trigger("statsUpdated")
|> htmx_headers.trigger_after_settle("scrollToNew")
```

Compare that to raw header strings:

```gleam
wisp.html_response(body, 201)
|> wisp.set_header("HX-Retarget", "#task-list")
|> wisp.set_header("HX-Reswap", "beforeend")
|> wisp.set_header("HX-Push-Url", "/boards/abc/tasks/42")
|> wisp.set_header("HX-Trigger", "statsUpdated")
|> wisp.set_header("HX-Trigger-After-Settle", "scrollToNew")
```

Both work. But the first version catches typos at compile time, reads more
fluently, and is easier to scan.

We will build the complete module in the Code Walkthrough.

**Complete reference table of all HTMX response headers:**

| Header | Value | Purpose | Example |
|--------|-------|---------|---------|
| `HX-Location` | URL or JSON | HTMX-powered navigation (no full reload) | `HX-Location: /boards/abc` |
| `HX-Push-Url` | URL or `false` | Add entry to browser history | `HX-Push-Url: /tasks/42` |
| `HX-Redirect` | URL | Full page navigation (reload) | `HX-Redirect: /login` |
| `HX-Refresh` | `true` | Reload the current page | `HX-Refresh: true` |
| `HX-Replace-Url` | URL or `false` | Replace current history entry | `HX-Replace-Url: /boards?q=bug` |
| `HX-Reswap` | swap strategy | Override `hx-swap` from server | `HX-Reswap: outerHTML` |
| `HX-Retarget` | CSS selector | Override `hx-target` from server | `HX-Retarget: #errors` |
| `HX-Trigger` | event name(s) or JSON | Fire event(s) immediately | `HX-Trigger: taskCreated` |
| `HX-Trigger-After-Swap` | event name(s) or JSON | Fire event(s) after swap | `HX-Trigger-After-Swap: highlight` |
| `HX-Trigger-After-Settle` | event name(s) or JSON | Fire event(s) after settle | `HX-Trigger-After-Settle: scroll` |

That is every HTMX response header. Ten headers, each with a focused purpose.
Together they give the server complete control over the client's behaviour.

---

## 2. Code Walkthrough

We are going to add server-driven control to the Teamwork board. Here is
what we will build:

1. A reusable `htmx_headers.gleam` helper module.
2. Server-driven target override for task creation (success vs. error).
3. URL management after task creation.
4. `HX-Location` for navigating from a task preview to its full page.
5. `HX-Trigger-After-Settle` for smooth scroll-to-new-task.
6. `HX-Refresh` for session invalidation.

### Step 1 -- Build `htmx_headers.gleam` Helper Module

Create a new module at `src/teamwork/htmx_headers.gleam`. This module wraps
every HTMX response header in a typed function. Each function follows the
same pattern: take a `wisp.Response`, return a `wisp.Response`.

```gleam
// src/teamwork/htmx_headers.gleam

import gleam/json
import gleam/list
import wisp

// --- Target and Swap Overrides ---

/// Override the swap target. The server decides where the content goes,
/// regardless of what `hx-target` the client specified.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(error_html, 422)
/// |> htmx_headers.retarget("#form-errors")
/// ```
pub fn retarget(
  response: wisp.Response,
  selector: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Retarget", selector)
}

/// Override the swap strategy. Works with any value that `hx-swap`
/// accepts, including modifiers like "innerHTML swap:200ms".
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.reswap("beforeend show:bottom")
/// ```
pub fn reswap(
  response: wisp.Response,
  strategy: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Reswap", strategy)
}

// --- URL Management ---

/// Push a new URL onto the browser's history stack.
/// The address bar updates and the back button returns to the previous URL.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.push_url("/boards/abc/tasks/42")
/// ```
pub fn push_url(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Push-Url", url)
}

/// Prevent a URL push, even if the client requested one with
/// `hx-push-url="true"`.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(error_html, 422)
/// |> htmx_headers.prevent_push()
/// ```
pub fn prevent_push(response: wisp.Response) -> wisp.Response {
  wisp.set_header(response, "HX-Push-Url", "false")
}

/// Replace the current URL in the browser's history (no new entry).
/// Useful for filter changes and sort operations.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(filtered_html, 200)
/// |> htmx_headers.replace_url("/boards/abc?filter=active")
/// ```
pub fn replace_url(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Replace-Url", url)
}

/// Prevent a URL replacement, even if the client requested one with
/// `hx-replace-url="true"`.
pub fn prevent_replace(response: wisp.Response) -> wisp.Response {
  wisp.set_header(response, "HX-Replace-Url", "false")
}

// --- Navigation ---

/// Full page redirect. The browser navigates to the URL as if the user
/// clicked a link. The entire page reloads.
///
/// Use for login redirects, session expiry, and transitions that need
/// a completely fresh page.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response("", 200)
/// |> htmx_headers.redirect("/boards")
/// ```
pub fn redirect(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Redirect", url)
}

/// HTMX-powered navigation. Performs an AJAX request to the URL and
/// swaps the result into the page. Faster than a full redirect because
/// only the content area changes.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response("", 200)
/// |> htmx_headers.location("/boards/abc")
/// ```
pub fn location(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Location", url)
}

/// HTMX-powered navigation with full control over target, swap, and
/// other options. Pass a JSON string with the desired configuration.
///
/// ## Example
///
/// ```gleam
/// let location_json =
///   json.object([
///     #("path", json.string("/boards/abc")),
///     #("target", json.string("#main-content")),
///     #("swap", json.string("innerHTML")),
///   ])
///   |> json.to_string
///
/// wisp.html_response("", 200)
/// |> htmx_headers.location(location_json)
/// ```
pub fn location_with_options(
  response: wisp.Response,
  path: String,
  target: String,
  swap: String,
) -> wisp.Response {
  let value =
    json.object([
      #("path", json.string(path)),
      #("target", json.string(target)),
      #("swap", json.string(swap)),
    ])
    |> json.to_string

  wisp.set_header(response, "HX-Location", value)
}

/// Reload the current page. Use sparingly -- this is the nuclear option.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response("", 200)
/// |> htmx_headers.refresh()
/// ```
pub fn refresh(response: wisp.Response) -> wisp.Response {
  wisp.set_header(response, "HX-Refresh", "true")
}

// --- Event Triggers ---

/// Fire a custom event immediately (before the swap).
/// Other elements can listen for this event to refresh themselves.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.trigger("statsUpdated")
/// ```
pub fn trigger(
  response: wisp.Response,
  event: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Trigger", event)
}

/// Fire multiple custom events immediately (before the swap).
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.trigger_many(["statsUpdated", "listChanged"])
/// ```
pub fn trigger_many(
  response: wisp.Response,
  events: List(String),
) -> wisp.Response {
  let value =
    events
    |> string_join(", ")

  wisp.set_header(response, "HX-Trigger", value)
}

/// Fire a custom event with JSON data. Listeners receive the data
/// via `event.detail`.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.trigger_with_data(
///   "taskCreated",
///   json.object([#("id", json.string("42"))]),
/// )
/// ```
pub fn trigger_with_data(
  response: wisp.Response,
  event: String,
  data: json.Json,
) -> wisp.Response {
  let value =
    json.object([#(event, data)])
    |> json.to_string

  wisp.set_header(response, "HX-Trigger", value)
}

/// Fire a custom event after the swap completes.
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.trigger_after_swap("highlightNew")
/// ```
pub fn trigger_after_swap(
  response: wisp.Response,
  event: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Trigger-After-Swap", event)
}

/// Fire a custom event after the settle phase (CSS transitions done).
///
/// ## Example
///
/// ```gleam
/// wisp.html_response(task_html, 201)
/// |> htmx_headers.trigger_after_settle("scrollToNew")
/// ```
pub fn trigger_after_settle(
  response: wisp.Response,
  event: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Trigger-After-Settle", event)
}

// --- Internal Helpers ---

fn string_join(items: List(String), separator: String) -> String {
  case items {
    [] -> ""
    [first, ..rest] ->
      list.fold(rest, first, fn(acc, item) {
        acc <> separator <> item
      })
  }
}
```

A few design notes:

- Every function takes `response` as the first argument. This is intentional --
  it enables pipe chaining with `|>`.
- The `trigger_with_data` function uses Gleam's `json` module to build the
  JSON value. This guarantees valid JSON without manual string concatenation.
- The `location_with_options` function only exposes the three most common
  options (`path`, `target`, `swap`). For the full JSON form, use `location`
  with a manually constructed JSON string.
- The `prevent_push` and `prevent_replace` functions take no extra arguments --
  they always set the value to `"false"`.

### Step 2 -- Server-Driven Target Override

Now let us use the helper module. When a user submits the task creation form,
the server decides where the response goes based on whether validation passed.

Here is the existing form in our layout:

```gleam
fn task_form(board_id: String) -> element.Element(t) {
  html.form(
    [
      hx.post("/boards/" <> board_id <> "/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
      attribute.id("task-form"),
    ],
    [
      html.div([attribute.id("form-errors")], []),
      html.div([attribute.class("form-group")], [
        html.label(
          [attribute.for("title")],
          [element.text("Task title")],
        ),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.placeholder("What needs to be done?"),
          attribute.required(True),
        ]),
      ]),
      html.button(
        [attribute.type_("submit"), attribute.class("btn btn-primary")],
        [element.text("Add Task")],
      ),
    ],
  )
}
```

The form targets `#task-list` with `beforeend`. That is the happy path. Now
here is the handler that uses server-driven overrides for the error path:

```gleam
import teamwork/htmx_headers

fn handle_create_task(
  req: wisp.Request,
  board_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")
    |> string.trim

  // Validate
  case title {
    "" -> {
      // Server decides: send errors to #form-errors, not #task-list
      let error_html =
        html.div(
          [attribute.class("error-message")],
          [element.text("Task title cannot be empty.")],
        )
        |> element.to_string

      wisp.html_response(error_html, 422)
      |> htmx_headers.retarget("#form-errors")
      |> htmx_headers.reswap("innerHTML")
    }

    _ -> {
      // Create the task
      let task_id = generate_id()
      let task = Task(id: task_id, title: title, done: False)
      // ... save to state ...

      // Return the new task item
      let task_html =
        task_item(board_id, task)
        |> element.to_string

      wisp.html_response(task_html, 201)
      |> htmx_headers.trigger("statsUpdated")
    }
  }
}
```

Let us trace through both paths:

**Success path:**
1. Form posts to `/boards/abc/tasks`.
2. Server validates the title -- it is not empty.
3. Server creates the task and returns the `<li>` HTML with status 201.
4. No `HX-Retarget` or `HX-Reswap` headers are set.
5. HTMX uses the form's `hx-target="#task-list"` and `hx-swap="beforeend"`.
6. The new task appears at the bottom of the list.
7. The `HX-Trigger: statsUpdated` header fires the `statsUpdated` event,
   which refreshes the task counter.

**Error path:**
1. Form posts to `/boards/abc/tasks`.
2. Server validates the title -- it is empty.
3. Server returns error HTML with status 422.
4. `HX-Retarget: #form-errors` overrides the form's `hx-target="#task-list"`.
5. `HX-Reswap: innerHTML` overrides the form's `hx-swap="beforeend"`.
6. The error message appears inside `#form-errors`, right above the input.
7. The task list is untouched.

The server made different decisions based on the outcome, and the client
responded accordingly. No JavaScript, no conditional logic in the HTML.

### Step 3 -- URL Management After Task Creation

When a user creates a task, we want the URL to reflect the new state. This
lets the user bookmark the board and share it with others. We will use
`HX-Push-Url` to add the board URL with a task count parameter.

Update the success branch of the handler:

```gleam
    _ -> {
      // Create the task
      let task_id = generate_id()
      let task = Task(id: task_id, title: title, done: False)
      // ... save to state ...

      let task_html =
        task_item(board_id, task)
        |> element.to_string

      let task_url =
        "/boards/" <> board_id <> "/tasks/" <> task_id

      wisp.html_response(task_html, 201)
      |> htmx_headers.push_url(task_url)
      |> htmx_headers.trigger("statsUpdated")
    }
```

After the swap, the address bar updates to something like
`/boards/abc/tasks/42`. The user can press the back button to return to the
previous URL, and the forward button to come back.

For filters, we use `replace_url` instead. When the user filters tasks by
status, the URL should update but the back button should not accumulate
filter steps:

```gleam
fn handle_filter_tasks(
  req: wisp.Request,
  board_id: String,
) -> wisp.Response {
  let query = wisp.get_query(req)
  let filter =
    list.key_find(query, "filter")
    |> result.unwrap("all")

  // ... fetch and filter tasks ...

  let tasks_html =
    task_list(board_id, filtered_tasks)
    |> element.to_string

  let filter_url =
    "/boards/" <> board_id <> "?filter=" <> filter

  wisp.html_response(tasks_html, 200)
  |> htmx_headers.replace_url(filter_url)
}
```

Now when the user selects "Active" from a filter dropdown, the URL changes
to `/boards/abc?filter=active`. Selecting "Completed" changes it to
`/boards/abc?filter=completed`. But pressing the back button skips all those
filter changes and goes to wherever the user was *before* they started
filtering. This is the right UX -- filter adjustments are refinements, not
navigation steps.

**Preventing URL changes on error:**

If the form has `hx-push-url="true"` but the server rejects the input, you
do not want the URL to change. Use `prevent_push`:

```gleam
    "" -> {
      let error_html =
        html.div(
          [attribute.class("error-message")],
          [element.text("Task title cannot be empty.")],
        )
        |> element.to_string

      wisp.html_response(error_html, 422)
      |> htmx_headers.retarget("#form-errors")
      |> htmx_headers.reswap("innerHTML")
      |> htmx_headers.prevent_push()
    }
```

### Step 4 -- `HX-Location` for Modal-to-Page Navigation

In [Chapter 22](22-modal-dialogs.md) we will build modals. But here is a preview of a pattern that
uses `HX-Location`. Imagine the user clicks on a task in the list and sees
a quick preview modal. The modal has a "View full page" button. When the user
clicks it, you want to:

1. Close the modal.
2. Navigate to the task's full page.
3. Update the URL.
4. Only swap the main content area (not reload the entire page).

`HX-Redirect` would work, but it reloads the entire page -- including the
sidebar, the header, the CSS, and all the JavaScript. Slow and janky.

`HX-Location` is the right tool:

```gleam
fn handle_task_full_view(
  _req: wisp.Request,
  board_id: String,
  task_id: String,
) -> wisp.Response {
  // The response body can be empty -- HX-Location triggers a NEW request
  wisp.html_response("", 200)
  |> htmx_headers.location_with_options(
    "/boards/" <> board_id <> "/tasks/" <> task_id,
    "#main-content",
    "innerHTML",
  )
}
```

When HTMX processes this response, it:

1. Ignores the empty body (navigation headers stop the swap).
2. Makes a new GET request to `/boards/abc/tasks/42`.
3. Takes the response and swaps it into `#main-content` using `innerHTML`.
4. Pushes `/boards/abc/tasks/42` onto the browser history.

The page transition is seamless. The header, sidebar, and navigation stay
put. Only the main content area changes. The URL updates. The back button
works. And it all happened because the server said so.

The handler for the full page view returns the task detail HTML:

```gleam
fn handle_task_detail(
  _req: wisp.Request,
  board_id: String,
  task_id: String,
) -> wisp.Response {
  // ... look up task ...
  let detail_html =
    task_detail_view(board_id, task)
    |> element.to_string

  wisp.html_response(detail_html, 200)
}

fn task_detail_view(
  board_id: String,
  task: Task,
) -> element.Element(t) {
  html.div([attribute.class("task-detail")], [
    html.h2([], [element.text(task.title)]),
    html.p(
      [attribute.class("task-meta")],
      [element.text("Board: " <> board_id)],
    ),
    html.div([attribute.class("task-status")], [
      element.text(case task.done {
        True -> "Completed"
        False -> "In progress"
      }),
    ]),
    html.a(
      [
        hx.get("/boards/" <> board_id),
        hx.target(hx.Selector("#main-content")),
        hx.swap(hx.InnerHTML),
        hx.push_url(True),
        attribute.class("btn"),
      ],
      [element.text("Back to board")],
    ),
  ])
}
```

Notice the "Back to board" link uses regular HTMX attributes (`hx-get`,
`hx-target`, `hx-push-url`). The client-side attributes work fine for
predictable navigation like this. The server-side `HX-Location` header is
for cases where the server needs to make the navigation decision -- like
closing a modal and redirecting based on server-side logic.

### Step 5 -- `HX-Trigger-After-Settle` for Scroll-to-New-Task

When a user adds a task to a long list, the new item appears at the bottom.
If the list is scrollable, the new task is out of view. The user has to
scroll down manually to see that their action worked. That is a poor
experience.

We can fix this with `HX-Trigger-After-Settle` and a small JavaScript
listener. The timeline:

1. User submits the form.
2. Server creates the task and returns the `<li>`.
3. HTMX swaps the `<li>` into `#task-list` (beforeend).
4. CSS transition plays (the new item fades in).
5. After the settle phase, HTMX fires the `scrollToNew` event.
6. A JavaScript listener scrolls the new item into view.

Why `After-Settle` and not `After-Swap`? Because during the swap phase, the
new element may still have `htmx-added` classes applied and CSS transitions
in progress. Its final position and dimensions are not yet determined. If you
scroll to it now, you might scroll to the wrong position. After settle, the
element is in its final state.

First, update the handler to fire the event:

```gleam
    _ -> {
      let task_id = generate_id()
      let task = Task(id: task_id, title: title, done: False)
      // ... save to state ...

      let task_html =
        task_item(board_id, task)
        |> element.to_string

      wisp.html_response(task_html, 201)
      |> htmx_headers.trigger("statsUpdated")
      |> htmx_headers.trigger_after_settle("scrollToNew")
    }
```

Then add a small JavaScript listener in the layout. This is one of the rare
cases where a `<script>` block is appropriate:

```gleam
fn layout(content: element.Element(t)) -> element.Element(t) {
  html.html([], [
    html.head([], [
      html.title([], "Teamwork -- Task Board"),
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
      html.link([
        attribute.rel("stylesheet"),
        attribute.href("/static/css/style.css"),
      ]),
    ]),
    html.body([], [
      html.div([attribute.id("main-content")], [content]),
      // Scroll-to-new-task listener
      html.script([], "
        document.addEventListener('scrollToNew', function(event) {
          var taskList = document.getElementById('task-list');
          if (taskList && taskList.lastElementChild) {
            taskList.lastElementChild.scrollIntoView({
              behavior: 'smooth',
              block: 'nearest'
            });
          }
        });
      "),
    ]),
  ])
}
```

The listener finds the last child of `#task-list` (which is the newly added
task) and scrolls it into view with smooth animation. The `block: 'nearest'`
option means it only scrolls if the element is not already visible -- no
jarring scroll when the list is short enough to show everything.

You could also use `_hyperscript` to avoid the script block entirely. Add
this attribute to the `#task-list` element:

```gleam
html.ul(
  [
    attribute.id("task-list"),
    attribute.attribute(
      "_",
      "on scrollToNew from body wait 50ms "
      <> "call my.lastElementChild.scrollIntoView("
      <> "{behavior: 'smooth', block: 'nearest'})",
    ),
  ],
  task_items,
)
```

The `wait 50ms` gives the browser a moment to finalize layout after the
settle phase. This is a safety margin -- in most cases the layout is already
stable, but the wait ensures it works even on slower devices.

**Sending event data with the trigger:**

If you have multiple lists on the page and need to know which one received
the new item, you can send data with the event:

```gleam
wisp.html_response(task_html, 201)
|> htmx_headers.trigger_with_data(
  "scrollToNew",
  json.object([
    #("listId", json.string("task-list")),
    #("taskId", json.string(task_id)),
  ]),
)
```

The JavaScript listener can then use the data:

```javascript
document.addEventListener('scrollToNew', function(event) {
  var listId = event.detail.listId;
  var taskId = event.detail.taskId;
  var task = document.getElementById('task-' + taskId);
  if (task) {
    task.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
  }
});
```

This is more precise -- it scrolls to the exact task, not just the last
child. Use this approach when your pages have multiple dynamic lists.

### Step 6 -- `HX-Refresh` for Exceptional Cases

`HX-Refresh` is the escape hatch. You should rarely need it, but when you
do, it is invaluable.

The most common scenario is **session invalidation**. Imagine a user is
working on the board when an admin invalidates all sessions (maybe the server
was compromised, or a security policy changed). The next HTMX request from
that user will fail authentication. Instead of showing a confusing error
message inside the task list, you want a full page refresh that redirects to
the login page.

Here is how the auth middleware handles this:

```gleam
fn require_auth(
  req: wisp.Request,
  next: fn(User) -> wisp.Response,
) -> wisp.Response {
  case get_session_user(req) {
    Ok(user) -> next(user)

    Error(_) -> {
      // Check if this is an HTMX request
      let is_htmx =
        list.key_find(req.headers, "hx-request")
        |> result.unwrap("") == "true"

      case is_htmx {
        True -> {
          // HTMX request with invalid session -- force a full refresh.
          // The refresh will hit the auth middleware again, this time
          // as a regular request, and get redirected to /login.
          wisp.html_response("", 200)
          |> htmx_headers.refresh()
        }

        False -> {
          // Regular request -- standard redirect to login
          wisp.redirect("/login")
        }
      }
    }
  }
}
```

Let us walk through the flow:

1. User's session expires while they are on `/boards/abc`.
2. User clicks "Add Task", triggering an HTMX POST.
3. The auth middleware sees the expired session.
4. Since this is an HTMX request (the `hx-request` header is present), the
   middleware returns `HX-Refresh: true`.
5. HTMX refreshes the entire page.
6. The browser makes a fresh GET to `/boards/abc` (a regular request, not
   HTMX).
7. The auth middleware sees the expired session again, but this time it is
   a regular request, so it returns a 302 redirect to `/login`.
8. The user sees the login page.

Why not use `HX-Redirect: /login` directly? You could, and in many cases it
works fine. But `HX-Refresh` has an advantage: it preserves the user's current
URL. After the refresh and login redirect, the browser's "previous page" in
history is still `/boards/abc`. Some login flows use this to redirect the user
back to where they were after authentication. With `HX-Redirect: /login`, the
history would show `/login` as the previous page, which is less useful.

Another use for `HX-Refresh` is when the server's HTML structure has changed
dramatically -- perhaps after a deployment that reorganizes the page layout.
Rather than trying to swap partial HTML into a page whose structure no longer
matches, a full refresh gets the user onto the new version cleanly.

But to be clear: if you are reaching for `HX-Refresh` regularly, something
is wrong. It should be an exceptional fallback, not a routine response.

---

## 3. Full Code Listing

Here is the complete `htmx_headers.gleam` module and the updated router with
all the patterns from this chapter.

### `src/teamwork/htmx_headers.gleam`

```gleam
// src/teamwork/htmx_headers.gleam
//
// Typed helper functions for HTMX response headers.
// Each function takes a wisp.Response and returns a wisp.Response,
// enabling pipe-chain composition.

import gleam/json
import gleam/list
import wisp

// ── Target and Swap Overrides ───────────────────────────────────────

/// Override the swap target from the server side.
/// The value is a CSS selector (e.g., "#task-list", ".error-container").
pub fn retarget(
  response: wisp.Response,
  selector: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Retarget", selector)
}

/// Override the swap strategy from the server side.
/// Accepts any valid hx-swap value, including modifiers:
/// "innerHTML", "outerHTML", "beforeend", "afterend",
/// "beforeend show:bottom", "innerHTML swap:200ms settle:100ms".
pub fn reswap(
  response: wisp.Response,
  strategy: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Reswap", strategy)
}

// ── URL Management ──────────────────────────────────────────────────

/// Push a new URL onto the browser history stack.
/// Creates a new back-button entry.
pub fn push_url(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Push-Url", url)
}

/// Prevent a URL push even if the client requested one.
pub fn prevent_push(response: wisp.Response) -> wisp.Response {
  wisp.set_header(response, "HX-Push-Url", "false")
}

/// Replace the current URL in browser history (no new entry).
/// Ideal for filter/sort changes.
pub fn replace_url(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Replace-Url", url)
}

/// Prevent a URL replacement even if the client requested one.
pub fn prevent_replace(response: wisp.Response) -> wisp.Response {
  wisp.set_header(response, "HX-Replace-Url", "false")
}

// ── Navigation ──────────────────────────────────────────────────────

/// Full page redirect. Triggers a complete page reload at the given URL.
/// Use for login redirects and session transitions.
pub fn redirect(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Redirect", url)
}

/// HTMX-powered navigation. Makes a new HTMX request to the URL
/// and swaps the response into the page (default: body innerHTML).
/// Faster than HX-Redirect because only content changes.
pub fn location(
  response: wisp.Response,
  url: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Location", url)
}

/// HTMX-powered navigation with explicit target and swap strategy.
/// The values are encoded as a JSON object in the header.
pub fn location_with_options(
  response: wisp.Response,
  path: String,
  target: String,
  swap: String,
) -> wisp.Response {
  let value =
    json.object([
      #("path", json.string(path)),
      #("target", json.string(target)),
      #("swap", json.string(swap)),
    ])
    |> json.to_string

  wisp.set_header(response, "HX-Location", value)
}

/// Reload the current page entirely.
/// Use sparingly: session invalidation, major layout changes.
pub fn refresh(response: wisp.Response) -> wisp.Response {
  wisp.set_header(response, "HX-Refresh", "true")
}

// ── Event Triggers ──────────────────────────────────────────────────

/// Fire a single custom event immediately (before the swap).
pub fn trigger(
  response: wisp.Response,
  event: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Trigger", event)
}

/// Fire multiple custom events immediately (before the swap).
pub fn trigger_many(
  response: wisp.Response,
  events: List(String),
) -> wisp.Response {
  let value = string_join(events, ", ")
  wisp.set_header(response, "HX-Trigger", value)
}

/// Fire a custom event with associated JSON data.
/// Listeners receive the data in `event.detail`.
pub fn trigger_with_data(
  response: wisp.Response,
  event: String,
  data: json.Json,
) -> wisp.Response {
  let value =
    json.object([#(event, data)])
    |> json.to_string

  wisp.set_header(response, "HX-Trigger", value)
}

/// Fire a single custom event after the swap completes.
pub fn trigger_after_swap(
  response: wisp.Response,
  event: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Trigger-After-Swap", event)
}

/// Fire a single custom event after the settle phase
/// (CSS transitions done, DOM stable).
pub fn trigger_after_settle(
  response: wisp.Response,
  event: String,
) -> wisp.Response {
  wisp.set_header(response, "HX-Trigger-After-Settle", event)
}

// ── Internal Helpers ────────────────────────────────────────────────

fn string_join(items: List(String), separator: String) -> String {
  case items {
    [] -> ""
    [first, ..rest] ->
      list.fold(rest, first, fn(acc, item) {
        acc <> separator <> item
      })
  }
}
```

### `src/teamwork/router.gleam` (relevant excerpts)

```gleam
// src/teamwork/router.gleam

import gleam/http
import gleam/list
import gleam/result
import gleam/string
import lustre/attribute
import lustre/element
import lustre/element/html
import hx
import teamwork/htmx_headers
import wisp

pub fn handle_request(req: wisp.Request) -> wisp.Response {
  use req <- middleware(req)

  case wisp.path_segments(req) {
    // Login
    ["login"] -> handle_login_routes(req)

    // Boards
    ["boards"] -> {
      use user <- require_auth(req)
      handle_boards(req, user)
    }

    // Board detail + tasks
    ["boards", board_id] -> {
      use user <- require_auth(req)
      handle_board_detail(req, board_id, user)
    }

    // Task operations
    ["boards", board_id, "tasks"] -> {
      use user <- require_auth(req)
      handle_tasks(req, board_id, user)
    }

    // Filter tasks (literal "filter" must precede variable task_id)
    ["boards", board_id, "tasks", "filter"] -> {
      use _user <- require_auth(req)
      handle_filter_tasks(req, board_id)
    }

    // Task detail
    ["boards", board_id, "tasks", task_id] -> {
      use user <- require_auth(req)
      handle_task_detail_routes(req, board_id, task_id, user)
    }

    // Task full view (from modal)
    ["boards", board_id, "tasks", task_id, "full"] -> {
      use _user <- require_auth(req)
      handle_task_full_view(req, board_id, task_id)
    }

    _ -> wisp.not_found()
  }
}

// ── Auth Middleware ──────────────────────────────────────────────────

fn require_auth(
  req: wisp.Request,
  next: fn(User) -> wisp.Response,
) -> wisp.Response {
  case get_session_user(req) {
    Ok(user) -> next(user)

    Error(_) -> {
      let is_htmx =
        list.key_find(req.headers, "hx-request")
        |> result.unwrap("")
        == "true"

      case is_htmx {
        True ->
          wisp.html_response("", 200)
          |> htmx_headers.refresh()

        False ->
          wisp.redirect("/login")
      }
    }
  }
}

// ── Task Creation ───────────────────────────────────────────────────

fn handle_tasks(
  req: wisp.Request,
  board_id: String,
  _user: User,
) -> wisp.Response {
  case req.method {
    http.Post -> handle_create_task(req, board_id)
    _ -> wisp.method_not_allowed([http.Post])
  }
}

fn handle_create_task(
  req: wisp.Request,
  board_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")
    |> string.trim

  case title {
    "" -> {
      let error_html =
        html.div(
          [attribute.class("error-message")],
          [element.text("Task title cannot be empty.")],
        )
        |> element.to_string

      wisp.html_response(error_html, 422)
      |> htmx_headers.retarget("#form-errors")
      |> htmx_headers.reswap("innerHTML")
      |> htmx_headers.prevent_push()
    }

    _ -> {
      let task_id = generate_id()
      let task = Task(id: task_id, title: title, done: False)
      // ... save task to state ...

      let task_html =
        task_item(board_id, task)
        |> element.to_string

      let task_url =
        "/boards/" <> board_id <> "/tasks/" <> task_id

      wisp.html_response(task_html, 201)
      |> htmx_headers.push_url(task_url)
      |> htmx_headers.trigger("statsUpdated")
      |> htmx_headers.trigger_after_settle("scrollToNew")
    }
  }
}

// ── Filter Tasks ────────────────────────────────────────────────────

fn handle_filter_tasks(
  req: wisp.Request,
  board_id: String,
) -> wisp.Response {
  let query = wisp.get_query(req)
  let filter =
    list.key_find(query, "filter")
    |> result.unwrap("all")

  // ... fetch and filter tasks based on filter value ...
  let filtered_tasks = filter_tasks(board_id, filter)

  let tasks_html =
    task_list_items(board_id, filtered_tasks)
    |> element.to_string

  let filter_url =
    "/boards/" <> board_id <> "?filter=" <> filter

  wisp.html_response(tasks_html, 200)
  |> htmx_headers.replace_url(filter_url)
}

// ── Task Full View (from Modal Preview) ─────────────────────────────

fn handle_task_full_view(
  _req: wisp.Request,
  board_id: String,
  task_id: String,
) -> wisp.Response {
  wisp.html_response("", 200)
  |> htmx_headers.location_with_options(
    "/boards/" <> board_id <> "/tasks/" <> task_id,
    "#main-content",
    "innerHTML",
  )
}

// ── Task Detail ─────────────────────────────────────────────────────

fn handle_task_detail_routes(
  req: wisp.Request,
  board_id: String,
  task_id: String,
  _user: User,
) -> wisp.Response {
  case req.method {
    http.Get -> {
      // ... look up task ...
      let detail_html =
        task_detail_view(board_id, task)
        |> element.to_string

      wisp.html_response(detail_html, 200)
    }

    _ -> wisp.method_not_allowed([http.Get])
  }
}

// ── View Functions ──────────────────────────────────────────────────

fn task_form(board_id: String) -> element.Element(t) {
  html.form(
    [
      hx.post("/boards/" <> board_id <> "/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
      attribute.id("task-form"),
    ],
    [
      html.div([attribute.id("form-errors")], []),
      html.div([attribute.class("form-group")], [
        html.label(
          [attribute.for("title")],
          [element.text("Task title")],
        ),
        html.input([
          attribute.type_("text"),
          attribute.name("title"),
          attribute.id("title"),
          attribute.placeholder("What needs to be done?"),
          attribute.required(True),
        ]),
      ]),
      html.button(
        [
          attribute.type_("submit"),
          attribute.class("btn btn-primary"),
        ],
        [element.text("Add Task")],
      ),
    ],
  )
}

fn task_item(
  board_id: String,
  task: Task,
) -> element.Element(t) {
  html.li(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item"),
    ],
    [
      html.input([
        attribute.type_("checkbox"),
        attribute.checked(task.done),
        hx.post(
          "/boards/" <> board_id <> "/tasks/" <> task.id <> "/toggle",
        ),
        hx.target(hx.Selector("#task-" <> task.id)),
        hx.swap(hx.OuterHTML),
      ]),
      html.span(
        [attribute.class(case task.done {
          True -> "task-title done"
          False -> "task-title"
        })],
        [element.text(task.title)],
      ),
    ],
  )
}

fn task_detail_view(
  board_id: String,
  task: Task,
) -> element.Element(t) {
  html.div([attribute.class("task-detail")], [
    html.h2([], [element.text(task.title)]),
    html.p(
      [attribute.class("task-meta")],
      [element.text("Board: " <> board_id)],
    ),
    html.div([attribute.class("task-status")], [
      element.text(case task.done {
        True -> "Completed"
        False -> "In progress"
      }),
    ]),
    html.a(
      [
        hx.get("/boards/" <> board_id),
        hx.target(hx.Selector("#main-content")),
        hx.swap(hx.InnerHTML),
        hx.push_url(True),
        attribute.class("btn"),
      ],
      [element.text("Back to board")],
    ),
  ])
}

fn task_list_items(
  board_id: String,
  tasks: List(Task),
) -> element.Element(t) {
  element.fragment(
    list.map(tasks, fn(task) { task_item(board_id, task) }),
  )
}

fn layout(content: element.Element(t)) -> element.Element(t) {
  html.html([], [
    html.head([], [
      html.title([], "Teamwork -- Task Board"),
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),
      html.link([
        attribute.rel("stylesheet"),
        attribute.href("/static/css/style.css"),
      ]),
    ]),
    html.body([], [
      html.div([attribute.id("main-content")], [content]),
      // Scroll-to-new-task listener
      html.script([], "
        document.addEventListener('scrollToNew', function(event) {
          var taskList = document.getElementById('task-list');
          if (taskList && taskList.lastElementChild) {
            taskList.lastElementChild.scrollIntoView({
              behavior: 'smooth',
              block: 'nearest'
            });
          }
        });
      "),
    ]),
  ])
}
```

### `static/css/style.css` (transition styles for new tasks)

```css
/* ── HTMX swap transitions ─────────────────────────────────── */

/*
  When HTMX adds new content, it applies the htmx-added class.
  We use this to create a fade-in effect for new tasks.
*/
.task-item.htmx-added {
  opacity: 0;
  transform: translateY(-10px);
}

.task-item {
  transition: opacity 300ms ease-out,
              transform 300ms ease-out;
}

/*
  The htmx-settling class is applied during the settle phase.
  By the time it is removed, the element has its final styles.
*/
.task-item.htmx-settling {
  opacity: 1;
  transform: translateY(0);
}

/* ── Error message styling ──────────────────────────────────── */

.error-message {
  color: #dc2626;
  background-color: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 4px;
  padding: 8px 12px;
  margin-bottom: 8px;
  font-size: 0.875rem;
}
```

---

## 4. Exercises

### Task 1: Build a Toast Notification System

Build a toast notification that appears at the top-right corner of the page
whenever a task is created, updated, or deleted. Use `HX-Trigger` with event
data to pass the notification message from the server.

**Requirements:**

1. Add a `#toast-container` div to the layout (fixed position, top-right).
2. When the server creates a task, include `HX-Trigger` with a `showToast`
   event carrying `{"message": "Task created!", "type": "success"}`.
3. Write a JavaScript listener (or `_hyperscript`) that receives the event,
   creates a toast element, displays it for 3 seconds, and then removes it.
4. Support three toast types: `success` (green), `warning` (yellow), and
   `error` (red).
5. Multiple toasts should stack vertically.

**Acceptance criteria:**
- Creating a task shows a green toast "Task created!" for 3 seconds.
- Validation failures show a red toast with the error message.
- Toasts stack if multiple arrive before the first disappears.
- Toasts fade out before being removed from the DOM.

### Task 2: Breadcrumb URL Tracking

Implement breadcrumb navigation that updates as the user navigates deeper
into the application. Use `HX-Push-Url` and `HX-Replace-Url` to keep the
breadcrumbs in sync with the address bar.

**Requirements:**

1. Add a breadcrumb bar to the layout: `Home > Boards > Board Name > Task`.
2. When the user navigates to a board, use `HX-Push-Url` and include an
   OOB swap to update the breadcrumb.
3. When the user applies a filter, use `HX-Replace-Url` (no breadcrumb
   change -- filters are not a navigation level).
4. The browser back button should update the breadcrumb correctly.

**Acceptance criteria:**
- Navigating to a board shows `Home > Boards > [Board Name]`.
- Opening a task shows `Home > Boards > [Board Name] > [Task Title]`.
- Applying a filter updates the URL but not the breadcrumb.
- Pressing "Back" restores the previous breadcrumb state.

### Task 3: Conditional Retargeting Based on Permissions

Implement server-driven retargeting based on user permissions. When a
regular user tries to delete a task they do not own, the server returns
an error message retargeted to a permission-error area instead of removing
the task.

**Requirements:**

1. Add an `owner_id` field to the `Task` type.
2. On delete, check if the current user is the task owner.
3. If yes, return 200 with an empty body and `HX-Trigger: statsUpdated`
   (the task was already removed via `outerHTML` swap of an empty string).
4. If no, return 403 with an error message, retargeted to `#permission-error`
   with `innerHTML` swap.
5. Add a `#permission-error` div above the task list.

**Acceptance criteria:**
- Task owners can delete their tasks normally.
- Non-owners see "You do not have permission to delete this task" in the
  permission error area.
- The task remains in the list when deletion is denied.
- The error message disappears after 5 seconds (use `_hyperscript` or JS).

### Task 4: Smart Navigation with HX-Location

Build a "quick view" panel that shows task details in a sidebar. When the
user clicks "Open full page" in the sidebar, use `HX-Location` to navigate
to the full task page without a full reload.

**Requirements:**

1. Add a sidebar panel (`#sidebar`) to the layout.
2. Clicking a task title loads task details into `#sidebar` via
   `hx-get` and `hx-target="#sidebar"`.
3. The sidebar has an "Open full page" button.
4. Clicking "Open full page" sends a request to the server.
5. The server responds with `HX-Location` targeting `#main-content`.
6. The sidebar should close (or empty) when full-page navigation happens.

**Acceptance criteria:**
- Clicking a task title shows details in the sidebar.
- Clicking "Open full page" replaces `#main-content` with the task detail
  view.
- The URL updates to `/boards/:id/tasks/:task_id`.
- The back button returns to the board view.
- The sidebar closes after navigation.

### Task 5: Response Header Debugging Middleware

Build a development middleware that logs all HTMX response headers to the
browser console. This is a useful debugging tool for development.

**Requirements:**

1. Create a `debug_htmx_headers` middleware function.
2. The middleware runs after the handler and inspects the response headers.
3. For every header that starts with `HX-`, the middleware adds an
   `HX-Trigger-After-Settle` event called `debugHeaders` with the header
   data.
4. Add a JavaScript listener that logs the data to `console.table`.
5. Only enable this middleware in development (check an environment variable).

**Acceptance criteria:**
- Every HTMX response logs its HTMX headers to the browser console.
- The console output is formatted as a table showing header name and value.
- The middleware does not affect the actual response behaviour.
- The middleware is disabled in production.

---

## 5. Exercise Solution Hints

> Try each exercise on your own before reading these hints.

### Hint for Task 1

For the toast container, add a fixed-position div to the layout:

```gleam
html.div(
  [
    attribute.id("toast-container"),
    attribute.styles([
      #("position", "fixed"),
      #("top", "1rem"),
      #("right", "1rem"),
      #("z-index", "9999"),
    ]),
  ],
  [],
)
```

Use `trigger_with_data` to send the event:

```gleam
wisp.html_response(task_html, 201)
|> htmx_headers.trigger_with_data(
  "showToast",
  json.object([
    #("message", json.string("Task created!")),
    #("type", json.string("success")),
  ]),
)
```

The JavaScript listener creates a toast element and removes it after a delay:

```javascript
document.addEventListener('showToast', function(event) {
  var container = document.getElementById('toast-container');
  var toast = document.createElement('div');
  toast.className = 'toast toast-' + event.detail.type;
  toast.textContent = event.detail.message;
  container.appendChild(toast);
  setTimeout(function() {
    toast.style.opacity = '0';
    setTimeout(function() { toast.remove(); }, 300);
  }, 3000);
});
```

### Hint for Task 2

The breadcrumb updates work best with OOB swaps combined with URL headers.
When navigating to a board:

```gleam
let breadcrumb_oob =
  html.nav(
    [
      attribute.id("breadcrumb"),
      attribute.attribute("hx-swap-oob", "innerHTML"),
    ],
    [
      html.a([attribute.href("/")], [element.text("Home")]),
      element.text(" > "),
      html.a([attribute.href("/boards")], [element.text("Boards")]),
      element.text(" > "),
      html.span([], [element.text(board.name)]),
    ],
  )
```

Include it in the response alongside the main content using
`element.fragment`. For the back button, HTMX restores the previous page
from its history cache. If the breadcrumb was part of the page content that
was cached, it will restore correctly.

For filter operations, only set `replace_url` -- no breadcrumb update needed
since filters do not change the navigation level.

### Hint for Task 3

The key is checking ownership in the delete handler and responding
differently:

```gleam
fn handle_delete_task(
  req: wisp.Request,
  board_id: String,
  task_id: String,
  user: User,
) -> wisp.Response {
  case find_task(board_id, task_id) {
    Error(_) -> wisp.not_found()

    Ok(task) ->
      case task.owner_id == user.id {
        True -> {
          // Delete the task
          // ... remove from state ...
          wisp.html_response("", 200)
          |> htmx_headers.trigger("statsUpdated")
        }

        False -> {
          let error_html =
            html.div(
              [
                attribute.class("permission-error"),
                attribute.attribute(
                  "_",
                  "on load wait 5s then transition *opacity to 0 "
                  <> "over 300ms then remove me",
                ),
              ],
              [element.text("You do not have permission to delete this task.")],
            )
            |> element.to_string

          wisp.html_response(error_html, 403)
          |> htmx_headers.retarget("#permission-error")
          |> htmx_headers.reswap("innerHTML")
        }
      }
  }
}
```

The `_hyperscript` attribute on the error div makes it auto-dismiss after
5 seconds with a fade-out animation.

### Hint for Task 4

The sidebar "Open full page" button should target a server endpoint that
returns `HX-Location`:

```gleam
html.button(
  [
    hx.get(
      "/boards/" <> board_id <> "/tasks/" <> task_id <> "/full",
    ),
    attribute.class("btn btn-secondary"),
  ],
  [element.text("Open full page")],
)
```

The server endpoint returns:

```gleam
fn handle_task_full_view(
  _req: wisp.Request,
  board_id: String,
  task_id: String,
) -> wisp.Response {
  wisp.html_response("", 200)
  |> htmx_headers.location_with_options(
    "/boards/" <> board_id <> "/tasks/" <> task_id,
    "#main-content",
    "innerHTML",
  )
}
```

To close the sidebar, fire an additional event:

```gleam
|> htmx_headers.trigger("closeSidebar")
```

However, remember the precedence rules: navigation headers (`HX-Location`)
stop all processing, including trigger headers. So you need a different
approach for closing the sidebar. Use `_hyperscript` on the sidebar element:

```gleam
html.div(
  [
    attribute.id("sidebar"),
    attribute.attribute(
      "_",
      "on htmx:beforeRequest from <button.open-full/> "
      <> "put '' into me",
    ),
  ],
  [],
)
```

This listens for the request event on the "Open full page" button and clears
the sidebar immediately.

### Hint for Task 5

The middleware wraps the handler and inspects the response:

```gleam
fn debug_htmx_headers(
  handler: fn() -> wisp.Response,
) -> wisp.Response {
  let response = handler()

  let htmx_header_names = [
    "HX-Location", "HX-Push-Url", "HX-Redirect", "HX-Refresh",
    "HX-Replace-Url", "HX-Reswap", "HX-Retarget", "HX-Trigger",
    "HX-Trigger-After-Swap", "HX-Trigger-After-Settle",
  ]

  let found_headers =
    list.filter_map(htmx_header_names, fn(name) {
      case list.key_find(response.headers, string.lowercase(name)) {
        Ok(value) -> Ok(#(name, value))
        Error(_) -> Error(Nil)
      }
    })

  case found_headers {
    [] -> response
    headers -> {
      let debug_data =
        json.object(
          list.map(headers, fn(pair) {
            #(pair.0, json.string(pair.1))
          }),
        )

      // Careful: don't overwrite an existing HX-Trigger-After-Settle
      // In a real implementation, you would merge the values
      response
    }
  }
}
```

A simpler approach is to log on the server side using `wisp.log_info` and
skip the browser console entirely. This avoids the complexity of merging
trigger headers:

```gleam
fn debug_htmx_headers(
  handler: fn() -> wisp.Response,
) -> wisp.Response {
  let response = handler()
  list.each(response.headers, fn(header) {
    case string.starts_with(header.0, "hx-") {
      True ->
        wisp.log_info(
          "HTMX header: " <> header.0 <> " = " <> header.1,
        )
      False -> Nil
    }
  })
  response
}
```

---

## 6. Key Takeaways

1. **HTMX response headers let the server override client-side decisions.**
   The client's `hx-target`, `hx-swap`, and `hx-push-url` attributes are
   defaults. The server's `HX-Retarget`, `HX-Reswap`, and `HX-Push-Url`
   headers are overrides. This inverts the typical SPA power dynamic -- the
   server, which has the full picture, makes the final call.

2. **`HX-Retarget` and `HX-Reswap` solve the "one form, two destinations"
   problem.** A form can target `#task-list` for success and `#form-errors`
   for validation failure, with the server deciding which path to take at
   runtime. This is more flexible than `response-targets` because the
   decision can depend on arbitrary server-side logic, not just HTTP status
   codes.

3. **`HX-Push-Url` creates history entries; `HX-Replace-Url` updates in
   place.** Use push for navigation (new pages, new resources). Use replace
   for state changes within a page (filters, sorts, tab switches). The
   distinction matters for the back button -- users should not have to undo
   every filter change to get back to the previous page.

4. **`HX-Location` is faster than `HX-Redirect` for in-app navigation.**
   `HX-Redirect` reloads the entire page. `HX-Location` makes an HTMX
   request and swaps only the content area. Use redirect for login flows
   and session transitions. Use location for everything else.

5. **The three trigger timings cover every use case.** `HX-Trigger` fires
   before the swap (for notifications and counters). `HX-Trigger-After-Swap`
   fires after content enters the DOM (for querying new elements).
   `HX-Trigger-After-Settle` fires after CSS transitions complete (for
   scrolling and measuring). Choose the timing that matches your listener's
   needs.

6. **Navigation headers cancel everything else.** If `HX-Redirect`,
   `HX-Location`, or `HX-Refresh` is present, HTMX skips the swap,
   ignores target/swap overrides, and does not fire trigger events. Do not
   combine navigation headers with swap headers -- it is a common mistake
   that leads to confusing bugs.

7. **A typed helper module catches header typos at compile time.** Wrapping
   `wisp.set_header` calls in named functions means `htmx_headers.retarget`
   will autocomplete in your editor and fail at compile time if misspelled.
   Raw strings like `"HX-Retaregt"` (note the typo) compile fine and fail
   silently at runtime.

8. **Pipe composition makes complex responses readable.** Gleam's `|>`
   operator lets you chain header functions naturally:
   `wisp.html_response(body, 201) |> push_url(url) |> trigger(event)`.
   Each line adds one piece of server-driven behaviour. The response builds
   up incrementally, and the intent is clear.

9. **Use `HX-Refresh` as an escape hatch, not a routine tool.** A full page
   reload is expensive and disruptive. Reserve it for session invalidation,
   major layout changes after deployments, and "something went very wrong"
   recovery. If you find yourself reaching for `HX-Refresh` regularly,
   reconsider your approach -- there is almost always a more targeted solution
   with the other headers.

---

## What's Next

You now have the complete toolkit for server-driven control. The
`htmx_headers.gleam` module gives you typed, composable functions for every
HTMX response header. You know when to push vs. replace URLs, when to
redirect vs. locate, and when to trigger events at each phase of the swap
lifecycle. These are the tools you will reach for in every remaining chapter
of this course.

In **[Chapter 21 -- Inline Editing](21-inline-editing.md)**, we will use several of these headers
together. Inline editing is the pattern where a user clicks on a piece of
text and it transforms into an editable input -- right there in the page, no
modal, no separate form. The server returns the edit form when the user
clicks, and when they save, it returns the updated display. We will use
`HX-Retarget` to direct save errors back to the edit form, `HX-Trigger` to
refresh related counters, and `HX-Trigger-After-Settle` to focus the input
field automatically. It is a pattern that ties together everything we have
learned about swaps, targets, and server-driven control.

Inline editing is one of the most satisfying patterns to build with HTMX.
It feels like magic to the user -- but behind the scenes, it is just HTML
fragments, response headers, and a server that knows what it wants.
