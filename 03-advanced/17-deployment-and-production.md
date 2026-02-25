# Chapter 17 -- Deployment and Production

**Phase:** Advanced (final chapter)
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapters 1 through 16 built a complete task board application with routing, HTMX interactions, forms, validation, search, inline editing, authentication, real-time updates, and more. Now it is time to deploy.

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why in-memory state is insufficient for production and migrate to SQLite using the `sqlight` package.
- Write database migration SQL that creates tables with proper constraints and defaults.
- Read environment variables at startup to configure ports, secrets, and database paths.
- Build a self-contained BEAM release with `gleam export erlang-shipment`.
- Write a multi-stage Dockerfile that produces a small, production-ready container image.
- Choose between CDN-hosted and self-hosted HTMX and explain the trade-offs.
- Set caching headers for static assets to improve performance.
- Describe a production checklist covering HTTPS, logging, backups, and monitoring.

---

## 1. Theory

### 1.1 Why In-Memory State Is Not Enough

Throughout this course, we stored tasks, boards, and user data inside Gleam actors -- OTP processes holding state in memory. This approach is excellent for learning and prototyping. The data is fast to read and write, and you do not need to install or configure anything.

But there is a fatal flaw: **when the server restarts, everything disappears.**

Restart scenarios are not rare. They happen during deployments, after crashes, when the host machine reboots, and when your cloud provider reschedules your container. In production, data loss on restart is not acceptable. You need persistence.

### 1.2 SQLite -- The Right Tool for Many Jobs

SQLite is a file-based relational database. Unlike PostgreSQL or MySQL, it does not run as a separate server process. The database is a single file on disk, and your application reads and writes to it directly through a library.

This makes SQLite uniquely well-suited for small-to-medium applications:

| Property                  | SQLite                                   | PostgreSQL / MySQL                    |
|---------------------------|------------------------------------------|---------------------------------------|
| **Setup**                 | Zero -- just a file path                 | Install, configure, run a server      |
| **Deployment**            | Ships with your app                      | Separate service to manage            |
| **Backups**               | Copy the file                            | `pg_dump` or replication              |
| **Concurrent reads**      | Excellent (WAL mode)                     | Excellent                             |
| **Concurrent writes**     | One writer at a time                     | Multiple concurrent writers           |
| **Scaling**               | Single server                            | Horizontal replication possible       |

For the Teamwork application -- a task board used by a team -- SQLite is more than sufficient. If your application ever outgrows it, migrating to PostgreSQL is straightforward because the SQL syntax is nearly identical.

In Gleam, we use the `sqlight` package to interact with SQLite.

### 1.3 BEAM Releases

During development, you run your application with `gleam run`. This compiles your Gleam code to Erlang, then starts the BEAM virtual machine to execute it. But `gleam run` requires the Gleam compiler, the Erlang compiler, and all your source code to be present on the machine.

For production, you want a **release** -- a self-contained bundle that includes:

- Your compiled application code (Erlang `.beam` files).
- All dependency code.
- The BEAM virtual machine itself.
- A startup script.

Gleam provides this with a single command:

```
gleam export erlang-shipment
```

The output lands in `build/erlang-shipment/`. You can copy this directory to any compatible machine (same OS and architecture) and run it with `./entrypoint.sh run`. No Gleam compiler needed. No Erlang installation needed. Everything is self-contained.

This is one of the BEAM's greatest operational strengths. The Erlang runtime has been designed from the ground up for deployment and long-running production systems.

### 1.4 Docker -- Consistent, Reproducible Deployment

Docker containers wrap your application and its runtime into an isolated, reproducible unit. The key benefits for deployment are:

- **Consistency.** The container behaves the same on your laptop, in CI, and in production. No "works on my machine" surprises.
- **Isolation.** Your application's dependencies do not conflict with anything else on the host.
- **Portability.** Every cloud provider, every PaaS, every VPS can run a Docker container.

We will use a **multi-stage build**: one stage compiles the application and creates the release, and a second stage copies just the release into a minimal runtime image. This keeps the final image small -- typically under 50 MB.

### 1.5 Environment Variable Configuration

Hardcoding secrets, ports, and file paths is a development convenience that becomes a production liability. Secrets end up in version control. Changing a port requires rebuilding the application. Different environments (development, staging, production) need different values.

The standard solution is environment variables. Your application reads configuration from the environment at startup:

| Variable            | Purpose                                      | Default           |
|---------------------|----------------------------------------------|-------------------|
| `SECRET_KEY_BASE`   | Cryptographic key for signing cookies         | Random (dev only) |
| `PORT`              | HTTP port the server listens on               | `8000`            |
| `DATABASE_PATH`     | Path to the SQLite database file              | `./teamwork.db`   |
| `LOG_LEVEL`         | Minimum log level (`debug`, `info`, `warning`)| `info`            |

In Gleam, the `envoy` package provides `envoy.get("VARIABLE_NAME")`, which returns `Ok(value)` or `Error(Nil)`.

### 1.6 Production Checklist

Before you point real users at your application, walk through this checklist:

**Security:**
- [ ] HTTPS everywhere. Use a reverse proxy like Caddy (automatic TLS) or nginx with Let's Encrypt.
- [ ] `SECRET_KEY_BASE` is a long, random string stored in the environment -- not in source code.
- [ ] Passwords are hashed (we did this in the authentication chapter).
- [ ] CSRF protection is enabled (Wisp provides this).
- [ ] Rate limiting on login and form submission endpoints.

