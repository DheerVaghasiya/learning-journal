# Flask Framework — Building Web Apps and APIs with Python

*Notes from Krish Naik's Complete Python Bootcamp — [Flask module](https://github.com/krishnaik06/Complete-Python-Bootcamp/tree/main/13-Flask/flask)*

Flask is one of the most widely used web frameworks in Python, and understanding it properly is what turns "I can write Python scripts" into "I can build something people actually interact with through a browser or an app." This note walks through Flask from the very first line of code up to building a working REST API, explaining not just *what* the code does but *why* it's written that way.

---

## 1. What is Flask, and what does it actually do?

**Definition:** Flask is a **micro web framework** for Python. "Web framework" means it's a toolkit for building applications that run on a web server and respond to requests coming in over HTTP (the protocol web browsers use to talk to servers). "Micro" means it deliberately keeps its core small and unopinionated — it gives you the essentials (routing, request handling, templates) and lets you add anything else (databases, authentication, etc.) yourself, rather than bundling everything in by default the way a bigger framework like Django does.

**How a web app actually works, at a high level, before touching any code:**
1. A user's browser sends an HTTP **request** to a specific URL (e.g. `yourapp.com/about`).
2. The server (running your Flask app) receives that request and figures out which piece of code should handle it, based on the URL.
3. That code runs and produces a **response** — this could be plain text, an HTML page, or structured data like JSON.
4. The response gets sent back to the browser, which then displays it.

Flask's entire job is to make steps 2 and 3 easy to define in Python.

### 1.1 What is WSGI?

You'll see the term **WSGI** (Web Server Gateway Interface) come up immediately in any Flask app. **Definition:** WSGI is a standard interface that defines how a Python web application talks to a web server. It's the "adapter" that lets any WSGI-compliant server (like the one Flask runs during development) know how to hand incoming requests to your Python code, and how to take your Python code's output and turn it into a proper HTTP response. Flask builds directly on top of this standard — when you create a Flask app, you're creating a WSGI application.

---

## 2. A simple Flask app skeleton

Here is the smallest possible Flask app, and it's worth understanding every single line, because this exact skeleton is the foundation every other Flask app in this note builds on:

```python
from flask import Flask

### WSGI Application
app = Flask(__name__)

@app.route("/")
def welcome():
    return "Welcome to this best Flask course. This should be an amazing course"

@app.route("/index")
def index():
    return "Welcome to the index page"

if __name__ == "__main__":
    app.run(debug=True)
```

**Breaking this down piece by piece:**

- `app = Flask(__name__)` — this line creates your actual Flask application object. `__name__` is a special built-in Python variable that holds the name of the current module; Flask uses it internally to figure out the root path of your application (useful for finding templates and static files later). Every Flask app starts with this exact line.

- `@app.route("/")` — this is a **decorator**, and it's the heart of how Flask works. It tells Flask: "whenever a request comes in for this specific URL path, run the function directly below me, and send back whatever it returns as the response." This is called **routing** — mapping a URL to a specific piece of Python code.

- The function itself (`welcome()`, `index()`) can be named anything — the name isn't what matters to the browser, the **route path** in the decorator is. Whatever the function `return`s becomes the actual content the browser receives and displays.

- `if __name__ == "__main__": app.run(debug=True)` — this is the standard Python "only run this if the file is executed directly" guard, and inside it, `app.run()` actually starts the development web server. `debug=True` turns on Flask's debug mode, which does two very useful things while developing: it automatically restarts the server whenever you save a code change, and it shows detailed, readable error pages in the browser instead of a generic crash message.

**How to actually run this:** save it as, say, `app.py`, then run `python app.py` in the terminal. Flask will start a local development server (usually at `http://127.0.0.1:5000`), and visiting `/` or `/index` in a browser will show whichever string that route's function returned.

---

## 3. Integrating HTML with a Flask app

Returning plain strings, like the skeleton above does, isn't how real web pages work — real pages are HTML. Flask offers two ways to bring HTML in.

