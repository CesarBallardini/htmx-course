# Appendix B -- \_hyperscript: HTMX's Companion Language

**Scope:** Reference and exercises for \_hyperscript, the front-end scripting
language designed to pair with HTMX.

---

## 1. What Is \_hyperscript?

\_hyperscript is a small, experimental front-end scripting language created by
Carson Gross -- the same author as HTMX. Where HTMX handles communication with
the server (requests, swaps, triggers), \_hyperscript handles **client-side
behaviour** -- the kind of DOM manipulation, class toggling, and animation logic
that normally requires imperative JavaScript.

The design philosophy is simple: write behaviour declarations **directly in your
HTML** using a syntax that reads like English. Instead of:

```javascript
document.querySelector('#menu').addEventListener('click', function() {
  this.classList.toggle('open');
});
```

You write:

```html
<button _="on click toggle .open on #menu">Toggle Menu</button>
```

The `_` attribute is where all \_hyperscript code lives. The name is intentional
-- it is short, visually distinct, and stays out of the way.

### When to use it

- **DOM manipulation** that does not involve the server: toggling classes,
  showing/hiding elements, moving focus.
- **Animations and transitions:** fading, sliding, timed sequences.
- **Small interactions:** disabling a button during a request, dismissing a
  notification, closing a dropdown when clicking outside.
- **Complementing HTMX:** responding to HTMX lifecycle events (`htmx:beforeRequest`,
  `htmx:afterSwap`) with client-side effects.

### When NOT to use it

- **Complex application logic.** If the behaviour spans dozens of lines, write
  JavaScript instead.
- **Data processing.** \_hyperscript is for the DOM, not for algorithms.
- **Anything the server can do.** The HTMX philosophy is to keep logic on the
  server. Only reach for \_hyperscript when a server round-trip would add
  unacceptable latency (animations, immediate UI feedback).

### Status caveat

\_hyperscript is **pre-1.0 software**. The API is stable for the features
covered here, but the language is still evolving. Pin your version in production
and check the changelog before upgrading.

### Installation

Add a single `<script>` tag. The current version at the time of writing is
0.9.14:

```html
<script src="https://unpkg.com/hyperscript.org@0.9.14"></script>
```

That is it. No build step, no bundler, no configuration. \_hyperscript
automatically initialises on page load and re-initialises on HTMX swaps (it
listens for `htmx:load` events).

---

## 2. Core Syntax

### The `_` attribute

All \_hyperscript code lives in the `_` attribute of an HTML element:

```html
<button _="on click add .highlight to me">Highlight</button>
```

You can place multiple statements on one line, separated by `then`:

```html
<button _="on click add .highlight to me then wait 2s then remove .highlight from me">
  Flash
</button>
```

For readability with longer scripts, you can use line breaks inside the
attribute (HTML allows multi-line attribute values):

```html
<button _="on click
             add .highlight to me
             wait 2s
             remove .highlight from me">
  Flash
</button>
```

### Event handling

The `on` keyword registers an event listener:

```html
<!-- Standard DOM events -->
<div _="on click ...">
<input _="on keyup ...">
<form _="on submit ...">

<!-- Multiple events -->
<div _="on mouseenter add .hovered to me
        on mouseleave remove .hovered from me">
```

You can filter events with square brackets:

```html
<!-- Only fire on Enter key -->
<input _="on keyup[key === 'Enter'] send submitForm to closest <form/>">
```

### Targeting

\_hyperscript provides several ways to refer to elements:

| Expression             | Meaning                                            |
|------------------------|----------------------------------------------------|
| `me`                   | The element that owns the `_` attribute             |
| `it` / `its`          | The result of the last expression                   |
| `<.selector/>`        | CSS selector wrapped in angle brackets              |
| `the next <div/>`     | The next sibling matching the selector              |
| `the previous <li/>`  | The previous sibling matching the selector          |
| `the closest <form/>` | Walk up the DOM tree to find the nearest match      |
| `in me`               | Children of the current element                     |

Examples:

```html
<!-- Toggle a class on a different element -->
<button _="on click toggle .open on the next <div/>">Toggle</button>
<div class="panel">Content here</div>

<!-- Target by CSS selector -->
<button _="on click add .active to <#sidebar/>">Show Sidebar</button>

<!-- Target the closest form ancestor -->
<button _="on click remove .error from the closest <form/>">Clear Errors</button>
```

