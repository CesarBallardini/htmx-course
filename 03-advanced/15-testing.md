# Chapter 15 -- Testing HTMX Applications

**Phase:** Advanced
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why HTMX applications are **easier to test** than typical SPA frontends.
- Describe the **testing pyramid** for hypermedia-driven applications.
- Use **`gleeunit`** to write and run tests for a Gleam project.
- Use the **`wisp/simulate`** module to simulate HTTP requests without starting a real server.
- Write **unit tests** for pure functions like validation and data transformation.
- Write **integration tests** that send requests to your router and assert on status codes, HTML fragments, and response headers.
- Verify that **out-of-band (OOB) swap elements** are present in responses.

---

## 1. Theory

### 1.1 Why HTMX Apps Are Easy to Test

Testing a single-page application built with React, Vue, or Svelte is hard. You
have to deal with client-side state, component lifecycles, mock stores, fake
timers, virtual DOM snapshots, and a testing library that changes best practices
every eighteen months.

HTMX eliminates most of that complexity:

> **In an HTMX application, the server is the single source of truth. It
> receives HTTP requests and returns HTML. Testing means sending a request and
> checking the HTML that comes back.**

There is no client-side state to mock. There is no virtual DOM to snapshot.
Your server is a function from requests to responses. You test it by calling
that function.

You do not need to test HTMX itself -- the library maintainers do that. You
need to test that your server returns the right HTML, with the right status
codes, and with the right headers. That is ordinary HTTP testing, and every
language has mature tools for it.

### 1.2 The Testing Pyramid for HTMX Apps

```
              /  E2E Tests  \           Few, slow, fragile
             / (Playwright)  \          Test the full browser experience
            /─────────────────\
           / Integration Tests \        Many, fast, reliable
          / (wisp/simulate +    \       Test request -> response cycle
         /   router handlers)    \
        /─────────────────────────\
       /       Unit Tests          \    Many, fastest, most reliable
      /  (pure functions: validation, \  Test logic in isolation
     /    rendering, data transforms)  \
    /───────────────────────────────────\
```

**Unit tests** -- Test pure functions: validation logic, data transformations,
HTML rendering. No network, no state. They run in microseconds.

**Integration tests** -- Test request handlers with `wisp/simulate`. Construct
an HTTP request, pass it through your router, assert on the response. These
exercise routing, middleware, state management, and rendering together. Still
fast -- no real HTTP server involved.

**End-to-end tests** -- Test the full experience in a real browser with tools
like Playwright. Slowest and most fragile, but catch problems integration tests
cannot (missing HTMX attributes, CSS issues, JavaScript errors). Use sparingly.

For HTMX applications, integration tests give the most value. They are fast,
they test the real request-response cycle, and they catch most bugs.

### 1.3 `gleeunit` -- Gleam's Test Framework

Gleam ships with `gleeunit` as the standard test framework. The rules:

1. Test files live in the `test/` directory with names ending in `_test.gleam`.
2. Each test is a public function whose name ends with `_test`.
3. Tests use `should` assertions from `gleeunit/should`.
4. Run all tests with `gleam test`.

The `should` module provides these assertions:

| Function            | Checks that...                              |
|---------------------|---------------------------------------------|
| `should.equal`      | Two values are equal                        |
| `should.not_equal`  | Two values are not equal                    |
| `should.be_true`    | A `Bool` is `True`                          |
| `should.be_false`   | A `Bool` is `False`                         |
| `should.be_ok`      | A `Result` is `Ok(_)`                       |
| `should.be_error`   | A `Result` is `Error(_)`                    |
| `should.fail`       | Always fails (marks incomplete tests)       |

That is the entire API. No test suites, no setup/teardown hooks, no runners to
configure. The simplicity is the point.

### 1.4 `wisp/simulate` -- Simulating HTTP Requests

The `wisp/simulate` module constructs HTTP requests and inspects responses
without starting a real server. The key functions:

| Function                                 | Produces                              |
|------------------------------------------|---------------------------------------|
| `simulate.request(method, path)`         | A request with the given HTTP method  |
| `simulate.form_body(request, data)`      | Adds url-encoded form data to request |
| `simulate.string_body(request, text)`    | Adds a string body to request         |
| `simulate.header(request, name, value)`  | Adds a header to request              |
| `simulate.read_body(response)`           | Extracts the response body as String  |

These functions are designed for chaining with the pipe operator. For example,
to send a POST with form data:

```gleam
simulate.request(http.Post, "/tasks")
|> simulate.form_body([#("title", "New task")])
```

