# Chapter 23 -- \_hyperscript Practical Patterns

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapter 22 introduced modal dialogs with HTMX and \_hyperscript for
open/close behaviour. This chapter collects production-ready \_hyperscript recipes
that you will reach for again and again.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Build a live character counter using \_hyperscript element-scoped variables.
2. Implement an accordion where only one section is open at a time.
3. Create a clipboard-copy button with success/error feedback.
4. Build drag-and-drop reorder using HTML drag events and \_hyperscript.
5. Combine multiple \_hyperscript behaviours on a single element without conflicts.
6. Apply the decision criteria for "use \_hyperscript" vs "write JavaScript."

---

## 1. Theory

### 1.1 From Reference to Recipes

Appendix B is the grammar book. It covers every command, every variable scope, every
targeting expression. This chapter is the cookbook.

The difference matters. Knowing that \_hyperscript has a `put` command and a `set`
command is like knowing that flour and butter exist. What you actually need is the
recipe for pie crust. You need to see the ingredients combined, in order, with the
right proportions, producing something that works.

Each pattern in this chapter meets three criteria:

1. **Short.** Between one and fifteen lines of \_hyperscript. If it takes more, you
   should write JavaScript instead.
2. **Self-contained.** The \_hyperscript lives on the element it affects. No
   external script files, no global state, no coordination between distant
   elements (except where the pattern explicitly calls for it, like drag-and-drop).
3. **Production-tested.** These are not toy examples. Every pattern handles edge
   cases -- what happens if the copy fails, what happens if the user drags outside
   the list, what happens if two animations overlap.

We will build five patterns:

| #  | Pattern                   | Lines | Key \_hyperscript concepts                     |
|----|---------------------------|-------|-------------------------------------------------|
| 1  | Live character counter    | 6     | `on input`, element variables, `put`, `if/else` |
| 2  | Exclusive accordion       | 3     | `on click`, `remove` from siblings, `toggle`    |
| 3  | Clipboard copy            | 5     | `on click`, `call`, `add/remove`, `wait`        |
| 4  | Drag-and-drop reorder     | ~15   | `on dragstart/dragover/drop`, DOM manipulation   |
| 5  | Multi-behaviour task card | ~12   | Multiple `on` handlers on one element            |

After the patterns, we will discuss when to stop using \_hyperscript and extract
to JavaScript -- because that threshold exists, and crossing it is the right call
sometimes.

### 1.2 Pattern: Live Character Counter

The task description textarea in Teamwork has a 200-character limit (enforced
server-side with `attribute.maxlength(200)`). A live character counter gives the
user immediate feedback as they type.

Here is the \_hyperscript:

```
on input
  put my value's length into #char-count's textContent
  if my value's length > 180
    add .warning to #char-count
  else
    remove .warning from #char-count
  end
```

Let us walk through it line by line.

**Line 1: `on input`**

The `input` event fires on every keystroke, paste, cut, or autofill change. It is
more reliable than `keyup` because it catches *all* value changes, including
right-click paste and drag-and-drop text into the field. Using `input` instead of
`keyup` is a deliberate choice -- `keyup` misses non-keyboard edits.

**Line 2: `put my value's length into #char-count's textContent`**

This is a single expression with several parts:

- `my` -- refers to the element that owns the `_` attribute (the textarea).
- `my value` -- the textarea's `.value` property (the text the user typed).
- `my value's length` -- the `.length` property of that string.
- `#char-count` -- a CSS selector targeting the element with `id="char-count"`.
- `#char-count's textContent` -- the `.textContent` property of that element.
- `put ... into ...` -- assigns the left side to the right side.

The result: every time the user types, the character count element updates to show
the current length.

**Lines 3-6: `if ... add .warning ... else ... remove .warning ... end`**

When the length exceeds 180 (90% of the 200-character limit), the counter gains a
`.warning` class. Below 180, the class is removed. The `.warning` class in your CSS
might turn the text red or change the background -- the specifics are up to you.

The `if/else/end` structure is mandatory. \_hyperscript does not use curly braces
or indentation for blocks. The `end` keyword closes the conditional.

**Why not use `toggle`?** Because `toggle` would flip the class on every keystroke
regardless of the current length. If the user is at 185 characters and presses a
key, `toggle` would remove `.warning` (wrong). The explicit `if/else` checks the
condition every time and applies the correct state.

Here is the HTML in context:

```html
<div class="form-group">
  <label for="description">Description</label>
  <textarea
    id="description"
    name="description"
    maxlength="200"
    _="on input
         put my value's length into #char-count's textContent
         if my value's length > 180
           add .warning to #char-count
         else
           remove .warning from #char-count
         end">
  </textarea>
  <span id="char-count" class="char-counter">0</span>
  <span class="char-limit">/ 200</span>
</div>
```

And in Gleam with Lustre:

```gleam
html.div([attribute.class("form-group")], [
  html.label(
    [attribute.for("description")],
    [element.text("Description")],
  ),
  html.textarea(
    [
      attribute.id("description"),
      attribute.name("description"),
      attribute.maxlength(200),
      attribute.attribute("_", "on input
        put my value's length into #char-count's textContent
        if my value's length > 180
          add .warning to #char-count
        else
          remove .warning from #char-count
        end"),
    ],
    "",
  ),
  html.span(
    [attribute.id("char-count"), attribute.class("char-counter")],
    [element.text("0")],
  ),
  html.span(
    [attribute.class("char-limit")],
    [element.text("/ 200")],
  ),
])
```

Notice the use of `attribute.attribute("_", "...")`. This is the pattern for all
\_hyperscript in Gleam. The first argument is the attribute name (the literal
underscore character), and the second is the \_hyperscript code as a plain string.
There is no special Gleam library needed -- \_hyperscript is just an HTML attribute.

The CSS for the counter:

```css
.char-counter {
  font-size: 0.85rem;
  color: #718096;
  transition: color 0.2s ease;
}

.char-counter.warning {
  color: #e53e3e;
  font-weight: 600;
}
```

The `transition` property on `.char-counter` makes the colour change smooth rather
than jarring. This is a CSS concern, not a \_hyperscript concern -- keep
presentation in stylesheets and behaviour in \_hyperscript.

**Edge case: pre-filled textareas.** If the textarea has a default value (e.g.
during inline editing from Chapter 21), the counter will show "0" until the user
types. To fix this, add an `init` handler:

```
init
  put my value's length into #char-count's textContent
on input
  put my value's length into #char-count's textContent
  if my value's length > 180
    add .warning to #char-count
  else
    remove .warning from #char-count
  end
```

The `init` keyword runs once when \_hyperscript initializes the element -- on page
load or after an HTMX swap. It sets the counter to the correct value immediately.

### 1.3 Pattern: Exclusive Accordion

The Teamwork board has a settings panel with collapsible sections: "General,"
"Notifications," "Permissions." The business requirement is that only one section
can be open at a time. Opening one closes the others.

This is one of \_hyperscript's best use cases because the alternative in vanilla
JavaScript is surprisingly verbose. You need to query siblings, iterate, remove
classes, and then toggle the clicked one. In \_hyperscript:

```
on click
  remove .open from .accordion-section in closest .accordion
  toggle .open on me
```

Three lines. Let us break them down.

**Line 1: `on click`**

The click handler. Each accordion section header has this \_hyperscript.

**Line 2: `remove .open from .accordion-section in closest .accordion`**

This is the key line. Read it right to left:

- `closest .accordion` -- walk up the DOM from the current element until you find
  an ancestor with class `.accordion`. This is the accordion container.
- `.accordion-section in closest .accordion` -- find all descendants of that
  container that have class `.accordion-section`. These are all the sections,
  including the one that was just clicked.
- `remove .open from ...` -- remove the `.open` class from every one of those
  sections.

The result: all sections close.

**Line 3: `toggle .open on me`**

Now toggle `.open` on the clicked section. If it was closed before (the normal
case), toggling opens it. If it was open before (we just closed it in line 2),
toggling adds the class back -- so clicking an open section keeps it open.

Wait, there is a subtlety. Line 2 removes `.open` from *all* sections, including
the one we just clicked. Then line 3 toggles `.open` on that same element. Since
line 2 already removed it, the toggle adds it back. So clicking a closed section
opens it (correct). But what about clicking an *already open* section? Line 2
removes `.open`. Line 3 toggles, which adds it back. That means clicking an open
section keeps it open -- not ideal.

