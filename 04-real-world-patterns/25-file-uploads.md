# Chapter 25 -- File Uploads

**Phase:** Real-World Patterns
**Project:** "Teamwork" -- a collaborative task board
**Previous:** Chapter 24 covered dynamic forms. Now we tackle one of the most
common real-world requirements: letting users attach files to tasks.

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain `application/x-www-form-urlencoded` vs `multipart/form-data` and when each is used.
- Configure HTMX file upload with `hx-encoding="multipart/form-data"`.
- Parse uploads in Wisp using `wisp.require_form` with its `.files` field.
- Validate file type and size server-side, returning 422 for rejected files.
- Store files with sanitized unique filenames and serve them with download links.
- Display upload progress using `htmx:xhr:progress` and `_hyperscript`.

---

## 1. Theory

### 1.1 File Upload Fundamentals

Back in Chapter 7, we built forms that submit text data. Every form we have
written so far uses the default encoding: `application/x-www-form-urlencoded`.
This encoding turns form fields into a flat string of key-value pairs:

```
title=Fix+the+login+bug&description=Users+cannot+log+in
```

It works perfectly for text. But it cannot carry files. A file is binary data --
an image, a PDF, a spreadsheet. You cannot percent-encode a 5 MB PNG into a URL
query string. Well, you *could*, but the result would be enormous, slow to
encode, and impossible to stream.

This is why `multipart/form-data` exists. It is the second encoding type that
HTML forms support, and it was designed specifically for file uploads. Instead of
encoding everything into a single string, it splits the request body into
*parts*, each with its own headers and content:

```
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="title"

Fix the login bug
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="attachment"; filename="screenshot.png"
Content-Type: image/png

<binary PNG data here>
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

Each part is separated by a boundary string. Text fields are included as plain
text. File fields include the original filename, the MIME type, and the raw
binary content. The browser handles all of this automatically when you set
`enctype="multipart/form-data"` on a `<form>`.

Here is a comparison:

| Property                     | `x-www-form-urlencoded`              | `multipart/form-data`                 |
|------------------------------|--------------------------------------|---------------------------------------|
| **Use case**                 | Text-only forms                      | Forms with file uploads               |
| **Encoding**                 | Key=value pairs, percent-encoded     | Parts separated by boundary           |
| **Binary data**              | Not supported                        | Fully supported                       |
| **Size overhead**            | Moderate (percent-encoding inflates) | Low (binary sent as-is)               |
| **Default for `<form>`**     | Yes                                  | No -- must be explicit                |
| **Streaming**                | Not practical                        | Supported by servers                  |

The rule is simple: if your form has a file input, use `multipart/form-data`.
If it does not, use the default `x-www-form-urlencoded`. There is no performance
penalty for using multipart on text-only forms, but there is no benefit either,
so stick with the default when files are not involved.

### 1.2 HTMX File Upload Configuration

In traditional HTML, you tell the browser to use multipart encoding with the
`enctype` attribute on the `<form>` element:

```html
<form method="post" action="/upload" enctype="multipart/form-data">
  <input type="file" name="attachment" />
  <button type="submit">Upload</button>
</form>
```

With HTMX, the form submission is intercepted by JavaScript. HTMX needs its own
way to know that the request should use multipart encoding. That attribute is
`hx-encoding`:

```html
<form hx-post="/upload" hx-encoding="multipart/form-data">
  <input type="file" name="attachment" />
  <button type="submit">Upload</button>
