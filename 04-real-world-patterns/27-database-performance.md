# Chapter 27 -- Database Performance and Queries

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 26](26-accessible-htmx.md) made the Teamwork board accessible with ARIA attributes, focus management, and screen-reader-friendly HTMX patterns. Now we turn inward -- to the database queries that power every single swap.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain what database indexes are (B-tree structure) and when to add them.
2. Write `CREATE INDEX` migrations and measure improvement with `EXPLAIN QUERY PLAN`.
3. Replace in-memory pagination with SQL `LIMIT`/`OFFSET` and explain the deep-page problem.
4. Implement cursor-based (keyset) pagination for efficient deep navigation.
5. Add full-text search using SQLite FTS5, replacing the `LIKE` pattern from [Chapter 10](../02-intermediate/10-lists-filters-search.md).
6. Identify and fix N+1 query patterns with JOINs.

---

## 1. Theory

### 1.1 Why Database Performance Matters for HTMX

Here is a truth that is easy to forget when you are building with HTMX: **every swap hits the server, and most server hits touch the database.**

In a traditional single-page application, the client might fetch a JSON payload once and then manipulate it locally for several minutes -- sorting, filtering, paginating -- without another server request. The database gets one query, and the client does the rest.

HTMX works differently. When the user types into the search box, the server runs a query. When they click a filter tab, the server runs a query. When they scroll down and trigger infinite scroll, the server runs a query. When the page polls every 30 seconds, the server runs a query. Each interaction is a fresh round-trip, and each round-trip almost certainly involves the database.

This is not a flaw. It is the architecture. The server is the source of truth, and every response is a freshly rendered view of the current state. That simplicity is what makes HTMX applications easy to reason about.

But it means that slow queries have nowhere to hide. In a client-heavy SPA, a slow query might be masked by client-side caching. In an HTMX application, a slow query is a slow swap. The user feels it immediately.

The good news: SQLite is extraordinarily fast when used correctly. A well-indexed query on a table with 100,000 rows returns in under a millisecond. The bad news: an unindexed query on the same table can take 50 milliseconds or more, and that gap only grows as the data grows.

This chapter is about closing that gap. We will add indexes, inspect query plans, fix pagination, upgrade search, and eliminate N+1 patterns. The result is a Teamwork board that stays fast at 10 tasks and stays fast at 100,000 tasks.

### 1.2 Indexes: The Single Biggest Win

Imagine you have a phone book with 50,000 entries sorted alphabetically by last name. Finding "Martinez" is fast -- you open to the middle, see that you are at "L", flip forward a bit, and land on "M" within a few seconds. That is a search on a sorted structure. It takes `O(log n)` steps.

Now imagine the same phone book, but the entries are in random order. Finding "Martinez" means reading every single entry from the beginning until you find it. That is `O(n)` -- a full scan. For 50,000 entries, that is 50,000 comparisons instead of 16.

Database indexes work like the sorted phone book. Without an index, SQLite must scan every row in the table to find matching ones. With an index, it navigates a **B-tree** data structure that lets it jump directly to the right rows.

A B-tree is a balanced tree where:

- Each node contains multiple keys in sorted order.
- Internal nodes have pointers to child nodes.
- Leaf nodes contain pointers to the actual table rows.
- The tree stays balanced, so every search takes the same number of steps: `O(log n)`.

When you create an index on a column, SQLite builds a B-tree that maps column values to row locations. When you query with a `WHERE` clause on that column, SQLite walks the B-tree instead of scanning the entire table.

Here is the critical insight: **SQLite does not automatically create indexes on columns you query.** It creates an index on the primary key (that is what `PRIMARY KEY` means) and on columns marked `UNIQUE`. Everything else is unindexed until you explicitly create one.

In the Teamwork database, consider the `tasks` table:

```sql
CREATE TABLE tasks (
  id TEXT PRIMARY KEY,
  board_id TEXT NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  done INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

The `id` column is indexed (primary key). But `board_id` is not. Every time we run `SELECT ... FROM tasks WHERE board_id = ?1`, SQLite must scan every row in the table to find the ones that belong to that board. When you have 5 boards and 100 tasks, this is fine. When you have 50 boards and 100,000 tasks, it is a problem.

The rule of thumb: **add an index on every column that appears in a `WHERE` clause, an `ORDER BY` clause, or a `JOIN` condition -- unless the table is small enough that it does not matter.**

What counts as "small enough"? In practice, tables under a few hundred rows are fine without extra indexes. The full scan is still fast because the entire table fits in a single disk page. Above that, measure with `EXPLAIN QUERY PLAN` before guessing.

There is a cost to indexes: they consume disk space, and they slow down `INSERT`, `UPDATE`, and `DELETE` operations because SQLite must update the B-tree every time the indexed column changes. But for read-heavy workloads -- which is what HTMX applications tend to be, since every swap is a read -- the trade-off is overwhelmingly positive.

### 1.3 `EXPLAIN QUERY PLAN`

How do you know if a query is using an index or scanning the whole table? You ask SQLite.

`EXPLAIN QUERY PLAN` is a diagnostic command that shows you *how* SQLite will execute a query without actually running it. You prepend it to any `SELECT` statement:

```sql
EXPLAIN QUERY PLAN
SELECT id, title, done FROM tasks WHERE board_id = 'board-1' ORDER BY created_at DESC;
```

The output is a tree of operations. There are two key terms to watch for:

**`SCAN`** means SQLite is reading every row in the table. This is the full-phone-book scenario. For small tables, it is fine. For large tables, it is a performance problem.

**`SEARCH`** means SQLite is using an index to jump directly to matching rows. This is the sorted-phone-book scenario. It is almost always what you want.

Here is what the output looks like without an index on `board_id`:

```
QUERY PLAN
`--SCAN tasks
```

That single line tells you everything: SQLite is scanning the entire `tasks` table. Every row is read, the `board_id` column is compared to `'board-1'`, and non-matching rows are discarded. If the table has 100,000 rows, 100,000 rows are read.

Now add an index and run the same `EXPLAIN`:

```
QUERY PLAN
`--SEARCH tasks USING INDEX idx_tasks_board_id (board_id=?)
```

This is dramatically different. SQLite is searching the B-tree index `idx_tasks_board_id`, jumping directly to rows where `board_id` matches. If only 200 rows belong to `board-1`, only about 200 rows are read.

What about the `ORDER BY created_at DESC`? Without an index on `created_at`, SQLite still needs to sort the results in memory after finding them. The plan might look like:

```
QUERY PLAN
|--SEARCH tasks USING INDEX idx_tasks_board_id (board_id=?)
`--USE TEMP B-TREE FOR ORDER BY
```

That `USE TEMP B-TREE FOR ORDER BY` line means SQLite built a temporary B-tree in memory to sort the results. For small result sets this is fast, but for large result sets it consumes memory and time. If you frequently query tasks for a board sorted by creation date, a **composite index** on both columns eliminates the sort:

```sql
CREATE INDEX idx_tasks_board_id_created_at ON tasks(board_id, created_at DESC);
```

Now the plan becomes:

```
QUERY PLAN
`--SEARCH tasks USING INDEX idx_tasks_board_id_created_at (board_id=?)
```

No temporary B-tree, no sort. The index itself is sorted by `board_id` first and `created_at DESC` second, so SQLite reads the results in the right order directly from the index.

The mental model for `EXPLAIN QUERY PLAN` is simple:

| What you see                 | What it means                       | Action            |
|------------------------------|-------------------------------------|--------------------|
| `SCAN table`                 | Full table scan, no index used      | Add an index       |
| `SEARCH table USING INDEX`   | Index lookup, good                  | None needed        |
| `USE TEMP B-TREE FOR ORDER BY` | Sort in memory after lookup       | Consider composite index |
| `SEARCH table USING COVERING INDEX` | Index contains all needed columns | Excellent -- no table lookup needed |

Get in the habit of running `EXPLAIN QUERY PLAN` on any query you write. It takes a fraction of a second and tells you exactly where your performance stands.

### 1.4 Pagination: Offset vs Cursor

In [Chapter 16](../03-advanced/16-extensions-and-patterns.md) we implemented infinite scroll with in-memory pagination. The server loaded all tasks from the database, then used `list.drop` and `list.take` in Gleam to extract the right page:

```gleam
// Chapter 16 approach: load everything, slice in memory
let all_tasks = db.list_tasks(ctx.db, board_id)
let page_tasks = all_tasks
  |> list.drop(page * page_size)
  |> list.take(page_size)