If you want clicking an open section to close it, use `add` with a condition
instead:

```
on click
  if I match .open
    remove .open from me
  else
    remove .open from .accordion-section in closest .accordion
    add .open to me
  end
```

This is six lines instead of three, but the behaviour is more intuitive. The
three-line version is fine when you always want exactly one section open.

Here is the full HTML structure:

```html
<div class="accordion">
  <div class="accordion-section">
    <button class="accordion-header"
            _="on click
                 remove .open from .accordion-section in closest .accordion
                 toggle .open on closest .accordion-section">
      General
    </button>
    <div class="accordion-body">
      <p>Board name, description, visibility settings.</p>
    </div>
  </div>

  <div class="accordion-section">
    <button class="accordion-header"
            _="on click
                 remove .open from .accordion-section in closest .accordion
                 toggle .open on closest .accordion-section">
      Notifications
    </button>
    <div class="accordion-body">
      <p>Email notifications, Slack integration, digest frequency.</p>
    </div>
  </div>

  <div class="accordion-section">
    <button class="accordion-header"
            _="on click
                 remove .open from .accordion-section in closest .accordion
                 toggle .open on closest .accordion-section">
      Permissions
    </button>
    <div class="accordion-body">
      <p>Role assignments, invite links, guest access.</p>
    </div>
  </div>
</div>
```

Notice that the \_hyperscript is on the `<button>` (the header), but we toggle
`.open` on `closest .accordion-section` (the parent). This is because the CSS
rules for showing/hiding the body are on the section element:

```css
.accordion-body {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease;
}

.accordion-section.open .accordion-body {
  max-height: 500px;
}
```

The `max-height` transition gives a smooth expand/collapse animation. Using
`max-height` instead of `height` is a CSS trick -- you cannot transition from
`height: 0` to `height: auto`, but you can transition from `max-height: 0` to
`max-height: 500px` (or whatever value is large enough for your content).

In Gleam, the accordion section becomes a reusable function:

```gleam
fn accordion_section(
  title: String,
  content: element.Element(t),
) -> element.Element(t) {
  html.div([attribute.class("accordion-section")], [
    html.button(
      [
        attribute.class("accordion-header"),
        attribute.attribute("_", "on click
          remove .open from .accordion-section in closest .accordion
          toggle .open on closest .accordion-section"),
      ],
      [element.text(title)],
    ),
    html.div(
      [attribute.class("accordion-body")],
      [content],
    ),
  ])
}
```

And the settings panel:

```gleam
fn settings_panel() -> element.Element(t) {
  html.div([attribute.class("accordion")], [
    accordion_section(
      "General",
      html.p([], [element.text("Board name, description, visibility settings.")]),
    ),
    accordion_section(
      "Notifications",
      html.p([], [element.text(
        "Email notifications, Slack integration, digest frequency.",
      )]),
    ),
    accordion_section(
      "Permissions",
      html.p([], [element.text(
        "Role assignments, invite links, guest access.",
      )]),
    ),
  ])
}
```

The \_hyperscript lives inside the `accordion_section` function. Every section gets
the same behaviour automatically. No JavaScript file to maintain, no event
delegation to wire up.

### 1.4 Pattern: Clipboard Copy

Every task on the Teamwork board has a shareable URL. You want a button that copies
the URL to the clipboard with a single click, shows a brief "Copied!" confirmation,
and handles errors gracefully.

Here is the \_hyperscript:

```
on click
  call navigator.clipboard.writeText(my dataset.url)
  add .copied to me
  wait 2s
  remove .copied from me
```

Line by line:

**Line 1: `on click`**

The click event fires the copy logic.

**Line 2: `call navigator.clipboard.writeText(my dataset.url)`**

This is JavaScript interop. The `call` command invokes a JavaScript expression.
`navigator.clipboard.writeText()` is the modern Clipboard API -- it writes text to
the system clipboard and returns a Promise.

`my dataset.url` reads the `data-url` attribute from the button element. In HTML,
`data-*` attributes are accessible via the `dataset` property. So
`<button data-url="/tasks/42">` gives `my dataset.url` the value `"/tasks/42"`.

\_hyperscript is Promise-aware. When a `call` expression returns a Promise,
\_hyperscript automatically `await`s it before proceeding to the next line. You do
not need to write `then` or `await` -- it just works.

**Line 3: `add .copied to me`**

After the text is copied, add the `.copied` class to the button. Your CSS can use
this to change the button's appearance -- green background, checkmark icon, whatever
signals success.

**Line 4: `wait 2s`**

Pause execution for two seconds. The button stays in the "copied" state.

**Line 5: `remove .copied from me`**

Revert to the default appearance.

The result is a smooth copy-confirm-reset cycle, all in five lines.

**Error handling.** The `navigator.clipboard.writeText()` call can fail -- the user
might deny permission, or the page might not be in a secure context (clipboard API
requires HTTPS or localhost). To handle this, wrap the call:

```
on click
  call navigator.clipboard.writeText(my dataset.url)
    catch e
      add .copy-error to me
      put 'Failed to copy' into me
      wait 2s
      remove .copy-error from me
      put 'Copy Link' into me
      halt
  end
  add .copied to me
  put 'Copied!' into me
  wait 2s
  remove .copied from me
  put 'Copy Link' into me
```

The `catch` block handles the rejection. If the copy fails, the button shows
"Failed to copy" with an error style, waits two seconds, and reverts. The `halt`
command stops execution so the success path does not run.

Here is the HTML:

```html
<button class="btn btn-small copy-link-btn"
        data-url="/tasks/42"
        _="on click
             call navigator.clipboard.writeText(my dataset.url)
               catch e
                 add .copy-error to me
                 put 'Failed to copy' into me
                 wait 2s
                 remove .copy-error from me
                 put 'Copy Link' into me
                 halt
             end
             add .copied to me
             put 'Copied!' into me
             wait 2s
             remove .copied from me
             put 'Copy Link' into me">
  Copy Link
</button>
```

And in Gleam:

```gleam
fn copy_link_button(task_id: String) -> element.Element(t) {
  let url = "/tasks/" <> task_id

  html.button(
    [
      attribute.class("btn btn-small copy-link-btn"),
      attribute.attribute("data-url", url),
      attribute.attribute("_", "on click
        call navigator.clipboard.writeText(my dataset.url)
          catch e
            add .copy-error to me
            put 'Failed to copy' into me
            wait 2s
            remove .copy-error from me
            put 'Copy Link' into me
            halt
        end
        add .copied to me
        put 'Copied!' into me
        wait 2s
        remove .copied from me
        put 'Copy Link' into me"),
    ],
    [element.text("Copy Link")],
  )
}
```

The CSS:

```css
.copy-link-btn {
  transition: background-color 0.2s ease, color 0.2s ease;
}

.copy-link-btn.copied {
  background-color: #48bb78;
  color: white;
}

.copy-link-btn.copy-error {
  background-color: #e53e3e;
  color: white;
}
```

**A note on full URLs.** The `data-url` attribute above contains a relative path
(`/tasks/42`). For a shareable link, you probably want the full URL. You can
construct it on the server:

```gleam
let full_url = "https://teamwork.example.com/tasks/" <> task_id
```

Or build it client-side in the \_hyperscript:

```
call navigator.clipboard.writeText(window.location.origin + my dataset.url)
```

The `window.location.origin` expression gives you `https://teamwork.example.com`,
and you concatenate the relative path. Either approach works -- the server-side
version is simpler if you have access to the base URL in your configuration.

### 1.5 Pattern: Drag-and-Drop Reorder

This is the most complex pattern in the chapter, but it is also the most rewarding.
Users can drag tasks to reorder them within a list, and the new order persists to
the server.

Drag-and-drop in HTML uses four native events:

| Event       | Fires on        | When                                        |
|-------------|-----------------|---------------------------------------------|
| `dragstart` | Dragged element | User starts dragging                        |
| `dragover`  | Drop target     | Dragged element is over a valid drop target |
| `dragenter` | Drop target     | Dragged element enters a drop target        |
| `drop`      | Drop target     | User releases the dragged element           |

The `dragover` event is special: by default, elements do not accept drops. You must
call `event.preventDefault()` on `dragover` to signal that the element accepts a
drop. Without this, the `drop` event will never fire.

Here is the \_hyperscript for a single draggable task item:

```
on dragstart
  set window.draggedEl to me
  add .dragging to me
on dragover
  call event.preventDefault()
on dragenter
  if me is not window.draggedEl and I do not match .drag-placeholder
    if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
      call me.parentElement.insertBefore(window.draggedEl, me)
    else
      call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
    end
  end
on drop
  call event.preventDefault()
  remove .dragging from window.draggedEl
  send reorder to closest .task-list
```

This is the longest \_hyperscript in the chapter. Let us trace through it.

**`on dragstart`:** When the user starts dragging, we store a reference to the
dragged element in `window.draggedEl` (a global variable, using the `window`
object directly). We also add a `.dragging` class so CSS can dim the element or
show a ghost effect.

Why a global variable? Because the `dragover`, `dragenter`, and `drop` events fire
on *different* elements -- the ones under the cursor, not the one being dragged. We
need a way to access the dragged element from those handlers. A global is the
simplest approach. In \_hyperscript, you could also use `$draggedEl` (the `$` prefix
scopes to window), but `window.draggedEl` is more explicit and easier to read.

**`on dragover`:** We call `event.preventDefault()` to signal that this element
accepts drops. Without this line, the browser will reject the drop and the `drop`
event will never fire. This is a quirk of the HTML Drag and Drop API that trips
up everyone the first time.

**`on dragenter`:** This is where the visual reorder happens. When the dragged
element enters another task item:

1. We check that the target is not the dragged element itself (`me is not
   window.draggedEl`) and not a placeholder (`I do not match .drag-placeholder`).
2. We compare vertical positions using `getBoundingClientRect().top`. If the target
   is above the dragged element, we insert the dragged element before the target.
   If below, we insert it after.
3. `me.parentElement.insertBefore(window.draggedEl, me)` is standard DOM API --
   it moves the dragged element to a new position in the list.

The DOM reorder happens in real time as the user drags. The list visually updates on
every `dragenter`, giving immediate feedback.

**`on drop`:** When the user releases:

1. `event.preventDefault()` completes the drop (another required call).
2. Remove the `.dragging` class.
3. `send reorder to closest .task-list` dispatches a custom `reorder` event to the
   task list container. This is where we trigger the server sync.

The task list container listens for the `reorder` event and sends the new order to
the server:

```
on reorder
  set ids to []
  for item in .task-item in me
    call ids.push(item.dataset.id)
  end
  set #reorder-input's value to ids.join(',')
  send submit to #reorder-form
```

This handler:

1. Creates an empty array `ids`.
2. Iterates over every `.task-item` in the list (now in the new visual order).
3. Pushes each item's `data-id` attribute into the array.
4. Joins the IDs into a comma-separated string and sets it as the value of a
   hidden input.
5. Triggers a form submit, which fires an HTMX request.

The hidden form looks like this:

```html
<form id="reorder-form"
      hx-post="/tasks/reorder"
      hx-swap="none">
  <input type="hidden" id="reorder-input" name="order" value="">
</form>
```

`hx-swap="none"` means we do not need to update the DOM from the response -- the
DOM is already in the correct order from the drag. We just need to persist the order
on the server.

Here is the complete HTML for a draggable task list:

```html
<div class="task-list" id="task-list"
     _="on reorder
          set ids to []
          for item in .task-item in me
            call ids.push(item.dataset.id)
          end
          set #reorder-input's value to ids.join(',')
          send submit to #reorder-form">

  <div class="task-item" draggable="true" data-id="1"
       _="on dragstart
            set window.draggedEl to me
            add .dragging to me
          on dragover
            call event.preventDefault()
          on dragenter
            if me is not window.draggedEl and I do not match .drag-placeholder
              if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
                call me.parentElement.insertBefore(window.draggedEl, me)
              else
                call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
              end
            end
          on drop
            call event.preventDefault()
            remove .dragging from window.draggedEl
            send reorder to closest .task-list">
    <span>Buy groceries</span>
  </div>

  <!-- More task items with the same _hyperscript... -->
</div>

<form id="reorder-form"
      hx-post="/tasks/reorder"
      hx-swap="none">
  <input type="hidden" id="reorder-input" name="order" value="">
</form>
```

In Gleam, the draggable task item becomes a function:

```gleam
const drag_hyperscript = "on dragstart
  set window.draggedEl to me
  add .dragging to me
on dragover
  call event.preventDefault()
on dragenter
  if me is not window.draggedEl and I do not match .drag-placeholder
    if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
      call me.parentElement.insertBefore(window.draggedEl, me)
    else
      call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
    end
  end
on drop
  call event.preventDefault()
  remove .dragging from window.draggedEl
  send reorder to closest .task-list"

fn draggable_task_item(
  task: Task,
) -> element.Element(t) {
  html.div(
    [
      attribute.class("task-item"),
      attribute.attribute("draggable", "true"),
      attribute.attribute("data-id", task.id),
      attribute.attribute("_", drag_hyperscript),
    ],
    [
      html.span([], [element.text(task.title)]),
      copy_link_button(task.id),
    ],
  )
}
```

Notice that we extracted the \_hyperscript string into a `const`. When the same
\_hyperscript code repeats across many elements, putting it in a constant avoids
duplication and makes updates easier. In Gleam, `const` values must be literal
strings, which is exactly what we have here.

The task list container:

```gleam
const reorder_hyperscript = "on reorder
  set ids to []
  for item in .task-item in me
    call ids.push(item.dataset.id)
  end
  set #reorder-input's value to ids.join(',')
  send submit to #reorder-form"

fn task_list(tasks: List(Task)) -> element.Element(t) {
  html.div(
    [
      attribute.id("task-list"),
      attribute.class("task-list"),
      attribute.attribute("_", reorder_hyperscript),
    ],
    list.map(tasks, draggable_task_item),
  )
}
```

The hidden reorder form:

```gleam
fn reorder_form() -> element.Element(t) {
  html.form(
    [
      attribute.id("reorder-form"),
      hx.post("/tasks/reorder"),
      hx.swap(hx.SwapNone),
    ],
    [
      html.input([
        attribute.type_("hidden"),
        attribute.id("reorder-input"),
        attribute.name("order"),
        attribute.value(""),
      ]),
    ],
  )
}
```

And the server handler:

```gleam
fn reorder_tasks(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let order_string =
    list.key_find(form_data.values, "order")
    |> result.unwrap("")

  // Split "3,1,2,5,4" into ["3", "1", "2", "5", "4"]
  let ids = string.split(order_string, ",")

  // Update positions in the database.
  // Each ID gets a position equal to its index in the list.
  list.index_map(ids, fn(id, index) {
    actor.send(ctx.tasks, SetTaskPosition(id, index))
  })

  wisp.html_response("", 204)
}
```

The server receives a comma-separated list of task IDs in the new order. It splits
the string and updates each task's position in the database. The response is 204
No Content -- there is nothing to swap because the DOM is already correct.

The CSS for drag feedback:

```css
.task-item {
  cursor: grab;
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.task-item.dragging {
  opacity: 0.4;
  transform: scale(0.98);
}

.task-item:active {
  cursor: grabbing;
}
```

The `.dragging` class dims the element while it is being dragged. The `cursor`
changes from `grab` to `grabbing` on mouse down. These are small touches that make
the interaction feel polished.

**Edge case: dragend.** What if the user drags an element outside the list and
releases? The `drop` event will not fire (because there is no valid drop target).
The element will be stuck with the `.dragging` class. Add a `dragend` handler to
clean up:

```
on dragend
  remove .dragging from me
```

This fires whenever the drag operation ends, whether the drop was successful or not.
Add it to the task item's \_hyperscript alongside the other handlers.

### 1.6 When to Stop Using \_hyperscript

\_hyperscript is excellent for the patterns we have covered so far. But there is a
threshold where it stops being the right tool. Knowing where that threshold is will
save you from writing \_hyperscript that should have been JavaScript.

**Signs you should switch to JavaScript:**

1. **More than 15-20 lines.** If the \_hyperscript for a single element exceeds
   fifteen or twenty lines, the code is hard to read, hard to debug, and hard to
   maintain. The drag-and-drop pattern above is close to the limit.

2. **Complex data transformations.** \_hyperscript has no `map`, `filter`, or
   `reduce`. If you need to transform an array or process structured data, use
   JavaScript. \_hyperscript's `for` loop can iterate, but it cannot produce a new
   array or compute an aggregate in a readable way.

