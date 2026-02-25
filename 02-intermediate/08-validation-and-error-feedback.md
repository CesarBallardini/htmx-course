# Chapter 8 -- Validation and Error Feedback

**Phase:** Intermediate
**Project:** "Teamwork" -- a collaborative task board

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why server-side validation is mandatory and client-side validation is optional.
- Use Gleam's `Result` type to express validation success and failure.
- Build a reusable validation module with typed error variants.
- Return HTTP `422 Unprocessable Entity` responses when validation fails.
- Describe how HTMX handles non-2xx responses and why this behaviour is useful.
- Re-render a form with error messages and preserved user input on validation failure.
- Style error states with CSS so users immediately see what went wrong.

---

## 1. Theory

### 1.1 Why Validate on the Server?

In [Chapter 7](07-forms-and-user-input.md) we added a form that lets users create tasks with `hx-post`. It
works -- but it also accepts anything. An empty title? Sure. A single character?
Fine. A 10,000-character string? No problem. That is a problem.

You might be tempted to solve this on the client side. HTML5 gives you
`required`, `minlength`, `maxlength`, and `pattern` attributes. They are easy
to add and they provide instant feedback. But they are **not security**. Anyone
can open the browser's developer tools, remove the `required` attribute, and
submit the form. Anyone can use `curl` to send a POST request with whatever
data they like. Client-side validation is a convenience for honest users. It
does not protect your application.

The rule is simple:

> **Always validate on the server.** Client-side validation is a UX enhancement,
> not a substitute.

In this chapter we will build server-side validation using Gleam's type system.
Once the server-side logic is solid, you are free to layer client-side
attributes on top for a snappier user experience -- but the server remains the
single source of truth.

### 1.2 Validation with the Result Type

Gleam's `Result` type is purpose-built for operations that can succeed or fail:

```gleam
Result(value, error)
//  Ok(value)    -- success
//  Error(error) -- failure
```

For validation, the pattern is:

- `Ok(valid_data)` -- the input passed all checks, here is the cleaned value.
- `Error(list_of_errors)` -- the input failed one or more checks, here is what
  went wrong.

Returning a `List` of errors (rather than a single error) lets you report
everything that is wrong at once. Users hate fixing one error, resubmitting,
and discovering another. Show them all the problems up front.

### 1.3 HTTP Status Codes for Validation Errors

When the server rejects input, it needs to communicate that through the HTTP
status code. The relevant codes are:

| Code  | Name                    | When to Use                                          |
|-------|-------------------------|------------------------------------------------------|
| `200` | OK                      | The request succeeded.                               |
| `201` | Created                 | A new resource was successfully created.             |
| `400` | Bad Request             | The request is malformed (missing fields, bad JSON). |
| `422` | Unprocessable Entity    | The request is well-formed but the data is invalid.  |
| `500` | Internal Server Error   | Something unexpected broke on the server.            |

The distinction between 400 and 422 matters. A `400` means "I could not even
parse your request." A `422` means "I understood your request, but the data
does not pass validation." For form submissions where the data arrives
correctly but fails business rules (empty title, title too short, etc.),
**422 Unprocessable Entity** is the right choice.

### 1.4 How HTMX Handles Non-2xx Responses

Here is the key insight that makes server-side validation work beautifully
with HTMX:

> **By default, HTMX swaps the response body into the target element
> regardless of the HTTP status code.**

Read that again. When your server returns a `422` with an HTML fragment, HTMX
does not throw an error or show a generic message. It takes the HTML from the
response body and swaps it into the target, exactly the same way it would for
a `200`. This is not a bug -- it is a design decision, and it is exactly what
we want.

The pattern becomes elegant:

1. **On success (200 or 201):** Return the new content -- a fresh empty form,
   an updated task list, whatever is appropriate.
2. **On validation failure (422):** Return the form re-rendered with error
   messages and the user's original input preserved.

HTMX swaps in either response. The user sees either a success state or error
messages, and we did not write a single line of JavaScript to make it happen.

If you ever need different behaviour based on status codes -- swapping errors
into a different element, for example -- HTMX provides the `response-targets`
extension. We will cover that in [Chapter 16](../03-advanced/16-extensions-and-patterns.md). For now, the default behaviour is
exactly what we need.

### 1.5 The Validation Flow

