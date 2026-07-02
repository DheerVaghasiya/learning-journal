# Week 1 – Day 2 Notes: Frontier Models & Building UIs with Gradio

Course: [ed-donner/llm_engineering](https://github.com/ed-donner/llm_engineering) (week1/day2.ipynb + guides)

Yesterday I got a raw script talking to OpenAI in a notebook cell. Today was about two things: actually understanding the landscape of frontier models I'm choosing between, and giving my scripts a real interface instead of just printing stuff in a cell.

## The frontier model landscape

Before writing more code, it helped to actually understand who the players are:

- **OpenAI (GPT)** – the one most people have used through ChatGPT. Strong all-rounder, good API docs, cheap "mini"/"nano" tiers great for learning without burning money.
- **Anthropic (Claude)** – known for careful, well-reasoned responses and being a favourite for coding tasks. Has a "system prompt" concept just like GPT.
- **Google (Gemini)** – strong multimodal support, generous free tier through AI Studio.
- **Ollama** – not a frontier API, but the free alternative: runs open-source models locally on your own machine, no API key or cost at all.

The big practical takeaway: these are mostly interchangeable at the API level. Once you've called one, calling another is just swapping the client and the model name. That's the whole point of "frontier model" thinking — pick the cheapest one that's good enough for the task, don't marry yourself to one vendor.

For this course specifically: stick to `gpt-4.1-nano` for OpenAI and `claude-3-haiku-20240307` for Anthropic — cheap enough that experimenting costs basically nothing.

## System prompt vs user prompt (properly this time)

Day 1 touched this, but Day 2 made it click:
- **System prompt** = the personality/instructions you set once, e.g. "You are a snarky assistant that replies in one sentence."
- **User prompt** = the actual message/question, changes every call.

Structuring it this way means I can reuse the same "personality" across many different user questions without repeating myself.

```python
system_message = "You are a helpful assistant that explains things simply."
user_message = "Explain what an API is."

messages = [
    {"role": "system", "content": system_message},
    {"role": "user", "content": user_message}
]
```

## Streaming responses

Instead of waiting for the whole answer and dumping it at once, both OpenAI and Claude support **streaming** — the response comes back chunk by chunk, like watching it being typed live. Makes any UI feel way more responsive.

```python
stream = openai.chat.completions.create(
    model="gpt-4.1-nano",
    messages=messages,
    stream=True
)

for chunk in stream:
    content = chunk.choices[0].delta.content or ""
    print(content, end="", flush=True)
```

Claude's streaming works similarly but through its own client and event structure — the pattern (loop over chunks, append text) is the same idea everywhere.

## Gradio: giving scripts a face

This was the highlight of the day. **Gradio** turns a plain Python function into a shareable web UI in about 3 lines:

```python
import gradio as gr

def shout(text):
    return text.upper()

gr.Interface(fn=shout, inputs="text", outputs="text").launch()
```

That `.launch()` spins up a local web app instantly — no HTML/CSS/JS needed. For quick AI prototypes this is huge; I don't need to know frontend to give something a usable interface.

Key pieces I now understand:
- `gr.Interface(fn=..., inputs=..., outputs=...)` — the basic building block, one function in, one result out.
- `gr.ChatInterface(fn=...)` — built specifically for chatbots, handles conversation history for you automatically instead of me managing a messages list manually.
- `share=True` in `.launch()` gives a temporary public URL — good for showing something to a friend without deploying anywhere.

## Building a multi-model chat interface

The natural next step once you have streaming + Gradio: let the *user* pick which model answers. This is where it stopped feeling like "a script" and started feeling like "a product" — same prompt, different brains, and I can compare which one I trust more for a given kind of question.

The pattern is basically: one Gradio input, a dropdown/radio to choose the model, and an if/else (or dict of functions) that routes to the right API.

## My takeaway from today

Day 1 was "can I talk to an LLM from code." Day 2 is "can I let someone *else* talk to an LLM without touching code." That's the actual shift from script-writer to AI engineer — wrapping a capability in something usable.

---

## Homework

The exercise was to build something that streams a response, wrapped in a Gradio interface. I don't have paid API keys set up yet, so I built the Ollama version instead — same pattern, but running a model locally on my own machine, completely free:

```python
import gradio as gr
from openai import OpenAI

# Ollama exposes an OpenAI-compatible endpoint on your own machine.
# No API key needed - the value below is just a placeholder the client library requires.
ollama_client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

MODEL = "llama3.2"
SYSTEM_MESSAGE = "You are a helpful, concise assistant. Keep answers clear and to the point."


def stream_ollama(prompt):
    messages = [
        {"role": "system", "content": SYSTEM_MESSAGE},
        {"role": "user", "content": prompt}
    ]
    stream = ollama_client.chat.completions.create(
        model=MODEL,
        messages=messages,
        stream=True
    )
    result = ""
    for chunk in stream:
        result += chunk.choices[0].delta.content or ""
        yield result


view = gr.Interface(
    fn=stream_ollama,
    inputs=gr.Textbox(label="Your prompt", lines=4),
    outputs=gr.Markdown(label="Response"),
    title="Local LLM Chat — Powered by Ollama",
    description="Type a prompt and watch a locally-running model stream its response back."
)

view.launch()
```

Before running this, Ollama needs to be installed and the model pulled once:
```bash
ollama pull llama3.2
```
Then just run the Python file — Ollama runs quietly in the background and the client talks to it on `localhost`.

Things I learned building this:
- Ollama deliberately mimics the OpenAI client's shape (`base_url` + `api_key`), so any code written for OpenAI's API can point at Ollama with almost no changes — just a different `base_url` and a placeholder key.
- Using `yield` instead of `return` inside the function is what makes Gradio treat it as a streaming output — every `yield` updates the UI live instead of waiting for the function to finish.
- `gr.Markdown()` as the output looks a lot nicer than a plain textbox once the model starts returning formatted text.
- Because there's no API key or billing involved, this is the easiest way to get a working demo screenshot without setting anything else up first.
