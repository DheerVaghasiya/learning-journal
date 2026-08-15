# Week 2 — Talking to Every Frontier Model, Building UIs, and Giving LLMs Hands (Tool Calling & Agents)

Week 1 was about *using* LLMs through chat interfaces and making our first API call. Week 2 is where things get real: we connect to every major model provider through code, build actual user interfaces around them with Gradio, turn a plain chatbot into something that can look things up and take action, and finish by combining text, tool calls, images and speech into one multi-modal assistant.

This file covers the entire week as one continuous topic instead of five separate days, because in practice these ideas build directly on top of each other — you can't understand tool calling without understanding the `messages` list, and you can't understand the multi-modal assistant without understanding Gradio's `Blocks`. So read it top to bottom.

> **A note on API keys used throughout this file:** every code example below is shown with the cloud provider (OpenAI, Anthropic, Gemini, etc.) *and* with an Ollama equivalent, because Ollama lets you run open-source models on your own machine for free, with no API key and no internet dependency. Whenever you see a `client = OpenAI(...)` block, assume there's a commented-out Ollama version right below it that does the same job locally.

---

## 1. The Big Idea: Every Model Provider Speaks (Almost) the Same Language

In Week 1 we called OpenAI's API directly. The natural question is: do Claude, Gemini, DeepSeek and the rest all need completely different code? Mostly, no — and this is the single most useful trick in this whole week.

OpenAI's Python library (`openai`) is really just a thin wrapper that sends HTTP requests to a URL. By default that URL is OpenAI's own servers. But the *shape* of the request (a `messages` list, a `model` name, a `temperature`, etc.) has become a de-facto industry standard, because so many providers built "OpenAI-compatible" endpoints so that developers wouldn't have to rewrite their code for every new model.

That means you can take the exact same `OpenAI` Python class and just point it at a different `base_url`:

```python
from openai import OpenAI

# Real OpenAI
openai = OpenAI()  # reads OPENAI_API_KEY from the environment automatically

# Anthropic (Claude), via OpenAI-compatible endpoint
anthropic = OpenAI(api_key=anthropic_api_key, base_url="https://api.anthropic.com/v1/")

# Google Gemini, via OpenAI-compatible endpoint
gemini = OpenAI(api_key=google_api_key, base_url="https://generativelanguage.googleapis.com/v1beta/openai/")

# DeepSeek
deepseek = OpenAI(api_key=deepseek_api_key, base_url="https://api.deepseek.com")

# Groq (super-fast inference of open models)
groq = OpenAI(api_key=groq_api_key, base_url="https://api.groq.com/openai/v1")

# xAI's Grok
grok = OpenAI(api_key=grok_api_key, base_url="https://api.x.ai/v1")

# OpenRouter — a single gateway that can reach dozens of providers/models
openrouter = OpenAI(api_key=openrouter_api_key, base_url="https://openrouter.ai/api/v1")

# Ollama — models running locally on YOUR machine, no real key needed
# Ollama needs a value in the api_key field even though it ignores it, so we just pass a placeholder string
ollama = OpenAI(api_key="ollama", base_url="http://localhost:11434/v1")
```

Once you've built one of these client objects, calling any of them looks identical:

```python
response = openai.chat.completions.create(model="gpt-4.1-mini", messages=messages)
response = anthropic.chat.completions.create(model="claude-sonnet-4-5-20250929", messages=messages)
response = ollama.chat.completions.create(model="llama3.2", messages=messages)

print(response.choices[0].message.content)
```

**Why this matters in the real world:** this pattern is exactly how production systems stay flexible. If OpenAI has an outage, or a cheaper/better model comes out from another provider, you swap one `base_url` and `model` string — none of your business logic, prompts, or UI code needs to change. This is the foundation that libraries like LangChain and LiteLLM (covered later) automate for you.

### Setting up your `.env` file

API keys should never be hard-coded into your scripts (if you push that to GitHub, bots will find and abuse it within minutes). Instead, put them in a `.env` file in your project root, which is loaded into environment variables at runtime and is excluded from git via `.gitignore`:

```
OPENAI_API_KEY=xxxx
ANTHROPIC_API_KEY=xxxx
GOOGLE_API_KEY=xxxx
DEEPSEEK_API_KEY=xxxx
GROQ_API_KEY=xxxx
GROK_API_KEY=xxxx
OPENROUTER_API_KEY=xxxx
```

Ollama doesn't need a real key at all — it's running on your own computer — but the OpenAI client library requires *some* string in the `api_key` field, so the convention is to just pass `"ollama"`.

Loading it in Python:

```python
import os
from dotenv import load_dotenv

load_dotenv(override=True)  # override=True re-reads the file even if you already loaded it once this session

openai_api_key = os.getenv('OPENAI_API_KEY')
anthropic_api_key = os.getenv('ANTHROPIC_API_KEY')
# ...and so on for each provider
```

Good habit: print only the first few characters of each key to confirm it loaded, never the whole thing:

```python
if openai_api_key:
    print(f"OpenAI API Key exists and begins {openai_api_key[:8]}")
else:
    print("OpenAI API Key not set")
```

**Important:** every time you edit `.env`, save the file *and* re-run `load_dotenv(override=True)` — otherwise your notebook keeps using the old, cached values.

### Reaching for the provider's own SDK instead

Going through OpenAI's client is convenient, but each provider also ships its own native Python library, which sometimes exposes provider-specific features the OpenAI-compatible shim doesn't. For example:

```python
# Google's native SDK
from google import genai
client = genai.Client()
response = client.models.generate_content(
    model="gemini-2.5-flash-lite",
    contents="Describe the color Blue to someone who's never been able to see, in 1 sentence"
)
print(response.text)

# Anthropic's native SDK
from anthropic import Anthropic
client = Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-5-20250929",
    messages=[{"role": "user", "content": "Describe the color Blue to someone who's never been able to see, in 1 sentence"}],
    max_tokens=100
)
print(response.content[0].text)
```

Notice Anthropic's native API requires `max_tokens` explicitly (the OpenAI-shim usually fills in a default for you) and returns the reply as `response.content[0].text` rather than `response.choices[0].message.content`. Small differences like this are exactly why "OpenAI-compatible" shims exist — to hide them.

---

## 2. Reasoning Effort and the New "Test-Time Scaling" Models

Newer reasoning-focused models (like the GPT-5 family) let you dial how much internal "thinking" the model does before answering, using a `reasoning_effort` parameter (`"minimal"`, `"low"`, `"medium"`, `"high"`):

```python
response = openai.chat.completions.create(
    model="gpt-5-nano",
    messages=easy_puzzle,
    reasoning_effort="minimal"
)
```