### Magic variables

| Variable  | Contains                                     |
|-----------|----------------------------------------------|
| `me`      | The current element (the one with the `_`)    |
| `it`      | The result of the last expression             |
| `event`   | The current event object                      |
| `target`  | `event.target` -- the element that fired the event |
| `detail`  | `event.detail` -- custom event data           |
| `body`    | `document.body`                               |

### Variable scoping

\_hyperscript has five scoping levels, each with its own prefix:

| Prefix      | Scope            | Example              | Meaning                       |
|-------------|------------------|----------------------|-------------------------------|
| (none)      | Local            | `set x to 10`        | Local to the current handler  |
| `:` (colon) | Element          | `set :count to 0`    | Stored on the DOM element     |
| `$`         | Global           | `set $theme to 'dark'` | Global (window-level)       |
| `@`         | Attribute        | `set @disabled to ''` | Sets an HTML attribute       |
| `*`         | Style            | `set *opacity to 0`  | Sets a CSS style property     |

Element-scoped variables (`:count`) are especially useful -- they persist across
event firings on the same element, acting like component-local state without
JavaScript.

---

## 3. Essential Commands

### `add` / `remove` / `toggle`

Manipulate classes and attributes:

```html
<!-- Classes -->
<button _="on click add .active to me">Activate</button>
<button _="on click remove .active from <.card/>">Deactivate All</button>
<button _="on click toggle .dark-mode on <body/>">Dark Mode</button>

<!-- Attributes -->
<button _="on click add @disabled to <#submit-btn/>">Disable</button>
```

### `put` / `set`

Set content and properties:

```html
<!-- Set innerHTML -->
<button _="on click put 'Done!' into me">Click Me</button>

<!-- Set a property -->
<button _="on click set <#input/>.value to ''">Clear Input</button>

<!-- Set a variable -->
<div _="on load set :count to 0">
```

### `show` / `hide`

Toggle element visibility:

```html
<button _="on click show <#details/>">Show Details</button>
<button _="on click hide me">Dismiss</button>

<!-- With a transition -->
<button _="on click hide me with *opacity">Fade Out</button>
```

### `wait` / `settle`

Async timing:

```html
<!-- Wait a fixed duration -->
<div _="on click add .flash to me wait 1s remove .flash from me">

<!-- Wait for HTMX to finish settling (transitions complete) -->
<div _="on htmx:afterSwap settle then add .highlight to me">
```

### `transition`

Animate CSS properties:

```html
<div _="on click transition *opacity to 0 over 500ms then remove me">
  Click to fade and remove
</div>

<div _="on mouseenter transition *transform to 'scale(1.05)' over 200ms
        on mouseleave transition *transform to 'scale(1)' over 200ms">
  Hover to grow
</div>
```

### `send` / `trigger`

Dispatch custom events:

```html
<!-- Send an event to another element -->
<button _="on click send closeModal to <#modal/>">Close</button>

<!-- Listen for that event -->
<div id="modal" _="on closeModal hide me with *opacity over 300ms">
  Modal content
</div>
```

### `fetch`

Make HTTP requests from \_hyperscript (for cases where HTMX is not involved):

```html
<button _="on click
            fetch /api/time as text
            put it into <#clock/>">
  Refresh Clock
</button>
```

### `call` / `get`

Interoperate with JavaScript:

```html
<!-- Call a JavaScript function -->
<button _="on click call alert('Hello!')">Alert</button>

<!-- Get a value from JavaScript -->
<div _="on load get new Date().toLocaleDateString() put it into me">
```

### Control flow

```html
<!-- If/else -->
<button _="on click
            if I match .active
              remove .active from me
              put 'Activate' into me
            else
              add .active to me
              put 'Deactivate' into me
            end">
  Activate
</button>

<!-- Repeat -->
<button _="on click repeat 3 times
             add .pulse to me
             wait 300ms
             remove .pulse from me
             wait 300ms
           end">
  Pulse 3x
</button>

<!-- For each -->
<button _="on click
            for item in <li/> in <#task-list/>
              add .checked to item
            end">
  Check All
</button>
```

---

## 4. HTMX Integration

### Automatic initialisation on HTMX swaps

When HTMX swaps new content into the DOM, it fires the `htmx:load` event.
\_hyperscript listens for this event and automatically initialises any `_`
attributes in the new content. You do not need to do anything special -- it
just works.

