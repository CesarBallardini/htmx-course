# Chapter 12 -- Cookies, Sessions, and Authentication

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what cookies are and how browsers send them with every request.
- Describe the difference between plain cookies and signed cookies, and why signing matters.
- Use `wisp.set_cookie` and `wisp.get_cookie` to store and retrieve session data.
- Build a login page that authenticates users and creates a session.
- Write an auth middleware using `use` that protects routes from unauthenticated access.
- Understand why HTTP 302 redirects do not work as expected with AJAX requests and use the `HX-Redirect` response header instead.
- Implement a logout handler that clears the session cookie.

---

## Theory

### What Are Cookies?

Cookies are small pieces of data that a server asks the browser to store. Once stored, the browser automatically sends them back with **every subsequent request** to the same server. This happens transparently -- no JavaScript required, no special configuration.

The lifecycle looks like this:

```
Browser                                         Server
  |                                                |
  |  POST /login                                   |
  |  (username=alice, password=secret)              |
  |----------------------------------------------->|
  |                                                |
  |  200 OK                                        |
  |  Set-Cookie: session=abc123; Max-Age=86400     |
  |<-----------------------------------------------|
  |                                                |
  |  Browser stores the cookie.                    |
  |                                                |
  |  GET /boards                                   |
  |  Cookie: session=abc123    <-- automatic!      |
  |----------------------------------------------->|
  |                                                |
  |  Server reads the cookie,                      |
  |  identifies the user as Alice,                 |
  |  returns her boards.                           |
  |                                                |
  |  200 OK                                        |
  |  <div>Alice's boards...</div>                  |
  |<-----------------------------------------------|
```

The key insight: cookies are the standard mechanism for maintaining state across HTTP requests. HTTP itself is stateless -- each request is independent. Cookies give us a way to say "this request came from the same person as the last one."

A cookie has several properties:

| Property   | Example            | Purpose                                                    |
|------------|--------------------|------------------------------------------------------------|
| **Name**   | `session`          | The identifier for this cookie.                            |
| **Value**  | `abc123`           | The data stored in the cookie.                             |
| **Max-Age**| `86400`            | How long (in seconds) the cookie should live. `0` deletes it. |
| **Path**   | `/`                | Which paths the cookie applies to.                         |
| **HttpOnly**| (flag)            | Prevents JavaScript from reading the cookie.               |
| **Secure** | (flag)             | Only send the cookie over HTTPS.                           |

### Sessions

A **session** is the concept of tracking a user across multiple requests. The most common approach is cookie-based sessions:

1. The user logs in with valid credentials.
2. The server creates a session identifier and stores it in a cookie.
3. On every subsequent request, the browser sends the cookie back.
4. The server reads the cookie, looks up the session, and identifies the user.

The session cookie can contain either:

- A **session ID** that points to server-side storage (a database row, an in-memory store, etc.).
- A **signed token** that contains the user data directly, verified by a cryptographic signature.

For this chapter we will use the second approach -- signed cookies -- because Wisp has built-in support for them and they require no external storage.

### Signed Cookies

A **signed cookie** is a cookie whose value has been cryptographically signed by the server. The signature is computed using a secret key that only the server knows. When the cookie comes back in a subsequent request, the server verifies the signature before trusting the value.

This gives us two guarantees:

1. **Integrity.** If anyone (a user, a browser extension, a proxy) modifies the cookie value, the signature will not match and the server will reject it.
2. **Authenticity.** Only the server that knows the secret key can produce a valid signature.

However, signed cookies are **not encrypted**. The data inside is readable by anyone who inspects the cookie. Do not store passwords, secret tokens, or other sensitive information in signed cookies. A user ID or username is fine.

In Wisp, signed cookies use the `secret_key_base` that we have been passing to `wisp_mist.handler` since Chapter 1. Wisp uses this key to sign and verify cookie values automatically.

### HTMX and Cookies

Here is the best part: **HTMX sends cookies automatically with every request.** Since HTMX makes standard browser requests (using `XMLHttpRequest` under the hood), the browser attaches all applicable cookies -- exactly as it would for a normal page navigation or form submission.

This means there is nothing special you need to do on the HTMX side. Your `hx-get`, `hx-post`, `hx-delete`, and every other HTMX request will include the session cookie. Your auth middleware works identically for HTMX requests and regular page loads.

This is one of the great advantages of the hypermedia approach. With a JSON API + SPA architecture, you typically need to manage authentication tokens manually -- storing them in `localStorage`, attaching them to `fetch()` headers, handling token refresh logic. With HTMX, the browser's native cookie mechanism handles everything.

### HTMX and Redirects

There is one area where HTMX requires special attention: **redirects**.