Here is the complete lifecycle of a form submission with validation:

```
  User fills out form and clicks "Add Task"
    |
    v
  Browser sends POST /tasks (via hx-post)
    |
    v
  Server receives form data
    |
    v
  Server validates the data
    |
    +--- Valid -----> Create the task
    |                   |
    |                   v
    |                 Return 201 with fresh empty form
    |                   |
    |                   v
    |                 HTMX swaps in the empty form (ready for next task)
    |
    +--- Invalid ---> Return 422 with form + error messages + preserved input
                        |
                        v
                      HTMX swaps in the form with errors displayed
                        |
                        v
                      User sees what went wrong, corrects it, resubmits
```

The server is in complete control. It decides what HTML to return based on
whether validation passes. HTMX just puts the HTML on the page.

---

## 2. Code Walkthrough

We are going to make four changes to the Teamwork project:

1. Create a validation module with typed error variants.
2. Write a form-rendering function that can display errors.
3. Update the POST handler to validate before creating a task.
4. Add CSS for error states.

### Step 1 -- Create the Validation Module

Create a new file at `src/teamwork/validation.gleam`. This module is
self-contained: it knows how to validate data and how to describe what went
wrong, but it has no knowledge of HTML or HTTP.

```gleam
// src/teamwork/validation.gleam

import gleam/int
import gleam/list
import gleam/string

/// A validation error describes what went wrong and which field it applies to.
pub type ValidationError {
  Required(field: String)
  TooShort(field: String, min: Int)
  TooLong(field: String, max: Int)
}

/// Validates a task title.
/// Returns Ok with the trimmed title, or Error with a list of problems.
pub fn validate_task_title(
  title: String,
) -> Result(String, List(ValidationError)) {
  let title = string.trim(title)
  case string.length(title) {
    0 -> Error([Required("title")])
    n if n < 3 -> Error([TooShort("title", 3)])
    n if n > 200 -> Error([TooLong("title", 200)])
    _ -> Ok(title)
  }
}

/// Converts a validation error into a human-readable message.
pub fn error_message(error: ValidationError) -> String {
  case error {
    Required(field) ->
      field <> " is required"
    TooShort(field, min) ->
      field <> " must be at least " <> int.to_string(min) <> " characters"
    TooLong(field, max) ->
      field <> " must be at most " <> int.to_string(max) <> " characters"
  }
}

/// Filters a list of errors to only those for a specific field.
pub fn errors_for(
  errors: List(ValidationError),
  field: String,
) -> List(ValidationError) {
  list.filter(errors, fn(e) {
    case e {
      Required(f) if f == field -> True
      TooShort(f, _) if f == field -> True
      TooLong(f, _) if f == field -> True
      _ -> False
    }
  })
}
```

Let's break down the key design decisions.

**Typed error variants.** Each variant of `ValidationError` carries enough
context to produce a clear message: which field failed and what the constraint
was. If you later add a `MaxValue` or `InvalidFormat` variant, you add it to
the type and update `error_message`. The compiler will tell you if you forget
a case.

**Trimming before validation.** The `validate_task_title` function trims
whitespace before checking length. This means `"   "` (three spaces) is treated
as empty, and `" Hi "` is treated as `"Hi"` (length 2, which fails the minimum
check). The `Ok` branch returns the trimmed string, so downstream code always
works with clean data.

**Returning a list of errors.** Even though `validate_task_title` currently
returns at most one error (it checks in order and returns early), the return
type is `List(ValidationError)`. This is forward-looking: when we add more
fields with their own validations, we can combine them into a single error list.

**The `errors_for` helper.** When rendering a form, you need to know which
errors apply to which field. This function filters the error list by field name,
making it easy to show errors next to the right input.

Note that the import for `gleam/list` is needed in `errors_for`. Here is the
complete import section for the module:

```gleam
import gleam/int
import gleam/list
import gleam/string
```

### Step 2 -- Render the Form with Error Support

Now we update the form-rendering function. In [Chapter 7](07-forms-and-user-input.md) we had a simple form
that always looked the same. Now it needs to handle three states:

1. **Initial render** -- empty form, no errors.
2. **Validation failure** -- form with error messages and the user's previous
   input preserved.
3. **Success** -- back to the empty form (errors cleared, input cleared).

We achieve this with two parameters: an error list and a values list.