This connects to a bigger idea worth understanding properly: **training-time scaling vs inference-time (test-time) scaling.**

- **Training-time scaling** is the classic approach: make the model bigger, train it on more data, for longer. This is what made GPT-3 → GPT-4 style jumps happen.
- **Inference-time scaling** is newer: keep the same model, but let it "think" for longer at the moment you ask it a question — generating an internal chain of reasoning before committing to a final answer. Reasoning-effort settings are a direct knob on this.

The trade-off is simple and important for real products: higher reasoning effort means better answers on hard logic/math problems, but slower responses and higher cost. For a simple factual lookup or a casual chatbot reply, `"minimal"` is usually plenty. For a hard multi-step reasoning task (debugging code, solving a word problem, planning), bump it up.

A nice way to see this in action is to give the same puzzle to a model at different effort levels and compare:

```python
easy_puzzle = [{"role": "user", "content":
    "You toss 2 coins. One of them is heads. What's the probability the other is tails? Answer with the probability only."}]

for effort in ["minimal", "low"]:
    response = openai.chat.completions.create(model="gpt-5-nano", messages=easy_puzzle, reasoning_effort=effort)
    print(effort, "->", response.choices[0].message.content)
```

(This particular puzzle is a classic probability trap — the naive answer "1/2" is wrong once you account for how the "one of them is heads" information was obtained; higher reasoning effort tends to catch this, lower effort often doesn't. It's a good gut-check for whether a model is actually reasoning or just pattern-matching.)

---

## 3. Benchmarking Models Against Each Other on Brain Teasers

Once you can call multiple providers through the same interface, the obvious next move is to run the *same* prompt against several models and compare. This is a genuinely useful engineering habit, not just a fun demo — it's how you decide which model is actually worth paying for on your specific task.

A classic "hard" reasoning riddle used in the course:

```python
hard = """
On a bookshelf, two volumes of Pushkin stand side by side: the first and the second.
The pages of each volume together have a thickness of 2 cm, and each cover is 2 mm thick.
A worm gnawed (perpendicular to the pages) from the first page of the first volume to the last page of the second volume.
What distance did it gnaw through?
"""
hard_puzzle = [{"role": "user", "content": hard}]

response = anthropic.chat.completions.create(model="claude-sonnet-4-5-20250929", messages=hard_puzzle)
response = openai.chat.completions.create(model="gpt-5", messages=hard_puzzle)
response = gemini.chat.completions.create(model="gemini-2.5-pro", messages=hard_puzzle)
```

The trick with this puzzle: on a real bookshelf, when two volumes stand side by side in reading order, volume 1's *first page* is actually pressed against volume 2, and volume 2's *last page* is also pressed against volume 1 from the other side — meaning the worm only has to travel through the two covers in between (4 mm total), not through either book's actual pages. It's a spatial-reasoning trap, and it's a great one-question test of whether a model is truly reasoning about the physical setup or just pattern-matching "worm + book = add up page thicknesses."

Another good benchmarking prompt is a game-theory dilemma, because it reveals a model's "personality" under pressure — do different models cooperate or defect?

```python
dilemma_prompt = """
You and a partner are contestants on a game show. You're each taken to separate rooms and given a choice:
Cooperate: Choose "Share" — if both of you choose this, you each win $1,000.
Defect: Choose "Steal" — if one steals and the other shares, the stealer gets $2,000 and the sharer gets nothing.
If both steal, you both get nothing.
Do you choose to Steal or Share? Pick one.
"""
dilemma = [{"role": "user", "content": dilemma_prompt}]

response = anthropic.chat.completions.create(model="claude-sonnet-4-5-20250929", messages=dilemma)
response = groq.chat.completions.create(model="openai/gpt-oss-120b", messages=dilemma)
response = deepseek.chat.completions.create(model="deepseek-reasoner", messages=dilemma)
response = grok.chat.completions.create(model="grok-4", messages=dilemma)
```

This is a miniature version of the famous "Prisoner's Dilemma" from game theory. There's no single "correct" answer — but comparing how confidently/consistently each model reasons through it, and whether it explains its choice well, tells you something about that model's alignment and reasoning style.

**Real-world takeaway:** before committing your product to a single LLM provider, run your *actual* task prompts (not generic riddles) against 3-4 candidate models and compare quality, latency, and cost. This "bake-off" pattern is standard practice at any company shipping an LLM feature.

---

## 4. Running Models Locally with Ollama

Every cloud call costs money and requires internet access and sends your data to a third party. Ollama solves all three problems by running open-source models directly on your own laptop.

```python
import requests
requests.get("http://localhost:11434/").content
# If this fails, run `ollama serve` from a terminal first
```

Pulling (downloading) a model — this only needs to happen once per model:

```bash
ollama pull llama3.2          # a small, fast general model — good for most laptops
ollama pull gpt-oss:20b       # a much bigger open-weight model — needs at least 16GB RAM
```

Once pulled, you call it exactly like any other OpenAI-compatible client, because Ollama exposes an OpenAI-compatible local server on port 11434:

```python
ollama = OpenAI(api_key="ollama", base_url="http://localhost:11434/v1")

response = ollama.chat.completions.create(model="llama3.2", messages=easy_puzzle)
print(response.choices[0].message.content)
```

**When to reach for Ollama in real projects:**
- Prototyping without burning API credits while you iterate on a prompt.
- Working with sensitive/private data that can't leave your machine.
- Building offline-capable tools.
- Learning — free, unlimited experimentation.

The trade-off is quality and speed: local open models (especially small ones like `llama3.2`) are generally weaker than frontier cloud models like GPT-5 or Claude Sonnet, and inference is only as fast as your own CPU/GPU. A very common real-world pattern is: develop and test with Ollama, then switch the `model` string and client to a cloud provider for production quality — which the "OpenAI-compatible everywhere" pattern from Section 1 makes almost effortless.

---

## 5. Frameworks for Managing Multiple LLMs: LangChain vs LiteLLM

Once you're juggling 5+ providers, you start wanting a library that removes the provider-specific boilerplate entirely. Two popular options:

### LangChain

LangChain is a large, full-featured framework for building LLM applications — chains of prompts, memory, agents, retrieval, and much more. For a single call, it looks like this:

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-5-mini")
response = llm.invoke(tell_a_joke)  # tell_a_joke is a normal messages list
print(response.content)
```

LangChain's strength is its huge ecosystem of pre-built components (document loaders, vector store integrations, agent frameworks). Its downside — one that a lot of practicing engineers complain about — is that it's *heavyweight*: lots of abstraction layers, frequent breaking changes between versions, and a steep learning curve for what can be a simple task.

### LiteLLM

LiteLLM takes a much lighter-weight approach: one function, `completion()`, and you just change the `model` string's prefix to pick the provider:

```python
from litellm import completion

