# Appendix A -- Bibliography and Resources

This appendix collects the official documentation, homepages, repositories, and
video channels for every tool and technology used in this course. Bookmark the
ones you use most -- they are the best place to go when you need answers beyond
what this course covers.

---

## 1. Core Stack

### HTMX

| Resource | Link |
|----------|------|
| Homepage | https://htmx.org |
| Documentation | https://htmx.org/docs/ |
| Reference (attributes) | https://htmx.org/reference/ |
| Extensions directory | https://extensions.htmx.org |
| GitHub | https://github.com/bigskysoftware/htmx |
| Examples | https://htmx.org/examples/ |
| Essays | https://htmx.org/essays/ |
| Discord | https://htmx.org/discord |
| YouTube (talks & demos) | https://www.youtube.com/@htmx |

HTMX is the foundation of this course. The **docs** page is the single best
reference for attributes, triggers, and swap strategies. The **essays** section
contains articles by the author on hypermedia architecture and why HTMX exists.
The **examples** page has copy-pasteable patterns for common interactions.

### Gleam

| Resource | Link |
|----------|------|
| Homepage | https://gleam.run |
| Language tour | https://tour.gleam.run |
| Documentation | https://gleam.run/documentation/ |
| Standard library docs | https://hexdocs.pm/gleam_stdlib/ |
| Package repository | https://packages.gleam.run |
| GitHub | https://github.com/gleam-lang/gleam |
| Discord | https://discord.gg/Fm8Pwmy |
| YouTube | https://www.youtube.com/@gleamlang |

The **language tour** is an interactive tutorial that runs in the browser -- the
fastest way to learn Gleam syntax. The **package repository** is where you find
all Gleam libraries (the equivalent of npm or crates.io).

### Erlang / OTP / BEAM

| Resource | Link |
|----------|------|
| Erlang homepage | https://www.erlang.org |
| Erlang documentation | https://www.erlang.org/docs |
| OTP design principles | https://www.erlang.org/doc/system/design_principles |
| Erlang Solutions YouTube | https://www.youtube.com/@ErlangSolutions |
| "Learn You Some Erlang" (free book) | https://learnyousomeerlang.com |

Gleam compiles to Erlang and runs on the BEAM virtual machine. You do not need
to learn Erlang to use Gleam, but understanding OTP concepts (actors, supervisors,
the "let it crash" philosophy) helps you understand why Gleam's concurrency model
works the way it does. The **OTP design principles** page is the authoritative
source.

---

## 2. Gleam Web Libraries

### Wisp (Web Framework)

| Resource | Link |
|----------|------|
| Hex docs | https://hexdocs.pm/wisp/ |
| GitHub | https://github.com/lpil/wisp |

Wisp provides routing, request/response handling, form parsing, cookie management,
static file serving, and middleware. It is the web framework used throughout every
chapter of this course.

### Mist (HTTP Server)

| Resource | Link |
|----------|------|
| Hex docs | https://hexdocs.pm/mist/ |
| GitHub | https://github.com/rawhat/mist |

Mist is the HTTP server that runs underneath Wisp. It handles TCP connections,
HTTP parsing, and the server lifecycle. In this course we use the `wisp_mist`
integration package to connect them.

### Lustre (HTML DSL)

| Resource | Link |
|----------|------|
| Hex docs | https://hexdocs.pm/lustre/ |
| GitHub | https://github.com/lustre-labs/lustre |

Lustre is primarily a client-side framework (similar to Elm), but in this course
we use its **server-side rendering** capabilities -- specifically its HTML element
builders and `element.to_document_string()` function. It gives us type-safe HTML
generation in Gleam.

### hx (HTMX Attribute Bindings)

| Resource | Link |
|----------|------|
| Hex docs | https://hexdocs.pm/hx/ |
| GitHub | https://github.com/brettkolodny/hx |

The `hx` library provides type-safe Gleam functions for HTMX attributes. Instead
of writing `attribute("hx-get", "/tasks")`, you write `hx.get("/tasks")`. It
catches typos at compile time and provides documentation through your editor's
autocomplete.

### sqlight (SQLite Bindings)

| Resource | Link |
|----------|------|
| Hex docs | https://hexdocs.pm/sqlight/ |
| GitHub | https://github.com/lpil/sqlight |

Gleam bindings for SQLite. Used in the deployment chapter for persistent storage.

### gleeunit (Test Framework)

| Resource | Link |
|----------|------|
| Hex docs | https://hexdocs.pm/gleeunit/ |
| GitHub | https://github.com/lpil/gleeunit |

The standard test framework for Gleam projects. Runs on the Erlang test
infrastructure and integrates with `gleam test`.

### gleam_erlang / gleam_otp

