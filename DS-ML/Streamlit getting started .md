# Getting Started with Streamlit

*Day X of my learning journal — Krish Naik's Complete Python Bootcamp ([Streamlit module](https://github.com/krishnaik06/Complete-Python-Bootcamp/tree/main/14-Streamlit))*

Out of everything in this bootcamp so far, this is the module that made me actually feel like "oh, I can ship something people can click on" instead of just running cells in a notebook. Streamlit turns a plain Python script into a working web app, no HTML/CSS/JS required. Writing this note the way I wish someone had framed it before I started.

---

## 1. What is Streamlit, and why does it feel like magic?

Normally, building a web app means learning a frontend (HTML/CSS/JS or React) *and* a backend (Flask/Django) *and* wiring them together. That's a lot of moving parts before you can even show someone a button.

Streamlit skips almost all of that. You write **plain Python**, top to bottom, and Streamlit turns it into a browser UI automatically. No routes, no templates, no `fetch()` calls to write.

**The analogy that made this click for me:** think of a normal web app like building a car from scratch — engine, chassis, wiring, all separate parts you have to assemble and connect yourself. Streamlit is more like a go-kart kit — way fewer parts, way less assembly, and you're driving in minutes. You trade some flexibility for a massive amount of speed. That trade-off is *exactly* why it's the go-to tool for data science demos, internal tools, and quick ML app prototypes — which is precisely why this module exists in a data science bootcamp instead of a "learn web dev" course.

### 1.1 Installing and running your first app

```bash
pip install streamlit
```

Every Streamlit app is just a `.py` file, run with a special command — **not** with `python app.py`:

```bash
streamlit run app.py
```

This opens a browser tab automatically, pointed at `localhost:8501`. That command is the one thing I kept forgetting the first day — running it with plain `python` does nothing useful, because Streamlit needs its own runtime to actually serve the page.

### 1.2 The core mental model: **the whole script reruns on every interaction**

This is the single most important thing to understand before writing any Streamlit code, and it wasn't obvious to me at first.

Every time you click a button, move a slider, or type into a text box, **Streamlit reruns your entire Python script from top to bottom** — not just the part near that widget. The output on screen just updates to match whatever the script produces this time.

**Why this matters practically:** if I have an expensive computation (like loading a big CSV or a trained model) sitting in the middle of my script with no caching, it reruns *every single time* the user touches any widget — even one that has nothing to do with that computation. This is exactly why caching (Section 5) isn't a "nice to have," it's basically required for any real app.

---

## 2. Building a web app — the core building blocks

### 2.1 Text and titles

```python
import streamlit as st

st.title("My First Streamlit App")
st.header("This is a header")
st.subheader("This is a subheader")
st.text("Plain text, no styling")
st.markdown("**Bold text** and *italic text* using markdown syntax")
st.write("st.write() is the do-it-all function — text, numbers, dataframes, charts, almost anything")
```

`st.write()` is the one I reach for the most when I'm just quickly checking something — it auto-detects what you passed it (string, number, DataFrame, chart) and renders it sensibly. The more specific functions (`st.title`, `st.header`, etc.) are what I use once I actually care about layout and structure.

### 2.2 Input widgets — this is where it stops being static

```python
name = st.text_input("Enter your name")
age = st.slider("Select your age", min_value=0, max_value=100, value=25)
department = st.selectbox("Choose your department", ["HR", "Engineering", "Sales", "Marketing"])
agree = st.checkbox("I agree to the terms")
submitted = st.button("Submit")

if submitted:
    st.write(f"Hello {name}, age {age}, from {department}!")
```

Every widget function **returns the current value the user picked**, right there in that variable. There's no separate "event listener" or "callback registration" step like you'd write in JS — you just read the return value directly, because remember: the whole script reruns every time anyway, so by the time you reach that line again, `name`/`age`/etc. already reflect the latest input.

Widgets I keep coming back to:

| Widget | Use case |
|---|---|
| `st.text_input()` | short text (name, search query) |
| `st.text_area()` | longer text (paragraph, feedback) |
| `st.number_input()` | a number with optional min/max/step |
| `st.slider()` | pick a number/range visually |
| `st.selectbox()` | pick one option from a dropdown |
| `st.multiselect()` | pick multiple options |
| `st.radio()` | pick one from a small visible set |
| `st.checkbox()` | on/off toggle |
| `st.button()` | trigger an action once |
| `st.file_uploader()` | let the user upload a file (CSV, image, etc.) |

