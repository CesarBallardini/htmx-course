# Chapter 26 -- Accessible HTMX

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapter 25 added file uploads with progress indicators and
server-side validation. This chapter retrofits the entire
application with accessibility features -- ARIA live regions, focus management,
keyboard navigation, and automated testing.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain why dynamic DOM updates break screen reader experience and how ARIA live regions solve it.
2. Add `aria-live="polite"` regions to announce swap results.
3. Manage focus after swaps: new content, deletions, modal open/close.
4. Ensure all interactive elements are keyboard accessible (tab, Enter/Space, Escape).
5. Add ARIA roles and labels to HTMX components (modals, inline editing, loading states).
6. Test accessibility with DevTools, the axe extension, and a screen reader.

---

## 1. Theory

### 1.1 The Accessibility Challenge with Dynamic Content

HTMX is a beautiful tool. You declare a trigger, a request, and a swap target,
and the server sends back HTML that replaces part of the page. No full-page
reload. No JavaScript framework. Just HTML over the wire.

But here is the thing that nobody mentions in the first seventeen chapters of an
HTMX course: **screen readers have no idea what just happened**.

When a sighted user clicks "Add Task" and sees a new item appear in the list,
the visual feedback is immediate and obvious. The list grows. The form clears.
Maybe there is a subtle animation. The user knows the action succeeded because
they can *see* the result.

A screen reader user gets none of this. The screen reader was focused on the
"Add Task" button. The button is still there. The task list -- somewhere else
in the DOM -- just changed, but the screen reader did not announce it. From the
user's perspective, nothing happened. Did the task get created? Did an error
occur? Silence.

This is not an HTMX-specific problem. Every JavaScript framework that updates
the DOM dynamically has the same issue. React, Vue, Angular -- they all require
explicit accessibility work to announce changes. The difference is that HTMX
makes it *easier* to forget, because the updates feel so natural. You swap some
HTML, the page changes, and you move on. But "the page changes" is only true for
people who can see the page.

The Web Content Accessibility Guidelines (WCAG) define four principles,
summarized as POUR:

| Principle      | Meaning                                                    |
|----------------|------------------------------------------------------------|
| **Perceivable**  | Users must be able to perceive the content (see it, hear it, or feel it). |
| **Operable**     | Users must be able to operate the interface (keyboard, switch, voice). |
| **Understandable** | Users must be able to understand the content and how the interface works. |
| **Robust**       | Content must work across assistive technologies and browsers. |

Dynamic DOM updates from HTMX can violate all four principles if you are not
careful:

- **Perceivable:** New content appears visually but is not announced to screen
  readers.
- **Operable:** Focus is lost after a swap, trapping keyboard users in limbo.
- **Understandable:** Loading states and errors are communicated only through
  visual cues (spinners, red text).
- **Robust:** Missing ARIA attributes mean assistive technologies cannot
  interpret the structure of your components.

The good news is that fixing all of this is straightforward. HTMX does not
fight accessibility -- it just does not do it for you. You need to add three
things:

1. **ARIA live regions** to announce changes.
2. **Focus management** to guide the keyboard after swaps.
3. **Semantic HTML and ARIA attributes** to describe your components.

Let us tackle each one.

### 1.2 ARIA Live Regions (`polite` vs `assertive`, Placement)

ARIA live regions are the mechanism that tells screen readers "something changed
on the page, and you should announce it." They solve the core problem of dynamic
content: making invisible updates perceivable.

A live region is just an HTML element with the `aria-live` attribute. When the
*content* of that element changes, the screen reader reads the new content aloud.
The element must already exist in the DOM *before* the content changes -- you
cannot dynamically insert a live region and expect it to work. Plant it early,
update it later.

There are two politeness levels:

| Value         | Behaviour                                                    | Use When                              |
|---------------|--------------------------------------------------------------|---------------------------------------|
| `polite`      | Waits until the screen reader finishes its current announcement, then reads the new content. | Most situations: status updates, success messages, new list items. |
| `assertive`   | Interrupts whatever the screen reader is currently saying and reads the new content immediately. | Urgent situations only: errors, session timeouts, destructive actions. |

The vast majority of your announcements should use `polite`. Think of it as
the difference between raising your hand in a meeting versus shouting across
the room. `polite` waits its turn. `assertive` interrupts. If you overuse
`assertive`, the screen reader becomes a cacophony of interruptions and the user
cannot follow anything.

Here is the pattern. You place a visually hidden div somewhere in your layout --
typically right after `<body>`:

```html
<div id="live-region" class="sr-only" aria-live="polite" role="status"></div>
```

This div is invisible to sighted users (hidden with CSS, which we will cover
shortly) but visible to screen readers. When you want to announce something, you
change the text content of this div. The screen reader picks it up and reads it
aloud.

The `role="status"` attribute reinforces the semantics. It tells assistive
technologies that this element contains advisory information -- things the user
should know about but that are not urgent enough to interrupt.

**Placement matters.** The live region must be in the DOM at page load. If you
add it dynamically (for example, inside a swapped fragment), screen readers may
not register it as a live region. The safest approach is to put it in your layout
template, outside of any swap target. It lives there permanently, and you update
its content as needed.