</form>
```

When `hx-encoding="multipart/form-data"` is present, HTMX uses a `FormData`
object (the browser's built-in API for multipart requests) instead of
serializing the form fields into a URL-encoded string. The `FormData` object
automatically includes any files selected in `<input type="file">` elements.

Without this attribute, HTMX will serialize the form as URL-encoded data. The
file input will be ignored -- no file data will reach the server. This is one of
the most common mistakes when adding file uploads to an HTMX application. If
your file is not arriving on the server, check `hx-encoding` first.

In Lustre, you set this attribute using `attribute.attribute`:

```gleam
attribute.attribute("hx-encoding", "multipart/form-data")
```

There is no dedicated function for `hx-encoding` in the `hx` library, so we use
the general `attribute.attribute(name, value)` function. This is the same
pattern we used in Chapter 18 for custom attributes.

You can also place `hx-encoding` on individual elements rather than the form.
For example, if you have a button with `hx-post` outside a form, you can set
`hx-encoding` directly on that button and use `hx-include` to reference a file
input. But the most common and clearest pattern is to put it on the form.

### 1.3 Server-Side File Handling in Wisp

When a multipart request arrives at your Wisp server, the same
`wisp.require_form` function you already know handles it. The difference is in
what it gives you back.

Recall from Chapter 7 that `wisp.require_form` returns a `FormData` value:

```gleam
use form_data <- wisp.require_form(req)
```

The `FormData` type has two fields:

```gleam
FormData(
  values: List(#(String, String)),
  files: List(#(String, UploadedFile)),
)
```

Until now, we have only used `form_data.values` -- the list of text field
key-value pairs. The `form_data.files` field has been sitting there the whole
time, waiting for this chapter.

Each entry in `form_data.files` is a tuple of `#(String, UploadedFile)` where
the first element is the field name (matching the `name` attribute on the
`<input type="file">`) and the second is an `UploadedFile` record:

```gleam
UploadedFile(
  file_name: String,   // The original filename from the user's computer
  path: String,         // A temporary file path on the server's disk
)
```

Two important things to understand:

1. **`file_name`** is the original name of the file as it was on the user's
   machine. This is user-supplied data. You must never trust it blindly. It
   could contain path separators (`../../../etc/passwd`), special characters,
   or be absurdly long. Always sanitize it before using it in a file path.

2. **`path`** is a temporary file path where Wisp has already written the
   uploaded data to disk. Wisp streams the upload to a temp file during parsing,
   so by the time your handler runs, the file bytes are on disk. You need to
   move or copy this file to a permanent location before the request ends,
   because the temp file may be cleaned up afterward.

The flow for handling an upload is:

```
Request arrives
  -> wisp.require_form parses the multipart body
  -> Text fields go into form_data.values
  -> File fields go into form_data.files (written to temp directory)
  -> Your handler reads form_data.files
  -> You validate the file (type, size, name)
  -> You copy/move the file from temp path to permanent storage
  -> You store a reference (filename, path) in your database
  -> You return an HTML response
```

### 1.4 Validation: Type, Size, and Safety

File uploads are one of the most dangerous features you can add to a web
application. An unrestricted upload endpoint is an invitation for abuse. Here
are the threats and how to defend against each:

**File type validation.**
Users should only be able to upload the file types your application expects.
For a task board, that might be images (PNG, JPG, GIF) and documents (PDF).
You do not want someone uploading an executable or a script.

The simplest validation is by file extension:

```gleam
fn is_allowed_extension(filename: String) -> Bool {
  let lower = string.lowercase(filename)
  string.ends_with(lower, ".png")
  || string.ends_with(lower, ".jpg")
  || string.ends_with(lower, ".jpeg")
  || string.ends_with(lower, ".gif")
  || string.ends_with(lower, ".pdf")
}
```

This is not foolproof -- someone can rename `malware.exe` to `malware.png` --
but it stops casual mistakes and provides a first line of defense. For stronger
validation, you would check the file's magic bytes (the first few bytes that
identify the format). That is beyond our scope, but the extension check is a
reasonable starting point.

**File size validation.**
Without a size limit, a single user could fill your server's disk with a
multi-gigabyte upload. Set a maximum and enforce it. Wisp provides a
configuration option to limit request body size, and you should also check the
file size in your handler:

```gleam
fn is_within_size_limit(path: String) -> Bool {
  case simplifile.file_info(path) {
    Ok(info) -> info.size <= 10_000_000  // 10 MB
    Error(_) -> False
  }
}
```

**Filename sanitization.**
Never use the user-supplied filename directly in a file path. Consider what
happens if the filename is `../../../etc/passwd` or
`<script>alert('xss')</script>.html`. Instead, generate a new filename on the
server:

```gleam
fn safe_filename(original: String) -> String {
  let ext = get_extension(original)
  let unique = wisp.random_string(16)
  unique <> ext
}
```

This produces filenames like `a3f8b2c1d9e4f507.png` -- unique, safe, and
predictable in length.

**Extension extraction** needs care too. You want the *last* dot-separated
segment, and you want to ensure it is from your allowlist:

```gleam
fn get_extension(filename: String) -> String {
  case string.split(filename, ".") {
    [_] -> ""
    parts -> {
      let assert Ok(ext) = list.last(parts)
      "." <> string.lowercase(ext)
    }
  }
}
```

Here is a summary of validation checks:

| Check                  | Why                                           | Response on failure |
|------------------------|-----------------------------------------------|---------------------|
| Extension allowlist    | Reject unexpected file types                  | 422 with error msg  |
| Size limit             | Prevent disk exhaustion                       | 422 with error msg  |
| Filename sanitization  | Prevent path traversal and XSS in filenames   | Generate safe name  |
| Non-empty file         | Reject empty uploads                          | 422 with error msg  |

### 1.5 Progress Indicators

File uploads take time, especially on slow connections. A user uploading a 5 MB
image on a mobile connection might wait 30 seconds or more. Without feedback,
they will wonder if the upload is stuck, click the button again (creating a
duplicate), or give up and leave.

HTMX provides access to upload progress through the `htmx:xhr:progress` event.
This event fires periodically during the upload with two key pieces of data:

- `loaded` -- the number of bytes uploaded so far
- `total` -- the total number of bytes to upload

These two values let you calculate a percentage and display a progress bar.

The event fires on the element that initiated the request -- typically the form
or the button with the `hx-post` attribute. You can listen for it with
JavaScript or with `_hyperscript`.

Using `_hyperscript`, the pattern looks like this:

```
on htmx:xhr:progress(loaded, total)
  set pct to (loaded / total) * 100
  put pct + '%' into #progress-text
  set #progress-bar's style.width to pct + '%'
```

This is a declarative script placed directly on the form. As the upload
progresses, it calculates the percentage, updates a text element, and sets the
width of a progress bar element. No external JavaScript file needed.

There are two related events:

| Event                  | When it fires                    | Useful for                        |
|------------------------|----------------------------------|-----------------------------------|
| `htmx:xhr:progress`   | Periodically during upload       | Progress bar, percentage display  |
| `htmx:xhr:loadstart`  | When the upload begins           | Showing the progress bar          |
| `htmx:xhr:loadend`    | When the upload completes        | Hiding the progress bar           |

The progress bar itself is a simple HTML element -- typically a `<div>` inside
a container `<div>`. The inner div has its `width` style set as a percentage:

```html
<div class="progress-container">
  <div id="progress-bar" class="progress-bar" style="width: 0%"></div>
</div>
<span id="progress-text">0%</span>
```

CSS handles the visual styling -- background color, height, border radius,
transition for smooth animation.

### 1.6 Security

File uploads introduce security concerns that go beyond input validation. Even
after you validate the file type and size, you need to think about how the file
is stored and served.

**Storage location.**
Uploaded files should live *outside* your application's source tree. Store them
in a dedicated directory like `./uploads/` or a configurable path from an
environment variable. This directory should never be inside `priv/static/` or
any directory that serves executable code. If an attacker uploads a `.beam` file
or a shell script, you do not want the server to execute it.

**Serving files safely.**
When a user downloads an uploaded file, the server should set the
`Content-Disposition` header to tell the browser how to handle it:

```
Content-Disposition: attachment; filename="report.pdf"
```

The `attachment` value forces the browser to download the file rather than
trying to render it inline. This prevents an uploaded HTML file from being
rendered in the browser (which could execute JavaScript -- an XSS vector).

For images that you want to display inline (in an `<img>` tag, for example),
you can use `Content-Disposition: inline`. But only do this for known safe
types (PNG, JPG, GIF, WebP) and never for HTML, SVG, or anything that could
contain scripts.

**No execution.**
The upload directory should have no execute permissions. On Linux, set the
directory permissions to `755` and file permissions to `644`. The web server
should serve files from this directory as static content only -- never pass
them through an interpreter or execution engine.

**Path traversal.**
We already covered filename sanitization, but it bears repeating: never
construct a file path by concatenating user input. Always generate filenames
on the server side. A filename like `../../config/database.yml` could escape
the upload directory and overwrite critical files.

Here is the security checklist for file uploads:

| Measure                       | Purpose                                        |
|-------------------------------|------------------------------------------------|
| Extension allowlist           | Block unexpected file types                    |
| Size limit                    | Prevent disk exhaustion                        |
| Generated filenames           | Prevent path traversal                         |
| `Content-Disposition: attachment` | Prevent browser execution of uploads       |
| External storage directory    | Keep uploads away from application code        |
| No execute permissions        | Prevent server-side execution of uploads       |

---

## 2. Code Walkthrough

We are going to add file attachments to tasks in the Teamwork app. Users will
be able to attach files when creating or editing a task, see the attachments
listed on the task, download them, and delete them. We will also add a progress
bar so uploads feel responsive.

### Where We Left Off

After 24 chapters, our Teamwork app has forms, validation, authentication,
real-time updates, and dynamic form behavior. Task forms submit text data with
`hx-post` and `application/x-www-form-urlencoded` encoding. We have static file
serving for CSS and JavaScript via `wisp.serve_static`. Now we extend the task
form to support file attachments.

### Step 1 -- Add File Input with `hx-encoding` to the Task Form

The first change is in the view layer. We need to add a file input to the task
creation form and tell HTMX to use multipart encoding.

Here is the updated form function:

```gleam
import lustre/element.{type Element}
import lustre/element/html
import lustre/attribute
import hx

fn attachment_upload_form(task_id: String) -> Element(t) {
  html.form(
    [
      attribute.id("upload-form"),
      hx.post("/tasks/" <> task_id <> "/attachments"),
      hx.target(hx.Selector("#attachment-list-" <> task_id)),
      hx.swap(hx.Beforeend),
      attribute.attribute("hx-encoding", "multipart/form-data"),
      attribute.attribute(
        "_",
        "on htmx:xhr:progress(loaded, total)
           set pct to (loaded / total) * 100
           put pct + '%' into #progress-text
           set #progress-bar's style.width to pct + '%'
         on htmx:afterRequest
           set #progress-bar's style.width to '0%'
           put '' into #progress-text",
      ),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("attachment")], [
          element.text("Attach a file"),
        ]),
        html.input([
          attribute.type_("file"),
          attribute.name("attachment"),
          attribute.id("attachment"),
          attribute.accept([".png", ".jpg", ".jpeg", ".gif", ".pdf"]),
        ]),
      ]),
      // Progress bar
      html.div([attribute.class("progress-container")], [
        html.div(
          [
            attribute.id("progress-bar"),
            attribute.class("progress-bar"),
            attribute.attribute("style", "width: 0%"),
          ],
          [],
        ),
      ]),
      html.span([attribute.id("progress-text")], []),
      html.button(
        [attribute.type_("submit"), attribute.class("btn-secondary")],
        [element.text("Upload")],
      ),
    ],
  )
}
```

Let us walk through every important detail.

**`attribute.attribute("hx-encoding", "multipart/form-data")`** -- This is the
critical attribute. Without it, HTMX will serialize the form as URL-encoded data
and the file will be silently dropped. The server will receive the text fields
but no file. If your upload is not working, this is the first thing to check.

**`hx.post("/tasks/" <> task_id <> "/attachments")`** -- We send the upload to a
dedicated endpoint for attachments, scoped to a specific task. This keeps the
attachment logic separate from the task creation/update logic.

**`hx.target(hx.Selector("#attachment-list-" <> task_id))`** -- The server's
response (an HTML fragment for the new attachment) will be appended to the
attachment list for this task. Note the use of `hx.Selector` -- not a raw
string.

**`hx.swap(hx.Beforeend)`** -- The new attachment is appended to the end of
the list. Note the casing: `Beforeend`, not `BeforeEnd`.

**`attribute.type_("file")`** -- This creates `<input type="file">`. The
browser renders a file chooser button. When the user selects a file, it is
held in the browser's memory until the form is submitted.

**`attribute.accept([".png", ".jpg", ".jpeg", ".gif", ".pdf"])`** -- The
`accept` attribute tells the browser to filter the file chooser dialog. The
user will see only files with these extensions by default. This is a UX
convenience, not a security measure -- the user can still select "All Files"
and choose anything. Server-side validation is still mandatory.

**`attribute.name("attachment")`** -- The name that will identify this file in
`form_data.files`. When we parse the upload on the server, we will look for
this name.

**The `_hyperscript` attribute** -- Two event handlers in one attribute. The
`htmx:xhr:progress` handler updates the progress bar during upload. The
`htmx:afterRequest` handler resets the progress bar after the upload completes
(whether it succeeded or failed). We will examine the progress bar in detail
in Step 5.

### Step 2 -- Parse Upload on the Server

Now we need the server-side endpoint that receives the upload, validates it,
and stores it. Here is the route handler:

```gleam
import gleam/http
import gleam/list
import gleam/string
import gleam/int
import gleam/result
import wisp
import simplifile
import lustre/element
import lustre/element/html
import lustre/attribute

fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())
  use <- wisp.serve_static(req, under: "/uploads", from: upload_directory())

  case wisp.path_segments(req) {
    // ... existing routes ...

    ["tasks", task_id, "attachments"] -> {
      case req.method {
        http.Post -> upload_attachment(req, ctx, task_id)
        _ -> wisp.method_not_allowed([http.Post])
      }
    }

    ["tasks", task_id, "attachments", attachment_id] -> {
      case req.method {
        http.Delete -> delete_attachment(req, ctx, task_id, attachment_id)
        _ -> wisp.method_not_allowed([http.Delete])
      }
    }

    _ -> wisp.not_found()
  }
}
```

Two things to notice:

**`wisp.serve_static(req, under: "/uploads", from: upload_directory())`** --
We add a second `serve_static` call that serves uploaded files from the upload
directory. This reuses the same pattern from Chapter 17 where we set up static
file serving for CSS and JavaScript. Any file in the upload directory will be
accessible at `/uploads/<filename>`.

**Two new routes.** `POST /tasks/:id/attachments` for uploading, and
`DELETE /tasks/:id/attachments/:attachment_id` for deleting. REST-style
resource naming keeps the API intuitive.

Now the upload handler:

```gleam
fn upload_attachment(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  // Parse the multipart form data.
  // Wisp handles the multipart parsing automatically.
  // Text fields go into form_data.values, files into form_data.files.
  use form_data <- wisp.require_form(req)

  // Look for the file field named "attachment".
  case list.key_find(form_data.files, "attachment") {
    Error(_) -> {
      // No file was submitted.
      let html = render_upload_error("No file selected.")
      wisp.html_response(html, 422)
    }

    Ok(uploaded_file) -> {
      // Validate the file.
      case validate_upload(uploaded_file) {
        Error(reason) -> {
          let html = render_upload_error(reason)
          wisp.html_response(html, 422)
        }

        Ok(_) -> {
          // Generate a safe, unique filename.
          let ext = get_extension(uploaded_file.file_name)
          let safe_name = wisp.random_string(16) <> ext
          let dest = upload_directory() <> "/" <> safe_name

          // Move the file from temp storage to permanent storage.
          case simplifile.copy(uploaded_file.path, dest) {
            Ok(_) -> {
              // Store the attachment record in the database.
              let attachment = Attachment(
                id: wisp.random_string(8),
                task_id: task_id,
                original_name: sanitize_display_name(uploaded_file.file_name),
                stored_name: safe_name,
              )
              let _ = save_attachment(ctx.db, attachment)

              // Return an HTML fragment for the new attachment.
              let html =
                render_attachment(attachment)
                |> element.to_string()
              wisp.html_response(html, 201)
            }

            Error(_) -> {
              let html = render_upload_error("Failed to store file.")
              wisp.html_response(html, 500)
            }
          }
        }
      }
    }
  }
}
```

Let us trace through this step by step.

**`use form_data <- wisp.require_form(req)`** -- Same function we used in
Chapter 7, but this time Wisp detects the `multipart/form-data` content type
and parses the multipart body. Text fields still go into `form_data.values`.
Files go into `form_data.files`.

**`list.key_find(form_data.files, "attachment")`** -- We search the files list
for the entry named `"attachment"`, matching the `name` attribute on the file
input. This returns `Ok(uploaded_file)` if a file was submitted, or `Error(Nil)`
if the field was empty or missing.

**`validate_upload(uploaded_file)`** -- We will build this function in Step 3.
It checks the extension allowlist and file size.

**`get_extension(uploaded_file.file_name)`** -- Extracts the extension from
the original filename. We use this to generate a safe filename that preserves
the original extension.

**`wisp.random_string(16)`** -- Generates a random alphanumeric string. Combined
with the extension, this gives us filenames like `a8f3b2c1d9e4f507.png`. No
path traversal possible, no naming conflicts, no special characters.

**`simplifile.copy(uploaded_file.path, dest)`** -- Copies the file from the
temporary path (where Wisp stored it during parsing) to the permanent upload
directory. We use `copy` rather than `rename` because the temp file and the
upload directory might be on different filesystems, and `rename` does not work
across filesystem boundaries. If you are certain they are on the same
filesystem, `simplifile.rename` is slightly more efficient because it avoids
copying the bytes.

**`sanitize_display_name`** -- We store a sanitized version of the original
filename for display purposes (so the user sees "screenshot.png" rather than
"a8f3b2c1d9e4f507.png" in the attachment list). This function strips path
separators and limits the length -- we will define it shortly.

**`wisp.html_response(html, 201)`** -- We return a `201 Created` status with
the HTML fragment for the new attachment. HTMX will swap this into the
attachment list. Note that `wisp.html_response` takes a `String`, not a
`StringBuilder` -- we convert the Lustre element with `element.to_string`.

The upload directory function returns a configurable path:

```gleam
import envoy

fn upload_directory() -> String {
  case envoy.get("UPLOAD_DIR") {
    Ok(dir) -> dir
    Error(_) -> "./uploads"
  }
}
```

During development, uploads land in `./uploads/` relative to the project root.
In production, set the `UPLOAD_DIR` environment variable to an appropriate path.

Make sure the directory exists before the first upload. You can create it at
application startup:

```gleam
fn main() {
  // Ensure the upload directory exists.
  let _ = simplifile.create_directory_all(upload_directory())
  // ... start the server ...
}
```

### Step 3 -- File Validation Module

Validation is critical for file uploads. Let us build a focused validation
module:

```gleam
import gleam/string
import gleam/list
import gleam/int
import gleam/result
import simplifile
import wisp

// The list of allowed file extensions.
const allowed_extensions = [".png", ".jpg", ".jpeg", ".gif", ".pdf"]

// Maximum file size in bytes (10 MB).
const max_file_size = 10_000_000

/// Validate an uploaded file. Returns Ok(Nil) if valid,
/// Error(String) with a human-readable reason if not.
pub fn validate_upload(
  uploaded_file: wisp.UploadedFile,
) -> Result(Nil, String) {
  use _ <- result.try(validate_extension(uploaded_file.file_name))
  use _ <- result.try(validate_size(uploaded_file.path))
  use _ <- result.try(validate_not_empty(uploaded_file.file_name))
  Ok(Nil)
}

fn validate_extension(filename: String) -> Result(Nil, String) {
  let ext = get_extension(filename)
  case list.contains(allowed_extensions, ext) {
    True -> Ok(Nil)
    False -> {
      let allowed =
        allowed_extensions
        |> string.join(", ")
      Error(
        "File type not allowed. Accepted types: " <> allowed <> ".",
      )
    }
  }
}

fn validate_size(path: String) -> Result(Nil, String) {
  case simplifile.file_info(path) {
    Ok(info) -> {
      case info.size <= max_file_size {
        True -> Ok(Nil)
        False -> {
          let limit_mb = int.to_string(max_file_size / 1_000_000)
          Error("File is too large. Maximum size is " <> limit_mb <> " MB.")
        }
      }
    }
    Error(_) -> Error("Could not read file information.")
  }
}

fn validate_not_empty(filename: String) -> Result(Nil, String) {
  case string.trim(filename) {
    "" -> Error("No file was selected.")
    _ -> Ok(Nil)
  }
}

/// Extract the file extension, lowercased, with the leading dot.
/// Returns "" if there is no extension.
pub fn get_extension(filename: String) -> String {
  case string.split(filename, ".") {
    [_] -> ""
    parts -> {
      case list.last(parts) {
        Ok(ext) -> "." <> string.lowercase(ext)
        Error(_) -> ""
      }
    }
  }
}

/// Sanitize a filename for display purposes.
/// Removes path separators, limits length, preserves extension.
pub fn sanitize_display_name(filename: String) -> String {
  // Remove any path separators (forward and back slashes).
  let name =
    filename
    |> string.replace("/", "")
    |> string.replace("\\", "")
    |> string.replace("..", "")

  // Limit total length to 100 characters.
  case string.length(name) > 100 {
    True -> {
      let ext = get_extension(name)
      let base = string.slice(name, 0, 100 - string.length(ext))
      base <> ext
    }
    False -> name
  }
}
```

Let us examine the design decisions.

**`result.try` chaining.** Each validation function returns
`Result(Nil, String)`. Using `result.try`, we chain them so the first failure
short-circuits and returns the error message. This is the same validation
pattern from Chapter 8, adapted for file uploads.

**Extension checking with `list.contains`.** We lowercase the extension with
`string.lowercase` and check it against the allowlist. This means `photo.PNG`,
`photo.Png`, and `photo.png` all pass. The `list.contains` function from
`gleam/list` checks if an element exists in the list.

**File size via `simplifile.file_info`.** The `file_info` function returns
metadata about the file, including its size in bytes. We check against our
10 MB limit. You can adjust this constant based on your application's needs --
5 MB for a task board is probably generous, while a document management system
might need 100 MB.

**Display name sanitization.** We strip path separators and double dots to
prevent path traversal attacks in displayed names. We also limit the length to
prevent layout issues in the UI. This sanitized name is stored in the database
for display -- the actual stored file uses the server-generated random name.

The `Attachment` type for database storage:

```gleam
pub type Attachment {
  Attachment(
    id: String,
    task_id: String,
    original_name: String,
    stored_name: String,
  )
}
```

And the database table:

```sql
CREATE TABLE IF NOT EXISTS attachments (
  id TEXT PRIMARY KEY,
  task_id TEXT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  original_name TEXT NOT NULL,
  stored_name TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

The `ON DELETE CASCADE` ensures that when a task is deleted, its attachments
are automatically removed from the database. You will still need to delete the
actual files from disk -- we handle that in the delete endpoint.

The `save_attachment` function inserts the record:

```gleam
import sqlight
import gleam/dynamic/decode

pub fn save_attachment(
  db: sqlight.Connection,
  attachment: Attachment,
) -> Result(Nil, sqlight.Error) {
  let sql = "
    INSERT INTO attachments (id, task_id, original_name, stored_name)
    VALUES (?1, ?2, ?3, ?4)
  "
  sqlight.query(
    sql,
    on: db,
    with: [
      sqlight.text(attachment.id),
      sqlight.text(attachment.task_id),
      sqlight.text(attachment.original_name),
      sqlight.text(attachment.stored_name),
    ],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}
```

And the query to list attachments for a task:

```gleam
pub fn list_attachments(
  db: sqlight.Connection,
  task_id: String,
) -> Result(List(Attachment), sqlight.Error) {
  let sql = "
    SELECT id, task_id, original_name, stored_name
    FROM attachments
    WHERE task_id = ?1
    ORDER BY created_at ASC
  "
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(task_id)],
    expecting: attachment_decoder(),
  )
}