```

This works when you have a few hundred tasks. But it has a fundamental problem: **every page request loads every task from the database into memory, then throws most of them away.** Page 50 loads all tasks, discards the first 49 pages worth, and keeps 20 items. The database does the work of reading thousands of rows. The application allocates memory for thousands of Task records. And then almost all of that work is wasted.

The fix is to push the pagination into SQL using `LIMIT` and `OFFSET`:

```sql
SELECT id, title, done, created_at
FROM tasks
WHERE board_id = ?1
ORDER BY created_at DESC
LIMIT 20 OFFSET 40
```

This tells SQLite: "Find the matching rows, sort them, skip the first 40, and return the next 20." SQLite does the slicing internally, which is far more efficient than loading everything into Gleam.

But `OFFSET` has its own problem: **the deep-page problem.**

When you say `OFFSET 10000`, SQLite must still locate and skip over 10,000 rows before returning any results. It does not have a magic shortcut to jump to row 10,001. The deeper the page, the more work SQLite does. Page 1 is fast (skip 0). Page 10 is fast (skip 180). Page 500 is slow (skip 9,980). Page 5,000 is very slow (skip 99,980).

For the Teamwork board, this probably does not matter. Most boards will have tens or hundreds of tasks, not tens of thousands. `OFFSET` pagination is simple, predictable, and fast enough.

But if you are building something where deep pages are common -- an activity log, a message history, a search engine results page -- you need a different approach: **cursor-based (keyset) pagination.**

Cursor-based pagination does not use `OFFSET`. Instead, it remembers where the previous page ended and asks the database for rows *after* that point:

```sql
SELECT id, title, done, created_at
FROM tasks
WHERE board_id = ?1 AND created_at < ?2
ORDER BY created_at DESC
LIMIT 20
```

The `?2` parameter is the `created_at` value of the last item on the previous page. This is the "cursor." SQLite uses the index on `created_at` to jump directly to that position and read the next 20 rows. It does not matter if the cursor points to row 10 or row 10,000 -- the query does the same amount of work either way.

The trade-offs between the two approaches:

| Property              | Offset Pagination              | Cursor Pagination                |
|-----------------------|-------------------------------|----------------------------------|
| **Implementation**    | Simple -- just a page number  | Requires tracking a cursor value |
| **Deep pages**        | Slow (scans skipped rows)     | Fast (index seek)                |
| **URL friendliness**  | `?page=5` (easy)              | `?cursor=2024-01-15T10:30:00` (ugly) |
| **Jump to page N**    | Easy                          | Not possible                     |
| **Consistent results**| May skip/duplicate on insert  | Stable (never skips/duplicates)  |
| **Best for**          | Small datasets, numbered pages | Large datasets, infinite scroll  |

For infinite scroll -- which is our Teamwork pattern from [Chapter 16](../03-advanced/16-extensions-and-patterns.md) -- cursor-based pagination is a natural fit. The user never needs to jump to "page 47." They just keep scrolling, and each batch picks up where the last one left off.

### 1.5 Full-Text Search with FTS5

In [Chapter 10](../02-intermediate/10-lists-filters-search.md) we implemented live search using the SQL `LIKE` operator:

```sql
SELECT id, title, done FROM tasks
WHERE board_id = ?1 AND title LIKE '%' || ?2 || '%'
```

This works, but it has two significant problems:

**Problem 1: Performance.** `LIKE '%something%'` cannot use a B-tree index. The leading wildcard means SQLite cannot narrow down the search -- it must check every row. This is a full table scan, period. For 100 tasks, nobody notices. For 100,000 tasks with a debounced search input firing on every keystroke, it becomes sluggish.

**Problem 2: Quality.** `LIKE` does exact substring matching. It does not understand word boundaries, stemming, or relevance. Searching for "design" will not match "designing" or "redesign." Searching for "task management" will not match a task titled "management of tasks." There is no concept of ranking results by relevance -- you get them in whatever order the database returns them.

SQLite's **FTS5** (Full-Text Search version 5) solves both problems. It is a built-in extension that provides:

- **Tokenization**: Text is split into words (tokens). Punctuation is ignored.
- **Stemming**: Optional. With the right tokenizer, "designing" and "design" match.
- **Relevance ranking**: The `bm25()` function ranks results by how well they match, using the BM25 algorithm (the same algorithm used by search engines).
- **Fast lookups**: FTS5 maintains an inverted index -- a data structure optimized for finding documents that contain a given word. Even with hundreds of thousands of documents, lookups are near-instant.
- **Boolean operators**: `AND`, `OR`, `NOT`, and phrase matching with quotes.

FTS5 uses a **virtual table** -- a special SQLite table that looks like a regular table but stores its data in a custom format optimized for text search. You create it with `CREATE VIRTUAL TABLE ... USING fts5(...)`.

There are two strategies for using FTS5:

**Strategy 1: Standalone FTS table.** The FTS table is the only storage. You insert directly into it and query it directly. This is simple but limited -- FTS tables do not support all SQL features (no `UPDATE`, no arbitrary columns, no foreign keys).

**Strategy 2: Content table with external content sync.** You keep your regular `tasks` table as the source of truth and create an FTS table that mirrors the searchable columns. You use triggers to keep them in sync. This is the approach we will use because it lets us keep our existing schema, queries, and foreign key relationships intact.

The FTS table declaration looks like this:

```sql
CREATE VIRTUAL TABLE tasks_fts USING fts5(
  title,
  description,
  content=tasks,
  content_rowid=rowid
);
```

The `content=tasks` parameter tells FTS5 that the content lives in the `tasks` table. The `content_rowid=rowid` parameter maps the FTS document ID to the `rowid` in the `tasks` table (SQLite automatically assigns a `rowid` to every row, even if your primary key is a TEXT column).

To keep the FTS index in sync with the main table, we use three triggers:

```sql
-- After INSERT: add the new row to the FTS index
CREATE TRIGGER tasks_ai AFTER INSERT ON tasks BEGIN
  INSERT INTO tasks_fts(rowid, title, description)
  VALUES (new.rowid, new.title, new.description);
END;

-- After DELETE: remove the row from the FTS index
CREATE TRIGGER tasks_ad AFTER DELETE ON tasks BEGIN
  INSERT INTO tasks_fts(tasks_fts, rowid, title, description)
  VALUES('delete', old.rowid, old.title, old.description);
END;

-- After UPDATE: remove the old version, insert the new version
CREATE TRIGGER tasks_au AFTER UPDATE ON tasks BEGIN
  INSERT INTO tasks_fts(tasks_fts, rowid, title, description)
  VALUES('delete', old.rowid, old.title, old.description);
  INSERT INTO tasks_fts(rowid, title, description)
  VALUES (new.rowid, new.title, new.description);