```gleam
import gleam/list
import gleam/result
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import hx

import teamwork/validation.{type ValidationError}

fn add_task_form(
  errors: List(ValidationError),
  values: List(#(String, String)),
) -> Element(t) {
  // Look up the current value for the title field (empty string if not found).
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  // Filter errors to only those relevant to the title field.
  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#form-container")),
        hx.swap(hx.OuterHTML),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("title")],
            [element.text("Task title")],
          ),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.value(title_value),
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
          ..list.map(title_errors, fn(e) {
            html.span(
              [attribute.class("error-text")],
              [element.text(validation.error_message(e))],
            )
          })
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

There are several things worth examining here.

**The `values` parameter.** This is a list of key-value pairs representing the
user's submitted form data. On initial render, we pass an empty list and every
field starts blank. On validation failure, we pass the submitted values back so
the user does not have to retype everything. The `list.key_find(values, "title")`
call looks up the value by field name, and `result.unwrap("")` provides a
default empty string if the key is not found.

**Conditional CSS class.** The input gets the class `"input"` when there are no
errors, or `"input input-error"` when there are errors for this field. This is
a straightforward `case` expression -- no special conditional-rendering
framework needed.

```gleam
attribute.class(case title_errors {
  [] -> "input"
  _ -> "input input-error"
})
```

**Error messages as a spread list.** The `..list.map(title_errors, ...)` at the
end of the `form-group` div's children list uses Gleam's spread syntax. If
there are no errors, `list.map` returns an empty list and no error spans are
rendered. If there are errors, each one becomes a `<span class="error-text">`
with the human-readable message.

**The `hx-target` and `hx-swap` attributes.** The form targets
`#form-container` with `OuterHTML` swap. This means the server's response will
replace the entire `#form-container` div -- including the div itself. This is
important because the response includes the new `#form-container` wrapper.
Whether the response is a clean form (success) or a form with errors (failure),
it replaces what was there before.

### Step 3 -- Update the POST Handler

The POST handler is where validation actually happens. It reads the form data,
runs validation, and returns either a success response or an error response.

```gleam
import wisp

import teamwork/validation

fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  // Parse the form body. If the body is not valid form data, return 400.
  use form_data <- wisp.require_form(req)

  // Extract the title from the submitted values.
  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      // Validation passed. Create the task.
      let task = Task(id: new_id(), title: valid_title, done: False)
      actor.send(ctx.tasks, AddTask(task))

      // Return a fresh empty form with status 201 (Created).
      let form_html = add_task_form([], [])
      wisp.html_response(element.to_string(form_html), 201)
    }

    Error(errors) -> {
      // Validation failed. Return the form with errors and preserved input.
      let form_html = add_task_form(errors, form_data.values)
      wisp.html_response(element.to_string(form_html), 422)
    }
  }
}
```

Follow the two paths through this function.

**The happy path** (`Ok(valid_title)`): Validation succeeded. We create the
task, add it to the state via the actor, and return status `201` with a fresh
empty form. The `add_task_form([], [])` call passes no errors and no values,
which produces a clean form ready for the next task. HTMX swaps it in, and the
user sees their input cleared -- a visual confirmation that the task was
created.

**The error path** (`Error(errors)`): Validation failed. We do **not** create
any task. Instead we re-render the form by passing the error list and the
original form values (`form_data.values`) back to `add_task_form`. This
produces a form with error messages visible and the user's input still in the
fields. We return this with status `422`. HTMX swaps it in, and the user sees
exactly what went wrong.

The critical detail: **the 422 response contains a complete HTML fragment**.
It is not a JSON error object. It is not a plain text message. It is the same
kind of HTML that the 201 response contains -- just with error styling applied.
This is the hypermedia pattern at work: the server always returns HTML, and the
HTML always reflects the current state.

### Step 4 -- Wire It into the Router

In your router module, add the new route to the pattern match. This is the
same routing pattern from earlier chapters:

```gleam
fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["tasks"] ->
      case req.method {
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Post])
      }
    _ -> wisp.not_found()
  }
}
```

When a POST arrives at `/tasks`, it hits `create_task`, which runs validation
and returns either a 201 or a 422. Both responses contain HTML. HTMX handles
both the same way.

### Step 5 -- Add Error Styles