3. **Third-party API integration.** If you are calling multiple methods on a
   complex JavaScript object (e.g. a charting library, a WebSocket client, a
   file upload manager), writing those calls in \_hyperscript quickly becomes
   awkward. The `call` command works for single function calls, but chaining
   multiple calls with error handling is verbose.

4. **Shared logic.** If two or more elements need the *same* complex behaviour,
   a JavaScript function is easier to share and test than duplicating \_hyperscript
   strings. (\_hyperscript does have `install` and `behavior` for reuse, but for
   complex logic, JavaScript modules are a better fit.)

5. **You need a debugger.** \_hyperscript does not integrate with browser dev
   tools. You cannot set breakpoints in \_hyperscript code, inspect variable values
   mid-execution, or step through logic. If you are debugging complex behaviour,
   JavaScript gives you the full power of the browser's debugger.

**The extraction pattern:**

When you hit the threshold, the migration path is straightforward:

1. Write a JavaScript function in a separate file (e.g. `priv/static/js/reorder.js`).
2. Include the script in your layout.
3. Replace the \_hyperscript with a single `call` to your function:

Before (\_hyperscript):

```html
<div _="on dragstart set window.draggedEl to me ... (15+ lines)">
```

After (JavaScript + \_hyperscript bridge):

```html
<div _="on dragstart call handleDragStart(me, event)
        on dragover call handleDragOver(event)
        on dragenter call handleDragEnter(me, event)
        on drop call handleDrop(me, event)">
```

The \_hyperscript still handles the *wiring* -- listening for events and dispatching
to JavaScript functions. The *logic* lives in JavaScript where it can be debugged,
tested, and shared.

This is a clean separation:

- **\_hyperscript:** event wiring, class toggling, simple feedback.
- **JavaScript:** complex logic, data transformation, third-party integration.
- **HTMX:** server communication, DOM swapping.
- **Server (Gleam):** business logic, validation, persistence.

Each layer does what it does best.

**Decision flowchart:**

```
Is the behaviour purely client-side?
├── No  → Use HTMX (server round-trip)
└── Yes
    ├── < 10 lines of logic?
    │   └── Yes → Use _hyperscript
    ├── 10-20 lines?
    │   └── Maybe → _hyperscript if straightforward, JS if complex
    └── > 20 lines?
        └── Use JavaScript
```

### 1.7 Combining Behaviours (Multiple `on` Handlers)

A single element can have multiple \_hyperscript behaviours. The `_` attribute is a
single string, but you can include as many `on` handlers as you need, separated by
newlines:

```
on click ...
on mouseenter ...
on mouseleave ...
on htmx:afterSwap ...
```

\_hyperscript processes all handlers independently. An `on click` handler does not
interfere with an `on mouseenter` handler. They share the element's variable scope
(`:variables`), which is useful for coordination but requires care to avoid naming
collisions.

**Example: a task card with three behaviours.**

```
on mouseenter add .hovered to me
on mouseleave remove .hovered from me
on click toggle .selected on me
```

Three independent behaviours, zero conflicts. The mouseenter/mouseleave pair handles
hover styling. The click handler toggles selection. They coexist because they
respond to different events and manipulate different classes.

**Conflict scenario:** what if two handlers manipulate the *same* class?

```
on click add .active to me
on htmx:afterSwap remove .active from me
```

This works correctly because the events fire at different times. The click handler
adds `.active` when the user clicks. The HTMX handler removes it when the swap
completes. They cooperate rather than conflict.

The real danger is two handlers on the *same event* that contradict each other:

```
on click add .open to me
on click remove .open from me
```

Both fire on click. One adds the class, the other removes it. The result depends on
execution order (which is the order they appear in the `_` attribute). In this case,
the class would be added and then immediately removed -- the element would never
appear open. Use a single `on click toggle .open on me` instead.

**The `install` keyword for reuse.**

If you find yourself copying the same \_hyperscript across many elements,
\_hyperscript provides the `install` keyword to define reusable behaviours:

```html
<script type="text/hyperscript">
  behavior Hoverable
    on mouseenter add .hovered to me
    on mouseleave remove .hovered from me
  end
</script>
```

Then install the behaviour on any element:

```html
<div _="install Hoverable">Task 1</div>
<div _="install Hoverable">Task 2</div>
<div _="install Hoverable">Task 3</div>
```

You can install multiple behaviours:

```html
<div _="install Hoverable install Draggable">Task 1</div>
```

And combine installed behaviours with inline handlers:

```html
<div _="install Hoverable
        on click toggle .selected on me">
  Task 1
</div>
```

In Gleam, the behaviour definition goes in the layout's head as a script block:

```gleam
html.script(
  [attribute.type_("text/hyperscript")],
  "behavior Hoverable
    on mouseenter add .hovered to me
    on mouseleave remove .hovered from me
  end

  behavior Draggable
    on dragstart set window.draggedEl to me then add .dragging to me
    on dragend remove .dragging from me
  end",
)
```

And elements install the behaviours:

```gleam
html.div(
  [
    attribute.class("task-item"),
    attribute.attribute("_", "install Hoverable install Draggable"),
  ],
  [element.text(task.title)],
)
```

Behaviours are especially useful when the \_hyperscript code is longer than a few
lines. Instead of repeating ten lines of drag-and-drop code on every task item, you
define a `Draggable` behaviour once and install it everywhere.

**When to use `install` vs const strings:**

- **`install` (behaviour):** when the same behaviour applies to many elements and
  you want a semantic name. The behaviour is defined in a `<script>` block and
  installed by name. It reads well: `install Draggable`.
- **Const string:** when you are rendering elements from a Gleam function and the
  \_hyperscript is already in a variable. The const string is simpler -- no extra
  `<script>` block needed. But it lacks the semantic naming.

Both approaches work. Use whichever fits your project's style.

---

## 2. Code Walkthrough

We are going to enhance the Teamwork task board with all five patterns from the
theory section. Each step builds on the previous one, and by the end, a single
task card will have a character counter on its edit form, a copy-link button, and
drag-to-reorder -- all coexisting without conflicts.

### Step 1 -- Character Counter for Task Description Textarea

The task creation form currently has a title field. We are adding a description
field with a 200-character limit and a live counter.

```gleam
fn add_task_form() -> element.Element(t) {
  html.div([attribute.id("form-container")], [
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
      ],
      [
        // Title field
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
          ]),
        ]),

        // Description field with character counter
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("description")],
            [element.text("Description (optional)")],
          ),
          html.textarea(
            [
              attribute.id("description"),
              attribute.name("description"),
              attribute.maxlength(200),
              attribute.placeholder("Add details..."),
              attribute.attribute("_", "init
                put my value's length into #char-count's textContent
              on input
                put my value's length into #char-count's textContent
                if my value's length > 180
                  add .warning to #char-count
                else
                  remove .warning from #char-count
                end"),
            ],
            "",
          ),
          html.div([attribute.class("char-counter-row")], [
            html.span(
              [attribute.id("char-count"), attribute.class("char-counter")],
              [element.text("0")],
            ),
            html.span(
              [attribute.class("char-limit")],
              [element.text("/ 200")],
            ),
          ]),
        ]),

        // Submit
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}
```

The key details:

- `attribute.maxlength(200)` -- an `Int`, not a `String`. This is the Gleam API.
  The browser enforces the limit, and the \_hyperscript counter provides visual
  feedback.
- `attribute.attribute("_", "...")` -- the \_hyperscript code as a plain string.
  This is how every `_` attribute is set in Lustre.
- The `init` block sets the counter on page load. The `on input` block updates it
  on every change.
- The counter and limit are in a wrapper div with `class="char-counter-row"` for
  layout (flexbox, aligned right).

The CSS additions:

```css
.char-counter-row {
  display: flex;
  justify-content: flex-end;
  gap: 0;
  margin-top: 0.25rem;
}

.char-counter {
  font-size: 0.85rem;
  color: #718096;
  transition: color 0.2s ease;
}

.char-counter.warning {
  color: #e53e3e;
  font-weight: 600;
}

.char-limit {
  font-size: 0.85rem;
  color: #a0aec0;
}
```

### Step 2 -- Accordion for Board Settings Panel

The board settings page uses the exclusive accordion pattern. Each section contains
a form that submits to the server with HTMX.