response = completion(model="openai/gpt-4.1", messages=tell_a_joke)
response = completion(model="gemini/gemini-2.5-flash-lite", messages=tell_a_joke)
# response = completion(model="ollama/llama3.2", messages=tell_a_joke)  # works locally too

reply = response.choices[0].message.content
```

A genuinely useful bonus: LiteLLM normalizes usage/cost reporting across providers, so you can do this regardless of which model you called:

```python
print(f"Input tokens: {response.usage.prompt_tokens}")
print(f"Output tokens: {response.usage.completion_tokens}")
print(f"Total tokens: {response.usage.total_tokens}")
print(f"Total cost: {response._hidden_params['response_cost']*100:.4f} cents")
```

**Which should you pick?** If you're building a small tool or script that just needs to call several LLMs interchangeably, LiteLLM is usually the pragmatic choice — it does one job and does it simply. If you're building a larger application that also needs retrieval-augmented generation, agents, memory, and document processing pipelines, LangChain's broader ecosystem may save you time despite the extra complexity. Many production teams actually use both: LiteLLM as the low-level "any model, any provider" call layer, and LangChain (or a lighter alternative) on top for orchestration.

---

## 6. Prompt Caching — Making Repeated Context Cheap

If you send the same long context (a system prompt, a document, a big instruction block) on every request, you're paying full price to re-process it every single time. Prompt caching lets providers remember a prefix of your prompt so repeated calls are cheaper and faster.

| Provider | How it works |
|---|---|
| **OpenAI** | Automatic for exact-prefix matches. Put static content (instructions, examples) at the *start* of your prompt and variable content at the *end*, since caching only works on exact prefix matches. Cached input tokens cost about 4× cheaper. |
| **Anthropic (Claude)** | You must explicitly mark what to cache. You pay ~25% *more* the first time to "prime" the cache, then ~10× less on subsequent reuses of that same cached block. |
| **Google Gemini** | Supports both *implicit* caching (automatic) and *explicit* caching (you control it yourself). |

A practical demonstration using LiteLLM against Gemini, loading the entire text of *Hamlet* as context:

```python
with open("hamlet.txt", "r", encoding="utf-8") as f:
    hamlet = f.read()

question = [{"role": "user", "content": "In Hamlet, when Laertes asks 'Where is my father?' what is the reply?"}]

# First call — no context yet, the model may hallucinate an answer
response = completion(model="gemini/gemini-2.5-flash-lite", messages=question)

# Now attach the whole play as context
question[0]["content"] += "\n\nFor context, here is the entire text of Hamlet:\n\n" + hamlet
response = completion(model="gemini/gemini-2.5-flash-lite", messages=question)

print(f"Input tokens: {response.usage.prompt_tokens}")
print(f"Cached tokens: {response.usage.prompt_tokens_details.cached_tokens}")
```

Run the *same* question again right after, and you'll see `cached_tokens` jump up and the reported cost drop — because Gemini recognized it had already processed that huge Hamlet text moments earlier.

**Real-world relevance:** any product that repeatedly sends a long system prompt, a knowledge base excerpt, or a big rulebook to the same model (e.g. a customer support bot that always includes your entire FAQ) should be designed with caching in mind — put the stable, reusable material first, and the unique per-request material last.

---

## 7. The `messages` List — the Backbone of Every Conversation

By now you've seen this structure over and over:

```python
[
    {"role": "system", "content": "system message here"},
    {"role": "user", "content": "user prompt here"}
]
```

This same structure extends naturally to represent an entire conversation history:

```python
[
    {"role": "system", "content": "system message here"},
    {"role": "user", "content": "first user prompt"},
    {"role": "assistant", "content": "the assistant's earlier reply"},
    {"role": "user", "content": "the new user prompt"},
]
```

Every time you call the API, you send the *whole* conversation so far — the model itself has no memory between calls; **you** are responsible for keeping and re-sending the history. This is the single most important mental model to internalize this week: an LLM API call is stateless, and "conversation" is an illusion your application code creates by re-sending growing history.

### A fun (and genuinely instructive) demo: two chatbots arguing with each other

```python
gpt_model = "gpt-4.1-mini"
claude_model = "claude-haiku-4-5"

gpt_system = "You are a chatbot who is very argumentative; you disagree with anything in the conversation and challenge everything, in a snarky way."
claude_system = "You are a very polite, courteous chatbot. You try to agree with everything the other person says, or find common ground. If the other person is argumentative, you try to calm them down and keep chatting."

gpt_messages = ["Hi there"]
claude_messages = ["Hi"]

def call_gpt():
    messages = [{"role": "system", "content": gpt_system}]
    for gpt, claude in zip(gpt_messages, claude_messages):
        messages.append({"role": "assistant", "content": gpt})
        messages.append({"role": "user", "content": claude})
    response = openai.chat.completions.create(model=gpt_model, messages=messages)
    return response.choices[0].message.content

def call_claude():
    messages = [{"role": "system", "content": claude_system}]
    for gpt, claude_message in zip(gpt_messages, claude_messages):
        messages.append({"role": "user", "content": gpt})
        messages.append({"role": "assistant", "content": claude_message})
    messages.append({"role": "user", "content": gpt_messages[-1]})
    response = anthropic.chat.completions.create(model=claude_model, messages=messages)
    return response.choices[0].message.content

for i in range(5):
    gpt_next = call_gpt()
    gpt_messages.append(gpt_next)
    claude_next = call_claude()
    claude_messages.append(claude_next)
```

Look closely at `call_gpt()` and `call_claude()`: from GPT's point of view, everything Claude ever said gets relabeled as `"user"` (because to GPT, Claude *is* the user it's talking to), and everything GPT itself said becomes `"assistant"`. From Claude's point of view, it's the exact opposite. This role-flipping is the key insight for building any multi-agent or multi-bot conversation.

**Extending to 3+ participants:** once you have more than two "speakers," the alternating `user`/`assistant` role trick breaks down (an API call only really supports one system role and one ongoing user/assistant thread). The reliable pattern for 3+ participants is to flatten the whole conversation into a single user prompt as plain text, and let a single system prompt define "who you are":

```python
system_prompt = """
You are Alex, a chatbot who is very argumentative; you disagree with anything in the conversation and challenge everything, in a snarky way.
You are in a conversation with Blake and Charlie.
"""