END;
```

The delete syntax looks odd -- `INSERT INTO tasks_fts(tasks_fts, ...)` with the table name as the first column value. This is FTS5's special "delete command" syntax. It tells FTS5 to remove the specified document from the index. The update trigger does a delete-then-insert because FTS5 does not support in-place updates.

To query the FTS table, use the `MATCH` operator:

```sql
SELECT t.id, t.title, t.description, t.done
FROM tasks t
JOIN tasks_fts fts ON t.rowid = fts.rowid
WHERE tasks_fts MATCH ?1 AND t.board_id = ?2
ORDER BY bm25(tasks_fts)
LIMIT 20
```

The `MATCH` operator is the FTS5 query interface. It accepts a search string and finds all documents that contain the specified terms. The `bm25(tasks_fts)` function returns a relevance score -- lower values mean better matches (it is a distance metric, not a similarity metric, so you sort ascending or just use `ORDER BY bm25(tasks_fts)`).

The `JOIN` connects the FTS results back to the original `tasks` table so we can access columns like `board_id` and `done` that are not in the FTS index.

### 1.6 N+1 Queries

The N+1 query problem is one of the most common performance mistakes in web applications. It happens when you:

1. Run one query to get a list of N items.
2. For each item, run another query to get related data.

The result is 1 + N queries. If you have 50 boards, that is 51 queries. If each query takes 1 millisecond, that is 51 milliseconds. If you have 200 boards, that is 201 milliseconds. The response time grows linearly with the number of items.

In the Teamwork application, imagine a "board listing" page that shows all boards with their task counts:

```gleam
// The N+1 pattern -- DO NOT DO THIS
pub fn list_boards_with_counts(
  db: sqlight.Connection,
) -> Result(List(BoardWithCount), sqlight.Error) {
  // Query 1: Get all boards
  use boards <- result.try(list_boards(db))

  // Queries 2..N+1: Get task count for each board
  list.try_map(boards, fn(board) {
    use count <- result.try(count_tasks_for_board(db, board.id))
    Ok(BoardWithCount(board: board, task_count: count))
  })
}
```

This code is clean and readable. Each function does one thing. But it is a performance trap. `list_boards` runs one query. Then `list.try_map` runs `count_tasks_for_board` once per board. If there are 50 boards, that is 51 queries.

The fix is a single query with a `JOIN` or subquery:

```sql
SELECT b.id, b.name, b.description, COUNT(t.id) AS task_count
FROM boards b
LEFT JOIN tasks t ON t.board_id = b.id
GROUP BY b.id
ORDER BY b.created_at DESC
```

One query. One round-trip. All the data.

The `LEFT JOIN` ensures boards with zero tasks still appear (with a count of 0). The `GROUP BY b.id` groups all tasks for each board together. The `COUNT(t.id)` counts the tasks in each group.

N+1 queries are insidious because the code that creates them looks perfectly reasonable. Each function call is sensible in isolation. The problem only becomes visible when you look at the pattern of database calls across the whole request.

The rule is: **if you are calling a database function inside a loop, you probably have an N+1 problem. Pull the loop into the SQL.**

### 1.7 WAL Mode and Connection Pooling

By default, SQLite uses a **rollback journal** for transactions. This means that during a write operation, the entire database file is locked. No reads can happen while a write is in progress. For a single-user command-line tool, this is fine. For a web server handling concurrent HTMX requests, it creates contention.

**WAL mode** (Write-Ahead Logging) changes this. Instead of locking the database file during writes, SQLite writes changes to a separate WAL file. Readers continue reading from the main database file. When a read starts, it sees a consistent snapshot of the database as it was at that moment -- even if a write is happening concurrently.

The result: **readers never block writers, and writers never block readers.**

To enable WAL mode:

```sql
PRAGMA journal_mode=WAL;
```

This is a one-time operation. Once set, WAL mode persists even after the database is closed and reopened. You should run this as part of your database initialization, right after opening the connection.

There are a few things to know about WAL mode:

- **Reads and writes can happen concurrently.** Multiple readers can read at the same time. One writer can write while readers are reading. But only one writer can write at a time -- if two writes happen simultaneously, one will wait.
- **The WAL file grows until a checkpoint.** SQLite periodically checkpoints (writes WAL contents back to the main database). You can trigger this manually with `PRAGMA wal_checkpoint`, but SQLite's automatic checkpointing is usually sufficient.
- **WAL mode does not help with write contention.** If your application has many concurrent writers (many users creating tasks at the same time), you will hit SQLite's single-writer limitation. For the Teamwork board, this is unlikely to be a problem. For a high-write application, consider PostgreSQL.

For connection management, the simplest approach is a single database connection shared across all requests. The BEAM runs your request handlers in lightweight processes, but they all share the same `sqlight.Connection`. SQLite handles the concurrency internally through its locking mechanism.

If you need more throughput, you can create a small pool of connections -- perhaps 2 to 4 -- and distribute requests across them. But for most SQLite applications, a single connection with WAL mode is sufficient for hundreds of concurrent users.

Another important pragma to set at initialization time:

```sql
PRAGMA foreign_keys=ON;
```

SQLite disables foreign key enforcement by default. This means `REFERENCES boards(id) ON DELETE CASCADE` in your schema is *documentation only* unless you explicitly enable it. You must run this pragma on every new connection -- it is not persistent like WAL mode.

A good initialization sequence:

```sql
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;
PRAGMA busy_timeout=5000;
```

The `busy_timeout` pragma tells SQLite to wait up to 5,000 milliseconds if the database is locked by another writer, rather than immediately returning an error. This prevents spurious "database is locked" errors during brief contention periods.

---

## 2. Code Walkthrough

We are going to make seven improvements to the Teamwork database layer:

1. Add indexes on frequently queried columns.
2. Build a debug endpoint that shows `EXPLAIN QUERY PLAN` output.
3. Replace in-memory pagination with SQL `LIMIT`/`OFFSET`.
4. Add a cursor-based pagination alternative.
5. Upgrade search from `LIKE` to FTS5.
6. Fix the N+1 pattern in board listing with a JOIN.
7. Enable WAL mode and other pragmas in database initialization.

### Step 1 -- Add Indexes

Open `src/teamwork/db.gleam` and update the `migrate` function to include index creation:

```gleam
// src/teamwork/db.gleam

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

    -- Indexes for common query patterns
    CREATE INDEX IF NOT EXISTS idx_tasks_board_id
      ON tasks(board_id);

    CREATE INDEX IF NOT EXISTS idx_tasks_board_id_created_at
      ON tasks(board_id, created_at DESC);

    CREATE INDEX IF NOT EXISTS idx_tasks_board_id_done
      ON tasks(board_id, done);
  ")
}
```

Three indexes, each serving a specific query pattern:

- **`idx_tasks_board_id`** -- Used whenever we fetch tasks for a board: `WHERE board_id = ?1`. This is the most frequently used filter in the entire application.

- **`idx_tasks_board_id_created_at`** -- A composite index used when fetching tasks for a board sorted by creation date: `WHERE board_id = ?1 ORDER BY created_at DESC`. The composite index satisfies both the filter and the sort, eliminating the need for a temporary B-tree.

- **`idx_tasks_board_id_done`** -- Used when filtering tasks by status within a board: `WHERE board_id = ?1 AND done = ?2`. This supports the filter tabs from [Chapter 10](../02-intermediate/10-lists-filters-search.md).

Notice the `IF NOT EXISTS` clause on each `CREATE INDEX`. This makes the migration idempotent -- safe to run on every application startup, just like `CREATE TABLE IF NOT EXISTS`.

You might wonder: do we need all three indexes? The `idx_tasks_board_id` index seems redundant with `idx_tasks_board_id_created_at` since the composite index also starts with `board_id`. You are right. SQLite can use a composite index for queries that filter on the leading column(s). So `idx_tasks_board_id_created_at` can serve double duty for both `WHERE board_id = ?1` and `WHERE board_id = ?1 ORDER BY created_at DESC`.

In practice, keeping the single-column index is harmless (a few extra kilobytes of disk space) and makes the intent explicit. But if you want to minimize index count, you can drop `idx_tasks_board_id` and rely on the composite index. Either way, **`EXPLAIN QUERY PLAN` is your guide.** Run it and check.

### Step 2 -- Debug Endpoint with EXPLAIN QUERY PLAN

During development, it is invaluable to see query plans without opening a SQLite shell. Let's add a debug route that shows the plan for our most common queries.

First, add a function to `db.gleam` that runs `EXPLAIN QUERY PLAN`:

```gleam
// src/teamwork/db.gleam

import gleam/dynamic/decode
import gleam/int
import gleam/string

/// Run EXPLAIN QUERY PLAN on a SQL statement and return the
/// plan as a list of text lines.
pub fn explain_query_plan(
  db: sqlight.Connection,
  sql: String,
) -> Result(List(String), sqlight.Error) {
  let explain_sql = "EXPLAIN QUERY PLAN " <> sql
  sqlight.query(
    explain_sql,
    on: db,
    with: [],
    expecting: explain_decoder(),
  )
}

/// Decode an EXPLAIN QUERY PLAN row.
/// Columns: id, parent, notused, detail
fn explain_decoder() -> decode.Decoder(String) {
  use _id <- decode.field(0, decode.int)
  use _parent <- decode.field(1, decode.int)
  use _notused <- decode.field(2, decode.int)
  use detail <- decode.field(3, decode.string)
  decode.success(detail)
}
```

`EXPLAIN QUERY PLAN` returns rows with four columns: `id`, `parent`, `notused`, and `detail`. The `detail` column is the human-readable plan step. We decode that and return a list of plan lines.

Now add a route handler:

```gleam
// src/teamwork/router.gleam

import gleam/list
import gleam/string
import lustre/attribute
import lustre/element
import lustre/element/html
import teamwork/db
import wisp

fn handle_debug_queries(
  _req: wisp.Request,
  ctx: Context,
) -> wisp.Response {
  // Queries to explain
  let queries = [
    #(
      "Tasks for a board (sorted by date)",
      "SELECT id, title, done, created_at FROM tasks
       WHERE board_id = 'board-1'
       ORDER BY created_at DESC",
    ),
    #(
      "Tasks filtered by status",
      "SELECT id, title, done FROM tasks
       WHERE board_id = 'board-1' AND done = 0",
    ),
    #(
      "Board listing with task counts",
      "SELECT b.id, b.name, COUNT(t.id)
       FROM boards b LEFT JOIN tasks t ON t.board_id = b.id
       GROUP BY b.id",
    ),
    #(
      "Search with LIKE (old pattern)",
      "SELECT id, title FROM tasks
       WHERE board_id = 'board-1' AND title LIKE '%design%'",
    ),
  ]

  let sections =
    list.map(queries, fn(entry) {
      let #(label, sql) = entry
      let plan = case db.explain_query_plan(ctx.db, sql) {
        Ok(lines) -> string.join(lines, "\n")
        Error(_) -> "Error running EXPLAIN"
      }

      html.div([attribute.class("debug-section")], [
        html.h3([], [element.text(label)]),
        html.pre(
          [attribute.class("debug-sql")],
          [element.text(sql)],
        ),
        html.pre(
          [attribute.class("debug-plan")],
          [element.text(plan)],
        ),
      ])
    })

  let page =
    html.div([attribute.class("debug-page")], [
      html.h1([], [element.text("Query Plans")]),
      html.p([], [
        element.text(
          "These are the EXPLAIN QUERY PLAN results for common queries. "
          <> "Look for SCAN (bad) vs SEARCH (good).",
        ),
      ]),
      ..sections
    ])

  wisp.html_response(element.to_string(page), 200)
}
```

Register the route in your router:

```gleam
fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  case wisp.path_segments(req) {
    // ... existing routes ...
    ["debug", "queries"] -> handle_debug_queries(req, ctx)
    _ -> wisp.not_found()
  }
}
```

Visit `/debug/queries` during development and you will see the plan for each query. Before adding the indexes from Step 1, you would see `SCAN tasks` everywhere. After adding them, you should see `SEARCH tasks USING INDEX` on every query except the `LIKE` search (which we will fix in Step 5).

This endpoint should be disabled in production. You could check an environment variable:

```gleam
["debug", "queries"] -> case ctx.debug_mode {
  True -> handle_debug_queries(req, ctx)
  False -> wisp.not_found()
}
```

Or simply remove the route before deploying. Either way, it is a useful development tool.

### Step 3 -- SQL-Based Offset Pagination

In [Chapter 16](../03-advanced/16-extensions-and-patterns.md), the infinite scroll handler loaded all tasks and sliced them in memory. Let's replace that with proper SQL pagination.

Update `db.gleam` with a paginated query:

```gleam
// src/teamwork/db.gleam

/// Fetch a page of tasks for a board using OFFSET pagination.
/// Returns the tasks for the requested page.
pub fn list_tasks_page(
  db: sqlight.Connection,
  board_id: String,
  page: Int,
  page_size: Int,
) -> Result(List(Task), sqlight.Error) {
  let offset = page * page_size

  let sql =
    "SELECT id, title, description, done, created_at
     FROM tasks
     WHERE board_id = ?1
     ORDER BY created_at DESC
     LIMIT ?2 OFFSET ?3"

  sqlight.query(
    sql,
    on: db,
    with: [
      sqlight.text(board_id),
      sqlight.int(page_size),
      sqlight.int(offset),
    ],
    expecting: task_with_date_decoder(),
  )
}