### 3.1 The quick-and-dirty way: HTML directly in the return statement

```python
@app.route("/")
def welcome():
    return "<html><H1>Welcome to the flask course</H1></html>"
```

This technically works — Flask just sends whatever string you return, and the browser interprets it as HTML if it looks like HTML. But this gets unmanageable fast: mixing full page layouts into Python strings makes both the Python and the HTML harder to read, and you lose things like your code editor's HTML syntax highlighting and autocomplete.

### 3.2 The proper way: `render_template()` and the `templates/` folder

**Definition:** `render_template()` is a Flask function that loads an HTML file from a folder named `templates` (which Flask looks for automatically, right alongside your Python file) and sends its contents back as the response.

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/index")
def index():
    return render_template('index.html')

@app.route('/about')
def about():
    return render_template('about.html')
```

And the actual HTML file it's loading, `templates/index.html`, is just a normal HTML document:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Flask App</title>
</head>
<body>
    <h1>Welcome to My Flask App!</h1>
    <p>This is a simple web application built with Flask.</p>
</body>
</html>
```

**Why this matters:** now the HTML lives in its own proper `.html` file, written and edited exactly like any other web page, and the Python code stays clean — it just says *which* template to show for a given route. This separation (Python logic in `.py` files, presentation in `.html` files inside `templates/`) is a foundational pattern in Flask, and it's what makes the next section — Jinja2 — actually useful, since Jinja2 works specifically *inside* these template files.

**The one folder-structure rule to remember:** Flask expects your HTML files to live in a folder literally named `templates`, sitting in the same directory as your main Python file. `render_template('about.html')` looks inside that folder automatically — you don't give it a full file path.

---

## 4. Working with HTTP verbs — GET and POST

**Definition:** HTTP verbs (also called HTTP methods) describe *what kind of action* a request is asking the server to perform. The two most fundamental ones:

- **GET** — "give me this resource." Used for simply requesting/reading data — visiting a page, loading an image, fetching a list. GET requests don't (and shouldn't) change anything on the server; they're just asking to view something. Any data sent with a GET request is typically visible right in the URL itself.
- **POST** — "here's some data, do something with it." Used for submitting data to the server — filling out a form, creating a new record, logging in. POST data is sent in the body of the request, not visible in the URL.

**How this shows up in a real Flask route** — a single route that behaves differently depending on which verb was used to reach it:

```python
from flask import Flask, render_template, request

app = Flask(__name__)

@app.route('/form', methods=['GET', 'POST'])
def form():
    if request.method == 'POST':
        name = request.form['name']
        return f'Hello {name}!'
    return render_template('form.html')
```

**Walking through the logic:** by default, a Flask route only responds to GET requests. Adding `methods=['GET', 'POST']` to the decorator tells Flask this route should also accept POST requests. Inside the function, `request.method` tells you which verb the *current* request actually used, so you can branch your logic: if it was a POST (meaning a form was just submitted), read the submitted value out of `request.form` and respond accordingly; otherwise (a plain GET, meaning someone just navigated to the page), show the empty form so they have something to fill out.

**`request.form` explained:** when an HTML form is submitted via POST, Flask makes all of its field values available through `request.form`, indexed by each input's `name` attribute. This is why the HTML form and the Python code have to agree on field names:

```html
<form action="/submit" method="post">
    <label for="name">Name:</label>
    <input type="text" id="name" name="name">
    <input type="submit" value="Submit">
</form>
```

The `name="name"` attribute on the `<input>` tag is exactly what shows up as the key in `request.form['name']` on the Python side. If these don't match, `request.form['name']` would throw a `KeyError` because that key simply wouldn't exist.

**Why one route can handle both GET and POST, instead of needing two separate ones:** it keeps the "show the form" and "process the form's submission" logic bundled together in one logical place — a genuinely common and idiomatic Flask pattern, seen constantly in real projects.

---

## 5. Building dynamic URLs and the variable rule

So far every route has been a fixed, hardcoded path (`/about`, `/index`). But a lot of real functionality needs URLs that change based on some value — think a user profile page at `/user/42` where `42` is a specific user's ID.