fn attachment_decoder() -> decode.Decoder(Attachment) {
  use id <- decode.field(0, decode.string)
  use task_id <- decode.field(1, decode.string)
  use original_name <- decode.field(2, decode.string)
  use stored_name <- decode.field(3, decode.string)
  decode.success(Attachment(
    id: id,
    task_id: task_id,
    original_name: original_name,
    stored_name: stored_name,
  ))
}
```

### Step 4 -- Display Uploaded Attachments with Download Links

When a task has attachments, we display them as a list with download links and
delete buttons. Here is the view:

```gleam
fn render_attachment_list(
  task_id: String,
  attachments: List(Attachment),
) -> Element(t) {
  html.div(
    [
      attribute.id("attachment-list-" <> task_id),
      attribute.class("attachment-list"),
    ],
    list.map(attachments, render_attachment),
  )
}

fn render_attachment(attachment: Attachment) -> Element(t) {
  html.div(
    [
      attribute.id("attachment-" <> attachment.id),
      attribute.class("attachment-item"),
    ],
    [
      // File icon based on extension
      html.span([attribute.class("attachment-icon")], [
        element.text(file_icon(attachment.original_name)),
      ]),
      // Download link
      html.a(
        [
          attribute.href("/uploads/" <> attachment.stored_name),
          attribute.attribute("download", attachment.original_name),
          attribute.class("attachment-link"),
        ],
        [element.text(attachment.original_name)],
      ),
      // Delete button
      html.button(
        [
          hx.delete(
            "/tasks/"
            <> attachment.task_id
            <> "/attachments/"
            <> attachment.id,
          ),
          hx.target(hx.Selector("#attachment-" <> attachment.id)),
          hx.swap(hx.OuterHTML),
          attribute.attribute("hx-confirm", "Delete this attachment?"),
          attribute.class("btn-danger btn-small"),
        ],
        [element.text("Remove")],
      ),
    ],
  )
}

