# Day 5: Building a Full Business Solution — A Multi-Step LLM Pipeline

## What we're building

Up to now (Days 1-4) we've made **one call** to an LLM and gotten **one response** back — summarize this text, answer this question, done. Day 5 is where things level up: we chain **two LLM calls together**, where the output of the first call feeds into the second. This pattern — using an LLM's output as an input to another LLM call — is the seed of something much bigger called **Agentic AI**, which we'll go deep on later in the course. For now, just hold onto this idea: *one call deciding what to do next, and another call doing the actual work.*

**The business problem:** given a company name and its website URL, automatically generate a marketing brochure — the kind of document you'd hand to a prospective customer, investor, or job candidate. A human doing this would (1) browse the website, (2) figure out which pages actually matter (About, Careers, not the Privacy Policy), (3) read those pages, and (4) write up a polished summary. We're going to make an LLM do all four steps.

---

## Quick recap: what we're standing on from Days 1-4

Before diving into new code, here's what's already familiar from earlier notes, because Day 5 reuses all of it:

- **System prompt vs user prompt** (Day 1): the system prompt tells the model *who it is and how to behave*; the user prompt gives it *the actual task and data*.
- **The OpenAI chat completions call** (`openai.chat.completions.create(...)`) (Day 1-2): the standard way to send messages and get a response.
- **Web scraping with `requests` + `BeautifulSoup`** (Day 1): pulling text content out of a webpage's HTML.
- **Streaming responses** (Day 4): getting text back token-by-token instead of waiting for the whole reply, for that "typewriter" effect.

Day 5 combines all four of these into a working pipeline. Nothing here is conceptually brand new — it's **composition**: taking small building blocks you already understand and wiring them together to do something a single call couldn't.

---

## The scraper: reading a website in code

We need two things from any webpage:
1. **The visible text content** (so an LLM can read it)
2. **The links on the page** (so we can decide which other pages to visit)

Here's the helper module (`scraper.py`) that does both:

```python
from bs4 import BeautifulSoup
import requests

# Websites often block requests that don't look like they're coming from a real browser.
# Sending a "User-Agent" header pretending to be Chrome on Windows helps us avoid being blocked.
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36"
}


def fetch_website_contents(url):
    """
    Return the title and contents of the website at the given url;
    truncate to 2,000 characters as a sensible limit
    """
    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.content, "html.parser")
    title = soup.title.string if soup.title else "No title found"
    if soup.body:
        # Strip out scripts, styling, images, and input fields - none of that is
        # useful text for an LLM to read, it's just noise
        for irrelevant in soup.body(["script", "style", "img", "input"]):
            irrelevant.decompose()
        text = soup.body.get_text(separator="\n", strip=True)
    else:
        text = ""
    return (title + "\n\n" + text)[:2_000]


def fetch_website_links(url):
    """
    Return the links on the website at the given url
    """
    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.content, "html.parser")
    links = [link.get("href") for link in soup.find_all("a")]
    return [link for link in links if link]  # drop any None values
```

**Why `.decompose()`?** When BeautifulSoup parses HTML it builds a tree of tags. `.decompose()` permanently removes a tag (and everything inside it) from that tree. We do this for `<script>`, `<style>`, `<img>`, and `<input>` tags because none of them contain human-readable text — they'd just clutter up what we send to the LLM and waste tokens (and money).

**Why truncate to 2,000 characters?** LLM APIs charge by token, and every model has a maximum context window. A big website's raw text could be huge. Truncating keeps costs and prompt size sane. It's a trade-off — we might cut off useful content — but for a landing page, the important stuff is usually near the top.

**Why does `fetch_website_links` return raw `href` values?** Because links on a page are often *relative* — something like `/about` instead of `https://company.com/about`. We haven't fixed that in Python; instead, we're about to hand this messy list to the LLM and ask *it* to figure out the full URLs. This is a neat trick: rather than writing brittle string-manipulation code to resolve relative links, we let the model's language understanding do it.

---

## Step 1: Ask the LLM which links actually matter

A real website has dozens of links — nav bar, footer, social icons, legal pages, cookie banners. We don't want to feed all of that to our brochure-writing call. So the *first* LLM call has one job: **look at the list of links and decide which ones are relevant to a company brochure** (About page, Careers page, etc.), and return them as clean, full URLs.