When a regular browser request (a link click or a form submission) receives an HTTP 302 redirect response, the browser navigates to the new URL -- a full page load. This is the standard behavior for "login succeeded, go to the dashboard."

But when an **AJAX request** receives a 302, the browser follows the redirect **transparently** and returns the final response to the JavaScript that made the request. The HTMX-initiated request never "sees" the redirect -- it just gets the HTML from the redirect target and swaps it into the current page. The URL bar does not change, and the user stays on the same page with some swapped-in content.

This is almost never what you want for authentication flows. When a user logs in, you want a full navigation to `/boards` -- URL bar change, fresh page load, the whole thing.

HTMX provides a solution: the **`HX-Redirect`** response header. When HTMX sees this header in a response, it performs a full page navigation to the specified URL, just like the user had clicked a link:

```
Browser                                         Server
  |                                                |
  |  POST /login  (via hx-post)                    |
  |  HX-Request: true                              |
  |----------------------------------------------->|
  |                                                |
  |  200 OK                                        |
  |  HX-Redirect: /boards                          |
  |  Set-Cookie: session=abc123                    |
  |<-----------------------------------------------|
  |                                                |
  |  HTMX reads HX-Redirect header.               |
  |  Performs full page navigation to /boards.     |
  |  (URL bar updates, page reloads)               |
```

The pattern is: detect whether the request came from HTMX (by checking for the `hx-request` header), and respond accordingly:

- **HTMX request:** Return status 200 with an `HX-Redirect` header.
- **Regular request:** Return a standard HTTP 302 redirect.

This way your application works correctly whether the user submits a regular form or an HTMX-enhanced one.

### Auth Patterns

Authentication in a server-rendered application follows a well-established pattern:

1. **Login page.** A form where the user enters their credentials.
2. **Login handler.** A POST endpoint that validates credentials, creates a session cookie, and redirects to a protected page.
3. **Auth middleware.** A function that checks for a valid session cookie on every request. If the cookie is missing or invalid, it redirects to the login page.
4. **Protected routes.** Routes that require authentication -- they call the auth middleware before doing anything else.
5. **Public routes.** Routes that do not require authentication (the login page itself, a public landing page).
6. **Logout handler.** Clears the session cookie and redirects to the login page.

For this tutorial we will use hardcoded users. In a production application you would store users in a database and hash passwords with a library like `argon2`. The auth *pattern* is the same regardless of where users come from.

---

## Code Walkthrough

We are going to add authentication to the Teamwork app. By the end of this section, users will need to sign in before accessing any boards, and each board will be associated with the user who created it.

### Step 1: Define the User Type

First, we need a type to represent a user. Create a simple record with an ID, username, and password:

```gleam
pub type User {
  User(id: String, username: String, password: String)
}
```

For this tutorial we will store a small list of hardcoded users. In a real application this would come from a database:

```gleam
fn users() -> List(User) {
  [
    User(id: "1", username: "alice", password: "password123"),
    User(id: "2", username: "bob", password: "secret456"),
    User(id: "3", username: "carol", password: "letmein789"),
  ]
}
```

> **Warning:** Never hardcode passwords in production code. This is for learning purposes only. In a real application, store salted and hashed passwords using a library like `argon2` or `bcrypt`.

### Step 2: The Authenticate Function

We need a function that takes a username and password and returns the matching user, or an error if the credentials are invalid:

```gleam
import gleam/list
import gleam/result

fn authenticate(
  username: String,
  password: String,
) -> Result(User, String) {
  users()
  |> list.find(fn(user) {
    user.username == username && user.password == password
  })
  |> result.replace_error("Invalid username or password")
}
```

The `list.find` function searches through the user list and returns `Ok(user)` if a match is found, or `Error(Nil)` if no match exists. We use `result.replace_error` to swap the unhelpful `Nil` error with a descriptive message.

We also need a function to look up a user by their ID (which is what we will store in the session cookie):

```gleam
fn find_user(user_id: String) -> Result(User, String) {
  users()
  |> list.find(fn(user) { user.id == user_id })
  |> result.replace_error("User not found")
}
```

### Step 3: The Login Page

The login page is a standard HTML form. Notice that the form uses a regular `method="post"` and `action="/login"` -- not HTMX attributes. Login forms work perfectly well as plain HTML forms, and using a regular form submission ensures the browser handles the redirect correctly:

```gleam
import gleam/option.{type Option, None, Some}

fn login_page(error: Option(String)) -> Element(t) {
  html.div([attribute.class("auth-container")], [
    html.h1([], [element.text("Sign In")]),
    case error {
      Some(msg) ->
        html.div([attribute.class("error-banner")], [element.text(msg)])
      None -> element.none()
    },
    html.form(
      [
        attribute.method("post"),
        attribute.action("/login"),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label([attribute.for("username")], [element.text("Username")]),
          html.input([
            attribute.type_("text"),
            attribute.name("username"),
            attribute.id("username"),
            attribute.required(True),
            attribute.autocomplete("username"),
          ]),
        ]),
        html.div([attribute.class("form-group")], [
          html.label([attribute.for("password")], [element.text("Password")]),
          html.input([
            attribute.type_("password"),
            attribute.name("password"),
            attribute.id("password"),
            attribute.required(True),
            attribute.autocomplete("current-password"),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn btn-primary")],
          [element.text("Sign In")],
        ),
      ],
    ),
  ])
}
```

Let's walk through the interesting parts:

- **Error display.** The function takes an `Option(String)` for the error message. When the user submits invalid credentials, we re-render the login page with `Some("Invalid credentials")`. On the first visit, we pass `None` and `element.none()` renders nothing.
- **`attribute.for("username")`** connects the label to the input via the `for` attribute, which improves accessibility.
- **`attribute.name("username")`** is critical. This is the key that Wisp uses to find the field's value in the form data. Without `name`, the field is not included in the POST body.
- **`attribute.required(True)`** makes the browser enforce that the field is filled in before submitting.
- **`attribute.autocomplete("username")`** helps password managers identify the field correctly.

### Step 4: The Login Handler

The login handler receives the POST request, extracts the form data, validates credentials, and either creates a session or shows an error:

```gleam
fn handle_login(req: wisp.Request) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let username =
    list.key_find(form_data.values, "username")
    |> result.unwrap("")
  let password =
    list.key_find(form_data.values, "password")
    |> result.unwrap("")

  case authenticate(username, password) {
    Ok(user) -> {
      wisp.redirect("/boards")
      |> wisp.set_cookie(req, "session", user.id, wisp.Signed, 60 * 60 * 24)
    }

    Error(_) -> {
      let page =
        layout("Sign In -- Teamwork", login_page(Some("Invalid credentials")))
      let body = element.to_document_string(page)
      wisp.html_response(body, 401)
    }
  }
}
```

Let's trace through the success path:

1. **`use form_data <- wisp.require_form(req)`** -- Parses the request body as form data. If the content type is wrong or the body cannot be parsed, Wisp returns a 400 Bad Request automatically. The `form_data` value has a `.values` field containing a `List(#(String, String))` of field name-value pairs.

2. **`list.key_find(form_data.values, "username")`** -- Searches the key-value list for the `"username"` key. Returns `Ok(value)` if found, `Error(Nil)` if not. We unwrap with a default of `""`.

3. **`authenticate(username, password)`** -- Checks the credentials against our user list.

4. **On success:** `wisp.redirect("/boards")` creates a 303 See Other response that tells the browser to navigate to `/boards`. We then pipe it into `wisp.set_cookie`, which adds a `Set-Cookie` header to the response.

5. **`wisp.set_cookie(req, "session", user.id, wisp.Signed, 60 * 60 * 24)`** -- This is the critical line. Let us break down each argument:
   - `req` -- The original request. Wisp needs it to access the `secret_key_base` for signing.
   - `"session"` -- The cookie name.
   - `user.id` -- The cookie value. We store the user's ID, not their username or password.
   - `wisp.Signed` -- Tells Wisp to sign the cookie value. The alternative is `wisp.PlainText`, which stores the value as-is (no signature, no tamper protection).
   - `60 * 60 * 24` -- The max age in seconds. This is 24 hours. After that, the browser deletes the cookie.

6. **On failure:** We re-render the login page with an error message and return it with status 401 (Unauthorized).

### Step 5: The Auth Middleware

The auth middleware is a function that checks for a valid session cookie and, if found, passes the authenticated user to the handler. If the cookie is missing or invalid, it redirects to the login page:

```gleam
fn require_auth(
  req: wisp.Request,
  handler: fn(User) -> wisp.Response,
) -> wisp.Response {
  case wisp.get_cookie(req, "session", wisp.Signed) {
    Ok(user_id) -> {
      case find_user(user_id) {
        Ok(user) -> handler(user)
        Error(_) -> wisp.redirect("/login")
      }
    }
    Error(_) -> wisp.redirect("/login")
  }
}
```

This function follows the same callback pattern as Wisp's built-in middleware. The `handler` parameter is a function that takes a `User` and returns a response. If authentication succeeds, we call the handler with the user. If it fails, we redirect to the login page.

**`wisp.get_cookie(req, "session", wisp.Signed)`** does the reverse of `wisp.set_cookie`:

1. It looks for a cookie named `"session"` in the request headers.
2. It verifies the cryptographic signature using the `secret_key_base`.
3. If the signature is valid, it returns `Ok(value)` -- the original user ID we stored.
4. If the cookie is missing, expired, or has been tampered with, it returns `Error(Nil)`.

