# Chapter 1 — AI in One Page for DevOps People

---

**Cost:** ₹0 (one read-only CLI command) · **Time:** 25–30 minutes · **Cleanup:** none — nothing billable is created in this chapter.

**The purpose of this chapter, in one sentence:** answer every question that's actually stopping you from starting — "do I need to code," "is my job at risk," "what does RAG even mean" — before Chapter 2 asks you to call a real model, so nothing in this book ever feels like jargon you were supposed to already know.

**If you've been putting off learning AI because you don't know where to start, this is that starting point.** Not a theory course, not a math course — a plain-English map of the territory, written for someone who already knows how to run a production system and just needs to see where AI actually fits into that world.

**By the end of this chapter you will have:**

- A straight answer to the questions that stop most DevOps engineers before they ever open a terminal — do I need to learn Python, do I need a GPU, is this just ChatGPT, is my job at risk
- A working map of every AI term you'll hear this week — tokens, LLM, context window, embeddings, RAG, agents, MCP, and more — in the order they'd actually come up, each explained in plain language
- A clear line between what this book teaches you hands-on and what it only names, so you're never lost when you hear a term later that this book didn't cover
- Confirmed, from the CLI, that Amazon Bedrock and the Nova Micro model are reachable in your account

> 🎥 **Video walkthrough:** the full chapter, step by step.
> `🎥 [VIDEO ch01 | QR code + short link | 12–15 min | public YouTube]`
> Everything is also written out below, so the video is optional.

---

## Why this chapter exists

Every other chapter in this book hands you commands to paste and output to compare. This one is different on purpose. Before you run a single AWS command, you're going to hit a wall of unfamiliar words — tokens, context windows, RAG, agents, MCP — and a handful of real worries that have nothing to do with AWS at all: *do I need to be a programmer for this, is this about to replace my job, how would I even know if the AI is wrong.*

Those questions don't go away just because you start typing commands. If they're not answered up front, they sit in the back of your mind through every later chapter, making everything feel slightly uncertain even when the commands work. So this chapter answers them directly, once, before anything else — and it treats "I don't know what half these words mean" as a completely normal starting point, not a gap you should be embarrassed about.

---

## Part 1: the questions that actually stop people from starting

These are the real worries, answered straight, before anything else.

**Do I need to learn Python or any real programming to do this?**
No. Every script in this book — and there are several — is something you copy, paste, and run, with comments explaining what the important lines do. You're not expected to write code from scratch or understand every line of Python syntax. What you do need is comfort in a Linux terminal, which, as a DevOps engineer, you already have.

**Where does the AI model actually run — do I need a GPU server of my own?**
No, and this is one of the most important things to understand early. Amazon Bedrock runs the model on AWS's own hardware, somewhere in a data center you never see or manage. You send text in over an API and get text back. You will never provision a GPU, install a driver, or manage model weights anywhere in this book. If you've ever called any other AWS API — S3, DynamoDB, anything — calling Bedrock feels the same, just with a different kind of answer coming back.

**Is this just ChatGPT running inside AWS instead of in a browser?**
Close, but not quite. ChatGPT is a finished product: a chat website with a specific model behind it, built and operated by OpenAI. Amazon Bedrock is the layer *underneath* products like that — a way to call a model directly, over an API, without any chat website wrapped around it. You're not using someone else's chat app; you're building the same kind of thing that chat app is built on top of, which is a more useful skill than it sounds, because it's the same underlying idea whether the model comes from Amazon, Anthropic, or anyone else.

**How do I even check if an AI's answer is "correct" — there's no clear pass/fail?**
This is a real limit, not something to paper over. An AI model's answer is a fast first hypothesis, not a verified fact — treat it the way you'd treat a confident answer from a colleague you trust but still double-check on anything that matters. This book doesn't pretend otherwise: Chapter 3 tells you outright that you still confirm what the model suggests, and Chapter 7 builds an actual technique for making answers checkable — grounding the model in a specific document and having it say "not covered" instead of guessing when the answer isn't actually in there.

**Which AWS service do I even start with — Bedrock? SageMaker? Something else?**
Bedrock, and that's deliberate. Bedrock is for *calling* ready-made models that Amazon and other companies have already built and trained — which is everything this book does. SageMaker is a different job entirely: building, training, and tuning your own models from scratch, which needs data science skills this book doesn't assume you have and most DevOps roles don't require. If a tutorial starts with SageMaker, it's teaching you to build a model. This book teaches you to safely and cheaply *use* one, which is the far more common day-to-day skill.

**Is this going to replace my job, or is it just one more thing I operate?**
AI changes what's worth automating, but something still has to call the model safely, keep a key off the server, watch what it costs, catch it if it misbehaves, and clean up after it — none of that disappears because the "logic" is now an AI call instead of a script. That operating layer is exactly what this book teaches, and it's squarely DevOps work. The honest framing: AI is a new kind of service you're responsible for running well, not a replacement for the engineer who runs services well.