**Definition:** A **variable rule** is Flask's way of marking part of a URL path as a placeholder, whose actual value gets captured and passed directly into your view function as an argument.

```python
@app.route('/success/<int:score>')
def success(score):
    res = ""
    if score >= 50:
        res = "PASSED"
    else:
        res = "FAILED"

    return render_template('result.html', results=res)
```

**Breaking down the syntax:** `<int:score>` inside the route path means "this segment of the URL is a variable named `score`, and it should be converted to an `int` automatically." So visiting `/success/75` calls `success(75)` — Flask has already parsed `"75"` out of the URL string and converted it to the actual integer `75` before your function even runs. Visit `/success/abc` and Flask returns a `404 Not Found` on its own, because `"abc"` can't be converted to an `int` — that type conversion doubles as free input validation.

Other type converters available besides `int`: `<string:name>` (or just `<name>`, since string is the default if you don't specify a type), `<float:price>`, `<path:filepath>` (which, unlike a normal string segment, allows forward slashes inside it).

This is a genuinely central concept — it's what lets one single route definition serve an unlimited number of different URLs (`/success/40`, `/success/90`, `/success/12`, all handled by the exact same function), instead of needing to hardcode a separate route for every possible score.

---

## 6. The Jinja2 template engine

Rendering a static HTML file is fine, but real pages need to display data that changes — a user's name, a list of items, a calculated result. That's exactly what Jinja2 (Flask's built-in template engine) is for.

**Definition:** Jinja2 is a templating language that lets you embed logic and dynamic values directly inside an HTML file, using special syntax that Flask processes on the server before sending the final HTML to the browser. By the time the browser receives the page, all the Jinja2 syntax has already been replaced with actual, final content — the browser itself never sees any Jinja2, only plain HTML.

Jinja2 has three main syntax forms, and knowing which is which removes almost all the initial confusion:

```
{{ ... }}   →  print/output a value (an "expression")
{% ... %}   →  logic: if statements, for loops, etc. (a "statement," produces no visible output itself)
{# ... #}   →  a comment (completely ignored, never shown to the user)
```

### 6.1 Passing data from Python into a template

```python
@app.route('/successres/<int:score>')
def successres(score):
    res = ""
    if score >= 50:
        res = "PASSED"
    else:
        res = "FAILED"

    exp = {'score': score, "res": res}

    return render_template('result1.html', results=exp)
```

The keyword argument in `render_template('result1.html', results=exp)` is exactly how data crosses from your Python code into the template — `results` becomes a variable named `results`, available for use inside `result1.html`, holding whatever value you passed (here, a dictionary).

### 6.2 Using `{{ }}` to display a value, and `{% if %}` for logic

```html
<h1>
  Based on the marks You have {{ results }}

  {% if results >= 50 %}
  <h1>You have passed with marks {{ results }}</h1>
  {% else %}
  <h2>You have failed with marks {{ results }}</h2>
  {% endif %}
</h1>
```

`{{ results }}` drops the actual value of the `results` variable directly into the HTML output. `{% if results >= 50 %} ... {% else %} ... {% endif %}` works exactly like a Python `if/else`, except it's deciding *which HTML gets included in the final page*, rather than which Python code runs. Notice `{% if %}` always needs a matching `{% endif %}` — Jinja2 blocks are explicitly closed, since (unlike Python) there's no indentation to mark where a block ends.

### 6.3 Using `{% for %}` to loop over data, and `{# #}` for comments

```html
<html>
<h2>Final Results</h2>
<body>
   {% for key, value in results.items() %}
      {# This is the comment section #}
      <h1>{{ key }}</h1>
      <h2>{{ value }}</h2>
   {% endfor %}
</body>
</html>
```

This loops over a dictionary's key/value pairs exactly the way `for key, value in results.items():` would in plain Python, repeating the `<h1>`/`<h2>` block once per pair, and building it all into the final HTML. `{# This is the comment section #}` is a Jinja2 comment — useful for leaving notes in the template itself without them ever appearing in the rendered page, unlike an HTML comment (`<!-- -->`) which *would* still show up if you view the page's raw source.

### 6.4 `redirect()` and `url_for()`

One more pair worth knowing, since it comes up constantly once forms and multi-step flows are involved:

```python
from flask import redirect, url_for

@app.route('/submit', methods=['POST', 'GET'])
def submit():
    total_score = 0
    if request.method == 'POST':
        science = float(request.form['science'])
        maths = float(request.form['maths'])
        c = float(request.form['c'])
        data_science = float(request.form['datascience'])

        total_score = (science + maths + c + data_science) / 4
    else:
        return render_template('getresult.html')
    return redirect(url_for('successres', score=total_score))
```

**`url_for('successres', score=total_score')`** builds the actual URL for the route whose function is named `successres`, automatically substituting `total_score` into wherever that route's variable rule expects a value — so you never have to hand-write the URL string yourself (`f'/successres/{total_score}'`), which is fragile if the route path ever changes later. **`redirect(...)`** then tells the browser to make a brand-new request to that generated URL — this is the standard "process a form, then send the user on to a results page" pattern, rather than just rendering the results directly in response to the POST.

---

## 7. Working with REST APIs — PUT and DELETE

Everything up to this point has been about serving HTML pages to a browser. But Flask is just as commonly used to build **APIs** — endpoints that don't return HTML at all, but structured data (almost always **JSON**), meant to be consumed by another program (a frontend JavaScript app, a mobile app, another service entirely) rather than displayed directly to a human.

**Definition: REST (Representational State Transfer)** is a set of conventions for designing web APIs, where each URL represents a specific **resource** (like "an item," "a user," "an order"), and the HTTP verb used on that URL determines what action to take on it. This gives us two more HTTP verbs to add to GET and POST from Section 4:

- **PUT** — "update this existing resource with this new data."
- **DELETE** — "remove this resource."

Put together, GET/POST/PUT/DELETE map cleanly onto the four CRUD operations (Create, Read, Update, Delete) — the exact same concept from working with databases, just expressed through HTTP verbs on URLs instead of SQL keywords.

| HTTP verb | CRUD meaning | Typical usage |
|---|---|---|
| GET | Read | fetch a resource, or a list of resources |
| POST | Create | create a brand-new resource |
| PUT | Update | modify an existing resource |
| DELETE | Delete | remove an existing resource |

### 7.1 A complete example: a to-do list API

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

## Initial data — normally this would live in a database, kept in memory here to keep the example simple
items = [
    {"id": 1, "name": "Item 1", "description": "This is item 1"},
    {"id": 2, "name": "Item 2", "description": "This is item 2"}
]

@app.route('/')
def home():
    return "Welcome To The Sample To DO List App"
```

**GET — retrieve all items:**

```python
@app.route('/items', methods=['GET'])
def get_items():
    return jsonify(items)
```

`jsonify()` is the Flask function that converts a Python data structure (here, a list of dictionaries) into a properly formatted JSON HTTP response — including setting the correct `Content-Type` header so whatever's consuming this API knows to expect JSON. This is the API equivalent of `render_template()` from earlier: instead of rendering HTML for a browser, it renders JSON for a program.

**GET — retrieve one specific item, using a dynamic URL:**

```python
@app.route('/items/<int:item_id>', methods=['GET'])
def get_item(item_id):
    item = next((item for item in items if item["id"] == item_id), None)
    if item is None:
        return jsonify({"error": "item not found"})
    return jsonify(item)
```

This uses exactly the same `<int:item_id>` variable-rule pattern from Section 5 — `/items/1` calls `get_item(1)`, `/items/2` calls `get_item(2)`, and so on, all through one single route definition.

**POST — create a new item:**

```python
@app.route('/items', methods=['POST'])
def create_item():
    if not request.json or 'name' not in request.json:
        return jsonify({"error": "item not found"})
    new_item = {
        "id": items[-1]["id"] + 1 if items else 1,
        "name": request.json['name'],
        "description": request.json["description"]
    }
    items.append(new_item)
    return jsonify(new_item)
```

Notice the difference from the HTML-form POST example back in Section 4: there, submitted data came in through `request.form` (standard HTML form encoding). Here, since this is a JSON API rather than an HTML form, incoming data is read through `request.json` instead — the request body is expected to be a JSON payload, not form-encoded fields. Same underlying HTTP verb, different way of reading the incoming data, because the *shape* of the request is different.

**PUT — update an existing item:**

```python
@app.route('/items/<int:item_id>', methods=['PUT'])
def update_item(item_id):
    item = next((item for item in items if item["id"] == item_id), None)
    if item is None:
        return jsonify({"error": "Item not found"})
    item['name'] = request.json.get('name', item['name'])
    item['description'] = request.json.get('description', item['description'])
    return jsonify(item)
```

Worth noticing here: `request.json.get('name', item['name'])` uses the dictionary `.get()` method with a fallback — if the incoming JSON includes a new `name`, use that; otherwise, keep whatever the item's `name` already was. This lets a PUT request update just one field (say, only `description`) without being forced to resend every single field every time.

**DELETE — remove an item:**

```python
@app.route('/items/<int:item_id>', methods=['DELETE'])
def delete_item(item_id):
    global items
    items = [item for item in items if item["id"] != item_id]
    return jsonify({"result": "Item deleted"})
```

This rebuilds the `items` list, keeping every item *except* the one whose ID matches `item_id` — effectively removing it. `global items` is required here specifically because the function is *reassigning* `items` to a brand-new list, rather than just modifying the existing one in place (which wouldn't need the `global` keyword, since methods like `.append()` mutate the existing object rather than replacing it).

**A note on testing an API like this:** since PUT and DELETE requests aren't something a browser address bar can send directly (typing a URL into a browser always sends a GET), testing this kind of route normally requires a dedicated tool — something like Postman, `curl` from the command line, or Python's own `requests` library — to actually construct and send a PUT/DELETE request to the endpoint and see the JSON response come back.

---

## 8. Cheat sheet

```python
from flask import Flask, render_template, request, jsonify, redirect, url_for

app = Flask(__name__)                      # 1. create the app

@app.route('/path')                        # 2. define a route
def view_function():
    return "plain text"                    # simplest possible response
    return render_template('page.html')    # render an HTML file from templates/
    return jsonify({"key": "value"})       # return JSON, for an API

@app.route('/path', methods=['GET', 'POST'])   # accept more than just GET
def handler():
    if request.method == 'POST':
        value = request.form['field_name']     # from an HTML form submission
        value = request.json['field_name']     # from a JSON API request body
    ...

@app.route('/items/<int:item_id>')         # variable rule — dynamic URL segment
def item_view(item_id):
    ...

if __name__ == "__main__":
    app.run(debug=True)                    # 3. run the dev server
```

| HTTP verb | Meaning | Read incoming data with |
|---|---|---|
| GET | fetch a resource | (usually no body data — use URL params/variable rules) |
| POST | create a resource | `request.form` (HTML forms) or `request.json` (APIs) |
| PUT | update a resource | `request.json` |
| DELETE | remove a resource | (usually just the URL, e.g. `<int:item_id>`) |

**Jinja2 quick reference:**

```
{{ variable }}              → output a value
{% if condition %} ... {% endif %}       → conditional logic
{% for x in list %} ... {% endfor %}     → loop
{# a comment #}              → invisible in final output
```

**The one-sentence summary worth remembering:** a route decorator maps a URL (optionally with dynamic `<type:name>` segments) to a Python function; that function reads any incoming data through `request` (`.form` for HTML forms, `.json` for APIs) and returns either rendered HTML (`render_template`, with Jinja2 handling the dynamic parts) or structured JSON (`jsonify`) — and which HTTP verb was used (GET/POST/PUT/DELETE) tells you what kind of action is being requested on that URL.