/// Decoder that includes created_at for pagination.
fn task_with_date_decoder() -> decode.Decoder(Task) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  use _created_at <- decode.field(4, decode.string)
  decode.success(Task(
    id: id,
    title: title,
    description: description,
    done: done,
  ))
}
```

We request `page_size` items. If we get fewer than `page_size` results, we know there are no more pages. This avoids a separate `COUNT(*)` query.

Now update the handler from [Chapter 16](../03-advanced/16-extensions-and-patterns.md):

```gleam
// src/teamwork/router.gleam

import gleam/int
import gleam/list
import gleam/result
import lustre/attribute
import lustre/element
import lustre/element/html
import teamwork/db
import hx

fn handle_tasks(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let page =
    list.key_find(query_params, "page")
    |> result.try(int.parse)
    |> result.unwrap(0)

  let page_size = 20

  let assert Ok(tasks) =
    db.list_tasks_page(ctx.db, board_id, page, page_size)

  let has_more = list.length(tasks) == page_size

  let body = task_list_page(tasks, board_id, page, has_more)

  wisp.html_response(element.to_string(body), 200)
}
```

The `task_list_page` rendering function is the same sentinel-based pattern from [Chapter 16](../03-advanced/16-extensions-and-patterns.md), but now the data comes from a SQL query that only fetches the rows we need:

```gleam
fn task_list_page(
  tasks: List(db.Task),
  board_id: String,
  page: Int,
  has_more: Bool,
) -> element.Element(t) {
  element.fragment(
    list.append(
      list.map(tasks, fn(task) { task_item(task) }),
      [
        case has_more {
          True ->
            html.div(
              [
                attribute.class("loading-sentinel"),
                hx.get(
                  "/boards/"
                  <> board_id
                  <> "/tasks?page="
                  <> int.to_string(page + 1),
                ),
                hx.trigger([hx.revealed()]),
                hx.swap(hx.OuterHTML),
              ],
              [
                html.span(
                  [attribute.class("spinner")],
                  [element.text("Loading...")],
                ),
              ],
            )

          False ->
            html.div(
              [attribute.class("end-of-list")],
              [element.text("You have reached the end.")],
            )
        },
      ],
    ),
  )
}
```

The view code is nearly identical to [Chapter 16](../03-advanced/16-extensions-and-patterns.md). The only change is underneath: instead of loading all tasks and slicing in Gleam, we load exactly the rows we need from SQLite. The user sees the same infinite scroll experience. The server does far less work.

Compare the before and after:

| Aspect            | [Chapter 16](../03-advanced/16-extensions-and-patterns.md) (in-memory)                  | [Chapter 27](27-database-performance.md) (SQL)                       |
|-------------------|------------------------------------------|----------------------------------------|
| **DB query**      | `SELECT * FROM tasks WHERE board_id=?1` | `SELECT ... LIMIT 20 OFFSET 40`       |
| **Rows loaded**   | All tasks in the board                   | Only 20 rows                           |
| **Memory usage**  | Grows with total tasks                   | Constant (always 20 Task structs)      |
| **Page 1 speed**  | Fast (small dataset)                     | Fast                                   |
| **Page 500 speed**| Fast (slicing is cheap in memory)        | Slower (OFFSET 10000 scans)            |

That last row is the deep-page problem. For Teamwork's scale, it is irrelevant. But let's solve it anyway in Step 4.

### Step 4 -- Cursor-Based Pagination

Cursor-based pagination replaces the `page` parameter with a `cursor` parameter -- the `created_at` value of the last item on the previous page. The query becomes:

```gleam
// src/teamwork/db.gleam

/// Fetch a page of tasks using cursor-based (keyset) pagination.
/// The cursor is the created_at timestamp of the last item from
/// the previous page. Pass an empty string for the first page.
pub fn list_tasks_cursor(
  db: sqlight.Connection,
  board_id: String,
  cursor: String,
  page_size: Int,
) -> Result(List(TaskWithDate), sqlight.Error) {
  let sql = case cursor {
    "" ->
      "SELECT id, title, description, done, created_at
       FROM tasks
       WHERE board_id = ?1
       ORDER BY created_at DESC
       LIMIT ?2"
    _ ->
      "SELECT id, title, description, done, created_at
       FROM tasks
       WHERE board_id = ?1 AND created_at < ?2
       ORDER BY created_at DESC
       LIMIT ?3"
  }

  let params = case cursor {
    "" -> [sqlight.text(board_id), sqlight.int(page_size)]
    _ -> [
      sqlight.text(board_id),
      sqlight.text(cursor),
      sqlight.int(page_size),
    ]
  }

  sqlight.query(sql, on: db, with: params, expecting: task_with_date_decoder())
}

/// Task type that includes created_at for cursor tracking.
pub type TaskWithDate {
  TaskWithDate(
    id: String,
    title: String,
    description: String,
    done: Bool,
    created_at: String,
  )
}

fn task_with_date_decoder() -> decode.Decoder(TaskWithDate) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  use created_at <- decode.field(4, decode.string)
  decode.success(TaskWithDate(
    id: id,
    title: title,
    description: description,
    done: done,
    created_at: created_at,
  ))
}
```

There are two SQL variants: one for the first page (no cursor, no `created_at` filter) and one for subsequent pages (with cursor). The first page just fetches the newest `page_size` tasks. Subsequent pages fetch tasks older than the cursor.

The handler changes to pass the cursor instead of a page number:

```gleam
fn handle_tasks_cursor(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let cursor =
    list.key_find(query_params, "cursor")
    |> result.unwrap("")

  let page_size = 20

  let assert Ok(tasks) =
    db.list_tasks_cursor(ctx.db, board_id, cursor, page_size)

  let has_more = list.length(tasks) == page_size

  // The next cursor is the created_at of the last task in this batch
  let next_cursor = case list.last(tasks) {
    Ok(last_task) -> last_task.created_at
    Error(_) -> ""
  }

  let body =
    task_list_page_cursor(tasks, board_id, next_cursor, has_more)

  wisp.html_response(element.to_string(body), 200)
}
```

The rendering function is similar to Step 3, but the sentinel's URL uses the cursor instead of a page number:

```gleam
fn task_list_page_cursor(
  tasks: List(db.TaskWithDate),
  board_id: String,
  next_cursor: String,
  has_more: Bool,
) -> element.Element(t) {
  element.fragment(
    list.append(
      list.map(tasks, fn(task) { task_item_from_dated(task) }),
      [
        case has_more {
          True ->
            html.div(
              [
                attribute.class("loading-sentinel"),
                hx.get(
                  "/boards/"
                  <> board_id
                  <> "/tasks?cursor="
                  <> next_cursor,
                ),
                hx.trigger([hx.revealed()]),
                hx.swap(hx.OuterHTML),
              ],
              [
                html.span(
                  [attribute.class("spinner")],
                  [element.text("Loading...")],
                ),
              ],
            )

          False ->
            html.div(
              [attribute.class("end-of-list")],
              [element.text("You have reached the end.")],
            )
        },
      ],
    ),
  )
}

fn task_item_from_dated(task: db.TaskWithDate) -> element.Element(t) {
  task_item(db.Task(
    id: task.id,
    title: task.title,
    description: task.description,
    done: task.done,
  ))
}
```

From the user's perspective, nothing changes -- they scroll down, more tasks load. But behind the scenes, every page loads in constant time regardless of how deep they scroll. SQLite uses the composite index `idx_tasks_board_id_created_at` to seek directly to the cursor position and read forward.

There is one subtlety with cursor-based pagination: **ties.** If two tasks have the same `created_at` value, the cursor `created_at < ?2` might skip one of them. The solution is to use a tiebreaker -- typically the primary key:

```sql
SELECT id, title, description, done, created_at
FROM tasks
WHERE board_id = ?1
  AND (created_at < ?2 OR (created_at = ?2 AND id < ?3))
