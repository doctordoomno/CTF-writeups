# notepad
## RECON

This challenges give you a web that take user input and render it out as html page showing it user input

<img width="337" height="172" alt="image" src="https://github.com/user-attachments/assets/ee8af4b9-c51a-4fec-ae6d-89c70bc26fc0" />

<img width="562" height="117" alt="image" src="https://github.com/user-attachments/assets/25e75dd6-bb27-489a-b780-a0794d516a9a" />

on source code app.py

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


