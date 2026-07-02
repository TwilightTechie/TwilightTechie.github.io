---
title: Giving AgentGateway a Semantic Brain with vLLM Semantic Router - Inside My Homelab
pubDatetime: 2026-06-22T00:00:00Z
description: "Giving AgentGateway a Semantic Brain with vLLM Semantic Router - Inside My Homelab"
tags:
  - blog
---

*Part 3 of the Homelab AI Series — [Part 1](https://dev.to/anup_sharma_86fa94612fe3c/i-built-an-ai-that-decides-which-ai-to-talk-to-running-247-from-my-living-room-211p) | [Part 2](https://dev.to/anup_sharma_86fa94612fe3c/i-traced-personal-agents-source-code-inside-was-pi-and-it-dreams-at-3-am-o0f)*

---

## The Problem Was Embarrassing

In [Part 1](https://dev.to/anup_sharma_86fa94612fe3c/i-built-an-ai-that-decides-which-ai-to-talk-to-running-247-from-my-living-room-211p), I showed how I built a personal AI agent (Pi) that runs 24/7 from my living room, using [AgentGateway](https://github.com/agentgateway/agentgateway) to route requests across three models: a local Ollama (`qwen2.5-coder:7b`) for coding, OpenAI (`gpt-4o`) for deep reasoning, and Gemini (`gemini-2.5-flash`) for fast general tasks.

The routing brain? A 100-line Python script sitting between Pi and AgentGateway:

```python
# router.py — The "AI brain" I was embarrassed to deploy
coding_keywords = ["code", "python", "javascript", "bash", "script",
                   "function", "bug", "error", "html", "css"]
reasoning_keywords = ["think", "analyze", "explain in detail",
                      "reasoning", "logic", "deduce"]

if any(k in prompt_lower for k in coding_keywords):
    intent = "coding"
elif len(prompt) > 400 or any(k in prompt_lower for k in reasoning_keywords):
    intent = "reasoning"
else:
    intent = "simple"
```

Yes. My "intelligent" AI routing was a glorified `if-elif-else` chain.

It worked — until it didn't. "Explain the async/await pattern in Rust" got classified as `simple` because none of the keywords matched. "Help me think about dinner options" got classified as `reasoning` because `think` was in the keyword list. And anything in Hindi or mixed-language prompts? Straight to the fallback, every single time.

After running this setup daily for two weeks, I collected some rough numbers:

| Metric | With Python Router |
|---|---|
| Misrouted requests (spot-checked) | ~18% |
| Monthly estimated API cost | ~$24 |
| Routing latency (Python proxy hop) | ~45ms |
| Keyword list maintenance | Manual, weekly tweaks |

Eighteen percent of requests going to the wrong model doesn't just waste money — it gives *bad answers*. When my cron-job agent sends a complex "summarize this week's calendar and suggest optimizations" to the 7B local model instead of Gemini or GPT-4o, the output is noticeably worse.

I needed something that *understood* the prompt, not just scanned it for keywords.

pubDatetime: 2026-06-22T00:00:00Z
description: "Giving AgentGateway a Semantic Brain with vLLM Semantic Router - Inside My Homelab"
tags:
  - blog
---

## Enter vLLM Semantic Router

While discussing with Maintainers of AgentGateway [AgentGateway](https://github.com/agentgateway/agentgateway), I discovered a first-class integration with [vLLM Semantic Router](https://github.com/vllm-project/semantic-router) thanks to [Keith Mattix](https://www.linkedin.com/in/keithmattix/) and [John Howard](https://www.linkedin.com/in/-johnhoward/). The architecture clicked immediately:


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qtfyqnooqrwbv6uokrt1.png)



Instead of my Python script sitting *in front* of AgentGateway as a janky reverse proxy, the Semantic Router runs as an **Envoy ExtProc sidecar**. AgentGateway pauses the request, sends the HTTP body to the SR's gRPC endpoint, gets back a header mutation (`x-selected-model: qwen-coder`), and resumes routing. Zero proxy hops. Zero Python processes. Just gRPC-native intelligence inside the gateway's own request lifecycle.

The SR uses an embedded **mmBERT** model (a 2D Matryoshka embedding model, ~130MB) to semantically classify every prompt and compare it against model descriptions you write in YAML. No keyword lists. No regex. Actual embeddings.

### The Architecture

```plaintext
┌─────────────────────────────────────────────────────┐
│                  Client (Pi Agent)                   │
│             POST /v1/chat/completions                │
│                  model: "MoM"                        │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│              AgentGateway (:3000)                     │
│                                                       │
│  1. Receive request                                   │
│  2. Pause → send body to ExtProc (gRPC :50051)       │
│  3. SR analyzes prompt with mmBERT embeddings         │
│  4. SR returns header: x-selected-model: qwen-coder  │
│  5. Resume → match route by header → forward          │
└──────┬──────────────┬──────────────┬────────────────┘
       │              │              │
       ▼              ▼              ▼
   ┌────────┐   ┌──────────┐   ┌──────────┐
   │ Ollama  │   │ OpenAI   │   │ Gemini   │
   │ :11434  │   │ Cloud    │   │ Cloud    │
   └────────┘   └──────────┘   └──────────┘
```

---

## Setting It Up (Two YAML Files)

The entire setup is defined in two config files. No code. No Python.

### 1. Semantic Router Config (`config.yaml`)

This tells the SR about your models and how to route between them:

```yaml
version: v0.3

providers:
  defaults:
    default_model: qwen-coder
  models:
    - name: qwen-coder
      provider_model_id: qwen2.5-coder:7b
      api_format: openai
      backend_refs:
        - name: local-ollama
          endpoint: host.docker.internal:11434
          protocol: http

    - name: gpt-4o
      provider_model_id: gpt-4o
      api_format: openai
      backend_refs:
        - name: openai-cloud
          base_url: https://api.openai.com/v1

    - name: gemini-flash
      provider_model_id: gemini-2.5-flash
      api_format: openai
      backend_refs:
        - name: gemini-cloud
          base_url: https://generativelanguage.googleapis.com/v1beta/openai

routing:
  modelCards:
    - name: qwen-coder
      param_size: 7B
      context_window_size: 32768
      description: >
        Specialized coding model optimized for programming tasks.
        Excellent at writing code, debugging, algorithms, data structures,
        code review, refactoring, and technical implementation in Python,
        Rust, JavaScript, Go. Best for code generation, fixing bugs,
        writing tests, and technical programming Q&A.

    - name: gpt-4o
      param_size: 200B+
      context_window_size: 128000
      description: >
        Frontier reasoning model with exceptional analytical capability.
        Best for complex multi-step reasoning, strategic analysis,
        comparing trade-offs, writing long-form essays, nuanced
        explanations, math proofs, scientific reasoning.

    - name: gemini-flash
      param_size: ~100B
      context_window_size: 1000000
      description: >
        Fast general-purpose model. Ideal for simple factual questions,
        quick lookups, summarization, casual conversation, translations,
        everyday tasks, and when speed matters more than depth.

  decisions:
    - name: MoM
      description: "Mixture of Models router"
      priority: 100
      rules: {}
      modelRefs:
        - model: qwen-coder
        - model: gpt-4o
        - model: gemini-flash
      algorithm:
        type: multi_factor
        multi_factor:
          weights:
            quality: 0.1
            latency: 0.4
            cost: 0.5
          slo:
            max_cost_per_1m: 0.5
```

The key insight: **you describe what each model is *good at* in natural language, and the SR uses those descriptions as semantic anchors**. No keyword lists to maintain. When a new prompt arrives, the SR embeds it and compares it against these descriptions using cosine similarity. The model whose description is closest to the prompt wins.

### 2. AgentGateway Config (`homelab_config.yaml`)

This tells AgentGateway to use the SR as an ExtProc sidecar, and to route based on the header it sets:

```yaml
# Gateway-level policy: ExtProc to Semantic Router
policies:
- name:
    name: semantic-router
    namespace: default
  target:
    gateway:
      gatewayName: default
  phase: gateway
  policy:
    extProc:
      host: "127.0.0.1:50051"
      processingOptions:
        requestBodyMode: buffered
        responseBodyMode: none
        requestHeaderMode: send
      failureMode: failOpen   # If SR is down, fall through

binds:
- port: 3000
  listeners:
  - routes:
    # When SR sets x-selected-model: qwen-coder → Local Ollama
    - matches:
      - headers:
        - name: "x-selected-model"
          value:
            exact: "qwen-coder"
      backends:
      - ai:
          provider:
            openAI: {}
          name: ollama
          hostOverride: "localhost:11434"

    # When SR sets x-selected-model: gpt-4o → OpenAI
    - matches:
      - headers:
        - name: "x-selected-model"
          value:
            exact: "gpt-4o"
      backends:
      - ai:
          provider:
            openAI: {}
          name: openai
        policies:
          backendAuth:
            key: $OPENAI_API_KEY

    # When SR sets x-selected-model: gemini-flash → Google
    - matches:
      - headers:
        - name: "x-selected-model"
          value:
            exact: "gemini-flash"
      backends:
      - ai:
          provider:
            gemini: {}
          name: gemini
        policies:
          backendAuth:
            key: $GEMINI_API_KEY

    # Fallback if SR is down (failOpen)
    - backends:
      - ai:
          provider:
            gemini: {}
          name: gemini-default
        policies:
          backendAuth:
            key: $GEMINI_API_KEY
```

Notice the **separation of concerns**: the Semantic Router *never* touches API keys. It classifies the prompt and mutates a header. AgentGateway owns the downstream auth. This is exactly how infrastructure teams design production gateways — the routing intelligence is decoupled from the security posture.

And that `failureMode: failOpen`? It means if the SR container ever crashes or is restarting, AgentGateway seamlessly falls through to the default Gemini route. I've tested this — during SR container restarts, Pi's requests still get answered without a single error. The agent doesn't even notice.

pubDatetime: 2026-06-22T00:00:00Z
description: "Giving AgentGateway a Semantic Brain with vLLM Semantic Router - Inside My Homelab"
tags:
  - blog
---

## The ARM64 Rabbit Hole (Two Bugs, Two PRs)

Here's where the story gets real. I run this on an **Apple Silicon Mac Mini** (M-series, ARM64). Everything installed fine. The SR container started. And then:

```json
{
  "msg": "embedding_models_init_completed",
  "embedding_ready": false,
  "tools_ready": false
}
```

The mmBERT model loaded but the embedding runtime never became ready. Every routing attempt logged:

```plaintext
Failed to embed model qwen-coder: failed to generate batched embedding (status: -1)
```

### Bug #1: Wrong FFI Dispatch ([#2172](https://github.com/vllm-project/semantic-router/issues/2172))

After deep-diving into the SR source code, I discovered the issue. The Go router was calling `candle_binding.GetEmbeddingBatched()` for *all* model types — but the Rust FFI backend only supports batched embeddings for `qwen3` architectures. For `mmbert` (the default), it returned `status: -1`.

The fix ([PR #2192](https://github.com/vllm-project/semantic-router/pull/2192)) was elegant — a 15-line change that adds a dispatch check:

```go
// Only qwen3 supports the batched FFI. Others use single-text FFI.
func candleEmbeddingSupportsBatched(modelType string) bool {
    return modelType == "qwen3"
}
```

For non-qwen3 models, it gracefully falls back to `GetEmbeddingWithModelType()`, which works perfectly on ARM64.

### Bug #2: Missing Model Files on First Boot ([#2173](https://github.com/vllm-project/semantic-router/issues/2173))

The second issue was subtler. When the SR container downloaded the mmBERT model files from HuggingFace on first boot, several required files (like `tokenizer.json` and `config.json`) weren't being fetched. This was a download-completeness bug in the model resolver.

Fixed in [PR #2195](https://github.com/vllm-project/semantic-router/pull/2195).

### A Huge Thank You 🙏

Both issues were triaged and fixed within days by the vLLM Semantic Router team, particularly [@WUKUNTAI-0211](https://github.com/WUKUNTAI-0211) who wrote the fix for the FFI dispatch and [@theohsiung](https://github.com/theohsiung) for the file completeness fix. The PRs are now merged into `main`. If you're running on ARM64/Apple Silicon, just pull the latest and it works. Also shout out to [AayushSaini101] (https://github.com/AayushSaini101) for encouraging me recently to contribute to repo. 

This is open source at its best. I filed two issues with reproduction steps and log snippets, and got working fixes merged into the upstream repo. The community aspect of this project is exceptional.

---

## The Proof: Real Routing Logs

Let me show you what it actually looks like when a request flows through. I send this:

```bash
curl http://localhost:3000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MoM",
    "messages": [
      {"role": "user", "content": "Write me a Python function to compute fibonacci numbers using memoization"}
    ]
  }'
```

### Step 1: SR Classifies the Prompt (1ms!)

```json
{
  "msg": "routing_decision",
  "original_model": "MoM",
  "selected_model": "qwen-coder",
  "reason_code": "auto_routing",
  "routing_latency_ms": 1,
  "component": "extproc"
}
```

One millisecond. The SR embedded the prompt, compared it against the three model descriptions, and decided this is a coding task → `qwen-coder`.

### Step 2: AgentGateway Routes to Ollama

```console
info  request
  gateway=default/default
  route=default/route0
  endpoint=localhost:11434
  http.status=200
  gen_ai.request.model=qwen2.5-coder:7b
  gen_ai.response.model=qwen2.5-coder:7b
  gen_ai.usage.input_tokens=41
  gen_ai.usage.output_tokens=366
  duration=22537ms
```

AgentGateway matched the `x-selected-model: qwen-coder` header, routed to the local Ollama endpoint, and the entire round-trip (including LLM generation) completed in 22.5 seconds. The routing overhead? **1ms**. The rest is just Ollama thinking.

### Step 3: The SR Startup Sequence

On container boot, you see the full model loading pipeline:

```json
{"msg":"embedding_models_init_started","mmbert_configured":true,"use_cpu":true}
```
```plaintext
INFO: mmBERT embedding model registered with 2D Matryoshka support
```
```json
{"msg":"embedding_models_initialized","use_batched":false}
```
```json
{"msg":"selection_factory_initialized","selector_count":14}
```
```json
{"msg":"startup_complete","embedding_ready":false,"sem_cache_enabled":true,
 "model_selection":true,"extproc_port":50051,"decisions":"MoM"}
```

14 selection algorithms available out of the box. Multi-factor, ELO, reinforcement-learning-driven, hybrid, latency-aware, session-aware, KNN, SVM, K-means — all registered and ready. I'm using `multi_factor` with cost-heavy weighting, but I can switch to any of these with a single YAML change. Try doing that with a Python keyword list.

pubDatetime: 2026-06-22T00:00:00Z
description: "Giving AgentGateway a Semantic Brain with vLLM Semantic Router - Inside My Homelab"
tags:
  - blog
---

## The Numbers After Two Weeks

After running the SR-powered setup alongside Pi for two weeks, here's the comparison:

| Metric | Python Router | vLLM Semantic Router |
|---|---|---|
| **Misrouted requests** | ~18% | ~3% (subjective spot-checks) |
| **Routing latency** | ~45ms (HTTP proxy) | **1-3ms** (gRPC ExtProc) |
| **Monthly estimated API cost** | ~$24 | **~$14** |
| **Maintenance effort** | Weekly keyword updates | Zero (model descriptions are stable) |
| **Failover behavior** | Manual restart | Automatic failOpen to Gemini |
| **Language support** | English keywords only | Multi-language (embedding-based) |
| **Config** | 100 lines of Python | 2 YAML files |

The cost savings come from fewer misroutes. When "explain the async/await pattern in Rust" correctly goes to the local Ollama instead of GPT-4o, that's a $0.003 request instead of $0.03. Across hundreds of daily requests from Pi's cron jobs and my direct usage, it adds up fast.

---

## Why Every Agent Builder Needs This

If you're building agents — whether it's a personal Pi running on a Mac Mini or a production fleet of agents in Kubernetes — you need a routing layer that *understands* prompts. Here's why:

1. **Cost control is the #1 agent problem.** Agents generate a lot of requests. Without intelligent routing, every request goes to your most expensive model. The SR's `multi_factor` algorithm explicitly weighs cost, latency, and quality.

2. **Keyword routing doesn't scale.** The moment your agent handles a domain you didn't anticipate (my Pi started doing recipe research — none of my keywords covered "sourdough starter hydration"), keyword-based routing silently fails.

3. **AgentGateway + SR is production-grade.** This isn't a hobby-tier setup. AgentGateway is a Gateway API data plane built in Rust. The SR is an Envoy ExtProc server written in Go and Rust, backed by the vLLM project. This is the same architecture you'd deploy in a Kubernetes cluster with 50 models.

4. **Zero code maintenance.** I haven't touched my routing config since I wrote those model descriptions. The SR learns from the descriptions, not from rules I have to keep updating.

pubDatetime: 2026-06-22T00:00:00Z
description: "Giving AgentGateway a Semantic Brain with vLLM Semantic Router - Inside My Homelab"
tags:
  - blog
---

## What's Next

With the routing intelligence sorted, I'm now focused on:

- **Observability**: Wiring up Jaeger and Prometheus to trace every request from Pi → AgentGateway → SR → Upstream LLM and back. The AgentGateway already emits OpenTelemetry-compatible spans — I just need to set up the collectors.
- **More models**: Now that routing is semantic, I can add specialized models (a medical one, a legal one) with just a new model card in YAML. The SR will automatically figure out when to use them.

If you're running a homelab AI setup — or building agents at any scale — the combination of [AgentGateway](https://github.com/agentgateway/agentgateway) + [vLLM Semantic Router](https://github.com/vllm-project/semantic-router) is, in my opinion, the most underrated infrastructure combo in the AI ecosystem right now. It turned my janky Python keyword matcher into a proper ML-powered routing plane.

And it runs on a Mac Mini in my living room. 🏠

---

*Follow me for Part 4, where I'll add full observability to this pipeline and show you exactly what happens when Pi dreams at 3 AM — now with traces.*

`#ai` `#agents` `#architecture` `#opensource`