**Reliability:**
- [ ] Data persists across restarts (SQLite or another database).
- [ ] Proper error handling -- no `let assert` on user input. Reserve `let assert` for startup and programmer errors.
- [ ] Logging with appropriate levels (`info` for normal events, `warning` for recoverable issues, `error` for failures).
- [ ] Graceful shutdown -- the BEAM handles this well by default.

**Performance:**
- [ ] Static assets served with long cache durations and immutable headers.
- [ ] HTMX loaded efficiently (CDN with SRI or self-hosted with cache headers).
- [ ] Database queries use indexes where needed.

**Operations:**
- [ ] Database backups. With SQLite, this is as simple as copying the file.
- [ ] Health check endpoint (`GET /health` returns 200).
- [ ] Monitoring and alerting (uptime checks, error rate, response times).
- [ ] A deployment process you can run confidently -- ideally automated.
- [ ] Tool versions pinned in `mise.toml` so every developer and CI server uses the same Gleam and Erlang versions.

### 1.7 HTMX CDN Strategy

Throughout this course, we loaded HTMX from a CDN:

```html
<script src="https://unpkg.com/htmx.org@2.0.8/dist/htmx.min.js"></script>
```

This is convenient for development, but in production you should consider two additional concerns:

**Subresource Integrity (SRI).** If you use a CDN, add an `integrity` attribute with a cryptographic hash. This ensures that even if the CDN is compromised, the browser will refuse to execute a modified file:

```html
<script src="https://unpkg.com/htmx.org@2.0.8/dist/htmx.min.js"
        integrity="sha384-..."
        crossorigin="anonymous"></script>
```

You can find the correct hash on the official HTMX website.

**Self-hosting.** Download `htmx.min.js` and serve it from your own `/static/js/` directory. This eliminates the CDN as a dependency. If unpkg goes down, your application keeps working. Combined with proper cache headers, the performance is identical.

For the Teamwork application, we will self-host HTMX. It is a single file, approximately 14 KB gzipped. There is no reason to depend on a third-party service for something this small.

---

## 2. Code Walkthrough

We are going to make four changes to the Teamwork application:

1. Add SQLite persistence with the `sqlight` package.
2. Read configuration from environment variables.
3. Build a release with `gleam export erlang-shipment`.
4. Containerize the application with Docker.

### 2.1 Adding SQLite

First, update `gleam.toml` to add the `sqlight` dependency:

```toml
[dependencies]
gleam_stdlib = ">= 0.50.0 and < 1.0.0"
gleam_http = ">= 4.0.0 and < 5.0.0"
gleam_erlang = ">= 1.0.0 and < 2.0.0"
gleam_otp = ">= 1.0.0 and < 2.0.0"
wisp = ">= 2.0.0 and < 3.0.0"
mist = ">= 5.0.0 and < 6.0.0"
lustre = ">= 5.0.0 and < 6.0.0"
hx = ">= 3.0.0 and < 4.0.0"
sqlight = ">= 1.0.0 and < 2.0.0"
```

Then download the new dependency:

```bash
gleam deps download
```

### 2.2 The Database Module

Create a new module at `src/teamwork/db.gleam`. This module handles the database connection, migrations, and all queries:

```gleam
// src/teamwork/db.gleam

import gleam/dynamic/decode
import gleam/result
import sqlight

pub type DbError {
  DbError(message: String)
}

/// Open a connection to the SQLite database at the given path.
/// The file is created automatically if it does not exist.
pub fn connect(path: String) -> Result(sqlight.Connection, sqlight.Error) {
  sqlight.open(path)
}

/// Run migrations to create the database schema.
/// CREATE TABLE IF NOT EXISTS is idempotent -- safe to run on every startup.
pub fn migrate(db: sqlight.Connection) -> Result(Nil, sqlight.Error) {
  sqlight.exec(db, "
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

Let's walk through the migration SQL.

**`CREATE TABLE IF NOT EXISTS`** makes the migration idempotent. The first time the application starts, it creates the tables. On subsequent starts, the statement is a no-op. This is the simplest migration strategy -- appropriate for applications that are still in active development. For production systems with evolving schemas, you would add a version table and run numbered migration scripts.

**`id TEXT PRIMARY KEY`** uses UUIDs as text. We generate the IDs in Gleam and pass them in, which keeps the application logic independent of the database.

**`done INTEGER NOT NULL DEFAULT 0`** stores the boolean as an integer (0 or 1). SQLite does not have a native boolean type, so this is the standard convention.

**`REFERENCES boards(id) ON DELETE CASCADE`** ensures that when a board is deleted, all its tasks are automatically removed. Referential integrity at the database level is a safety net that catches bugs your application code might miss.

### 2.3 Query Functions

Now add the query functions to the same `db.gleam` module:

```gleam
// -- Task queries --

pub type Task {
  Task(id: String, title: String, description: String, done: Bool)
}

