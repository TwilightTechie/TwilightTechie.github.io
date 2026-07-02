---
title: I Built an AI That Decides Which AI to Talk To — Running 24/7 From My Living Room
pubDatetime: 2026-06-10T00:00:00Z
description: "I Built an AI That Decides Which AI to Talk To — Running 24/7 From My Living Room"
tags:
  - blog
---
Last Saturday when I woke up, my AI agent reviewed 14 restaurant ratings in Indiranagar, updated a shared Google Sheet, signed a 20-page PDF I'd been ignoring for a week, and wrote a bash script to clean up my server logs.

I didn't ask it to do any of that. It just... does things now.

Meet **OpenClaw** — my long-running autonomous agent that lives on a Raspberry Pi, plugged into Discord, running 24/7. It manages my memory, handles research, writes code, edits documents, finds the best weekend spots in Bangalore by scraping live ratings — basically, it runs half my life on autopilot.

**But a few weeks ago, I noticed something that bothered me.**

I asked it: _"Write a Python script to parse JSON logs."_ Simple coding task. It sent that request to a cloud API, waited 3 seconds, burned tokens I paid for, and came back with an answer — when I had a perfectly capable local LLM sitting idle on my Mac Mini, three feet away.

Then I asked: _"Think step by step about the trade-offs between event-driven vs polling architecture for my notification system."_ That's a hard reasoning question. I want that going to a frontier model. That's worth the tokens.

Same agent. Same endpoint. Completely different needs.

And that's when a stupid idea hit me:

**What if the system could figure out which brain to use — before the request even reaches a model?**

Turns out, it's not stupid at all. And it took me a weekend, a Raspberry Pi, a Mac Mini, 50 lines of Python, and an open-source gateway to build it.

Here's how.

## The Setup

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xd9x1tqn50cf1lew8lu8.png)


Here's what's running in my living room:

**Raspberry Pi** → Runs OpenClaw, my autonomous agent. It takes input from Discord, manages context, memory, and orchestrates everything.
**Mac Mini** → The brain farm. Runs three things:
Ollama with qwen2.5-coder:7b — a local coding model that never leaves my network
**AgentGateway** — an open-source AI gateway from Google that handles routing, auth, observability
**A lightweight Python router** — the "intent classifier" I wrote in ~50 lines of code
The magic? OpenClaw doesn't know any of this is happening. It just sends a request to one endpoint. Behind the scenes, the system figures out the rest.

## The Architecture


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gy4hte82eh0nergs3xya.png)


Three models. Three price points. One unified endpoint. OpenClaw just hits http://192.168.1.15:1234/v1/chat/completions and forgets about it.


### Why AgentGateway?
I evaluated a few options — raw Envoy, Nginx with Lua scripting, even building a full proxy from scratch. But **AgentGateway** stood out for a few reasons:

What it gives you out of the box:
**Protocol translation** — It speaks OpenAI-compatible API on the frontend, but can talk to Gemini, Vertex AI, Bedrock, Ollama, and more on the backend. I don't write a single line of provider-specific code.
**Backend authentication** — API keys are managed at the gateway level. OpenClaw never sees or stores any API key. I just set backendAuth: key: $GEMINI_API_KEY in the config and it handles the rest.
**Model aliasing** — OpenClaw sends model: "inteli-llm" in every request. AgentGateway silently translates that to qwen2.5-coder:7b, gpt-4o, or gemini-2.5-flash depending on which route matched. The client has no idea.
**Observability** — Every request gets logged with provider name, model, token counts, and latency. I can see exactly how many tokens are going to OpenAI vs staying local.
**Prompt guards & rate limiting** — Built-in regex-based PII masking, webhook-based content moderation, and rate limiting. Enterprise-grade features I get for free.
**Weighted load balancing & failover** — If Ollama crashes (it happens), I can configure automatic failover to a cloud model. No downtime.
**What it doesn't do (yet):** Content-aware routing. AgentGateway routes based on path, headers, and methods — which is the right design for a gateway. It doesn't peek into your request body to decide where to send it. That's a feature, not a bug — gateways should be fast and protocol-level, not parsing JSON payloads.

But I needed content-aware routing. So instead of searching for other tool, I extended it.

### The 50-Line Router That Makes It All Work

I wrote a tiny FastAPI proxy that sits in front of AgentGateway. Here's what it does:

- Intercepts the incoming OpenAI-compatible request
- Reads the last message in the chat
- Classifies intent using simple keyword matching + prompt length heuristics:
   - Contains `code`, `python`, `script`, `function`, `bug`? → coding
   - Contains `think`, `analyze`, `reasoning`, `deduce`? Or prompt > 400 chars? → reasoning
   - Everything else? → simple
- Injects an x-intent HTTP header
- Forwards the request to AgentGateway untouched
That's it. No ML model for classification. No vector databases. No semantic similarity. Just good old keyword matching that works 90% of the time — and that's good enough for a homelab.

```python
coding_keywords = ["code", "python", "javascript", "bash", "script", "function", "bug"]
reasoning_keywords = ["think", "analyze", "explain in detail", "reasoning", "logic", "deduce"]

if any(k in prompt_lower for k in coding_keywords):
    intent = "coding"
elif len(prompt) > 400 or any(k in prompt_lower for k in reasoning_keywords):
    intent = "reasoning"
else:
    intent = "simple"
```

### The Cost Equation
Here's what this setup actually saves me:

| Intent | Model | Where it runs | Cost per 1M tokens |
| :--- | :--- | :--- | :--- |
| **Coding** | qwen2.5-coder:7b | Local (Ollama) | $0 |
| **Simple Q&A** | gemini-2.5-flash | Google Cloud | ~$0.15 |
| **Deep Reasoning** | gpt-4o | OpenAI | ~$2.50 |

Before this setup, every single request was going to a cloud API. Now, roughly 60-70% of my queries stay local — coding questions, quick lookups, simple formatting tasks. They're fast, free, and private.

The expensive reasoning model only gets called when I genuinely need it. And the mid-tier Gemini handles everything in between.

My monthly API bill dropped significantly, and the local responses are actually faster.

### Design Choices & Why They Worked
**1. Header-based routing over path-based routing** Initially, I was going to use URL paths (`/coding`, `/reasoning`, `/simple`) and strip them with URL rewriting. But header injection is cleaner — the original request path stays intact, and AgentGateway's header matching is first-class.

**2. Classification at the proxy, not the gateway** I could have tried to use AgentGateway's CEL expressions or ExtProc policies for classification. But those run after backend selection, not before. Keeping classification in a separate lightweight layer means I can swap algorithms without touching my gateway config.

**3. Keyword heuristics over ML classifiers** Could I use a small classifier model or even RouteLLM for smarter routing? Absolutely. But for a homelab, keyword matching is:
- Zero latency overhead
- Zero dependencies
- Easy to debug (just read the logs)
- Surprisingly accurate for my use cases

**4. One unified model name** OpenClaw sends model: `"inteli-llm"` for everything. AgentGateway's `modelAliases` feature translates it per-route. This means I can swap out backend models without touching a single line of OpenClaw's config. Last week it was `gemini-1.5-flash`, this week it's `gemini-2.5-flash`. OpenClaw never knew.

## What's Next
**Smarter classification** — Maybe a tiny local classifier model, or even using the first few tokens of a response to reclassify and retry on a better model.
**Metrics dashboard** — AgentGateway already emits OpenTelemetry traces. I want to hook up a Grafana dashboard to see which models are handling what, with latency and token breakdowns.
**Failover chains** — If Ollama is under heavy load, automatically fall back to Gemini for coding tasks. AgentGateway supports priority groups for this.
**More agents** — OpenClaw is just the beginning. I want to run specialized agents for different domains, all routing through the same gateway.

## The Takeaway
You don't need a Kubernetes cluster or a $10K GPU server to build a multi-model AI system. A Raspberry Pi, a Mac Mini, an open-source gateway, and 50 lines of Python got me:

✅ An always-on autonomous agent ✅Intelligent routing ✅across 3 different LLMs ✅Local-first for privacy and speed ✅Cloud when I need the horsepower ✅Zero API keys exposed to the client ✅A monthly bill I actually don't mind paying

The best part? The entire config is a single YAML file and a single Python script. No Docker. No Kubernetes. No Terraform. Just two processes on a Mac Mini and an agent on a Pi.

Sometimes the best infrastructure is the one you can explain in a napkin sketch.

If you're building something similar or want to see the config files, drop a comment — happy to share the full setup.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6rmzapxb0p2vk8wdzjgn.png)


#AI #HomeAssistant #LLM #AgentGateway #Ollama #OpenAI #Gemini #HomeLab #BuildInPublic #MacMini #RaspberryPi #AIEngineering
