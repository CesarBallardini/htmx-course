# Chapter 18 -- Integrating Third-Party JavaScript: Flatpickr

**Phase:** Advanced (bonus chapter)
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 17](17-deployment-and-production.md) deployed the application. This chapter adds due dates
using Flatpickr, demonstrating the general pattern for integrating third-party
JavaScript libraries with HTMX.

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why native `<input type="date">` is insufficient and when a third-party picker is justified.
- Install and configure Flatpickr via CDN -- no build step.
- Solve the HTMX swap re-initialization problem using the `htmx:load` event.
- Use `_hyperscript` as a declarative alternative: `_="on load call flatpickr(me, {...})"`.
- Parse date strings from form data in Gleam and store them in SQLite.
- Build a date-range filter with two Flatpickr instances and `hx-include`.

---

## 1. Theory

### 1.1 The Problem with Native Date Inputs

HTML provides `<input type="date">`. It works. It renders a calendar. So why would you ever load a third-party library?

Because browsers cannot agree on what "works" means:

| Behaviour                | Chrome / Edge        | Firefox              | Safari               | Mobile (iOS/Android) |
|--------------------------|----------------------|----------------------|----------------------|----------------------|
| **Calendar popup**       | Built-in             | Built-in             | Built-in (macOS 14+) | Native OS picker     |
| **Date format display**  | Follows OS locale    | Follows OS locale    | Follows OS locale    | Follows OS locale    |
| **Format control**       | None                 | None                 | None                 | None                 |
| **First day of week**    | Follows OS           | Follows OS           | Follows OS           | Follows OS           |
| **Styling**              | Minimal              | Minimal              | Minimal              | None                 |
| **Min/max enforcement**  | Yes                  | Yes                  | Yes                  | Yes                  |
| **Date range selection** | No                   | No                   | No                   | No                   |
| **Disable specific days**| No                   | No                   | No                   | No                   |
| **Custom calendar UI**   | No                   | No                   | No                   | No                   |

The native input handles the simplest case well: let the user pick a single date. But the moment you need to control the display format, disable weekends, select a date range, or match the calendar's appearance to your application's design, you hit a wall.

There is no way to change the date format shown to the user. There is no way to disable specific days (like holidays). There is no range mode. There is no way to embed the calendar inline instead of as a popup. For these requirements, you need a JavaScript date picker.

### 1.2 Flatpickr -- Philosophy and Fit

Flatpickr is a lightweight date and time picker. It has three properties that make it a natural fit for HTMX projects:

1. **Small.** ~16 KB minified and gzipped. Comparable to HTMX itself.
2. **Zero dependencies.** No jQuery, no Moment.js, no framework required.
3. **No build step.** Load two files from a CDN -- one CSS, one JS -- and you are done.

These properties align with HTMX's philosophy of simplicity and minimal tooling.

| Library       | Size (min+gz) | Dependencies | Build step required | Range mode | Time picker |
|---------------|---------------|--------------|---------------------|------------|-------------|
| **Flatpickr** | ~16 KB        | None         | No                  | Yes        | Yes         |
| Pikaday       | ~8 KB         | None         | No                  | No         | No          |
| Litepicker    | ~12 KB        | None         | No                  | Yes        | No          |
| Native input  | 0 KB          | None         | No                  | No         | Separate    |

Flatpickr strikes the best balance of features and size. Pikaday is smaller but lacks range mode and time picking. Litepicker is similar in size but lacks time support. The native input is free but inflexible.

### 1.3 Installation and Basic Usage

Installation requires two tags in your `<head>`:

```html
<!-- Flatpickr CSS -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css">

<!-- Flatpickr JS -->
<script src="https://cdn.jsdelivr.net/npm/flatpickr"></script>
```

Then initialize it on any input:

```javascript
flatpickr("#my-date-input", {
  dateFormat: "Y-m-d",
});
```

That is the minimal setup. Flatpickr takes over the input: clicking it opens a calendar popup, and selecting a date fills in the value in the specified format.

Here are the most useful configuration options:

| Option         | Type                      | Default       | Description                                                |
|----------------|---------------------------|---------------|------------------------------------------------------------|
| `dateFormat`   | String                    | `"Y-m-d"`     | Format of the value submitted to the server.               |
| `altInput`     | Boolean                   | `false`       | Show a human-friendly format while submitting machine format. |
| `altFormat`    | String                    | `"F j, Y"`    | Format shown to the user when `altInput` is `true`.        |
| `defaultDate`  | String / Date             | `null`        | Pre-selected date on initialization.                       |
| `minDate`      | String / Date             | `null`        | Earliest selectable date.                                  |
| `maxDate`      | String / Date             | `null`        | Latest selectable date.                                    |
| `mode`         | `"single"` / `"range"` / `"multiple"` | `"single"` | Selection mode.                                   |
| `enableTime`   | Boolean                   | `false`       | Add a time picker below the calendar.                      |
| `inline`       | Boolean                   | `false`       | Render the calendar inline instead of as a popup.          |
| `disable`      | Array of dates/functions  | `[]`          | Dates or date-testing functions to disable.                |

The `altInput` option is particularly important for HTMX forms. When enabled, Flatpickr creates a hidden input with the machine-readable `dateFormat` value (e.g. `"2026-03-15"`) and a visible input with the human-readable `altFormat` value (e.g. `"Mar 15, 2026"`). The hidden input is the one that gets submitted with the form. This gives you the best of both worlds: the user sees a friendly format, and the server receives a predictable one.

### 1.4 The HTMX Swap Problem

This is the core concept of this chapter. It applies to Flatpickr, but it applies equally to **any** JavaScript library that attaches state, listeners, or DOM nodes to elements.

Here is what happens:

1. Your page loads. An `<input>` element exists in the DOM.
2. JavaScript runs `flatpickr(input, {...})`. Flatpickr attaches event listeners, creates a hidden calendar `<div>`, and stores internal state on the input element.
3. The user interacts with the form. HTMX makes a request and **swaps** the response into the page.
4. The swap replaces the old DOM nodes with new ones from the server.
5. The new `<input>` is a fresh element. It has no event listeners, no calendar, no Flatpickr state. **The date picker is gone.**

Here is the lifecycle:

```
Page load
  │
  ▼
<input name="due_date">     ◄── Fresh DOM element
  │
  ▼
flatpickr(input, {...})     ◄── JS attaches listeners, creates calendar
  │
  ▼
<input name="due_date">     ◄── Now has calendar popup, event handlers, etc.
  │
  ▼
HTMX swaps new HTML         ◄── Server responds with new <input name="due_date">
  │
  ▼
<input name="due_date">     ◄── Fresh DOM element again -- NO calendar, NO listeners
```