The `wisp.Signed` argument must match what was used in `wisp.set_cookie`. If you set the cookie as `wisp.Signed` but try to read it as `wisp.PlainText` (or vice versa), it will fail.

### Step 6: Using the Auth Middleware in Routes

Now we wire the auth middleware into our router. Public routes (login, logout) do not use it. Protected routes (boards, tasks) do:

```gleam
fn handle_request(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    // Public routes -- no auth required
    ["login"] -> {
      case req.method {
        http.Get -> {
          let page = layout("Sign In -- Teamwork", login_page(None))
          let body = element.to_document_string(page)
          wisp.html_response(body, 200)
        }
        http.Post -> handle_login(req)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }
    }

    ["logout"] -> {
      use <- wisp.require_method(req, http.Post)
      handle_logout(req)
    }

    // Protected routes -- auth required
    ["boards"] -> {
      use user <- require_auth(req)
      boards_index(req, user)
    }

    ["boards", board_id] -> {
      use user <- require_auth(req)
      board_detail(req, user, board_id)
    }

    // Root redirects to boards (which will redirect to login if needed)
    [] -> wisp.redirect("/boards")

    _ -> not_found_page()
  }
}
```

Look at how the auth middleware is used:

```gleam
["boards"] -> {
  use user <- require_auth(req)
  boards_index(req, user)
}
```

The `use user <- require_auth(req)` line says: "Call `require_auth` with the request. If it succeeds, bind the authenticated user to `user` and continue. If it fails (redirects to login), everything below this line never runs."

This is exactly the same `use` pattern we have been using for `wisp.log_request` and `wisp.require_method`. The auth middleware slots naturally into Gleam's callback-based middleware system.

### Step 7: HTMX-Aware Redirects

As discussed in the theory section, standard HTTP redirects do not work well with AJAX requests. We need a helper function that detects HTMX requests and responds with the `HX-Redirect` header instead:

```gleam
fn htmx_redirect(url: String, req: wisp.Request) -> wisp.Response {
  case list.key_find(req.headers, "hx-request") {
    Ok("true") ->
      wisp.response(200)
      |> wisp.set_header("HX-Redirect", url)
    _ -> wisp.redirect(url)
  }
}
```

This function checks the request headers for `hx-request: true`, which HTMX adds to every request it makes. If found, we return a 200 response with the `HX-Redirect` header. If not, we return a standard 302 redirect.

Use this helper anywhere you need to redirect after an HTMX-initiated action. For example, if you add an HTMX-powered login form later:

```gleam
case authenticate(username, password) {
  Ok(user) -> {
    htmx_redirect("/boards", req)
    |> wisp.set_cookie(req, "session", user.id, wisp.Signed, 60 * 60 * 24)
  }
  // ...
}
```

You should also update the auth middleware to be HTMX-aware. When an HTMX request hits a protected route without a valid session, a standard redirect would be silently followed by the AJAX call -- the login page HTML would be swapped into the current page instead of navigating to it. Use `htmx_redirect` to force a full navigation:

```gleam
fn require_auth(
  req: wisp.Request,
  handler: fn(User) -> wisp.Response,
) -> wisp.Response {
  case wisp.get_cookie(req, "session", wisp.Signed) {
    Ok(user_id) -> {
      case find_user(user_id) {
        Ok(user) -> handler(user)
        Error(_) -> htmx_redirect("/login", req)
      }
    }
    Error(_) -> htmx_redirect("/login", req)
  }
}
```

Now when a session expires mid-use and an HTMX request fires, the user is properly navigated to the login page instead of seeing broken HTML swapped in.

### Step 8: The Logout Handler

Logging out means clearing the session cookie and redirecting to the login page:

```gleam
fn handle_logout(req: wisp.Request) -> wisp.Response {
  wisp.redirect("/login")
  |> wisp.set_cookie(req, "session", "", wisp.Signed, 0)
}
```

The trick is setting the cookie's max age to `0`. This tells the browser to delete the cookie immediately. We also set the value to an empty string, though the max age of 0 is what actually clears it.

### Step 9: Showing the Current User in the Navigation

Now that we have authentication, we can personalize the navigation bar. The layout function needs to accept an optional user:

```gleam
fn layout(
  title: String,
  content: Element(t),
) -> Element(t) {
  layout_with_user(title, None, content)
}

fn layout_with_user(
  title: String,
  current_user: Option(User),
  content: Element(t),
) -> Element(t) {
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
      nav_bar(current_user),
      html.main([], [content]),
    ]),
  ])
}
```

The `nav_bar` function renders differently depending on whether a user is logged in:

```gleam
fn nav_bar(current_user: Option(User)) -> Element(t) {
  html.nav([attribute.class("nav")], [
    html.div([attribute.class("nav-left")], [
      html.a([attribute.href("/boards"), attribute.class("nav-brand")], [
        element.text("Teamwork"),
      ]),
    ]),
    html.div([attribute.class("nav-right")], case current_user {
      Some(user) -> [
        html.span([attribute.class("nav-user")], [
          element.text("Hello, " <> user.username),
        ]),
        html.form(
          [
            attribute.method("post"),
            attribute.action("/logout"),
            attribute.class("nav-logout"),
          ],
          [
            html.button(
              [attribute.type_("submit"), attribute.class("btn btn-small")],
              [element.text("Sign Out")],
            ),
          ],
        ),
      ]
      None -> [
        html.a([attribute.href("/login"), attribute.class("btn btn-small")], [
          element.text("Sign In"),
        ]),
      ]
    }),
  ])
}
```

When a user is logged in, the nav shows "Hello, alice" and a "Sign Out" button. The sign-out button is inside a form that POSTs to `/logout`. When no user is logged in, we show a "Sign In" link instead.

Notice that the logout button uses a `<form>` with `method="post"`, not a plain link. This is intentional: logging out is a state-changing operation, so it should use POST, not GET. A GET request should never change server state -- this principle prevents issues with browser prefetching and accidental logouts from link crawlers.

### Step 10: Updating Protected Pages to Pass the User

Now update the board handlers to use the authenticated user and pass it to the layout:

```gleam
fn boards_index(_req: wisp.Request, user: User) -> wisp.Response {
  let page =
    layout_with_user("Boards -- Teamwork", Some(user), html.div([], [
      html.h1([], [element.text("Your Boards")]),
      html.p([], [
        element.text("Welcome back, " <> user.username <> "!"),
      ]),
      // ... board list rendering ...
    ]))
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}
```

---

## Full Code Listing

Here is the complete updated `src/teamwork.gleam` with authentication integrated.

### `src/teamwork.gleam`

