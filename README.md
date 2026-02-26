# Teamwork: An HTMX Course with Gleam

A 28-chapter course teaching HTMX through building a collaborative task board called **Teamwork**, using Gleam as the backend language.

## About

This course takes you from HTTP fundamentals to production-ready HTMX patterns. Each chapter builds incrementally on the previous one, constructing a full-featured task board with real-time updates, authentication, inline editing, file uploads, and more.

**Who is this for?** Developers who want to learn HTMX with a typed, functional backend. Familiarity with HTML and basic web concepts is assumed; no prior Gleam experience is required.

**What gets built?** A collaborative task board supporting multiple boards, real-time SSE updates, drag-and-drop, modal dialogs, search/filter, user authentication, and background jobs.

## Tech Stack

| Technology | Role |
|---|---|
| [HTMX](https://htmx.org) 2.0.8 | Frontend interactivity via HTML attributes |
| [Gleam](https://gleam.run) | Backend language (runs on the BEAM VM) |
| [Wisp](https://hexdocs.pm/wisp/) | Web framework |
| [Mist](https://hexdocs.pm/mist/) | HTTP server |
| [Lustre](https://hexdocs.pm/lustre/) | Type-safe HTML DSL |
| [hx](https://hexdocs.pm/hx/) | Typed HTMX attributes for Lustre |
| [_hyperscript](https://hyperscript.org) | Companion scripting language |
| SQLite ([sqlight](https://hexdocs.pm/sqlight/)) | Database |
| OTP actors | Concurrency / background jobs |

## Prerequisites

- **Erlang/OTP 27** and **Gleam >= 1.0** (install via [mise](05-appendices/A3-installing-mise.md))
- Basic knowledge of HTML and how the web works (Chapter 1 covers the fundamentals)

## Table of Contents

### Part 1 -- Beginner (Chapters 1--6)

HTTP basics, Mist/Wisp setup, Lustre HTML, first HTMX interactions, server state.

| # | Chapter |
|---|---|
| 1 | [How the Web Actually Works](01-beginner/01-how-the-web-works.md) |
| 2 | [Serving HTML Pages](01-beginner/02-serving-html-pages.md) |
| 3 | [Routing: Multiple Pages](01-beginner/03-routing.md) |
| 4 | [Your First HTMX Interaction](01-beginner/04-first-htmx-interaction.md) |
| 5 | [Click, Trigger, Swap: The HTMX Triad](01-beginner/05-click-trigger-swap.md) |
| 6 | [Server State: Data That Lives Between Requests](01-beginner/06-server-state.md) |

### Part 2 -- Intermediate (Chapters 7--12)

Forms, validation, swap strategies, search/filters, navigation, authentication.

| # | Chapter |
|---|---|
| 7 | [Forms and User Input](02-intermediate/07-forms-and-user-input.md) |
| 8 | [Validation and Error Feedback](02-intermediate/08-validation-and-error-feedback.md) |
| 9 | [Swap Strategies and Targets: Out-of-Band Updates](02-intermediate/09-swap-strategies-and-targets.md) |
| 10 | [Lists, Filters, and Search](02-intermediate/10-lists-filters-search.md) |
| 11 | [Multiple Boards and Navigation](02-intermediate/11-multiple-boards-navigation.md) |
| 12 | [Cookies, Sessions, and Authentication](02-intermediate/12-cookies-sessions-auth.md) |

### Part 3 -- Advanced (Chapters 13--18)

SSE, security, testing, extensions, deployment, third-party JS.

| # | Chapter |
|---|---|
| 13 | [Real-Time with Server-Sent Events](03-advanced/13-real-time-with-sse.md) |
| 14 | [Security Hardening](03-advanced/14-security-hardening.md) |
| 15 | [Testing HTMX Applications](03-advanced/15-testing.md) |
| 16 | [Extensions and Advanced Patterns](03-advanced/16-extensions-and-patterns.md) |
| 17 | [Deployment and Production](03-advanced/17-deployment-and-production.md) |
| 18 | [Integrating Third-Party JavaScript: Flatpickr](03-advanced/18-flatpickr-and-third-party-js.md) |

### Part 4 -- Real-World Patterns (Chapters 19--28)

Error handling, response headers, inline editing, modals, _hyperscript, dynamic forms, file uploads, accessibility, database performance, background jobs.

| # | Chapter |
|---|---|
| 19 | [Error Handling and Graceful Degradation](04-real-world-patterns/19-error-handling-and-graceful-degradation.md) |
| 20 | [HTMX Response Headers and Server-Driven Control](04-real-world-patterns/20-response-headers-and-server-control.md) |
| 21 | [Inline Editing (Click-to-Edit)](04-real-world-patterns/21-inline-editing.md) |
| 22 | [Modal Dialogs via HTMX](04-real-world-patterns/22-modal-dialogs.md) |
| 23 | [_hyperscript Practical Patterns](04-real-world-patterns/23-hyperscript-practical-patterns.md) |
| 24 | [Dynamic and Dependent Forms](04-real-world-patterns/24-dynamic-and-dependent-forms.md) |
| 25 | [File Uploads](04-real-world-patterns/25-file-uploads.md) |
| 26 | [Accessible HTMX](04-real-world-patterns/26-accessible-htmx.md) |
| 27 | [Database Performance and Queries](04-real-world-patterns/27-database-performance.md) |
| 28 | [Background Jobs with BEAM Processes](04-real-world-patterns/28-background-jobs.md) |

### Appendices

| | Appendix |
|---|---|
| A | [Bibliography and Resources](05-appendices/A1-bibliography-and-resources.md) |
| B | [_hyperscript: HTMX's Companion Language](05-appendices/A2-hyperscript.md) |
| C | [Installing mise](05-appendices/A3-installing-mise.md) |

## A Taste of the Code

The course teaches type-safe HTMX patterns using Gleam. Here's what routing looks like with Wisp:

```gleam
case wisp.path_segments(req) {
  [] -> handle_home(req, ctx)
  ["tasks"] -> handle_tasks(req, ctx)
  ["tasks", id] -> handle_task(req, ctx, id)
  _ -> wisp.not_found()
}
```

And here's type-safe HTML with HTMX attributes using Lustre + hx:

```gleam
html.button(
  [hx.post("/tasks"), hx.target(hx.Selector("#task-list")), hx.swap(hx.InnerHTML)],
  [html.text("Add Task")],
)
```

## Chapter Format

Each chapter follows a consistent structure:

1. **Learning objectives** -- what you'll know by the end
2. **Theory** -- concepts and background
3. **Code walkthrough** -- step-by-step implementation
4. **Full code listing** -- complete, copy-pasteable code
5. **Exercises** -- practice problems with solution hints
6. **Key takeaways** -- summary of main points
7. **What's next** -- preview of the following chapter

## How to Read This Course

This is a **content-only repository** -- it contains markdown files with embedded code blocks, not a runnable application. To follow along, create your own Gleam project and implement each chapter's code as you read.

Chapters are sequential and cross-reference each other with relative links. All "Chapter X" and "Appendix X" mentions in the prose are clickable links.

## Toolchain Setup

To set up your development environment, see [Appendix C -- Installing mise](05-appendices/A3-installing-mise.md), which covers installing:

- **[mise](https://mise.jdx.dev/)** -- polyglot toolchain manager
- **Erlang/OTP 27** -- the BEAM runtime
- **Gleam** -- the language compiler and build tool

## Documentation Links

| Library | Documentation |
|---|---|
| HTMX | [htmx.org](https://htmx.org) |
| Gleam | [gleam.run](https://gleam.run) |
| Wisp | [hexdocs.pm/wisp](https://hexdocs.pm/wisp/) |
| Mist | [hexdocs.pm/mist](https://hexdocs.pm/mist/) |
| Lustre | [hexdocs.pm/lustre](https://hexdocs.pm/lustre/) |
| hx | [hexdocs.pm/hx](https://hexdocs.pm/hx/) |
| _hyperscript | [hyperscript.org](https://hyperscript.org) |