**What's a "prompt," and do I need to be good at writing them?**
A prompt is just the text you send the model — the question or instruction. In this book, every prompt is already written for you inside the scripts, so you don't need to craft one from scratch to finish any chapter. But since you'll use this skill far beyond this book, the short version of writing a decent prompt is: be specific about what you want, say what format the answer should come in, and give an example if the task is unusual. That's most of what "prompt engineering" actually is — there's no deeper trick hiding behind the term.

---

## Part 2: the map — every AI term you'll hear this week

Here's the vocabulary, in the order you'd actually run into it, each in plain English. The last column tells you honestly whether this book teaches it hands-on or just defines it so you're never lost later.

| Term | In plain English | In this book? |
|---|---|---|
| **Model** | A program that takes text in and produces text out, based on patterns learned from huge amounts of training data. Not a file you manage — a service you call. | Called in every chapter from Ch2 on |
| **LLM** (large language model) | The category of model this book uses: one trained specifically on language, able to answer questions, summarize, and write text. | Ch2 onward |
| **Model calling / inference** | The act of sending text to a model and getting an answer back. "Inference" is the formal name for using an already-trained model, as opposed to training one. | Ch2 onward |
| **Token** | A chunk of a word. Models read and write in tokens, and you're billed per token — roughly 1,000 tokens ≈ 750 English words. | Ch2 onward |
| **Prompt** (system vs. user) | The text you send. A *system* prompt sets the model's role and rules; a *user* prompt is the actual question. | Ch2 onward |
| **Context window** | The maximum amount of text (measured in tokens) a model can consider in one call — your prompt plus its answer have to fit inside it. | Explains *why* Ch3 and Ch7 work the way they do |
| **Context history** | What a model "remembers" in an ongoing conversation — in reality, the earlier messages are simply resent with every new call. The model has no memory of its own between calls. | Named here; not built hands-on in this book |
| **Temperature** | A setting that controls how predictable or varied the model's answer is. Low temperature, used throughout this book, means consistent, repeatable answers. | Ch2 onward |
| **Hallucination** | A model stating something confidently and fluently that is simply wrong. This is normal model behavior, not a bug — and it's the reason Chapters 5 and 7 exist. | Named here; addressed directly in Ch5 and Ch7 |
| **Embedding** | A way of converting text into a list of numbers that captures its meaning, so similar ideas end up as similar numbers. The basis for most real-world search-by-meaning systems. | Named here; Ch7 does a hand-rolled version of the idea without a real embeddings API |
| **Vector database** | A database built to store embeddings and quickly find the ones most similar to a new piece of text — the usual engine behind "search by meaning" at scale. | Named here; Ch7 deliberately skips it and shows why a plain Python approach is enough at small scale |
| **RAG** (retrieval-augmented generation) | Finding the relevant piece of your own data first, then handing only that piece to the model, so it answers from real text instead of guessing. | Built by hand, start to finish, in Ch7 |
| **Knowledge graph** | A structured map of facts and how they relate to each other (this person manages this team, this service depends on that one) — a different way of grounding answers than RAG's "find the relevant paragraph" approach. | Named here only |
| **Tool calling / function calling** | Letting a model decide to run a specific action — call an API, run a query — instead of just producing text. The mechanism that turns a model from "answers questions" into "can also act." | Named here only — this book never lets a model decide to act, by design |
| **MCP** (Model Context Protocol) | A standard way of connecting a model to tools and data sources, so different models can use the same tool connections. The reason it's suddenly everywhere is that it solves tool calling's biggest mess: every model previously needed its own custom wiring. | Named here only |
| **AI agent** | A system where the model doesn't just answer once — it plans, decides what to check or do next, and can take multiple steps on its own toward a goal. | Named here only — deliberately not in this book, see below |
| **Fine-tuning** | Further training an existing model on your own data so it behaves differently by default, rather than just being told what to do in the prompt. | Named here only; nothing in this book requires it |
| **Foundation model / inference profile / provider** | A foundation model is the base model itself (Nova Micro); an inference profile is a routable ID for it (`us.amazon.nova-micro-v1:0`) that lets AWS pick the best region; a provider is who built the model (Amazon, Anthropic, Meta, and others, all callable through Bedrock). | Ch2 onward, explained in full in Ch4 |
| **AI framework** (LangChain and similar) | A library that wraps the raw model-calling API with extra conveniences — chaining steps, managing memory, connecting tools. | Named here only; this book calls the raw API directly, on purpose, so you understand what the frameworks are wrapping |
| **Multimodal** | A model that can take in or produce more than just text — images, audio, video. | Named here only; every call in this book is plain text |

