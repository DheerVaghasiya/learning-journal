# LLM Engineering Fundamentals — Transformers, Tokens, and the Illusion of Memory

*Day 4 of my learning journal — Ed Donner's LLM Engineering course ([Week 1, Day 4](https://github.com/ed-donner/llm_engineering/blob/main/week1/day4.ipynb))*

This is the note where things stopped being "call an API and get text back" and started being "okay but what is actually happening inside that black box." Writing this one carefully because everything else in LLM engineering — prompting, RAG, agents, fine-tuning — sits on top of these fundamentals.

---

## 1. From LSTMs to Transformers: why we needed a new architecture

Before transformers, the dominant way to process sequences (text, speech, time series) was **RNNs and LSTMs** (Long Short-Term Memory networks). The core idea: read a sequence **one token at a time, left to right**, carrying a "hidden state" forward like a running memory as you go.

**The analogy that helped me:** an LSTM reads a book one word at a time with a notepad, constantly erasing and rewriting a short summary as it goes, because it only has room for so much on that notepad. By the time it reaches word 500, it's relying on a heavily compressed summary of everything from word 1 to word 499 — a lot of detail has already been lost or blurred.

Two big problems fall out of this:
1. **Sequential bottleneck** — you can't process word 500 until you've processed words 1 through 499, one at a time. This makes LSTMs slow to train on long sequences and hard to parallelize on GPUs.
2. **Long-range forgetting** — even with LSTM's memory gates specifically designed to fight this, information from early in a long sequence tends to fade by the time you reach the end.

**Transformers (introduced in the 2017 paper "Attention Is All You Need") fixed both problems with one idea: instead of reading sequentially and carrying a compressed memory forward, look at the *entire* sequence at once, and let every word directly "attend to" every other word, no matter how far apart they are.**

That one architectural shift is the entire reason GPT, LLaMA, Claude, DeepSeek, and basically every modern LLM exist. It's genuinely the pivot point of the whole field.

---

## 2. Attention — the mechanism behind the architecture

**Attention answers one question for every single word: "which other words in this sequence matter most for understanding *me*, right now?"**

Take the sentence: *"The trophy didn't fit in the suitcase because it was too small."*

