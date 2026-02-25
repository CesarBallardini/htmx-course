# Chapter 28 -- Background Jobs with BEAM Processes

**Phase:** Real-World Patterns (final chapter)
**Project:** "Teamwork" -- a collaborative task board
**Previous:** [Chapter 27](27-database-performance.md) covered database performance -- WAL mode, query optimization, indexing strategies, and avoiding N+1 queries. Now we tackle the last major production concern: long-running work that should not block HTTP requests.

---

## Learning Objectives

By the end of this chapter you will be able to:

1. Explain why long-running operations should not block HTTP handlers and describe the user experience consequences of synchronous work.
2. Use `process.start` for fire-and-forget tasks, returning `202 Accepted` immediately.
3. Build a supervised job worker actor that processes a queue with retry logic on failure.
4. Report job progress to the client via polling or SSE.
5. Implement periodic cleanup jobs using `process.send_after` for self-rescheduling.
6. Handle failures with OTP supervision trees and the "let it crash" philosophy.

---

## 1. Theory

### 1.1 The Problem with Synchronous Work

Every HTTP request occupies a BEAM process for as long as it takes to generate a response. For most requests -- fetching a task list, saving a form, rendering a page -- that is a few milliseconds. The user clicks, sees a result, and moves on. Nobody notices.

But some operations take longer. Generating a PDF report from thousands of tasks might take five seconds. Processing an uploaded image into thumbnails might take three seconds. Sending notification emails to fifty team members might take ten seconds if the mail server is slow.

When these operations run inside the HTTP handler, two bad things happen:

**The user waits.** The browser shows a spinner for five, ten, fifteen seconds. The user does not know what is happening. They wonder if something broke. They click the button again, starting a second identical operation. They close the tab. The experience is terrible.

**The server is less responsive.** While a Mist process is blocked generating a PDF, it is not handling other requests. The BEAM scheduler is fair -- it will preempt the process -- but the HTTP connection is still occupied. Under load, this means slower response times for everyone, not just the user who triggered the long operation.

The solution is always the same: **accept the work, respond immediately, do the work later.**

This is not a new idea. Every email system works this way. You click "send" and the email appears in your outbox instantly. The actual delivery happens in the background, possibly minutes later. You do not stare at a spinner waiting for the recipient's mail server to acknowledge receipt.

In web applications, the pattern looks like this:

```
  Client                         Server
  ──────                         ──────
    │                               │
    │  POST /reports/generate       │
    │──────────────────────────────►│
    │                               │  ← Enqueue the job
    │  202 Accepted                 │
    │  { "job_id": "abc123" }       │
    │◄──────────────────────────────│  ← Respond immediately
    │                               │
    │  GET /jobs/abc123/status      │
    │──────────────────────────────►│
    │  { "status": "in_progress",   │
    │    "progress": 42 }           │
    │◄──────────────────────────────│
    │                               │
    │  (poll again, or use SSE)     │
    │                               │
    │  GET /jobs/abc123/status      │
    │──────────────────────────────►│
    │  { "status": "complete",      │
    │    "result": "/reports/r42" } │
    │◄──────────────────────────────│
    │                               │
```

The server accepts the request and returns `202 Accepted` -- an HTTP status code that means "I heard you, I will deal with it, but I am not done yet." The client then checks back periodically (polling) or listens for updates (SSE) until the job finishes.

This is the pattern we will build in this chapter.

### 1.2 The BEAM Advantage

Most web frameworks need external infrastructure for background jobs. Rails applications use Sidekiq backed by Redis. Django applications use Celery backed by RabbitMQ. Node.js applications use Bull backed by Redis. The pattern is always the same: serialize the job description, put it in an external queue, have a separate worker process consume from the queue.

The BEAM is different. Background processing is not an add-on -- it is built into the runtime.

**Lightweight processes.** A BEAM process costs a few hundred bytes of memory to create and a few microseconds to start. You can spawn a million of them on a single machine. Starting a background job is as cheap as calling a function.

**Isolation.** Each process has its own heap and garbage collector. If a background job crashes, it does not corrupt the state of other processes. The HTTP server keeps running. The task actor keeps running. Only the failed job dies.

**Preemptive scheduling.** The BEAM scheduler gives every process a fair slice of CPU time. A background job computing a large report cannot starve HTTP handlers of processing time. The scheduler preempts it after a fixed number of reductions (roughly, function calls) and switches to another process. Your web application stays responsive even while heavy work is running.

**Built-in messaging.** Processes communicate through message passing. There is no need for Redis, RabbitMQ, or any external queue. The "queue" is the process mailbox. The "worker" is an actor processing messages one at a time. The "pub/sub" system is a list of subjects that receive broadcasts.

**Supervision.** The OTP framework provides supervisors -- processes whose only job is to monitor other processes and restart them if they crash. This gives you fault tolerance without writing error-handling code in every function.

This is why Erlang powers telephone switches, WhatsApp, and Discord. The runtime was designed from the ground up for exactly the kind of concurrent, fault-tolerant, background-processing workload we are about to build.

For the Teamwork application, this means we can add background jobs -- report generation, image processing, notification emails, periodic cleanup -- without adding a single new dependency. Everything we need is already in the runtime.

### 1.3 Fire-and-Forget Jobs

The simplest kind of background work is fire-and-forget. You start a process, it does some work, and when it finishes, nobody cares about collecting a result. The process just dies.

Gleam provides this with `process.start`:

```gleam
process.start(fn() {
  // Do the work here
  send_notification_emails(team_members)
}, linked: False)
```

The `linked` parameter is important. When `linked` is `True`, the spawned process is linked to the parent. If either one crashes, both die. This is useful for processes that must live or die together. But for background jobs, we almost always want `linked: False` -- if the job crashes, the HTTP handler should not be affected (and the handler has already returned a response anyway).

The HTTP handler pattern is:

1. Receive the request.
2. Do any validation or quick work.
3. Spawn the background process.
4. Return `202 Accepted`.

The response goes back to the client in milliseconds, regardless of how long the background work takes.

For HTMX, you can pair the `202` response with an `HX-Trigger` response header. This fires a custom event on the client side, which you can use to show a notification, start polling, or update a status indicator:

```gleam
wisp.response(202)
|> wisp.set_header("HX-Trigger", "jobStarted")
```

On the client side, you can listen for this event with `hx-trigger="jobStarted from:body"` on a status element that starts polling for progress.

Fire-and-forget is appropriate when you do not need to report progress or results to the user. Sending emails, logging analytics events, cleaning up temporary files, pre-warming a cache -- these are all good candidates.

### 1.4 Job Queues with Actors

Fire-and-forget works for independent tasks, but what about work that needs to be ordered, tracked, or retried? For that, we need a job queue.

An actor is a natural fit for a job queue. It holds a queue of pending jobs as its state, processes them one at a time, and can retry failed jobs with backoff. Because the actor processes messages sequentially, there are no race conditions -- only one job runs at a time.

The pattern has three parts:

**The job definition.** A type that describes the work to be done:

```gleam
pub type Job {
  GenerateReport(board_id: String)
  ProcessThumbnail(upload_id: String, file_path: String)
  SendNotifications(task_id: String, recipients: List(String))
}

pub type JobStatus {
  Pending
  InProgress(progress: Int)
  Complete(result: String)
  Failed(reason: String)
}
```

**The worker actor.** Holds the queue and processes jobs:

```gleam
pub type JobMessage {
  Enqueue(job: Job, reply_to: process.Subject(String))
  ProcessNext
  GetStatus(job_id: String, reply_to: process.Subject(JobStatus))
  Tick
}
```

The `Enqueue` message adds a job to the queue and replies with a job ID. The `ProcessNext` message tells the worker to pick up the next job. The `GetStatus` message lets clients check on a job's progress.

**The retry logic.** When a job fails, the worker does not crash. It catches the error, marks the job as failed, increments a retry counter, and re-enqueues the job with an exponential backoff delay. After a maximum number of retries, the job is marked as permanently failed.

The key insight is that the actor's sequential processing gives you serialization for free. You do not need locks, semaphores, or distributed coordination. The mailbox is the queue. The message handler is the consumer. The actor state is the tracking database.

For higher throughput, you can run multiple worker actors -- each with its own queue -- and distribute jobs across them. But for most applications, a single worker per job type is sufficient.

### 1.5 Progress Reporting

Once you move work to the background, the client needs a way to know when it finishes. There are two approaches, both of which we have already used in this course.

**Polling.** The client asks the server repeatedly at a fixed interval. HTMX makes this trivially easy with `hx-trigger="every 2s"`. The endpoint returns the current status as an HTML fragment. When the status is "complete," the response includes the result and stops the polling by not including the polling trigger in the returned HTML.

This is the approach from [Chapter 16](../03-advanced/16-extensions-and-patterns.md) (extensions and patterns). It is simple, stateless on the server side, and works through any proxy or CDN. The downside is latency -- the client might wait up to one polling interval after the job finishes before it sees the result.

**Server-Sent Events.** The server pushes status updates to the client as they happen. This is the approach from [Chapter 13](../03-advanced/13-real-time-with-sse.md) (real-time with SSE). It is more responsive -- the client sees progress updates the instant they happen -- but requires a persistent connection.

For most background jobs, polling is the better choice. It is simpler to implement, easier to debug, and does not require SSE infrastructure. Use SSE when you need real-time progress updates (like a file upload progress bar) or when many clients are watching the same job.

Both approaches share the same server-side requirement: the job worker must store its status somewhere that the HTTP handler can read. The actor state is the natural place. The client asks the HTTP handler, the handler asks the actor, the actor replies with the current status.

### 1.6 Scheduled and Periodic Jobs

Some work does not happen in response to a user action. It happens on a schedule. Cleaning up old temporary files every hour. Sending daily digest emails. Pruning expired sessions every fifteen minutes.

In many frameworks, you would use cron, a task scheduler library, or an external service. On the BEAM, you can build periodic jobs with a single function: `process.send_after`.

`process.send_after` sends a message to a subject after a delay. The trick is to have the actor send the message to itself:

```
  Actor starts
      │
      ▼
  Schedule first Tick (send_after 1 hour)
      │
      ▼
  Wait for messages...
      │
      ▼
  Tick arrives
      │
      ├── Do the cleanup work
      │
      └── Schedule the next Tick (send_after 1 hour)
              │
              ▼
          Wait for messages...
              │
              ▼
          Tick arrives
              │
              └── (repeat forever)
```

This is a self-rescheduling loop. The actor sends itself a `Tick` message on a delay. When the `Tick` arrives, it does the work and schedules the next `Tick`. This continues indefinitely.

The pattern is simple and robust:

- If the work takes one second and the interval is one hour, the next tick happens one hour after the work finishes, not one hour after it started. This prevents drift and overlap.
- If the actor crashes and is restarted by a supervisor, it schedules a new first tick in its initialization. No ticks are permanently lost.
- The interval can be changed dynamically by storing it in the actor state and using a different delay for the next `send_after`.

`process.send_after` takes three arguments: the subject to send to, the delay in milliseconds, and the message to send. It returns a `Timer` that you can cancel with `process.cancel_timer` if needed.

### 1.7 Supervision and "Let It Crash"

The BEAM has a distinctive philosophy about error handling: **let it crash.**

This sounds reckless, but it is the opposite. The idea is that instead of writing defensive code to handle every possible failure in every possible place, you write processes that do their job correctly under normal circumstances. When something unexpected happens -- a network timeout, a corrupt file, a bug -- the process crashes. A supervisor notices the crash and restarts the process in a clean state.

This works because of isolation. When a process crashes, no other process is affected. There is no shared memory to corrupt, no locks to leave in a bad state, no global variables to become inconsistent. The crashed process simply disappears, and a fresh one takes its place.

A **supervisor** is a process that starts child processes and monitors them. When a child dies, the supervisor restarts it according to a strategy:

| Strategy         | Behavior                                                       |
|------------------|----------------------------------------------------------------|
| **One-for-one**  | Only restart the child that crashed                            |
| **One-for-all**  | Restart all children if any one crashes                        |
| **Rest-for-one** | Restart the crashed child and all children started after it    |

For our job workers, one-for-one is usually right. If the report worker crashes, restart it. The cleanup worker and the notification worker are unaffected.

Supervision trees can be nested. A top-level supervisor watches several sub-supervisors, each of which watches a group of workers. This creates a hierarchy where failures are handled at the appropriate level.

In Gleam, supervision is provided by the `gleam/otp/supervisor` module. You define the children, choose a strategy, and start the supervisor. From that point on, the supervisor handles restarts automatically.

The practical upshot is that your background jobs are fault-tolerant without you writing try-catch blocks everywhere. A job crashes? The supervisor restarts the worker. The worker initializes with a clean state. Pending jobs might need to be re-enqueued (which is why storing job state in a separate actor or database is a good idea), but the system keeps running.

This is the BEAM's superpower. It was designed for telephone switches that must run for years without downtime. That same reliability is available to your task board's background job system.

---

## 2. Code Walkthrough

We are going to build four things in this walkthrough:

1. A synchronous endpoint that demonstrates the problem.
2. A fire-and-forget refactor that solves it.
3. A full job worker actor with queuing, progress tracking, and retry logic.
4. A periodic cleanup worker.

Along the way, we will wire up HTMX to show progress via polling and SSE, and connect everything to a supervision tree.

### Step 1 -- The Slow Endpoint Problem

Let us start with the wrong way to do things. Suppose the Teamwork application has a "Generate Board Report" feature. The handler fetches all tasks, computes statistics, and builds a formatted report. This takes several seconds.

Here is the naive implementation:

```gleam
// src/teamwork/router.gleam

import gleam/int
import gleam/list
import gleam/string
import gleam/erlang/process
import gleam/otp/actor
import lustre/element
import lustre/element/html
import lustre/attribute.{attribute}
import wisp
import teamwork/web.{type Context}
import teamwork/state
import teamwork/views

fn handle_generate_report(req: wisp.Request, ctx: Context) -> wisp.Response {
  // Fetch all tasks -- fast
  let tasks = actor.call(
    ctx.tasks,
    1000,
    state.GetAllTasks,
  )

  // Generate report -- SLOW (simulating 5 seconds of work)
  let report = generate_board_report(tasks)

  // Render the result
  let html = views.report_page(report)
  wisp.html_response(element.to_string(html), 200)
}

fn generate_board_report(tasks: List(state.Task)) -> String {
  // Simulate expensive computation
  process.sleep(5000)

  let total = list.length(tasks)
  let done = list.count(tasks, fn(t) { t.done })
  let pending = total - done

  "Board Report: "
  <> int.to_string(total) <> " total, "
  <> int.to_string(done) <> " done, "
  <> int.to_string(pending) <> " pending"
}
```

When a user clicks "Generate Report," nothing happens for five seconds. The browser shows a spinner in the tab title. The HTMX request indicator spins. The user wonders if it is broken. If they click again, they start a second five-second computation.

And here is the HTMX trigger on the client side (this snippet assumes `import hx` is in scope):

```gleam
fn report_button() -> element.Element(t) {
  html.button(
    [
      hx.post("/reports/generate"),
      hx.target(hx.Selector("#report-output")),
      hx.swap(hx.InnerHTML),
      attribute("class", "btn btn-primary"),
    ],
    [html.text("Generate Report")],
  )
}

fn report_section() -> element.Element(t) {
  html.div(
    [attribute("id", "report-section")],
    [
      report_button(),
      html.div(
        [attribute("id", "report-output")],
        [],
      ),
    ],
  )
}
```

The user clicks the button, waits five seconds, and eventually sees the report. This is unacceptable. Let us fix it.

### Step 2 -- Fire-and-Forget with `process.start`

The simplest fix is to spawn the work in a separate process and respond immediately. For cases where the user does not need to see the result right away -- like triggering an email notification or starting a data export that will be emailed when complete -- this is all you need.

```gleam
// src/teamwork/router.gleam

fn handle_generate_report(req: wisp.Request, ctx: Context) -> wisp.Response {
  // Spawn the work in a background process
  process.start(fn() {
    let tasks = actor.call(
      ctx.tasks,
      1000,
      state.GetAllTasks,
    )
    let report = generate_board_report(tasks)
    // In a real application, you might save this to the database
    // or send it via email
    save_report_to_db(ctx.db, report)
  }, linked: False)

  // Respond immediately
  wisp.response(202)
  |> wisp.set_header("HX-Trigger", "jobStarted")
}
```

The handler returns in microseconds. The background process does the expensive work on its own schedule. The `linked: False` parameter means that if the background process crashes, the HTTP handler (which has already returned) is not affected.

On the HTMX side, we show immediate feedback:

```gleam
fn report_button() -> element.Element(t) {
  html.button(
    [
      hx.post("/reports/generate"),
      hx.target(hx.Selector("#report-output")),
      hx.swap(hx.InnerHTML),
      attribute("class", "btn btn-primary"),
    ],
    [html.text("Generate Report")],
  )
}
```

And the server's 202 response includes a fragment that tells the user what is happening:

```gleam
fn handle_generate_report(req: wisp.Request, ctx: Context) -> wisp.Response {
  process.start(fn() {
    let tasks = actor.call(ctx.tasks, 1000, state.GetAllTasks)
    let report = generate_board_report(tasks)
    save_report_to_db(ctx.db, report)
  }, linked: False)

  let html = html.div(
    [attribute("id", "report-output")],
    [
      html.p(
        [attribute("class", "text-muted")],
        [html.text("Report generation started. You will receive an email when it is ready.")],
      ),
    ],
  )

  wisp.html_response(element.to_string(html), 202)
}
```

This is a huge improvement. The user gets instant feedback. But what if they want to see the report in the browser, not wait for an email? For that, we need progress tracking.

### Step 3 -- The Job Worker Actor

Now we build the real thing: an actor that manages a queue of jobs, processes them one at a time, tracks their status, and supports retry on failure.

First, the types. Create a new module:

```gleam
// src/teamwork/jobs.gleam

import gleam/dict.{type Dict}
import gleam/erlang/process.{type Subject}
import gleam/int
import gleam/list
import gleam/option.{type Option, None, Some}
import gleam/otp/actor
import gleam/string

/// A job represents a unit of background work.
pub type Job {
  GenerateReport(board_id: String)
  ProcessThumbnail(upload_id: String, file_path: String)
  SendNotifications(task_id: String, recipients: List(String))
}

/// The status of a job in the queue.
pub type JobStatus {
  Pending
  InProgress(progress: Int)
  Complete(result: String)
  Failed(reason: String)
}

/// A tracked job with metadata.
pub type TrackedJob {
  TrackedJob(
    id: String,
    job: Job,
    status: JobStatus,
    attempts: Int,
    max_attempts: Int,
  )
}

/// Messages the job worker actor understands.
pub type JobMessage {
  Enqueue(job: Job, reply_to: Subject(String))
  ProcessNext
  GetStatus(job_id: String, reply_to: Subject(JobStatus))
  UpdateProgress(job_id: String, progress: Int)
  JobCompleted(job_id: String, result: String)
  JobFailed(job_id: String, reason: String)
  Tick
}

/// The worker actor's state.
pub type WorkerState {
  WorkerState(
    self: Subject(JobMessage),
    jobs: Dict(String, TrackedJob),
    queue: List(String),
    next_id: Int,
    processing: Option(String),
  )
}
```

Let us walk through each type:

- **`Job`** describes the work. Each variant carries the data needed to do that specific job. `GenerateReport` needs a board ID. `ProcessThumbnail` needs an upload ID and file path. `SendNotifications` needs a task ID and recipient list.

- **`JobStatus`** tracks where a job is in its lifecycle. `Pending` means it is in the queue but not started. `InProgress` includes a progress percentage (0-100). `Complete` carries the result. `Failed` carries the error reason.

- **`TrackedJob`** wraps a `Job` with its ID, current status, attempt count, and maximum attempts. The attempt tracking enables retry logic.

- **`JobMessage`** is the actor's message protocol. `Enqueue` adds a job and replies with its ID. `ProcessNext` tells the worker to pick up the next queued job. `GetStatus` lets external code check on a job. `UpdateProgress`, `JobCompleted`, and `JobFailed` are sent by the job execution process back to the worker. `Tick` is for periodic self-scheduling.

- **`WorkerState`** holds everything the actor needs: a reference to itself (for self-messaging), the job dictionary, the queue (a list of job IDs in order), an auto-incrementing ID counter, and which job is currently processing (if any).

Now the actor startup:

```gleam
// src/teamwork/jobs.gleam (continued)

pub fn start() -> Result(Subject(JobMessage), actor.StartError) {
  actor.new(Nil)
  |> actor.on_message(fn(_, message) {
    // This temporary handler will be replaced once we have the subject
    actor.continue(Nil)
  })
  |> actor.start
  |> fn(result) {
    case result {
      Ok(started) -> {
        let subject = started.data
        // Now reinitialize with the real state and handler
        // by sending an init message
        Ok(subject)
      }
      Error(err) -> Error(err)
    }
  }
}
```

Wait -- we have a bootstrap problem. The actor needs its own subject in its state (so it can send messages to itself with `process.send_after`), but the subject is not available until after `actor.start` returns. Let us solve this properly:

```gleam
// src/teamwork/jobs.gleam (continued)

pub fn start() -> Result(Subject(JobMessage), actor.StartError) {
  // We use a two-phase initialization:
  // 1. Start the actor with a placeholder state
  // 2. Send it an Init message with its own subject

  let initial_state = WorkerState(
    // self will be set after we get the subject
    self: process.new_subject(),
    jobs: dict.new(),
    queue: [],
    next_id: 1,
    processing: None,
  )

  let result =
    actor.new(initial_state)
    |> actor.on_message(handle_job_message)
    |> actor.start

  case result {
    Ok(started) -> {
      let subject = started.data
      // Send the subject to the actor so it can store it as self
      actor.send(subject, Init(subject))
      Ok(subject)
    }
    Error(err) -> Error(err)
  }
}
```

Actually, let us simplify. We can add an `Init` variant to our message type and handle the bootstrap cleanly:

```gleam
/// Messages the job worker actor understands.
pub type JobMessage {
  Init(self: Subject(JobMessage))
  Enqueue(job: Job, reply_to: Subject(String))
  ProcessNext
  GetStatus(job_id: String, reply_to: Subject(JobStatus))
  UpdateProgress(job_id: String, progress: Int)
  JobCompleted(job_id: String, result: String)
  JobFailed(job_id: String, reason: String)
  Tick
}
```

And the start function becomes:

```gleam
pub fn start() -> Result(Subject(JobMessage), actor.StartError) {
  let placeholder_subject = process.new_subject()

  let initial_state = WorkerState(
    self: placeholder_subject,
    jobs: dict.new(),
    queue: [],
    next_id: 1,
    processing: None,
  )

  let result =
    actor.new(initial_state)
    |> actor.on_message(handle_job_message)
    |> actor.start

  case result {
    Ok(started) -> {
      let subject = started.data
      // Tell the actor its own address
      actor.send(subject, Init(subject))
      Ok(subject)
    }
    Error(err) -> Error(err)
  }
}
```

Now the message handler. This is the core of the job worker:

```gleam
fn handle_job_message(
  state: WorkerState,
  message: JobMessage,
) -> actor.Next(WorkerState, JobMessage) {
  case message {
    // Phase 1: Store our own subject for self-messaging
    Init(subject) -> {
      actor.continue(WorkerState(..state, self: subject))
    }

    // Phase 2: Accept a new job into the queue
    Enqueue(job, reply_to) -> {
      let job_id = "job-" <> int.to_string(state.next_id)

      let tracked = TrackedJob(
        id: job_id,
        job: job,
        status: Pending,
        attempts: 0,
        max_attempts: 3,
      )

      let new_state = WorkerState(
        ..state,
        jobs: dict.insert(state.jobs, job_id, tracked),
        queue: list.append(state.queue, [job_id]),
        next_id: state.next_id + 1,
      )

      // Reply with the job ID so the client can track it
      process.send(reply_to, job_id)

      // Trigger processing if we are not already busy
      case state.processing {
        None -> actor.send(state.self, ProcessNext)
        Some(_) -> Nil
      }

      actor.continue(new_state)
    }

    // Phase 3: Pick up the next job from the queue
    ProcessNext -> {
      case state.queue {
        [] -> {
          // Nothing to do
          actor.continue(WorkerState(..state, processing: None))
        }
        [next_id, ..rest] -> {
          case dict.get(state.jobs, next_id) {
            Ok(tracked) -> {
              // Mark as in progress
              let updated_job = TrackedJob(
                ..tracked,
                status: InProgress(0),
                attempts: tracked.attempts + 1,
              )
              let new_jobs = dict.insert(state.jobs, next_id, updated_job)

              // Spawn a process to do the actual work
              let self = state.self
              process.start(fn() {
                execute_job(self, next_id, tracked.job)
              }, linked: False)

              actor.continue(WorkerState(
                ..state,
                jobs: new_jobs,
                queue: rest,
                processing: Some(next_id),
              ))
            }
            Error(_) -> {
              // Job was removed; skip it
              actor.send(state.self, ProcessNext)
              actor.continue(WorkerState(..state, queue: rest))
            }
          }
        }
      }
    }

    // Phase 4: Status queries
    GetStatus(job_id, reply_to) -> {
      let status = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> tracked.status
        Error(_) -> Failed("Job not found")
      }
      process.send(reply_to, status)
      actor.continue(state)
    }

    // Phase 5: Progress updates from the executing process
    UpdateProgress(job_id, progress) -> {
      let new_jobs = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> {
          let updated = TrackedJob(..tracked, status: InProgress(progress))
          dict.insert(state.jobs, job_id, updated)
        }
        Error(_) -> state.jobs
      }
      actor.continue(WorkerState(..state, jobs: new_jobs))
    }

    // Phase 6: Job completed successfully
    JobCompleted(job_id, result) -> {
      let new_jobs = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> {
          let updated = TrackedJob(..tracked, status: Complete(result))
          dict.insert(state.jobs, job_id, updated)
        }
        Error(_) -> state.jobs
      }

      // Process the next job in the queue
      actor.send(state.self, ProcessNext)

      actor.continue(WorkerState(..state, jobs: new_jobs, processing: None))
    }

    // Phase 7: Job failed
    JobFailed(job_id, reason) -> {
      let new_state = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> {
          case tracked.attempts < tracked.max_attempts {
            True -> {
              // Retry: put the job back in the queue
              let updated = TrackedJob(..tracked, status: Pending)
              let new_jobs = dict.insert(state.jobs, job_id, updated)

              // Exponential backoff: 1s, 2s, 4s
              let delay = 1000 * pow2(tracked.attempts - 1)
              process.send_after(state.self, delay, ProcessNext)

              WorkerState(
                ..state,
                jobs: new_jobs,
                queue: [job_id, ..state.queue],
                processing: None,
              )
            }
            False -> {
              // Max retries exceeded -- mark as permanently failed
              let updated = TrackedJob(
                ..tracked,
                status: Failed("Max retries exceeded: " <> reason),
              )
              let new_jobs = dict.insert(state.jobs, job_id, updated)

              // Move on to the next job
              actor.send(state.self, ProcessNext)

              WorkerState(
                ..state,
                jobs: new_jobs,
                processing: None,
              )
            }
          }
        }
        Error(_) -> {
          actor.send(state.self, ProcessNext)
          WorkerState(..state, processing: None)
        }
      }

      actor.continue(new_state)
    }

    // Phase 8: Periodic tick (used for scheduled jobs, covered later)
    Tick -> {
      actor.continue(state)
    }
  }
}
```

That is a big handler, but each case is straightforward. Let us trace through a complete lifecycle:

1. The HTTP handler calls `actor.call(worker, 5000, fn(reply) { Enqueue(GenerateReport("board-1"), reply) })`. This sends an `Enqueue` message and waits for the job ID.

2. The actor receives `Enqueue`. It creates a `TrackedJob` with status `Pending`, adds it to the dictionary and queue, replies with the job ID `"job-1"`, and sends itself `ProcessNext` (since nothing is currently processing).

3. The actor receives `ProcessNext`. It pops `"job-1"` from the queue, marks it `InProgress(0)`, spawns a process to execute the job, and records `processing: Some("job-1")`.

4. The spawned process calls `execute_job`, which does the real work. As it progresses, it sends `UpdateProgress("job-1", 25)`, `UpdateProgress("job-1", 50)`, etc. When done, it sends `JobCompleted("job-1", "/reports/board-1.pdf")`.

5. Meanwhile, the HTMX client polls `GET /jobs/job-1/status`. The HTTP handler calls `actor.call(worker, 5000, fn(reply) { GetStatus("job-1", reply) })` and returns the status as HTML.

6. The actor receives `JobCompleted`. It marks the job as `Complete`, clears the `processing` field, and sends itself `ProcessNext` to pick up any queued jobs.

Now the job execution function:

```gleam
fn execute_job(
  worker: Subject(JobMessage),
  job_id: String,
  job: Job,
) -> Nil {
  case job {
    GenerateReport(board_id) -> {
      // Simulate a multi-step report generation
      actor.send(worker, UpdateProgress(job_id, 10))
      process.sleep(1000)

      actor.send(worker, UpdateProgress(job_id, 30))
      process.sleep(1000)

      actor.send(worker, UpdateProgress(job_id, 60))
      process.sleep(1000)

      actor.send(worker, UpdateProgress(job_id, 90))
      process.sleep(1000)

      let result = "/reports/" <> board_id <> ".html"
      actor.send(worker, UpdateProgress(job_id, 100))
      actor.send(worker, JobCompleted(job_id, result))
    }

    ProcessThumbnail(upload_id, file_path) -> {
      actor.send(worker, UpdateProgress(job_id, 20))
      // In a real application: read the file, resize, save thumbnail
      process.sleep(2000)

      actor.send(worker, UpdateProgress(job_id, 100))
      actor.send(
        worker,
        JobCompleted(job_id, "/uploads/thumbs/" <> upload_id <> ".jpg"),
      )
    }

    SendNotifications(task_id, recipients) -> {
      let total = list.length(recipients)
      list.index_map(recipients, fn(recipient, index) {
        // Simulate sending an email
        process.sleep(200)
        let progress = { { index + 1 } * 100 } / total
        actor.send(worker, UpdateProgress(job_id, progress))
      })
      actor.send(
        worker,
        JobCompleted(
          job_id,
          "Sent " <> int.to_string(total) <> " notifications",
        ),
      )
    }
  }
}

/// Simple power-of-2 helper for exponential backoff.
fn pow2(n: Int) -> Int {
  case n <= 0 {
    True -> 1
    False -> 2 * pow2(n - 1)
  }
}
```

In a real application, `execute_job` would call actual functions -- database queries, file system operations, HTTP calls to external services. Here we simulate the work with `process.sleep` to focus on the pattern.

The important thing is the communication protocol. The executing process sends `UpdateProgress`, `JobCompleted`, or `JobFailed` messages back to the worker actor. The worker updates its state, and any polling clients see the latest status.

Now the HTTP handler that ties it all together:

```gleam
// src/teamwork/router.gleam

import gleam/otp/actor
import gleam/erlang/process
import lustre/element
import lustre/element/html
import lustre/attribute.{attribute}
import hx
import wisp
import teamwork/jobs.{
  type JobStatus, Complete, Failed, GenerateReport, InProgress, Pending,
}
import teamwork/web.{type Context}

fn handle_generate_report(req: wisp.Request, ctx: Context) -> wisp.Response {
  // Enqueue the job and get a job ID back
  let job_id = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) { jobs.Enqueue(GenerateReport("board-1"), reply) },
  )

  // Return a progress indicator that starts polling
  let html = job_progress_fragment(job_id, Pending)
  wisp.html_response(element.to_string(html), 202)
}

fn handle_job_status(
  req: wisp.Request,
  ctx: Context,
  job_id: String,
) -> wisp.Response {
  let status = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) { jobs.GetStatus(job_id, reply) },
  )

  let html = job_progress_fragment(job_id, status)
  wisp.html_response(element.to_string(html), 200)
}
```

The `handle_generate_report` handler enqueues the job, gets back an ID, and returns a progress fragment that immediately starts polling. The `handle_job_status` handler is the polling endpoint -- it checks the job's status and returns the appropriate HTML.

### Step 4 -- Polling Progress with `hx-trigger="every 2s"`

Now we build the HTMX fragments that show progress and automatically poll for updates. This is the technique from [Chapter 16](../03-advanced/16-extensions-and-patterns.md) applied to background jobs.

```gleam
// src/teamwork/views.gleam

import gleam/int
import gleam/time/duration
import lustre/element.{type Element}
import lustre/element/html
import lustre/attribute.{attribute}
import hx
import gleam/option
import teamwork/jobs.{type JobStatus, Complete, Failed, InProgress, Pending}

pub fn job_progress_fragment(
  job_id: String,
  status: JobStatus,
) -> Element(t) {
  case status {
    Pending -> {
      // Job is queued -- show waiting message with polling
      html.div(
        [
          attribute("id", "job-progress"),
          hx.get("/jobs/" <> job_id <> "/status"),
          hx.trigger_polling(
            timing: duration.milliseconds(2000),
            filters: option.None,
            on_load: True,
          ),
          hx.swap(hx.OuterHTML),
          attribute("class", "job-status pending"),
        ],
        [
          html.div(
            [attribute("class", "progress-bar-container")],
            [
              html.div(
                [
                  attribute("class", "progress-bar"),
                  attribute("style", "width: 0%"),
                ],
                [],
              ),
            ],
          ),
          html.p([], [html.text("Job queued. Waiting to start...")]),
        ],
      )
    }

    InProgress(progress) -> {
      // Job is running -- show progress bar with continued polling
      let width = int.to_string(progress) <> "%"
      html.div(
        [
          attribute("id", "job-progress"),
          hx.get("/jobs/" <> job_id <> "/status"),
          hx.trigger_polling(
            timing: duration.milliseconds(2000),
            filters: option.None,
            on_load: True,
          ),
          hx.swap(hx.OuterHTML),
          attribute("class", "job-status in-progress"),
        ],
        [
          html.div(
            [attribute("class", "progress-bar-container")],
            [
              html.div(
                [
                  attribute("class", "progress-bar"),
                  attribute("style", "width: " <> width),
                ],
                [],
              ),
            ],
          ),
          html.p([], [
            html.text(
              "Processing... " <> int.to_string(progress) <> "% complete",
            ),
          ]),
        ],
      )
    }

    Complete(result) -> {
      // Job is done -- show result, NO polling trigger
      html.div(
        [
          attribute("id", "job-progress"),
          attribute("class", "job-status complete"),
        ],
        [
          html.div(
            [attribute("class", "progress-bar-container")],
            [
              html.div(
                [
                  attribute("class", "progress-bar complete"),
                  attribute("style", "width: 100%"),
                ],
                [],
              ),
            ],
          ),
          html.p([], [html.text("Complete!")]),
          html.a(
            [
              attribute("href", result),
              attribute("class", "btn btn-success"),
            ],
            [html.text("Download Report")],
          ),
        ],
      )
    }

    Failed(reason) -> {
      // Job failed -- show error, NO polling trigger
      html.div(
        [
          attribute("id", "job-progress"),
          attribute("class", "job-status failed"),
        ],
        [
          html.p(
            [attribute("class", "error-message")],
            [html.text("Job failed: " <> reason)],
          ),
          html.button(
            [
              hx.post("/reports/generate"),
              hx.target(hx.Selector("#report-output")),
              hx.swap(hx.InnerHTML),
              attribute("class", "btn btn-retry"),
            ],
            [html.text("Try Again")],
          ),
        ],
      )
    }
  }
}
```

The key insight here is how polling starts and stops. When the job is `Pending` or `InProgress`, the fragment includes `hx.trigger_polling(...)` which generates the `hx-trigger="every 2s"` attribute. HTMX automatically sends a GET request every two seconds, and the response -- which is a new fragment -- replaces the old one via `hx.swap(hx.OuterHTML)`.

When the job reaches `Complete` or `Failed`, the response fragment does **not** include a polling trigger. HTMX sees that the replacement element has no `hx-trigger="every 2s"`, so polling stops. This is the same self-canceling pattern from [Chapter 16](../03-advanced/16-extensions-and-patterns.md).

The flow looks like this:

```
  Browser                              Server
  ───────                              ──────
    │                                     │
    │  POST /reports/generate             │
    │────────────────────────────────────►│
    │                                     │  Enqueue job-1
    │  202: <div hx-trigger="every 2s">  │
    │        "Job queued..."              │
    │◄────────────────────────────────────│
    │                                     │
    │  (2 seconds pass)                   │
    │                                     │
    │  GET /jobs/job-1/status             │
    │────────────────────────────────────►│
    │  200: <div hx-trigger="every 2s">  │
    │        "Processing... 30%"          │
    │◄────────────────────────────────────│
    │                                     │
    │  (2 seconds pass)                   │
    │                                     │
    │  GET /jobs/job-1/status             │
    │────────────────────────────────────►│
    │  200: <div hx-trigger="every 2s">  │
    │        "Processing... 75%"          │
    │◄────────────────────────────────────│
    │                                     │
    │  (2 seconds pass)                   │
    │                                     │
    │  GET /jobs/job-1/status             │
    │────────────────────────────────────►│
    │  200: <div> (NO polling trigger)    │
    │        "Complete! [Download]"        │
    │◄────────────────────────────────────│
    │                                     │
    │  (polling stops)                    │
    │                                     │
```

The progress bar fills up as the job progresses. When it completes, the user sees a download link and polling stops automatically. No JavaScript needed beyond what HTMX provides.

### Step 5 -- SSE-Based Progress Alternative

Polling every two seconds is good enough for most background jobs. But if you want more responsive progress updates -- or if many clients are watching the same job -- SSE is a better fit.

This builds on the SSE patterns from [Chapter 13](../03-advanced/13-real-time-with-sse.md). The difference is that instead of broadcasting task list changes, we are broadcasting job progress updates.

First, the SSE endpoint:

```gleam
// src/teamwork.gleam (hybrid router section)

import gleam/http/request
import gleam/http/response
import gleam/erlang/process
import gleam/otp/actor
import gleam/string_tree
import mist
import teamwork/jobs

fn handle_job_sse(
  req: request.Request(mist.Connection),
  ctx: Context,
  job_id: String,
) -> response.Response(mist.ResponseData) {
  mist.server_sent_events(
    request: req,
    initial_response: response.new(200),
    init: fn(subject) {
      // Create a subject for receiving progress updates
      let progress_subject = process.new_subject()

      // Subscribe to progress updates for this job
      actor.send(ctx.job_worker, jobs.Subscribe(job_id, progress_subject))

      // Send an initial status update so the loop fires immediately
      let status = actor.call(
        ctx.job_worker,
        5000,
        fn(reply) { jobs.GetStatus(job_id, reply) },
      )
      process.send(progress_subject, status)

      Ok(actor.initialised(progress_subject))
    },
    loop: fn(progress_subject, message, conn) {
      // message is a JobStatus update
      let html = views.job_progress_sse_fragment(job_id, message)
      let event =
        mist.event(string_tree.from_string(html))
        |> mist.event_name("job-progress")

      case mist.send_event(conn, event) {
        Ok(_) -> {
          case message {
            jobs.Complete(_) | jobs.Failed(_) -> {
              // Send a close event and stop
              let close_event =
                mist.event(string_tree.from_string(""))
                |> mist.event_name("job-done")
              let _ = mist.send_event(conn, close_event)
              actor.stop()
            }
            _ -> actor.continue(progress_subject)
          }
        }
        Error(_) -> {
          // Client disconnected
          actor.stop()
        }
      }
    },
  )
}
```

The client-side HTMX markup uses the SSE extension:

```gleam
pub fn job_progress_sse(job_id: String) -> Element(t) {
  html.div(
    [
      attribute("hx-ext", "sse"),
      attribute("sse-connect", "/jobs/" <> job_id <> "/stream"),
      attribute("sse-close", "job-done"),
    ],
    [
      html.div(
        [
          attribute("id", "job-progress"),
          attribute("sse-swap", "job-progress"),
        ],
        [html.text("Connecting...")],
      ),
    ],
  )
}
```

When the SSE connection opens, the server sends the current status immediately. As the job progresses, the worker actor sends status updates to all subscribed SSE connections. When the job completes, the server sends a `job-done` event, and `sse-close="job-done"` tells the HTMX SSE extension to close the connection.

To support subscriptions, you would add a `Subscribe` message to the job worker:

```gleam
pub type JobMessage {
  // ... existing variants ...
  Subscribe(job_id: String, subscriber: Subject(JobStatus))
}
```

And in the handler, maintain a list of subscribers per job:

```gleam
Subscribe(job_id, subscriber) -> {
  let new_subscribers = case dict.get(state.subscribers, job_id) {
    Ok(existing) -> dict.insert(state.subscribers, job_id, [subscriber, ..existing])
    Error(_) -> dict.insert(state.subscribers, job_id, [subscriber])
  }
  actor.continue(WorkerState(..state, subscribers: new_subscribers))
}
```

Then, whenever the status changes (in `UpdateProgress`, `JobCompleted`, `JobFailed`), broadcast to all subscribers:

```gleam
fn broadcast_status(
  subscribers: Dict(String, List(Subject(JobStatus))),
  job_id: String,
  status: JobStatus,
) -> Nil {
  case dict.get(subscribers, job_id) {
    Ok(subs) -> {
      list.each(subs, fn(sub) {
        process.send(sub, status)
      })
    }
    Error(_) -> Nil
  }
}
```

For most Teamwork use cases, polling is sufficient. Use the SSE approach when you need smoother progress updates or when multiple team members are watching the same report generation.

### Step 6 -- Periodic Cleanup Worker

The Teamwork application stores uploaded files (from [Chapter 25](25-file-uploads.md)) in a temporary directory. After files are processed, the originals should be cleaned up. Instead of doing this on every upload, we run a periodic cleanup job that sweeps old files once per hour.