This is not a Flatpickr-specific problem. The same thing happens with:

- **Sortable.js** -- drag handles stop working after a swap.
- **Chart.js** -- the canvas goes blank after a swap.
- **CodeMirror** -- the editor reverts to a plain textarea after a swap.
- **Tooltip libraries** -- tooltips disappear after a swap.
- **Any** library that calls something like `new Widget(element)` or `Widget.init(element)`.

The root cause is always the same: HTMX replaces DOM nodes, and the JavaScript library's state was attached to the old nodes.

> **NOTE:** This swap re-initialization problem is the single most common issue
> when integrating third-party JavaScript with HTMX. Every library that attaches
> to the DOM will have this problem. The solution in the next section is universal.

### 1.5 Solution: The `htmx:load` Event

HTMX fires the `htmx:load` event on every element that is newly added to the DOM. This includes:

- Elements present on initial page load (fired once, on `<body>`).
- Elements inserted by any HTMX swap (fired on each new element).

This is the single event you need to listen for. It tells you "new content is in the DOM -- initialize your libraries now."

There are two approaches.

**Approach A: Global JavaScript Listener**

Add a `<script>` block in your layout that listens for `htmx:load` and initializes Flatpickr on any element with a `data-flatpickr` marker attribute:

```javascript
document.addEventListener("htmx:load", function(event) {
  // event.detail.elt is the newly loaded element
  var inputs = event.detail.elt.querySelectorAll("[data-flatpickr]");
  inputs.forEach(function(input) {
    // Guard against double initialization
    if (input._flatpickr) return;
    flatpickr(input, {
      dateFormat: "Y-m-d",
      altInput: true,
      altFormat: "M j, Y",
    });
  });
});
```

The `input._flatpickr` guard is important. Flatpickr stores a reference to its instance on the DOM element as `._flatpickr`. If the element already has one (because it was not swapped), skip initialization. Without this guard, you could end up with multiple calendars attached to the same input.

**Approach B: `_hyperscript` Inline**

If you loaded `_hyperscript` (covered in [Appendix B](../05-appendices/A2-hyperscript.md)), you can initialize Flatpickr directly on the element with a single attribute:

```html
<input name="due_date"
       _="on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})" />
```

The `on load` event in `_hyperscript` fires when the element is added to the DOM -- which aligns perfectly with `htmx:load`. The `me` keyword refers to the current element. The result is a one-line, self-contained initialization that travels with the element.

**When to use which approach:**

| Scenario                              | Recommended approach       |
|---------------------------------------|----------------------------|
| Many inputs with the same config      | Global JS listener         |
| One or two inputs with unique configs | `_hyperscript` inline      |
| No `_hyperscript` in the project      | Global JS listener         |
| Quick prototyping                     | `_hyperscript` inline      |

For the Teamwork application, we will use `_hyperscript` inline. The task form has a single date input, and the filter bar has two. The inline approach keeps the initialization logic next to the element it affects, making it easy to read and maintain.

### 1.6 Dates on the Server Side

Flatpickr submits the date as a string in the `dateFormat` you configured. With `dateFormat: "Y-m-d"`, the form value will be something like `"2026-03-15"`.

**Extracting the date in Gleam:**

When the form is submitted, Wisp parses the form body into a `FormData` with a `.values` field of type `List(#(String, String))`. Extract the date with `list.key_find`:

```gleam
use form_data <- wisp.require_form(req)
let due_date = case list.key_find(form_data.values, "due_date") {
  Ok("") -> option.None
  Ok(date) -> option.Some(date)
  Error(_) -> option.None
}
```

An empty string means the user did not pick a date. We represent this as `option.None`.

**Storing in SQLite:**

Store the date as `TEXT` in ISO format (`YYYY-MM-DD`). SQLite does not have a dedicated date type, but its built-in `date()` and `datetime()` functions understand ISO format strings. More importantly, ISO date strings sort correctly in lexicographic order -- `"2026-01-15"` comes before `"2026-03-01"` in any string comparison. This means you can filter and order by dates in SQL without any parsing. In Gleam, the `<` operator works on numbers (`Int` and `Float`) but not on `String`, so you use `string.compare(a, b)` from `gleam/string` to compare date strings (it returns `order.Lt`, `order.Eq`, or `order.Gt`).

```sql
-- This works because ISO dates sort lexicographically
SELECT * FROM tasks WHERE due_date >= '2026-03-01' AND due_date <= '2026-03-31';
```

**Getting today's date:**

To highlight overdue tasks, you need today's date on the server. Gleam does not have a built-in "today as YYYY-MM-DD string" function, but you can get it from Erlang's `calendar` module with a small FFI helper:

```gleam
// src/teamwork/date_helpers.gleam

/// Returns today's date as a "YYYY-MM-DD" string in UTC.
@external(erlang, "teamwork_date_ffi", "today_string")
pub fn today() -> String
```

The Erlang implementation:

```erlang
%% src/teamwork_date_ffi.erl

-module(teamwork_date_ffi).
-export([today_string/0]).

today_string() ->
    {{Year, Month, Day}, _Time} = calendar:universal_time(),
    Formatted = io_lib:format("~4..0B-~2..0B-~2..0B", [Year, Month, Day]),
    list_to_binary(lists:flatten(Formatted)).
```

This returns a binary (Gleam `String`) like `<<"2026-03-15">>`. The `~4..0B` format pads the year to 4 digits with leading zeros, and `~2..0B` pads month and day to 2 digits.

### 1.7 Modes, Themes, and Localization

Flatpickr has several features beyond basic date picking that are worth knowing about.

**Modes:**

- `"single"` (default) -- pick one date. The value is `"2026-03-15"`.
- `"range"` -- pick a start and end date. The value is `"2026-03-01 to 2026-03-15"`.
- `"multiple"` -- pick several individual dates. The value is `"2026-03-01, 2026-03-15, 2026-03-20"`.

Range mode is useful for filter inputs. The separator is always `" to "` (space-to-space).

**Themes:**

Flatpickr ships with several CSS themes. Swap the CSS `<link>` to change the appearance:

```html
<!-- Default -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css">

<!-- Dark theme -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/themes/dark.css">

<!-- Material Blue -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/themes/material_blue.css">
```

No JavaScript changes needed. The theme is purely CSS.

**Localization:**

Flatpickr defaults to English. To use another language, load the locale file and set the `locale` option:

```html
<script src="https://cdn.jsdelivr.net/npm/flatpickr/dist/l10n/es.js"></script>
<script>
  flatpickr("#date", { locale: "es" });
</script>
```

Month names, day names, and the first day of the week are all localized.

**Disabling dates:**

The `disable` option accepts an array of dates or functions:

```javascript
flatpickr("#date", {
  minDate: "today",         // No past dates
  disable: [
    function(date) {
      // Disable weekends (0 = Sunday, 6 = Saturday)
      return date.getDay() === 0 || date.getDay() === 6;
    }
  ],
});
```

**The `onChange` callback:**

Flatpickr fires callbacks when the user interacts with the calendar:

```javascript
flatpickr("#date", {
  onChange: function(selectedDates, dateStr, instance) {
    // selectedDates: Array of Date objects
    // dateStr: formatted date string
    // instance: the Flatpickr instance
    console.log("Selected:", dateStr);
  },
});
```

This is useful for triggering dynamic behaviour -- for example, enabling a submit button only after a date is selected.

---

## 2. Code Walkthrough

We are going to make eight changes to the Teamwork application:

1. Add an Erlang FFI helper to get today's date.
2. Update the database schema to include a `due_date` column.
3. Load Flatpickr and `_hyperscript` in the layout.
4. Add a due date field to the task creation form.
5. Parse the due date on the server.
6. Display due dates in the task list.
7. Style overdue tasks.
8. Add a date range filter.

### Step 1 -- Get Today's Date from Erlang

Create a small FFI module to get the current date as an ISO string.

The Gleam side declares the external function:

```gleam
// src/teamwork/date_helpers.gleam

@external(erlang, "teamwork_date_ffi", "today_string")
pub fn today() -> String
```

The Erlang side implements it:

```erlang
%% src/teamwork_date_ffi.erl

-module(teamwork_date_ffi).
-export([today_string/0]).

today_string() ->
    {{Year, Month, Day}, _Time} = calendar:universal_time(),
    Formatted = io_lib:format("~4..0B-~2..0B-~2..0B", [Year, Month, Day]),
    list_to_binary(lists:flatten(Formatted)).
```

This is the only Erlang code in the entire chapter. The function returns a UTC date string like `"2026-03-15"`. We use UTC to avoid timezone complications -- for a task board, UTC dates are sufficient.

### Step 2 -- Update the Database Schema

Add a `due_date` column to the tasks table. In the migration:

```gleam
// In db.gleam, update the migrate function:

pub fn migrate(db: sqlight.Connection) -> Result(Nil, sqlight.Error) {
  sqlight.exec(db, "
    PRAGMA journal_mode=WAL;
    PRAGMA foreign_keys=ON;

    CREATE TABLE IF NOT EXISTS boards (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      description TEXT NOT NULL DEFAULT '',
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS tasks (
      id TEXT PRIMARY KEY,
      board_id TEXT NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
      title TEXT NOT NULL,
      description TEXT NOT NULL DEFAULT '',
      done INTEGER NOT NULL DEFAULT 0,
      due_date TEXT NOT NULL DEFAULT '',
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS users (
      id TEXT PRIMARY KEY,
      username TEXT NOT NULL UNIQUE,
      password_hash TEXT NOT NULL,
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
  ")
}
```

The `due_date` column is `TEXT NOT NULL DEFAULT ''`. An empty string means "no due date." This is simpler than using `NULL` because SQLite's `NULL` handling can be surprising in comparisons.

> **NOTE:** If you already have an existing database with data, adding a new column
> requires an `ALTER TABLE` statement instead of relying on `CREATE TABLE IF NOT EXISTS`.
> For development, the simplest approach is to delete the database file and let the
> migration recreate it. For production, add a versioned migration:
> `ALTER TABLE tasks ADD COLUMN due_date TEXT NOT NULL DEFAULT '';`

Update the `Task` type to include the due date:

```gleam
import gleam/option

pub type Task {
  Task(
    id: String,
    title: String,
    description: String,
    done: Bool,
    due_date: option.Option(String),
  )
}
```

We use `option.Option(String)` in the Gleam type even though the database stores an empty string. The decoder converts between the two representations:

```gleam
fn task_decoder() -> decode.Decoder(Task) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  use due_date_raw <- decode.field(4, decode.string)
  let due_date = case due_date_raw {
    "" -> option.None
    date -> option.Some(date)
  }
  decode.success(Task(
    id: id,
    title: title,
    description: description,
    done: done,
    due_date: due_date,
  ))
}
```

Empty string becomes `option.None`. Any non-empty string becomes `option.Some(date)`.

Update `list_tasks` to select the new column:

```gleam
pub fn list_tasks(
  db: sqlight.Connection,
  board_id: String,
) -> Result(List(Task), sqlight.Error) {
  let sql = "
    SELECT id, title, description, done, due_date
    FROM tasks
    WHERE board_id = ?1
    ORDER BY created_at DESC
  "
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(board_id)],
    expecting: task_decoder(),
  )
}
```

Update `create_task` to accept and store the due date:

```gleam
pub fn create_task(
  db: sqlight.Connection,
  board_id: String,
  task: Task,
) -> Result(Nil, sqlight.Error) {
  let due_date_str = case task.due_date {
    option.Some(date) -> date
    option.None -> ""
  }
  let sql = "
    INSERT INTO tasks (id, board_id, title, description, done, due_date)
    VALUES (?1, ?2, ?3, ?4, ?5, ?6)
  "
  sqlight.query(
    sql,
    on: db,
    with: [
      sqlight.text(task.id),
      sqlight.text(board_id),
      sqlight.text(task.title),
      sqlight.text(task.description),
      sqlight.int(case task.done { True -> 1 False -> 0 }),
      sqlight.text(due_date_str),
    ],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}
```

### Step 3 -- Load Flatpickr in the Layout

Add Flatpickr's CSS and JavaScript to the layout's `<head>`, alongside the existing HTMX script. Also add `_hyperscript` if it is not already loaded:

```gleam
// In web.gleam, update the layout function's <head>:

fn layout(content: element.Element(Nil)) -> element.Element(Nil) {
  html.html([], [
    html.head([], [
      html.title([], "Teamwork"),
      html.meta([attribute.attribute("charset", "utf-8")]),
      html.meta([
        attribute.name("viewport"),
        attribute.attribute("content", "width=device-width, initial-scale=1"),
      ]),
      // Application CSS
      html.link([
        attribute.rel("stylesheet"),
        attribute.href("/static/css/style.css"),
      ]),
      // Flatpickr CSS
      html.link([
        attribute.rel("stylesheet"),
        attribute.href(
          "https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css",
        ),
      ]),
      // HTMX
      html.script(
        [attribute.src("/static/js/htmx.min.js")],
        "",
      ),
      // Flatpickr JS
      html.script(
        [attribute.src("https://cdn.jsdelivr.net/npm/flatpickr")],
        "",
      ),
      // _hyperscript
      html.script(
        [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
        "",
      ),
    ]),
    html.body([hx.boost(True)], [content]),
  ])
}
```