/// Fetch all tasks for a given board, newest first.
pub fn list_tasks(
  db: sqlight.Connection,
  board_id: String,
) -> Result(List(Task), sqlight.Error) {
  let sql = "
    SELECT id, title, description, done
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

/// Insert a new task into the database.
pub fn create_task(
  db: sqlight.Connection,
  board_id: String,
  task: Task,
) -> Result(Nil, sqlight.Error) {
  let sql = "
    INSERT INTO tasks (id, board_id, title, description, done)
    VALUES (?1, ?2, ?3, ?4, ?5)
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
    ],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}

/// Toggle a task's done status. Returns the updated task.
pub fn toggle_task(
  db: sqlight.Connection,
  task_id: String,
) -> Result(Task, sqlight.Error) {
  let sql = "
    UPDATE tasks SET done = CASE WHEN done = 0 THEN 1 ELSE 0 END
    WHERE id = ?1
    RETURNING id, title, description, done
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

/// Delete a task by ID.
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

/// Decode a database row into a Task.
fn task_decoder() -> decode.Decoder(Task) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  decode.success(Task(id: id, title: title, description: description, done: done))
}
```

A few things to notice about these queries:

**Parameterized queries.** We use `?1`, `?2`, etc. as placeholders and pass values through `sqlight.text(...)` and `sqlight.int(...)`. This prevents SQL injection. Never concatenate user input into SQL strings.

**`RETURNING` clause.** The `toggle_task` function uses `UPDATE ... RETURNING` to get the updated row back in a single query, avoiding a separate `SELECT`.

**The decoder.** `sqlight` uses Gleam's `gleam/dynamic/decode` module to decode rows. The `task_decoder` function maps column positions to the `Task` fields using `decode.field(index, decoder)`. Column 3 (the `done` integer) is decoded using `sqlight.decode_bool()`, which converts SQLite's 0/1 integers to Gleam booleans.

### 2.4 Environment Variable Configuration

Update the `main` function to read configuration from the environment:

```gleam
// src/teamwork.gleam

import envoy
import gleam/erlang/process
import gleam/int
import gleam/result
import mist
import wisp
import wisp/wisp_mist
import teamwork/db
import teamwork/router

pub fn main() {
  wisp.configure_logger()

  // --- Configuration from environment ---

  let secret_key_base = case envoy.get("SECRET_KEY_BASE") {
    Ok(key) -> key
    Error(_) -> {
      wisp.log_warning(
        "No SECRET_KEY_BASE set, using random key (not suitable for production)",
      )
      wisp.random_string(64)
    }
  }

  let port = case envoy.get("PORT") {
    Ok(port_str) -> {
      let assert Ok(p) = int.parse(port_str)
      p
    }
    Error(_) -> 8000
  }

  let db_path =
    envoy.get("DATABASE_PATH")
    |> result.unwrap("./teamwork.db")

  // --- Database setup ---

  let assert Ok(db) = db.connect(db_path)
  let assert Ok(_) = db.migrate(db)

  // --- Start the server ---

  let ctx = router.Context(db: db, secret_key_base: secret_key_base)

  let assert Ok(_) =
    wisp_mist.handler(router.handle_request(_, ctx), secret_key_base)
    |> mist.new
    |> mist.port(port)
    |> mist.start

  wisp.log_info("Teamwork started on port " <> int.to_string(port))

  process.sleep_forever()
}
```

Let's examine the configuration logic:

**`SECRET_KEY_BASE`** -- In production, this must be set. In development, we generate a random key and log a warning. The warning ensures you notice if you accidentally deploy without setting it.

**`PORT`** -- Defaults to 8000. Cloud platforms like Fly.io and Railway set the `PORT` environment variable automatically. Using `let assert Ok(p) = int.parse(port_str)` is appropriate here because if someone sets `PORT` to a non-integer, crashing immediately with a clear error is better than running on a wrong port.

**`DATABASE_PATH`** -- Defaults to `./teamwork.db` in the current working directory. In Docker, we will set this to `/data/teamwork.db` and mount a persistent volume at `/data`.

### 2.5 Updating the Context

The `Context` type that gets passed to route handlers now carries the database connection instead of (or in addition to) actor references:

```gleam
// src/teamwork/router.gleam

import sqlight

pub type Context {
  Context(db: sqlight.Connection, secret_key_base: String)
}
```

Route handlers that previously called actors now call database functions:

```gleam
// Before (in-memory actor):
let tasks = actor.call(ctx.tasks, 1000, GetTasks)

// After (SQLite):
let assert Ok(tasks) = db.list_tasks(ctx.db, board_id)
```

The change is mechanical. Every place you called an actor, replace it with the corresponding `db` function call. The view layer does not change at all -- it still receives the same `List(Task)` and renders the same HTML.

### 2.6 Self-Hosting HTMX

Download the HTMX file and place it in your static assets directory:

```bash
mkdir -p priv/static/js
curl -o priv/static/js/htmx.min.js https://unpkg.com/htmx.org@2.0.8/dist/htmx.min.js
```

Then update your layout function to reference the local file instead of the CDN:

```gleam
// Before (CDN):
html.script(
  [attribute.src("https://unpkg.com/htmx.org@2.0.8/dist/htmx.min.js")],
  "",
)

