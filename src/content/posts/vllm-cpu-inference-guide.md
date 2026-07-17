---
title: I Ran vLLM on a Mac Mini With No GPU — Here's Everything I Learned About Inference
pubDatetime: 2026-07-18T00:00:00Z
description: "A complete, beginner-friendly guide to vLLM: what inference actually is, how to build vLLM from source on an Apple Silicon Mac with no GPU, every command explained, the three errors I hit and fixed, the flags that matter, and real throughput numbers from my living room."
tags:
  - blog
---

Everyone talks about *training* models. Almost nobody explains the part that happens ten million times more often: **inference** — actually running the model to get an answer. That's the part you pay for, the part your users wait on, and the part that decides whether your AI feature feels instant or sluggish.

So this weekend I sat down with [vLLM](https://github.com/vllm-project/vllm) — the serving engine that half the industry runs in production — and did the most honest thing I could: I built it from source on a **Mac Mini with no GPU**, ran a real model, captured every log, and wrote down what each command actually does. No CUDA. No cloud. Just an Apple M4, 16GB of RAM, and a lot of patience.

This post is the guide I wish I'd had. By the end you'll understand what inference *is*, what makes vLLM special, how to run it yourself, which flags actually matter, and roughly how fast a tiny model goes on plain CPU. And yes — I hit three errors on the way, and I'm leaving all of them in, because you'll probably hit them too.

---

## First, What Is "Inference" Anyway?

If you've only ever called an API, inference feels like magic: you send text, you get text back. But underneath, a language model does exactly one thing — it predicts the next token (a token is roughly a word-piece), over and over, until it decides to stop.

There are really two phases, and knowing them explains everything vLLM does:

1. **Prefill** — the model reads your entire prompt at once and builds an internal state for it. This is compute-heavy but happens in one big parallel gulp.
2. **Decode** — the model generates the answer one token at a time. Each new token depends on all the tokens before it, so this part is sequential and it's where most of the wall-clock time goes.

The trick that makes decode fast is the **KV cache**. When the model processes a token, it computes "key" and "value" vectors for every layer. Instead of recomputing those for the whole sequence on every step, it *caches* them and only computes the new token. That cache is the single biggest consumer of memory during inference, and — spoiler — managing it well is exactly what made vLLM famous.

Keep those three words in your head: **prefill, decode, KV cache.** Everything below is a story about them.

---

## What Makes vLLM Different

Before I ran anything, here's the one-paragraph version of why vLLM exists, because it framed everything I saw in the logs.

Naive inference servers reserve a big contiguous block of memory for each request's KV cache, sized for the *maximum* possible length. Most requests never use all of it, so you waste huge amounts of memory and can't fit many requests at once. vLLM's headline invention is **PagedAttention**: it treats the KV cache like an operating system treats RAM — chopped into small fixed-size *blocks* (pages) that get handed out on demand. No giant reservation, almost no waste.

That one idea unlocks the second big feature: **continuous batching**. Because memory is paged, vLLM can pack many requests into the same forward pass, and — crucially — swap finished requests out and new ones in *mid-flight*, without waiting for the whole batch to complete. Requests don't wait in line for a slow neighbor. This is why the same hardware serves dramatically more throughput under vLLM than under a plain `model.generate()` loop.

You'll see both of these show up as real numbers later. Now let's build it.

---

## My Setup

Here's what I ran everything on — deliberately modest hardware, because the whole point is to demystify this:

| Thing | Value |
|---|---|
| Machine | Mac Mini, Apple **M4**, 10 cores |
| RAM | 16 GB (shared, no discrete GPU) |
| OS | macOS 26.5 |
| Python | 3.12.13 (via Homebrew) |
| vLLM | built from source, commit `bf578e1` |
| PyTorch | 2.11.0 (CPU build) |
| Model | `Qwen/Qwen2.5-0.5B-Instruct` (a 0.5-billion-parameter chat model) |

A note on the model choice: 0.5B is *tiny* by 2026 standards. I picked it on purpose. On a GPU-less 16GB machine, a small model is the difference between "runs in a minute" and "swaps to disk and dies." The concepts are identical whether it's 0.5B or 70B — only the numbers change.

---

## Step 1: Get Python and Clone vLLM

vLLM officially supports Python 3.10–3.14, but my system Python was 3.14 and the prebuilt wheels for the CPU path were happiest on 3.12, so I installed that first:

```bash
brew install python@3.12
```

**What this does:** installs a clean, isolated Python 3.12 interpreter via Homebrew. This matters because you never want to install a heavy project like vLLM into your system Python — one bad dependency and you've broken other tools.

Then clone the source. On a Mac with no NVIDIA GPU, you can't just `pip install vllm` and get the fast prebuilt wheel (those are compiled for CUDA). You build the **CPU backend** from source, so you need the repo:

```bash
git clone --depth 1 https://github.com/vllm-project/vllm.git
```

**What `--depth 1` means:** a *shallow* clone. It grabs only the latest commit instead of the project's entire multi-year history. vLLM's history is large; this turned a long download into a few seconds. You lose the ability to `git log` back in time, which you don't need here.

Now make an isolated virtual environment and put the build tools in it:

```bash
python3.12 -m venv .venv
.venv/bin/pip install --upgrade pip setuptools wheel
```

**What a venv is:** a self-contained folder with its own copy of Python and its own installed packages. Activating it (or calling `.venv/bin/pip` directly, as I do) means every install lands *here*, not in your global Python. This is the single most important habit in Python — one venv per project, always.

---

## Step 2: Install the Dependencies

vLLM splits its requirements by hardware target. There's `cuda.txt` for NVIDIA, `rocm.txt` for AMD, and — the one I want — `cpu.txt`. There's also a separate file for the *build* tools:

```bash
.venv/bin/pip install -r vllm/requirements/build/cpu.txt
.venv/bin/pip install -r vllm/requirements/cpu.txt
```

**What these do:** the first installs the machinery needed to *compile* vLLM (CMake config, the right setuptools, etc.). The second installs everything vLLM needs to *run* on CPU — most importantly the CPU build of PyTorch (`torch==2.11.0`, no `+cu` CUDA suffix), plus `transformers`, `numpy`, the `openai` client, and dozens of others.

This is the big download — PyTorch alone is hundreds of megabytes. I ran it in the background and went to read the docs. When it finished, `pip` had pulled in the whole scientific-Python stack.

---

## Step 3: Compile vLLM (The Long Part)

This is the command that actually builds vLLM's C++ and Rust guts against your CPU:

```bash
cd vllm
VLLM_TARGET_DEVICE=cpu ../.venv/bin/pip install -e . --no-build-isolation
```

Let me break this one down piece by piece, because it's the heart of the whole thing:

- **`VLLM_TARGET_DEVICE=cpu`** — an environment variable that tells vLLM's build system "compile the CPU kernels, not CUDA." Without it, the build looks for a CUDA toolkit, doesn't find one, and fails. This is the single flag that makes a GPU-less build possible.
- **`pip install -e .`** — the `-e` means *editable* (or "develop") install. Instead of copying files into `site-packages`, it links the package back to this source folder. Handy if you want to poke at vLLM's source later — your edits take effect without reinstalling.
- **`--no-build-isolation`** — normally pip builds a package in a throwaway clean environment. That would re-download build dependencies I *already* installed in Step 2. This flag says "use the tools already in my venv," which is both faster and necessary here since vLLM's CPU build expects the specific setuptools I pinned.

On my M4 this compiled in **2 minutes 55 seconds**, pegging the cores at ~400% CPU (it parallelizes across them). The tail of the log is the sentence you want to see:

```plaintext
Successfully built vllm
Successfully installed vllm-0.1.dev1+gbf578e1ab.cpu
```

That `.cpu` suffix on the version is your confirmation you built the right backend.

---

## The Three Errors I Hit (And Exactly How I Fixed Them)

Here's where the honesty pays off. My first three attempts to run inference all failed. Each error is common enough that you'll likely meet at least one, so here's each with its real message and its fix.

### Error 1: "cannot import name 'LLM' from 'vllm'"

My first script sat in the same folder as the cloned `vllm/` repo, and importing failed with `ImportError: cannot import name 'LLM' from 'vllm' (unknown location)`.

The tell is **"unknown location."** When you run Python from a directory that contains a folder literally named `vllm`, Python finds *that folder* first and treats it as the package — but the repo root isn't the package (the real code is in `vllm/vllm/`). So `import vllm` resolves to an empty shell with no `LLM` in it.

**The fix:** run your scripts from anywhere *except* the folder holding the clone. I moved mine into an `examples/` subdirectory:

```bash
mkdir examples && cd examples
../.venv/bin/python 01_offline_basic.py
```

Once I did, `import vllm` correctly resolved to the installed package. Small thing, ten minutes lost, worth knowing.

### Error 2: "An attempt has been made to start a new process before... bootstrapping"

Next run got further, then blew up with a wall of multiprocessing traceback ending in:

```plaintext
RuntimeError: An attempt has been made to start a new process before the
current process has finished its bootstrapping phase.
```

Here's why. vLLM's engine doesn't run in your script's process — it **spawns a separate `EngineCore` worker process**. On macOS (and Windows), Python creates that child with the *spawn* method, which works by **re-importing your script from the top** inside the child. If your `LLM(...)` call sits at the top level of the file, the child re-runs it, tries to spawn *another* engine, and you get infinite recursion — which Python catches and turns into that error.

**The fix** is the classic Python multiprocessing idiom: put your code inside a function and guard it.

```python
def main():
    llm = LLM(model="Qwen/Qwen2.5-0.5B-Instruct", ...)
    # ... generate ...

if __name__ == "__main__":
    main()
```

The `if __name__ == "__main__"` line means "only run this when the file is executed directly, not when it's imported." The spawned child imports the file, sees it's *not* main, and skips straight past — no recursion. On Linux this often isn't needed (it uses *fork*), which is exactly why so many tutorials omit it and then break on a Mac.

### Error 3: "Available memory... is less than desired CPU memory utilization"

Third run got *all* the way to loading the model, then:

```plaintext
ValueError: Available memory on node 0 (3.86/16.0 GiB) on startup is less than
desired CPU memory utilization (0.92, 14.72 GiB). On the CPU backend, the
`--gpu-memory-utilization` flag controls the fraction of CPU memory reserved
(despite its name).
```

This one is a genuinely confusing bit of vLLM's design, and the error message (to its credit) spells it out. There's a flag called `gpu_memory_utilization` that defaults to `0.92` — reserve 92% of memory for the KV cache pool. On the CPU backend, **that same flag controls system RAM**, despite the "gpu" in its name. So vLLM tried to grab 14.7 GB of my 16 GB, found only 3.86 GB actually free (browsers, mostly), and refused to start.

**The fix:** cap it. In the Python API:

```python
llm = LLM(
    model="Qwen/Qwen2.5-0.5B-Instruct",
    gpu_memory_utilization=0.2,   # ~3.2 GB of 16 GB — plenty for a 0.5B model
)
```

or on the server, `--gpu-memory-utilization 0.2`. A 0.5B model's weights are only ~0.9 GB, so 3.2 GB leaves comfortable room for the KV cache. After this, it ran.

> One more harmless thing you'll see on every Mac run: `Triton not installed or not compatible`. Triton is a GPU kernel compiler; there's nothing to install on a Mac and nothing is wrong. Ignore it.

---

## Step 4: The First Real Inference

Here's the whole script (`01_offline_basic.py`). This is "offline" or "batch" inference — no server, you just import vLLM and call it, like any other library.

```python
import time
from vllm import LLM, SamplingParams

def main():
    prompts = [
        "The capital of India is",
        "Explain what a KV cache is in one paragraph:",
        "Write a haiku about running LLMs on a Mac mini:",
        "The three laws of robotics are",
    ]

    sampling_params = SamplingParams(
        temperature=0.7,   # randomness: 0 = always pick the top token, higher = more varied
        top_p=0.95,        # nucleus sampling: only consider tokens in the top 95% of probability
        max_tokens=100,    # stop after at most 100 generated tokens
    )

    llm = LLM(
        model="Qwen/Qwen2.5-0.5B-Instruct",
        max_model_len=2048,          # biggest prompt+output we'll allow (smaller = less KV memory)
        enforce_eager=True,          # skip graph compilation for a faster cold start
        gpu_memory_utilization=0.2,  # on CPU this caps RAM (see Error 3)
    )

    outputs = llm.generate(prompts, sampling_params)
    for o in outputs:
        print(o.prompt, "→", o.outputs[0].text)

if __name__ == "__main__":
    main()
```

The two objects you'll use constantly:

- **`LLM(...)`** — the engine. Creating it loads the weights, profiles memory, and carves out the KV cache. This is slow (it's a one-time cost) and you reuse the object for every request.
- **`SamplingParams(...)`** — *how* to generate. `temperature` and `top_p` control creativity; `max_tokens` caps length. These are the same knobs you set in the OpenAI API, just as a Python object.

Watching the logs stream by is the best way to *see* the concepts from the top of this post become real. Here are the lines that matter, annotated:

```plaintext
Loading weights took 1.61 seconds
Warming up model for the compilation...
Warming up done.                                    ← ~27s of one-time warmup
Auto set (1.25/16.0) GiB for KV cache on node 0     ← PagedAttention sizing its pool
GPU KV cache size: 109,440 tokens                   ← how many tokens it can hold at once
Maximum concurrency for 2,048 tokens per request: 53.44x   ← ~53 full-length requests in parallel
init engine (profile, create kv cache, warmup) took 27.47 s
```

Read that KV cache line again, because it's PagedAttention talking: from a mere 1.25 GB pool, vLLM can juggle **109,440 tokens** — enough for ~53 simultaneous max-length requests. That "53x concurrency" number is the whole reason vLLM exists, printed right there at startup.

And the output? The model answered all four prompts. It's a 0.5B model so it's charmingly unreliable — it confidently claimed the three laws of robotics are "Momentum, Forces, and Acceleration" — but the *mechanics* are flawless. The speed line:

```plaintext
--- 400 tokens in 6.47s = 61.8 tok/s across 4 prompts ---
```

**61.8 tokens per second on a CPU, no GPU.** For reference, that's faster than most people read. On a laptop-class chip. For a model you can run entirely offline and private.

---

## Step 5: Watching Continuous Batching Earn Its Name

The single-prompt number is fine, but vLLM's real party trick is throughput under load. So the second script (`02_offline_batch.py`) fires **32 prompts at once** — "explain X in one sentence" for 32 different topics — and measures total throughput.

Same model, same machine, same everything. The result:

```plaintext
--- 32 prompts, 1920 tokens in 16.82s = 114.1 tok/s ---
```

Look at that against the single-stream run:

| Run | Prompts | Throughput |
|---|---|---|
| Basic | 4 (short) | 61.8 tok/s |
| Batch | 32 at once | **114.1 tok/s** |

**Nearly double the tokens per second, on identical hardware**, purely because vLLM packed all 32 requests through the model together instead of one at a time. That's continuous batching. On a GPU with a bigger model the gap is far more dramatic — often 10–20× — because GPUs love wide batches even more than CPUs do. This is the single most important thing to understand about serving: **throughput is not one request times N; batching changes the physics.**

---

## Step 6: Serving It Like a Real API

Batch scripts are great for offline jobs. But usually you want an always-on server that speaks the **OpenAI API**, so any existing tool or SDK can point at it. That's one command:

```bash
vllm serve Qwen/Qwen2.5-0.5B-Instruct \
  --max-model-len 2048 \
  --gpu-memory-utilization 0.2 \
  --enforce-eager \
  --port 8000
```

**What `vllm serve` does:** boots the same engine, but wraps it in a web server exposing OpenAI-compatible endpoints — `/v1/completions`, `/v1/chat/completions`, `/v1/models`, plus a `/metrics` endpoint for monitoring. The flags are the CLI twins of the Python arguments above. After ~30 seconds of warmup you get:

```plaintext
INFO:     Application startup complete.
```

Now it's just an API. Ask it what models it serves:

```bash
curl http://localhost:8000/v1/models
```

Send a chat request in the exact shape you'd send to OpenAI — same JSON, same fields:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-0.5B-Instruct",
    "messages": [{"role": "user", "content": "Give me one tip for learning Rust."}],
    "max_tokens": 80
  }'
```

It came back with a tidy answer and, importantly, a **usage** block: `37 prompt + 80 completion tokens`. That token accounting is built in — the same numbers you'd get billed for on a cloud provider, here for free.

Because it's OpenAI-shaped, the official `openai` Python SDK works with a one-line base-URL swap. The only thing to know: vLLM ignores the API key unless you started it with `--api-key`, but the SDK still requires *some* non-empty string:

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

resp = client.chat.completions.create(
    model="Qwen/Qwen2.5-0.5B-Instruct",
    messages=[{"role": "user", "content": "Why is the sky blue? Two sentences."}],
)
print(resp.choices[0].message.content)
```

I also measured the thing users actually feel — **time to first token** — by streaming the response:

```plaintext
time to first token: 214ms, total: 5.40s
```

214 milliseconds before the first word appears. That's the latency metric that makes a chat UI feel alive versus dead, and vLLM exposes it directly when you stream (`stream=True`).

---

## The Flags That Actually Matter

vLLM has a *lot* of flags (`vllm serve --help=all` is genuinely overwhelming). Here are the ones I reached for, and when you'd touch each. These work both as `LLM(...)` Python arguments (with underscores) and `vllm serve` CLI flags (with dashes).

| Flag | What it controls | When to change it |
|---|---|---|
| `--model` | Which model to load (a HuggingFace ID or local path) | Always — it's the one required argument |
| `--gpu-memory-utilization` | Fraction of memory reserved for the KV cache pool. **On CPU this means system RAM.** | Lower it if you get out-of-memory at startup; raise it (toward 0.9) to serve more concurrent requests |
| `--max-model-len` | Max tokens per request (prompt + output combined) | Lower it to save KV cache memory; raise it if you need long context and have the RAM |
| `--max-num-seqs` | Max requests processed in one iteration (batch width) | Lower on tight memory; raise to push throughput when you have headroom |
| `--dtype` | Numeric precision of weights (`auto`, `bfloat16`, `float16`, `float32`) | `auto` is right almost always; force one only to work around a specific hardware quirk |
| `--quantization` | Load a compressed model (AWQ, GPTQ, etc.) | To fit a bigger model in less memory, at a small quality cost |
| `--tensor-parallel-size` | Split one model across N GPUs | Multi-GPU only — leave at 1 on a Mac |
| `--enforce-eager` | Skip graph compilation (CUDA graphs / torch.compile) | On for faster cold starts and debugging; off in production for max steady-state speed |
| `--api-key` | Require a key in request headers | Any time the server is reachable by anything but you |
| `--port` | Which port to serve on | To avoid clashes / run multiple servers |

If you remember only two: **`--gpu-memory-utilization`** is your pressure-release valve when memory is tight, and **`--max-model-len`** is the other big memory lever. Between them you can squeeze vLLM onto surprisingly small machines — like, well, a Mac Mini.

---

## Observability Comes Free

One thing I didn't expect to love: the `/metrics` endpoint. vLLM emits **Prometheus metrics** out of the box, so you can see exactly what the engine is doing:

```plaintext
vllm:prompt_tokens_total          118.0
vllm:generation_tokens_total      399.0
vllm:num_requests_running         0.0
vllm:num_requests_waiting         0.0
vllm:request_success_total{finished_reason="stop"}     1.0
vllm:request_success_total{finished_reason="length"}   3.0
```

You can read the story of my session right there: 118 prompt tokens in, 399 generated out, one request that stopped naturally and three that hit the `max_tokens` length cap. Point Grafana at this and you've got a production dashboard with zero extra code. For anyone who's tried to bolt monitoring onto a homegrown inference loop, getting this for free is a small joy.

---

## The Short Version

If you skimmed, here's the whole thing in one breath:

1. **Inference = predicting the next token, repeatedly.** Prefill reads your prompt, decode writes the answer one token at a time, and the KV cache is what keeps decode fast.
2. **vLLM's superpower is memory.** PagedAttention pages the KV cache like an OS pages RAM, which enables continuous batching, which is why it serves so much more throughput than a naive loop.
3. **Building on a Mac is: `brew install python@3.12`, shallow-clone the repo, make a venv, install the `cpu.txt` requirements, then `VLLM_TARGET_DEVICE=cpu pip install -e . --no-build-isolation`.** ~3 minutes to compile on an M4.
4. **Three errors to expect:** the `vllm/` folder shadowing your import (run from elsewhere), the missing `if __name__ == "__main__"` guard (Macs use spawn, not fork), and `gpu_memory_utilization` defaulting too high (it means RAM on CPU — set it to 0.2).
5. **Two ways to run it:** offline (`from vllm import LLM`) for batch jobs, or `vllm serve` for an always-on OpenAI-compatible API.
6. **The numbers, on CPU, no GPU:** ~62 tok/s single-stream, ~114 tok/s batching 32 prompts, 214 ms to first token. All private, all offline, all on a machine that fits in your palm.

The thing that stuck with me: I always assumed "you need a GPU" was a hard wall for running LLMs. It's not. It's a speed dial. A 0.5B model on a Mac Mini is genuinely usable, and the *exact same commands and flags* scale up to a rack of H100s serving thousands of users. Learn it small, run it anywhere.

And yes — this one also runs in my living room. 🏠

`#ai` `#vllm` `#inference` `#llm` `#macmini` `#opensource` `#buildinpublic`