### 1.5 What to Test in an HTMX Application

A practical checklist:

- **Routes return correct status codes.** GET `/` returns 200. POST `/tasks`
  with valid data returns 201. Invalid data returns 422. Unknown paths return 404.
- **HTML fragments contain expected content.** Use `string.contains` to check
  for task titles, error messages, and other key content.
- **Validation rejects bad input.** Empty titles produce 422 with error messages.
- **Auth middleware blocks unauthenticated requests.** Protected routes without
  a session cookie return 302.
- **CRUD operations modify state.** After creating a task, the list includes it.
  After deleting, it does not.
- **OOB elements are present.** Responses that update counters must contain
  `hx-swap-oob` and the target element ID.
- **Wrong HTTP methods are rejected.** A PUT to a GET-only route returns 405.

Notice what is *not* on this list: testing that HTMX swaps HTML into the DOM.
That is HTMX's job. You test the server.

---

## 2. Code Walkthrough

After fourteen chapters of building features, we have a feature-complete app
with zero tests. Time to fix that.

### Step 1 -- Setting Up the Test File

Create `test/teamwork_test.gleam`:

```gleam
import gleam/http
import gleam/string
import gleeunit
import gleeunit/should
import wisp/simulate
import teamwork/state
import teamwork/web.{type Context, Context}
import teamwork

pub fn main() {
  gleeunit.main()
}
```

### Step 2 -- A Setup Helper

Every test needs a fresh context. If tests shared state, a task created in one
test might appear in another.

```gleam
fn setup_context() -> Context {
  let assert Ok(task_store) = state.start()
  Context(tasks: task_store)
}
```

Each call creates a new BEAM actor with an empty task list. Creating a process
is cheap -- microseconds -- so spinning up a fresh actor per test is fine.

### Step 3 -- Testing a GET Route

```gleam
pub fn home_page_test() {
  let ctx = setup_context()

  let response =
    simulate.request(http.Get, "/")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)

  response
  |> simulate.read_body
  |> should.not_equal("")
}
```

The flow: build a request with `simulate.request`, pass it through `handle_request`,
assert on the response. This catches routing regressions, broken imports, and
crash-on-render bugs.

### Step 4 -- Testing Task Creation (POST with form data)

```gleam
pub fn create_task_test() {
  let ctx = setup_context()

  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "New task")])
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(201)

  let body = simulate.read_body(response)
  string.contains(body, "New task") |> should.be_true()
}
```

The `simulate.form_body` function encodes key-value pairs as
`application/x-www-form-urlencoded` -- identical to what a browser sends. We
use `string.contains` rather than an exact match so the test survives HTML
refactors. What matters is that the task title appears in the response.

### Step 5 -- Testing Validation (empty title returns 422)

```gleam
pub fn create_task_empty_title_test() {
  let ctx = setup_context()

  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "")])
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(422)

  let body = simulate.read_body(response)
  string.contains(body, "required") |> should.be_true()
}
```

Status 422 "Unprocessable Entity" is the correct code for validation failures.
We check for "required" in the body -- if the error is later rephrased from
"Title is required" to "A title is required", the test still passes.

### Step 6 -- Testing Delete

Deleting requires setup: create a task first, then delete it.

```gleam
pub fn delete_task_test() {
  let ctx = setup_context()

  // Create a task so there is something to delete.
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "To delete")])
    |> teamwork.handle_request(ctx)

  // Delete it.
  let response =
    simulate.request(http.Delete, "/tasks/1")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
}
```

For higher confidence, verify the side effect -- the task is gone from the list:

```gleam
pub fn delete_task_removes_from_list_test() {
  let ctx = setup_context()

  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "To delete")])
    |> teamwork.handle_request(ctx)

  let _ =
    simulate.request(http.Delete, "/tasks/1")
    |> teamwork.handle_request(ctx)

  let response =
    simulate.request(http.Get, "/tasks")
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "To delete") |> should.be_false()
}
```

### Step 7 -- Testing OOB Elements in Responses

OOB elements are easy to break during refactoring. A developer might
restructure the task HTML and forget the counter update.

```gleam
pub fn create_task_includes_oob_counter_test() {
  let ctx = setup_context()

  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "OOB test")])
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "hx-swap-oob") |> should.be_true()
  string.contains(body, "task-counter") |> should.be_true()
}
```

### Step 8 -- Testing Auth Middleware