```gleam
fn settings_page() -> element.Element(t) {
  html.div([attribute.class("settings-page")], [
    html.h2([], [element.text("Board Settings")]),
    html.div([attribute.class("accordion")], [
      // General section
      accordion_section(
        "General",
        html.form(
          [
            hx.post("/settings/general"),
            hx.target(hx.Selector("#settings-feedback")),
            hx.swap(hx.InnerHTML),
          ],
          [
            html.div([attribute.class("form-group")], [
              html.label(
                [attribute.for("board-name")],
                [element.text("Board name")],
              ),
              html.input([
                attribute.type_("text"),
                attribute.name("board_name"),
                attribute.id("board-name"),
                attribute.value("Teamwork"),
              ]),
            ]),
            html.button(
              [attribute.type_("submit"), attribute.class("btn")],
              [element.text("Save")],
            ),
          ],
        ),
      ),

      // Notifications section
      accordion_section(
        "Notifications",
        html.form(
          [
            hx.post("/settings/notifications"),
            hx.target(hx.Selector("#settings-feedback")),
            hx.swap(hx.InnerHTML),
          ],
          [
            html.div([attribute.class("form-group")], [
              html.label(
                [attribute.for("email-notify")],
                [element.text("Email notifications")],
              ),
              html.select(
                [attribute.name("email_notify"), attribute.id("email-notify")],
                [
                  html.option(
                    [attribute.value("all")],
                    "All activity",
                  ),
                  html.option(
                    [attribute.value("mentions")],
                    "Mentions only",
                  ),
                  html.option(
                    [attribute.value("none")],
                    "None",
                  ),
                ],
              ),
            ]),
            html.button(
              [attribute.type_("submit"), attribute.class("btn")],
              [element.text("Save")],
            ),
          ],
        ),
      ),

      // Permissions section
      accordion_section(
        "Permissions",
        html.div([], [
          html.p([], [element.text(
            "Manage roles and invite links for this board.",
          )]),
          html.a(
            [
              attribute.href("/settings/permissions"),
              attribute.class("btn"),
            ],
            [element.text("Manage Permissions")],
          ),
        ]),
      ),
    ]),

    // Feedback area for HTMX responses
    html.div([attribute.id("settings-feedback")], []),
  ])
}
```

The `accordion_section` function was defined in the theory section. Each section's
content is different -- one has a text input form, one has a select dropdown, and
one has a link. The accordion behaviour is identical across all of them because it
lives in the reusable `accordion_section` function.

The HTMX forms inside the accordion submit to different endpoints but target the
same feedback area (`#settings-feedback`). This is a clean pattern: the accordion
handles the open/close UI with \_hyperscript, and the forms handle server
communication with HTMX.

### Step 3 -- Copy-Task-URL-to-Clipboard Button

Every task item gets a copy-link button. We update the task item renderer to include
it:

```gleam
fn task_item(task: Task) -> element.Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item"),
      attribute.attribute("data-id", task.id),
    ],
    [
      html.div([attribute.class("task-content")], [
        html.span(
          [attribute.class("task-title")],
          [element.text(task.title)],
        ),
        case task.description {
          "" -> element.none()
          desc ->
            html.p(
              [attribute.class("task-description")],
              [element.text(desc)],
            )
        },
      ]),
      html.div([attribute.class("task-actions")], [
        // Copy link button
        copy_link_button(task.id),
        // Delete button
        html.button(
          [
            attribute.class("btn btn-danger btn-small"),
            hx.delete("/tasks/" <> task.id),
            hx.target(hx.Selector("#task-" <> task.id)),
            hx.swap(hx.OuterHTML),
          ],
          [element.text("Delete")],
        ),
      ]),
    ],
  )
}
```

The `copy_link_button` function was defined in section 1.4. It uses
`attribute.attribute("data-url", url)` to store the URL and \_hyperscript to
handle the copy, feedback, and error states.

Notice the use of `element.none()` for the conditional description. If the task
has no description (empty string), `element.none()` renders nothing. This is the
correct Lustre API for conditional rendering -- no `if` expression that returns
`html.text("")` (which would insert an empty text node), but `element.none()`
which inserts nothing at all.

### Step 4 -- Drag-to-Reorder Task List

Now we upgrade the task list to support drag-and-drop reordering. This requires
changes to the task item renderer, the task list container, and the addition of
the hidden reorder form.

First, update the task item to be draggable. We combine the drag handlers with
the existing task content and actions:

```gleam
const drag_hyperscript = "on dragstart
  set window.draggedEl to me
  add .dragging to me
on dragover
  call event.preventDefault()
on dragenter
  if me is not window.draggedEl and I do not match .drag-placeholder
    if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
      call me.parentElement.insertBefore(window.draggedEl, me)
    else
      call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
    end
  end
on drop
  call event.preventDefault()
  remove .dragging from window.draggedEl
  send reorder to closest .task-list
on dragend
  remove .dragging from me"

fn draggable_task_item(task: Task) -> element.Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item"),
      attribute.attribute("draggable", "true"),
      attribute.attribute("data-id", task.id),
      attribute.attribute("_", drag_hyperscript),
    ],
    [
      // Drag handle
      html.span(
        [attribute.class("drag-handle")],
        [element.text(":::")],
      ),

      // Task content
      html.div([attribute.class("task-content")], [
        html.span(
          [attribute.class("task-title")],
          [element.text(task.title)],
        ),
      ]),

      // Task actions
      html.div([attribute.class("task-actions")], [
        copy_link_button(task.id),
        html.button(
          [
            attribute.class("btn btn-danger btn-small"),
            hx.delete("/tasks/" <> task.id),
            hx.target(hx.Selector("#task-" <> task.id)),
            hx.swap(hx.OuterHTML),
          ],
          [element.text("Delete")],
        ),
      ]),
    ],
  )
}
```

Key points:

- `attribute.attribute("draggable", "true")` -- this is the HTML drag-and-drop API
  activation. The value must be the string `"true"`, not a boolean. In Lustre, we
  use `attribute.attribute()` (not a typed helper) because `draggable` is not in
  Lustre's standard attribute set.
- The drag handle (`:::`) is a visual affordance. Users see it and understand that
  dragging is possible. You could use a grip icon SVG instead.
- The `drag_hyperscript` const includes the `on dragend` cleanup handler from the
  edge case discussion in section 1.5.

Now the task list container with the reorder listener:

```gleam
fn task_list_view(tasks: List(Task)) -> element.Element(t) {
  html.div([], [
    html.div(
      [
        attribute.id("task-list"),
        attribute.class("task-list"),
        attribute.attribute("_", "on reorder
          set ids to []
          for item in .task-item in me
            call ids.push(item.dataset.id)
          end
          set #reorder-input's value to ids.join(',')
          send submit to #reorder-form"),
      ],
      list.map(tasks, draggable_task_item),
    ),

    // Hidden form for persisting reorder
    html.form(
      [
        attribute.id("reorder-form"),
        hx.post("/tasks/reorder"),
        hx.swap(hx.SwapNone),
      ],
      [
        html.input([
          attribute.type_("hidden"),
          attribute.id("reorder-input"),
          attribute.name("order"),
          attribute.value(""),
        ]),
      ],
    ),
  ])
}
```

The reorder route handler:

```gleam
fn reorder_tasks(req: wisp.Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let order_string =
    list.key_find(form_data.values, "order")
    |> result.unwrap("")

  let ids = string.split(order_string, ",")

  // Update each task's position in the store
  list.index_map(ids, fn(id, index) {
    actor.send(ctx.tasks, SetTaskPosition(id, index))
  })

  // No content -- the DOM is already in the correct order
  wisp.html_response("", 204)
}
```

Add the route to the router:

```gleam
["tasks", "reorder"] -> {
  use <- wisp.require_method(req, http.Post)
  reorder_tasks(req, ctx)
}
```

The CSS for drag handles and drag state:

```css
.drag-handle {
  cursor: grab;
  color: #a0aec0;
  font-weight: bold;
  letter-spacing: 2px;
  user-select: none;
  padding: 0 0.5rem;
}

.drag-handle:active {
  cursor: grabbing;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  margin-bottom: 0.5rem;
  background: white;
  transition: opacity 0.2s ease, transform 0.15s ease, box-shadow 0.15s ease;
}

.task-item.dragging {
  opacity: 0.4;
  transform: scale(0.98);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.task-content {
  flex: 1;
}

.task-actions {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}
```

