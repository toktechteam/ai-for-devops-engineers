# DevOps → AI: The Lab Book

**By [theOpsKart](https://github.com/theopskart) — hands-on AWS AI labs for DevOps engineers, written and tested in a real AWS account.**

## What this is

This is a free, open-source set of hands-on labs that take a DevOps or cloud engineer from "I've never touched AI" to having built seven small, real, working AI projects on AWS — each one backed by working code, a tested set of commands, and a short YouTube video.

The series is complete: Chapter 0 (setup) through Chapter 7 (mini-RAG), eight chapters in total, each with its own video.

There is no paid version of this. Clone the repo, follow the chapters in order, and build everything yourself in your own AWS account using free-tier credits.

## Who this is for

You're a DevOps engineer, cloud engineer, SRE, or fresher with 0–3 years of experience who is comfortable with a Linux terminal and basic AWS concepts (EC2, IAM, the console), but has never called an AI model from code. You don't need to know Python — every script is copy-paste with the comments explaining what each part does. You don't need a GPU, a data science background, or any machine learning theory.

## The goal

By the time you finish all eight chapters, you will have:

- Walked away from Chapter 1 with every basic AI term and "wait, but how does this actually work" question answered, before touching a single line of code
- Called a real AI model (Amazon Bedrock, Nova Micro) from a server, from a Lambda function, and from the command line — using **zero long-lived access keys**, the same way a production system would
- Built a tool that reads a real error log and tells you the root cause and the fix, in plain English
- Exposed an AI call as a public HTTPS API, secured with IAM, with no server left running
- Put a real safety layer in front of an AI feature — blocking secrets before they reach the model, and masking personal data in your own code
- Built a monitoring dashboard and a budget alarm for an AI feature, the way a team would watch any production service
- Built a basic "answer questions from my own document" system (RAG) without touching a vector database
- Seven working, documented projects in your own GitHub repo, with real AWS CLI output you generated yourself — not copy-pasted from a tutorial

**Why this matters for your career, not just your learning:** every one of these labs maps to a real, recurring task in a DevOps/SRE/cloud role, and together they're a concrete answer to the interview question "have you actually worked with AI, or just read about it." You'll finish with working code and real screenshots/video of you running it — material you can put directly into your resume and portfolio.

## What you'll actually learn, chapter by chapter

| # | Chapter | What you build | Why it's worth your time |
|---|---|---|---|
| 0 | [Start Here](guide/ch00_start_here.md) | A safe AWS account: MFA, an admin IAM user, a $5 budget alert, region set, CloudShell verified | Almost every "it didn't work" moment in the rest of the book traces back to skipping this setup. Do it once, properly, and never think about it again. |
| 1 | [AI in One Page for DevOps People](guide/ch01_ai_in_one_page.md) | Nothing deployed — straight answers to every "wait, but..." question a DevOps engineer has before touching AI, plus the full term map (tokens, RAG, agents, MCP, and the rest) and a confirmed, working connection to Bedrock from the CLI | Skips this and every later chapter still works, but you'll be guessing at vocabulary the whole way. Fifteen minutes here means nothing after this ever feels like jargon you were supposed to already know. |
| 2 | [Your First AI Call](guide/ch02_first_ai_call.md) | An EC2 server that calls Bedrock with **zero access keys**, using an IAM role | This is the one idea every later chapter reuses: give a server a role, not a key. Understand it here and everything after this is "the same trick, different wrapper." |
| 3 | [AI Log Explainer](guide/ch03_log_explainer.md) | A script that reads any error log and returns a root cause and fix | The first genuinely useful tool in the book — not a demo, something you'll likely keep using on real incidents. |
| 4 | [Serverless AI API](guide/ch04_serverless_ai_api.md) | The same AI call, behind a Lambda Function URL — a real HTTPS API, no server to manage | Turns a script only you can run into something a website, bot, or app can call. The core "serverless API" pattern used everywhere in cloud work. |
| 5 | [Guardrails](guide/ch05_guardrails.md) | A Bedrock Guardrail that blocks secrets, plus your own code that masks emails/IPs/phone numbers | Answers the first question any security team asks about an AI feature: what stops secrets and customer data from leaking into the model? |
| 6 | [Watch Your AI](guide/ch06_watch_your_ai.md) | A CloudWatch dashboard and a budget alarm for your AI calls | Moves you from "I printed a token count once" to "I'll get an email if this starts costing too much" — the monitoring habit every production service needs. |
| 7 | [Mini-RAG](guide/ch07_mini_rag.md) | A system that answers questions only from your own runbook, with no vector database | Teaches the real idea behind "RAG" — retrieve the relevant piece, then answer only from it — in code short enough to read top to bottom, before you ever need a managed search service. |

## What you'll need

- An AWS account (new accounts get **$100 in free credits** at sign-up, with a path to $100 more — Chapter 0 shows you how)
- A laptop with Chrome — no software to install; every lab runs in AWS CloudShell, a terminal in your browser
- About 30–45 minutes per chapter
- No coding experience required

## How to use this repo

1. Start with **Chapter 0** and go in order — later chapters assume the IAM role and setup from earlier ones.
2. Every chapter is a single markdown file with the exact commands to run and the exact output to expect.
3. Each chapter has a short YouTube video covering the console walkthrough — the `.md` file has every command, the video has the "watch me click through it" version. Either one is enough on its own.
4. **Every chapter ends with a cleanup step.** Run it. This book creates real AWS resources, and leaving them running is the only way any of this costs you real money. If you stop partway through a chapter, go back and run that chapter's cleanup commands before moving on.
5. Hit a problem? Every chapter has a troubleshooting table covering the errors that came up during real testing in a real account.

## A note on how this was built

Every chapter in this repo was written, then actually run — command by command — in a real AWS account before being published. Where something didn't work as documented (for example, a Bedrock Guardrail feature that didn't behave as expected during testing — see Chapter 5), the chapter says so honestly rather than pretending the first draft was correct. If you hit something that doesn't match what's written, please open an issue — this repo is actively maintained and corrected against real testing.

## License and contributions

This guide is free to use, fork, and share. If you build something from it, tag **@theOpsKart** — I'd like to see it. Corrections and pull requests are welcome, especially fixes based on your own testing in your own account.