This means \_hyperscript code in HTMX responses works correctly without any
extra setup.

### Listening to HTMX events

HTMX fires events throughout its lifecycle. \_hyperscript can listen for any of
them:

| Event                  | When it fires                                    |
|------------------------|--------------------------------------------------|
| `htmx:beforeRequest`  | Just before the AJAX request is sent             |
| `htmx:afterRequest`   | After the request completes (success or error)   |
| `htmx:beforeSwap`     | Just before the DOM is updated                   |
| `htmx:afterSwap`      | After the DOM is updated                         |
| `htmx:afterSettle`    | After CSS transitions complete                   |
| `htmx:confirm`        | Before a request, allows cancellation            |
| `htmx:responseError`  | When the server returns an error status          |

Example -- disable a button while an HTMX request is in flight:

```html
<button hx-post="/tasks"
        hx-target="#task-list"
        hx-swap="beforeend"
        _="on htmx:beforeRequest add @disabled to me
           on htmx:afterRequest remove @disabled from me">
  Add Task
</button>
```

### Common patterns

**Loading state during HTMX request:**

```html
<form hx-post="/tasks" hx-target="#task-list"
      _="on htmx:beforeRequest
           add .loading to me
           put 'Saving...' into <button/> in me
         on htmx:afterRequest
           remove .loading from me
           put 'Add Task' into <button/> in me">
  <input type="text" name="title">
  <button type="submit">Add Task</button>
</form>
```

**Confirmation dialog before a destructive action:**

```html
<button hx-delete="/tasks/42"
        hx-target="#task-42"
        hx-swap="outerHTML"
        hx-confirm="Are you sure?"
        _="on htmx:confirm(issueRequest)
             halt the event
             call Swal.fire({title: 'Delete this task?', showCancelButton: true})
             if its.isConfirmed
               call issueRequest()
             end">
  Delete
</button>
```

**Post-swap animation -- highlight new content:**

```html
<div id="task-list"
     _="on htmx:afterSwap
          set newItems to the target's children
          for item in newItems
            add .flash to item
            wait 1s
            remove .flash from item
          end">
</div>
```

### Teamwork example: HTMX + \_hyperscript together

Here is how you might enhance the Teamwork task board from Chapter 16. The form
uses HTMX for the server request and \_hyperscript for client-side feedback:

```html
<form hx-post="/tasks"
      hx-target="#task-list"
      hx-swap="beforeend"
      _="on htmx:beforeRequest
           add @disabled to <button/> in me
           put 'Adding...' into <button/> in me
         on htmx:afterRequest
           remove @disabled from <button/> in me
           put 'Add Task' into <button/> in me
           if event.detail.successful
             set <input[name='title']/>.value to ''
           end">
  <input type="text" name="title" placeholder="New task...">
  <button type="submit">Add Task</button>
</form>
```

In Gleam with Lustre, the same pattern uses the `attribute` function for the
`_` attribute:

```gleam
html.form(
  [
    hx.post("/tasks"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.Beforeend),
    attribute("_", "on htmx:beforeRequest
      add @disabled to <button/> in me
      put 'Adding...' into <button/> in me
    on htmx:afterRequest
      remove @disabled from <button/> in me
      put 'Add Task' into <button/> in me
      if event.detail.successful
        set <input[name='title']/>.value to ''
      end"),
  ],
  [
    html.input([
      attribute.type_("text"),
      attribute.name("title"),
      attribute.placeholder("New task..."),
    ]),
    html.button(
      [attribute.type_("submit")],
      [element.text("Add Task")],
    ),
  ],
)
```

The `_` attribute is a plain string, so it works with any templating system.
Gleam does not need a special library for \_hyperscript -- just pass the code as
an attribute value.

---

## 5. Practical Examples

### Example 1: Toggle dark mode

```html
<button _="on click toggle .dark-mode on <body/>
           if <body/> matches .dark-mode
             put '🌙 Dark' into me
           else
             put '☀️ Light' into me
           end">
  ☀️ Light
</button>
```

### Example 2: Dismiss flash messages with fade animation

```html
<div class="flash-message"
     _="on click
          transition *opacity to 0 over 300ms
          then remove me">
  <p>Task created successfully!</p>
  <button>Dismiss</button>
</div>
```

### Example 3: Confirm-before-delete with HTMX

This pattern intercepts HTMX's built-in confirmation to use a custom dialog:

```html
<button hx-delete="/tasks/42"
        hx-target="#task-42"
        hx-swap="outerHTML"
        _="on click
             if :confirmed
               set :confirmed to false
             else
               halt the event
               call confirm('Delete this task? This cannot be undone.')
               if it
                 set :confirmed to true
                 trigger click on me
               end
             end">
  Delete
</button>
```

The `:confirmed` flag prevents an infinite loop: the first click halts the event and
shows the dialog; if the user confirms, we set the flag and re-trigger the click, which
this time passes through to HTMX.

A simpler approach uses `hx-confirm` directly:

```html
<button hx-delete="/tasks/42"
        hx-target="#task-42"
        hx-swap="outerHTML"
        hx-confirm="Delete this task? This cannot be undone.">
  Delete
</button>
```

The \_hyperscript version gives you control over the confirmation UI (you could
use a styled modal instead of the browser's native `confirm()` dialog).

### Example 4: Loading spinner during HTMX request

```html
<button hx-get="/slow-endpoint"
        hx-target="#results"
        _="on htmx:beforeRequest
             put my innerHTML into :originalContent
             put '<span class=\"spinner\"></span> Loading...' into me
             add @disabled to me
           on htmx:afterRequest
             put :originalContent into me
             remove @disabled from me">
  Load Data
</button>
```

The element-scoped variable `:originalContent` stores the button's original HTML
so it can be restored after the request completes.

### Example 5: Auto-dismiss notification after timeout

```html
<div class="notification"
     _="on load wait 5s
         transition *opacity to 0 over 500ms
         then remove me">
  Your changes have been saved.
</div>
```

This notification appears, waits 5 seconds, fades out over 500ms, and removes
itself from the DOM. No JavaScript needed.

### Example 6: Click-outside-to-close dropdown

```html
<div class="dropdown" _="on click from elsewhere hide me">
  <ul>
    <li>Option A</li>
    <li>Option B</li>
    <li>Option C</li>
  </ul>
</div>
```

The `from elsewhere` modifier listens for clicks that happen **outside** the
current element. When detected, the dropdown hides itself.

---

## 6. Exercises

### Task 1 -- Fade-out dismiss for an error banner

Add \_hyperscript to the error banner from Chapter 16 so that clicking the
"Dismiss" button fades the banner out over 300ms and then removes it from the
DOM.

**Requirements:**
- The banner should fade (transition `opacity` to 0 over 300ms).
- After the fade completes, the banner element should be removed entirely.
- The dismiss behaviour should work on banners injected by HTMX responses (not
  just the initial page load).

### Task 2 -- Confirmation dialog on task delete

Replace the browser's native `confirm()` on the delete button with a
\_hyperscript-powered confirmation. When the user clicks "Delete":

1. A confirmation message appears next to the button ("Are you sure?")
   with "Yes" and "No" buttons.
2. Clicking "Yes" triggers the HTMX delete request.
3. Clicking "No" hides the confirmation message.

**Requirements:**
- Do not use `window.confirm()` -- build it in HTML.
- The confirmation message should be positioned inline, next to the delete
  button.
- Use \_hyperscript's `send` and `on` to communicate between the confirmation
  buttons and the delete logic.

### Task 3 -- Loading state on form submit

Add a loading state to the task creation form:

1. When the form submits (HTMX request starts), disable the submit button and
   change its text to "Saving...".
2. When the request completes, re-enable the button and restore its original
   text.
3. On success, clear the title input field.

**Requirements:**
- Use \_hyperscript event handlers for `htmx:beforeRequest` and
  `htmx:afterRequest`.
- Use the element-scoped variable `:originalText` to store the button's
  original text.
- Check `event.detail.successful` to determine if the request succeeded.

### Task 4 -- Auto-scroll to new content after HTMX swap

When the infinite scroll loads a new batch of tasks, automatically scroll the
first new task into view smoothly.

**Requirements:**
- Listen for `htmx:afterSwap` on the task list container.
- Identify the first new element that was swapped in.
- Use JavaScript interop (`call`) to invoke `element.scrollIntoView({behavior: 'smooth'})`.

---

## 7. Exercise Solution Hints

Try each task on your own before reading these hints.

### Hint for Task 1

Use `transition` to animate opacity and `then remove me` to clean up:

```html
<button _="on click
            transition *opacity to 0 on the closest <.error-banner/> over 300ms
            then remove the closest <.error-banner/>">
  Dismiss
</button>
```

The key is targeting the parent banner element, not the button itself. Use
`the closest <.error-banner/>` to walk up the DOM tree.

Because \_hyperscript auto-initialises on `htmx:load`, this works on banners
injected by HTMX responses.

### Hint for Task 2

Create the confirmation message as a hidden sibling and toggle it:

```html
<div class="task-actions">
  <button _="on click hide me show the next <.confirm-delete/>">
    Delete
  </button>
  <span class="confirm-delete" style="display:none"
        _="on confirmed
              send doDelete to the previous <button/>
              hide me
           on cancelled
              hide me
              show the previous <button/>">
    Are you sure?
    <button _="on click send confirmed to the closest <.confirm-delete/>">Yes</button>
    <button _="on click send cancelled to the closest <.confirm-delete/>">No</button>
  </span>
</div>
```

The delete button's HTMX attributes (`hx-delete`, `hx-target`, `hx-swap`)
should be triggered by the `doDelete` event instead of by a direct click.

### Hint for Task 3

Place the \_hyperscript on the `<form>` element, since HTMX events bubble:

```html
_="on htmx:beforeRequest
     set :originalText to <button[type='submit']/> in me's innerHTML
     put 'Saving...' into <button[type='submit']/> in me
     add @disabled to <button[type='submit']/> in me
   on htmx:afterRequest
     put :originalText into <button[type='submit']/> in me
     remove @disabled from <button[type='submit']/> in me
     if event.detail.successful
       set <input[name='title']/> in me's value to ''
     end"
```

The `:originalText` variable uses element scope (the `:` prefix), so it persists
across the two separate event handlers on the same element.

### Hint for Task 4

Listen for `htmx:afterSwap` on the task list container:

```html
<div id="task-list"
     _="on htmx:afterSwap
          set firstNew to the first <.task-item:last-child/> in me
          if firstNew
            call firstNew.scrollIntoView({behavior: 'smooth', block: 'nearest'})
          end">
```

A simpler approach: add the scroll behaviour to the sentinel element itself,
since it is replaced by the new content. Use `htmx:afterSettle` (which fires
after transitions complete) for smoother results.

---

## 8. Key Takeaways

1. **\_hyperscript fills the gap between HTMX and JavaScript.** HTMX handles
   server communication; \_hyperscript handles client-side behaviour. Together,
   they cover most web interactions without writing vanilla JavaScript.

2. **The `_` attribute keeps behaviour in the HTML.** Like `hx-` attributes,
   \_hyperscript code lives on the element it applies to. This makes it easy to
   see what an element does by reading the markup alone.

3. **Natural-language-like syntax lowers the barrier.** Commands like
   `add .active to me`, `wait 2s`, and `toggle .open on the next <div/>` read
   almost like English. The learning curve is gentle for people who already
   know HTML.

4. **Element-scoped variables act like component state.** The `:variable` prefix
   stores data on the DOM element itself. This survives across event firings and
   does not pollute the global scope.

5. **HTMX integration is automatic.** \_hyperscript initialises on `htmx:load`,
   so it works in dynamically swapped content. You can also listen for any HTMX
   lifecycle event (`htmx:beforeRequest`, `htmx:afterSwap`, etc.) to add
   client-side effects.

6. **Use it for small interactions, not business logic.** Fade-outs, loading
   states, confirmations, class toggles -- these are ideal \_hyperscript use
   cases. Complex logic still belongs on the server or in JavaScript modules.

7. **It is pre-1.0 and evolving.** Pin your version, test before upgrading, and
   keep an eye on the changelog. The core features are stable, but the language
   continues to grow.

---

## 9. Further Resources

- **Official documentation:** https://hyperscript.org
- **Language reference:** https://hyperscript.org/reference/
- **Cookbook (HTMX + \_hyperscript recipes):** https://hyperscript.org/cookbook/
- **"Hypermedia Systems" book** by Carson Gross, Adam Stepinski, and Deniz Aksimsek
  -- covers the philosophy behind HTMX and \_hyperscript in depth.
- **Source code:** https://github.com/bigskysoftware/_hyperscript

> **See also:** **Chapter 23 -- \_hyperscript Practical Patterns** for
> production-ready recipes including character counters, accordions, clipboard
> copy, drag-and-drop reorder, and guidance on when to switch from
> \_hyperscript to JavaScript.