user_prompt = f"""
You are Alex, in conversation with Blake and Charlie.
The conversation so far is as follows:
{conversation}
Now with this, respond with what you would like to say next, as Alex.
"""
```

Here `conversation` would just be a plain-text transcript like `"Blake: hello\nCharlie: hi there\n..."`. This is a much more general pattern than the role-flipping trick, and it's how most real multi-agent frameworks work under the hood.

---

## 8. Gradio: Building Real User Interfaces Without Any Frontend Code

Everything so far has lived inside a notebook. Gradio turns any Python function into a shareable web app in a couple of lines — no HTML, CSS, or JavaScript required.

### The absolute basics

```python
import gradio as gr

def shout(text):
    return text.upper()

gr.Interface(fn=shout, inputs="textbox", outputs="textbox", flagging_mode="never").launch()
```

`gr.Interface` takes a function, wires up one input and one output component, and gives you a running local web app (by default at `http://127.0.0.1:7860`). Three useful launch options:

```python
.launch(share=True)      # generates a public URL others can access (uses HTTP tunneling — some antivirus/corporate networks block this)
.launch(inbrowser=True)  # automatically opens a browser tab for you
.launch(auth=("ed", "bananas"))  # adds basic username/password protection
```

**Security note:** the `auth=(...)` tuple above is fine for a demo, but never hard-code real credentials this way in a shared or production app — pull them from your `.env` file at minimum, and use a proper auth system for anything real.

Gradio also lets you force dark mode via a small injected JavaScript snippet (Gradio recommends *against* doing this by default, since theme should usually be the user's own OS/browser preference, but it's there if you need it):

```python
force_dark_mode = """
function refresh() {
    const url = new URL(window.location);
    if (url.searchParams.get('__theme') !== 'dark') {
        url.searchParams.set('__theme', 'dark');
        window.location.href = url.href;
    }
}
"""
gr.Interface(fn=shout, inputs="textbox", outputs="textbox", js=force_dark_mode).launch()
```

### Nicer components, and wiring in an actual LLM

Instead of the shorthand `"textbox"` strings, you can build proper `gr.Textbox` components with labels, hints, and sizing — and swap in a real model call as the function:

```python
def message_gpt(prompt):
    messages = [{"role": "system", "content": "You are a helpful assistant"}, {"role": "user", "content": prompt}]
    response = openai.chat.completions.create(model="gpt-4.1-mini", messages=messages)
    return response.choices[0].message.content
    # Ollama version:
    # response = ollama.chat.completions.create(model="llama3.2", messages=messages)
    # return response.choices[0].message.content

message_input = gr.Textbox(label="Your message:", info="Enter a message for GPT-4.1-mini", lines=7)
message_output = gr.Textbox(label="Response:", lines=8)

gr.Interface(
    fn=message_gpt,
    title="GPT",
    inputs=[message_input],
    outputs=[message_output],
    examples=["hello", "howdy"],
    flagging_mode="never"
).launch()
```

Swap the output component for `gr.Markdown(label="Response:")` and instruct the model to reply in Markdown, and replies render with proper formatting (headings, bold, lists) instead of raw text.

### Streaming responses

Waiting several seconds for a full response to appear feels sluggish. Real chat products stream text token-by-token as it's generated — you can do the same with a Python **generator** function (one that uses `yield` instead of `return`):

```python
def stream_gpt(prompt):
    messages = [{"role": "system", "content": system_message}, {"role": "user", "content": prompt}]
    stream = openai.chat.completions.create(model='gpt-4.1-mini', messages=messages, stream=True)
    result = ""
    for chunk in stream:
        result += chunk.choices[0].delta.content or ""
        yield result   # yield the growing string each time — Gradio re-renders it live
```

If `yield`/generators are unfamiliar: a normal function returns one value and ends. A generator function can pause and hand back a value multiple times using `yield`, resuming where it left off on the next iteration. Gradio specifically knows how to consume a generator function as a live-updating output — every `yield` refreshes what the user sees, which is exactly how streaming chat UIs work.

The same pattern works identically for Claude — just swap the client and model:

```python
def stream_claude(prompt):
    messages = [{"role": "system", "content": system_message}, {"role": "user", "content": prompt}]
    stream = anthropic.chat.completions.create(model='claude-sonnet-4-5-20250929', messages=messages, stream=True)
    result = ""
    for chunk in stream:
        result += chunk.choices[0].delta.content or ""
        yield result
```

And you can let the *user* pick which model to use with a dropdown, routing to whichever streaming function matches:

```python
def stream_model(prompt, model):
    if model == "GPT":
        result = stream_gpt(prompt)
    elif model == "Claude":
        result = stream_claude(prompt)
    elif model == "Ollama":
        result = stream_ollama(prompt)  # same pattern, pointed at the local Ollama client
    else:
        raise ValueError("Unknown model")
    yield from result   # forwards every yielded chunk from the inner generator

model_selector = gr.Dropdown(["GPT", "Claude", "Ollama"], label="Select model", value="GPT")
```

### A real mini-project: a company brochure generator

Combining web scraping (from Week 1) with streaming and a model picker gives you a genuinely useful tool in under 30 lines:

```python
from scraper import fetch_website_contents  # reused from Week 1

system_message = """
You are an assistant that analyzes the contents of a company website landing page
and creates a short brochure about the company for prospective customers, investors and recruits.
Respond in markdown without code blocks.
"""

def stream_brochure(company_name, url, model):
    yield ""  # clear the previous output immediately when a new request starts
    prompt = f"Please generate a company brochure for {company_name}. Here is their landing page:\n"
    prompt += fetch_website_contents(url)
    result = stream_gpt(prompt) if model == "GPT" else stream_claude(prompt)
    yield from result

gr.Interface(
    fn=stream_brochure,
    title="Brochure Generator",
    inputs=[gr.Textbox(label="Company name:"), gr.Textbox(label="Landing page URL"), gr.Dropdown(["GPT", "Claude"], value="GPT")],
    outputs=[gr.Markdown(label="Response:")],
    examples=[["Hugging Face", "https://huggingface.co", "GPT"]],
).launch()
```

---

## 9. Building Conversational Assistants with `gr.ChatInterface`

`gr.Interface` is great for "one input → one output" tools. For an ongoing back-and-forth conversation, Gradio provides a purpose-built component: `gr.ChatInterface`. It automatically manages the chat bubble UI and conversation history for you — your job is just to write one callback function:

```python
def chat(message, history):
    return "bananas"

gr.ChatInterface(fn=chat, type="messages").launch()
```