### Step 5 -- Multi-Behaviour Task Card

Now let us combine everything. A fully-featured task card has:

1. **Drag-and-drop** for reordering.
2. **Copy-link** for sharing.
3. **Hover highlight** for visual feedback.
4. **Inline edit trigger** (from Chapter 21).
5. **Delete** via HTMX.

Five distinct behaviours on a single element. The \_hyperscript handles the first
three (plus inline edit triggering). HTMX handles the delete. They coexist because
they respond to different events and manipulate different properties.

Here is the full task card:

```gleam
const multi_behavior_hyperscript = "on dragstart
  set window.draggedEl to me
  add .dragging to me
on dragover
  call event.preventDefault()
on dragenter
  if me is not window.draggedEl and I do not match .drag-placeholder
    if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
      call me.parentElement.insertBefore(window.draggedEl, me)
    else
      call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
    end
  end
on drop
  call event.preventDefault()
  remove .dragging from window.draggedEl
  send reorder to closest .task-list
on dragend
  remove .dragging from me
on mouseenter
  add .hovered to me
on mouseleave
  remove .hovered from me"

fn full_task_card(task: Task) -> element.Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item"),
      attribute.attribute("draggable", "true"),
      attribute.attribute("data-id", task.id),
      attribute.attribute("_", multi_behavior_hyperscript),
    ],
    [
      // Drag handle
      html.span(
        [attribute.class("drag-handle")],
        [element.text(":::")],
      ),

      // Task content (double-click to inline edit, from Chapter 21)
      html.div(
        [
          attribute.class("task-content"),
          hx.get("/tasks/" <> task.id <> "/edit"),
          hx.trigger([hx.custom("dblclick")]),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
        ],
        [
          html.span(
            [attribute.class("task-title")],
            [element.text(task.title)],
          ),
          case task.description {
            "" -> element.none()
            desc ->
              html.p(
                [attribute.class("task-description")],
                [element.text(desc)],
              )
          },
        ],
      ),

      // Actions
      html.div([attribute.class("task-actions")], [
        copy_link_button(task.id),
        html.button(
          [
            attribute.class("btn btn-danger btn-small"),
            hx.delete("/tasks/" <> task.id),
            hx.target(hx.Selector("#task-" <> task.id)),
            hx.swap(hx.OuterHTML),
          ],
          [element.text("Delete")],
        ),
      ]),
    ],
  )
}
```

Let us count the behaviours:

1. **Drag-and-drop:** `on dragstart`, `on dragover`, `on dragenter`, `on drop`,
   `on dragend` -- five handlers in the \_hyperscript.
2. **Hover highlight:** `on mouseenter`, `on mouseleave` -- two handlers.
3. **Copy-link:** on the child button element (separate `_` attribute).
4. **Inline edit:** `hx-get` with `hx-trigger="dblclick"` on the content div -- an
   HTMX attribute, not \_hyperscript.
5. **Delete:** `hx-delete` on the delete button -- another HTMX attribute.

The \_hyperscript behaviours (1 and 2) live on the outer task card div. The
copy-link behaviour (3) lives on its own button. The HTMX behaviours (4 and 5)
live on the content div and the delete button, respectively.

There are zero conflicts because:

- Each behaviour responds to different events.
- Each behaviour manipulates different classes or targets different elements.
- The \_hyperscript handlers do not interfere with HTMX attributes.
- HTMX and \_hyperscript are designed to coexist -- \_hyperscript reinitializes
  after every HTMX swap.

**The only coordination point** is the drag handle. When the user grabs the drag
handle, the drag events fire. When they double-click the task content, the inline
edit triggers. These are spatially separated (handle on the left, content in the
middle), so the user's intent is clear.

---

## 3. Full Code Listing

Here is the complete code for all five patterns, integrated into the Teamwork
application.

### `src/teamwork/web.gleam`

```gleam
import gleam/http
import gleam/int
import gleam/list
import gleam/otp/actor
import gleam/result
import gleam/string
import lustre/attribute.{attribute}
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}
import hx

import teamwork/context.{type Context}
import teamwork/task.{type Task, Task, SetTaskPosition}

// ── _hyperscript Constants ──────────────────────────────────────────────

const char_counter_hs = "init
  put my value's length into #char-count's textContent
on input
  put my value's length into #char-count's textContent
  if my value's length > 180
    add .warning to #char-count
  else
    remove .warning from #char-count
  end"

const accordion_header_hs = "on click
  remove .open from .accordion-section in closest .accordion
  toggle .open on closest .accordion-section"

const drag_item_hs = "on dragstart
  set window.draggedEl to me
  add .dragging to me
on dragover
  call event.preventDefault()
on dragenter
  if me is not window.draggedEl and I do not match .drag-placeholder
    if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
      call me.parentElement.insertBefore(window.draggedEl, me)
    else
      call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
    end
  end
on drop
  call event.preventDefault()
  remove .dragging from window.draggedEl
  send reorder to closest .task-list
on dragend
  remove .dragging from me
on mouseenter
  add .hovered to me
on mouseleave
  remove .hovered from me"

const reorder_listener_hs = "on reorder
  set ids to []
  for item in .task-item in me
    call ids.push(item.dataset.id)
  end
  set #reorder-input's value to ids.join(',')
  send submit to #reorder-form"

const copy_btn_hs = "on click
  call navigator.clipboard.writeText(my dataset.url)
    catch e
      add .copy-error to me
      put 'Failed to copy' into me
      wait 2s
      remove .copy-error from me
      put 'Copy Link' into me
      halt
  end
  add .copied to me
  put 'Copied!' into me
  wait 2s
  remove .copied from me
  put 'Copy Link' into me"

// ── Layout ──────────────────────────────────────────────────────────────

fn layout(content: element.Element(t)) -> Response {
  let page =
    html.html([], [
      html.head([], [
        html.title([], "Teamwork -- Task Board"),
        // HTMX
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        // _hyperscript
        html.script(
          [attribute.src("https://unpkg.com/hyperscript.org@0.9.14")],
          "",
        ),
        // Stylesheets
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body([], [
        html.div([attribute.class("container")], [
          html.h1([], [element.text("Teamwork Task Board")]),
          content,
        ]),
      ]),
    ])

  let body = element.to_document_string(page)
  wisp.html_response(body, 200)
}

// ── Components ──────────────────────────────────────────────────────────

fn copy_link_button(task_id: String) -> element.Element(t) {
  let url = "/tasks/" <> task_id

  html.button(
    [
      attribute.class("btn btn-small copy-link-btn"),
      attribute.attribute("data-url", url),
      attribute.attribute("_", copy_btn_hs),
    ],
    [element.text("Copy Link")],
  )
}

fn accordion_section(
  title: String,
  content: element.Element(t),
) -> element.Element(t) {
  html.div([attribute.class("accordion-section")], [
    html.button(
      [
        attribute.class("accordion-header"),
        attribute.attribute("_", accordion_header_hs),
      ],
      [element.text(title)],
    ),
    html.div(
      [attribute.class("accordion-body")],
      [content],
    ),
  ])
}

fn task_card(task: Task) -> element.Element(t) {
  html.div(
    [
      attribute.id("task-" <> task.id),
      attribute.class("task-item"),
      attribute.attribute("draggable", "true"),
      attribute.attribute("data-id", task.id),
      attribute.attribute("_", drag_item_hs),
    ],
    [
      // Drag handle
      html.span(
        [attribute.class("drag-handle")],
        [element.text(":::")],
      ),

      // Task content (double-click to inline edit)
      html.div(
        [
          attribute.class("task-content"),
          hx.get("/tasks/" <> task.id <> "/edit"),
          hx.trigger([hx.custom("dblclick")]),
          hx.target(hx.Selector("#task-" <> task.id)),
          hx.swap(hx.OuterHTML),
        ],
        [
          html.span(
            [attribute.class("task-title")],
            [element.text(task.title)],
          ),
          case task.description {
            "" -> element.none()
            desc ->
              html.p(
                [attribute.class("task-description")],
                [element.text(desc)],
              )
          },
        ],
      ),

      // Actions
      html.div([attribute.class("task-actions")], [
        copy_link_button(task.id),
        html.button(
          [
            attribute.class("btn btn-danger btn-small"),
            hx.delete("/tasks/" <> task.id),
            hx.target(hx.Selector("#task-" <> task.id)),
            hx.swap(hx.OuterHTML),
          ],
          [element.text("Delete")],
        ),
      ]),
    ],
  )
}

// ── Forms ───────────────────────────────────────────────────────────────

fn add_task_form() -> element.Element(t) {
  html.div([attribute.id("form-container")], [
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
      ],
      [
        // Title
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
          ]),
        ]),

        // Description with character counter
        html.div([attribute.class("form-group")], [
          html.label(
            [attribute.for("description")],
            [element.text("Description (optional)")],
          ),
          html.textarea(
            [
              attribute.id("description"),
              attribute.name("description"),
              attribute.maxlength(200),
              attribute.placeholder("Add details..."),
              attribute.attribute("_", char_counter_hs),
            ],
            "",
          ),
          html.div([attribute.class("char-counter-row")], [
            html.span(
              [attribute.id("char-count"), attribute.class("char-counter")],
              [element.text("0")],
            ),
            html.span(
              [attribute.class("char-limit")],
              [element.text("/ 200")],
            ),
          ]),
        ]),

        // Submit
        html.button(
          [attribute.type_("submit"), attribute.class("btn")],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}

fn reorder_form() -> element.Element(t) {
  html.form(
    [
      attribute.id("reorder-form"),
      hx.post("/tasks/reorder"),
      hx.swap(hx.SwapNone),
    ],
    [
      html.input([
        attribute.type_("hidden"),
        attribute.id("reorder-input"),
        attribute.name("order"),
        attribute.value(""),
      ]),
    ],
  )
}

// ── Pages ───────────────────────────────────────────────────────────────

fn home_page(ctx: Context) -> Response {
  let tasks = actor.call(ctx.tasks, 1000, task.GetTasks)

  let content =
    html.div([], [
      add_task_form(),
      html.div(
        [
          attribute.id("task-list"),
          attribute.class("task-list"),
          attribute.attribute("_", reorder_listener_hs),
        ],
        list.map(tasks, task_card),
      ),
      reorder_form(),
    ])

  layout(content)
}

fn settings_page() -> Response {
  let content =
    html.div([attribute.class("settings-page")], [
      html.h2([], [element.text("Board Settings")]),
      html.div([attribute.class("accordion")], [
        accordion_section(
          "General",
          html.p([], [element.text(
            "Board name, description, visibility settings.",
          )]),
        ),
        accordion_section(
          "Notifications",
          html.p([], [element.text(
            "Email notifications, Slack integration, digest frequency.",
          )]),
        ),
        accordion_section(
          "Permissions",
          html.p([], [element.text(
            "Role assignments, invite links, guest access.",
          )]),
        ),
      ]),
    ])

  layout(content)
}

// ── Handlers ────────────────────────────────────────────────────────────

fn create_task(req: Request, ctx: Context) -> Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  let description =
    list.key_find(form_data.values, "description")
    |> result.unwrap("")

  let id = int.to_string(int.random(100_000))
  let new_task = Task(id: id, title: title, description: description, done: False, position: 0)
  actor.send(ctx.tasks, task.AddTask(new_task))

  let card = task_card(new_task)
  wisp.html_response(element.to_string(card), 201)
  |> wisp.set_header("HX-Trigger", "taskAdded")
}

fn delete_task(id: String, ctx: Context) -> Response {
  actor.send(ctx.tasks, task.DeleteTask(id))

  wisp.html_response("", 200)
  |> wisp.set_header("HX-Trigger", "taskAdded")
}

fn reorder_tasks(req: Request, ctx: Context) -> Response {
  use form_data <- wisp.require_form(req)

  let order_string =
    list.key_find(form_data.values, "order")
    |> result.unwrap("")

  let ids = string.split(order_string, ",")

  list.index_map(ids, fn(id, index) {
    actor.send(ctx.tasks, SetTaskPosition(id, index))
  })

  wisp.html_response("", 204)
}

// ── Router ──────────────────────────────────────────────────────────────

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(ctx)

    ["tasks"] -> {
      use <- wisp.require_method(req, http.Post)
      create_task(req, ctx)
    }

    ["tasks", "reorder"] -> {
      use <- wisp.require_method(req, http.Post)
      reorder_tasks(req, ctx)
    }

    ["tasks", id] -> {
      use <- wisp.require_method(req, http.Delete)
      delete_task(id, ctx)
    }

    ["settings"] -> settings_page()

    _ -> wisp.not_found()
  }
}
```