```gleam
// src/teamwork/cleanup.gleam

import gleam/erlang/process.{type Subject}
import gleam/otp/actor
import gleam/list
import gleam/int
import gleam/io

pub type CleanupMessage {
  Tick
  RunNow
  GetStats(reply_to: Subject(CleanupStats))
}

pub type CleanupStats {
  CleanupStats(
    files_cleaned: Int,
    last_run_at: Int,
    next_run_in_ms: Int,
  )
}

pub type CleanupState {
  CleanupState(
    self: Subject(CleanupMessage),
    upload_dir: String,
    interval_ms: Int,
    files_cleaned_total: Int,
    last_run_at: Int,
  )
}

pub fn start(upload_dir: String) -> Result(Subject(CleanupMessage), actor.StartError) {
  let placeholder = process.new_subject()
  let interval = 3_600_000  // 1 hour in milliseconds

  let initial_state = CleanupState(
    self: placeholder,
    upload_dir: upload_dir,
    interval_ms: interval,
    files_cleaned_total: 0,
    last_run_at: 0,
  )

  let result =
    actor.new(initial_state)
    |> actor.on_message(handle_cleanup_message)
    |> actor.start

  case result {
    Ok(started) -> {
      let subject = started.data
      // Store the real subject
      actor.send(subject, Init(subject))
      // Schedule the first tick
      process.send_after(subject, interval, Tick)
      io.println(
        "Cleanup worker started. First run in "
        <> int.to_string(interval / 1000)
        <> " seconds.",
      )
      Ok(subject)
    }
    Error(err) -> Error(err)
  }
}
```

Wait, we need to add `Init` to the message type:

```gleam
pub type CleanupMessage {
  Init(self: Subject(CleanupMessage))
  Tick
  RunNow
  GetStats(reply_to: Subject(CleanupStats))
}
```

Now the message handler:

```gleam
fn handle_cleanup_message(
  state: CleanupState,
  message: CleanupMessage,
) -> actor.Next(CleanupState, CleanupMessage) {
  case message {
    Init(subject) -> {
      actor.continue(CleanupState(..state, self: subject))
    }

    Tick -> {
      // Do the cleanup work
      let files_cleaned = delete_old_temp_files(state.upload_dir)

      io.println(
        "Cleanup: removed "
        <> int.to_string(files_cleaned)
        <> " old temporary files",
      )

      // Schedule the next tick
      process.send_after(state.self, state.interval_ms, Tick)

      // Update stats
      let now = current_time_ms()
      actor.continue(CleanupState(
        ..state,
        files_cleaned_total: state.files_cleaned_total + files_cleaned,
        last_run_at: now,
      ))
    }

    RunNow -> {
      // Manual trigger -- do the work and reschedule
      let files_cleaned = delete_old_temp_files(state.upload_dir)

      io.println(
        "Manual cleanup: removed "
        <> int.to_string(files_cleaned)
        <> " old temporary files",
      )

      let now = current_time_ms()
      actor.continue(CleanupState(
        ..state,
        files_cleaned_total: state.files_cleaned_total + files_cleaned,
        last_run_at: now,
      ))
    }

    GetStats(reply_to) -> {
      let stats = CleanupStats(
        files_cleaned: state.files_cleaned_total,
        last_run_at: state.last_run_at,
        next_run_in_ms: state.interval_ms,
      )
      process.send(reply_to, stats)
      actor.continue(state)
    }
  }
}

fn delete_old_temp_files(upload_dir: String) -> Int {
  // In a real application, you would:
  // 1. List files in the upload directory
  // 2. Check each file's age (creation time or last modified time)
  // 3. Delete files older than a threshold (e.g., 24 hours)
  // 4. Return the count of deleted files
  //
  // For this example, we simulate the operation:
  0
}

fn current_time_ms() -> Int {
  // In a real application, use erlang:system_time(millisecond)
  // For now, return 0 as a placeholder
  0
}
```

The self-rescheduling pattern is the heart of this worker. When `Tick` arrives:

1. The actor does the cleanup work.
2. It calls `process.send_after(state.self, state.interval_ms, Tick)` to schedule the next tick.
3. It updates its state with the results.

This creates an infinite loop, but a cooperative one. Between ticks, the actor is idle -- it is not consuming CPU or blocking anything. The BEAM scheduler only wakes it when a message arrives in its mailbox.

The `RunNow` variant lets administrators trigger an immediate cleanup without waiting for the next scheduled tick. You could wire this to a button in an admin panel:

```gleam
fn admin_cleanup_button() -> Element(t) {
  html.button(
    [
      hx.post("/admin/cleanup"),
      hx.target(hx.Selector("#cleanup-status")),
      hx.swap(hx.InnerHTML),
      attribute("class", "btn btn-warning"),
    ],
    [html.text("Run Cleanup Now")],
  )
}

fn handle_admin_cleanup(req: wisp.Request, ctx: Context) -> wisp.Response {
  actor.send(ctx.cleanup_worker, cleanup.RunNow)

  let html = html.p([], [
    html.text("Cleanup triggered. Check the server logs for results."),
  ])
  wisp.html_response(element.to_string(html), 200)
}
```

You can also add a stats endpoint to show when the last cleanup ran and how many files have been cleaned in total:

```gleam
fn handle_cleanup_stats(req: wisp.Request, ctx: Context) -> wisp.Response {
  let stats = actor.call(
    ctx.cleanup_worker,
    5000,
    cleanup.GetStats,
  )

  let html = html.div(
    [attribute("id", "cleanup-status")],
    [
      html.p([], [
        html.text("Files cleaned (total): " <> int.to_string(stats.files_cleaned)),
      ]),
      html.p([], [
        html.text("Next run in: " <> int.to_string(stats.next_run_in_ms / 60_000) <> " minutes"),
      ]),
    ],
  )
  wisp.html_response(element.to_string(html), 200)
}
```

### Step 7 -- Supervision Tree

Now we wrap our workers in a supervision tree. When the job worker crashes (bad input, unexpected error, resource exhaustion), the supervisor restarts it automatically. The application keeps running.

Here is how you set up a supervision tree for the Teamwork application:

```gleam
// src/teamwork/supervisor.gleam

import gleam/erlang/process
import gleam/otp/supervisor
import gleam/option.{None}
import teamwork/jobs
import teamwork/cleanup

pub type Workers {
  Workers(
    job_worker: process.Subject(jobs.JobMessage),
    cleanup_worker: process.Subject(cleanup.CleanupMessage),
  )
}

pub fn start_workers(upload_dir: String) -> Workers {
  // Start the job worker
  let assert Ok(job_worker) = jobs.start()

  // Start the cleanup worker
  let assert Ok(cleanup_worker) = cleanup.start(upload_dir)

  Workers(
    job_worker: job_worker,
    cleanup_worker: cleanup_worker,
  )
}
```

In a more sophisticated setup, you would use `gleam/otp/supervisor` to define the tree declaratively:

```gleam
// Conceptual supervision tree for Teamwork
//
//           teamwork_sup (one_for_one)
//           ┌─────────┼──────────────┐
//           │         │              │
//     task_actor   job_worker   cleanup_worker
//
// If job_worker crashes:
//   1. Supervisor detects the crash
//   2. Supervisor starts a new job_worker
//   3. task_actor and cleanup_worker are unaffected
//
// If cleanup_worker crashes:
//   1. Supervisor detects the crash
//   2. Supervisor starts a new cleanup_worker
//   3. task_actor and job_worker are unaffected
```

The one-for-one strategy is correct here because the workers are independent. The task actor does not depend on the job worker, and the cleanup worker does not depend on either. If one crashes, only that one needs to restart.

In your `main()` function, the wiring looks like this:

```gleam
// src/teamwork.gleam

import gleam/erlang/process
import gleam/otp/actor
import mist
import wisp
import wisp/wisp_mist
import teamwork/state
import teamwork/supervisor as sup
import teamwork/web.{type Context, Context}

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the task state actor
  let assert Ok(task_store) = state.start()

  // Start supervised background workers
  let workers = sup.start_workers("./uploads/tmp")

  // Build the context with all shared resources
  let ctx = Context(
    tasks: task_store,
    job_worker: workers.job_worker,
    cleanup_worker: workers.cleanup_worker,
  )

  // Start the HTTP server
  let assert Ok(_) =
    wisp_mist.handler(fn(req) { router.handle_request(req, ctx) }, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

The `Context` type now includes references to all background workers:

```gleam
// src/teamwork/web.gleam

import gleam/erlang/process.{type Subject}
import teamwork/state
import teamwork/jobs
import teamwork/cleanup