// After (self-hosted):
html.script(
  [attribute.src("/static/js/htmx.min.js")],
  "",
)
```

Wisp serves files from `priv/static/` under the `/static/` URL path by default (via the `wisp.serve_static` middleware). By self-hosting, you eliminate a runtime dependency on an external CDN. Your application works even if unpkg is down.

### 2.7 Caching Static Assets

Static assets like JavaScript files, CSS stylesheets, and images rarely change. When they do change, you deploy a new version. Between deployments, the browser should cache them aggressively to avoid unnecessary downloads.

Add a middleware function that sets cache headers on static asset responses:

```gleam
fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.rescue_crashes

  // Serve static files with caching
  use <- serve_static(req)

  // ... route matching ...
}

fn serve_static(req: wisp.Request, next: fn() -> wisp.Response) -> wisp.Response {
  case wisp.path_segments(req) {
    ["static", ..] -> {
      // Serve the file; if not found, return 404
      let response = {
        use <- wisp.serve_static(req, under: "/static", from: priv_directory())
        wisp.not_found()
      }
      // Add cache headers to successful static file responses
      case response.status {
        200 ->
          response
          |> wisp.set_header("Cache-Control", "public, max-age=31536000, immutable")
        _ -> response
      }
    }
    // Not a static file request -- continue to the router
    _ -> next()
  }
}
```

The `Cache-Control` header tells browsers:
- `public` -- any cache (browser, CDN, proxy) can store this response.
- `max-age=31536000` -- cache it for one year (the maximum recommended value).
- `immutable` -- the file will never change at this URL. Do not even bother revalidating.

This means that after the first visit, the browser never re-requests your CSS or JavaScript files. The `immutable` directive is particularly powerful -- without it, browsers may still send conditional requests (`If-None-Match`) on page loads, adding latency even when the file has not changed.

If you need to bust the cache when you update a file, use versioned filenames (e.g., `htmx.min.2.0.8.js`) or query string parameters (e.g., `/static/js/htmx.min.js?v=2`).

### 2.8 Graceful Degradation

One of HTMX's philosophical commitments is progressive enhancement. Your application should work -- at least at a basic level -- without JavaScript.

Here are three practices to follow:

**1. Use real `<form>` elements with `action` and `method` attributes.**

```gleam
html.form([
  attribute.action("/tasks"),
  attribute.method("post"),
  hx.post("/tasks"),
  hx.target(hx.Selector("#task-list")),
  hx.swap(hx.Beforeend),
], [
  // ... form fields ...
  html.button([attribute.type_("submit")], [element.text("Add Task")]),
])
```

Without JavaScript, the form submits normally via the `action` and `method` attributes. With JavaScript, HTMX intercepts the submission and makes an AJAX request instead. Both paths work.

**2. Use `hx-boost` on navigation links.**

```gleam
html.a([
  attribute.href("/boards/123"),
  hx.boost(True),
], [element.text("My Board")])
```

Without JavaScript, this is a normal link. With JavaScript, `hx-boost` turns it into an AJAX request that swaps the page content without a full reload. The link works either way.

**3. Add a `<noscript>` fallback.**

```gleam
html.noscript([], [
  html.p([attribute.class("noscript-notice")], [
    element.text(
      "This application works best with JavaScript enabled. "
      <> "Core features like adding and managing tasks work without it, "
      <> "but real-time updates and search-as-you-type require JavaScript."
    ),
  ]),
])
```

This message appears only when JavaScript is disabled, informing the user about what they are missing without blocking their access entirely.

### 2.9 Production Logging

Wisp integrates with the BEAM's built-in logging system. In production, you want structured, appropriately leveled logs:

```gleam
// Informational -- normal operations
wisp.log_info("Server started on port " <> int.to_string(port))

// Warnings -- recoverable issues
wisp.log_warning("No SECRET_KEY_BASE set, using random key")

// Errors -- failures that need attention
wisp.log_error("Failed to connect to database: " <> error_message)
```

The BEAM's logger supports multiple backends. In production, you might route logs to a file, a log aggregation service, or stdout (which Docker and most PaaS providers capture automatically).

A health check endpoint gives monitoring tools a simple way to verify the application is running:

```gleam
["health"] -> wisp.ok()
```

This returns a `200 OK` with no body. Uptime monitoring services can poll this endpoint every 30 seconds to detect outages.

### 2.10 Building a Release

With all the code changes in place, build the release:

```bash
# Make sure everything compiles cleanly
gleam build

# Export the release
gleam export erlang-shipment
```

The release appears in `build/erlang-shipment/`. Examine its structure:

```
build/erlang-shipment/
├── entrypoint.sh          # Startup script
├── erlang/                 # The BEAM virtual machine
│   ├── bin/
│   ├── lib/
│   └── ...
└── teamwork/               # Your compiled application
    └── ebin/
        ├── teamwork.app
        └── *.beam          # Compiled modules
```

To run the release:

```bash
SECRET_KEY_BASE="your-64-char-secret-here" \
DATABASE_PATH="./teamwork.db" \
PORT=8000 \
./build/erlang-shipment/entrypoint.sh run
```

This starts the application exactly as `gleam run` would, but without needing the Gleam or Erlang compilers. The `entrypoint.sh` script finds the bundled BEAM VM and launches your application on it.

### 2.11 The Dockerfile

Here is a multi-stage Dockerfile that builds the release in one stage and creates a minimal runtime image in the second:

```dockerfile
# ============================================================
# Stage 1: Build
# ============================================================
FROM ghcr.io/gleam-lang/gleam:v1.14.0-erlang-alpine AS builder