```gleam
pub fn protected_route_requires_auth_test() {
  let ctx = setup_context()

  let response =
    simulate.request(http.Get, "/boards")
    |> teamwork.handle_request(ctx)

  // No session cookie -> redirect to login.
  response.status |> should.equal(302)
}

pub fn authenticated_user_can_access_boards_test() {
  let ctx = setup_context()

  let response =
    simulate.request(http.Get, "/boards")
    |> simulate.header("cookie", "session=valid-session-token")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
}
```

### Step 9 -- Testing Pure Functions (validation)

Create `test/teamwork/validation_test.gleam`:

```gleam
import gleam/string
import gleeunit
import gleeunit/should
import teamwork/validation

pub fn main() {
  gleeunit.main()
}

pub fn validate_empty_title_test() {
  validation.validate_task_title("")
  |> should.be_error()
}

pub fn validate_valid_title_test() {
  validation.validate_task_title("A valid task title")
  |> should.be_ok()
}

pub fn validate_short_title_test() {
  validation.validate_task_title("Ab")
  |> should.be_error()
}

pub fn validate_title_at_minimum_length_test() {
  validation.validate_task_title("Abc")
  |> should.be_ok()
}

pub fn validate_whitespace_only_title_test() {
  validation.validate_task_title("   ")
  |> should.be_error()
}

pub fn validate_very_long_title_test() {
  let long_title = string.repeat("a", 300)
  validation.validate_task_title(long_title)
  |> should.be_error()
}
```

These tests run in microseconds. Notice the boundary tests: `"Ab"` fails,
`"Abc"` passes. Boundary conditions are where bugs hide.

### Step 10 -- Running the Tests

```shell
gleam test
```

Output looks like this:

```
Compiled in 0.42s
Running teamwork_test.main
Running teamwork/validation_test.main
.................

Finished in 0.031s
17 tests, 0 failures
```

A failing test shows a clear message:

```
FAIL: create_task_empty_title_test

  Expected: 422
  Got:      400

  test/teamwork_test.gleam:42
```

### Project Structure

Mirror your source files in your test directory:

```
teamwork/
├── gleam.toml
├── src/
│   └── teamwork/
│       ├── state.gleam
│       ├── validation.gleam
│       └── web.gleam
├── test/
│   ├── teamwork_test.gleam           <-- integration tests (router)
│   └── teamwork/
│       ├── validation_test.gleam     <-- unit tests (validation)
│       └── state_test.gleam          <-- unit tests (state actor)
```

---

## 3. Full Code Listing

### `test/teamwork_test.gleam`

```gleam
import gleam/http
import gleam/string
import gleeunit
import gleeunit/should
import wisp/simulate
import teamwork/state
import teamwork/web.{type Context, Context}
import teamwork

pub fn main() {
  gleeunit.main()
}

fn setup_context() -> Context {
  let assert Ok(task_store) = state.start()
  Context(tasks: task_store)
}

// --- Home page ---

pub fn home_page_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
  response
  |> simulate.read_body
  |> should.not_equal("")
}

pub fn home_page_contains_heading_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/")
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "Teamwork") |> should.be_true()
}

// --- Task creation ---

pub fn create_task_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "New task")])
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(201)
  let body = simulate.read_body(response)
  string.contains(body, "New task") |> should.be_true()
}

pub fn create_task_empty_title_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "")])
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(422)
  let body = simulate.read_body(response)
  string.contains(body, "required") |> should.be_true()
}

pub fn create_task_whitespace_title_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "   ")])
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(422)
}

pub fn create_task_includes_oob_counter_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "OOB test")])
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "hx-swap-oob") |> should.be_true()
  string.contains(body, "task-counter") |> should.be_true()
}

// --- Task listing ---

pub fn list_tasks_empty_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/tasks")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
}

pub fn list_tasks_after_creation_test() {
  let ctx = setup_context()
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "Task Alpha")])
    |> teamwork.handle_request(ctx)
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "Task Beta")])
    |> teamwork.handle_request(ctx)

  let response =
    simulate.request(http.Get, "/tasks")
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "Task Alpha") |> should.be_true()
  string.contains(body, "Task Beta") |> should.be_true()
}

// --- Task deletion ---

pub fn delete_task_test() {
  let ctx = setup_context()
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "To delete")])
    |> teamwork.handle_request(ctx)

  let response =
    simulate.request(http.Delete, "/tasks/1")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
}

pub fn delete_task_removes_from_list_test() {
  let ctx = setup_context()
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "To delete")])
    |> teamwork.handle_request(ctx)
  let _ =
    simulate.request(http.Delete, "/tasks/1")
    |> teamwork.handle_request(ctx)

  let response =
    simulate.request(http.Get, "/tasks")
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "To delete") |> should.be_false()
}

// --- Routing ---

pub fn not_found_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/this-does-not-exist")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(404)
}

pub fn tasks_put_not_allowed_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Put, "/tasks")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(405)
}

// --- Auth middleware ---

pub fn protected_route_requires_auth_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/boards")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(302)
}

pub fn authenticated_user_can_access_boards_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/boards")
    |> simulate.header("cookie", "session=valid-session-token")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
}
```

