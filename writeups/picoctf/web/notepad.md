# notepad
## RECON

This challenges give you a web that
* Receives `content` from the user
* writes it as an HTML file in the `static/` directory, and redirects to that file

<img width="337" height="172" alt="image" src="https://github.com/user-attachments/assets/ee8af4b9-c51a-4fec-ae6d-89c70bc26fc0" />

<img width="562" height="117" alt="image" src="https://github.com/user-attachments/assets/25e75dd6-bb27-489a-b780-a0794d516a9a" />

on source code app.py
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
what this code do ?
web recieve content from user input
```python
content = request.form.get("content", "")
```
filters "_" and "/" from input then render index.html page where error="bad_content"
```python
    if "_" in content or "/" in content:
        return redirect(url_for("index", error="bad_content"))
```
if string is longer than 512 render index.html page where error="bad_content"
```python
    if len(content) > 512:
        return redirect(url_for("index", error="long_content", len=len(content)))
```
then redirect to /static "url_fix" take first 128 character to filename change content to correct url and end with random 8 character

**note** url_fix will change " \ " to " / "
```python
name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```
this is index.html in source code

<img width="650" height="301" alt="image" src="https://github.com/user-attachments/assets/1a4fa4f5-4cb2-4982-93fb-6ac71911c8a8" />

on **{{ error }}** there's is nothing we can do it completely escaped

but **{% include "errors/" + error + ".html" ignore missing %} {% endif %}** this take ```error``` directly in path render it as html

we need to make content input to here so

# PLAN
1.find a way to make input appear on ```error``` template

2.craft a payload that bypass filter "_" and " / " and put it after 128 character

# TEST
* make payload appear on error page

<img width="715" height="293" alt="image" src="https://github.com/user-attachments/assets/59f311e9-0585-42dc-84f6-791259aef358" />

```python
name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```
when source code put our payoad in it will look like this
```static/../templates/errors/test-<token>.html```
when resolve on real disk it will look like this
```/app/templates/errors/test-<token>.html```
which mean we have our templates on ```\test-<token>.html```