| Resource | Link |
|----------|------|
| gleam_erlang docs | https://hexdocs.pm/gleam_erlang/ |
| gleam_otp docs | https://hexdocs.pm/gleam_otp/ |

These packages provide Gleam interfaces to Erlang/OTP primitives: processes,
actors, supervisors, and the runtime system. The actor model used in Chapter 6
onwards comes from `gleam_otp`.

---

## 3. HTMX Extensions

All official HTMX extensions are documented at https://extensions.htmx.org.

### response-targets

| Resource | Link |
|----------|------|
| Documentation | https://extensions.htmx.org/ext/response-targets/ |
| npm / unpkg | https://unpkg.com/htmx-ext-response-targets/ |

Swap different content into different targets based on HTTP status codes. Used in
Chapter 16 for differentiated error handling.

### preload

| Resource | Link |
|----------|------|
| Documentation | https://extensions.htmx.org/ext/preload/ |
| npm / unpkg | https://unpkg.com/htmx-ext-preload/ |

Prefetch linked content on hover or mousedown so navigation feels instant. Used
in Chapter 16 for instant navigation.

### SSE (Server-Sent Events)

| Resource | Link |
|----------|------|
| Documentation | https://extensions.htmx.org/ext/sse/ |
| npm / unpkg | https://unpkg.com/htmx-ext-sse/ |

Connect to an SSE endpoint and swap incoming messages into the DOM. Used in
Chapter 13 for real-time updates.

### Other extensions referenced

| Extension | Documentation |
|-----------|--------------|
| loading-states | https://extensions.htmx.org/ext/loading-states/ |
| multi-swap | https://extensions.htmx.org/ext/multi-swap/ |
| path-deps | https://extensions.htmx.org/ext/path-deps/ |
| restored | https://extensions.htmx.org/ext/restored/ |
| head-support | https://extensions.htmx.org/ext/head-support/ |

---

## 4. \_hyperscript

| Resource | Link |
|----------|------|
| Homepage | https://hyperscript.org |
| Language reference | https://hyperscript.org/reference/ |
| Cookbook | https://hyperscript.org/cookbook/ |
| GitHub | https://github.com/bigskysoftware/_hyperscript |

\_hyperscript is the companion client-side scripting language introduced in
[Chapter 16](../03-advanced/16-extensions-and-patterns.md) and covered in depth in [Appendix B](A2-hyperscript.md).

---

## 5. Third-Party JavaScript Libraries

### Flatpickr (Date/Time Picker)

| Resource | Link |
|----------|------|
| Homepage | https://flatpickr.js.org |
| Documentation | https://flatpickr.js.org/options/ |
| GitHub | https://github.com/flatpickr/flatpickr |
| CDN | https://cdn.jsdelivr.net/npm/flatpickr |
| Themes | https://flatpickr.js.org/themes/ |
| Localization | https://flatpickr.js.org/localization/ |

Flatpickr is a lightweight (~16 KB), zero-dependency date and time picker. It
provides a consistent calendar widget across all browsers, replacing the
inconsistent native `<input type="date">`. Its no-build-step philosophy makes it
a natural companion for HTMX projects. When using Flatpickr with HTMX, remember
to re-initialize pickers after swaps by listening for the `htmx:load` event.

---

## 6. Databases

### SQLite

| Resource | Link |
|----------|------|
| Homepage | https://sqlite.org |
| Documentation | https://sqlite.org/docs.html |
| SQL syntax reference | https://sqlite.org/lang.html |

SQLite is the database used in the deployment chapter. It is a file-based
database that requires no separate server process -- ideal for small to
medium applications.

### PostgreSQL (mentioned as alternative)

| Resource | Link |
|----------|------|
| Homepage | https://www.postgresql.org |
| Documentation | https://www.postgresql.org/docs/ |
| YouTube | https://www.youtube.com/@PostgresTV |

---

## 7. Deployment & Infrastructure

### mise (Tool Version Manager)

| Resource | Link |
|----------|------|
| Homepage | https://mise.jdx.dev |
| Installation guide | https://mise.jdx.dev/installing-mise.html |
| Getting started | https://mise.jdx.dev/getting-started.html |
| Configuration reference | https://mise.jdx.dev/configuration.html |
| Tool registry | https://mise.jdx.dev/registry.html |
| GitHub | https://github.com/jdx/mise |

mise is used throughout this course to install and manage Gleam and Erlang/OTP
versions. The project root contains a `mise.toml` that pins compatible versions.
See **[Appendix C](A3-installing-mise.md)** for detailed installation instructions.

### Docker

| Resource | Link |
|----------|------|
| Homepage | https://www.docker.com |
| Documentation | https://docs.docker.com |
| Dockerfile reference | https://docs.docker.com/reference/dockerfile/ |
| YouTube | https://www.youtube.com/@DockerInc |