### `test/teamwork/validation_test.gleam`

```gleam
import gleam/string
import gleeunit
import gleeunit/should
import teamwork/validation

pub fn main() {
  gleeunit.main()
}

pub fn validate_empty_title_test() {
  validation.validate_task_title("")
  |> should.be_error()
}

pub fn validate_valid_title_test() {
  validation.validate_task_title("A valid task title")
  |> should.be_ok()
}

pub fn validate_short_title_test() {
  validation.validate_task_title("Ab")
  |> should.be_error()
}

pub fn validate_title_at_minimum_length_test() {
  validation.validate_task_title("Abc")
  |> should.be_ok()
}

pub fn validate_whitespace_only_title_test() {
  validation.validate_task_title("   ")
  |> should.be_error()
}

pub fn validate_very_long_title_test() {
  let long_title = string.repeat("a", 300)
  validation.validate_task_title(long_title)
  |> should.be_error()
}

pub fn validate_title_with_special_characters_test() {
  validation.validate_task_title("Fix bug #42 -- urgent!")
  |> should.be_ok()
}

pub fn validate_title_with_leading_whitespace_test() {
  validation.validate_task_title("  Valid after trimming  ")
  |> should.be_ok()
}
```

### `test/teamwork/state_test.gleam`

```gleam
import gleam/list
import gleeunit
import gleeunit/should
import teamwork/state

pub fn main() {
  gleeunit.main()
}

pub fn start_with_empty_state_test() {
  let assert Ok(store) = state.start()
  let tasks = state.get_all(store)
  tasks |> should.equal([])
}

pub fn add_task_test() {
  let assert Ok(store) = state.start()
  state.add(store, "Test task")
  let tasks = state.get_all(store)
  list.length(tasks) |> should.equal(1)
}

pub fn add_multiple_tasks_test() {
  let assert Ok(store) = state.start()
  state.add(store, "Task one")
  state.add(store, "Task two")
  state.add(store, "Task three")
  let tasks = state.get_all(store)
  list.length(tasks) |> should.equal(3)
}

pub fn delete_task_test() {
  let assert Ok(store) = state.start()
  let id = state.add(store, "To delete")
  state.delete(store, id)
  let tasks = state.get_all(store)
  tasks |> should.equal([])
}

pub fn delete_nonexistent_task_test() {
  let assert Ok(store) = state.start()
  state.add(store, "Keep me")
  state.delete(store, "nonexistent-id")
  let tasks = state.get_all(store)
  list.length(tasks) |> should.equal(1)
}

pub fn tasks_preserve_order_test() {
  let assert Ok(store) = state.start()
  state.add(store, "First")
  state.add(store, "Second")
  state.add(store, "Third")
  let tasks = state.get_all(store)
  let titles = list.map(tasks, fn(t) { t.title })
  titles |> should.equal(["First", "Second", "Third"])
}
```

---

## 4. Exercise

Write a comprehensive test suite for the Teamwork application:

### Task 1 -- Complete CRUD Tests

Write tests for all four CRUD operations:

- **Create**: Valid data returns 201 with task title in body. Empty title
  returns 422. Whitespace-only title returns 422.
- **Read**: Task list returns 200. After creating tasks, the list contains them.
- **Update**: PATCHing with a new title returns 200 and the updated title in
  the response. PATCHing with an empty title returns 422.
- **Delete**: Deleting an existing task returns 200 and removes it from the
  list. Deleting a nonexistent task returns 404.

### Task 2 -- Validation Edge Cases

Write tests for the validation module covering: empty string, whitespace-only,
below minimum length, exactly at minimum length (should pass), above maximum
length, special characters, and strings with leading/trailing whitespace.

### Task 3 -- Auth Middleware

Write tests for:

- Unauthenticated GET to `/boards` returns 302.
- Authenticated GET to `/boards` returns 200.
- The login page (`/login`) is accessible without authentication.

### Task 4 -- Search and Filter