ORDER BY created_at DESC, id DESC
LIMIT ?4
```

This uses `(created_at, id)` as a composite cursor. If two rows share the same timestamp, the `id` comparison breaks the tie. For the Teamwork board, where tasks are created by humans (not bulk-inserted), timestamp ties are extremely rare. But if you are building a system where ties are common (log entries, automated insertions), the tiebreaker is essential.

### Step 5 -- FTS5 Full-Text Search

Now let's upgrade the search from `LIKE` to FTS5. This is a three-part change: migration, query function, and handler update.

**Part 1: Migration**

Add the FTS5 virtual table and sync triggers to the `migrate` function:

```gleam
// src/teamwork/db.gleam

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

    -- Indexes
    CREATE INDEX IF NOT EXISTS idx_tasks_board_id
      ON tasks(board_id);

    CREATE INDEX IF NOT EXISTS idx_tasks_board_id_created_at
      ON tasks(board_id, created_at DESC);

    CREATE INDEX IF NOT EXISTS idx_tasks_board_id_done
      ON tasks(board_id, done);

    -- Full-text search (FTS5)
    CREATE VIRTUAL TABLE IF NOT EXISTS tasks_fts USING fts5(
      title,
      description,
      content=tasks,
      content_rowid=rowid
    );

    -- Sync triggers: keep FTS index in sync with tasks table
    CREATE TRIGGER IF NOT EXISTS tasks_ai AFTER INSERT ON tasks BEGIN
      INSERT INTO tasks_fts(rowid, title, description)
      VALUES (new.rowid, new.title, new.description);
    END;

    CREATE TRIGGER IF NOT EXISTS tasks_ad AFTER DELETE ON tasks BEGIN
      INSERT INTO tasks_fts(tasks_fts, rowid, title, description)
      VALUES('delete', old.rowid, old.title, old.description);
    END;

    CREATE TRIGGER IF NOT EXISTS tasks_au AFTER UPDATE ON tasks BEGIN
      INSERT INTO tasks_fts(tasks_fts, rowid, title, description)
      VALUES('delete', old.rowid, old.title, old.description);
      INSERT INTO tasks_fts(rowid, title, description)
      VALUES (new.rowid, new.title, new.description);
    END;
  ")
}
```

A note on existing data: if you already have tasks in the database from previous chapters, the FTS index will be empty -- triggers only fire for new inserts, updates, and deletes. To populate the FTS index with existing data, run a one-time rebuild:

```gleam
/// Rebuild the FTS index from the tasks table.
/// Run once after adding FTS to an existing database.
pub fn rebuild_fts(db: sqlight.Connection) -> Result(Nil, sqlight.Error) {
  sqlight.exec(db, "
    INSERT INTO tasks_fts(tasks_fts) VALUES('rebuild');
  ")
}
```

The `INSERT INTO tasks_fts(tasks_fts) VALUES('rebuild')` command is another FTS5 special command. It tells FTS5 to read every row from the content table and rebuild the entire index. Call this once during migration when upgrading an existing database, or call it after bulk imports.

**Part 2: Query Function**

Add a full-text search function to `db.gleam`:

```gleam
// src/teamwork/db.gleam

/// Search tasks using FTS5 full-text search.
/// The query supports FTS5 syntax: terms, phrases ("exact phrase"),
/// and boolean operators (AND, OR, NOT).
pub fn search_tasks_fts(
  db: sqlight.Connection,
  board_id: String,
  query: String,
  page_size: Int,
) -> Result(List(Task), sqlight.Error) {
  // Sanitize the query: escape special FTS5 characters
  let safe_query = sanitize_fts_query(query)

  case safe_query {
    "" -> list_tasks_page(db, board_id, 0, page_size)
    _ -> {
      let sql =
        "SELECT t.id, t.title, t.description, t.done
         FROM tasks t
         JOIN tasks_fts fts ON t.rowid = fts.rowid
         WHERE tasks_fts MATCH ?1 AND t.board_id = ?2
         ORDER BY bm25(tasks_fts)
         LIMIT ?3"

      sqlight.query(
        sql,
        on: db,
        with: [
          sqlight.text(safe_query),
          sqlight.text(board_id),
          sqlight.int(page_size),
        ],
        expecting: task_decoder(),
      )
    }
  }
}

/// Sanitize user input for FTS5 queries.
/// FTS5 has special syntax characters that could cause errors
/// or unexpected behavior if passed through directly.
fn sanitize_fts_query(query: String) -> String {
  query
  |> string.trim
  |> escape_fts_special_chars
}

fn escape_fts_special_chars(query: String) -> String {
  // Wrap each word in double quotes to treat them as literals.
  // This prevents FTS5 syntax characters (*, -, ^, etc.)
  // from being interpreted as operators.
  query
  |> string.split(" ")
  |> list.filter(fn(word) { word != "" })
  |> list.map(fn(word) { "\"" <> word <> "\"" })
  |> string.join(" ")
}
```

The `sanitize_fts_query` function deserves attention. FTS5 has its own query syntax with special characters: `*` for prefix matching, `-` for exclusion, `^` for boosting, `NEAR` for proximity, and more. If you pass raw user input directly to `MATCH`, a user could craft a query that causes unexpected behavior or errors.

The simplest defense is to wrap each search term in double quotes, which tells FTS5 to treat the term as a literal string. The query `design "meeting notes"` becomes `"design" "meeting" "notes"` -- each word is matched literally. This sacrifices some FTS5 power (no boolean operators for end users), but it is safe and predictable.

If you want to expose FTS5 operators to users (for example, a "power search" mode), you would need more sophisticated sanitization -- allowing `AND`, `OR`, and quoted phrases while rejecting dangerous constructs.

**Part 3: Handler Update**

Update the search handler from [Chapter 10](../02-intermediate/10-lists-filters-search.md) to use FTS5:

```gleam
// src/teamwork/router.gleam

fn handle_task_search(
  req: wisp.Request,
  ctx: Context,
  board_id: String,
) -> wisp.Response {
  let query_params = wisp.get_query(req)

  let search_query =
    list.key_find(query_params, "q")
    |> result.unwrap("")

  let status_filter =
    list.key_find(query_params, "status")
    |> result.unwrap("all")

  let page_size = 20

  // Use FTS5 search when there is a query, otherwise list all
  let assert Ok(tasks) = case search_query {
    "" -> db.list_tasks_page(ctx.db, board_id, 0, page_size)
    _ -> db.search_tasks_fts(ctx.db, board_id, search_query, page_size)
  }

  // Apply status filter in Gleam (the FTS query already limits results)
  let filtered_tasks = case status_filter {
    "active" -> list.filter(tasks, fn(t) { !t.done })
    "done" -> list.filter(tasks, fn(t) { t.done })
    _ -> tasks
  }

  let body = task_list_fragment(filtered_tasks)

  wisp.html_response(element.to_string(body), 200)
}
```

Wait -- why are we filtering by status in Gleam instead of SQL? Good question. We *could* add a `done` filter to the FTS query:

```sql
WHERE tasks_fts MATCH ?1 AND t.board_id = ?2 AND t.done = ?3
```

But this requires different SQL for the "all" case (no `done` filter) versus the "active"/"done" cases. You could handle that with conditional SQL building:

```gleam
pub fn search_tasks_fts_filtered(
  db: sqlight.Connection,
  board_id: String,
  query: String,
  status: String,
  page_size: Int,
) -> Result(List(Task), sqlight.Error) {
  let safe_query = sanitize_fts_query(query)

  let #(sql, params) = case safe_query, status {
    "", "all" -> #(
      "SELECT id, title, description, done
       FROM tasks WHERE board_id = ?1
       ORDER BY created_at DESC LIMIT ?2",
      [sqlight.text(board_id), sqlight.int(page_size)],
    )

    "", "active" -> #(
      "SELECT id, title, description, done
       FROM tasks WHERE board_id = ?1 AND done = 0
       ORDER BY created_at DESC LIMIT ?2",
      [sqlight.text(board_id), sqlight.int(page_size)],
    )

    "", "done" -> #(
      "SELECT id, title, description, done
       FROM tasks WHERE board_id = ?1 AND done = 1
       ORDER BY created_at DESC LIMIT ?2",
      [sqlight.text(board_id), sqlight.int(page_size)],
    )

    _, "all" -> #(
      "SELECT t.id, t.title, t.description, t.done
       FROM tasks t
       JOIN tasks_fts fts ON t.rowid = fts.rowid
       WHERE tasks_fts MATCH ?1 AND t.board_id = ?2
       ORDER BY bm25(tasks_fts) LIMIT ?3",
      [sqlight.text(safe_query), sqlight.text(board_id),
       sqlight.int(page_size)],
    )

    _, _ -> {
      let done_val = case status {
        "done" -> 1
        _ -> 0
      }
      #(
        "SELECT t.id, t.title, t.description, t.done
         FROM tasks t
         JOIN tasks_fts fts ON t.rowid = fts.rowid
         WHERE tasks_fts MATCH ?1 AND t.board_id = ?2
           AND t.done = ?3
         ORDER BY bm25(tasks_fts) LIMIT ?4",
        [sqlight.text(safe_query), sqlight.text(board_id),
         sqlight.int(done_val), sqlight.int(page_size)],
      )
    }
  }

  sqlight.query(sql, on: db, with: params, expecting: task_decoder())
}
```

This is more code, but it pushes all filtering to the database. For small result sets, either approach works. For large result sets, the SQL-based approach is better because the database can limit results *before* sending them to your application.

The search box from [Chapter 10](../02-intermediate/10-lists-filters-search.md) requires no changes on the client side. It still sends `GET /boards/:id/tasks?q=design&status=active` with debounced keystrokes. The only change is server-side: the query goes to FTS5 instead of `LIKE`.

Let's look at the difference:

```
-- Chapter 10 (LIKE):
SELECT id, title, done FROM tasks
WHERE board_id = ?1 AND title LIKE '%design%'

  QUERY PLAN:
  `--SCAN tasks

-- Chapter 27 (FTS5):
SELECT t.id, t.title, t.description, t.done
FROM tasks t
JOIN tasks_fts fts ON t.rowid = fts.rowid
WHERE tasks_fts MATCH '"design"' AND t.board_id = ?2
ORDER BY bm25(tasks_fts)

  QUERY PLAN:
  |--SCAN tasks_fts VIRTUAL TABLE INDEX 0:M1
  `--SEARCH tasks USING INTEGER PRIMARY KEY (rowid=?)