```gleam
import gleam/erlang/process
import gleam/http
import gleam/list
import gleam/option.{type Option, None, Some}
import gleam/result
import lustre/attribute
import lustre/element.{type Element}
import lustre/element/html
import mist
import wisp
import wisp/wisp_mist

// --- Types ---

pub type User {
  User(id: String, username: String, password: String)
}

// --- Data ---

fn users() -> List(User) {
  [
    User(id: "1", username: "alice", password: "password123"),
    User(id: "2", username: "bob", password: "secret456"),
    User(id: "3", username: "carol", password: "letmein789"),
  ]
}

fn authenticate(
  username: String,
  password: String,
) -> Result(User, String) {
  users()
  |> list.find(fn(user) {
    user.username == username && user.password == password
  })
  |> result.replace_error("Invalid username or password")
}

fn find_user(user_id: String) -> Result(User, String) {
  users()
  |> list.find(fn(user) { user.id == user_id })
  |> result.replace_error("User not found")
}

// --- Main ---

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

// --- Middleware ---

fn middleware(
  req: wisp.Request,
  handle_request: fn(wisp.Request) -> wisp.Response,
) -> wisp.Response {
  let req = wisp.method_override(req)
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  handle_request(req)
}

fn static_directory() -> String {
  let assert Ok(priv_directory) = wisp.priv_directory("teamwork")
  priv_directory <> "/static"
}

// --- Auth Middleware ---

fn require_auth(
  req: wisp.Request,
  handler: fn(User) -> wisp.Response,
) -> wisp.Response {
  case wisp.get_cookie(req, "session", wisp.Signed) {
    Ok(user_id) -> {
      case find_user(user_id) {
        Ok(user) -> handler(user)
        Error(_) -> htmx_redirect("/login", req)
      }
    }
    Error(_) -> htmx_redirect("/login", req)
  }
}

fn htmx_redirect(url: String, req: wisp.Request) -> wisp.Response {
  case list.key_find(req.headers, "hx-request") {
    Ok("true") ->
      wisp.response(200)
      |> wisp.set_header("HX-Redirect", url)
    _ -> wisp.redirect(url)
  }
}

// --- Router ---

fn handle_request(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    // Public routes
    ["login"] -> {
      case req.method {
        http.Get -> {
          let page = layout("Sign In -- Teamwork", login_page(None))
          let body = element.to_document_string(page)
          wisp.html_response(body, 200)
        }
        http.Post -> handle_login(req)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }
    }

    ["logout"] -> {
      use <- wisp.require_method(req, http.Post)
      handle_logout(req)
    }

    // Protected routes
    ["boards"] -> {
      use user <- require_auth(req)
      boards_index(req, user)
    }

    ["boards", board_id] -> {
      use user <- require_auth(req)
      board_detail(req, user, board_id)
    }

    // Root redirects to boards
    [] -> wisp.redirect("/boards")

    _ -> not_found_page()
  }
}

// --- Auth Handlers ---

fn handle_login(req: wisp.Request) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let username =
    list.key_find(form_data.values, "username")
    |> result.unwrap("")
  let password =
    list.key_find(form_data.values, "password")
    |> result.unwrap("")

  case authenticate(username, password) {
    Ok(user) -> {
      wisp.redirect("/boards")
      |> wisp.set_cookie(req, "session", user.id, wisp.Signed, 60 * 60 * 24)
    }

    Error(_) -> {
      let page =
        layout("Sign In -- Teamwork", login_page(Some("Invalid credentials")))
      let body = element.to_document_string(page)
      wisp.html_response(body, 401)
    }
  }
}

fn handle_logout(req: wisp.Request) -> wisp.Response {
  wisp.redirect("/login")
  |> wisp.set_cookie(req, "session", "", wisp.Signed, 0)
}

// --- Page Handlers ---

fn boards_index(_req: wisp.Request, user: User) -> wisp.Response {
  let page =
    layout_with_user(
      "Boards -- Teamwork",
      Some(user),
      html.div([attribute.class("container")], [
        html.h1([], [element.text("Your Boards")]),
        html.p([], [
          element.text("Welcome back, " <> user.username <> "!"),
        ]),
        html.p([], [
          element.text(
            "Your boards and tasks will appear here. "
            <> "This page is protected -- only authenticated users can see it.",
          ),
        ]),
      ]),
    )
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

fn board_detail(
  _req: wisp.Request,
  user: User,
  board_id: String,
) -> wisp.Response {
  let page =
    layout_with_user(
      "Board " <> board_id <> " -- Teamwork",
      Some(user),
      html.div([attribute.class("container")], [
        html.h1([], [element.text("Board " <> board_id)]),
        html.p([], [
          element.text("Viewing board " <> board_id <> " as " <> user.username),
        ]),
      ]),
    )
  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

fn not_found_page() -> wisp.Response {
  let page =
    layout("Not Found -- Teamwork", html.div([attribute.class("container")], [
      html.h1([], [element.text("404 -- Page Not Found")]),
      html.p([], [
        element.text("The page you are looking for does not exist."),
      ]),
      html.a([attribute.href("/")], [element.text("Go back home")]),
    ]))
  let body = element.to_document_string(page)
  wisp.html_response(body, 404)
}

// --- Views ---

fn login_page(error: Option(String)) -> Element(t) {
  html.div([attribute.class("auth-container")], [
    html.h1([], [element.text("Sign In")]),
    case error {
      Some(msg) ->
        html.div([attribute.class("error-banner")], [element.text(msg)])
      None -> element.none()
    },
    html.form(
      [
        attribute.method("post"),
        attribute.action("/login"),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label([attribute.for("username")], [element.text("Username")]),
          html.input([
            attribute.type_("text"),
            attribute.name("username"),
            attribute.id("username"),
            attribute.required(True),
            attribute.autocomplete("username"),
          ]),
        ]),
        html.div([attribute.class("form-group")], [
          html.label([attribute.for("password")], [element.text("Password")]),
          html.input([
            attribute.type_("password"),
            attribute.name("password"),
            attribute.id("password"),
            attribute.required(True),
            attribute.autocomplete("current-password"),
          ]),
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn btn-primary")],
          [element.text("Sign In")],
        ),
      ],
    ),
  ])
}

// --- Layout ---

fn layout(title: String, content: Element(t)) -> Element(t) {
  layout_with_user(title, None, content)
}

fn layout_with_user(
  title: String,
  current_user: Option(User),
  content: Element(t),
) -> Element(t) {
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
      nav_bar(current_user),
      html.main([], [content]),
    ]),
  ])
}

fn nav_bar(current_user: Option(User)) -> Element(t) {
  html.nav([attribute.class("nav")], [
    html.div([attribute.class("nav-left")], [
      html.a([attribute.href("/boards"), attribute.class("nav-brand")], [
        element.text("Teamwork"),
      ]),
    ]),
    html.div([attribute.class("nav-right")], case current_user {
      Some(user) -> [
        html.span([attribute.class("nav-user")], [
          element.text("Hello, " <> user.username),
        ]),
        html.form(
          [
            attribute.method("post"),
            attribute.action("/logout"),
            attribute.class("nav-logout"),
          ],
          [
            html.button(
              [attribute.type_("submit"), attribute.class("btn btn-small")],
              [element.text("Sign Out")],
            ),
          ],
        ),
      ]
      None -> [
        html.a([attribute.href("/login"), attribute.class("btn btn-small")], [
          element.text("Sign In"),
        ]),
      ]
    }),
  ])
}
```