WORKDIR /app

# Copy dependency manifests first -- Docker caches this layer
# so dependencies are only re-downloaded when gleam.toml changes
COPY gleam.toml manifest.toml ./
RUN gleam deps download

# Copy source code
COPY src/ src/
COPY priv/ priv/
COPY test/ test/

# Build the release
RUN gleam export erlang-shipment

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM alpine:3.20

# Install only the runtime libraries the BEAM needs
RUN apk add --no-cache libstdc++ ncurses-libs sqlite-libs

# Copy the release from the build stage
COPY --from=builder /app/build/erlang-shipment /app

# Copy static assets (they are needed at runtime for serving)
COPY --from=builder /app/priv /app/priv

WORKDIR /app

# Default environment variables
ENV PORT=8000
ENV DATABASE_PATH=/data/teamwork.db

# Create the data directory for the SQLite database
RUN mkdir -p /data

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["run"]
```

Let's trace through what happens at each step.

**Stage 1 (Build):** We start from the official Gleam Docker image, which includes Gleam, Erlang, and rebar3. We copy `gleam.toml` and `manifest.toml` first and run `gleam deps download`. Docker caches this layer, so if your dependencies have not changed, subsequent builds skip the download step entirely. Then we copy the source code and run `gleam export erlang-shipment`.

**Stage 2 (Runtime):** We start from `alpine:3.20`, a minimal Linux distribution (~7 MB). We install only the three shared libraries the BEAM needs: `libstdc++` (C++ standard library), `ncurses-libs` (terminal handling), and `sqlite-libs` (SQLite). Then we copy the release from the build stage and the static assets.

The final image is typically **40-60 MB**. Compare that to the build image, which is several hundred MB. Multi-stage builds are one of Docker's most valuable features.

**Building and running the image:**

```bash
# Build the image
docker build -t teamwork .

# Run it
docker run -d \
  --name teamwork \
  -p 8000:8000 \
  -v teamwork-data:/data \
  -e SECRET_KEY_BASE="$(openssl rand -hex 32)" \
  teamwork
```

The `-v teamwork-data:/data` flag creates a Docker volume and mounts it at `/data` inside the container. This is where the SQLite database file lives. The volume persists even if the container is deleted, so your data survives container restarts and redeployments.

### 2.12 HTTPS with a Reverse Proxy

Your application listens on HTTP. In production, you need HTTPS. The simplest approach is a reverse proxy that handles TLS termination.

**Caddy** is particularly easy because it obtains and renews TLS certificates automatically:

```
# Caddyfile
teamwork.example.com {
    reverse_proxy localhost:8000
}
```

That is the entire configuration. Caddy automatically obtains a certificate from Let's Encrypt, redirects HTTP to HTTPS, and proxies requests to your application.

If you prefer **nginx**, the configuration is more verbose but equally capable:

```nginx
server {
    listen 443 ssl;
    server_name teamwork.example.com;

    ssl_certificate /etc/letsencrypt/live/teamwork.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/teamwork.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Either way, your Gleam application does not need to know about TLS. It serves plain HTTP, and the reverse proxy handles encryption.

---

## 3. Full Code Listing

Here are the complete production-ready files. These listings incorporate every change from the walkthrough above.

### `gleam.toml`

```toml
name = "teamwork"
version = "1.0.0"
target = "erlang"

[dependencies]
gleam_stdlib = ">= 0.50.0 and < 1.0.0"
gleam_http = ">= 4.0.0 and < 5.0.0"
gleam_erlang = ">= 1.0.0 and < 2.0.0"
gleam_otp = ">= 1.0.0 and < 2.0.0"
wisp = ">= 2.0.0 and < 3.0.0"
mist = ">= 5.0.0 and < 6.0.0"
lustre = ">= 5.0.0 and < 6.0.0"
hx = ">= 3.0.0 and < 4.0.0"
sqlight = ">= 1.0.0 and < 2.0.0"
envoy = ">= 1.0.0 and < 2.0.0"

[dev-dependencies]
gleeunit = ">= 1.0.0 and < 2.0.0"
```

### `src/teamwork.gleam`

```gleam
import envoy
import gleam/erlang/process
import gleam/int
import gleam/result
import mist
import wisp
import wisp/wisp_mist
import teamwork/db
import teamwork/router

pub fn main() {
  wisp.configure_logger()

  // --- Configuration from environment ---

  let secret_key_base = case envoy.get("SECRET_KEY_BASE") {
    Ok(key) -> key
    Error(_) -> {
      wisp.log_warning(
        "No SECRET_KEY_BASE set, using random key (not suitable for production)",
      )
      wisp.random_string(64)
    }
  }

  let port = case envoy.get("PORT") {
    Ok(port_str) -> {
      let assert Ok(p) = int.parse(port_str)
      p
    }
    Error(_) -> 8000
  }

  let db_path =
    envoy.get("DATABASE_PATH")
    |> result.unwrap("./teamwork.db")

  // --- Database setup ---

  let assert Ok(db) = db.connect(db_path)
  let assert Ok(_) = db.migrate(db)

  wisp.log_info("Database connected at " <> db_path)

  // --- Start the server ---

  let ctx = router.Context(db: db, secret_key_base: secret_key_base)

  let assert Ok(_) =
    wisp_mist.handler(router.handle_request(_, ctx), secret_key_base)
    |> mist.new
    |> mist.port(port)
    |> mist.start

  wisp.log_info("Teamwork started on port " <> int.to_string(port))

  process.sleep_forever()
}
```

### `src/teamwork/db.gleam`

```gleam
import gleam/dynamic/decode
import gleam/result
import sqlight

pub type Task {
  Task(id: String, title: String, description: String, done: Bool)
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
    SELECT id, title, description, done
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
  let sql = "
    INSERT INTO tasks (id, board_id, title, description, done)
    VALUES (?1, ?2, ?3, ?4, ?5)
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
    RETURNING id, title, description, done
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
  decode.success(Task(id: id, title: title, description: description, done: done))
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

### `src/teamwork/router.gleam` (relevant excerpt)

```gleam
import gleam/http
import gleam/list
import gleam/result
import gleam/string
import lustre/element
import sqlight
import wisp
import teamwork/db

pub type Context {
  Context(db: sqlight.Connection, secret_key_base: String)
}

pub fn handle_request(
  req: wisp.Request,
  ctx: Context,
) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.rescue_crashes
  use <- serve_static(req)

  case wisp.path_segments(req) {
    [] -> home_page(req, ctx)
    ["health"] -> wisp.ok()
    ["boards", board_id, "tasks"] -> handle_tasks(req, ctx, board_id)
    ["boards", board_id, "tasks", task_id, "toggle"] ->
      toggle_task(req, ctx, task_id)
    ["boards", board_id, "tasks", task_id, "delete"] ->
      delete_task(req, ctx, task_id)
    _ -> wisp.not_found()
  }
}