### 2.3 Displaying data — DataFrames and charts

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "age": [30, 25, 28],
    "department": ["HR", "Engineering", "Sales"]
})

st.dataframe(df)          # interactive table — sortable, scrollable
st.table(df)              # static table — simpler, no interactivity

chart_data = pd.DataFrame(np.random.randn(20, 3), columns=["a", "b", "c"])
st.line_chart(chart_data)
st.bar_chart(chart_data)
st.area_chart(chart_data)
```

`st.dataframe()` vs `st.table()` was a small but useful distinction to learn: `dataframe` gives the user a scrollable, sortable grid (better for exploring real data), while `table` just renders a fixed snapshot (better for small, final summaries you don't want the user fiddling with).

### 2.4 Layout — columns, sidebar, tabs

Once an app has more than a handful of widgets, dumping everything in one vertical column gets messy fast. Streamlit gives a few layout tools:

```python
# sidebar - great for filters/settings that shouldn't take up main space
st.sidebar.title("Settings")
theme = st.sidebar.selectbox("Theme", ["Light", "Dark"])

# columns - place widgets side by side
col1, col2 = st.columns(2)
with col1:
    st.write("Left column content")
with col2:
    st.write("Right column content")

# tabs - separate sections the user clicks between
tab1, tab2 = st.tabs(["Overview", "Details"])
with tab1:
    st.write("This is the overview tab")
with tab2:
    st.write("This is the details tab")
```

**My rule of thumb now:** filters/settings that apply to the *whole* app → sidebar. Things that need to be compared side by side → columns. Distinct sections that don't need to be visible at the same time → tabs.

### 2.5 File uploads

```python
uploaded_file = st.file_uploader("Upload a CSV file", type=["csv"])

if uploaded_file is not None:
    df = pd.read_csv(uploaded_file)
    st.write("Preview of uploaded data:")
    st.dataframe(df.head())
```

The `type=["csv"]` restricts what file types the uploader accepts — a small thing, but it stops users from uploading a random `.exe` and confusing my app. `uploaded_file` behaves like a file object, so it plugs directly into `pd.read_csv()` without me needing to save it to disk first.

---

## 3. Putting it together — a small complete app

This is the "everything above, in one file" example I built to make sure the pieces actually connect:

```python
import streamlit as st
import pandas as pd

st.title("Employee Explorer")
st.write("A tiny app to filter and explore employee data.")

uploaded_file = st.sidebar.file_uploader("Upload employee CSV", type=["csv"])

if uploaded_file is not None:
    df = pd.read_csv(uploaded_file)

    department = st.sidebar.selectbox("Filter by department", ["All"] + list(df["department"].unique()))
    min_age = st.sidebar.slider("Minimum age", 18, 65, 18)

    filtered = df[df["age"] >= min_age]
    if department != "All":
        filtered = filtered[filtered["department"] == department]

    st.write(f"Showing {len(filtered)} employees")
    st.dataframe(filtered)
    st.bar_chart(filtered.groupby("department")["age"].mean())
else:
    st.info("Upload a CSV to get started.")
```

This is basically the shape of most small internal data tools I can already imagine building — upload/load data, filter it with sidebar widgets, show the result as a table and a chart. Almost every "data app" starter template boils down to this exact pattern.

---

## 4. Real-world example: an ML app with Streamlit

This is the part that made Streamlit click as *the* tool for showing off ML work — I don't need a separate frontend dev to demo a trained model, I can wrap it in an app myself in an afternoon.

**The idea:** train a simple classifier once, save it, then build a Streamlit app where a user enters some feature values and gets a live prediction back.

### 4.1 Training and saving a model (a one-time setup script)

```python
# train_model.py — run this once, separately, not part of the Streamlit app
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
import pickle

data = load_iris()
X, y = data.data, data.target

model = RandomForestClassifier()
model.fit(X, y)

with open("iris_model.pkl", "wb") as f:
    pickle.dump(model, f)

