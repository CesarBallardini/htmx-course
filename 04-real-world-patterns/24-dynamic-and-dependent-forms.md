# Chapter 24 -- Dynamic and Dependent Forms

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapter 23 explored practical `_hyperscript` patterns for
client-side micro-interactions. We used `_hyperscript` to toggle classes,
handle drag events, and animate elements without writing standalone
JavaScript files.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Build a cascading select (category -> subcategory) where changing the first
   fetches options for the second from the server.
2. Use `hx-get` with `hx-trigger="change"` and `hx-include` to fetch dependent
   field values based on a selection.
3. Implement conditional form sections that appear or disappear based on a
   checkbox or dropdown selection.
4. Build an "add another" pattern for repeating field groups using `Beforeend`
   swap.
5. Handle variable-length field submissions with indexed names (`subtask[0]`,
   `subtask[1]`) and parse them on the server.
6. Preserve form state across partial updates using `hx-include` so that
   changing one field does not wipe out the values of others.

---

## 1. Theory

### 1.1 Why Server-Rendered Dynamic Forms

Most web applications eventually need forms that change shape based on user
input. Pick "Bug report" as the issue type, and new fields appear for
severity, steps to reproduce, and affected version. Choose "United States"
as the country, and the state dropdown fills with fifty options. Check "Set
a due date", and a date picker slides into view.

In a JavaScript-heavy application, this logic lives in the browser. You
write event handlers that show, hide, and populate fields. You duplicate
validation rules between client and server. You ship a bundle of JS that
encodes business logic the server already knows.

With HTMX, you take a different approach: **the server renders the fields**.
When the user changes a dropdown, HTMX sends the selected value to the
server. The server decides which fields to show, validates the selection,
and returns the appropriate HTML fragment. HTMX swaps it into the page.

This gives you three advantages:

1. **Single source of truth.** The server defines which subcategories belong
   to which category. There is no JSON blob duplicated in the frontend.
2. **Validation in one place.** If a category is retired, the server simply
   stops returning it. No client-side code to update.
3. **Gleam, not JavaScript.** Your form logic is type-checked Gleam code
   running on the BEAM. You get pattern matching, exhaustive case checks,
   and the safety guarantees Gleam provides. No runtime surprises in the
   browser.

The trade-off is a network round trip for every dynamic change. In practice,
this round trip is fast -- often under 50 milliseconds on a local network,
and under 200 milliseconds over the public internet. For form interactions
where the user pauses to think (selecting a category, checking a box), this
latency is imperceptible. And HTMX gives you indicators to handle the rare
case where it is not.

### 1.2 The Cascading Select Pattern

A cascading select is a pair of dropdown menus where the second depends on
the first. The classic example is country -> city, but any parent-child
relationship works: department -> team, make -> model, category ->
subcategory.

Here is the pattern expressed in HTMX attributes:

```
First <select>:
  - hx-get="/subcategories"
  - hx-trigger="change"
  - hx-target="#subcategory"
  - hx-swap="innerHTML"
  - hx-include="this"

Second <select>:
  - id="subcategory"
  - Starts empty or with a placeholder option
  - Gets its <option> elements from the server
```

When the user changes the first select:

1. HTMX fires because of `hx-trigger="change"`.
2. It sends a GET request to `/subcategories`, including the first select's
   value (because of `hx-include="this"` -- HTMX includes the triggering
   element's `name` and `value` as a query parameter).
3. The server reads the selected category, looks up matching subcategories,
   and returns a list of `<option>` elements.
4. HTMX swaps those options into the second select (`hx-swap="innerHTML"`
   on `hx-target="#subcategory"`), replacing whatever was there before.

The second select now shows only the subcategories that belong to the
selected category. No JavaScript arrays, no `JSON.parse`, no manual DOM
manipulation.

An important detail: `hx-include` with `hx.This` (targeting the triggering
element) or `hx.Selector(...)` (targeting by CSS selector) ensures the
server knows *which* category was selected. Without it, the GET request
would have no parameters and the server would not know what to look up.

### 1.3 Conditional Form Sections

Sometimes you want an entire section of a form to appear or disappear based
on a user's choice. "Set a due date?" -- if checked, show the date picker.
"Add shipping address?" -- if checked, reveal the address fields.

There are two approaches, and each is appropriate in different situations.

**Approach A: Server-fetched sections.** The checkbox or dropdown triggers
an `hx-get` request. The server returns the section HTML (or an empty
string to hide it). HTMX swaps the result into a container div.

This is the right choice when the conditional section is complex, when the
server needs to populate it with data (e.g., pre-filling address fields
from a saved profile), or when you want the server to enforce which sections
are available.

**Approach B: `_hyperscript` toggle.** The checkbox uses
`_="on click toggle .hidden on #due-date-section"` to show or hide a
section that is already in the DOM.

This is the right choice when the section is simple, contains no
server-dependent data, and you want instant feedback with zero network
latency. It is purely a visibility toggle -- no data fetching involved.

In this chapter we will use Approach A for the due-date section because
it involves Flatpickr initialization (which we covered in Chapter 18),
and the server needs to render the Flatpickr input with the correct
`_hyperscript` init attribute. But we will also show Approach B as an
alternative for simpler cases.

### 1.4 Repeating Field Groups

Some forms need a variable number of identical field groups. Think subtasks
on a task, line items on an invoice, or phone numbers on a contact form. The
user clicks "Add another" and a new row appears with empty inputs.

The HTMX pattern for this is straightforward:

1. A container `<div>` holds the existing field groups.
2. An "Add" button has `hx-get` pointing to an endpoint that returns a
   single new field group.
3. The button uses `hx-swap="beforeend"` and `hx-target` pointing at the
   container, so the new group is appended at the end.
4. Each field group has a "Remove" button. Since removal is a purely visual
   operation (we do not need the server to know about it until the form is
   submitted), we use `_hyperscript`: `_="on click remove closest .subtask-row"`.
5. The server endpoint increments an index so each new field has a unique
   name: `subtask[0]`, `subtask[1]`, `subtask[2]`, etc.

The index is important for two reasons. First, it ensures every input has a
unique `name` attribute, which is required for form submission to work
correctly. Second, it gives the server a way to reconstruct the list order
when parsing the submitted form data.

A subtlety: the "Add" button needs to know the *next* index. You can pass
it as a query parameter (`/tasks/subtask-field?index=3`), or the server
can track it, or you can use `_hyperscript` to count the existing rows.
We will use the query parameter approach because it keeps the server
stateless and the logic transparent.

### 1.5 Preserving State with `hx-include`

When HTMX sends a request from one element, it does not automatically
include the values of other form fields (unless they are inside the same
`<form>` element and it is the form being submitted). This means that if
your category select triggers a request to fetch subcategories, the values
of other fields in the form (like a title input or a description textarea)
are not sent along.

Most of the time, this is fine -- the server does not need those values
just to render subcategory options. But there are situations where you want
to preserve them:

- When the server returns a larger chunk of the form (not just the dependent
  field) and needs to populate the other fields with their current values.
- When the conditional section depends on multiple fields simultaneously.
- When you want the server to validate partial form state as the user fills
  things in.

For these cases, `hx-include` is your tool. On the triggering element, add
`hx.include(hx.Selector("form"))` or target specific fields by name:
`hx.include(hx.Selector("[name='title'], [name='description']"))`.

The included values travel with the request as query parameters (for GET)
or form body parameters (for POST). The server can read them, use them for
rendering decisions, and pass them back as `value` attributes in the
response so the user's input is preserved.

This is the same `hx-include` pattern we used in Chapter 10 for coordinating
the search box and filter tabs. The principle is the same: when multiple
pieces of state need to travel together, `hx-include` bundles them into a
single request.

### 1.6 Parsing Dynamic Fields in Gleam

When a form has a variable number of fields with indexed names like
`subtask[0]`, `subtask[1]`, `subtask[2]`, the submitted form data arrives
as:

```
subtask[0]=Design+mockups&subtask[1]=Write+tests&subtask[2]=Deploy
```

Wisp's `wisp.require_form` parses this into:

```gleam
[
  #("subtask[0]", "Design mockups"),
  #("subtask[1]", "Write tests"),
  #("subtask[2]", "Deploy"),
]
```

The keys are plain strings -- Wisp does not interpret the bracket notation.
To extract all subtask values, you filter the form data by key prefix:

```gleam
let subtasks =
  list.filter(form_data.values, fn(pair) {
    string.starts_with(pair.0, "subtask[")
  })
  |> list.map(fn(pair) { pair.1 })
```

This gives you a `List(String)` of subtask values in submission order. The
`string.starts_with` function from `gleam/string` checks if a string begins
with a given prefix. We use `pair.0` and `pair.1` to access the first and
second elements of each tuple.

If you need to preserve the original order (which is the order the browser
serializes them), the list already comes in order from `wisp.require_form`.
If you need explicit ordering, you could parse the index from the key:

```gleam
let subtasks =
  form_data.values
  |> list.filter(fn(pair) { string.starts_with(pair.0, "subtask[") })
  |> list.sort(fn(a, b) { string.compare(a.0, b.0) })
  |> list.map(fn(pair) { pair.1 })
```

The `string.compare` call sorts by key name, which sorts lexicographically.
For indices under 10, this works perfectly. For larger forms (10+ rows),
you would need numeric parsing. In practice, most repeating field groups
stay well under ten items.

You should also filter out empty values. If the user clicks "Add another"
three times but only fills in two of them, you do not want to store a blank
subtask:

```gleam
let subtasks =
  form_data.values
  |> list.filter(fn(pair) { string.starts_with(pair.0, "subtask[") })
  |> list.map(fn(pair) { string.trim(pair.1) })
  |> list.filter(fn(value) { value != "" })
```

This pipeline filters by prefix, extracts and trims the values, then drops
any empty strings. Clean, declarative, and fully type-safe.

---

## 2. Code Walkthrough

We are going to extend the Teamwork task creation form with four new
capabilities:

1. A cascading category/subcategory select.
2. A conditional "Set due date" section with Flatpickr.
3. Repeating subtask fields with add/remove.
4. Server-side parsing of all these dynamic fields.

By the end, the "Create Task" form will look like this:

```
 ┌─────────────────────────────────────────────────────────┐
 │  Create Task                                            │
 │                                                         │
 │  Title:        [________________________]               │
 │                                                         │
 │  Category:     [ Work           ▼]                      │
 │  Subcategory:  [ Code review    ▼]                      │
 │                                                         │
 │  ☑ Set due date                                         │
 │  Due date:     [ Mar 15, 2026   📅]                     │
 │                                                         │
 │  Subtasks:                                              │
 │    [Design mockups___________] [Remove]                 │
 │    [Write tests______________] [Remove]                 │
 │                             [+ Add subtask]             │
 │                                                         │
 │                             [Create Task]               │
 └─────────────────────────────────────────────────────────┘
```

### Step 1 -- Category and Subcategory Data

Before we build any UI, we need data. In a real application, categories and
subcategories would come from a database. For our walkthrough, we will define
them as module-level constants. Create (or update) your data module:

```gleam
// src/teamwork/categories.gleam

import gleam/list

pub type Category {
  Category(value: String, label: String, subcategories: List(Subcategory))
}

pub type Subcategory {
  Subcategory(value: String, label: String)
}

pub fn all_categories() -> List(Category) {
  [
    Category(value: "work", label: "Work", subcategories: [
      Subcategory(value: "code-review", label: "Code review"),
      Subcategory(value: "bug-fix", label: "Bug fix"),
      Subcategory(value: "feature", label: "Feature"),
      Subcategory(value: "documentation", label: "Documentation"),
    ]),
    Category(value: "personal", label: "Personal", subcategories: [
      Subcategory(value: "health", label: "Health"),
      Subcategory(value: "errands", label: "Errands"),
      Subcategory(value: "finance", label: "Finance"),
    ]),
    Category(value: "home", label: "Home", subcategories: [
      Subcategory(value: "cleaning", label: "Cleaning"),
      Subcategory(value: "repair", label: "Repair"),
      Subcategory(value: "shopping", label: "Shopping"),
      Subcategory(value: "garden", label: "Garden"),
    ]),
  ]
}

pub fn subcategories_for(category_value: String) -> List(Subcategory) {
  case list.find(all_categories(), fn(c) { c.value == category_value }) {
    Ok(category) -> category.subcategories
    Error(_) -> []
  }
}
```

The `subcategories_for` function takes a category value string (like
`"work"`) and returns the matching subcategories. If the category does not
exist, it returns an empty list. This is the function our endpoint will call
when the user changes the category dropdown.

Note that `list.find` searches for the first element matching a predicate
and returns a `Result`. This is different from `list.key_find`, which
searches a list of tuples by the first element. We use `list.find` here
because our list contains `Category` records, not tuples.

### Step 2 -- The Cascading Select View

Now let us build the category and subcategory dropdowns. The category
select triggers a request when its value changes, and the subcategory
select receives the response:

```gleam
// In your views module

import gleam/list
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import hx
import teamwork/categories.{type Category, type Subcategory}

fn category_select(
  all_cats: List(Category),
  selected: String,
) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label(
      [attribute.for("category")],
      [element.text("Category")],
    ),
    html.select(
      [
        attribute.name("category"),
        attribute.id("category"),
        attribute.class("input"),
        hx.get("/tasks/subcategories"),
        hx.trigger([hx.change()]),
        hx.target(hx.Selector("#subcategory")),
        hx.swap(hx.InnerHTML),
        hx.include(hx.This),
      ],
      [
        html.option(
          [attribute.value(""), attribute.selected(selected == "")],
          "Select a category...",
        ),
        ..list.map(all_cats, fn(cat) {
          html.option(
            [
              attribute.value(cat.value),
              attribute.selected(cat.value == selected),
            ],
            cat.label,
          )
        })
      ],
    ),
  ])
}

fn subcategory_select(
  subcats: List(Subcategory),
  selected: String,
) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label(
      [attribute.for("subcategory")],
      [element.text("Subcategory")],
    ),
    html.select(
      [
        attribute.name("subcategory"),
        attribute.id("subcategory"),
        attribute.class("input"),
      ],
      case subcats {
        [] -> [
          html.option(
            [attribute.value("")],
            "Select a category first",
          ),
        ]
        _ -> [
          html.option(
            [attribute.value(""), attribute.selected(selected == "")],
            "Select a subcategory...",
          ),
          ..list.map(subcats, fn(sub) {
            html.option(
              [
                attribute.value(sub.value),
                attribute.selected(sub.value == selected),
              ],
              sub.label,
            )
          })
        ]
      },
    ),
  ])
}
```

Walk through the category select's HTMX attributes:

- **`hx.get("/tasks/subcategories")`** -- When triggered, send a GET request
  to `/tasks/subcategories`.
- **`hx.trigger([hx.change()])`** -- Fire on the `change` event, which fires
  when the user selects a different option. Note the list syntax: `hx.trigger`
  takes a list of typed events, not a string.
- **`hx.target(hx.Selector("#subcategory"))`** -- Swap the response into the
  element with `id="subcategory"`, which is our second `<select>`.
- **`hx.swap(hx.InnerHTML)`** -- Replace the inner HTML (the `<option>`
  elements) of the target. The `<select>` element itself remains -- only
  its children change.
- **`hx.include(hx.This)`** -- Include the triggering element's value in
  the request. This sends `?category=work` (or whatever the user selected)
  as a query parameter. Without this, the server would not know which
  category was selected.

The subcategory select is simpler -- it has no HTMX attributes because it
does not trigger any requests. It is purely a receiver. When the page first
loads and no category is selected, it shows a placeholder message. Once the
user selects a category and the server returns subcategory options, those
replace the placeholder.

Notice how `html.option` takes a list of attributes and a `String` label
(not a list of children). This is a Lustre API detail worth remembering:
`html.option([attribute.value("work"), attribute.selected(True)], "Work")`.

### Step 3 -- The Subcategory Endpoint

The endpoint that serves subcategory options reads the selected category
from the query parameters and returns `<option>` elements:

```gleam
// In your router module

import gleam/list
import gleam/result
import lustre/attribute
import lustre/element
import lustre/element/html
import wisp
import teamwork/categories

fn get_subcategories(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let category_value =
    list.key_find(query_params, "category")
    |> result.unwrap("")

  let subcats = categories.subcategories_for(category_value)

  let options = case subcats {
    [] -> [
      html.option([attribute.value("")], "No subcategories available"),
    ]
    _ -> [
      html.option([attribute.value("")], "Select a subcategory..."),
      ..list.map(subcats, fn(sub) {
        html.option([attribute.value(sub.value)], sub.label)
      })
    ]
  }

  let html_string =
    element.fragment(options)
    |> element.to_string

  wisp.html_response(html_string, 200)
}
```

This endpoint does three things:

1. Reads the `category` query parameter from the request.
2. Looks up matching subcategories using `categories.subcategories_for`.
3. Renders `<option>` elements and returns them as an HTML fragment.

The response is *just* the `<option>` elements, not a full `<select>` or a
full page. This is because the category select has `hx-swap="innerHTML"` --
it replaces the inner content of the subcategory `<select>`, not the select
element itself.

We use `element.fragment(options)` to wrap multiple elements into a single
renderable unit, then `element.to_string` (not `to_document_string`) because
this is a fragment, not a full page.

Wire this into your router:

```gleam
fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: ctx.static)

  case wisp.path_segments(req) {
    // ... existing routes ...
    ["tasks", "subcategories"] -> {
      case req.method {
        http.Get -> get_subcategories(req)
        _ -> wisp.method_not_allowed([http.Get])
      }
    }
    // ... other routes ...
  }
}
```

When the user selects "Work" from the category dropdown, here is what
happens:

```
Browser                                Server
  |                                       |
  |  User selects "Work" from category    |
  |                                       |
  |  GET /tasks/subcategories?category=work
  |  HX-Request: true                     |
  |-------------------------------------->|
  |                                       |  wisp.get_query(req)
  |                                       |    -> [#("category", "work")]
  |                                       |  categories.subcategories_for("work")
  |                                       |    -> [CodeReview, BugFix, Feature, Docs]
  |                                       |  Render <option> elements
  |                                       |
  |  200 OK                               |
  |  <option value="">Select...</option>  |
  |  <option value="code-review">         |
  |    Code review</option>               |
  |  <option value="bug-fix">             |
  |    Bug fix</option>                   |
  |  <option value="feature">             |
  |    Feature</option>                   |
  |  <option value="documentation">       |
  |    Documentation</option>             |
  |<--------------------------------------|
  |                                       |
  HTMX replaces innerHTML of #subcategory
  with the new <option> elements.

  The subcategory dropdown now shows:
  [ Select a subcategory... ▼]
    Code review
    Bug fix
    Feature
    Documentation
```

No JavaScript arrays. No client-side filtering. The server is the single
source of truth for which subcategories belong to which category.

### Step 4 -- Conditional Due Date Section

The "Set due date" checkbox, when checked, reveals a Flatpickr date input.
When unchecked, the date section disappears. We will use the server-fetched
approach (Approach A from section 1.3) because the Flatpickr input needs
proper initialization.

First, the checkbox:

```gleam
fn due_date_checkbox(is_checked: Bool) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label([attribute.class("checkbox-label")], [
      html.input([
        attribute.type_("checkbox"),
        attribute.name("has_due_date"),
        attribute.value("true"),
        attribute.checked(is_checked),
        hx.get("/tasks/due-date-section"),
        hx.trigger([hx.change()]),
        hx.target(hx.Selector("#due-date-container")),
        hx.swap(hx.InnerHTML),
        hx.include(hx.This),
      ]),
      element.text(" Set due date"),
    ]),
  ])
}
```

The checkbox includes `hx.include(hx.This)` so the server knows whether
the box is checked or unchecked. When a checkbox is checked, its `name`
and `value` are included in the request (`?has_due_date=true`). When it
is unchecked, the browser does not include it at all -- the absence of the
parameter tells the server the box is unchecked.

Next, the container and the date section itself:

```gleam
fn due_date_container(show: Bool, current_date: String) -> Element(t) {
  html.div([attribute.id("due-date-container")], [
    case show {
      True -> due_date_section(current_date)
      False -> element.none()
    },
  ])
}

fn due_date_section(current_date: String) -> Element(t) {
  html.div([attribute.class("form-group due-date-section")], [
    html.label(
      [attribute.for("due_date")],
      [element.text("Due date")],
    ),
    html.input([
      attribute.type_("text"),
      attribute.name("due_date"),
      attribute.id("due_date"),
      attribute.value(current_date),
      attribute.class("input"),
      attribute.placeholder("Pick a date..."),
      attribute.attribute(
        "_",
        "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
      ),
    ]),
  ])
}
```

The `due_date_section` function renders a text input with a `_hyperscript`
attribute that initializes Flatpickr when the element is loaded into the
DOM. This is the same pattern from Chapter 18: `on load call flatpickr(me, {...})`.
Every time HTMX swaps this element into the page, `_hyperscript` fires the
`load` event and Flatpickr initializes on the new input.

The endpoint that serves this section:

```gleam
fn get_due_date_section(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let has_due_date =
    list.key_find(query_params, "has_due_date")
    |> result.unwrap("")
    == "true"

  let html = case has_due_date {
    True -> due_date_section("")
    False -> element.none()
  }

  wisp.html_response(element.to_string(html), 200)
}
```

When the checkbox is checked, the server returns the date input with
Flatpickr initialization. When unchecked, it returns nothing
(`element.none()` renders as an empty string). Either way, HTMX replaces
the contents of `#due-date-container`.

For comparison, here is how the same toggle would look with pure
`_hyperscript` (Approach B), if you did not need server-side rendering:

```gleam
// Approach B: _hyperscript toggle (simpler, but no server involvement)
fn due_date_checkbox_simple() -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label([attribute.class("checkbox-label")], [
      html.input([
        attribute.type_("checkbox"),
        attribute.name("has_due_date"),
        attribute.value("true"),
        attribute.attribute(
          "_",
          "on change toggle .hidden on #due-date-section",
        ),
      ]),
      element.text(" Set due date"),
    ]),
  ])
}
```

With Approach B, the date section is always in the DOM (just hidden with
CSS). No network request, no server round trip, instant toggle. But if
Flatpickr needs to be initialized fresh each time (which it does when the
element was not in the DOM initially), Approach A is more reliable.

### Step 5 -- Repeating Subtask Fields

Now for the "add another" pattern. Each subtask is a row with a text input
and a remove button. The "Add subtask" button fetches a new row from the
server and appends it.

First, the container and the "Add" button:

```gleam
fn subtask_section(
  existing_subtasks: List(#(Int, String)),
) -> Element(t) {
  let next_index = case existing_subtasks {
    [] -> 0
    _ -> list.length(existing_subtasks)
  }

  html.div([attribute.class("form-group")], [
    html.label([], [element.text("Subtasks")]),
    html.div([attribute.id("subtask-list")],
      list.map(existing_subtasks, fn(pair) {
        subtask_row(pair.0, pair.1)
      }),
    ),
    html.button([
      attribute.type_("button"),
      attribute.class("btn-secondary"),
      attribute.id("add-subtask-btn"),
      hx.get("/tasks/subtask-field?index=" <> int.to_string(next_index)),
      hx.target(hx.Selector("#subtask-list")),
      hx.swap(hx.Beforeend),
      attribute.attribute(
        "_",
        "on htmx:afterRequest increment :index then "
        <> "set my @hx-get to "
        <> "'/tasks/subtask-field?index=' + :index",
      ),
    ], [element.text("+ Add subtask")]),
  ])
}
```

Let us unpack the "Add subtask" button:

- **`attribute.type_("button")`** -- Important! Without this, the button
  would default to `type="submit"` and try to submit the entire form.
  `type="button"` makes it a plain button that only triggers HTMX.
- **`hx.get("/tasks/subtask-field?index=0")`** -- Fetches a new subtask row
  from the server, passing the next index.
- **`hx.target(hx.Selector("#subtask-list"))`** -- The response goes into
  the subtask list container.
- **`hx.swap(hx.Beforeend)`** -- Append at the end, not replace. Each
  "Add" click adds a new row without disturbing existing ones. Note the
  casing: `Beforeend`, not `BeforeEnd`.
- **The `_hyperscript` attribute** -- This is a clever trick. After each
  HTMX request completes, it increments a local variable `:index` and
  updates the button's `hx-get` attribute so the next click fetches the
  next index. This keeps the button self-updating without additional
  server logic.

Now the subtask row itself:

```gleam
fn subtask_row(index: Int, value: String) -> Element(t) {
  let index_str = int.to_string(index)
  let field_name = "subtask[" <> index_str <> "]"

  html.div(
    [attribute.class("subtask-row"), attribute.id("subtask-" <> index_str)],
    [
      html.input([
        attribute.type_("text"),
        attribute.name(field_name),
        attribute.value(value),
        attribute.class("input subtask-input"),
        attribute.placeholder("Subtask " <> index_str),
      ]),
      html.button([
        attribute.type_("button"),
        attribute.class("btn-remove"),
        attribute.attribute(
          "_",
          "on click remove closest .subtask-row",
        ),
      ], [element.text("Remove")]),
    ],
  )
}
```

Each subtask row has:

- An input with `name="subtask[N]"` where N is the index. This is how the
  server will identify which subtask is which.
- A remove button with `_hyperscript`: `on click remove closest .subtask-row`.
  This is purely client-side -- clicking "Remove" immediately removes the
  row from the DOM. No server request needed. The `closest .subtask-row`
  selector walks up the DOM tree from the button to the nearest ancestor
  with class `subtask-row`, and removes it.

The server endpoint that returns a new subtask row:

```gleam
fn get_subtask_field(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let index =
    list.key_find(query_params, "index")
    |> result.unwrap("0")
    |> int.parse
    |> result.unwrap(0)

  let html = subtask_row(index, "")
  wisp.html_response(element.to_string(html), 200)
}
```

This endpoint reads the `index` query parameter, parses it as an integer
(defaulting to 0 if parsing fails), and returns a fresh subtask row with
an empty value. The `int.parse` function returns a `Result(Int, Nil)`,
and we use `result.unwrap(0)` to handle the error case.

Wire the new routes into the router:

```gleam
["tasks", "subtask-field"] -> {
  case req.method {
    http.Get -> get_subtask_field(req)
    _ -> wisp.method_not_allowed([http.Get])
  }
}
["tasks", "due-date-section"] -> {
  case req.method {
    http.Get -> get_due_date_section(req)
    _ -> wisp.method_not_allowed([http.Get])
  }
}
```

Here is the interaction when the user clicks "Add subtask" for the third
time:

```
Browser                                Server
  |                                       |
  |  User clicks "+ Add subtask"          |
  |  Button's hx-get is                   |
  |  "/tasks/subtask-field?index=2"       |
  |                                       |
  |  GET /tasks/subtask-field?index=2     |
  |-------------------------------------->|
  |                                       |  Parse index = 2
  |                                       |  subtask_row(2, "")
  |                                       |
  |  200 OK                               |
  |  <div class="subtask-row"             |
  |       id="subtask-2">                 |
  |    <input name="subtask[2]" .../>     |
  |    <button _="on click remove         |
  |       closest .subtask-row">          |
  |       Remove</button>                 |
  |  </div>                               |
  |<--------------------------------------|
  |                                       |
  HTMX appends to #subtask-list
  (hx-swap="beforeend")

  _hyperscript on the button increments
  :index to 3, updates hx-get to
  "/tasks/subtask-field?index=3"
```

### Step 6 -- Parsing Everything on the Server

When the user fills out the entire form and submits it, the server receives
all the fields together: the title, category, subcategory, optional due
date, and a variable number of subtasks. Let us build the handler that
parses all of this:

```gleam
fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  // Extract standard fields
  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")
    |> string.trim

  let category =
    list.key_find(form_data.values, "category")
    |> result.unwrap("")

  let subcategory =
    list.key_find(form_data.values, "subcategory")
    |> result.unwrap("")

  // Extract optional due date
  let due_date =
    list.key_find(form_data.values, "due_date")
    |> result.unwrap("")
    |> string.trim

  // Extract all subtask values, filtering out empty ones
  let subtasks =
    form_data.values
    |> list.filter(fn(pair) { string.starts_with(pair.0, "subtask[") })
    |> list.sort(fn(a, b) { string.compare(a.0, b.0) })
    |> list.map(fn(pair) { string.trim(pair.1) })
    |> list.filter(fn(value) { value != "" })

  // Validate
  case title {
    "" -> {
      // Return the form with errors
      // (simplified for clarity -- see Full Code Listing for complete version)
      wisp.bad_request()
    }
    _ -> {
      // Build the task
      let id = new_id()
      let task = Task(
        id: id,
        title: title,
        category: category,
        subcategory: subcategory,
        due_date: due_date,
        subtasks: subtasks,
        done: False,
      )

      // Store it
      actor.send(ctx.tasks, AddTask(task))

      // Return the new task fragment
      let html = task_item(task)
      wisp.html_response(element.to_string(html), 201)
    }
  }
}
```

The subtask extraction pipeline is worth repeating because it demonstrates
a common Gleam pattern for handling dynamic form data:

```gleam
let subtasks =
  form_data.values                                              // 1. Start with all values
  |> list.filter(fn(pair) { string.starts_with(pair.0, "subtask[") })  // 2. Keep only subtask fields
  |> list.sort(fn(a, b) { string.compare(a.0, b.0) })           // 3. Sort by key (preserves order)
  |> list.map(fn(pair) { string.trim(pair.1) })                  // 4. Extract and trim values
  |> list.filter(fn(value) { value != "" })                       // 5. Drop empty entries
```

Each step in the pipeline is a pure function that transforms the data.
There are no side effects, no mutations, no exceptions. If any step
produces an empty list, the subsequent steps simply operate on that empty
list. Gleam's type system guarantees every step produces the right type.

Let us trace what happens with a real form submission. Suppose the user
submitted:

```
title=Redesign+homepage
&category=work
&subcategory=feature
&has_due_date=true
&due_date=2026-04-01
&subtask[0]=Create+wireframes
&subtask[1]=
&subtask[2]=Build+prototype
```

After parsing:

- `title` = `"Redesign homepage"`
- `category` = `"work"`
- `subcategory` = `"feature"`
- `due_date` = `"2026-04-01"`
- `subtasks` = `["Create wireframes", "Build prototype"]`
  (subtask[1] was empty and got filtered out)

The task is created with only the non-empty subtasks. The user added three
subtask rows but only filled in two, and the server quietly ignored the
empty one.

### Step 7 -- Preserving Values During Partial Updates

There is one more detail to handle. When the user changes the category
dropdown, HTMX sends a request and swaps the subcategory options. But what
if the user has already typed a title and some subtasks? Those values are
safe -- the swap only targets `#subcategory`, so the rest of the form is
untouched.

However, if you ever need to replace a larger section of the form (for
example, if changing the category should also show or hide certain fields),
you need `hx-include` to send the current field values along with the
request so the server can render them back:

```gleam
fn category_select_with_include(
  all_cats: List(Category),
  selected: String,
) -> Element(t) {
  html.select(
    [
      attribute.name("category"),
      attribute.id("category"),
      attribute.class("input"),
      hx.get("/tasks/form-section"),
      hx.trigger([hx.change()]),
      hx.target(hx.Selector("#dynamic-fields")),
      hx.swap(hx.InnerHTML),
      hx.include(hx.Selector("#task-form")),
    ],
    [
      html.option(
        [attribute.value(""), attribute.selected(selected == "")],
        "Select a category...",
      ),
      ..list.map(all_cats, fn(cat) {
        html.option(
          [
            attribute.value(cat.value),
            attribute.selected(cat.value == selected),
          ],
          cat.label,
        )
      })
    ],
  )
}
```

The key difference here is `hx.include(hx.Selector("#task-form"))` -- this
tells HTMX to include all named inputs inside the element with
`id="task-form"` in the request. The server receives the title, description,
subcategory, and everything else the user has typed so far.

On the server, you would read those values and pass them back when rendering
the response:

```gleam
fn get_form_section(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let category =
    list.key_find(query_params, "category")
    |> result.unwrap("")

  let title =
    list.key_find(query_params, "title")
    |> result.unwrap("")

  let subcats = categories.subcategories_for(category)

  // Return the dynamic fields section with preserved values
  let html = element.fragment([
    subcategory_select(subcats, ""),
    // ... other fields that depend on category, with preserved values
  ])

  wisp.html_response(element.to_string(html), 200)
}
```

The general principle: **if a swap replaces an area that contains user
input, the server must echo those values back in the response.** Read
them from the request (via `hx-include`), and set them as `value`
attributes in the rendered HTML. The user never notices their input
disappeared and reappeared -- it looks like only the dependent field
changed.

For our category/subcategory cascade, this is not necessary because we
only swap the inner HTML of the subcategory select -- the title,
description, and other fields are outside the swap target and remain
untouched. But knowing the pattern is important for more complex forms
where multiple sections need to change at once.

---

## 3. Full Code Listing

Here is the complete code for this chapter, assembled into their respective
modules. Files that have not changed from previous chapters are omitted.

### `src/teamwork/categories.gleam`

```gleam
import gleam/list

pub type Category {
  Category(value: String, label: String, subcategories: List(Subcategory))
}

pub type Subcategory {
  Subcategory(value: String, label: String)
}

pub fn all_categories() -> List(Category) {
  [
    Category(value: "work", label: "Work", subcategories: [
      Subcategory(value: "code-review", label: "Code review"),
      Subcategory(value: "bug-fix", label: "Bug fix"),
      Subcategory(value: "feature", label: "Feature"),
      Subcategory(value: "documentation", label: "Documentation"),
    ]),
    Category(value: "personal", label: "Personal", subcategories: [
      Subcategory(value: "health", label: "Health"),
      Subcategory(value: "errands", label: "Errands"),
      Subcategory(value: "finance", label: "Finance"),
    ]),
    Category(value: "home", label: "Home", subcategories: [
      Subcategory(value: "cleaning", label: "Cleaning"),
      Subcategory(value: "repair", label: "Repair"),
      Subcategory(value: "shopping", label: "Shopping"),
      Subcategory(value: "garden", label: "Garden"),
    ]),
  ]
}

pub fn subcategories_for(category_value: String) -> List(Subcategory) {
  case list.find(all_categories(), fn(c) { c.value == category_value }) {
    Ok(category) -> category.subcategories
    Error(_) -> []
  }
}
```

### `src/teamwork/task.gleam` (updated)

```gleam
import gleam/bool
import gleam/list
import gleam/string

pub type Task {
  Task(
    id: String,
    title: String,
    category: String,
    subcategory: String,
    due_date: String,
    subtasks: List(String),
    done: Bool,
  )
}

pub fn filter_by_status(
  tasks: List(Task),
  status: String,
) -> List(Task) {
  case status {
    "active" -> list.filter(tasks, fn(t) { bool.negate(t.done) })
    "done" -> list.filter(tasks, fn(t) { t.done })
    _ -> tasks
  }
}

pub fn filter_by_search(
  tasks: List(Task),
  query: String,
) -> List(Task) {
  case query {
    "" -> tasks
    q -> {
      let lower_q = string.lowercase(q)
      list.filter(tasks, fn(t) {
        string.contains(string.lowercase(t.title), lower_q)
      })
    }
  }
}

/// Parse subtask values from form data, filtering out empty entries.
pub fn parse_subtasks(
  values: List(#(String, String)),
) -> List(String) {
  values
  |> list.filter(fn(pair) { string.starts_with(pair.0, "subtask[") })
  |> list.sort(fn(a, b) { string.compare(a.0, b.0) })
  |> list.map(fn(pair) { string.trim(pair.1) })
  |> list.filter(fn(value) { value != "" })
}
```

### `src/teamwork/views/task_form_view.gleam`

```gleam
import gleam/int
import gleam/list
import gleam/result
import lustre/attribute.{attribute}
import lustre/element.{type Element}
import lustre/element/html
import hx
import teamwork/categories.{type Category, type Subcategory}

/// Renders the complete task creation form.
pub fn create_task_form(
  all_cats: List(Category),
  selected_category: String,
  subcats: List(Subcategory),
  selected_subcategory: String,
  title_value: String,
  has_due_date: Bool,
  due_date_value: String,
  existing_subtasks: List(#(Int, String)),
  errors: List(String),
) -> Element(t) {
  html.div([attribute.id("form-container"), attribute.class("form-card")], [
    html.h2([], [element.text("Create Task")]),
    // Show errors if any
    case errors {
      [] -> element.none()
      _ ->
        html.div(
          [attribute.class("error-summary")],
          list.map(errors, fn(err) {
            html.p(
              [attribute.class("error-text")],
              [element.text(err)],
            )
          }),
        )
    },
    html.form(
      [
        attribute.id("task-form"),
        hx.post("/tasks"),
        hx.target(hx.Selector("#form-container")),
        hx.swap(hx.OuterHTML),
      ],
      [
        // Title field
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("title")],
            [element.text("Title")],
          ),
          html.input([
            attribute.type_("text"),
            attribute.name("title"),
            attribute.id("title"),
            attribute.value(title_value),
            attribute.class("input"),
            attribute.placeholder("What needs to be done?"),
            attribute.required(True),
          ]),
        ]),
        // Category select
        category_select(all_cats, selected_category),
        // Subcategory select
        subcategory_select(subcats, selected_subcategory),
        // Due date checkbox + conditional section
        due_date_checkbox(has_due_date),
        due_date_container(has_due_date, due_date_value),
        // Subtask section
        subtask_section(existing_subtasks),
        // Submit button
        html.div([attribute.class("form-actions")], [
          html.button(
            [attribute.type_("submit"), attribute.class("btn-primary")],
            [element.text("Create Task")],
          ),
        ]),
      ],
    ),
  ])
}

fn category_select(
  all_cats: List(Category),
  selected: String,
) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label(
      [attribute.for("category")],
      [element.text("Category")],
    ),
    html.select(
      [
        attribute.name("category"),
        attribute.id("category"),
        attribute.class("input"),
        hx.get("/tasks/subcategories"),
        hx.trigger([hx.change()]),
        hx.target(hx.Selector("#subcategory")),
        hx.swap(hx.InnerHTML),
        hx.include(hx.This),
      ],
      [
        html.option(
          [attribute.value(""), attribute.selected(selected == "")],
          "Select a category...",
        ),
        ..list.map(all_cats, fn(cat) {
          html.option(
            [
              attribute.value(cat.value),
              attribute.selected(cat.value == selected),
            ],
            cat.label,
          )
        })
      ],
    ),
  ])
}

fn subcategory_select(
  subcats: List(Subcategory),
  selected: String,
) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label(
      [attribute.for("subcategory")],
      [element.text("Subcategory")],
    ),
    html.select(
      [
        attribute.name("subcategory"),
        attribute.id("subcategory"),
        attribute.class("input"),
      ],
      case subcats {
        [] -> [
          html.option(
            [attribute.value("")],
            "Select a category first",
          ),
        ]
        _ -> [
          html.option(
            [attribute.value(""), attribute.selected(selected == "")],
            "Select a subcategory...",
          ),
          ..list.map(subcats, fn(sub) {
            html.option(
              [
                attribute.value(sub.value),
                attribute.selected(sub.value == selected),
              ],
              sub.label,
            )
          })
        ]
      },
    ),
  ])
}

fn due_date_checkbox(is_checked: Bool) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label([attribute.class("checkbox-label")], [
      html.input([
        attribute.type_("checkbox"),
        attribute.name("has_due_date"),
        attribute.value("true"),
        attribute.checked(is_checked),
        hx.get("/tasks/due-date-section"),
        hx.trigger([hx.change()]),
        hx.target(hx.Selector("#due-date-container")),
        hx.swap(hx.InnerHTML),
        hx.include(hx.This),
      ]),
      element.text(" Set due date"),
    ]),
  ])
}

fn due_date_container(show: Bool, current_date: String) -> Element(t) {
  html.div([attribute.id("due-date-container")], [
    case show {
      True -> due_date_section(current_date)
      False -> element.none()
    },
  ])
}

fn due_date_section(current_date: String) -> Element(t) {
  html.div([attribute.class("form-group due-date-section")], [
    html.label(
      [attribute.for("due_date")],
      [element.text("Due date")],
    ),
    html.input([
      attribute.type_("text"),
      attribute.name("due_date"),
      attribute.id("due_date"),
      attribute.value(current_date),
      attribute.class("input"),
      attribute.placeholder("Pick a date..."),
      attribute.attribute(
        "_",
        "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
      ),
    ]),
  ])
}

fn subtask_section(
  existing_subtasks: List(#(Int, String)),
) -> Element(t) {
  let next_index = list.length(existing_subtasks)

  html.div([attribute.class("form-group")], [
    html.label([], [element.text("Subtasks")]),
    html.div(
      [attribute.id("subtask-list")],
      list.map(existing_subtasks, fn(pair) {
        subtask_row(pair.0, pair.1)
      }),
    ),
    html.button([
      attribute.type_("button"),
      attribute.class("btn-secondary"),
      attribute.id("add-subtask-btn"),
      hx.get(
        "/tasks/subtask-field?index=" <> int.to_string(next_index),
      ),
      hx.target(hx.Selector("#subtask-list")),
      hx.swap(hx.Beforeend),
      attribute.attribute(
        "_",
        "on htmx:afterRequest increment :index then "
        <> "set my @hx-get to "
        <> "'/tasks/subtask-field?index=' + :index",
      ),
    ], [element.text("+ Add subtask")]),
  ])
}

pub fn subtask_row(index: Int, value: String) -> Element(t) {
  let index_str = int.to_string(index)
  let field_name = "subtask[" <> index_str <> "]"

  html.div(
    [attribute.class("subtask-row"), attribute.id("subtask-" <> index_str)],
    [
      html.input([
        attribute.type_("text"),
        attribute.name(field_name),
        attribute.value(value),
        attribute.class("input subtask-input"),
        attribute.placeholder("Subtask " <> index_str),
      ]),
      html.button([
        attribute.type_("button"),
        attribute.class("btn-remove"),
        attribute.attribute(
          "_",
          "on click remove closest .subtask-row",
        ),
      ], [element.text("Remove")]),
    ],
  )
}
```

### `src/teamwork/router.gleam` (updated sections)

```gleam
import gleam/http
import gleam/int
import gleam/list
import gleam/otp/actor
import gleam/result
import gleam/string
import lustre/attribute
import lustre/element
import lustre/element/html
import wisp
import teamwork/categories
import teamwork/task.{type Task, Task}
import teamwork/views/task_form_view

fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: ctx.static)

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["tasks"] -> {
      case req.method {
        http.Get -> list_tasks(req, ctx)
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }
    }
    ["tasks", "subcategories"] -> {
      case req.method {
        http.Get -> get_subcategories(req)
        _ -> wisp.method_not_allowed([http.Get])
      }
    }
    ["tasks", "due-date-section"] -> {
      case req.method {
        http.Get -> get_due_date_section(req)
        _ -> wisp.method_not_allowed([http.Get])
      }
    }
    ["tasks", "subtask-field"] -> {
      case req.method {
        http.Get -> get_subtask_field(req)
        _ -> wisp.method_not_allowed([http.Get])
      }
    }
    ["tasks", id] -> {
      case req.method {
        http.Delete -> delete_task(ctx, id)
        _ -> wisp.method_not_allowed([http.Delete])
      }
    }
    _ -> wisp.not_found()
  }
}

fn get_subcategories(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let category_value =
    list.key_find(query_params, "category")
    |> result.unwrap("")

  let subcats = categories.subcategories_for(category_value)

  let options = case subcats {
    [] -> [
      html.option([attribute.value("")], "No subcategories available"),
    ]
    _ -> [
      html.option([attribute.value("")], "Select a subcategory..."),
      ..list.map(subcats, fn(sub) {
        html.option([attribute.value(sub.value)], sub.label)
      })
    ]
  }

  let html_string =
    element.fragment(options)
    |> element.to_string

  wisp.html_response(html_string, 200)
}

fn get_due_date_section(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let has_due_date =
    list.key_find(query_params, "has_due_date")
    |> result.unwrap("")
    == "true"

  let html = case has_due_date {
    True ->
      html.div([attribute.class("form-group due-date-section")], [
        html.label(
          [attribute.for("due_date")],
          [element.text("Due date")],
        ),
        html.input([
          attribute.type_("text"),
          attribute.name("due_date"),
          attribute.id("due_date"),
          attribute.value(""),
          attribute.class("input"),
          attribute.placeholder("Pick a date..."),
          attribute.attribute(
            "_",
            "on load call flatpickr(me, {dateFormat: 'Y-m-d', altInput: true, altFormat: 'M j, Y'})",
          ),
        ]),
      ])
    False -> element.none()
  }

  wisp.html_response(element.to_string(html), 200)
}

fn get_subtask_field(req: wisp.Request) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let index =
    list.key_find(query_params, "index")
    |> result.unwrap("0")
    |> int.parse
    |> result.unwrap(0)

  let html = task_form_view.subtask_row(index, "")
  wisp.html_response(element.to_string(html), 200)
}

fn create_task(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  // Extract standard fields
  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")
    |> string.trim

  let category =
    list.key_find(form_data.values, "category")
    |> result.unwrap("")

  let subcategory =
    list.key_find(form_data.values, "subcategory")
    |> result.unwrap("")

  // Extract optional due date
  let due_date =
    list.key_find(form_data.values, "due_date")
    |> result.unwrap("")
    |> string.trim

  // Extract subtasks
  let subtasks = task.parse_subtasks(form_data.values)

  // Validate
  case title {
    "" -> {
      let all_cats = categories.all_categories()
      let subcats = categories.subcategories_for(category)
      let has_due_date = due_date != ""

      // Reconstruct existing subtasks with indices for re-rendering
      let indexed_subtasks =
        subtasks
        |> list.index_map(fn(value, index) { #(index, value) })

      let form_html = task_form_view.create_task_form(
        all_cats,
        category,
        subcats,
        subcategory,
        title,
        has_due_date,
        due_date,
        indexed_subtasks,
        ["Title is required."],
      )

      wisp.html_response(element.to_string(form_html), 422)
    }
    _ -> {
      let id = new_id()
      let new_task = Task(
        id: id,
        title: title,
        category: category,
        subcategory: subcategory,
        due_date: due_date,
        subtasks: subtasks,
        done: False,
      )

      actor.send(ctx.tasks, AddTask(new_task))

      // Return a fresh empty form
      let all_cats = categories.all_categories()
      let form_html = task_form_view.create_task_form(
        all_cats, "", [], "", "", False, "", [], [],
      )

      wisp.html_response(element.to_string(form_html), 201)
    }
  }
}
```

### `priv/static/css/style.css` (additions for this chapter)

```css
/* ─── Form card ─── */

.form-card {
  background: white;
  border-radius: 8px;
  padding: 1.5rem;
  margin-bottom: 2rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.form-card h2 {
  margin: 0 0 1.25rem 0;
  color: #16213e;
  font-size: 1.25rem;
}

/* ─── Form groups ─── */

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
  border: 1px solid #ccc;
  border-radius: 4px;
  transition: border-color 0.15s ease;
  background: white;
}

.input:focus {
  outline: none;
  border-color: #16213e;
  box-shadow: 0 0 0 3px rgba(22, 33, 62, 0.15);
}

/* ─── Checkbox label ─── */

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
  font-weight: 500;
}

.checkbox-label input[type="checkbox"] {
  width: 1.125rem;
  height: 1.125rem;
  cursor: pointer;
}

/* ─── Due date section ─── */

.due-date-section {
  padding: 0.75rem;
  margin-top: 0.5rem;
  background: #f8f9fa;
  border-radius: 6px;
  border: 1px solid #e9ecef;
}

/* ─── Subtask rows ─── */

.subtask-row {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
  align-items: center;
}

.subtask-input {
  flex: 1;
}

.btn-remove {
  padding: 0.375rem 0.75rem;
  background: #fff;
  color: #dc3545;
  border: 1px solid #dc3545;
  border-radius: 4px;
  font-size: 0.8125rem;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;
}

.btn-remove:hover {
  background: #dc3545;
  color: white;
}

/* ─── Secondary button (Add subtask) ─── */

.btn-secondary {
  padding: 0.375rem 0.75rem;
  background: white;
  color: #16213e;
  border: 1px dashed #adb5bd;
  border-radius: 4px;
  font-size: 0.875rem;
  cursor: pointer;
  transition: all 0.15s ease;
  margin-top: 0.25rem;
}

.btn-secondary:hover {
  border-color: #16213e;
  background: #f8f9fa;
}

/* ─── Form actions ─── */

.form-actions {
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #e9ecef;
}

/* ─── Error summary ─── */

.error-summary {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
  padding: 0.75rem 1rem;
  margin-bottom: 1rem;
}

.error-summary .error-text {
  color: #dc2626;
  margin: 0;
  font-size: 0.875rem;
}
```

---

## 4. Exercises

### Exercise 1 -- Add a Priority Cascading Select

Add a "Priority" dropdown to the form with three options: Low, Medium, High.
When the user selects "High", display an additional text input labelled
"Justification" (using the conditional section pattern). When any other
priority is selected, hide the justification field.

**Acceptance criteria:**
- The Priority dropdown renders with Low, Medium, and High options.
- Selecting "High" triggers a server request that returns a justification
  text input.
- Selecting "Low" or "Medium" removes the justification input.
- The justification value is submitted with the form when present.
- The server correctly reads the justification from form data.

### Exercise 2 -- Three-Level Cascade

Extend the category system to three levels: Category -> Subcategory -> Item
Type. For example, Work -> Bug fix -> Frontend, Backend, Database. When the
user changes the subcategory, a third select populates with item types.

**Acceptance criteria:**
- Changing the category populates the subcategory dropdown (existing).
- Changing the subcategory populates a new "Item type" dropdown.
- The item type dropdown resets when the subcategory changes.
- The item type value is included in the form submission.
- The server endpoint handles the second cascade request.

### Exercise 3 -- Subtask Limit

Limit the number of subtasks to five. When the user reaches five subtasks,
disable the "Add subtask" button. When they remove one, re-enable it.

**Acceptance criteria:**
- The "Add subtask" button becomes disabled after five subtasks are added.
- Removing a subtask re-enables the button.
- The disabled button has a visual indicator (greyed out, changed cursor).
- Attempting to click the disabled button does not trigger a request.

### Exercise 4 -- Subtask Drag Reorder

Using `_hyperscript` (not a third-party library), implement a basic
drag-to-reorder for subtask rows. When the user reorders the subtasks,
update the `name` attributes so the indices reflect the new order.

**Acceptance criteria:**
- Subtask rows can be dragged to a new position.
- After reordering, the `name` attributes are `subtask[0]`, `subtask[1]`,
  etc., in the new visual order.
- Submitting the form after reordering sends the subtasks in the correct
  new order.
- The reorder is purely client-side (no server request).

### Exercise 5 -- Form State Persistence with hx-include

Modify the category select so that when the user changes it, the server
returns not just the subcategory options but also updates a "category
description" paragraph below the subcategory select. Use `hx-include` to
send the current title value along with the request, and have the server
echo the title back so it is not lost during the swap.

**Acceptance criteria:**
- Changing the category updates both the subcategory options and a
  description paragraph.
- The description paragraph shows text like "Work tasks are tracked with
  JIRA integration" (different per category).
- The user's title input is preserved across the category change.
- The swap target is a wrapper div that contains both the subcategory
  select and the description paragraph.

---

## 5. Exercise Solution Hints

> Try each exercise on your own before reading these hints.

### Hint for Exercise 1

The priority select is similar to the category select but simpler -- it does
not need its own data module. Add the `hx-get`, `hx-trigger`, and
`hx-target` attributes just like the category select:

```gleam
fn priority_select(selected: String) -> Element(t) {
  html.div([attribute.class("form-group")], [
    html.label(
      [attribute.for("priority")],
      [element.text("Priority")],
    ),
    html.select(
      [
        attribute.name("priority"),
        attribute.id("priority"),
        attribute.class("input"),
        hx.get("/tasks/priority-section"),
        hx.trigger([hx.change()]),
        hx.target(hx.Selector("#priority-extras")),
        hx.swap(hx.InnerHTML),
        hx.include(hx.This),
      ],
      [
        html.option(
          [attribute.value("low"), attribute.selected(selected == "low")],
          "Low",
        ),
        html.option(
          [attribute.value("medium"), attribute.selected(selected == "medium")],
          "Medium",
        ),
        html.option(
          [attribute.value("high"), attribute.selected(selected == "high")],
          "High",
        ),
      ],
    ),
    html.div([attribute.id("priority-extras")], []),
  ])
}
```

The endpoint checks if the priority is "high" and returns a justification
input or `element.none()`.

### Hint for Exercise 2

Add a third select with `id="item-type"`. The subcategory select needs
HTMX attributes similar to the category select:

```gleam
html.select(
  [
    attribute.name("subcategory"),
    attribute.id("subcategory"),
    attribute.class("input"),
    hx.get("/tasks/item-types"),
    hx.trigger([hx.change()]),
    hx.target(hx.Selector("#item-type")),
    hx.swap(hx.InnerHTML),
    hx.include(hx.This),
  ],
  // ... options
)
```

Extend the `categories.gleam` module to include item types on each
subcategory. The endpoint `/tasks/item-types` reads the `subcategory`
query parameter and returns matching `<option>` elements.

### Hint for Exercise 3

Use `_hyperscript` on the "Add subtask" button to check the count after
each addition:

```gleam
attribute.attribute(
  "_",
  "on htmx:afterRequest "
  <> "increment :index "
  <> "then set my @hx-get to '/tasks/subtask-field?index=' + :index "
  <> "then if #subtask-list.children.length >= 5 "
  <> "add @disabled to me "
  <> "end",
)
```

For re-enabling on remove, add a handler on the remove buttons:

```gleam
attribute.attribute(
  "_",
  "on click remove closest .subtask-row "
  <> "then remove @disabled from #add-subtask-btn",
)
```

### Hint for Exercise 4

HTML5 drag and drop with `_hyperscript` can be implemented using the
`draggable` attribute and `dragstart`, `dragover`, and `drop` events.
This is a stretch exercise -- a simple approach uses `_hyperscript` to
swap element positions:

```gleam
attribute.attribute("draggable", "true")
attribute.attribute(
  "_",
  "on dragstart set event.dataTransfer to my id "
  <> "on dragover halt the event "
  <> "on drop get event.dataTransfer "
  <> "then swap me and the #{it} in the DOM",
)
```

After the swap, iterate over all `.subtask-row` elements and renumber
their input `name` attributes. This is the trickiest part -- you may
want to use a small JavaScript snippet via
`attribute("hx-on::after-settle", "renumberSubtasks()")`.

### Hint for Exercise 5

Change the category select to target a wrapper div instead of just the
subcategory select:

```gleam
hx.target(hx.Selector("#category-dependent-section")),
hx.swap(hx.InnerHTML),
hx.include(hx.Selector("[name='title']")),
```

The wrapper div contains both the subcategory select and the description:

```gleam
html.div([attribute.id("category-dependent-section")], [
  subcategory_select(subcats, ""),
  category_description(selected_category),
])
```

The endpoint reads both `category` and `title` from the query parameters.
It does not use the `title` value directly in the rendered HTML of this
section, but it would if the title input were inside the swap target.

In this exercise, since the title input is *outside* the swap target, you
do not actually need to echo it back. The `hx-include` is useful for
scenarios where the server needs the title to make rendering decisions
(like showing a warning if the title mentions a keyword related to the
selected category). The key learning is understanding when `hx-include`
is necessary versus when the existing DOM state is preserved automatically.

---

## 6. Key Takeaways

1. **Cascading selects need three HTMX attributes.** `hx-get` to fetch
   options, `hx-trigger="change"` to fire on selection, and `hx-include`
   (with `hx.This` or `hx.Selector(...)`) to send the current value. The
   response is just `<option>` elements, swapped into the dependent select
   with `hx-swap="innerHTML"`.

2. **The server is the single source of truth for dependent data.** Category
   -> subcategory mappings live on the server in Gleam code. There are no
   JSON arrays duplicated in the frontend. Changing the data in one place
   updates everything.

3. **Conditional form sections can be server-fetched or client-toggled.**
   Server-fetched (HTMX) sections are appropriate when the section has
   complex content, needs data from the server, or requires JS library
   initialization (like Flatpickr). Client-toggled (`_hyperscript`) sections
   are appropriate when the content is already in the DOM and you just
   need to show or hide it.

4. **The "add another" pattern uses `Beforeend` swap.** A button with
   `hx-get` and `hx-swap="beforeend"` appends a new field group to a
   container. Each new group gets a unique indexed name (`subtask[0]`,
   `subtask[1]`). Removal is purely client-side with `_hyperscript`:
   `on click remove closest .subtask-row`.

5. **Parse dynamic fields by filtering form data by prefix.** Use
   `list.filter` with `string.starts_with` to extract all keys matching a
   pattern like `"subtask["`. Then `list.map` to extract values, and
   `list.filter` again to drop empty entries. This pipeline pattern is
   clean, declarative, and type-safe.

6. **`hx-include` preserves form state across partial updates.** When a swap
   targets an area containing user input, `hx-include` sends those values
   with the request so the server can echo them back as `value` attributes.
   When the swap target does not contain the user's inputs, `hx-include`
   is not needed -- the existing DOM values are untouched.

7. **Always use `type="button"` on non-submit buttons inside forms.** Without
   this attribute, the browser defaults to `type="submit"`, and clicking the
   "Add subtask" button would submit the entire form. This is a standard
   HTML gotcha, not specific to HTMX.

8. **`element.fragment` wraps multiple elements for rendering.** When a
   server endpoint needs to return several sibling elements (like a list of
   `<option>` tags), `element.fragment(options)` combines them into a single
   renderable value. Use `element.to_string` (not `to_document_string`) for
   fragments.

9. **`_hyperscript` handles client-side bookkeeping.** Incrementing the
   subtask index, removing rows, and toggling visibility are all
   micro-interactions that do not need server involvement. `_hyperscript`
   keeps them declarative and co-located with the elements they affect,
   avoiding standalone JavaScript files.

---

## What's Next

In Chapter 25, we will tackle **File Uploads** -- letting users attach
files to tasks using `hx-encoding="multipart/form-data"`, showing upload
progress indicators, and handling `UploadedFile` data in Wisp. We will
build a drag-and-drop upload zone and display file previews, all with
server-rendered HTML and minimal JavaScript.