fn serve_static(
  req: wisp.Request,
  next: fn() -> wisp.Response,
) -> wisp.Response {
  case wisp.path_segments(req) {
    ["static", ..] -> {
      let response = {
        use <- wisp.serve_static(req, under: "/static", from: priv_directory())
        wisp.not_found()
      }
      case response.status {
        200 ->
          response
          |> wisp.set_header(
            "Cache-Control",
            "public, max-age=31536000, immutable",
          )
        _ -> response
      }
    }
    _ -> next()
  }
}

fn handle_tasks(
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

  let query_params = wisp.get_query(req)
  let search_query =
    list.key_find(query_params, "q")
    |> result.unwrap("")
  let status_filter =
    list.key_find(query_params, "status")
    |> result.unwrap("all")

  let filtered =
    tasks
    |> filter_by_status(status_filter)
    |> filter_by_search(search_query)

  let is_htmx =
    list.key_find(req.headers, "hx-request")
    |> result.unwrap("")
    == "true"

  case is_htmx {
    True -> {
      let html = task_list_html(filtered)
      wisp.html_response(
        element.to_string(html),
        200,
      )
    }
    False -> {
      let page = tasks_page(filtered, search_query, status_filter)
      wisp.html_response(
        element.to_document_string(page),
        200,
      )
    }
  }
}

fn priv_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}
```

### `Dockerfile`

```dockerfile
# ============================================================
# Stage 1: Build
# ============================================================
FROM ghcr.io/gleam-lang/gleam:v1.14.0-erlang-alpine AS builder

WORKDIR /app

COPY gleam.toml manifest.toml ./
RUN gleam deps download

COPY src/ src/
COPY priv/ priv/
COPY test/ test/

RUN gleam export erlang-shipment

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM alpine:3.20

RUN apk add --no-cache libstdc++ ncurses-libs sqlite-libs

COPY --from=builder /app/build/erlang-shipment /app
COPY --from=builder /app/priv /app/priv

WORKDIR /app

ENV PORT=8000
ENV DATABASE_PATH=/data/teamwork.db

RUN mkdir -p /data

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["run"]
```

### `.dockerignore`

```
build/
_build/
.git/
.gitignore
*.md
```

---

## 4. Exercise

Now it is your turn. Work through the following tasks to solidify everything from this chapter and from the course as a whole.

### Task 1: Migrate In-Memory State to SQLite

Replace all actor-based state management with database calls:

1. Add the `sqlight` dependency to `gleam.toml`.
2. Create the `db.gleam` module with `connect`, `migrate`, and all query functions.
3. Update `main` to open the database and run migrations on startup.
4. Replace every `actor.call(ctx.tasks, ...)` with the corresponding `db.list_tasks(ctx.db, ...)`, `db.create_task(ctx.db, ...)`, etc.
5. Remove the actor module entirely -- it is no longer needed.

**Acceptance criteria:** Add a few tasks through the web interface. Stop the server (`Ctrl+C`). Start it again (`gleam run`). Your tasks are still there.

### Task 2: Environment Variable Configuration

1. Read `SECRET_KEY_BASE`, `PORT`, and `DATABASE_PATH` from environment variables with sensible defaults.
2. Log a warning when `SECRET_KEY_BASE` is not set.
3. Test by running: `PORT=3000 gleam run` and verifying the server starts on port 3000.

**Acceptance criteria:** The server respects all three environment variables. When none are set, it starts on port 8000 with a random secret key and a local `teamwork.db` file.

### Task 3: Self-Host HTMX

1. Download `htmx.min.js` into `priv/static/js/`.
2. Update the layout function to load HTMX from `/static/js/htmx.min.js`.
3. Verify the application still works as before.

**Acceptance criteria:** Open the browser's Network tab. The HTMX file is loaded from `localhost:8000/static/js/htmx.min.js`, not from unpkg.

### Task 4: Add Caching Headers

1. Implement the `serve_static` middleware that adds `Cache-Control: public, max-age=31536000, immutable` to all responses under `/static/`.
2. Verify the header is present by inspecting the response headers in your browser's Network tab.

**Acceptance criteria:** Static assets have the `Cache-Control` header. Non-static routes (like `/tasks`) do not.

### Task 5: Build a Docker Image

1. Create the `Dockerfile` as shown in the full code listing.
2. Create the `.dockerignore` file.
3. Build the image: `docker build -t teamwork .`
4. Run the container: `docker run -d --name teamwork -p 8000:8000 -v teamwork-data:/data -e SECRET_KEY_BASE="$(openssl rand -hex 32)" teamwork`
5. Visit `http://localhost:8000` and verify the application works.
6. Add some tasks. Stop and remove the container: `docker stop teamwork && docker rm teamwork`. Start a new container with the same volume. Verify your tasks are still there.