Three new tags: Flatpickr CSS, Flatpickr JS, and `_hyperscript`. All loaded from CDNs with no build step. If you self-hosted HTMX in [Chapter 17](17-deployment-and-production.md), you can download and self-host these as well.

### Step 4 -- Add Due Date Field to the Task Form

Add a date input to the task creation form with `_hyperscript` to initialize Flatpickr:

```gleam
fn task_form(board_id: String) -> element.Element(Nil) {
  html.form(
    [
      attribute.id("task-form"),
      attribute.method("post"),
      attribute.action("/boards/" <> board_id <> "/tasks"),
      hx.post("/boards/" <> board_id <> "/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
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
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("due_date")], [element.text("Due date")]),
        html.input([
          attribute.type_("text"),
          attribute.name("due_date"),
          attribute.id("due_date"),
          attribute.placeholder("Pick a date (optional)"),
          attribute.attribute(
            "_",
            "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
          ),
        ]),
      ]),
      html.button([attribute.type_("submit")], [element.text("Add Task")]),
    ],
  )
}
```

Let's break down the `_hyperscript` attribute:

- `on load` -- fires when this element is added to the DOM (initial load or after an HTMX swap).
- `call flatpickr(me, {...})` -- calls the global `flatpickr` function, passing `me` (this element) as the target.
- `dateFormat: 'Y-m-d'` -- the value submitted to the server.
- `altInput: true` -- shows a human-readable version to the user.
- `altFormat: 'M j, Y'` -- the human-readable format (e.g. "Mar 15, 2026").

In Gleam/Lustre, we use `attribute.attribute("_", "...")` to set the `_` attribute. This is the generic raw attribute function from `lustre/attribute`.

Notice the `type_` is `"text"`, not `"date"`. We want Flatpickr to control the input entirely. Using `type="date"` would cause the browser's native picker to appear alongside Flatpickr, creating a confusing double-picker experience.

### Step 5 -- Parse the Due Date on the Server

In the request handler that creates a task, extract the `due_date` field from the form data:

```gleam
fn create_task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let assert Ok(title) = list.key_find(form_data.values, "title")
  let due_date = case list.key_find(form_data.values, "due_date") {
    Ok("") -> option.None
    Ok(date) -> option.Some(date)
    Error(_) -> option.None
  }

  let task_id = wisp.random_string(16)
  let task =
    db.Task(
      id: task_id,
      title: title,
      description: "",
      done: False,
      due_date: due_date,
    )

  let assert Ok(_) = db.create_task(ctx.db, board_id, task)

  let today = date_helpers.today()
  wisp.html_response(
    element.to_string(task_item(task, today)),
    201,
  )
}
```

The `list.key_find(form_data.values, "due_date")` call returns `Ok("")` when the user submitted the form without picking a date (the input is present but empty), or `Error(Nil)` when the field was not in the form at all. Both cases map to `option.None`.

We pass `today` to `task_item` so the view can determine whether the task is overdue.

### Step 6 -- Display Due Dates in the Task List

Update the `task_item` function to show the due date when present, and mark overdue tasks:

```gleam
import gleam/bool
import gleam/order
import gleam/string
import teamwork/date_helpers

fn task_item(task: db.Task, today: String) -> element.Element(Nil) {
  let is_overdue = case task.due_date {
    option.Some(date) ->
      string.compare(date, today) == order.Lt && bool.negate(task.done)
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
      html.span([attribute.class("task-title")], [element.text(task.title)]),
      due_date_badge(task.due_date, is_overdue),
      // ... toggle and delete buttons ...
    ],
  )
}

fn due_date_badge(
  due_date: option.Option(String),
  is_overdue: Bool,
) -> element.Element(Nil) {
  case due_date {
    option.None -> element.none()
    option.Some(date) -> {
      let class = case is_overdue {
        True -> "due-date overdue"
        False -> "due-date"
      }
      html.span([attribute.class(class)], [element.text(date)])
    }
  }
}
```

Several things to notice:

**String comparison for dates.** Gleam's `<` operator works on numbers but not strings, so we use `string.compare(date, today)` which returns `order.Lt`, `order.Eq`, or `order.Gt`. Because ISO format is year-month-day with zero-padding, lexicographic comparison produces correct chronological ordering. No date parsing needed.

**`element.none()`.** When there is no due date, we render nothing. `element.none()` produces no output -- it is Lustre's way of saying "skip this element."

**Overdue logic.** A task is overdue when it has a due date that is before today (`string.compare(date, today) == order.Lt`) AND the task is not done (`bool.negate(task.done)`). Completed tasks are never shown as overdue, even if their due date has passed.

Update `list_tasks` in the handler to pass today's date to each task item:

```gleam
fn list_tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  let assert Ok(tasks) = db.list_tasks(ctx.db, board_id)
  let today = date_helpers.today()

  // ... filtering logic ...

  let html =
    element.fragment(
      list.map(filtered, fn(task) { task_item(task, today) }),
    )

  wisp.html_response(element.to_string(html), 200)
}
```

### Step 7 -- Style Overdue Tasks

Add CSS for the due date badge and overdue state:

```css
/* Due date badge */
.due-date {
  display: inline-block;
  padding: 2px 8px;
  margin-left: 8px;
  font-size: 0.85em;
  color: #555;
  background-color: #f0f0f0;
  border-radius: 4px;
}

.due-date.overdue {
  color: #b91c1c;
  background-color: #fee2e2;
  font-weight: 600;
}

/* Overdue task row */
.task-item.overdue {
  border-left: 3px solid #b91c1c;
}
```

Overdue tasks get a red left border and their date badge turns red. Completed tasks never have the `.overdue` class, so their dates remain neutral gray even if the date has passed.

### Step 8 -- Add a Date Range Filter

Add two Flatpickr inputs above the task list that filter tasks by date range. This demonstrates two things: multiple Flatpickr instances on the same page, and using `hx-include` to send filter values with a request.

The filter bar:

```gleam
fn filter_bar(board_id: String) -> element.Element(Nil) {
  html.div([attribute.class("filter-bar")], [
    html.label([], [
      element.text("From: "),
      html.input([
        attribute.type_("text"),
        attribute.name("from_date"),
        attribute.class("filter-date"),
        attribute.placeholder("Start date"),
        attribute.attribute(
          "_",
          "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
        ),
        hx.get("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.InnerHTML),
        hx.trigger([hx.change()]),
        hx.include(hx.Selector(".filter-bar")),
      ]),
    ]),
    html.label([], [
      element.text("To: "),
      html.input([
        attribute.type_("text"),
        attribute.name("to_date"),
        attribute.class("filter-date"),
        attribute.placeholder("End date"),
        attribute.attribute(
          "_",
          "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
        ),
        hx.get("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.InnerHTML),
        hx.trigger([hx.change()]),
        hx.include(hx.Selector(".filter-bar")),
      ]),
    ]),
  ])
}
```