Gradio calls your `chat(message, history)` function every time the user sends a message. `message` is the new text they typed; `history` is the full list of previous turns (already in the `{"role": ..., "content": ...}` format from Section 7). Your job is to turn that into a real LLM call:

```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model=MODEL, messages=messages)
    return response.choices[0].message.content
```

And streaming works exactly the same way as before — just `yield` a generator instead of `return`:

```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history + [{"role": "user", "content": message}]
    stream = openai.chat.completions.create(model=MODEL, messages=messages, stream=True)
    response = ""
    for chunk in stream:
        response += chunk.choices[0].delta.content or ''
        yield response

gr.ChatInterface(fn=chat, type="messages").launch()
```

### Shaping behavior with the system prompt (one-shot / few-shot prompting)

The system prompt is your main lever for controlling *how* the assistant behaves, without touching any code. This is called **few-shot prompting** when you include example exchanges to teach a pattern (one example = "one-shot"):

```python
system_message = """You are a helpful assistant in a clothes store. You should try to gently encourage
the customer to try items that are on sale. Hats are 60% off, and most other items are 50% off.
For example, if the customer says 'I'm looking to buy a hat',
you could reply something like, 'Wonderful - we have lots of hats - including several that are part of our sales event.'
Encourage the customer to buy hats if they are unsure what to get."""
```

You can even build **dynamic** system prompts that adapt based on what the user just said — a lightweight form of steering the model's behavior per-message:

```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    relevant_system_message = system_message
    if 'belt' in message.lower():
        relevant_system_message += " The store does not sell belts; if asked for belts, point out other sale items instead."
    messages = [{"role": "system", "content": relevant_system_message}] + history + [{"role": "user", "content": message}]
    stream = openai.chat.completions.create(model=MODEL, messages=messages, stream=True)
    response = ""
    for chunk in stream:
        response += chunk.choices[0].delta.content or ''
        yield response
```

This "detect a keyword, tweak the system prompt" trick is a simple but genuinely production-useful pattern — a cheap way to add targeted behavior without a full tool-calling setup.

### A first taste of RAG (Retrieval-Augmented Generation)

You've actually already brushed up against the *idea* behind RAG earlier in this file, without naming it: in Section 6, we manually pasted the entire text of Hamlet into the prompt so the model could answer questions about it accurately instead of guessing from memory. That — "give the model the specific facts it needs, right inside the prompt, instead of hoping it already knows them" — is the whole idea behind RAG.

**Retrieval-Augmented Generation** is what you call this pattern once it's done properly at scale:

1. You have a large knowledge source (documents, a database, your company's internal wiki) — far too large to paste into every prompt.
2. When a user asks a question, you first **retrieve** just the small handful of relevant chunks (usually using a search technique like vector/semantic search).
3. You **augment** the prompt by inserting those retrieved chunks as context.
4. The model then **generates** an answer grounded in that specific, retrieved information instead of relying purely on what it memorized during training.

Why this matters: LLMs can be confidently wrong (a "hallucination") about facts they weren't trained on, facts that changed after their training cutoff, or facts specific to your business (your product catalog, your internal policies). RAG fixes this by grounding every answer in real, retrieved source material — and because you control what gets retrieved, you also get accurate citations ("this answer came from page 12 of the manual"). We'll build a full RAG pipeline with proper vector search later in the course — for now, the mental model to hold onto is simply: *retrieve the relevant facts first, then let the LLM write the answer using those facts.*

---

## 10. Tool Calling — Giving the LLM the Power to Act, Not Just Talk

Everything up to now has been "the model talks, you display the words." Tool calling (also called *function calling*) is the mechanism that lets a model actually **do** things: look up real data, call an API, run a calculation, query a database.

### How it actually works (there's no real magic)

It's worth being very explicit about this because it's commonly misunderstood: **the LLM never runs your Python code itself.** What actually happens is a structured conversation:

1. You describe your function to the model (name, description, what parameters it needs) using a specific JSON schema.
2. You send the user's message *along with* that list of available "tools."
3. Instead of replying with normal text, the model can reply saying, in effect, *"I want you to call `get_ticket_price` with `destination_city="Paris"`."*
4. **Your code** — not the model — actually runs that Python function and gets a real result.
5. You send that result back to the model as a new message.
6. *Now* the model writes a normal, final answer to the user, using the real data you gave it.

This is why it's "no magic, just prompts under the hood" — the model was trained to recognize when a task needs external information and to output a specific, parseable request for it, instead of guessing. You're always the one executing real code, which is also why tool calling is safe by design: the model can only ever suggest calling functions *you* explicitly wrote and exposed to it.

### Building a tool step by step

**Step 1 — write the actual function**, exactly like any other Python function:

```python
ticket_prices = {"london": "$799", "paris": "$899", "tokyo": "$1400", "berlin": "$499"}

def get_ticket_price(destination_city):
    price = ticket_prices.get(destination_city.lower(), "Unknown ticket price")
    return f"The price of a ticket to {destination_city} is {price}"
```

**Step 2 — describe it to the model** using the required schema:

```python
price_function = {
    "name": "get_ticket_price",
    "description": "Get the price of a return ticket to the destination city.",
    "parameters": {
        "type": "object",
        "properties": {
            "destination_city": {
                "type": "string",
                "description": "The city that the customer wants to travel to",
            },
        },
        "required": ["destination_city"],
        "additionalProperties": False
    }
}
tools = [{"type": "function", "function": price_function}]
```

The `description` fields aren't just documentation for humans — the model reads them to decide *when* and *how* to use the tool, so write them clearly and precisely, the same way you'd write a clear docstring for a human teammate.

**Step 3 — pass `tools=tools` on every call**, and check whether the model asked to use one:

```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)

    if response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        response = handle_tool_call(message)
        messages.append(message)
        messages.append(response)
        response = openai.chat.completions.create(model=MODEL, messages=messages)  # ask again, now with real data

    return response.choices[0].message.content
```

**Step 4 — write the handler that actually executes the function** and reports the result back in the exact format the model expects:

```python
def handle_tool_call(message):
    tool_call = message.tool_calls[0]
    if tool_call.function.name == "get_ticket_price":
        arguments = json.loads(tool_call.function.arguments)  # arguments arrive as a JSON string — must be parsed
        city = arguments.get('destination_city')
        price_details = get_ticket_price(city)
        response = {
            "role": "tool",
            "content": price_details,
            "tool_call_id": tool_call.id  # links this result back to the specific tool call that requested it
        }
    return response
```

### Handling multiple tools calls robustly

A model can ask for more than one tool call in a single turn (e.g. "what's the price to Paris *and* Tokyo?"), so real code needs to loop over `message.tool_calls`, not just grab the first one:

```python
def handle_tool_calls(message):
    responses = []
    for tool_call in message.tool_calls:
        if tool_call.function.name == "get_ticket_price":
            arguments = json.loads(tool_call.function.arguments)
            city = arguments.get('destination_city')
            price_details = get_ticket_price(city)
            responses.append({"role": "tool", "content": price_details, "tool_call_id": tool_call.id})
    return responses
```

And a model might even need to call tools *repeatedly, one after another*, before it has enough information to answer (e.g. look up a price, then look up availability, then answer). The robust pattern is a `while` loop instead of a single `if`:

```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)

    while response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        responses = handle_tool_calls(message)
        messages.append(message)
        messages.extend(responses)
        response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)

    return response.choices[0].message.content
```

This `while` loop is the actual heart of every "agentic" system you'll build from here on: keep letting the model call tools until it decides it has enough information, *then* it produces a normal final answer.

### Persisting tool data with a real database

Hard-coded Python dictionaries are fine for a demo, but real tools usually read/write a real data store. Swapping the in-memory dictionary for SQLite takes only a few lines and is a much more realistic pattern:

```python
import sqlite3

DB = "prices.db"
with sqlite3.connect(DB) as conn:
    cursor = conn.cursor()
    cursor.execute('CREATE TABLE IF NOT EXISTS prices (city TEXT PRIMARY KEY, price REAL)')
    conn.commit()

def get_ticket_price(city):
    with sqlite3.connect(DB) as conn:
        cursor = conn.cursor()
        cursor.execute('SELECT price FROM prices WHERE city = ?', (city.lower(),))
        result = cursor.fetchone()
        return f"Ticket price to {city} is ${result[0]}" if result else "No price data available for this city"

def set_ticket_price(city, price):
    with sqlite3.connect(DB) as conn:
        cursor = conn.cursor()
        cursor.execute(
            'INSERT INTO prices (city, price) VALUES (?, ?) ON CONFLICT(city) DO UPDATE SET price = ?',
            (city.lower(), price, price)
        )
        conn.commit()
```

Notice the `?` placeholders instead of directly inserting the city name into the SQL string — this is **parameterized querying**, and it's non-negotiable in real code. Building SQL strings by directly pasting in user input opens you up to SQL injection attacks; parameterized queries let the database driver handle escaping safely. This matters even more once an *LLM* is the one supplying the arguments, since you can't fully control what a model might pass through.

You could extend this same pattern to add a `set_ticket_price` *tool* too (giving the assistant the ability to update prices, not just read them) — that's a great self-practice exercise, and it's exactly the kind of extension the original course suggests.

**Real-world relevance of tool calling:** this is the exact mechanism behind every "AI assistant that can actually do things" you've used — a customer support bot that looks up your real order status, a coding assistant that runs your actual tests, a scheduling assistant that checks your real calendar. The pattern (describe the function → let the model request it → you execute it → feed the result back) is identical no matter how sophisticated the surrounding product looks.

---

## 11. Agentic AI: When Tool Calling Becomes a Workflow

"Agentic AI" is one of those terms that gets used loosely, so it's worth pinning down precisely. An **agent**, in this context, is a system where an LLM doesn't just answer once — it operates in a loop, deciding for itself:

- What information it still needs
- Which tool (if any) to call to get that information
- Whether it has enough to give a final answer, or needs to take another action first

You've actually already built the core mechanic of an agent in Section 10 — that `while response.choices[0].finish_reason == "tool_calls":` loop **is** agentic behavior in miniature. The airline assistant deciding on its own to call `get_ticket_price` before answering a pricing question, without you hard-coding "if the user asks about price, call this function," is the model exercising autonomy over *when* to act.

Common real-world agentic use cases:

- **Customer support agents** that look up order status, process a refund, or escalate to a human — deciding for themselves which action fits the user's request.
- **Coding agents** that read a file, run tests, see the failure, and try a fix — looping until the tests pass.
- **Research agents** that search the web, read several pages, and synthesize an answer — deciding for themselves how many sources are "enough."
- **Multi-tool business assistants** — an onboarding assistant that can check HR policy documents, look up a new employee's schedule, *and* file a support ticket, choosing the right tool for each part of a conversation.

The multi-modal airline assistant we build later in this file (Section 12) is a good miniature example: it can decide, per message, whether it needs to look up a ticket price (a tool), whether it should generate a destination image (a separate action), and how to phrase its final spoken reply — all without you writing explicit "if/else" branching logic for every possible user request. That decision-making, delegated to the model itself, is what separates "agentic" from "just a chatbot with a fixed script."

---

## 12. What's Actually Happening Inside Gradio

Understanding roughly how Gradio works under the hood makes it much less "magic" and much easier to debug:

1. Gradio reads your Python description of the UI (which components, which callback functions) and constructs a matching frontend web app, built with a framework called Svelte.
2. Gradio starts a real backend web server (built on a framework called Starlette) on a free local port, which serves that Svelte frontend to your browser.
3. Gradio automatically creates backend API routes for each of your callback functions (like `chat()`), so that when you click "Submit" in the browser, it sends a request to the right route, which calls your actual Python function and sends the result back to update the UI.

That's genuinely the whole mechanism — there's no hidden AI involved in the UI itself; it's a fairly standard "frontend talks to backend over HTTP" web app, just auto-generated for you.

### The three flavors of Gradio UI

| Component | Best for |
|---|---|
| `gr.Interface` | Simple "one (or a few) input(s) → one output" tools — a form that produces a result |
| `gr.ChatInterface` | Standard chatbot UIs — handles the message bubbles and history automatically |
| `gr.Blocks` | Fully custom layouts — you control every component and how they're wired together |

You reach for `gr.Blocks` the moment your UI needs more than "a chat box in, a chat box out" — for example, a chat window *plus* an image panel *plus* an audio player, all updating together, which is exactly the multi-modal assistant we build next.

---

## 13. Multi-Modal Assistants: Combining Text, Images, and Speech

A "multi-modal" assistant works across more than one type of media — here, that means text conversation, AI-generated images, and AI-generated speech, all wired together in one app.

### Generating images with `gpt-image-1-mini`

```python
import base64
from io import BytesIO
from PIL import Image

def artist(city):
    image_response = openai.images.generate(
        model="gpt-image-1-mini",
        prompt=f"An image representing a vacation in {city}, showing tourist spots and everything unique about {city}, in a vibrant pop-art style",
        size="1024x1024",
        n=1,
    )
    image_base64 = image_response.data[0].b64_json
    image_data = base64.b64decode(image_base64)
    return Image.open(BytesIO(image_data))
```

Image generation is priced per image (a few cents each with `gpt-image-1-mini`), so during development call it sparingly and cache results where you can rather than regenerating on every test run.

### Generating speech (text-to-speech)

```python
def talker(message):
    response = openai.audio.speech.create(
        model="gpt-4o-mini-tts",
        voice="onyx",   # other options include alloy, coral
        input=message
    )
    return response.content  # raw audio bytes — Gradio's Audio component can play this directly
```

### Wiring text + tools + image + audio together with `gr.Blocks`

This is where `gr.Blocks` earns its keep — a chat callback that returns *three* different kinds of output (updated chat history, an audio clip, and possibly an image) at once:

```python
def chat(history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history
    response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)
    cities = []
    image = None

    while response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        responses, cities = handle_tool_calls_and_return_cities(message)
        messages.append(message)
        messages.extend(responses)
        response = openai.chat.completions.create(model=MODEL, messages=messages, tools=tools)

    reply = response.choices[0].message.content
    history += [{"role": "assistant", "content": reply}]
    voice = talker(reply)
    if cities:
        image = artist(cities[0])  # if a ticket-price tool was called, illustrate that destination

    return history, voice, image
```

And the actual `gr.Blocks` layout, wiring components and events together explicitly:

```python
def put_message_in_chatbot(message, history):
    return "", history + [{"role": "user", "content": message}]

with gr.Blocks() as ui:
    with gr.Row():
        chatbot = gr.Chatbot(height=500, type="messages")
        image_output = gr.Image(height=500, interactive=False)
    with gr.Row():
        audio_output = gr.Audio(autoplay=True)
    with gr.Row():
        message = gr.Textbox(label="Chat with our AI Assistant:")

    # this chains two events: first add the user's message to the chatbot, THEN call our chat() function
    message.submit(put_message_in_chatbot, inputs=[message, chatbot], outputs=[message, chatbot]).then(
        chat, inputs=chatbot, outputs=[chatbot, audio_output, image_output]
    )

ui.launch(inbrowser=True, auth=("ed", "bananas"))
```

The `.then(...)` chaining is a distinctly `gr.Blocks` feature: `gr.Interface` and `gr.ChatInterface` hide this kind of multi-step event wiring from you, but `Blocks` exposes it so you can build exactly the sequence of actions you need — here, "show the user's message immediately, *then* go generate the assistant's full multi-modal reply."

**Real-world relevance:** this exact shape — text chat + tool calling + generated media, all in one interface — is essentially the blueprint for products like AI travel-planning assistants, AI tutoring apps with generated diagrams, or internal company assistants that can both answer questions and produce visual/audio material on demand.

---

## 14. Bonus: Comparing Frontier Models on a Genuinely Hard Creative Task — SVG Generation

Generating an SVG image (a description made entirely of lines, shapes and coordinates, written as text/XML) from a text prompt is a very different challenge from generating a raster image — the model has to reason spatially and describe geometry precisely in code, with no visual feedback loop. It's inspired by Simon Willison's well-known "pelican riding a bicycle" test for evaluating how well a model can reason about shapes and composition purely through text.

```python
from openai import OpenAI
import os

openrouter = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=os.getenv("OPENROUTER_API_KEY"))

challenge = "a panda rollerblading to work"
prompt = f"Generate an SVG of {challenge}. Respond with the SVG only, no code blocks."
messages = [{"role": "user", "content": prompt}]

def artist(model, effort=None):
    response = openrouter.chat.completions.create(model=model, messages=messages, reasoning_effort=effort)
    return response.choices[0].message.content

results = [
    artist("openai/gpt-oss-120b"),
    artist("openai/gpt-5-nano", effort="low"),
    artist("deepseek/deepseek-v3.2"),
    artist("anthropic/claude-opus-4.5"),
    artist("google/gemini-3-pro-preview"),
]
```

Because OpenRouter gives you access to dozens of providers through one single API key and base URL, this kind of head-to-head model comparison — across completely different labs — takes only a handful of lines. This is genuinely how a lot of "which model should we use" evaluation work happens in practice before a team commits to a provider for a specific creative or spatial-reasoning task.

---

## 15. Cheat Sheet — The Patterns Worth Memorizing

- **Any OpenAI-compatible provider** = `OpenAI(api_key=..., base_url=...)`. Learn this once, reuse everywhere, including Ollama locally.
- **Conversation = a growing `messages` list** you resend in full every single call. The model has no memory of its own.
- **Streaming = a generator function using `yield`** instead of `return`, paired with Gradio's live-updating output components.
- **Tool calling = describe the function → model requests it → you execute the real code → you feed the result back → model gives the final answer.** Use a `while` loop so the model can chain multiple tool calls.
- **Agentic AI** is that same tool-calling loop, just with the model given enough autonomy to decide *when* and *which* tools to use, across a multi-step task.
- **`gr.Interface`** for single-shot tools, **`gr.ChatInterface`** for standard chatbots, **`gr.Blocks`** the moment you need a custom multi-component layout (chat + image + audio, etc.).
- **RAG** = retrieve the relevant facts first, then let the model generate an answer grounded in those facts — the fix for hallucination and out-of-date knowledge.

---

## 16. Solving the Week 2 End-of-Week Exercise

**The brief:** take the Week 1 technical question-answering tool and rebuild it with everything from Week 2 — a Gradio UI, streaming responses, a system prompt that gives the assistant real expertise, the ability to switch between models, and (for bonus points) a working tool call. Optionally, add audio input/output.

The solved version below is a **"Concept Tutor"** — a technical-explainer assistant aimed at someone learning programming/AI concepts (fitting the same spirit as Dheer's own learning journal work). It includes:

- A Gradio `Blocks` UI with a proper chat window
- Full response **streaming**
- A **system prompt** that gives the assistant a consistent "patient technical tutor" persona and expertise
- A **model switcher** — GPT, Claude, and a local Ollama model, all through the same OpenAI-compatible pattern from Section 1
- A working **tool call**: a `lookup_glossary_term` tool backed by a small local glossary, so the assistant can pull an exact, consistent definition instead of relying purely on its own memory whenever a user asks to define a known term
- Text-to-speech output as the optional "talk to it" bonus, using the same `talker()` pattern from Section 13

```python
# concept_tutor.py
# Week 2 end-of-week exercise — a streaming, multi-model, tool-using technical tutor

import os
import json
from dotenv import load_dotenv
from openai import OpenAI
import gradio as gr

load_dotenv(override=True)

openai_api_key = os.getenv("OPENAI_API_KEY")
anthropic_api_key = os.getenv("ANTHROPIC_API_KEY")

openai = OpenAI(api_key=openai_api_key)
anthropic = OpenAI(api_key=anthropic_api_key, base_url="https://api.anthropic.com/v1/")

# Ollama needs no real key — this lets the whole app run for free, fully offline, if you don't
# want to spend on cloud credits while testing
ollama = OpenAI(api_key="ollama", base_url="http://localhost:11434/v1")

MODEL_MAP = {
    "GPT": ("gpt-4.1-mini", openai),
    "Claude": ("claude-sonnet-4-5-20250929", anthropic),
    "Ollama (local)": ("llama3.2", ollama),
}

SYSTEM_MESSAGE = """
You are a patient, encouraging technical tutor who specializes in explaining programming,
machine learning, and computer science concepts to a beginner in a clear, step-by-step way.
Use plain language first, then introduce correct terminology. Use short analogies where they help.
Keep answers focused and well-structured, using markdown formatting (headings, bold, bullet points)
where it improves clarity. If a term from the glossary tool is relevant, use the lookup_glossary_term
tool to fetch the exact definition instead of guessing, so your answer stays consistent and accurate.
"""

# --- A tiny local "knowledge base" for the tool ---
GLOSSARY = {
    "overfitting": "When a model learns the training data too closely, including its noise and quirks, so it performs well on training data but poorly on new, unseen data.",
    "gradient descent": "An optimization algorithm that repeatedly nudges a model's parameters in the direction that most reduces error, based on the gradient (slope) of a loss function.",
    "tokenization": "The process of breaking text into smaller units (tokens) — words, sub-words, or characters — that a language model can process numerically.",
    "embedding": "A numeric vector representation of a piece of data (a word, sentence, or image) positioned so that semantically similar items end up close together in vector space.",
    "rag": "Retrieval-Augmented Generation: retrieving relevant facts from an external source and inserting them into a prompt so an LLM can generate an answer grounded in real, specific information.",
}

def lookup_glossary_term(term):
    definition = GLOSSARY.get(term.lower().strip())
    if definition:
        return f"{term}: {definition}"
    return f"No glossary entry found locally for '{term}'. Please explain it from general knowledge instead."

glossary_tool = {
    "name": "lookup_glossary_term",
    "description": "Look up the exact, approved definition of a technical term from the local glossary, if it exists there.",
    "parameters": {
        "type": "object",
        "properties": {
            "term": {
                "type": "string",
                "description": "The technical term to look up, e.g. 'overfitting' or 'gradient descent'",
            },
        },
        "required": ["term"],
        "additionalProperties": False,
    },
}
tools = [{"type": "function", "function": glossary_tool}]


def handle_tool_calls(message):
    responses = []
    for tool_call in message.tool_calls:
        if tool_call.function.name == "lookup_glossary_term":
            arguments = json.loads(tool_call.function.arguments)
            term = arguments.get("term")
            result = lookup_glossary_term(term)
            responses.append({"role": "tool", "content": result, "tool_call_id": tool_call.id})
    return responses


def talker(message, client=openai):
    """Optional bonus: text-to-speech for the assistant's reply."""
    try:
        response = client.audio.speech.create(model="gpt-4o-mini-tts", voice="alloy", input=message)
        return response.content
    except Exception:
        # Not every provider/model in the dropdown supports TTS (e.g. local Ollama doesn't) — fail quietly
        return None


def stream_chat(message, history, model_choice):
    model_name, client = MODEL_MAP[model_choice]

    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": SYSTEM_MESSAGE}] + history + [{"role": "user", "content": message}]

    # Ollama's local models often don't support tool calling reliably, so we only pass tools
    # to providers that handle it well (GPT and Claude)
    use_tools = model_choice in ("GPT", "Claude")

    response = client.chat.completions.create(
        model=model_name,
        messages=messages,
        tools=tools if use_tools else None,
    )

    # Resolve any tool calls before we start streaming the final answer
    while use_tools and response.choices[0].finish_reason == "tool_calls":
        tool_message = response.choices[0].message
        responses = handle_tool_calls(tool_message)
        messages.append(tool_message)
        messages.extend(responses)
        response = client.chat.completions.create(model=model_name, messages=messages, tools=tools)

    # Now stream the final answer
    stream = client.chat.completions.create(model=model_name, messages=messages, stream=True)
    result = ""
    for chunk in stream:
        result += chunk.choices[0].delta.content or ""
        yield result


with gr.Blocks(title="Concept Tutor") as ui:
    gr.Markdown("# 🎓 Concept Tutor — Week 2 Exercise Solution")
    gr.Markdown(
        "Ask about any programming, ML, or CS concept. Switch models freely — GPT and Claude call "
        "the cloud; Ollama runs fully local and free. Ask about a glossary term (e.g. *'what is overfitting?'*) "
        "to see the tool call in action."
    )
    model_choice = gr.Dropdown(list(MODEL_MAP.keys()), value="GPT", label="Model")
    chatbot = gr.ChatInterface(
        fn=stream_chat,
        additional_inputs=[model_choice],
        type="messages",
    )

if __name__ == "__main__":
    ui.launch(inbrowser=True)
```

**How this satisfies the brief, piece by piece:**

- **Gradio UI:** built with `gr.Blocks` wrapping a `gr.ChatInterface`, so it gets the polished chat UI plus a custom title/description and model dropdown above it.
- **Streaming:** `stream_chat` is a generator (`yield`), identical to the pattern from Section 8/9.
- **System prompt for expertise:** `SYSTEM_MESSAGE` gives the assistant a specific persona and teaching style, exactly like Section 9's clothing-store example.
- **Model switching:** the `model_choice` dropdown, resolved through `MODEL_MAP`, reuses the exact "same code, different `base_url`" trick from Section 1 — including a fully local, free Ollama option.
- **Tool calling (bonus):** `lookup_glossary_term` follows the full pattern from Section 10 — schema, `handle_tool_calls`, and a `while` loop before the final streamed answer.
- **Audio bonus:** `talker()` reuses the text-to-speech pattern from Section 13; you can wire its output into a `gr.Audio` component in the `Blocks` layout the same way the airline assistant did, if you want the spoken-reply bonus fully live in the UI.

**To run it:** save as `concept_tutor.py` in the same folder as your `.env` file, make sure `ollama serve` is running locally if you want the local model option, then run `python concept_tutor.py`.

**Where to take it further (self-practice):** add a second tool that can search your own GitHub learning journal repo for a matching note and summarize it, so the tutor can say "you already wrote notes on this — here's a refresher" — a small step toward a genuinely personalized RAG-based study companion.