### `priv/static/css/style.css` (additions for this chapter)

```css
/* ── Character Counter ── */

.char-counter-row {
  display: flex;
  justify-content: flex-end;
  gap: 0;
  margin-top: 0.25rem;
}

.char-counter {
  font-size: 0.85rem;
  color: #718096;
  transition: color 0.2s ease;
}

.char-counter.warning {
  color: #e53e3e;
  font-weight: 600;
}

.char-limit {
  font-size: 0.85rem;
  color: #a0aec0;
}

/* ── Accordion ── */

.accordion {
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  overflow: hidden;
}

.accordion-section + .accordion-section {
  border-top: 1px solid #e2e8f0;
}

.accordion-header {
  display: block;
  width: 100%;
  padding: 0.85rem 1rem;
  background: #f7fafc;
  border: none;
  text-align: left;
  font-size: 1rem;
  font-weight: 600;
  color: #2d3748;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.accordion-header:hover {
  background: #edf2f7;
}

.accordion-body {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease, padding 0.3s ease;
  padding: 0 1rem;
}

.accordion-section.open .accordion-body {
  max-height: 500px;
  padding: 1rem;
}

/* ── Clipboard Copy ── */

.copy-link-btn {
  transition: background-color 0.2s ease, color 0.2s ease;
}

.copy-link-btn.copied {
  background-color: #48bb78;
  color: white;
}

.copy-link-btn.copy-error {
  background-color: #e53e3e;
  color: white;
}

/* ── Drag and Drop ── */

.drag-handle {
  cursor: grab;
  color: #a0aec0;
  font-weight: bold;
  letter-spacing: 2px;
  user-select: none;
  padding: 0 0.5rem;
}

.drag-handle:active {
  cursor: grabbing;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  margin-bottom: 0.5rem;
  background: white;
  transition: opacity 0.2s ease, transform 0.15s ease, box-shadow 0.15s ease;
}

.task-item.dragging {
  opacity: 0.4;
  transform: scale(0.98);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.task-item.hovered {
  background: #f7fafc;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
}

.task-content {
  flex: 1;
  cursor: default;
}

.task-title {
  font-weight: 500;
}

.task-description {
  font-size: 0.85rem;
  color: #718096;
  margin-top: 0.25rem;
}

.task-actions {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

/* ── Settings ── */

.settings-page {
  max-width: 600px;
}
```

---

## 4. Exercises

### Exercise 1 -- Enhanced Character Counter with Dynamic Colour

Extend the character counter so that it changes colour in three stages:

- **0-150 characters:** default grey colour (no class).
- **151-180 characters:** add a `.caution` class (yellow/orange).
- **181-200 characters:** add a `.warning` class (red).

You will need to extend the `if/else` to an `if/else if/else` chain. The
\_hyperscript syntax for this is:

```
if condition1
  ...
else if condition2
  ...
else
  ...
end
```

Make sure to remove both `.caution` and `.warning` in the "safe" branch (0-150),
so that the counter resets correctly when the user deletes text.

### Exercise 2 -- Accordion with Open-by-Default Section

Modify the accordion so that the first section starts open when the page loads.
You will need two things:

1. Add the `.open` class to the first `accordion-section` in the Gleam code.
2. Ensure the \_hyperscript still works correctly -- clicking the first section
   should close it, and clicking another should open that one and close the first.

Test by loading the page (first section should be open), clicking the second section
(first should close, second should open), and clicking the second section again
(to verify it toggles correctly).

### Exercise 3 -- Copy Full URL to Clipboard

Modify the clipboard copy button so that it copies the full URL (including the
domain) instead of just the relative path. Use the \_hyperscript expression
`window.location.origin` to construct the full URL at copy time:

```
call navigator.clipboard.writeText(window.location.origin + my dataset.url)
```

Verify by clicking "Copy Link" and pasting the result. It should be something like
`http://localhost:8000/tasks/42` instead of just `/tasks/42`.

### Exercise 4 -- Drag-and-Drop with Drop Zone Indicator