fn file_icon(filename: String) -> String {
  let ext = get_extension(filename)
  case ext {
    ".pdf" -> "[PDF]"
    ".png" | ".jpg" | ".jpeg" | ".gif" -> "[IMG]"
    _ -> "[FILE]"
  }
}
```

Several details here deserve attention.

**`attribute.attribute("download", attachment.original_name)`** -- The HTML
`download` attribute on an `<a>` tag does two things. First, it tells the
browser to download the file rather than navigate to it. Second, the value
specifies the filename to use for the download. The user sees "screenshot.png"
in their downloads folder, even though the file on the server is named
`a8f3b2c1d9e4f507.png`. This preserves the original filename for the user
while keeping the safe generated name on the server.

**`attribute.href("/uploads/" <> attachment.stored_name)`** -- The link points
to the stored filename under the `/uploads/` path, which is served by
`wisp.serve_static`. The browser makes a GET request to this URL, and Wisp
serves the file from the upload directory.

**`hx.delete` on the delete button** -- Sends a DELETE request to remove the
attachment. We use `hx.swap(hx.OuterHTML)` so the entire attachment item is
replaced by the server's response (which will be empty, effectively removing
the item from the list). The `hx-confirm` attribute adds a browser
confirmation dialog before the delete request is sent -- the same pattern from
Chapter 14.

**`hx.target(hx.Selector("#attachment-" <> attachment.id))`** -- Targets the
specific attachment item for removal. Each attachment has a unique ID, so
deleting one does not affect the others.

The error rendering function for failed uploads:

```gleam
fn render_upload_error(message: String) -> String {
  html.div(
    [attribute.class("upload-error"), attribute.attribute("role", "alert")],
    [
      html.span([attribute.class("error-icon")], [element.text("!")]),
      element.text(message),
    ],
  )
  |> element.to_string()
}
```

We use `attribute.attribute("role", "alert")` for accessibility. Screen readers
will announce the error message automatically when it appears in the DOM. This
is important because the upload error appears asynchronously -- without the
ARIA role, a screen reader user would not know that something went wrong.

To integrate the attachment list into the task view, add it to the task item
component:

```gleam
fn task_detail(task: Task, attachments: List(Attachment)) -> Element(t) {
  html.div([attribute.class("task-detail")], [
    html.h2([], [element.text(task.title)]),
    html.p([], [element.text(task.description)]),
    // Attachment section
    html.div([attribute.class("attachments-section")], [
      html.h3([], [element.text("Attachments")]),
      render_attachment_list(task.id, attachments),
      attachment_upload_form(task.id),
    ]),
  ])
}
```

The attachment list and upload form sit together in an "Attachments" section.
New uploads are appended to the list via `hx.swap(hx.Beforeend)`, so the user
sees their upload appear immediately without a page reload.

### Step 5 -- Progress Bar with `_hyperscript`

Let us examine the progress bar mechanism in detail. We already placed the
`_hyperscript` attribute on the form in Step 1. Now let us understand exactly
how it works.

The `_hyperscript` code on the form:

```
on htmx:xhr:progress(loaded, total)
  set pct to (loaded / total) * 100
  put pct + '%' into #progress-text
  set #progress-bar's style.width to pct + '%'