**Acceptance criteria:** The application runs inside Docker. Data persists across container restarts because the SQLite database is on a mounted volume.

### Task 6: Add a Health Check Endpoint

Add a `GET /health` route that returns a `200 OK` response. This is the simplest possible route, but it is essential for production monitoring.

**Acceptance criteria:** `curl http://localhost:8000/health` returns a 200 status code.

---

## 5. Exercise Solution Hints

Try each task on your own before reading these hints.

### Hint for Task 1

`sqlight.open("./teamwork.db")` creates the database file if it does not exist. You do not need to create the file manually.

The biggest change is in how you handle errors. Actors cannot fail (the process traps errors), but database calls return `Result` types. Use `let assert Ok(tasks) = db.list_tasks(...)` during initial development, then add proper error handling later.

If you see an error about SQLite not being found, you may need to install the SQLite development libraries on your system. On macOS: `brew install sqlite`. On Ubuntu: `sudo apt install libsqlite3-dev`.

### Hint for Task 2

The key function is `envoy.get("VARIABLE_NAME")`, which returns `Result(String, Nil)`. Pattern match on it:

```gleam
let port = case envoy.get("PORT") {
  Ok(port_str) -> {
    let assert Ok(p) = int.parse(port_str)
    p
  }
  Error(_) -> 8000
}
```

Remember to `import envoy` and add `envoy = ">= 1.0.0 and < 2.0.0"` to `gleam.toml`.

### Hint for Task 3

```bash
curl -o priv/static/js/htmx.min.js https://unpkg.com/htmx.org@2.0.8/dist/htmx.min.js
```

Make sure the `priv/static/js/` directory exists first. After downloading, update your layout to reference `/static/js/htmx.min.js` instead of the CDN URL.

### Hint for Task 4

The static serving middleware pattern uses early return. When `wisp.serve_static` succeeds (the request path matches a file), add the cache header and return immediately. When it fails (not a static file request), call `next()` to continue to the router.

Test with `curl -v http://localhost:8000/static/js/htmx.min.js` and look for the `Cache-Control` header in the response.

### Hint for Task 5

Docker multi-stage builds keep the final image small. The build stage includes the Gleam compiler, Erlang, and all build tools -- several hundred MB. The runtime stage only needs the BEAM runtime and your compiled code -- around 40-60 MB.

If the build fails with a memory error, increase Docker's memory limit in your Docker Desktop settings.

The `-v teamwork-data:/data` flag is critical. Without a volume, the SQLite database is stored inside the container's filesystem and is lost when the container is removed.

### Hint for Task 6

This is intentionally simple:

```gleam
["health"] -> wisp.ok()
```

`wisp.ok()` returns a `200 OK` response with no body. That is all a health check needs.

---

## 6. Key Takeaways

1. **In-memory state is for development; persistence is for production.** Actors are great for prototyping, but every server restart wipes the slate clean. SQLite gives you durable storage with zero infrastructure overhead.

2. **SQLite is a production-grade database.** It handles millions of reads per second, supports transactions and referential integrity, and its "database is a file" model makes backups trivial. Do not underestimate it.

3. **`PRAGMA journal_mode=WAL` and `PRAGMA foreign_keys=ON` are essential.** WAL (Write-Ahead Logging) mode allows concurrent reads during writes. Foreign key enforcement is off by default in SQLite -- you must enable it explicitly on every connection.

4. **Environment variables are the standard for configuration.** They keep secrets out of source code, make your application environment-aware, and work seamlessly with Docker, CI/CD, and every cloud platform.

5. **`gleam export erlang-shipment` creates a self-contained release.** The release bundles your compiled code and the BEAM VM. No Gleam or Erlang installation required on the target machine.

6. **Multi-stage Docker builds are the right approach.** One stage for building (large, has all the tools), one stage for running (small, has only the runtime). Final images around 40-60 MB.

7. **Self-hosting HTMX eliminates a runtime dependency.** HTMX is a single ~14 KB file. Serving it from your own static assets directory means your application works even if the CDN is down.