Enhance the drag-and-drop pattern so that when the user drags a task, a visual
"drop zone" indicator appears between items. Add a `.drop-indicator` class to the
target element during `dragenter`, and remove it on `dragleave` and `drop`.

The CSS for the indicator:

```css
.task-item.drop-indicator {
  border-top: 3px solid #4a6fa5;
}
```

You will need to add:

1. An `on dragenter` addition: `add .drop-indicator to me`.
2. An `on dragleave` handler: `remove .drop-indicator from me`.
3. Cleanup in `on drop`: `remove .drop-indicator from .task-item in closest .task-list`.

### Exercise 5 -- Behaviour Extraction

The drag-and-drop \_hyperscript is getting long. Extract it into a named
`behavior Draggable` in a `<script type="text/hyperscript">` block. Then
replace the inline \_hyperscript on each task item with
`install Draggable`.

Add the behaviour definition to the layout's `<head>`:

```gleam
html.script(
  [attribute.type_("text/hyperscript")],
  "behavior Draggable
    ...
  end",
)
```

Verify that drag-and-drop still works after the extraction. The behaviour should
be functionally identical to the inline version.

---

## 5. Exercise Solution Hints

Try each exercise on your own before reading these hints.

### Hint for Exercise 1

The three-stage counter uses nested conditions. Remove both classes in the safe
branch to handle the user deleting text:

```
on input
  put my value's length into #char-count's textContent
  if my value's length > 180
    remove .caution from #char-count
    add .warning to #char-count
  else if my value's length > 150
    remove .warning from #char-count
    add .caution to #char-count
  else
    remove .warning from #char-count
    remove .caution from #char-count
  end
```

Each branch explicitly removes the class it does not want. This prevents the
scenario where a user types past 180, gets `.warning`, deletes back to 160, and
still has `.warning` lingering because only `.caution` was added.

### Hint for Exercise 2

In Gleam, pass a boolean parameter to the `accordion_section` function:

```gleam
fn accordion_section(
  title: String,
  content: element.Element(t),
  open: Bool,
) -> element.Element(t) {
  let section_class = case open {
    True -> "accordion-section open"
    False -> "accordion-section"
  }
  html.div([attribute.class(section_class)], [
    // ... same as before
  ])
}
```

Then call it with `True` for the first section:

```gleam
accordion_section("General", general_content, True),
accordion_section("Notifications", notif_content, False),
accordion_section("Permissions", perms_content, False),
```

The \_hyperscript does not need to change. The `toggle .open` command works
correctly regardless of the initial state -- if the first section starts with
`.open`, clicking another section removes `.open` from all sections (including
the first) and then toggles it on the clicked one.

### Hint for Exercise 3

The change is in the `call` expression. Instead of:

```
call navigator.clipboard.writeText(my dataset.url)
```

Use:

```
call navigator.clipboard.writeText(window.location.origin + my dataset.url)
```

The `+` operator in \_hyperscript delegates to JavaScript's `+`, which concatenates
strings. `window.location.origin` returns `"http://localhost:8000"` in development,
and your actual domain in production.

### Hint for Exercise 4

Add the visual indicator classes. The key insight is that you need to clean up
the indicator on both `dragleave` (cursor leaves the element) and `drop` (item
is dropped). The `dragleave` handler goes on each task item:

```
on dragleave
  remove .drop-indicator from me
```

And in the `on drop` handler, clean up all indicators in the list:

```
on drop
  call event.preventDefault()
  remove .dragging from window.draggedEl
  remove .drop-indicator from .task-item in closest .task-list
  send reorder to closest .task-list
```

The `on dragenter` handler adds the indicator:

```
on dragenter
  add .drop-indicator to me
  if me is not window.draggedEl ...
```

Put the indicator addition before the position-swap logic so the visual feedback
appears immediately.

### Hint for Exercise 5

The behaviour definition wraps the existing \_hyperscript in a named block:

```html
<script type="text/hyperscript">
  behavior Draggable
    on dragstart
      set window.draggedEl to me
      add .dragging to me
    on dragover
      call event.preventDefault()
    on dragenter
      if me is not window.draggedEl and I do not match .drag-placeholder
        if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
          call me.parentElement.insertBefore(window.draggedEl, me)
        else
          call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
        end
      end
    on drop
      call event.preventDefault()
      remove .dragging from window.draggedEl
      send reorder to closest .task-list
    on dragend
      remove .dragging from me
  end
</script>
```

Then each task item becomes:

```gleam
attribute.attribute("_", "install Draggable
  on mouseenter add .hovered to me
  on mouseleave remove .hovered from me"),
```

The `install` keyword adds the behaviour, and you can still add inline handlers
after it. The hover behaviour is short enough to stay inline.

In Gleam, add the `<script>` block to the layout:

```gleam
html.script(
  [attribute.type_("text/hyperscript")],
  "behavior Draggable
    on dragstart
      set window.draggedEl to me
      add .dragging to me
    on dragover
      call event.preventDefault()
    on dragenter
      if me is not window.draggedEl and I do not match .drag-placeholder
        if me.getBoundingClientRect().top < window.draggedEl.getBoundingClientRect().top
          call me.parentElement.insertBefore(window.draggedEl, me)
        else
          call me.parentElement.insertBefore(window.draggedEl, me.nextSibling)
        end
      end
    on drop
      call event.preventDefault()
      remove .dragging from window.draggedEl
      send reorder to closest .task-list
    on dragend
      remove .dragging from me
  end",
)
```

---

## 6. Key Takeaways

1. **\_hyperscript recipes are 1-15 lines.** Every pattern in this chapter fits in
   a single `_` attribute. If your \_hyperscript exceeds fifteen lines, consider
   extracting the logic to JavaScript and using \_hyperscript only for event wiring.

2. **Element-scoped variables (`:variable`) are component state.** The character
   counter and clipboard-copy patterns both use the element itself as the source
   of truth. \_hyperscript's `:` prefix stores data directly on the DOM element,
   surviving across event firings without global state.

3. **`put ... into ...` is the workhorse command.** It updates text content,
   input values, innerHTML, and variables. It reads left-to-right: "put this value
   into that target." Most patterns use it at least once.

4. **CSS does the animation; \_hyperscript toggles the class.** The accordion
   animates with `max-height` transitions. The drag feedback uses `opacity` and
   `transform` transitions. \_hyperscript's job is to add and remove classes at the
   right time -- CSS handles the visual effect. Keep this separation clean.

5. **`call` is the JavaScript escape hatch.** The clipboard API, DOM manipulation
   (`insertBefore`), and `getBoundingClientRect()` are all JavaScript interop via
   `call`. When \_hyperscript lacks a native command for something, `call` lets you
   reach into JavaScript without leaving the `_` attribute.

6. **Multiple `on` handlers coexist without conflicts.** A single element can have
   handlers for `dragstart`, `dragover`, `dragenter`, `drop`, `dragend`,
   `mouseenter`, `mouseleave`, and more. They share the element's `:variables` but
   operate independently on different events. Conflicts only arise when two handlers
   for the *same* event contradict each other.

7. **Named behaviours (`install`) reduce duplication.** When the same \_hyperscript
   applies to many elements, define a `behavior` in a script block and `install` it.
   This keeps the inline `_` attributes short and gives the behaviour a semantic
   name.

8. **\_hyperscript and HTMX complement each other.** \_hyperscript handles
   client-side concerns (class toggling, drag feedback, clipboard copy). HTMX
   handles server communication (inline edit, delete, reorder persistence). They
   coexist on the same element without interference because \_hyperscript
   reinitializes after every HTMX swap.

9. **Know when to stop.** The decision flowchart is simple: purely client-side,
   under fifteen lines, no complex data transformation -- use \_hyperscript.
   Otherwise, use JavaScript. The drag-and-drop pattern in this chapter is near the
   upper bound of what \_hyperscript should handle. Beyond that, extract to a
   JavaScript module and bridge with `call`.

---

## What's Next

In **Chapter 24 -- Dynamic and Dependent Forms**, we will build forms where one
field's value determines the options in another. Think cascading dropdowns: selecting
a project loads its task categories, selecting a category loads its priority levels.
This pattern uses `hx-trigger="change"` and `hx-include` to send partial form
state to the server, which returns updated HTML for the dependent fields. It is one
of the most common patterns in real-world applications, and HTMX handles it with
zero JavaScript.