on htmx:afterRequest
  set #progress-bar's style.width to '0%'
  put '' into #progress-text
```

This is two event handlers in one `_` attribute.

**The first handler** listens for `htmx:xhr:progress`. This is an HTMX event
that fires periodically during an XMLHttpRequest upload. The event carries
`loaded` (bytes uploaded so far) and `total` (total bytes to upload) as event
detail properties. `_hyperscript` can destructure these directly in the event
handler signature.

Inside the handler:

1. `set pct to (loaded / total) * 100` -- Calculate the upload percentage.
2. `put pct + '%' into #progress-text` -- Update the text display (e.g., "73%").
3. `set #progress-bar's style.width to pct + '%'` -- Set the progress bar's
   CSS width to match the percentage.

**The second handler** listens for `htmx:afterRequest`. This fires when the
HTMX request completes (whether it succeeded or failed). It resets the progress
bar to zero and clears the percentage text, so the form is ready for the next
upload.

The HTML structure for the progress bar:

```gleam
// Inside the form, after the file input and before the submit button:

// Progress bar container
html.div([attribute.class("progress-container")], [
  html.div(
    [
      attribute.id("progress-bar"),
      attribute.class("progress-bar"),
      attribute.attribute("style", "width: 0%"),
    ],
    [],
  ),
]),
html.span([attribute.id("progress-text")], []),
```

The CSS to make this look like an actual progress bar:

```css
.progress-container {
  width: 100%;
  height: 20px;
  background-color: #e0e0e0;
  border-radius: 10px;
  overflow: hidden;
  margin: 8px 0;
}

.progress-bar {
  height: 100%;
  background-color: #4caf50;
  border-radius: 10px;
  transition: width 0.2s ease;
  width: 0%;
}

#progress-text {
  font-size: 0.85rem;
  color: #666;
}
```

The `transition: width 0.2s ease` on `.progress-bar` creates a smooth animation
as the width changes. Without it, the bar would jump in discrete steps. The
`overflow: hidden` on the container ensures the bar's rounded corners look
correct at any width.

You can also show or hide the progress bar based on upload state. Here is an
enhanced `_hyperscript` that hides the progress elements when no upload is in
progress:

```gleam
attribute.attribute(
  "_",
  "on htmx:xhr:progress(loaded, total)
     set pct to (loaded / total) * 100
     put pct + '%' into #progress-text
     set #progress-bar's style.width to pct + '%'
     show .progress-container
   on htmx:afterRequest
     set #progress-bar's style.width to '0%'
     put '' into #progress-text
     hide .progress-container with *opacity
     wait 300ms
     hide .progress-container",
)
```

The `show` and `hide` commands in `_hyperscript` toggle the element's
visibility. The `hide ... with *opacity` creates a fade-out effect by
transitioning the opacity before fully hiding the element. The `wait 300ms`
gives the fade animation time to complete.

If you prefer not to use `_hyperscript` for the progress bar, you can achieve
the same result with a small JavaScript event listener:

```javascript
document.addEventListener('htmx:xhr:progress', function(event) {
  var loaded = event.detail.loaded;
  var total = event.detail.total;
  var pct = (loaded / total) * 100;
  document.getElementById('progress-bar').style.width = pct + '%';
  document.getElementById('progress-text').textContent = pct.toFixed(0) + '%';
});
```

Both approaches work. The `_hyperscript` version keeps the behavior co-located
with the form element. The JavaScript version is more familiar to developers
who do not use `_hyperscript`. Choose whichever fits your project's conventions.

### Step 6 -- Delete Attachment Endpoint

The delete button we added in Step 4 sends `DELETE /tasks/:task_id/attachments/:attachment_id`.
Here is the handler:

```gleam
fn delete_attachment(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
  attachment_id: String,
) -> wisp.Response {
  // Look up the attachment in the database.
  case get_attachment(ctx.db, attachment_id) {
    Error(_) -> wisp.not_found()

    Ok(attachment) -> {
      // Verify the attachment belongs to the specified task.
      case attachment.task_id == task_id {
        False -> wisp.not_found()

        True -> {
          // Delete the file from disk.
          let file_path = upload_directory() <> "/" <> attachment.stored_name
          let _ = simplifile.delete(file_path)

          // Delete the database record.
          let _ = delete_attachment_record(ctx.db, attachment_id)

          // Return empty response. HTMX will swap this into the target,
          // effectively removing the attachment item from the DOM.
          wisp.html_response("", 200)
        }
      }
    }
  }
}
```

**Ownership check.** We verify that the attachment's `task_id` matches the
`task_id` in the URL. This prevents a user from deleting attachments on other
tasks by guessing IDs. If the IDs do not match, we return `404 Not Found` --
not `403 Forbidden`, because revealing that the resource exists but is not
accessible leaks information.

**File deletion.** We use `simplifile.delete` to remove the file from disk.
We ignore the result (`let _`) because the database record is the source of
truth. If the file is already gone (perhaps manually deleted), we still want
to clean up the database record.

**Empty response.** By returning an empty string with status 200, HTMX swaps
nothing into the target element. Since the target is the attachment item itself
(with `hx.swap(hx.OuterHTML)`), the entire item is replaced with nothing --
it disappears from the page.

The database functions:

```gleam
pub fn get_attachment(
  db: sqlight.Connection,
  attachment_id: String,
) -> Result(Attachment, Nil) {
  let sql = "
    SELECT id, task_id, original_name, stored_name
    FROM attachments
    WHERE id = ?1
  "
  case
    sqlight.query(
      sql,
      on: db,
      with: [sqlight.text(attachment_id)],
      expecting: attachment_decoder(),
    )
  {
    Ok([attachment]) -> Ok(attachment)
    _ -> Error(Nil)
  }
}

pub fn delete_attachment_record(
  db: sqlight.Connection,
  attachment_id: String,
) -> Result(Nil, sqlight.Error) {
  let sql = "DELETE FROM attachments WHERE id = ?1"
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(attachment_id)],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}
```

When a task is deleted, the `ON DELETE CASCADE` constraint removes the
attachment database records automatically. But the files on disk remain. To
clean those up, delete the files before deleting the task:

```gleam
fn delete_task(req: wisp.Request, ctx: Context, task_id: String) -> wisp.Response {
  // First, get all attachments for the task and delete their files.
  case list_attachments(ctx.db, task_id) {
    Ok(attachments) -> {
      list.each(attachments, fn(attachment) {
        let path = upload_directory() <> "/" <> attachment.stored_name
        let _ = simplifile.delete(path)
      })
    }
    Error(_) -> Nil
  }

  // Then delete the task (CASCADE handles the DB records).
  let _ = delete_task_record(ctx.db, task_id)
  wisp.html_response("", 200)
}
```

This ensures that deleting a task cleans up both the database records and the
files on disk. Without this step, orphaned files would accumulate over time.

---

## 3. Full Code Listing

Here is the complete code consolidated into one reference listing. In a real
project, you would split this across multiple files (router, views, validation,
database). We show it together here so you can see how all the pieces connect.

### Types

```gleam
// types.gleam

pub type Task {
  Task(
    id: String,
    title: String,
    description: String,
    done: Bool,
  )
}

pub type Attachment {
  Attachment(
    id: String,
    task_id: String,
    original_name: String,
    stored_name: String,
  )
}

pub type Context {
  Context(db: sqlight.Connection)
}
```

### Validation