What does "it" refer to — the trophy or the suitcase? A human resolves this instantly using context. Attention is the mechanism that lets the model do the same thing mathematically: when processing the word "it," the model computes a relevance score against every other word in the sentence, and learns to weight "trophy" or "suitcase" more heavily depending on context (here, "small" tips it toward the suitcase — trophies don't shrink to fit).

**Analogy that stuck with me:** attention is like being in a group conversation where, before you respond to something someone said, you briefly glance around the room to see who else is relevant to what's being discussed, then weigh their input accordingly before forming your answer. Every word in a transformer does this "glance around the room" for every other word, simultaneously, not one at a time.

Mechanically, this happens through **Query, Key, and Value vectors**:
- **Query** — "what am I looking for?" (from the current word's perspective)
- **Key** — "what do I have to offer?" (from every other word's perspective)
- **Value** — "here's my actual content, if you decide I'm relevant"

Every word's Query gets compared against every other word's Key to produce a relevance score, and those scores decide how much of each word's Value gets blended into the current word's understanding of itself. This is called **self-attention** — "self" because the sequence is attending to itself, not to some separate external sequence.

**Multi-head attention** just means doing this whole process several times in parallel, with different learned "lenses" — one head might specialize in tracking grammatical relationships (subject/verb), another might track long-range topic references, another might track something more abstract there's no clean English word for. Stack enough of these attention layers, and the model builds up an increasingly rich understanding of how every word relates to every other word in the input.

**Why this fixed both LSTM problems:**
- It's parallelizable — every word's attention computation can happen simultaneously on a GPU, instead of waiting in a sequential chain.
- There's no "long-range forgetting" — word 1 and word 10,000 can attend to each other directly, with no compressed memory bottleneck in between.

---

## 3. Emergent intelligence and agentic AI

Here's the part that still feels slightly wild to me: nobody explicitly programmed GPT to reason, write code, or plan multi-step tasks. Those abilities **emerged** simply from training a large enough transformer on enough text to get good at one deceptively simple task: **predict the next token.**

**Emergent intelligence** refers to capabilities that appear at scale, that weren't present (or were much weaker) in smaller versions of the same architecture. Things like in-context learning, chain-of-thought reasoning, and code generation weren't specifically engineered features — they showed up as a side effect of scaling up model size, data, and compute far enough. Nobody fully predicted in advance exactly which capabilities would emerge at which scale, which is part of why this is such an active area of research.

**Agentic AI** builds on top of this emergent capability. Instead of using an LLM for a single one-shot prompt → response, you give it:
- a **goal** instead of a fixed instruction
- **tools** it can call (search the web, run code, query a database, hit an API)
- a **loop** where it can observe results, decide the next step, and keep going until the goal is met

**Analogy:** a plain LLM call is like asking a very knowledgeable person a single question over a phone call — they answer from what's already in their head, then it's over. An agent is like hiring that same person as an actual employee — they can go look things up, use tools, come back with results, and keep working across multiple steps until the job is actually done, not just answered once.

This distinction matters a lot for what I'm learning next in this course — RAG and agentic workflows are essentially "give the base transformer's next-token prediction a scaffold of tools and a loop," not a fundamentally different model underneath.

---

## 4. Parameters: from millions to trillions

A **parameter** is one learned number inside the model — a weight in the attention mechanism, a weight in a feed-forward layer, etc. Training is the process of adjusting billions (or trillions) of these numbers so the model gets better at predicting the next token across a massive training dataset.

Rough scale, to actually anchor the "millions to trillions" jump in my head:

| Model era | Approx. parameters | Note |
|---|---|---|
| Early LSTMs / small NLP models | millions | could run on a laptop |
| GPT-2 (2019) | ~1.5 billion | first "whoa, this can actually write" moment for a lot of people |
| GPT-3 (2020) | ~175 billion | the jump that kicked off the current LLM era |
| GPT-4, LLaMA 3, DeepSeek-V3 era | hundreds of billions to low trillions (estimates vary, exact numbers for closed models aren't published) | modern frontier scale |

**Why more parameters generally means more capability:** each parameter is a tiny piece of learned pattern-matching capacity. More parameters means the model can represent more nuanced, more numerous patterns from its training data — up to a point, and only if there's also enough training data and compute to actually make use of that extra capacity (this relationship is studied under what's called "scaling laws").

**LLaMA (Meta)** and **DeepSeek** are both worth knowing by name specifically because they're **open-weight** — meaning the actual trained parameter files are published for anyone to download and run themselves, unlike GPT-4/Claude where you only get API access, not the weights. This matters a lot practically: open-weight models are what let me realistically run something locally or fine-tune it myself, while closed models I can only ever call through an API.

DeepSeek in particular got attention in the field for reaching competitive performance while reportedly being trained more efficiently/cheaply than some Western frontier labs claimed for comparable models — a reminder that "more parameters" isn't the only lever; training technique and data quality matter enormously too.

---

## 5. What are tokens? From characters to GPT's tokenizer

Before any of the attention/transformer machinery I described above can run, raw text has to be converted into numbers. That conversion step is **tokenization**.

**The naive approach would be:** split text into individual characters, or individual whitespace-separated words. Both have real problems:
- **Character-level** → sequences become enormous (this sentence would be ~60+ tokens instead of ~12), and the model has to work much harder to learn that "c-a-t" means something as a unit.
- **Word-level** → the vocabulary explodes (every tense of every verb, every plural, every typo becomes its own entry), and any word not seen during training becomes a complete unknown, with no way to represent it at all.

**The actual solution GPT and most modern LLMs use: subword tokenization** (specifically a method called BPE — Byte Pair Encoding, or variants of it). The idea: break text into chunks that are bigger than a single character but often smaller than a full word — common words might be a single token, rarer words get split into a few meaningful pieces, and truly novel strings can always be built from individual bytes/characters as a fallback, so nothing is ever "unrepresentable."

**Analogy that helped me:** think of tokens as LEGO bricks instead of individual grains of sand (characters) or entire pre-built houses (whole words). Bricks are a reusable, manageable unit — common shapes get their own single brick, unusual shapes get built from a few bricks combined, and you're never stuck unable to build something just because it's unusual.

---

## 6. Understanding tokenization — how GPT actually breaks down text

A few concrete rules of thumb I picked up (and then verified myself in code, see Section 7 below):
- A token is **not** the same as a word or a character. Roughly: **1 token ≈ 4 characters of English text ≈ ¾ of a word**, on average — though this varies a lot by language and content type.
- Common short words ("the," "is," "and") are usually a single token each.
- Longer or rarer words often get split into 2+ tokens — e.g. "banoffee" might become something like `ban` + `off` + `ee` rather than one clean token, because it's not common enough in training data to earn its own dedicated token.
- Numbers, punctuation, and even individual spaces can be their own tokens, or bundled with adjacent characters, depending on the specific tokenizer.
- Every model family has its own tokenizer with its own fixed vocabulary (a set list of every possible token, usually tens of thousands of entries) — this is why the exact same sentence can have a different token count depending on which model you send it to.

---

## 7. Tokenizing with code — `tiktoken`

`tiktoken` is OpenAI's official tokenizer library — it lets me see, in code, exactly how a given model would break a sentence into tokens, before ever sending it to the API.

```python
import tiktoken

encoding = tiktoken.encoding_for_model("gpt-4.1-mini")

tokens = encoding.encode("Hi my name is Ed and I like banoffee pie")
print(tokens)
```

This returns a list of integers — each integer is that model's internal ID for a specific token, not the text itself. Numbers, not words, are what actually get fed into the transformer.

To see what each ID actually represents, decode them one at a time:

```python
for token_id in tokens:
    token_text = encoding.decode([token_id])
    print(f"{token_id} = {token_text}")
```

This is the exact test I ran for myself, and it's genuinely the best way to build intuition — running this on "banoffee pie" specifically shows the split-into-pieces behavior for a less common word in real time, instead of just reading about it. I can also decode a single token ID on its own to sanity-check what it maps to:

```python
encoding.decode([326])
```

**Practical habit I'm keeping from this:** any time I'm not sure how many tokens something will cost (for API pricing or for fitting inside a context window), I run it through `tiktoken` first instead of guessing. It takes one line and removes all the guesswork.

---

## 8. The illusion of memory — LLMs are stateless

This is the single biggest "wait, what?" moment from this whole day of the course, and I want to preserve exactly how it played out because it's such a clean demonstration.

**Step 1 — tell the model your name:**

```python
from openai import OpenAI
openai = OpenAI()

messages = [
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "Hi! I'm Ed!"}
]

response = openai.chat.completions.create(model="gpt-4.1-mini", messages=messages)
print(response.choices[0].message.content)
# → "Hi Ed! How can I assist you today?"
```

**Step 2 — ask it your name, in a brand-new call:**

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "What's my name?"}
]

response = openai.chat.completions.create(model="gpt-4.1-mini", messages=messages)
print(response.choices[0].message.content)
# → it has no idea. It was never told, in THIS call.
```

The model has no memory of the first call. It genuinely doesn't know. And the reason is the core fact I need to always remember about LLMs:

> **Every single call to an LLM is completely stateless.** There is no persistent memory between API calls. Each call starts from zero, with nothing but exactly what's inside that one request's messages.

**Step 3 — the actual trick that makes ChatGPT/Claude "feel" like they remember you:**

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "Hi! I'm Ed!"},
    {"role": "assistant", "content": "Hi Ed! How can I assist you today?"},
    {"role": "user", "content": "What's my name?"}
]

response = openai.chat.completions.create(model="gpt-4.1-mini", messages=messages)
print(response.choices[0].message.content)
# → "Your name is Ed!"
```

Now it works — not because the model "remembered" anything between calls, but because **the entire conversation history was re-sent, in full, as part of this one single call.** The model isn't recalling a past interaction; it's predicting the most statistically likely next tokens given a prompt that happens to contain "Hi! I'm Ed!" earlier in the same input.

**The recap I keep coming back to (this is the part I want to never forget):**
1. Every LLM API call is stateless — nothing persists on the model's side between calls.
2. Apps like ChatGPT create the *illusion* of memory by resending the entire conversation so far, every single time, as part of the current prompt.
3. There's no actual "remembering" happening — it's next-token prediction over a prompt that just happens to include the earlier conversation as context.
4. This means: yes, you (or the API caller) pay for the entire conversation history being reprocessed, on every single message, for the whole life of that conversation.

That last point isn't a flaw — it's the intended trade-off. We *want* the model to reprocess the full conversation every time, because that reprocessing is exactly what lets it "look back" and stay coherent. The cost is the price of that coherence, not a bug to work around.

**This single concept quietly explains a huge amount of what comes later in this course** — why context windows matter so much, why RAG exists (to avoid resending an entire knowledge base every time), and why conversation length directly drives API cost. It's one idea with a lot of downstream consequences.

---

## 9. Context windows

The **context window** is the maximum number of tokens a model can consider in a single call — input plus output combined, all counted together against one limit.

**Analogy:** think of it as the model's short-term working memory span, or a whiteboard of fixed size. Anything that doesn't fit on the whiteboard simply isn't there for this call — not forgotten in some deep sense, it just was never written down to begin with.

Because of the stateless-memory point from Section 8, the context window is really the hard ceiling on "how much conversation history + instructions + retrieved documents + expected output can I cram into one call." Once a conversation grows past that limit, something has to be dropped, summarized, or trimmed — this is exactly the kind of engineering problem that motivates RAG (retrieve only the relevant chunks instead of sending everything) rather than just "paste the whole knowledge base into every prompt."

Context window sizes vary a lot by model and have grown massively over the last couple of years (early GPT-3 era models had windows in the low thousands of tokens; many current frontier models offer context windows in the hundreds of thousands, and some go even further) — but the fundamental limitation is exactly the same shape regardless of how big the number is: there's always some ceiling, and every token inside it costs money and processing time.

---

## 10. API costs and token limits

Since every API call is priced by tokens, not by "requests" or "characters," the token-counting mental model from Sections 5–7 directly turns into a cost model:

- **Input tokens** — everything sent to the model: system prompt, conversation history, any retrieved documents, the actual question.
- **Output tokens** — everything the model generates back.
- Providers typically price these **separately**, and output tokens are usually priced higher than input tokens per token, since generating each one requires a full forward pass through the model, while input tokens can be processed more efficiently as a batch.

**Direct consequence of the "illusion of memory" trick (Section 8):** a long-running conversation doesn't just cost more for the latest message — every single call resends the *entire* conversation so far as input tokens. A 50-message conversation's 51st message pays for reprocessing all 50 previous messages as input, every time. Cost grows with conversation length, not just with message count.

Practical habits I'm taking from this for whenever I build something with an LLM API:
1. Count tokens with `tiktoken` (or the equivalent for other providers) **before** sending, especially for anything with variable-length input like user-uploaded documents.
2. Watch conversation length in any chat-style app — either summarize/trim old messages, or lean on RAG-style retrieval instead of dumping full history/documents into every call.
3. Remember output tokens usually cost more per token than input tokens — a model that "thinks out loud" with a long generated response costs more than one prompted to answer concisely, all else equal.
4. Context window limits and cost limits are two different constraints that both point at the same root cause — every token, in or out, has to fit inside a budget (of both size and money) that resets fresh on every single stateless call.

---

## 11. My personal cheat sheet

| Concept | One-line takeaway |
|---|---|
| LSTMs → Transformers | sequential + forgetful → parallel + full-sequence attention |
| Attention | every word directly weighs relevance against every other word, via Query/Key/Value |
| Emergent intelligence | capabilities like reasoning weren't hand-coded — they showed up from scale |
| Agentic AI | give the model a goal + tools + a loop, instead of one single prompt/response |
| Parameters | learned numbers inside the model; millions (early NLP) → hundreds of billions/trillions (frontier LLMs) |
| LLaMA / DeepSeek | notable open-weight models — you can download and run the actual weights, unlike closed API-only models |
| Tokens | subword chunks of text — bigger than a character, often smaller than a word |
| Tokenization | how GPT breaks text into those chunks, via a fixed per-model vocabulary |
| `tiktoken` | run text through it to see the exact token breakdown/count before sending an API call |
| Illusion of memory | every LLM call is stateless — "memory" is just resending the full conversation every time |
| Context window | the max tokens (input + output combined) one single call can hold |
| API costs | priced per input/output token; long conversations cost more because full history gets resent every call |

Rules going forward, for whenever I'm actually building with these APIs:
1. Never assume the model "remembers" anything across calls — if it needs to know something, it has to be in *this* prompt.
2. Check token counts with `tiktoken` before sending anything with unpredictable length.
3. Treat context window size as a hard budget, not a suggestion — plan for trimming/retrieval before hitting it, not after.
4. Remember that in a growing conversation, cost scales with the *whole history*, not just the newest message — this is the direct, practical cost of the "illusion of memory" trick.

Next up: this note is basically the foundation for whatever comes next in the course — I'm guessing RAG is next, and now I actually understand *why* it exists (avoiding resending huge amounts of context on every call) instead of just knowing it as a buzzword.