Let's trace through what happens when the user picks a date in either filter input:

1. Flatpickr sets the input's value and fires a `change` event.
2. `hx.trigger([hx.change()])` catches the `change` event and triggers an HTMX request.
3. `hx.include(hx.Selector(".filter-bar"))` tells HTMX to include all input values from the `.filter-bar` container. This means both `from_date` and `to_date` are sent, even if the user only changed one.
4. `hx.get(...)` sends a GET request to the tasks endpoint with `from_date` and `to_date` as query parameters.
5. `hx.target(hx.Selector("#task-list"))` + `hx.swap(hx.InnerHTML)` replaces the task list contents with the filtered results.

On the server, parse the filter parameters and apply them:

```gleam
fn list_tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  let assert Ok(tasks) = db.list_tasks(ctx.db, board_id)
  let today = date_helpers.today()

  let query_params = wisp.get_query(req)

  let from_date =
    list.key_find(query_params, "from_date")
    |> result.unwrap("")

  let to_date =
    list.key_find(query_params, "to_date")
    |> result.unwrap("")

  let filtered =
    tasks
    |> filter_by_date_range(from_date, to_date)

  let html =
    element.fragment(
      list.map(filtered, fn(task) { task_item(task, today) }),
    )

  wisp.html_response(element.to_string(html), 200)
}

fn filter_by_date_range(
  tasks: List(db.Task),
  from_date: String,
  to_date: String,
) -> List(db.Task) {
  list.filter(tasks, fn(task) {
    case task.due_date {
      option.None -> {
        // Tasks without a due date: show them only if no date filter is active
        from_date == "" && to_date == ""
      }
      option.Some(date) -> {
        let after_from = case from_date {
          "" -> True
          from -> string.compare(date, from) != order.Lt
        }
        let before_to = case to_date {
          "" -> True
          to -> string.compare(date, to) != order.Gt
        }
        after_from && before_to
      }
    }
  })
}
```

The `filter_by_date_range` function applies both filters. If a filter value is empty (the user has not set it), that filter is skipped. Tasks without a due date are shown only when no date filter is active -- this prevents undated tasks from cluttering filtered results.

**Alternative: Global JavaScript initializer**

If you prefer not to use `_hyperscript`, here is the global listener approach for the same filter inputs. Add this `<script>` to your layout:

```javascript
document.addEventListener("htmx:load", function(event) {
  var inputs = event.detail.elt.querySelectorAll("[data-flatpickr]");
  inputs.forEach(function(input) {
    if (input._flatpickr) return;
    flatpickr(input, JSON.parse(input.dataset.flatpickrConfig || "{}"));
  });
});
```

Then replace the `_` attribute on each input with `data-flatpickr` and a JSON config:

```gleam
html.input([
  attribute.type_("text"),
  attribute.name("from_date"),
  attribute.attribute("data-flatpickr", ""),
  attribute.attribute(
    "data-flatpickr-config",
    "{\"dateFormat\": \"Y-m-d\", \"altInput\": true, \"altFormat\": \"M j, Y\"}",
  ),
  // ... hx attributes ...
])
```

Both approaches produce the same result. The global listener is more scalable when you have many date inputs. The `_hyperscript` approach is more readable for a small number of inputs.

---

## 3. Full Code Listing

Here are the complete files, incorporating every change from the walkthrough.

### `src/teamwork_date_ffi.erl`

```erlang
-module(teamwork_date_ffi).
-export([today_string/0]).

today_string() ->
    {{Year, Month, Day}, _Time} = calendar:universal_time(),
    Formatted = io_lib:format("~4..0B-~2..0B-~2..0B", [Year, Month, Day]),
    list_to_binary(lists:flatten(Formatted)).
```

### `src/teamwork/date_helpers.gleam`

```gleam
/// Returns today's date as a "YYYY-MM-DD" string in UTC.
@external(erlang, "teamwork_date_ffi", "today_string")
pub fn today() -> String
```

### `src/teamwork/db.gleam`