```gleam
// validation.gleam

import gleam/string
import gleam/list
import gleam/int
import gleam/result
import simplifile
import wisp

const allowed_extensions = [".png", ".jpg", ".jpeg", ".gif", ".pdf"]

const max_file_size = 10_000_000

pub fn validate_upload(
  uploaded_file: wisp.UploadedFile,
) -> Result(Nil, String) {
  use _ <- result.try(validate_extension(uploaded_file.file_name))
  use _ <- result.try(validate_size(uploaded_file.path))
  use _ <- result.try(validate_not_empty(uploaded_file.file_name))
  Ok(Nil)
}

fn validate_extension(filename: String) -> Result(Nil, String) {
  let ext = get_extension(filename)
  case list.contains(allowed_extensions, ext) {
    True -> Ok(Nil)
    False -> {
      let allowed = string.join(allowed_extensions, ", ")
      Error("File type not allowed. Accepted types: " <> allowed <> ".")
    }
  }
}

fn validate_size(path: String) -> Result(Nil, String) {
  case simplifile.file_info(path) {
    Ok(info) -> {
      case info.size <= max_file_size {
        True -> Ok(Nil)
        False -> {
          let limit_mb = int.to_string(max_file_size / 1_000_000)
          Error("File is too large. Maximum size is " <> limit_mb <> " MB.")
        }
      }
    }
    Error(_) -> Error("Could not read file information.")
  }
}

fn validate_not_empty(filename: String) -> Result(Nil, String) {
  case string.trim(filename) {
    "" -> Error("No file was selected.")
    _ -> Ok(Nil)
  }
}

pub fn get_extension(filename: String) -> String {
  case string.split(filename, ".") {
    [_] -> ""
    parts -> {
      case list.last(parts) {
        Ok(ext) -> "." <> string.lowercase(ext)
        Error(_) -> ""
      }
    }
  }
}

pub fn sanitize_display_name(filename: String) -> String {
  let name =
    filename
    |> string.replace("/", "")
    |> string.replace("\\", "")
    |> string.replace("..", "")

  case string.length(name) > 100 {
    True -> {
      let ext = get_extension(name)
      let base = string.slice(name, 0, 100 - string.length(ext))
      base <> ext
    }
    False -> name
  }
}
```

### Database

```gleam
// database.gleam

import gleam/result
import gleam/dynamic/decode
import sqlight

pub fn migrate(db: sqlight.Connection) -> Result(Nil, sqlight.Error) {
  sqlight.exec(
    "
    CREATE TABLE IF NOT EXISTS attachments (
      id TEXT PRIMARY KEY,
      task_id TEXT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
      original_name TEXT NOT NULL,
      stored_name TEXT NOT NULL,
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
    ",
    db,
  )
}

pub fn save_attachment(
  db: sqlight.Connection,
  attachment: Attachment,
) -> Result(Nil, sqlight.Error) {
  let sql = "
    INSERT INTO attachments (id, task_id, original_name, stored_name)
    VALUES (?1, ?2, ?3, ?4)
  "
  sqlight.query(
    sql,
    on: db,
    with: [
      sqlight.text(attachment.id),
      sqlight.text(attachment.task_id),
      sqlight.text(attachment.original_name),
      sqlight.text(attachment.stored_name),
    ],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}

pub fn list_attachments(
  db: sqlight.Connection,
  task_id: String,
) -> Result(List(Attachment), sqlight.Error) {
  let sql = "
    SELECT id, task_id, original_name, stored_name
    FROM attachments
    WHERE task_id = ?1
    ORDER BY created_at ASC
  "
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(task_id)],
    expecting: attachment_decoder(),
  )
}

pub fn get_attachment(
  db: sqlight.Connection,
  attachment_id: String,
) -> Result(Attachment, Nil) {
  let sql = "
    SELECT id, task_id, original_name, stored_name
    FROM attachments
    WHERE id = ?1
  "
  case
    sqlight.query(
      sql,
      on: db,
      with: [sqlight.text(attachment_id)],
      expecting: attachment_decoder(),
    )
  {
    Ok([attachment]) -> Ok(attachment)
    _ -> Error(Nil)
  }
}

pub fn delete_attachment_record(
  db: sqlight.Connection,
  attachment_id: String,
) -> Result(Nil, sqlight.Error) {
  let sql = "DELETE FROM attachments WHERE id = ?1"
  sqlight.query(
    sql,
    on: db,
    with: [sqlight.text(attachment_id)],
    expecting: decode.dynamic,
  )
  |> result.map(fn(_) { Nil })
}

fn attachment_decoder() -> decode.Decoder(Attachment) {
  use id <- decode.field(0, decode.string)
  use task_id <- decode.field(1, decode.string)
  use original_name <- decode.field(2, decode.string)
  use stored_name <- decode.field(3, decode.string)
  decode.success(Attachment(
    id: id,
    task_id: task_id,
    original_name: original_name,
    stored_name: stored_name,
  ))
}
```

### Views

```gleam
// views.gleam

import gleam/list
import gleam/string
import lustre/element.{type Element}
import lustre/element/html
import lustre/attribute
import hx

pub fn attachment_upload_form(task_id: String) -> Element(t) {
  html.form(
    [
      attribute.id("upload-form"),
      hx.post("/tasks/" <> task_id <> "/attachments"),
      hx.target(hx.Selector("#attachment-list-" <> task_id)),
      hx.swap(hx.Beforeend),
      attribute.attribute("hx-encoding", "multipart/form-data"),
      attribute.attribute(
        "_",
        "on htmx:xhr:progress(loaded, total)
           set pct to (loaded / total) * 100
           put pct + '%' into #progress-text
           set #progress-bar's style.width to pct + '%'
         on htmx:afterRequest
           set #progress-bar's style.width to '0%'
           put '' into #progress-text",
      ),
    ],
    [
      html.div([attribute.class("form-group")], [
        html.label([attribute.for("attachment")], [
          element.text("Attach a file"),
        ]),
        html.input([
          attribute.type_("file"),
          attribute.name("attachment"),
          attribute.id("attachment"),
          attribute.accept([".png", ".jpg", ".jpeg", ".gif", ".pdf"]),
        ]),
      ]),
      html.div([attribute.class("progress-container")], [
        html.div(
          [
            attribute.id("progress-bar"),
            attribute.class("progress-bar"),
            attribute.attribute("style", "width: 0%"),
          ],
          [],
        ),
      ]),
      html.span([attribute.id("progress-text")], []),
      html.button(
        [attribute.type_("submit"), attribute.class("btn-secondary")],
        [element.text("Upload")],
      ),
    ],
  )
}

pub fn render_attachment_list(
  task_id: String,
  attachments: List(Attachment),
) -> Element(t) {
  html.div(
    [
      attribute.id("attachment-list-" <> task_id),
      attribute.class("attachment-list"),
    ],
    list.map(attachments, render_attachment),
  )
}

pub fn render_attachment(attachment: Attachment) -> Element(t) {
  html.div(
    [
      attribute.id("attachment-" <> attachment.id),
      attribute.class("attachment-item"),
    ],
    [
      html.span([attribute.class("attachment-icon")], [
        element.text(file_icon(attachment.original_name)),
      ]),
      html.a(
        [
          attribute.href("/uploads/" <> attachment.stored_name),
          attribute.attribute("download", attachment.original_name),
          attribute.class("attachment-link"),
        ],
        [element.text(attachment.original_name)],
      ),
      html.button(
        [
          hx.delete(
            "/tasks/"
            <> attachment.task_id
            <> "/attachments/"
            <> attachment.id,
          ),
          hx.target(hx.Selector("#attachment-" <> attachment.id)),
          hx.swap(hx.OuterHTML),
          attribute.attribute("hx-confirm", "Delete this attachment?"),
          attribute.class("btn-danger btn-small"),
        ],
        [element.text("Remove")],
      ),
    ],
  )
}

fn render_upload_error(message: String) -> String {
  html.div(
    [attribute.class("upload-error"), attribute.attribute("role", "alert")],
    [
      html.span([attribute.class("error-icon")], [element.text("!")]),
      element.text(message),
    ],
  )
  |> element.to_string()
}

fn file_icon(filename: String) -> String {
  let ext = get_extension(filename)
  case ext {
    ".pdf" -> "[PDF]"
    ".png" | ".jpg" | ".jpeg" | ".gif" -> "[IMG]"
    _ -> "[FILE]"
  }
}

pub fn task_detail(task: Task, attachments: List(Attachment)) -> Element(t) {
  html.div([attribute.class("task-detail")], [
    html.h2([], [element.text(task.title)]),
    html.p([], [element.text(task.description)]),
    html.div([attribute.class("attachments-section")], [
      html.h3([], [element.text("Attachments")]),
      render_attachment_list(task.id, attachments),
      attachment_upload_form(task.id),
    ]),
  ])
}
```