### `priv/static/css/style.css` (additions)

Add the following rules to your existing stylesheet:

```css
/* --- Auth styles --- */

.auth-container {
  max-width: 400px;
  margin: 4rem auto;
  padding: 2rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.auth-container h1 {
  text-align: center;
  margin-bottom: 1.5rem;
}

.error-banner {
  background-color: #fee;
  color: #c00;
  padding: 0.75rem 1rem;
  border-radius: 4px;
  margin-bottom: 1rem;
  border: 1px solid #fcc;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.25rem;
  font-weight: 600;
  color: #333;
}

.form-group input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
}

.form-group input:focus {
  outline: none;
  border-color: #4a90d9;
  box-shadow: 0 0 0 2px rgba(74, 144, 217, 0.25);
}

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
}

.btn-primary {
  width: 100%;
  background-color: #4a90d9;
  color: white;
  font-weight: 600;
  margin-top: 0.5rem;
}

.btn-primary:hover {
  background-color: #357abd;
}

.btn-small {
  padding: 0.25rem 0.75rem;
  font-size: 0.875rem;
}

/* --- Nav styles --- */

.nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 1.5rem;
  background: white;
  border-bottom: 1px solid #e0e0e8;
  margin-bottom: 2rem;
}

.nav-brand {
  font-weight: 700;
  font-size: 1.25rem;
  color: #16213e;
  text-decoration: none;
}

.nav-right {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.nav-user {
  color: #666;
  font-size: 0.875rem;
}

.nav-logout {
  display: inline;
}
```

### Project Structure

```
teamwork/
├── gleam.toml
├── priv/
│   └── static/
│       └── css/
│           └── style.css
├── src/
│   └── teamwork.gleam
└── test/
    └── teamwork_test.gleam
```

### Testing It Out

Start the server with `gleam run` and walk through the following flow:

1. **Visit** `http://localhost:8000/` -- You are redirected to `/boards`, then redirected again to `/login` because you are not authenticated.
2. **Enter** username `alice` and password `password123` -- You are redirected to `/boards` and see "Welcome back, alice!" The nav bar shows "Hello, alice" and a "Sign Out" button.
3. **Open** your browser's developer tools, go to the Application tab (Chrome) or Storage tab (Firefox), and look at Cookies. You should see a `session` cookie with a signed value.
4. **Click** "Sign Out" -- You are redirected to the login page. The session cookie is cleared.
5. **Try** visiting `http://localhost:8000/boards` directly -- You are redirected to `/login` because the session is gone.
6. **Try** logging in with wrong credentials -- You see the error banner "Invalid credentials" and remain on the login page.

---

## Exercise

Now it is your turn. Extend the authentication system with the following tasks:

### Task 1: Add a Signup Page

Create a `GET /signup` route that renders a registration form with username and password fields, and a `POST /signup` handler that creates a new user. Since we do not have a database yet, store the new user in actor state (as you learned in earlier chapters) or simply validate that the username is not taken and add it to a mutable list.

After successful registration, log the user in automatically (set the session cookie) and redirect to `/boards`.

### Task 2: Show the Current User in Board Content

Update the boards page so that each board card or list item shows who created it. The authenticated `User` value is already available in your handler -- use `user.username` to display ownership information.

### Task 3: Add a "Remember Me" Checkbox

Add a checkbox to the login form labeled "Remember me". When checked, set the session cookie's max age to 30 days (`60 * 60 * 24 * 30`) instead of 24 hours. You will need to:

1. Add the checkbox input to the login form.
2. Read its value from the form data in `handle_login`.
3. Choose the max age based on whether it was checked.

### Task 4: Per-User Board Filtering

Associate each board with a user ID. Update the boards index handler so that each user only sees their own boards. When Alice logs in, she should only see Alice's boards. When Bob logs in, he should only see Bob's boards.

### Task 5: Verify HTMX Redirect Behavior

Add an HTMX-powered "Delete Board" button to the board detail page. When clicked, it should POST to `/boards/:id/delete`. In the handler, delete the board and then use `htmx_redirect` to send the user back to `/boards`. Verify in your browser's Network tab that:

- The request includes the `HX-Request: true` header.
- The response includes the `HX-Redirect: /boards` header.
- The browser performs a full navigation to `/boards` (URL bar changes).

---

## Exercise Solution Hints

If you get stuck, here are some pointers. Try to solve each task on your own before reading these.

### Hint for Task 1

The signup form is almost identical to the login form. The POST handler should:

1. Parse the form data to get username and password.
2. Check if the username is already taken (search the user list).
3. If not taken, create a new `User` with a generated ID.
4. Set the session cookie and redirect to `/boards`.

If you are using actor state from a previous chapter, send a message to add the new user. If not, you can keep a simple in-memory approach -- just remember that the data resets when the server restarts.