Docker is used in [Chapter 17](../03-advanced/17-deployment-and-production.md) for containerised deployment with multi-stage
builds.

---

## 8. Web Standards & Browser APIs

These are not libraries you install -- they are web platform features referenced
throughout the course.

| Technology | Specification / Reference |
|------------|--------------------------|
| HTTP/1.1 | https://httpwg.org/specs/rfc9110.html |
| HTML Living Standard | https://html.spec.whatwg.org |
| Server-Sent Events | https://html.spec.whatwg.org/multipage/server-sent-events.html |
| Content-Security-Policy | https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP |
| Subresource Integrity | https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity |
| Page Visibility API | https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API |
| EventSource API | https://developer.mozilla.org/en-US/docs/Web/API/EventSource |
| CSS Transitions | https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_transitions |

---

## 9. Books

### "Hypermedia Systems"

| | |
|---|---|
| Authors | Carson Gross, Adam Stepinski, Deniz Aksimsek |
| Read online (free) | https://hypermedia.systems |
| Publisher | Self-published, 2023 |

The definitive book on the hypermedia architecture that HTMX embodies. Covers the
philosophy, history, and practical application of hypermedia-driven applications.
Written by the creator of HTMX. If you read one book after finishing this course,
make it this one.

### "Learn You Some Erlang for Great Good!"

| | |
|---|---|
| Author | Fred Hébert |
| Read online (free) | https://learnyousomeerlang.com |

A friendly, illustrated guide to Erlang and OTP. Useful for understanding the
runtime that powers Gleam -- actors, supervisors, fault tolerance, and the BEAM
virtual machine.

---

## 10. Community & Support

| Community | Link |
|-----------|------|
| HTMX Discord | https://htmx.org/discord |
| Gleam Discord | https://discord.gg/Fm8Pwmy |
| Gleam discussions (GitHub) | https://github.com/gleam-lang/gleam/discussions |
| Erlang Forums | https://erlangforums.com |
| HTMX subreddit | https://www.reddit.com/r/htmx/ |
| Gleam subreddit | https://www.reddit.com/r/gleamlang/ |

---

## 11. Accessibility Tools

These tools are referenced in **[Chapter 26 -- Accessible HTMX](../04-real-world-patterns/26-accessible-htmx.md)** for testing
and auditing the accessibility of HTMX-powered applications.

| Tool | Link | Description |
|------|------|-------------|
| axe DevTools | https://www.deque.com/axe/devtools/ | Browser extension for automated accessibility audits. Identifies WCAG violations and suggests fixes. |
| NVDA | https://www.nvaccess.org/download/ | Free, open-source screen reader for Windows. Use it to verify ARIA live regions and focus management. |
| VoiceOver | Built into macOS/iOS | Apple's screen reader. Activate with Cmd+F5 on macOS. |
| pa11y | https://pa11y.org | Command-line accessibility testing tool. Integrates into CI pipelines. |
| WAVE | https://wave.webaim.org | Web accessibility evaluation tool with browser extension and API. |
| Lighthouse | Built into Chrome DevTools | Google's automated auditing tool. Includes an accessibility score. |

---

## 12. SQLite Advanced Features

These resources support **[Chapter 27 -- Database Performance and Queries](../04-real-world-patterns/27-database-performance.md)**.

| Resource | Link |
|----------|------|
| FTS5 (Full-Text Search) | https://www.sqlite.org/fts5.html |
| EXPLAIN QUERY PLAN | https://www.sqlite.org/eqp.html |
| WAL Mode | https://www.sqlite.org/wal.html |
| SQLite Query Planner | https://www.sqlite.org/queryplanner.html |
| sqlight (Gleam bindings) | https://hexdocs.pm/sqlight/ |

---

## 13. Quick Reference Card

The tools in this course and their roles at a glance:

| Layer | Tool | Role |
|-------|------|------|
| Language | **Gleam** | Application code |
| Runtime | **BEAM** (Erlang VM) | Execution, concurrency, fault tolerance |
| HTTP server | **Mist** | Accepts connections, parses HTTP |
| Web framework | **Wisp** | Routing, middleware, request/response helpers |
| HTML generation | **Lustre** | Type-safe HTML element builders |
| HTMX attributes | **hx** | Type-safe `hx-*` attribute functions |
| Interactivity | **HTMX** | Server-driven HTML swaps via AJAX |
| Client scripting | **\_hyperscript** | Declarative DOM manipulation |
| Database | **SQLite** + sqlight | Persistent storage |
| Testing | **gleeunit** | Unit and integration tests |
| Deployment | **Docker** | Containerised builds and releases |