### Router

```gleam
// router.gleam

import gleam/http
import gleam/list
import gleam/string
import gleam/result
import wisp
import simplifile
import lustre/element
import envoy

fn upload_directory() -> String {
  case envoy.get("UPLOAD_DIR") {
    Ok(dir) -> dir
    Error(_) -> "./uploads"
  }
}

fn static_directory() -> String {
  case envoy.get("STATIC_DIR") {
    Ok(dir) -> dir
    Error(_) -> "./priv/static"
  }
}

pub fn handle_request(req: wisp.Request, ctx: Context) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.serve_static(req, under: "/static", from: static_directory())
  use <- wisp.serve_static(req, under: "/uploads", from: upload_directory())

  case wisp.path_segments(req) {
    // ... existing routes omitted for brevity ...

    ["tasks", task_id, "attachments"] -> {
      case req.method {
        http.Post -> upload_attachment(req, ctx, task_id)
        _ -> wisp.method_not_allowed([http.Post])
      }
    }

    ["tasks", task_id, "attachments", attachment_id] -> {
      case req.method {
        http.Delete -> delete_attachment(req, ctx, task_id, attachment_id)
        _ -> wisp.method_not_allowed([http.Delete])
      }
    }

    _ -> wisp.not_found()
  }
}

fn upload_attachment(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
) -> wisp.Response {
  use form_data <- wisp.require_form(req)

  case list.key_find(form_data.files, "attachment") {
    Error(_) -> {
      let html = render_upload_error("No file selected.")
      wisp.html_response(html, 422)
    }

    Ok(uploaded_file) -> {
      case validate_upload(uploaded_file) {
        Error(reason) -> {
          let html = render_upload_error(reason)
          wisp.html_response(html, 422)
        }

        Ok(_) -> {
          let ext = get_extension(uploaded_file.file_name)
          let safe_name = wisp.random_string(16) <> ext
          let dest = upload_directory() <> "/" <> safe_name

          case simplifile.copy(uploaded_file.path, dest) {
            Ok(_) -> {
              let attachment = Attachment(
                id: wisp.random_string(8),
                task_id: task_id,
                original_name: sanitize_display_name(
                  uploaded_file.file_name,
                ),
                stored_name: safe_name,
              )
              let _ = save_attachment(ctx.db, attachment)

              let html =
                render_attachment(attachment)
                |> element.to_string()
              wisp.html_response(html, 201)
            }

            Error(_) -> {
              let html = render_upload_error("Failed to store file.")
              wisp.html_response(html, 500)
            }
          }
        }
      }
    }
  }
}

fn delete_attachment(
  req: wisp.Request,
  ctx: Context,
  task_id: String,
  attachment_id: String,
) -> wisp.Response {
  case get_attachment(ctx.db, attachment_id) {
    Error(_) -> wisp.not_found()

    Ok(attachment) -> {
      case attachment.task_id == task_id {
        False -> wisp.not_found()

        True -> {
          let file_path =
            upload_directory() <> "/" <> attachment.stored_name
          let _ = simplifile.delete(file_path)
          let _ = delete_attachment_record(ctx.db, attachment_id)
          wisp.html_response("", 200)
        }
      }
    }
  }
}
```

### CSS

```css
/* Add to your existing style.css */

/* Upload form */
.attachments-section {
  margin-top: 1.5rem;
  border-top: 1px solid #e0e0e0;
  padding-top: 1rem;
}

.form-group {
  margin-bottom: 0.75rem;
}

/* Progress bar */
.progress-container {
  width: 100%;
  height: 20px;
  background-color: #e0e0e0;
  border-radius: 10px;
  overflow: hidden;
  margin: 8px 0;
}

.progress-bar {
  height: 100%;
  background-color: #4caf50;
  border-radius: 10px;
  transition: width 0.2s ease;
  width: 0%;
}

#progress-text {
  font-size: 0.85rem;
  color: #666;
  display: inline-block;
  min-width: 3em;
}

/* Attachment list */
.attachment-list {
  list-style: none;
  padding: 0;
  margin: 0.5rem 0;
}

.attachment-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  margin-bottom: 0.5rem;
  background: #fafafa;
}

.attachment-icon {
  font-family: monospace;
  font-size: 0.8rem;
  color: #888;
}

.attachment-link {
  flex: 1;
  color: #1a73e8;
  text-decoration: none;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.attachment-link:hover {
  text-decoration: underline;
}

.btn-small {
  padding: 0.25rem 0.5rem;
  font-size: 0.8rem;
}

.btn-danger {
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-danger:hover {
  background-color: #c82333;
}

/* Upload error */
.upload-error {
  background-color: #fdecea;
  border: 1px solid #f5c6cb;
  border-radius: 4px;
  padding: 0.75rem;
  margin: 0.5rem 0;
  color: #721c24;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.error-icon {
  font-weight: bold;
  background: #dc3545;
  color: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  flex-shrink: 0;
}
```

---

## 4. Exercises

### Exercise 1: Multiple File Upload

Modify the upload form to accept multiple files at once.

1. Add the `multiple` attribute to the file input using `attribute.multiple(True)`.
2. Update the server to iterate over all files in `form_data.files` with the name `"attachment"` (use `list.filter` to find all entries with that key, since `list.key_find` only returns the first match).
3. Return a fragment containing all newly created attachment items.
4. Each file should be individually validated -- if one file fails, the valid files should still be saved, and error messages should be shown for the rejected files.

**Acceptance criteria:** The user can select multiple files in the file chooser dialog. All valid files are uploaded and appear in the attachment list. Any invalid files produce individual error messages displayed alongside the successful uploads.

### Exercise 2: Image Preview Thumbnails

For image attachments (PNG, JPG, GIF), show a small thumbnail preview instead of the text icon.

1. In `render_attachment`, check if the file extension is an image type.
2. For images, render an `html.img` element with `attribute.src("/uploads/" <> attachment.stored_name)` and a CSS class that limits the width to 48px.
3. For non-image files (PDF), keep the text icon.
4. Add `attribute.attribute("loading", "lazy")` on the `<img>` for performance.

**Acceptance criteria:** Image attachments show a small thumbnail preview. PDF attachments show the text icon `[PDF]`. Thumbnails load lazily when scrolled into view.

### Exercise 3: Drag-and-Drop Upload

Add a drag-and-drop zone that allows users to drop files onto the task to upload them, without clicking the file input.

1. Add a `<div>` with `_hyperscript` that listens for `dragover` and `drop` events.
2. On `dragover`, prevent default behavior and add a CSS class for visual feedback.
3. On `drop`, extract the files from the event and submit them programmatically using `htmx.ajax('POST', url, {source: element, values: formData})`.
4. Style the drop zone with a dashed border that highlights on hover.

**Acceptance criteria:** Users can drag files from their desktop onto the drop zone. The zone highlights when a file is dragged over it. Dropped files are uploaded and appear in the attachment list. The progress bar updates during upload.

### Exercise 4: File Size Display

Show the file size alongside each attachment in the list.

1. Add a `size` field to the `Attachment` type (store the file size in bytes).
2. After copying the file to permanent storage, read its size with `simplifile.file_info` and store it in the database.
3. Add a `size INTEGER NOT NULL DEFAULT 0` column to the attachments table.
4. In `render_attachment`, format the size in a human-readable way (e.g., "1.2 MB", "456 KB").
5. Write a `format_file_size` function that converts bytes to the appropriate unit.

**Acceptance criteria:** Each attachment in the list shows its file size in a human-readable format (bytes for < 1 KB, KB for < 1 MB, MB for >= 1 MB). The size is stored in the database and survives page reloads.

### Exercise 5: Server-Side Content-Disposition Header

Add a dedicated download endpoint that sets proper `Content-Disposition` headers instead of relying on the `download` attribute.