8. **Aggressive cache headers make static assets effectively free.** `Cache-Control: public, max-age=31536000, immutable` tells browsers to cache the file for a year and never revalidate. Use versioned filenames or query parameters to bust the cache when you update files.

9. **Graceful degradation is not extra work -- it is free.** Using real `<form>` elements and `<a>` tags means your application works without JavaScript by default. HTMX enhances the experience; it does not replace the foundation.

10. **A reverse proxy handles HTTPS.** Caddy does it in two lines. Nginx does it in ten. Your Gleam application serves plain HTTP and lets the proxy handle TLS. This separation of concerns keeps your application code simple.

---

## Where to Go from Here

You have built a complete web application from scratch. You started with a blank `gleam new` project and an empty browser tab. Seventeen chapters later, you have a collaborative task board with routing, HTMX-powered interactivity, forms with validation, search and filtering, inline editing, authentication, real-time updates, and a production deployment pipeline.

That is a real application. Not a toy. Not a tutorial that skips the hard parts. You have confronted every layer of the stack -- from HTTP fundamentals to Docker containers -- and built something that works.

Here is where you can go next.

### Deepen Your Gleam Knowledge

- **Join the Gleam Discord community.** The Gleam community is small, welcoming, and active. If you get stuck, ask a question. If you build something, share it. [https://discord.gg/Fm8Pwmy](https://discord.gg/Fm8Pwmy)
- **Read the Gleam language tour.** You have learned a lot of Gleam through this course, but there are features we did not cover: use expressions in more depth, opaque types, bit arrays, and more. [https://tour.gleam.run](https://tour.gleam.run)
- **Explore the Gleam package ecosystem.** Browse [https://packages.gleam.run](https://packages.gleam.run) for libraries that solve common problems: JSON encoding, email, background jobs, and more.

### Deepen Your HTMX Knowledge

- **Read the HTMX documentation.** We covered the most important attributes, but there are many more: `hx-disabled-elt`, `hx-on`, `hx-history-elt`, response headers like `HX-Trigger`, and the full event system. [https://htmx.org/docs/](https://htmx.org/docs/)
- **Study the HTMX examples.** The official examples page shows dozens of common UI patterns implemented in pure HTMX: click-to-edit, infinite scroll, bulk update, lazy loading, and more. [https://htmx.org/examples/](https://htmx.org/examples/)
- **Read "Hypermedia Systems" by Carson Gross.** The creator of HTMX wrote a book on the philosophy and architecture of hypermedia-driven applications. It is available free online and is the definitive guide to thinking in hypermedia. [https://hypermedia.systems](https://hypermedia.systems)

### Extend the Teamwork Application

The application you have built is a solid foundation. Here are features you could add:

- **Drag-and-drop task ordering.** Use the Sortable.js library with HTMX events to reorder tasks by dragging. Send the new order to the server with an `hx-post` on the `sortupdate` event.
- **Labels and tags.** Add a `tags` table with a many-to-many relationship to tasks. Let users create, assign, and filter by tags.
- **Due dates.** Add a `due_date` column to the tasks table. Show overdue tasks in red. Add a date picker to the task creation form. (This is covered in [Chapter 18](18-flatpickr-and-third-party-js.md) with Flatpickr.)
- **File attachments.** Handle multipart file uploads with Wisp. Store files on disk or in S3. Show thumbnails for images.
- **WebSocket-powered real-time collaboration.** Mist supports WebSockets natively. When one user adds a task, push the update to all connected clients. HTMX has WebSocket support through the `ws` extension.
- **Email notifications.** Send an email when a task is assigned to you or when a task you created is completed.

### Deploy to a Real Server

You have a Docker image. Now put it somewhere:

- **Fly.io** -- Deploy with `fly launch`. Fly supports persistent volumes for SQLite. This is one of the simplest paths from Docker image to running application.
- **Railway** -- Connect your Git repository and Railway builds and deploys automatically on every push.
- **A VPS (DigitalOcean, Hetzner, Linode)** -- Rent a small server, install Docker, and run your container. You have full control and it costs a few dollars a month.
- **Your own hardware** -- A Raspberry Pi running your Gleam application is a perfectly viable deployment target for small teams.

### Keep Building

The most important thing is to keep building. The gap between "I followed a tutorial" and "I can build things on my own" is bridged by building things on your own. Start a project that solves a real problem for you. You will get stuck. You will look things up. You will learn more from that one project than from ten more tutorials.

You have the tools. You have the knowledge. Go build something.

---

*This is the final chapter of the core curriculum. If you want to see how to
integrate third-party JavaScript libraries (like date pickers) with HTMX,
continue to **[Chapter 18](18-flatpickr-and-third-party-js.md) -- Integrating Third-Party JavaScript: Flatpickr**.
After that, **[Chapters 19-28](../04-real-world-patterns/19-error-handling-and-graceful-degradation.md) (Real-World Patterns)** cover error handling,
response headers, inline editing, modals, `_hyperscript` recipes, dynamic
forms, file uploads, accessibility, database performance, and background jobs.
The appendices contain additional reference material: **[Appendix A](../05-appendices/A1-bibliography-and-resources.md)** collects
official documentation links, homepages, YouTube channels, and books for every
tool used in this course, and **[Appendix B](../05-appendices/A2-hyperscript.md)** covers \_hyperscript (HTMX's
companion client-side scripting language).*

*Thank you for following along.*