**Clearing the region.** After an announcement, you should clear the region
after a short delay. If you leave the text in place and then set the same text
again (for example, the user adds two tasks in a row and both trigger "Task
added successfully"), the screen reader may not announce the second one because
the content did not change. Clearing it first and then setting it again ensures
each announcement is recognized as new content.

Here is the full lifecycle:

```
1. Page loads with empty live region: <div id="live-region"></div>
2. User adds a task.
3. Server responds. HTMX swaps the task into the list.
4. _hyperscript sets the live region text: "Task added successfully"
5. Screen reader announces: "Task added successfully"
6. After 3 seconds, _hyperscript clears the live region: ""
7. Ready for the next announcement.
```

### 1.3 Focus Management (After Add, Delete, Modal Open/Close)

Focus management is the second pillar of accessible dynamic content. When the
DOM changes, the browser does not automatically move focus to the new content.
Worse, if the element that *had* focus is removed from the DOM (because it was
inside a swap target), focus falls back to `<body>` -- which means the keyboard
user is now at the top of the page, miles away from where they were working.

There are four common scenarios you need to handle:

**Scenario 1: After adding content (e.g., adding a task).**
The user fills out a form and submits. A new task appears in the list. Where
should focus go?

Best practice: Return focus to the form input so the user can immediately add
another task. This is the "conveyor belt" pattern -- the user stays in the
same position and can keep working without navigating back to the form.

**Scenario 2: After deleting content (e.g., deleting a task).**
The user clicks "Delete" on a task. The task is removed from the DOM. The
element that had the delete button no longer exists. Where should focus go?

Best practice: Move focus to the next sibling in the list. If there is no next
sibling (the user deleted the last item), move to the previous sibling. If the
list is now empty, move to a sensible fallback like the "Add Task" input or
a heading.

**Scenario 3: Opening a modal or dialog.**
The user clicks a button that opens a modal. Focus should move *into* the modal,
typically to the first focusable element (a close button or the first form
field). Focus must also be *trapped* inside the modal -- pressing Tab should
cycle through the modal's focusable elements without escaping to the page behind
it.

**Scenario 4: Closing a modal or dialog.**
When the modal closes, focus should return to the element that opened it. This
is the "return trip" pattern. The user leaves a context (the page), enters
another context (the modal), and when they are done, they return to exactly
where they left.

The challenge with HTMX is that swaps happen asynchronously. You cannot just
call `.focus()` in your Gleam code because the server does not control the
client-side DOM. You need a client-side mechanism that runs *after* the swap
completes.

There are three approaches:

1. **`htmx:afterSwap` event + _hyperscript.** Listen for the swap event and
   move focus in the handler. This is the most flexible approach.
2. **`autofocus` attribute.** HTML's native `autofocus` works on elements that
   are swapped into the DOM, but only on some browsers and only for the first
   such element. It is unreliable for anything beyond the simplest case.
3. **HTMX `hx-on:htmx:after-swap` attribute.** Inline JavaScript that runs
   after the swap. This works but mixes concerns and is harder to maintain.

We will use _hyperscript for most focus management because it is declarative,
readable, and integrates naturally with HTMX events.

### 1.4 Keyboard Navigation Patterns (Tab Order, Roving Tabindex)

Keyboard navigation is the third pillar. Every interactive element in your
application must be reachable and operable with the keyboard alone. This is not
just for screen reader users -- it is also for power users who prefer the
keyboard, users with motor impairments who use switch devices, and anyone whose
mouse battery just died.

The rules are simple:

1. **Use semantic HTML.** `<button>`, `<a>`, `<input>`, `<select>`, and
   `<textarea>` are all natively focusable and respond to keyboard events. If
   you use these elements, keyboard navigation works for free.

2. **Do not put click handlers on divs or spans.** A `<div onclick="...">` is
   not focusable, not announced as interactive, and does not respond to
   Enter or Space. Always use `<button>` for actions and `<a>` for navigation.

3. **Tab order follows DOM order.** The Tab key moves focus through focusable
   elements in the order they appear in the HTML source. If your visual layout
   does not match the DOM order (because of CSS flexbox or grid reordering),
   keyboard users will experience a confusing, non-linear focus sequence.

4. **Use `tabindex` carefully.** `tabindex="0"` adds an element to the natural
   tab order (useful for custom components). `tabindex="-1"` makes an element
   programmatically focusable but removes it from the tab order (useful for
   focus management targets). Never use `tabindex` values greater than 0 --
   they create a parallel tab order that is nearly impossible to maintain.

For complex components like lists of tasks, the **roving tabindex** pattern is
the gold standard:

| Pattern         | How it works                                             | Best for               |
|-----------------|----------------------------------------------------------|------------------------|
| **Tab through all** | Every item in the list has `tabindex="0"`. Tab moves through each one. | Short lists (< 5 items). |
| **Roving tabindex** | Only one item has `tabindex="0"` (the "active" one). All others have `tabindex="-1"`. Arrow keys move the active item. Tab leaves the list. | Long lists, toolbars, menus. |
| **aria-activedescendant** | A container has focus. Arrow keys change which descendant is "active" (via `aria-activedescendant`). No tabindex changes needed. | Comboboxes, autocomplete. |

The roving tabindex pattern works like this:

1. The list container is not focusable itself.
2. The first item has `tabindex="0"`. All other items have `tabindex="-1"`.
3. When the user presses the Down arrow, the current item gets `tabindex="-1"`
   and the next item gets `tabindex="0"` and receives focus.
4. When the user presses the Up arrow, the reverse happens.
5. When the user presses Tab, focus leaves the list entirely and moves to the
   next focusable element after the list. One Tab press exits the whole list,
   not just one item.

This pattern is important for the Teamwork task list. With 50 tasks, you do not
want the user to press Tab 50 times to get past the list. Tab should enter the
list (landing on the active item), arrow keys should navigate within the list,
and Tab should exit the list.

### 1.5 ARIA for HTMX Components (`aria-busy`, `aria-expanded`, `role="status"`)

ARIA attributes add semantic meaning to HTML elements. They tell screen readers
things that cannot be expressed with HTML alone: "this region is currently
loading," "this section is collapsed," "this element acts as a dialog."

Here are the ARIA attributes most relevant to HTMX applications:

| Attribute          | Values                     | Purpose                                                         |
|--------------------|----------------------------|-----------------------------------------------------------------|
| `aria-live`        | `polite`, `assertive`, `off` | Announces content changes to screen readers.                   |
| `aria-busy`        | `true`, `false`            | Indicates that an element is being updated and assistive technology should wait before announcing changes. |
| `aria-expanded`    | `true`, `false`            | Indicates whether a collapsible section is open or closed.      |
| `aria-label`       | String                     | Provides an accessible name for elements that lack visible text. |
| `aria-labelledby`  | ID reference               | Points to another element that labels this one.                 |
| `aria-describedby` | ID reference               | Points to another element that describes this one.              |
| `aria-hidden`      | `true`, `false`            | Hides an element from assistive technology (but not visually).  |
| `aria-disabled`    | `true`, `false`            | Indicates that an element is disabled.                          |
| `role`             | `status`, `alert`, `dialog`, `region`, etc. | Defines the element's purpose when HTML semantics are insufficient. |

**`aria-busy` for loading states.**
When HTMX sends a request and the response has not arrived yet, the target
element is in a "loading" state. Sighted users see this through CSS indicators
(spinners, opacity changes). Screen reader users need `aria-busy="true"` to
know that the content is being updated and they should wait.

The flow is:

1. Before the request: set `aria-busy="true"` on the swap target.
2. The screen reader pauses announcements for this region.
3. After the response: remove `aria-busy` (or set it to `"false"`).
4. The screen reader announces the new content (if the region is a live region)
   or waits for the user to navigate to it.

**`aria-expanded` for collapsible sections.**
If your task board has expandable task details (click a task to see its
description, due date, assignee), the trigger element should have
`aria-expanded="false"` when collapsed and `aria-expanded="true"` when expanded.
This tells screen readers the current state without requiring the user to
navigate into the section to discover whether it is open.

**`role="dialog"` for modals.**
When you open a modal via HTMX (swapping a dialog into the page), the container
needs `role="dialog"` and `aria-modal="true"`. This tells screen readers to
treat it as a dialog and helps with focus trapping.

**`role="status"` for the live region.**
As discussed in 1.2, `role="status"` on the live region reinforces the ARIA
semantics. It is equivalent to `aria-live="polite"` but more explicit about the
element's purpose.

### 1.6 Testing Accessibility (DevTools, axe, NVDA/VoiceOver)

Accessibility is not something you "think looks right." It needs to be tested
with tools and, ideally, with actual screen readers. There are four levels of
testing, each catching different issues:

**Level 1: Browser DevTools (Accessibility Inspector).**
Every modern browser has an accessibility tree inspector. In Chrome, open
DevTools, go to the Elements panel, and click the "Accessibility" tab. This
shows you how the browser interprets your HTML for assistive technologies. You
can see:

- The computed role of each element.
- Its accessible name and description.
- Its state (`expanded`, `disabled`, `busy`).
- Any ARIA attribute violations.

This is your first line of defense. Check it after every major change.

**Level 2: Automated scanning with axe.**
The axe browser extension (by Deque) scans your page and reports accessibility
violations. It checks for:

- Missing alt text on images.
- Insufficient colour contrast.
- Missing form labels.
- Invalid ARIA attributes.
- Focus order issues.

axe catches about 30-40% of accessibility issues automatically. That sounds low,
but those are the issues that are easiest to fix and easiest to miss during
manual testing.

| Tool       | Type      | Cost  | What it catches                                    |
|------------|-----------|-------|----------------------------------------------------|
| axe        | Extension | Free  | ARIA issues, contrast, labels, structure           |
| Lighthouse | Built-in  | Free  | Similar to axe (uses axe-core internally)          |
| WAVE       | Extension | Free  | Visual overlay of issues on the page               |
| pa11y      | CLI       | Free  | Automated testing in CI pipelines                  |

**Level 3: Keyboard testing.**
Put your mouse in a drawer. Use only the keyboard to complete every task in your
application:

- Can you reach every interactive element with Tab?
- Can you activate buttons with Enter or Space?
- Can you close modals with Escape?
- Can you navigate lists with arrow keys?
- Does focus move logically after each action?
- Can you always see where focus is (focus indicator/outline)?

Keyboard testing catches issues that no automated tool can detect: logical
focus order, missing keyboard handlers, and focus traps.

**Level 4: Screen reader testing.**
This is the gold standard. Use an actual screen reader and listen to what it
announces:

| Screen Reader | Platform     | Cost                          |
|---------------|--------------|-------------------------------|
| **NVDA**      | Windows      | Free (open source)            |
| **VoiceOver** | macOS / iOS  | Free (built into the OS)      |
| **JAWS**      | Windows      | Commercial (~$1,000/year)     |
| **TalkBack**  | Android      | Free (built into the OS)      |

For development, NVDA on Windows or VoiceOver on macOS are the practical
choices. You do not need to test with all of them during development, but you
should test with at least one.

The basic VoiceOver workflow on macOS:

1. Press `Cmd + F5` to enable VoiceOver.
2. Use `Control + Option + Right Arrow` to move through elements.
3. Listen to what VoiceOver announces for each element.
4. Perform your HTMX interactions and listen for announcements.
5. Press `Cmd + F5` again to disable VoiceOver.

The basic NVDA workflow on Windows:

1. Download and install NVDA from nvaccess.org.
2. Press `Insert` (the NVDA key) to activate commands.
3. Use `Tab` to move through interactive elements.
4. Use arrow keys to read through content.
5. Perform interactions and listen for announcements.

### 1.7 Common Mistakes (div Instead of button, Missing alt Text, Color-Only Indicators)

Here is a table of the most common accessibility mistakes in HTMX applications
and how to fix them. Print this out. Tape it to your monitor. Check every
component against it.

| Mistake | Why it fails | Fix |
|---------|-------------|-----|
| Using `<div>` or `<span>` with `hx-post` as a clickable element | Not focusable, not announced as interactive, no keyboard support | Use `<button>` with `hx-post`. Buttons are focusable, announced as "button," and respond to Enter/Space. |
| Missing labels on form inputs | Screen readers announce "edit text" with no context about what to type | Add `<label for="id">` or `aria-label="..."` to every input. |
| Color-only error indicators (red border, no text) | Users who cannot see colour (colourblind or screen reader users) get no error feedback | Add text error messages alongside colour changes. Use icons with `aria-label` too. |
| Removing elements without moving focus | Focus falls to `<body>`, losing the user's place | Move focus to the next sibling, previous sibling, or a sensible fallback. |
| Using `aria-live` on the swap target itself | The entire content of the target is read aloud after every swap, which is overwhelming | Use a separate, small live region that only contains announcement text. |
| Not clearing the live region between announcements | Identical consecutive announcements are not read aloud (no content change detected) | Clear the region, wait a tick, then set the new text. |
| Modal opens without focus moving inside | Keyboard users cannot interact with the modal without tabbing through the entire page | Move focus to the first focusable element in the modal on open. |
| Modal does not trap focus | Tab moves focus behind the modal to elements the user cannot see | Use _hyperscript or JavaScript to trap focus within the modal. |
| Loading states communicated only with spinners | Screen readers do not see spinners | Add `aria-busy="true"` during loading. Announce completion via live region. |
| Images without alt text | Screen readers announce the filename ("IMG_2847.jpg") instead of meaningful content | Add `alt="description"` to every `<img>`. Use `alt=""` for decorative images. |
| Using `tabindex="5"` or other positive values | Creates a parallel tab order that is impossible to maintain as the page changes | Use only `tabindex="0"` (add to tab order) and `tabindex="-1"` (programmatic focus only). |
| Skip links missing | Keyboard users must Tab through the entire header/nav on every page | Add a "Skip to main content" link as the first focusable element. |

These are not edge cases. They are the most common accessibility failures on
the web today. The fix for each one is typically a single line of HTML.

---

## 2. Code Walkthrough

We are going to retrofit the Teamwork task board with comprehensive
accessibility support. Every step builds on code from previous chapters. By the
end, the application will be usable with a keyboard alone, will announce
all dynamic changes to screen readers, and will pass an axe audit with zero
violations.

### Step 1 -- Add a Live Region to the Layout

The live region is the foundation of accessible HTMX. It is a visually hidden
element that screen readers monitor for changes. When you put text into it,
the screen reader speaks that text aloud.

First, we need the `.sr-only` CSS class. This is the standard technique for
visually hiding content while keeping it accessible to screen readers. Unlike
`display: none` or `visibility: hidden`, which hide content from *everyone*
(including screen readers), `.sr-only` moves the element off-screen so it
is invisible but still present in the accessibility tree.

Add this to your `style.css`:

```css
/* ─── Screen reader only ─── */

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

This is the same technique used by Bootstrap, Tailwind, and every major CSS
framework. The element takes up zero visual space but remains in the DOM and the
accessibility tree.

Now update the layout function to include the live region and _hyperscript:

```gleam
import lustre/attribute.{attribute}
import lustre/element
import lustre/element/html
import wisp

fn layout(content: element.Element(t)) -> wisp.Response {
  let page =
    html.html([attribute("lang", "en")], [
      html.head([], [
        html.title([], "Teamwork -- Task Board"),
        // Core HTMX
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        // _hyperscript for client-side accessibility behaviour
        html.script(
          [attribute.src(
            "https://unpkg.com/hyperscript.org@0.9.14",
          )],
          "",
        ),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body([], [
        // Skip link -- first focusable element on the page
        html.a(
          [
            attribute.href("#main-content"),
            attribute.class("skip-link"),
          ],
          [element.text("Skip to main content")],
        ),
        // ARIA live region -- visually hidden, screen-reader accessible
        html.div(
          [
            attribute.id("live-region"),
            attribute.class("sr-only"),
            attribute.attribute("aria-live", "polite"),
            attribute.attribute("role", "status"),
          ],
          [element.text("")],
        ),
        // Assertive live region for errors
        html.div(
          [
            attribute.id("live-region-error"),
            attribute.class("sr-only"),
            attribute.attribute("aria-live", "assertive"),
            attribute.attribute("role", "alert"),
          ],
          [element.text("")],
        ),
        // Main content
        html.main(
          [attribute.id("main-content")],
          [
            html.h1([], [element.text("Teamwork Task Board")]),
            content,
          ],
        ),
      ]),
    ])

  let body = element.to_document_string(page)

  wisp.html_response(body, 200)
}
```

Let us break down what we added:

1. **`lang="en"` on `<html>`.** Screen readers use this to select the correct
   pronunciation engine. Without it, a screen reader configured for French
   might try to pronounce your English text with French phonetics.

2. **Skip link.** The very first focusable element on the page. When a keyboard
   user presses Tab, this link appears. Pressing Enter jumps focus to
   `#main-content`, skipping the navigation. This is WCAG 2.4.1 (bypass
   blocks).

3. **Polite live region** (`#live-region`). For routine announcements: "Task
   added," "Task completed," "3 results found." Uses `aria-live="polite"` so it
   waits for the screen reader to finish before speaking.

4. **Assertive live region** (`#live-region-error`). For error announcements:
   "Validation failed: title is required," "Server error, please try again."
   Uses `aria-live="assertive"` so it interrupts immediately.

5. **`<main>` element.** Wrapping the content in a semantic `<main>` element
   gives screen readers a landmark to navigate to. It also serves as the skip
   link target.

Add the skip link CSS:

```css
/* ─── Skip link ─── */

.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  padding: 0.5rem 1rem;
  background: #1a202c;
  color: #fff;
  z-index: 1000;
  text-decoration: none;
  font-weight: 600;
}

.skip-link:focus {
  top: 0;
}
```

The skip link is positioned off-screen by default. When it receives focus
(the user pressed Tab on page load), it slides into view at the top of the page.
When focus moves away, it slides back off-screen. Sighted users who do not use
the keyboard never see it. Keyboard users see it exactly when they need it.

### Step 2 -- Announce Swap Results via `HX-Trigger-After-Swap` and \_hyperscript

Now we need to actually put text into the live region when things happen. There
are several ways to do this. The cleanest is to combine the `HX-Trigger`
response header with _hyperscript event listeners.

The pattern:

1. The server includes `HX-Trigger-After-Swap: {"taskAdded": "Task added successfully"}` in the response header.
2. HTMX receives the response, performs the swap, and then fires a `taskAdded`
   custom event on the body with the detail from the header value.
3. A _hyperscript listener on the body catches the event and writes the message
   into the live region.

However, there is a simpler approach that does not require JSON in headers. We
use `HX-Trigger` (which fires after the swap by default when used as a response
header) and _hyperscript with hardcoded messages. Since our server already sends
`HX-Trigger: taskAdded` when a task is created, we just need the _hyperscript
side.

Add the _hyperscript listener to the body element:

```gleam
html.body(
  [
    attribute.attribute("_", "
      on taskAdded from body
        put 'Task added successfully' into #live-region
        wait 3s
        put '' into #live-region
      end
      on taskDeleted from body
        put 'Task deleted' into #live-region
        wait 3s
        put '' into #live-region
      end
      on taskUpdated from body
        put 'Task updated' into #live-region
        wait 3s
        put '' into #live-region
      end
      on validationFailed from body
        put 'Validation error. Please check the form.' into #live-region-error
        wait 5s
        put '' into #live-region-error
      end
    "),
  ],
  [
    // ... skip link, live regions, main content ...
  ],
)
```

Each listener follows the same structure:

1. Listen for a named event on the body.
2. Write an announcement message into the appropriate live region.
3. Wait a few seconds.
4. Clear the live region so the next announcement will be detected as new
   content.

The `wait 3s` is important. Without it, if you clear the region immediately, the
screen reader might not have time to pick up the change. Three seconds gives
every screen reader ample time to announce the text.

Now update the server handlers to send the appropriate trigger events. We
already have `taskAdded` from Chapter 16. Let us add `taskDeleted` and the
validation failure event:

```gleam
fn create_task(req: Request, ctx: Context) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      let id = int.to_string(int.random(100_000))
      let new_task = Task(id: id, title: valid_title, done: False)
      actor.send(ctx.tasks, task.AddTask(new_task))

      let task_html = task_item(id, valid_title)
      wisp.html_response(element.to_string(task_html), 201)
      |> wisp.set_header("HX-Trigger", "taskAdded")
    }

    Error(errors) -> {
      let form_html = add_task_form(errors, form_data.values)
      wisp.html_response(element.to_string(form_html), 422)
      |> wisp.set_header("HX-Trigger", "validationFailed")
    }
  }
}

fn delete_task(_req: Request, id: String, ctx: Context) -> wisp.Response {
  actor.send(ctx.tasks, task.DeleteTask(id))

  wisp.html_response("", 200)
  |> wisp.set_header("HX-Trigger", "taskDeleted")
}
```

Notice that the 422 validation response now sends `HX-Trigger: validationFailed`.
This fires the _hyperscript listener on the body, which announces the error
through the assertive live region. The sighted user sees the form errors
visually; the screen reader user hears "Validation error. Please check the form."

The combination of visual feedback (the form re-renders with error messages) and
audible feedback (the live region announces the error) ensures that both user
groups are informed.

### Step 3 -- Focus Management After Task Addition

When a user adds a task, the form submits and HTMX swaps the new task into the
list. But what happens to focus? If the form re-renders (because we are using
OOB to clear it), the input the user was focused on is replaced with a new DOM
element. Focus falls to `<body>`.

We want focus to return to the title input so the user can immediately add
another task. This is the "conveyor belt" pattern.

Update the form to manage focus after the swap:

```gleam
fn add_task_form(
  errors: List(validation.ValidationError),
  values: List(#(String, String)),
) -> element.Element(t) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.div(
      [
        attribute.id("form-errors"),
        attribute.attribute("aria-live", "polite"),
      ],
      case title_errors {
        [] -> []
        errs ->
          list.map(errs, fn(e) {
            html.span(
              [
                attribute.class("error-text"),
                attribute.attribute("role", "alert"),
              ],
              [element.text(validation.error_message(e))],
            )
          })
      },
    ),
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        // Focus management: after swap, refocus the title input
        attribute.attribute("_", "
          on htmx:afterSwap
            set the value of #title to ''
            focus() the #title
        "),
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
            attribute.attribute("aria-required", "true"),
            attribute.attribute("aria-invalid", case title_errors {
              [] -> "false"
              _ -> "true"
            }),
            attribute.attribute("aria-describedby", "form-errors"),
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
        ]),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn"),
          ],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}
```

Several accessibility improvements here:

1. **`aria-required="true"`** on the input tells screen readers this field is
   mandatory. The screen reader will announce "Task title, required, edit text."

2. **`aria-invalid="true"/"false"`** tells screen readers whether the current
   value is valid. After a validation error, the screen reader announces
   "Task title, invalid entry, required, edit text" -- much more informative
   than just "Task title, edit text."

3. **`aria-describedby="form-errors"`** links the input to the error container.
   When the user focuses the input, the screen reader reads both the label
   ("Task title") and the description (the error messages).

4. **`aria-live="polite"` on `#form-errors`** ensures that when error messages
   appear (after a validation failure), they are announced without the user
   needing to navigate to them.

5. **The _hyperscript focus handler** clears the input value and returns focus
   to `#title` after a successful swap. The user hears the live region
   announce "Task added successfully" and finds themselves back in the input
   field, ready to type another task.

### Step 4 -- Focus Management After Task Deletion

Deletion is trickier. When a task is deleted, the entire `<li>` that contained
the delete button is removed from the DOM. Focus has nowhere to go. We need to
figure out the right destination *before* the element is removed.

The strategy: before the delete request fires, find the next sibling task item.
If there is no next sibling, find the previous one. If the list will be empty
after deletion, fall back to the input field.

Update the task item component:

```gleam
fn task_item(id: String, title: String) -> element.Element(t) {
  html.li(
    [
      attribute.id("task-" <> id),
      attribute.class("task-item"),
    ],
    [
      html.span(
        [attribute.class("task-title")],
        [element.text(title)],
      ),
      html.button(
        [
          attribute.class("btn btn-danger btn-small"),
          hx.delete("/tasks/" <> id),
          hx.target(hx.Selector("#task-" <> id)),
          hx.swap(hx.OuterHTML),
          attribute.attribute("aria-label", "Delete task: " <> title),
          // Focus management: before deletion, find the next focusable element
          attribute.attribute("_", "
            on htmx:beforeRequest
              set nextItem to the next <li/> from the closest <li/>
              if nextItem is null
                set nextItem to the previous <li/> from the closest <li/>
              end
              if nextItem is not null
                set nextItem to the first <button/> in nextItem
              end
            end
            on htmx:afterSwap from body
              if nextItem is not null
                focus() the nextItem
              else
                focus() the #title
              end
            end
          "),
        ],
        [element.text("Delete")],
      ),
    ],
  )
}
```

Let us walk through the _hyperscript logic:

1. **`on htmx:beforeRequest`**: Before the delete request fires, we look for
   the next `<li>` sibling. If there is none (we are deleting the last item),
   we look for the previous `<li>`. If we find a sibling, we drill into it to
   find the first `<button>` (so focus lands on an interactive element, not
   just the list item).

2. **`on htmx:afterSwap from body`**: After the swap completes (the deleted
   item is gone from the DOM), we focus the target we found earlier. If the
   list is now empty and there is no sibling, we focus the title input.

The `aria-label="Delete task: " <> title` is also critical. Without it, the
screen reader announces "Delete, button" -- the user does not know *which* task
they are about to delete. With the label, it announces "Delete task: Buy
groceries, button." Clear, unambiguous, and safe.

### Step 5 -- Keyboard Navigation for Inline Editing

In Chapter 21, we built inline editing for task titles. A user clicks the task
title, it turns into an input, they type, and they save. But what about keyboard
users? They need to:

1. Navigate to the task title with Tab or arrow keys.
2. Activate inline editing with Enter.
3. Type the new title.
4. Save with Enter.
5. Cancel with Escape.

Here is the accessible inline edit component:

```gleam
fn task_title_editable(id: String, title: String) -> element.Element(t) {
  html.span(
    [
      attribute.id("title-display-" <> id),
      attribute.class("task-title editable"),
      attribute.attribute("tabindex", "0"),
      attribute.attribute("role", "button"),
      attribute.attribute("aria-label", "Edit task: " <> title
        <> ". Press Enter to edit."),
      // Click or Enter/Space activates inline editing
      hx.get("/tasks/" <> id <> "/edit"),
      hx.target(hx.Selector("#title-display-" <> id)),
      hx.swap(hx.OuterHTML),
      hx.trigger([hx.click()]),
      attribute.attribute("_", "
        on keydown[key=='Enter' or key==' ']
          halt the event
          trigger click on me
        end
      "),
    ],
    [element.text(title)],
  )
}

fn task_title_edit_form(id: String, title: String) -> element.Element(t) {
  html.form(
    [
      attribute.id("title-display-" <> id),
      attribute.class("inline-edit-form"),
      hx.put("/tasks/" <> id),
      hx.target(hx.Selector("#title-display-" <> id)),
      hx.swap(hx.OuterHTML),
      // Focus the input when the form appears
      attribute.attribute("_", "
        on load
          focus() the first <input/> in me
        end
      "),
    ],
    [
      html.input([
        attribute.type_("text"),
        attribute.name("title"),
        attribute.value(title),
        attribute.attribute("aria-label", "New title for task"),
        // Save on Enter, cancel on Escape
        attribute.attribute("_", "
          on keydown[key=='Escape']
            halt the event
            htmx.ajax('GET', '/tasks/" <> id <> "/title', {target: closest <form/>})
          end
        "),
      ]),
      html.button(
        [
          attribute.type_("submit"),
          attribute.class("btn btn-small"),
          attribute.attribute("aria-label", "Save task title"),
        ],
        [element.text("Save")],
      ),
      html.button(
        [
          attribute.type_("button"),
          attribute.class("btn btn-small btn-secondary"),
          attribute.attribute("aria-label", "Cancel editing"),
          hx.get("/tasks/" <> id <> "/title"),
          hx.target(hx.Selector("#title-display-" <> id)),
          hx.swap(hx.OuterHTML),
        ],
        [element.text("Cancel")],
      ),
    ],
  )
}
```

The display version (`task_title_editable`) has these accessibility features:

- **`tabindex="0"`** makes the span focusable via Tab key.
- **`role="button"`** tells screen readers this is an interactive element.
- **`aria-label`** provides clear instructions: "Edit task: Buy groceries. Press
  Enter to edit."
- **The _hyperscript keydown handler** listens for Enter or Space and triggers
  the same action as a click. This is the keyboard equivalent of clicking.

The edit form (`task_title_edit_form`) has:

- **Auto-focus on load.** When the form swaps in, _hyperscript immediately
  focuses the input field. The user does not need to Tab to find it.
- **Escape to cancel.** The _hyperscript on the input listens for Escape and
  fetches the read-only title view, effectively reverting the edit.
- **Explicit labels on all buttons.** "Save task title" and "Cancel editing"
  instead of just "Save" and "Cancel" -- context matters for screen reader
  users who may not perceive the surrounding visual layout.

### Step 6 -- `aria-busy` During HTMX Requests via \_hyperscript

When HTMX sends a request, there is a delay before the response arrives. During
this delay, sighted users see loading indicators (spinners, opacity changes).
Screen reader users need `aria-busy` to know that content is being updated.

The pattern uses _hyperscript to toggle `aria-busy` on the swap target:

```gleam
fn task_list_container(tasks: List(Task)) -> element.Element(t) {
  html.ul(
    [
      attribute.id("task-list"),
      attribute.class("task-list"),
      attribute.attribute("role", "list"),
      attribute.attribute("aria-label", "Task list"),
      // Toggle aria-busy during HTMX requests targeting this element
      attribute.attribute("_", "
        on htmx:beforeRequest from <* [hx-target='#task-list']/> or me
          add @aria-busy='true' to me
        end
        on htmx:afterRequest from <* [hx-target='#task-list']/> or me
          remove @aria-busy from me
        end
      "),
    ],
    list.map(tasks, fn(task) { task_item(task.id, task.title) }),
  )
}
```

The _hyperscript listens for HTMX request events from any element that targets
`#task-list` (or from the list itself). Before the request, it sets
`aria-busy="true"`. After the request, it removes the attribute.

While `aria-busy` is `true`, screen readers know to hold off on announcing
changes to this region. When it is removed, the screen reader can announce the
updated content (if the region is a live region) or let the user discover it
through navigation.

You can also combine `aria-busy` with a visual loading indicator. Here is the
CSS:

```css
/* ─── Loading state ─── */

[aria-busy="true"] {
  opacity: 0.6;
  pointer-events: none;
  position: relative;
}

[aria-busy="true"]::after {
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  width: 1.5rem;
  height: 1.5rem;
  margin: -0.75rem 0 0 -0.75rem;
  border: 3px solid #e2e8f0;
  border-top-color: #4a6fa5;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
```

This CSS uses the `aria-busy` attribute *as the styling hook*. No extra classes
needed. The `[aria-busy="true"]` selector reduces opacity and adds a spinner.
When `aria-busy` is removed, the styling reverts automatically. This is
accessibility-driven CSS -- the ARIA attribute serves double duty as both a
semantic signal for screen readers and a styling hook for visual users.

This approach is superior to using a separate CSS class for loading states
because:

1. You cannot forget to add the visual indicator -- it is automatic when
   `aria-busy` is set.
2. You cannot have a visual loading state without the ARIA attribute -- they
   are the same thing.
3. The code is DRY. One mechanism serves both purposes.

### Step 7 -- Accessibility Audit Walkthrough with axe

Now that we have added all the accessibility features, let us run an audit to
verify everything works. Here is a step-by-step walkthrough using the axe
browser extension.

**Step 1: Install axe DevTools.**
Go to the Chrome Web Store (or Firefox Add-ons) and search for "axe DevTools."
Install the free version.

**Step 2: Open your Teamwork application.**
Start the Gleam server and navigate to `http://localhost:8080` in your browser.

**Step 3: Open DevTools and find the axe tab.**
Press `F12` to open DevTools. Look for the "axe DevTools" tab (it may be behind
the `>>` overflow menu). Click it.

**Step 4: Run a scan.**
Click "Scan ALL of my page." axe will analyze the current DOM and report any
violations.

**Step 5: Review the results.**
axe categorizes issues by severity:

| Severity  | Meaning                                                   | Examples                          |
|-----------|-----------------------------------------------------------|-----------------------------------|
| Critical  | Blocks access for some users entirely                     | Missing form labels, no keyboard access |
| Serious   | Significantly impacts usability for some users            | Insufficient colour contrast       |
| Moderate  | Causes difficulty but does not block access                | Missing landmark regions           |
| Minor     | Best practice violation with minimal user impact          | Redundant ARIA roles               |

**Step 6: Fix violations.**
Click each violation to see which element caused it and how to fix it. axe
provides a "More info" link with detailed guidance for each issue.

Here is an example of what axe might find before our accessibility work:

```
BEFORE accessibility work:

  Critical:  2 violations
    - Form element without a label (1 element)
    - Button without accessible name (3 elements)

  Serious:   1 violation
    - Colour contrast insufficient (2 elements)

  Moderate:  2 violations
    - Page does not have a main landmark
    - Document does not have a lang attribute
```

And after applying everything from this chapter:

```
AFTER accessibility work:

  Critical:  0 violations
  Serious:   0 violations
  Moderate:  0 violations
  Minor:     0 violations

  "Congratulations! No accessibility violations found."
```

**Step 7: Test with keyboard.**
Close axe. Put your mouse away. Start from the top of the page:

1. Press Tab. The skip link should appear. Press Enter to skip to main content.
2. Press Tab. Focus should be on the task title input.
3. Type "Accessibility test" and press Enter. The task should be added. Focus
   should return to the input.
4. Tab to the first task's delete button. The screen reader should announce
   "Delete task: Accessibility test, button."
5. Press Enter to delete. Focus should move to the next task's delete button
   (or to the input if the list is empty).
6. Tab to an editable task title. Press Enter. The inline edit form should
   appear with focus on the input.
7. Press Escape. The edit should cancel and focus should return to the title
   display.

If all seven steps work, your application is keyboard accessible.

**Step 8: Test with a screen reader.**
Enable NVDA (Windows) or VoiceOver (macOS). Repeat the keyboard test and listen:

1. On page load, the screen reader should announce the page title and
   landmark structure.
2. When you add a task, you should hear "Task added successfully" from the
   live region.
3. When you delete a task, you should hear "Task deleted."
4. When a validation error occurs, you should hear "Validation error. Please
   check the form."
5. While a request is in flight, `aria-busy` prevents premature announcements
   of the loading state.

---

## 3. Full Code Listing

Here is the complete updated module with all accessibility features integrated.

### `src/teamwork/web.gleam`

```gleam
import gleam/http
import gleam/int
import gleam/list
import gleam/otp/actor
import gleam/result
import lustre/attribute.{attribute}
import lustre/element
import lustre/element/html
import wisp.{type Request, type Response}
import hx

import teamwork/context.{type Context}
import teamwork/task.{type Task, Task}
import teamwork/validation

// ── Layout ──────────────────────────────────────────────────────────────

fn layout(content: element.Element(t)) -> Response {
  let page =
    html.html([attribute("lang", "en")], [
      html.head([], [
        html.title([], "Teamwork -- Task Board"),
        html.meta([
          attribute("charset", "utf-8"),
        ]),
        html.meta([
          attribute.name("viewport"),
          attribute("content", "width=device-width, initial-scale=1"),
        ]),
        // Core HTMX
        html.script(
          [attribute.src("https://unpkg.com/htmx.org@2.0.8")],
          "",
        ),
        // _hyperscript for client-side accessibility behaviour
        html.script(
          [attribute.src(
            "https://unpkg.com/hyperscript.org@0.9.14",
          )],
          "",
        ),
        html.link([
          attribute.rel("stylesheet"),
          attribute.href("/static/css/style.css"),
        ]),
      ]),
      html.body(
        [
          // Global event listeners for live region announcements
          attribute("_", "
            on taskAdded from body
              put 'Task added successfully' into #live-region
              wait 3s
              put '' into #live-region
            end
            on taskDeleted from body
              put 'Task deleted' into #live-region
              wait 3s
              put '' into #live-region
            end
            on taskUpdated from body
              put 'Task updated' into #live-region
              wait 3s
              put '' into #live-region
            end
            on validationFailed from body
              put 'Validation error. Please check the form.' into #live-region-error
              wait 5s
              put '' into #live-region-error
            end
          "),
        ],
        [
          // Skip link -- first focusable element on the page
          html.a(
            [
              attribute.href("#main-content"),
              attribute.class("skip-link"),
            ],
            [element.text("Skip to main content")],
          ),
          // ARIA live region -- polite announcements
          html.div(
            [
              attribute.id("live-region"),
              attribute.class("sr-only"),
              attribute.attribute("aria-live", "polite"),
              attribute.attribute("role", "status"),
            ],
            [element.text("")],
          ),
          // ARIA live region -- assertive (errors)
          html.div(
            [
              attribute.id("live-region-error"),
              attribute.class("sr-only"),
              attribute.attribute("aria-live", "assertive"),
              attribute.attribute("role", "alert"),
            ],
            [element.text("")],
          ),
          // Main content area with landmark
          html.main(
            [attribute.id("main-content")],
            [
              html.h1([], [element.text("Teamwork Task Board")]),
              content,
            ],
          ),
        ],
      ),
    ])

  let body = element.to_document_string(page)

  wisp.html_response(body, 200)
}

// ── Navigation ──────────────────────────────────────────────────────────

fn nav_bar() -> element.Element(t) {
  html.nav(
    [
      attribute.class("nav"),
      attribute.attribute("aria-label", "Main navigation"),
    ],
    [
      html.a(
        [
          attribute.href("/"),
          hx.boost(True),
        ],
        [element.text("Board")],
      ),
      html.a(
        [
          attribute.href("/boards"),
          hx.boost(True),
        ],
        [element.text("All Boards")],
      ),
      html.a(
        [
          attribute.href("/settings"),
          hx.boost(True),
        ],
        [element.text("Settings")],
      ),
    ],
  )
}

// ── Components ──────────────────────────────────────────────────────────

fn task_item(id: String, title: String) -> element.Element(t) {
  html.li(
    [
      attribute.id("task-" <> id),
      attribute.class("task-item"),
    ],
    [
      // Editable title with keyboard support
      html.span(
        [
          attribute.id("title-display-" <> id),
          attribute.class("task-title editable"),
          attribute.attribute("tabindex", "0"),
          attribute.attribute("role", "button"),
          attribute.attribute("aria-label", "Edit task: " <> title
            <> ". Press Enter to edit."),
          hx.get("/tasks/" <> id <> "/edit"),
          hx.target(hx.Selector("#title-display-" <> id)),
          hx.swap(hx.OuterHTML),
          hx.trigger([hx.click()]),
          attribute.attribute("_", "
            on keydown[key=='Enter' or key==' ']
              halt the event
              trigger click on me
            end
          "),
        ],
        [element.text(title)],
      ),
      // Delete button with accessible label and focus management
      html.button(
        [
          attribute.class("btn btn-danger btn-small"),
          hx.delete("/tasks/" <> id),
          hx.target(hx.Selector("#task-" <> id)),
          hx.swap(hx.OuterHTML),
          attribute.attribute("aria-label", "Delete task: " <> title),
          attribute.attribute("_", "
            on htmx:beforeRequest
              set nextItem to the next <li/> from the closest <li/>
              if nextItem is null
                set nextItem to the previous <li/> from the closest <li/>
              end
              if nextItem is not null
                set nextItem to the first <button/> in nextItem
              end
            end
            on htmx:afterSwap from body
              if nextItem is not null
                focus() the nextItem
              else
                focus() the #title
              end
            end
          "),
        ],
        [element.text("Delete")],
      ),
    ],
  )
}

fn task_title_edit_form(id: String, title: String) -> element.Element(t) {
  html.form(
    [
      attribute.id("title-display-" <> id),
      attribute.class("inline-edit-form"),
      hx.put("/tasks/" <> id),
      hx.target(hx.Selector("#title-display-" <> id)),
      hx.swap(hx.OuterHTML),
      attribute.attribute("_", "
        on load
          focus() the first <input/> in me
        end
      "),
    ],
    [
      html.input([
        attribute.type_("text"),
        attribute.name("title"),
        attribute.value(title),
        attribute.attribute("aria-label", "New title for task"),
        attribute.attribute("_", "
          on keydown[key=='Escape']
            halt the event
            htmx.ajax('GET', '/tasks/" <> id <> "/title', {target: closest <form/>})
          end
        "),
      ]),
      html.button(
        [
          attribute.type_("submit"),
          attribute.class("btn btn-small"),
          attribute.attribute("aria-label", "Save task title"),
        ],
        [element.text("Save")],
      ),
      html.button(
        [
          attribute.type_("button"),
          attribute.class("btn btn-small btn-secondary"),
          attribute.attribute("aria-label", "Cancel editing"),
          hx.get("/tasks/" <> id <> "/title"),
          hx.target(hx.Selector("#title-display-" <> id)),
          hx.swap(hx.OuterHTML),
        ],
        [element.text("Cancel")],
      ),
    ],
  )
}

fn add_task_form(
  errors: List(validation.ValidationError),
  values: List(#(String, String)),
) -> element.Element(t) {
  let title_value =
    list.key_find(values, "title")
    |> result.unwrap("")

  let title_errors = validation.errors_for(errors, "title")

  html.div([attribute.id("form-container")], [
    html.div(
      [
        attribute.id("form-errors"),
        attribute.attribute("aria-live", "polite"),
      ],
      case title_errors {
        [] -> []
        errs ->
          list.map(errs, fn(e) {
            html.span(
              [
                attribute.class("error-text"),
                attribute.attribute("role", "alert"),
              ],
              [element.text(validation.error_message(e))],
            )
          })
      },
    ),
    html.form(
      [
        hx.post("/tasks"),
        hx.target(hx.Selector("#task-list")),
        hx.swap(hx.Beforeend),
        attribute.attribute("_", "
          on htmx:afterSwap
            set the value of #title to ''
            focus() the #title
          end
        "),
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
            attribute.attribute("aria-required", "true"),
            attribute.attribute("aria-invalid", case title_errors {
              [] -> "false"
              _ -> "true"
            }),
            attribute.attribute("aria-describedby", "form-errors"),
            attribute.class(case title_errors {
              [] -> "input"
              _ -> "input input-error"
            }),
          ]),
        ]),
        html.button(
          [
            attribute.type_("submit"),
            attribute.class("btn"),
          ],
          [element.text("Add Task")],
        ),
      ],
    ),
  ])
}

fn task_list_container(tasks: List(Task)) -> element.Element(t) {
  html.ul(
    [
      attribute.id("task-list"),
      attribute.class("task-list"),
      attribute.attribute("role", "list"),
      attribute.attribute("aria-label", "Task list"),
      attribute.attribute("_", "
        on htmx:beforeRequest from <* [hx-target='#task-list']/> or me
          add @aria-busy='true' to me
        end
        on htmx:afterRequest from <* [hx-target='#task-list']/> or me
          remove @aria-busy from me
        end
      "),
    ],
    list.map(tasks, fn(task) { task_item(task.id, task.title) }),
  )
}

// ── Pages ───────────────────────────────────────────────────────────────

fn home_page(_req: Request, ctx: Context) -> Response {
  let all_tasks = actor.call(ctx.tasks, 1000, task.GetTasks)

  let content =
    html.div([], [
      nav_bar(),
      // Section with heading for screen reader navigation
      html.section(
        [attribute.attribute("aria-labelledby", "add-task-heading")],
        [
          html.h2(
            [attribute.id("add-task-heading")],
            [element.text("Add a Task")],
          ),
          add_task_form([], []),
        ],
      ),
      // Task list section
      html.section(
        [attribute.attribute("aria-labelledby", "task-list-heading")],
        [
          html.h2(
            [attribute.id("task-list-heading")],
            [element.text("Tasks")],
          ),
          task_list_container(all_tasks),
        ],
      ),
      // Stats panel
      html.section(
        [attribute.attribute("aria-labelledby", "stats-heading")],
        [
          html.h2(
            [
              attribute.id("stats-heading"),
              attribute.class("sr-only"),
            ],
            [element.text("Task Statistics")],
          ),
          html.div(
            [
              attribute.id("task-counter"),
              attribute.class("stats-panel"),
              hx.get("/stats"),
              attribute("hx-trigger",
                "load, taskAdded from:body, taskDeleted from:body, every 30s [document.visibilityState === 'visible']"),
              hx.swap(hx.InnerHTML),
              attribute.attribute("aria-live", "polite"),
              attribute.attribute("aria-label", "Task statistics"),
            ],
            [element.text("Loading stats...")],
          ),
        ],
      ),
    ])

  layout(content)
}

// ── Handlers ────────────────────────────────────────────────────────────

fn create_task(req: Request, ctx: Context) -> Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      let id = int.to_string(int.random(100_000))
      let new_task = Task(id: id, title: valid_title, done: False)
      actor.send(ctx.tasks, task.AddTask(new_task))

      let task_html = task_item(id, valid_title)
      wisp.html_response(element.to_string(task_html), 201)
      |> wisp.set_header("HX-Trigger", "taskAdded")
    }

    Error(errors) -> {
      let form_html = add_task_form(errors, form_data.values)
      wisp.html_response(element.to_string(form_html), 422)
      |> wisp.set_header("HX-Trigger", "validationFailed")
    }
  }
}

fn delete_task(_req: Request, id: String, ctx: Context) -> Response {
  actor.send(ctx.tasks, task.DeleteTask(id))

  wisp.html_response("", 200)
  |> wisp.set_header("HX-Trigger", "taskDeleted")
}

fn update_task(req: Request, id: String, ctx: Context) -> Response {
  use form_data <- wisp.require_form(req)

  let title =
    list.key_find(form_data.values, "title")
    |> result.unwrap("")

  case validation.validate_task_title(title) {
    Ok(valid_title) -> {
      actor.send(ctx.tasks, task.UpdateTask(id, valid_title))

      let title_display =
        html.span(
          [
            attribute.id("title-display-" <> id),
            attribute.class("task-title editable"),
            attribute.attribute("tabindex", "0"),
            attribute.attribute("role", "button"),
            attribute.attribute("aria-label", "Edit task: " <> valid_title
              <> ". Press Enter to edit."),
            hx.get("/tasks/" <> id <> "/edit"),
            hx.target(hx.Selector("#title-display-" <> id)),
            hx.swap(hx.OuterHTML),
            hx.trigger([hx.click()]),
            attribute.attribute("_", "
              on keydown[key=='Enter' or key==' ']
                halt the event
                trigger click on me
              end
              on load focus() me end
            "),
          ],
          [element.text(valid_title)],
        )

      wisp.html_response(element.to_string(title_display), 200)
      |> wisp.set_header("HX-Trigger", "taskUpdated")
    }

    Error(_errors) -> {
      let edit_form = task_title_edit_form(id, title)
      wisp.html_response(element.to_string(edit_form), 422)
      |> wisp.set_header("HX-Trigger", "validationFailed")
    }
  }
}

fn get_task_edit_form(id: String, ctx: Context) -> Response {
  let tasks = actor.call(ctx.tasks, 1000, task.GetTasks)
  let task_opt =
    list.find(tasks, fn(t) { t.id == id })

  case task_opt {
    Ok(found_task) -> {
      let form_html = task_title_edit_form(found_task.id, found_task.title)
      wisp.html_response(element.to_string(form_html), 200)
    }
    Error(_) -> wisp.not_found()
  }
}

fn get_task_title_display(id: String, ctx: Context) -> Response {
  let tasks = actor.call(ctx.tasks, 1000, task.GetTasks)
  let task_opt =
    list.find(tasks, fn(t) { t.id == id })

  case task_opt {
    Ok(found_task) -> {
      let title_display =
        html.span(
          [
            attribute.id("title-display-" <> id),
            attribute.class("task-title editable"),
            attribute.attribute("tabindex", "0"),
            attribute.attribute("role", "button"),
            attribute.attribute("aria-label", "Edit task: " <> found_task.title
              <> ". Press Enter to edit."),
            hx.get("/tasks/" <> id <> "/edit"),
            hx.target(hx.Selector("#title-display-" <> id)),
            hx.swap(hx.OuterHTML),
            hx.trigger([hx.click()]),
            attribute.attribute("_", "
              on keydown[key=='Enter' or key==' ']
                halt the event
                trigger click on me
              end
              on load focus() me end
            "),
          ],
          [element.text(found_task.title)],
        )

      wisp.html_response(element.to_string(title_display), 200)
    }
    Error(_) -> wisp.not_found()
  }
}

fn get_stats(ctx: Context) -> Response {
  let tasks = actor.call(ctx.tasks, 1000, task.GetTasks)
  let total = list.length(tasks)
  let done =
    tasks
    |> list.filter(fn(t) { t.done })
    |> list.length

  let html =
    html.div([], [
      html.span([attribute.class("stat")], [
        element.text(int.to_string(total) <> " total"),
      ]),
      html.span(
        [attribute.class("stat-separator"), attribute.attribute("aria-hidden", "true")],
        [element.text(" | ")],
      ),
      html.span([attribute.class("stat")], [
        element.text(int.to_string(done) <> " done"),
      ]),
      html.span(
        [attribute.class("stat-separator"), attribute.attribute("aria-hidden", "true")],
        [element.text(" | ")],
      ),
      html.span([attribute.class("stat")], [
        element.text(int.to_string(total - done) <> " remaining"),
      ]),
    ])

  wisp.html_response(element.to_string(html), 200)
}

// ── Router ──────────────────────────────────────────────────────────────

pub fn handle_request(req: Request, ctx: Context) -> Response {
  use <- wisp.serve_static(req, under: "/static", from: "priv/static")

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)

    ["tasks"] ->
      case req.method {
        http.Get -> home_page(req, ctx)
        http.Post -> create_task(req, ctx)
        _ -> wisp.method_not_allowed([http.Get, http.Post])
      }

    ["tasks", id] ->
      case req.method {
        http.Put -> update_task(req, id, ctx)
        http.Delete -> delete_task(req, id, ctx)
        _ -> wisp.method_not_allowed([http.Put, http.Delete])
      }

    ["tasks", id, "edit"] -> {
      use <- wisp.require_method(req, http.Get)
      get_task_edit_form(id, ctx)
    }

    ["tasks", id, "title"] -> {
      use <- wisp.require_method(req, http.Get)
      get_task_title_display(id, ctx)
    }

    ["stats"] -> {
      use <- wisp.require_method(req, http.Get)
      get_stats(ctx)
    }

    _ -> wisp.not_found()
  }
}
```

### `priv/static/css/style.css` (accessibility additions)

```css
/* ─── Screen reader only ─── */

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* ─── Skip link ─── */

.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  padding: 0.5rem 1rem;
  background: #1a202c;
  color: #fff;
  z-index: 1000;
  text-decoration: none;
  font-weight: 600;
  border-radius: 0 0 4px 0;
}

.skip-link:focus {
  top: 0;
}

/* ─── Focus indicators ─── */

:focus-visible {
  outline: 3px solid #4a6fa5;
  outline-offset: 2px;
}

button:focus-visible,
a:focus-visible,
input:focus-visible,
[tabindex="0"]:focus-visible {
  outline: 3px solid #4a6fa5;
  outline-offset: 2px;
}

/* Never remove outlines entirely */
:focus:not(:focus-visible) {
  outline: none;
}

/* ─── Loading state (aria-busy driven) ─── */

[aria-busy="true"] {
  opacity: 0.6;
  pointer-events: none;
  position: relative;
}

[aria-busy="true"]::after {
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  width: 1.5rem;
  height: 1.5rem;
  margin: -0.75rem 0 0 -0.75rem;
  border: 3px solid #e2e8f0;
  border-top-color: #4a6fa5;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* ─── Editable task titles ─── */

.editable {
  cursor: pointer;
  padding: 0.125rem 0.25rem;
  border-radius: 2px;
  border: 1px solid transparent;
}

.editable:hover,
.editable:focus-visible {
  border-color: #4a6fa5;
  background: #f0f4f8;
}

/* ─── Inline edit form ─── */

.inline-edit-form {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
}

.inline-edit-form input {
  font-size: inherit;
  padding: 0.125rem 0.25rem;
  border: 1px solid #4a6fa5;
  border-radius: 2px;
}

/* ─── Button variants ─── */

.btn-secondary {
  background-color: #718096;
}

.btn-secondary:hover {
  background-color: #4a5568;
}

/* ─── High contrast mode support ─── */

@media (forced-colors: active) {
  .editable:hover,
  .editable:focus-visible {
    border-color: Highlight;
  }

  :focus-visible {
    outline-color: Highlight;
  }

  [aria-busy="true"]::after {
    border-color: ButtonText;
    border-top-color: Highlight;
  }
}

/* ─── Reduced motion ─── */

@media (prefers-reduced-motion: reduce) {
  [aria-busy="true"]::after {
    animation: none;
    border-top-color: #4a6fa5;
  }

  * {
    transition-duration: 0.01ms !important;
    animation-duration: 0.01ms !important;
  }
}
```

Notice two important media queries at the bottom:

- **`forced-colors: active`** handles Windows High Contrast Mode. When the user
  enables this mode, all your custom colours are overridden by the system palette.
  The `Highlight` and `ButtonText` keywords map to the user's chosen colours.

- **`prefers-reduced-motion: reduce`** handles users who have enabled "reduce
  motion" in their operating system settings. We disable the spinner animation
  and collapse all transitions to near-zero duration. Some users experience
  motion sickness or seizures from animations.

---

## 4. Exercises

### Exercise 1 -- Add Polite Announcements for Task Completion

**Description:** When a user marks a task as done (toggling the checkbox), the
live region should announce "Task marked as complete" or "Task marked as
incomplete" depending on the new state. Update the server handler to send
a custom `HX-Trigger` event with the appropriate message, and add the
_hyperscript listener to the body.

**Acceptance Criteria:**

- Toggling a task to "done" causes the `#live-region` to contain "Task marked as complete."
- Toggling a task to "not done" causes the `#live-region` to contain "Task marked as incomplete."
- The live region clears itself after 3 seconds.
- The announcement does not interrupt the screen reader (uses `polite`, not `assertive`).
- An axe scan reports zero violations after the change.

### Exercise 2 -- Roving Tabindex for the Task List

**Description:** Implement the roving tabindex pattern on the task list. Only
one task should have `tabindex="0"` at a time. Arrow Up and Arrow Down should
move focus between tasks. Tab should exit the list entirely.

**Acceptance Criteria:**

- The first task item has `tabindex="0"`. All other items have `tabindex="-1"`.
- Pressing Down Arrow moves focus to the next task. The old task gets `tabindex="-1"`, the new task gets `tabindex="0"`.
- Pressing Up Arrow moves focus to the previous task.
- Pressing Down Arrow on the last task wraps to the first task (or stays put -- your choice, but be consistent).
- Pressing Tab moves focus out of the task list to the next focusable element.
- After a task is added or deleted, the roving tabindex state is recalculated.

### Exercise 3 -- Accessible Modal Dialog

**Description:** Build a "Task Details" modal that opens when a user clicks a
task title while holding Shift (or clicks a dedicated "Details" button). The
modal should be fully accessible: focus trap, Escape to close, return focus
to the trigger element on close, and `role="dialog"`.

**Acceptance Criteria:**

- The modal container has `role="dialog"`, `aria-modal="true"`, and `aria-labelledby` pointing to the modal title.
- When the modal opens, focus moves to the first focusable element inside it (the close button).
- Tab cycles through focusable elements inside the modal without escaping to the page behind it.
- Pressing Escape closes the modal.
- When the modal closes, focus returns to the element that opened it.
- The modal backdrop is announced as non-interactive (`aria-hidden="true"` on the page content behind the modal).

### Exercise 4 -- Accessible Drag-and-Drop with Keyboard Alternative

**Description:** If your task board supports drag-and-drop reordering (from
Chapter 23 or a custom implementation), add a keyboard-accessible alternative.
Provide "Move Up" and "Move Down" buttons that appear when a task is focused,
allowing keyboard users to reorder tasks.

**Acceptance Criteria:**

- When a task item is focused, "Move Up" and "Move Down" buttons become visible (hidden for mouse users via CSS, visible on focus-within).
- Pressing "Move Up" sends a request to reorder the task and announces "Task moved up" via the live region.
- Pressing "Move Down" does the same in the opposite direction.
- Focus remains on the moved task after the reorder completes.
- The first task's "Move Up" button is disabled. The last task's "Move Down" button is disabled.
- The live region announces the new position: "Task moved to position 3 of 7."

### Exercise 5 -- Automated Accessibility Testing in CI

**Description:** Add pa11y (an automated accessibility testing tool) to your
project's test suite. Write a test script that starts the Gleam server, runs
pa11y against the home page, and fails the CI build if any violations are found.

**Acceptance Criteria:**

- A `test_accessibility.sh` script (or equivalent) starts the server, waits for it to be ready, and runs `pa11y http://localhost:8080`.
- pa11y runs with WCAG 2.1 AA standard.
- The script exits with code 0 if no violations are found, and non-zero if violations exist.
- The script cleans up (stops the server) regardless of whether the test passes or fails.
- At least one test run passes with zero violations on the accessible version of the page.

---

## 5. Exercise Solution Hints

Try each exercise on your own before reading these hints.

### Hint for Exercise 1

You need two custom events. Use `HX-Trigger` with JSON to pass data:

```gleam
// When marking complete:
wisp.set_header("HX-Trigger", "{\"taskCompleted\": \"complete\"}")

// When marking incomplete:
wisp.set_header("HX-Trigger", "{\"taskCompleted\": \"incomplete\"}")
```

Then in _hyperscript, listen for the event and check the detail:

```
on taskCompleted(status) from body
  if status is 'complete'
    put 'Task marked as complete' into #live-region
  else
    put 'Task marked as incomplete' into #live-region
  end
  wait 3s
  put '' into #live-region
end
```

Alternatively, use two separate events (`taskMarkedDone` and
`taskMarkedUndone`) if you find the JSON approach complex.

### Hint for Exercise 2

Roving tabindex needs a container-level _hyperscript that manages focus. The
container listens for arrow key events and moves tabindex between children:

```gleam
attribute.attribute("_", "
  on keydown[key=='ArrowDown'] from <li/> in me
    halt the event
    set current to the target's closest <li/>
    set next to the next <li/> from current
    if next is null
      set next to the first <li/> in me
    end
    remove @tabindex from current
    set current's tabindex to '-1'
    set next's tabindex to '0'
    focus() next
  end
  on keydown[key=='ArrowUp'] from <li/> in me
    halt the event
    set current to the target's closest <li/>
    set prev to the previous <li/> from current
    if prev is null
      set prev to the last <li/> in me
    end
    remove @tabindex from current
    set current's tabindex to '-1'
    set prev's tabindex to '0'
    focus() prev
  end
"),
```

Make sure the first `<li>` renders with `tabindex="0"` and all others with
`tabindex="-1"`. After a task is added or deleted, use an `htmx:afterSwap`
listener to reset the tabindex state.

### Hint for Exercise 3

Focus trapping is the hardest part. The _hyperscript approach:

```
on keydown[key=='Tab'] from me
  set focusables to [<a/>, <button/>, <input/>, <select/>, <textarea/>] in me
  set first to the first in focusables
  set last to the last in focusables
  if event.shiftKey and document.activeElement is first
    halt the event
    focus() last
  else if not event.shiftKey and document.activeElement is last
    halt the event
    focus() first
  end
end
```

Store a reference to the trigger element before opening the modal:

```
on click
  set window.modalTrigger to me
  -- fetch and show modal --
end
```

On close, restore focus:

```
on closeModal
  focus() window.modalTrigger
end
```

### Hint for Exercise 4

The "Move Up" and "Move Down" buttons are hidden by default and shown with
CSS `li:focus-within .move-buttons { display: flex; }`. Each button sends a
request to reorder the task:

```gleam
html.button(
  [
    attribute.class("btn btn-small move-btn"),
    attribute.attribute("aria-label", "Move task up"),
    hx.post("/tasks/" <> id <> "/move-up"),
    hx.target(hx.Selector("#task-list")),
    hx.swap(hx.InnerHTML),
  ],
  [element.text("Move Up")],
)
```

For the position announcement, the server can include the new position in the
trigger:

```gleam
wisp.set_header("HX-Trigger",
  "{\"taskMoved\": \"Task moved to position " <> int.to_string(new_pos)
  <> " of " <> int.to_string(total) <> "\"}")
```

### Hint for Exercise 5

A minimal `test_accessibility.sh` script:

```bash
#!/bin/bash
set -e

# Start the server in the background
gleam run &
SERVER_PID=$!

# Wait for server to start
sleep 3

# Run pa11y
npx pa11y http://localhost:8080 --standard WCAG2AA

# Capture exit code
EXIT_CODE=$?

# Cleanup
kill $SERVER_PID

exit $EXIT_CODE
```

Install pa11y with `npm install -g pa11y`. For CI, add it to your pipeline's
test step. If you want HTML output for review, add `--reporter html > a11y-report.html`.

---

## 6. Key Takeaways

1. **Dynamic DOM updates are invisible to screen readers by default.** HTMX
   swaps content visually, but screen readers only announce changes to elements
   marked with `aria-live`. Without explicit live regions, your dynamic updates
   are silent.

2. **Use two live regions: polite and assertive.** Put routine announcements
   (task added, stats updated) in an `aria-live="polite"` region. Put errors
   and urgent messages in an `aria-live="assertive"` region. Clear both after
   a few seconds so the next announcement is detected as new content.

3. **Focus management prevents keyboard users from getting lost.** After adding
   content, return focus to the input. After deleting content, move focus to the
   next sibling. After opening a modal, move focus inside. After closing a
   modal, return focus to the trigger. Every swap should have a focus strategy.

4. **Use `aria-busy="true"` as both a semantic signal and a CSS hook.** Set it
   on the swap target before the HTMX request and remove it after. The CSS
   selector `[aria-busy="true"]` handles the visual loading state. One
   attribute serves two purposes -- accessibility and styling.

5. **Semantic HTML is your first line of defense.** Use `<button>` for actions,
   `<a>` for navigation, `<label>` for form inputs, `<main>` for content, and
   `<nav>` for navigation. These elements are accessible by default. Every
   `<div>` with a click handler is a bug waiting to happen.

6. **Keyboard navigation must work without a mouse.** Tab to reach, Enter/Space
   to activate, Escape to dismiss, arrow keys to navigate within components.
   Test by unplugging your mouse and completing every user flow.

7. **`aria-label` provides context that visual layout cannot.** A button that
   says "Delete" visually is ambiguous to a screen reader. "Delete task: Buy
   groceries" is unambiguous. Always label interactive elements with enough
   context to stand alone.

8. **Test with axe for automated checks, then keyboard, then a screen reader.**
   Automated tools catch about 30-40% of issues (the easy ones). Keyboard
   testing catches focus order and interactivity problems. Screen reader testing
   catches announcement and timing issues. All three levels are necessary.

9. **Accessibility is not a feature -- it is a quality attribute.** You do not
   "add accessibility" at the end. You build it into every component from the
   start. The `.sr-only` class, the live regions, the `aria-label` attributes,
   the focus management -- these are as fundamental as your CSS and your routing.
   If your application does not work for everyone, it does not work.

---

## What's Next

The Teamwork task board is now usable by everyone: sighted users, screen reader
users, keyboard-only users, and users with reduced motion preferences. The live
regions announce every dynamic change, focus management guides the keyboard
through every interaction, and `aria-busy` bridges the gap between visual
loading states and assistive technology.

In **Chapter 27 -- Database Performance**, we will tackle the backend side of
real-world applications. We will replace the in-memory actor with a database,
add pagination at the SQL level, implement connection pooling, and profile
query performance -- all while keeping the HTMX frontend responsive and the
accessibility features intact.