```gleam
import gleam/dynamic/decode
import gleam/option
import gleam/result
import sqlight

pub type Task {
  Task(
    id: String,
    title: String,
    description: String,
    done: Bool,
    due_date: option.Option(String),
  )
}

pub type Board {
  Board(id: String, name: String, description: String)
}

pub type User {
  User(id: String, username: String, password_hash: String)
}

// -- Connection and migrations --

pub fn connect(path: String) -> Result(sqlight.Connection, sqlight.Error) {
  sqlight.open(path)
}

pub fn migrate(db: sqlight.Connection) -> Result(Nil, sqlight.Error) {
  sqlight.exec(db, "
    PRAGMA journal_mode=WAL;
    PRAGMA foreign_keys=ON;

    CREATE TABLE IF NOT EXISTS boards (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      description TEXT NOT NULL DEFAULT '',
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS tasks (
      id TEXT PRIMARY KEY,
      board_id TEXT NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
      title TEXT NOT NULL,
      description TEXT NOT NULL DEFAULT '',
      done INTEGER NOT NULL DEFAULT 0,
      due_date TEXT NOT NULL DEFAULT '',
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS users (
      id TEXT PRIMARY KEY,
      username TEXT NOT NULL UNIQUE,
      password_hash TEXT NOT NULL,
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
  ")
}

// -- Task queries --

pub fn list_tasks(
  db: sqlight.Connection,
  board_id: String,
) -> Result(List(Task), sqlight.Error) {
  let sql = "
    SELECT id, title, description, done, due_date
    FROM tasks
    WHERE board_id = ?1
    ORDER BY created_at DESC
  "
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(board_id)],
    expecting: task_decoder(),
  )
}

pub fn create_task(
  db: sqlight.Connection,
  board_id: String,
  task: Task,
) -> Result(Nil, sqlight.Error) {
  let due_date_str = case task.due_date {
    option.Some(date) -> date
    option.None -> ""
  }
  let sql = "
    INSERT INTO tasks (id, board_id, title, description, done, due_date)
    VALUES (?1, ?2, ?3, ?4, ?5, ?6)
  "
  sqlight.query(
    sql,
    on: db,
    with: [
      sqlight.text(task.id),
      sqlight.text(board_id),
      sqlight.text(task.title),
      sqlight.text(task.description),
      sqlight.int(case task.done { True -> 1 False -> 0 }),
      sqlight.text(due_date_str),
    ],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}

pub fn toggle_task(
  db: sqlight.Connection,
  task_id: String,
) -> Result(Task, sqlight.Error) {
  let sql = "
    UPDATE tasks SET done = CASE WHEN done = 0 THEN 1 ELSE 0 END
    WHERE id = ?1
    RETURNING id, title, description, done, due_date
  "
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(task_id)],
    expecting: task_decoder(),
  )
  |> result.map(fn(rows) {
    let assert [task] = rows
    task
  })
}

pub fn delete_task(
  db: sqlight.Connection,
  task_id: String,
) -> Result(Nil, sqlight.Error) {
  let sql = "DELETE FROM tasks WHERE id = ?1"
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(task_id)],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}

// -- Board queries --

pub fn list_boards(
  db: sqlight.Connection,
) -> Result(List(Board), sqlight.Error) {
  let sql = "SELECT id, name, description FROM boards ORDER BY created_at DESC"
  sqlight.query(
    sql,
    on: db,
    with: [],
    expecting: board_decoder(),
  )
}

pub fn create_board(
  db: sqlight.Connection,
  board: Board,
) -> Result(Nil, sqlight.Error) {
  let sql = "INSERT INTO boards (id, name, description) VALUES (?1, ?2, ?3)"
  sqlight.query(
    sql,
    on: db,
    with: [
      sqlight.text(board.id),
      sqlight.text(board.name),
      sqlight.text(board.description),
    ],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}

// -- User queries --

pub fn find_user_by_username(
  db: sqlight.Connection,
  username: String,
) -> Result(User, Nil) {
  let sql = "SELECT id, username, password_hash FROM users WHERE username = ?1"
  case
    sqlight.query(
      sql,
      on: db,
      with: [sqlight.text(username)],
      expecting: user_decoder(),
    )
  {
    Ok([user]) -> Ok(user)
    _ -> Error(Nil)
  }
}

// -- Decoders --

fn task_decoder() -> decode.Decoder(Task) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  use due_date_raw <- decode.field(4, decode.string)
  let due_date = case due_date_raw {
    "" -> option.None
    date -> option.Some(date)
  }
  decode.success(Task(
    id: id,
    title: title,
    description: description,
    done: done,
    due_date: due_date,
  ))
}

fn board_decoder() -> decode.Decoder(Board) {
  use id <- decode.field(0, decode.string)
  use name <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  decode.success(Board(id: id, name: name, description: description))
}

fn user_decoder() -> decode.Decoder(User) {
  use id <- decode.field(0, decode.string)
  use username <- decode.field(1, decode.string)
  use password_hash <- decode.field(2, decode.string)
  decode.success(User(id: id, username: username, password_hash: password_hash))
}
```

### `src/teamwork/web.gleam`

```gleam
import gleam/bool
import gleam/http
import gleam/list
import gleam/option
import gleam/order
import gleam/result
import gleam/string
import hx
import lustre/attribute
import lustre/element
import lustre/element/html
import sqlight
import wisp
import teamwork/date_helpers
import teamwork/db

pub type Context {
  Context(db: sqlight.Connection, secret_key_base: String)
}

// -- Layout --

pub fn layout(content: element.Element(Nil)) -> element.Element(Nil) {
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
      // Flatpickr CSS
      html.link([
        attribute.rel("stylesheet"),
        attribute.href(
          "https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css",
        ),
      ]),
      // HTMX (self-hosted)
      html.script([attribute.src("/static/js/htmx.min.js")], ""),
      // Flatpickr JS
      html.script(
        [attribute.src("https://cdn.jsdelivr.net/npm/flatpickr")],
        "",
      ),
      // _hyperscript
      html.script(
        [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
        "",
      ),
    ]),
    html.body([hx.boost(True)], [content]),
  ])
}

// -- Task form --

pub fn task_form(board_id: String) -> element.Element(Nil) {
  html.form(
    [
      attribute.id("task-form"),
      attribute.method("post"),
      attribute.action("/boards/" <> board_id <> "/tasks"),
      hx.post("/boards/" <> board_id <> "/tasks"),
      hx.target(hx.Selector("#task-list")),
      hx.swap(hx.Beforeend),
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
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("due_date")], [element.text("Due date")]),
        html.input([
          attribute.type_("text"),
          attribute.name("due_date"),
          attribute.id("due_date"),
          attribute.placeholder("Pick a date (optional)"),
          attribute.attribute(
            "_",
            "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
          ),
        ]),
      ]),
      html.button([attribute.type_("submit")], [element.text("Add Task")]),
    ],
  )
}

// -- Date range filter --

pub fn filter_bar(board_id: String) -> element.Element(Nil) {
  html.div([attribute.class("filter-bar")], [
    html.label([], [
      element.text("From: "),
      html.input([
        attribute.type_("text"),
        attribute.name("from_date"),
        attribute.class("filter-date"),
        attribute.placeholder("Start date"),
        attribute.attribute(
          "_",
          "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
        ),
        hx.get("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.InnerHTML),
        hx.trigger([hx.change()]),
        hx.include(hx.Selector(".filter-bar")),
      ]),
    ]),
    html.label([], [
      element.text("To: "),
      html.input([
        attribute.type_("text"),
        attribute.name("to_date"),
        attribute.class("filter-date"),
        attribute.placeholder("End date"),
        attribute.attribute(
          "_",
          "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
        ),
        hx.get("/boards/" <> board_id <> "/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.InnerHTML),
        hx.trigger([hx.change()]),
        hx.include(hx.Selector(".filter-bar")),
      ]),
    ]),
  ])
}

// -- Task item --

pub fn task_item(task: db.Task, today: String) -> element.Element(Nil) {
  let is_overdue = case task.due_date {
    option.Some(date) ->
      string.compare(date, today) == order.Lt && bool.negate(task.done)
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
      html.span([attribute.class("task-title")], [element.text(task.title)]),
      due_date_badge(task.due_date, is_overdue),
    ],
  )
}

fn due_date_badge(
  due_date: option.Option(String),
  is_overdue: Bool,
) -> element.Element(Nil) {
  case due_date {
    option.None -> element.none()
    option.Some(date) -> {
      let class = case is_overdue {
        True -> "due-date overdue"
        False -> "due-date"
      }
      html.span([attribute.class(class)], [element.text(date)])
    }
  }
}

// -- Request handlers --

pub fn handle_tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  case req.method {
    http.Get -> list_tasks(req, ctx, board_id)
    http.Post -> create_task(req, ctx, board_id)
    _ -> wisp.method_not_allowed([http.Get, http.Post])
  }
}

fn list_tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  let assert Ok(tasks) = db.list_tasks(ctx.db, board_id)
  let today = date_helpers.today()

  let query_params = wisp.get_query(req)

  let from_date =
    list.key_find(query_params, "from_date")
    |> result.unwrap("")

  let to_date =
    list.key_find(query_params, "to_date")
    |> result.unwrap("")

  let filtered = filter_by_date_range(tasks, from_date, to_date)

  let task_html =
    element.fragment(
      list.map(filtered, fn(task) { task_item(task, today) }),
    )

  wisp.html_response(element.to_string(task_html), 200)
}

fn create_task(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let assert Ok(title) = list.key_find(form_data.values, "title")
  let due_date = case list.key_find(form_data.values, "due_date") {
    Ok("") -> option.None
    Ok(date) -> option.Some(date)
    Error(_) -> option.None
  }

  let task_id = wisp.random_string(16)
  let task =
    db.Task(
      id: task_id,
      title: title,
      description: "",
      done: False,
      due_date: due_date,
    )

  let assert Ok(_) = db.create_task(ctx.db, board_id, task)

  let today = date_helpers.today()
  wisp.html_response(element.to_string(task_item(task, today)), 201)
}

// -- Date range filtering --

fn filter_by_date_range(
  tasks: List(db.Task),
  from_date: String,
  to_date: String,
) -> List(db.Task) {
  list.filter(tasks, fn(task) {
    case task.due_date {
      option.None -> from_date == "" && to_date == ""
      option.Some(date) -> {
        let after_from = case from_date {
          "" -> True
          from -> string.compare(date, from) != order.Lt
        }
        let before_to = case to_date {
          "" -> True
          to -> string.compare(date, to) != order.Gt
        }
        after_from && before_to
      }
    }
  })
}
```