Open `priv/static/css/style.css` and add rules for the error states:

```css
/* Form layout */
.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.25rem;
  color: #1a1a2e;
}

.input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  font-size: 1rem;
  border: 2px solid #d0d0d8;
  border-radius: 4px;
  transition: border-color 0.15s ease;
}

.input:focus {
  outline: none;
  border-color: #4a6fa5;
}

/* Error states */
.input-error {
  border-color: #e74c3c !important;
}

.input-error:focus {
  border-color: #c0392b !important;
}

.error-text {
  color: #e74c3c;
  font-size: 0.875rem;
  margin-top: 0.25rem;
  display: block;
}

/* Button */
.btn {
  display: inline-block;
  padding: 0.5rem 1.25rem;
  font-size: 1rem;
  font-weight: 600;
  color: #fff;
  background-color: #4a6fa5;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.btn:hover {
  background-color: #3b5998;
}
```

The important details:

- **`.input-error`** overrides the border colour to red. The `!important` is
  necessary here because it must win over the `:focus` state as well. When a
  field has an error and the user clicks into it, the border should stay red
  until they resubmit and the error clears.
- **`.error-text`** is `display: block` so each error message appears on its
  own line below the input, rather than inline.
- The **transition** on `.input` provides a subtle animation when the border
  colour changes, making the error state feel less jarring.

### Step 6 -- Understand the Full Cycle

Let's trace what happens when a user submits an empty form.

```
Browser                                Server
  |                                       |
  |  POST /tasks                          |
  |  title=                               |
  |  HX-Request: true                     |
  |-------------------------------------->|
  |                                       |  wisp.require_form
  |                                       |    -> values: [#("title", "")]
  |                                       |  validate_task_title("")
  |                                       |    -> Error([Required("title")])
  |                                       |  add_task_form(errors, values)
  |                                       |    -> <div id="form-container">
  |                                       |         <form ...>
  |                                       |           <input class="input input-error" ...>
  |                                       |           <span class="error-text">
  |                                       |             title is required
  |                                       |           </span>
  |                                       |           ...
  |                                       |         </form>
  |                                       |       </div>
  |                                       |  wisp.html_response(html, 422)
  |  422 Unprocessable Entity             |
  |  <div id="form-container">...</div>   |
  |<--------------------------------------|
  |                                       |
  HTMX sees hx-target="#form-container"
  HTMX swaps response into #form-container
  using OuterHTML swap.

  User sees the form with a red border
  on the title input and the message
  "title is required" below it.
```

Now the user types "Buy groceries" and submits again:

```
Browser                                Server
  |                                       |
  |  POST /tasks                          |
  |  title=Buy groceries                  |
  |  HX-Request: true                     |
  |-------------------------------------->|
  |                                       |  wisp.require_form
  |                                       |    -> values: [#("title", "Buy groceries")]
  |                                       |  validate_task_title("Buy groceries")
  |                                       |    -> Ok("Buy groceries")
  |                                       |  Create task, send to actor
  |                                       |  add_task_form([], [])
  |                                       |    -> <div id="form-container">
  |                                       |         <form ...>
  |                                       |           <input class="input" value="" ...>
  |                                       |           (no error spans)
  |                                       |           ...
  |                                       |         </form>
  |                                       |       </div>
  |                                       |  wisp.html_response(html, 201)
  |  201 Created                          |
  |  <div id="form-container">...</div>   |
  |<--------------------------------------|
  |                                       |
  HTMX swaps in the clean form.
  User sees the input cleared, no errors.
  The task was created.
```

Both responses follow the same shape. Both get swapped by HTMX. The only
difference is the status code and whether the HTML contains error styling.

---

## 3. Full Code Listing

Here is every file involved in this chapter's changes.

### `src/teamwork/validation.gleam`