print("Model trained and saved.")
```

### 4.2 The Streamlit prediction app

```python
# app.py
import streamlit as st
import pickle
import numpy as np

st.title("Iris Flower Predictor")
st.write("Enter the flower's measurements below and I'll predict its species.")

@st.cache_resource
def load_model():
    with open("iris_model.pkl", "rb") as f:
        return pickle.load(f)

model = load_model()

sepal_length = st.slider("Sepal length (cm)", 4.0, 8.0, 5.8)
sepal_width = st.slider("Sepal width (cm)", 2.0, 4.5, 3.0)
petal_length = st.slider("Petal length (cm)", 1.0, 7.0, 4.0)
petal_width = st.slider("Petal width (cm)", 0.1, 2.5, 1.2)

species_names = ["Setosa", "Versicolor", "Virginica"]

if st.button("Predict species"):
    features = np.array([[sepal_length, sepal_width, petal_length, petal_width]])
    prediction = model.predict(features)[0]
    probabilities = model.predict_proba(features)[0]

    st.success(f"Predicted species: **{species_names[prediction]}**")

    st.write("Prediction confidence:")
    for name, prob in zip(species_names, probabilities):
        st.write(f"{name}: {prob * 100:.1f}%")
        st.progress(prob)
```

Walking through why each piece is there:
- **Training happens separately, once** — the app itself never re-trains the model. It just loads an already-trained one. This is the pattern I'll follow for any real ML app: train offline, serve online.
- **`st.slider()` for every feature** lets the user "build" an input row visually instead of typing raw numbers, which makes the whole thing feel like an actual interactive tool instead of a form.
- **`st.button("Predict species")`** means the prediction only runs when explicitly asked for, not on every single slider nudge — a small UX choice, but it stops the app from feeling twitchy while someone's still adjusting values.
- **`st.success()`** and **`st.progress()`** are the little polish touches — `success` gives a nice green highlight for the result, `progress` turns a plain probability number into a visual bar, which reads faster than a wall of decimals.

### 4.3 `st.cache_resource` — the piece that makes ML apps actually usable

Remember the core mental model from Section 1.2: **the whole script reruns on every interaction.** Without caching, `load_model()` would re-read the pickle file from disk *every single time* the user moves a slider — wasteful, and it would make the app feel sluggish for no reason.

`@st.cache_resource` tells Streamlit: "run this function once, keep the result in memory, and just hand back the cached object on every rerun instead of redoing the work." This decorator is specifically meant for things like models, database connections, or anything that's expensive to create but doesn't need to be recreated per interaction.

(There's a sibling decorator, `@st.cache_data`, for caching things like loaded DataFrames or computed results — same idea, just meant for data instead of "resource" objects like models/connections.)

This one decorator was the "oh, *that's* why my early test app felt slow" moment for me — I'd built a version without it first, on purpose, just to feel the difference.

---

## 5. My personal cheat sheet

```python
import streamlit as st

st.title("...")                    # page title
st.write(anything)                 # do-it-all display function

x = st.text_input("label")         # widgets return their current value directly
y = st.slider("label", 0, 100, 50)
clicked = st.button("Go")

if clicked:
    st.write("do something with x, y here")

st.sidebar.selectbox(...)          # sidebar for filters/settings
col1, col2 = st.columns(2)         # side-by-side layout
tab1, tab2 = st.tabs(["A", "B"])   # separate sections

@st.cache_resource                 # cache expensive one-time setup (models, connections)
def load_model(): ...

@st.cache_data                     # cache expensive data loading/computation
def load_data(): ...
```

Rules going forward:
1. Always run with `streamlit run app.py` — never plain `python`.
2. Remember the whole script reruns on *every* interaction — design around that instead of fighting it.
3. Cache anything expensive (`st.cache_resource` for models/connections, `st.cache_data` for data) — this is not optional for anything beyond a toy demo.
4. Sidebar = global controls, columns = side-by-side comparisons, tabs = separate sections. Pick layout deliberately instead of stacking everything vertically.
5. For ML apps specifically: train and pickle the model **offline**, load it once with caching, and let the app only handle input widgets + prediction + display.

Next up: want to try deploying one of these on Streamlit Community Cloud so I actually have a live link to put on LinkedIn instead of just a local `localhost:8501` screenshot.