```gleam
["signup"] -> {
  case req.method {
    http.Get -> {
      let page = layout("Sign Up -- Teamwork", signup_page(None))
      let body = element.to_document_string(page)
      wisp.html_response(body, 200)
    }
    http.Post -> handle_signup(req)
    _ -> wisp.method_not_allowed([http.Get, http.Post])
  }
}
```

### Hint for Task 2

You already have the `User` value inside your handler. Just use it in the HTML:

```gleam
html.span([attribute.class("board-owner")], [
  element.text("Created by " <> user.username),
])
```

### Hint for Task 3

HTML checkboxes are only included in form data when they are checked. When unchecked, the key is absent entirely. So you check for presence:

```gleam
let remember_me =
  list.key_find(form_data.values, "remember_me")
  |> result.is_ok

let max_age = case remember_me {
  True -> 60 * 60 * 24 * 30   // 30 days
  False -> 60 * 60 * 24        // 1 day
}

wisp.redirect("/boards")
|> wisp.set_cookie(req, "session", user.id, wisp.Signed, max_age)
```

The checkbox input in the form:

```gleam
html.div([attribute.class("form-group checkbox-group")], [
  html.label([], [
    html.input([
      attribute.type_("checkbox"),
      attribute.name("remember_me"),
      attribute.value("true"),
    ]),
    element.text(" Remember me"),
  ]),
])
```

### Hint for Task 4

If your boards are stored in actor state, add a `user_id` field to the `Board` type. When listing boards, filter by the current user:

```gleam
let user_boards =
  boards
  |> list.filter(fn(board) { board.user_id == user.id })
```

### Hint for Task 5

The key insight is that `htmx_redirect` checks for the `hx-request` header. When you use `hx-post` on the delete button, HTMX sends the request with that header. Your handler detects it and returns `HX-Redirect` instead of a 302:

```gleam
["boards", board_id, "delete"] -> {
  use <- wisp.require_method(req, http.Post)
  use user <- require_auth(req)
  // ... delete the board ...
  htmx_redirect("/boards", req)
}
```

On the client side:

```gleam
html.button(
  [
    hx.post("/boards/" <> board_id <> "/delete"),
    attribute.class("btn btn-danger"),
  ],
  [element.text("Delete Board")],
)
```

Remember: signed cookies are secure against **tampering** but are not **encrypted**. Anyone can read the cookie value -- they just cannot modify it without invalidating the signature. Never store passwords or secret tokens in cookies, even signed ones. A user ID is safe because knowing the ID alone does not grant access without a valid signature.

---

## Key Takeaways

1. **Cookies are the standard mechanism for web sessions.** The server sets a cookie with `Set-Cookie`, and the browser sends it back with every subsequent request. No JavaScript involved.

2. **Signed cookies prevent tampering.** Wisp signs cookie values using the `secret_key_base`. If anyone modifies the cookie, the signature check fails and Wisp rejects it. Use `wisp.Signed` for session cookies.

3. **HTMX sends cookies automatically.** Because HTMX uses standard browser requests, cookies work exactly as they do for regular page loads and form submissions. No special token management needed.

4. **Use `HX-Redirect` for HTMX-initiated redirects.** Standard HTTP 302 redirects are followed transparently by AJAX requests -- the browser swaps the redirected content into the page instead of navigating. The `HX-Redirect` response header tells HTMX to perform a full page navigation.

5. **Auth middleware is just another `use` callback.** The pattern `use user <- require_auth(req)` fits naturally into Gleam's middleware system. If auth fails, the handler body never runs.

6. **Separate public and protected routes.** The login page and logout endpoint must be accessible without a session. Everything else goes through `require_auth`.

7. **Logout means setting max age to 0.** `wisp.set_cookie(req, "session", "", wisp.Signed, 0)` tells the browser to delete the cookie immediately.

8. **Check `hx-request` to detect HTMX requests.** HTMX adds an `hx-request: true` header to every request. Use this to decide between `HX-Redirect` (for HTMX) and a standard 302 redirect (for regular requests).

9. **Never store passwords in plain text in production.** Our hardcoded users are for learning only. In a real application, use `argon2` or `bcrypt` to hash passwords, and store users in a database.

10. **Logout should use POST, not GET.** State-changing operations belong behind POST requests. A GET-based logout can be triggered accidentally by browser prefetching, link crawlers, or cached pages.

---

## What's Next

This chapter concludes the intermediate phase. You now have a Teamwork app with routing, HTMX interactivity, actor-based state, multiple boards, and cookie-based authentication. In the [advanced phase](../03-advanced/13-real-time-with-sse.md), we will add a real database with SQLite, server-sent events for real-time updates, and deploy the application to production.