1. Add a route `GET /tasks/:task_id/attachments/:attachment_id/download`.
2. Look up the attachment in the database.
3. Read the file from disk using `simplifile.read_bits`.
4. Return the file content with `wisp.response(200)` and set `Content-Disposition: attachment; filename="original_name.ext"` using `wisp.set_header`.
5. Also set `Content-Type` based on the file extension.
6. Update the download link in `render_attachment` to point to this new endpoint.

**Acceptance criteria:** Clicking the download link triggers a browser download with the original filename. The `Content-Disposition` header is set to `attachment`. The `Content-Type` header matches the file type (e.g., `image/png` for PNG files).

---

## 5. Exercise Solution Hints

Try each exercise on your own before reading these hints.

### Hint for Exercise 1

The `multiple` attribute on a file input allows selecting multiple files:

```gleam
html.input([
  attribute.type_("file"),
  attribute.name("attachment"),
  attribute.multiple(True),
  attribute.accept([".png", ".jpg", ".jpeg", ".gif", ".pdf"]),
])
```

On the server, `form_data.files` will contain multiple entries with the same key
`"attachment"`. Use `list.filter` instead of `list.key_find`:

```gleam
let attachment_files =
  form_data.files
  |> list.filter(fn(entry) {
    let #(name, _file) = entry
    name == "attachment"
  })
  |> list.map(fn(entry) {
    let #(_name, file) = entry
    file
  })
```

Then iterate over each file, validating and saving individually. Collect the
results and build a combined HTML fragment.

### Hint for Exercise 2

Branch on the extension in `render_attachment`:

```gleam
fn attachment_visual(attachment: Attachment) -> Element(t) {
  let ext = get_extension(attachment.original_name)
  case ext {
    ".png" | ".jpg" | ".jpeg" | ".gif" ->
      html.img([
        attribute.src("/uploads/" <> attachment.stored_name),
        attribute.class("attachment-thumbnail"),
        attribute.attribute("loading", "lazy"),
        attribute.alt(attachment.original_name),
      ])
    _ ->
      html.span([attribute.class("attachment-icon")], [
        element.text(file_icon(attachment.original_name)),
      ])
  }
}
```

CSS for the thumbnail:

```css
.attachment-thumbnail {
  width: 48px;
  height: 48px;
  object-fit: cover;
  border-radius: 4px;
}
```

### Hint for Exercise 3

The drop zone uses `_hyperscript` for drag events. The key challenge is
constructing a `FormData` object and submitting it via `htmx.ajax`:

```gleam
html.div(
  [
    attribute.class("drop-zone"),
    attribute.attribute(
      "_",
      "on dragover halt the event
         add .drag-over to me
       on dragleave
         remove .drag-over from me
       on drop halt the event
         remove .drag-over from me
         set formData to new FormData()
         repeat for file in event.dataTransfer.files
           call formData.append('attachment', file)
         end
         fetch /tasks/{taskId}/attachments {method: 'POST', body: formData}
         then put the result's text after the last <div/> in #attachment-list",
    ),
  ],
  [element.text("Drop files here")],
)
```

Note: you may need to use plain JavaScript for the `htmx.ajax` call with
`FormData`. The exact `_hyperscript` syntax for `fetch` with `FormData` can be
tricky -- test it carefully. An alternative is a small `<script>` block.

### Hint for Exercise 4

For the `format_file_size` function:

```gleam
fn format_file_size(bytes: Int) -> String {
  case bytes {
    b if b < 1024 ->
      int.to_string(b) <> " B"
    b if b < 1_048_576 ->
      int.to_string(b / 1024) <> " KB"
    b ->
      let mb = b / 1_048_576
      let remainder = { b % 1_048_576 } / 104_858
      int.to_string(mb) <> "." <> int.to_string(remainder) <> " MB"
  }
}
```

After copying the file, get the size:

```gleam
case simplifile.file_info(dest) {
  Ok(info) -> info.size
  Error(_) -> 0
}
```

### Hint for Exercise 5

For the download endpoint, read the file and set headers:

```gleam
fn download_attachment(
  ctx: Context,
  task_id: String,
  attachment_id: String,
) -> wisp.Response {
  case get_attachment(ctx.db, attachment_id) {
    Error(_) -> wisp.not_found()
    Ok(attachment) -> {
      let path = upload_directory() <> "/" <> attachment.stored_name
      case simplifile.read_bits(path) {
        Error(_) -> wisp.not_found()
        Ok(bits) -> {
          let content_type = mime_type(attachment.original_name)
          let disposition =
            "attachment; filename=\"" <> attachment.original_name <> "\""
          wisp.response(200)
          |> wisp.set_header("content-type", content_type)
          |> wisp.set_header("content-disposition", disposition)
          |> wisp.set_body(wisp.Bytes(bytes_tree.from_bit_array(bits)))
        }
      }
    }
  }
}

fn mime_type(filename: String) -> String {
  let ext = get_extension(filename)
  case ext {
    ".png" -> "image/png"
    ".jpg" | ".jpeg" -> "image/jpeg"
    ".gif" -> "image/gif"
    ".pdf" -> "application/pdf"
    _ -> "application/octet-stream"
  }
}
```

The `wisp.set_body` call with `wisp.Bytes` sets the response body to raw binary
data. The `bytes_tree.from_bit_array` converts the `BitArray` from
`simplifile.read_bits` into the `BytesTree` type that Wisp expects.

---

## 6. Key Takeaways

1. **File uploads require `multipart/form-data` encoding.** The default
   `application/x-www-form-urlencoded` encoding cannot carry binary data. In
   HTMX, set `hx-encoding="multipart/form-data"` on the form -- without it,
   file inputs are silently ignored and no file data reaches the server.

2. **Wisp's `FormData` type has two fields: `values` and `files`.** Text inputs
   go into `values` as `List(#(String, String))`. File inputs go into `files`
   as `List(#(String, UploadedFile))`. The `UploadedFile` record gives you
   `file_name` (the original name from the user's machine) and `path` (a
   temporary path on the server where Wisp has already written the bytes).

3. **Never trust user-supplied filenames.** Generate unique filenames on the
   server using `wisp.random_string(16)` combined with the validated extension.
   This prevents path traversal attacks, naming conflicts, and filename-based
   XSS. Store the sanitized original name separately for display purposes.

4. **Server-side validation is mandatory for file uploads.** Check the file
   extension against an allowlist, enforce a size limit, and reject empty
   uploads. Return HTTP 422 with a descriptive error message for each
   validation failure. Client-side `accept` attributes are a UX convenience,
   not a security measure.

5. **`htmx:xhr:progress` gives you real-time upload feedback.** The event fires
   periodically with `loaded` and `total` byte counts. Use `_hyperscript` or
   JavaScript to calculate the percentage and update a progress bar element's
   CSS width. Reset the bar on `htmx:afterRequest`.

6. **Use `simplifile.copy` to move files from temp to permanent storage.**
   The temp path in `UploadedFile` may be cleaned up after the request. Copy
   the file to your upload directory immediately. Use `copy` instead of
   `rename` to handle cross-filesystem boundaries safely.

7. **Serve uploaded files with `wisp.serve_static` and protect with
   `Content-Disposition`.** Add a second `serve_static` call for the upload
   directory. For downloads, use the HTML `download` attribute or a dedicated
   endpoint with `Content-Disposition: attachment` to prevent browsers from
   executing uploaded content.

8. **Clean up files on disk when deleting attachments or tasks.** Database
   `ON DELETE CASCADE` handles the database records, but orphaned files remain
   on disk. Delete files with `simplifile.delete` before or alongside the
   database deletion to prevent storage leaks over time.

9. **Keep the upload directory outside your application's source tree.** Store
   uploads in a dedicated directory (configurable via environment variable)
   that is separate from `priv/static/` and application code. This prevents
   uploaded files from being interpreted as executable code and keeps your
   deployment artifact clean.

---

## What's Next

In **Chapter 26 -- Accessible HTMX**, we will address a crucial aspect of
real-world applications: making HTMX interactions work for all users, including
those who rely on screen readers, keyboard navigation, and other assistive
technologies. HTMX's dynamic content swaps present unique accessibility
challenges -- content that appears without a page load can be invisible to
assistive technology unless you use the right ARIA attributes and focus
management patterns. We will cover live regions, focus management after swaps,
keyboard-accessible controls, and how to test your application with a screen
reader.