pub type Context {
  Context(
    tasks: Subject(state.Message),
    job_worker: Subject(jobs.JobMessage),
    cleanup_worker: Subject(cleanup.CleanupMessage),
  )
}
```

Every request handler can now enqueue jobs, check job status, or trigger cleanup -- all through the context, all type-checked by the compiler. No global state. No hidden dependencies.

When a worker crashes, the "let it crash" philosophy means you do not write defensive code in every function. Instead, you write clean, straightforward code that handles the normal case. The supervisor handles the abnormal cases by restarting the worker in a known-good state.

There is one caveat: when the job worker restarts, its in-memory job queue is lost. Jobs that were `Pending` or `InProgress` at the time of the crash are gone. For a production system, you would want to persist the job queue to the database (as we discussed in [Chapter 27](27-database-performance.md)) so that pending jobs survive restarts. The actor would load its queue from the database on startup, and update the database as jobs progress.

This is a common pattern in BEAM applications: use actors for fast, in-memory processing, but back them with a database for durability. The actor is the fast path; the database is the safety net.

### Step 8 -- Upload Thumbnail Processing

Let us connect background jobs to the file upload system from [Chapter 25](25-file-uploads.md). When a user uploads an image to a task, we want to generate a thumbnail in the background rather than making the user wait.

The upload handler:

```gleam
fn handle_file_upload(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  use formdata <- wisp.require_form(req)

  case list.find(formdata.files, fn(f) { f.0 == "attachment" }) {
    Ok(#(_, file)) -> {
      // Save the uploaded file
      let upload_id = generate_id()
      let file_path = save_uploaded_file(file, upload_id)

      // Enqueue thumbnail generation as a background job
      let job_id = actor.call(
        ctx.job_worker,
        5000,
        fn(reply) {
          jobs.Enqueue(
            jobs.ProcessThumbnail(upload_id, file_path),
            reply,
          )
        },
      )

      // Respond immediately with the upload confirmation
      // and a placeholder thumbnail that will be replaced
      let html = html.div(
        [attribute("id", "attachment-" <> upload_id)],
        [
          html.div(
            [attribute("class", "attachment-info")],
            [html.text("File uploaded successfully")],
          ),
          html.div(
            [
              attribute("id", "thumb-" <> upload_id),
              attribute("class", "thumbnail-placeholder"),
              hx.get("/jobs/" <> job_id <> "/thumbnail"),
              hx.trigger_polling(
                timing: duration.milliseconds(2000),
                filters: option.None,
                on_load: True,
              ),
              hx.swap(hx.OuterHTML),
            ],
            [
              html.span(
                [attribute("class", "spinner")],
                [],
              ),
              html.text("Generating thumbnail..."),
            ],
          ),
        ],
      )

      wisp.html_response(element.to_string(html), 200)
    }
    Error(_) -> {
      wisp.html_response("<p class=\"error\">No file uploaded</p>", 400)
    }
  }
}
```

The thumbnail polling endpoint:

```gleam
fn handle_thumbnail_status(
  req: wisp.Request,
  ctx: Context,
  job_id: String,
) -> wisp.Response {
  let status = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) { jobs.GetStatus(job_id, reply) },
  )

  let html = case status {
    Complete(thumbnail_path) -> {
      // Thumbnail is ready -- show it (no more polling)
      html.div(
        [attribute("class", "thumbnail")],
        [
          html.img([
            attribute("src", thumbnail_path),
            attribute("alt", "Attachment thumbnail"),
          ]),
        ],
      )
    }
    Failed(reason) -> {
      // Thumbnail generation failed -- show a generic icon
      html.div(
        [attribute("class", "thumbnail-fallback")],
        [html.text("Preview unavailable")],
      )
    }
    _ -> {
      // Still processing -- keep polling
      html.div(
        [
          attribute("class", "thumbnail-placeholder"),
          hx.get("/jobs/" <> job_id <> "/thumbnail"),
          hx.trigger_polling(
            timing: duration.milliseconds(2000),
            filters: option.None,
            on_load: True,
          ),
          hx.swap(hx.OuterHTML),
        ],
        [
          html.span(
            [attribute("class", "spinner")],
            [],
          ),
          html.text("Generating thumbnail..."),
        ],
      )
    }
  }

  wisp.html_response(element.to_string(html), 200)
}
```

This creates a smooth user experience:

1. The user uploads a file. The response appears instantly with a "Generating thumbnail..." placeholder and a spinner.
2. The placeholder polls every two seconds.
3. When the thumbnail is ready, the placeholder is replaced by the actual image. Polling stops.
4. If thumbnail generation fails, a fallback icon appears. Polling stops.

The user never waits for image processing. They can continue working on other tasks while the thumbnail generates in the background.

---

## 3. Full Code Listing

Here is the complete job worker module, consolidated and cleaned up:

```gleam
// src/teamwork/jobs.gleam

import gleam/dict.{type Dict}
import gleam/erlang/process.{type Subject}
import gleam/int
import gleam/list
import gleam/option.{type Option, None, Some}
import gleam/otp/actor
import gleam/io

// ── Types ──────────────────────────────────────────────────────────

pub type Job {
  GenerateReport(board_id: String)
  ProcessThumbnail(upload_id: String, file_path: String)
  SendNotifications(task_id: String, recipients: List(String))
}

pub type JobStatus {
  Pending
  InProgress(progress: Int)
  Complete(result: String)
  Failed(reason: String)
}

pub type TrackedJob {
  TrackedJob(
    id: String,
    job: Job,
    status: JobStatus,
    attempts: Int,
    max_attempts: Int,
  )
}

pub type JobMessage {
  Init(self: Subject(JobMessage))
  Enqueue(job: Job, reply_to: Subject(String))
  ProcessNext
  GetStatus(job_id: String, reply_to: Subject(JobStatus))
  UpdateProgress(job_id: String, progress: Int)
  JobCompleted(job_id: String, result: String)
  JobFailed(job_id: String, reason: String)
  Tick
}

pub type WorkerState {
  WorkerState(
    self: Subject(JobMessage),
    jobs: Dict(String, TrackedJob),
    queue: List(String),
    next_id: Int,
    processing: Option(String),
  )
}

// ── Public API ─────────────────────────────────────────────────────

pub fn start() -> Result(Subject(JobMessage), actor.StartError) {
  let placeholder = process.new_subject()

  let initial_state = WorkerState(
    self: placeholder,
    jobs: dict.new(),
    queue: [],
    next_id: 1,
    processing: None,
  )

  let result =
    actor.new(initial_state)
    |> actor.on_message(handle_job_message)
    |> actor.start

  case result {
    Ok(started) -> {
      let subject = started.data
      actor.send(subject, Init(subject))
      Ok(subject)
    }
    Error(err) -> Error(err)
  }
}

// ── Message Handler ────────────────────────────────────────────────

fn handle_job_message(
  state: WorkerState,
  message: JobMessage,
) -> actor.Next(WorkerState, JobMessage) {
  case message {
    Init(subject) -> {
      actor.continue(WorkerState(..state, self: subject))
    }

    Enqueue(job, reply_to) -> {
      let job_id = "job-" <> int.to_string(state.next_id)

      let tracked = TrackedJob(
        id: job_id,
        job: job,
        status: Pending,
        attempts: 0,
        max_attempts: 3,
      )

      let new_state = WorkerState(
        ..state,
        jobs: dict.insert(state.jobs, job_id, tracked),
        queue: list.append(state.queue, [job_id]),
        next_id: state.next_id + 1,
      )

      process.send(reply_to, job_id)

      case state.processing {
        None -> actor.send(state.self, ProcessNext)
        Some(_) -> Nil
      }

      actor.continue(new_state)
    }

    ProcessNext -> {
      case state.queue {
        [] -> {
          actor.continue(WorkerState(..state, processing: None))
        }
        [next_id, ..rest] -> {
          case dict.get(state.jobs, next_id) {
            Ok(tracked) -> {
              let updated_job = TrackedJob(
                ..tracked,
                status: InProgress(0),
                attempts: tracked.attempts + 1,
              )
              let new_jobs = dict.insert(state.jobs, next_id, updated_job)

              let self = state.self
              process.start(fn() {
                execute_job(self, next_id, tracked.job)
              }, linked: False)

              actor.continue(WorkerState(
                ..state,
                jobs: new_jobs,
                queue: rest,
                processing: Some(next_id),
              ))
            }
            Error(_) -> {
              actor.send(state.self, ProcessNext)
              actor.continue(WorkerState(..state, queue: rest))
            }
          }
        }
      }
    }

    GetStatus(job_id, reply_to) -> {
      let status = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> tracked.status
        Error(_) -> Failed("Job not found")
      }
      process.send(reply_to, status)
      actor.continue(state)
    }

    UpdateProgress(job_id, progress) -> {
      let new_jobs = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> {
          let updated = TrackedJob(..tracked, status: InProgress(progress))
          dict.insert(state.jobs, job_id, updated)
        }
        Error(_) -> state.jobs
      }
      actor.continue(WorkerState(..state, jobs: new_jobs))
    }

    JobCompleted(job_id, result) -> {
      let new_jobs = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> {
          let updated = TrackedJob(..tracked, status: Complete(result))
          dict.insert(state.jobs, job_id, updated)
        }
        Error(_) -> state.jobs
      }
      actor.send(state.self, ProcessNext)
      actor.continue(WorkerState(..state, jobs: new_jobs, processing: None))
    }

    JobFailed(job_id, reason) -> {
      let new_state = case dict.get(state.jobs, job_id) {
        Ok(tracked) -> {
          case tracked.attempts < tracked.max_attempts {
            True -> {
              let updated = TrackedJob(..tracked, status: Pending)
              let new_jobs = dict.insert(state.jobs, job_id, updated)
              let delay = 1000 * pow2(tracked.attempts - 1)
              process.send_after(state.self, delay, ProcessNext)
              WorkerState(
                ..state,
                jobs: new_jobs,
                queue: [job_id, ..state.queue],
                processing: None,
              )
            }
            False -> {
              let updated = TrackedJob(
                ..tracked,
                status: Failed("Max retries exceeded: " <> reason),
              )
              let new_jobs = dict.insert(state.jobs, job_id, updated)
              actor.send(state.self, ProcessNext)
              WorkerState(..state, jobs: new_jobs, processing: None)
            }
          }
        }
        Error(_) -> {
          actor.send(state.self, ProcessNext)
          WorkerState(..state, processing: None)
        }
      }
      actor.continue(new_state)
    }

    Tick -> {
      actor.continue(state)
    }
  }
}

// ── Job Execution ──────────────────────────────────────────────────

fn execute_job(
  worker: Subject(JobMessage),
  job_id: String,
  job: Job,
) -> Nil {
  case job {
    GenerateReport(board_id) -> {
      actor.send(worker, UpdateProgress(job_id, 10))
      process.sleep(1000)
      actor.send(worker, UpdateProgress(job_id, 30))
      process.sleep(1000)
      actor.send(worker, UpdateProgress(job_id, 60))
      process.sleep(1000)
      actor.send(worker, UpdateProgress(job_id, 90))
      process.sleep(1000)
      actor.send(worker, UpdateProgress(job_id, 100))
      actor.send(
        worker,
        JobCompleted(job_id, "/reports/" <> board_id <> ".html"),
      )
    }

    ProcessThumbnail(upload_id, file_path) -> {
      actor.send(worker, UpdateProgress(job_id, 20))
      process.sleep(2000)
      actor.send(worker, UpdateProgress(job_id, 100))
      actor.send(
        worker,
        JobCompleted(job_id, "/uploads/thumbs/" <> upload_id <> ".jpg"),
      )
    }

    SendNotifications(task_id, recipients) -> {
      let total = list.length(recipients)
      list.index_map(recipients, fn(recipient, index) {
        process.sleep(200)
        let progress = { { index + 1 } * 100 } / total
        actor.send(worker, UpdateProgress(job_id, progress))
      })
      actor.send(
        worker,
        JobCompleted(
          job_id,
          "Sent " <> int.to_string(total) <> " notifications",
        ),
      )
    }
  }
}

fn pow2(n: Int) -> Int {
  case n <= 0 {
    True -> 1
    False -> 2 * pow2(n - 1)
  }
}
```

The cleanup worker module:

```gleam
// src/teamwork/cleanup.gleam

import gleam/erlang/process.{type Subject}
import gleam/int
import gleam/otp/actor
import gleam/io

pub type CleanupMessage {
  Init(self: Subject(CleanupMessage))
  Tick
  RunNow
  GetStats(reply_to: Subject(CleanupStats))
}

pub type CleanupStats {
  CleanupStats(
    files_cleaned: Int,
    last_run_at: Int,
    next_run_in_ms: Int,
  )
}