### `priv/static/css/style.css` (additions)

```css
/* ============================================================
   Due dates
   ============================================================ */

/* Due date badge */
.due-date {
  display: inline-block;
  padding: 2px 8px;
  margin-left: 8px;
  font-size: 0.85em;
  color: #555;
  background-color: #f0f0f0;
  border-radius: 4px;
}

.due-date.overdue {
  color: #b91c1c;
  background-color: #fee2e2;
  font-weight: 600;
}

/* Overdue task row */
.task-item.overdue {
  border-left: 3px solid #b91c1c;
}

/* Filter bar */
.filter-bar {
  display: flex;
  gap: 16px;
  align-items: center;
  margin-bottom: 16px;
  padding: 12px;
  background-color: #f9fafb;
  border-radius: 6px;
}

.filter-bar label {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 0.9em;
  color: #374151;
}

.filter-date {
  width: 160px;
  padding: 6px 10px;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 0.9em;
}
```

---

## 4. How It All Fits Together

Here is the complete picture, showing two scenarios.

**Scenario 1: Happy path -- creating a task with a due date**

```
Browser                              Server
  │                                    │
  │  Page loads                        │
  │  ──────────────────────────────►   │
  │                                    │  Returns full page HTML
  │  ◄──────────────────────────────   │
  │                                    │
  │  htmx:load fires on <body>        │
  │  _hyperscript sees "on load"      │
  │  flatpickr() called on input      │
  │  Calendar is ready                 │
  │                                    │
  │  User picks "2026-03-15"           │
  │  User types task title             │
  │  User submits form                 │
  │                                    │
  │  POST /boards/abc/tasks            │
  │  title=Clean+up&due_date=2026-03-15│
  │  ──────────────────────────────►   │
  │                                    │  Parses form data
  │                                    │  Stores task in SQLite
  │                                    │  Returns task_item HTML
  │  ◄──────────────────────────────   │
  │                                    │
  │  HTMX swaps response into         │
  │  #task-list (Beforeend)            │
  │  Task appears with "2026-03-15"    │
  │  badge                             │
```

**Scenario 2: Swap re-initialization -- form error and retry**

```
Browser                              Server
  │                                    │
  │  User submits form with empty      │
  │  title (validation error)          │
  │                                    │
  │  POST /boards/abc/tasks            │
  │  title=&due_date=2026-03-15        │
  │  ──────────────────────────────►   │
  │                                    │  Validation fails
  │                                    │  Returns form HTML with errors
  │  ◄──────────────────────────────   │
  │                                    │
  │  HTMX swaps form (OuterHTML)       │
  │  Old <input name="due_date">       │
  │    is DESTROYED                    │
  │  New <input name="due_date">       │
  │    has NO calendar                 │
  │                                    │
  │  htmx:load fires on new elements  │
  │  _hyperscript sees "on load"      │
  │    on the new input                │
  │  flatpickr() called               │
  │  Calendar works again!             │
```

The key insight is this: **`htmx:load` is the universal integration point for all DOM-attaching JavaScript libraries.** It fires on initial page load and on every swap. If you initialize your library in response to `htmx:load`, it will always be initialized, regardless of how the content arrived in the DOM.

This pattern generalizes to any library:

```
htmx:load → find elements that need initialization → initialize the library
```

Whether the library is Flatpickr, Sortable.js, Chart.js, CodeMirror, or anything else, the pattern is the same. The only thing that changes is the initialization call.

---

## 5. Exercises

### Exercise 1: Add Time Selection

Enable time selection in the Flatpickr configuration. Tasks should be submittable with both a date and a time.

1. Add `enableTime: true` and update `dateFormat` to `"Y-m-d H:i"`.
2. Update `altFormat` to include the time: `"M j, Y h:i K"` (e.g. "Mar 15, 2026 2:30 PM").
3. Update the server to handle the longer date-time string.
4. Update the task display to show the time alongside the date.

**Acceptance criteria:** Tasks can be created with a date and time. The task list shows both. Overdue detection still works (compare the full date-time string).

### Exercise 2: Disable Past Dates and Weekends

Configure the task form's Flatpickr instance to prevent selecting past dates and weekends.

1. Add `minDate: "today"` to the Flatpickr configuration.
2. Add a `disable` function that returns `true` for Saturday (day 6) and Sunday (day 0).
3. Verify that past dates and weekends are grayed out in the calendar.

**Acceptance criteria:** The calendar shows past dates and weekends as unselectable (grayed out). Clicking them does nothing.

### Exercise 3: Apply a Flatpickr Theme

Replace the default Flatpickr CSS with a theme that matches your application's design.

1. Browse the themes at `flatpickr.js.org/themes/`.
2. Change the CSS `<link>` in your layout to load a different theme (e.g. `dark.css` or `material_blue.css`).
3. No JavaScript changes needed.

**Acceptance criteria:** The calendar popup uses the new theme's colors and styling.

### Exercise 4: Inline Calendar Sidebar

Add a persistently visible calendar in the sidebar (or above the task list) using Flatpickr's `inline: true` mode. Clicking a date filters the task list to show only tasks due on that date.