```

The FTS5 plan shows a virtual table scan (which is actually an indexed lookup inside the FTS data structure) followed by a lookup in the main table to fetch the non-FTS columns. This is fast even for hundreds of thousands of rows.

And the search quality improves too:

| Query       | LIKE result                    | FTS5 result                          |
|-------------|--------------------------------|--------------------------------------|
| `design`    | Only titles containing "design" | Titles and descriptions with "design" |
| `meeting notes` | Nothing (exact substring) | Tasks matching "meeting" AND "notes" |
| `task`      | "task", "tasks", "multitask"   | "task" (exact word boundary)         |
| `des`       | "design", "description", "desk"| Nothing (FTS5 matches whole words)   |

The last row is important: FTS5 matches whole words by default, not substrings. If you want prefix matching (so "des" matches "design"), append `*` to the query term: `"des"*`. You could add this in the sanitization function:

```gleam
fn escape_fts_special_chars(query: String) -> String {
  query
  |> string.split(" ")
  |> list.filter(fn(word) { word != "" })
  |> list.map(fn(word) { "\"" <> word <> "\"" <> "*" })
  |> string.join(" ")
}
```

Now `des` becomes `"des"*`, which matches "design", "description", and "desk" via prefix expansion. This gives you the "search-as-you-type" behavior that users expect from a live search box.

### Step 6 -- Fix N+1 in Board Listing with JOIN

The Teamwork board listing page shows all boards. In earlier chapters, we might have been tempted to show the number of tasks in each board. Here is the N+1 version that we want to avoid:

```gleam
// BAD: N+1 query pattern
pub fn list_boards_with_counts_n_plus_one(
  db: sqlight.Connection,
) -> Result(List(BoardWithCount), sqlight.Error) {
  // Query 1: fetch all boards
  use boards <- result.try(list_boards(db))

  // Queries 2..N+1: one per board
  list.try_map(boards, fn(board) {
    let sql = "SELECT COUNT(*) FROM tasks WHERE board_id = ?1"
    use rows <- result.try(
      sqlight.query(
        sql,
        on: db,
        with: [sqlight.text(board.id)],
        expecting: {
          use count <- decode.field(0, decode.int)
          decode.success(count)
        },
      ),
    )
    let assert [count] = rows
    Ok(BoardWithCount(board: board, task_count: count))
  })
}
```

If there are 20 boards, this runs 21 queries. Let's fix it with a single JOIN query:

```gleam
// src/teamwork/db.gleam

pub type BoardWithCount {
  BoardWithCount(
    id: String,
    name: String,
    description: String,
    task_count: Int,
  )
}

/// Fetch all boards with their task counts in a single query.
pub fn list_boards_with_counts(
  db: sqlight.Connection,
) -> Result(List(BoardWithCount), sqlight.Error) {
  let sql =
    "SELECT b.id, b.name, b.description, COUNT(t.id) AS task_count
     FROM boards b
     LEFT JOIN tasks t ON t.board_id = b.id
     GROUP BY b.id
     ORDER BY b.created_at DESC"

  sqlight.query(
    sql,
    on: db,
    with: [],
    expecting: board_with_count_decoder(),
  )
}

fn board_with_count_decoder() -> decode.Decoder(BoardWithCount) {
  use id <- decode.field(0, decode.string)
  use name <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use task_count <- decode.field(3, decode.int)
  decode.success(BoardWithCount(
    id: id,
    name: name,
    description: description,
    task_count: task_count,
  ))
}
```

One query, no matter how many boards. The `LEFT JOIN` ensures boards with zero tasks still appear (the `COUNT(t.id)` returns 0 for them because there are no matching task rows to count). The `GROUP BY b.id` groups all matching task rows per board so that `COUNT` produces one number per board.

Let's use this in the board listing handler:

```gleam
fn handle_boards_list(
  _req: wisp.Request,
  ctx: Context,
) -> wisp.Response {
  let assert Ok(boards) = db.list_boards_with_counts(ctx.db)

  let body =
    html.div([attribute.id("boards-list")], [
      html.h2([], [element.text("Your Boards")]),
      ..list.map(boards, fn(board) {
        html.div([attribute.class("board-card")], [
          html.a(
            [
              attribute.href("/boards/" <> board.id),
              hx.boost(True),
            ],
            [
              html.h3([], [element.text(board.name)]),
            ],
          ),
          html.p(
            [attribute.class("board-description")],
            [element.text(board.description)],
          ),
          html.span(
            [attribute.class("task-count")],
            [
              element.text(
                int.to_string(board.task_count) <> " tasks",
              ),
            ],
          ),
        ])
      })
    ])

  wisp.html_response(element.to_string(body), 200)
}
```

Each board card shows the board name, description, and task count. The data comes from a single database round-trip. The `EXPLAIN QUERY PLAN` for this query shows:

```
QUERY PLAN
|--SCAN boards USING INDEX sqlite_autoindex_boards_1
|--SEARCH tasks USING INDEX idx_tasks_board_id (board_id=?)
`--USE TEMP B-TREE FOR GROUP BY
```

SQLite scans the boards table (which is small), uses our `idx_tasks_board_id` index to efficiently find tasks for each board, and uses a temporary B-tree for the `GROUP BY` aggregation. This is about as efficient as it gets.

You can take this pattern further. If the board listing also needs to show the most recent task for each board, you might reach for a subquery:

```gleam
pub fn list_boards_with_latest_task(
  db: sqlight.Connection,
) -> Result(List(BoardSummary), sqlight.Error) {
  let sql =
    "SELECT b.id, b.name, b.description,
            COUNT(t.id) AS task_count,
            (SELECT title FROM tasks
             WHERE board_id = b.id
             ORDER BY created_at DESC LIMIT 1) AS latest_task
     FROM boards b
     LEFT JOIN tasks t ON t.board_id = b.id
     GROUP BY b.id
     ORDER BY b.created_at DESC"

  sqlight.query(sql, on: db, with: [], expecting: board_summary_decoder())
}
```

The correlated subquery `(SELECT title FROM tasks WHERE board_id = b.id ORDER BY created_at DESC LIMIT 1)` runs once per board, but it is using the `idx_tasks_board_id_created_at` index so each execution is a single index seek. This is still far better than an N+1 pattern in application code.

### Step 7 -- Enable WAL Mode in Database Initialization

Finally, update the database initialization to enable WAL mode and other important pragmas:

```gleam
// src/teamwork/db.gleam

import gleam/result

/// Initialize the database: open connection, set pragmas, run migrations.
pub fn initialize(path: String) -> Result(sqlight.Connection, sqlight.Error) {
  use db <- result.try(sqlight.open(path))

  // Enable WAL mode for concurrent read/write performance.
  // This persists across connections -- only needs to run once per database,
  // but it is safe to run every time.
  use _ <- result.try(sqlight.exec(db, "PRAGMA journal_mode=WAL;"))

  // Enable foreign key enforcement.
  // MUST run on every new connection -- this is NOT persistent.
  use _ <- result.try(sqlight.exec(db, "PRAGMA foreign_keys=ON;"))

  // Wait up to 5 seconds if the database is locked by another writer.
  // Prevents spurious "database is locked" errors during brief contention.
  use _ <- result.try(sqlight.exec(db, "PRAGMA busy_timeout=5000;"))

  // Run schema migrations.
  use _ <- result.try(migrate(db))

  Ok(db)
}
```

And update `main` to use the new `initialize` function:

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

  // Database setup: open, set pragmas, migrate
  let assert Ok(db) = db.initialize(db_path)

  let debug_mode =
    envoy.get("DEBUG")
    |> result.map(fn(v) { v == "true" || v == "1" })
    |> result.unwrap(False)

  let ctx = router.Context(
    db: db,
    secret_key_base: secret_key_base,
    debug_mode: debug_mode,
  )

  let assert Ok(_) =
    wisp_mist.handler(router.handle_request(_, ctx), secret_key_base)
    |> mist.new
    |> mist.port(port)
    |> mist.start

  wisp.log_info("Teamwork started on port " <> int.to_string(port))

  process.sleep_forever()
}
```

The initialization sequence is deliberate:

1. **Open the connection.** `sqlight.open` creates the file if it does not exist.
2. **Set WAL mode.** This changes the journal mode for the database file. It persists across connections, but running it every time is harmless and ensures new databases start with WAL mode.
3. **Enable foreign keys.** This does *not* persist. It must be set on every new connection. Forgetting it means your `ON DELETE CASCADE` and `REFERENCES` constraints silently do nothing.
4. **Set busy timeout.** When two concurrent requests try to write at the same time, one will wait up to 5 seconds instead of immediately failing. This is critical for HTMX applications where multiple swaps can trigger concurrent writes.
5. **Run migrations.** Create tables, indexes, and FTS virtual tables.

---

## 3. Full Code Listing

Here is the complete updated `db.gleam` module with all the changes from this chapter:

```gleam
// src/teamwork/db.gleam

import gleam/dynamic/decode
import gleam/list
import gleam/result
import gleam/string
import sqlight

// --------------- Types ---------------

pub type Task {
  Task(id: String, title: String, description: String, done: Bool)
}

pub type TaskWithDate {
  TaskWithDate(
    id: String,
    title: String,
    description: String,
    done: Bool,
    created_at: String,
  )
}

pub type Board {
  Board(id: String, name: String, description: String)
}

pub type BoardWithCount {
  BoardWithCount(
    id: String,
    name: String,
    description: String,
    task_count: Int,
  )
}

// --------------- Initialization ---------------

/// Initialize the database: open connection, set pragmas, run migrations.
pub fn initialize(path: String) -> Result(sqlight.Connection, sqlight.Error) {
  use db <- result.try(sqlight.open(path))
  use _ <- result.try(sqlight.exec(db, "PRAGMA journal_mode=WAL;"))
  use _ <- result.try(sqlight.exec(db, "PRAGMA foreign_keys=ON;"))
  use _ <- result.try(sqlight.exec(db, "PRAGMA busy_timeout=5000;"))
  use _ <- result.try(migrate(db))
  Ok(db)
}

/// Run migrations to create the database schema.
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

    -- Indexes
    CREATE INDEX IF NOT EXISTS idx_tasks_board_id
      ON tasks(board_id);

    CREATE INDEX IF NOT EXISTS idx_tasks_board_id_created_at
      ON tasks(board_id, created_at DESC);

    CREATE INDEX IF NOT EXISTS idx_tasks_board_id_done
      ON tasks(board_id, done);

    -- Full-text search
    CREATE VIRTUAL TABLE IF NOT EXISTS tasks_fts USING fts5(
      title,
      description,
      content=tasks,
      content_rowid=rowid
    );

    CREATE TRIGGER IF NOT EXISTS tasks_ai AFTER INSERT ON tasks BEGIN
      INSERT INTO tasks_fts(rowid, title, description)
      VALUES (new.rowid, new.title, new.description);
    END;

    CREATE TRIGGER IF NOT EXISTS tasks_ad AFTER DELETE ON tasks BEGIN
      INSERT INTO tasks_fts(tasks_fts, rowid, title, description)
      VALUES('delete', old.rowid, old.title, old.description);
    END;

    CREATE TRIGGER IF NOT EXISTS tasks_au AFTER UPDATE ON tasks BEGIN
      INSERT INTO tasks_fts(tasks_fts, rowid, title, description)
      VALUES('delete', old.rowid, old.title, old.description);
      INSERT INTO tasks_fts(rowid, title, description)
      VALUES (new.rowid, new.title, new.description);
    END;
  ")
}

