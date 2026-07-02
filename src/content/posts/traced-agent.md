---
title: I Traced Personal Agent's Source Code. Inside Was Pi... And It Dreams at 3 AM.
date: 2026-06-15
---

*This is Part 2 of my homelab AI series. In [Part 1](https://dev.to/anup_sharma_86fa94612fe3c/i-built-an-ai-that-decides-which-ai-to-talk-to-running-247-from-my-living-room-211p), I built a system where one AI decides which AI to talk to. This time, I popped the hood on the agent itself — and what I found inside changed how I think about AI software.*

---

Last week I wrote about an [autonomous agent OpenClaw running on a Raspberry Pi](https://dev.to/anup_sharma_86fa94612fe3c/i-built-an-ai-that-decides-which-ai-to-talk-to-running-247-from-my-living-room-211p): an autonomous agent called **OpenClaw** running on a Raspberry Pi, routing requests through AgentGateway to three different LLMs based on intent. People loved it. A few folks DMed me asking how OpenClaw *actually works* — like, what happens after the routing? How does an autonomous agent that edits PDFs, writes code, schedules research, and finds the best restaurants in Indiranagar every Friday actually... *do* all that?

Honestly? I didn't fully know either. I knew OpenClaw was powerful. I used it daily. I'd even contributed some code. But I'd never really sat down and traced a request all the way through. So last weekend, I did.

And about 30 minutes in, I hit a line in `package.json` that stopped me cold:

```json
"@earendil-works/pi-agent-core": "0.75.4",
"@earendil-works/pi-coding-agent": "0.75.4"
```

OpenClaw doesn't have its own agent engine. Buried inside it — embedded as an SDK, not a subprocess, not an API call — is a tiny coding agent called **Pi**. Then I directly jump into youtube and found a great talk from [Mario at AI Engineer Conference ](https://www.youtube.com/watch?v=RjfbvDXpFls&t=16s)

And Pi might be the most elegant piece of AI software I've ever read.

---

## Wait, What is Pi?


![Pi on ghostty terminal running](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/motdupf9fotifuiheek5.png)



Pi is an open-source terminal coding agent written in TypeScript by **Mario Zechner**. If you've been in the AI coding agent space, you've probably heard of Cursor, Windsurf, Aider, or Claude Code. Pi sits in the same category but takes a radically different approach.

Where other agents keep adding features, Pi keeps removing them.

Where other agents have massive system prompts spanning thousands of tokens, Pi's is almost embarrassingly short.

Where other agents ship with dozens of built-in tools, Pi ships with **four**.

Yes. Four.

```plaintext
read   →  Read a file
write  →  Write a file
edit   →  Edit a file
bash   →  Run a shell command
```

That's it. That's the entire toolkit the LLM gets to work with.

And here's the thing that broke my brain: *it's enough.*

Think about it. What can you do with a terminal? You can read files, write files, edit files, and run commands. That's literally everything. `grep`? That's a bash command. `git commit`? Bash. `npm install`? Bash. `curl` an API? Bash. Run tests? Bash. Deploy to production? ...also bash.

Pi doesn't try to build a specialized tool for every possible operation. It gives the LLM the same primitives that *you* have as a developer, and trusts the model to compose them.

Armin Ronacher (of Flask fame) wrote about Pi back in January and called it a [glimpse into the future of software](https://lucumr.pocoo.org/2026/1/31/pi/). After spending a weekend inside the source code, I think he undersold and explain it very well. 

---

## How Pi Actually Runs Inside OpenClaw

Here's what surprised me the most: Pi isn't a separate service that OpenClaw calls over HTTP. It's not a subprocess. It's not even an RPC server.

OpenClaw literally imports Pi as an npm package and runs the agent loop **in the same process**.

```plaintext
OpenClaw starts up
    ↓
Calls createAgentSession() from @earendil-works/pi-coding-agent
    ↓
Pi's agent loop starts running in-process
    ↓
OpenClaw subscribes to Pi's events (message_start, tool_execution, turn_end, etc.)
    ↓
OpenClaw replaces Pi's default tools with its own extended set
    ↓
User sends a message on Discord → OpenClaw calls session.prompt(message)
    ↓
Pi takes over: talks to LLM, executes tools, streams responses
    ↓
OpenClaw receives events, formats them, sends back to Discord
```

This is wild to me. Pi is designed as a standalone CLI agent. You can `npm install -g @earendil-works/pi-coding-agent` and use it directly in your terminal. But Mario architected it so cleanly that the entire agent core can be extracted and embedded into another application like a library.

OpenClaw is the vehicle. Pi is the engine.


![Openclaw dependency on PI](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ennrdsthgam7y0jxyqbd.png)



---

## The Agent Loop: Where the Magic Happens

Let me walk you through what actually happens when I send a message to OpenClaw on Discord. This is where it gets fun.

Pi's agent loop lives in a single 743-line file (`agent-loop.ts`), and it follows a deceptively simple cycle:

```plaintext
┌─────────────────────────────────────────────────┐
│                  USER PROMPT                     │
└─────────────┬───────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────┐
│   Transform Context (extensions can modify)      │
│   Convert AgentMessages → LLM Messages           │
│   Send to LLM provider (streaming)               │
└─────────────┬───────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────┐
│          ASSISTANT RESPONSE                      │
│   ┌──────────────┐    ┌──────────────────┐      │
│   │  Text Reply   │    │  Tool Calls       │      │
│   └──────────────┘    └───────┬──────────┘      │
└───────────────────────────────┼──────────────────┘
              ↓                 ↓
        (no tools?)      Execute tools
         ↓                (parallel by default)
    ┌──────────┐              ↓
    │ Check     │      Tool Results
    │ follow-up │         ↓
    │ queue     │   ┌─────────────────────┐
    └──────────┘   │ Check steering queue │
         ↓         │ (user interrupts?)   │
    Empty? STOP    └─────────┬───────────┘
    Has msgs?                ↓
    → loop again       Loop back to LLM
                       with tool results
```

But here's where Pi gets clever. See those two queues?

### The Dual Queue System

Most agents have a simple loop: user says something → agent responds → done. Pi has two hidden message queues that make it far more powerful:

**1. Steering Queue** — "Hey, change direction."
These messages get injected *between* tool results and the next LLM call. If the agent is mid-task and you send a new message saying "actually, use TypeScript instead of Python," Pi doesn't wait for the current task to finish. It slides your message into the conversation right before the next LLM turn. The model sees the tool results AND your course correction, and adapts.

**2. Follow-Up Queue** — "Before you stop, consider this too."
These get checked *after* the agent would normally stop (no more tool calls). If there are follow-up messages, the agent continues instead of ending. Extensions use this to chain multi-step workflows without the user having to manually prompt each step.

This is elegant. Most agents treat conversations as request-response. Pi treats them as *navigable streams* that can be redirected mid-flight.

---

## The Part That Changed How I Think: Append-Only Tree Sessions

This is where I went from "oh, this is a nice agent" to "okay, this is genuinely brilliant engineering."

Most AI chat apps store conversations as a flat list. Message 1, message 2, message 3... linear. If you want to try a different approach, you either edit your message (and lose the original response) or start a new conversation entirely.

Pi stores conversations as an **append-only tree**.

```plaintext
                    Session Start
                         │
                    User: "Build me a REST API"
                         │
                    Assistant: "Sure, I'll use Express..."
                         │
              ┌──────────┴──────────┐
              │                      │
         [Branch A]             [Branch B]
    "Use FastAPI instead"    "Add authentication"
              │                      │
    Assistant: "Okay,         Assistant: "I'll add
    switching to Python..."   JWT middleware..."
              │                      │
         [Branch A1]            [Branch B1]
    "Add rate limiting"      "Use OAuth instead"
```

Every message is a node with an `id` and a `parentId`. When you fork a conversation, Pi creates a new branch from any point in the tree. The original branch stays untouched. You can navigate back and forth between branches, compare approaches, and even branch from a branch.

The session file is JSONL (one JSON object per line, append-only). It's never rewritten, never mutated. New messages just get appended with pointers to their parent.

Why does this matter? Three reasons:

**1. It's crash-proof.** Append-only means no data corruption on unexpected shutdown. Your Raspberry Pi loses power at 3 AM mid-response? The session is fine. Just re-open and continue from the last complete message.

**2. It enables time travel.** You can jump back to any point in the conversation and fork. "What if I'd asked for Rust instead of Python?" Just navigate back and try it. Both histories coexist.

**3. It makes compaction elegant.** When the context window fills up, Pi doesn't throw away old messages. It summarizes them into a `CompactionEntry` node in the tree. The original messages are still in the file — they're just not loaded into context anymore. You can always go back.


![branching features](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/muynsyy7f1ptx36843ym.png)



---

## Iterative Compaction: How Pi Remembers What Matters

Every AI agent has the same problem: context windows are finite. Eventually, your conversation gets too long and you hit the token limit. Most agents handle this by... well, by crashing. Or by silently dropping the oldest messages. Or by starting a new session.

Pi does something smarter. It runs **iterative compaction**.

When the context is getting full, Pi:

1. Walks backward from the newest messages, counting tokens
2. Keeps the most recent ~20,000 tokens intact (you want your recent context fresh)
3. Takes everything older and generates a structured summary via the LLM itself
4. Stores that summary as a `CompactionEntry` in the session tree
5. On the next context build, it loads the summary instead of the original messages

But here's the key word: **iterative**. When compaction runs a second time, Pi doesn't regenerate the summary from scratch. It takes the *existing* summary and **merges** new information into it. The summary evolves over time, like a living document.

The summary follows a structured format:

```markdown
## Goal
## Constraints & Preferences  
## Progress (Done / In Progress / Blocked)
## Key Decisions
## Next Steps
## Critical Context
```

It also tracks which files were read and modified across the entire session, even across multiple compactions. So if you ask "what files have we changed today?" after 6 hours of work and 3 compactions, Pi knows.

---

## OpenClaw's Memory: The Part Where AI Dreams

Okay, this is where things get genuinely sci-fi. And I mean that literally.

Pi handles context management within a single session beautifully. But what about *across* sessions? What about things you told the agent three weeks ago? What about your preferences, your coding style, the fact that you always want biryani recommendations from places with 4.5+ ratings?

OpenClaw builds a multi-layered memory system on top of Pi:

### Layer 1: File-Based Memory
- **`MEMORY.md`** — Long-term memory, loaded at every session start
- **`memory/YYYY-MM-DD.md`** — Daily notes (today + yesterday auto-loaded)
- **`DREAMS.md`** — A dream diary. Yes, really.

### Layer 2: Active Memory
Before every reply, a bounded sub-agent runs a quick memory search and injects relevant past context into the prompt. It has a circuit breaker — if it takes too long, it gets skipped.

### Layer 3: The Dreaming System 🌙

This is the one that made me put my laptop down and take a walk.

Every night at 3 AM, OpenClaw runs a **three-phase memory consolidation cycle** inspired by how human sleep works:

**Light Sleep** — Sorts through recent short-term memories. Stages candidates. Doesn't write anything yet. Just organizes.

**REM Sleep** — Reflects on recurring themes, patterns, and connections across memories. Still no writes. Just thinking.

**Deep Sleep** — Scores each memory candidate across 6 weighted signals and decides what gets promoted to long-term storage:

```plaintext
Relevance:            30%   (how useful is this?)
Frequency:            24%   (how often did this come up?)
Query Diversity:      15%   (was it relevant to different topics?)
Recency:              15%   (is it still timely?)
Consolidation:        10%   (does it connect to existing memories?)
Conceptual Richness:   6%   (is it a deep insight or just a fact?)
```

Memories that score high enough get written to `MEMORY.md`. Everything else fades.

**The agent literally sleeps, dreams, and wakes up smarter the next morning.**

I'm not going to pretend I wasn't a little unsettled the first time I realized my agent had reorganized its own memory overnight without being asked. But also... it remembered that I prefer tabs over spaces three weeks later without me mentioning it again. So, worth it.

<add screenshot of MEMORY.md or DREAMS.md showing consolidated memories>

---

## The Extension System: How OpenClaw Bends Pi to Its Will

Pi ships with 4 tools. OpenClaw's agent has dozens — browser automation, web search, image generation, cron scheduling, subagent spawning, Discord actions, PDF extraction, memory search, and more.

How? Pi's extension system.

Extensions are TypeScript files that hook into Pi's **30+ lifecycle events**:

```plaintext
Session Events:     session_start, session_before_compact, session_shutdown
Agent Events:       before_agent_start, agent_start, agent_end, turn_start, turn_end
Message Events:     message_start, message_update, message_end
Tool Events:        tool_call (can block!), tool_result (can modify!)
Input Events:       input (can intercept and transform user input)
Model Events:       model_select, thinking_level_select
Resource Events:    resources_discover
```

An extension can:
- **Register new tools** that the LLM can call
- **Intercept tool calls** before they execute (for safety, logging, sandboxing)
- **Modify tool results** after execution
- **Inject messages** mid-conversation (steering queue!)
- **Register custom LLM providers**
- **Override the system prompt**
- **Add UI widgets** to the terminal

When OpenClaw boots up, it calls `createAgentSession()` from Pi and then runs a **7-stage tool pipeline** that completely replaces Pi's default 4 tools with OpenClaw's full suite:

```plaintext
Pi's defaults → Custom replacements → OpenClaw tools → Channel-specific tools
    → Policy filtering → Schema normalization → AbortSignal wrapping
```

This is what good software architecture looks like. Pi doesn't try to be everything. It gives you a clean, minimal core and says: "Here are 30 hooks. Build whatever you want."

---

## Why This Architecture Works

After spending a weekend inside this codebase, I think Pi gets three things right that most AI agents get wrong:

### 1. Trust the Model, Don't Hand-Hold It

Most agents build a specialized tool for every operation: `search_files`, `list_directory`, `run_tests`, `git_commit`, `install_package`... 

Pi says: here's `bash`. Figure it out.

This seems reckless until you realize that modern LLMs are *really good at shell commands*. They know `grep`. They know `find`. They know `git`. Giving them `bash` and getting out of the way produces better results than giving them 50 narrow tools with rigid parameter schemas.

### 2. State is a Tree, Not a Line

Linear chat history is a lie. Real problem-solving is branching. You try approach A, realize it's wrong, backtrack, try approach B. Pi's tree sessions make this a first-class operation instead of a hack.

### 3. Extensions > Features

Instead of shipping a monolithic agent with every feature imaginable, Pi ships a tiny core with a powerful extension system. OpenClaw adds 129 extensions. My homelab setup is much simpler. Both work, because the core doesn't care what you bolt onto it.

---

## Setting This Up For Yourself

If you want to try Pi standalone (no OpenClaw, just the coding agent):

```bash
npm install -g @earendil-works/pi-coding-agent
pi
```

That's it. You now have a terminal coding agent with 4 tools, tree sessions, and iterative compaction.

If you want the full OpenClaw experience — Discord integration, dreaming, multi-agent orchestration, 129 extensions — check out [openclaw.ai](https://openclaw.ai). Fair warning: once you have an agent that dreams and remembers your preferences across weeks, going back to stateless ChatGPT feels like using a typewriter.

---

## What I'm Building Next

In Part 1, I built the routing layer (which AI answers). In this post, I explored the engine (how the AI thinks). The next piece of the puzzle: **observability**.

AgentGateway already emits OpenTelemetry traces for every LLM call. Pi tracks token usage, tool execution times, and compaction events. I want to pipe all of this into a Grafana dashboard so I can see, in real-time:

- Which model is handling which type of request
- How many tokens stay local vs go to the cloud
- How long tool executions take
- When compaction fires and how much context it saves
- What the dreaming system promoted to long-term memory

Stay tuned.

---

*If you made it this far — first of all, respect. Second, if you're building something similar or want to nerd out about agent architectures, hit me up. I live for this stuff.*


![Running PI Agent](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3vu8sulu35sljysd90h6.png)



#AI #CodingAgent #OpenClaw #Pi #LLM #AgentArchitecture #HomeLab #BuildInPublic #AIEngineering #OpenSource

