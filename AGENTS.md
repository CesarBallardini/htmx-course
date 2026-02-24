# AGENTS.md

This file provides guidance to AI coding agents working with this repository.

## Project Overview

**"Teamwork"** — a 28-chapter HTMX course that builds a collaborative task board. The course uses Gleam as the backend language; chapters are markdown files with embedded code blocks (no separate runnable source code repo).

**Tech stack taught in the course:**
- **Backend:** Gleam + Wisp (web framework) + Mist (HTTP server)
- **HTML generation:** Lustre (type-safe HTML DSL) + `hx` library (typed HTMX attributes for Lustre)
- **Interactivity:** HTMX 2.0.8, _hyperscript (companion scripting language)
- **Database:** SQLite via `sqlight`
- **Concurrency:** BEAM VM (Erlang/OTP runtime), OTP actors
- **Testing:** gleeunit, `wisp/simulate` for integration tests

## Repository Structure

```
01-beginner/              Chapters 1–6: HTTP basics, Mist/Wisp setup, Lustre HTML, first HTMX interactions, server state
02-intermediate/          Chapters 7–12: Forms, validation, swap strategies, search/filters, navigation, auth/sessions
03-advanced/              Chapters 13–18: SSE, security, testing, extensions, deployment, third-party JS (Flatpickr)
04-real-world-patterns/   Chapters 19–28: Error handling, response headers, inline editing, modals, _hyperscript, dynamic forms, file uploads, accessibility, DB performance, background jobs
05-appendices/            A1 (bibliography & resources), A2 (_hyperscript reference), A3 (installing mise)
```

Chapters are sequential and cross-reference each other. Each chapter follows this structure: learning objectives → theory → code walkthrough → full code listing → exercises → solution hints → key takeaways → what's next.

## Writing Conventions

- All code examples use Gleam with these libraries at these version ranges:
  - `wisp >= 2.0.0 < 3.0.0`, `mist >= 5.0.0 < 6.0.0`, `lustre >= 4.0.0 < 6.0.0`, `hx >= 3.0.0 < 4.0.0`
  - `gleam_stdlib >= 0.50.0`, `gleam_http >= 4.0.0`, `gleam_erlang >= 1.0.0`, `gleam_otp >= 1.0.0`
- The `hx` library provides typed HTMX attributes: `hx.get()`, `hx.target(hx.Selector("#id"))`, `hx.swap(hx.InnerHTML)`, `hx.trigger([...])`, `hx.push_url(True)`, etc.
- Lustre HTML uses `html.div(attrs, children)` pattern; render with `element.to_document_string()` (full page) or `element.to_string()` (fragment).
- Routing uses Wisp's `wisp.path_segments(req)` pattern-matching style.
- Form data accessed via `use form_data <- wisp.require_form(req)` then `list.key_find(form_data.values, "field_name")`.
- State management uses BEAM actors: `actor.new(state) |> actor.on_message(handler) |> actor.start`.

## API Correctness

All 28 chapters have been verified against a 14-pattern grep sweep for API correctness. When editing chapters, maintain correct API usage for Wisp, Lustre, hx, and sqlight. Key pitfalls to avoid:
- `element.to_document_string` (full HTML page with doctype) vs `element.to_string` (HTML fragment for HTMX partial responses)
- `hx.Selector("#id")` requires the `#` prefix — it is a CSS selector, not a bare ID
- `hx.trigger()` takes a list of trigger specs, not a single string
- Wisp's `wisp.require_form(req)` uses Gleam's `use` syntax for callback-based control flow

## Key Documentation References

| Library | Docs |
|---------|------|
| HTMX | https://htmx.org |
| Gleam | https://gleam.run |
| Wisp | https://hexdocs.pm/wisp/ |
| Mist | https://hexdocs.pm/mist/ |
| Lustre | https://hexdocs.pm/lustre/ |
| hx | https://hexdocs.pm/hx/ |
| _hyperscript | https://hyperscript.org |