/// Rebuild the FTS index from the tasks table.
pub fn rebuild_fts(db: sqlight.Connection) -> Result(Nil, sqlight.Error) {
  sqlight.exec(db, "INSERT INTO tasks_fts(tasks_fts) VALUES('rebuild');")
}

// --------------- Decoders ---------------

fn task_decoder() -> decode.Decoder(Task) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  decode.success(Task(id: id, title: title, description: description, done: done))
}

fn task_with_date_decoder() -> decode.Decoder(TaskWithDate) {
  use id <- decode.field(0, decode.string)
  use title <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use done <- decode.field(3, sqlight.decode_bool())
  use created_at <- decode.field(4, decode.string)
  decode.success(TaskWithDate(
    id: id,
    title: title,
    description: description,
    done: done,
    created_at: created_at,
  ))
}

fn board_with_count_decoder() -> decode.Decoder(BoardWithCount) {
  use id <- decode.field(0, decode.string)
  use name <- decode.field(1, decode.string)
  use description <- decode.field(2, decode.string)
  use task_count <- decode.field(3, decode.int)
  decode.success(BoardWithCount(
    id: id,
    name: name,
    description: description,
    task_count: task_count,
  ))
}

// --------------- Query Functions ---------------

/// Fetch all tasks for a board, newest first.
pub fn list_tasks(
  db: sqlight.Connection,
  board_id: String,
) -> Result(List(Task), sqlight.Error) {
  let sql =
    "SELECT id, title, description, done
     FROM tasks
     WHERE board_id = ?1
     ORDER BY created_at DESC"

  sqlight.query(sql, on: db, with: [sqlight.text(board_id)], expecting: task_decoder())
}

/// Fetch a page of tasks using OFFSET pagination.
pub fn list_tasks_page(
  db: sqlight.Connection,
  board_id: String,
  page: Int,
  page_size: Int,
) -> Result(List(Task), sqlight.Error) {
  let offset = page * page_size

  let sql =
    "SELECT id, title, description, done, created_at
     FROM tasks
     WHERE board_id = ?1
     ORDER BY created_at DESC
     LIMIT ?2 OFFSET ?3"

  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(board_id), sqlight.int(page_size), sqlight.int(offset)],
    expecting: {
      use id <- decode.field(0, decode.string)
      use title <- decode.field(1, decode.string)
      use description <- decode.field(2, decode.string)
      use done <- decode.field(3, sqlight.decode_bool())
      use _created_at <- decode.field(4, decode.string)
      decode.success(Task(id: id, title: title, description: description, done: done))
    },
  )
}

/// Fetch a page of tasks using cursor-based (keyset) pagination.
pub fn list_tasks_cursor(
  db: sqlight.Connection,
  board_id: String,
  cursor: String,
  page_size: Int,
) -> Result(List(TaskWithDate), sqlight.Error) {
  let #(sql, params) = case cursor {
    "" -> #(
      "SELECT id, title, description, done, created_at
       FROM tasks
       WHERE board_id = ?1
       ORDER BY created_at DESC
       LIMIT ?2",
      [sqlight.text(board_id), sqlight.int(page_size)],
    )
    _ -> #(
      "SELECT id, title, description, done, created_at
       FROM tasks
       WHERE board_id = ?1 AND created_at < ?2
       ORDER BY created_at DESC
       LIMIT ?3",
      [sqlight.text(board_id), sqlight.text(cursor), sqlight.int(page_size)],
    )
  }

  sqlight.query(sql, on: db, with: params, expecting: task_with_date_decoder())
}

/// Search tasks using FTS5 full-text search.
pub fn search_tasks_fts(
  db: sqlight.Connection,
  board_id: String,
  query: String,
  page_size: Int,
) -> Result(List(Task), sqlight.Error) {
  let safe_query = sanitize_fts_query(query)

  case safe_query {
    "" -> list_tasks_page(db, board_id, 0, page_size)
    _ -> {
      let sql =
        "SELECT t.id, t.title, t.description, t.done
         FROM tasks t
         JOIN tasks_fts fts ON t.rowid = fts.rowid
         WHERE tasks_fts MATCH ?1 AND t.board_id = ?2
         ORDER BY bm25(tasks_fts)
         LIMIT ?3"

      sqlight.query(
        sql,
        on: db,
        with: [
          sqlight.text(safe_query),
          sqlight.text(board_id),
          sqlight.int(page_size),
        ],
        expecting: task_decoder(),
      )
    }
  }
}

/// Fetch all boards with their task counts in a single query.
pub fn list_boards_with_counts(
  db: sqlight.Connection,
) -> Result(List(BoardWithCount), sqlight.Error) {
  let sql =
    "SELECT b.id, b.name, b.description, COUNT(t.id) AS task_count
     FROM boards b
     LEFT JOIN tasks t ON t.board_id = b.id
     GROUP BY b.id
     ORDER BY b.created_at DESC"

  sqlight.query(sql, on: db, with: [], expecting: board_with_count_decoder())
}

/// Insert a new task.
pub fn create_task(
  db: sqlight.Connection,
  board_id: String,
  task: Task,
) -> Result(Nil, sqlight.Error) {
  let sql =
    "INSERT INTO tasks (id, board_id, title, description, done)
     VALUES (?1, ?2, ?3, ?4, ?5)"

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

/// Toggle a task's done status.
pub fn toggle_task(
  db: sqlight.Connection,
  task_id: String,
) -> Result(Task, sqlight.Error) {
  let sql =
    "UPDATE tasks SET done = CASE WHEN done = 0 THEN 1 ELSE 0 END
     WHERE id = ?1
     RETURNING id, title, description, done"

  sqlight.query(sql, on: db, with: [sqlight.text(task_id)], expecting: task_decoder())
  |> result.map(fn(rows) {
    let assert [task] = rows
    task
  })
}

/// Delete a task.
pub fn delete_task(
  db: sqlight.Connection,
  task_id: String,
) -> Result(Nil, sqlight.Error) {
  let sql = "DELETE FROM tasks WHERE id = ?1"
  sqlight.query(sql, on: db, with: [sqlight.text(task_id)], expecting: decode.dynamic)
  |> result.map(fn(_) { Nil })
}

/// Run EXPLAIN QUERY PLAN on a SQL statement.
pub fn explain_query_plan(
  db: sqlight.Connection,
  sql: String,
) -> Result(List(String), sqlight.Error) {
  let explain_sql = "EXPLAIN QUERY PLAN " <> sql
  sqlight.query(explain_sql, on: db, with: [], expecting: {
    use _id <- decode.field(0, decode.int)
    use _parent <- decode.field(1, decode.int)
    use _notused <- decode.field(2, decode.int)
    use detail <- decode.field(3, decode.string)
    decode.success(detail)
  })
}

// --------------- Helpers ---------------

fn sanitize_fts_query(query: String) -> String {
  query
  |> string.trim
  |> string.split(" ")
  |> list.filter(fn(word) { word != "" })
  |> list.map(fn(word) { "\"" <> word <> "\"" <> "*" })
  |> string.join(" ")
}
```

---

## 4. Exercises

### Exercise 1: Measure Before and After

**Description:** Create a script or handler that inserts 10,000 test tasks into a board, then measures the time taken by three queries: (a) all tasks for the board without an index, (b) all tasks for the board with the `idx_tasks_board_id` index, (c) tasks filtered by `board_id` and `done` status. Use `timestamp.system_time()` from `gleam/time/timestamp` to measure elapsed time.

**Acceptance criteria:**
- A function inserts 10,000 tasks with realistic titles into a test board.
- Each query is timed using `timestamp.system_time()` and `timestamp.to_unix_seconds_and_nanoseconds()` before and after execution.
- The results are displayed in a debug page that shows the elapsed time for each query.
- The output clearly shows the performance difference between SCAN and SEARCH.
- The function cleans up test data after measurement (deletes the test board and its tasks).

### Exercise 2: Cursor-Based Pagination with Tiebreaker

**Description:** Modify the cursor-based pagination from Step 4 to handle timestamp ties. Use `(created_at, id)` as a composite cursor. Insert several tasks with the same `created_at` value and verify that no tasks are skipped during pagination.

**Acceptance criteria:**
- The SQL query uses `(created_at < ?2 OR (created_at = ?2 AND id < ?3))` as the cursor condition.
- The `ORDER BY` clause is `created_at DESC, id DESC` to match the cursor condition.
- The handler extracts both `cursor_date` and `cursor_id` from query parameters.
- A test with 30 tasks having the same `created_at` value, paginated in groups of 10, returns all 30 tasks without skips or duplicates.
- The sentinel URL includes both cursor components.

### Exercise 3: FTS5 with Prefix and Phrase Search

**Description:** Extend the search handler to support two modes: simple search (the default, matching word prefixes) and phrase search (when the user wraps their query in double quotes). Display a small help text below the search box explaining the syntax.

**Acceptance criteria:**
- A search for `des` matches tasks with "design", "description", and "desk" (prefix matching).
- A search for `"meeting notes"` (with quotes) matches only tasks that contain the exact phrase "meeting notes" adjacent to each other.
- A search for `meeting notes` (without quotes) matches tasks that contain both words, even if they are not adjacent.
- The sanitization function detects user-provided double quotes and preserves them as FTS5 phrase markers.
- A help text element appears below the search input with the text: `Use "quotes" for exact phrases`.

### Exercise 4: Board Activity Dashboard

**Description:** Build a board activity dashboard that shows, for each board: the total task count, the number of active tasks, the number of completed tasks, and the title of the most recently created task. All data must come from a single SQL query using JOINs and subqueries.

**Acceptance criteria:**
- A single SQL query returns all required data (no N+1 pattern).
- Boards with zero tasks show counts of 0 and no latest task.
- The dashboard renders as a grid of board cards with the four data points.
- Each card links to the board using `hx.boost(True)` for SPA-style navigation.
- The `EXPLAIN QUERY PLAN` for the query uses the `idx_tasks_board_id` index.

### Exercise 5: Search Results with Highlighting

**Description:** When displaying FTS5 search results, highlight the matching terms in the task title and description. Use the FTS5 `highlight()` function to wrap matching terms in `<mark>` tags. Render the highlighted text safely in Lustre.

**Acceptance criteria:**
- The SQL query uses `highlight(tasks_fts, 0, '<mark>', '</mark>')` for the title column and `highlight(tasks_fts, 1, '<mark>', '</mark>')` for the description column.
- The highlighted HTML is rendered using `attribute.attribute("dangerous-unescaped-html", ...)` or by injecting raw HTML through a Lustre escape hatch.
- Search results visually highlight matching terms with a yellow background (via CSS styling on `<mark>` elements).
- Non-search task listings do not use highlighting.
- The highlight function handles multiple matches within a single field.

---

## 5. Exercise Solution Hints

### Hint for Exercise 1

Use `gleam/time/timestamp` to measure time:

```gleam
import gleam/time/timestamp

fn timestamp_to_ms(ts: timestamp.Timestamp) -> Int {
  let #(seconds, nanoseconds) =
    timestamp.to_unix_seconds_and_nanoseconds(ts)
  seconds * 1000 + nanoseconds / 1_000_000
}

fn time_query(
  db: sqlight.Connection,
  label: String,
  query_fn: fn() -> Result(a, b),
) -> #(String, Int) {
  let start = timestamp_to_ms(timestamp.system_time())
  let _ = query_fn()
  let elapsed = timestamp_to_ms(timestamp.system_time()) - start
  #(label, elapsed)
}
```

To insert 10,000 test tasks efficiently, use a transaction:

```gleam
sqlight.exec(db, "BEGIN TRANSACTION;")
// ... insert 10,000 tasks ...
sqlight.exec(db, "COMMIT;")
```

Without a transaction, each `INSERT` triggers a separate disk sync. With a transaction, all 10,000 inserts share a single sync -- this is the difference between 30 seconds and 0.5 seconds.

To test without an index, drop it temporarily:

```gleam
sqlight.exec(db, "DROP INDEX IF EXISTS idx_tasks_board_id;")
// ... time the query ...
sqlight.exec(db, "CREATE INDEX idx_tasks_board_id ON tasks(board_id);")
// ... time the query again ...
```

### Hint for Exercise 2

The composite cursor requires two parameters. The URL for the sentinel becomes:

```gleam
"/boards/" <> board_id <> "/tasks?cursor_date="
<> last_task.created_at <> "&cursor_id=" <> last_task.id
```

And the SQL uses:

```sql
WHERE board_id = ?1
  AND (created_at < ?2 OR (created_at = ?2 AND id < ?3))