pub type CleanupState {
  CleanupState(
    self: Subject(CleanupMessage),
    upload_dir: String,
    interval_ms: Int,
    files_cleaned_total: Int,
    last_run_at: Int,
  )
}

pub fn start(
  upload_dir: String,
) -> Result(Subject(CleanupMessage), actor.StartError) {
  let placeholder = process.new_subject()
  let interval = 3_600_000

  let initial_state = CleanupState(
    self: placeholder,
    upload_dir: upload_dir,
    interval_ms: interval,
    files_cleaned_total: 0,
    last_run_at: 0,
  )

  let result =
    actor.new(initial_state)
    |> actor.on_message(handle_cleanup_message)
    |> actor.start

  case result {
    Ok(started) -> {
      let subject = started.data
      actor.send(subject, Init(subject))
      process.send_after(subject, interval, Tick)
      io.println(
        "Cleanup worker started. First run in "
        <> int.to_string(interval / 1000)
        <> " seconds.",
      )
      Ok(subject)
    }
    Error(err) -> Error(err)
  }
}

fn handle_cleanup_message(
  state: CleanupState,
  message: CleanupMessage,
) -> actor.Next(CleanupState, CleanupMessage) {
  case message {
    Init(subject) -> {
      actor.continue(CleanupState(..state, self: subject))
    }

    Tick -> {
      let files_cleaned = delete_old_temp_files(state.upload_dir)
      io.println(
        "Cleanup: removed "
        <> int.to_string(files_cleaned)
        <> " old temporary files",
      )
      process.send_after(state.self, state.interval_ms, Tick)
      let now = current_time_ms()
      actor.continue(CleanupState(
        ..state,
        files_cleaned_total: state.files_cleaned_total + files_cleaned,
        last_run_at: now,
      ))
    }

    RunNow -> {
      let files_cleaned = delete_old_temp_files(state.upload_dir)
      io.println(
        "Manual cleanup: removed "
        <> int.to_string(files_cleaned)
        <> " old temporary files",
      )
      let now = current_time_ms()
      actor.continue(CleanupState(
        ..state,
        files_cleaned_total: state.files_cleaned_total + files_cleaned,
        last_run_at: now,
      ))
    }

    GetStats(reply_to) -> {
      let stats = CleanupStats(
        files_cleaned: state.files_cleaned_total,
        last_run_at: state.last_run_at,
        next_run_in_ms: state.interval_ms,
      )
      process.send(reply_to, stats)
      actor.continue(state)
    }
  }
}

fn delete_old_temp_files(upload_dir: String) -> Int {
  // Implementation depends on your file system library.
  // Walk the directory, check file ages, delete old ones.
  // Return the count of deleted files.
  0
}

fn current_time_ms() -> Int {
  // In a real application, use erlang:system_time(millisecond)
  // For now, return 0 as a placeholder
  0
}
```

The updated Context and router:

```gleam
// src/teamwork/web.gleam

import gleam/erlang/process.{type Subject}
import teamwork/state
import teamwork/jobs
import teamwork/cleanup

pub type Context {
  Context(
    tasks: Subject(state.Message),
    job_worker: Subject(jobs.JobMessage),
    cleanup_worker: Subject(cleanup.CleanupMessage),
  )
}
```

```gleam
// src/teamwork/router.gleam (relevant routes)

import gleam/erlang/process
import gleam/int
import gleam/otp/actor
import lustre/element
import lustre/element/html
import lustre/attribute.{attribute}
import hx
import wisp
import teamwork/jobs
import teamwork/cleanup
import teamwork/views
import teamwork/web.{type Context}

pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())

  case wisp.path_segments(req) {
    // ... existing routes ...

    // Background job routes
    ["reports", "generate"] ->
      handle_generate_report(req, ctx)

    ["jobs", job_id, "status"] ->
      handle_job_status(req, ctx, job_id)

    ["jobs", job_id, "thumbnail"] ->
      handle_thumbnail_status(req, ctx, job_id)

    // Admin routes
    ["admin", "cleanup"] ->
      handle_admin_cleanup(req, ctx)

    ["admin", "cleanup", "stats"] ->
      handle_cleanup_stats(req, ctx)

    _ -> wisp.not_found()
  }
}

fn handle_generate_report(req: wisp.Request, ctx: Context) -> wisp.Response {
  let job_id = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) { jobs.Enqueue(jobs.GenerateReport("board-1"), reply) },
  )

  let html = views.job_progress_fragment(job_id, jobs.Pending)
  wisp.html_response(element.to_string(html), 202)
}

fn handle_job_status(
  req: wisp.Request,
  ctx: Context,
  job_id: String,
) -> wisp.Response {
  let status = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) { jobs.GetStatus(job_id, reply) },
  )

  let html = views.job_progress_fragment(job_id, status)
  wisp.html_response(element.to_string(html), 200)
}

fn handle_thumbnail_status(
  req: wisp.Request,
  ctx: Context,
  job_id: String,
) -> wisp.Response {
  let status = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) { jobs.GetStatus(job_id, reply) },
  )

  let html = case status {
    jobs.Complete(thumbnail_path) ->
      html.div([attribute("class", "thumbnail")], [
        html.img([
          attribute("src", thumbnail_path),
          attribute("alt", "Attachment thumbnail"),
        ]),
      ])
    jobs.Failed(_reason) ->
      html.div([attribute("class", "thumbnail-fallback")], [
        html.text("Preview unavailable"),
      ])
    _ ->
      html.div(
        [
          attribute("class", "thumbnail-placeholder"),
          hx.get("/jobs/" <> job_id <> "/thumbnail"),
          hx.trigger_polling(
            timing: duration.milliseconds(2000),
            filters: option.None,
            on_load: True,
          ),
          hx.swap(hx.OuterHTML),
        ],
        [
          html.span([attribute("class", "spinner")], []),
          html.text("Generating thumbnail..."),
        ],
      )
  }
  wisp.html_response(element.to_string(html), 200)
}

fn handle_admin_cleanup(req: wisp.Request, ctx: Context) -> wisp.Response {
  actor.send(ctx.cleanup_worker, cleanup.RunNow)

  let html = html.p([], [
    html.text("Cleanup triggered. Check the server logs for results."),
  ])
  wisp.html_response(element.to_string(html), 200)
}

fn handle_cleanup_stats(req: wisp.Request, ctx: Context) -> wisp.Response {
  let stats = actor.call(
    ctx.cleanup_worker,
    5000,
    cleanup.GetStats,
  )

  let html = html.div(
    [attribute("id", "cleanup-status")],
    [
      html.p([], [
        html.text(
          "Files cleaned (total): " <> int.to_string(stats.files_cleaned),
        ),
      ]),
      html.p([], [
        html.text(
          "Next run in: "
          <> int.to_string(stats.next_run_in_ms / 60_000)
          <> " minutes",
        ),
      ]),
    ],
  )
  wisp.html_response(element.to_string(html), 200)
}

fn static_directory() -> String {
  let assert Ok(priv) = wisp.priv_directory("teamwork")
  priv <> "/static"
}
```

The main module wiring:

```gleam
// src/teamwork.gleam

import gleam/erlang/process
import mist
import wisp
import wisp/wisp_mist
import teamwork/state
import teamwork/jobs
import teamwork/cleanup
import teamwork/router
import teamwork/web.{type Context, Context}

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)

  // Start the task state actor
  let assert Ok(task_store) = state.start()

  // Start background workers
  let assert Ok(job_worker) = jobs.start()
  let assert Ok(cleanup_worker) = cleanup.start("./uploads/tmp")

  // Build the context
  let ctx = Context(
    tasks: task_store,
    job_worker: job_worker,
    cleanup_worker: cleanup_worker,
  )

  // Start the HTTP server
  let assert Ok(_) =
    wisp_mist.handler(
      fn(req) { router.handle_request(req, ctx) },
      secret_key_base,
    )
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

---

## 4. Exercises

### Exercise 1: Email Notification Job

Add a new job type that sends email notifications when a task is assigned to a user. When a user assigns a task, the handler should enqueue a `SendAssignmentEmail` job and return immediately. The HTMX interface should show a small "Notification sent" toast after the job completes.

**Acceptance Criteria:**
- A new `SendAssignmentEmail(task_id: String, assignee_email: String)` variant exists in the `Job` type.
- The `execute_job` function handles the new variant (simulate the email with `process.sleep(1000)`).
- The task assignment handler enqueues the job and returns `202 Accepted`.
- The response includes an `HX-Trigger: notificationSent` header.
- A listener element with `hx-trigger="notificationSent from:body"` shows a temporary success message.

### Exercise 2: Job Dashboard

Build an admin dashboard that displays all jobs and their statuses. The dashboard should auto-refresh every five seconds using polling.

**Acceptance Criteria:**
- A `GET /admin/jobs` endpoint returns an HTML table of all tracked jobs.
- Each row shows the job ID, type, status, attempt count, and progress.
- The table uses `hx.trigger_polling(timing: duration.milliseconds(5000), filters: option.None, on_load: True)` to refresh automatically.
- A new `ListAllJobs(reply_to: Subject(List(TrackedJob)))` message variant is added to the job worker.
- Completed and failed jobs are styled differently (green for complete, red for failed).

### Exercise 3: Job Cancellation

Add the ability to cancel a pending or in-progress job. The HTMX interface should show a "Cancel" button next to each active job.

**Acceptance Criteria:**
- A new `CancelJob(job_id: String, reply_to: Subject(Bool))` message variant is added to the job worker.
- Pending jobs are removed from the queue and marked as `Failed("Cancelled by user")`.
- In-progress jobs are marked as `Failed("Cancelled by user")` (the executing process will complete but its result is ignored).
- The handler returns `200` with the updated job fragment on success, `404` if the job does not exist.
- Each job row in the dashboard (from Exercise 2) has a "Cancel" button that uses `hx.post("/admin/jobs/" <> job_id <> "/cancel")`.

### Exercise 4: Rate-Limited Periodic Digest

Build a periodic worker that sends a daily summary of task activity. Instead of sending the digest immediately, it should batch activity and send once every 24 hours.

**Acceptance Criteria:**
- A `DigestWorker` actor collects `TaskEvent` messages (task created, completed, deleted) throughout the day.
- Every 24 hours (use a shorter interval like 30 seconds for testing), the worker formats a summary string and logs it (simulating an email send).
- The worker uses `process.send_after(state.self, interval, SendDigest)` for self-scheduling.
- After sending, the event buffer is cleared and the cycle restarts.
- An admin endpoint `GET /admin/digest/preview` shows what the next digest will contain without sending it.