**The one line worth remembering from this whole table:** everything this book builds, Chapters 2 through 7, is one request in, one response out — a model that answers but never decides anything or acts on its own. Agents, tool calling, and MCP are all about a model that *acts across multiple steps*, which is a genuinely different kind of system to build, secure, and operate. That's a deliberate boundary for this book, not a gap in it — and it's exactly where a follow-up series on agents picks up, once this one has given you the request/response fundamentals to compare it against.

---

## Concept: what actually happens when you "call an AI model"

Strip away every buzzword above and a single Bedrock call looks like this, every single time, in every chapter from here on:

```
You (or your script)
   │
   │  prompt = system instructions + your question
   ▼
Amazon Bedrock  ──▶  the model reads your prompt, token by token
   │
   │  the model generates an answer, token by token
   ▼
Response: the answer text + how many tokens were used
```

That's it. No memory between calls unless you resend the earlier conversation yourself (context history). No ability to take an action unless you wire that up explicitly (tool calling — not in this book). No guarantee the answer is correct (hallucination is always possible). Every later chapter is this same loop, with one piece changed at a time: Chapter 2 does it from a server instead of your laptop, Chapter 4 puts it behind a public URL, Chapter 5 adds a check before the prompt goes in, Chapter 6 watches what it costs, Chapter 7 adds a retrieval step before the prompt is built. Understanding this one diagram is genuinely most of what you need to follow the rest of the book.

---

## Hands-on: confirm Bedrock is reachable in your account

One command, read-only, free. This just proves the model you'll use in every later chapter is actually available before you build anything on top of it.

Signed in as `opskart-admin`, region **us-east-1**, open **CloudShell**.

### Step 1: List the Bedrock models available in your account

```bash
aws bedrock list-foundation-models \
  --region us-east-1 \
  --query 'modelSummaries[?contains(modelId, `nova-micro`)].[modelId,modelName,providerName]' \
  --output table
```

**Expected output:**

```
---------------------------------------------------------------------------
|                          ListFoundationModels                          |
+------------------------------+------------------+-----------------------+
|  amazon.nova-micro-v1:0      |  Nova Micro      |  Amazon               |
+------------------------------+------------------+-----------------------+
```

That confirms two things at once: your account can reach Bedrock at all, and the specific model this entire book uses — Nova Micro — is available to you, with no access-request form needed.

### Step 2: See the full list, if you're curious

```bash
aws bedrock list-foundation-models \
  --region us-east-1 \
  --query 'modelSummaries[].[providerName,modelId]' \
  --output table
```

**Expected output:** a much longer table — dozens of models from Amazon, Anthropic, Meta, and other providers, all reachable through the exact same API shape. This is the "provider" row from the table above, made real: you're looking at one API in front of several companies' models.

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `list-foundation-models` returns an empty list | Wrong region | Confirm you're in `us-east-1`: `echo $AWS_REGION` |
| `AccessDeniedException` | Signed in as the wrong user, or IAM permissions are missing | Run `aws sts get-caller-identity` and confirm you're `opskart-admin` with AdministratorAccess (from Chapter 0) |
| `nova-micro` doesn't appear in the filtered query | A typo in the `contains()` filter, or AWS renamed the model | Re-run Step 2's full list and search the output for "Nova" yourself |
| Command runs but takes a long time | Normal — the full model list can be large | No action needed; it still returns in a few seconds |

---

## Interview questions

**1. What's the difference between Amazon Bedrock and Amazon SageMaker?**
Bedrock is for calling existing, pre-trained foundation models through an API — you never manage the model itself. SageMaker is for building, training, and tuning your own models, which is a data science task with a different skill set and a different cost profile entirely.

**2. What is a token, and why does it matter for cost?**
A token is roughly a word-piece — about 1,000 tokens equals 750 English words. Bedrock (and every major AI API) bills per input and output token, so token count is the number that turns directly into cost, which is why Chapter 6 builds monitoring around it rather than waiting for the AWS bill.

**3. What's the practical difference between what this book builds and an "AI agent"?**
Everything in this book is one request, one response — the model answers but never decides on its own to take an action or a further step. An agent plans and acts across multiple steps, choosing its own next move, which is a fundamentally different system to secure and operate: different failure modes, different monitoring needs, and a much harder "what did it just do and why" question to answer after the fact.

---

## Chapter 1 checklist

- [ ] Can explain, in one sentence each, what a token, a context window, and hallucination are
- [ ] Know the difference between Bedrock and SageMaker, and why this book uses Bedrock
- [ ] Understand that nothing in this book is an "agent" — every call is one request, one response
- [ ] Ran `list-foundation-models` and confirmed Nova Micro is available in your account
- [ ] Comfortable that no programming, no GPU, and no data science background is required to continue

**Next → Chapter 2: Your First AI Call from a Server.** Everything from here is hands-on — the first real Bedrock call, from a server, with zero access keys.