### The system prompt

```python
link_system_prompt = """
You are provided with a list of links found on a webpage.
You are able to decide which of the links would be most relevant to include in a brochure about the company,
such as links to an About page, or a Company page, or Careers/Jobs pages.
You should respond in JSON as in this example:

{
    "links": [
        {"type": "about page", "url": "https://full.url/goes/here/about"},
        {"type": "careers page", "url": "https://another.full.url/careers"}
    ]
}
"""
```

Two important ideas packed into this short prompt:

**1. One-shot prompting.** We don't just tell the model "respond in JSON" — we show it *exactly* what that JSON should look like, with a concrete example. This is called **one-shot prompting** (giving the model one example of the desired output format). It's far more reliable than describing a format in words, because the model can literally pattern-match the shape of its answer to your example. (If you gave *zero* examples, that'd be "zero-shot"; several examples would be "few-shot".)

**2. Why this task actually needs an LLM.** Imagine writing regular Python code to decide "is this link relevant to a brochure?" You'd need a pile of if/else rules — check if the URL contains "about" or "career" or "team", exclude anything with "privacy" or "terms"... and it would still break on any website that names things differently. This is a classic case where the task requires *nuanced understanding of meaning*, not pattern-matching on strings — exactly what LLMs are good at and traditional code is bad at.

### The user prompt

```python
def get_links_user_prompt(url):
    user_prompt = f"""
Here is the list of links on the website {url} -
Please decide which of these are relevant web links for a brochure about the company, 
respond with the full https URL in JSON format.
Do not include Terms of Service, Privacy, email links.

Links (some might be relative links):

"""
    links = fetch_website_links(url)
    user_prompt += "\n".join(links)
    return user_prompt
```

This just builds a plain-text prompt: instructions, followed by the raw list of scraped links dumped one per line. Simple string building — no magic here.

### Making the call, and forcing JSON output

```python
def select_relevant_links(url):
    print(f"Selecting relevant links for {url} by calling {MODEL}")
    response = openai.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": link_system_prompt},
            {"role": "user", "content": get_links_user_prompt(url)}
        ],
        response_format={"type": "json_object"}
    )
    result = response.choices[0].message.content
    links = json.loads(result)
    print(f"Found {len(links['links'])} relevant links")
    return links
```

**The key new piece here is `response_format={"type": "json_object"}`.** This is OpenAI's **JSON mode**. Normally, an LLM's output is just free-form text — even if you ask nicely for JSON, it might add a sentence before or after it, or produce almost-valid JSON with a stray comma. JSON mode makes the API *guarantee* the raw output is syntactically valid JSON, so `json.loads(result)` won't throw an error. This matters because we're about to treat this response as *structured data* — looping over `links['links']` — not just displaying it to a human.

> **Side note on Structured Outputs:** there's an even stricter version of this idea called "Structured Outputs," where you give the API an exact schema (like a Pydantic model) and it's forced to match that schema field-for-field, not just "some valid JSON." That's covered later in the course (Week 8, Agentic AI) — for now, `json_object` mode is enough.

At this point, calling `select_relevant_links("https://edwarddonner.com")` gives us back something like:

```json
{
    "links": [
        {"type": "about page", "url": "https://edwarddonner.com/about"},
        {"type": "connect four project", "url": "https://edwarddonner.com/connect-four"}
    ]
}
```

The LLM read a page full of raw hrefs and turned it into a clean, typed, structured list of *only* the URLs that matter for a brochure. That's step one of our pipeline done.

---

## Step 2: Gather all the content

Now we combine the landing page content with the content of every relevant link the first LLM call found:

```python
def fetch_page_and_all_relevant_links(url):
    contents = fetch_website_contents(url)
    relevant_links = select_relevant_links(url)
    result = f"## Landing Page:\n\n{contents}\n## Relevant Links:\n"
    for link in relevant_links['links']:
        result += f"\n\n### Link: {link['type']}\n"
        result += fetch_website_contents(link["url"])
    return result
```

Notice this function calls `select_relevant_links` (our first LLM call) *inside* it, then loops through the result and scrapes each of those pages too. So a single call to `fetch_page_and_all_relevant_links("https://huggingface.co")` triggers:
- 1 scrape of the landing page
- 1 LLM call to pick relevant links
- N more scrapes (one per relevant link found)

All of that gets assembled into one big markdown-ish string with headers, ready to hand to our second LLM call.

---

## Step 3: Write the actual brochure

### The system prompt

```python
brochure_system_prompt = """
You are an assistant that analyzes the contents of several relevant pages from a company website
and creates a short brochure about the company for prospective customers, investors and recruits.
Respond in markdown without code blocks.
Include details of company culture, customers and careers/jobs if you have the information.
"""
```

Notice: **no JSON mode this time.** For step 1 we needed structured data we could loop over in Python. For step 2, the final output is meant for a *human to read*, so free-form markdown text is exactly right. This is a good rule of thumb: use JSON mode when your code needs to programmatically consume the output; use plain text/markdown when a human is the final consumer.

The prompt also has a commented-out alternative version that asks for a "humorous, entertaining, witty" tone instead — a nice demonstration of how easily you can reshape an LLM's entire personality and output style just by editing a few words in the system prompt. Nothing else in the code changes.

### The user prompt

```python
def get_brochure_user_prompt(company_name, url):
    user_prompt = f"""
You are looking at a company called: {company_name}
Here are the contents of its landing page and other relevant pages;
use this information to build a short brochure of the company in markdown without code blocks.\n\n
"""
    user_prompt += fetch_page_and_all_relevant_links(url)
    user_prompt = user_prompt[:5_000]  # Truncate if more than 5,000 characters
    return user_prompt
```

Same truncation idea as before, just with a bigger budget (5,000 chars) since this prompt needs to hold the landing page *plus* several linked pages worth of content.

### Putting it together

```python
def create_brochure(company_name, url):
    response = openai.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "system", "content": brochure_system_prompt},
            {"role": "user", "content": get_brochure_user_prompt(company_name, url)}
        ],
    )
    result = response.choices[0].message.content
    display(Markdown(result))
```

A quick thing worth noticing: the *first* call uses a smaller/cheaper model (`gpt-5-nano`) for the "which links matter" decision, while the *second* call uses a slightly stronger model (`gpt-4.1-mini`) for the actual writing. This is a real cost-optimization pattern in production LLM systems — use a cheap, fast model for simple classification/decision steps, and reserve the more expensive model for the step that actually needs strong writing quality.

`display(Markdown(result))` is a Jupyter-notebook-only trick: it renders the markdown text as actual formatted text (headers, bold, bullet points) right in the notebook output, instead of showing raw `#` and `**` characters.

---

## Step 4: Make it stream

The last upgrade is to make the brochure appear progressively, word by word, instead of all at once (the "typewriter" effect from Day 4):

```python
def stream_brochure(company_name, url):
    stream = openai.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "system", "content": brochure_system_prompt},
            {"role": "user", "content": get_brochure_user_prompt(company_name, url)}
        ],
        stream=True
    )
    response = ""
    display_handle = display(Markdown(""), display_id=True)
    for chunk in stream:
        response += chunk.choices[0].delta.content or ''
        update_display(Markdown(response), display_id=display_handle.display_id)
```

How this works, piece by piece:
- `stream=True` tells OpenAI to send the response back as a series of small chunks instead of one complete blob.
- `display_handle = display(Markdown(""), display_id=True)` creates an *empty* markdown display in the notebook, but keeps a handle (`display_handle`) so we can update that same spot later instead of printing a new block every time.
- The `for chunk in stream:` loop receives each small piece of text as it arrives. `chunk.choices[0].delta.content` is the *new* bit of text in this chunk (it can be `None` on some chunks, hence `or ''` so we don't accidentally append the string `"None"`).
- We keep appending each new chunk to `response`, and call `update_display(...)` on every single chunk — so the same on-screen block keeps growing, giving that live "typing" feel.

---

## Full runnable script (OpenAI version)

```python
import os
import json
from dotenv import load_dotenv
from IPython.display import Markdown, display, update_display
from scraper import fetch_website_links, fetch_website_contents
from openai import OpenAI

# --- Setup ---
load_dotenv(override=True)
api_key = os.getenv('OPENAI_API_KEY')

if api_key and api_key.startswith('sk-proj-') and len(api_key) > 10:
    print("API key looks good so far")
else:
    print("There might be a problem with your API key!")

MODEL = 'gpt-5-nano'
openai = OpenAI()

# --- Step 1: pick relevant links ---
link_system_prompt = """
You are provided with a list of links found on a webpage.
You are able to decide which of the links would be most relevant to include in a brochure about the company,
such as links to an About page, or a Company page, or Careers/Jobs pages.
You should respond in JSON as in this example:

{
    "links": [
        {"type": "about page", "url": "https://full.url/goes/here/about"},
        {"type": "careers page", "url": "https://another.full.url/careers"}
    ]
}
"""

def get_links_user_prompt(url):
    user_prompt = f"""
Here is the list of links on the website {url} -
Please decide which of these are relevant web links for a brochure about the company,
respond with the full https URL in JSON format.
Do not include Terms of Service, Privacy, email links.

Links (some might be relative links):

"""
    links = fetch_website_links(url)
    user_prompt += "\n".join(links)
    return user_prompt

def select_relevant_links(url):
    response = openai.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": link_system_prompt},
            {"role": "user", "content": get_links_user_prompt(url)}
        ],
        response_format={"type": "json_object"}
    )
    result = response.choices[0].message.content
    return json.loads(result)

# --- Step 2: gather content ---
def fetch_page_and_all_relevant_links(url):
    contents = fetch_website_contents(url)
    relevant_links = select_relevant_links(url)
    result = f"## Landing Page:\n\n{contents}\n## Relevant Links:\n"
    for link in relevant_links['links']:
        result += f"\n\n### Link: {link['type']}\n"
        result += fetch_website_contents(link["url"])
    return result

# --- Step 3: write the brochure ---
brochure_system_prompt = """
You are an assistant that analyzes the contents of several relevant pages from a company website
and creates a short brochure about the company for prospective customers, investors and recruits.
Respond in markdown without code blocks.
Include details of company culture, customers and careers/jobs if you have the information.
"""

def get_brochure_user_prompt(company_name, url):
    user_prompt = f"""
You are looking at a company called: {company_name}
Here are the contents of its landing page and other relevant pages;
use this information to build a short brochure of the company in markdown without code blocks.\n\n
"""
    user_prompt += fetch_page_and_all_relevant_links(url)
    return user_prompt[:5_000]

def stream_brochure(company_name, url):
    stream = openai.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "system", "content": brochure_system_prompt},
            {"role": "user", "content": get_brochure_user_prompt(company_name, url)}
        ],
        stream=True
    )
    response = ""
    display_handle = display(Markdown(""), display_id=True)
    for chunk in stream:
        response += chunk.choices[0].delta.content or ''
        update_display(Markdown(response), display_id=display_handle.display_id)

# --- Run it ---
stream_brochure("HuggingFace", "https://huggingface.co")
```

---

## The Ollama version — running this fully locally, no API key needed

Everything above costs money and needs internet + an OpenAI key. **Ollama** lets you run open-weight models (like Llama 3.2, Mistral, or Qwen) directly on your own machine, for free, offline. The genuinely great news: Ollama exposes an **OpenAI-compatible API**, so almost none of our code above needs to change — we just point the `OpenAI` client at a local address instead of OpenAI's servers, and swap the model name.

### One-time setup

```bash
# 1. Install Ollama: https://ollama.com/download
# 2. Pull a model (only needs to be done once - downloads the model weights)
ollama pull llama3.2

# 3. Ollama runs a local server automatically after install,
#    but you can also start it manually:
ollama serve
```

By default, Ollama serves an API at `http://localhost:11434`.

### The code, adapted for Ollama

```python
import os
import json
from IPython.display import Markdown, display, update_display
from scraper import fetch_website_links, fetch_website_contents
from openai import OpenAI

# --- Setup: point the OpenAI client at Ollama instead of OpenAI's servers ---
OLLAMA_BASE_URL = "http://localhost:11434/v1"
MODEL = "llama3.2"

# Ollama doesn't check the API key at all, but the OpenAI client
# requires *some* non-empty string to be passed in, so we use a dummy value
ollama_client = OpenAI(base_url=OLLAMA_BASE_URL, api_key="ollama")

# --- Step 1: pick relevant links ---
link_system_prompt = """
You are provided with a list of links found on a webpage.
You are able to decide which of the links would be most relevant to include in a brochure about the company,
such as links to an About page, or a Company page, or Careers/Jobs pages.
You should respond in JSON as in this example:

{
    "links": [
        {"type": "about page", "url": "https://full.url/goes/here/about"},
        {"type": "careers page", "url": "https://another.full.url/careers"}
    ]
}
"""

def get_links_user_prompt(url):
    user_prompt = f"""
Here is the list of links on the website {url} -
Please decide which of these are relevant web links for a brochure about the company,
respond with the full https URL in JSON format.
Do not include Terms of Service, Privacy, email links.

Links (some might be relative links):

"""
    links = fetch_website_links(url)
    user_prompt += "\n".join(links)
    return user_prompt

def select_relevant_links(url):
    response = ollama_client.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": link_system_prompt},
            {"role": "user", "content": get_links_user_prompt(url)}
        ],
        format="json"  # Ollama's way of nudging JSON-shaped output
    )
    result = response.choices[0].message.content
    return json.loads(result)

# --- Step 2: gather content (unchanged - no LLM call here) ---
def fetch_page_and_all_relevant_links(url):
    contents = fetch_website_contents(url)
    relevant_links = select_relevant_links(url)
    result = f"## Landing Page:\n\n{contents}\n## Relevant Links:\n"
    for link in relevant_links['links']:
        result += f"\n\n### Link: {link['type']}\n"
        result += fetch_website_contents(link["url"])
    return result

# --- Step 3: write the brochure ---
brochure_system_prompt = """
You are an assistant that analyzes the contents of several relevant pages from a company website
and creates a short brochure about the company for prospective customers, investors and recruits.
Respond in markdown without code blocks.
Include details of company culture, customers and careers/jobs if you have the information.
"""

def get_brochure_user_prompt(company_name, url):
    user_prompt = f"""
You are looking at a company called: {company_name}
Here are the contents of its landing page and other relevant pages;
use this information to build a short brochure of the company in markdown without code blocks.\n\n
"""
    user_prompt += fetch_page_and_all_relevant_links(url)
    return user_prompt[:5_000]

def stream_brochure(company_name, url):
    stream = ollama_client.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": brochure_system_prompt},
            {"role": "user", "content": get_brochure_user_prompt(company_name, url)}
        ],
        stream=True
    )
    response = ""
    display_handle = display(Markdown(""), display_id=True)
    for chunk in stream:
        response += chunk.choices[0].delta.content or ''
        update_display(Markdown(response), display_id=display_handle.display_id)

# --- Run it ---
stream_brochure("HuggingFace", "https://huggingface.co")
```

### Things that trip people up with the Ollama version

- **JSON reliability is worse.** Small local models (like `llama3.2` at 3B parameters) are noticeably less reliable at producing perfectly valid JSON than GPT-5-nano. If `json.loads(result)` throws an error, that's the model going slightly off-script. Fixes: use a bigger local model (e.g. `qwen2.5:14b` or `llama3.1:8b`), lower the `temperature` (add `temperature=0` to the call, which makes output more deterministic/less "creative"), or wrap the `json.loads` call in a `try/except` and retry.
- **`format="json"` is Ollama's own parameter name** — it's not identical to OpenAI's `response_format={"type": "json_object"}`, but does the same job: instructs the model to only output valid JSON.
- **Speed depends entirely on your hardware.** No GPU means noticeably slower generation, especially for the streaming brochure step. This is the trade-off for "free and private" vs. "fast and API-hosted."
- **You can mix both approaches** in one project: use Ollama (free) for the cheap link-selection step, and OpenAI (paid, higher quality) for the final brochure-writing step — same idea as using `gpt-5-nano` vs `gpt-4.1-mini`, just swapping in a free local model for the easy part.

---

## Glossary — new terms from today

| Term | Meaning |
|---|---|
| **Multi-step LLM pipeline** | Chaining multiple LLM calls so one call's output becomes the next call's input. |
| **Agentic AI (preview)** | The broader design pattern where LLMs make decisions and take actions in sequence — this brochure generator is a tiny first taste of it. |
| **One-shot prompting** | Giving the model exactly one example of your desired output format inside the prompt, so it can mimic the pattern. |
| **JSON mode (`response_format`)** | An API setting that forces the raw output to be syntactically valid JSON, so your code can safely parse it. |
| **Structured Outputs** | A stricter version of JSON mode where you supply an exact schema the model must match field-for-field (covered later, Week 8). |
| **Streaming (`stream=True`)** | Receiving the response as a sequence of small chunks as they're generated, instead of waiting for the full text. |
| **`display_id` / `update_display`** | Jupyter-specific tools that let you keep re-rendering the *same* output cell instead of printing a new block each time — this is what makes streaming look smooth. |
| **Ollama** | A tool for running open-weight LLMs locally on your own machine, with an API that's compatible with the OpenAI Python client. |

---

## Why this matters for real business use

This pipeline — scrape → decide what's relevant → summarize/generate — generalizes far beyond brochures. Swap the prompts and you get: automated product tutorials from a spec, personalized outreach emails from a lead's company website, competitor analysis reports, or first-draft investor decks. The pattern (cheap model filters/decides → stronger model produces the final content) is a genuinely reusable architecture, not just a one-off notebook exercise.

---

## Week 1 End-of-Week Exercise: Build a technical Q&A tool

### The exercise, as given

> Demonstrate your familiarity with OpenAI API, and also Ollama, by building a tool that takes a technical question, and responds with an explanation. This is a tool you'll be able to use yourself during the course.

The starter notebook is basically a skeleton with the pieces named but not filled in:

```python
MODEL_GPT = 'gpt-4o-mini'
MODEL_LLAMA = 'llama3.2'

question = """
Please explain what this code does and why:
yield from {book.get("author") for book in books if book.get("author")}
"""

# Get gpt-4o-mini to answer, with streaming
# ...

# Get Llama 3.2 to answer
# ...
```

So the actual task is: take that `question` (a genuinely tricky piece of Python), send it to **both** a hosted model (GPT-4o-mini) and a local model (Llama 3.2 via Ollama), and get back a clear explanation from each — with the GPT call streaming its answer.

### Why this particular question is a good test case

`yield from {book.get("author") for book in books if book.get("author")}` packs three separate Python concepts into one line:
1. A **set comprehension** — `{... for book in books if ...}` builds a *set* (not a list), so duplicate authors automatically collapse into one.
2. A **conditional filter** inside the comprehension — `if book.get("author")` skips any book where `author` is missing, `None`, or an empty string (all of which are "falsy" in Python).
3. **`yield from`** — this turns the enclosing function into a generator that lazily yields each item from that set, one at a time, instead of returning them all as one collection at once.

Explaining *that* clearly, in plain English, to someone who doesn't already know all three concepts is exactly the kind of task an LLM is great at — which is why it's the perfect exercise to close out Week 1's "make LLMs answer questions well" theme.

### My solution — full code

```python
# imports
import os
from dotenv import load_dotenv
from openai import OpenAI
from IPython.display import Markdown, display, update_display

# constants
MODEL_GPT = 'gpt-4o-mini'
MODEL_LLAMA = 'llama3.2'

# set up environment
load_dotenv(override=True)
api_key = os.getenv('OPENAI_API_KEY')
openai = OpenAI()  # talks to OpenAI's servers, using OPENAI_API_KEY from the environment

# Ollama client - same OpenAI class, just pointed at localhost instead
ollama_via_openai = OpenAI(base_url='http://localhost:11434/v1', api_key='ollama')

# the question to ask
question = """
Please explain what this code does and why:
yield from {book.get("author") for book in books if book.get("author")}
"""

system_prompt = (
    "You are a helpful technical tutor who explains code clearly to a student "
    "who is still learning Python. Break your explanation into simple steps, "
    "and explain the purpose of each part of the code, not just what it does mechanically."
)

messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": question}
]

# --- Get gpt-4o-mini to answer, with streaming ---
def stream_gpt_answer():
    stream = openai.chat.completions.create(
        model=MODEL_GPT,
        messages=messages,
        stream=True
    )
    response = ""
    display_handle = display(Markdown(""), display_id=True)
    for chunk in stream:
        response += chunk.choices[0].delta.content or ''
        # strip stray markdown code fences some models like to add
        response = response.replace("```", "").replace("markdown", "")
        update_display(Markdown(response), display_id=display_handle.display_id)
    return response

stream_gpt_answer()

# --- Get Llama 3.2 to answer (via Ollama, running locally) ---
def get_llama_answer():
    response = ollama_via_openai.chat.completions.create(
        model=MODEL_LLAMA,
        messages=messages
    )
    result = response.choices[0].message.content
    display(Markdown(result))
    return result

get_llama_answer()
```

### Walking through the solution

- **Two separate `OpenAI` client objects.** `openai` points at the real OpenAI API (needs a valid `OPENAI_API_KEY`). `ollama_via_openai` points at `http://localhost:11434/v1` — Ollama's local, OpenAI-compatible server — and just needs *some* string for `api_key` since Ollama doesn't check it at all. This is the same trick from the Day 5 Ollama brochure code: one client class, two different backends, because they speak the same API shape.
- **One `messages` list reused for both calls.** Since both APIs accept the same `messages=[{"role": ..., "content": ...}]` format, we don't need to write the prompt twice — this is the practical payoff of Ollama being OpenAI-compatible.
- **A system prompt that shapes the *style* of the explanation**, not just "answer the question." Telling the model to explain to "a student who is still learning Python" and to explain *purpose*, not just mechanics, is what turns a terse one-line answer into an actual teaching explanation — same "system prompt controls tone and depth" idea we saw with the humorous-vs-formal brochure prompt in Day 5.
- **Streaming only for the GPT call.** The exercise specifically asks for streaming on the GPT-4o-mini answer, so that's where `stream=True` plus the `display_handle`/`update_display` pattern from Day 5 gets reused verbatim.
- **The `.replace("```", "")` line** is a small defensive touch — some models wrap explanations in a markdown code fence out of habit, which looks broken when rendered via `display(Markdown(...))`. Stripping stray fences keeps the rendered output clean.
- **The Llama call is plain, non-streaming**, since the exercise doesn't ask for streaming there — it's the simplest possible version: send messages, get one complete response back, display it.

### What I learned from this exercise

- **The same `messages` structure genuinely works unmodified across two completely different model providers** — that's the real "aha" of Ollama's OpenAI-compatible API. Once you've learned the OpenAI client shape, you already know how to talk to a huge range of local models too.
- **System prompts aren't just personality flavoring — they control depth and pedagogy.** Asking for "explain to a beginner, explain purpose not just mechanics" produced a genuinely more useful explanation than just forwarding the raw question with no framing at all.
- **Streaming is purely a "generic answer" pattern from Day 4/5, not something tied to any single business use case.** The exact same `display_handle` + `update_display` loop that streamed a company brochure also streams a code explanation — it's a reusable UI pattern, not brochure-specific code.
- **Local models are a genuinely usable second opinion, not just a novelty.** Running the same question through Llama 3.2 locally, next to GPT-4o-mini's answer, is a good habit to build early — comparing answers surfaces where a smaller local model's explanation is thinner or slightly wrong, which builds better intuition for when a local model is "good enough" versus when a hosted model's extra quality is worth paying for.
- **This exercise doubles as a genuinely useful personal tool.** The brief calls this out directly — "a tool you'll be able to use yourself during the course" — and it's true: any time a piece of code, an error message, or a concept doesn't make sense later in the course, this same question → dual-model-explanation script is ready to reuse as-is.

---

## Bonus Exercise: A standalone technical tutor, powered entirely by Ollama

### The exercise

Build a small, reusable tool: a **technical tutor** that runs fully on Ollama (no OpenAI key, no internet, no cost) and can answer *any* question I type in — not just one hardcoded question, but an interactive loop I can keep asking things in.

This is the natural next step after the Week 1 exercise above: instead of one question baked into the script, make it a proper little tool I can reach for any time something doesn't make sense — with a system prompt tuned specifically for *teaching*, not just answering.

### My solution — full code

```python
from openai import OpenAI
from IPython.display import Markdown, display, update_display

# --- Setup: Ollama's local, OpenAI-compatible server ---
MODEL = "llama3.2"
ollama = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

# --- The tutor's personality and teaching style ---
tutor_system_prompt = """
You are a patient, encouraging technical tutor helping a student learn 
data science, Python, and LLM engineering from scratch.

When answering:
- Assume the student is a beginner unless the question suggests otherwise.
- Explain the "why", not just the "what" - motivate why a concept or piece 
  of code exists before describing exactly what it does.
- Break multi-part answers into clearly labeled steps.
- Use a short, concrete example wherever it helps understanding.
- If the question is ambiguous, state the assumption you're making and answer anyway 
  rather than just asking for clarification.
- Keep answers focused - a good tutor doesn't ramble.
Respond in markdown.
"""

def ask_tutor(question, stream=True):
    """Send a single question to the local tutor and display the answer."""
    messages = [
        {"role": "system", "content": tutor_system_prompt},
        {"role": "user", "content": question}
    ]

    if not stream:
        response = ollama.chat.completions.create(model=MODEL, messages=messages)
        display(Markdown(response.choices[0].message.content))
        return

    stream_resp = ollama.chat.completions.create(model=MODEL, messages=messages, stream=True)
    result = ""
    display_handle = display(Markdown(""), display_id=True)
    for chunk in stream_resp:
        result += chunk.choices[0].delta.content or ''
        update_display(Markdown(result), display_id=display_handle.display_id)


def tutor_loop():
    """Keep asking the tutor questions until I type 'exit' or 'quit'."""
    print("Technical Tutor (Ollama / llama3.2) - type 'exit' to stop\n")
    while True:
        question = input("Your question: ")
        if question.strip().lower() in ("exit", "quit"):
            print("Goodbye!")
            break
        ask_tutor(question)


# One-off usage, e.g. inside a notebook cell:
ask_tutor("What's the difference between a list and a generator in Python?")

# Or run the interactive version in a terminal / script:
# tutor_loop()
```

### Walking through the solution

- **Single Ollama client, no OpenAI dependency at all.** Unlike the Week 1 exercise (which compared GPT and Llama side by side), this tool is deliberately Ollama-only — free, offline, private, and always available regardless of API credits or internet access.
- **`tutor_system_prompt` is the real design work here.** This is where a generic "answer questions" script becomes an actual *tutor*: assuming beginner level by default, explaining "why" before "what," using concrete examples, and keeping answers focused instead of rambling. This is the same idea from earlier notes — system prompts control depth and pedagogy, not just tone — pushed further into a full teaching persona.
- **`ask_tutor(question, stream=True)`** is a single reusable function instead of one-off notebook cells. Passing `stream=False` gives a plain non-streaming answer (useful for shorter factual questions); the default streams the answer live using the same `display_handle`/`update_display` pattern from Day 5 and the Week 1 exercise.
- **`tutor_loop()`** turns this into an actual interactive tool: a `while True` loop that keeps calling `input()` for a new question each time, and exits cleanly on typing `exit` or `quit`. This is the part that makes it a genuine standalone tool rather than a single hardcoded example — I can open this script any time during the course and just start asking things.
- **No conversation memory (yet).** Each call to `ask_tutor` sends a fresh `messages` list with just the system prompt and the new question — it doesn't remember earlier questions in the same session. That's a deliberate simplification for this exercise; extending it to a real multi-turn conversation (appending each question and answer to a running `messages` list) is a natural next step once memory-in-conversation is covered later in the course.

### What I learned from this exercise

- **A good "tutor" system prompt is mostly about explicit teaching instructions, not clever wording.** Telling the model *how* to teach (assume beginner, explain why before what, use examples, stay focused) did more work than any amount of fiddling with tone.
- **Wrapping a single API call in a small reusable function (`ask_tutor`) pays off immediately** — I can now call it from anywhere (a notebook cell, a script, later projects) instead of copy-pasting the same `messages`/`stream` boilerplate every time I have a new question.
- **A simple `while True` + `input()` loop is enough to turn a one-shot script into an actual interactive tool.** No web framework, no UI library needed — just a loop and an exit condition.
- **Running this fully on Ollama means zero friction to actually use it.** No API key to manage, no cost per question, works offline — which matters a lot for a tool I'm meant to reach for constantly throughout a two-year learning plan, not just once for a graded exercise.
- **The obvious next upgrade is conversation memory** (keeping the full `messages` history across questions so follow-ups like "can you explain that differently?" make sense) — good to flag for a future note once that concept is formally introduced.