```gleam
import gleam/int
import gleam/list
import gleam/string

/// A validation error describes what went wrong and which field it applies to.
pub type ValidationError {
  Required(field: String)
  TooShort(field: String, min: Int)
  TooLong(field: String, max: Int)
}

/// Validates a task title.
/// Returns Ok with the trimmed title, or Error with a list of problems.
pub fn validate_task_title(
  title: String,
) -> Result(String, List(ValidationError)) {
  let title = string.trim(title)
  case string.length(title) {
    0 -> Error([Required("title")])
    n if n < 3 -> Error([TooShort("title", 3)])
    n if n > 200 -> Error([TooLong("title", 200)])
    _ -> Ok(title)
  }
}

/// Converts a validation error into a human-readable message.
pub fn error_message(error: ValidationError) -> String {
  case error {
    Required(field) ->
      field <> " is required"
    TooShort(field, min) ->
      field <> " must be at least " <> int.to_string(min) <> " characters"
    TooLong(field, max) ->
      field <> " must be at most " <> int.to_string(max) <> " characters"
  }
}

/// Filters a list of errors to only those for a specific field.
pub fn errors_for(
  errors: List(ValidationError),
  field: String,
) -> List(ValidationError) {
  list.filter(errors, fn(e) {
    case e {
      Required(f) if f == field -> True
      TooShort(f, _) if f == field -> True
      TooLong(f, _) if f == field -> True
      _ -> False
    }
  })
}
```

### `src/teamwork/web.gleam` (updated sections)

```gleam
import gleam/http
import gleam/list
import gleam/otp/actor
import gleam/result
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import hx
import wisp

import teamwork/validation.{type ValidationError}

// ... (Context type, Task type, actor messages from previous chapters) ...

fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["tasks"] ->
      case req.method {
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Post])
      }
    _ -> wisp.not_found()
  }
}

fn home_page(_req: wisp.Request, ctx: Context) -> wisp.Response {
  let tasks = actor.call(ctx.tasks, 1000, GetTasks)
  let page =
    layout("Teamwork", html.div([attribute.class("container")], [
      html.h1([], [element.text("Teamwork")]),
      add_task_form([], []),
      task_list(tasks),
    ]))
  let html_string = element.to_document_string(page)
  wisp.html_response(html_string, 200)
}

fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      // Validation passed -- create the task.
      let task = Task(id: new_id(), title: valid_title, done: False)
      actor.send(ctx.tasks, AddTask(task))

      // Return a fresh empty form (status 201).
      let form_html = add_task_form([], [])
      wisp.html_response(element.to_string(form_html), 201)
    }

    Error(errors) -> {
      // Validation failed -- return the form with errors (status 422).
      let form_html = add_task_form(errors, form_data.values)
      wisp.html_response(element.to_string(form_html), 422)
    }
  }
}

fn add_task_form(
  errors: List(ValidationError),
  values: List(#(String, String)),
) -> Element(t) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#form-container")),
        hx.swap(hx.OuterHTML),
      ],
      [
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("title")],
            [element.text("Task title")],
          ),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.value(title_value),
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
          ..list.map(title_errors, fn(e) {
            html.span(
              [attribute.class("error-text")],
              [element.text(validation.error_message(e))],
            )
          })
        ]),
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}

fn task_list(tasks: List(Task)) -> Element(t) {
  html.div([attribute.id("task-list")], [
    html.ul(
      [attribute.class("task-list")],
      list.map(tasks, fn(task) {
        html.li([attribute.class("task-item")], [
          element.text(task.title),
        ])
      }),
    ),
  ])
}
```

### `priv/static/css/style.css` (additions)

```css
/* Existing styles from previous chapters ... */

/* ─── Form layout ─── */

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.25rem;
  color: #1a1a2e;
}

.input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  font-size: 1rem;
  border: 2px solid #d0d0d8;
  border-radius: 4px;
  transition: border-color 0.15s ease;
}

.input:focus {
  outline: none;
  border-color: #4a6fa5;
}

/* ─── Error states ─── */

.input-error {
  border-color: #e74c3c !important;
}

.input-error:focus {
  border-color: #c0392b !important;
}

.error-text {
  color: #e74c3c;
  font-size: 0.875rem;
  margin-top: 0.25rem;
  display: block;
}

/* ─── Button ─── */

.btn {
  display: inline-block;
  padding: 0.5rem 1.25rem;
  font-size: 1rem;
  font-weight: 600;
  color: #fff;
  background-color: #4a6fa5;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.btn:hover {
  background-color: #3b5998;
}
```

---

## 4. Exercise

Work through these tasks in order. Each one builds on the previous.

### Task 1 -- Verify the Title Validation

Start the server with `gleam run` and open the task board in your browser.

1. Click "Add Task" without typing anything. You should see the error message
   "title is required" appear below the input, and the input border should
   turn red.