1. Add a `<div>` with a hidden `<input>` and `inline: true`.
2. Use the `onChange` callback to trigger an HTMX request.
3. Highlight dates that have tasks due (use the `onDayCreate` callback to add dots or badges).

**Acceptance criteria:** A calendar is always visible. Clicking a date shows only tasks due on that date.

### Exercise 5: Clear Filter Button

Add a "Clear" button to the date range filter bar that resets both Flatpickr inputs and reloads the unfiltered task list.

1. Add a button to the filter bar.
2. When clicked, call `.clear()` on both Flatpickr instances to reset their values.
3. Trigger a request to reload the task list without date filters.

**Acceptance criteria:** Clicking "Clear" empties both date inputs and shows all tasks again.

---

## 6. Exercise Solution Hints

Try each exercise on your own before reading these hints.

### Hint for Exercise 1

Update the `_hyperscript` attribute on the input:

```
on load call flatpickr(me, {dateFormat: 'Y-m-d H:i', altInput: true, altFormat: 'M j, Y h:i K', enableTime: true})
```

The server does not need to change much. The `due_date` string is now `"2026-03-15 14:30"` instead of `"2026-03-15"`. The `string.compare` comparison still works for overdue detection because `"2026-03-15 14:30"` sorts before `"2026-03-16 09:00"` lexicographically.

### Hint for Exercise 2

The `disable` option takes an array. Functions in the array receive a `Date` object and return `true` for dates that should be disabled:

```
on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y', minDate: 'today', disable: [function(date) { return date.getDay() === 0 || date.getDay() === 6; }]})
```

This is getting long for a `_hyperscript` attribute. If it feels unwieldy, switch to the global JavaScript listener approach instead and keep the config in a `<script>` block.

### Hint for Exercise 3

Change only the CSS link in your layout function:

```gleam
html.link([
  attribute.rel("stylesheet"),
  attribute.href(
    "https://cdn.jsdelivr.net/npm/flatpickr/dist/themes/dark.css",
  ),
])
```

Available theme filenames: `dark.css`, `material_blue.css`, `material_green.css`, `material_orange.css`, `material_red.css`, `airbnb.css`, `confetti.css`.

### Hint for Exercise 4

For inline mode, use a container `<div>` and initialize Flatpickr on a child input:

```gleam
html.div([attribute.class("calendar-sidebar")], [
  html.input([
    attribute.type_("hidden"),
    attribute.name("filter_date"),
    attribute.attribute(
      "_",
      "on load call flatpickr(me, {inline: true, dateFormat: 'Y-m-d', onChange: function(dates, str) { htmx.ajax('GET', '/boards/abc/tasks?from_date=' + str + '&to_date=' + str, '#task-list') }})",
    ),
  ]),
])
```

The `onChange` callback uses `htmx.ajax()` to trigger a request programmatically. This is one of the few cases where calling HTMX from JavaScript is the right approach.

### Hint for Exercise 5

Give each Flatpickr input an `id` so you can find its instance:

```javascript
// In a script block or _hyperscript
var from = document.getElementById("from_date")._flatpickr;
var to = document.getElementById("to_date")._flatpickr;
from.clear();
to.clear();
```

For the clear button, you can use `_hyperscript`:

```html
<button _="on click
  set fromPicker to #from_date._flatpickr
  set toPicker to #to_date._flatpickr
  call fromPicker.clear()
  call toPicker.clear()
  then trigger change on #from_date">
  Clear
</button>
```

The `trigger change` at the end fires the `change` event that HTMX is listening for, which reloads the task list with no filters.

---

## 7. Key Takeaways

1. **Third-party JavaScript libraries work with HTMX, but they need re-initialization after swaps.** Any library that attaches state, event listeners, or DOM nodes to elements will lose that state when HTMX replaces the elements. This is the most common integration issue.

2. **Flatpickr matches HTMX's no-build-step philosophy.** At ~16 KB with zero dependencies, it loads from two CDN tags and requires no bundler, no transpiler, and no configuration file.

3. **`htmx:load` fires on initial page load AND on every swap.** This single event is all you need to listen for. It tells you "new content is in the DOM -- initialize your libraries now."

4. **`_hyperscript`'s `on load` is a one-line solution for re-initialization.** Writing `_="on load call flatpickr(me, {...})"` directly on the element is self-contained and travels with the element across swaps.

5. **Guard against double initialization.** Check `el._flatpickr` (or the equivalent for other libraries) before initializing. Without this guard, you risk creating duplicate instances when an element is not actually swapped but `htmx:load` fires on a parent.

6. **`altInput` separates human-readable display from machine-readable submission.** The user sees "Mar 15, 2026" while the form submits `"2026-03-15"`. This is a clean separation that avoids date parsing on the server.

7. **ISO date strings sort correctly as plain strings.** Lexicographic comparison of ISO dates produces correct chronological ordering. In SQLite, `WHERE due_date >= '2026-03-01'` works directly. In Gleam, use `string.compare(date_a, date_b)` which returns `order.Lt`, `order.Eq`, or `order.Gt`. No date parsing library needed.

8. **`hx-include` lets filter inputs outside the form participate in requests.** The date range filter inputs are not inside the task form, but `hx.include(hx.Selector(".filter-bar"))` ensures their values are sent with every filter request.

9. **The pattern generalizes to any DOM-attaching library.** Sortable.js, Chart.js, CodeMirror, tooltip libraries -- they all follow the same pattern: load via CDN, initialize on `htmx:load` (or `_hyperscript` `on load`), guard against double initialization.

---

## What's Next

You now have a general pattern for integrating any third-party JavaScript library with HTMX:

1. **Load** the library via CDN (or self-hosted).
2. **Initialize** on `htmx:load` (global listener) or `_hyperscript` `on load` (inline).
3. **Guard** against double initialization.

This pattern works for date pickers, drag-and-drop libraries, rich text editors, charting libraries, mapping libraries, and anything else that attaches to DOM elements.

The next section -- **Real-World Patterns ([Chapters 19-28](../04-real-world-patterns/19-error-handling-and-graceful-degradation.md))** -- builds on
everything you have learned so far. It covers error handling, response headers,
inline editing, modal dialogs, `_hyperscript` recipes, dynamic forms, file
uploads, accessibility, database performance, and background jobs. These are the
patterns you will reach for daily in production applications.

Continue to **[Chapter 19](../04-real-world-patterns/19-error-handling-and-graceful-degradation.md) -- Error Handling and Graceful Degradation** to begin
the Real-World Patterns section.

For more on `_hyperscript`, see **[Appendix B](../05-appendices/A2-hyperscript.md)**. For documentation links to all
tools used in this course, see **[Appendix A](../05-appendices/A1-bibliography-and-resources.md)**.
