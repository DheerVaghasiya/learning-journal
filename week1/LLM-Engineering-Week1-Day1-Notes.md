# LLM Engineering — My Learning Notes

*Personal notes from my journey through [ed-donner/llm_engineering](https://github.com/ed-donner/llm_engineering). Writing these down in my own words as I go, so this section covers what I learned in the very first session.*

---

## What Actually Is an LLM (in practical terms)

Before writing any code, the first thing that clicked for me: a **frontier model** just means one of the current top-tier, most capable LLMs (think GPT-4o, Claude, Gemini) — as opposed to smaller **open-source models** you can run yourself on your own laptop, like Llama 3.2.

The distinction matters because it shapes *how* you access them:
- Frontier models → you call them through an API, and you pay per use
- Open-source models → you can download and run them locally, for free, offline

Knowing both paths exist means I'm never blocked by cost. I can always fall back to a local model to keep experimenting.

---

## Running My First Model Locally (Ollama)

I installed [Ollama](https://ollama.com) and ran a model directly from my terminal:

```bash
ollama run llama3.2
```

That's it — no API key, no signup, no cost. Within a minute I had a real LLM responding to me from my own machine.

**Note to self:** stick to `llama3.2`, not `llama3.3` — the latter is a 70B parameter model, way too heavy for a normal laptop to handle smoothly.

If the model doesn't start up, running `ollama serve` in a separate terminal window first fixes it — this starts the background service that `ollama run` depends on.

This was a good "instant gratification" moment — proof that I can build and test things without spending money while I'm still learning the ropes.

---

## Calling a Real Frontier Model via API

Next, I moved to calling an actual frontier model (`gpt-4o-mini`) through the OpenAI API. The core shape of the code is something I know I'll be reusing constantly for the rest of this course:

```python
from openai import OpenAI
openai = OpenAI()

response = openai.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Your question here"}
    ]
)
print(response.choices[0].message.content)
```

**Good to know:** if I don't want to spend anything on API credits, I can point the exact same `OpenAI` client at my local Ollama instance instead:

```python
openai = OpenAI(base_url='http://localhost:11434/v1', api_key='ollama')
# and swap model="gpt-4o-mini" → model="llama3.2"
```

Same code, same structure — just a different backend. That's a genuinely useful trick to keep in my back pocket.

---

## The Idea That Changed How I Think About Prompts

The biggest conceptual takeaway from Day 1 was understanding the difference between a **system prompt** and a **user prompt** — this is the actual backbone of how you "program" an LLM's behavior.

- **System prompt** = who the model *is* and how it should behave. Its role, its tone, its constraints, its output format.
- **User prompt** = the actual question or task I'm asking it to do, in the moment.

The way I think about it now: the system prompt is like giving someone a job description and instructions before they start; the user prompt is the actual task you hand them once they're already briefed. Getting the system prompt right does more heavy lifting than obsessively tweaking the user prompt.

---

## My First Real Project: Website Summarizer

This is where it stopped feeling like a tutorial and started feeling like building something real.

**What it does:** takes any website URL, scrapes the actual text content off the page, and gets an LLM to summarize it cleanly.

**How I built it, step by step:**

1. **Scrape the page** using `requests` to fetch the HTML and `BeautifulSoup` to parse it
2. **Clean the text** — strip out `<script>`, `<style>` tags, and navigation clutter that isn't actual content
3. **Write two prompts:**
   - A system prompt telling the model: *you're a summarizer, ignore navigation-related text, respond in Markdown*
   - A user prompt containing the actual scraped website text
4. **Call the API** and print the response — since it's Markdown, it renders nicely and readably in Jupyter

**Why this project actually matters to me:** this isn't just a toy example — summarization is one of the most commercially useful LLM applications out there. The exact same pipeline (scrape/gather → clean → prompt → call model → use output) applies to summarizing documents, meeting transcripts, reports, articles — basically any long text someone doesn't have time to read in full.

---

## What I'm Taking Away From This

- I don't need to spend a rupee to start learning — Ollama means I always have a free fallback
- The jump from "local model" to "frontier model via API" is a small code change but a real capability upgrade
- System vs. User prompt is a mental model I'll be using in *every single project* going forward, not just this one
- The real pattern behind LLM projects: **gather data → clean it → prompt engineering → call the model → do something useful with the output**

This is genuinely the first time I've built something with an LLM that felt like a real, usable tool rather than a demo — and that's a good sign for what's coming next.

---

*These are my personal study notes, written while working through Week 1, Day 1 of the [LLM Engineering course by Ed Donner](https://github.com/ed-donner/llm_engineering).*