If your app has `GET /tasks?q=keyword`, write tests for: matching queries
returning results, non-matching queries returning empty, and empty queries
returning all tasks.

### Task 5 -- One Test Per Route

Review your router and ensure every route has at least one happy-path test and
one error-path test.

---

## 5. Exercise Solution Hints

### Hint for Task 1

For update tests, create a task first, then PATCH it:

```gleam
pub fn update_task_test() {
  let ctx = setup_context()
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "Original")])
    |> teamwork.handle_request(ctx)

  let response =
    simulate.request(http.Patch, "/tasks/1")
    |> simulate.form_body([#("title", "Updated")])
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
  let body = simulate.read_body(response)
  string.contains(body, "Updated") |> should.be_true()
}
```

For deleting a nonexistent task:

```gleam
pub fn delete_nonexistent_task_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Delete, "/tasks/99999")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(404)
}
```

### Hint for Task 2

Test the boundaries. If the maximum length is 200 characters:

```gleam
pub fn validate_title_at_max_length_test() {
  let title = string.repeat("a", 200)
  validation.validate_task_title(title)
  |> should.be_ok()
}

pub fn validate_title_over_max_length_test() {
  let title = string.repeat("a", 201)
  validation.validate_task_title(title)
  |> should.be_error()
}
```

### Hint for Task 3

```gleam
pub fn login_page_accessible_test() {
  let ctx = setup_context()
  let response =
    simulate.request(http.Get, "/login")
    |> teamwork.handle_request(ctx)

  response.status |> should.equal(200)
  let body = simulate.read_body(response)
  string.contains(body, "login") |> should.be_true()
}
```

### Hint for Task 4

Create tasks with known titles, then search:

```gleam
pub fn search_tasks_matching_test() {
  let ctx = setup_context()
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "Buy groceries")])
    |> teamwork.handle_request(ctx)
  let _ =
    simulate.request(http.Post, "/tasks")
    |> simulate.form_body([#("title", "Write report")])
    |> teamwork.handle_request(ctx)

  let response =
    simulate.request(http.Get, "/tasks?q=Buy")
    |> teamwork.handle_request(ctx)

  let body = simulate.read_body(response)
  string.contains(body, "Buy groceries") |> should.be_true()
  string.contains(body, "Write report") |> should.be_false()
}
```

### Hint for Task 5

List every route in your router. For each one, write the expected status code
for the happy path and at least one error path. A typical Teamwork app has
about 8 routes, giving you 16 or more tests as a solid baseline.

---

## 6. Key Takeaways

1. **HTMX applications are tested on the server.** No client-side state to
   mock, no component tree to render. Send HTTP requests, check HTML responses.

2. **The testing pyramid still applies.** Many unit tests for pure functions.
   Many integration tests for request handlers. A few E2E tests for critical
   flows. Integration tests give the most value per line of test code.

3. **`gleeunit` keeps testing simple.** Tests are functions ending in `_test`.
   Assertions use `should.equal`, `should.be_true`, `should.be_ok`. Run with
   `gleam test`. No configuration, no plugins.

4. **`wisp/simulate` simulates HTTP without a server.** Build requests with
   `simulate.request`. Send form data with `simulate.form_body`.
   Extract bodies with `simulate.read_body`. Everything runs in-process.

5. **Create a fresh context for every test.** A setup helper that starts a
   new state actor prevents tests from leaking state into each other. BEAM
   actors are cheap -- do not reuse them across tests.

6. **Test HTML with `string.contains`, not exact matches.** Loose assertions
   survive refactors. Tight assertions break on every CSS class rename.

7. **Test both the happy path and the error path.** Valid data expects 201,
   invalid expects 422. Protected routes tested with and without auth. If you
   only test the happy path, broken validation goes undetected.

8. **Mirror your source structure in your test structure.** Integration tests
   in `test/teamwork_test.gleam`. Unit tests for `src/teamwork/validation.gleam`
   in `test/teamwork/validation_test.gleam`.

9. **Tests are a refactoring safety net.** The app is feature-complete. A solid
   test suite means you can refactor with confidence -- `gleam test` will tell
   you within seconds if something broke.

---

## What's Next

In [Chapter 16](16-extensions-and-patterns.md), we will explore **HTMX extensions and advanced patterns** --
adding response-targets for differentiated error handling, preloading for
instant navigation, infinite scroll, smart polling, and a first look at
`_hyperscript` for client-side micro-interactions. The Teamwork board gains
the polish that separates a prototype from a real application.