2. Type "Hi" (two characters) and submit. You should see "title must be at
   least 3 characters".
3. Type "Buy groceries" and submit. The error should disappear and the form
   should reset to empty.

**Acceptance criteria:** All three scenarios behave as described.

### Task 2 -- Add a Description Field

Add an optional description field to the task form. The description is not
required, but if the user enters one, it must be at most 1000 characters.

1. Add a new validation function `validate_task_description` in
   `validation.gleam`. It should accept an empty string (returning `Ok("")`)
   but reject strings longer than 1000 characters.
2. Add a `<textarea>` to the form with `name="description"`.
3. Update `create_task` to validate both fields. Combine the error lists.
4. Display description errors below the textarea.

**Acceptance criteria:** Submitting with an empty description succeeds.
Submitting with a description over 1000 characters shows an error. The title
and description fields are validated independently -- errors on one do not
affect the other.

### Task 3 -- Style the Error States

Make sure the error styling is clear and accessible:

1. The input border turns red when there are errors.
2. Error messages appear in red below the field.
3. When the user corrects the error and resubmits, the error styling disappears.

Try adding a subtle background colour to the error input (e.g., a very light
red, `#fef2f2`) to make the error state even more visible.

**Acceptance criteria:** Error fields are visually distinct from valid fields.
Correcting the error and resubmitting returns the field to its normal
appearance.

### Task 4 -- Inspect the Network Tab

Open your browser's developer tools and go to the **Network** tab.

1. Submit an empty form. Find the POST request to `/tasks`. Click on it and
   check the **status code**. It should be `422`.
2. Look at the **response body**. You should see HTML containing the error
   message.
3. Now submit a valid title. The status code should be `201`.
4. Notice that HTMX swapped in the response body in both cases.

**Acceptance criteria:** You can see 422 and 201 status codes in the Network
tab, and you understand that HTMX treated both responses the same way (swap
the HTML into the target).

### Task 5 -- Preserve User Input on Error

Verify that the user's input is preserved when validation fails:

1. Type a very long title (over 200 characters). Submit.
2. Confirm that the error message appears AND the title field still contains
   what you typed.
3. Shorten the title and resubmit. Confirm it succeeds.

If the input is not being preserved, check that you are passing
`form_data.values` to `add_task_form` in the error branch of `create_task`.

**Acceptance criteria:** The user never has to retype their input after a
validation error.

---

## 5. Exercise Solution Hints

Try each task on your own before reading these hints.

### Hint for Task 1

This should work out of the box if you have followed the code walkthrough. If
you see no error message, check:

- The form's `hx-target` points to `#form-container`.
- The `hx-swap` is set to `OuterHTML`.
- The wrapping `div` has `id="form-container"` in **both** the page render and
  the response from `create_task`.

### Hint for Task 2

The description validation function:

```gleam
pub fn validate_task_description(
  description: String,
) -> Result(String, List(ValidationError)) {
  let description = string.trim(description)
  case string.length(description) {
    n if n > 1000 -> Error([TooLong("description", 1000)])
    _ -> Ok(description)
  }
}
```

Notice that an empty string returns `Ok("")` -- the field is optional.

To combine errors from multiple fields, validate each field separately and then
merge the results:

```gleam
let title_result = validation.validate_task_title(title)
let description_result = validation.validate_task_description(description)

case title_result, description_result {
  Ok(valid_title), Ok(valid_description) -> {
    // Both valid -- create the task
    // ...
    wisp.html_response(element.to_string(add_task_form([], [])), 201)
  }
  _, _ -> {
    // At least one failed -- collect all errors
    let title_errors = case title_result {
      Ok(_) -> []
      Error(errs) -> errs
    }
    let desc_errors = case description_result {
      Ok(_) -> []
      Error(errs) -> errs
    }
    let all_errors = list.append(title_errors, desc_errors)
    let form_html = add_task_form(all_errors, form_data.values)
    wisp.html_response(element.to_string(form_html), 422)
  }
}
```

For the textarea in the form:

```gleam
let desc_value =
  list.key_find(values, "description")
  |> result.unwrap("")
let desc_errors = validation.errors_for(errors, "description")

html.div([attribute.class("form-group")], [
  html.label(
    [attribute.for("description")],
    [element.text("Description (optional)")],
  ),
  html.textarea(
    [
      attribute.name("description"),
      attribute.id("description"),
      attribute.class(case desc_errors {
        [] -> "input"
        _ -> "input input-error"
      }),
    ],
    desc_value,
  ),
  ..list.map(desc_errors, fn(e) {
    html.span(
      [attribute.class("error-text")],
      [element.text(validation.error_message(e))],
    )
  })
])
```

### Hint for Task 3

To add a subtle background colour on error inputs, add this to your CSS:

```css
.input-error {
  border-color: #e74c3c !important;
  background-color: #fef2f2;
}
```

The `#fef2f2` colour is a very light red that is noticeable without being
overwhelming. When the error clears, HTMX swaps in a form without the
`input-error` class, so the background returns to normal automatically.

### Hint for Task 4

This is an observation task. In Chrome or Firefox:

1. Open DevTools with `F12`.
2. Go to the **Network** tab.
3. Submit the form.
4. Look for the POST request to `/tasks` in the request list.
5. Click on it. The **Status** column (or the Headers panel) shows the code.
6. The **Response** tab shows the HTML that was returned.

If you want to use `curl` instead:

```bash
# Submit an empty title -- expect 422
curl -v -X POST http://localhost:8000/tasks -d "title="

# Submit a valid title -- expect 201
curl -v -X POST http://localhost:8000/tasks -d "title=Buy+groceries"
```

The `-v` flag shows request and response headers, including the status code.

### Hint for Task 5

The key is in the error branch of `create_task`:

```gleam
Error(errors) -> {
  let form_html = add_task_form(errors, form_data.values)
  //                                     ^^^^^^^^^^^^^^^^
  //                                     This preserves the user's input.
  // ...
}
```

And in `add_task_form`, the `attribute.value(title_value)` sets the input's
value attribute to whatever the user originally typed. When HTMX swaps in this
HTML, the browser renders the input with the preserved text.

If the value is not being preserved, make sure you are passing
`form_data.values` (not an empty list) in the `Error` branch.

---

## 6. Key Takeaways

1. **Server-side validation is not optional.** Client-side checks (HTML5
   attributes, JavaScript) improve the user experience, but they can be
   bypassed. The server is the only place where validation is enforced.

2. **Gleam's `Result` type models validation naturally.** `Ok(valid_data)`
   means the input is good. `Error(list_of_errors)` means it failed, and the
   list tells you exactly what went wrong. The compiler ensures you handle
   both cases.

3. **Use `422 Unprocessable Entity` for validation failures.** This status
   code means "I understood your request, but the data is invalid." It is more
   specific than `400 Bad Request`, which implies the request itself is
   malformed.

4. **HTMX swaps responses regardless of status code.** A `422` response with
   an HTML fragment gets swapped into the target just like a `200`. This is
   the foundation of the pattern: return error HTML on failure, success HTML
   on success, and let HTMX handle the swap.

5. **Re-render the form with errors and preserved input.** When validation
   fails, the server returns the same form with two additions: error messages
   next to the invalid fields, and `value` attributes that restore whatever
   the user typed. The user never loses their work.

6. **A dedicated validation module keeps things clean.** By separating
   validation logic (`ValidationError` types, `validate_*` functions,
   `error_message`) from HTTP handling and HTML rendering, each piece stays
   focused and testable. You can unit-test your validation functions without
   spinning up a server.

7. **Conditional rendering in Lustre is just pattern matching.** There is no
   special `if` directive or conditional component. You use `case` expressions
   to decide which class to apply or which elements to include. An empty list
   of errors means no error spans are rendered. A non-empty list means they
   appear. Gleam's type system does the rest.

8. **CSS transitions smooth the experience.** A `transition` on `border-color`
   makes the shift to and from the error state feel deliberate rather than
   jarring. Small polish like this adds up.

---

## What's Next

We now have forms that validate input and show clear error feedback. But there
is a gap: when a task is successfully created, the form resets -- but the task
list does not update. The user has to refresh the page to see their new task.

In [Chapter 9](09-swap-strategies-and-targets.md) we will solve this with **Out-of-Band (OOB) Swaps**. OOB swaps
let a single server response update multiple parts of the page at once. The
form will reset *and* the task list will update, all from one POST response.
No page refresh needed.
