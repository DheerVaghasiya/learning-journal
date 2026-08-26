# Inside the Mind of an LLM: Base, Chat & Reasoning Models Explained

Self notes — going beyond just calling APIs, to actually understanding what's different between the models I'm calling and how people are using them in the wild.

Up until now I've mostly treated "the model" as a black box — send a prompt, get text back. This round of learning was about opening that box a little: what kind of model am I actually talking to, and what are the frontier labs racing toward right now.

## Base vs Chat vs Reasoning models — they're not the same thing

This was the biggest "oh" moment. Not all LLMs are built or trained the same way:

- **Base models** — trained purely to predict the next word from huge amounts of internet text. No instruction-following, no "assistant" personality baked in. If you prompt a base model with a question, it might just continue the sentence rather than answer it, because that's literally all it was trained to do.
- **Chat models** — a base model that's gone through extra training (instruction tuning + RLHF) specifically to follow instructions and hold a conversation. This is what GPT-4, Claude, and Gemini actually are when you talk to them normally — they've been shaped to behave like a helpful assistant.
- **Reasoning models** — a newer category trained to "think" through a problem step-by-step before answering, often showing (or hiding, depending on the model) a chain of reasoning first. Slower and more expensive per answer, but noticeably better at math, logic, and multi-step problems where a regular chat model tends to jump to a plausible-sounding but wrong answer.

The practical takeaway for me: picking a model isn't just "which company" — it's "what was this specific model actually optimized to do," and using a reasoning model for a simple chat task (or a fast chat model for a hard logic puzzle) is a mismatch either way.

## The frontier models — strengths and pitfalls

Went and actually tested these side by side instead of just reading about them:

**GPT (OpenAI)**
- Strengths: strong general all-rounder, huge ecosystem, good at following detailed instructions, solid coding ability.
- Pitfalls: can be confidently wrong (hallucinate) without flagging uncertainty; personality can feel a bit generic compared to Claude.

**Claude (Anthropic)**
- Strengths: careful, well-structured reasoning, strong at writing and coding, tends to be more upfront about uncertainty.
- Pitfalls: more conservative — sometimes declines or hedges on things a person would consider reasonable.

**Gemini (Google)**
- Strengths: strong multimodal handling (text, images, sometimes video/audio in one context), tightly integrated with Google's own tools and search.
- Pitfalls: response quality/tone felt less consistent across tasks compared to GPT/Claude in my own testing.

**Grok (xAI)**
- Strengths: more real-time awareness through X/Twitter integration, casual/less filtered tone.
- Pitfalls: less consistent on careful, structured tasks compared to the more "enterprise-trained" models.

**DeepSeek**
- Strengths: genuinely strong reasoning performance for the cost, open-weight options available.
- Pitfalls: less polished UI/product experience, data handling concerns some people raise given where it's hosted.

## Testing frontier models through their web UIs

Instead of only calling these through code, I spent time just using the actual chat products — ChatGPT, Claude.ai, Gemini, Grok, DeepSeek — side by side on the same prompts. Things I noticed testing this way that you don't see through the API alone:
- Web UIs often have extra features bolted on (web search, memory, canvas/artifact-style outputs) that change the actual experience a lot more than the raw model would suggest.
- Response style differs more in casual conversation than in structured tasks — this only shows up when you're chatting naturally, not just sending one-off API prompts.
- Rate limits and "free tier" restrictions matter a lot for casual comparison testing — some models throttle you fast, others don't.

## Deep Research mode

Several of these platforms now have a "Deep Research" feature — instead of one quick answer, the model spends several minutes autonomously searching, reading multiple sources, and compiling a structured report. Tested this across ChatGPT's deep research, and compared how Claude, Gemini, Grok, and DeepSeek handle similar multi-step research prompts.

What stood out: this is a genuinely different mode of "using an LLM" — you're not chatting, you're delegating a whole research task and getting a report back. Quality depended heavily on how specific the initial prompt was; vague prompts got vague (if long) reports.

## Agentic AI in action

The next level past "answering questions" is **agentic AI** — a model that doesn't just respond, but takes actions, uses tools, and works through multi-step tasks with some autonomy (browsing, running code, calling APIs, checking its own output before continuing).

Two concrete examples I looked at:
- **Claude Code / Agent mode** — an agent that can read a codebase, plan changes, write code, run it, see the errors, and fix them — a full loop instead of a single generation.
- General agent mode in chat products — where the model plans a multi-step task on its own (e.g. "book me the cheapest flight" type workflows) instead of me manually chaining each step myself.

The shift here: with a regular chat model, *I* am the loop — prompt, read, decide next prompt. With an agent, the model runs its own loop and only comes back to me when it's done or stuck.

## My takeaway from this round of learning

Calling an API teaches you the mechanics. Actually testing these models against each other on the same prompts is what showed me they're not interchangeable in practice — each one has a personality and a lane it's genuinely better in, and picking the right one for a task is its own skill separate from just knowing how to write a prompt.

---

## Homework — Frontier Model Showdown: a mini LLM competition game

The idea: give the same prompt to multiple models, then have another LLM act as judge and score the responses. Built this using Ollama locally so it costs nothing to run and experiment with.

```python
import gradio as gr
from openai import OpenAI

# Ollama's OpenAI-compatible local endpoint - no API key or cost involved
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

# Using differently-sized/tuned local models to stand in for "competitors"
CONTESTANTS = {
    "Model A (llama3.2)": "llama3.2",
    "Model B (mistral)": "gemma3:270m",
}

JUDGE_MODEL = "llama3.2"


def get_response(model_name, prompt):
    result = client.chat.completions.create(
        model=model_name,
        messages=[{"role": "user", "content": prompt}]
    )
    return result.choices[0].message.content


def run_competition(prompt):
    responses = {}
    for label, model_name in CONTESTANTS.items():
        responses[label] = get_response(model_name, prompt)

    # Build a judging prompt that shows the judge both answers, unlabeled by contestant name
    judge_prompt = f"""You are judging two AI responses to the same question.
Question: {prompt}

Response 1: {responses['Model A (llama3.2)']}

Response 2: {responses['Model B (gemma3:270m)']}

Decide which response is clearer, more accurate, and more helpful.
Reply with just "Response 1" or "Response 2" and one sentence explaining why."""

    verdict = get_response(JUDGE_MODEL, judge_prompt)

    output = ""
    for label, text in responses.items():
        output += f"### {label}\n{text}\n\n"
    output += f"### 🏆 Judge's Verdict\n{verdict}"

    return output


view = gr.Interface(
    fn=run_competition,
    inputs=gr.Textbox(label="Enter a prompt for the models to compete on", lines=3),
    outputs=gr.Markdown(label="Results"),
    title="Frontier Model Showdown",
    description="Same prompt, two local models, one AI judge — who wins?"
)

view.launch()
```

Before running: pull both models once (`ollama pull llama3.2` and `ollama pull mistral`).

Things I learned building this:
- Using a model as a **judge** is basically the same API call pattern as everything else — the "judging" is just a cleverly written prompt, not a special feature.
- Keeping contestant labels generic ("Response 1", "Response 2") in the judge prompt matters — otherwise the judge model can be biased by a model's name/reputation instead of judging the actual text.
- This is a small, contained version of what "LLM-as-a-judge" evaluation looks like at a much bigger scale in real ML pipelines.

Next step for myself: swap in real frontier models (GPT, Claude, Gemini) as contestants once I've got API keys, and see if the ranking changes from the local-only version.