ORDER BY created_at DESC, id DESC
LIMIT ?4
```

To create test tasks with the same timestamp, insert them within a single second or use an explicit `created_at` value:

```sql
INSERT INTO tasks (id, board_id, title, created_at)
VALUES (?1, ?2, ?3, '2024-06-15 10:00:00');
```

### Hint for Exercise 3

To detect user-provided quotes, check if the entire query is wrapped in double quotes:

```gleam
fn sanitize_fts_query(query: String) -> String {
  let trimmed = string.trim(query)
  case string.starts_with(trimmed, "\""), string.ends_with(trimmed, "\"") {
    True, True ->
      // User wants a phrase search -- pass through as-is
      trimmed
    _, _ ->
      // Default: prefix match on each word
      trimmed
      |> string.split(" ")
      |> list.filter(fn(word) { word != "" })
      |> list.map(fn(word) { "\"" <> word <> "\"" <> "*" })
      |> string.join(" ")
  }
}
```

For the help text:

```gleam
html.small(
  [attribute.class("search-help")],
  [element.text("Use \"quotes\" for exact phrases")],
)
```

### Hint for Exercise 4

The single query with subqueries:

```sql
SELECT b.id, b.name, b.description,
       COUNT(t.id) AS total,
       SUM(CASE WHEN t.done = 0 THEN 1 ELSE 0 END) AS active,
       SUM(CASE WHEN t.done = 1 THEN 1 ELSE 0 END) AS completed,
       (SELECT title FROM tasks
        WHERE board_id = b.id
        ORDER BY created_at DESC LIMIT 1) AS latest_title
FROM boards b
LEFT JOIN tasks t ON t.board_id = b.id
GROUP BY b.id
ORDER BY b.created_at DESC
```

The `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` pattern is a conditional count. It counts only the rows that match the condition. Use `COALESCE(..., 0)` around the `SUM` calls if you want to ensure 0 instead of NULL for boards with no tasks.

The decoder needs to handle the nullable `latest_title`:

```gleam
use latest_title <- decode.field(6, decode.optional(decode.string))
```

### Hint for Exercise 5

The FTS5 `highlight()` function returns HTML strings:

```sql
SELECT t.id,
       highlight(tasks_fts, 0, '<mark>', '</mark>') AS title_hl,
       highlight(tasks_fts, 1, '<mark>', '</mark>') AS desc_hl,
       t.done
FROM tasks t
JOIN tasks_fts fts ON t.rowid = fts.rowid
WHERE tasks_fts MATCH ?1 AND t.board_id = ?2
ORDER BY bm25(tasks_fts)
LIMIT ?3
```

The `highlight()` arguments are: the FTS table name, the column index (0 for title, 1 for description), the opening tag, and the closing tag.

To render raw HTML in Lustre, you can use `element.element` to create a custom element or use `attribute.attribute("dangerous-unescaped-html", highlighted_text)`. The simplest approach is to use a `div` with the `innerHTML` property:

```gleam
html.span(
  [
    attribute.class("search-result-title"),
    attribute.attribute("dangerous-unescaped-html", highlighted_title),
  ],
  [],
)
```

Be cautious with this approach: it injects raw HTML into the page. The `highlight()` function output is safe (it only adds `<mark>` tags), but you should never render user-generated HTML without sanitization. In this case, the HTML comes from FTS5's `highlight()` function, which only wraps matched tokens -- it does not inject arbitrary content.

---

## 6. Key Takeaways

1. **Every HTMX swap is a database round-trip.** Unlike SPAs that cache data on the client, HTMX applications query the database on every interaction. Fast queries are not a nice-to-have -- they are a requirement for a snappy user experience.

2. **Indexes are the single biggest performance win.** A missing index on a `WHERE` column turns an `O(log n)` lookup into an `O(n)` scan. Adding `CREATE INDEX` to your migrations is a one-line change with outsized impact.

3. **`EXPLAIN QUERY PLAN` is your diagnostic tool.** Look for `SCAN` (bad) versus `SEARCH` (good). Look for `USE TEMP B-TREE FOR ORDER BY` (could be better). Make it a habit to check the plan for every new query.

4. **Push pagination into SQL.** In-memory pagination with `list.drop`/`list.take` loads all rows and discards most of them. `LIMIT`/`OFFSET` in SQL loads only the rows you need. For deep pages, cursor-based pagination eliminates the offset penalty entirely.

5. **FTS5 turns SQLite into a search engine.** Replace `LIKE '%query%'` with FTS5 `MATCH` for faster, higher-quality search results. The virtual table with sync triggers adds a few lines to your migration and dramatically improves the search experience.

6. **N+1 queries are a loop calling the database.** If you see a database function called inside `list.map` or `list.try_map`, that is the N+1 pattern. Pull the loop into a single SQL query with JOINs or subqueries.

7. **WAL mode is essential for concurrent access.** Without it, readers block writers and writers block readers. With it, concurrent reads and writes work smoothly. Enable it on every database initialization along with `PRAGMA foreign_keys=ON` and `PRAGMA busy_timeout=5000`.

8. **Measure, do not guess.** The debug endpoint from Step 2 shows query plans during development. The timing approach from Exercise 1 shows actual execution times. Use both before and after making changes so you know the impact is real.

9. **Simple wins compound.** An index saves 5 milliseconds per query. SQL pagination saves 10 milliseconds. FTS5 saves 20 milliseconds. Fixing N+1 saves 50 milliseconds. Add them up and the page that used to take 200 milliseconds now takes 20. The user notices.

---

## What's Next

The Teamwork database layer is now production-ready: indexed, paginated, searchable, and free of N+1 patterns. Every HTMX swap hits a fast query path.

In **[Chapter 28](28-background-jobs.md) -- Background Jobs with BEAM Processes** we will tackle work that does not belong in the request-response cycle: sending email notifications, generating reports, cleaning up stale data, and scheduling recurring maintenance. The BEAM's lightweight processes and OTP supervision trees make this natural -- no external job queue needed.