### Exercise 5: Concurrent Worker Pool

Instead of processing jobs one at a time, build a pool of three worker processes that can handle jobs concurrently. Jobs should be distributed to available workers.

**Acceptance Criteria:**
- The `WorkerState` tracks three worker slots instead of a single `processing` field.
- When a job is enqueued and a slot is available, it starts immediately.
- When all three slots are busy, the job waits in the queue.
- When a job completes, the slot is freed and the next queued job starts.
- The job dashboard (from Exercise 2) shows which worker slot each in-progress job is using.
- The system correctly handles the case where a worker slot crashes (mark the job as failed and free the slot).

---

## 5. Exercise Solution Hints

### Hint for Exercise 1

Add the new variant to the `Job` type and a case clause to `execute_job`. The HTTP handler is straightforward:

```gleam
fn handle_assign_task(req: wisp.Request, ctx: Context, task_id: String) -> wisp.Response {
  // ... parse the assignee from the form ...

  let _job_id = actor.call(
    ctx.job_worker,
    5000,
    fn(reply) {
      jobs.Enqueue(
        jobs.SendAssignmentEmail(task_id, assignee_email),
        reply,
      )
    },
  )

  wisp.response(202)
  |> wisp.set_header("HX-Trigger", "notificationSent")
}
```

For the toast, use a listener element in the layout:

```gleam
html.div(
  [
    attribute("id", "toast-container"),
    attribute("hx-trigger", "notificationSent from:body"),
    hx.get("/toasts/notification-sent"),
    hx.swap(hx.Beforeend),
  ],
  [],
)
```

The toast endpoint returns a small `<div>` with a CSS animation that fades out after a few seconds. You can use `_hyperscript` (from [Appendix B](../05-appendices/A2-hyperscript.md)) for the auto-dismiss: `attribute("_", "on load wait 3s then remove me")`.

### Hint for Exercise 2

Add a `ListAllJobs` variant to `JobMessage`:

```gleam
ListAllJobs(reply_to: Subject(List(TrackedJob)))
```

In the handler, reply with all tracked jobs:

```gleam
ListAllJobs(reply_to) -> {
  let all_jobs = dict.values(state.jobs)
  process.send(reply_to, all_jobs)
  actor.continue(state)
}
```

The view function maps over the list and renders a table row for each job. Use pattern matching on `tracked.status` to apply different CSS classes.

### Hint for Exercise 3

For cancellation, you need to distinguish between pending and in-progress jobs:

```gleam
CancelJob(job_id, reply_to) -> {
  case dict.get(state.jobs, job_id) {
    Ok(tracked) -> {
      let updated = TrackedJob(..tracked, status: Failed("Cancelled by user"))
      let new_jobs = dict.insert(state.jobs, job_id, updated)
      let new_queue = list.filter(state.queue, fn(id) { id != job_id })
      process.send(reply_to, True)
      actor.continue(WorkerState(..state, jobs: new_jobs, queue: new_queue))
    }
    Error(_) -> {
      process.send(reply_to, False)
      actor.continue(state)
    }
  }
}
```

For in-progress jobs, you cannot stop the spawned process easily (this is by design -- BEAM processes are autonomous). Instead, mark the job as cancelled in the actor state. When the executing process sends `JobCompleted`, check if the job was cancelled and ignore the result.

### Hint for Exercise 4

The digest worker state holds an event buffer:

```gleam
pub type DigestState {
  DigestState(
    self: Subject(DigestMessage),
    events: List(TaskEvent),
    interval_ms: Int,
  )
}
```

When `SendDigest` arrives, format the events into a summary, clear the buffer, and reschedule:

```gleam
SendDigest -> {
  let summary = format_digest(state.events)
  io.println("Daily digest: " <> summary)
  process.send_after(state.self, state.interval_ms, SendDigest)
  actor.continue(DigestState(..state, events: []))
}
```

Other actors (like the task state actor) send `TaskEvent` messages to the digest worker whenever tasks change. The digest worker appends them to its buffer.

### Hint for Exercise 5

Replace the `processing: Option(String)` field with a dict of worker slots:

```gleam
pub type WorkerState {
  WorkerState(
    // ... other fields ...
    slots: Dict(Int, Option(String)),  // slot_id -> Option(job_id)
    max_slots: Int,
  )
}
```

Initialize with three empty slots:

```gleam
let slots = dict.from_list([
  #(1, None),
  #(2, None),
  #(3, None),
])
```

In `ProcessNext`, find the first available slot (one with `None`). In `JobCompleted` and `JobFailed`, free the slot by setting it back to `None`. The rest of the logic is the same -- you just check `dict.values(state.slots) |> list.any(fn(s) { s == None })` to decide whether to process the next job.

---

## 6. Key Takeaways

1. **Never make users wait for background work.** If an operation takes more than a second or two, move it to a background process. Return `202 Accepted` with a tracking ID and let the client check back. Users will tolerate almost anything if you show them progress.

2. **The BEAM makes background jobs trivial.** Other frameworks need Redis, Sidekiq, Celery, or Bull. On the BEAM, `process.start(fn() { ... }, linked: False)` is all you need for fire-and-forget work. For queued work, an actor with a list in its state is a complete job queue.

3. **Actors are natural job queues.** An actor processes messages one at a time, maintains ordered state, and communicates through typed messages. This maps perfectly onto the job queue pattern: enqueue, process, report status, retry on failure.

4. **Polling with self-cancellation is the simplest progress pattern.** Return HTML fragments with `hx-trigger="every 2s"` while the job is running. When the job finishes, return a fragment without the trigger. Polling stops automatically. No JavaScript needed.

5. **`process.send_after` enables periodic jobs without cron.** The self-rescheduling pattern -- do work, schedule next tick, repeat -- is simple, reliable, and built into the runtime. The interval adapts naturally because it schedules after the work completes, not at fixed wall-clock times.

6. **"Let it crash" is a production strategy, not recklessness.** Supervision trees restart failed processes automatically. The key prerequisite is isolation: BEAM processes do not share memory, so a crash in one cannot corrupt another. Write clean code for the normal case. Let supervisors handle the abnormal cases.

7. **In-memory job queues need database backing for durability.** An actor's state is lost when it restarts. For production systems, persist the job queue to the database. Load pending jobs on startup. This combines the speed of in-memory processing with the safety of durable storage.

8. **Background jobs and HTMX compose naturally.** HTMX's polling, SSE, and event-triggering capabilities map directly onto the three phases of background work: accept (return 202), wait (poll or SSE), complete (swap in the result). The server sends HTML at every stage. No JSON APIs. No client-side state management.

9. **Start simple, add complexity as needed.** Fire-and-forget covers most cases. Add a job queue when you need ordering and retry. Add progress reporting when users need visibility. Add supervision when you need fault tolerance. Add database persistence when you need durability. Each layer is independent and can be added incrementally.

---

## What's Next

This is the final chapter of the Real-World Patterns section. Let us take a moment to look at how far you have come.

### What We Covered in Chapters 19-28

Over the last ten chapters, you moved from a working application to a production-grade system:

- **[Chapter 19](19-error-handling-and-graceful-degradation.md)** introduced error handling and graceful degradation -- structured error responses, retry patterns, and fallback UI so the application stays usable when things go wrong.
- **[Chapter 20](20-response-headers-and-server-control.md)** covered HTMX response headers and server-driven control -- using `HX-Trigger`, `HX-Redirect`, `HX-Retarget`, and other response headers to orchestrate client behavior from the server.
- **[Chapter 21](21-inline-editing.md)** tackled inline editing with click-to-edit patterns -- swapping display elements for edit forms in place, saving with `hx-put`, and cancelling gracefully.
- **[Chapter 22](22-modal-dialogs.md)** explored modal dialogs via HTMX -- loading modal content from the server, managing open/close state, and handling form submissions inside modals.
- **[Chapter 23](23-hyperscript-practical-patterns.md)** built practical \_hyperscript patterns -- combining HTMX with \_hyperscript for client-side interactions like toggling, animations, and keyboard shortcuts.
- **[Chapter 24](24-dynamic-and-dependent-forms.md)** addressed dynamic and dependent forms -- cascading selects, conditional fields, and adding/removing form sections dynamically with HTMX.
- **[Chapter 25](25-file-uploads.md)** implemented file uploads with progress indicators, using HTMX's built-in multipart support and background processing.
- **[Chapter 26](26-accessible-htmx.md)** added keyboard shortcuts and accessibility, making the task board usable for everyone, not just mouse users.
- **[Chapter 27](27-database-performance.md)** covered database performance -- connection pooling, query optimization, indexing, and the N+1 problem.
- **[Chapter 28](28-background-jobs.md)** (this chapter) built background job processing, periodic tasks, and supervision trees, using the BEAM's native concurrency to handle work that does not belong in HTTP handlers.

Together, these chapters covered the patterns that separate a tutorial project from a real application. Not every application needs every pattern, but now you know they exist and how to implement them.

### Where to Go from Here

**[Appendix B](../05-appendices/A2-hyperscript.md)** covers \_hyperscript, HTMX's companion client-side scripting language. If you have been using `attribute("_", "...")` in your code and want to understand it more deeply, start there.

**[Appendix A](../05-appendices/A1-bibliography-and-resources.md)** is the bibliography and resource guide. It collects the official documentation, homepages, repositories, and video channels for every tool and technology used in this course. Bookmark the ones you use most.

### Build Something

The most important thing you can do next is build something of your own.

You have twenty-eight chapters of patterns, techniques, and working code. You have a backend language (Gleam) that catches errors at compile time and runs on the most battle-tested concurrent runtime in existence. You have a frontend approach (HTMX) that eliminates the complexity of client-side frameworks by putting the server back in charge of rendering.

This is a powerful combination. It is also an unusual one. Most developers have never seen a stack like this. That means the documentation is thinner, the Stack Overflow answers are fewer, and the community is smaller. You will get stuck. You will have to read source code. You will have to figure things out.

That is not a weakness. That is how expertise is built.

Pick a project that matters to you. A tool you wish existed. A problem at work that nobody has solved well. A side project that has been rattling around in your head. Start small. Get something working. Then iterate.

The gap between "I followed a course" and "I can build things" is bridged by building things. You have the tools. You have the knowledge. You have twenty-eight chapters of proof that this stack works.

Go build something.

---

*This is the final chapter of the HTMX with Gleam course. The appendices contain additional reference material: **[Appendix A](../05-appendices/A1-bibliography-and-resources.md)** collects documentation links, homepages, and resources for every tool used in this course, and **[Appendix B](../05-appendices/A2-hyperscript.md)** covers \_hyperscript.*

*Thank you for reading.*
